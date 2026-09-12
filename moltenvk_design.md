# MoltenVK 架构与实现方案分析

> 基于本仓库源码分析（Vulkan 1.4 分支）。MoltenVK 是 Khronos 官方的 Vulkan → Metal 移植层，运行于 macOS / iOS / tvOS / visionOS。

---

## 1. 项目定位

MoltenVK 解决的核心问题是：Apple 平台没有原生 Vulkan 驱动，只有 Metal。它作为 **Vulkan ICD（Installable Client Driver）/ Layer** 实现，把整套 Vulkan 1.4 API 调用翻译成 Metal 调用，同时把 SPIR-V 着色器转换为 MSL（Metal Shading Language）。

- 只使用 Apple 公开 API，不含任何私有 API，应用可正常上架 App Store。
- 支持 `VK_KHR_portability_subset` 扩展，配合 Vulkan Loader 的 portability 枚举机制工作。

两大产物：

| 产物 | 说明 |
|---|---|
| **MoltenVK** 运行库 | Vulkan 1.4 几乎完整子集的实现，产出 `libMoltenVK.dylib` / xcframework。可作为 ICD 被 Vulkan Loader 加载，也可直接链接 |
| **MoltenVKShaderConverter** | SPIR-V → MSL 转换器。既内嵌于运行库做运行时转换，也打包为独立 macOS 命令行工具 |

---

## 2. 仓库顶层结构

```
MoltenVK/
├── MoltenVK/                  # 运行库主体
│   └── MoltenVK/
│       ├── API/               # 公共配置/数据类型头文件（对外 API）
│       ├── Commands/          # Vulkan 命令录制与 Metal 编码
│       ├── GPUObjects/        # VkInstance/Device/Image/Buffer 等资源对象
│       ├── Layers/            # Layer/Extension 枚举
│       ├── OS/                # 平台粘合（CAMetalLayer 等 ObjC 分类）
│       ├── Utility/           # 基础设施（对象池、SmallVector、日志等）
│       ├── Vulkan/            # Vulkan C API 入口适配层
│       └── icd/               # ICD JSON 描述文件
├── MoltenVKShaderConverter/   # SPIR-V → MSL 转换器（库 + 命令行工具）
├── Common/                    # 运行库与转换器共享代码（MVKOSExtensions 等）
├── Demos/Cube/                # 示例
├── Docs/                      # 用户指南、配置参数文档
├── Scripts/                   # 打包/CI 脚本
├── Templates/cmake/           # CMake 模板
├── cmake/recipes/             # CPM 依赖配方
└── fetchDependencies          # 依赖获取脚本
```

---

## 3. 构建体系

双构建系统并存：

- **Xcode 工程**（传统主路径）：`MoltenVK.xcodeproj`、`MoltenVKShaderConverter.xcodeproj`、`MoltenVKPackaging.xcodeproj`；顶层 `Makefile` + `fetchDependencies` 拉取 submodule 并通过 `Scripts/` 打包 xcframework。
- **CMake**（较新路径）：顶层 `CMakeLists.txt` + `cmake/recipes/`（CPM.cmake 拉取 SPIRV-Cross、SPIRV-Tools、Vulkan-Headers、cereal、Volk 等），支持 `MoltenVKOptions.cmake` 定制。

第三方依赖的版本由 `ExternalRevisions/*.repo_revision` 锁定（SPIRV-Cross、SPIRV-Tools、SPIRV-Headers、Vulkan-Headers、Volk、cereal 等），保证可复现构建。CI（`.github/workflows/CI.yml`）在 macOS runner 上构建并发布 artifact。

---

## 4. 运行库分层架构

### 4.1 总体调用链

```
应用程序
   │  Vulkan C API
   ▼
Vulkan/          （vulkan.mm —— 纯 C 符号入口 + 调用追踪包装）
   ▼
GPUObjects/      （MVKInstance / MVKDevice / 资源对象 —— 核心实现）
   ▼
Commands/        （命令录制 → 提交时 Metal 编码）
   ▼
Metal API ──► GPU
```

