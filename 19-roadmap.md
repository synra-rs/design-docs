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

★ **插件卡接上了**：按钮先调 `plugins_sync_plan`、再调 `plugins_sync_run`。计划里 `online` 是唯一能看到"对端答了没有"
的地方，页面据此决定：没有任何 `needsArtifact` 时只说一句"都已是最新"，而不是运行一遍全是零的报告。计划与运行
是内核的两个命令，页面不合并它们 —— 计划是可以**读**的，这正是它存在的理由。

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

★ **插件卡上的"打开"入口接上了**：`PluginSnapshot` 增加 `hasUi`（读 manifest 的 `runtime.ui`），它与 `hybrid`
**不是**一件事 —— 只有 UI 的插件有页面、没有 headless 半，用 `hybrid` 做门会漏掉这个形态。按钮只在 `hasUi` 为真时
出现，点击走 `plugin_ui_open`（窗口的 label / origin / token 仍由宿主在开窗前决定）。
★ 顺带修了守卫：`check-shapes` 的 `command-arguments` 原来只把 `state` 当作"宏注入、调用方不传"，`app: tauri::AppHandle`
被当成必填键 —— 而它和 `state` 一样由 `#[tauri::command]` 从调用上下文解析，页面只该传 `pluginId`。

★ **`self.actions` / `self.invoke` 接上了，顺序是先有声明**：`Host::action_declare` 现在把声明写进 `plugin_actions`
（按 `(plugin_id, name)` upsert，schema 必须是 JSON，否则 `badArgument`），`action_list` 读出来并展开成
`ActionDescriptor`。RPC 的 `self.invoke` 先查这张表 —— **action 必须属于本插件**，这正是设计要求的检查；然后以
`Work::InvokeAction` + `CallBudget::from_quota` 调 `PluginSet::call`。结果映射：`Ok` 解析 payload，`Failed` 保留插件
自己的错误码（`ipc_code` 先认 host-call 的 camelCase，再让 `ErrorCode` 反序列化认协议拼写 —— 不再维护第二张码表），
`Timeout` 是 `PluginTimeout`，其余是 `PluginTrapped`。跨插件的 `action_invoke(target)` 仍拒绝：那是
`action:invoke:*` 的另一条权限路径，host 够不到 `PluginSet`。
★ 前端 `self` 命名空间随之实现；同时修了 `command-names` 守卫的一个误判：它原来匹配**任何** `.call(`/`.invoke('x')`，
把测试里的 `synra.self.invoke('clear')`（插件 action，不是命令）当成了命令；现在只认 `kernel` / `client` / `bridge`
三个接缝对象，并允许 `call<T>(...)` 的类型参数。被调用命令数从 11 修正为 16。

**仍然缺的**：transfer.*（需要 WebView 到内核的大文件通道）。

### 8.10 插件页的大对象传输（第一版）

★ 决定的形状：**base64 走现有 `plugin_ui_rpc`**，而不是新增 Tauri 命令。理由：新增命令要同时改契约表、命令守卫和
fake kernel，而发送路径已经完整（`transfers::transfer_send_file`：拨号、逐块发送、进度事件、错误映射）。页面把一个
Blob 编码成 base64，内核解码到 `cache/plugin-transfers/<uuid>/<fileName>`，交给那条路径，结束后删除临时文件。
**4 MiB 上限**：一条 IPC 消息里的 base64 不是大文件通道；分块/流式是下一版。
`transfer.status` 读注册表；已完成的传输已被 `finish_transfer` 移出注册表，所以**门面缓存它自己发起的传输的最终状态**，
`onProgress` 对晚订阅者回放它。
已知限制：`send` 在传输**结束**后才返回（没有后台线程），因此 `onProgress` 只有在 `send` 之前订阅才能看到逐块进度；
接收端沿用既有行为（落到 inbox），还没有"询问用户是否接收"的步骤。

**仍然缺的**：分块/流式通道（大文件）、接收端的接受/拒绝 UI。

★ **入站传输也进同一张表了**：`listener::serve` 一直只把"完成"交回来，接收中的传输在界面上不可见。现在新增
`serve_with` / `handle_within_with`，用 `ProgressCallback` 在 `Decision::Accepted` 与每个 `Decision::Stored` 上报
`InboundProgress`（字节、块、文件名、是否 bundle）。`FinishedTransfer` 补了 `transfer_id`（完成事件原本只有路径与长度，
没有 id，壳无法关掉它开的行）；壳的接收回调先 announce `completed` 快照、再 `TransferFinished`。
两个旧入口（`serve` / `handle`）以空回调委托，daemon 不需要改。

### 8.11 托盘与关窗语义

★ 窗口的关闭键是"收起来"，不是"退出"：这个进程是内核（监听端口、传输在飞、插件已载入），对别的设备可达；把
"整理窗口"和"结束服务"绑在同一个手势上，会让一个只想让窗口让开的用户停掉所有传输。托盘菜单只有 **Show** 与
**Quit**；Quit 只调 `app.exit(0)`，停止逻辑统一在 8.12 的 `ExitRequested` 一处 —— 退出本来会释放端口，但重启时与
旧进程抢同一个端口，正是那个 token 存在的原因。
`tray_menu` 与 `tray` 分开：前者的 id → 动作映射不需要 Tauri，所以它在跑测试的构建里也有断言；后者是事件循环与
图标的接线，失败只报告不致命（有的 Linux 桌面没有 indicator host，窗口仍必须能开）。

### 8.12 单实例：一个数据目录一个进程

★ 用**操作系统文件锁**（`File::try_lock`：Windows 独占锁、Unix `flock`），不是 pid 文件：进程被杀时锁由内核释放，
而手写的 pid 文件会留下一个"看起来还在跑"的记录。锁文件在数据目录下，所以**不同安装（不同数据目录）可以并存**。
第二次启动拿到 `WouldBlock` 时写 `<data>/instance.show` 并退出；第一次启动在窗口存在后起一个 500 ms 的轮询线程，
看到文件就删掉并 `show_main`。
★ **窗口改为 `visible: false`，在单实例检查通过之后才 show**：Tauri 的 setup 之前就会按配置创建窗口，第二次启动
否则会先闪一下再退出。第一次启动在 setup 末尾显式 `show()`。
第二次启动在 `app_state` **之前**退出，因此不会开第二个数据库、不会绑第二个端口；锁拿不到（只读目录等）时**继续运行**
并打印原因 —— 两个实例比一个打不开的窗口轻。

★ **退出路径只有一处**：`run_window` 的 `RunEvent::ExitRequested` 调 `stop_background`，取消 listener 的 token 与
discovery 的 token；托盘的 Quit 只调 `app.exit(0)`，不自己拼一份停止逻辑。一个按名字列出 worker 的关停，会在下一个
worker 出现时忘记它。discovery 的 `stop` 只持有 supervisor 的 **shutdown token**（不是 handle）：线程真正读的就是它，
而这个壳没有"关停报告"要用 handle。

★ **`env.platformSupports` 接上了真实答案**：宿主在开窗前用 `CommandPlatform::supported_actions()` 生成
`platformCapabilities` 放进 seed，门面同步查表。两层映射是刻意的：`PlatformAction → 客户端拼写`
（`open_url`/`clipboard_read`/…）与能力拼写（`platform:open-url`）**不是**一套；`simulate_keys` 只在策略允许时出现，
所以按钮不会提供一个宿主会拒绝的动作。`PlatformAction` 是 `non_exhaustive`，未命名的动作被丢弃（答案 false），
而不是猜一个名字。

★ **剪贴板能力拼写统一了**：生成的契约表与 host 闸门一直是
`platform:read-clipboard` / `platform:write-clipboard`，而 `synra-capability` 的文档、`PlatformAction::capability()`
与 `admission` 的映射用 `platform:clipboard:read` / `platform:clipboard:write` —— 于是按文档写的 manifest 声明了一个
闸门从不检查的权限。现在以契约为准：`PlatformAction::capability` 与 `admission` 改成平铺名；
`synra-capability::canonical_spelling` 在**边界**（manifest 读取、grant 迁移）把旧拼写改成正名 —— 不在匹配器里教第二遍，
否则旧名会永久存在。store 的 v11 迁移修正已存 grant，并处理同一插件同时有两种拼写的碰撞（保留正名那一行）；
设计文档 02/10/14/22 的示例同步更新。

★ **导航范围限制也做了**：`plugin_ui_open` 现在装 `on_navigation`，谓词是 `stays_on_page(candidate, opened)` ——
**只能留在打开的这一个页面**（含 `#` 片段路由），另一个路径、另一个插件的 origin、`https:`/`file:`/`data:` 全部拒绝。
为什么不是"同插件 origin"：初始化脚本挂在 WebView 上而不是地址上，跳到同 bundle 的另一页会**保留本页的 token 与
授权**却显示打开者没选过的内容。`window.open` 一并 `Deny` —— 新窗口没有初始化脚本，等于一个没有 token 的页面。
谓词在无 WebView 的构建里照样编译并有测试，只有把它交给 builder 的那一行在 `tauri-shell` 里。

### 8.13 包归档：先让 `.synra` 装得进去

★ 容器定为 **zip**：内容就是安装器接受的目录（`manifest.json`、manifest 命名的 artifact、有签名时的
`signature.json`）。解包复用 `synra_plugin_ui::artifact::unpack`，因为 zip-slip、绝对路径、符号链接、条目数与大小、
压缩比这些规则是关于**归档**的，不是关于 UI 的；`ArchiveLimits::package()` 只是把 64 MiB / 32 MiB 那两个数字填进去。
`install::from_package` 解到 `plugins/` **外面**的 `.install-staging/`（plugins 下带 manifest 的目录会被当成已安装插件），
再调用 `from_directory`；失败时按 unpack 返回的清单逐文件清理（不做递归删除），成功也清理。
`install::package_directory` 是发送侧：把已安装目录压成包，**跳过解包出的 `ui/`**（包里要的是 `ui.zip`，`ui/` 是本机产物），
并检查 64 MiB 总量。
★ 本轮只到"装得进去"：接收端的同步回调仍发/收单个 wasm，`plugins_sync_run` 与 `record_received` 都还没换成包。
★ **接上了**：`plugins_sync_run` 把 `plugins/<id>/` 打成 `.synra`（一次一轮，用完删除），`send_to` 发的是包；
接收侧窗口与 daemon 都在 `on_finished` 里先 `install::from_package`，成功后再 `record_received`。安装失败时回
`SyncOutcome::refused`，不再写一条"已装"的历史。`proposed_name` 也从 `plugin-<id>.wasm` 改成 `.synra`。
★ 由此留下的一件事：`record_received` 里的 `insert_plugin` 在安装之后会失败（插件行已经由安装器写好），现在被
`let _` 吞掉；它不是错的状态，但那行 `let _` 值得在下一轮收成一个显式的"已安装则跳过"。

### 8.14 接收端的接受/拒绝

★ `Inbox::absorb` 之外新增 `absorb_with`：在**写任何字节之前**把 `Offer`（对端、地址、请求、落点文件名）交给宿主，
由它回答 `Accept` / `Refuse`。返回 `Refuse` 时先给对端发 `file.transfer.abort`（`UserCancelled`），再以
`Decision::Failed` 结束连接 —— 不告诉发送端，它会把整个传输窗口等完。
★ `OfferDecision` 是 `non_exhaustive`（稳定 crate 的公共枚举）；`auto_accept` 让所有旧入口保持原行为。
★ **窗口侧**：`TransferDesk`（`AppState.transfer_desk`）在一张表上等答案，`ask` 先登记再**发布**
`TransferUpdated(state = "offered")`，然后等；`transfer_respond` 命令写答案；监听线程在答案后发
`TransferFinished` 把行收掉。**超时（120 s）按接受处理** —— 提示没人看见时不能变成"文件没人收"；这是旧的默认行为。
★ **插件包不提问**：一轮同步是用户按过"同步"的决定，中途弹窗会让这一轮依赖有人看着窗口；只有 `Attachment` 会问。
★ UI 先放在 HomePage 的 in-flight 列表：`state === "offered"` 的行显示文件名与 Accept / Reject，答案是命令，
行的消失靠 `transferFinished` 事件（不在页面里提前删）。
**仍然缺的**：提示不是全局浮层（用户在别的页面时看不到），也没有"记住这个对端"的偏好。

★ **提示改为全局浮层了**：`TransferPrompt.vue` 挂在 `App.vue` 根部（`fixed`，不占布局），从 intents store 里取
`state === "offered"` 的第一行；HomePage 的按钮撤掉，免得同一个问题有两个回答入口。一次只显示一个 —— 一张提示
对应一次等待中的连接。仍然没有"记住这个对端"的偏好。

★ **"总是接受这个设备"做了**：提示里一个复选框；`transfer_respond` 多一个可选 `remember`，接受且勾选时把设备 id
写进配置键 `transfers.acceptFrom`（JSON 数组，设置快照里可见）。监听侧在提问前先查这个列表，命中的设备直接接受、
不闪提示。**拒绝从不被记住** —— "不"是对这个文件的，不是对这台设备的。存储失败是静默的：下一次再问一次，而不是丢文件。
**仍然缺的**：没有移除入口（设置页还没有这一项），也没有按设备显示“已信任”的列表。

★ **移除入口和列表都做了**：设置页新增 “Files without asking”，读 `transfers.acceptFrom`，每行一个 “Ask again”；
移除最后一个时**unset 整个键**（空数组和“没有偏好”是同一件事，不该让界面去解释）。设备名从 devices store 来，
查不到就显示 id —— 偏好仍然是真的，藏掉一行等于藏掉一个用户改不了的设置。
★ **忘记设备会一并忘记信任**：`devices_forget` 调 `forget_sender`，否则同一个 id 再出现时会静默恢复信任。

### 8.17 后台线程都有停止条件

★ 事件桥（`events::spawn`）与 pairing 监视线程（`pairing::watch`）以前是 `loop`，只随进程结束；现在各自拿一个
`Cancellation`，`stop_background` 在 `ExitRequested` 里连同 listener、discovery 一起取消。两个 token 放在一个
`Background { cancels }` 里由 `app.manage` 持有 —— 一个类型一个值，不必为每种线程起一个 newtype。
★ 没有 join：两个循环各自在 `POLL` / 250 ms 的间隔里看到 token 后返回，而退出本来就不等任何人。**这是一个决定**：
等一个可能正在 `emit` 的线程没有收益，而阻塞退出换来的“干净”只是进程结束时操作系统本来就会做的事。

### 8.18 abort 在接收端也要留下“没有文件”

★ 发送端取消会发 `file.transfer.abort`；接收端以前只是 `self.receiving = None` 然后报 `Decision::Failed` —— **半成品
文件留在 inbox，界面上的接收行也永远停在“arriving”**（那一行由 progress 回调在 Accepted 时打开，而 `on_finished` 不会来）。
现在：
- `inbound` 在进入块循环前记下 `partial`，`Received::Aborted` 与 `Received::Failed` 两条路径都 `remove_file` —— 取消在
  两边是同一个意思；
- `InboundProgressEvent` 多一个 `aborted`，`Decision::Failed` 时用最后一次 `InboundProgress` 发一个“结束”事件；
  窗口的 progress 回调看到它调 `transfers::withdraw_received`（只发 `TransferFinished`，不声称 completed）。
