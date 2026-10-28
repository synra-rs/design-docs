# 2026-10-28 · 插件阶段签名提交与固定版本同步

## 已提交的阶段

已有 Pinentry 等待由用户在本机完成解锁。以下提交均验证为有效 GPG 签名（`%G? = G`），
`%b` 为空，并已推送；除公共文档外使用 main，公共文档只推送工作分支，未部署网站。

subject 使用实际执行的 PowerShell 命令计数：

```powershell
[System.Globalization.StringInfo]::new('<subject>').LengthInTextElements
```

表中 subject 代入该命令即为对应实测，均不超过 50 字符。

| 仓库 | Commit | 完整 subject | 字符数 |
| --- | --- | --- | --- |
| Core | cadb89b | test: require exact command lookup matches | 42 |
| Core | 238d561 | feat: build installable hybrid plugin packages | 46 |
| Core | dd45c53 | fix: release database locks before activation | 45 |
| Core | 4f6dd24 | fix: expose actual scoped host SDK capabilities | 47 |
| Core | 9d1e060 | feat: track persistent plugin authority epochs | 46 |
| Core | e0c5481 | fix: compile plugins on a bounded host stack | 44 |
| App | a7cdd3a | feat: host isolated hybrid plugin pages | 39 |
| JS | 1c28d77 | docs: define paired device list visibility | 42 |
| Starter | 54dd4f5 | feat: add starter diagnostics and target selection | 50 |
| Starter | ba09a44 | ci: guard matching SDK and author CLI revisions | 47 |
| Touchpad | cc56900 | chore: align plugin SDK and author CLI pins | 43 |
| Touchpad | 084fa57 | ci: guard matching SDK and author CLI revisions | 47 |
| Docs 工作分支 | ce17bcd | docs: explain installable hybrid plugin packages | 48 |

公共文档分支为 `fix/plugin-event-identity`，未推送或合入会触发 Pages 部署的 main。
上述阶段签名及推送后，各仓库工作区为空、与 upstream 比较为 `0 0`；设计文档随本记录独立提交。

## 固定版本与定向验证

两个插件的 Cargo.toml、Cargo.lock、README 与 CI author CLI 统一到已签名 Core
`e0c5481494b409af6fdbbdec9f78786ae4ee0612`，没有发布 SDK、打 tag 或上传插件包。
CI 通过 Cargo metadata 比对 SDK source 和统一的 `SYNRA_CORE_REVISION`，避免再次只更新 SDK 而遗漏 CLI。

Starter 增加固定 Node 26.12.1、pnpm 12.14.0 的 UI CI 步骤，frozen-lockfile 安装通过。
更新固定 Git source 后重新完成 Native/Wasm Clippy、Rust 1.88 Native/Wasm 检查及 Native 测试：
Starter 3 项、Touchpad 18 项均通过；新 Core CLI 已从当前干净源码重建，两插件实际 build/verify 通过。
Starter 为 4 ok、Touchpad 为 3 ok，均为 0 warn、0 fail。

Starter 固定版本同步前后的组件 SHA-256 都为
`b5051779d1ca4a85f67fea6ac1ce8308b1db7e19e52966cddaa821b8fbf42471`。
因此复用此前同一组件的实际 Wasm 与桌面调用证据，不因仅 Git source 改变而重复整套 App 测试。
界面和宿主证据见[目标选择记录](./2026-10-28-starter-targets.md)与[真实桌面记录](./2026-10-28-desktop-smoke.md)。

GitHub 匿名 Actions API 返回 `403 rate limit exceeded`，本轮未确认远端 CI 结果；本地门禁通过不冒充远端 CI 成功。

## 目标仍未完成

Starter 的电脑选中文本、手机剪贴板、电脑打开链接及共享认证请求/回复仍待开发；
Touchpad 仍未接入手机 UI、认证会话与原生鼠标输入。没有把本轮阶段提交视为 M0–M6 完成。