### 4.2 Vulkan/ —— API 适配层

`Vulkan/vulkan.mm`（约 4500 行）是全部 Vulkan 入口函数的集中定义处。每个函数遵循固定模板：

```cpp
MVK_PUBLIC_VULKAN_SYMBOL VkResult vkCreateInstance(...) {
    MVKTraceVulkanCallStart();          // 可配置的调用追踪
    MVKInstance* mvkInst = new MVKInstance(pCreateInfo);
    *pInstance = mvkInst->getVkInstance();
    ...
    MVKTraceVulkanCallEnd();
}
```

要点：

- `vkGetInstanceProcAddr` / `vkGetDeviceProcAddr` 是函数名 → 实现的分发点；设备级函数由 `MVKDevice::getProcAddr()` 解析。
- 通过 `MVK_CONFIG_TRACE_VULKAN_CALLS` 可配置打印每次 Vulkan 调用的进入/退出/耗时/线程 ID。
- `Vulkan/mvk_api.mm` 提供 MoltenVK 私有扩展 API（运行时查询/修改配置等）。

### 4.3 GPUObjects/ —— 核心对象层

Vulkan 对象几乎一一对应到 MVK C++ 类，均继承自 `MVKDispatchableVulkanAPIObject` / `MVKVulkanAPIObject` 基类（提供 `getVkInstance`、dispatch 表填充、debug name 等公共能力），内部持有对应 Metal 对象：

| Vulkan 对象 | MoltenVK 类 | 对应 Metal 对象 |
|---|---|---|
| VkInstance | `MVKInstance` | —（聚合 VkPhysicalDevice 列表） |
| VkPhysicalDevice | `MVKPhysicalDevice` | `id<MTLDevice>` |
| VkDevice | `MVKDevice` | —（聚合队列/资源/编码池） |
| VkQueue | `MVKQueue` | `id<MTLCommandQueue>` |
| VkBuffer | `MVKBuffer` | `id<MTLBuffer>` |
| VkImage / VkImageView | `MVKImage` / `MVKImageView` | `id<MTLTexture>` |
| VkDeviceMemory | `MVKDeviceMemory` | `id<MTLBuffer>`（含 memoryless texture 托管） |
| VkPipeline | `MVKPipeline`（Graphics/Compute） | `MTLRenderPipelineState` / `MTLComputePipelineState` |
| VkShaderModule | `MVKShaderModule` | SPIR-V + 转换后的 MSL + `MTLLibrary` |
| VkRenderPass / VkFramebuffer | `MVKRenderPass` / `MVKFramebuffer` | 渲染描述配置（Metal 无独立对象） |
| VkSemaphore / VkFence / VkEvent | `MVKSemaphore` / `MVKFence` / `MVKEvent` | `MTLSharedEvent` / CPU 条件变量 |
| VkSwapchainKHR | `MVKSwapchain` | `CAMetalLayer` + `CAMetalDrawable` |
| VkDescriptorSet 系列 | `MVKDescriptorSetLayout` / `MVKDescriptorPool` / `MVKDescriptorSet` | Metal argument buffer（参数缓冲） |

`MVKPixelFormats` 维护 Vulkan ↔ Metal 格式与能力映射表，是所有格式相关查询与转换的统一入口。`MVKDeviceFeatureStructs.def` 用 X-macro 统一管理几百个 feature/property 结构体的查询与合成。

### 4.4 Commands/ —— 命令录制与编码层

这是 MoltenVK 最具特色的部分：把 Vulkan 的"录制-提交"两阶段模型映射到 Metal 的"编码-提交"模型。

核心类：

