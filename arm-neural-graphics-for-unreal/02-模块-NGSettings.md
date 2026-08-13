# 模块文档 02:NGSettings(设置与控制台变量)

> 源码位置:`Source/NGSettings/`(三个引擎分支均有)。
> 依赖:Public `NGShared`;Private `Core/Engine/Projects/RenderCore/Renderer/RHI/CoreUObject/EngineSettings/DeveloperSettings`。

## 1. 职责

- 集中声明插件全部 **控制台变量(CVar)**(`TAutoConsoleVariable<int32>`),供 NSS/NFRU/NGVulkanBackend 引用。
- 提供 **`UNGSettings : UDeveloperSettings`**,把 CVar 暴露为编辑器"项目设置 → Plugins"界面。
- 启动时从 ini(`/Script/NGSettings.NGSettings` 段)回放 CVar 值。

## 2. 原理与算法分析

### 2.1 为什么用 CVar 而非普通成员变量

- **运行期可调**:CVar 可在控制台/ini/蓝图侧随时修改,是渲染技术调试的标配(如开关、切质量档);
- **线程安全**:渲染线程与游戏线程并发读,CVar 提供 `GetValueOnGameThread()/GetValueOnRenderThread()/GetValueOnAnyThread()`。
  本模块 CVar 均标记 `ECVF_RenderThreadSafe`——该标志让引擎在 CVar 修改时维护**每线程快照**,
  保证渲染线程读到的值是某一帧的一致状态,而不是写入一半的中间值;
- **单一事实来源**:ini → CVar → 代码读取,避免"配置散落各处"。

### 2.2 UDeveloperSettings 与 CVar 的双向绑定原理

`UNGSettings` 的每个 `UPROPERTY` 用 `meta=(ConsoleVariable="r.NSS.Enable")` 声明**所属 CVar**。
引擎的 UDeveloperSettings 框架自动提供两个方向的同步:

- **界面 ← 当前值**(`ImportConsoleVariableValues`,在 `PostInitProperties` 中调用):
  把 CVar 当前值写入属性,界面显示真实状态(例如用户之前用控制台开过 NSS,界面复选框应显示开);
- **界面 → CVar**(`ExportValuesToConsoleVariables`,在 `PostEditChangeProperty` 中调用):
  用户改属性时,把新值写回 CVar,立即生效。

> 注意 `ImportConsoleVariableValues` 只在 `IsTemplate()`(默认对象)上执行——属性绑定的是
> 类默认对象,而非运行时实例,避免每个实例都重复导入。

### 2.3 ini 回放机制

`ApplyCVarSettingsFromIni("/Script/NGSettings.NGSettings", *GEngineIni, ECVF_SetByProjectSetting)`
把 `DefaultEngine.ini` 里对应段的键值写成 CVar,`ECVF_SetByProjectSetting` 优先级保证
"项目设置 > 默认值",同时**低于**用户在控制台临时输入的 `ECVF_SetByConsole`——这保证
ini 不会覆盖玩家/测试的命令行设置。

## 3. 控制台变量总表

所有 CVar 均为 `TAutoConsoleVariable<int32>`、声明 `extern NGSETTINGS_API` 于头文件、
定义于 `NGSettings.cpp`(见架构总览 §9 表格)。这里补充实现细节:

### 3.1 通用

```cpp
TAutoConsoleVariable<int32> CVarNGDeferDelete(
    TEXT("r.NG.DeferDelete"), 5,
    TEXT("Number of frames to defer deletion. Must be greater than 0."),
    ECVF_RenderThreadSafe);
// 默认 5 帧来自经验值:GPU 队列深度 + 渲染线程流水深度,保证"CPU 删除时 GPU 早已不再引用"
```

### 3.2 NSS

| CVar | 默认 | 说明 |
|---|---|---|
| `CVarEnableNSS` (`r.NSS.Enable`) | 0 | 总开关 |
| `CVarEnableNSSInEditor` (`r.NSS.EnableInEditorViewport`) | 0 | 编辑器视口默认启用 |
| `CVarNSSDebug` (`r.NSS.Debug`) | 0 | 调试视图(帮助文本描述 4 行×4 列排布) |
| `CVarNSSAdjustMipBias` (`r.NSS.AdjustMipBias`) | 1 | 允许改写 `r.ViewTextureMipBias.Min/Offset`,`ECVF_ReadOnly`(只读,避免运行时误改) |
| `CVarNSSUseFragmentShader` (`r.NSS.UseFragmentShader`) | 1 | 1=PS 路径 0=CS 路径 |
| `CVarNSSShaderQualityMode` (`r.NSS.ShaderQualityMode`) | 1 | 0=Quality 1=Balanced 2=Performance |

### 3.3 NFRU

| CVar | 默认 | 说明 |
|---|---|---|
| `CVarEnableNFRU` (`r.NFRU.Enable`) | 0 | 总开关 |
| `CVarNFRUCaptureDebugUI` (`r.NFRU.CaptureDebugUI`) | `!UE_BUILD_SHIPPING` | 强制检测/复制仅首帧绘制的调试 UI |
| `CVarNFRUUpdateGlobalFrameTime` (`r.NFRU.UpdateGlobalFrameTime`) | 0 | 把含插值的帧时间写回 `GAverageMS/GAverageFPS` |
| `CVarNFRUModifySlateDeltaTime` (`r.NFRU.ModifySlateDeltaTime`) | 1 | Slate 重绘时置 delta time=0(Slate Redraw 模式,UI 提取模式下忽略) |
| `CVarNFRUPaceAdjuster` (`r.NFRU.PaceAdjuster`) | 0 | 动态 FPS 调节开关 |
| `CVarNFRUUpAdjustFrameCount` (`r.NFRU.UpAdjustFrameCount`) | 40 | 升频需要连续的空闲帧数(经验值) |
| `CVarNFRUDownAdjustFrameCount` (`r.NFRU.DownAdjustFrameCount`) | 20 | 降频需要的连续忙碌帧数(经验值) |
| `CVarNFRUUseFragmentShader` (`r.NFRU.UseFragmentShader`) | 1 | 1=PS 路径 0=CS 路径 |
| `CVarNFRUOnlyInterpolatedFrames` (`r.NFRU.OnlyInterpolatedFrames`) | 0 | 仅显示插值帧(仅非 SHIPPING 编译) |
| `CVarNFRUShowDebugView` (`r.NFRU.ShowDebugView`) | 0 | 调试视图(仅非 SHIPPING 编译) |

