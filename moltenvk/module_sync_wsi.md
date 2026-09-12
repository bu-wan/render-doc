# 模块设计：同步与 WSI（Sync/Swapchain/Surface/Barrier）

> 本文是 [moltenvk_design.md](moltenvk_design.md) 的模块级细化文档，聚焦 Fence/Semaphore/Event 的 Metal 实现策略与窗口系统集成。总体架构见主文档。

---

## 1. 模块职责

- 将 Vulkan 的 CPU/GPU 同步原语（Fence、Semaphore、Event、Timeline Semaphore、Deferred Operation）映射到 Metal 提供的机制。
- 实现 WSI：VkSurfaceKHR/VkSwapchainKHR 基于 CAMetalLayer/CAMetalDrawable。
- 为 Vulkan 屏障语义（无 Metal 等价 API）提供落地策略。

## 2. CPU 侧基础件：MVKSemaphoreImpl

`MVKSemaphoreImpl`（MVKSync.h:42）是纯 CPU 信号量：`mutex + condition_variable` + 预留计数。

- `reserve()/release()/wait(timeout, reserveAgain)`；
- `waitAll=false` 模式下一次 release 唤醒全部等待者；
- 用于 Fence 的 CPU 等待与内部资源租借（如 MTLCommandQueue 池）。

## 3. Semaphore 体系（策略类层次）

`MVKSemaphore` 抽象基类定义四个关键虚函数，**同一 API 可被"盲调两次"**（一次带 mtlCmdBuff 编码等待，一次不带以阻塞），由子类 `isUsingCommandEncoding()` 决定实际行为：

| 子类 | 机制 | encodeWait/Signal | 适用场景 |
|---|---|---|---|
| `MVKSemaphoreSingleQueue` | Metal 单队列提交顺序 + hazard tracking 保证 | 不编码（CPU 无操作） | 单队列应用，零开销 |
| `MVKSemaphoreMTLEvent` | `MTLEvent` + `std::atomic<uint64_t> _mtlEventValue` | 在 MTLCommandBuffer 上 encodeWait/encodeSignal | GPU-GPU 同步首选 |
| `MVKSemaphoreEmulated` | CPU：MTLFence（GPU 侧）+ `MVKSemaphoreImpl`（CPU 侧） | 不编码 | 跨队列/CPU 等待 |
| `MVKSemaphoreMTLSharedEvent` | `MTLSharedEvent` | 编码 | 跨进程/导出场景 |

选择逻辑：由 `MVK_CONFIG_SYNCHRONOUS_QUEUE_SUBMITS`、设备能力与应用用法（单/多队列）在创建时决定（MVKDevice 创建 semaphore 时按配置实例化对应子类）。

### 3.1 Timeline Semaphore

- `MVKTimelineSemaphore : MVKSemaphore`（二进制基类的 timeline 扩展），实现子类 `MVKTimelineSemaphoreMTLEvent`（基于 MTLSharedEvent 的 timeline 值语义）。
- `deferSignal()` / `encodeDeferredSignal(mtlCmdBuff, token)`：两段式信号——先取 token，再在确切时机编码信号。**专为 swapchain 设计**：保证 image-acquire 信号量使用正确的 MTLSharedEvent 值（注释见 MVKSync.h:164-183）。

## 4. Fence / Event / DeferredOperation

- **MVKFence**：CPU 侧一次性信号。`MVKFenceSitter` 聚合多个 fence/timeline semaphore 供 `vkWaitForFences` 统一等待（避免逐个轮询）。
- **MVKEvent**：
  - `MVKEventMTLEvent`：基于 `MTLSharedEvent`，host set/reset 直接改 shared event 值，GPU 内 set/wait 编码为 MTLEvent 信号/等待命令；
  - `MVKEventSingleThreaded`：单线程优化的 CPU 实现（无 Metal 开销）。
  - `encodeSignal(mvkEvent, status)` 由 MVKCommandEncoder 提供（Commands 层）。
- **MVKDeferredOperation**：`VK_KHR_deferred_host_operations` 的实现——把工作函数（如管线编译）包进可延迟执行的槽位，支持 `vkDeferredOperationJoinKHR` 多线程汇合（`MVKDeferredOperationFunctionType` 枚举当前支持的操作类型）。

## 5. 屏障与布局转换的落地

Metal 没有显式 API 级内存屏障。MoltenVK 的组合策略：

1. **Metal hazard tracking**：Metal 默认的渲染/资源跟踪覆盖大部分 buffer/image 冲突，MVK 尽量依赖（`MTLHazardTrackingModeTracked`）。
2. **`MVKPipelineBarrier` 解析**：Commands 层（MVKCmdPipeline）把 `vkCmdPipelineBarrier` 的各 stage/access/src-dst mask 解析为资源级操作：
   - host-visible 内存的 host↔device 一致性：对 Managed 存储触发 `didModifyRange`/blit 同步；
   - image layout 转换：MVKImagePlane 记录新布局，必要时 blit encoder 拷贝/同步；
