# 模块文档 01:NGShared(公共层)

> 源码位置:`Source/NGShared/`(三个引擎分支均有;4.27 分支无 `FFXRDGBuilder*`/`FFXSlateApplication*` 复制头)。
> 依赖:Public `Engine`/`RHI`;Private `Core`/`Engine`。IncludePath 含 `NG-SDK/sdk/include` 与 `NG-SDK/ffx-api/include`。

## 1. 职责

NGShared 是插件所有模块的公共底座,承担四类工作:

1. **FFX/NG-SDK 头文件统一封装**(`NGShared.h`、`NGShared.cpp`):屏蔽平台差异、导出宏、
   以及"按引擎分支选择复制头"的宏体系。
2. **后端抽象接口**(`NGSharedBackend.h`):`INGSharedBackend` —— 插件与具体图形 API 之间的
   契约接口,NSS/NFRU 只依赖此接口,不直接触碰 Vulkan。
3. **格式/状态映射**(`NGSharedBackend.cpp`):`GetFFXApiFormat`(UE `EPixelFormat` → `FfxApiSurfaceFormat`)
   与 `GetUEAccessState`(FFX 资源状态 → UE `ERHIAccess`)。
4. **引擎私有成员访问器**(`FFXRDGBuilder*`、`FFXSlateApplication*`):通过复制引擎头文件 + 宏改写,
   让插件能访问引擎类的私有成员(找 Lumen 反射纹理、Slate 渲染器指针/时间戳)。

## 2. 原理与算法分析

### 2.1 为什么需要"后端抽象"接口

NG-SDK 是原生图形 API 的消费者:它的 `ffxDispatch` 需要 **VkImage / VkCommandBuffer / VkDevice**
这类原生句柄;而插件代码运行在 UE 的 RHI 抽象层之上,手里只有 `FRHITexture`/`FRHICommandListImmediate`。
两者之间必须有翻译层,这就是 `INGSharedBackend`:

```
NSS / NFRU(只认识 UE RHI 对象)
   │  ffxCreateContext / ffxDispatch / GetNativeResource / GetNativeCommandBuffer
   ▼
INGSharedBackend(接口契约:UE 对象 ↔ FFX 描述)
   │
   ▼
NGVulkanBackend(具体实现:从 RHI 对象里掏出 VkImage/VkCommandBuffer 等)
```

接口化带来的好处:

- **单一职责**:NSS/NFRU 不包含任何 `#include "VulkanRHI"` 的代码,可独立编译/测试;
- **可扩展**:若未来接入 DX12,只需新增一个 `INGSharedBackend` 实现,业务模块零改动;
- **与 FFX 的 host/device 模型对齐**:FFX 本身把"设备绑定"设计为创建描述上的 `pNext` 扩展
  (`ffxCreateBackendVKDesc`),后端接口正好承载这部分语义。

### 2.2 为什么格式/状态要显式映射

- UE 的 `EPixelFormat` 是**跨 API 抽象**(如 `PF_FloatR11G11B10`),而 FFX 的表面格式枚举是
  **具体数值**(如 `FFX_API_SURFACE_FORMAT_R11G11B10_FLOAT`)。两边枚举值并不同源,必须查表映射。
- **SRGB 是独立枚举值**:`FFX_API_SURFACE_FORMAT_R8G8B8A8_SRGB` ≠ `_UNORM`(SRGB 采样/写入语义不同),
  因此 `GetFFXApiFormat` 必须接收 `bSRGB` 参数区分。
- **资源状态同理**:UE 用 `ERHIAccess` 位掩码描述布局,RHI 用它在渲染图里做自动布局转换;
  FFX 用单值枚举描述状态。`GetUEAccessState` 负责把 SDK 要求的状态翻译成 UE 能识别的访问标志,
  映射错误会导致布局转换缺失 → 驱动校验错误、黑屏或数据错误。

### 2.3 复制头 + 宏改写原理(重点)

需求:插件要读取 `FRDGBuilder` 的私有纹理表、`FSlateApplication` 的私有渲染器指针,但引擎源码不可改。

可行性的根基:**C++ 类布局由成员声明顺序决定,与头文件被包含的位置无关**。
于是可以把引擎头文件逐字节复制到插件内,与引擎头内容完全相同——两份定义布局一致,
只是名字不同(经宏改名)或可见性不同(经 `#define private protected` 改写)。

