# 04 · MyVK 封装层详解

## 1. 定位

`dep/MyVK` 提供两个子库:

- **`myvk::vulkan`(MyVK_Vulkan)**:对 Vulkan 对象的 RAII 封装(约 40 个类),基于 volk(动态加载)与 VMA(内存分配);
- **`myvk::rg`(MyVK_RenderGraph)**:渲染图框架(见 03 文档);
- **`myvk::glfw` / `myvk::imgui`**:GLFW 窗口/交换链与 ImGui 集成的可选扩展(通过编译宏 `MYVK_ENABLE_GLFW` / `MYVK_ENABLE_IMGUI` 开关)。

本文只讲 `myvk::vulkan` 部分的关键类。整体设计原则:

1. **RAII + 引用计数**:所有对象持 `myvk::Ptr<T>`(内部 `std::shared_ptr`),析构自动释放 Vulkan 资源;
2. **延迟创建**:`Create` 静态工厂执行完整初始化,失败返回空 Ptr;
3. **薄封装**:不隐藏 Vulkan 句柄(`GetHandle()`),不重造语义,只把"生命周期 + 常用操作组合"收拢;
4. **DeviceObjectBase 分层**:凡绑定设备资源的对象都继承 `DeviceObjectBase`(持有 `Ptr<Device>`),便于统一校验与追踪(`ObjectTracker`)。

## 2. 对象追踪与调试

`ObjectTracker` 是全局注册表(单例):

- 所有 `DeviceObjectBase` 在构造/析构时登记自身;
- 提供 `GetObjects()` 等查询,可枚举当前存活的 Vulkan 对象——用于调试资源泄漏与驱动崩溃(常见手法:崩溃前打印所有对象类型/句柄)。

## 3. Device 与 Instance

### 3.1 Instance

```cpp
class Instance : DeviceObjectBase {
    static Ptr<Instance> Create(VkInstanceCreateInfo, ...);
    static Ptr<Instance> CreateWithGlfwExtensions(); // GLFW 窗口必需扩展(若启用 glfw 层)
};
```

持有 `VkInstance` 与 volk 加载的 Vulkan API 函数指针。`CreateWithGlfwExtensions` 收集 glfw 要求的所有实例扩展(swapchain、surface 等)并创建。

### 3.2 PhysicalDevice

```cpp
class PhysicalDevice {
    struct Properties { VkPhysicalDeviceProperties vk10; 
                        VkPhysicalDeviceVulkan11Properties vk11;
                        VkPhysicalDeviceVulkan12Properties vk12;
                        VkPhysicalDeviceVulkan13Properties vk13; };
    static std::vector<Ptr<PhysicalDevice>> Fetch(instance); // 枚举所有设备
    VkPhysicalDeviceFeatures GetDefaultFeatures() const;      // 初始化 features 结构(含 1.1/1.2/1.3 链)
    bool GetExtensionSupport(const char *ext) const;
};
```

**设计细节**:`GetDefaultFeatures()` 把 `VkPhysicalDeviceVulkan11/12/13Features` 结构按 pNext 链串好并初始化为 0,应用只需改需要开启的位(见 main.cpp)。这避免每个应用手工堆叠 pNext 链的样板代码。

### 3.3 Device

```cpp
class Device : DeviceObjectBase {
    static Ptr<Device> Create(physical_device, QueueSelector, features, extensions);
    VmaAllocator GetAllocatorHandle() const;                    // 普通分配器
    VmaAllocator GetDeviceAddressAllocatorHandle() const;       // 带 DEVICE_ADDRESS 的分配器
    VkDevice GetHandle() const;
};
```

持有 `VkDevice` + 两个 VMA 分配器(是否启用 `VMA_ALLOCATOR_CREATE_BUFFER_DEVICE_ADDRESS_BIT` 分开,避免所有分配都承担设备地址开销)。`Device::Create` 内部:
- 创建队列(按 `QueueSelector` 提供的族索引);
- 用 volk 加载设备级函数指针(`volkLoadDevice`)。

