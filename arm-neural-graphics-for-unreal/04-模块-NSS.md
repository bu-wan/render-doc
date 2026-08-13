# 模块文档 04:NSS(Neural Super Sampling 上采样)

> 源码位置:`Source/NSS/`(三个引擎分支均有)。
> 依赖:Public `Engine`/`NGShared`;Private `Core/Engine/Projects/RenderCore/Renderer/RHI/Landscape/CoreUObject/NGSettings/NGVulkanBackend`。
> IncludePath 含引擎 `Renderer/Private` 与 `Renderer/Internal`(需访问引擎内部渲染 API)。

## 1. 模块组成与角色

| 文件 | 角色 |
|---|---|
| `NSSModule.h/.cpp` | 模块入口;持有 `NSS` 单例与 `NSSViewExtension`;暴露 `INSSModule` 接口 |
| `NSSViewExtension.h/.cpp` | `FSceneViewExtensionBase` 派生:决策何时启用 NSS、把 `NSSProxy` 设进 ViewFamily、维护 mip bias |
| `NSS.h/.cpp` | **核心**:`NSS : ITemporalUpscaler + IScreenSpaceDenoiser`,实现 `AddPasses` 全流程 |
| `NSSProxy.h/.cpp` | 转发壳:`SetTemporalUpscalerInterface` 需要的独立实例,转发到核心 NSS |
| `NSSHistory.h/.cpp` | 历史对象:持有 `NSSState`(内含 ffxContext),跨帧传递;析构时归还状态池 |
| `INSSHistory.h` | `INSSHistory : ITemporalUpscaler::IHistory`,暴露 `GetNSSContext/GetNSSContextDesc` |
| `NSSInclude.h` | FFX NSS 头文件封装(同 NGShared.h 手法) |
| `LogNSS.h` | 日志分类 `LogNSS` |

## 2. 原理与算法分析

### 2.1 时序上采样的数据流与"学习式融合"

NSS 本质是"用神经网络实现 TAA 上采样"(完整理论见架构总览 §2.1)。这里给出算法级视角:

```
第 N 帧:
  引擎用带 jitter 的投影渲染低分辨率 SceneColor(渲染成本 ↓)
  引擎输出:SceneColor / SceneDepth / SceneVelocity(稀疏)/ jitter / 曝光
  ─────────────────────────────────────────────────────────
  插件:
   1. 速度场稠密化(见 NGVulkanBackend §2.2)
   2. 组装 dispatch 参数:renderSize / upscaleSize / jitter / 相机 / reset / frameTimeDelta
   3. ffxDispatch:
       输入 = 当前颜色 + 当前深度 + 稠密速度 + 上一帧全分辨率输出(outputTm1)
       输出 = 全分辨率颜色(下一帧的 outputTm1)
  ─────────────────────────────────────────────────────────
  引擎:把输出作为最终 SceneColor,写入 TemporalAAHistory 供下一帧使用
```

三个算法的关键点:

- **jitter 取反**:UE 的 `TemporalJitterPixels` 表示"本帧投影相对无抖动位置的偏移",
  而 SDK 的 `jitterOffset` 需要的是"采样位置相对屏幕中心的偏移",两者符号相反——对齐错误会导致
  历史与当前帧错位半个像素,出现模糊/抖动。
- **reset 语义**:`reset=true` 表示 SDK 丢弃内部历史(相机切换、无历史、NSS 刚启用)。
  相机切换后旧历史与当前画面无对应关系,继续融合会产生拖影(ghosting)。
- **曝光值**:线性 HDR 场景亮度跨度大,网络需要曝光值做内部归一化,输出再按曝光还原。

### 2.2 ffxContext 状态池:为什么必须复用与延迟删除

`ffxContext` 是 SDK 的"会话句柄":创建时分配内部资源(着色器、**Data Graph 会话**,单份约
3.5MB 显存)、编译模型变体。频繁创建/销毁会带来:

- 显存分配抖动(峰值内存暴涨、碎片化);
- GPU 空闲气泡(每帧创建时设备要等待编译完成)。

因此代码维护 **状态池 + 复用判定 + 延迟删除** 三层策略:

```
优先级 1:复用"当前历史"携带的 state —— 上一帧同一个视图、参数未变、本帧未被消费
优先级 2:复用池(AvailableStates)中闲置的 state —— 参数未变、非本帧已用、属于同一视图
优先级 3:新建 state 并 ffxCreateContext
淘汰:池中 state 闲置超过 r.NG.DeferDelete(默认 5)帧 → 真正销毁
```

**并发正确性**:渲染线程可能同时有多个 `AddPasses`(立体渲染/多视图)。认领 state 时在锁内
盖 `LastUsedFrame = 当前帧号` 与 `ViewID`,后续并发调用看到"本帧已用"即跳过,杜绝双写同一个 context。

**为什么"本帧已消费"要重置历史**:立体渲染时左右眼共用同一渲染分辨率,参数相同可复用同一个
context,但两眼的画面内容不同(视差),历史必须丢弃,否则右眼会融合左眼的历史 → 鬼影。

### 2.3 参数比较驱动重建(IsNssContextParamsChanged)

上下文能否复用取决于 5 个参数是否变化:

```cpp
maxRenderSize / maxUpscaleSize / flags / qualityMode
```

- 分辨率变化(窗口缩放)→ 内部资源尺寸需重建;
- flags 变化(fragment 路径切换、深度反转、HDR)→ 着色器变体/管线需重建;
- qualityMode 变化 → 模型变体需重建。
比较命中任一变化,旧 context 销毁、按新参数创建;否则跨帧复用。

### 2.4 质量模式(Quality/Balanced/Performance)的代价模型

SDK 提供三档模型变体:质量越高,网络层数/通道越多,重建细节越好但 GPU 开销越大;
性能档反之。插件只负责:

- 把 `r.NSS.ShaderQualityMode` 映射为 `FfxApiNssShaderQualityMode`;
- 在创建 context 与 dispatch 时透传;切换档位时参数比较触发重建(§2.3)。
另外,质量档位与 `r.ScreenPercentage` 联动:渲染分辨率越低,越依赖模型重建能力,
因此代码把 `r.ScreenPercentage` 强制限制在 (0,100),越界重置为 50。

### 2.5 为什么设置 mip bias

渲染分辨率降低后,纹理采样 mip 选择仍按原分辨率引导,导致低分辨率下采样过度模糊。
NSS 通过改写 `r.ViewTextureMipBias.Min/Offset` 让 mip 选择偏向更清晰的高分辨率层级
(等价于负 LOD 偏移),补偿降分辨率采样的模糊。开关 NSS 时必须恢复原值,否则影响其他渲染。

### 2.6 1×1 资源保护与裁剪防越界

早期帧(引擎尚未产出全分辨率 RT)时,SceneColor/SceneDepth 可能是 1×1 占位;而 SceneVelocity
在无运动时也是 1×1。若直接 `vkCmdCopyImage` 到目标尺寸会**越界读取源图像**,在 Mali 驱动上
触发断言崩溃。处理:

- SceneColor/SceneDepth 小于目标尺寸 → 直接跳过本次 dispatch(BlankOutput);
- SceneVelocity 小于目标尺寸 → 创建同尺寸零纹理 + ClearUAV,让 SDK 收到合法的零速度。
这是移动端稳定运行的必需品,不是防御式过度设计。

## 3. 接口定义

```cpp
// NSSModule.h —— 供 MovieRenderPipeline 等外部一致使用的接口
class INSSModule : public IModuleInterface {
public:
    virtual NSS* GetNSSUpscaler() const = 0;              // 核心实例
    virtual INSS* GetTemporalUpscaler() const = 0;        // 同一实例按 ITemporalUpscaler 视图
    virtual bool IsPlatformSupported(EShaderPlatform Platform) const = 0;
    virtual void SetEnabledInEditor(bool bEnabled) = 0;
};
```