具体手法:

```cpp
#define private protected        // ① 私有成员 → 受保护:派生类可访问
#define FRDGBuilder FFXRDGBuilderBase  // ② 类改名:避免与引擎同名类冲突
#include FFX_BUILD_VERSIONED_INCLUDE_PATH(RDGBuilder)  // ③ 包含复制头
#undef FRDGBuilder               // ④ 撤销宏,恢复命名空间
#undef private
```

- **为什么 `protected` 而不是 `public`**:只暴露给"我们的派生类",最小化可见性扩散;
- **为什么要改名包含**:引擎的其他头(如 `RenderGraphBuilder.h` 被引擎内部大量引用)在编译插件时
  也会被间接包含;若复制头仍叫 `FRDGBuilder`,会与引擎真类重复定义——改名后两套定义并存,
  插件代码通过 `FFXRDGBuilder` 访问复制版,引擎代码用 `FRDGBuilder` 访问真版;
- **为什么 `static_assert(sizeof(...) == sizeof(...))`**:两份定义必须布局一致,否则通过
  `FFXRDGBuilder*` 强转访问真对象是未定义行为。引擎若改动成员布局,该断言在编译期立即失败,
  强制维护者更新复制头——这是"复制头方案"的防呆保险;
- **为什么要版本化头文件**(`FRDGBuilder_5_4_0.h` 按引擎主次版本命名):不同引擎分支的类布局/接口
  可能不同,复制头必须与目标引擎分支绑定;`FFX_BUILD_VERSIONED_INCLUDE_PATH` 宏在预处理期按
  `ENGINE_MAJOR/MINOR_VERSION` 自动选择对应文件;对超出已知范围的新引擎,用 `#error` 强制显式处理。

### 2.4 为什么注入自定义分配器

NG-SDK 在 `ffxCreateContext` 时会分配大量内部资源。若不注入分配器,SDK 走默认 `malloc/free`;
而 UE 用 `FMemory::Malloc/Free`(可能带自定义对齐、统计、崩溃处理)。混用两个分配体系会造成
内存统计错乱乃至崩溃。`NGSharedAllocCallbacks` 把 UE 分配器桥接为 `ffxAllocationCallbacks`。

## 3. NGShared.h —— 头文件封装与版本宏

```cpp
// 版本比较宏:≥ 指定版本为真。用于跨引擎分支差异分支
#define UE_VERSION_AT_LEAST(MajorVersion, MinorVersion, PatchVersion) \
    UE_GREATER_SORT(ENGINE_MAJOR_VERSION, MajorVersion, \
        UE_GREATER_SORT(ENGINE_MINOR_VERSION, MinorVersion, \
            UE_GREATER_SORT(ENGINE_PATCH_VERSION, PatchVersion, true)))
```

要点:

- `PLATFORM_WINDOWS` 下定义 `FFX_ENABLE_DX12 1` 并包 `Windows/AllowWindowsPlatformTypes.h`;
  非 Windows 定义 `FFX_GCC`。因为 NG-SDK 的 `ffx_api.h` 内部按平台分支编译。
- 通过 `THIRD_PARTY_INCLUDES_START/END` 包裹第三方头,避免 UE 的警告检查。
- 非 Windows 分支 `#define FFX_API __declspec(dllexport)`(Windows 上 SDK 头已声明导入)。
- 包含的 SDK 头:NSS 用 `ffx_nss.h` 与 `FidelityFX/gpu/nss/ffx_nss_resources.h`;
  帧生成用 `ffx_framegeneration.h`、`ffx_frameinterpolation_resources.h`、`ffx_interface.h` 等;
  NGShared 自身主要含 `ffx_api.h`、`ffx_api_types.h`、`ffx_types.h`。
- 引擎版本化包含宏(配合复制头使用):
  ```cpp
  #define FFX_BUILD_INCLUDE_PATH(name, major, minor, patch) \
      F##name##Versions/F##name##_##major##_##minor##_##patch.h
  #define FFX_BUILD_VERSIONED_INCLUDE_PATH(name) \
      FFX_STRINGIFY(FFX_BUILD_INCLUDE_PATH(name, ENGINE_MAJOR_VERSION, ENGINE_MINOR_VERSION, 0))
  ```
  例如 5.4 展开为 `FRDGBuilderVersions/FRDGBuilder_5_4_0.h`、`FSlateApplicationVersions/FSlateApplication_5_4_0.h`。

