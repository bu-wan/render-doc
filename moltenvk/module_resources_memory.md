# 模块设计：资源与内存（Buffer/Image/DeviceMemory/Descriptor）

> 本文是 [moltenvk_design.md](moltenvk_design.md) 的模块级细化文档，聚焦 Vulkan 资源/内存模型到 Metal 的映射与描述符体系。总体架构见主文档。

---

## 1. 模块职责

实现 Vulkan 的"内存分配 + 资源绑定"模型与 Metal"直接创建资源"模型的桥接，以及 Vulkan 描述符集到 Metal argument buffer 的转换。

## 2. 内存模型：MVKDeviceMemory

### 2.1 核心设计

Vulkan：先 `vkAllocateMemory` 得到 DeviceMemory，再 `vkBindBufferMemory/vkBindImageMemory` 绑定资源。
Metal：资源对象（MTLBuffer/MTLTexture）自带存储。

**MoltenVK 方案：MVKDeviceMemory 持有真正的 Metal 资源，资源对象是视图。**

- `MVKDeviceMemory` 内部持有 `id<MTLBuffer> _mtlBuffer`（绝大多数情况）或 `id<MTLTexture> _mtlTexture`（memoryless/linear tiling image 特例）。
- 映射（`map()`）直接返回 `_mtlBuffer.contents`；host-coherent 写入即时可见，非 coherent 内存记录 `_mappedRanges`，在 `vkFlushMappedMemoryRanges/vkInvalidateMappedMemoryRanges` 时对 Managed 存储调用 `didModifyRange:`（MVKDeviceMemory.mm:89）同步 CPU↔GPU。
- 隐式 Integer 内存类型：`isDedicated`（`VK_KHR_dedicated_allocation`）与 `externalMemoryHandleTypes` 支持外部 MTLBuffer/IOSurface 导入导出。

### 2.2 存储模式选择

| Vulkan 用途 | Metal 存储模式 |
|---|---|
| HOST_VISIBLE（macOS） | `MTLStorageModeManaged`（非 coherent）或 `Shared`（coherent） |
| HOST_VISIBLE（iOS 系） | `MTLStorageModeShared` |
| DEVICE_LOCAL | `MTLStorageModePrivate` |
| memoryless attachment | `MTLStorageModeMemoryless` |

### 2.3 MVKResource 基类

```cpp
class MVKResource : public MVKVulkanAPIDeviceObject {
    VkResult bindDeviceMemory(MVKDeviceMemory* mvkMem, VkDeviceSize memOffset);
    virtual void applyMemoryBarrier(MVKPipelineBarrier&, MVKCommandEncoder*, MVKCommandUse) = 0;
protected:
    MVKDeviceMemory* _deviceMemory;  VkDeviceSize _deviceMemoryOffset, _byteCount, _byteAlignment;
};
```

- bind 后资源通过 `_deviceMemory->getHostMemoryAddress() + offset` 访问主机内存；
- `applyMemoryBarrier()` 由 Commands 层屏障解析调用，对 host-visible 资源触发 Managed 同步或 blit 编码。

## 3. Buffer 体系（MVKBuffer / MVKBufferView）

- **MVKBuffer : MVKResource**：bind 前惰性创建 `MTLBuffer`（两种模式：`vkCreateBuffer` 即创建并可选复用 `_deviceMemory` 的 buffer，或 bind 时在目标 memory 的 MTLBuffer 上做零拷贝切片）。
- 支持 `VK_KHR_buffer_device_address`（`getMTLBufferID`/GPU 地址）、外部内存（`vkGetMemoryMTLBufferMVK` 桥接）。
- **MVKBufferView**：把 buffer 区间解释为格式化纹理（`MTLTextureTypeTextureBuffer`），依赖设备支持（macOS 10.13+/iOS 11+），用于 typed load/store。

## 4. Image 体系（MVKImage / MVKImageView）

### 4.1 MVKImage

- 不继承 MVKResource（image 需要多平面/多视图的独立管理），直接绑定 `MVKDeviceMemory`。
- **子平面机制**：depth/stencil 或多平面 YCbCr 格式在 Metal 中是多个 MTLPixelFormat 不同的纹理 → `MVKImagePlane` 数组，每 plane 一个 `id<MTLTexture>`（可以是 alias view）。
- **创建路径**：
  - 普通路径：按 `VkImageCreateInfo` 配置 `MTLTextureDescriptor`（类型/格式/usage/存储模式），从 `MTLHeap`（`_device->getMTLDevice().newHeap`）或直接 `newTextureWithDescriptor` 分配；
  - **IOSurface 路径**（macOS/可共享）：`useIOSurface(IOSurfaceRef)` 把已有 IOSurface 包装成 MTLTexture，支撑 `VK_EXT_external_memory` 与 swapchain 共享；
  - **导入路径**：`vkSetMTLTextureMVK` 直接注入外部 MTLTexture。