```cpp
// NSS.h —— 核心类(摘录关键成员)
class NSS final : public INSS, public IScreenSpaceDenoiser
{
public:
    void Initialize() const;                       // 解析后端、预检、包裹 denoiser(惰性,首次 AddPasses 前)
    INSS::FOutputs AddPasses(FRDGBuilder&, const NSSView&, const NSSPassInput&) const override;
    INSS* Fork_GameThread(const FSceneViewFamily&) const override;  // 返回 NSSProxy
    float GetMinUpsampleResolutionFraction() const override;        // 0.25f(支持时)
    float GetMaxUpsampleResolutionFraction() const override;        // 1.0f
    void SetPostProcessingInputs(FPostProcessingInputs const&);     // 绑定半透明等后处理输入
    void EndOfFrame();                               // 清 PostInputs、跑延迟清理
    void UpdateDynamicResolutionState();             // 读取动态分辨率状态
    void RequestHistoryReset() const;                // 标记下一帧 reset 历史
    void RunDeferredCleanup() const;                 // 回收过期状态
    // IScreenSpaceDenoiser 全部方法(DenoiseReflections/DenoiseAmbientOcclusion/…) → 转发 WrappedDenoiser
private:
    void DeferredCleanup(uint64 FrameNum) const;     // 状态池清理(§2.2 淘汰逻辑)
    mutable FPostProcessingInputs PostInputs;
    mutable TSet<NSSStateRef> AvailableStates;       // 可复用状态池(互斥锁保护)
    mutable EFFXBackendAPI Api;
    mutable INGSharedBackend* ApiAccessor;
    mutable FRDGBuilder* CurrentGraphBuilder;
    mutable const IScreenSpaceDenoiser* WrappedDenoiser;
    mutable TRefCountPtr<IPooledRenderTarget> MotionVectorRT;   // 稠密速度缓存
    mutable bool bNeedsHistoryReset;
    static float SavedScreenPercentage;
};
```

### NSSState / NSSHistory(状态生命周期)

```cpp
// NSSHistory.h
struct NSSState : public FRHIResource {          // 继承 RHI 资源 → 引用计数随 GPU 安全释放
    INGSharedBackend* Backend;
    ffxApiCreateContextDescNss Params;           // 创建参数(用于 §2.3 复用比较)
    ffxContext Nss;                              // SDK context
    uint64 LastUsedFrame;                        // 最近使用帧号(GFrameCounterRenderThread,§2.2 认领标记)
    uint32 ViewID;                               // 所属视图 UniqueID
    ~NSSState() { if (Backend && Nss) Backend->ffxDestroyContext(&Nss); }
};
typedef TRefCountPtr<NSSState> NSSStateRef;

class NSSHistory final : public INSSHistory, public FRefCountBase {
    NSSStateRef Nss;
    NSS* Upscaler;
    TRefCountPtr<IPooledRenderTarget> UpscaledColour;  // 上采样结果缓存(下一帧的 OutputTm1)
    ~NSSHistory() { if (模块初始化且 Upscaler) Upscaler->ReleaseState(Nss); }  // 归还池
    ffxContext* GetNSSContext() const;      // &Nss->Nss
    ffxApiCreateContextDescNss* GetNSSContextDesc() const;
    static TCHAR const* GetUpscalerName();  // "NSS"
};
```

设计意图:ffxContext 持有大块显存(Data Graph 会话)。因此 NSS 维护 **状态池**
(`AvailableStates`),配合 **延迟删除**(`CVarNGDeferDelete` 帧)减少创建/销毁抖动;
`NSSHistory` 作为引擎历史对象跨帧传递当前 context,析构时把 state 归还池而非销毁。

## 4. 启用链路(ViewExtension)

### 4.1 构造与注册

```cpp
// NSSModule::StartupModule:LoadModuleChecked("NGVulkanBackend");cook 检测;挂 OnPostEngineInit
// NSSModule::OnPostEngineInit:
//   检查 INGSharedBackend::GetApiAccessor(Api) 失败则禁用
//   ViewExtension = FSceneViewExtensions::NewExtension<NSSViewExtension>();

// NSSViewExtension 构造:
//   记录 r.ViewTextureMipBias.Min/Offset 初值(§2.5 恢复用);
//   若模块还没有 upscaler,创建 MakeShared<NSS>() 并 SetTemporalUpscaler
```

### 4.2 SetupViewFamily(游戏线程)

- 特性级别 ≥ ES3_1 才继续。
- 若 `CVarEnableNSS` 开且 upscaler 未初始化 → `Initialize()`;失败则把 `CVarEnableNSS` 置 0。
- 非编辑器或 `CVarEnableNSSInEditor` 时,若 `CVarNSSAdjustMipBias` 打开:
  改写两个引擎 CVar 以提升采样质量(§2.5):
  ```cpp
  CVarMinAutomaticViewMipBiasMin->Set(float(log2(1.f/3.f) - 1.f + FLT_EPSILON));
  CVarMinAutomaticViewMipBiasOffset->Set(float(-1.f + FLT_EPSILON));
  ```