## 4. NGSharedBackend.h —— 后端抽象

### 4.1 枚举与分配器

```cpp
enum class EFFXBackendAPI : uint8 { Vulkan, Unsupported, Unknown };
// Unknown  = 尚未探测;Unsupported = 探测过但不可用(非 Vulkan RHI / SDK 未加载 / 设备不支持)
// 语义:插件拿 Unknown 时会尝试 Initialize,拿 Unsupported 则直接禁用

struct NGSharedAllocCallbacks {
    // 把 UE 分配器桥接给 SDK(原理见 §2.4):
    static void* ffxAlloc(void*, uint64_t size) { return FMemory::Malloc(size); }
    static void  ffxDealloc(void*, void* pMem)  { return FMemory::Free(pMem); }
    ffxAllocationCallbacks Cbs;   // 构造时绑定上面两个静态函数,userData=nullptr
};
```

### 4.2 核心接口 INGSharedBackend(必须逐一实现)

```cpp
class INGSharedBackend {
public:
    // —— FFX 五件套(直接转发到 SDK 函数表) ——
    // context 是整个生命周期句柄;desc 一律以 header.type 区分类型、header.pNext 挂扩展
    virtual ffxReturnCode_t ffxCreateContext(ffxContext* context, ffxCreateContextDescHeader* desc) = 0;
    virtual ffxReturnCode_t ffxDestroyContext(ffxContext* context) = 0;
    virtual ffxReturnCode_t ffxConfigure(ffxContext* context, const ffxConfigureDescHeader* desc) = 0;
    virtual ffxReturnCode_t ffxQuery(ffxContext* context, ffxQueryDescHeader* desc) = 0;
    virtual ffxReturnCode_t ffxDispatch(ffxContext* context, const ffxDispatchDescHeader* desc) = 0;

    virtual void Init() = 0;
    virtual EFFXBackendAPI GetAPI() const = 0;
    virtual FfxSwapchain GetSwapchain(void* swapChain) = 0;
    // UE 纹理 → FFX 资源(FRHITexture / FRDGTexture 两个重载;内部取 GetRHI() 再统一处理)
    virtual FfxApiResource GetNativeResource(FRHITexture* Texture, FfxApiResourceState State) = 0;
    virtual FfxApiResource GetNativeResource(FRDGTexture* Texture, FfxApiResourceState State) = 0;
    // UE 命令列表 → 原生 command buffer(取 Vulkan 活动命令缓冲,SDK 直接在其中录制)
    virtual FfxCommandList GetNativeCommandBuffer(FRHICommandListImmediate& RHICmdList, FRHITexture* Texture) = 0;
    virtual bool EnsureSupportedRHI() = 0;       // 当前 RHI 是否 Vulkan
    virtual bool IsNeuralGraphicSupported() = 0; // 设备是否支持 VK_ARM_tensors + VK_ARM_data_graph
    virtual bool IsLoaded() = 0;                 // SDK DLL 是否加载成功
    virtual void ForceUAVTransition(FRHICommandListImmediate&, FRHITexture*, ERHIAccess) = 0;

    // —— NFRU 帧生成专用(当前实现为 check(false) 占位,见 NGVulkanBackend 文档) ——
    virtual FfxApiResource GetInterpolationOutput(FfxSwapchain) = 0;
    virtual void* GetInterpolationCommandList(FfxSwapchain) = 0;
    virtual void RegisterFrameResources(FRHIResource*, uint64 FrameID) = 0;
    virtual bool GetAverageFrameTimes(float& AvgTimeMs, float& AvgFPS) = 0;
    virtual void CopySubRect(FfxCommandList, FfxApiResource, FfxApiResource, FIntPoint, FIntPoint) = 0;
    virtual void Flush(FRHITexture* Tex, FRHICommandListImmediate& RHICmdList) = 0; // 结束外部计算工作

    // 全局访问器:返回当前可用后端(单例),并输出 API 类型
    static NGSHARED_API INGSharedBackend* GetApiAccessor(EFFXBackendAPI& Api);
};
```