★ 为什么不用 `on_finished`：它的语义是“文件到了”，一个没有文件的传输不能借用它；而 abort 需要一个“结束”的信号，
`aborted` 就是那个信号。

### 8.19 发布者信任：先记住，再决定

★ `publishers` 表（migration v12）记 `publisher_id`、`fingerprint`、`pubkey`、`trust`（`seen` / `trusted` / `revoked`）、
`first_seen_ms`、`decided_ms`。`pubkey` 必须存：指纹不能反推出 key。
★ `install::from_directory` 每次从表构建 `PublisherTrustStore`（trusted → map，revoked → set）再交给 `verify`，
所以一个被撤销的 key 会在 `publisher-trust` 阶段被拒 —— 签名本身是真的，撤销不是密码学事实，只能从表里来。
★ 安装**成功后**把签名发布者记成 `seen`；失败不记（一次被拒的安装不该让下次提示说“见过”）。`DO NOTHING` 保证
一次安装既不会把 key 升成 trusted，也不会把用户的决定重置。
**仍然缺的**：表只有安装这一个写者。没有 `plugins_publisher_trust` / `plugins_publisher_revoke` 命令，也没有指纹确认
UI —— 用户还不能把 `seen` 变成 `trusted`，TOFU 的后半段（“首次使用即信任”）还没接上。

★ **写者有了**：`plugins_publishers`（列表）与 `plugins_publisher_decide`（`seen|trusted|revoked`）进了契约，插件页
新增 Publishers 区：id、指纹、状态、Trust / Revoke。命令只改**已存在**的行：一个没被签名见过的 key 不能被信任，
否则界面可以信任一个从未出现过的公钥。列表改成和 `plugins_list` 并行读（少一次等待），两个答案分别报错。
**仍然缺**：安装后没有即时的指纹确认步骤。

★ **即时确认接上了**：`Installed.signed_by` 与 `InstallReport.publisherId` / `fingerprint`（`skip_serializing_if`，
无签名就缺失）；插件页在 `await load()` **之后**判断——签名 key 还没被 trusted 才显示提示，所以已信任的 key 与无签名
的包都不会多问一次。指纹来自安装时验证过的那一份，不是页面自己再从包里解析的。
**仍然缺**：提示里只有 Trust / Not now，没有“永远拒绝这个发布者”的一键入口（要去 Publishers 列表 Revoke）。

★ **补上了，并顺手清掉一处假设置**：提示现在有 Trust / **Never trust this publisher** / Not now；中间那个写
`revoked`，之后同一 key 的安装会在 `publisher-trust` 阶段被拒。设置页删掉了“这些还不能改”的整段说明 —— 自动同步开关
是真的会写的，一段全局的“都不能改”对它是假话；`ui.theme` 只展示、不提供选择（28 §6 R22：只有暗色一套 token）。
**决定**：主题不切换（28 §6 R22）—— `@synra/theme` 只提供暗色；设置页把 `ui.theme` 印出来并说明只有一套调色板。

### 8.15 发送提前返回与取消

★ `transfer_send_file` / `transfer_send_text` 不再等传输结束：准备（解析地址、打开 sender、建行）仍同步完成，失败仍是
命令的错误；之后 `start_send` 把 sender + 行 + 临时文件交给一个具名线程，命令立刻返回 `transferId`。行和取消 token
在返回**之前**就存在，所以紧接着的 `transfer_cancel` 有东西可停。
★ 取消：`Transfers` 多一张 `cancels` 表；`transfer_cancel` 先 `stop()`（取走 token 并 cancel），再 `forget()` 与
`TransferFinished`。发送循环在**每个块之前**检查 token，取消时先给对端发 `file.transfer.abort`（`UserCancelled`）再走
`finish_transfer` —— 不然发送端会把自己的传输窗口等完。
★ 临时文件（文本发送的 outbox）由 worker 在**每条退出路径**删除；`finish_transfer` 同时 `untrack` token，避免一张只增不减的表。
★ **仍然缺的**：插件页的 `transfer.send` 还是同步等结束（`plugin_ui_rpc` 直接调同步函数），分块/流式上传也还没有；
取消的 worker 在阻塞的 `link.send_chunk` 里要等当前块写完才会看到 token。

### 8.16 插件页的分块上传

★ 页面通道的 4 MiB 上限是**一条 IPC 消息**的上限，不是传输的上限。新增四个 `PluginUiOp`：
`transfer.begin`（返回 `uploadId`，建目录与文件）、`transfer.chunk`（base64，每块 ≤1 MiB，追加写）、
`transfer.finish`（核对字节数后走桌面发送路径）、`transfer.abort`（删除半成品）。整文件上限 64 MiB（16 §2.1）。
★ `Uploads` 挂在 `AppState`，键是宿主生成的 `uploadId`；每个 upload 记住 `plugin_id`，**只有开始的插件能追加/完成**。
写块时锁只覆盖 map 的读写，文件 I/O 在锁外 —— 一个页面的慢磁盘不能卡住另一个上传。
★ 门面 `synra.transfer.send` 不再一次调用：begin → 循环 1 MiB 块 → finish，任何一步失败都尽力 `transfer.abort`，
不留半成品。`transfer.send`（单条）仍保留给旧页面。
★ **仍然缺的**：`finish` 仍是同步发送，页面 `send` 还是要等传输结束才拿到 id；真正的“页面提前返回”要等 finish 走
worker（桌面命令已经是这样，见 §8.15）。

★ **接上了**：`plugin_ui_rpc` 之外多了一个 `plugin_ui_rpc_arc`（Tauri 命令用它），把 `Arc<AppState>` 带到 dispatch 的
`transfer.finish`。有 managed state 时，finish 用 `prepare_file_send` + `start_send` 交给 worker，**立刻返回 transferId**；
文件的删除与上传目录的清理由 worker 负责（`start_send` 多一个 `cleanup_dir`）。没有 Arc 的入口（测试、旧调用）仍走同步
发送，所以两条路径都能测。页面侧 `synra.transfer.send` 的语义随之改变：**返回 = 字节已交给发送端**，结束状态用
`status` / `onProgress`；门面不再把它缓存成 `completed`。

### 8.20 列表过滤不是目录搜索

★ 插件页加了一个过滤框，匹配 **id / 标题 / 描述**。它明确**不是** registry 搜索：这个壳还没有 registry 客户端（16 §4），
一个看起来像搜索的框会承诺它取不到的结果。放在已安装列表上方而不是页面顶部，因为它的作用域就是那张列表；没有插件时
不显示 —— 空列表上方的过滤框是无意义的控件。
★ 过滤是客户端行为（`visible = installed.filter(...)`），不新增命令；`plugins_catalog_search` 仍只在契约里，
`InstallSourceDto::Registry` 仍按名拒绝。

### 8.21 Registry 的第一步：字节响应

★ `synra-fetch` 原来只有 `fetch` → `Response { body: String }`。registry 的 tarball 是二进制，按文本读会在签名检查之前
先把字节改坏。新增 `fetch_bytes` → `BinaryResponse { body: Vec<u8> }`，**共用**同一个 `NetworkGuard`、同一个重定向循环、
同一个 `max_response_bytes` 上限；差别只在最后一步是否解码。
★ 内部把公共部分收成 `request()` + `send(mode)`，两个公开入口只是把 `Fetched` 塑成不同形状 —— 复制一份重定向循环会
让“每个 hop 都重新校验”这条规则出现第二个实现。
★ 测试：本地 `TcpListener` 直接写原始字节（`0x00 0xFF 0xFE …`），断言字节原样回来；同一上限对二进制也生效
（4 字节上限读 16 字节 → `ResponseTooLarge`）。
**仍然缺的**：还没有 registry 客户端 —— packument 解析、`.tgz` 安全解包、命令与 UI 都在下一步。

### 8.22 Registry 的第二步：文档与归档

★ 新 crate `synra-registry`（Internal）装两件纯函数：
- `Packument::parse`：`dist-tags` 用显式 `rename = "dist-tags"`（`rename_all = "camelCase"` 会拼成 `distTags`，字段解析成空
  会让每个文档都像没有 `latest`）。`latest()` 返回标签指向的版本；标签指向不存在的版本时返回 `None`，**不**退化成“键排序
  最大”。
- `extract_tgz`：gzip + tar；按**类型**先拒绝符号链接/硬链接（一个叫 `manifest.json` 的链接看起来最普通），再按
  `Path::components` 拒绝 `..`/绝对路径/Windows 前缀；条目数、单文件、总量三道上限在写之前检查；失败按写入清单逆序删除，
  只 `remove_dir` 不递归。npm 的 `package/` 前缀由调用方传 `strip_prefix`，不带前缀直接拒。
★ 测试 10 条：latest 与指定版本、标签缺失、非 packument；正常解包、`../` 逃逸（用手写 tar 头构造，因为 `tar::Builder`
自己就拒绝写这种条目）、符号链接、单文件上限、前缀不匹配、gzip 损坏。
**仍然缺**：没有网络客户端与命令 —— `plugins_catalog_search`、`InstallSourceDto::Registry` 还是拒绝的。

### 8.23 Registry 的第三步：真的能装

★ `commands/registry.rs`：配置键 `plugins.registry`（缺失 → `Unsupported`，不猜默认）。一条 `GET {base}/{id}` 取
packument（`/` 编码成 `%2F`），`dist-tags.latest` 或指定版本，`dist.tarball` **必须与 base 同 origin**（一个让设备去取
别处的 packument 是在索取未被配置的权限）。下载走 `fetch_bytes`，交给 `extract_tgz`（`package/` 前缀），再走**同一个**
`install_directory` —— 签名、限额、manifest、store 都还只有一份实现。
★ `http` 只对 loopback 放开（本地镜像/测试）；公网走 `https`，这是 `NetworkGuard` 的默认而不是这里加的规则。
★ `plugins_install` 的 `Path` 分支被抽成 `install_directory`，registry 分支与它共用。测试用一个本地 `TcpListener` 服务
packument 与构造的 `.tgz`，断言装上了、库里有行、cache 里的解包目录被清掉；没有配置 registry 时仍是 `Unsupported`。
**仍然缺**：`plugins_catalog_search`、镜像/回退（§4.2）、设置页里的 registry 配置与安装 UI。

### 8.24 Registry 的用户入口

★ 设置页多了一节 **Registry**：输入 base URL，保存写 `plugins.registry`；清空写 `unset`（不是空字符串 —— 空字符串会被读成
一个“配置了但不是地址”的值）。输入框是 store 的缓冲：`ConfigUpdated` 到了会覆盖它，页面不自己写第二份真相。
★ 插件页的安装框改成 **From a directory / From the registry** 两种模式。模式是控件而不是对输入文本的猜测：`@acme/hello`
与 `C:\plugins\hello` 都是“带斜杠的字符串”，猜错的那一次会去碰网络。registry 模式调 `installFromRegistry`，把
`{source:'registry', pluginId}` 交给 `plugins_install`。
★ 测试：设置页写 URL 与清空各一次；插件页切到 registry、输入 id、点 Install，断言命令收到的是 registry source。
**仍然缺**：catalog 搜索、镜像/回退、安装前的权限预览。

### 8.25 安装前先问一句

★ 新命令 `plugins_preview`（契约 45）：读 packument 的内联 manifest，返回 `InstallPreviewDto`（名称、版本、发布者、
capabilities、runtime、发布者声称的体积）。`choose()` 被预览与安装共用，两者不会对“用户看到的是哪个版本”有分歧。
★ 插件页的 registry 安装变成两步：先预览，弹确认框显示权限；确认后才 `plugins_install`。框里的每一句都是发布者的说法 ——
签名在字节上校验，装完默认关闭，所以这个框是**问题**，不是批准。
★ 测试：本地服务返回带 `synra` 内联 manifest 的 packument，先 `plugins_preview` 断言 capabilities，再安装断言行与 cache
清理；插件页挂载测试断言点 Install 只触发 `plugins_preview`、确认后才触发 `plugins_install`。
**仍然缺**：catalog 搜索、镜像/回退。

### 8.26 搜索也接上了

★ `plugins_catalog_search` 从“已声明、未注册”变成真命令：请求 npm 的 `/-/v1/search`，查询固定带
`keywords:synra-plugin`（空搜索就是它本身），映射到已有 `CatalogItem`。`publisher.username`、`installed` 来自本机插件表；
名字不是插件 id 的结果被过滤 —— 搜索命中不等于可安装。
★ `capability_count` 在搜索里是 0 而不是猜一个数：搜索响应没有 manifest，权限列表在**预览**里显示。插件页给每条结果一个
**Preview** 按钮，走的还是 §8.25 的同一个确认框，所以搜索不能绕过权限确认。
★ 测试：本地搜索响应里混入一个非法名字，断言只返回合法项、`installed` 为真；fake kernel 与契约 arity 表同步
（`plugins_catalog_search` 现在注册，README 40/45/23）。
**仍然缺**：排序/分页、镜像与回退。

### 8.27 镜像与回退

★ `plugins.registry` 接受字符串或数组。`get_text` 按顺序请求 packument/搜索；`download` 用服务 packument 的镜像，再按同一
路径尝试其它已配置镜像 —— 一个把 tarball 指到未配置 origin 的 packument 仍被拒。
★ 回退条件只有网络错误与 404/408/429/5xx；401/403 立即拒绝，不再问下一个镜像。tarball 下载成功后只有**一次**
`install_directory`，所以签名/摘要失败不会、也不能触发镜像重试 —— 回退发生在字节到达之前。
★ 测试：镜像 A 给 packument 但拒自己的 tarball，镜像 B 同路径给包；断言安装成功且 B 被请求过（A 只重试一次也一样会失败
这个断言）。
**仍然缺**：设置页的镜像列表编辑（数组目前写在配置文件里）、排序/分页。

### 8.28 镜像列表的编辑

★ 设置页的 Registry 从单个输入变成可增删的列表：每行一个 URL，顺序即尝试顺序。保存时 1 个写字符串（旧配置的形状），
多个写数组，全部清空写 `unset` —— 打开页面不会把一个旧配置改写成另一种形状。
★ 输入用 `:value` + `@update:value` 而不是 `v-model` 到 `registryUrls[index]`：`noUncheckedIndexedAccess` 让那个表达式是
`string | undefined`，而输入框的 `value` 不接受 `undefined`。
★ 测试：两个镜像载入为两行；删掉一行后保存成单个字符串；再加一行并填值后保存成数组。
**仍然缺**：搜索的排序/分页。

### 8.29 搜索的分页与排序

★ `CatalogQuery` 多 `offset` 与 `sort`（`CatalogSort`）。offset 映射到 npm 的 `from`；排序只有 `Relevance`（索引自己的
顺序）与 `Name`（本地按标题比较）—— npm 搜索没有 `sort` 参数，设计里其余的排序（下载量、更新时间）是搜索响应不带的
数字，所以不提供，而不是换个名字返回 relevance。
★ 插件页每页 20 条；最后一页满才有 **Load more**（没有 total 时唯一的“可能还有”信号）；下一页 offset 是已显示条数，
重复结果按 pluginId 去重。
★ 测试：本地服务断言请求带 `from=0`，两个乱序结果在 `Name` 下按名称返回；挂载测试点 Load more 后断言命令收到
`offset: 20` 且新结果被追加。
**仍然缺**：镜像已做（§8.27），排序里的下载量/更新时间要等搜索端点提供数字。