- 监听 `CVarEnableNSS` 变化(`PreviousNSSState` 跟踪):重新打开时重设 mip bias,关闭时恢复原值。

### 4.3 BeginRenderViewFamily(游戏线程)—— 决定是否接管上采样

```cpp
// 遍历视图:任一视图 PrimaryScreenPercentageMethod == TemporalUpscale 即请求
if (IsTemporalUpscalingRequested && CVarEnableNSS && 视图族尚未设置 upscaler
    && (非编辑器 || CVarEnableNSSInEditor || 存在游戏视图))
{
    Upscaler->UpdateDynamicResolutionState();
    InViewFamily.SetTemporalUpscalerInterface(new NSSProxy(Upscaler));
}
```

### 4.4 PrePostProcessPass_RenderThread

- `CVarEnableNSS` 时把 `FPostProcessingInputs`(含半透明分离数据)传给 upscaler:
  `GetNSSUpscaler()->SetPostProcessingInputs(Inputs)`。

### 4.5 PostRenderViewFamily_RenderThread

- 帧末清理:`GetNSSUpscaler()->EndOfFrame()`(清 PostInputs、跑延迟清理、重置 editor 状态),
  避免悬垂指针与资源泄漏。

## 5. 核心流程 AddPasses(渲染线程,逐步)

### 5.0 前置

```cpp
InputExtents  = View.ViewRect.Size();          // 渲染分辨率
OutputExtents = View.GetSecondaryViewRectSize();// 上采样分辨率
Initialize();                                   // 惰性:解析后端/预检/包裹 denoiser
if (!IsApiSupported() || 非 TemporalUpscale) return BlankOutput(...);
```

- `BlankOutput`:创建纯黄(1,1,0)清屏纹理 + 空 NSSHistory,保证 NSS 关闭时管线仍能消费历史对象。
- 预检 `InitCreateContext`:用 8×8/16×16 最小尺寸创建一次 context 验证 SDK 可用(失败则禁用)。

### 5.1 历史有效性判定

```cpp
bHistoryValid = View.PrevViewInfo.TemporalAAHistory.IsValid() && View.ViewState && !View.bCameraCut;
if (bNeedsHistoryReset) bHistoryValid = false;   // 由 OnChangeNSSEnable 关闭回调触发
CanWritePrevViewInfo = !View.bStatePrevViewInfoIsReadOnly && View.ViewState;
```

### 5.2 上下文获取/复用/创建(核心算法,原理见 §2.2)

1. 构造 `ffxApiCreateContextDescNss Params`:
   - flags:`FFX_API_NSS_CONTEXT_FLAG_QUANTIZED`(仅支持量化模型)+ `DEPTH_INVERTED`(按 RHI 深度约定)
     + `HIGH_DYNAMIC_RANGE | DEPTH_INFINITE`;非 Windows 追加 `ALLOW_16BIT`;
     fragment 路径追加 `PRE_PROCESS_FRAGMENT | POST_PROCESS_FRAGMENT`。
   - `qualityMode` 取 `CVarNSSShaderQualityMode`(§2.4)。
   - `maxUpscaleSize` = OutputExtents;`maxRenderSize` = InputExtents。
   - 调试构建注册 `fpMessage = NSS::OnNSSMessage`(FFX 错误/警告 → UE_LOG)。
2. 复用判定(优先级):
   - **当前历史**:`CustomHistory->GetState()` 参数未变、`ViewID` 匹配本视图、且本帧未被消费
     (`LastUsedFrame != GFrameCounterRenderThread`)→ 直接复用(立体渲染二次使用时清历史防鬼影,§2.2)。
   - **状态池**:遍历 `AvailableStates`,跳过"本帧已用"或"属于其他视图"的,参数相同则认领
     (锁内盖 `LastUsedFrame`/`ViewID`,防止并发 AddPasses 双重认领),并 `bHistoryValid=false`。
   - 否则:**新建** `NSSState` 并 `ffxCreateContext`(失败则回滚并 BlankOutput)。