- **布局**：Vulkan 的 image layout（General/TransferSrc/ColorAttachment/... + present）由 MVKImagePlane 记录，布局转换在 Commands 层屏障处理中通过 blit encoder 或 usage 追踪落实。
- 交换链图像：`MVKPresentableSwapchainImage : MVKImage`（见 module_sync_wsi.md）。

### 4.2 MVKImageView

- 一个 MVKImageView 可含多个 `MVKImageViewPlane`（如 combined depth-stencil 视图拆分）。
- 视图参数（类型/格式/swizzle/component mapping）映射为 `MTLTextureViewUsage`；swizzle 中 Metal 不支持的分量交换通过**着色器端 swizzle 常量**（SPIRV-Cross 转换配置传入）补偿。
- 输入附件视图（input attachment）与 framebuffer 兼容性规则由 `MVKImageView::getSampleCount/compatibleAsLinearOrSRGB` 等辅助判定。

## 5. 描述符体系（MVKDescriptorSet 系列）

### 5.1 总体方案：Metal Argument Buffer

Metal 无"描述符集合"原生概念（除 argument buffer）。MoltenVK 把 descriptor set 落到 **argument buffer**（放在 MTLBuffer 中的资源指针表），shader 端由 SPIRV-Cross 把 set/binding 翻译为 `[[buffer(n)], [[texture(n)]]` 或 argument buffer 访问。

### 5.2 类分工

- **MVKDescriptorSetLayout**：解析 `VkDescriptorSetLayoutCreateInfo`，为每个 binding 计算 Metal 索引（buffer/texture/sampler 各自的槽位空间）与 argument buffer 偏移；持有 `MVKDescriptorSetLayoutBinding` 数组；可变描述符数（`VkDescriptorBindingFlags` 的 `VARIABLE_DESCRIPTOR_COUNT`）支持运行时数量。
- **MVKDescriptorPool**：从 `MTLHeap`/MTLBuffer 分配 argument buffer 存储与描述符内存；`MVKDescriptorPoolFreeList` 管理已释放集合的复用；支持 `VkDescriptorPoolInlineUniformBlockCreateInfo`（inline uniform block 特殊处理为普通 buffer 区间）。
- **MVKDescriptor**（抽象）→ 每类描述符子类（buffer/image/sampler/acceleration structure/texel buffer view）：`write()/read()` 直接编码为 Metal 资源指针写入 argument buffer；动态偏移在绑定/编码期叠加。
- **MVKDescriptorUpdateTemplate**：把 `VkDescriptorUpdateTemplate` 的更新路径编译为"位置+步长"描述，`vkUpdateDescriptorSetWithTemplate` 按模板批量 memcpy/写指针，避免逐条解释。
- **MVKPipelineLayout**：聚合 set layouts + push constant 范围；push constant 写入专用小 MTLBuffer（每编码器缓存），转换配置把 push constant block 映射为固定 buffer 槽位。
- **绑定路径**：`vkCmdBindDescriptorSets` → `MVKResourcesCommandEncoderState`（Commands 层）记录脏绑定 → `finalizeDrawState` 时对 Metal 编码器 `setBuffer(setBufferOffset:)`/`setTexture:` 或整体绑定 argument buffer。

### 5.3 Metal 特有约束的适配

| 约束 | 适配 |
|---|---|
| Metal 槽位数上限（31 buffer/128 texture 等） | 超限时自动把部分绑定折叠进 argument buffer（`VK_EXT_descriptor_indexing` 的 runtime array 必走 argument buffer） |
| sampler 与 texture 分离 | `MTLSamplerState` 单独槽位或 argument buffer 内联 |
| `descriptor_buffer`（VK_EXT_descriptor_buffer） | 以裸 MTLBuffer 形式暴露 argument buffer，绕过传统 set 绑定 |

## 6. 与其他模块的接口

- Commands 层：屏障落地（`applyMemoryBarrier`）、资源绑定状态（`MVKResourcesCommandEncoderState`）、clear/copy 操作直接操作本模块对象。
- Shader 层：SPIRV-Cross 的 `MSLResourceBinding` 由 MVKPipelineLayout/MVKDescriptorSetLayout 生成（见 module_shaders_pipeline.md）。
- Sync 层：image layout 状态被屏障/子pass 依赖查询。
- 入口层：`vkUseIOSurfaceMVK`/`vkGetMTLTextureMVK` 等互操作 API 落到本模块方法。