### 8.30 接收端也能取消

★ 以前 `transfer_cancel` 只停发送端的 worker；接收中的行只能等对端放弃。现在两条路径分开：
- 发送端：有 worker 的取消令牌 → 停 worker、删临时文件、发 `TransferFinished`（原行为）。
- 接收端：连接线程独占那条链路，命令**不能**从外面写 abort。`AppState.inbound_cancels` 收下 transfer id，listener 在
  两条消息之间消费它，调 `Inbox::cancel`：发 `UserCancelled` abort、删半成品、清状态，再发一个 `aborted` 进度事件 ——
  `withdraw_received` 由这个事件关掉行，所以行不会在“还没告诉发送者”时先消失。
★ `serve_with`/`handle_within_with` 多一个 `CancelCallback`（`Fn(&str) -> bool`），`serve`/`handle` 包装传“永不取消”；
检查在 `absorb` **之前**，所以取消不会再多写一个 block。
★ 测试：接收方向的 `transfer_cancel` 只把 id 放进集合、行保留；已有发送测试补上“登记取消令牌”，否则它们现在会被当成
接收方向（正是这两个测试先失败，才发现“有 token 才是发送端”这条区分）。
**仍然缺**：界面还没有取消按钮（发送与接收都没有），这条命令目前只有 API 可达。

### 8.31 传输行上的取消

★ 首页 “Sending now” 的每一行加了一个 **Cancel**：调 `transfer_cancel`，两个方向同一个命令 —— 行不需要知道内核是去停
worker 还是把 id 留给 listener。按钮在请求期间 loading，失败走 `ErrorNotice`；行由 `TransferFinished` 事件移除。
★ 测试：向 store 塞一条 `transferUpdated`，点 Cancel，断言命令收到 `{ transferId }`。
**仍然缺**：没有进度条（行只显示对端与状态），也没有“取消前的确认”。

### 8.32 传输行的进度

★ 同一行现在有一条 `AProgress` 与一句 “5 B of 10 B”：百分比来自 `receivedBytes / totalBytes`（total 为 0 时为 0，上限 100），
字节数用 `formatBytes`（B/KiB/MiB/GiB/TiB，一位小数）。
★ 为什么两个都要：只写 “transferring” 是一个用户无法判断的状态，只有进度条没有数字则无法在报告里引用。测试断言
“50%” 与 “5 B of 10 B” 同时出现。
**仍然缺**：取消前的确认。

### 8.33 取消前先问一句

★ Cancel 外面包了 `APopconfirm`（“Abandon this transfer?” / **Abandon** / **Keep sending**）：取消会丢掉可能已传了大部分的
字节，一个手滑就能按到的按钮迟早会被按错。
★ 测试：点 Cancel 只打开问题、命令**没有**被调用；再触发确认才断言 `transfer_cancel` 收到 id。
**仍然缺**：没有“不再问”的选项（每次取消都问）。

### 8.34 确认是一次偏好

★ 新增配置 `transfers.confirmCancel`（默认**问**；缺席等价于 true）。设置页 “Transfers” 一节给了开关；首页按这个值决定
Cancel 外面是 `APopconfirm` 还是按钮本身。
★ 默认是问而不是不问：丢字节的按钮默认应当有一次摩擦，用户明确关掉才省掉。
★ 测试：设置页第二个开关写 `{ transfers.confirmCancel: false }`；首页在 store 里放 false 时没有 `APopconfirm`、一次点击
就调 `transfer_cancel`。
**仍然缺**：主题仍是只读；每窗口 CSP。

### 8.35 检查更新

★ `plugins_check_update`（契约里一直是 `Maybe`，现在注册）把已安装版本与 registry 的 `latest` 比较；版本不同则返回
`UpdateInfo{ version, needsNewConsent, addedCapabilities }`，其中新权限 = 新版内联 manifest 有、本机 manifest.json 没有的
capability。相同版本返回 `null`。
★ 检查是**用户动作**（插件卡上的 “Check for updates”），不是挂载时的 fan-out：十个插件就是十次请求，打开页面不该依赖
registry 可达。结果存在页面本地的 `updates` map 里，显示版本与新增权限。
★ 测试：本地 packument 最新 2.0.0 + 内联 manifest 多要 `device:send:*`，断言 `addedCapabilities`；页面点击后断言命令参数
与提示文本。
**仍然缺**：`plugins_update`（真正安装新版本）仍未实现，所以卡片只报告“有新版本”，没有“更新”按钮。

### 8.36 真正安装新版本

★ `plugins_update`（契约里是 `Unit`，现在注册）：registry 取 `latest` → 下载 → 与安装**同一条**校验/写盘管线
（`install::update_from_directory`，内部是 `from_directory` 的 replace 分支）→ 目录切换 → 记录新版本。
★ 顺序与回滚：先完整校验并写到 staging；再把旧目录改名为 `.install-staging/previous-<id>`；新目录就位；`plugin_versions`
记新行；全部成功后才删除旧目录。任一步失败都把旧目录改回去 —— 更新失败等于没有更新，而不是两个版本的混合。
★ **权限变了就关掉**：新旧 manifest 的 capability 集合比较；新版本多要任何一项，插件回来时 `enabled=false` 且
`trusted=false`（旧批准只覆盖旧清单），由同意屏幕重新问；权限没变则保留用户原来的开关与信任。
★ 目录树删除移到 `synra_store::remove_tree`（`plugin_files.rs` 是守卫允许的两个 owner 之一），`install.rs` 不再调用
`remove_dir_all`；`remove_tree` 带 `is_within` 检查。
★ 插件卡的 “Update” 按钮只在检查到新版本时出现；更新后重新读列表。
★ 测试：daemon 两条（同权限保留 enabled/trusted；多一个 capability 则两者都清），更新后 active 版本与 digest 已换、无备份残留。
**仍然缺**：安装前的 `synra-plugin pack`（npm 兼容 `.tgz`）还没有，registry 发布仍是手工。

### 8.37 `pack`：作者侧的最后一步

★ `synra-plugin pack [--dir] [--out] [--dry-run]`：读 `manifest.json`，只打包 manifest 点名的 artifacts + 可选的
`signature.json`，写成 `package/` 前缀的 `.tgz`；默认输出 `dist/<id>-<version>.tgz`。
★ 可复现：mode `0644`、mtime `0`、成员顺序固定 —— 同样的输入两次打包字节相同，registry 的摘要不会因为构建时钟变化。
★ 测试 4 条：用 **`synra_registry::extract_tgz`**（registry 安装实际用的解包器）读回三个文件；两次打包字节相等；缺 artifact
时点名拒绝；默认文件名是 `@acme-hello-2.0.0.tgz`。
★ **先跑 `verify`**：pack 在写文件之前执行与安装同源的检查；失败时返回报告（exit 1）而不是出包。
★ **签名覆盖检查**：存在 `signature.json` 时，pack 调安装侧同一个 `sign::verify`（空 trust store）核对 manifest 与所有
artifact 的摘要；签名后改过 wasm 或 manifest 都会被拒绝。测试三条：正确签名随包、换 wasm 拒绝、改 manifest 拒绝。
★ 发布仍然不在这个工具里：`npm publish dist/*.tgz` 或上传到自建 registry；账号、scope、策略属于发布目标。
★ **`--registry <url>`**：只校验形状（https、scheme+host、无路径；http 仅 loopback），通过后把地址变成
`npm publish --registry <url> <file>` 一行，**不发请求**。
**仍然缺**：`publish` 子命令、静态 index 适配层。

### 8.38 静态 index 的解析

★ `synra-registry::parse_static_index`：把桶上能放的最小文档（`name` / 可选 `latest` / `versions[]`，每项带
`version`、`tarball`、可选 `integrity`/`synra`/`manifest`）解析成与 packument 相同的 `Packument`，安装路径不需要知道源的
形状。
★ `latest` 缺省取**列表最后一项**：没有 semver 实现，文本排序会把 `1.10.0` 排在 `1.9.0` 前，回退顺序交给作者。
★ 拒绝：空列表、重复版本、`latest` 指向未列出的版本。
★ 测试 6 条（显式 latest、缺省最后一项、latest 缺失、重复、内联 manifest 解析、非 JSON）。
**仍然缺**：客户端还没把 `plugins.registry` 接到静态 index；缓存/ETag 留给接入时处理。

### 8.39 静态 index 接进客户端

★ `plugins.registry` 的地址加 `static:` 前缀即静态 index：按 `{base}/{pluginId}.json` 取文档，`parse_static_index` 解析成
`Packument`；安装/预览/更新走同一条路径。
★ 静态 index 的 tarball 是完整 URL，允许在另一个主机（CDN）；npm 地址仍要求同 origin 或已配置镜像的同路径。
★ 搜索只在 npm 兼容地址上做；全是静态时返回 `Unsupported`，不把“没有索引”说成“没有匹配”。
★ 测试：本地服务 `hello.json` + `pkg.tgz`，配置 `static:http://…`，安装成功；全静态时搜索 `Unsupported`。
**仍然缺**：`index.json` 的缓存/ETag、静态源的签名目录（signature 清单）。

### 8.40 静态 index 的条件请求

★ `AppState.registry_cache`：按 URL 存 `{ etag, last_modified, document }`。第二次取同一个静态 index 时带
`If-None-Match` / `If-Modified-Since`；304 直接复用解析结果，不读 body。锁在请求前释放，网络往返不占锁。
★ **`synra-fetch` 的修正**：`304 Not Modified` 不是重定向，没有 `Location`。之前所有 3xx 都走进重定向分支，于是一个
缓存命中被报成“304 没有 location”的协议错误。现在 304 作为普通响应返回，测试锁定。
★ 测试：本地服务第一次回 200 + `etag: "v1"`，第二次只在带 `If-None-Match: "v1"` 时回 304；第二次 `plugins_check_update`
仍得到缓存里的版本（如果缓存没生效，服务器会回一个不同的 3.0.0 文档）。
**仍然缺**：缓存失效（目前进程内不过期，靠 ETag 续期）、静态源的签名目录。

### 8.41 `max-age` 到期就不再验证

★ `CachedDocument` 多 `expires_at_ms`；`Cache-Control: max-age=N` 记为 `now + N`。过期的条目不再作为条件请求的候选：
服务器已经说了寿命，过期后再问“变了吗”只是拿第二个意见。
★ 304 若带新的 `max-age` 则续期；`no-store`/`no-cache` 视为没有寿命（每次重新验证）。
★ 测试：第一次 200 + `max-age=0` + `etag: "v1"`；第二次**不带** `If-None-Match`、读到新版本 4.0.0。
**仍然缺**：静态源的签名目录。

### 8.42 目录可以声明发布者公钥

★ 静态 index 多一个可选 `publishers`（发布者 id → fingerprint），`synra-registry` 的 `parse` 返回
`StaticIndex { document, publishers }`；安装/预览走的仍是同一个 `Packument`。
★ `InstallPreviewDto` 多 `publisherKey`；预览对话框在下载 tarball 之前就能显示“目录说这个发布者的钥匙是 …”。
★ 它是**声明**而不是验证：真正的判断是对下载字节验签；目录与签名不一致时是一条给人看的事实，不是自动结论。
★ 测试：解析 `publishers`；静态源 `plugins_preview` 返回该指纹；npm 源为 `None`。
**仍然缺**：两侧不一致时的界面提示（现在只在预览里显示目录声明）。

### 8.43 目录与签名不一致时说出来

★ `registry::install` 安装完成后比较：目录 `publishers[publisherId]` 与 `InstallReport.fingerprint`；不同则往
`warnings` 追加一条同时写出两个值的句子，界面把它跟在“装上但关闭”后面。
★ **不是拒绝**：两份文档都出自作者，任何一份都可能过期；拒绝会让一个 `publishers` 的笔误挡住一次发布，沉默则浪费目录
提供的唯一第二个意见。
★ 测试 3 条：匹配静默、不同则句子含两个值、缺任一侧不算不一致。

### 8.44 `pack` 顺手写静态 index

★ `synra-plugin pack --index <base>`：在 tarball 旁边写 `<pluginId>.json`（`dist/@acme/hello.json`，与客户端的
`{base}/{pluginId}.json` 同形）；`tarball` 用 `{base}/{文件名}`，`manifest` 取自 manifest 文件，`publishers` 取自刚核对过的
签名（无签名则空）。
★ `--index` 的 base 允许路径（桶前缀），仍要求 https / loopback http。
★ 测试：签名包写出的 index 能被 `parse_static_index` 读回（latest、tarball URL、publisher key、内联 manifest）；无签名时
`publishers` 为空。
**仍然缺**：多版本 index 的累加（现在每次只写当前版本，发布多个版本需要作者合并）。

### 8.45 多版本 index 是追加

★ `merge_index`：目标已存在时读取，保留其他版本、替换同版本条目、`latest` 指向当前版本、`publishers` 合并；同版本重打包
替换而不是列两次（解析器拒绝重复）；读不懂的现有文件拒绝覆盖。
★ 测试 3 条：两次 pack（1.2.3 → 1.3.0）后 index 有两条且 latest 是 1.3.0；同版本第二次合并替换 tarball；非 JSON 拒绝。

### 8.46 相对 tarball

★ 静态 index 的 `tarball` 可以是相对路径：客户端用 index 自身的 URL 解析（同级、`/` 开头的 origin 根、绝对 URL 直接放行），
`..` 与空路径拒绝。
★ `pack --index` 不带值时写相对文件名；带值时仍写 `{base}/{文件名}`。
★ 测试：客户端解析三种形式、拒绝 `..`/空；静态安装测试改用 `"pkg.tgz"`；`pack --index`（空值）写出的 index 里 tarball 是
`one.tgz`。
**仍然缺**：`publish` 子命令；主题切换；每窗口 CSP。

### 8.47 `publish`：发之前自查

★ `synra-plugin publish --index <file> [--base <url>]`：解析 index → 定位每个 tarball（相对路径按 index 目录，绝对 URL 用
`--base` 映射）→ 用安装侧的 `synra-registry::extract_tgz` 解包 → 有 `signature.json` 就用安装侧同一个 `sign::verify` 核对 →
检查 inline manifest 点名的 artifacts 都在。**不上传**。
★ 解包在临时目录，清理逐个文件/目录（不用递归删除）；失败时成功的版本行仍保留，作者一次就能看到哪几个还没过。
★ 默认布局修正：`pack --index`（相对）把 tarball 放进 index 所在目录（`dist/@acme/@acme-hello-1.0.0.tgz`），index 在
`dist/@acme/hello.json`；否则相对名解析到另一个目录。
★ 测试 4 条：打包产物通过检查；缺 tarball 点名；签名后改 manifest 被拒；绝对 URL 需要 `--base`。
**仍然缺**：上传本身（`aws s3 cp` 或等价物）；主题切换；每窗口 CSP。

### 8.48 `publish --verify`：回读桶