3. 新 state 加入池(`AvailableStates.Add`),使即使 UE 不回传历史(只读 PrevViewInfo)也能复用。
4. 若已有 context 但参数变化(`IsNssContextParamsChanged` 比较 maxRender/maxUpscale/flags/qualityMode,§2.3):
   销毁重建。

### 5.3 历史写入准备

```cpp
if (CanWritePrevViewInfo) {
    View.ViewState->PrevFrameViewInfo.TemporalAAHistory.SafeRelease();  // 释放旧历史纹理
    TemporalAAHistory.ViewportRect = FIntRect(0,0,OutputExtents.X,OutputExtents.Y);
    TemporalAAHistory.ReferenceBufferSize = OutputExtents;
}
NewHistory = new NSSHistory(CurrentNSSState, this);
```

### 5.4 输入整理与 Dispatch 参数

```cpp
ffxApiDispatchDescNss NssDispatchParams = {};
flags |= bRenderDebugViews ? FFX_API_NSS_DISPATCH_FLAG_DRAW_DEBUG_VIEW : 0;
reset       = !bHistoryValid;                       // 相机切换/无历史时放弃 SDK 内部历史(§2.1)
frameTimeDelta = 帧时长 ms                          // 网络据此缩放运动
jitterOffset   = -PassInputs.TemporalJitterPixels   // 符号取反(§2.1)
renderSize / upscaleSize
motionVectorScale = InputExtents                    // 速度单位换算
cameraFovAngleVertical = 垂直 FOV                   // 网络内部几何一致性
cameraNear/cameraFar = 深度反转约定(反转:近=FLT_MAX,远=近平面;否则近=近平面,远=FLT_MAX)
```

### 5.5 资源创建与鲁棒性保护

- `UpscaledOutputColorDesc`:Extent=OutputExtents,`Format=PF_FloatR11G11B10`,
  flags = fragment ? `ShaderResource|RenderTargetable` : `ShaderResource|UAV`。
  - **为什么 R11G11B10**:线性 HDR、半精度、带宽友好(相对 RGBA16F 减半);无 alpha 通道,
    上采样不需要 alpha。已知部分硬件的暗斑问题见架构总览 §11。
- **1×1 保护**(§2.6):SceneColor/SceneDepth 仍为 1×1 时直接 BlankOutput,避免 DDK 非法拷贝。
- `CopyAndCropIfNeeded`:把 SceneColor/SceneDepth/SceneVelocity 裁剪到 InputExtents;
  源比目标小时(如早期帧 SceneVelocity=1×1)创建零纹理 + ClearUAV,防 Mali 驱动断言。

### 5.6 Vulkan 后端路径(当前唯一实现)

1. 稠密速度转换:缓存 `MotionVectorRT`(PF_G16R16F,按 InputExtents 建),
   调 `helper_NGConvertVelocity`(见模块 03 §6)。
2. `DoAddPass` 模板(compute/fragment 两个参数结构体,`NSSPass::FComputeParameters/FFragmentParameters`):
   - 绑定 Color/Depth/Velocity(稠密)/OutputTm1(历史上采样结果或黑纹理)/Output/DebugViews。
   - PassFlags:fragment = `Raster|SkipRenderPass`;compute = `Compute|Raster|SkipRenderPass`。
   - lambda 内:
     - 按路径选状态:`PIXEL_READ/COMPUTE_READ`(读),`RENDER_TARGET/UNORDERED_ACCESS`(写);
       `PLATFORM_CPU_X86_FAMILY` 强制 UAV 写(模拟层不支持 fragment 读 tensor,自动回退 compute)。
     - `ApiAccess->ForceUAVTransition(RHICmdList, Output, RTV|UAVMask)`(显式布局转换)。
     - `RHICmdList.EnqueueLambda`:取活动 command buffer → `ffxDispatch(&State->Nss, &DispatchParams.header)`。
     - `RHICmdList.ImmediateFlush(DispatchToRHIThread)`。
3. 输出:调试视图开 → `Outputs.FullRes = DebugViews`;否则 `= UpscaledOutputColor`。

### 5.7 历史提取(Part 2)

```cpp
if (CanWritePrevViewInfo) {
    GraphBuilder.QueueTextureExtraction(UpscaledOutputColor,
        &View.ViewState->PrevFrameViewInfo.TemporalAAHistory.RT[0]);   // 引擎历史
    GraphBuilder.QueueTextureExtraction(UpscaledOutputColor, &NewHistory->UpscaledColour); // NSS 历史
}
Outputs.NewHistory = NewHistory;
```

