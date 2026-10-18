# 19 · 实施路线图

> 本文给出从零到可用的分阶段计划。每阶段都有**可运行的产物**与**明确的验收标准**——不允许"全部写完才能跑"。
> 阶段划分以"能否验证"为准，不以"模块完整性"为准。

---

## 0. 总体节奏

```
M0  骨架与验收工具        ──▶ 能 `cargo run -p synra-cli -- status`
M2  意图与插件（最小可用）  ──▶ 一个 headless 插件能处理分享并调用宿主能力
M3  插件安全与分发         ──▶ 签名安装、闸门、审计、跨设备同步
M4  桌面应用完整           ──▶ Tauri 桌面 App 可用（UI 含插件页面）
M5  移动端                 ──▶ Android/iOS 可用
M6  生态与优化             ──▶ 更新、观测、性能、可选集群
```

**关键原则**：
- **M1 结束就能验证"内核是否真的独立于 UI"**（两个 CLI 进程互相通信）；
- **M2 结束就能验证"插件是否真的隔离"**（一个 Wasm 插件在沙箱里跑）；
- **不把"前端"排在最后**——M0 就要有前端骨架（假内核），避免前端成为瓶颈。

### 0.1 ⚠️ M0 的第一个动作：技术验证 spike

调研 `D:/Projects` 后的一个重要发现（[06 §9.3](./06-workspace-layout.md)）：
**这个生态里没有任何项目用过 `wasmtime` / 组件模型，也没有项目做过 Tauri 移动端，也没有组网代码。**
[ADR-013](./20-adr.md) 也说这是"技术风险最高的一块"。

因此 **M0 的第一周不是搭骨架，而是把架构假设证伪**。

> ⚠️ **业主已确认：当前没有 macOS 环境**（[23 R-14](./22-risks.md)）。
> 因此下表的 **iOS 相关项（spike 3、10）推迟**，Android 优先。
> 这不是"跳过"，而是**调整顺序**：把 iOS 的风险留到 M5 之前用一个专门的 spike 周期处理，
> 但**必须**在那之前完成 Android 版本的等价验证——它能覆盖"移动端能否承载内核"这个问题的大部分。

| #  | Spike                                    | 最小验证                                                        | 失败的话                        | 工时   | 平台     |
| -- | ---------------------------------------- | --------------------------------------------------------------- | ------------------------------- | ------ | -------- |
| 1  | Tauri 2 移动端能跑 tokio                  | `cargo tauri android dev` 起最小 App，内部 spawn tokio task      | 移动端改用 wry/tao 直连，或内核拆分 | 1–2 天 | Android |
| 2  | Tauri 2 移动端能跑一个 wasm 实例            | 在 spike 1 的基础上加一次 wasmtime 实例化 + 调 host function      | 移动端 headless 插件不成立        | 1 天   | Android |
| 3  | `wasmtime` 交叉编译到 Android aarch64      | `cargo check --target aarch64-linux-android`（依赖 wasmtime）     | 移动端改用 `wasmi`              | 0.5 天 | Android |
| 4  | `rusqlite bundled` 交叉编译到 Android      | 最小 SQLite 项目交叉编译                                          | 改用 `redb`（纯 Rust）           | 0.5 天 | Android |
| 5  | Tauri 多 WebView 独立 origin              | 桌面起两个 WebView，各自 `synra-plugin://` 协议 + 不同 origin，验证跨 origin 访问被阻止 | UI 插件退化为 sandbox iframe | 1–2 天 | 桌面     |
| 6  | 自定义协议 + CSP 能阻止 UI 插件网络请求      | 一个页面尝试 `fetch('http://evil.com')`，断言被 CSP 拦            | 需要在协议层再加一层代理           | 0.5 天 | 桌面     |
| 8  | Wasm host call 延迟在多插件并发下可接受      | 10 个 wasm 实例并发调 host function，测 P50/P99                   | 需要批量化接口或降低审计开销        | 1 天   | 桌面     |
| —  | **推迟到 M4 之前**：                        |                                                                 |                                 |        |          |
| 9  | 移动端包体积实测（wasmtime 裁剪后）          | 出 Android release apk，量体积                                    | 切 `wasmi` 或按需下载运行时        | 1 天   | Android |
| 10 | `wasmtime` 交叉编译到 iOS + Tauri iOS 工程  | 需 macOS；与 spike 1–3 等价                                        | iOS 不支持 headless 插件（能力分级） | 2 天   | iOS      |

**M0 内的 8 个 spike 总计约 6.5–8.5 人日**（不含推迟的 9、10）。
这笔投入换的是"不会在 M5 发现移动端根本走不通"。

> **判据**：spike 只写**最小代码**（独立小 crate，**不进 workspace**），
> 目的是**证伪**而不是"做出能用的东西"。每个 spike 的产出是一段结论 + 一段可复现的命令。
> 结果写进 `synra-core/spike/RESULTS.md`，并回填 [22 §4](./22-risks.md)。

```
synra-core/spike/            # 不进 workspace，用完可删（或归档）
├─ RESULTS.md                     # ★ 每行：假设 / 结论 / 证伪命令
├─ 01-tauri-android-tokio/
├─ 02-tauri-android-wasm/
├─ 03-wasmtime-android/
├─ 04-rusqlite-android/
├─ 05-tauri-multi-webview-origin/
├─ 06-csp-blocks-fetch/
├─ 07-protocol-conformance/
└─ 08-host-call-latency/
```

---

## M0 · 骨架与验收工具（1–2 周）

### 目标
建立 workspace、CI、以及"内核可 headless"的**自动化验收**。
**在此之前先跑 §0.1 的 8 个技术验证 spike。**

### 交付物

| 交付物                                    | 说明                                                    |
| ----------------------------------------- | ------------------------------------------------------- |
| **8 个 spike 的结论**（§0.1）              | ★ 先做这个；任一失败都会改架构                             |
| `Cargo.toml` workspace + 11 个空 crate      | [06 §1](./06-workspace-layout.md) 的结构                 |
| `[workspace.lints]` 严格档 + `panic = "unwind"` | [06 §7](./06-workspace-layout.md)、[ADR-014](./20-adr.md) |
| `.cargo/config.toml`（移动端 target 参数）   | [06 §7](./06-workspace-layout.md)                        |
| `tools/guards`（Rust 二进制）                  | ★ **一组结构守卫**（条数不写在这里 —— 它一直在长）；不是 `justfile` 也不是 `xtask/guards.sh` ——
  它们从未存在，而当初这一行就是这么写的。运行方式是 `cargo run -q -p guards` |
| CI（`checks` + `msrv` + feature 矩阵）       | **固定 `dtolnay/rust-toolchain` 版本**（[ADR-014](./20-adr.md)）；MSRV job 跑在 checks 之前 |
| `crates/synra-protocol` 的 ID/错误/版本类型  | newtype + `SynraError` + 稳定错误码                       |
| `crates/synra-observe`                      | `tracing` 初始化 + redaction layer（最小）                |
| `crates/synra-platform` 的 `HeadlessPlatform` | 所有能力返回 `Unsupported`                              |
| `apps/synra-cli`                            | `synra status` / `synra doctor`（输出环境信息）            |
| **`xtask check-matrix`**                    | ★ 跑 [06 §6](./06-workspace-layout.md) 的 feature 矩阵     |
| `src/`（app 仓库） 骨架                            | Vite + Vue + 假内核（静态 snapshot），能渲染四页壳层        |
| `cargo install harbor cargo-bumpp`（文档）   | 发布工具就位（[ADR-015](./20-adr.md)）                     |

### 验收标准

```bash
# 1. spike 全部有结论（不是"还在试"）
cat spike/RESULTS.md     # 8 行，每行：假设 / 结论 / 证伪命令

# 2. 内核可无 UI 编译并运行
cargo run -p synra-cli -- status
# → 输出：版本、平台、数据目录、监听端口（未启动）、插件数 0

# 3. 内核不依赖 GUI
cargo tree -p synra-core -p synra-net -p synra-plugin-host | grep -E 'tauri|wry|tao'   # 必须为空
cargo tree -p synra-protocol | grep -E 'tokio|rusqlite|wasmtime'                        # 必须为空
cargo deny check bans                                                                   # 必须通过

# 4. 结构守卫
cargo run -q -p guards   # 全过才有退出码 0

# 5. feature 矩阵 + MSRV 全绿
just matrix
just msrv          # 用 Cargo.toml 的 rust-version 真编一遍

# 6. 前端骨架能跑
just ui-dev        # 浏览器里看到四页壳层

# 7. 全部 lint/测试通过
just check
```

**这一阶段的产出价值**：证明"内核边界干净"这件事**从第一天起就可验证**，而不是等到最后才发现内核依赖了 UI。

---

## M1 · 协议与传输内核（3–4 周）

### 目标
两个 Rust 进程能：发现彼此 → 握手 → 配对 → 互发消息 → 心跳保持 → 断线重连。
并且**协议行为在两个独立实现里完全一致**。

### 交付物

| 交付物                                                | 对应文档                                |
| ----------------------------------------------------- | --------------------------------------- |
| `synra-protocol`：帧、信封、事件、发现报文、能力 DSL      | [03](./04-wire-protocol.md)、[09](./04-wire-protocol.md) |
| `synra-capability`：解析 + 匹配 + 授权集 + proptest       | [10 §2–§3](./10-capability-and-sandbox.md) |
| `synra-net`：UDP 发现 + mDNS + TCP 传输 + 心跳 + 握手       | [04 §2–§4](./04-wire-protocol.md)、[04 §3–§4](./04-wire-protocol.md) |
| `synra-store`：SQLite + identity + 配对记录 | [17](./17-storage-and-config.md)       |
| `synra-core`：`Kernel` + `Identity`/`Discovery`/`Trust`/`Transport` 聚合根 | [07 §2–§4](./07-core-kernel.md) |
| `Supervisor`（受监督任务）+ `CancellationToken` 关停    | [07 §5](./07-core-kernel.md)      |
| `KernelEvent` + `broadcast` 订阅                        | [07 §8](./07-core-kernel.md)      |
| `synra-testkit`：`KernelPair` + 内存传输 + 虚拟时钟       | [07 §10](./07-core-kernel.md)     |
| CLI 子命令：`devices` / `pair` / `send` / `status`       | —                                       |
| **协议一致性 fixture 回放测试**                          | [04 §9](./04-wire-protocol.md)、[21 §3](./21-testing-strategy.md) |
| **协议一致性套件**（fixture 回放 + 双内核对称性）              | [21 §3](./21-testing-strategy.md)          |

