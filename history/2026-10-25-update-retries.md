# 自动更新重试与扫描测试稳定性（2026-10-25–26）

本记录保存本次开发的改动与验证证据，不替代当前设计或任务状态。
契约见 [分发 §6](../16-plugin-distribution.md)，剩余架构工作见 [路线图 D2–D4](../19-roadmap.md)。

## 自动更新失败排期

- store 持久化失败计数及下次到期时间，计数最多为 3；首次失败等待 30 分钟，第二次等待 60 分钟，之后恢复每日检查。
- 同一已安装版本、候选版本和来源保留排期，任一变化重新计数；写入失败记录时，在同一 SQL 内复核这些字段、检查时间及当前安装事实，拒绝过期请求。
- core 统一两个宿主的到期筛选和失败记录准入；取消、策略变化及关闭自动更新后不新增失败排期，时间回拨重新检查。
- app 后台检查显式传入当前时间；daemon 与 app 均在网络操作前释放数据库锁，并报告无法保存重试状态的错误。
- 两个宿主分别通过真实 HTTP 服务注入 503 或损坏 tarball，重开 SQLite 后验证到期前不请求、计数不丢失、第三次失败恢复每日检查，且旧安装版本保留。

这仍是调度切片：完整更新工作流、统一事件、操作任务模型和提交恢复尚未完成，不将 D2–D4 标记为完成。

## 扫描测试竞态

app 的首次完整测试发现 `a_network_scan_records_itself_while_it_runs_and_clears_that_before_it_answers` 失败，
原因是观察线程可能在 60 毫秒的扫描结束后才获调度；相同模式也存在于 discovery 的截止时间测试。
现在生产扫描复用可传入时间和等待函数的路径，在等待阶段直接断言运行快照及工作线程可见的截止时间，
并断言返回前均已清理；不再依赖观察线程抢到短暂窗口，也不通过延长睡眠或重复运行掩盖问题。
生产入口继续使用真实时钟和原有有界等待。

## 本次验证

所有命令分别在对应仓库根目录执行；通过结果只覆盖本记录对应的改动。

| 仓库 | 命令 | 结果 |
| --- | --- | --- |
| core | `cargo test -p synra-store --lib sqlite::updates::tests` | 9 个定向测试通过 |
| core | `cargo test -p synra-core --lib plugin_updates::tests` | 4 个定向测试通过 |
| core | `cargo test -p synra-daemon runtime::updates::tests` | 18 个更新测试通过 |
| core | `cargo test --workspace` | 通过 |
| core | `cargo clippy --workspace --all-targets -- -D warnings` | 通过 |
| core | `cargo test -p synra-plugin-host --features wasm-runtime --quiet` | 352 个通过，1 个忽略 |
| core | `cargo run -q -p guards` | 12 项通过，包含调用方构建、MSRV 1.88、文档链接和可检查的文档计数 |
| app | `cargo test --no-default-features --manifest-path src-tauri/Cargo.toml --lib commands::plugin_updates` | 10 个更新测试通过 |
| app | `cargo test --no-default-features --manifest-path src-tauri/Cargo.toml --lib scan` | 12 个扫描相关测试通过 |
| app | `cargo test --no-default-features --manifest-path src-tauri/Cargo.toml --quiet` | 修复竞态后 317 个测试通过 |
| app | `cargo clippy --no-default-features --manifest-path src-tauri/Cargo.toml --all-targets -- -D warnings` | 通过 |
| app | `cargo check --manifest-path src-tauri/Cargo.toml` | 含 Tauri 默认 feature 的编译通过 |
| app | `pnpm test` | 35 个文件、328 个测试通过 |
| app | `pnpm run lint`、`pnpm run build`、`pnpm run shapes` | 通过；构建仍有已有的大 chunk 提示 |

修改的独立模块已通过 rustfmt 检查，三个仓库的 `git diff --check` 通过。
尚未以本次改动验证 Android 运行行为；本次没有发布操作。
