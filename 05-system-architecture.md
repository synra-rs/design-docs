# 05 · 系统架构

> 本文定义 Rust 重构的**顶层形态**：哪些东西跑在内核、哪些跑在宿主壳、哪些跑在前端；进程/线程/沙箱边界画在哪里；以及为什么这样画。
> 后续所有设计文档都必须与本文一致。
>
> 📍 **仓库归属**（[README §1](./README.md)、[ADR-016](./20-adr.md)）：
> 第 1 环（内核）在 **`synra-core`**；第 2 环的 Tauri 外壳与第 3 环（前端）在 **`synra-app`**。
> 本文 §2.1「桌面端」与 §5「UI 插件」的具体落点在 app 仓库（见 synra-app 的 `design-docs/10-ipc-and-frontend.md`）。

---

## 1. 三环模型

```text
╔═══════════════════════════════════════════════════════════════════════════════╗
║                          第 3 环 · 表示层（Presentation）                      ║
║                                                                               ║
║   Tauri WebView（桌面 / 移动共用一套前端代码）                                  ║
║     ├─ 宿主 UI：Home / Plugins / Devices / Settings                           ║
║     └─ 插件 UI：每个 UI 插件一个隔离的渲染上下文（iframe 或独立 WebView）        ║
║                                                                               ║
║   与内核的唯一通道：Tauri IPC（command 调用 + event 订阅），类型由 Rust 生成     ║
╚═══════════════════════════════════╦═══════════════════════════════════════════╝
                                    ║  IPC（强类型，可审计）
╔═══════════════════════════════════▼═══════════════════════════════════════════╗
║                          第 2 环 · 宿主壳（Host Shell）                        ║
║                                                                               ║
║   Tauri 应用（synra-app 仓库：src-tauri + src）                          ║
║     ├─ 窗口 / 菜单 / 托盘 / 通知 / 深链 / 文件关联                              ║
║     ├─ Tauri command 与 event 的注册（薄适配层，不含业务）                       ║
║     ├─ 平台能力实现（Platform trait 的 Tauri 实现）                             ║
║     └─ 内核生命周期管理（启动 / 关闭 / 崩溃恢复 / 单实例）                        ║
╚═══════════════════════════════════╦═══════════════════════════════════════════╝
                                    ║  直接函数调用（同进程）或 IPC（跨进程）
╔═══════════════════════════════════▼═══════════════════════════════════════════╗
║                          第 1 环 · 内核（Kernel）                              ║
║                                                                               ║
║   synra-core  —  领域模型、聚合根、状态机、编排                                  ║
║   synra-net   —  发现（UDP/mDNS）、传输（TCP/QUIC）、帧、心跳、重连              ║
║   synra-protocol — 信封、事件、线格式、版本协商                                  ║
║   synra-plugin-host — Wasm 沙箱、能力闸门、生命周期、注册表                      ║
║   synra-plugin-ui   — UI 插件 artifact 管理与加载协调（不含渲染）                ║
║   synra-store —  SQLite / KV / 文件沙箱 / 密钥                                   ║
║   synra-observe — tracing / metrics / 诊断包                                     ║
║                                                                               ║
║   全部为纯 Rust，无 UI 依赖，可单独测试，可被 CLI / daemon 复用                  ║
╚═══════════════════════════════════════════════════════════════════════════════╝
```

### 1.1 三条铁律

| # | 铁律                                                                                                       |
| - | ---------------------------------------------------------------------------------------------------------- |
| 1 | **内核不知道 UI 存在**。内核 crate 不得依赖 `tauri`、`wry`、任何 GUI 库。                                  |
| 2 | **前端不知道网络存在**。前端不得直接持有 socket、不得解析协议；它只与内核 IPC 对话。                        |
| 3 | **插件不能直接碰内核**。插件永远通过沙箱边界（Wasm host function / UI 插件的宿主 API）与内核交互。            |

这三条分别消除了 [05](./11-design-principles.md) 的 R1、R4、R2。

---

## 2. 进程与线程模型

### 2.1 桌面端（Windows / macOS / Linux）

