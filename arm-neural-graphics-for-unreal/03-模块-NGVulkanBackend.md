# 模块文档 03:NGVulkanBackend(Vulkan 后端)

> 源码位置:`Source/NGVulkanBackend/`(三个引擎分支均有)。
> 依赖:Private `Core/Engine/Projects/RenderCore/Renderer/RHI/RHICore/NGShared/VulkanRHI`;
> Windows/Linux/Android 平台下加引擎 Vulkan 第三方依赖与 `VulkanRHI/Private` 平台子目录;
> 链接 `Binaries/ThirdParty/PreBuiltNGSDK/` 下的 SDK 库。

## 1. 职责

1. **加载 NG-SDK 动态库**(`ngsdk_windows_x64.dll` / `libngsdk_linux_{x64,arm}.so` / `libngsdk_android_arm.so`)
   并通过 `ffxLoadFunctions` 解析 FFX 函数表。
2. **在引擎初始化前注册 Vulkan 扩展与特性层**(`VK_ARM_tensors`、`VK_ARM_data_graph`、
   `VK_ARM_data_graph_optical_flow` 及依赖扩展、`VK_LAYER_ARM_NG` 层)。
3. **实现 `INGSharedBackend`**:把 UE 的 RHI 对象(VkDevice/VkInstance/纹理/命令缓冲)桥接为
   NG-SDK 可消费的原生对象。
4. 提供 **速度场转换着色器** `NGConvertVelocity`(UE 稀疏速度 → FFX 稠密速度)。
5. 负责 **Android 打包**(APL:拷贝 .so、loadLibrary)。

## 2. 原理与算法分析

### 2.1 扩展/层注册为什么必须在 RHI 初始化前

Vulkan 的设备扩展列表在 `vkCreateDevice` 时**一次性确定**,之后不可追加。SDK 的神经网络执行
依赖 `VK_ARM_tensors`/`VK_ARM_data_graph` 设备扩展;若设备创建时没启用,运行期查询必然失败。
UE 的 VulkanRHI 提供 `IVulkanDynamicRHI::AddEnabledDeviceExtensionsAndLayers` 钩子,把插件要的
扩展并入引擎自己的创建参数——所以这段代码放在模块 `StartupModule`(引擎 RHI 尚未初始化)立即执行。

**特性层(Vulkan layer)机制**:`VK_LAYER_ARM_NG` 是**实例层**,由 Vulkan loader 加载:
- Windows/Linux:loader 通过环境变量 `VK_LAYER_PATH` 指向的 `VkLayer_arm_NG.json` 清单发现层,
  清单声明层名与库路径(因此插件把 json+dll/so 随包部署到游戏目录,并要求用户设置 VK_LAYER_PATH);
- Android:loader(API≥28)自动扫描 app 的 `nativeLibraryDir` 下 `libVkLayer_*.so`,无需显式注册。
层的作用:为 SDK 补足驱动缺失的扩展行为(如桌面模拟环境的 tensor/data_graph 语义)。

### 2.2 速度场转换的算法与约定

UE 的 `SceneVelocity` 是**稀疏编码**:只在有运动的像素写入速度(编码在 `x` 通道,`x>0` 表示有效),
且是**半分辨率**、方向与 FSR 系列约定相反。SDK 需要**全分辨率稠密**速度场,因此必须先转换:

```
逐像素:
  1) 若 EncodedVelocity.x > 0(该像素有运动):解码得到真实运动速度;
  2) 否则(静态像素):用深度 + View.ClipToPrevClip 计算"相机运动引起的视差速度"
        PrevClip = mul(float4(ScreenPos, Depth, 1), View.ClipToPrevClip)
        velocity = PosN - PrevClip.xyz/PrevClip.w
  3) 输出 = velocity * (-0.5, 0.5)
```

- 为什么 `ClipToPrevClip`:它是"当前裁剪空间 → 上一帧裁剪空间"的投影矩阵,把当前像素坐标
  变换到上一帧位置,差值即两帧间的视差——这是对静态场景相机运动速度的解析解;
- 为什么乘 `(-0.5, 0.5)`:① UE 速度方向与 FSR 约定相反(负号);② FSR 速度按 UV 半像素单位
  (半分辨率纹理的 0.5 归一)。两步合并为一个系数。

