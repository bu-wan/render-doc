# 模块文档 05:NFRU(Neural Frame Rate Upscaling 帧生成)

> 源码位置:`Source/NFRU/`(仅 5.x 引擎分支存在,4.27 分支无此模块)。
> 依赖:Public `Engine`;Private `Core/Engine/Projects/RenderCore/Renderer/RHI/ApplicationCore/CoreUObject/EngineSettings/DeveloperSettings/SlateCore/Slate/NGShared/NGVulkanBackend/NGSettings/VulkanRHI`。
> IncludePath 含引擎 `Renderer/Private` 与 `Renderer/Internal`。

## 1. 模块组成与角色

| 文件 | 角色 |
|---|---|
| `NFRUModule.h/.cpp` | 模块入口;`NFRUModule : INFRUModule`,`GetImpl()` 返回 `NFRU*` |
| `NFRU.h/.cpp` | **核心**:`NFRU : INFRU`;ViewExtension 数据采集、`InterpolateFrame`、Slate 回调、FPS 统计 |
| `INFRU.h` | 对外接口:`CreateCustomPresent(...)`、`GetAverageFrameTimes(...)`;`INFRUCustomPresent : FRHICustomPresent` 抽象 |
| `NFRUViewExtension.h/.cpp` | `FSceneViewExtensionBase` 派生:桌面 `PrePostProcessPass_RenderThread` + 移动 `PrePostProcessPassMobile_RenderThread` |
| `NFRUCustomPresent.h/.cpp` | **呈现控制核心**:`NFRUCustomPresent : INFRUCustomPresent` + `NFRUResources`(context 容器) |
| `NFRUSlate.h/.cpp` | `NFRUSlateRenderer : FSlateRenderer`,包装原渲染器,广播窗口/后缓冲事件 |
| `NFRUStats.h` | Stat 组 `FrameGen`(RenderFPS/RenderInterval/PresentFPS/PresentInterval) |
| `LogNFRU.h` | 日志分类 `LogNFRU` + `ShouldUseCustomFramePacing()`(Android Swappy 判断) |

## 2. 原理与算法分析

### 2.1 帧生成算法:光流 → 变形 → 合成

帧生成的目标:在真实帧 N 与真实帧 N+1 之间合成中间帧 N+0.5,让显示端每半个刷新周期
都看到新画面。SDK 内部的经典管线(插件负责喂数据):

```
输入:当前帧颜色/深度 + 上一帧颜色/深度 + 运动矢量 + 相机矩阵 + 帧时间差
   │
   ├─ 1) 运动估计:以运动矢量 + 深度为线索(optical flow hints),估计稠密运动场
   ├─ 2) 变形(warp):把上一帧颜色沿运动方向搬运到中间时刻 → 中间帧主体
   ├─ 3) 合成/修补:用当前帧信息修补遮挡与新暴露区域;时间一致性处理
   └─ 输出:中间帧(插值帧)
```

插件侧的关键决策:

- **为什么帧时间差和相机矩阵是必需的**:插值系数 α(中间帧在 N~N+1 间的时刻)由帧时间差决定;
  相机矩阵保证几何一致(如旋转镜头时背景运动是投影变换而非平移)。
- **jitter 传递**:上采样链路(NSS)的 jitter 会让"无 jitter 的颜色"与"带 jitter 的渲染"错位,
  因此 NFRU 要把 jitter 一并传给 SDK,由其内部校正。
- **MANAGE_PREVIOUS_DEPTH flag**:显示高度 > 1080 时 SDK 自行管理上一帧深度
  (自己保存/对齐),插件不再提供 `depthTm1`;否则插件负责把当前深度拷贝为"上一帧深度"。
  这是对高分辨率下深度资源带宽的优化——避免每帧多一次全屏拷贝。

### 2.2 双缓冲 ping-pong:为什么需要"上一帧颜色"

