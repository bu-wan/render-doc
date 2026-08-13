# VkNRC 项目代码分析文档

本文档集对 Vulkan Neural Radiance Caching(VkNRC)项目进行深入代码分析,涵盖整体架构、模块设计、实现方案、关键算法与原理。分析基于当前 master 分支(`a667859`)源码。

## 项目简介

VkNRC 是 NVIDIA 论文 *[Real-time Neural Radiance Caching for Path Tracing](https://research.nvidia.com/publication/2021-06_real-time-neural-radiance-caching-path-tracing)*(NRC) 的 Vulkan 实现。核心亮点:

- 使用 `VK_NV_cooperative_matrix` 实现**全融合 MLP(Fully-Fused MLP)**,前向/反向传播全部在单个 compute shader 内完成,反传性能优于作者官方的 tiny-cuda-nn 实现。
- 基于自研的 **MyVK** 库(轻量 Vulkan C++ 封装 + 渲染图框架 `myvk_rg`),声明式描述帧内所有 Pass 及其资源依赖,由框架自动完成调度、屏障与描述符管理。
- 渲染管线:V-Buffer 预通道 → 路径追踪(ray query)→ NRC 推理/训练 → 屏幕合成,全部在一条命令缓冲内执行,支持 3 重帧缓冲并行。

## 文档目录

| 文档 | 内容 |
| --- | --- |
| [01-架构总览.md](01-架构总览.md) | 总体架构、模块划分、渲染管线流程、数据流、线程模型 |
| [02-应用层模块.md](02-应用层模块.md) | main、Camera、Scene、VkScene、VkSceneBLAS/VkSceneTLAS、VkNRCState 设计与实现 |
| [03-渲染图框架.md](03-渲染图框架.md) | myvk_rg 渲染图框架:接口层、依赖分析、调度、屏障、描述符、命令生成 |
| [04-MyVK封装层.md](04-MyVK封装层.md) | MyVK Vulkan 资源封装:Device、Buffer、Image、Allocator、FrameManager 等 |
| [05-NRC算法与着色器.md](05-NRC算法与着色器.md) | NRC 算法原理、全融合 MLP 实现、路径追踪与训练着色器详解 |
| [06-构建与依赖.md](06-构建与依赖.md) | CMake 构建体系、第三方依赖、硬件/扩展要求 |

## 快速导览

```
VkNRC/
├── CMakeLists.txt          # 顶层构建
├── src/                    # 应用程序层
│   ├── main.cpp            # 程序入口(窗口、设备、主循环)
│   ├── Camera.{hpp,cpp}    # 自由视角相机
│   ├── Scene.{hpp,cpp}     # OBJ 加载与实例化(SAH 划分)
│   ├── VkScene.{hpp,cpp}   # GPU 场景资源上传
│   ├── VkSceneBLAS.{hpp,cpp}# 底层加速结构构建(含压缩)
│   ├── VkSceneTLAS.{hpp,cpp}# 顶层加速结构构建/更新
│   ├── VkNRCState.{hpp,cpp}# NRC 状态:权重、优化器、累积缓冲
│   └── rg/                 # 渲染图 Pass 集合
│       ├── NRCRenderGraph  # 渲染图定义与资源接线
│       ├── VBufferPass     # 可见性预通道
│       ├── PathTracerPass  # 路径追踪主着色器
│       ├── NNInference     # NRC 前向推理
│       ├── NNTrain         # NRC 训练(梯度+优化器)
│       └── ScreenPass      # 色调映射/累积合成
├── shader/src/             # GLSL 着色器源码
├── dep/MyVK/               # 自研 Vulkan 封装库
└── test/                   # 训练核独立测试(可选)
```