- **MVKCommandBuffer**：对应 VkCommandBuffer。录制时 `addCommand()` 把命令对象追加进内部链表；`submit()` 时逐条编码为 Metal 调用。支持 `begin/end/reset` 生命周期、secondary command buffer（`recordExecuteCommands`）、以及 Metal 预填充模式（`clearPrefilledMTLCommandBuffer` 等）。
- **MVKCommand 基类 + 具体命令类**：`MVKCmdDraw*`、`MVKCmdDispatch*`、`MVKCmdTransfer*`、`MVKCmdRendering*`、`MVKCmdPipeline*`、`MVKCmdQueries*`、`MVKCmdDebug*`，每类一对 `.h/.mm`。命令对象从 `MVKCommandPool` 的类型化内存池分配；`MVKCommandTypePools.def` 用 X-macro 统一注册全部命令类型池，并对多参数命令按参数个数阈值生成特化模板类（如 `MVKCmdDrawIndexed<k>`），避免运行时动态分配。
- **MVKCommandEncoder**：提交时创建，把 MVKCommand 逐条翻译成 Metal 编码调用，管理 render/compute/blit 编码器的切换。
- **MVKCommandEncoderState** 及子类：跟踪管线绑定、资源绑定、顶点/片段/深度模板等 Metal 编码器状态，只把"脏"状态写入 Metal，避免冗余状态切换（`MVKStateTracking.h` 提供底层脏标记支持）。
- **MVKCommandEncodingPool / MVKCommandResourceFactory**：为内部辅助操作（清屏、查询、拷贝、水印等）缓存 Metal 管线状态、临时缓冲/图像。
- **MVKQueue**：持有 `MTLCommandQueue`，内部用 GCD 串行执行队列 + `MVKQueueSubmission` 对象序列化提交；支持同步/异步提交（`MVK_CONFIG_SYNCHRONOUS_QUEUE_SUBMITS`）与 Metal 命令缓冲错误回调。

录制 → 提交流程：

```
vkCmdDraw(...)   ──record──►  MVKCmdDraw 对象入链表（命令池分配）
vkEndCommandBuffer
vkQueueSubmit ──► MVKQueueCommandBufferSubmission
                     │ 遍历命令链表
                     ▼
              MVKCommandEncoder 将每条命令编码为
              MTLRenderCommandEncoder / MTLComputeCommandEncoder 调用
                     │
                     ▼
              [mtlCmdBuff commit] ──► GPU 执行 ──► fence/semaphore 信号
```

### 4.5 Layers/、OS/、Utility/、Common/