★ `publish --verify --base <url>`：GET `{base}/{pluginId}.json`，用 `parse_static_index` 解析，按版本比对；每个 tarball
按相对/绝对规则解析后 GET，与本地文件比 SHA-256（不比大小：同尺寸的不同文件正是大小检查漏掉的）。
★ 网络走 `synra-fetch` + 宿主 `NetworkGuard`：https 默认，http 仅 loopback；响应上限 64 MiB + 1。
★ 测试 2 条：本地服务回读通过并打印 `digest matches`；桶里 tarball 改一个字节 → 报 `different bytes`。
**仍然缺**：上传本身；回读失败后自动重传（现在只报告）。

### 8.49 插件页自带 CSP

★ `plugin_ui::document_response`：`synra-plugin://` 的 200 响应加 `Content-Security-Policy`，策略与 09 §416 一致
（`default-src 'none'`、`connect-src 'none'`、`script-src 'self' 'unsafe-inline'`、`form-action 'none'`）。
★ 理由：Tauri 2.12 没有公开 per-window CSP，页面只继承 `tauri.conf.json` 的全局策略；把策略放进文档响应头后，
`connect-src 'none'` 不再依赖一个页面看不见的设置。两者同时生效时浏览器取交集。
★ 测试：`document_response` 的状态与四个关键指令（默认 / 连接 / 脚本 / 表单）。
**仍然缺**：`'self'` 在自定义协议 origin 下的真实语义仍需一次运行确认；上传本身。

### 8.50 关闭窗口即结束会话

★ `plugin_ui::forget_window`：删除 `hosted_ui` 条目并 `ui_sessions.revoke(nonce)`。`plugin_ui_open` 在窗口
`Destroyed` 时调用它；`CloseRequested` 可能被取消，所以不用它。
★ 修的是“只增不减”：此前 `hosted_ui` 没有任何删除点，一个关掉的页面仍能通过 `plugin_ui_ready` 与协议 handler 得到回答，
同 label 的新窗口还会继承旧 nonce。
★ 测试：会话存在 → `forget_window` → `hosted_ui` 无条目且 `validate` 拒绝。
**仍然缺**：多窗口（同一插件两个页面）的会话计数 —— 现在一个 label 一个会话，关掉一个就结束。

### 8.51 一个插件一个页面

★ `plugin_ui::window_label(id)` 从 id 派生窗口 label（`plugin-@acme__hello`）；`open_request` 用它，`plugin_ui_open` 也用它。
★ `plugin_ui_open` 先 `app.get_window(label)`：已有 → `show` + `set_focus` 后返回，不再建第二个窗口、不再发第二个令牌。
此前第二次打开会发新 nonce、覆盖 `hosted_ui` 里的旧条目，第一个页面的令牌就没有表项了。
★ 为拿 `Manager::get_window`，`tauri` 依赖启用 `unstable` feature（2.12 的该方法在该 feature 下）。
★ 测试：`window_label` 对 scoped id 的映射（`plugin-@acme__hello`）。
**仍然缺**：窗口已存在时的“聚焦”在无 GUI 的单测里验证不了，只能依赖 Tauri 的 `show`/`set_focus`。

### 8.52 打开是原子的

★ `open_request` 返回 `OpenDecision::{Open, AlreadyOpen}`，在 `hosted_ui` 的锁内检查并插入；插入若发现旧条目，放回旧条目、
`revoke` 本次多发的 nonce 并返回 `AlreadyOpen`（快速点击不再各发一个令牌）。
★ 建窗失败时 `plugin_ui_open` 调 `forget_window`：一次失败不能让插件永久处于“已打开”而窗口不存在。
★ 测试：第二次 `open_request` 返回 `AlreadyOpen`、`ui_sessions.issued()` 不变、`hosted_ui` 仍只有一条。
**仍然缺**：聚焦本身无 GUI 不可测；上传本身。

### 8.53 回读失败时点名重传

★ `publish --verify` 在失败时，对每个被点名的版本追加 `re-upload <version> <local file> -> <url>`，并加一行
`upload these again:`。本地路径按 index 目录/`--base` 解析，URL 按远端 index 解析 —— 两个值都由工具算，作者只复制。
★ 测试：桶里 tarball 改一个字节时，输出同时含 `different bytes` 与 `re-upload … 1.0.0`。
**仍然缺**：自动重传；上传本身。

### 8.54 `upload`：用作者的命令上传

★ `synra-plugin upload --index <file> --base <url> --command '<template>' [--dry-run]`：计划 = index 本身 + 每个版本
tarball（本地路径按 index 目录/`--base` 解析，URL 按 `--base` 拼），逐个执行模板。
★ 模板按空白切分，`{file}`/`{url}` 逐 token 替换，**不经过 shell**；路径含空格仍是单个参数。缺任一占位符拒绝启动。
★ 失败即停，报告哪一条命令退出码非零、其后文件未上传。`--dry-run` 只打印命令。
★ 测试 4 条：计划顺序与 URL、模板展开（含空格路径）、缺 `{url}` 拒绝、dry-run 不执行且打印两条。
**仍然缺**：上传后自动 `--verify`；模板中的引号/环境变量（有意不支持，避免再次引入 shell）。

### 8.55 上传后自动回读

★ `upload --verify`：全部上传命令成功后调用 `publish --verify` 的检查（同一个 `publish_plugin`），输出合并；失败时附
`re-upload` 清单并以非零退出。
★ `--dry-run` 不上传也不回读：dry run 没有新字节可读，去回读只会把旧桶的状态说成这次上传的结果。
★ 测试：`--dry-run --verify` 指向不可达地址仍成功（证明没有发起网络请求）。

### 8.56 契约里那条更简单的搜索注册了

★ `plugins_request_registry_search(query)` 是契约里 `plugins_catalog_search(CatalogQuery)` 的简化入口：同一个端点、
只给文本。它此前挂在 `NOT_YET_REGISTERED` 上，理由是“没有接 registry 搜索”，而端点在 `plugins_catalog_search` 落地时就
已经存在 —— 于是它成了一个长期假缺口。
★ 现在注册为对 `plugins_catalog_search` 的薄包装（`registry/limit/offset/sort` 全为 `None`）；命令清单 45 条中注册 43 条。
★ 清单、fake kernel 的参数表与返回都同步；README 的计数由 `shapes` 的 `readme-counts` 守住。

### 8.57 npm packument 也走条件缓存

★ `packument_with_base` 的缓存不再只对静态 index 生效：npm packument 以同样的 `ETag`/`Last-Modified` 缓存，304 直接复用
解析结果。验证器属于注册表，与它说什么协议无关。
★ 测试：本地 npm 注册表第一次回 200 + `etag: "p1"`，第二次只在带 `If-None-Match: "p1"` 时回 304；两次
`plugins_check_update` 都得到 2.0.0，且共两次请求。
★ 跨进程缓存见 §8.58。

### 8.58 缓存跨进程

★ 文档与验证器写到 `<cache>/registry-documents/<sha256(url)>.json`，内容 `{ version, url, kind, etag, lastModified,
expiresAtMs, body }`。`packument_with_base` 先查内存，未命中再读盘，仍按条件请求使用验证器；200 时同时写盘。
★ 读盘时：格式版本不符、URL 不符、`kind` 不符 → 忽略；`max-age` 已过期 → 删文件并忽略。写盘失败被忽略（缓存是优化，
不是正确性）。
★ 测试 3 条：文档+验证器可跨进程读回（含 `publishers`）；过期文件被删；未来版本的文件被忽略。
**仍然缺**：缓存文件的总量上限/清理策略（现在只在读取过期文件时删）。

### 8.59 旧缓存会被清理

★ `registry::prune_documents(state, now)`：删掉 `registry-documents/` 里 mtime 早于 30 天的文件，返回删除数；
`AppState::new` 每次构造时调一次（不是每请求）。
★ 为什么需要：只带 `ETag` 的文档没有 `expiresAtMs`，读路径永远不会删它；没有 prune，目录会随每个问过的注册表一直增长。
★ 测试 2 条：40 天前的文件被删；新文件保留且 `removed == 0`。
**仍然缺**：总量上限（时间维度有了，空间维度还没有）。

### 8.60 缓存也有了空间上限

★ `prune_documents` 先按 30 天 TTL 删，再在剩余总量超过 16 MiB 时按 mtime 从旧到新删到阈值以下；`prune_directory(dir,
now, max_bytes)` 把上限作为参数，测试可以用小值。
★ 为什么时间不够：一个脚本一个月内问几千个注册表，每个文档都“新”，合起来仍然是一整个目录。
★ 测试 1 条：两个 10 字节文件、上限 10 字节 → 较旧的那个被删、较新的保留、`removed == 1`。
**仍然缺**：没有（缓存策略到此完整）。

### 8.61 审计时间线到了插件页

★ `plugins_audit` 从“已注册、没有调用方”变成插件卡上的一个折叠区：每行一条能力调用（capability、允许/拒绝、时间），
拒绝的行用 danger 色并把 `reason`（闸门的稳定结果名，例如 `denied`）写在旁边。空环与读不到是两句不同的话：
`[]` 说 “No calls have been recorded.”，拒绝给 `ErrorNotice` —— 一个读不出审计的宿主不能把“没有记录”说成结论。

★ **拉取而不是挂载时扇出**：页面打开时**不**问任何插件（十个插件就是十次 IPC），展开时才调
`plugins_audit { pluginId, limit: 20 }`；而且**每次展开都问**，因为环在内存里，一次读到的记录在用户回来时已经过期。
`undefined`（没问过）与 `[]`（问过、没有）因此是两种状态，空态只在后者出现。

★ **顺序在动作层翻转**：命令答的是环的**尾部**、旧→新（`audit_for` 按发生顺序报），而读的人问的是“最近在做什么”。
`pages/plugins/actions.ts` 的 `audit()` 返回前 `reverse()`，页面不再对一份顺序已知的列表排序。
测试钉住两端：动作层断言第一条是最新的，挂载测试断言 DOM 的第一行是后发生的那条。
（§8.64 之后命令直接读表、SQL 已是新→旧，这个反转被删掉了。）

★ **键里带下标**：同一毫秒内两次调用同一能力是可能的（环会都记），只用 `atMs-capability` 会让 Vue 把两次调用画成一行。

★ **没有做的**：完整历史（只有内存环，没有 DB 读取路径，30 天保留期那张表没有读者）、
每条的 `elapsedMs`/`traceId`（DTO 有，页面不显示；`trace_id` 仍然是 `None`，因为记录里没有 trace）。
清空当时也还没有，紧接着的 §8.62 把它补上了。
测试：动作层 3 条（参数与 limit、拒绝的映射、最新在前），挂载 2 条（展开才读且画出两行、拒绝走 `ErrorNotice`
而不是空态）。

### 8.62 清空一个插件的审计

★ `plugins_audit` 只回答了“它做过什么”，[18 §5.2](./18-observability.md) 要的另一半是“用户可以把这一个插件的记录清掉”。
新增 `plugins_audit_clear`（契约里第 46 条，`Unit`、`mutates: true`）：壳的 `commands::plugins::plugins_audit_clear`
拿 `plugin_set()` 然后 `clear_audit`，后者是 `PluginSet` 上的一行 —— `AuditRing::clear_plugin` 早就存在且有测试，
缺的是从命令到它的这条缝。

★ **为什么是“一个插件”而不是整环**：动作发生在某张插件卡的展开区里，整环清空会让一张卡上的按钮擦掉另一张卡上的记录。
`AuditRing::clear_plugin` 同时清掉计数 —— 一份“已清空却仍报总数”的历史等于没清。

★ **没有引擎时拒绝，而不是静默成功**：与 `plugins_audit` 同一个答案（`Unsupported`）。一个读不出记录的构建
不能声称它把记录删了。

★ **持久那份不在这个命令里**：`plugin_audit` 表有 schema、没有读者，也没有被写进它的生产路径（ring 是这一轮唯一的
真实轨迹）。所以命令的文档注释写的是“清掉本进程的环”，不把“历史”说大；等 DB 有了读者，清空应当在同一个调用里做。
（§8.64 把它接上了：现在同一个命令删表里的行。）

★ **界面**：Clear history 只在那一节**有行**时出现（清一个空列表的按钮只会在原地重画那句空话）；不弹确认 ——
清的是“插件做过什么”的日志，不是插件产出的数据，而它已经在用户主动展开的那一节里；页面清完直接写 `[]`，
不再读一次（命令刚删了全部，读回来的答案已知）。

★ 测试：core 1 条（清 A 不动 B）、壳 1 条（没有引擎时拒绝而不是成功 —— §8.64 之后这条答案被移除）、动作层 2 条（参数、拒绝映射）、
挂载 1 条（点一下、命令收到 id、行消失而空态出现、`plugins_audit` 只读了那一次）；
两处“契约条数”的硬编码从 45 改为 46，README 的 46/44/28（§8.63 之后被调用数变成 29）由 `readme-counts` 盯着。
**仍然缺**：`trace_id` 仍是 `None`（DB 那份历史与清空见 §8.64）。

### 8.63 性能面板与它需要的类型

★ `diag_metrics` 之前“注册了、没有调用方”，而 [18 §4.2](./18-observability.md) 把设置页的性能面板列为记录器存在的
理由之一。现在设置页底部有一节：挂载读一次、每 5 秒重读（`onUnmounted` 清 interval），Refresh 立刻读；
失败走页面既有的 `ErrorNotice`，notice 的 Retry 改成同时重读配置、身份与指标 —— 否则“重试”不会重试出错的那一个。

★ **“实时”是轮询而不是事件**：指标没有事件流（`KernelEvent` 里没有 metric 变体，也不该有 —— 一条每次调用都发的
事件会把观测成本变成 IPC 成本）。5 秒是页面能接受的频率，而 `diag_metrics` 只是读一个内存记录器。

★ **键用内核的拼写**：`SeriesKey::render` 的 `name{label=value}` 与日志行、诊断包一致；页面另起一种写法会让
“报告里的那个指标”和“面板里的那个指标”变成两个要对照的东西。直方图只说两个数：观测次数与平均值
（`sum/count`，两位小数）；没有观测时平均是 `—`，不是 0 —— 0/0 写成 0 会让从未发生过的直方图看起来像 0 毫秒。

★ **类型先补进 `@synra/protocol`**：`MetricsSnapshot` / `SeriesValue` / `SeriesKey` / `MetricValue` /
`HistogramSnapshot` 住在 `synra-observe` 而不是 `synra-ipc` 的 DTO 目录，所以 `dto-coverage` 的扫描根加了第三个
（`synra-observe/src/metrics`），否则这几个 interface 会被报成“没有同名 Rust struct”。
**而这一步顺手抓到了两处真缺口**：`PublisherDto` 与 `InstallPreviewDto` 在 Rust 里早已存在，镜像里没有 ——
界面当时正手抄它们（`PublisherRow` / `InstallPreview`），而手抄的 shape 不会在 Rust 改字段时编译失败。
现在两份都在 `ui.ts` 里，插件页改用 `@synra/protocol` 的类型。

★ 测试：js 侧 `shapes` 14 项（31 个 DTO 逐字段对照）、`pnpm test` 34 条、`pnpm run check` 全绿；
app 侧设置页 2 条新测试（键/值/平均值、overflow 句子），全量 246 条。
**仍然缺**：指标名还是“谁写谁起名”（§4.1 的清单没有全部落地），面板只画当前值、没有趋势。

