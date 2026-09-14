# NeuralPostProcessing 模块分析

> 分析对象:UE 6.0.0 beta(fork 分支 `6.0.0-beta-ohos`)
> 特性状态:Experimental(插件 `NeuralRendering`,默认禁用)
> 本文要点:架构分层、执行流程、tile 系统、成熟度评估

## 1. 模块定位

NeuralPostProcessing 是 UE 6.0 新增的**实验性神经网络后处理**特性:允许在后处理材质(Post Process Material)中挂载一个神经网络模型(ONNX),由渲染管线在 GPU 上执行推理,典型用途是降噪、超分、风格化等基于学习的屏幕空间效果。

插件描述(`NeuralRendering.uplugin`):

- `FriendlyName`: "Neural Rendering"
- `Description`: "Enable neural rendering features including: neural post processing"
- `IsExperimentalVersion: true`,`EnabledByDefault: false`
- 平台白名单:仅 **Win64 / Linux / Mac**
- 唯一模块 `NeuralPostProcessing`(类型 `RuntimeAndProgram`,加载阶段 `PostConfigInit`),强制启用依赖插件 `NNERuntimeORT`

推理能力完全建立在 **NNE(Neural Network Engine)** 之上——UE 的 ONNX 推理抽象层。

## 2. 代码分布(三层解耦)

代码分散在三处,形成"引擎接口层 + 插件实现层"的架构,通过两个全局接口指针解耦:

| 位置 | 角色 |
|---|---|
| `Engine/Plugins/Experimental/NeuralRendering/Source/NeuralPostProcessing` | 实现模块(模型实例管理、tile 切块、RDG 执行、compute shader) |
| `Engine/Source/Runtime/Renderer/Private/PostProcess/NeuralPostProcess.cpp/.h`<br>`Engine/Source/Runtime/Renderer/Internal/PostProcess/NeuralPostProcessInterface.h` | 渲染器侧桥接:`INeuralPostProcessInterface` 纯虚接口,全局 `GNeuralPostProcess` 指针由插件注入 |
| `Engine/Source/Runtime/Engine`(NeuralProfile、材质表达式节点)<br>`Engine/Shaders/Private/NeuralPostProcessCommon.usf` | 资产定义、材质节点、材质 shader 公共代码 |

插件 `StartupModule()` 时注册两份实现:

```cpp
// NeuralPostProcessing.cpp
GNeuralProfileManager.Reset(new FNeuralProfileManager());  // 引擎侧 profile 管理接口
GNeuralPostProcess.Reset(new FNeuralPostProcess());        // 渲染器侧推理接口
```

引擎侧 `NeuralProfile.h` 同样只暴露接口 `NeuralProfile::INeuralProfileManager`(全局 `GNeuralProfileManager`),实现由插件提供——即插件禁用时引擎侧代码全部空转,`IsNeuralPostProcessEnabled()` 返回 false。

## 3. 三层架构详解

### 3.1 资产层 —— UNeuralProfile

文件:`Engine/Source/Runtime/Engine/Classes/Engine/NeuralProfile.h`

一个 NeuralProfile 资产(`FNeuralProfileStruct`)包含:

| 字段 | 说明 |
|---|---|
| `NNEModelData` | ONNX 导入的 `UNNEModelData` 模型数据 |
| `RuntimeType` | 推理运行时:`NNERuntimeORTDml`(DirectML,默认)/ `NNERuntimeRDGHlsl`(算子不全) |
| `InputFormat` / `OutputFormat` | 32bit/16bit 格式(**仅有字段,转换未实现**) |
| `InputDimension` / `OutputDimension` | 模型输入/输出维度(只读,由插件查询模型得到) |
| `BatchSizeOverride` | 批维度为动态(-1)时的覆盖值 |
| `TileSize` | 切块模式:`OneByOne` / `TwoByTwo` / `FourByFour` / `EightByEight` / `Auto` |
| `TileOverlap` | tile 重叠边框像素(Left\|Right, Top\|Bottom) |
| `TileOverlapResolveType` | 重叠区回写策略:`Ignore` / `Feathering` |

约束:

- 引擎最多同时挂载 `MAX_NEURAL_PROFILE_COUNT = 64` 个 profile
- tile 重叠上限为输入尺寸的 1/4(`ClampOverlap`)

`ENeuralModelTileType::Auto` 语义(枚举注释原文):模型输入维度 (1×3×200×200)、后处理 buffer 1000×1000 时,自动切 5×5 tile((5×5)×3×200×200)分别推理后重组。

### 3.2 材质层 —— 两个新材质表达式

文件:`Engine/Source/Runtime/Engine/Public/Materials/MaterialExpressionNeuralPostProcessNode.h`

**`UMaterialExpressionNeuralNetworkInput`**(继承 `UMaterialExpressionCustomOutput`)
- 在 **Neural Prepass** 中执行,把任意材质逻辑(通常是 SceneColor)写入网络输入 buffer
- 本质是 CustomOutput:材质图中它成为写入点,HLSL 侧由 `SaveToNeuralBuffer()` 承接