### 验收标准

```bash
# 1. 双内核互通（同机两进程，模拟两台设备）
synra --data-dir /tmp/a --port 32100 dev &
synra --data-dir /tmp/b --port 32110 dev &
synra --data-dir /tmp/a devices        # → 发现 b（TrustLevel ≥ Handshaked）
synra --data-dir /tmp/a pair <b-id>    # → b 接受（CLI 交互）
synra --data-dir /tmp/a send --to <b-id> --text "hello"   # → b 收到

just interop-test
# → 核心 B 能发现 A、配对、互发消息、传文件，且双方对链路状态的认知一致

# 3. 协议 fixture 逐字节一致
cargo nextest run -p synra-protocol --test wire_fixtures
# → 所有 v1 fixture 的 encode/decode 逐字节一致

# 4. 能力匹配的 proptest 全绿（含 * vs ** 边界）

# 5. 断线重连
# 杀掉 b 进程 → a 在 3s/5s/10s 后重连 → 重启 b → a 自动恢复

# 6. 超长帧：发一个 300 KiB 帧 → 对端关链路并记录 FrameTooLarge（不是挂死）

# 7. 无 UI 依赖
cargo tree -p synra-net -p synra-core | grep -E 'tauri|wry'   # 空
```


---

## M2 · 意图路由与最小插件（3–4 周）

### 目标
一个 **Rust headless Wasm 插件**能：
1. 被安装（本地开发路径）；
2. 参与意图匹配（`can_handle`）；
3. 在被授予的能力内执行动作（`execute` → `platform.open_url`）；
4. 被 capability 闸门拦截越权调用。

### 交付物

| 交付物                                          | 对应文档                                  |
| ----------------------------------------------- | ----------------------------------------- |
| `wit/synra-plugin.wit` 完整定义                   | [04 §4](./09-plugin-system.md)、[15 §4](./15-plugin-sdk.md) |
| `tools/wit-codegen`                              | WIT → Rust 绑定生成                        |
| `synra-plugin-api`（manifest + 共享类型）          | [15 §2](./15-plugin-sdk.md)                |
| `synra-plugin-host`：Engine + 实例 + `gate.rs` + 9 组 host functions | [13](./12-wasm-host.md)、[10 §3](./10-capability-and-sandbox.md) |
| `synra-plugin-sdk`：9 namespace + `#[plugin]` 宏   | [15 §4](./15-plugin-sdk.md)                |
| `git_host_fn!` 宏 + CI 检查（所有 host fn 过闸门）  | [10 §3.1](./10-capability-and-sandbox.md)   |
| `Intent` 聚合根 + `dispatch_intent` 流程            | [07 §6](./07-core-kernel.md)         |
| `synra-plugin-host` 的最小生命周期（Local install → Activate → Active） | [14 §1–§2](./14-plugin-lifecycle.md) |
| `tools/synra-plugin-cli`：`new` / `build` / `dev`   | [15 §3](./15-plugin-sdk.md)                 |
| 示例插件 `synra-plugin-github`（识别 GitHub 链接并打开浏览器） | — |
| `synra-testkit::MockHost` + 插件契约测试            | [15 §5](./15-plugin-sdk.md)                 |

### 验收标准

```bash
# 1. 插件脚手架 + 构建
synra-plugin new github --template headless
cd github && synra-plugin build        # → dist/plugin.wasm
synra-plugin verify                    # → 全部通过（含 imports ⊆ capabilities）

# 2. 本地装载 + 意图匹配
synra plugin install --local ./github --trust-dev
synra plugin list                      # → github installed, active
synra share "imba97/smserialport"      # → 候选：github / open-url
# 接受 → 调用 platform.open_url → 浏览器打开

# 3. 闸门生效（★ 关键验收）
# 从插件的 capability 里移除 platform:open-url → 重新激活
synra share "imba97/smserialport"
# → 明确错误：capability denied: platform:open-url（不是静默失败）

# 4. 沙箱生效（★ 关键验收）
# 写一个恶意的测试插件：
#   · 死循环 → 被 fuel 中断（返回 FuelExhausted，内核不受影响）
#   · 分配 2GB 内存 → 被内存限制中断
#   · 调用未声明的 host function → 装载期就拒绝
#   · 尝试读 /etc/passwd → 沙箱拒绝
cargo nextest run -p synra-plugin-host --test malicious_plugins

# 5. 无副作用保证
# can_handle 里调用 platform.open_url → SDK 编译期报错（或契约测试失败）

# 6. 调用性能
cargo bench -p synra-plugin-host -- gate_call
# → host call 中位数 < 20μs（含闸门 + 审计）

# 7. 前端可接入（可选）
just ui-dev    # → 插件页显示 github 插件卡片；点开打开插件页
```

---

## M3 · 插件安全与分发（3–4 周）

### 目标
插件能**从 registry 安全安装**、能被**签名校验**、能**同步到另一台设备**、用户的授权是**显式且可审计**的。

### 交付物

| 交付物                                                | 对应文档                              |
| ----------------------------------------------------- | ------------------------------------- |
| `signature.json` 生成（CLI）+ 校验（宿主）              | [16 §3](./16-plugin-distribution.md)   |
| `PublisherTrustStore`（TOFU + 指纹绑定 + 撤销）         | [16 §3.3–§3.5](./16-plugin-distribution.md) |
| Registry 客户端（packument + 镜像优先级 + 校验失败不回退） | [16 §4](./16-plugin-distribution.md)  |
| 完整 `VerifyPipeline`（11 个步骤）                      | [14 §2.3](./14-plugin-lifecycle.md)    |
| 归档安全（大小/条目/路径穿越/解包炸弹）                   | [14 §2.3](./14-plugin-lifecycle.md)    |
| `AwaitingConsent` + `CapabilityRequest` + 差异展示       | [14 §2.4](./14-plugin-lifecycle.md)    |
| 完整生命周期状态机（13 态）+ `ActivationGuard` 回滚       | [14 §1–§2](./14-plugin-lifecycle.md)   |
| `plugin_audit` + 审计 UI 数据接口                       | [10 §7](./10-capability-and-sandbox.md)、[18 §5](./18-observability.md) |
| `plugin_ui_rpc` + 会话令牌 + 自定义协议 + CSP（UI 插件）   | [08 §7](./08-ipc-and-frontend.md)、[10 §6](./10-capability-and-sandbox.md) |
| 跨设备同步协议（9 步）+ `sync.result` + 冲突策略          | [16 §5](./16-plugin-distribution.md)   |
| 前端：插件页（安装/授权/审计/同步 UI）                    | [08 §5](./08-ipc-and-frontend.md)      |
| 安全测试套件                                           | [21 §5](./21-testing-strategy.md)      |

### 验收标准

```bash
# 1. 签名链路
synra-plugin keygen --out ./dev.key
synra-plugin sign --key ./dev.key
synra-plugin publish --registry http://localhost:4873
synra plugins install @synra-plugin/github
# → 显示发布者指纹 → 用户确认 → 校验签名 → 安装

# 2. 篡改检测（★ 关键验收）
# 修改 registry 上的 plugin.wasm 一个字节（不改 signature.json）
synra plugins install @synra-plugin/github
# → 拒绝：artifact digest mismatch

# 3. 权限变更重新确认
# 发布 v2，新增 network:fetch:api.github.com
synra plugins update @synra-plugin/github
# → 弹出"新增权限"确认（不是静默升级）

# 4. 跨设备同步
synra --data-dir /tmp/a plugins sync --to <b-id> --plugin github
# → b 收到 → 校验 → 安装；a 收到 sync.result

# 5. 同步的完整性（★ 关键验收）
# 在传输过程中篡改一个 chunk
# → 接收端 sha256 校验失败 → nack → 重传；最终结果正确

# 6. 原子性与回滚（★ 关键验收）
# 让 activate 失败（插件缺一个 artifact）
synra plugins enable github
# → 状态变为 Faulted（不是卡在 activating）→ 可以重试/停用/卸载

# 7. 恶意插件套件全绿
cargo nextest run --workspace --test security
#       SSRF（file:// / localhost / 云元数据）、路径逃逸（../ 与 Windows 短路径）

# 8. 审计可见
# 前端插件详情页显示：允许/拒绝的能力调用时间线
```

---

## M4 · 桌面应用完整（3–4 周）

### 目标
`synra-desktop` 是一个**完整可用的桌面应用**：四页 UI、设备管理、插件管理（含 UI 插件）、分享入口、文件传输、诊断页。

### 交付物

