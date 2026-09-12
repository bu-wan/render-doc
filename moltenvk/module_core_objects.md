# 模块设计：核心对象层（GPUObjects — Instance/Device/Queue/PixelFormats）

> 本文是 [moltenvk_design.md](moltenvk_design.md) 的模块级细化文档，聚焦 Vulkan 实例/设备/队列/格式体系。总体架构见主文档。

---

## 1. 模块职责

实现 Vulkan 对象模型中"设备侧"的骨架：VkInstance、VkPhysicalDevice、VkDevice、VkQueue 及格式映射，是其余所有资源对象的宿主与工厂。

## 2. 对象基类体系

```
MVKBaseObject                  （调试名、报告消息、内存告警）
 └─ MVKVulkanAPIObject         （VkObjectType/debugReportType、getInstance）
     └─ MVKDispatchableVulkanAPIObject  （dispatch 表填充，VkHandle 即对象指针）
         ├─ MVKInstance
         ├─ MVKPhysicalDevice
         ├─ MVKQueue
         └─ MVKDevice
     └─ MVKVulkanAPIDeviceObject        （device 级资源对象基类）
         ├─ MVKResource → MVKBuffer / MVKImage
         ├─ MVKBufferView / MVKDeviceMemory
         ├─ MVKSemaphore / MVKFence / MVKEvent / MVKQueryPool
         ├─ MVKPipeline / MVKPipelineCache / MVKShaderModule
         ├─ MVKDescriptorSetLayout / MVKDescriptorPool / MVKDescriptorUpdateTemplate
         ├─ MVKRenderPass / MVKFramebuffer / MVKSwapchain / MVKSurface
         └─ MVKCommandPool / MVKPrivateDataSlot
```

关键约定：

- **dispatchable 对象**：`getVkHandle()` 返回自身指针，`getDispatchableObject()` 静态反解，C 入口零查找开销。
- **MVKDeviceTrackingMixin**：device 级对象混入，统一 `getDevice()/getInstance()` 访问。
- 对象创建统一走 `MVKInlineConstructible`（placement new + allocator 回调），支持 `VkAllocationCallbacks` 自定义分配器。

## 3. MVKInstance 详细设计

- **职责**：聚合物理设备、管理入口表、调试回调、实例级配置覆盖。
- **入口表**：`_entryPoints: unordered_map<string, MVKEntryPoint>`，构造时 `initProcAddrs()` 注册全部函数；`MVKEntryPoint` 记录函数指针 + 两个（core版本/扩展名）启用条件，`isEnabled()` 实现 Vulkan 规范中复杂的扩展-版本组合判定（代码注释原文承认"逻辑 horrible 但规范要求"）。
- **物理设备枚举**：构造时扫描 `MTLCopyAllDevices()`（macOS）或 `MTLCreateSystemDefaultDevice()`（iOS 系平台），生成 `_physicalDevices`；支持 `VkPhysicalDeviceIDProperties`（LUID/UUID）匹配。
- **配置覆盖**：若启用 `VK_EXT_layer_settings`，实例持有私有 `_mvkConfig` 与 `_mvkConfigStringHolders[]`（配置字符串生命周期由实例管理），否则回落全局配置。
- **调试回调**：`MVKDebugReportCallback` / `MVKDebugUtilsMessenger` 列表 + `_dcbLock` 互斥；`debugReportMessage()` 分发到所有已注册回调。

## 4. MVKPhysicalDevice 详细设计

- **职责**：包装 `id<MTLDevice>`，实现全部"只读查询"API（features/properties/limits/format/queue family/surface 支持性/外部内存能力）。
- **特征数据**（构造时一次性探测并缓存）：
  - `_properties`、`_limits`、GPU family（Apple/Mac/RD 类，`MTLGPUFamily`）；
  - `MVKPhysicalDeviceMetalFeatures`：Metal 侧原生能力快照（argument buffer 层级、纹理数组、memoryless、稀疏支持等），经 `vkGetPhysicalDeviceMetalFeaturesMVK` 暴露；
  - `MVKDeviceFeatureStructs.def`：X-macro 驱动的几百个 feature/property 结构体的批量查询实现。
- **能力裁剪**：根据 GPU family 与模拟器环境，从 Metal 真实能力推导 Vulkan features2 链各结构体（含端口子集 `VK_KHR_portability_subset` 的限制值）。
- **队列族**：`MVKQueueFamily` 按图形/计算/传输组合划分（Apple GPU 通常 1 族多功能，Intel/NV/AMD 显卡另计），每族持有少量 `MTLCommandQueue` 池（`kMVKQueueCountPerQueueFamily`）+ `_qLock`。
- **GPU 识别**：`isNVIDIAGPU()/isIntelGPU()`、`isMacGPUFamily1()`、`shouldEmulateReversedDepthViewport()` 等谓词驱动平台相关行为分支。
- WSI 查询：`getSurfaceSupport/getSurfaceCapabilities/getSurfaceFormats/getPresentModes`，直接换算 CAMetalLayer 属性（见 module_sync_wsi.md）。