- **Layers/**：`MVKExtensions.def` + `MVKLayers.mm` 用 X-macro 声明/枚举支持的扩展与 Layer（含 `VK_KHR_portability_subset`）。
- **OS/**：ObjC 分类扩展 `CAMetalLayer`（swapchain 集成）、`MTLRenderPipelineDescriptor`、`MTLSamplerDescriptor`；`MVKGPUCapture` 支持 GPU 帧捕获。
- **Utility/**：基础设施——`MVKBaseObject`（对象基类）、`MVKObjectPool`（对象池，命令与编码资源大量使用）、`MVKSmallVector`（栈优先小向量，减少堆分配）、`MVKLogging.h`、`MVKInflectionMap`、`MVKCodec`（KTX/DDS 压缩纹理软解）、`MVKEnvironment.h`（编译期/运行期配置宏 `MVK_CONFIG_*`）。
- **Common/**：运行库与转换器共享的平台代码（`MVKOSExtensions`：时钟、锁、页大小、进程内存信息等）。

---

## 5. 着色器转换子系统

### 5.1 组成

```
MoltenVKShaderConverter/
├── MoltenVKShaderConverter/          # 转换库（运行库与工具共用）
│   ├── SPIRVToMSLConverter.{h,cpp}   # SPIR-V → MSL 主转换器
│   ├── SPIRVConversion.h             # 转换配置/结果数据结构
│   ├── SPIRVReflection.h             # SPIR-V 反射（资源绑定、入口点等）
│   ├── SPIRVSupport.{h,cpp}          # SPIR-V 解析/校验辅助
│   └── FileSupport / OSSupport       # 文件与 OS 辅助
├── MoltenVKShaderConverterTool/      # 独立 macOS 命令行工具（mssl / spirv 命令）
├── SPIRV-Cross/                      # 核心：SPIR-V → MSL 编译器（submodule）
└── SPIRV-Tools/                      # SPIR-V 优化/校验（submodule）
```

### 5.2 转换流程

1. **输入**：应用通过 `vkCreateShaderModule` 提供 SPIR-V。
2. **配置**：`SPIRVToMSLConversionConfiguration` 聚合转换上下文：
   - `SPIRVToMSLConversionOptions`：MSL 版本、纹理坐标 Y 翻转、entry point stage/name、tessellation/原点等固定管线状态；
   - `MSLResourceBinding` 列表：Vulkan set/binding → Metal buffer/texture/sampler 索引映射；
   - `MSLShaderInterfaceVariable`：顶点属性 location → MSL attribute ID；
   - 特化常量宏与 descriptor 类型信息（决定 MSL 中的资源类型，如 texture2d vs depth2d）。
3. **转换**：`SPIRVToMSLConverter::convert()` 驱动 **SPIRV-Cross** 的 `CompilerMSL` 生成 MSL 源码；`validateSPIRV()` 前置校验 SPIR-V 合法性；可选接入 SPIRV-Tools 优化。
4. **结果**：`SPIRVToMSLConversionResult`（MSL 源码 + 反射信息 + 转换日志）。`MVKShaderModule` 持有转换结果；`MVKPipeline` 创建时用 Metal 编译器将 MSL 编译为 `MTLLibrary`，再用 Metal 管线反射校验资源绑定，失败时回退重新转换。
5. **缓存**：`MVKPipelineCache`（对应 VkPipelineCache）借助 cereal 序列化库把 SPIR-V、转换配置与结果持久化到磁盘，加速二次启动；`MVKShaderModule` 内部也有 SPIR-V → MSL 的内存级缓存（keyed by conversion configuration）。

### 5.3 双形态复用

同一个转换库同时服务：

- 运行库内：`MVKShaderModule` 运行时按需转换；
- 独立工具：`MoltenVKShaderConverterTool` 在开发期离线批量转换（`-c msl` 命令），便于调试 MSL 输出。

---

## 6. 资源、内存与同步模型

### 6.1 内存模型

Vulkan 是"先分配 DeviceMemory，再 bind 到 Buffer/Image"；Metal 是"直接创建资源对象"。MoltenVK 的桥接方式：

- `MVKDeviceMemory` 持有真正的 Metal 资源（`MTLBuffer`，或 memoryless/managed 场景下的 `MTLTexture`），`MVKBuffer` / `MVKImage` 作为轻量视图绑定到其上（`MVKResource` 基类封装 bind 逻辑）。
- 主机可见内存：`MTLStorageModeShared` 直接映射 CPU 指针；`MTLStorageModeManaged` 额外在写入后触发 `didModifyRange` 同步；`MTLStorageModeMemoryless` 用于临时渲染目标。
- Sparse 资源通过 Metal 的 sparse texture/heap 机制映射（受设备能力限制）。

### 6.2 同步与屏障

- **Fence**：CPU 侧信号，基于 `MVKSemaphoreImpl`（信号量 + 条件变量）实现 CPU 等待。
- **Semaphore**：优先用 `MTLSharedEvent`（`MVKTimelineSemaphore` 支持 timeline 语义与 `deferSignal`/`encodeDeferredSignal` 延迟编码信号）；二进制信号量在 CPU/GPU 两种等待模式间选择。
- **Event**：基于 `MTLSharedEvent`，支持 host set/reset 与命令缓冲内 set/wait。
- **Barrier / Image Layout 转换**：Metal 没有显式内存屏障 API，MoltenVK 在 Metal 编码阶段用 `MTLBlitCommandEncoder`（同步纹理/缓冲内容）与 Metal 自动跟踪的 texture usage 标记来近似实现；跨 subpass 依赖转换为 Metal render pass 的 memory barrier 语义（`MVKCommandEncodingContext` 中的 `BarrierFenceSlots` 用 Metal fence 槽位串接跨编码器的依赖）。
- **QueryPool**：基于 Metal counter sample buffer 或 transform feedback 时序缓冲实现，`MVKQueryPool` 各子类对应 occlusion/timestamp/query 统计。

### 6.3 WSI / Swapchain

- `MVKSurface` 封装 `CAMetalLayer`；`MVKSwapchain` 管理一组 `MVKPresentableSwapchainImage`（可呈现图像）。
- 呈现：获取 `CAMetalDrawable`，把最终图像 blit/copy 到 drawable 纹理，`presentDrawable` 提交；支持 mailbox/fifo 等 present mode 的近似、HDR（EDR）元数据、`VkSwapchainPresentFenceInfoEXT` 等。
- 帧间隔由 Metal 的 drawable 队列（通常 3 张）天然实现，`vkAcquireNextImageKHR` 通过 signal handler + 信号量回调返回可用图像索引。

---

## 7. 配置与扩展机制

- **编译期**：`MVKEnvironment.h` 中 `MVK_CONFIG_*` 宏（默认值可被构建参数覆盖，如 `MVK_CONFIG_SYNCHRONOUS_QUEUE_SUBMITS`、`MVK_CONFIG_PREFILL_METAL_COMMAND_BUFFERS`、`MVK_CONFIG_MAX_ACTIVE_METAL_COMMAND_BUFFERS_PER_QUEUE` 等）。
- **运行期**：`mvk_config.h` / `vk_mvk_moltenvk.h` 暴露 `VkMoltenVKConfigurationMVK` 结构，应用可在创建实例前后读写配置；`MVKConfigMembers.def` 用 X-macro 统一配置成员的序列化与枚举。
- **扩展管理**：`MVKExtensions.def` X-macro 声明全部扩展，`MVKExtensions.mm` 维护启用状态。
- **ICD 集成**：`MoltenVK/icd/MoltenVK_icd.json` 描述 JSON 让 Vulkan Loader 发现并加载 `libMoltenVK.dylib`。

---

## 8. 关键设计取舍小结

| 问题 | MoltenVK 的方案 |
|---|---|
| Vulkan 对象模型 → Metal | 一一对应的 C++ 包装类 + dispatchable object 基类，VkHandle 即对象指针 |
| 录制-提交模型 | 命令对象池 + 链表录制，提交时统一编码；可选 Metal 预填充 |
| 着色器语言差异 | 运行时 SPIRV-Cross 转换 + pipeline cache 持久化 |
| 内存模型差异 | MVKDeviceMemory 持有 Metal 资源，Buffer/Image 做视图 |
| 无显式屏障 | Blit encoder + Metal fence 槽位 + usage 追踪近似 Vulkan 同步语义 |
| 堆分配开销 | MVKObjectPool / MVKSmallVector / 类型化命令池，热路径零堆分配 |
| 平台差异隔离 | OS/ 目录 ObjC 分类 + Common/ 共享平台代码 |
| 可配置性 | 编译期宏 + 运行期配置结构 + 私有 API（mvk_api.mm） |

---

## 9. 参考文档

- `Docs/MoltenVK_Runtime_UserGuide.md` —— 集成与使用指南
- `Docs/MoltenVK_Configuration_Parameters.md` —— 全部运行期配置参数说明
- `Docs/Whats_New.md` —— 版本变更
- `MoltenVK/MoltenVK/API/mvk_config.h` —— 配置项权威定义