| 交付物                                        | 对应文档                            |
| --------------------------------------------- | ----------------------------------- |
| `synra-app`（Tauri 2 + Vue）            | [05 §2.1](./05-system-architecture.md)、[10](./08-ipc-and-frontend.md) |
| ~40 个 Tauri command                           | [08 §2.2](./08-ipc-and-frontend.md)  |
| `ts-rs` + `tauri-specta` 类型生成 + CI `types-check` | [08 §3](./08-ipc-and-frontend.md) |
| 前端：`src/`（app 仓库） 四页 + stores + event bridge   | [08 §5](./08-ipc-and-frontend.md)    |
| UI 插件承载（WebView/origin/CSP/令牌）            | [08 §7](./08-ipc-and-frontend.md)、[04 §5](./09-plugin-system.md) |
| `DesktopPlatform`（打开 URL/文件、剪贴板、通知、选区捕获） | [07 §3](./07-core-kernel.md)、[24 §3.2](./24-migration-map.md) |
| Tauri 集成：托盘、单实例、深链（`synra://`）、文件关联、开机自启 | [05 §3.1](./05-system-architecture.md) |
| 系统分享入口（Windows 分享目标 / macOS Share Extension / 拖拽 / 剪切板） | [02 §2–§3](./02-domain-and-flows.md) |
| 诊断包导出 + `probe_*` 环境诊断                  | [18 §6](./18-observability.md)        |
| 指标面板 + 日志查看                                | [18 §4](./18-observability.md)        |
| 打包（msi/nsis / dmg / AppImage+deb）+ 自动更新     | —                                   |
| 前端行为回归（四页、事件桥接、未知事件兜底）            | [21 §7](./21-testing-strategy.md)     |

### 验收标准

```bash
# 1. 桌面 App 启动
just desktop-dev
# → 四页可见；Devices 页显示扫描到的设备；Plugins 页显示已装插件

# 2. 端到端：手机分享 → 桌面执行（可用第二台机器或模拟）
# 手机（或 CLI 模拟）分享 "https://github.com/imba97/smserialport"
# → 手机弹确认 → 确认 → 桌面浏览器打开

# 3. UI 插件隔离（★ 关键验收）
# 故意写一个恶意 UI 插件：
#   · 尝试 fetch('http://evil.com') → 被 CSP 的 connect-src 'none' 阻止
#   · 尝试访问宿主 DOM（window.parent.document）→ 跨 origin 被阻止
#   · 尝试用别的插件的令牌 → IPC 拒绝
#   · 尝试调 PluginInstall → 方法白名单拒绝

# 4. 首屏性能
# 冷启动到可交互（时间到第一个有意义绘制）
# → 目标 < 500ms（进程启动 + WebView 首次渲染）

# 5. 诊断闭环
# 拔网线 → App 显示"网络不可用" → 导出诊断包 → 里面能看到 probe 结果与日志

# 6. 崩溃安全模式
# 让一个插件导致 panic → 重启 → 进入安全模式（插件全部停用）→ 提示用户

# 7. 打包产物能装能跑
just desktop-build
```

---

## M5 · 移动端（3–4 周）

### 目标
Android 与 iOS 可用，且**插件代码一份到处跑**（headless Wasm 在移动端也执行）。

### 交付物

| 交付物                                    | 对应文档                            |
| ----------------------------------------- | ----------------------------------- |
| `synra-app`（Tauri 2 mobile）       | [05 §2.2](./05-system-architecture.md) |
| Rust 内核编译为 `.so` / `.a`（Android / iOS） | ADR-005                              |
| **AOT/解释器 Wasm 后端**（iOS 无 JIT）       | [12 §2](./12-wasm-host.md)            |
| `AndroidPlatform` / `IosPlatform`            | [07 §3.1](./07-core-kernel.md) |
| 移动端生命周期（前后台、Doze、通知）           | [23 风险 R-03](./22-risks.md)          |
| 系统分享入口（Android Intent / iOS Share Extension） | [02 §2](./02-domain-and-flows.md) |
| 移动端授权 UI（通知权限、文件访问）             | —                                   |
| 移动端构建（Gradle / Xcode）接入 `just`        | —                                   |
| 真机验证（Android：至少一台；iOS：至少一台）      | —                                   |

### 验收标准

```bash
# 1. 编译
just mobile-build-android          # → apk 可安装
just mobile-build-ios              # → 需 macOS + Xcode

# 2. 移动端跑 headless 插件（★ 关键验收）
# 在 Android 上装一个 headless 插件 → 用 share 触发 → 执行成功
# → 证明 Wasm 后端在移动端工作

# 3. 跨端同步（★ 关键验收）
# 桌面装插件 → 手机收到通知 → 一键同步 → 手机可用

# 4. 前后台
# 手机切后台 5 分钟 → 回前台 → 重连成功（不等 25s 心跳超时）
# → 说明 app-foreground 触发了重连

# 5. 与另一台真机共存验证（双向分享 + 同时双向传输）

# 6. 包体积
# apk 大小目标；wasmtime 的移动端裁剪是否达标
```

---

## M6 · 生态与优化（持续）

| 项                          | 说明                                              | 优先级 |
| --------------------------- | ------------------------------------------------- | ------ |
| 插件自动更新策略             | [16 §6](./16-plugin-distribution.md)               | 高     |
| Catalog 搜索与展示           | [16 §4.3](./16-plugin-distribution.md)             | 高     |
| 跨设备 trace 展示            | [18 §3.4](./18-observability.md)                   | 中     |
| 性能优化（二进制帧、滑动窗口） | [04 §6.3](./04-wire-protocol.md)                   | 中     |
| 传输断点续传                  | [14 §2](./14-plugin-lifecycle.md)（`transfer_sessions`） | 中  |
| 意图离线排队                  | [07 §6.3](./07-core-kernel.md)              | 中     |
| 插件评分/评论（自建 catalog）  | [16 §4.3](./16-plugin-distribution.md)             | 低     |
| **集群 / 单主机中继 / 选主**   | 见 §7                                             | 低     |
| OpenTelemetry 导出            | [18 §4.2](./18-observability.md)                   | 低     |
| 插件开发者门户                | —                                                 | 低     |

---

## 7. 关于集群 / 选主（明确的范围决策）


**本路线图的决策**：

| 选项                                   | 评价                                                          |
| -------------------------------------- | ------------------------------------------------------------- |
| A. 不做（保持对等直连）                  | ✅ **推荐**。当前产品的核心场景（分享→远端执行、剪切板、文件）**不需要**主机中继 |
| B. 做中心化主机（指定一台"主机"，其他连它） | 中等复杂度；能解决"N×N 连接爆炸"；但引入单点故障               |
| C. 做完整选主（Raft 风格）               | ❌ 复杂度极高；对等网络下的选主本身就是难题；收益不明           |

**决定**：
- **M0–M6 不做集群**（明确写在文档里，避免"文档承诺、代码没有"的老问题重演）；
- 保留 `synra-cluster` crate 的 feature 占位（`cargo check --features cluster` 必须能过，但内部返 `Unsupported`）；
- 如果将来需要（例如 N > 10 设备的规模），**重新设计**：优先考虑 B（指定主机 + 优雅降级），不做 C；
- **任何"集群"相关的事件/字段在协议里必须标 `#[cfg(feature = "cluster")]`**，不允许再出现幽灵字段。

---

## 8. 里程碑依赖与关键路径

```
M0 ──▶ M1 ──▶ M2 ──▶ M3
       │      │      │
       │      │      └──▶ M4 ──▶ M5
       │      │
       └──────┴──────────────▶ M6（与 M4/M5 并行）

关键路径：M0 → M1 → M2 → M3 → M4 → M5

可以并行的工作：
  · UI 骨架（M0 起就做，用假内核）
  · 平台能力（Tauri 集成、os-selection、通知、托盘）可以在 M1 期间做
  · 前端页面切换（M3 期间做设备页与插件页）
  · 移动端环境搭建（Tauri mobile 工具链、真机、签名）可以在 M2 期间准备
```

### 并行度建议

| 角色            | M0–M1                     | M2–M3                        | M4–M5                  |
| --------------- | ------------------------- | ---------------------------- | ---------------------- |
| Rust 内核        | 协议 + 传输 + 存储          | 意图 + 插件宿主 + 沙箱         | 平台能力 + 集成          |
| Rust 工具链      | workspace + CI            | WIT codegen + 插件 CLI        | 打包                    |
| 前端            | 骨架 + 假内核 + 组件迁移     | 设备页 + 插件页切换            | UI 插件承载 + 诊断页     |
| 移动端          | —                          | 环境准备 + 交叉编译验证         | 移动端完整              |

---
## 8.6 剩余工作的切片（2026-10-18 评估，代替"每轮试探一次"）

里程碑 M0–M3 的主体与 M4 的一部分已经落地（见 [08 §9](./08-ipc-and-frontend.md) 的逐条状态）。剩下三件是**子系统**，
不是接线；下面是按依赖与可验证性排的切片，写在这里是为了让下一轮直接选一块开工，而不是先花一轮探测。

| 工作 | 前置 | 估计 | 可验证性 |
| --- | --- | --- | --- |
| **配置事件**（`ConfigUpdated`） | 无 | 1 轮 | 完全可验证：wire 变体 + TS 类型 + `synra-js` 的 `dto-coverage` 守卫会盯着 |
| **插件 UI 宿主** | 无（`plugin_ui_*` 三条命令、`plugin-ui` crate 已有 `check_path`/`content_type_for`） | 3–5 轮 | 部分：origin 注册与令牌签发可测；WebView 创建只能在 `tauri-shell` 下编译验证 |
| **传输子系统** | `synra-net` + 04 章的帧 + 会话 | ~~6–10 轮~~ → **1–3 轮**（见下） | 部分：帧与状态机可测；真实收发要靠 `synra-net` 既有测试 |

★ **2026-10-18 更正**：这一行原来写"6–10 轮，前置是 `synra-net`"，依据是 `synra-net` 里只有 `backoff`/`liveness`/
`write_queue`。**那个依据是错的** —— `synra-core` 里早就有整套传输：
`link.rs`（`TcpStream` 的 `LinkStream` 实现 + localhost 双端的测试）、`listener.rs`、`socket.rs`、`network.rs`、
`messaging.rs`、`routing.rs`、`transfer/{sender,receiver}.rs`、`plugin_sync.rs`。帧也在 `synra-protocol/src/frame.rs`
（`FRAME_HEADER_LEN`、`MAX_FRAME_LEN`、`encoded_len`）。
真正缺的只有**壳**：`synra-rust-app` 至今没有 listener/link（`with_discovery_or_report` 只起了 UDP 发现），
所以 `transfer_*` 四条命令没有东西可接。那不是"做一个子系统"，是"像发现那样起一个 worker 并接上四条命令"。