```
┌─────────────────────────────────────────────────────────────────────────┐
│ synra-desktop（app 仓库产物）                                                      │
│                                                                         │
│  ┌──────────────────────┐        ┌───────────────────────────────────┐  │
│  │ 主进程（main thread） │        │ tokio runtime（多线程）            │  │
│  │  Tauri 事件循环       │◄──────►│                                   │  │
│  │  窗口 / 菜单 / 托盘    │  通道  │  Kernel（Arc<Kernel>）             │  │
│  └──────────┬───────────┘        │   ├─ DiscoveryService (UDP task)  │  │
│             │                    │   ├─ TransportRegistry (per-link) │  │
│             │ IPC                │   ├─ IntentRouter                 │  │
│  ┌──────────▼───────────┐        │   ├─ PluginHost                   │  │
│  │ WebView 渲染进程(es)  │        │   │    └─ Wasm 实例池             │  │
│  │  宿主 UI              │        │   ├─ TransferSessions             │  │
│  │  插件 UI (iframe)     │        │   └─ Store (SQLite)               │  │
│  └──────────────────────┘        └───────────────────────────────────┘  │
└─────────────────────────────────────────────────────────────────────────┘
```

要点：
- **内核与 Tauri 同进程**：桌面端没必要跨进程，直接在同一进程内用 `Arc<Kernel>`；Tauri 的 command handler 就是薄包装。
- **Wasm 插件不额外起进程**：wasmtime 实例在同进程内，靠线性内存 + 无 syscall + fuel/epoch 实现隔离。省掉了进程间序列化开销。
- **可选：内核独立进程**。若需要"UI 关闭后继续服务"，把内核跑在一个 `synra-daemon` 子进程里，Tauri 侧通过本地 socket（unix domain / named pipe）与之通信。见 [07](./06-workspace-layout.md) §5.2 的 `kernel-remote` feature。
- **渲染进程**：Tauri 默认使用系统 WebView（Windows: WebView2 / macOS: WKWebView / Linux: WebKitGTK）。多个 WebView 实例可用于 UI 插件隔离（见 §5）。