### 8.64 审计以表为准：读、清、保留都接上

★ **`plugins_audit` 改读 `plugin_audit`。** 表从第 69 轮起就有写者（两个宿主的 `audit_sink` 都在每次调用后同步写一行），
但命令一直读内存的环 —— 于是“最近活动”在重启后是空的，虽然用户能在别处看到昨天的拒绝。现在
`audit_trail_for(plugin, limit)` 一条 SQL 回答：`WHERE plugin_id = ?1 ORDER BY at_ms DESC, id DESC LIMIT ?2`，
索引 `(plugin_id, at_ms DESC)` 正是为它建的。**表是环的超集**（sink 在 `drain_audit` 里同步写），所以读表不会漏掉环里的东西，
而环仍留给 `refusals_for` 与诊断包用。

★ **由此去掉“没有引擎 → Unsupported”的答案**：轨迹比引擎活得久，一个跑不了插件的构建仍然能回答这个数据库记了什么。
上一轮的测试 `without_an_engine_the_audit_command_says_so...` 因此被删掉，换成“fixture 没有引擎也能读到行”。

★ **顺序由 SQL 保证，前端不再反转**：`actions.ts` 的 `audit()` 原来把环的尾部翻成“最新在前”，现在原样返回；
动作层测试与挂载测试的 fixture 都改成新→旧，并删掉那句“命令答的是环的尾部”。

★ **清空同时删两份**：`plugins_audit_clear` 调 `forget_audit_for(plugin)` 删表里的行，并且（有引擎时）
`PluginSet::clear_audit` 清环 —— 只删一份会让另一份在下次读取或诊断包里把它救回来。没有引擎时跳过环那一半而不是拒绝。

★ **保留期有了执行者**：`AppState::new` 构造时调一次 `prune_audit(now)`，删掉 7 天以前的记录。这个数字在守护进程里
早就写死（`AUDIT_RETENTION_MS`），现在两个宿主一致；`audit_retention_days` 这个配置键**仍然没有**进 `Catalog`，
所以命令的文档注释写的是“默认值”，而不是假装它能被设置。和 registry 文档缓存一样，prune 失败是诊断，不是拒绝启动。

★ **outcome 的拼写只有一份**：表里存的是 `AuditOutcome::as_str()`，所以读的时候要反过来解析。
`AuditOutcome::from_stable_name` 放在 `as_str` 旁边，测试遍历所有变体做往返。一个本构建不认识的
名字被报成 `allowed: false` + 名字本身 —— 把一个来自更新构建的结果显示成“allowed”是唯一可能掩盖拒绝的答案。
同一条映射里 `failed` 是 **allowed**（闸门放行了、操作自己失败），用字符串比较 `"allowed"` 会把它画成拦截。

★ 测试：observe 1 条（名称往返 + 未知名）、store 3 条（按插件读且限流/排序、limit 0、按插件删且不动别人）、
壳 4 条（行的映射含 failed 与未知名、无引擎也能读、清空只清一个插件、7 天 prune）、
前端 2 条跟随顺序变化；全量 245（壳）/ 246（前端）。
**仍然缺**：保留期的配置键；UI 仍只显示最近 20 条（完整历史没有翻页 —— §8.67 补上了）。

### 8.65 意图与插件的指标：壳这一侧补两组

★ **意图**：`intent_dispatch` 的计数与时长现在真的在写。原先它只是“路由 → 发布事件”，没有任何指标；而 18 §4.1 里
`intents_total`、`intent_duration_ms`、`intent_no_handler_total` 三个名字一直只有清单。现在路由与计时分成
`route()` + `record_dispatch()`：计时在**两个出口**都发生（成功与失败），因为一次被拒绝的分享也花了匹配的时间，
只由成功构成的分布会用一个错误的问题的答案回答“分享有多慢”。

★ **标签与守护进程对齐，这一条是本轮唯一一次返工**：第一版写的是 `intents_total{result=<Routed::state>}`
（`noHandler`/`choosing`/…），查守护进程时发现它从第 14x 轮起就写 `intents_total{outcome=ok|refused}`。
同一个计数器在两个宿主里各有一种拼写，正是这份文档反复记录的那种漂移。现在两个宿主都用 `outcome`；
**settle 在哪个阶段**由卡片的 `state` 说，不由指标说。

★ **时长也补到守护进程**：`intent_duration_ms` 原来是壳新加、守护进程没有 —— 那会让“两个宿主的同一件事”在快照里
只有一半。守护进程的 share 路径同样在 `?` 之前 observe；那条已有的计数器测试多断言一次直方图的 `count >= 1`。
（不用精确值：一次 dispatch 只 observe 一次，但断言“至少一次”不会因为夹具多做一次路由而碎。）

★ **插件两个 gauge**：`plugins_installed` / `plugins_active` 在 `plugins_list` **读库时** set，而不是在每次写操作里
increment：写失败半途不会留下一个要补正的计数，而被外部工具改过的数据库在下一次列表里就是对的。`active` 是用户
的 `enabled` 开关，不是“本进程载入了几个”—— 一个 faulted 但 enabled 的插件仍然是开启的。

★ 测试：壳 247 条（新增“一次 dispatch 写下 outcome/无处理/时长”与“两个 gauge 分别是 2 与 1”），
守护进程的意图计数器测试扩了一条直方图断言；两边 clippy `-D warnings` 都过。
**仍然没有生产者**：壳侧的网络、发现、传输三组；以及 §4.1 的 `plugin_calls_total`、`transfer_throughput_bps`、
存储与进程那几组。

### 8.66 壳的传输指标：一个汇合点，三种结果

★ **发送组接上了**：`transfers_total{kind,outcome=delivered|cancelled|failed}`、`transfer_bytes{kind}`、
`transfer_duration_ms{kind}`。名字与守护进程已有的两项（`transfer_bytes{kind}`、`transfer_duration_ms{kind}`）一致；
`transfers_total` 多一个 `outcome` —— 守护进程第 144 轮**故意没做**它，因为它的 `send_file` 有多条早退且是 trait
方法、无法包一层。壳不是 trait impl，所以这里能做，而且做了才有“失败率”这个数字。

★ **落在哪两处**：一次传输的开始与结束必须汇合在同一处，否则计数器只会数成功的那部分。壳的路径是
`send_over_with`（拨号）→ `send_ready_since`（逐块）。于是：
* 拨号失败在 `send_over_with` 里记 `failed` —— 这是用户最常问的那种失败，一个块都没动；
* 其余出口（成功、取消、块被拒）在 `send_ready_since` 结尾记一次；
* 时长从 `send_over_with` 进入时开始算，所以含拨号时间；`send_ready_with` 作为直接调用的入口传 `Instant::now()`，
  测试因此也会留下一条记录。

★ **字节数只在 delivered 时记**：`transfer_bytes` 回答“这台设备发出去的文件有多大”，取消/失败的计划大小不是发出
去了多少。取消的那条测试因此断言 size **不存在**，success 的那条断言 sum 等于文件字节数 —— 两条一起钉住这个判断。

★ **收件方向仍然没有**：listener 的 `FinishedCallback` 只交回“到了”（路径、长度、id），没有耗时与方向，
把一个只有一半的 transfer 指标硬塞进去会把“发出”和“收到”混成一个数字。等收件回调带上字节数与时长再接。

★ 测试：壳 247 条通过（成功与取消两条各多一段指标断言），clippy 两种配置 `-D warnings` 通过。
**仍然缺**：收件方向的传输指标；壳侧的网络与发现两组；`transfer_throughput_bps`（需要块级采样）。

### 8.67 完整历史：offset 分页

★ 审计卡原来只能看最近 20 条。表里有整个保留窗口，而 18 §5.1 的草图一直有“查看完整历史”。现在
`plugins_audit` 多一个**可选** `offset`：壳的包装器把它转成 `Option<u32>`（缺省 0），命令交给
`audit_trail_for(plugin, offset, limit)`，SQL 是 `LIMIT ?2 OFFSET ?3` —— 沿 `(plugin_id, at_ms DESC)` 这个索引走下一页，
而不是把已看过的行再读一遍。

★ **可选而不是必填**：现有的调用点只知道 `pluginId` + `limit`，一个必填的 `offset` 会让它们全部变成缺参数；
`Option<u32>` 是 Tauri 唯一“缺席就 None”的形状（守卫 `command-arguments` 也按这个规则判断必填键）。

★ **界面的“还有更多”信号**：limit-only 查询不返回总数，所以唯一的信号是**上一页满页**。卡片在
`!auditDone && 行数 > 0` 时显示 **Show more**，用“已显示行数”作 offset，返回后**追加**；页短就置
`auditDone` 并把按钮拿掉。重复点一次最多多一次空读，而不是一个在加载中消失的控件。

★ **一个只有守卫才会发现的坑**：第一版把“可选”的注释写在参数列表**里面**，而 `command-arguments` 的解析器按
`field: type` 读每个参数 —— 注释粘在 `offset` 前面，`/^\w+$/` 不匹配，于是它只看见 `pluginId`、`limit`，报
“passes offset, which declares pluginId, limit”。注释移到函数上方后通过。**守卫按字面读源码，注释也是字面**。

★ 测试：store 1 条（offset 走同一顺序、越界为空）、壳 1 条（命令把 offset 传下去：第二新的那一行）、
前端 2 条（默认带 `offset: 0`、点 Show more 后请求 `offset: 20` 且行数 20→24、按钮消失）；全量 248（壳）/ 248（前端），
shapes 8 项、clippy 两种配置、guards 12 项。
**仍然缺**：保留期的配置键；收件方向的传输指标；壳侧网络与发现指标。

### 8.68 收件方向的传输指标

★ 发送侧在 §8.66 接上后，`transfers_total{kind,outcome=...}` 只有三个值；现在补上 `received` 与 `aborted`。
挂点只有两处，都在 listener 里：
* **开始**：progress 回调第一次看到某个 transfer 时把 `Instant` 放进一张共享表。`FinishedTransfer` 只有长度与落点、
  没有开始时间，而 progress 回调是唯一知道第一块到达的地方 —— 一张 `Arc<Mutex<HashMap<TransferId, Instant>>>`
  由 finished 与 progress 两个闭包共享，一次传输只有一个开始（重复的 progress 不覆盖）。
* **结束**：`FinishedCallback` 在文件校验并（插件包时）安装之后记 `received`、大小与时长；`aborted` 的 progress
  事件记 `aborted`、**不记大小**。大小直方图描述的是“到达的文件”，与发送侧“取消的没有大小”是同一条规则。

★ **时长可以缺**：abort 若在第一次 progress 之前到达，就没有开始可减。`record_attempt` 因此收 `Option<Duration>`，
没有就**不 observe**，而不是记 0 —— 一个从未移动的传输出现在延迟分布里会让那个分布撒谎。

★ **两个 helper 合并**：`record_send` 与 `record_received` 都走 `record_attempt(state, kind, bytes, elapsed, outcome,
record_bytes)`，于是“哪些结果记大小”只有一个判断。名字仍与守护进程一致（`transfer_bytes{kind}`、
`transfer_duration_ms{kind}`）；`transfers_total` 的 `outcome` 是壳自己的补充。

★ **可验证性的边界（写清而不是假装）**：`listener::spawn` 需要一个 `AppHandle`，所以记录逻辑抽成
`record_received` 由单测断言（`aborted` 无 size、`received` 有 sum 等于字节数）；“两个回调真的调用了它”只有编译期
证据 —— 与 `frames_received_total` 在 listener 里的处境相同，而那是这份文档早先记录过的同一种缺口。
另有一个 `#[cfg_attr(not(feature = "tauri-shell"), allow(dead_code))]`：没有外壳的构建里唯一调用者不存在，但测试还在。

★ 测试：壳 249 条通过（新增 1 条）；两种配置的 clippy `-D warnings` 通过。
**仍然缺**：壳侧的网络与发现两组；`transfer_throughput_bps`。

### 8.69 壳的发现指标：设备数而不是包数

★ **三个名字，与守护进程一致**：`scans_total{result=ran|unavailable}`、`scan_duration_ms`、`candidates_found`。
`unavailable` 在 `devices_scan_start` 拒绝（这是“有网络却绑不上 socket”）时记；`ran` 在一次真正的网络扫描结束时记；
cached 与零预算的“读已知”不计 —— 它们没有扫描，记进去会把“扫了”和“看了一眼”混成一个数字。

★ **`candidates_found` 读的是集合**：worker 在 `record` 之后把接受的设备 id 放进一个
`Arc<Mutex<HashSet<String>>>`（`Sightings`），命令在扫描前 `reset_sightings`、扫描后取 `len`。一台设备在一个扫描窗口
里会广播多次，计数器会把它报成两台；守护进程读的 `host.discovery().len()` 也是一个集合，两个宿主的数字必须回答
同一个问题。集合由 `note_sighting` 这个自由函数写入，worker 闭包持有的是 Arc 而不是整个 `Discovery` —— 后者在
worker 启动时还不存在。

★ **一个不在设计清单里的选择**：拒绝也是一次扫描尝试，所以 `unavailable` 与 `ran` 同一个 counter、同一个标签名。
设计写的是“按结果”，守护进程用的就是这两个结果词；壳只是补齐了另一半。

★ 测试：真实扫描（60ms 预算）断言 `scans_total{result=ran}`、`candidates_found` 的 `count == 1`、`scan_duration_ms`
存在；无 worker 的拒绝断言 `unavailable`；集合去重一条（同一 id 两次 == 一台）。壳 250 条测试、
两种配置 clippy `-D warnings`、`cargo check`（含 Tauri）。
**仍然缺**：壳侧网络组（`links_*`/`frames_*`/`bytes_*`）；`transfer_throughput_bps`。

### 8.70 桌面宿主的 Host 终于记账了

★ `Host::with_metrics` 从第 76 轮起就在 core 里，守护进程用了它，而**窗口没有**：`AppState::new` 里那行是
`Host::new(identity, peer)`，listener 里那行也是。于是 `frames_sent_total`、`frames_received_total`、
`links_opened_total` 在桌面宿主里从来没被写过 —— 指标清单上"网络三项已接"说的是 daemon，不是这个进程。

★ **修法是两行，但顺序有讲究**：`AppState::new` 原来在结构体字面量里内联建记录器，而 Host 也在字面量里；要让两者
指向同一个 `Arc`，记录器必须先于 `Host` 建立。现在先 `let metrics = Arc::new(RingRecorder::new())`，再
`Host::new(..).with_metrics(Arc::clone(&metrics))`，最后 `metrics` 作为字段。listener 的 `bind` 用同一份
`state.metrics`。

★ **测试接在公开方法上**：`Host::count_frame_sent` 是公开的，所以测试可以调它再断言 `state.metrics.snapshot()` 里
出现 `frames_sent_total{event=kernel.test}`。不这样做的话，"Host 有 metrics 字段"与"壳的 Host 有 metrics"看起来
一样，而只有后者是这次要修的东西。

★ **仍然缺（写清楚，别让计数看起来完整）**：壳的拨号链路用 `dial_peer` + `PeerLink`，绕开 `Host::send`，所以拨出去
的帧仍不计数；`bytes_*`、`links_closed_total`、`reconnect_attempts_total` 两个宿主都没有；`transfer_throughput_bps`
仍是清单上的名字。