## 4. 队列与命令

### 4.1 Queue / QueueSelector

```cpp
class Queue { // 通用队列:图形/计算/传输
    VkQueue GetHandle() const;
    const Ptr<Device> &GetDevicePtr() const;
    void Submit(command_buffers, fence);  // 封装 vkQueueSubmit
    void WaitIdle() const;
};
class PresentQueue : public Queue { void Present(swapchain, image_index, wait_semaphores) const; };
```

`QueueSelector`(如 `GenericPresentQueueSelector`)是一个回调式工厂:给定 PhysicalDevice,选出"支持图形 + 呈现"的队列族,创建图形队列与呈现队列(可能同一族)。

### 4.2 CommandPool / CommandBuffer

```cpp
class CommandPool : DeviceObjectBase {
    static Ptr<CommandPool> Create(queue);
    Ptr<CommandBuffer> Allocate(VkCommandBufferLevel level) const;
};
class CommandBuffer {
    void Begin(VkCommandBufferUsageFlags);
    void End();
    void Submit(const Ptr<Fence> &fence) const;      // 提交到所属队列
    void Submit(Ptr<Semaphore> wait, Ptr<Semaphore> signal, Ptr<Fence>) const; // 带信号量
    // 大量便捷封装:绑定、draw、dispatch、push constant、拷贝、屏障、管线屏障……
    VkCommandBuffer GetHandle() const;
};
```

设计上,`CommandBuffer::Submit` 直接关联命令池所属队列,避免每次提交都传队列。还提供 `CmdPipelineBarrier`(vkCmdPipelineBarrier2)、`CmdCopy`、`CmdDispatchIndirect` 等高频操作的封装。

## 5. 资源:Buffer / Image / ImageView / Sampler

### 5.1 Buffer(基于 VMA)

```cpp
class Buffer : BufferBase {
    static Ptr<Buffer> Create(device, size, allocation_flags, usage,
                              memory_usage = VMA_MEMORY_USAGE_AUTO, access_queues = {});
    template<typename Iter> static Ptr<Buffer> CreateStaging(device, begin, end); // 一次性上传
    void *GetMappedData() const;             // 若分配时带 MAPPED 位,直接返回映射指针
    VkDeviceAddress GetDeviceAddress() const;
    template<typename T> void UpdateData(data, byte_offset); // 拷入映射区
};
```

要点:
- **映射内存**:`VMA_ALLOCATION_CREATE_MAPPED_BIT` 在创建时即映射,`GetMappedData()` 零成本访问,被渲染图框架的 `SetMapped(true)` 使用;
- **设备地址**:`GetDeviceAddress` 缓存 VkDeviceAddress,供加速结构/着色器使用;带设备地址的缓冲需由 `DeviceAddressAllocator` 分配(见 3.3);
- `CreateStaging` 模板自动按元素类型算字节数,并直接 `UpdateData` 填数据——上传样板代码收进一个函数。

### 5.2 Image / ImageView

```cpp
class ImageBase : DeviceObjectBase {
    VkImage GetHandle() const;  VkFormat GetFormat() const; VkExtent2D GetExtent() const;
    VkBufferImageCopy GetBufferImageCopy(...) const;
    VkImageMemoryBarrier2 GetMemoryBarrier(...) / GetDstMemoryBarriers(...); // 屏障便捷构造
};
class Image : ImageBase {
    static Ptr<Image> CreateTexture2D(device, extent, mip_levels, format, usage); // 常用快捷方式
};
class ImageView : DeviceObjectBase {
    static Ptr<ImageView> Create(image, view_type); // 默认 full subresource range
    VkImageSubresourceRange GetSubresourceRange() const;
};
```