3. **BarrierFenceSlots**（MVKCommandBuffer.h:53，位于 `MVKCommandEncodingContext`）：跨 Metal 编码器/命令缓冲的执行依赖用 **MTLFence 槽位数组**（按队列/用途分槽）串接——前一编码器 `encodeWaitForFence`，后一编码器 `encodeSignalFence`，近似 Vulkan 的 execution dependency chain。
4. **subpass 依赖**：render pass 内的依赖映射为 Metal render pass 的 memory barrier 调用（`MTLRenderCommandEncoder memoryBarrierWithScope`，能力受限时退化为 store/load action 与多 pass 拆分）。

## 6. WSI：Surface / Swapchain

### 6.1 MVKSurface

- 包装 `CAMetalLayer*`：`initLayer(mtlLayer, vkFuncName, isHeadless)`；
- 三种创建源：`VkMetalSurfaceCreateInfoEXT`（直接传 CAMetalLayer，推荐）、`VkHeadlessSurfaceCreateInfoEXT`（headless：`isHeadless()==true`，无 layer）、旧版 `Vk_PLATFORM_SurfaceCreateInfoMVK`（传 NSView/UIView，取其 layer）。
- `getCAMetalLayer()` 供 swapchain 与能力查询使用；`getRenderSurfaceExtent()` 返回 layer 无缩放的自然尺寸。

### 6.2 MVKSwapchain

- **initCAMetalLayer()**：把 Vulkan 参数翻译到 CAMetalLayer——pixelFormat（MVKPixelFormats 查询）、maximumDrawableCount（由 present mode 推导：FIFO→2-3、MAILBOX→2 等）、颜色空间（EDR/HDR via `VkSwapchainColorSpaceKHR`）、opaque/composited 标志。
- **图像管理**：创建 `MVKPresentableSwapchainImage` 数组（数量 = minImageCount），每张图像通过 IOSurface 与 CAMetalDrawable 关联；图像生命周期由 present history 跟踪。
- **present history 环形缓冲**（`_presentHistoryCount/Index/HeadIndex` + `_presentHistoryLock`）：记录最近 N 次呈现时间，用于 pacing 与"最旧可复用图像"选择（内存警告时收缩图像数量，`markAsTracked`/`markAsNotTracked` 配合）。
- **acquireNextImage()**：
  1. 从 CAMetalLayer 取可用图像（内部由 `MTLSharedEvent`/deferred signal 管理占位状态）；
  2. 返回索引，把图像可用性信号量通过 `deferSignal()/encodeDeferredSignal()` 与 acquire 时刻绑定；
  3. 超时语义由 `MVKSemaphoreImpl::wait(timeout)` 支撑。
- **呈现路径**：`vkQueuePresentKHR` → `MVKQueuePresentSurfaceSubmission` → 每张图像 `presentCAMetalDrawable`（把最终内容 blit 到 drawable 纹理后 `[mtlCmdBuff presentDrawable:]`，实际调用在 MVKImage.mm 的 presentable image 实现中）。
- **兼容 present mode**：`_compatiblePresentModes` 报告与所选 mode 兼容的其他 mode（FIFO/MAILBOX/IMMEDIATE 的 Metal 近似）。
- **surface 失效**：窗口尺寸/layer 变化通过 CAMetalLayer 通知链（`CAMetalLayer+MoltenVK` 分类、`MVKBlockObserver` KVO 观察 drawableSize）触发 `VK_ERROR_OUT_OF_DATE_KHR`/`VK_SUBOPTIMAL_KHR`。

### 6.3 呈现时机与帧控制

- Metal drawable 队列（通常 3）天然实现 FIFO 节流；`vkAcquireNextImageKHR` 在无可用 drawable 时阻塞在 Metal 分配器上。
- `MVKSwapchain::getPresentProgress` 类的 present-history 机制支撑 `VK_GOOGLE_display_timing`（`vkGetRefreshCycleDurationGOOGLE/vkGetPastPresentationTimingGOOGLE`，数据源即 present history 环）。

## 7. 与其他模块的接口

- Queue 层：present 提交对象、semaphore 的 wait/signal 编码时机（module_commands.md §6）。
- Image 层：presentable image 继承 MVKImage，IOSurface 共享（module_resources_memory.md §4）。
- Commands 层：`BarrierFenceSlots` 与 `MVKPipelineBarrier` 的解析/落地协作。
- API 层：`vkGetMTLTextureMVK`/`vkUseIOSurfaceMVK` 互操作（module_api_entry.md §2.2）。