**配置事件**是最便宜的一件，也是唯一"完全可验证"的一件：加一个 `ConfigUpdated(Box<ConfigSnapshot>)` 变体、在
`config_set` 之后发布、`ui.ts` 与 `index.ts` 各加一处、前端 store 加一个 handler。它同时让设置页不必靠重读。

**插件 UI 宿主**的正确起点不是 WebView，而是 `synra-plugin-ui` 里"这个 URL 允许不允许"与"令牌怎么签发"——那两件事
是纯逻辑，可以像 `consent` 一样先在内核里做成有测试的函数，再由壳去用它。WebView 的创建留给最后的壳层适配。

★ **零件的名字（2026-10-18 查过，免得下一轮再找一遍）**：`synra_core::transfer` 公开
`TransferSender`、`TransferReceiver`、`plan_blocks`、`Block`、`ObjectDigest`、`DEFAULT_CHUNK_SIZE`/`MAX_CHUNK_SIZE`、
`TransferError`；`synra-protocol::frame` 提供帧常量与 `encoded_len`；`synra-core/src/link.rs` 的 `LinkStream` 已为
`TcpStream` 实现，并在 localhost 双端有测试。**壳侧缺的是一个 `AppState.transfers`（`TransferId → TransferSnapshot`）
与一个像 `discovery` 那样起 worker 的地方** —— worker 用 `TransferSender`/`Receiver` 推进，每次状态变化发布
`TransferUpdated`（前端 store 与事件类型都已就绪），`transfer_list` 从那张表答。
★ **2026-10-18：前两件已落地。** `commands::transfers::Transfers`（`record`/`forget`/`list`，替换而非合并、按 id 排序，
三条测试）、`AppState.transfers`、`transfer_list` 命令并注册（注册数 25 → 26）。
**一处与设计签名不同**：`08 §2` 把它写成 `Result<Vec<TransferSnapshot>, IpcError>`，实现返回 `Vec<TransferSnapshot>` ——
一个"列出我持有的东西"的读不可能失败，用 `Result` 只为形状统一会让调用方写一段永远走不到的 `if (!ok)`。
按"文档是权威、但偏离要写明"的规矩记在这里；若要统一，改的应是文档而不是实现。
剩下的仍是 worker：它需要 `LinkStream` + `TransferSender`/`Receiver`，并在每次状态变化发布 `TransferUpdated`。
★ **worker 要调用的方法签名（2026-10-18 查过，`crates/synra-core/src/transfer/`）**：`TransferSender::open(path, chunk_size)` 或 `open_bundle(..)` → `request()` 得到 `TransferRequest` → `blocks()` /
`read_block(index)` / `mark_sent(index)` / `has_sent` / `adopt_acknowledged(&ProgressBitmap)` / `is_complete()` /
`outstanding()`；接收侧 `TransferReceiver::accept(&request, destination)` → `absorb(index, data) -> Absorbed` /
`received_count()` / `path()` / `finish(&SriDigest) -> ObjectDigest`。
也就是说两端都是**逐块推进**的：worker 从链路拿到一帧就调一次 `absorb` 或 `read_block`+`mark_sent`，然后把进度交给
`announce_transfer`（已有）。壳侧不再需要任何新的状态机。
★ **链路的形状（同一轮查的，`crates/synra-core/src/link.rs`）**：`link.receive_inbound()` 返回 `Inbound`，两种形态 ——
`Envelope(Box<Envelope>)`（控制面）与 `Block { chunk_id, data: Bytes }`（文件块）；发送侧是 `send(&Envelope)` 与
`send_chunk(chunk_id, data)`，另有 `ping()`/`pong()` 与读写超时（`set_read_deadline`/`set_write_deadline`，
`Link` 上还有 `write_deadline()` 与 `supports(Feature)`）。
**所以 worker 的主循环就是**：`receive_inbound()` → `Block { chunk_id, data }` 交 `receiver.absorb` / 或从 `sender`
★ **`TransferRequest`（`synra-protocol/src/session.rs:506`）的字段**：`transfer_id`、`kind: TransferKind`、
`byte_length: u64`、`chunk_size: u32`、`digest: SriDigest`、`sync_session_id: Option<SessionId>`、
`meta: TransferMeta`（标了 `#[serde(flatten)]`，逐 kind 的元数据在里面）。
**它不含对端的名字或 id** —— 那两个只有在建链时才知道，所以 `TransferSnapshot` 的 `peer_device`/`peer_name`
必须由 worker 从链路一侧填，而不是从请求里读。这一点决定了 `snapshot_from` 的签名要收一个 `peer`，而不是只收请求。
`read_block(chunk_id)` 后 `send_chunk` → `announce_transfer`；`Envelope` 交现有的消息分派（`messaging.rs` 已有）。
壳里还缺的是**谁来建这个 link**（监听 + 拨号），而那正是 `discovery` 之外 `with_*_or_report` 还没写的那一个。
★ **2026-10-18 追加一个必须先回答的问题**：`serve` 要的那几样壳里**一个都没有** —— `Host`（`synra-core/src/host/kernel.rs`）、
`PairingDesk`、`Router` 都活在 **daemon** 里，壳从来没构造过它们（`host/mod.rs` 只对外给了 `Host`、`dial_peer`、
`SharedHost`/`shared`）。所以"壳里起传输"不是加一个 `with_*` 函数，而是先决定：
(a) 传输归 daemon，壳通过某种 IPC 拿进度（今天**没有**壳↔daemon 的通道）；
(b) 把 `Host`/`Desk`/`Router` 的构造搬进 core 并让壳也用一份（那是 daemon 的编排，搬迁面很大）；
(c) 壳只做"发送"这一侧（`dial_peer` 已导出），接收留在 daemon。
**这三条是设计决定，不是实现细节**；在回答之前动手，就等于在壳里重建一遍 daemon 的编排。
`snapshot_from`、`announce_transfer`、注册表、`transfer_cancel` 都已就绪，无论选哪条都不浪费。
★ **2026-10-18 决定走 (c)，并且查证过它可行**：`Host::new(identity: Identity, local: LocalPeer)`（`host/kernel.rs:53`）
是可构造的 —— 壳里已有 identity（`identity_get` 就是读它）与设备标识，所以"没有 Host"这个理由不成立；
而 `dial_peer(ours: &LocalPeer, flags: Flags, address: SocketAddr, token: Option<LinkToken>, now_ms: u64) -> Dialled`
（`host/connections.rs`）**根本不需要 Host**。于是壳侧发送只需：从已配对设备取地址 → `dial_peer` → `TransferSender::open`
→ `send_chunk` 循环 → `announce_transfer`；接收仍归 daemon，等壳↔daemon 通道具备再说。
★ **2026-10-18 查 `send_text` 时发现的第二处设计缺口**：传输引擎**没有"文本"这种东西** —— `TransferSender` 只有
`open(path, chunk_size)` 与 `open_bundle(..)`，而 `TransferMeta` 只有 `PluginBundle` 与 `Attachment`。
所以 `transfer_send_text` 必须是"把文本写成临时文件、按 `Attachment` 发、发完删掉"，而临时文件的名字、清理时机
（成功/取消/失败三种）与 `file_name` 展示规则都是要定下来的东西，不是一行代码。
★ **2026-10-18 决定这些规则**（写清楚再动手，因为每一步都有一个容易忽略的约束）：
1. **文件放哪**：`paths.tmp`（或数据目录下的 `tmp/`）里的 `<transfer_id>.txt`。**不建子目录** —— 清理时只需删一个文件，
   而删目录要 `remove_dir_all`，那是守卫只允许沙箱调用的操作（本会话已经在 installer 那里撞过一次）。
   ★ **2026-10-18 实现时改了两处，都是往更安全的方向**：位置定在 **`cache_dir()/outbox/`**（所以**确实建了一个子目录** ——
   `create_dir_all` 与 `remove_dir_all` 不是一回事，建目录不受限，删目录才受限）；名字是 **`message-<unix_ms>.txt`**。
   理由：系统临时目录对机器上每个进程可读，而这个文件装的是人打的字 —— "会被删掉"与"存在期间谁都能读"是两件事。
   测试相应改成断言 **outbox 目录不存在**（拒绝发生在写文件之前），比原先数 `env::temp_dir()` 里文件个数更准，也不受别的进程干扰。

### 8.7 三个待决问题（2026-10-18 收尾时列，每个都是"先决定再写"）

传输的**发送侧已经无缺口**（四条命令、后台执行、失败经事件、临时文件私有目录且取消即删、9 条测试）。
剩下三件事都不是实现难度问题，而是各自缺一个决定：