### 4.3 全局函数

```cpp
extern NGSHARED_API FfxApiSurfaceFormat GetFFXApiFormat(EPixelFormat UEFormat, bool bSRGB);
extern NGSHARED_API ERHIAccess GetUEAccessState(FfxApiResourceState State);
```

- `GetFFXApiFormat`:约 30 个分支的查表函数,把 UE `EPixelFormat` 映射为 FFX 表面格式。
  注意几个特例:`PF_DepthStencil → FFX_API_SURFACE_FORMAT_R32_FLOAT`(SDK 只消费深度值,不关心模板);
  `PF_R8G8B8A8/PF_B8G8R8A8` 按 bSRGB 区分 `_SRGB/_UNORM`;
  `PF_FloatRGB/PF_FloatR11G11B10 → R11G11B10_FLOAT`。未知格式 `check(false)`(防静默错映射)。

  ```cpp
  FfxApiSurfaceFormat Format = FFX_API_SURFACE_FORMAT_UNKNOWN;
  switch (UEFormat) {
      case PF_R8G8B8A8:
          if (bSRGB) { Format = FFX_API_SURFACE_FORMAT_R8G8B8A8_SRGB; break; }  // SRGB 是独立枚举
          Format = FFX_API_SURFACE_FORMAT_R8G8B8A8_UNORM;
          break;
      // …其余分支同构
      default: check(false);   // 未覆盖格式 = 编程错误,直接断言
  }
  ```

- `GetUEAccessState`:FFX 资源状态 → UE 访问标志,如
  `FFX_API_RESOURCE_STATE_PIXEL_READ → ERHIAccess::SRVGraphics`(仅图形管线可读)、
  `UNORDERED_ACCESS → UAVMask`、`COPY_DEST → CopyDest`、`PRESENT → Present`、
  `GENERIC_READ → ReadOnlyExclusiveComputeMask` 等。

  ```cpp
  switch (State) {
      case FFX_API_RESOURCE_STATE_PIXEL_READ:  return ERHIAccess::SRVGraphics; // PS 采样
      case FFX_API_RESOURCE_STATE_COMPUTE_READ: return ERHIAccess::SRVCompute;  // CS 采样
      case FFX_API_RESOURCE_STATE_UNORDERED_ACCESS: return ERHIAccess::UAVMask; // 可写
      case FFX_API_RESOURCE_STATE_PRESENT: return ERHIAccess::Present;          // 呈现队列
      // …
      default: return ERHIAccess::Unknown;
  }
  ```

### 4.4 GetApiAccessor 实现要点

```cpp
INGSharedBackend* INGSharedBackend::GetApiAccessor(EFFXBackendAPI& Api)
{
    Api = EFFXBackendAPI::Unknown;
    INGSharedBackendModule* VkBackend =
        FModuleManager::GetModulePtr<INGSharedBackendModule>(TEXT("NGVulkanBackend"));
    // 三个条件缺一不可:特性级别 ≥ ES3_1、后端模块存在、DLL 已加载且 RHI 受支持
    if (IsFeatureLevelSupported(GMaxRHIShaderPlatform, ERHIFeatureLevel::ES3_1) && VkBackend) {
        ApiAccessor = VkBackend->GetBackend();
        if (ApiAccessor && ApiAccessor->IsLoaded() && ApiAccessor->EnsureSupportedRHI()) {
            Api = EFFXBackendAPI::Vulkan;
            return ApiAccessor;
        }
    }
    Api = EFFXBackendAPI::Unsupported;   // 探测失败:调用方据此禁用功能
    return ApiAccessor;
}
```

即:通过模块名 `"NGVulkanBackend"` 拿模块 → 取其 `INGSharedBackend` 单例 → 校验已加载 + RHI 支持。
NSS/NFRU 的 `Initialize()/OnPostEngineInit()` 都以此结果决定启用/禁用。

```cpp
class INGSharedBackendModule : public IModuleInterface {
public:
    virtual INGSharedBackend* GetBackend() = 0;   // NGVulkanBackendModule 实现
};
```

## 5. FFXRDGBuilder / FFXSlateApplication —— 引擎私有成员访问器

设计动机与原理见 §2.3。机制三件套:

### 5.1 复制引擎头文件(版本化)