插值必须基于**连续两帧真实渲染**。插件用 `BackBufferRT[2]` 做 ping-pong:

```
帧 N:  BackBuffer(引擎刚渲染完的真实帧) → 拷贝到 BackBufferRT[Index]
       插值输入:当前真实帧 = BackBufferRT[Index]
                 上一真实帧 = BackBufferRT[(Index+1)%2]   ← 帧 N-1 时拷贝的那份
       插值输出 → InterpolatedRT(或直接写 BackBuffer)
帧 N+1:Index 翻转,角色互换
```

- 为什么不能直接用 BackBuffer 当"当前真实帧":BackBuffer 在 Present 后会被引擎复用/覆盖,
  必须先把真实帧"保存"到私有 RT;
- 为什么两帧而不是一帧:插值需要两个锚点(帧 N 与 N-1),单缓冲只够存一个;
- 为什么不是三帧:帧生成只插 1 帧(numGeneratedFrames=1),两帧锚点足够。

### 2.3 呈现节奏:状态机 + Slate 重绘

帧生成与上采样的本质区别在于**呈现节奏**:

```
真实帧(引擎 Present)     插值帧(Slate 强制重绘后 Present)
     │                        │
     └── 两个刷新周期交替 ──────┘
```

实现依赖两套机制:

1. **CustomPresent 状态机**(`NFRUCustomPresentStatus`):渲染线程与 RHI 线程之间用
   `InterpolateRT → InterpolateRHI → PresentRT → PresentRHI` 四个状态同步
   "插值帧已渲染/已提交/真实帧开始呈现/已提交",`NeedsNativePresent()`/`Present()` 据此
   决定每帧是否走原生 Present(当前恒走原生 Swapchain,`bUseFFXSwapchain=false`)。
2. **Slate 强制重绘**:`NFRU::OnSlateWindowRendered` 在游戏线程检测到 presenter 启用时,
   把"上一真实帧"拷回 BackBuffer 显示,然后 `App.ForceRedrawWindow` 触发第二次 DrawWindows,
   让插值帧占据下一个刷新周期。**必须把 Slate delta time 置 0**(`r.NFRU.ModifySlateDeltaTime`):
   否则 UI 小部件会以为时间前进了两帧,动画/计时器加速。

### 2.4 UI 调试捕获算法(FFXFIAdditionalUICS)

问题:部分调试 UI(stat 面板等)只在**首帧**绘制一次,帧生成会交替显示"有 UI 的真实帧"
与"无 UI 的插值帧",UI 闪烁/丢失。方案是四纹理差异合成(着色器 `PostProcessFFX_FIAdditionalUI.usf`):

```
输入: FirstFrame(首帧无UI) / FirstFrameWithUI(首帧有UI)
      SecondFrame(第二帧无UI) / SecondFrameWithUI(第二帧有UI,UAV 输出)
算法(逐像素):
  若 首帧有 UI 差异(any(FirstFrame - FirstFrameWithUI) ≠ 0)
  且 (第二帧无 UI 差异 或 两帧 UI 差异过大):
      把 UI 差异叠加到第二帧,并做 min/max 夹取
      Result = clamp(SecondFrameWithUI + (FirstFrameWithUI - FirstFrame), min, max)
```

- 条件"差异过大"防止把真实场景变化误当 UI;
- min/max 夹取把结果限制在两帧实际像素值范围内,防止过冲/噪点;
- 该着色器只在 `r.NFRU.CaptureDebugUI` 打开时执行(默认非 SHIPPING 开启)。

### 2.5 FPS 调节器:节拍与升降频

显示器刷新率固定时,帧生成需要把"真实帧 + 插值帧"精确排布进刷新周期(节拍 pacing)。
`CustomFramePacerAdjust` 是**自适应节拍器**:

```
每帧:
  TimeToSleep = 目标节拍时刻 - 当前时刻        // 距离下一个理想提交点还差多少
  升频:TimeToSleep > 下一档间隔,且连续 UpAdjustFrameCount(40)帧 → 提高目标 FPS
  降频:TimeToSleep < 0(已过点,掉帧),且连续 DownAdjustFrameCount(20)帧 → 降低目标 FPS
```

- FPS 档位由 `InitFpsLevels(MaxFPS)` 生成阶梯 {Max, Max/2, Max/3, …}(>10 停止),升/降频
  在阶梯间移动——保证目标是显示器刷新率的整数约数,避免节拍错位;
- 升频阈值(40)大于降频阈值(20):"掉帧"体验伤害大,降频反应更快;升频需更久观察确认稳定;
- 生效方式:发 `r.SetFramePace <fps>` 控制台命令;关闭 NFRU 时恢复 `DefaultFPS`
  (见 `ResetFPS`),避免帧率卡在插值档位;
- 统计:渲染帧率用 0.75/0.25 EMA,Present(插值)帧率用 0.9/0.1 EMA——不同时间常数匹配各自
  的波动特性(Present 节奏更稳定,历史权重更高)。

### 2.6 为什么需要引导期(ResetState)

插值需要上一帧深度/颜色作锚点。刚启用或相机切换后的前几帧锚点无效,直接插值会出鬼影。
`ResetState` 从 3 递减到 0:引导期内只采集/呈现真实帧,不插值;`bReset = bCameraCut || ResetState==0`
传给 SDK,让 SDK 也重置内部历史。

## 3. 对外接口

```cpp
// INFRU.h
enum ENFRUPresentMode { ENFRUPresentModeRHI, ENFRUPresentModeNative };

class INFRUCustomPresent : public FRHICustomPresent {
public:
    virtual void InitViewport(FViewport* InViewport, FViewportRHIRef ViewportRHI) = 0;
    virtual void SetMode(ENFRUPresentMode Mode) = 0;
    virtual void SetUseFFXSwapchain(bool bEnabled) = 0;
};

class INFRU {
public:
    virtual INFRUCustomPresent* CreateCustomPresent(INGSharedBackend* Backend, uint32_t Flags,
        FIntPoint RenderSize, FIntPoint DisplaySize, FfxSwapchain RawSwapChain, FfxCommandQueue Queue,
        FfxApiSurfaceFormat Format, EFFXBackendAPI Api) = 0;
    virtual bool GetAverageFrameTimes(float& AvgTimeMs, float& AvgFPS) = 0;
};
```

## 4. NFRU 核心类结构

```cpp
class NFRU : public INFRU {
public:
    void OnPostEngineInit();
    void OnViewportCreatedHandler_SetCustomPresent();
    void OnBeginDrawHandler();
    template <typename TInputs> void SetupView(const FSceneView&, const TInputs&);  // 桌面/移动模板
    void InterpolateFrame(FRDGBuilder& GraphBuilder);       // 挂 PostRenderDelegateEx
    void OnSlateWindowRendered(SWindow&, void* ViewportRHIPtr);
    void OnBackBufferReadyToPresentCallback(SWindow&, const FTexture2DRHIRef&);
    INFRUCustomPresent* CreateCustomPresent(...) final;
    bool GetAverageFrameTimes(float&, float&) final;
    void ResetFPS(IConsoleVariable*);       // 恢复原始帧率(§2.5)
    void RegisterCVarCallbacks();
private:
    struct NFRUView {                        // 每视图采集数据
        FRDGTextureRef ViewFamilyTexture, SceneDepth, SceneVelocity;
        FIntRect ViewRect;
        FIntPoint InputExtentsQuantized, OutputExtents;
        FVector2D TemporalJitterPixels;
        float CameraNear, CameraFOV, GameTimeMs;
        bool bReset, bEnabled;
    };
    bool InterpolateView(FRDGBuilder&, NFRUCustomPresent*, const FSceneView*,
        NFRUView const&, FRDGTextureRef FinalBuffer, FRDGTextureRef ColorBufferTm1,
        FRDGTextureRef InterpolatedRDG, FRDGTextureRef BackBufferRDG, uint32 Index);
    void CalculateFPSTimings();
    TMap<const FSceneView*, NFRUView> Views;
    TSharedPtr<NFRUViewExtension> ViewExtension;
    TRefCountPtr<IPooledRenderTarget> BackBufferRT[2];   // ping-pong 历史颜色(§2.2)
    TRefCountPtr<IPooledRenderTarget> InterpolatedRT;    // 插值结果
    TRefCountPtr<IPooledRenderTarget> AsyncBufferRT[2];
    TMap<FfxSwapchain, NFRUCustomPresent*> SwapChains;
    TMap<SWindow*, FRHIViewport*> Windows;
    float GameDeltaTime, AverageTime, AverageFPS;
    uint64 InterpolationCount, PresentCount;
    uint32 Index, ResetState;               // ResetState:引导期倒计数(§2.6)
    bool bInterpolatedFrame, bLastUseFragmentShader, bResetFPS;
    int32 DefaultFPS, LastFPS;   // FPlatformRHIFramePacer::GetFramePace()
};
```