1. **接收侧**：`listener::serve(host, cancel, data_dir, desk, router, on_finished)` 需要 `Host`、`PairingDesk`、`Router`，
   三者都在 daemon 里。**要决定的是**：壳与 daemon 之间是否有一条通道（今天没有），还是把编排搬进 core 让两边共用。
   在此之前壳无法接收入站连接 —— 这不是"还没写"，是"没有地方放"。
   ★★★★ **已决定（2026-10-18）：壳必须自己接收，不能委托 daemon。** 依据不是新判断，而是 `discovery.rs:3` 里早就写下的
   同一套理由：**壳与 daemon 是两个宿主、各自的数据目录**（壳是 `%APPDATA%\com.synra.app`，daemon 是 `SYNRA_DATA_DIR`），
   所以 daemon 收到的连接写进的是**它自己的**表，而界面读的是这个进程的。委托的结果就是"对端连上了、计划却看不见"，
   与当年"发现到三台设备却一台都不显示"是同一种失败。
   于是缺的不是"通道"，而是**让壳能构造 `Host`/`PairingDesk`/`Router`**（`Host::new(identity, LocalPeer)` 已可构造，
   第 158 轮查过；缺的是 desk 与 router 的构造，它们现在只活在 daemon 的启动代码里）。
   最小可行切片：把 `PairingDesk`/`Router` 的构造搬进 core（不搬 daemon 的启动策略），壳照 `discovery` 的形状起一个
   listener 线程，`on_finished` 里写 `sync_records` —— 那也顺带修好上一段说的"表没有写入方"。
   ★★★★ **两个零件的真实成本（2026-10-18 查 `apps/synra-daemon/src/serve.rs`）**：
   `PairingDesk::new(local_id, SessionTimeouts::default().pairing)` 一行即可，**它本来就在 core**；
   而 `Router` 是 **`crate::runtime::intent::Router::new(Arc<plugins>, Arc<board>)` —— 它住在 daemon 里**，
   不在 core。所以这条切片真正的第一步是**把 Router 搬进 core**（连同它依赖的 `plugins`/`board` 两个 `Arc`），
   那是跨 crate 的重构，不是"传两个参数"。`Host` 同理已可构造（`Host::new(identity, LocalPeer)`）。
   **附带一条旁证**：daemon 的 `serve.rs` 里有一段注释说"`insert_plugin` 从来没有生产调用方，所以 `db.plugins()` 永远为空"
   —— 与本会话发现的 `sync_records` / `plugin_versions` 是**同一个模式**的第三次出现：表与函数都在，**中间的调用没人写**。
   ★★★★ **再量一次，`Router` 的搬迁是 660 行（2026-10-18）**：`apps/synra-daemon/src/runtime/intent.rs` 660 行，
   依赖 `synra_control::Refusal`、`synra_core::dispatch::orchestrate`、`synra_core::dispatch::PluginSet`、
   `synra_core::intent::Board`，测试里还依赖 `daemon::runtime::plugin_host::KernelHost` —— **最后这一条决定了不能只搬一个文件**。
   ★ **改建议（这是我的决定）**：不搬 `Router`，而是**让 `listener::serve` 收一个 trait**（例如 `trait SharedInbound`，
   只有一个"收到一个 share/intent 时怎么路由"的方法），`Router` 实现它，壳传一个极简实现（收下就丢弃或只记审计）。
   理由：壳今天**没有**意图编排的需求（它不跑插件），为了一个用不到的能力搬 660 行是把 daemon 的复杂度复制到壳里；
   而 trait 只多一个方法，改动面在 `listener.rs` 与 `serve` 的调用点。**代价**：`serve` 的签名变一次，
   daemon 与壳各改一处 —— 比搬 660 行小得多。
   **如果将来壳真的需要意图编排**（比如它要自己跑插件），再搬不迟；那时 `Router` 已经是一个 trait 的实现，搬的是实现不是接口。
   ★★★★★ **上面两条都作废（2026-10-18，本会话最重要的一次更正）**：`listener::serve` 收的**不是** daemon 那个 660 行的
   `runtime::intent::Router`，而是 **`synra_core::dispatch::orchestrate::Router`** —— 它只有两个字段，构造函数是
   `Router::new(Arc<Mutex<PluginSet>>, Arc<Mutex<Board>>)`（`orchestrate.rs:301`），而**壳里两样都已经有了**：
   `AppState.board: Arc<Mutex<Board>>`（`state.rs:57`）与 `AppState.plugins: Option<Arc<Mutex<PluginSet>>>`（`state.rs:59`）。
   也就是说接收侧需要的三样 —— `Host::new(identity, LocalPeer)`、`PairingDesk::new(id, timeout)`、
   `Router::new(plugins, board)` —— **全部可以在壳里就地构造，一个 trait 都不需要，660 行一行也不用搬**。
   我前两轮把 daemon 的 `Router` 和 core 的 `Router` 当成了同一个东西（名字相同、职责相关、但类型不同），
   于是把一个"三行构造"的活儿估成了"跨 crate 重构"。
   **修正后的切片**：壳照 `discovery` 的形状起 listener 线程 → `serve(host, cancel, data_dir, desk, &router, &on_finished)`
   → `on_finished` 里写 `sync_records`。`plugins` 是 `Option`（壳可以没有插件宿主），为 `None` 时用一个空的 `PluginSet` 构造 Router 即可。
   ★★★★★ **接收侧已经实现（2026-10-18，同一会话）**：`src-tauri/src/listener.rs` 的 `spawn(app)` 做完全部四件事 ——
   `Host::new(identity, local_peer)`、`listener::bind(&mut host, 0)`、`PairingDesk::new` + `Router::new(state.plugins, state.board)`、
   起线程跑 `serve`。`shell.rs` 的 `setup` 在 `manage(state)` 之后调它并 `manage` 返回的取消令牌。
   `on_finished` 现在是真回调：只对 `bundle.is_some()` 的传输调 `record_sync(plugin, peer, version, digest, now)` ——
   **`sync_records` 因此第一次有了生产写入方**，同步计划不再是空表上的猜测。
   两条刻意的边界写在模块文档里：(1) **没有插件宿主就不监听**（`PluginSet::new` 拒绝在没有 host 时存在，
   而一个接了 share 却路由不到任何地方的监听器是"答应了却什么也不做"）；(2) **令牌今天没人调用** ——
   它是给进程内停止准备的，退出进程会自己释放端口，注释里写明了这一点而不是假装有关窗取消。
   **剩下的**：两个进程互发的端到端测试（今天只有构造与编译验证）。
   ★ **那条测试的做法（2026-10-18 预留配方，下一轮照此写）**：不需要两个进程 —— `serve` 的 peer 是一个真实的
   `TcpStream`，所以一个进程内就能起两端：
   1. `let mut host = Host::new(identity, local_peer)`，`listener::bind(&mut host, 0)` 拿到端口；
   2. `let cancel = Cancellation::new()`，起一个线程跑 `serve(&Arc::new(Mutex::new(host)), &cancel, &data_dir, &desk, &router, &finished)`；
   3. 主线程 `dial_peer(&their_local_peer, Flags::production(), addr, Some(token), now)` → `into_link` →
      `send_blocks(...)` 推一个 bundle；
   4. 断言 `finished` 回调被调用过，且 `db.sync_records()` 里出现了对端那一行。
   **难点只有一个**：握手要双方同意（`PairingDesk` 的 token 与 `Flags`），所以这条测试实际上也在验证配对；
   如果配对太麻烦，退化版本是**只验证 `serve` 接受了连接并结束了传输**（`on_finished` 被调一次即成功），
   把 `sync_records` 的写入留给已有的单测（`record_sync` 本身有测试）。
2. **`plugins_sync_plan`**：策略函数 `decide(ours_version, ours_digest, theirs: &RemoteState, policy)`（`plugin_sync.rs:188`）
   与 DTO `SyncPlanDto { plugin_id, targets }`（`dto/plugin.rs:200`）都已存在。**要决定的是**：对端"已有什么"存在哪 ——
   (a) 建一张 `peer_plugins` 表，会话交换清单时写入；还是 (b) 只在会话内计算，命令不持久化。今天的 `RemoteState`
   只在 `SyncSession` 的生命周期里活着，所以 (b) 更省，但界面在会话之外就得不到计划。
