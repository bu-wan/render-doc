# 模块设计：着色器与管线（MoltenVKShaderConverter + MVKShaderModule/MVKPipeline）

> 本文是 [moltenvk_design.md](moltenvk_design.md) 的模块级细化文档，聚焦 SPIR-V → MSL 转换器内部结构、MTLLibrary 编译与缓存、管线创建流程。总体架构见主文档。

---

## 1. 模块组成

```
MoltenVKShaderConverter/            （独立库，运行库以源码/子项目方式嵌入）
├── SPIRVToMSLConverter.{h,cpp}     转换器主类
├── SPIRVConversion.h               配置/结果数据结构
├── SPIRVReflection.h               SPIR-V 反射辅助
├── SPIRVSupport.{h,cpp}            SPIR-V 解析/校验
└── MoltenVKShaderConverterTool/    独立命令行工具（离线转换）

MoltenVK/MoltenVK/GPUObjects/
├── MVKShaderModule.{h,mm}          ShaderModule + MVKShaderLibrary + 缓存
├── MVKPipeline.{h,mm}              Pipeline/Layout/Cache + 编译器
└── MVKMetalCompiler.mm             MTLLibrary 编译基类（在 Utility）
```

## 2. 转换数据结构（SPIRVConversion.h）

| 结构 | 字段要点 |
|---|---|
| `SPIRVToMSLConversionOptions` | mslVersion、entryPointStage/Name、texelBufferTextureWidth、`shouldFlipVertexY`（默认 true，`MVK_CONFIG_SHADER_CONVERSION_FLIP_VERTEX_Y`）、tessellation/原点（upper-left）等固定管线状态标志 |
| `MSLResourceBinding` | `descriptorSet/binding` → `mtlBufferType/mtlBufferIndex/mtlTextureIndex/mtlSamplerIndex` + descriptor 类型（决定 MSL 资源声明） |
| `MSLShaderInterfaceVariable` | 顶点/片段 stage、location → Metal attribute/color attachment ID |
| `SPIRVToMSLConversionConfiguration` | options + resourceBindings + interfaceVariables + 特化宏 + dynamicBufferDescriptors 等，整体作为**缓存 key** |
| `SPIRVToMSLConversionResult` | msl 源码 + `SPIRVToMSLConversionResultInfo`（entry point、workgroup size、使用的资源绑定、非零 swizzle 标记等反射信息） |

`operator==` 定义在 configuration 上，保证同一配置命中缓存；任何管线状态差异（如 swizzle、格式宽度）都会改变 key。

## 3. SPIRVToMSLConverter 转换流程

```
convert(shaderConfig)                     SPIRVToMSLConverter.cpp:260
  1. validateSPIRV()                      模块头/魔数/版本校验（:483）
  2. 检测 physical storage buffer 地址模型（usesPhysicalStorageBufferAddressesCapability）
  3. 构造 SPIRV-Cross CompilerMSL：
       - 设置 MSL 版本、entry point、固定状态
       - 逐条应用 resourceBindings → add_msl_resource_binding
       - 应用 interfaceVariables → set_msl_shader_interface_var
       - 应用特化常量（可特化为 Metal function constant 或宏）
  4. compile() → 生成 MSL 源码
  5. 收集反射信息（entry point 名、workgroup size、每绑定实际使用情况）回填 resultInfo
  6. 失败时 logError/logWarning 累积转换日志
```

- **双库联动**：SPIRV-Cross（编译核心）与 SPIRV-Tools（可选优化/校验）均为 git submodule，版本由 `ExternalRevisions` 锁定。
- **反射闭环**：转换前的 resourceBindings 由管线创建时 MVKPipelineLayout/MVKDescriptorSetLayout 生成；转换后的 resultInfo（如某些绑定实际未用）反馈给管线绑定逻辑。

## 4. MVKShaderModule 与三层缓存

### 4.1 MVKShaderLibrary（MVKShaderModule.h:90）

- 包装一个 `id<MTLLibrary>`；持有压缩后的 MSL 源码（`MVKCompressor<string>`，降低缓存内存占用）与转换结果信息。
- `compileLibrary(msl, macroDefs)` 经 `MVKMetalCompiler::newMTLLibrary` 编译（见 §6）。
- **宏特化变体**：Vulkan 特化常量若无法映射为 Metal function constant，则转为 MSL 宏 → 需要重新编译源码。`_specializationVariants: map<宏值对, MVKShaderLibrary*>` 惰性编译并缓存各特化变体（`getMTLFunction` 内查/建，MVKShaderModule.mm:80-107）。
- `MVKMTLFunction`：MTLFunction + threadgroup size（SPIR-V workgroup size 反射所得）。