## 5. 生命周期与钩子注册

```cpp
NFRU::NFRU() {
    UGameViewportClient::OnViewportCreated().AddRaw(this, &NFRU::OnViewportCreatedHandler_SetCustomPresent);
    FCoreDelegates::OnPostEngineInit.AddRaw(this, &NFRU::OnPostEngineInit);
    RegisterCVarCallbacks();
}

void NFRU::OnPostEngineInit() {
    if (!INGSharedBackend::GetApiAccessor(Api)) return;      // 无 Vulkan/SDK 则禁用
    if (FSlateApplication::IsInitialized()) {
        // 1) 用 NFRUSlateRenderer 包装并替换 Slate 渲染器(所有后端都需要,否则等 DrawBuffer)
        TSharedRef<NFRUSlateRenderer> RendererWrapper = MakeShared<NFRUSlateRenderer>(App.GetRenderer 的 ref);
        App.InitializeRenderer(RendererWrapper, true);
        // 2) 挂窗口渲染 / 后缓冲就绪回调
        SlateRenderer->OnSlateWindowRendered().AddRaw(this, &NFRU::OnSlateWindowRendered);
        SlateRenderer->OnBackBufferReadyToPresent().AddRaw(this, &NFRU::OnBackBufferReadyToPresentCallback);
        // 3) 每帧后处理完成后执行插值(§2.1)
        GEngine->GetPostRenderDelegateEx().AddRaw(this, &NFRU::InterpolateFrame);
        // 4) ViewExtension 采集输入
        ViewExtension = FSceneViewExtensions::NewExtension<NFRUViewExtension>(this);
    }
}
```

### Viewport 接管

```cpp
void NFRU::OnViewportCreatedHandler_SetCustomPresent() {
    if (视口 RHI 有效且尚无 CustomPresent)
        GEngine->GameViewport->OnBeginDraw().AddRaw(this, &NFRU::OnBeginDrawHandler);
}
void NFRU::OnBeginDrawHandler() {
    // a) 自定义帧率调节(§2.5)
    // b) 若视口尚无 CustomPresent:
    //    从 SwapChains(按原生 swapchain 句柄)找 presenter;找不到则创建:
    //    CreateCustomPresent(VulkanBackend, Flags,
    //        SwapChainSize, SwapChainSize, nullptr, (FfxCommandQueue)GDynamicRHI,
    //        GetFFXApiFormat(SurfaceFormat, false), EFFXBackendAPI::Vulkan)
    //    → InitViewport(...)(内部 SetCustomPresent(this))
}
```

## 6. 输入采集 SetupView(模板,桌面/移动共用)

由 `NFRUViewExtension` 在 PostProcess 前回调:

- 桌面:`PrePostProcessPass_RenderThread`(SM5 起);移动:`PrePostProcessPassMobile_RenderThread`(ES3_1)。
- 仅 `InView.bIsViewInfo` 的视图被采集。
- 采集字段:ViewFamilyTexture、SceneDepth(`SceneTextures->SceneDepthTexture`)、
  SceneVelocity(桌面 `GBufferVelocityTexture`,移动 `SceneVelocityTexture`)、
  ViewRect、`QuantizeSceneBufferSize(GetSecondaryViewRectSize, OutputExtents)`(并取与输入的最大值)、
  bReset=`bCameraCut`、CameraNear、垂直 FOV、`TemporalJitterPixels`、
  bEnabled = `bIsGameView && !bIsSceneCapture && !bIsSceneCaptureCube && !bIsReflectionCapture && !bIsPlanarReflection`(5.1+ 含 Cube)。
- 加入 `Views` 映射。

> 为什么桌面/移动分开:移动后处理(`FMobilePostProcessingInputs`)的速度在
> `SceneVelocityTexture` 而非 `GBufferVelocityTexture`,且无 `GetSecondaryViewRectSize`
> (输出=输入)。模板参数 + `if constexpr` 消除重复代码。

## 7. InterpolateFrame / InterpolateView(渲染线程核心)

### 7.1 InterpolateFrame(每帧入口)

```cpp
bAllowed = CVarEnableNFRU && Presenter && (Views.Num()>0) && bNeuralGraphicSupported;
#if WITH_EDITORONLY_DATA
bAllowed &= !GIsEditor;      // 编辑器禁用
#endif
if (bAllowed) {
    BackBufferRDG = RegisterExternalTexture(GraphBuilder, RHIGetViewportBackBuffer(ViewportRHI));
    // ping-pong 纹理(BackBufferRT[2] + InterpolatedRT)按尺寸/格式/是否 fragment 重建
    Presenter->BeginFrame();
    Presenter->SetPreUITextures(BackBufferRT[Index], InterpolatedRT);  // UI 捕获用(无 UI 版)
    Presenter->SetEnabled(true);
    FinalBuffer     = RegisterExternalTexture(BackBufferRT[Index]);        // 当前帧颜色(§2.2)
    ColorBufferTm1  = RegisterExternalTexture(BackBufferRT[(Index+1)%2]);  // 上一帧颜色(历史)
    InterpolatedRDG = RegisterExternalTexture(InterpolatedRT);
    AddCopyTexturePass(GraphBuilder, BackBufferRDG, FinalBuffer, Info);    // 保存真实帧
    for (Views 中每视图,若 bEnabled 且 ViewFamilyTexture 尺寸==视口尺寸)
        bInterpolated |= InterpolateView(..., InterpolateIndex++);
    Presenter->EndFrame();
    Index = (Index + 1) % 2;   // ping-pong 切换
}
Views.Empty();
ResetState = bInterpolatedFrame ? 3u : 0u;   // 引导期倒计数(§2.6)
```

### 7.2 InterpolateView(单视图插值)

1. **准备参数**:
   ```cpp
   ffxApiDispatchDescFrameGenerationPrepare UpscalerDesc;
   // frameID、frameTimeDelta、cameraNear/Far(深度反转约定,见 NGVulkanBackend §2.3)、
   // cameraFovAngleVertical、viewSpaceToMetersFactor = 1/WorldToMetersScale、
   // jitterOffset=TemporalJitterPixels、motionVectorScale=InputExtents、
   // viewProjection[16](View*ProjectionNoAA,列主序平铺)
   ```