3. **`transfer_send_text` 的成功路径测试**：需要把链路注入命令层（`MemoryStream::pair_with_capacity` 已证明可行，
   见 `send/tests.rs` 的 `the_blocks_arrive_over_a_real_link`）。**要决定的是**：注入点放在 `send_over` 的签名上
   （多一个 `Link` 参数，生产代码传真实链路），还是在测试里替换 `dial_peer`。
   ★ **已决定并已实现（2026-10-18）**：选第一种，而且切得比"多一个参数"更利落 —— `send_over` 只保留"查地址 + 拨号 +
   `into_link`"，之后的一切搬进 **`send_ready(state, to, &mut PeerLink, sender, row)`**。
   理由：`send_ready` 才是需要测试的一半，而"必须拨号的测试跑不起来"。测试
   `a_send_over_a_connected_link_reports_every_block_and_finishes` 用内存链路跑通了整条成功路径
   （逐块到达 + `TransferFinished` + 注册表清空）。副作用：`finish` 闭包提成了模块级 `finish_transfer`，
   因为"删行"与"通知"必须成对，闭包让它们在两个地方各写一遍。
   ★ **第 2 条也一并决定（同日）**：选 (b) 的变体 —— **按需计算**。所以 `plugins_sync_plan(device_id)` 的形状是
   拨号 → 交换清单 → `decide` → 返回 `SyncPlanDto`，会话结束即丢。**代价是每次计划要一次拨号**；好处是不建表、不过期、
   不需要"上次同步时对方有什么"这种会撒谎的状态。**第 1 条（接收侧）仍未决定** —— 它需要一条壳↔daemon 通道。
   ★★★ **2026-10-18 再更正一次，这次推翻的是上面那个决定的前提**：`sync_records` 表**早就存在**（迁移 2，`migrations.rs:121`：
   `plugin_id, peer_device, version, digest, synced_at_ms`，主键 `(plugin_id, peer_device)`），`SyncRow` 是它的记录类型。
   也就是说"上次同步时对方有什么"**本来就有地方存**，我上一句"不建表"里那个"不"是多此一举。
   **修正后的计划**：`plugins_sync_plan` 读 `sync_records` 得到每个对端的 `RemoteState`（`RemoteState::from_summary` 的兄弟
   `from_row` 即可），`online` 由壳填，**不拨号**。`plugin_list_request`/`response` 仍然有用 —— 它们是"在会话里问一次、
   然后把答复写进 sync_records"的那条路径，也就是让这张表有新鲜数据的手段。
   **教训**：我在决定"要不要建表"之前没有查 `migrations.rs`，而那是一个 `grep sync` 就能回答的问题。
   ★★ **命令本体还差的最后一块数据（2026-10-18 查清）**：`plan_for` 需要"本机这个插件的 version 与 digest"，而
   `PluginRow`（`plugins` 表）**只有 `plugin_id`、`enabled`、`trusted`、`health`、两个时间戳 —— 没有版本、没有 digest**。
   它们在 **`plugin_versions`**（`migrations.rs:74`：`plugin_id, version, artifact, digest, manifest, installed_ms, active`，
   主键 `(plugin_id, version)`，并有 `plugin_versions_active` 的**部分索引**指向当前生效的那一行）。
   所以命令的取数完整路径是：`plugins()` 拿 allowed 列表 → 对每个插件用 `plugin_versions` 的 active 行取 `version`/`digest`
   → `sync_records()` 拿每个对端的状态 → `plan_for`。**下一个动作是查 `plugin_versions` 的读函数名**（`sqlite/` 下），
   然后这条命令就没有未知了。
   ★★★ **查了，结论是：`plugin_versions` 没有读者（2026-10-18）**。全仓 `plugin_versions` 的出现只有五处：
   建表、建索引、目录名、一条测试的 INSERT、以及 `plugins.rs` 里一句注释。`sqlite/` 下**没有任何函数从它读** ——
   写入在 daemon 的安装路径里，而读取从来没被需要过。
   所以命令本体的**真正第一步是给 store 加一个读函数**（`active_plugin_version(plugin_id) -> Option<(version, digest)>`，
   走 `plugin_versions_active` 那个部分索引），配一条测试（`sqlite/tests.rs:149` 已有 INSERT 的先例可仿）。
   **这一幕在本会话里出现了第五次**："接线"的对面缺一块基础能力，而它藏在一个没人读过的表后面。
   五个待补的基础能力分别是：`server` 端接收、`RemoteState` 的来源、`plugin_versions` 的读、监听循环、壳↔daemon 通道。
   ★★★★ **`plugins_sync_plan` 已经做完（2026-10-18，同一会话）**：`active_plugin_version` 落在
   `sqlite/versions.rs`（走部分索引，2 条测试）；`plugins_sync_plan` 落在壳的 `commands/sync_plan.rs`
   （`plugins()`/`active_plugin_version`/`sync_records()` → `plan_for`，**不拨号**），并已注册（壳 30 条命令）。
   `plan_for` 有 4 条测试、`RemoteState::from_summary` 3 条、请求/应答 3 条、协议类型 3 条。
   **这条线剩下的是"让 `sync_records` 有新鲜数据"** —— 那需要一次会话真的交换清单，而会话需要链路与
   `PairingDesk`/`Router`，也就是第 1 条那个未决定的架构问题。**所以 `plugins_sync_plan` 现在返回的计划基于"上次同步时的
   对端状态"，这必须在界面上说清楚**（`online` 只表示"能拨"，不表示"刚问过"）。
   ★★★★ **再查一层，结论比上面更严重（2026-10-18）**：`sync_records` **在生产里没有任何写入方** —— `record_sync`
   只被 `plugin_sync_tests.rs` 调用，生产路径用的是内存里的 `SyncTable`（`plugin_sync.rs:987` 的注释明确说这两者是
   "内存答案"与"持久答案"，但**中间那一行写入从来没有过**）。
   所以 `plugins_sync_plan` 今天会报告"每个对端都没有这个插件"，于是计划说每个都要发。
   **这错在安全的一侧**（多传一次，不是漏传一次），但它让这个功能看起来完整而实际基于空表 —— 已写进函数的文档注释，
   因为读那个函数的人正好在问"我该信这个计划吗"。
   **优先级的连带修正**：第 1 条（接收侧）不只是"让壳能收东西"，它决定**这张表有没有数据**，也就决定同步计划是否可信。
   它从"可选"变成"这条线的前置"。
   ★★ **2026-10-18 更正那句话里的一个错误**：我原先写"`SyncSession` 本来就要一条链路才能开"，暗示计划走 `SyncSession`。
   **不对** —— `SyncSession::open(session, plugin_id, source, transfer, declared_version, declared_digest, total_chunks)`
   （`plugin_sync.rs:434`）是**接收一个包**时逐块装配的状态机（`have: BTreeSet<u32>`、`duplicates`），与"比较双方有什么"
   毫无关系。计划要的是 `RemoteState` + `decide`，而 `RemoteState` 得从对端**交换**得来 —— 那条请求/应答在
   `messaging.rs` 的 envelope 层，还没查。**所以下一轮的第一个动作是查 envelope：有没有"问对方插件清单"的消息类型**，
   而不是从 `SyncSession` 开始。
   ★★ **查过了（2026-10-18，同一会话）**：`EventName` 是**语法检查而不是目录**（`envelope.rs:794` 附近的测试明确断言
   `device.something-new` 合法），所以线上**没有**一个"插件清单"事件，也没有任何地方在**处理**它。于是
   `plugins_sync_plan` 的前置不是壳里的 wrapper，而是**core 里一对新的请求/应答**（问清单 / 答清单 + 把答复变成
   `RemoteState`），那是 2–3 轮的工作，做完之后壳里的命令才有东西可调。
   **这与传输的教训一样**：命令看起来缺一层胶水，实际缺的是另一端。
   ★★ **还有一处分层事实（2026-10-18，同一会话，再查一层）**：`synra-core` **不依赖 `synra-ipc`**（它的 `Cargo.toml`
   只有 protocol/store/capability/observe 一类），所以 **core 不能产出 `SyncPlanDto`** —— 那是 wire 类型。正确形状因此是：
   core 的 `decide(...)` 给出判定，**壳（`synra-rust-app`，同时依赖两边）负责把它映射成 `SyncPlanDto`**。
   `SyncTarget` 的完整字段是 `device_id`、`display_name`、`needs_artifact`、`version: Option<String>`、`online: bool`，
   其中 `online` 只有壳知道（它握着链路状态）—— 这从另一侧证实了映射属于壳。
   **所以 `plugins_sync_plan` 的实现顺序是**：core 加"问清单/答清单"的消息 → 壳把对端答复与本地插件喂给 `decide` →
   组装 `SyncPlanDto`（`online` 由壳填）。三步，第一步最大。

三条都写在这里，是为了让下一轮**先选一个、写清理由、再动手**，而不是试探着写一半。
2. **展示名**：`open()` 从路径推 `file_name`，所以磁盘上的名字就是对方看到的名字 —— 用 `message-<yyyymmdd>.txt`
   而不是裸 uuid。**代价**：同一天两条文本会在对方那里同名，可接受（对方会自己加后缀，那是接收侧的事）。
3. **清理**：三条退出路径（成功、拨号失败、块被拒）都删那个文件；`transfer_cancel` 目前只能删注册表里的行，
   **删不到文件** —— 所以取消一条文本传输会留一个孤儿文件。**这是已知缺口**，记在这里而不是假装它不存在：
   正解是让注册表多存一个"这个传输的临时文件路径"，由 worker 在取消时删。
4. **契约**：`SendTextRequest { to, text, context_id }`，返回 `TransferId`，与 `send_file` 同形。
**`LocalPeer` 的六个字段**（`peer/host.rs:13`）：`device_id`、`display_name`、`platform`、`app_version`、`features`、`requires`；
`Flags::production()` / `Flags::no_timeout()`（`peer/config.rs`），`LinkToken::from_bytes([u8; 32])`。
这些是 `dial_peer` 与 `LocalPeer` 的全部输入面 —— 剩下的未知只有"已配对设备的地址从哪张表取"。
★ **最后那个未知也答了（2026-10-18）**：地址在 `devices_seen.address`（`synra-store/src/sqlite/devices.rs:71`，
`SeenDeviceRow.address: Option<String>`，第 7 列），与 `first_seen_ms`/`last_seen_ms`/`source` 同一张表。
所以 `transfer_send_file` 的取数路径是：`devices_seen` 里按 id 取 `address` → 解析成 `SocketAddr` → `dial_peer`。
★ **但 `Dialled` 把结果挡住了（2026-10-18 查，`host/connections.rs`）**：它的四个字段全**私有**且**没有 impl 块**
（`grep 'impl Dialled'` 与 `fn into_link`/`fn link` 都无结果），`stream` 字段是 `TcpStream` —— 也就是说拨完号的连接
只有 `connections.rs` 内部能拿到。**壳里因此拿不到链路**，即使拨号成功。
结论：`send_file` 不该在壳里拼装，而应在 core 里加一个小函数（`link::send_file(dialled, sender, announce)` 之类），
壳只负责取地址与拨号参数。这比在壳里加访问器更好 —— 访问器会把"连接之后怎么用"的责任交给两个宿主。
★ **`into_link` 的确切形状（2026-10-18，`link.rs:228` 与 `host/connections.rs`）**：
`Link::new(stream: impl LinkStream + 'static, local: DeviceId, peer: DeviceId, agreed: Agreed, flags: Flags)` —— 五个参数，
`peer` 与 `agreed` 都能从 `Dialled.negotiated` 取（`Negotiated` 有 `device_id`、`display_name`、`platform`、
`app_version`、`agreed: Agreed`），`local` 必须由调用方给（`Dialled` 自己**不存** `ours`），`flags` 也是。
所以最省事的做法是把 `into_link` 写成 `Dialled::into_link(self, local: DeviceId, flags: Flags) -> Link`，
由它的调用方（本模块里的 `send_file`）把 `ours.device_id` 与 `flags` 传进来 —— 这样 `Dialled` 不必多存一个字段，
而"谁的连接"仍然只有一个来源。
★ **2026-10-18 尝试写 `transfer_send_file` 时的四个签名错误（已回退，别再踩）**：
(1) `SeenDeviceRow.device_id` 是 **`String`**，不是 `DeviceId`，比较要先转；
(2) `SeenDeviceRow` **没有 `address` 字段** —— 我看到的 `address` 是**另一张表/另一行**的字段（`devices.rs:71` 的 `row.get(7)`），
所以"地址在 `devices_seen.address`"这个结论**需要重新核实**；
★ **核实结果（同一轮）：结论是错的，而且这是一个真缺口。** `SeenDeviceRow`（`synra-store/src/records.rs:193`）的字段只有
`device_id: String`、`display_name`、`platform`、`first_seen_ms`、`last_seen_ms`、`source` —— **没有 `address`**。
`devices.rs:71` 那个 `address` 属于另一个 `from_row`（不是这张表）。也就是说：**发现层记下了设备的名字与来源，却没记它从哪个地址来**，
所以壳今天**无法**"查表拿地址再拨号"。
于是 `transfer_send_file` 真正的前置不是胶水，而是二选一：
(a) 发现时把 `SocketAddr` 一并落库（`devices_seen` 加一列 + 迁移 + `record()` 里填）；或
(b) 发送前重新发现一次（广播、等应答、拿到地址再拨）—— 慢，但不动 schema。
★ **2026-10-18 决定 (a)，并写清实现清单**（迁移是 11，因为当前最高是 `version: 10`）：
★★ **这一整条清单在下一轮被证伪，不要照做 —— 见下面的"但地址本来就有"。**
1. `crates/synra-store/src/db/migrations.rs`：新增 `version: 11` 的 `ALTER TABLE devices_seen ADD COLUMN address TEXT`；
   **可为空**，因为旧行没有地址，而"不知道"与"没有"在这里是同一件事；
