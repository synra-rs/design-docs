# 2026-11-03 零权限插件的运行确认

应用提交：`76b5865dd0e86939e205dbaabc54e03a593e2cb5`；内核提交：`3eb38459a9e5296a96c47435449d55799e60bd52`。均已普通push至各仓库 `origin/main`。

## 问题与实际来源

用户的 `@synra-plugin/touchpad` 显示“需要检查权限”，进入权限页却只有“无需权限”。检查本地Touchpad的 `manifest.json`，其 `capabilities` 确实是空数组；没有丢失一组待显示的权限。

存在两个相互独立的遗漏：权限页只在存在能力项时显示提交操作；内核 `Decision::trusted()` 只接受至少一个已允许能力。因此即使向旧内核提交空决定，也永远不能批准零权限插件。这与此前空数组字段省略导致的渲染崩溃是不同路径，上一批没有覆盖本场景。

## 当前行为

- “无需权限”保留其真实含义。尚待确认的零权限插件另显示说明与“允许运行”，使用现有 `plugins_consent_respond` 提交 `granted: []`。
- 零权限清单的显式空决定表示同意运行，不创建虚构能力或自动同意安装。仍需用户明确提交；安装和启用开关本身不批准插件。
- 请求了能力的插件仍允许部分授权，全部拒绝仍不批准，未知能力仍拒绝。
- “需要检查权限”状态改为“需要确认”。列表中的零权限待办为“确认运行”；有权限项的待办仍为“检查权限”。
- 已启用的插件确认后可运行；尚未启用的插件确认后保持停用，随后启用无需重复批准。已有运行状态的零权限插件只显示无需权限，不重复显示确认按钮。
- 新增独立Rust命令集成测试与前端行为测试；浏览器预览包含零权限未确认样例，确认结果保留在假内核自身状态中。

没有修改Touchpad清单或给它补不存在的权限，也没有直接修改真实设备数据库。core原有未提交的 `crates/synra-store/src/db/migrations.rs` 保留原样，构建依赖该本地工作区。

## 验证

| 检查 | 本轮结果 |
| --- | --- |
| 修复前内核回归 | 新增零权限明确同意用例失败，复现 `trusted() == false` |
| 修复后同意规则 | 6项通过，包括部分允许、全部拒绝、未知能力和零权限 |
| 核心完整单元测试 | 最终 `cargo test -p synra-core --lib` 753项通过 |
| 应用原生命令测试 | 373项通过；新增2项经真实命令、SQLite与加载器验证确认、启用及重启。Wasm后端使用ScriptedBackend，不声称证明Touchpad实际触控效果 |
| 默认功能 `cargo check` | 通过 |
| 前端完整测试 | 67个文件、565项通过，包含4项零权限页面行为及假内核确认持久性 |
| 应用 lint、类型与生产构建 | 通过 |
| 应用 shapes | 15项通过；README同步实际命令调用数43 |
| 格式 | 本轮Rust改动文件的rustfmt检查通过；修改与新增源码文件均不超过300行 |
| 浏览器 | 1280×720，零权限待确认、点击允许运行、返回运行中；见 [截图](../ui-design/evidence/2026-11-03-zero-permissions/README.md) |

首次扩大测试时，核心传输超时用例 `missing_confirmation_expires_and_restores_caller_timeouts` 在前端/多个Cargo任务并行负载下失败，752项通过；降低并发负载后重新运行完整753项全部通过。未修改该传输用例或放宽超时断言。

额外Clippy检查未通过，原因均位于本轮未修改文件：包含依赖时被 `synra-observe/src/wire.rs:137` 的 `needless_pass_by_value` 阻止；`--no-deps` 检查被 `link.rs` 文档空行、`host/connections.rs` 的两个const建议、`listener.rs` 的多余借用和unit绑定阻止。没有降低lint级别，亦未将这些检查记录成通过。

开发服务在初次加载和依赖重优化期间还记录过一次 `ResizeObserver loop completed with undelivered notifications`。零权限页面与确认后的状态转移正常；这次未将该独立开发期提示冒充为已修复。保存的浏览器console记录中只有Vite调试消息，它不等同于收集所有window错误事件。未在卓易通真机重装与复测。

## Android复测包

`pnpm exec tauri android build --apk --target aarch64 --ci` 构建通过，APK签名验证通过，仅含arm64-v8a Native库。已上传 [Synra-0.1.0-arm64-20261103-76b5865.apk](ftp://192.168.0.201/Synra/Synra-0.1.0-arm64-20261103-76b5865.apk)，保留旧版本文件；上传后回读完整文件，大小和SHA-256均与本地一致。

- 大小：51726484字节，约49.3 MiB。
- SHA-256：`A3811272956665905BB4367D869736A9C8948964AC438DA343832015B38111E2`。
- 本地产物：应用仓库 `src-tauri/gen/android/app/build/outputs/apk/universal/release/app-universal-release.apk`。

## 提交凭证

| 仓库 | 提交 | subject | 实测字符数 |
| --- | --- | --- | --- |
| core | `3eb3845` | `fix: allow approval without capability grants` | 45 |
| app | `76b5865` | `fix: confirm plugins that request no permissions` | 48 |

两个提交body均为空，已推送 `origin/main`。实际计数命令：

```powershell
[System.Globalization.StringInfo]::new('fix: allow approval without capability grants').LengthInTextElements
[System.Globalization.StringInfo]::new('fix: confirm plugins that request no permissions').LengthInTextElements
```