2. **上下文描述** `ffxApiCreateContextDescFrameGeneration FgDesc`:
   - 后缓冲格式检查(Windows):`PF_R8G8B8A8/PF_B8G8R8A8/PF_FloatR11G11B10/PF_R8`,否则
     `ReportUnsupportedNfruBackBufferFormat`(一次弹窗)+ `SetEnabled(false)` 返回。
   - `backBufferFormat = GetFFXApiFormat(...)`;displaySize/renderSize。
   - flags:`DEPTH_INVERTED` + `HIGH_DYNAMIC_RANGE | DEPTH_INFINITE`;
     高度 > 1080 追加 `MANAGE_PREVIOUS_DEPTH`(§2.1);
     fragment 路径追加 `ALL_STAGES_FRAGMENT | MV_HINTS_FRAGMENT`。
   - `initialViewProjection[16]` = 当前 VP(第一帧历史矩阵种子)。
   - `fpMessage = NFRUMsgCallback`(FFX 消息 → UE_LOG)。
3. **上下文获取**:`Presenter->UpdateContexts(GraphBuilder, ViewState->UniqueID, FgDesc)`
   → 从 `OldResources` 里按 UniqueID 复用,校验 displaySize/renderSize/backBufferFormat/flags 未变,
   否则新建 `NFRUResources` 并 `ffxCreateContext`;返回 `FFXFIResourceRef`。
4. **速度转换**:`Context->MotionVectorRT`(PF_G16R16F,按输入尺寸)
   → `helper_NGConvertVelocity(...)`(同 NSS,见模块 03 §6)。
5. **上一帧深度**:`DepthTM1RT`(与 SceneDepth 同尺寸同格式,`FClearValueBinding::DepthFar`);
   若 SDK 不管理上一帧深度(`!(Desc.flags & MANAGE_PREVIOUS_DEPTH)`),插值 pass 后
   `AddCopyTexturePass(SceneDepth → DepthTm1Texture)`。
6. **颜色中转**:displaySize 与视口尺寸不一致时,建 `Context->Color`(FIColor)与 `Context->Inter`(FIInter)
   池化纹理;把 FinalBuffer 按 OutputPoint 拷贝到 ColorBuffer(fragment 时 Inter 直接用作输出并 Clear)。
7. **插值 Pass**(`DoAddPass` 模板,NFRUPass::FComputeParameters / FFragmentParameters):
   - 参数:ColorTexture(=ColorBuffer)、BackBufferTexture、ColorBufferTm1、SceneDepth/SceneDepthTm1、
     MotionVectors、HudTexture、InterpolatedRT、Interpolated。
   - 计算 VSync 区间:`interpolateParams->numGeneratedFrames = 1`,reset = bReset(`bCameraCut || ResetState==0`,§2.6)。
   - `ffxApiConfigureDescFrameGeneration ConfigDesc`:swapChain(来自 `Backend->GetSwapchain`)、
     frameGenerationEnabled=true、frameID、`NO_SWAPCHAIN_CONTEXT_NOTIFY`、调试视图 flag。
   - PassFlags:fragment = `Raster|NeverCull|SkipRenderPass|Copy`;compute = `Compute|NeverCull|Copy`。
   - lambda 内(RHI 线程):
     - `bDirectToBackBuffer`:整屏 + fragment + 不捕获调试 UI 时,插值结果直接写回 BackBuffer,省一次拷贝。
     - PrepareDesc:depth/depthTm1/colorTm1 读状态,`motionVectors` 写状态(fragment 为 RTV);
       `InterpolatedRes` = 直接写 BackBuffer ? RTV(BackBuffer) : RTV/UAV(InterpolatedRT)。
     - `SetCustomPresentStatus(InterpolateRT)` → EnqueueLambda:
       `ffxConfigure`(swapChain=null 时跳过回调)→ `SetCustomPresentStatus(InterpolateRHI)` →
       取 command buffer → `ffxDispatch(Prepare)` → `ffxDispatch(FrameGeneration, outputs[0]=InterpolatedRes)`。
     - `Backend->Flush(...)`(结束外部计算,见 NGVulkanBackend §2.4)。
     - 非直写时:把 InterpolatedRT 拷贝到 InterpolatedRDG;`ENFRUPresentModeRHI` 下再拷到 BackBuffer。
   - 结束后(非 MANAGE_PREVIOUS_DEPTH)拷 SceneDepth → DepthTm1。