**`UMaterialExpressionNeuralNetworkOutput`**
- 在主 pass 中执行,读取网络输出 buffer
- HLSL 侧由 `ReadFromNeuralBuffer()` 承接

两者均支持两种寻址(`ENeuralIndexType`):

- `NIT_TextureIndex`:Texture2D viewport UV
- `NIT_BufferIndex`:张量索引(Batch, StartChannel, WH ∈ 0..1 映射到 B×C×H×W)

材质需勾选 **Used with Neural Networks** 使用标记并绑定 NeuralProfile(`FMaterial::IsUsedWithNeuralNetworks()` / `GetNeuralProfileId()`),渲染器用 `ShouldApplyNeuralPostProcessForMaterial()` 判定是否走神经路径。

公共 USF(`Engine/Shaders/Private/NeuralPostProcessCommon.usf`)提供材质侧可调用函数:

- `SaveToNeuralBuffer(Batch, StartChannel, BufferWH, float3 Value)` —— 仅在 `MATERIAL_NEURAL_POST_PROCESS && NEURAL_POSTPROCESS_PREPASS` 下编译,手写 NCHW 布局索引写 `InputNeuralBuffer`(struct-of-array:每通道一个 plane)
- `ReadFromNeuralBuffer(Batch, StartChannel, BufferWH, inout float3 Value)` —— 主 pass 读 `OutputNeuralBuffer`
- `UpdateNeuralProfileSourceType(int)` —— 预 pass 写 `NeuralSourceType[0]`,供后续 indirect dispatch 参数构建在 GPU 侧动态确定线程数(纹理为 0 / buffer 为 1)

### 3.3 管线层 —— hook 在后处理材质 pass 中

文件:`Engine/Source/Runtime/Renderer/Private/PostProcess/PostProcessMaterial.cpp`

`AddPostProcessMaterialPass()` 检测到神经材质(`PostProcessMaterial.cpp:918`)后,流程变为:

1. **Neural Prepass**(`AddNeuralPostProcessPass`,~line 585)
   - 创建独立临时输出纹理(不污染 SceneColor)
   - 以 `NEURAL_POSTPROCESS_PREPASS` shader 排列执行材质 pixel shader,把输入写入 `InputNeuralBuffer`,记录 SourceType
   - 仅当材质实际使用了神经参数时才绘制(`IsNeuralPostProcessShaderParameterUsed`)
2. **执行网络**:`ApplyNeuralPostProcess()` → 插件 `FNeuralPostProcess::Apply()`
3. **主 pass**:材质正常执行,但 shader 参数换绑为读侧(`SetupNeuralPostProcessShaderParametersForRead`:输出 buffer SRV + 结果纹理 SRV)

`FNeuralPostProcessResource` 承载一帧内的资源:输入/输出 buffer、SourceType buffer、结果纹理、维度向量、ProfileId。

## 4. 网络执行流程(插件侧)

核心类 `UNeuralPostProcessModelInstance`(`Private/NeuralPostProcessModelInstance.h/.cpp`),包装 NNE 的 `UE::NNE::IModelInstanceRDG`。

模型要求:**输入/输出张量均 rank-4(B×C×H×W)**,单输入单输出;动态维度默认解析为 1,批维度可由 profile 覆盖(`ModifyInputShape(0, BatchSize)`,仅可改符号维度)。

`ApplyNeuralNetworks_RenderingThread()`(NeuralPostProcessing.cpp)四步:

### 步骤 1 —— 预处理:切块打包

- 固定 tile 模式(非 Auto):先 `FDownScaleTextureCS` 把视口缩到网络输入尺寸(×TileDim)
- `FCopyBetweenTextureAndOverlappedTileBufferCS`(方向 = `ToOverlappedTiles`):把整幅图像切成带 overlap 边框的 tile,按 batch 维堆进 `TiledInputBuffer`
- dispatch 线程数由 GPU 侧 indirect args(`FNeuralPostProcessingBuildIndirectDispatchArgsCS` 读 SourceType)动态构建
- Auto 模式下视口未被 tile 整除的部分先清零(`@TODO: 只清边界区域`)

### 步骤 2 —— 推理(`Execute()`)

- `DispatchSize == 1`:单次 `EnqueueRDG`
- `DispatchSize > 1`(tile 数 > 模型 batch):循环逐 dispatch——copy 一段进网络 buffer → `EnqueueRDG` → copy 回 tiled buffer 对应槽位;**最后一次 dispatch 后插入 `SubmitAndBlockUntilGPUIdle()` 全帧同步**

### 步骤 3 —— 重组

- `FromOverlappedTiles` 方向把 tile 写回中间纹理
- overlap 区按 `ETileOverlapResolveType` 处理:
  - `Ignore`:重叠区无贡献
  - `Feathering`:双线性羽化过渡,消除 tile 接缝伪影(输入输出尺寸相同时效果最佳)
  - overlap 双向为 0 时自动降级为 Ignore(性能优化)
- 网络输出尺寸 ≠ 输入尺寸(如超分模型)时,`FUpscaleTexture` / `FDownScaleTexture` 缩放回视口尺寸(overlap 同比例换算)