> ### ★ 本节第一句是设计，而仓库里现在没有一处按它做
>
> 记在这里，因为这张图里的 `Kernel（Arc<Kernel>）` 对应的是 `synra_core::Kernel`（`crates/synra-core/src/kernel.rs`，
> 701 行）：它有 `Supervisor`、有事件总线（`subscribe`/`next_event`）、有审计环、有 `startup_verdict`。
> **而没有任何生产代码构造它**——构造点只有 `kernel_tests.rs`（`Kernel::in_memory`/`open`，`:13`/`:42`/`:45`/…）。
>
> ★ **注意它没有 listener。** 监听在 `Host` 上（`crates/synra-core/src/host/kernel.rs`，223 行）：
> `listener: Option<TcpListener>`（`:25`），`Host::listen_on`（`:160`）里 `TcpListener::bind`（`:161`）。
> 两个同名文件不是一件事，第一版笔记把它们混为一谈了。
>
> 更要紧的是**图里那两层服务没有注册者**。`Kernel::register_task(TaskSpec)` 认识 `"discovery.udp"` 与
> `"transport.accept"` 这两个名字，而这两个字符串**只出现在 `kernel_tests.rs`**（`:103`、`:107`、`:116`、`:117`）。
>
> 于是现在有三份各占一半的实现，**没有一份是图里那个**：
>
> | | store / identity / host | 发现 | 接受连接 | 插件 + 意图 |
> |---|---|---|---|---|
> | `synra_core::Kernel` | ✅ | ❌ | ❌（但 `Host::listen_on` 在库里） | ❌ |
> | daemon 的 `Runtime` | ✅ | ✅ | ✅ `transport.rs`，765 行，**在二进制里** | ✅ |
> | app 的 `AppState` | ✅（与上面各重复一份） | ❌ | ❌（`Host` 在手却从不 `listen_on`） | ✅ |
>
> ★ **这一条解释了此前二十来轮里散落的一整族缺口**，它们不是彼此独立的 bug：
> shell 扫不了描（没有发现任务）、`DesktopHost::device_send` 拒绝一切（从不监听，也就没有连接可发）、
> `Kernel::record_action` 没有调用者（没有 Kernel）、`AppState` 与 daemon 各持一份 store/identity/host。
>
> ★ 而 `synra-daemon` 在文档里是**可选的**那个（见上面第三条要点与之对应的 `kernel-remote` feature），
> 现在却是唯一跑起来的网络实现。
>
> ★ **第一步已完成（已提交）：接受连接的那一半现在在库里。**
> * `crates/synra-core/src/listener.rs` ← daemon 的 `transport.rs`（`serve`/`handle`/`handle_within`/`bind`）。
> * `crates/synra-core/src/inbound.rs` ← daemon 的 `dispatch.rs`（`Decision`/`Inbox`/`PairingAnswer`），
>   以及 daemon 的 `pairing.rs`（现为 `inbound::pairing_state_for`）。
> * daemon 的三个文件变成一行 `pub use` —— **而它的测试留了下来**
>   （`apps/synra-daemon/src/transport/tests.rs`），因为它们跑的是真的 `KernelHost`；
>   在库里重写一份就得先捧一个库自己想出来的 host，那种测试测的不是产品会跑的东西。
> * 证据：搬前搬后 `cargo test --workspace` 都是 **55 suites / 2771 passed**，clippy 与 5 个 guard 均绿。
>
> ★ **第二步也完成了：`Supervisor` 不再只是记账。**
> 它一直完整存在（注册、取消令牌、重启策略、每任务一条记录、停机报告）并且有测试，
> 而**真正跑任务的那一个函数 `run_task` 除了它自己的测试没有任何调用者**；daemon 用的是裸的
> `std::thread::spawn` 加一个进程全局 `Arc<AtomicBool>`。现在：
> * `listener::serve` 与 `worker::run`/`run_recording` 改收 **`&Cancellation`**，取消令牌就是它们的停止条件；
>   一个单向令牌不会像那个叫 `stop` 却表示“继续跑”的布尔值一样把极性写反——而那个极性**确实被写反过一次**，
>   后果是监听循环第一次检查就退出，于是设备绑定了端口、广播了端口，却静默拒绝每一个连接。
> * daemon 建一个 `Supervisor`，注册 `transport.accept` 与 `discovery.udp`，用 `run_task` 跑起来；
> ★ **而第 111 轮发现另两个长驻循环还是裸的 `std::thread::spawn`**：心跳与保留期清理。
> 裸 spawn 的问题不是没有监督，而是**一个 panic 会杀掉那条线程而进程照常服务** ——
> 死掉的任务与安静的任务在任何地方都长得一样。现在两者都走 `run_task`
> （`start_loops()`），与上面两个服务同一条路。
> ★ **而本轮先写错了一次**：我自己写了一个 `supervised()` 函数，
> 它做的事与 `run_task` 一模一样 —— 因为我只 grep 了 `.register(` 而没发现 `run_task` 已经在用。
> **一个已有的助手旁边又写一个，正是这几轮一直在清的那个毛病。**
> ★ 而剩下的裸 spawn（控制循环、接受后的每连接线程等）**仍是裸的**，记在这里。
>   心跳发现自己不再拥有目录时调 `begin_shutdown()`，而不是去改一个全局布尔值 —— **停机只剩一条路**。
> * 证据：`cargo test --workspace` **55 suites / 2772 passed**（+1），clippy、5 guard、matrix 均绿。
>   新增那一条断言“工作者被取消后会返回”——它自己的文档原本声称它不会。
>   阴性对照：把 `while !cancel.is_cancelled()` 改回 `loop`，该测试**挂死**而不是失败。
>
> ★ **还差的**：`synra_core::Kernel` 依然无人构造——daemon 拿的是一个裸的 `Supervisor`，
> 而不是图里那个把 store/identity/host/事件总线/审计环一起持有的 `Kernel`。
>
> ★ **为什么这一步不能只做一半（已量过，供下一轮直接用）。**
> `Kernel::bootstrap` **吃掉** `SynraDb`，而 daemon 的 `Runtime` 自己持有一份
> `Arc<Mutex<SynraDb>>`（`state.rs:67`，其文档说明"一个连接意味着写入顺序是这段代码的性质而不是锁的性质"）。
> 于是 `Runtime` **不能**同时持有 `Kernel` 和自己的 `db`——那会是同一个 SQLite 文件上的两个连接，
> 恰好丢掉那条性质。所以这不是"加一个字段"，而是"把三组访问点换掉"：
>
> | 字段 | 访问点 | 转换形状 |
> |---|---|---|
> | `db` | **30** | `self.db().x()` → `self.kernel.with_store(\|db\| db.x())` —— 近乎机械，因为 `with_store` 直接返回闭包的 `T` |
> | `host` | **13** | `self.host.lock()` → 需要重组作用域，**不是重命名**：守卫的生命周期在今天决定了代码块边界 |
> | `identity` | **10** | `self.identity` → `self.kernel.identity()` —— 机械 |
>
> ★ 而剩下的字段（`discovery`/`pairing`/`plugins`/`board`/`ports`/`scan_until_ms`）**不属于** `Kernel`，
> 它们是宿主自己的决定——`state.rs:86-90` 已经写明了这一点（"引擎是宿主的决定，而这是宿主"）。
> 所以正确的终态是 `Runtime { kernel: Arc<Kernel>, ...宿主的那些... }`，而不是把所有东西塞进 `Kernel`。

