# 有界桌面命令 IO 与剪贴板输入

日期：2026-10-26；对应[共享平台边界](../07-core-kernel.md#92-桌面命令后端的预算与数据边界)与[官方插件需求](../29-official-plugins.md)。

## 缺口与实际修复

macOS 写剪贴板只启动 pbcopy，没有传入文本；新增回归先复现 runner 收到 None 而非输入数据。
CommandRunner 现显式接收可选 stdin，并报告自身可用性；三个桌面写入后端都把完整 UTF-8 文本送入 stdin，完成后关闭管道。
Windows 剪贴板、通知和按键脚本均固定，不再在 -Command 后追加用户文本；脚本异常终止，不把通知模块初始化当作发送成功。
macOS 的脚本参数和 Linux 的通知、按键参数使用选项终止符，避免以 -e 等开头的数据改变命令解释。

SystemRunner 保持同步 API，用每进程共享的八个命令名额限制工作线程、缓冲及待回收进程。
请求协调线程使进程启动或 IO 阻塞不拖住调用方的期限；运行默认 5 s，另留 500 ms 清理。
stdin 上限 256 KiB，stdout 2 MiB、stderr 16 KiB，argv 最多 32 项、合计 64 KiB；均先验证或在读取时限制。
stdout 严格解码 UTF-8，失败错误不拼接 stderr；结构化诊断只包含程序、类别、限额及清理状态。
策略层的剪贴板读写统一按 256 KiB UTF-8 字节限制，修复 headless 与桌面限额及单位不一致。

## 取消与所有权

结束或超时后终止并确认回收直属启动进程，不终止用户浏览器等应用进程树。
管道被后代进程继承时，调用方仍按时返回，未完成的 pipe worker 持有名额；后续请求不能绕过旧工作而无界创建线程。
直属进程暂未确认回收时留在有界回收表中，后续调用与状态查询在不持表锁的情况下重检。
预算直到进程及所有 IO 所有者均退出才释放；Debug 只展示计数，避免沿回收表与共享 lease 的循环引用递归格式化。

## 实际进程验证

Rust 测试可执行文件自身提供受控子进程入口，不依赖额外脚本或解释器。
覆盖无 stdin、空 stdin、引号和 Unicode、stdout/stderr 超限、无效 UTF-8、非零退出、敏感 stderr 不进入错误、阻塞 stdin 超时及实际进程已退出。
另覆盖后代继承管道、超时后名额仍保留、忙时拒绝、管道关闭后恢复名额，以及回收表释放和安全 Debug。
所有 fixture 进程都受期限或短生命周期约束；临时目录由测试持有并清理。

Windows 还执行真实 PowerShell，验证含脚本符号的 UTF-8 输入只作为数据、脚本失败不会静默成功，以及剪贴板 cmdlet 参数与 nullable JSON 行为。
该测试发现 Windows PowerShell 的 ConvertTo-Json 对 null 不输出内容；固定脚本现用同一个辅助函数显式输出 null，空文本仍输出 JSON 空字符串。
测试只查询 cmdlet 元数据并处理自有字符串，不读取或覆盖当前用户剪贴板，不触发通知或按键。

App 的生产 UI RPC 与 Wasm 宿主适配均保留命令错误的类别、预算、限额及 cleanupPending；缺少权限或输入超限不调用 runner。
IPC 区分 timeout、rate_limited、payload_too_large；WIT 的 JSON details 与 IPC params 由同一平台错误提供。

参数处理参考 [Microsoft PowerShell 说明](https://learn.microsoft.com/en-us/powershell/module/microsoft.powershell.core/about/about_powershell_exe?view=powershell-5.1)
及 [Apple 的 osascript 参数示例](https://developer.apple.com/documentation/security/customizing-the-xcode-archive-process?language=objc)。

## 阶段验证

Core 全工作区 3,613 项测试通过，无失败或忽略；平台模块包含 93 项测试。
全工作区、全目标 Clippy 以 -D warnings 通过；新增 runner、管道和 PowerShell 模块通过 rustfmt 检查，已有文件只格式化本轮修改范围。
12 项仓库守卫通过，包含 Core MSRV 1.88、App 调用方构建、117 份文档链接及可检验计数。
Android aarch64 的平台、内核及启用 Wasm runtime 的插件宿主编译通过；编译不等于真机验证。

App 无默认特性全量 332 项测试、全目标严格 Clippy、默认 Tauri 构建检查及 MSRV 1.90 全目标编译通过。
检查串行执行并限制单个 Cargo 构建任务，避免已观察到的 Windows 内存压力。
本轮没有修改 WIT、SDK、前端或插件仓库，沿用其上一阶段已通过的对应验证，不重复生成或全量构建。
所有测试日志位于各仓库忽略的 target 目录；未执行发版或网站部署。

## 剩余边界

macOS/Linux 本轮覆盖命令表、输入数据和共享进程服务；未在这两个桌面环境实际修改剪贴板或注入按键。
Linux 仍需要显示会话及 xclip 等程序，Windows 通知仍需要相应后端；缺失或失败明确拒绝，不静默成功。
移动平台的原生剪贴板、选区捕获、键盘语义和交互限制、鼠标输入服务、完整取消与撤销仍待后续接入。
本轮不代表 Starter 的跨设备复制或触摸板手机控制电脑已经完成。