**屏障便捷构造**是本库很实用的抽象:`GetDstMemoryBarriers(copy, src_access, dst_access, old_layout, new_layout)` 直接生成拷贝所需的 image barrier 数组,大量减少应用侧样板(见 VkScene 的贴图上传)。

### 5.3 Sampler

```cpp
class Sampler : DeviceObjectBase {
    static Ptr<Sampler> Create(device, mag_filter, address_mode, mipmap_mode, mip_lod_bias = 0);
};
```

用于路径追踪/推理 Pass 的贴图采样器(线性/最近邻各一个,由渲染图在 Pass 构造时创建)。

## 6. 管线相关

### 6.1 ShaderModule

```cpp
class ShaderModule : DeviceObjectBase {
    static Ptr<ShaderModule> Create(device, spv_words, size);   // 字节数组内嵌 SPIR-V
    void AddSpecialization(uint32_t constant_id, T value);      // 追加特化常量
    VkPipelineShaderStageCreateInfo GetPipelineShaderStageCreateInfo(stage) const;
};
```

本项目所有着色器都是**构建期编译为 SPIR-V 数字数组**再以 `#include <shader/xxx.comp.u32>` 内嵌进 C++(见 06 文档),`Create` 直接吃数组。特化常量用于纹理数量(默认 1024)与 subgroup 大小。

### 6.2 PipelineLayout / GraphicsPipeline / ComputePipeline

```cpp
class PipelineLayout {
    static Ptr<PipelineLayout> Create(device, descriptor_set_layouts, push_constant_ranges);
};
class PipelineBase : DeviceObjectBase {  // 管线基类
    VkPipeline GetHandle() const;
    const Ptr<PipelineLayout> &GetPipelineLayoutPtr() const;
};
class GraphicsPipelineState {  // 集中描述图形管线所有状态
    VertexInputState / InputAssemblyState / RasterizationState
    DepthStencilState / MultisampleState / ColorBlendState / ViewportState
    void Enable(...);  // 链式初始化
};
class GraphicsPipeline : PipelineBase {
    static Ptr<GraphicsPipeline> Create(pipeline_layout, render_pass, shader_stages, state, subpass);
};
class ComputePipeline : PipelineBase {
    static Ptr<ComputePipeline> Create(pipeline_layout, shader_module);
    static Ptr<ComputePipeline> Create(pipeline_layout, VkComputePipelineCreateInfo); // 允许塞 requiredSubgroupSize 等扩展
};
```

`ComputePipeline::Create(pipeline_layout, create_info)` 的重载让应用能直接传自定义 create info(带 `VkPipelineShaderStageRequiredSubgroupSizeCreateInfo` 的 pNext),满足"要求完整子组"的协作矩阵需求。

## 7. RenderPass / Framebuffer / 描述符

### 7.1 RenderPass 与 Framebuffer

```cpp
class RenderPass : DeviceObjectBase {
    static Ptr<RenderPass> Create(device, attachments, subpasses, dependencies);
};
class FramebufferBase / Framebuffer;         // 常规帧缓冲
class ImagelessFramebuffer;                  // 免图像帧缓冲(VK_KHR_imagination… 即 VK_KHR_imageless_framebuffer)
```

**ImagelessFramebuffer 是渲染图框架的核心配套**:交换链/内部图像每帧可能变化,若用常规 Framebuffer,每帧都要重建;ImagelessFramebuffer 在创建时只绑定 RenderPass 与尺寸,实际附件图像在 `VkRenderPassAttachmentBeginInfo` 里每次命令录制时指定(见 VkRunner::Run 的 `attachment_begin_info`)。这正好配合框架"每帧换绑定外部资源"的模型。

### 7.2 DescriptorSetLayout / DescriptorPool / DescriptorSet