### 4.2 MVKShaderLibraryCache（:169）

- **每个 shader module 一个**：`getShaderLibrary(pShaderConfig, ...)` 按 conversion configuration 查找（同配置不同管线的多 pass/tessellation 变体分别缓存），未命中则调用转换器惰性转换+编译。
- `merge()` 供 pipeline cache 恢复时合并磁盘态。
- 支持管线创建反馈（`VkPipelineCreationFeedback`）与 `MVK_CONFIG_SHADER_CONVERSION` 相关的"管线编译必须命中缓存"模式（`pWasAdded=false` 严格模式）。

### 4.3 MVKPipelineCache（MVKPipeline.h:471）

- **跨模块、可持久化**：`writeData/readData` 用 cereal 二进制序列化（MVKPipeline.mm:36-39 引入 cereal 头，:2582 捕获 cereal::Exception）。
- 序列化内容：SPIR-V、conversion configuration、转换结果（MSL）、反射信息；恢复时重建 `MVKShaderLibraryCache` 条目。
- `MVKPipelineAuxCache`（辅助对象缓存）记录各管线的 compilation feedback 汇总。

## 5. MVKPipeline 创建流程

### 5.1 MVKPipelineLayout

- 聚合 `MVKDescriptorSetLayout` 数组 + push constant 范围；
- 为**每个 shader stage** 计算 set/binding → Metal 槽位的完整映射表（即转换所需的 `MSLResourceBinding` 集合）；
- push constant block 映射到保留的固定 buffer 槽位。

### 5.2 MVKGraphicsPipeline（:240）

```
vkCreateGraphicsPipelines
  1. 逐 shader stage 取 SPIR-V（VkShaderModule），组装 conversion configuration
     （layout 映射 + render pass 状态：采样数、attachments、top-left 原点、tessellation 分区等）
  2. MVKShaderLibraryCache::getShaderLibrary() —— 命中缓存或转换+编译
  3. 用 MTLFunction 组装 MTLRenderPipelineDescriptor：
       - vertex/fragment 函数、rasterization/blending/vertex descriptors
       - 反射接口变量 → MTLVertexDescriptor attribute/step function
  4. newRenderPipelineState（经 MVKMetalCompiler，支持异步编译 KHR_pipeline_creation_feedback/deferred）
  5. 失败回退：反射校验不符时记录详细错误
```

- **多 pass 展开**：tessellation 管线拆 vertex+compute(tess) 两阶段；multiview 按层展开；这些状态进入 conversion configuration，产生不同的 shader library 变体。
- indirect draw：indirect 命令的参数补齐用内置 compute kernel（Commands 层）。

### 5.3 MVKComputePipeline（:423）

- 单 stage：转换（含 entry point/workgroup size 反射）→ `MTLComputePipelineDescriptor` → `newComputePipelineState`，`threadgroup size` 结合 Metal 最大线程组宽度裁剪。

## 6. MVKMetalCompiler（编译基础设施）

- 所有 `MTLLibrary/MTLPipelineState/MTLSamplerState` 创建经此类：集中错误处理（`compileLibrary` 失败转 `MVKShaderLibrary::handleCompilationError`）、编译耗时统计（performance statistics 中的 shader compile 项）、`MVK_CONFIG_DEBUG` 下串行编译。
- 支持**异步管线编译**：`VK_KHR_deferred_host_operations` + pipeline creation feedback 时，编译工作可交给 MVKDeferredOperation 线程池（module_sync_wsi.md §4）。

## 7. 离线工具（MoltenVKShaderConverterTool）

- 复用同一转换库：`mssl` 命令批量把 `.spv` 转 `.metal`，支持从 JSON/GPA 反射文件注入绑定配置；`spirv` 命令做 SPIR-V 校验。
- 用途：开发期离线检视 MSL 输出、构建预转换着色器资产。

## 8. 与其他模块的接口

- Resources 层：`MSLResourceBinding` 由 MVKPipelineLayout/MVKDescriptorSetLayout 生成（module_resources_memory.md §5.2）。
- Commands 层：管线绑定命令与 `finalizeDrawState` 使用本模块产出的 MTLRender/ComputePipelineState。
- Sync 层：管线编译可挂到 deferred operation（module_sync_wsi.md §4）。
- API 层：`vkSetWorkgroupSizeMVK` 等私有桥接直接调用 MVKShaderModule 方法。