2. `crates/synra-store/src/records.rs`：`SeenDeviceRow` 加 `pub address: Option<String>`，`from_row` 用下标 7 读它；
3. `crates/synra-store/src/sqlite/devices.rs`：`INSERT INTO devices_seen` 的列与参数各加一处（第 8 个），
   `ON CONFLICT` 的更新里**不要覆盖成 NULL**（`address = COALESCE(excluded.address, devices_seen.address)`）——
   一次没有地址的发现不该抹掉上一次有的；
4. `record()` 的调用方（发现路径）把 `SocketAddr` 一并传入；
5. 校验：`guards`（doc-claims 会核对 schema 版本）、`synra-store` 的单测（加一条"没有地址的更新不会抹掉已有的地址"）。
**这五步是下一轮的全部工作**，做完 `transfer_send_file` 的地址来源就有了。
★★ **但地址本来就有（2026-10-18 晚，同一天内第二次更正）：`paired_devices` 表已经有 `address` 列。**
`DeviceRow`（`records.rs`）与它的 `from_columns` 都带着 `address: Option<String>`，`paired_devices()` 的 `SELECT` 是
`records::columns::DEVICE` 的第 7 列（`devices.rs:71`）。**而转账的目标本来必须是已配对设备**，所以壳该查的是
`paired_devices()`，不是 `devices_seen()` —— 上面那五步（迁移 11、加列、`COALESCE`、发现路径传地址）**全部作废**。
教训：我把"设备"当成了两张表里的一张，而实际上"看得见"与"可拨号"是两件事，后者早就有地址。
`transfer_send_file` 的取数因此是一行：`db.paired_devices()` → 按 `device_id` 找 → `address`（`None` 就是
`ErrorCode::Unsupported`，"从没连过，没有地址"）。
**这是设计决定，先定再写。** 在它定下来之前，`transfer_send_file` 写不出来（我这一轮就是写到这里才发现，四处签名错只是表象）。
(3) `synra_core::host::millis_reading` **要参数**（`clock.rs:14`），不是零参；
(4) `TransferSender` **没有 `chunk_size()`** 访问器，进度里的"已发字节"只能自己按块数乘常量或从 `blocks()` 的块大小取。
★ **2026-10-18：`transfer_send_file` 已经写完并注册（28 条命令），两条测试覆盖了"未配对拒绝"与"拨号失败后不留行"。**
写它时又纠正了两件事：(a) 路径必须**真实存在** —— 命令先 `TransferSender::open` 再拨号，假路径会以 `InvalidArgument`
提前返回，根本到不了拨号；(b) `Paired` 的公开路径是 `synra_store::Paired`（`sqlite::pairing` 是私有模块）。
**成功路径的测试不需要 socket**：`synra_core::link::memory::MemoryStream::pair()`（`link/memory.rs:153`，模块 `pub`）
造出一对内存流，`PeerLink::new(stream, local, peer, agreed: Agreed, flags)` 就能在一端发、另一端收 `Inbound::Block`。
剩下的唯一未知是 `Agreed` 的构造（一次 `grep 'pub struct Agreed'` 即可）。
★ **试过了，这条路暂时不通（2026-10-18，已回退）**：`Agreed { common, missing_required }` 确实可以字面量构造，
但**接收端会永久阻塞** —— `PeerLink::receive_inbound()` 在没有数据时不返回，而 `PeerLink` 上**没有** `set_read_deadline`
（那是 `LinkStream` trait 的方法），`Flags { timeout: Some(..) }` 只作用于握手，不作用于读。
于是测试挂死（第一次跑还留下一个 `synra_core-*.exe` 占着链接器输出，导致 `link.exe` 报 1104）。
**结论**：要么给 `PeerLink` 加一个读超时覆盖（它内部已有 `read_timeout_override` 字段，只是没有 setter），
要么在测试里只发**一块**并在发之前就先 `receive`（时序仍不可靠）。
**建议加那个 setter**：字段已经存在，缺的只是一个 `pub fn with_read_timeout(self, ..) -> Self`，
而它对真实链路也有用 —— "读卡住不能永远等"。**这一步没做，因为它在 core 的公开 API 上，值得单独一轮。**
**教训与之前几次相同**：先 `grep` 出每个类型的字段与每个函数的参数，再一次写成；我这次是照着印象写的，四处全错。
**这一步把工作从"壳里几十行"变成"core 里几十行 + 壳里十行"，而 core 那侧的测试可以用 `link.rs` 现成的 localhost 双端。**
**传输发送侧已无未知**：地址表、`LocalPeer`、`Flags`、`LinkToken`、`dial_peer`、`TransferSender` 逐块方法、
`snapshot_from`、`announce_transfer` 全部就位且各有出处；剩下的是把它们连起来的那几十行。
**这是本会话对传输的最后一次决策**：发送在壳里做完，接收留待通道。
★ **建链的入口也已经有了（2026-10-18 查过，`crates/synra-core/src/listener.rs`）**：
`bind(host, port) -> SocketAddr` 绑定端口，`serve(host, cancel, data_dir, desk, router, on_finished)` 是**现成的连接循环**
—— 它接受连接、按 `Cancellation` 退出、把每条连接交给自己的线程，并用 `FinishedCallback` 回报完成的传输
（`FinishedTransfer`、`Served`、`no_answer()`）。
**所以壳里不是"写一个监听器"，而是"调用 `serve` 并给它一个 `FinishedCallback`"** —— 那个回调正是
`announce_transfer` + 结束时 `forget`/`TransferFinished` 的落点，`Router` 与 `PairingDesk` 在 daemon 里已有构造先例。
`snapshot_from(request, peer, state, started_at_ms)` 已在壳里写好并有两条测试。
★ **2026-10-18 再补一条：`transfer_cancel` 也注册了（27 条），而且端到端。** 它从注册表移除并发布
`TransferFinished`，前端的 `intents` store 收到就删掉那一行 —— 没有这个事件，被取消的传输会一直躺在进度列表里
直到页面重开，那正是其它命令当初被修掉的那种静默过期。三种结果各有理由：id 不合法是调用方的错（`InvalidArgument`）、
不在飞行中说明面板过时（`TransferFailed`，并命名 id 让面板能重读）、在飞行中就移除。
**一条明确的边界**：它不通知对端 —— 那需要一帧，而承载它的链路正是 worker 的职责。今天它停的是壳自己的记录。
`TransferId` 用 `FromStr`（`"…".parse()`），不是 `TryFrom<String>`；这一点查了才知道，猜错过一次。

**传输子系统**不该在一轮里开工：它的第一个切片应该是"一个本机回环的传输计划"（`plugins_sync_plan` 已存在但未注册），先把计划与状态机跑通，再谈真正的收发。

### 8.8 同步链路：表终于有了写者，计划问的是现在（2026-10-18 晚）

★★ **`plugins_sync_plan` 的那张空表补上了，而缺口比"计划"低一层。** 再查 `plugin_versions` 时发现**生产代码里没有写入方**：
`synra_core::install` 只写 `plugins` 行与文件，从不写版本行。于是 `active_plugin_version` 在真实安装之后返回 `None`，
`plugins_sync_plan` 对任何已安装插件都报 `PluginNotInstalled`。**那句话本身（"写入在安装路径里"）是文档里的一个推断，
而安装路径里没有那一行** —— 与 `sync_records` / `plugin_versions` 是同一种模式的又一次出现。

现在的链条（四段，各有测试）：

1. **写**：`SynraDb::record_plugin_version` 一个事务里先清旧 `active`、再 upsert 新行；旧行**保留**（回滚要用的 artifact 还在）。
   `install::from_directory` 在写完 `plugins` 行后调用它，失败时连插件行一起撤销 —— 它因此收 `&mut SynraDb`，
   因为"两行必须一起退"不是风格问题。
2. **答**：`plugin_sync::local_list` 把 `plugins` 与 `plugin_versions` 的 active 行 join 成 `PluginList`
   （没有 active 行的插件不进清单：没有 artifact 可以描述），`answer_plugin_list` 把答复造成信封。
   listener 在收到 `_synra.plugin.list.request` 时就地作答；`Decision::Delivered` 因此带上信封 —— 回复需要请求的 id、
   trace 与发送方，而这三样只在信封里。
3. **问**：`plugin_sync::ask_plugin_list` 发请求并按 deadline 读答复；`MemoryStream` 两端有测试，两条半程第一次真正对上。
4. **计划**：`plugins_sync_plan` 不再读 `sync_records` 的旧答案。它对每台已配对设备拨号、问一次，把答复交给
   `replace_sync_for_peer`（同一事务里先删该对端全部行再写：一次活的答复是**完整**的，卸载掉的插件不会出现在里面，
   而 upsert 看不见"没有"），再用答复里的本插件状态喂 `decide`。`online` 的语义因此变成"**它答复了**"，
   而不是"它有地址"；没答复的对端仍按"需要发"计入 —— 安全方向。