### 2.2 移动端（Android / iOS）

```
┌──────────────────────────────────────────────────────────────────┐
│ Synra.app / Synra.apk                                            │
│                                                                  │
│  ┌────────────────────────────────────┐                          │
│  │ 宿主 Activity / ViewController      │                          │
│  │  WebView (UI)                       │                          │
│  │   + 插件 UI (iframe / 次级 WebView) │                          │
│  └───────────────┬────────────────────┘                          │
│                  │ Tauri IPC (JNI / C FFI)                       │
│  ┌───────────────▼────────────────────┐                          │
│  │ Rust 内核（编译为 .so / .a）         │                          │
│  │  tokio runtime + Kernel             │                          │
│  │   ├─ Discovery (UDP socket)         │                          │
│  │   ├─ Transport (TCP socket)         │                          │
│  │   └─ PluginHost (wasmtime)          │                          │
│  └────────────────────────────────────┘                          │
└──────────────────────────────────────────────────────────────────┘
```

**关键决策：移动端不用 wasmtime 跑 headless 插件？**

这是一个必须提前决定的取舍（详见 [21 ADR-005](./20-adr.md)）：

| 方案                            | 优点                                     | 缺点                                                     |
| ------------------------------- | ---------------------------------------- | -------------------------------------------------------- |
| A. 移动端也内嵌 wasmtime         | 语义完全一致，插件一份 artifact 到处跑      | 二进制体积增加（wasmtime 约 5–15 MB，可裁剪）；iOS 对 JIT 有限制，需用 AOT/pulley 解释器 |
| B. 移动端不跑 headless 插件      | 体积小；移动端本来也很少有"重计算插件"      | 需要"插件能力按平台分级"；跨端同步要处理"目标端不支持"      |
| C. 移动端用轻量 Wasm 解释器       | 体积小（如 `wasmi` ~1 MB）                | 性能低（但 headless 插件通常不重计算）                     |

> **建议：默认 A（wasmtime，`cranelift` 关闭、用 `pulley` 解释器或 `winch`）**，并在 feature flag 上允许切到 C（`wasmi`）。
> 理由：插件生态的一致性比移动端那几 MB 更值钱；且 iOS 本来就不允许 JIT，任何方案都得走解释/AOT。

---

## 3. Tauri 承担什么、不承担什么

### 3.1 Tauri 负责（宿主壳职责）

| 职责               | 具体                                                                          |
| ------------------ | ----------------------------------------------------------------------------- |
| 窗口与多窗口管理    | 主窗口、插件 UI 窗口（可选）、设置窗口                                          |
| 系统集成           | 托盘、菜单、全局快捷键、开机自启、单实例                                        |
| 通知               | 系统通知（插件同步完成、收到剪切板、执行失败）                                   |
| 深链与文件关联      | `synra://` scheme、注册文件类型                                                 |
| 权限申请           | 移动端网络权限、通知权限、相册/文件访问权限                                      |
| IPC 边界           | 注册 `#[tauri::command]`，把内核错误映射为可序列化错误                           |
| 平台能力实现        | `Platform` trait 的 Tauri 实现：打开 URL、打开文件、模拟按键、读剪贴板、通知       |
| WebView 生命周期    | UI 插件的 WebView/iframe 创建与销毁                                             |
| 自动更新           | Tauri updater（应用本体，不含插件）                                              |

### 3.2 Tauri **不**负责

| 不负责              | 原因                                                       |
| ------------------- | ---------------------------------------------------------- |
| 设备发现与连接       | 属于内核（`synra-net`）                                     |
| 协议编解码           | 属于内核（`synra-protocol`）                                |
| 插件沙箱与能力闸门    | 属于内核（`synra-plugin-host`）                             |
| 业务状态机           | 属于内核（`synra-core`）                                    |
| 插件安装与校验        | 属于内核（`synra-plugin-host` + `synra-store`）              |
| 数据持久化           | 属于内核（`synra-store`）；Tauri 只提供路径                  |

> **判据**：如果一个功能在"没有 UI"的情况下也需要工作，它就不属于 Tauri。

### 3.3 为什么选 Tauri（与 ADR-001 呼应）

