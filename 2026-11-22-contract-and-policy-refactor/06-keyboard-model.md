# 06 · 结构化按键与平台后端

状态：结构化按键及宿主调用链已接入。Windows ARM64 实机和 Alpine ARM64 虚拟机内的 X11 事件接收已验收，macOS 原生按键仍待实测。平台验证见[实施记录](./IMPLEMENTATION.md)。优先级：P2。

## 1. 改造前的证据

[actions.rs:68](https://github.com/synra-rs/synra-core/blob/a75d399/crates/synra-platform/src/desktop/platform/actions.rs#L68) 将同一 chord 字符串直接交给 Linux xdotool、macOS AppleScript 和 Windows 后端。这些后端本身使用不同语法，字符串不构成跨平台语义。

[simulate_windows_native:187](https://github.com/synra-rs/synra-core/blob/a75d399/crates/synra-platform/src/desktop/platform/actions.rs#L187) 仅识别 ^c、^x、^v、^z、^y、^a、^s、^f、{LWIN}e，其他输入回退到 SendKeys。注释明确说明这些组合来自 hotkey 插件的内置功能。提交历史 `8e12715 fix: use native Windows key input` 可作为该局部修复的沿革。

[guard/text.rs](https://github.com/synra-rs/synra-core/blob/a75d399/crates/synra-platform/src/guard/text.rs) 只限制长度和空输入，没有完整 chord 语法。现有 rate limit 与独立授权继续保留，它们不是本次要移除的分支。

## 2. 根因

契约把一个平台相关的命令文本当成“按键组合”。为了支持某个新组合，又在文本上加特例选择不同执行器。继续增加 match 分支会让可用组合、错误和资源释放行为随拼写变化。

## 3. 目标设计

### 3.1 统一表达语义

在 synra-platform 定义 KeyChord：规范化的 modifiers 集合与一个主键。主键使用有限的键值类型，例如字符、功能键、方向键和命名特殊键。模型不包含程序名、脚本片段或插件身份。

外部仍需要字符串时，使用单一规范语法，例如 Ctrl+Shift+S、Meta+E；在能力入口解析一次为 KeyChord，后端不再重新解析。第一阶段仅支持单次组合，文本输入和多步宏不混进 chord。SDK 如提供 builder，输出同一规范形式。

Ctrl、Alt、Shift、Meta 的具体物理含义在文档中说明；不要隐式把 Ctrl 替换成 macOS Command。需要“平台主快捷键”时由调用者显式选择，避免 core 猜业务意图。

### 3.2 后端只负责执行

- Windows 使用统一原生路径执行所有受支持的 KeyChord，不按插件内置组合选择 Native/SendKeys。
- Linux/macOS 后端从同一 KeyChord 映射到自身支持的执行参数，禁止把原字符串原样转发并假定语义相同。
- 缺少执行依赖或不支持某类键时返回结构化 Unsupported，不能静默降成文本键入。
- 支持查询区分后端已实现、当前环境可用和策略允许；仅“允许询问”不能让 platformSupports 宣称可执行。
- 每次组合记录已成功按下的 modifiers，任何后续失败都逆序尝试释放；保留原始失败和释放失败诊断，避免 modifier 卡住。

## 4. 实施步骤与文件

1. 盘点 Platform::simulate_keys、PlatformAction::SimulateKeys、WIT platform 参数、SDK 包装和插件字符串调用；记录当前格式分布。
2. 在 `synra-platform/src/` 增加按键值、解析、验证及后端接口，按责任拆分，避免继续扩展 actions.rs 的大 match。
3. 保留现有能力 gate、策略与节流，解析非法输入在进入执行器前失败。
4. Windows 先实现通用原生执行及失败释放，移除九个组合白名单和 SendKeys 回退。
5. 将 Linux/macOS 映射到相同模型；本次不能验证的能力明确标注，不把 recording backend 测试当作平台实机证明。
6. 更新 SDK/CLI 示例、生成契约及真实调用方；若修改 WIT，用现有 codegen 生成绑定，不手改生成物。
7. 清理旧 chord 语法和按功能名挑执行器的分支。尚有外部 JS/SDK 使用者时按发行边界说明变更，不暗中保留两套解析语义。

## 5. 验收场景

| 层次 | 场景 |
| --- | --- |
| 解析 | 大小写归一、修饰键顺序、重复 modifier、空组合、未知键、加号字符等歧义有明确结果 |
| 权限 | 默认关闭、无 grant、允许后节流均保持原有策略 |
| 通用性 | Ctrl+Shift+S、Alt+Tab、Meta+E 等非旧白名单组合不需要新增 chord 分支 |
| 资源释放 | 第一个 modifier、第二个 modifier、主键、释放动作各自失败都不遗留按键 |
| 参数隔离 | 输入不能成为脚本或命令参数结构的一部分 |
| 平台 | Windows 真实应用验证；Linux/macOS 按实际后端环境分别记录，不以跨平台编译替代效果验证 |

## 6. 风险与完成条件

键盘布局、IME 与当前焦点影响物理效果，测试应区分逻辑键、字符输入和扫描码含义。首批不扩大为任意输入自动化系统，也不改变用户逐次确认或授权边界。

完成条件是统一输入模型、通用后端选择、失败释放和至少 Windows 实机验收均有证据。原有常见复制粘贴组合仍需回归，不能只证明新增组合有效。

## 7. 实施规则

`KeyChord` 解析后按 `Ctrl`、`Alt`、`Shift`、`Meta` 顺序按下修饰键，最后按下主键。释放从主键开始，逆序释放所有已按下的键。外部执行器超时且输入状态不确定时，同时尝试释放可能已按下的键。同一个桌面宿主的并发组合由键盘互斥锁保护。

Windows 使用原生虚拟键码，macOS 用固定 ANSI 位置表示字母和数字键，并读取辅助功能权限来判断可用性。X11 用探测命令检查显示连接和 `xdotool`，再把按键值转换为固定键名。宿主选区复制通过同一执行路径，macOS 明确选择 `Meta+C`，其他桌面选择 `Ctrl+C`。[Windows 键码定义](https://learn.microsoft.com/en-us/windows/win32/inputdev/virtual-key-codes) [xdotool 按键入口](https://github.com/jordansissel/xdotool/blob/main/cmd_key.c)