## 5. MVKDevice 详细设计

- **职责**：VkDevice 的具体实现，是几乎所有运行时对象的创建工厂与资源协调中心。
- **聚合成员**：
  - 各队列族激活的 `MVKQueue` 实例（按 `VkDeviceQueueCreateInfo` 优先级创建）；
  - `MVKPixelFormats`（每设备一份，含本设备禁用的格式）；
  - `MVKCommandResourceFactory`（内部辅助 Metal 资源，全设备共享）；
  - 性能统计 `MVKPerformanceStatistics`（按 API/队列/着色器分组计数）；
  - 活动资源注册：`MVKLiveList`（`_liveResources`）在 debug 配置下跟踪存活对象，检测 use-after-destroy。
- **getProcAddr**：设备级函数入口解析；返回 `vkDestroyDevice`、`vkGetDeviceQueue` 等由 vulkan.mm 实现的静态函数或对象相关函数。
- **延迟释放**：`deferDestroy()` 机制将销毁操作排队到安全时机（避免正在 GPU 执行的对象被立即释放）。
- **私有扩展**：`vkSetMTLTextureMVK` 等互操作入口最终落到 device 级对象方法。

## 6. MVKQueue 详细设计

- **MVKQueueFamily**：持有 `MTLCommandQueue` 小池（每族 `kMVKQueueCountPerQueueFamily` 个），`getMTLCommandQueue(queueIndex)` 惰性创建；族属性（时间戳有效位、最小 image transfer 粒度等）静态维护。
- **MVKQueue**：
  - `_mtlQueue: id<MTLCommandQueue>`；提交路径见 module_commands.md。
  - **执行序列化**：双模式——同步模式（`MVK_CONFIG_SYNCHRONOUS_QUEUE_SUBMITS=1`，提交在调用线程完成编码，等待用条件变量 `_execQueueJobCount` 计数）或 GCD 串行 `_execQueue` 异步模式。
  - **提交对象**：`MVKQueueSubmission` 基类 → `MVKQueueCommandBufferSubmission`（命令缓冲+wait/signal 信号量+fence）/ `MVKQueuePresentSurfaceSubmission`（呈现）。每个提交对象封装一次 submit 的全部参数。
  - **Metal 命令缓冲获取**：`getMTLCommandBuffer(cmdUse)` 从 `_mtlQueue` 取出并打标签（`_mtlCmdBuffLabelQueueSubmit` 等预生成 NSString，避免热路径字符串构造）。
  - **错误处理**：`handleMTLCommandBufferError()` 在 Metal 命令缓冲失败时打印 GPU 错误日志并返回 `VK_ERROR_DEVICE_LOST`。
  - GPU capture：`MVKGPUCaptureScope` 支持 Xcode/Instruments 帧捕获集成。
- **waitIdle**：提交一个空 Metal 命令缓冲并 `waitUntilCompleted`，或直接等待执行队列清空。

## 7. MVKPixelFormats 详细设计

- **双表结构**：
  - `MVKVkFormatDesc[]`：VkFormat → MTLPixelFormat + VkFormatProperties3 + 能力标志；
  - `MVKMTLFormatDesc[]`：MTLPixelFormat → VkFormat + `MVKMTLFmtCaps`（读写/render/blend/filter/ms 等位标志）。
- **双向映射**：`getMTLPixelFormat(VkFormat)` / `getVkFormat(MTLPixelFormat)`；不支持的格式返回 `MTLPixelFormatInvalid`/`VK_FORMAT_UNDEFINED`。
- **能力推导**：`MVKMTLFmtCaps` 从 GPU family + macOS/iOS 版本推导（同一 MTLPixelFormat 在不同芯片上能力不同）；`getVkFormatProperties3()` 合成 Vulkan 三类属性（buffer/render/sample）。
- **YCbCr 多平面**：chroma subsampling 系列函数处理多平面格式（plane 数、每 plane 格式、块尺寸），支撑 `VK_KHR_sampler_ycbcr_conversion`。
- **特殊映射**：packed 格式、sRGB 别名（`compatibleAsLinearOrSRGB`）、depth/stencil 拆分视图（`MVKImageViewPlane` 子平面机制）。

## 8. 与其他模块的接口

- Commands 层从 MVKQueue 获取 `MTLCommandQueue`/`MTLCommandBuffer`；
- 资源对象构造均以 MVKDevice 为第一参数（`MVKVulkanAPIDeviceObject(device)`）；
- 格式判定贯穿资源创建、渲染管线、swapchain（`MVKPixelFormats` 是共享只读服务）；
- 配置来源：`MVKDevice::getMVKConfig()` 委托实例级或全局配置（见 module_api_entry.md §2.2）。