| 需求                     | Tauri 2 | 自研 wry/tao | Electron | 纯原生 |
| ------------------------ | ------- | ------------ | -------- | ------ |
| Rust 一等公民             | ✅      | ✅           | ❌       | ✅     |
| 桌面 + 移动一套代码       | ✅      | ❌           | ❌       | ❌     |
| 强类型 IPC                | ✅      | 自行实现     | ❌       | —      |
| 系统 WebView（体积小）     | ✅      | ✅           | ❌       | ✅     |
| 插件生态（通知/托盘/深链） | ✅      | ❌           | ✅       | —      |
| 迁移成本                  | 低      | 高           | 中（但失去 Rust 收益） | 极高 |

**已知取舍**：
- 系统 WebView 意味着渲染差异（WebView2 / WKWebView / WebKitGTK）。前端需按 `target: webview2 | wkwebview | webkitgtk` 做兼容矩阵（[10](./08-ipc-and-frontend.md) §6）。
- Linux 上 WebKitGTK 的 WebView 隔离能力弱于 Chromium；UI 插件隔离方案在 Linux 上退化为 iframe（同 realm，见 [11](./09-plugin-system.md) §5.4）。

---

## 4. 数据流：一次"手机分享 → PC 执行"

这是检验架构是否成立的核心路径。

```
① [手机 UI] 用户分享链接给 Synra
     前端：invoke('share_intake', { items: [...] })
                                         ── IPC ──▶
② [手机内核] ShareIntake
     a. 规范化 → ShareItem { kind: Url, url, source_app, ts }
     b. 查询"哪些插件声明了 share 入口"（本地 PluginStore）
     c. 逐个问插件 can_handle(item)：Wasm host function 调用（有超时 + fuel 上限）
     d. 收集候选 → 发布事件 plugin.candidates_ready 给前端
                                         ◀── event ──
③ [手机 UI] 渲染候选列表 / 或按插件配置自动选
     前端：invoke('intent_dispatch', { plugin_id, action, payload, target_device })
                                         ── IPC ──▶
④ [手机内核] IntentRouter
     a. capability 闸门：该插件的 grants 是否覆盖 device:send:<target> + 该 action
     b. 解析目标设备 → TransportRegistry 拿链路（若无则先建连）
     c. 构造 Envelope { from, to, event: "_plugin.<slug>.<action>", request_id, trace_id, payload }
     d. 交给 synra-net 发送
                                         ── 网络 ──▶
⑤ [PC 内核] 入站
     a. 帧解析 → Envelope 校验（白名单、版本、来源信任）
     b. 事件路由 → 找到目标插件（按 slug）
     c. capability 闸门：该插件是否声明了该 action 的处理权
     d. 交给插件执行（Wasm host function）
     e. 插件需要"打开浏览器" → 调用 host function platform.open_url
        内核把请求转给宿主壳（Platform trait 实现）
                                         ── 平台调用 ──▶ 浏览器打开
     f. 构造回执 Envelope { event: "_synra.intent.result", reply_request_id, payload: {ok, code?} }
                                         ── 网络 ──▶
⑥ [手机内核] 收到回执 → 匹配 request_id → 发布事件 intent.result 给前端
                                         ◀── event ──
⑦ [手机 UI] toast：成功 / 失败原因
```

**关键点**：
- 第 ②/④/⑤ 步全在内核，**前端无法绕过**；
- 每一步都有 `trace_id`，可在 [18](./18-observability.md) 里串起来；
- 插件在 ②（probe）与 ⑤（execute）都被调用，但**永远不接触网络**——它只说"我要发这个信封"，发不发由内核决定。

---

## 5. 插件运行的两种形态（本架构的核心创新点）

```text
┌─────────────────────────────────────────────────────────────────────────────┐
│  形态 A：Headless 插件（推荐用于所有"动作/数据处理"插件）                       │
│                                                                             │
│   artifact: <plugin>.wasm  （wasm32-wasip2 组件）                            │
│   运行位置: 内核内的 wasmtime 实例                                            │
│   接口:     WIT 世界（synra:plugin@1.0.0）                                    │
│   能力:     只能通过 host functions 访问宿主；无文件、无网络、无时钟（除非授予）│
│   配额:     线性内存上限 / fuel / epoch 中断 / 输出大小上限                     │
│   生命周期: 内核管理（实例池、空闲回收）                                        │
│   适用:     分享匹配、动作执行、数据处理、快捷键、协议解析                       │
└─────────────────────────────────────────────────────────────────────────────┘

┌─────────────────────────────────────────────────────────────────────────────┐
│  形态 B：UI 插件（只有需要自定义界面时才用）                                    │
│                                                                             │
│   artifact: <plugin>.zip 含 ui/index.js (+ 资源)                             │
│   运行位置: 隔离的 WebView / iframe                                          │
│   接口:     宿主注入的 `window.synra` API（由内核 IPC 代理）                    │
│   能力:     渲染 UI + 通过 IPC 调用内核能力（受同一套 capability 约束）          │
│   隔离:     iframe (sandbox) / 独立 WebView + 独立 origin                     │
│   适用:     需要自定义交互的插件（聊天界面、快捷键面板、复杂配置）                │
└─────────────────────────────────────────────────────────────────────────────┘

一个插件可以同时提供两者（manifest 里声明 `headless: true` + `ui: true`），
但两者是独立 artifact、独立加载、独立授权。
```