```cpp
class DescriptorSetLayout { static Ptr<Create>(device, bindings); };
class DescriptorPool  { static Ptr<Create>(device, sizes); Allocate(layout, count); };
class DescriptorSet {
    static Ptr<Create>(pool, layout);
    void UpdateImage(binding, array_element, view, sampler, descriptor_type);
    void UpdateBuffer(binding, array_element, buffer, offset, size, descriptor_type);
    void UpdateAccelerationStructure(binding, ...);   // 加速结构描述符
    void UpdateBufferAddress(...);                    // 设备地址描述符
    VkDescriptorSet GetHandle() const;
};
```

渲染图框架据此生成 Pass 的描述符集;外部资源每帧通过 `VkDescriptor::VkUpdateExternal` 更新。

## 8. 同步对象

```cpp
class Fence / Semaphore / Event { /* 薄封装 + GetHandle */ };
```

`Fence` 提供 `Wait()/Reset()`,`Semaphore` 用于交换链信号量,`Event` 用于命令缓冲内事件(框架未用,保留)。

## 9. 交换链与帧管理(FrameManager)

### 9.1 FrameManager 职责

```
FrameManager(device, graphics_queue, present_queue, vsync, frame_count=3)
├─ 创建 Swapchain(颜色格式/呈现模式/图像数 = frame_count)
├─ 创建 swapchain 图像 ImageView
├─ 每帧:
│    NewFrame():  acquire next image(信号量同步)
│                 Wait 该帧 fence(CPU 侧同步 in-flight)
│                 返回当前帧命令缓冲(每帧独立分配)
│    Render():    提交 + 呈现 + 信号量链
├─ 处理 resize:   Resize() → 重建 swapchain + 回调 m_resize_func(extent)
└─ WaitIdle()
```

### 9.2 三重缓冲同步(关键实现)

```cpp
std::vector<Ptr<Fence>>      m_frame_fences;          // 每帧一个 CPU fence
std::vector<Ptr<Semaphore>>  m_acquire_done_semaphores; // 图像获取完成
std::vector<Ptr<Semaphore>>  m_render_done_semaphores;  // 渲染完成(呈现等待)
std::vector<Ptr<CommandBuffer>> m_frame_command_buffers; // 每帧预分配的命令缓冲
```

- 帧 N 的命令缓冲在 CPU 端等待帧 N-3(帧数 3)的 fence 释放后才允许录制提交,避免 GPU 落后过多;
- 呈现等待 `render_done` 信号量,图像获取等待 `acquire_done`,形成标准三重缓冲流水线;
- `NewFrame()` 内部还处理交换链失效(VK_ERROR_OUT_OF_DATE_KHR)→ 触发 `recreate_swapchain()` 与 resize 回调。

### 9.3 与渲染图的衔接

- `SwapchainImage` 资源(渲染图侧)每帧 `GetCurrentSwapchainImageView()` 指向当前呈现图像;
- `SetResizeFunc` 回调里应用重置累积图像与计数(main.cpp 中传入 lambda)。

## 10. ImGui 集成

`ImGuiHelper` / `ImGuiRenderer`(启用 `MYVK_ENABLE_IMGUI`):

- `ImGuiInit(window, command_pool)` 初始化 ImGui 上下文与字体纹理上传;
- `ImGuiNewFrame()` 每帧调用;
- `myvk_rg::ImGuiPass` 在渲染图内渲染 ImGui 绘制数据(单全屏三角形 + 字体图集),输出到交换链。

## 11. 生命周期与线程安全

- 所有对象通过 `myvk::Ptr`(shared_ptr)管理,引用计数保证"最后一处释放时销毁 Vulkan 对象",避免悬垂;
- 销毁顺序依赖 shared_ptr 的析构顺序:子对象(如 CommandBuffer 引用 CommandPool、ImageView 引用 Image)内部持有父 Ptr,天然保证"先销毁子,后销毁父";
- 多线程加载贴图时,每个线程独立创建 CommandPool,避免 CommandPool 跨线程使用(Vulkan 命令池非线程安全)。
