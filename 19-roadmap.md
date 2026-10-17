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