## 6. CVar 回调与状态管理

| 回调 | 行为 |
|---|---|
| `OnChangeNSSEnable` | 开:`SaveScreenPercentage()` + `UpdateScreenPercentage()`(越界重置 50);关:`RestoreScreenPercentage()` + 渲染线程 `BlockUntilGPUIdle()` 后 `RequestHistoryReset()`(让 NX 内部状态干净) |
| `OnChangeScreenPercentage` | 开 NSS 时拦截 `r.ScreenPercentage`,强制 (0,100),越界日志 + 置 50 |
| `OnChangeNSSShaderQualityMode` | 越界 [0,2] 重置为 1(Balanced) |

- `SaveScreenPercentage/UpdateScreenPercentage/RestoreScreenPercentage`:质量模式强制时保存/恢复
  `r.ScreenPercentage`(NSS 需要屏幕百分比在 (0,100),理想值 50)。
- **为什么关闭 NSS 要 `BlockUntilGPUIdle`**:SDK 的异步执行(NX 引擎内部)在抢占/恢复场景下
  需要干净的状态;等 GPU 完全空闲后标记历史重置,避免下一次启用时读到半完成的历史。
- `DeferredCleanup(FrameNum)`:遍历池,`LastUsedFrame` 距当前超过 `CVarNGDeferDelete` 帧的 state 移除(释放 context)。

## 7. 调试视图布局

`r.NSS.Debug 1` 时输出 4×4 网格,每个格子是 SDK 内部中间 buffer,自上而下、自左而右:

| 行 | 内容 |
|---|---|
| 0 | history_color \| input_depth \| prev_depth \| nearest_offset |
| 1 | low_res_color \| motion_vector \| luma_deriv_tm1 \| temporal_feedback |
| 2 | lr_warped_history \| disocclusion_mask \| luma_deriv_t \| depth_dilated |
| 3 | unjittered_color \| motion_detector \| luma_instability \| warp_feedback |

这些 buffer 揭示算法内部:历史颜色、输入深度、**去遮挡掩码**(disocclusion,处理新暴露区域)、
**亮度导数**(luma_deriv,边缘/细节线索)、**运动检测器**、**扭曲后历史**(warped history)等——
调试视图是理解"学习式 TAA"各阶段的最佳工具。注意 `PLATFORM_CPU_X86_FAMILY` 下调试视图
强制 UAV(fragment 路径读 tensor 受限,见架构总览 §2.7)。

## 8. 关键常量与约定

- 上采样比例范围:`GetMinUpsampleResolutionFraction()=0.25f`,`GetMax…()=1.0f`。
- 输出格式:`PF_FloatR11G11B10`(线性 HDR,兼顾质量/性能;部分硬件有暗斑已知问题)。
- jitter 取反、速度 `(-0.5, 0.5)`、深度反转约定——三者是 NSS 与 SDK 之间的全局契约,
  与 NFRU 保持一致。

## 9. 复现要点(若需重新实现)

1. 必须同时实现 `ITemporalUpscaler` 与 `IScreenSpaceDenoiser` 两套接口;
   ViewExtension 在 `BeginRenderViewFamily` 里用 `SetTemporalUpscalerInterface(new NSSProxy(...))` 接入。
2. ffxContext 创建参数中 `DEPTH_INVERTED`、`QUANTIZED`、尺寸必须精确;
   上下文是否重建只取决于 `IsNssContextParamsChanged` 比较的 5 个字段。
3. 状态池 + 延迟删除是防抖关键;并发场景(立体渲染/多视图)用 `LastUsedFrame`+`ViewID` 双重认领防冲突。
4. 速度场必须转成稠密 `PF_G16R16F` 并乘 `(-0.5,0.5)`;jitter 取反;深度无限远约定(近=FLT_MAX)。
5. 1×1 资源保护与裁剪防越界(Mali 断言)是移动端稳定运行的必需处理。
6. 关闭 NSS 时必须 `BlockUntilGPUIdle` + 标记历史重置,否则 SDK 状态在抢占/恢复时出错。
7. mip bias 覆盖与恢复、ScreenPercentage 保存/恢复是"开关不残留副作用"的关键。