### 2.3 深度反转(Reversed-Z)约定

UE 默认使用 D3D 风格 reversed-Z(近处深度值大),而 SDK 需要知道深度方向才能正确重建
`cameraNear/cameraFar` 语义。约定:

- `ERHIZBuffer::IsInverted` 为真(反转深度)时,SDK 描述里 `cameraNear = FLT_MAX`(无限远)、
  `cameraFar = 近平面`(数值上:深度=1 是近处);
- 非反转则 `cameraNear = 近平面`、`cameraFar = FLT_MAX`。
这个约定同时出现在 NSS 的 dispatch 参数与 NFRU 的 context 描述中,必须全局一致。

### 2.4 为什么需要 Flush(RHIFinishExternalComputeWork)

SDK 的 dispatch 直接录制在 UE 的**活动 VkCommandBuffer** 上。UE 的 RHI 不知道这段外部命令,
后续的布局转换/同步可能漏掉它。`Flush` 通过 `RHIFinishExternalComputeWork` 告知 RHI:
"外部计算已提交,请插入同步"。漏掉会导致数据竞争、画面撕裂或挂起。

## 3. 模块启动(StartupModule)

```cpp
void NGVulkanBackendModule::StartupModule()
{
    // 1) 注册着色器虚拟路径:/Plugin/NG → 插件根/Shaders/
    //    引擎编译 USF 时按 /Plugin/NG/... 解析到插件目录,必须与 IMPLEMENT_GLOBAL_SHADER 的路径一致
    FString PluginShaderDir = FPaths::Combine(
        IPluginManager::Get().FindPlugin(TEXT("NG"))->GetBaseDir(), TEXT("Shaders"));
    AddShaderSourceDirectoryMapping(TEXT("/Plugin/NG"), PluginShaderDir);

    // 2) 引擎完全初始化后检查 RHI(此刻 GDynamicRHI 才可用)
    FCoreDelegates::OnPostEngineInit.AddLambda(
        []() { NGVulkanBackend::sNGVulkanBackend.PostEngineInit(); });

    // 3) 加载 SDK DLL
    if (NGVulkanBackend::sNGVulkanBackend.LoadDLL())
    {
        // 4) 注册实例层 + 设备扩展(原理见 §2.1:必须在 vkCreateDevice 之前!)
        TArray<const ANSICHAR*> Extensions;
        Extensions.Add("VK_ARM_tensors");               // tensor 资源(神经网络输入输出载体)
        Extensions.Add("VK_ARM_data_graph");            // 数据图执行(神经网络推理)
        Extensions.Add("VK_ARM_data_graph_optical_flow"); // 光流加速(NFRU 运动估计)
        Extensions.Add("VK_KHR_maintenance5");          // 上述扩展的依赖
        Extensions.Add("VK_KHR_deferred_host_operations");
        Extensions.Add("VK_KHR_synchronization2");

        static const ANSICHAR* Layers[] = {"VK_LAYER_ARM_NG"};
        IVulkanDynamicRHI::AddEnabledInstanceExtensionsAndLayers(
            TArrayView<const ANSICHAR* const>(), TArrayView<const ANSICHAR* const>(Layers, 1));
        IVulkanDynamicRHI::AddEnabledDeviceExtensionsAndLayers(Extensions, TArrayView<const ANSICHAR* const>());
    }
    else
    {
        // 加载失败:报错日志;Windows 非 cook 时弹窗提示 DLL 缺失
    }
}
```

要点:

- **扩展注册的时机**必须在 Vulkan 设备创建前,因此放在模块 Startup(PostConfigInit 阶段)立即执行。
- Windows/Linux 特性层需要 `VkLayer_arm_NG.json` 清单(随 RuntimeDependencies 部署在游戏目录,
  并需用户设置 `VK_LAYER_PATH`);Android 由 loader 自动发现 `libVkLayer_*.so`。

## 4. DLL 加载(LoadDLL)

