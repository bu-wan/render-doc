# 模块文档 06:SmokeTests(自动化冒烟测试)

> 源码位置:`Source/SmokeTests/`(三个引擎分支均有)。
> 类型:Editor 模块(`FDefaultModuleImpl`,无自定义 Startup/Shutdown)。
> 依赖:Public `Core/CoreUObject/Engine/InputCore/Slate/SlateCore`;Private `Projects/AutomationTest/RenderCore/NGSettings`(Editor 目标加 `UnrealEd`)。

## 1. 职责

通过 UE 自动化测试框架(`IMPLEMENT_SIMPLE_AUTOMATION_TEST`)验证 NSS 在编辑器/游戏视口中的
开关行为与调试视图,以 **前后截图对比** 的方式给出可视证据。测试用例集中在 `SmokeTestCases.cpp`
(`#if WITH_EDITOR` 包裹,仅编辑器编译)。

## 2. 原理与算法分析

### 2.1 为什么用"延迟命令 + 截图"而非断言

渲染效果无法用纯 CPU 断言验证(画面好坏是视觉问题),因此测试采用**事件序列编排 + 截图取证**:

```
自动化测试框架逐帧驱动:
  ADD_LATENT_AUTOMATION_COMMAND(命令1) → 命令1 执行(Update 返回 true 后进入命令2)
  → 命令2 … → 截图 → 改 CVar → 等待 → 截图 → 恢复
```

- `IAutomationLatentCommand::Update()` 每帧被调用一次,返回 true 表示该命令完成、进入下一条;
  这使测试能与渲染循环同步(等 1 帧 = 等一次渲染);
- 截图(`FTakeEditorScreenshotCommand`)在指定窗口(Preview/PIE)捕获当前帧,保存为 PNG;
- 人工比对前后截图即可确认功能生效(如 `r.NSS.Enable` 前后画面清晰度变化、`r.NSS.Debug` 前后
  出现调试网格)。

### 2.2 为什么统一走"编辑器视口 + 测试地图"

- 自动化测试运行在编辑器进程内,无需启动独立游戏;`ShowFlag.VisualizeTemporalUpscaler`
  让引擎在屏幕角落显示"当前上采样器名",是验证"NSS 已接管 TAA 上采样"的最直接手段;
- 固定测试地图(`r.TestMap`,默认 `/Game/ThirdPerson/Maps/ThirdPersonMap`)保证每次测试场景一致,
  截图可复现;地图必须由使用方工程提供(测试不内置资产)。

## 3. 基础设施

### 3.1 FSetConsoleVariableLatentCommand(延迟命令)

```cpp
class FSetConsoleVariableLatentCommand : public IAutomationLatentCommand {
public:
    FSetConsoleVariableLatentCommand(const FString& InConsoleVarName, float InValue);
    virtual bool Update() override {
        // 首次调用时执行:找到 CVar → 设置值(ECVF_SetByConsole 优先级,模拟用户在控制台输入)
        // 立即返回 true(单次动作,无需多帧等待)
    }
};
```

### 3.2 测试地图

```cpp
static bool TryLoadTestMap(const FString& MapName, FAutomationTestBase* Test);
    // AutomationOpenMap 失败则 AddError(报告到自动化结果面板)

static TAutoConsoleVariable<FString> CVarTestMap(
    TEXT("r.TestMap"),
    TEXT("/Game/ThirdPerson/Maps/ThirdPersonMap"),   // 默认地图,需项目内存在
    TEXT("The map to load for the NSS tests. Ensure this map exists in your project."));
```

### 3.3 通用流程(每个用例)

1. `ShowFlag.VisualizeTemporalUpscaler 1`(可视化当前时序上采样器,验证 NSS 是否接管);
2. 设置目标 CVar 初值 → 加载测试地图 → `FWaitLatentCommand(5.0f)` 等地图渲染稳定
   (5 秒足够首帧编译着色器、稳定画面);
3. 找 "Preview"/"PIE" 标题窗口(自动化测试默认在这种窗口内渲染);
4. 截图 before → 改 CVar → 等 1 秒 → 截图 after → 恢复默认。

## 4. 三个测试用例

| 测试 | 名称 | 步骤要点 |
|---|---|---|
| `FNSSEnableTest` | `NG.PluginTests.NSSEnableTest` | `r.NSS.Enable false` → 加载地图 → 截图 → `true` → 截图 → 恢复 false |
| `FNSSDebugTest` | `NG.PluginTests.NSSDebugTest` | `r.NSS.Enable true` → `r.NSS.Debug false` → 截图 → `true` → 截图 → 恢复 false |
| `FNSSAdjustMipBiasTest` | `NG.PluginTests.NSSAdjustMipBiasTest` | `r.NSS.Enable true` → `r.NSS.AdjustMipBias false` → 截图 → `true` → 截图 |

运行方式:编辑器 → Window → Developer Tools → Automation(或命令行 `-ExecCmds=Automation RunTest NG.PluginTests.NSSEnableTest`),
截图输出到项目 Saved 目录(`NG_NSS_Enable_before/after.png`、`NSS_Debug_before/after.png`、`NG_NSS_AdjustMipBias_False/True.png`)。

## 5. 复现要点

1. 测试模块必须 `Type = Editor`、加载阶段 PostConfigInit(4.27 分支为 PostEngineInit),并依赖 `AutomationTest`。
2. 截图需真实渲染上下文,因此测试 flags 要含 `NonNullRHI`(5.x 实现含
   `EditorContext|ClientContext|ServerContext|CommandletContext|EngineFilter|NonNullRHI`)。
3. 用例不设通过/失败断言(仅截图 + 日志),属"冒烟/人工比对"性质;若需要 CI 化,可加
   像素比对(如用 `FImageComparison` 之类工具)判断前后差异。
4. 若把测试扩展为 CI 自动门禁,建议:① 用确定性帧序列替代等待时间;② 对比基准截图
   计算感知差异阈值;③ 覆盖 NFRU(开关 + 调试视图 + 仅插值帧模式)。