8. 返回 `bInterpolated`。

## 8. 呈现节奏与 FPS 调节

### 8.1 CustomPresent 状态机(§2.3)

```cpp
enum class NFRUCustomPresentStatus : uint8 {
    InterpolateRT = 0,   // 插值已渲染到 RT(渲染线程进入)
    InterpolateRHI = 1,  // 插值命令已提交到 RHI 线程
    PresentRT = 2,       // 准备呈现真实帧
    PresentRHI = 3       // 原生 Present 已提交
};
```

- `SetCustomPresentStatus` 同时维护 `bHasInterpolatedRT/bHasInterpolatedRHI/bPresentRHI/bNeedsNativePresentRT`。
- `NeedsNativePresent()`:Status==PresentRT 时返回 `bUseFFXSwapchain ? bNeedsNativePresentRT : true`
  (当前恒走 UE 原生 swapchain,恒 true)。
- `NeedsAdvanceBackbuffer()` = false(索引推进由本插件控制)。
- `Present(int32& SyncInterval)`:计算 FPS 统计 + `CustomFramePacerAdjust()`(§2.5),返回
  `!bUseFFXSwapchain || bDrawDebugView`(恒 true → 原生 Present)。
- `OnBackBufferResize()`:置 `bResized=true` + `FlushRenderingCommands()`(等待 GPU 空闲再重建)。

### 8.2 Slate 侧双帧呈现(§2.3)

- `NFRUSlateRenderer`(包装原 `FSlateRenderer`):AcquireDrawBuffer/ReleaseDrawBuffer 委托底层,
  用 6 个自维护 DrawBuffer;把 `OnSlateWindowRendered/Destroyed/BackBufferReadyToPresent/Resize` 等
  通过 Thunk 广播出去,让 NFRU 可以挂钩。
- `NFRU::OnSlateWindowRendered`(游戏线程):当 Presenter 启用且 `ENFRUPresentModeRHI`:
  - 记录 `Windows.Add(&SlateWindow, Viewport)`。
  - 渲染线程命令:把 `BackBufferRT[1-Index]`(刚渲染完的真实帧)拷贝回 BackBuffer 显示,
    状态机置 PresentRT→PresentRHI。
  - 若 `CVarNFRUModifySlateDeltaTime`:临时把 `FSlateApplication::LastTickTime` 置为当前时间
    (使 widgets 的 NativeTick 不被重复触发),`App.ForceRedrawWindow(Window)` 补画一帧(显示插值帧),再恢复。
- `OnBackBufferReadyToPresentCallback`(渲染线程):`ResetState` 递减;
  在 `ResetState>0` 的引导期内 `Presenter->CopyBackBufferRT(BackBuffer)` 采集帧。

### 8.3 CopyBackBufferRT(UI 调试捕获,§2.4)

按当前 Status 处理(仅 `CVarNFRUCaptureDebugUI` 且 RHI 模式):

- `InterpolateRT`:把当前 BackBuffer(插值帧)拷贝到 `Current.Interpolated`。
- `PresentRT`:把真实帧拷到 `Current.RealFrame`,然后跑 **FFXFIAdditionalUICS** 计算着色器
  (`/Plugin/NG/Private/PostProcessFFX_FIAdditionalUI.usf`,`MainCS`):
  输入 `FirstFrame/FirstFrameWithUI/SecondFrame/SecondFrameWithUI`(四纹理),
  逻辑(§2.4):若首帧有 UI 差异且第二帧无 UI 或差异大,则把 UI 差异叠加进第二帧并做 min/max 夹取。
  最终把结果拷回 BackBuffer。

### 8.4 FPS 调节器(§2.5)

