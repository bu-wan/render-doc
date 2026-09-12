# 模块设计：命令系统（Commands/）

> 本文是 [moltenvk_design.md](moltenvk_design.md) 的模块级细化文档，聚焦命令录制、命令池、Metal 编码与状态跟踪。总体架构见主文档。

---

## 1. 模块职责

实现 Vulkan 的"录制-提交"两阶段模型：录制期把 `vkCmd*` 调用物化为轻量命令对象，提交期由编码器逐条翻译为 Metal 编码调用，并以脏状态跟踪最小化 Metal 状态切换。

## 2. 类体系总览

```
MVKCommandPool ──── 持有 MVKCommandTypePool<T>（每命令类型一个对象池）
                     （池定义由 MVKCommandTypePools.def X-macro 生成）

MVKCommandBuffer ── 命令链表首尾指针 + 录制/提交状态机
MVKCommand ──────── 抽象基类：encode(cmdEncoder) + getTypePool(cmdPool)
 ├─ MVKSingleValueCommand<Tv>      单值命令模板（如 vkSetEvent）
 ├─ MVKCmdXxx<k>                   按参数个数阈值特化的模板命令类
 │   MVKCmdDraw / MVKCmdDrawIndexed / MVKCmdDispatch / MVKCmdCopyImage ...
 └─ MVKCommandsCommandBuffer       嵌入式 secondary 命令缓冲

MVKCommandEncoder ─ 提交期编码上下文（每 Metal 命令缓冲一个）
 ├─ MVKCommandEncoderState 体系（脏状态跟踪）
 ├─ MVKCommandEncodingPool（辅助 Metal 资源缓存）
 └─ MVKCommandResourceFactory（辅助管线/缓冲/图像工厂）
```

## 3. 命令对象设计

### 3.1 MVKCommand 基类

```cpp
class MVKCommand : public MVKBaseObject, public MVKLinkableMixin<MVKCommand> {
    virtual void encode(MVKCommandEncoder* cmdEncoder) = 0;
    virtual MVKCommandTypePool<MVKCommand>* getTypePool(MVKCommandPool* cmdPool) = 0;
};
```

- 继承 `MVKLinkableMixin` 获得链表 prev/next 指针，命令缓冲内串成双向链表。
- 每个具体命令类必须提供 `setContent(MVKCommandBuffer*, ...)`（vulkan.mm 的模板按此约定填充参数）与静态 `getTypePool()`（实现在 MVKCommandPool.mm 中由 X-macro 批量生成，按类名分发到对应类型池）。

### 3.2 类型化对象池

- `MVKCommandTypePool<T> : MVKObjectPool<T>`：`newObject()` 为 `new T()`；回收时仅析构语义重置、内存复用（`isPooling` 可关闭以支持内存调试）。
- **阈值特化**：多参数命令按参数数量生成 `MVKCmdDrawIndexed<k>` 特化类（`MVK_CMD_TYPE_POOLS_FROM_n_THRESHOLDS` 宏），把顶点属性/附件数量等编译期定长，避免堆上可变数组——热路径零堆分配。
- `MVKCommandTypePools.def` 是全部命令类型池的唯一注册点，新增命令 = 新类 + def 中加一行。

### 3.3 命令文件组织

按 Vulkan 命令域分文件，每类一对 `.h/.mm`：

| 文件 | 覆盖命令域 |
|---|---|
| MVKCmdPipeline.h/mm | bindPipeline、管线屏障（`MVKPipelineBarrier` 结构在此解析） |
| MVKCmdRendering.h/mm | beginRenderPass/nextSubpass/endRenderPass、dynamic rendering |
| MVKCmdDraw.h/mm | draw 系列全家族（含 indirect、multiDraw、mesh） |
| MVKCmdDispatch.h/mm | dispatch 系列（含 indirect） |
| MVKCmdTransfer.h/mm | copy/blit/fill/update/clear/query resolve |
| MVKCmdQueries.h/mm | beginQuery/endQuery/resetQueryPool/copyQueryPoolResults |
| MVKCmdDebug.h/mm | debug label / begin-end debug utils |

## 4. MVKCommandBuffer 录制模型

- **录制**：`vkCmdXxx` 入口（vulkan.mm）→ 池取对象 → `setContent()` → `addCommand()` 追加链表；`_commandCount` 递增。
- **状态机**：`begin()` 校验 one-time-submit/reset 标志并复位链表；`end()` 收尾（记录 timestamp 命令占位等）；`reset()` 释放全部命令回池（`releaseRecordedCommands()` 逆序回收入池）。
- **Immediate 模式**：`MVK_CONFIG_PREFILL_METAL_COMMAND_BUFFERS` 启用 Metal 预填充时，命令录制即时同步到 Metal command buffer（`flushImmediateCmdEncoder()`），`clearPrefilledMTLCommandBuffer()` 处理 reset 语义。
- **Secondary**：`beginSecondaryEncoding()`/`recordExecuteCommands()` 支持二级命令缓冲在 primary 提交期展开。
- **提交**：`submit(MVKQueueCommandBufferSubmission*)` 触发 `MVKCommandEncoder::encode()` 遍历链表；提交完成后命令对象全部回池。
- **辅助结构**：`MVKCommandEncodingContext`（跨命令携带：当前 renderpass、`BarrierFenceSlots` 跨编码器 fence 槽位）、`MVKCurrentSubpassInfo`、`GPUCounterQuery`。