★ 测试：壳 251 条通过（新增 1 条），两种配置 clippy `-D warnings`、`cargo check`（含 Tauri）通过。
**仍然缺**：拨号侧帧计数、bytes/关闭/重连三类、吞吐直方图、审计保留期配置键。

### 8.71 保留期从常量变成设置

★ 审计的 7 天保留期一直是一个常量（`AUDIT_RETENTION_MS`），而 [17 §411](./17-storage-and-config.md) 的配置结构里写着
`audit_retention_days: u32 // 默认 7`。现在这个键真的被读了：`audit.retentionDays`（配置键用点分路径 + camelCase，
与 `transfers.confirmCancel`、`plugins.registry` 一致），缺省 7 天，**夹在 1–3650 之间**。

★ **为什么夹**：配置文件可以被手改，而 `0` 会把今天刚看到的拒绝一起删掉 —— 那正是用户来查的东西。夹取发生在读取处而
不是屏幕上，因为文件不经过屏幕。

★ **读不出来的值用缺省，不拒绝启动**：`prune_audit` 跑在 `AppState::new` 里，一个坏值不该让窗口打不开。屏幕读的是
`config_get` 的往返结果，所以它显示的就是存着的值；读不出来的值不可能是屏幕写出来的。

★ **界面**：设置页最后一个 “Audit history” 一节，数字框（1–3650）+ Save，写回同一个配置键；保存前在页面里也夹一次，
这样框里的数与发出去的是同一个。放在最后而不是紧挨 Transfers：它改的是内核下次启动时执行的清理，不是用户会盯着看的
控件。

★ **一次被既有测试抓住的布局错误**：最初把这一节插在 Transfers 之后，于是设置页里出现**第二个 Save 按钮**，而 registry
的两条测试用 `findAll('button').find(text 含 'Save')` 取第一个 —— 它们开始点审计的 Save，断言失败在
`config_set` 收到 `audit.retentionDays: 7`。把这一节移到页面末尾之后恢复；审计自己的测试则按“输入框旁边的按钮”定位，
不依赖按钮顺序。

★ 测试：壳 253 条（新增 2 条：配置 1 天会删掉默认 7 天保留的那行、0 被夹成 1 且今天的不删）；前端 249 条（新增 1 条：
显示 30、改为 90 并写回）；两种配置 clippy、build、shapes 8 项、guards 12 项。
**仍然缺**：`log_retention_days` 仍是同样的常量状态；审计的 DB 清空只按插件，没有“清空全部”。

### 8.72 插件调用的计数与延迟

★ 指标清单的插件组一直只有名字。现在 `PluginSet::call` 这个唯一漏斗记三项：
`plugin_calls_total{interface,plugin}`、`plugin_call_duration_us{interface,plugin}`、`plugin_faulted_total{outcome,plugin}`。
`interface` 来自 `Work::interface`（`share`/`actions`/`lifecycle`），与协议同一套词；`outcome` 来自
`CallOutcome::as_str`（`trap`/`timeout`/`fuel-exhausted`/…）。

★ **为什么在 `call` 里而不是 `run_on` 里**：`run_on` 拿到的是一次调用（实例 + 工作 + 预算），不知道是哪个插件、哪个
接口；`call` 三个都知道，而且匹配、动作、生命周期三个入口都会经过它。放错一层，指标只会覆盖其中一个入口，而那一层
看起来同样"在记录"。

★ **延迟用微秒，而且不用 `as_micros() as f64`**：设计名是 `plugin_call_duration_us`，目标 <20μs；`as_secs_f64() *
1_000_000.0` 避免一次整数截断的 cast（本会话早些时候在 transfer 的字节数上已经为同一类 cast 加过 allow）。

★ **fault 的 outcome 是标签而不是第二张计数器**：`trap` 与 `timeout` 对用户是不同的处境，而 `is_misbehaviour` 已经回答
了"是否 fault"；再写一张按 kind 分表的计数器会是同一个判断的第二种写法。

★ 测试：core 670 条（新增 2：一次调用留下计数与直方图；一次 trap 留下 `outcome=trap` 的 fault），clippy `-D warnings`、
guards 12 项。
**仍然缺**：`plugin_fuel_consumed`、`plugin_memory_bytes`（引擎没把读数交给调用方）、`plugin_pool_hit_total` /
`plugin_pool_miss_total`（`pool.acquire` 不回答命中与否）、`plugin_install_total`（安装路径没有按阶段上报）。

### 8.73 链路自己数字节

★ `bytes_sent_total` / `bytes_received_total` 落在 `PeerLink`，因为它是唯一拿着 payload 的地方：`Host` 只数帧，
加不出一个它从没取过的数；信封在 `Host::send` 里已经计数，块根本不经过那个函数。

★ **payload 字节，不是线上字节**：帧头（长度、kind、chunk_id）是传输层自己的账。读者问的是“这台设备搬了多少
内容”，把 header 加进来会让同一个文件在两端报出两个略有差别的数字。计数在 `write_frame` 成功之后、`parse_envelope`
/ chunk 解码成功之后 —— 被拒绝的写一个字节都没动。

★ **`with_metrics` 是 builder**：`PeerLink::new` 的调用点有测试、daemon 与壳几十处；给构造器加参数会让每一处都
改一次，而其中大多数不记账。`Host` 在 `dial` 与 `accept` 造出链接后把**自己的**记录器传进去（`Option` 为 None 时
链接仍然是一条链接），壳的四处拨号（传输、配对、同步计划、同步执行）也各自 `with_metrics`。

★ 测试：core 671 条（新增 1：一条链接把 4 字节的块计进发送与接收两条序列，信封无 payload 时贡献 0），
clippy `-D warnings`、guards 12 项。
**仍然缺**：`links_closed_total` / `reconnect_attempts_total`；拨号侧的**帧**计数（Host 的 `send` 不在那条路径上）；
`transfer_throughput_bps`。

### 8.74 传输的吞吐：两个已有的数相除

★ `transfer_throughput_bps{kind}` 不需要新的测量，只需要把 `transfer_bytes` 与 `transfer_duration_ms` 已有的两个数
相除 —— 它们都在 `record_attempt` 里，所以吞吐也在那里，前提是**字节确实移动了**（`record_bytes`）且**有时长可除**
（`elapsed` 是 `Some` 且非零）。

★ **窗口是整次尝试，发送含拨号**：所以这是端到端速率，不是链路线速。这个选择要写出来，因为两个数都能叫“传输速度”，
而读者拿它和探测到的带宽比较时，差别来自拨号与握手。取消/abort **不记**速率：它们没有搬完自己的字节，一个“失败的
传输有多快”的直方图只会让失败的慢速重传看起来像网络问题。

★ 测试：成功发送的用例断言 `transfer_throughput_bps{kind=attachment}` 的 `count == 1` 且 `sum > 0`（速率的具体值取决
于内存链路花了多久，硬编码它会是测试时钟的测试）；abort 的用例断言这条序列**不存在**。壳 253 条、两种配置 clippy
`-D warnings`、`cargo check`（含 Tauri）、shapes 8 项、guards 12 项。
**仍然缺**：`transfer_retries_total`（没有重传机制）、`links_closed_total` / `reconnect_attempts_total`、拨号侧的帧计数、
`plugin_fuel_consumed` / `plugin_memory_bytes` / pool 命中、存储与进程两组。

### 8.75 诊断导出有了界面，它的答案也进了契约

★ `diag_export` 注册了很久，实现也写过测试，但**界面没有调用方** —— 一个用户在窗口里找不到
“把诊断发给我”。现在设置页最后的 “Diagnostics” 一节有两个自愿勾选（审计轨迹、debug 日志；后者才显示行数）与一个
Export；结果回显**路径 + 包含的章节 + 略过的章节**，因为只给一个路径，用户没法在把文件发出去之前确认审计没有混进去。

★ **`DiagExportResult` 从壳搬进了 `synra-ipc` 的 DTO**。它原来是 `commands::diag` 里的私有结构，于是页面要读它
就只能手抄一份 —— 而手抄的形状不会在内核加字段时编译失败。搬进 DTO 还有一个副作用：`dto-coverage` 的正向检查从此会
要求 `ui.ts` 有同名 interface，所以那份镜像不会再悄悄落后。顺带修正了 08 §2.2 里两处过时的签名
（`opts`/`Utf8PathBuf` → `options`/`DiagExportResult`）。

★ **两处刻意的选择**：两个可选章节默认关（它们分别会说出用户装了哪些插件、可能带上用户没打算分享的内容）；
`logLines` 只在 debug 勾选时发送，而不是永远带一个默认值 —— 一个没被要求的行数不该出现在请求里。

★ **dev 构建也有答案**：`FakeKernel` 补了 `diag_export` 的返回（路径 + 四个必含章节）与 `REQUIRED_ARGS` 的 `options`，
否则在没有 Tauri 的开发构建里点 Export 会拿到 `undefined`，而 `commands.test` 的 arity 表也补了它。

★ 测试：前端 251 条（新增 1：勾选审计、点 Export、断言请求只带 `includeAudit`、回显路径与两个章节列表），
壳 253 条、js 34 条 + `pnpm run check`、两种配置 clippy、build、shapes、guards 12 项。
**仍然缺**：探针/doctor 的界面展示；`diag_logs_tail` 的界面（日志只在导出里出现）。

### 8.76 诊断面板的日志，以及一个没人检查的数字

★ **最近日志**：设置页 Diagnostics 一节的折叠区，展开时调 `diag_logs_tail { lines: 200 }`。选择“展开才读”与审计同一
理由：日志会增长，一次复制在用户看的时候就已经旧了；而“日志为空”与“还没读过”是两句不同的话，空态只在**问过之后**
才画。关闭不重读 —— 已经读到的行仍然是内核写过的行。

★ **README 的“被调用 N 条”一直是散文**。`readme-counts` 只比注册数，注释里写着原因：调用点总数（`checked`）不是不同
命令数，比较它们曾经报过 15 对 10 并会误杀一句真话。但 `checkCommandNames` 本来就算出了**不同命令的集合** —— 这一轮
加导出与日志两个调用方时，散文从 29 悄悄落后到 31，而没有任何东西会红。

★ 现在 `checkCommandNames` 返回那个集合，`readme-counts` 同时比注册数与调用数。**负控制实测过**：把 README 改成 30，
`pnpm run shapes` 报 `README.md says 30 called commands, and src calls 31 distinct ones`；改回 31 即绿。另外把
“读被调用的名字”挪到读内核检出**之前**，这样兄弟仓库缺失时调用数仍然被检查（缺失只让“名字是否合法”失去对照）。

★ 测试：前端 253 条（新增 1：展开日志才读、断言 `{ lines: 200 }` 与两行内容、未展开时不调用），shapes 8 项、
lint 0 error、build、guards 12 项。
**仍然缺**：探针/`doctor` 的界面展示。

### 8.77 窗口也能跑探针了

★ `run_probes` 的调用方一直只有 CLI。窗口要问同一组问题，就需要一个命令、一份契约形状和一个界面 —— 三件都在这一轮
补上：`diag_probes`（`Diag`/`Host`/`One`/非 mutating）、`ProbeReportDto`/`ProbeResultDto`、设置页 Diagnostics 的
“Run checks”。

★ **`status` 在契约里是字符串**（`ok`/`warn`/`fail`），不是 `synra-observe` 的枚举：DTO 复用那个枚举会为一格字段把
观察 crate 拖进 `@synra/protocol` 的镜像，而镜像要逐字段对照 Rust 的，边界上多一个 crate 就多一处会漂的东西。

★ **端口用 0**，与 CLI 同一个理由：探针问的是“这台机器有没有空闲端口”。在已经在跑的窗口里探已经绑定的端口，会得到
“被占用”，而占用者正是我们自己 —— 那会把正常运行报成故障。

★ **宿主事实留空**：命令层没有拿到窗口的 WebView 版本与钥匙串的句柄，所以 `HostFacts::default()`；那两个探针回答
`unknown` 并写明原因。编一个事实比缺一个更糟：这份报告是维护者当证据读的。

★ **按需运行，不在挂载时跑**：探针会绑端口、写文件、读时钟；每次打开设置页都跑一遍是用户没要求的工作。
`FakeKernel` 同样补了答案（一条 ok + 一条 warn），否则开发构建里点下去只会拿到 `undefined`。

★ **两处被测试抓住的计数**：契约从 46 变 47，`commands.test`/`registration.test` 里两个硬编码的 46 同时红了 ——
它们本来就是钉住“解析器没有漏条目”的；README 的 47/45/32 也同步，且现在由 `readme-counts` 检查（上一轮加的）。

★ 测试：壳 254 条（新增 1：窗口问的探针名与 CLI 相同 —— `tcp_port_available`/`data_dir_writable`/`webview_version`，
状态都在三个词里），前端 255 条（新增 1：点 Run checks 前不调用、点后出现名字/状态/建议），core 123 条 IPC 测试，
js 34 条 + `pnpm run check`，两种配置 clippy、build、shapes、guards 12 项。
**仍然缺**：`DoctorReport` 的汇总与“只看失败”没有界面；探针的 WebView/钥匙串事实仍然由 CLI/窗口共同留空。

### 8.78 探针报告的摘要与过滤

★ 八个到十二个结果读起来是一屏。报告上方一行**从结果算出来的**计数（`N OK · N warnings · N failed`），下方一个
“Only problems” 勾选把通过的行藏起来 —— 这两个都是页面的事，不需要新命令，也**不另存一份计数**：`probeReport` 是唯一
来源，摘要与列表都是它的视图（存一份计数是第二次让它们可能不一致）。

★ **单数/复数**在这一行里是代码：`warning` / `warnings`。界面文案的复数规则不交给调用方，否则每个调用点都会自己拼。

★ 测试：前端 255 条（扩展现有的探针用例：摘要 `1 OK · 1 warning · 0 failed`，勾选后 `tcp_port_available` 消失、
`wasm_backend` 仍在），build、lint 0 error、shapes 8 项、guards 12 项。
**仍然缺**：`DoctorReport` 自己的排版与退出码语义（窗口看原始结果，不看 doctor 的过滤参数）。

### 8.79 存储组的两项：采样而不是计数

★ 设计里存储组有三个名字，其中两个是**大小**：`db_size_bytes` 与 `plugin_data_bytes{plugin}`。大小不适合在写入点累加 ——
数据库文件的大小取决于页、WAL、VACUUM，插件沙箱里的字节可能被插件自己删掉。所以两者都在**读的时候采样**：

- `db_size_bytes`（Gauge）：`diag_metrics` 每次读快照前从 `paths.db_path()` 取文件元数据；读不到就是 0（内存数据库、
  尚未建库），不是错误。
- `plugin_data_bytes{plugin}`（GaugeVec）：`plugins_list` 读列表时逐个插件遍历 `plugin_sandbox_dir`；不存在的目录是 0，
  不可读的目录也是 0 —— 大小是诊断，不该让列表失败。

★ **遍历写的是递归而不是只读顶层**：测试特意在沙箱里放了一个嵌套文件（10 + 5 字节），只读顶层会漏掉 5 字节而测试仍然
会绿。

★ **一次顺手的清理**：`diag.rs` 的 `diag_metrics` 文档注释里有一处提交进来的模板残渣 `""" + S + """`，改成正常句子。