### 步骤 4 —— 供材质读取

输出 tiled buffer + 维度(B×C×H×W,B 已乘 dispatch 数)绑定给主 pass 的 `ReadFromNeuralBuffer`。

### 管理类

- `FNeuralPostProcessModelInstanceManager`(单例):ProfileId → `UNeuralPostProcessModelInstance*` 映射
- `FNeuralProfileManager`(实现引擎接口):UpdateModel / RemoveModel / tile 与 batch 配置更新 / 查询模型 IO 维度(供编辑器 UI 显示)

### Console 变量

| CVar | 默认 | 说明 |
|---|---|---|
| `r.NeuralPostProcess.Apply` | 1 | 0 关闭推理(此时材质读回的是输入 buffer,便于调试材质侧) |
| `r.NeuralPostProcess.TileOverlap` | -1 | ≥0 时覆盖 profile 的 overlap |
| `r.NeuralPostProcess.TileOverlap.ResolveType` | -1 | 0=Ignore / 1=Feathering 覆盖 |
| `r.NeuralPostProcess.TileOverlap.Visualize` | 0 | 可视化 overlap 区域 |
| `r.NeuralPostProcess.TileOverlap.Visualize.Intensity` | 1.0 | 可视化强度 |

## 5. 成熟度评估:早期原型

架构设计是生产级思路(资产/材质/管线/运行时四层解耦、tile+overlap 抗接缝、GPU 侧 indirect dispatch),但实现有明显的未完成痕迹:

**性能硬伤**

- 多 dispatch 循环中 `SubmitAndBlockUntilGPUIdle()` 导致整帧 GPU stall,等待网络跑完——原型级同步方案,生产环境应改为 RDG pass 间的资源依赖

**未实现/TODO**

- `InputFormat`/`OutputFormat`(16bit)仅有枚举字段,无转换实现
- 材质输入节点的 `Mask` 输入注明"用于优化性能"但未实现
- `GetTotalModelTileCount` 的 Auto 分支返回 -1(实际 tile 数在别处运行时计算)
- `GetInputBuffer/GetOutputBuffer` 处 `//TODO: different type support`(仅 float)
- `@TODO: conditionally create the intermediate buffer texture`(中间纹理无条件创建)

**代码细节问题**

- 命名空间拼写错误:`NeuralPostProcessng`(缺 o);结构体 `FNueralPostProcessInput`(Neural→Nueral)
- `CreateRDGBuffers` 中输出 buffer 名字复制粘贴自 Subsurface:`"SubsurfacePostProcessing.OutputBuffer"`
- `ReadFromNeuralBuffer` 的编译条件写重复:`#if MATERIAL_NEURAL_POST_PROCESS && MATERIAL_NEURAL_POST_PROCESS`(应为 `!NEURAL_POSTPROCESS_PREPASS`?现状导致预 pass 也能读输出 buffer,无害但不严谨)
- 单输入单输出张量、仅 float——对模型结构约束较多

**整体判断**:架构完整、实现粗糙。官方放置于 Experimental 且默认禁用,符合预期。

## 附:关键文件清单

| 文件 | 内容 |
|---|---|
| `Engine/Plugins/Experimental/NeuralRendering/NeuralRendering.uplugin` | 插件描述 |
| `Engine/Plugins/Experimental/NeuralRendering/Source/NeuralPostProcessing/NeuralPostProcessing.Build.cs` | 模块构建(依赖 NNE/RenderCore/RHI/Renderer) |
| `.../Private/NeuralPostProcessing.cpp` | 模块入口、profile 管理器、执行主流程(795 行) |
| `.../Private/NeuralPostProcessModelInstance.h/.cpp` | NNE 模型实例包装、RDG buffer、dispatch 循环 |
| `.../Private/NeuralPostProcessingCS.h/.cpp` | 全局 compute shader 声明(tile 搬运/缩放/indirect args) |
| `.../Shaders/NeuralPostProcessing.usf` | compute shader 实现(515 行) |
| `Engine/Source/Runtime/Renderer/Internal/PostProcess/NeuralPostProcessInterface.h` | 渲染器侧纯虚接口 |
| `Engine/Source/Runtime/Renderer/Private/PostProcess/NeuralPostProcess.cpp/.h` | 渲染器桥接、资源分配、参数绑定 |
| `Engine/Source/Runtime/Renderer/Private/PostProcess/PostProcessMaterial.cpp` | 管线 hook(AddNeuralPostProcessPass,~line 585/918) |
| `Engine/Source/Runtime/Engine/Classes/Engine/NeuralProfile.h` | profile 资产与引擎侧接口 |
| `Engine/Source/Runtime/Engine/Private/Rendering/NeuralProfile.cpp` | profile 注册/生命周期 |
| `Engine/Source/Runtime/Engine/Public/Materials/MaterialExpressionNeuralPostProcessNode.h` | 两个材质表达式节点 |
| `Engine/Shaders/Private/NeuralPostProcessCommon.usf` | 材质侧读写函数(97 行) |