```cpp
bool LoadDLL()
{
    FString Name;
#if PLATFORM_WINDOWS
    Name = TEXT("ngsdk_windows_x64.dll");
#elif PLATFORM_LINUX
    Name = PLATFORM_CPU_ARM_FAMILY ? TEXT("libngsdk_linux_arm.so") : TEXT("libngsdk_linux_x64.so");
#elif PLATFORM_ANDROID
    Name = TEXT("libngsdk_android_arm.so");
#endif

#if PLATFORM_WINDOWS && WITH_EDITOR
    // 编辑器下把插件 Binaries/ThirdParty/PreBuiltNGSDK/ 加入 DLL 搜索路径
    // (否则编辑器加载的是项目 Binaries 目录里的拷贝,可能与插件版本不一致)
    FPlatformProcess::AddDllDirectory(Dir);
    Name = FPaths::Combine(Dir, Name);
#endif
    FfxModule = FPlatformProcess::GetDllHandle(*Name);
    if (FfxModule) {
        ffxLoadFunctions(&FfxFunctions, (FfxModuleHandle)FfxModule);  // 解析函数表
        bOk = FfxFunctions.CreateContext ? true : false;   // 以关键函数存在性校验导出完整性
    }
    bLoaded = bOk;
    return bOk;
}
```

- `FfxFunctions` 是 `ffxFunctions` 结构体(函数指针集合,SDK 导出 `ffxLoadFunctions` 填充)。
- 所有 `ffx*` 接口方法都直接转发 `FfxFunctions.*`。

## 5. INGSharedBackend 实现(核心类 NGVulkanBackend)

单例:`static NGVulkanBackend sNGVulkanBackend;`(模块 `GetBackend()` 返回其地址)。

### 5.1 ffxCreateContext —— 绑定 Vulkan 设备

```cpp
ffxReturnCode_t ffxCreateContext(ffxContext* context, ffxCreateContextDescHeader* desc) final
{
    ffxCreateBackendVKDesc VulkanHeader = {};
    VulkanHeader.header.type = FFX_API_CREATE_CONTEXT_DESC_TYPE_BACKEND_VK;
    // 从 UE 的 Vulkan RHI 取出原生句柄(这是"桥接"的核心动作)
    VulkanHeader.vkDevice = GetIVulkanDynamicRHI()->RHIGetVkDevice();
    VulkanHeader.vkPhysicalDevice = GetIVulkanDynamicRHI()->RHIGetVkPhysicalDevice();
    VulkanHeader.vkInstance = GetIVulkanDynamicRHI()->RHIGetVkInstance();
    // 函数指针必须从 RHI 取,保证与 RHI 内部加载的 Vulkan 库一致(避免混用两套 vkGet*ProcAddr)
    VulkanHeader.vkGetInstanceProcAddr =
        (PFN_vkGetInstanceProcAddr)GetIVulkanDynamicRHI()->RHIGetVkInstanceProcAddr("vkGetInstanceProcAddr");
    VulkanHeader.vkDeviceProcAddr =
        (PFN_vkGetDeviceProcAddr)GetIVulkanDynamicRHI()->RHIGetVkDeviceProcAddr("vkGetDeviceProcAddr");
    desc->pNext = (ffxApiHeader*)&VulkanHeader;   // 通过 pNext 链附加后端描述
    return FfxFunctions.CreateContext(context, desc, &AllocCbs.Cbs);
}
```

- `GetIVulkanDynamicRHI()` 来自 `IVulkanDynamicRHI.h`(Public 依赖 `VulkanRHI`)。
- 这是整个桥接最关键的一步:SDK 需要实例/设备/函数指针才能创建内部 Vulkan 管线与 tensor 图。

### 5.2 GetNativeResource —— 纹理桥接

```cpp
FfxApiResource GetNativeResource(FRHITexture* Texture, FfxApiResourceState State) final
{
    // 仅支持 Texture2D(SDK 的上采样/插值输入输出都是 2D)
    const auto imgFormat = (VkFormat)GetNativeTextureFormat(Texture);  // FVulkanTexture::ViewFormat
    FfxApiResource resource = {};
    resource.resource = Texture->GetNativeResource();                 // VkImage
    resource.state = State;
    resource.description = { width, height, depth=1, mipCount,
                             ffxApiGetSurfaceFormatVK(imgFormat),     // VkFormat → FFX 格式枚举
                             usage };
    // usage 按需加 DEPTHTARGET/STENCILTARGET/RENDERTARGET/UAV
    if (State & FFX_API_RESOURCE_STATE_RENDER_TARGET)      usage |= RENDERTARGET;
    if (State & FFX_API_RESOURCE_STATE_UNORDERED_ACCESS)   usage |= UAV;
    return resource;
}
```

