# 平台动作分发与剪贴板返回值

日期：2026-10-26；对应 [SDK](../15-plugin-sdk.md)、[官方插件](../29-official-plugins.md) 与 [M2](../19-roadmap.md)。

## 生产入口发现的缺口

共享 HostCalls 发送 openUrl、readClipboard 等 camelCase 名称，两个生产宿主却匹配 open-url、read-clipboard。
旧测试直接调用宿主时使用后者，因而没有覆盖真实调用方与被调用方之间的契约。
新增桌面入口测试首先收到 badArgument，证明剪贴板读取尚未执行就已经分发失败。

桌面宿主还把 read_clipboard 返回值映射为 unit，丢弃了读到的文本；daemon 适配器也有同样的丢弃写法。
共享层随后把空返回值改为 null，使空字符串与没有文本无法区分。
真实 WIT 组件回归先复现了空文本返回 null，而预期为 JSON 空字符串。

## 当前契约与实现

Host::platform 直接接收 PlatformAction 枚举，两个宿主按同一枚举分发，不保留字符串动作兼容入口。
未知 WIT 函数仍由公共入口拒绝；类型化宿主边界不再接受任意动作名称。
普通剪贴板读取返回原始 Option<String>，适配器保留值，共享层负责一次 JSON 编码：

| 平台值 | 成功 outcome 中的 JSON 文本 | 含义 |
| --- | --- | --- |
| `None` | `null` | 没有文本 |
| `Some("")` | `""` | 有效空文本 |
| `Some("null")` | `"null"` | 内容为 null 的文本，并非没有文本 |
| `Some(其他文本)` | 正常转义的 JSON 字符串 | 保留引号、换行、反斜杠及 Unicode |

WIT、SDK 示例及两个生成的 WIT 资源同步说明这条值契约，未修改二进制函数形状。

## 定向验证

真实 WIT 组件对四种平台值分别执行有授权和无授权调用，检查完整返回和八次调用计数。
无授权调用返回 capabilityDenied，不增加宿主调用次数。
桌面测试经过 HostCalls、实际 DesktopHost、真实内存数据库和 CommandPlatform，仅将系统进程执行替换为可记录的后端。
另覆盖同一入口打开链接的实际分发与无授权拒绝，不启动测试机浏览器。
daemon 入口覆盖坏 URL 的 badArgument、合法 URL 和剪贴板读取的 notImplemented。

## 阶段门禁

core workspace 全量累计 3600 项测试通过，零失败、零忽略，包含真实引擎 Host 373 项。
workspace 全目标严格 Clippy、12 项架构守卫通过，守卫覆盖 Rust 1.88 MSRV、实际 App 调用方及 116 份文档链接。
六个生成物只读检查通过，真实 SDK guest 往返 1 项、CLI 构建装载和失败保留旧产物 2 项通过。
App 无默认特性全量 328 项测试、全目标严格 Clippy 及默认 Tauri 编译通过；本轮未修改前端、依赖或原生命令实现。
定向文档链接检查 79 份零问题，数量声明无冲突，保留既有 3 项不可自动检查项。

第一次并发门禁遇到 Windows 页面文件不足（os error 1455），随后 App 编译也未能读取依赖元数据。
确认原任务退出并恢复可用内存后，重型检查改为单任务串行，使用 -j 1 或 CARGO_BUILD_JOBS=1；相同源码的上述门禁全部通过。
未降低 lint 或测试要求，也未清空依赖缓存、修改系统配置或终止无关程序。

## 剩余平台工作

本轮修复宿主接线和返回值，不把替换进程后的测试当作操作系统剪贴板或两台设备的验收。
CommandPlatform 的 macOS pbcopy 目前没有输入数据，Linux 命令后端仍拒绝写入；Windows 在 PowerShell -Command 后追加数据的方式需要实际进程验证和修复。
SystemRunner 的进程等待和输出采集仍无明确上限，UTF-8 与读写大小限制也须一起收敛；后续单独实现有界标准输入、输出、超时与清理。
选中文本捕获、跨设备请求回复、手机 UI 和手机到电脑的真实输入仍待实现，不以普通剪贴板读取替代选区获取。

本节保留该阶段观察；后续命令数据传递、预算与平台可用性修复见[有界命令 IO 记录](./2026-10-26-command-io.md)。