**`plugins_sync_run` 也已经落地（同日晚些时候）**，那三个缺口各自有答案：
1. **artifact 的读者**：`SynraDb::active_plugin` 返回 `ActiveVersion { version, artifact, digest }`
   （`artifact` 是相对 `plugins/<id>/` 的**文件**名；UI-only 插件是空串），`active_plugin_version` 变成它的一层投影，
   两个读者因此不会对"哪个版本生效"给出两个答案；
2. **接收侧**：第 179 轮起两个宿主共用 `record_received`，壳也回 `SyncResult`；
3. **轮次**：`SyncRound { plugin_id, version, session: SessionId::new() }` —— 发起端命名，协议里本来就有构造器。
运行器在壳的 `commands/sync_run.rs`：每台 `needs_artifact` 的设备一次拨号 + `core::send_bundle`，然后
`verdict(events, pushed)` 把答复分成三种 —— **装上了**（记接收端说的版本）、**拒绝了**（什么都不记）、
**没有确认**（不记：一次没被确认的传输说不出对端有什么）。`ok: true` 却不给版本按拒绝处理：
拿"我推过去的版本"补上，会让下一次计划跳过一次可能需要的发送。

**仍然没有的**是真正的解包：接收端写 `plugins` 行，但 `plugin-<id>.wasm` 还留在 inbox 里，没有 manifest、
没有签名覆盖目录，也没有 `plugins/<id>/` 下的文件。所以这条同步链的下一步不是"再发一次"，而是**包格式**：
在它存在之前，一次同步的终点是"对端说它装了"，而不是"对端能跑"。

### 8.9 插件页的调用通道（2026-10-18 晚）

★ `plugin_ui_rpc` 注册了，`PluginUiOp` 在 `synra-ipc` 里是一张**封闭的**方法表：不允许的方法（`fs.*`、
`platform.simulateKeys`、`plugins_*`、`devices_*`、`config.set`）不是变体，所以"不存在"与"不允许"给同一个答案。
`device.list` 是第一个有实现的方法（四个字段）；其余在表面上、以 `Unsupported` 回答 —— 不是空列表，也不是假承诺。

★ 两处修正比枚举本身更重要：
1. **令牌不再是计数器**：`open_request` 原来用 `issue_placeholder`，而注册表自己的文档写着 not for production ——
   一个页面能猜到另一个页面的令牌。现在用 `uuid::Uuid::new_v4().as_u128()`。
2. **会话带上用户的授权**：原来签发时用 `CapabilitySet::new()`，于是 `env.can` 对用户批准过的能力也答 false。
   现在读 `plugin_grants` 里 `granted = 1` 的行；一个解析不了的 grant 拒绝开页，而不是拿猜出来的集合开。

★ 前端第一次守住接口分离：`device.list` 以前直接调 **Host** 命令 `devices_snapshot`，而 `command.rs` 的
`Interface::allows` 说插件接口不能到 Host 那一边。现在它走 `plugin_ui_rpc`，内核把卡片裁成四个字段
（地址、blocked、传输进度都不给插件看）。

★ **存储也接上了**：`storage.get/set/delete/keys/quota` 走同一个 RPC。grant 由**会话的 `CapabilitySet`** 检查
（`storage:local:read` / `write`），不是门面那套字符串集合 —— 后者只用于同步的 `env.can`。配额用 `PluginSet::quota`
（没有引擎的窗口用默认 64 MiB），写入算法与 `HostCalls` 一致：`used - replaced + new`；超限返回 `PayloadTooLarge`
（协议没有 quota 专用码，而 `RateLimited` 会让插件"等一会儿"—— 错的方向）。门面把值编码成 JSON；读到一个
headless 半写入的裸字符串时按原样返回，而不是抛 `SyntaxError`。

★ **桌面动作也接上了**：`clipboard.read/write`、`platform.openUrl/notify` 走同一通道，grant 是
`platform:read-clipboard` / `platform:write-clipboard` / `platform:open-url` / `platform:notify`。
`CommandPlatform` 从 `DesktopHost` 里提出来、放进 `AppState.platform` 共享 —— 它带**每实例**的通知限流，
每次调用新建一个就等于给插件第二个小时配额。`PlatformError` 的映射是显式的：rate → `RateLimited`、
quota → `RateLimited`（这一次"等一会儿"就是正确的动作）、过长 → `PayloadTooLarge`、坏 URL → `InvalidArgument`。
前端同时修了一个拼写：旧代码检查 `clipboard_read`，而能力表里是 `platform:read-clipboard` —— 一个声明了真实能力的
插件仍然会被拒。测试用一个记录型 `CommandRunner` 替换真实程序，所以不会真的打开浏览器、改剪贴板或弹通知。

★ **出网也接上了**：`network.fetch` 走 `PluginSet` 的 **host**，而不是在壳里新建一个 fetcher —— 同一个客户端才有
同一条 in-flight 限制、同一份 grant 视图；`fetcher_for` 读 `network:fetch:<host>`，空列表是拒绝而不是"任意 host"。
`synra-fetch` 的 `Response` 补了 `statusText`、`url`、`redirected`：契约里的 `SynraResponse` 声明了它们，而之前
core 的响应把重定向后的地址丢掉了。门面把 host 的纯文本 body 变成 `text/json/bytes` 三个**互斥**的读法，
第二次读是 `invalid_argument`；`timeoutMs` 与字节 body 在门面与 RPC 两处都拒绝（这一层的超时是客户端的，
传一个会被静默忽略的值等于让作者以为它生效了）。

**仍然缺的**是把 `createInjection(context)` 装进插件页。初始化脚本只放 label 与 token；可行的做法是把注入门面
单独构建成一份资源（Vite 的第二个 entry），开窗时**拼进 `initialization_script`** —— 这样 `window.synra` 在页面
任何脚本之前就存在，插件作者也不必改 HTML。要先决定生成物放哪（提交生成物 + 守卫，还是随安装包发资源），
所以留成独立一轮。

★★ **而这一轮把它做完了（同日更晚）**，选的是"提交生成物 + 守卫"：

- `src/ipc/injection/bootstrap.ts` 是第二个 Vite 构建入口（`pnpm run build:facade`，配置在 `vite.inject.config.ts`），
  产出一份 IIFE：`src-tauri/inject/facade.js`（约 8.5 KB，含内联的 `@tauri-apps/api`）。
- 壳用 `include_str!` 把它嵌进来；`initialization_script` 生成
  `(function(seed){ <facade>; SynraInjection.boot(seed) })(<seed>)`。seed 带 label / token / pluginId / granted /
  denied / locale / darkMode / sdkVersion / platform / hostKind / appVersion —— 全部由宿主在页面加载前决定。
  ★ **seed 是包装函数的参数，不是全局**：设计文档要求 token 在闭包里，而 `window.__SYNRA_PLUGIN__` 会被页面引入的任何
  脚本读到（一个分析脚本、一个出了问题的依赖）。作为参数它只活在构造门面的那次调用里，门面把值复制进自己的闭包；
  `window.synra` 仍是插件唯一的入口。
- **为什么生成物要提交**：Rust 构建不能依赖 Node。所以 `facade.js` 进仓库，`scripts/check-shapes.ts` 增加
  `facade-fresh`：用 Vite 的 `build()` 在临时目录重建一次、逐字节比较（先把 CRLF 归一化）。生成物过期就是红。
  提交生成物通常是个坏主意；这里成立的前提是**有一个能证明它是最新的守卫**。
- 两个小决定：`dist`/`src-tauri/gen` 之外，eslint 也忽略 `src-tauri/inject`（压缩产物的风格不是本仓库的风格）；
  `sdkVersion` 取 manifest 的 `sdk.preferred`（没写才退回 `min`）——`min` 是兼容下限，报它会告诉插件它是对更老的宿主构建的。

**仍然缺的**：transfer.*、self.invoke/actions 两个命名空间（都在方法表里、都以 `Unsupported` 回答），
以及插件卡上的"打开"入口。

## 9. 每阶段的"退出条件"（不允许跳过的）

| 阶段 | 退出条件（缺一不可）                                                                   |
| ---- | -------------------------------------------------------------------------------------- |
| M0   | `cargo tree` 证明内核无 GUI 依赖；feature 矩阵全绿；前端骨架能渲染四页                     |
| M1   | 双内核互通 + **协议一致性套件全绿** + 超长帧断链而不是错位 + 重连可用            |
| M2   | Wasm 插件能匹配并执行；越权被闸门拦住；**恶意插件套件全绿**                                |
| M3   | 篡改被检出；权限变更需确认；跨设备同步校验通过；**activate 失败可回滚**                      |
| M4   | 端到端场景可用；**UI 插件隔离对抗测试通过**；冷启动 < 500ms                                |
| M5   | 移动端能跑 headless 插件；跨端同步可用；前后台恢复可用                                     |
| M6   | 无（持续）                                                                              |

---

## 10. 交付物清单（最终）

| 产物                      | 路径                              | 形态                    |
| ------------------------- | --------------------------------- | ----------------------- |
| 桌面应用                  | `synra-app`               | `.msi` / `.dmg` / `.AppImage` |
| 移动应用                  | `synra-app`                | `.apk` / `.aab` / `.ipa` |
| CLI                       | `apps/synra-cli`                   | 单二进制                 |
| 守护进程（可选）            | `apps/synra-daemon`                | 单二进制                 |
| 插件开发工具               | `tools/synra-plugin-cli`           | 单二进制                 |
| Rust SDK                  | `crates/synra-plugin-sdk`          | crate                   |
| WIT 定义                  | `wit/synra-plugin.wit`             | 接口定义                 |
| TS 类型（给插件作者）       | `@synra/client`（[23](./23-client-library.md)）        | `.d.ts` only，零运行时    |
| 插件模板                  | `synra-plugin new`                 | 三种模板                 |
| 文档                      | `design-docs/` + `design-docs/`（本文档集）      | Markdown                |