### 3.4 值得注意的默认值设计

- `r.NSS.Debug` 帮助文本直接描述了 4×4 调试视图的每个格子含义——这是 SDK 调试输出布局的
  文档化入口,排布与 SDK 内部各中间 buffer 一一对应(可对照 NSS 文档 §7);
- `r.NFRU.UpAdjustFrameCount=40 > DownAdjustFrameCount=20`:升频需要更长的稳定观察期,
  降频响应更快——因为"掉帧"比"可提升"更影响体验,算法倾向保守;
- 调试专用 CVar(`OnlyInterpolatedFrames`/`ShowDebugView`)用 `#if (UE_BUILD_DEBUG || UE_BUILD_DEVELOPMENT || UE_BUILD_TEST)`
  包裹,SHIPPING 下不编译,避免调试能力泄漏进发布包。

## 4. UNGSettings(UDeveloperSettings)

```cpp
UCLASS(Config = Engine, DefaultConfig, DisplayName = "Neural Graphics")
class NGSETTINGS_API UNGSettings : public UDeveloperSettings
{
    // —— 三段式定位:容器=Project,分类=Plugins,节=NGSettings ——
    // 决定了设置出现在:项目设置 → Plugins → Neural Graphics
    virtual FName GetContainerName() const override;  // "Project"
    virtual FName GetCategoryName() const override;   // "Plugins"
    virtual FName GetSectionName() const override;    // "NGSettings"

    // —— 属性示例:每个属性用 meta=(ConsoleVariable="r.…") 与 CVar 双向绑定(原理见 §2.2) ——
    UPROPERTY(Config, EditAnywhere, Category="NG",
        meta=(ConsoleVariable="r.NG.DeferDelete", DisplayName="Frames to defer deletion",
              EditCondition="bNSSEnabled || bNFRUEnabled"))       // 任一功能开启才可编辑
    int32 bNGDeferDelete;

    UPROPERTY(Config, EditAnywhere, Category="NSS",
        meta=(ConsoleVariable="r.NSS.Enable", DisplayName="Enable"))
    bool bNSSEnabled;

    UPROPERTY(Config, EditAnywhere, Category="NSS",
        meta=(ConsoleVariable="r.NSS.ShaderQualityMode", DisplayName="Shader Quality Mode",
              ToolTip="NSS Shader Quality Mode\n0 = Quality\n1 = Balanced\n2 = Performance"))
    ENSSShaderQualityMode NSSShaderQualityMode = ENSSShaderQualityMode::Balanced;  // 默认与 CVar 一致

    UPROPERTY(Config, EditAnywhere, Category="NFRU",
        meta=(ConsoleVariable="r.NFRU.Enable", DisplayName="Enable"))
    bool bNFRUEnabled;
    // …其余属性同构:bNSSEnabledInEditorViewport、bNSSDebug、bNSSAdjustMipBias、
    //   bCaptureDebugUI、bUpdateGlobalFrameTime、bModifySlateDeltaTime
};
```

- `ENSSShaderQualityMode : uint8` 枚举(Quality/Balanced/Performance,BlueprintType):
  以枚举而非整数暴露质量档,界面友好且编译期类型安全。
- `PostInitProperties()`(仅模板对象 `IsTemplate()`):`ImportConsoleVariableValues()`
  把 ini/CVar 当前值导入属性,保证界面显示与实际状态一致。
- `PostEditChangeProperty()`(WITH_EDITOR):`ExportValuesToConsoleVariables(Property)` 把界面改动
  写回 CVar——实现"改设置即时生效"。
- `EditCondition` 元数据:某功能未启用时,相关项置灰,防止用户配置无效组合。

## 5. 模块启动逻辑

```cpp
void NGSettingsModule::StartupModule()
{
    // 从 DefaultEngine.ini 回放设置(原理见 §2.3)
    UE::ConfigUtilities::ApplyCVarSettingsFromIni(
        TEXT("/Script/NGSettings.NGSettings"), *GEngineIni, ECVF_SetByProjectSetting);
}
```

- 加载阶段:5.x 分支为 `EarliestPossible`(尽可能早,先于其他模块使用 CVar);
  4.27 分支为 `PreDefault`。
- 启动即从 `DefaultEngine.ini` 的 `[/Script/NGSettings.NGSettings]` 段回放设置。

## 6. 复现要点

1. CVar 全部声明为 `extern NGSETTINGS_API …` 供其他模块跨 DLL 引用;定义集中在 NGSettings.cpp。
2. `meta=(ConsoleVariable="r.…")` 是 UDeveloperSettings 的官方机制,不要手写同步逻辑。
3. 属性默认值要与 CVar 默认值一致(如 `NSSShaderQualityMode = Balanced` ↔ CVar 默认 1)。
4. `ECVF_ReadOnly` 仅用于 `r.NSS.AdjustMipBias`(避免运行时被误改导致状态不一致)。
5. 新加设置项三步走:加 CVar → 加绑定属性 → 在 CVar 总表(架构文档 §9)登记,缺一不可。
