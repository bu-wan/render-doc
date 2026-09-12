# 模块设计：Vulkan API 入口层（Vulkan/ · Layers/ · API/ · Common/）

> 本文是 [moltenvk_design.md](moltenvk_design.md) 的模块级细化文档，聚焦 API 适配层的详细设计。总体架构见主文档。

---

## 1. 模块职责

- **Vulkan/**：导出全部 Vulkan C ABI 符号；把 C 入口调用分派到 MVK C++ 对象方法。
- **Layers/**：Layer 与扩展的注册、枚举、启用状态管理。
- **API/**：对应用暴露的公共头文件（配置、数据类型、私有/弃用 API 声明）。
- **Common/**：运行库与着色器转换器共享的平台代码。

---

## 2. Vulkan/ 详细设计

### 2.1 vulkan.mm —— C ABI 入口（约 4500 行）

每个 Vulkan 函数遵循统一模板：

```cpp
MVK_PUBLIC_VULKAN_SYMBOL VkResult vkCreateInstance(...) {
    MVKTraceVulkanCallStart();                    // 1. 可选调用追踪
    MVKInstance* mvkInst = new MVKInstance(pCreateInfo);   // 2. 句柄→对象
    *pInstance = mvkInst->getVkInstance();
    VkResult rslt = mvkInst->getConfigurationResult();
    ...
    MVKTraceVulkanCallEnd();                      // 3. 可选耗时统计
}
```

设计要点：

1. **句柄即对象指针**：Vulkan 的 dispatchable handle（VkInstance/VkDevice/VkQueue 等）首个成员指向 dispatch 表，MoltenVK 让 handle 直接就是 `MVKxxx*` 指针，`getDispatchableObject()` 反解后零开销访问对象。非 dispatchable 对象（VkBuffer 等）同理直接映射。
2. **调用追踪**：`MVKTraceVulkanCallStart/End` 宏受 `MVK_CONFIG_TRACE_VULKAN_CALLS` 控制，支持 6 档（无 / 进入 / 进出 / 耗时 / 带线程 ID 变体），写入 stderr。
3. **two-call 惯例**：所有 `vkEnumerate*/vkGet*` 查询函数统一实现 `pCount==NULL → 返回数量` 与 `pCount 指定 → 填充+VK_INCOMPLETE` 两段式语义。
4. **入口静态表**：`initProcAddrs()` 在 `MVKInstance` 构造时把每个 `vkXxx` 函数名与函数指针、所需 core 版本/扩展注册进 `_entryPoints` 哈希表（`MVKEntryPoint` 结构，支持"core 版本 OR 扩展 OR 第二扩展"的复杂启用判定）。`vkGetInstanceProcAddr`/`vkGetDeviceProcAddr` 查该表；设备级函数再委托 `MVKDevice::getProcAddr()`。
5. **命令创建模板**：`vkCmdXxx` 入口从命令池取命令对象 → `setContent()` → `cmdBuff->addCommand()`，由 `vulkan.mm` 顶部的通用模板宏生成，避免重复样板。

### 2.2 mvk_api.mm —— MoltenVK 私有 API（183 行）

`mvk_private_api.h` 声明的扩展入口，全部实现于此：

| 函数 | 作用 |
|---|---|
| `vkGet/SetMoltenVKConfigurationMVK` | 读/写运行时全局配置（`MVKConfiguration`，已标记 deprecated，推荐 `VK_EXT_layer_settings` 或环境变量） |
| `vkGetPhysicalDeviceMetalFeaturesMVK` | 暴露 `MVKPhysicalDeviceMetalFeatures`（Metal 能力快照） |
| `vkGetPerformanceStatisticsMVK` | 读取 `MVKPerformanceStatistics`（各阶段耗时/内存计数） |

`mvkCopyGrowingStruct<S>()` 模板处理**跨版本结构体增长**：按 `min(*pCopySize, sizeof(S))` 拷贝，长度不符返回 `VK_INCOMPLETE`，保证新旧版本二进制兼容。

`mvk_deprecated_api.h` 的桥接 API（`vkGetMTLDeviceMVK`、`vkSet/GetMTLTextureMVK`、`vkUseIOSurfaceMVK` 等）在此直接转发到对应 MVK 对象方法，实现 Metal 对象互操作（IOSurface 共享、外部 Metal 纹理导入等）。

### 2.3 mvk_datatypes.{h,hpp,mm} —— 数据类型转换（约 1200 行）

- `API/mvk_datatypes.h`：枚举转换接口声明（VkFormat↔MTLPixelFormat、VkCompareOp↔MTLCompareFunction 等几十组）。
- `Vulkan/mvk_datatypes.mm`：基于 `MVKPixelFormats` 表的转发实现 + 其余枚举的 switch 映射。
- 旧接口逐步废弃，新代码应直接使用 `MVKPixelFormats`（见 module_core_objects.md）。

---

## 3. Layers/ 详细设计

### 3.1 X-macro 扩展注册

`MVKExtensions.def` 每行声明一个扩展：

```
MVK_EXTENSION(EXT_metal, EXT_METAL, INSTANCE,  device, instance)
```

`MVKExtensions.mm` / `MVKLayers.mm` 在不同宏定义下重复 include 该文件，生成：
- 扩展枚举成员、名称字符串、类型标志；
- `MVKExtensionList` 的 `isEnabled()/disableAllButEnabled...()` 实现。

新增扩展只需改 `.def` 一处。`VK_KHR_portability_subset` 在此注册，使 MoltenVK 符合 macOS 上 Vulkan Loader 的 portability 设备枚举规则。

### 3.2 MVKLayer / MVKLayerManager

- 只有一个 **driver layer**（名称 `MoltenVK`，specVersion 取 `apiVersionToAdvertise`，implementationVersion 取 `MVK_VERSION`）。
- `MVKLayerManager` 为进程级单例（double-checked locking，`_globalManager`），持有 layers 数组；`getLayerNamed(NULL)` 返回 driver layer。
- `MVKLayer` 内嵌 `MVKExtensionList`，实例化时 `disableAllButEnabledInstanceExtensions()` 过滤掉被编译选项/环境变量禁用的扩展。

---

## 4. API/ 公共头文件

| 文件 | 内容 |
|---|---|
| `mvk_vulkan.h` | 总入口，聚合 vulkan_core.h + 全部 MVK 扩展头 |
| `mvk_config.h` | `MVKConfiguration`、`MVK_CONFIG_*` 枚举常量（运行期配置的权威定义） |
| `mvk_datatypes.h` | Vulkan↔Metal 类型转换接口 |
| `mvk_private_api.h` | `vkGetMoltenVKConfigurationMVK` 等私有扩展 |
| `mvk_deprecated_api.h` | Metal 互操作桥接 API（IOSurface、MTLTexture 导出等） |
| `vk_mvk_moltenvk.h` | 兼容旧版的聚合头 |

---

## 5. Common/ 共享平台层

- `MVKOSExtensions.{h,mm}`：跨运行库与转换器使用的 OS 设施——高精度时钟（`mvkGetTimestamps`）、`MVKMutex`（无 pthread 的轻量锁）、页大小、进程内存查询、`MVK操作系统版本` 判定宏。
- `MVKStrings.h`、`MVKCommonEnvironment.h`：字符串与环境宏工具。

---

## 6. 与其他模块的接口

```
vulkan.mm ──调用──► MVKInstance/MVKDevice/MVKCommandBuffer ...（GPUObjects/Commands）
   └── 追踪/日志 ──► Utility(MVKLogging, MVKEnvironment)
MVKLayers ◄──引用── MVKInstance（驱动层枚举、扩展启用判定）
mvk_api.mm ──读取── MVKDevice 性能统计 / MVKPhysicalDevice Metal 能力
```

- 入口层无状态：所有可变状态都在 GPUObjects 层的对象内，入口函数只做句柄反解与转发。
- 配置读取统一走 `getGlobalMVKConfig()`（Utility/MVKEnvironment），或实例级覆盖（`MVKInstance::getMVKConfig()`）。
