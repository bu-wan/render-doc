# Neural Graphics Plugin for Unreal® Engine — 工程分析文档

本目录是对仓库 `neural-graphics-for-unreal`(Arm Neural Graphics Plugin,包含 UE4.27 / UE5.4 / UE5.6
三个引擎分支的插件)的完整逆向分析文档。
目标是:**仅凭这些文档即可重新实现插件的全部能力**(NSS 神经超采样 + NFRU 神经帧率提升,
底层算法由 Neural Graphics SDK for Game Engines 提供)。

**文档约定**:不记录发布版本号、日期与变更史(由 git 管理);引擎分支用仓库目录名(UE4.27 / UE5.4 / UE5.6)指代;
代码片段内的引擎 API 版本门控属于源码内容,予以保留。每篇文档均含:

- **原理与算法分析**章节:先讲清"为什么这样做"(时序超采样/帧生成的算法本质、FFX 模型、Vulkan 扩展机制、
  状态池、节拍器等),再进入代码;
- **详细注释的代码片段**:关键逻辑逐行注解;
- **复现要点**:重新实现时必须满足的硬性条件与易踩的坑。

## 文档索引

| 文档 | 内容 | 阅读顺序 |
|---|---|---|
| [00-架构总览.md](./00-架构总览.md) | 项目定位、原理与算法分析(时序超分/帧生成/FFX模型/Vulkan扩展/延迟删除/复制头)、仓库布局、模块划分与依赖图、启动时序、渲染管线接入点、每帧数据流、引擎版本适配机制、CVar 总表、构建打包、平台矩阵 | **先读** |
| [01-模块-NGShared.md](./01-模块-NGShared.md) | 公共层:后端抽象原理、FFX/SDK 头封装、`INGSharedBackend` 接口、格式/状态映射、复制头 + 宏改写原理 | 2 |
| [02-模块-NGSettings.md](./02-模块-NGSettings.md) | CVar 机制原理(线程安全/双向绑定/ini 回放)、全部控制台变量、`UNGSettings` 编辑器界面 | 3 |
| [03-模块-NGVulkanBackend.md](./03-模块-NGVulkanBackend.md) | 扩展/层注册原理(为何必须在设备创建前)、DLL 加载、Vulkan 后端桥接、速度场转换算法、深度反转约定、Android APL | 4 |
| [04-模块-NSS.md](./04-模块-NSS.md) | NSS 上采样:学习式 TAA 算法、ffxContext 状态池/延迟删除、参数比较重建、质量模式代价模型、`AddPasses` 全流程、历史管理 | 5 |
| [05-模块-NFRU.md](./05-模块-NFRU.md) | NFRU 帧生成:光流/变形/合成算法、双缓冲 ping-pong、CustomPresent 状态机、Slate 重绘、UI 差异捕获算法、自适应 FPS 节拍器 | 6 |
| [06-模块-SmokeTests.md](./06-模块-SmokeTests.md) | 自动化冒烟测试:延迟命令机制、截图取证流程、三个用例 | 7 |
| [07-引擎分支差异与移植指南.md](./07-引擎分支差异与移植指南.md) | 引擎分支差异总表、差异技术原因、新引擎移植清单、从零重建 8 步、常见坑速查 | 8 |

## 快速速览(30 秒版)

- **NSS**:实现 `UE::Renderer::Private::ITemporalUpscaler`,在视图请求 `TemporalUpscale`
  时通过 `SetTemporalUpscalerInterface(NSSProxy)` 替换引擎 TAA 上采样;`AddPasses` 把
  低分辨率 SceneColor/SceneDepth/速度 + 相机参数交给 NG-SDK(`ffxDispatch(NSS)`)得到
  全分辨率 R11G11B10 输出;ffxContext 用"状态池 + 延迟删除"复用,避免显存抖动。
  本质是**学习式 TAA**:用神经网络替代手工滤波融合"带 jitter 的当前帧 + 重投影历史"。
- **NFRU**:ViewExtension 采集每视图输入 → `GEngine->GetPostRenderDelegateEx` 每帧调
  `InterpolateFrame` → `ffxDispatch(PREPARE + FRAMEGENERATION)` 生成 1 帧插值帧 →
  自定义 `FRHICustomPresent` + Slate 强制重绘交替呈现真实帧/插值帧;附带 UI 调试捕获
  与自适应 FPS 节拍器。
- **共同底座**:`NGVulkanBackend` 加载 `ngsdk_*.dll/so`、在 RHI 初始化前注册
  `VK_ARM_tensors`/`VK_ARM_data_graph` 扩展与 `VK_LAYER_ARM_NG` 层,并通过
  `INGSharedBackend` 把 UE 资源桥接为 SDK 原生资源;`NGSettings` 集中管理 CVar。

## 覆盖范围说明

- 源码覆盖:三个引擎分支全部 6 个模块的 `.Build.cs`、公开/私有头文件、实现文件、着色器
  (USF)、插件清单(`.uplugin`)、构建脚本(`BuildSDK.py/.bat`)、Android APL。
- 外部依赖(不在本仓库,按需获取):Neural Graphics SDK for Game Engines(子模块,
  https://github.com/arm/neural-graphics-sdk-for-game-engines)、
  Arm ML Emulation Layer for Vulkan(https://github.com/arm/ai-ml-emulation-layer-for-vulkan)。
- 已知未深挖:NG-SDK 内部算法(量化模型、Data Graph 执行)属于 SDK 源码范畴,本文档聚焦
  插件侧"如何桥接与驱动 SDK",但通过调试视图布局、参数语义与算法章节解释了 SDK 的
  输入契约与内部阶段。

## 重新实现的最小路径

1. 按 [07-引擎分支差异与移植指南.md](./07-引擎分支差异与移植指南.md) §3 的 8 步顺序实施;
2. 每步结束用该文档 §2.5 的验证手段自测;
3. 对照各模块文档末尾的"复现要点"逐条核对(状态池、扩展注册时机、速度场符号、
   1×1 资源保护、Flush 同步、帧率恢复等)。