> **为什么这样分？** 见 [11](./09-plugin-system.md) §2 与 [21 ADR-002](./20-adr.md)。
> **一句话**：把"必须隔离的"（逻辑/动作）放进 Wasm，把"必须渲染的"（界面）放进 WebView——各自用自己领域里最强的隔离手段。

---

## 6. 部署形态

| 形态            | 产物                                    | 说明                                                     |
| --------------- | --------------------------------------- | -------------------------------------------------------- |
| 桌面应用        | `synra-desktop`（.msi/.dmg/.AppImage/deb） | Tauri bundle，内含内核与默认 UI                           |
| 移动应用        | `synra-mobile`（.apk/.aab/.ipa）          | Tauri mobile，Rust 编译为 native library                  |
| CLI             | `synra`                                  | 复用内核：`synra devices` / `synra send` / `synra plugin ...` |
| 后台守护        | `synra-daemon`                           | 无 UI，仅内核；可被桌面 UI 连接（可选形态）               |
| 插件开发工具    | `synra-plugin`（CLI）                     | `new` / `build` / `verify` / `sign` / `publish` / `dev`    |
| 插件运行时（Wasm） | `<plugin>.wasm`                        | 分发 artifact                                             |
| 插件 UI         | `<plugin>.zip`                           | 分发 artifact                                             |

> **CLI 与 daemon 是"内核不依赖 UI"这条铁律的验收标准**：如果它们能编译并工作，说明内核边界是干净的。

---

## 7. 协议的共存策略

设备不一定同时升级。策略由 [04 §9](./04-wire-protocol.md) 规定，要点：

```
握手时交换 protocol（u16）与 capabilities 中的 Feature 列表
   │
   ├─ major 相同 → 完成握手；该链路的行为 = 双方 Feature 的**交集**（一次性确定）
   └─ major 不同 → 回 Error 帧（reason: ProtocolVersionUnsupported），
                   UI 提示"需要升级对方设备"
```

★ **不存在"兼容模式"这段代码**：协议从第一版就设计成"相邻版本可以共存"，
每个 Feature 只增不改（[ADR-010](./20-adr.md)）。

细节见 [04 §9](./04-wire-protocol.md)。

---

## 8. 关键决策速查

完整的 ADR 见 [20](./20-adr.md)。这里只列影响架构形态的几条：

| 决策                     | 选了                                                            | 为什么（一句话）                                        |
| ------------------------ | --------------------------------------------------------------- | ----------------------------------------------------- |
| 宿主外壳                  | Tauri 2（桌面 + 移动）                                            | 内核已是 Rust，同进程调用没有序列化开销                    |
| 插件形态                  | 双轨：headless Wasm + UI WebView                                  | 逻辑必须强隔离且可 headless；界面必须能自由绘制            |
| Wasm 接口                 | 组件模型 + WIT                                                    | 需要接口级别类型安全与版本化演进                           |
| 传输                      | TCP + 自定义帧起步，QUIC 可选                                       | 局域网够用；明文可调试；性能瓶颈不在传输                   |
| 移动端 Wasm               | 内嵌 wasmtime（pulle/winch）                                       | 换"一份 artifact 全平台跑"（iOS 无 JIT）                   |
| UI 插件隔离               | 独立 origin + CSP；桌面**可选**独立 WebView                         | `connect-src 'none'` 让插件无法绕过内核出网                |
| 插件安装                  | 分发预构建 artifact + 强制签名                                       | 不在用户机执行构建/安装脚本                               |
| 持久化                    | SQLite（`rusqlite` bundled）                                      | 有关系数据 + 需要事务 + 需要并发读                          |
| 集群 / 选主               | **不做**                                                          | 当前场景全是对等点对点；复杂度收益不成立                    |