## 5. MVKCommandEncoder 编码模型

### 5.1 编码器生命周期

- `beginEncoding(mtlCmdBuff, ctx)` → 逐条 `encodeCommands(cmd)` → `endEncoding()`。
- **Metal 编码器切换**：`getMTLRenderEncoder()/getMetalComputeEncoding()/getMTLBlitEncoder()` 按需结束当前编码器再开新编码器（Metal 不允许跨域同时编码）。
- **Render pass 管理**：`beginRenderpass/beginNextSubpass/beginRendering/beginMetalRenderPass`；`restartMetalRenderPassIfNeeded()` 在 attachment store action 或输入附件语义需要时重启 Metal render pass（Vulkan subpass 链 → 多个 Metal render pass 的映射）；multiview 用 `_multiviewPassIndex` 多 pass 展开。
- **绘制前定妆**：`finalizeDrawState(stage)` / `finalizeDispatchState()` 在 draw/dispatch 前把所有脏状态（管线、资源绑定、顶点偏移、视口剪裁）写入 Metal 编码器。`MVKGraphicsStage` 区分 vertex/compute 转换阶段（tessellation、indirect 参数生成需要中间 compute pass）。

### 5.2 状态跟踪（MVKCommandEncoderState）

- `MVKCommandEncoderState` 基类：`markDirty()/markClean()` 脏标记；`encode()` 时仅将脏状态写入 Metal。
- 子类分工：
  - `MVKRenderPassCommandEncoderState`：attachments store/load action、render area；
  - `MVKGraphicsCommandEncoderState` / `MVKComputeCommandEncoderState`：管线绑定；
  - `MVKResourcesCommandEncoderState` → graphics/compute 两个子类：descriptor 绑定与 use-resource 计数（配合 Metal heap/argument buffer）；
  - 顶点/片段/depth-stencil/tessellation 细分状态类。
- `MVKStateTracking.h` 提供底层脏位工具。目标：重复提交同一命令缓冲时，Metal 侧状态调用次数最优。

### 5.3 辅助资源

- `MVKCommandResourceFactory`：为清屏、查询、blit 辅助、水印等内部操作创建/缓存 `MTLRenderPipelineState`、`MTLComputePipelineState`、临时 buffer/texture（着色器源码内嵌于 `MVKCommandPipelineStateFactoryShaderSource.h`，设备初始化时一次性编译）。
- `MVKCommandEncodingPool`：上述资源的运行期缓存容器，按用途 key 查找，避免重复编译。
- `MVKMTLBufferAllocation` + `MVKMTLBufferAllocationPool`：临时设备可见内存的环形分配（查询结果、indirect 参数、可见性缓冲等），从 `MTLBuffer` 切片复用。

## 6. MVKQueue 提交路径（与 GPUObjects 协作）

```
vkQueueSubmit ──► vulkan.mm ──► MVKQueue::submit(pSubmits, fence)
                                   │ 构造 MVKQueueCommandBufferSubmission
                                   │ （打包 wait/semaphore、cmdBuffs、fence、perf 计时）
                                   ▼
                          同步：本线程执行 / 异步：GCD _execQueue
                                   ▼
                     对每个 MVKCommandBuffer::submit() → Encoder 编码
                                   ▼
                     [mtlCmdBuff commit] → signal 信号量 / fence
```

- wait 信号量在编码前处理：支持命令编码型（MTLEvent，在 mtlCmdBuff 上 encodeWait）与 CPU 型（阻塞提交线程）两类（见 module_sync_wsi.md）。
- 呈现提交 `MVKQueuePresentSurfaceSubmission` 走同一执行队列，保证与图形提交的顺序（见 module_sync_wsi.md）。
- `MVK_CONFIG_SYNCHRONOUS_QUEUE_SUBMITS=1`（默认）时编码在本线程完成，降低 GPU 饥饿；关闭则完全异步。

## 7. 与其他模块的接口

- 依赖 GPUObjects：MVKQueue/MVKDevice（宿主）、MVKPipeline（绑定）、MVKRenderPass/MVKFramebuffer（render pass 语义）、MVKBuffer/MVKImage（transfer 目标）、MVKQueryPool（查询）。
- 资源绑定细节（descriptor → Metal argument buffer 的写入）由 `MVKResourcesCommandEncoderState` 与 MVKDescriptorSet 协作（见 module_resources_memory.md）。
- 屏障解析结果（`MVKPipelineBarrier`）的同步语义落地（blit/fence 槽位）见 module_sync_wsi.md。