★ **一条旧测试因此变了意思**：`an_empty_shell_reports_no_metrics_rather_than_refusing` 断言空快照，而 `db_size_bytes`
让空壳也有一个样本。测试没有删，改成断言“只有这一个样本”，并把规则（回答而不是拒绝、不编答案）留在注释里 ——
断言应该描述新的真相，而不是被删掉。

★ 测试：壳 256 条（新增 2：沙箱 15 字节的嵌套文件 → `plugin_data_bytes{plugin=@synra-plugin/chat}=15`；32 字节的
“数据库文件” → `db_size_bytes=32`），两种配置 clippy、check、前端 255 条、shapes 8 项、guards 12 项。
**仍然缺**：`db_write_duration_us`（写路径没有计时钩子）；进程组四项（`memory_rss_bytes`/`cpu_percent`/`threads`/
`uptime_seconds`）需要平台读数。

### 8.80 进程组的第一项：uptime 不需要平台调用

★ 进程组四项里三项要平台读数（RSS、CPU、线程数），`uptime_seconds` 不用。`AppState` 多了一个单调
`std::time::Instant`，`diag_metrics` 每次读快照前把“距上次采样的秒数”加进计数器；第一次采样加全部已运行时间。

★ **delta 而不是 set**：设计里它是 `Counter`。每次轮询把整个 uptime 写进去会让它随**刷新次数**增长，而不是随时间；
`record_uptime(10)` 之后 `record_uptime(14)` 的结果是 14，不是 24 —— 测试就是照这个差值写的。

★ **为什么不用 `started_at_ms`**：那是调用方的墙钟，测试里固定（`FIXED_NOW_MS`），用户也可能在两次读之间改系统时间。
uptime 问的是“这个壳开了多久”，时钟回拨不该让它倒退。

★ **锁的顺序**：先取 `uptime_reported` 的锁算出差值，**放锁之后**再碰记录器 —— 记录器内部有自己的锁，两把锁在同一
函数里按一个顺序拿、在别处按另一个顺序拿，是死锁的标准配方。clippy 的 `significant_drop_tightening` 也正好指出了
这一点。

★ 测试：壳 257 条（新增 1：两次采样 10/14 → 计数器 14；`diag_metrics` 的空壳用例改成断言两个样本
`db_size_bytes` 与 `uptime_seconds`），两种配置 clippy、check、guards 12 项。
**仍然缺**：`memory_rss_bytes`、`cpu_percent`、`threads`（需要平台读数）、`db_write_duration_us`。

### 8.81 进程组的三项：把平台读数借进来

★ `memory_rss_bytes`、`cpu_percent`、`threads` 需要一个只有平台知道的事实。用 `sysinfo` 0.38（本来就在锁文件里，是
Tauri 自己的依赖），只开 `system` feature，所以它的磁盘/网络/用户读数不进这个二进制。

★ **`System` 存在 `AppState`**：CPU 是两次刷新之间的差值；每次调用新建一个 `System` 会永远报 0，而一个永远为 0 的
CPU gauge 比没有更糟。构造时先刷新一次作为基线，之后每次 `diag_metrics` 刷新。

★ **`threads` 只在平台真的报告时写**：`Process::tasks()` 在有的平台上返回 `None`。用 0 表示“不知道”会被读成“没有
线程”，所以干脆不写这条序列；平台不肯描述进程时三项都不写。

★ **两把锁的顺序**：读 `System` 的锁在拷贝完三个数之后 `drop`，再去写记录器（记录器有自己的锁）。clippy 的
`significant_drop_tightening` 又一次替我们指出了这一点 —— 上一轮的 `record_uptime` 是同一个形状。

★ **一条旧断言**：`an_empty_shell_reports_no_metrics_rather_than_refusing` 现在看到四个样本（`db_size_bytes`、
`uptime_seconds`、`memory_rss_bytes`、`cpu_percent`），于是从“长度等于 2”改成“包含这四个、且没有别的”，并允许
`threads` 缺席 —— 断言描述的是新的真相，不是新代码的借口。

★ 测试：壳 258 条（新增 1：`memory_rss_bytes` 是大于 0 的 gauge —— 具体值没有测试能知道），两种配置 clippy、check、
guards 12 项。
**仍然缺**：`db_write_duration_us`（写路径没有计时钩子）。

### 8.82 安装停在哪：一个计数器的两种标签

★ `plugin_install_total{stage,result}` 在 `run_install` 的出口记**一次**：成功是 `stage=complete,result=ok`，失败用
拒绝自己命名的阶段（`source`/`manifest`/`read`/`verify`/`write`/`record`/`update`）加 `result=refused`。阶段名不是这里
另起的词表 —— 它来自 `InstallRefusal::stage`，也就是同一个 `IpcError` 的 `params.stage`，所以日志、错误与指标说的是同一个
阶段。

★ **一次尝试一个样本，不是每步一个**：计数器回答的是“安装停在哪”。如果把中途成功的步骤也记上，每次失败的尝试都会看
起来像一串成功，而“九步成功、一步失败”与“九次成功安装”在面板上不可区分。

★ 测试：壳 259 条。成功路径扩展已有的 `installing_a_directory_records_it_switched_off`（报告说成功，记录器也要说）；
新增拒绝路径：装一个不存在的目录 → `params.stage == "source"`，记录器出现
`plugin_install_total{result=refused,stage=source}`。两种配置 clippy、check、guards 12 项。
**仍然缺**：`plugin_fuel_consumed`、`plugin_memory_bytes`（引擎没把读数交出来）、pool 命中/未命中。

### 8.83 卸载的“询问”：从一行确认到一个能问的对话框

★ 卸载的四步在 2026-10-18 就落地了，但页面把 `remove_data` 写死成 `false`：命令一直接受两个取值，**界面只能发一个**。
这不是少一个按钮，而是“删除我的数据”这个选项在窗口里不存在。

★ **换成 `AModal` + 勾选**：不勾是保留（默认、可逆），勾了才把 `remove_data: true` 传下去；每次打开都重置为不勾 ——
记住上次的破坏性选择等于第二次访问默认删数据。用 `get-container="false"` 让对话框留在组件内而不是 teleport 到 `body`，
页面测试才看得见那个勾选框；一个测不到的对话框就是没人验证过的决定。

★ **两次被测试抓住的细节**：
① 加 `note.value = ...` 之后 eslint 报 `no-use-before-define` —— `note` 声明在 `remove()` 后面，于是把声明挪到前面，
   而不是加一行 eslint-disable。
② 页面测试第一次失败是因为按钮按文字找：对话框自己的 OK 按钮（关闭时也在 DOM 里）排在卡片之前，测试点的是它。
   改成按 `ant-btn-dangerous` 找卡片上的按钮，并额外断言对话框的 `open` 才是真正的证据。

★ 测试：前端 257 条（新增 2：页面“先问再删”整条路径；动作层 `removeData: true` 也发得出去），build、lint 0 error、
shapes 8 项、guards 12 项。
**仍然缺**：`RemoveReport`/`bytes_freed`；对话框里显示数据目录路径。

### 8.84 卸载报告：一个声明为 `One` 的命令返回过 `()`

★ `plugins_uninstall` 在契约表里一直是 `One`，但壳的包装器返回 `()`、界面按 `null` 读。守卫检查名字与参数，不检查
返回值，所以这个差集活了很久：一个能看见的字段（`RemoveReport`）从没被写过，而“删了多少”只有一句没有数字的话。

★ **现在返回 `RemoveReport{bytes_freed, data_kept, leftovers}`**。字节数在删除**之前**量：删完之后再量只会得到 0，那是
把“没有东西”伪装成“测量”。两个目录都算 —— artifact 总是删，sandbox 只在 `remove_data` 为真时删；报告里的
`bytes_freed` 是总数，因为用户关心的是“腾出了多少”。`leftovers` 恒空：`PluginFiles::remove` 要么整棵删掉要么拒绝，
没有“删了一半”的状态。

★ **格式化也合并了**：`formatBytes` 从 `HomePage.vue` 抽到 `src/format.ts`，首页的传输条与插件页的卸载报告共用一个
实现 —— 两份拷贝第一次改就会对同一个数给出两种写法。

★ **假内核也补了**：`plugins_uninstall` 原来在 `FakeKernel` 里没有处理分支，开发构建点 Remove 会拿到 `undefined`，
而页面现在要读 `bytesFreed`。补上答案，并把它加进 `REQUIRED_ARGS`（`pluginId`、`removeData`）—— `command-arguments`
守卫**当场抓到了**这个遗漏，消息是“fake answers plugins_uninstall, which the shell refuses without …”。

★ 测试：壳 259 条（两条卸载测试现在断言 `bytes_freed`：保留数据 = manifest 长度；删除 = manifest + 5 字节）、
前端 258 条（页面报告句带 `1.5 KiB`；动作层断言 `RemoveReport` 原样传回）、build、lint 0 error、shapes 8 项、
guards 12 项。
**仍然缺**：对话框里显示数据目录路径（需要 `PluginSnapshot.dataPath`）。

### 8.85 删除之前，先看见路径

★ 卸载对话框有两个结果，其中一个删掉一个目录。上一轮它问“要不要删数据”，但**没有说删的是哪个目录** —— 一个关于
删除的问题却让用户看不见删除对象。

★ `PluginSnapshot` 增加 `data_path`（`String`），壳用 `Paths::plugin_sandbox_dir` 构造它 —— 与
`plugins_uninstall` 里 `PluginFiles::remove` 用的是同一个函数。两份“数据在哪”的推导就是两个答案，而其中一个会在
某个 `sanitise_component` 改动后开始说谎。

★ **契约改动照例三处**：`synra-ipc` 的 DTO（带 `#[serde(default)]`，旧的序列化数据仍读得回来）、`@synra/protocol`
的镜像（`dto-coverage` 逐字段对照，34 个 DTO 全对）、壳的映射与两个夹具（`fake-kernel-fixtures` 的两张卡、页面测试的
`card()`）。页面的对话框把路径用等宽字体显示在勾选框上方。

★ 测试：壳 259 条（`a_card_says_what_the_plugin_asks_for` 增加断言：卡片的 `data_path` 等于
`plugin_sandbox_dir(ID)`）、core 123 条、前端 258 条（对话框里出现 `Data folder: <path>`）、js shapes 14 项、
两种配置 clippy、build、lint 0 error、guards 12 项。

### 8.86 插件占了多少内存：读实例，不估引擎

★ `PluginInstance::memory_bytes()` 一直存在（scripted 与 wasm 两个后端都实现了），而指标里没有它。现在
`PluginSet::call` 在 `run_on` 返回之后、池 `release` 句柄之前读一次，记 `plugin_memory_bytes{plugin}`。

★ **为什么必须是那一刻**：句柄归还后实例可能被复用、回收或下一次调用才重建；调用结束后、释放之前是集合唯一还能看见
“这个插件的实例现在占多少”的时刻。

★ **gauge 而不是 hist/counter**：内存会被还回去。一个只增的计数器会把“峰值一次”记成“一直这么多”。

★ **顺手的一处 clippy**：两层 `if let` 被 `collapsible_if` 拒绝，改成 let-chain（edition 2024 支持，仓库里已有先例）。

★ 测试：core 671 条（扩展现有的调用计数用例：scripted 实例每次调用加 4096 字节，断言
`plugin_memory_bytes{plugin=@synra/aaa-notes}` 恰为 4096 —— 一个恒为 0 的实现也能让“序列存在”通过，而这里断言的是
实例自己的读数到达了记录器），clippy `-D warnings`、guards 12 项。
**仍然缺**：`plugin_fuel_consumed`（引擎没把读数交出来）、pool 命中/未命中。

### 8.87 插件花了多少指令：在调用前后各读一次油箱

★ `PluginInstance` 新增默认方法 `last_fuel_consumed() -> Option<u64>`（scripted 后端不覆盖，返回 `None`）。wasm 后端在
`dispatch` 之前读一次 `store.get_fuel()`（此刻油箱刚被设为预算或 `u64::MAX`），调用结束再读一次，差值就是这次调用花掉的
指令数。

★ **`None` 不是 0**：没有指令计数器的后端（scripted）保持“什么都没记”。`PluginSet::call` 只在 `Some` 时记
`plugin_fuel_consumed{plugin}` 直方图 —— 记一个 0 会把“不知道”说成“没花指令”，而这正是设计里 `plugin_fuel_consumed`
与 `plugin_calls_total` 要分开的原因。

★ **读的位置**：`if let (Some(granted), Ok(remaining))` 在 match 之后。一个**没启动**的调用（dispatch 之前就失败）不写
读数，`last_fuel` 保留上一次的值而不是编一个零。

★ 测试：plugin-host 316 条（返回的 guest：`last_fuel_consumed` 大于 0；被 fuel 停下的 guest：恰好等于预算 `100_000`，
在第二次调用覆盖读数之前断言），core 671 条（新增断言：scripted 后端**没有** `plugin_fuel_consumed` 序列 —— `None`
分支的测试），两种 clippy、guards 12 项。
**仍然缺**：plugin pool 命中/未命中（`pool.acquire` 不回答）。

### 8.88 池的命中：在调用之前看一眼 `live`

★ `InstancePool::acquire` 只说“给你一个 handle”，不说它是不是热的。但 `PluginSet` 自己有一张 `live: handle → instance`
—— 调用**之前**这个 handle 在不在表里，正好就是“池交给你的槽位是否已经有实例”。

★ 一次调用记一个：`plugin_pool_hit_total` 或 `plugin_pool_miss_total`。池拒绝（到达实例上限）的调用在 `acquire` 就返回
了，到不了这里，所以两者之和就是池真正服务过的调用数 —— 一个“只记 miss”的实现会让热池永远看起来是冷的。

★ 语义写清楚：这是**池的行为**，不是引擎的花费。一个 trap 之后实例被丢弃的调用仍然算 hit（池确实交出了热的那个），
下一次才会 miss。

★ 测试：core 672 条（新增一条：同一插件调两次 → miss 1、hit 1；scripted 的 stateless 策略让第二次拿到同一个 handle），
clippy `-D warnings`、guards 12 项。
**仍然缺**：daemon 侧的 `plugins_installed`/`plugins_active` 与 `plugin_install_total`（安装与列表都在窗口里）。

### 8.89 daemon 也说自己有几个插件

★ `plugins_installed`/`plugins_active` 一直是窗口的 `plugins_list` 设的。daemon 没有安装命令，于是它从不设这两个 ——
在同一个内核运行的 headless 机器上，面板只能读成“这个内核没有插件”。现在 `plugins_value`（daemon 的 `plugins` 报告）
在读表之后设这两个 gauge：列表本来就是它唯一能学到“装了几个、几个开着”的地方。

★ `plugin_install_total` 仍然只在壳里：安装是一个界面动作，daemon 没有这条路径。写进文档而不是硬塞一个永远为零的
序列。

★ 测试：daemon 213 条（新增：两行不同 `enabled` 的插件 → `plugins_installed=2`、`plugins_active=1`；一行两用会被
这个断言抓住），clippy `-D warnings`、guards 12 项。
**仍然缺**：`plugin_install_total` 的 daemon 侧（没有安装路径）。