- FPS 阶梯:`InitFpsLevels(MaxFPS)` 生成 {MaxFPS, MaxFPS/2, MaxFPS/3, …}(>10 为止),排序缓存。
- 每帧计算 `TimeToSleep`(目标节拍 - 实际耗时):
  - `TimeToSleep > 下一档间隔` 连续 `CVarNFRUUpAdjustFrameCount`(40)帧 → 升频;
  - `TimeToSleep < 0` 连续 `CVarNFRUDownAdjustFrameCount`(20)帧 → 降频。
- 通过 `r.SetFramePace` 生效;`NFRU::ResetFPS` 在 `CVarEnableNFRU`/`CVarNFRUPaceAdjuster` 关闭时
  恢复 `DefaultFPS`。
- `ShouldUseCustomFramePacing()`(LogNFRU.h):Android + Swappy 时取反(用 Swappy 则本插件不接管)。

## 9. NFRUResources(上下文容器)

```cpp
struct NFRUResources : public FRHIResource {
    uint32 UniqueID;                                  // ViewState->UniqueID(复用匹配键)
    ffxApiCreateContextDescFrameGeneration Desc;      // 用于复用比较(尺寸/格式/flags)
    ffxContext Context;
    TRefCountPtr<IPooledRenderTarget> Color, Hud, Inter, MotionVectorRT, DepthTM1RT;
    INGSharedBackend* Backend;
    bool bDebugView;
    ~NFRUResources() { if (Backend && Context) Backend->ffxDestroyContext(&Context); }
};
typedef TRefCountPtr<NFRUResources> FFXFIResourceRef;
```

- Presenter 用 `Resources`(本帧)/`OldResources`(上帧可复用)/`DeferredDestroyQueue`(延迟销毁,
  `CVarNGDeferDelete` 帧)三级管理:`BeginFrame` 把 Resources 转 Old;`EndFrame` 把 Old 入队并裁剪队首。
- 复用匹配:同 `UniqueID`(同一视图)+ 描述完全一致 → 直接复用 context,避免重建
  (与 NSS 状态池同理,原理见 NSS 文档 §2.2)。

## 10. 统计

- `NFRUStats.h`:Stat 组 `FrameGen`(`RenderFPS/RenderInterval/PresentFPS/PresentInterval`);
  CSV 类别 `FrameGen`。
- `NFRU::CalculateFPSTimings`:EMA(0.75/0.25)更新 `AverageTime/AverageFPS`(§2.5 时间常数说明);
  `CVarNFRUUpdateGlobalFrameTime` 时写回引擎全局 `GAverageMS/GAverageFPS`;
  `GetAverageFrameTimes` 从 Presenter 取插值节奏统计(0.9/0.1 EMA)。

## 11. 复现要点

1. **三处挂钩缺一不可**:ViewExtension 采集输入、PostRenderDelegateEx 执行插值、
   Slate 渲染器包装 + 窗口重绘呈现插值帧、FRHICustomPresent 控制原生 Present。
2. `InterpolateFrame` 的 ping-pong(BackBufferRT[Index])保存"真实帧",插值必须用上一真实帧做历史,
   否则鬼影(§2.2)。
3. 插值输出路径选择:整屏 fragment 且不捕获 UI → 直写 BackBuffer(省拷贝);
   否则写 InterpolatedRT 再拷贝。
4. 首帧引导:`ResetState`(3 帧)内只采集不插值;`bReset=bCameraCut || ResetState==0` 传给 SDK(§2.6)。
5. UI 捕获只有在 `r.NFRU.CaptureDebugUI` 时启用,且依赖四纹理计算着色器,注意其"差异夹取"算法(§2.4)。
6. 自定义帧率调节要配合 `r.SetFramePace`,并在关闭 NFRU 时恢复默认帧率(否则帧率卡在插值档位,§2.5)。
7. 后缓冲格式限制(Windows)是硬性门禁;高度 >1080 时 `MANAGE_PREVIOUS_DEPTH` 可省一次深度拷贝。