引擎头文件整体拷贝到 `NGShared/Public/{FRDGBuilderVersions,FSlateApplicationVersions}/`
(按引擎主次版本命名,如 `FRDGBuilder_5_4_0.h`、`FSlateApplication_5_4_0.h`)。

### 5.2 宏改写后包含

```cpp
// FFXRDGBuilderBase.h —— 生成 "受保护版 FRDGBuilder"
#define private protected                // 私有→受保护
#define FRDGBuilder FFXRDGBuilderBase    // 改名,避免与真类冲突
#include FFX_BUILD_VERSIONED_INCLUDE_PATH(RDGBuilder)   // 按引擎版本选复制头
#undef FRDGBuilder
#undef private

#if UE_VERSION_AT_LEAST(5, 7, 0)
#error "Unsupported Unreal Engine 5 version - update the definition for FFXRDGBuilderBase"
#endif
```

`FFXSlateApplicationBase.h` 同构(作用于 `FSlateApplication`)。

### 5.3 派生类补方法 + 布局断言

```cpp
class FFXRDGBuilder : public FFXRDGBuilderBase {
public:
    // 遍历图内纹理链表,按名字找到目标纹理(历史用途:取 Lumen 反射纹理)
    FRDGTextureRef FindTexture(TCHAR const* Name)
    {
        for (FRDGTextureHandle It = Textures.Begin(); It != Textures.End(); ++It) {
            FRDGTextureRef Texture = Textures.Get(It);
            if (FCString::Strcmp(Texture->Name, Name) == 0)
                return Texture;
        }
        return nullptr;
    }
};
static_assert(sizeof(FRDGBuilder) == sizeof(FFXRDGBuilder),
    "FFXRDGBuilder must match the layout of FRDGBuilder so we can access the Lumen reflection texture!");
```

```cpp
class FFXSlateApplication : public FFXSlateApplicationBase {
public:
    inline TSharedPtr<FSlateRenderer>* GetRendererRef() { return &Renderer; }   // 渲染器指针(可替换)
    inline double& LastTickTimeRef() { return LastTickTime; }                   // 时间戳(可改写)
};
static_assert(sizeof(FSlateApplication) == sizeof(FFXSlateApplication),
    "FFXSlateApplication must match the layout of FSlateApplication so we can access the time delta!");
```

> 为什么需要 `FindTexture`:早期 FSR3 插件在 `AddPasses` 前从图里按名找出 Lumen 反射纹理注入
> denoiser 流程(历史用途);当前 NSS 主流程已不依赖,但 `FFXRDGBuilder` 仍保留。
> `FFXSlateApplication` 被 NFRU 使用:替换渲染器指针、修改 `LastTickTime` 实现
> "重绘时把 Slate delta time 置 0"(见 NFRU 文档 §7.2)。

## 6. 模块实现(Module 骨架)

`NGSharedModule.cpp` 只是空实现:

```cpp
IMPLEMENT_MODULE(NGShared, NGShared)
void NGShared::StartupModule() {}    // 本模块无运行期状态,纯"头文件 + 工具函数"库
void NGShared::ShutdownModule() {}
```

真正的工作在 `NGSharedBackend.cpp`(格式映射 + GetApiAccessor)与各复制头中。

## 7. 复现要点(若需重新实现)

1. 定义 `UE_VERSION_AT_LEAST` 与版本化包含宏(一处定义,全局复用)。
2. 保证 `INGSharedBackend` 的语义:创建 context 时后端必须通过 `desc->pNext` 附加
   `ffxCreateBackendVKDesc`(见 NGVulkanBackend 文档),否则 SDK 不知道如何绑定设备。
3. `GetFFXApiFormat` 的映射表要与 NG-SDK 的 `ffx_api` 枚举严格一致;
   深度格式映射为 R32_FLOAT 是 NSS/NFRU 的约定。
4. 新增引擎分支时:复制新引擎的 `RenderGraphBuilder.h` / `SlateApplication.h` 到版本目录,
   并去掉 `#error` 上限;必须同步确认 `sizeof` 断言仍成立。
5. `GetApiAccessor` 的"三条件"(特性级别 + 模块存在 + 加载成功且 RHI 支持)是全局开关的
   唯一判据,不要在各业务模块里重复实现不同的探测逻辑。