### 8.90 帧计数搬到链接：一条拨号链路的 offer 不再隐形

★ `frames_sent_total{event}` 原先记在 `Host::send`，`frames_received_total{event}` 记在 listener 的 `Delivered` 分支。
两者都只覆盖**经过某个特定函数**的帧：壳的四处拨号（传输的 offer、配对请求、同步计划/执行）直接 `link.send`，
拨号链路收到的回答（配对响应、插件列表）也不经过 listener。于是最需要诊断的那条路径恰好没有数字。

★ 现在由 `PeerLink::send` 与 `PeerLink::receive_inbound` 记。链接是**每一帧**都经过的唯一对象；代价是 `Host::send` 与
listener 必须**不再**记，否则每条路由消息会变成 2。两个 `Host::count_frame_*` 方法保留（没有记录器的宿主、以及壳的接线
测试仍用它），文档写明“不要与链接同时用”。

★ **收件侧的语义因此变宽了一点**：链路记的是“解析并通过身份检查的帧”，listener 原先记的是“被投递的帧”。重复帧、
被上层拒绝的帧现在也算收到 —— 这正是“frames received”的字面意思，而“投递”有自己的计数（审计、消息）。

★ 测试：core 672 条（链接测试现在断言收发两个方向的事件计数；host 的帧测试改成钉住 Host 自己的方法，而不是假装它
还在生产路径上），daemon 213 条、壳 259 条全部仍绿 —— 说明“从 Host 移走”没有让任何真实路径少计。clippy、guards 12 项。
**仍然缺**：`links_closed_total`、`reconnect_attempts_total`。

### 8.91 插件卡的警告：健康档位一直有形状，没有数据

★ 契约里的 `PluginHealthDto`（档位、句子、准入/拒绝、生效配额）从写下的那天起**没有任何代码构造**：壳的 `snapshot()`
把 `health` 固定为 `None`。后果是具体的 —— 一个被配额降级、甚至已停用的插件，在界面上和一个从未运行过的插件完全一样。

★ 现在：`PluginSet::report_of` 返回 `QuotaReport`（引擎的稳定类型），壳的 `snapshot()` 映射成 DTO，卡片在档位不是
`good` 时显示一条警告：`overreaching`/`degraded` 用 warning，`faulted` 用 error，正文是配额自己的 `explain()` 句子 ——
不是界面另写一句。`admitted`/`refused` 是**门**的调用数（脚本化的 guest 不调 host 函数时是 0），这一点写进了测试注释。

★ **顺手更正契约注释**：`explanation` 写的是“message key”，而值与 `Health::explain()` 一样是英文句子 —— 说成 key 会让
界面以为它可以翻译一个它翻译不了的东西。镜像里的中文注释同步。

★ 测试：壳 260 条（新增：加载一个插件后卡片的 `health` 是 `good`、`calls_admitted=0`、`usage` 非空；无引擎时 `None`），
前端 259 条（新增：`degraded` 的卡片出现警告标签与配额句子），core 672 条（`report_of` 已加载有报告、未加载为 `None`），
两种配置 clippy、build、shapes 8 项、guards 12 项。
**仍然缺**：`usage` 里的配额是整块 JSON，界面只展示档位与句子，没有把几个字段画出来。

### 8.92 CLI 也能卸载：把“没人做过的决定”变成 `--data`

★ `synra plugin remove` 一直解析、出现在文档清单里，然后回答“removing needs a decision about the plugin's stored data
that nothing has made yet”。这个决定后来做过了（窗口的卸载对话框）：默认保留，只有明确要求才删。于是这句拒绝的理由已经
不成立，剩下的只是**没有 request**。

★ 这一轮把它接完：`Request::PluginRemove { plugin_id, remove_data }`、`Kernel::plugin_remove`（默认拒绝，理由与
`plugin_install` 同）、守护进程实现、CLI 映射。守护进程**先量字节再删**：`PluginFiles::size_of`（store 新方法，artifact，
加 `--data` 时的 sandbox），然后删行、删目录，答案 `{pluginId, bytesFreed, dataKept}`。字节数在删除之后量只能是 0，
那是把“没有东西”伪装成“测量”。

★ **`Runtime` 因此多了一个 `paths` 字段**：`PluginFiles` 需要 `Paths`（artifact 与 sandbox 是两个根），而 Runtime 原先只有
`data_dir`/`plugins_dir` 两个字符串路径。让命令自己拼 `data_dir.join("plugin-data")` 就是把布局知识复制到第二个地方。

★ **CLI 的 prose 现在有一句真话**：`removed X; N bytes freed; its data was kept/deleted`。把 `dataKept` 字段名丢给用户，
等于让他们读一个字段来判断自己的设置有没有被删。JSON 输出不受影响。

★ 测试：control 66 条（`every_request_round_trips` 增加 remove）、daemon 215 条（删行+artifact、数据默认保留、
未安装即 `PluginNotInstalled`）、CLI 88 条（`--data` 进入请求；prose 句子）、store 225 条（`size_of` 的 4/8/0 字节），
workspace clippy、guards 12 项。
**仍然缺**：`plugin remove` 不报告“哪些文件没删掉”（`PluginFiles::remove` 要么整棵删掉要么拒绝）；守护进程仍不知道窗口
是否正打开着这个插件。

### 8.93 `--follow` 真的跟随：读写入者写的那个文件

★ `synra logs --follow` 的状态一直是“拒绝，因为需要一个流式协议”。本轮没有加协议，而是换了实现：
`synra_observe::logging::follow` 读**写入者写的那个文件**，初始尾部之后按 offset 增量读，200ms 轮询。

★ **为什么这条路是诚实的**：控制 socket 是数据目录里的一个路径，日志文件是同一个目录里的另一个路径 —— CLI 与 daemon
必然在同一台机器上，所以“流式协议”只为本地字节增加第二套生命周期与分帧。用户要的是“有新行就显示”，读同一个文件
就是那个行为。`--verbose` 的提级仍然发生在内核（先发一个普通 `logs` 请求拿尾部），所以它不是纯粹绕开内核：内核决定
级别，文件提供新行。

★ **写明的代价**：内核回答与跟随者第一次读之间写入的行会漏掉（毫秒级窗口）；`--output json` 对跟随输出不生效，
因为跟随是一行一行的流而不是一个结果。

★ **跟随器的四个测试都是行为，不是形状**：追加一行 → 它出现；`DEBUG` 在非 verbose 下不出现；文件变短（轮转/截断）→
从零重读；写入到一半的行 → 等换行，不拆成两行。CLI 一条测试：没有内核时立刻报“no kernel is listening”，而不是在
文件上干等。

★ **`main` 的第一个分流**：跟随没有可返回的结束，所以它不能变成 `CommandOutcome`。`main` 在 dispatch 之前分流，
`follow` 负责其余一切；这是“库里可测、main 三行”规则的一处显式例外，写在同一行注释里。

★ 测试：synra-observe 277 条（+4 跟随）、CLI 89 条（旧“必须拒绝”的测试被替换）、workspace clippy、guards 12 项。
**仍然缺**：`--follow` 不处理日志轮转后的 `.1` 历史（只跟当前文件）；`--trace` 依然没有东西可过滤，因为日志行里
还没有 trace id。

### 8.94 链路为什么结束：三个出口与一组原因

★ `links_closed_total` 一直没有生产者，因为“关闭”没有一个统一的地方。listener 的读循环有三个出口，而原因的词汇就在
`LinkError` 里：`Decision::Idle` → `idle`；对端 `Bye` → `closed`；握手未完成 → `handshake`；其余错误按
`timeout`/`transport`/`protocol` 分类，传输层错误 → `transfer`。

★ **只在 listener 记**：`Host::count_link_closed` 的注释写明了范围 —— 它数的是 listener 服务过的连接。拨号链路是短暂的
一问一答，没有“关闭”这个时刻；因此 `links_opened_total`（在 `adopt` 记、两个方向都算）与它不是一对必须相等的数。
如果要让两者相等，需要给拨号链路的结束也定义语义，那是另一件事。

★ **测试抓到的两处语义**：
① 一个未握手的连接在 300ms 后结束，counter 是 `handshake` 而不是 `timeout` —— 握手失败发生在进入读循环之前。
② 一个握手后挂断的连接是 `closed`（对端 FIN），不是 `idle` —— `idle` 要等读循环自己的 patience。测试因此按实际观察到
的原因断言，而不是按我一开始以为的。

★ 测试：core 672 条、daemon 215 条（transport 8 条：`closed` 与 `handshake` 各有断言，外加一条握手后挂断 → `closed`）、
workspace clippy、guards 12 项。
**仍然缺**：`reconnect_attempts_total`（没有重连机制）；拨号链路结束的语义。

### 8.95 trace id 有了来源：快照带上它，路由记下它

★ `--trace` 一直有过滤、没有生产者：日志行里没有 trace id，用户也没有地方拿到一个 id。这一轮补了两半的**第一块**：

- **来源**：`IntentSnapshot` 增加 `trace_id`。板上的 `IntentView.trace` 从第一版就有，快照把它丢了 —— 用户唯一能粘进
  `--trace` 的标识，界面反而看不见。首页 “Received” 的每一行显示 `Trace <id>`（等宽、可复制）。
- **生产者**：daemon 的 `intent_dispatch` 在路由成功后 `tracing::info!(trace_id = ..., intent = ..., "share routed")`。
  这行会写进 `synra.log`，`--trace <id>` 过滤到它。

★ **写清这不是“trace 完成”**：决定、执行、传输的日志仍然没有 trace 字段（它们大多用 `eprintln!`，根本不进日志文件）。
所以 `--trace` 目前只会返回路由那一行。把整条动作链的每一步都挂上 trace 是“把 daemon 的自述搬进 tracing”的更大一件事。

★ 测试：core ipc 123 条（快照序列化包含 trace）、壳 260 条（列表与单读同一映射，且 `trace_id` 等于板上
`raise` 返回的 trace）、前端 259 条（首页行出现 `INTENT.traceId`）、daemon 216 条、js shapes 14 项（34 个 DTO 逐字段）、
workspace clippy、guards 12 项。
**仍然缺**：动作链其余步骤的 trace 字段；daemon 的 `eprintln!` 自述仍未进日志文件。

### 8.96 决定的日志也带 trace：不另造 id

★ 上一轮让路由那一行带 trace。这一轮把**决定**接上：`Decided` 增加 `trace`，在 `confirm` 的同一把锁里从板上读 ——
不是新生成一个 id，也不是从调用方传进来；一个动作的 trace 只有板上那一个。

★ `to_json` 输出 `"trace"`，daemon 在 `intent_decide` 结束时记
`tracing::info!(trace_id = %decided.trace, intent, ok, …)`。`ok` 是执行结果，所以这一行同时是“决定了”和“跑完了”；
插件的执行细节在调用内部，没有单独的 start/finish —— 写在这里，免得把一行读成完整的执行轨迹。

★ 测试：daemon 216 条（`a_decided_share_runs_the_plugin_and_records_what_it_returned` 增加断言：决定的 `trace` 等于
路由的 `trace`），core 672 条、workspace clippy、guards 12 项。
**仍然缺**：远端完成（`apply_result`）没有 trace；daemon 的 `eprintln!` 自述仍未进日志文件。

### 8.97 远程动作闭环：收信、回信、应用结果

★ 设计里“发信、收信后本地跑插件、回 `INTENT_RESULT`”三件都写在 07，代码里却只有**发信**在生产路径上：
`accept_remote` / `accept_cancel` 只有测试在调，listener 收到 `INTENT_INVOKE` 只记一个事件名就落到 “nothing is wired”；
发送侧发完立刻 `bye`，所以回信即使产生也没有连接可回，本侧也从不读。一个远程 decide 的结果是：对面没跑，本侧永远
`dispatching`。

本轮补齐三处：

- **收信**：listener 新增 `KernelEventCallback` / `KernelEventOutcome`（`Handled` / `Reply` / `Refused` / `Ignored`，
  `#[non_exhaustive]`），由 daemon 在 `serve` 里用 `accept_remote` / `accept_cancel` 实现。回信在**发起请求的那条连接**
  上发出 —— reply 属于提问者，换一条连接回答等于回答一个没人听的问题。
- **应用结果**：`orchestrate::apply_result` 用 `reply_to` + `from` 找到 `Board::dispatched_to` 的意图并 `conclude`；
  发送侧 `send_outbound(outbound, wait)` 在 invoke 后等 3 s（`REMOTE_RESULT_WAIT`），拿到就 checkpoint，`intent_decide`
  用真实的 `ok/result/state` 回答。没等到就保持 `dispatching`，并写明**不会重试**、更慢的回信会丢。
- **一个真的 bug**：`remote::invoke` 造信封时新铸 request id，而板上 `Dispatching` 记的是 `confirm` 的 id ——
  `reply_to` 于是永远匹配不上，收信侧的幂等键也不同于本侧记录。现在信封携带 `confirm` 的 id；测试断言
  `outbound.envelope.request_id == decided.request_id`。这条 bug 只有在“真的等回信”时才会暴露，正是本轮做的原因。

★ 测试：core 673（+1 `dispatched_to` 的两个半键与结束后不再命中）、daemon 218（+2：`apply_result` 把 dispatching
变成 succeeded 且重复回答被忽略；一条真实 TCP 连接上 `INTENT_INVOKE` 得到同一链接上的 `INTENT_RESULT`，`reply_to`
等于请求 id）、ipc 123、workspace clippy、guards 12。
**仍然缺**：3 s 之外的迟到结果（无重试、无离线队列）；远端取消没有回执（取消本身不需要答案）。
（控制 socket 的排队已在 §8.98 消掉：现在每连接一线程。）

### 8.98 控制连接：接受与伺候分开，每个客户端一条线程

★ 上一轮让远程 decide 等 3 s 回信，于是“控制 socket 一次只服务一个客户端”从一句注释变成了用户能感到的排队：一台
`status` 会排在一条 `intent decide` 后面。根因不是等待本身，而是 accept 和 serve 是同一个调用
（`Listener::serve_one`）—— 循环里没有任何地方可以“先收下、再交给别人”。

- `Listener::accept` 公开，`serve_one` 改为 `accept` + `serve_stream` 的组合；
- daemon 的控制循环只做 accept，每条连接 `std::thread::spawn` 一个 `serve_stream`，`Runtime` 放进 `Arc`；
- accept 失败时**睡 10 ms**：以前失败的调用本身就是那个会阻塞的调用，现在拆开后一个永久失败的 listener 会烧满一核。

★ **为什么线程模型没有扩权**：socket 是本地的，每个客户端都是用户自己启动的进程 —— 与“一次一个”相同的信任边界，
只是没有队列。每个连接线程持有 `Arc<Runtime>`，命令内部本来就用锁（host/plugins/board/db），与 transport 线程并发
调用的是同一套。

★ 测试：control 67（+1：`accept` 只交出连接、`serve_stream` 在另一个线程上把 `Status` 答完 —— 断言的是跨交接触发的
回答，不是 `accept` 返回了一个 socket）、daemon 218、core 673、workspace clippy、guards 12。
**仍然缺**：每条连接一个线程没有上限（本地 socket 的信任边界内）；shutdown 不 join 连接线程，进程退出时它们随进程结束。

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