> 为什么用 `ViewFormat` 而非 `Format`:UE 纹理创建时可能指定了与存储格式不同的**视图格式**
> (如 R8G8B8A8 存储 + R8G8B8A8_SRGB 视图),SDK 采样/写入必须匹配视图格式才能拿到正确语义。

### 5.3 GetNativeCommandBuffer / Flush

```cpp
FfxCommandList GetNativeCommandBuffer(FRHICommandListImmediate&, FRHITexture*) final
{
    return GetIVulkanDynamicRHI()->RHIGetActiveVkCommandBuffer();  // 当前活动 VkCommandBuffer
}
void Flush(FRHITexture* Tex, FRHICommandListImmediate& RHICmdList) final
{
    RHICmdList.EnqueueLambda([this, Tex](FRHICommandListImmediate& cmd) {
        // 告知 RHI"外部计算已提交到活动命令缓冲",插入同步(原理见 §2.4)
        GetIVulkanDynamicRHI()->RHIFinishExternalComputeWork(
            static_cast<VkCommandBuffer>(GetNativeCommandBuffer(cmd, Tex)));
    });
}
```

### 5.4 能力探测

```cpp
bool IsNeuralGraphicSupported()
{
    // 枚举设备扩展,要求 VK_ARM_tensors && VK_ARM_data_graph 同时存在(结果静态缓存)
    // 任一缺失 → 本设备无法跑神经网络渲染,插件应禁用 NSS/NFRU
}
bool EnsureSupportedRHI() { return bVulkanRHI; }   // GDynamicRHI 名称为 "Vulkan"
void PostEngineInit()
{
    bVulkanRHI = !GDynamicRHI || FString(GDynamicRHI->GetName()) == TEXT("Vulkan");
    if (!bVulkanRHI) { /* 弹窗:请改用 Vulkan RHI */ }
}
```

### 5.5 占位接口

`GetInterpolationOutput`/`GetInterpolationCommandList`/`RegisterFrameResources`/`CopySubRect`
均为 `check(false)` 占位——当前插件用 UE 原生 Swapchain 与自定义 Present 实现帧生成,
不走 FFX 的 swapchain 接管路径(`bUseFFXSwapchain = false`)。

## 6. 速度场转换:NGConvertVelocity

### 6.1 C++ 侧(NGConvertVelocity.h/.cpp)

```cpp
class NGConvertVelocity : public FGlobalShader {
    // FParameters: InputDepth/InputVelocity(SRV)、InvContentSize、View(统一缓冲)、RenderTargets[0]
    static bool ShouldCompilePermutation(...) { return IsFeatureLevelSupported(Platform, ES3_1); }
};
IMPLEMENT_GLOBAL_SHADER(NGConvertVelocity, "/Plugin/NG/Private/NGConvertVelocity.usf", "main", SF_Pixel);

NGVULKANBACKEND_API void helper_NGConvertVelocity(
    FRDGBuilder& GraphBuilder,
    FRDGTextureRef MotionVectorTexture,   // 输出:PF_G16R16F 稠密速度
    FRDGTextureRef SceneDepth, FRDGTextureRef SceneVelocity,
    FIntRect ViewRect, const FViewInfo* View)
{
    // 1) 建 SRV:深度、速度作为着色器输入
    // 2) 绑定 View 统一缓冲(着色器需要 View.ClipToPrevClip / View.ViewRectMin)
    // 3) 输出绑到 RenderTargets[0](全屏像素着色器路径)
    // 4) FPixelShaderUtils::AddFullscreenPass 一次性执行全屏 PS
}
```

### 6.2 着色器语义(NGConvertVelocity.usf)

```hlsl
float3 ComputeStaticVelocity(float2 ScreenPos, float DeviceZ)
{
    float4 ThisClip = float4(ScreenPos, DeviceZ, 1);
    float4 PrevClip = mul(ThisClip, View.ClipToPrevClip);   // 重投影到上一帧裁剪空间
    float3 PrevScreen = PrevClip.xyz / PrevClip.w;          // 透视除法还原上一帧屏幕坐标
    return PosN - PrevScreen;                               // 静态(相机运动)速度
}

float2 main(float4 SvPosition : SV_POSITION) : SV_Target0
{
    uint2 Pos = uint2(SvPosition.xy);
    float4 EncodedVelocity = InputVelocity[Pos + View.ViewRectMin.xy];
    if (EncodedVelocity.x > 0.0)
        Velocity = DecodeVelocityFromTexture(EncodedVelocity).xy;  // UE 编码速度(稀疏,有运动像素)
    else
        Velocity = ComputeStaticVelocity(ScreenPos, Depth).xy;     // 深度回退(静态像素,相机视差)
    return Velocity * float2(-0.5, 0.5);   // FSR 约定:负速度 + 半像素归一(原理见 §2.2)
}
```

关键点:

- UE 的 `SceneVelocity` 是**稀疏编码**的(只在有运动的像素写值,`x>0` 表示有效),解码后即真实运动;
  其余像素用深度 + `ClipToPrevClip` 计算相机运动产生的"静态速度"。
- 输出必须乘 `(-0.5, 0.5)`:FFX 期望的方向与 UE 相反,且要换算成 UV 空间半像素单位。
- 4.27 分支为 compute 风格(`NssConvertVelocityPS.usf`,`FNssConvertVelocity` 全局着色器),
  另有 `NssMirrorPad.usf`(`FNssMirrorPadPS`,镜像填充,供 SDK 输入边缘处理)。

## 7. Android 打包(APL)

`NGVulkanBackend_APL.xml`:

```xml
<root xmlns:android="...">
    <init/>
    <resourceCopies>
        <copyFile src="$S(PluginDir)/../../Binaries/ThirdParty/PreBuiltNGSDK/libngsdk_android_arm.so"
                  dst="$S(BuildDir)/libs/arm64-v8a/libngsdk_android_arm.so"/>
        <copyFile src="$S(PluginDir)/../../Binaries/ThirdParty/PreBuiltNGSDK/libVkLayer_arm_NG.so"
                  dst="$S(BuildDir)/libs/arm64-v8a/libVkLayer_arm_NG.so"/>
    </resourceCopies>
    <soLoadLibrary>
        <loadLibrary name="ngsdk_android_arm" failmsg="Could not load ngsdk library"/>
    </soLoadLibrary>
</root>
```

Build.cs 中通过 `AdditionalPropertiesForReceipt.Add("AndroidPlugin", …)` 让 UAT 在打包时执行。
`libVkLayer_arm_NG.so` 进 `libs/arm64-v8a/` 后由 Android loader 自动发现(§2.1)。

## 8. Build.cs 平台分支要点

- **Windows**:链接 `ngsdk_windows_x64.lib`,`PublicDelayLoadDLLs` 加 `ngsdk_windows_x64.dll`,
  `RuntimeDependencies` 部署 DLL;特性层 `VkLayer_arm_NG.dll` + json 同样部署;
  若 `PreBuiltNGSDK` 缺库,则**自动调用 `../../BuildSDK.py -b vk_windows_x64`** 构建。
- **Linux**:按架构选 `libngsdk_linux_{x64,arm}.so`;`VULKAN_SDK` 环境变量决定 Vulkan 头/库来源;
  层文件放 `Linux/` 或 `LinuxArm64/` 子目录。
- **Android**:`libngsdk_android_arm.so`(缺失则自动 BuildSDK 构建)+ 层文件 + APL。

## 9. 复现要点

1. 扩展/层注册必须在 RHI 设备创建前(Startup 立即执行),否则 `VK_ARM_*` 不在设备扩展列表里,
   `IsNeuralGraphicSupported()` 会失败。
2. `ffxCreateContext` 必须把 `ffxCreateBackendVKDesc` 挂在 `desc->pNext` 上;漏掉则 SDK 无设备可用。
3. 纹理桥接必须用 `FVulkanTexture::ViewFormat`(而非 Format),否则格式映射错位。
4. SDK 调用插在 UE 活动命令缓冲上,帧结束前必须 `Flush`(`RHIFinishExternalComputeWork`)。
5. 平台缺失库时自动构建只对 Windows/Android 生效(文件存在性检查),Linux 直接要求预构建产物存在。
6. 速度场转换的 `(-0.5, 0.5)` 与深度反转约定是全局不变的契约,改动会同时破坏 NSS 与 NFRU。
