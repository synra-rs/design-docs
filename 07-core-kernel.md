# 07 · 领域内核

> ★ **本次核对发现的核心事实，它一次解释了此前几轮分别记下的好几个洞：**
> **`synra_core::Kernel` 这个类型在生产代码里没有任何使用者 —— 只有它自己的测试。**
>
> 证据（全仓 `crates/`、`apps/`、`tools/` 搜过）：
> * `Kernel::in_memory` / `Kernel::open` / `Kernel::desktop` / `Kernel::headless`
>   只出现在 `crates/synra-core/tests/kernel_tests.rs`。
> * `Kernel::` 这个前缀一共出现在五个文件里：它自己的 `kernel.rs`、`lib.rs`（re-export）、
>   `kernel_tests.rs`，以及 **daemon/control 侧的 `synra_control::Kernel`（那是另一个东西，见下）**。
> * `synra-rust-app` 的 Rust 侧一次都没提过它。
>
> ★ **有两个不同的 `Kernel`，这正是它容易被忽略的原因：**
> * `synra_core::Kernel` —— **结构体**，本文第 9 节讲的"内核构造与生命周期"。有 bus、supervisor、
>   审计环、数据库句柄、事件订阅、快照。
> * `synra_control::Kernel` —— **trait**，`apps/synra-daemon/src/runtime/commands.rs:19`
>   有一句 `impl Kernel for Runtime`。守护进程实现的是这个。
> 两者同名而无关，所以"搜 Kernel"永远能搜到东西，而**搜到的不是那个没人用的**。
>
> ★ **守护进程没有用 `Kernel`，而是把它的零件手工拼了起来**：
> `Runtime` 里是 `synra_core::Host`、`DiscoveryService`、`Identity`、`SynraDb`、`PairingDesk`、`PluginSet`
> —— 都是本文描述的同一批东西，只是由 `apps/synra-daemon/src/serve.rs` 自己组装，
> 而不是交给 `Kernel::bootstrap`。
> 这不是"少写了一行"：`Kernel` 自己持有 bus 与 supervisor，所以**不用它就意味着那两个子系统没有生产者**。
>
> ★ **这就是下面这些现象的共同原因**（前几轮分别记录，现在归到一处）：
> * **没有事件到达任何界面** —— bus 在 `Kernel` 里，而 `Kernel` 没人构造；
>   `Kernel::publish` / `next_event` / `subscribe` 在生产里零调用。
>   于是 app 侧 `TauriKernel.listen('kernel-event', …)` 订阅的是一个永远不触发的通道
>   （[08](./08-ipc-and-frontend.md) §10 记的那条）。
> * **审计是空的** —— `Kernel::record_action`（`kernel.rs:676`）是审计环唯一的写入路径，
>   而它零调用；`AuditRing` 有界、有测试、有查询（`for_plugin` / `refusals_for_plugin`），
>   就是没有生产者。
> * **监督树没有驱动** —— `Supervisor` / `TaskSpec` 只经由 `Kernel` 可达。
>
> ★ **本文其余部分因此要这样读**：它描述的是一个**被完整实现、被完整测试、但没有被任何宿主采用**的编排层。
> 每一节说的东西大概都在代码里找得到；找不到的是**那个把它们连起来并在生产中运行的调用者**。
> 下一步不是再补一个零件，而是决定：让守护进程改用 `Kernel::bootstrap`，
> 还是把 bus / 审计 / supervisor 从 `Kernel` 里取出来接到 `Runtime` 上。
> 这是一个架构决定，不是一行改动，所以本轮只把它记清楚。

> 本文是 Rust 重构的**技术核心**。它定义内核的领域模型、状态归属、并发结构、错误模型与扩展点。
> 所有实现细节都以"Rust 最佳实践"为准：类型状态、`trait` 边界、结构化并发、显式错误。

---

## 1. 设计目标与不变量

| # | 不变量                                                                     | 验收方式                                                     |
| - | -------------------------------------------------------------------------- | ------------------------------------------------------------ |
| 1 | 内核可在无 UI 环境编译并运行                                                | `cargo check -p synra-cli`；`synra status` 可工作             |
| 2 | 内核不依赖任何 GUI crate                                                    | `cargo-deny` banned list                                     |
| 3 | 每个聚合根是唯一状态权威                                                     | 代码审查：不存在第二处 `HashMap<DeviceId, _>` 表示同一状态      |
| 4 | 所有跨设备副作用必须经 `Kernel` 的显式入口                                    | 插件只有 host function；`synra-net` 不对外暴露裸 send          |
| 5 | 所有状态变更以事件形式对外发布                                                | `Kernel::subscribe()` 是 UI 获取状态的唯一途径                 |
| 6 | 任何 `async` 任务的失败都有监督与重启策略                                     | `supervisor.rs` 注册表覆盖所有长驻任务                          |
| 7 | 关键路径（帧编解码、capability 匹配、状态转移）有 proptest 覆盖                | CI 覆盖率门禁                                                 |

> 不变量 3 直接消灭 [05 P12](./11-design-principles.md)（状态分散）；不变量 4 消灭 P6/P7（插件绕过）；不变量 5 消灭 P13（竞态）。

---

## 2. 聚合根与所有权划分

```
                         ┌──────────────────────────────────────────┐
                         │              Kernel                      │
                         │  Arc<Kernel>  ·  构造一次，共享使用       │
                         └──────┬───────────────────────────────────┘
                                │
     ┌──────────────┬───────────┼────────────┬──────────────┬─────────────┐
     │              │           │            │              │             │
┌────▼─────┐ ┌──────▼─────┐ ┌───▼───────┐ ┌──▼─────────┐ ┌──▼────────┐ ┌──▼─────────┐
│ Identity │ │ Discovery  │ │  Trust    │ │ Transport  │ │  Intent   │ │  Plugins   │
│          │ │            │ │           │ │            │ │           │ │            │
│ 本机身份 │ │ 局域网可见  │ │ 配对与信任 │ │ 链路生命周期│ │ 意图路由  │ │ 插件生命周期│
│ 持久      │ │ 会话       │ │ 持久       │ │ 会话       │ │ 会话      │ │ 持久+会话  │
└──────────┘ └────────────┘ └───────────┘ └────────────┘ └───────────┘ └────────────┘
     │              │           │            │              │             │
     └──────────────┴───────────┴────────────┴──────────────┴─────────────┘
                                │
                    ┌───────────▼────────────┐
                    │  TransferSessions      │
                    │  大对象传输会话（会话）  │
                    └───────────┬────────────┘
                                │
                    ┌───────────▼────────────┐
                    │  Config                │
                    │  配置聚合（持久）        │
                    └────────────────────────┘
```

### 2.1 每个聚合根的职责与状态

| 聚合根               | 拥有的状态                                                        | 生命周期 | 持久化           |
| -------------------- | ----------------------------------------------------------------- | -------- | ---------------- |
| `Identity`           | `device_id`、`display_name`、`profile`（平台/版本/能力）            | 进程     | ✅ store         |
| `Discovery`          | 候选设备表（UDP/mDNS/probe 原始结果）、本机 IP 集合、扫描任务状态     | 会话     | ❌（易失）        |
| `Trust`              | 已配对设备记录、待接受配对请求、配对握手状态                          | 进程     | ✅ store         |
| `Transport`          | 链路表（`DeviceId → Link`）、出站别名、socket 所有权                 | 会话     | ❌               |
| `Intent`             | 进行中的意图（匹配中/待确认/执行中/已回执）、幂等去重表                | 会话     | ⚠️ 可选（离线排队）|
| `Plugins`            | 插件清单、安装记录、启用状态、权限授予、运行实例                       | 进程     | ✅ store         |
| `TransferSessions`   | 进行中的传输会话、分块缓冲区、checkpoint                            | 会话     | ⚠️ 可选（续传）    |
| `Config`             | 用户设置（网络、同步策略、UI 偏好）                                   | 进程     | ✅ store         |

**共有状态视图**（UI 看到的"设备卡片"）是**派生**的，不单独存储：

```
DeviceCard = f(Discovery.candidate, Trust.paired?, Transport.link_state, Identity.is_self)
```

> Rust 版的 `DeviceCard` 是**每次从四个聚合根读出来算一遍**（或用一个 `watch` 通道缓存），不存在"合并"这一步。

### 2.2 为什么不用一个巨大的 `Mutex<State>`

容易想到的写法是：

```rust
// ❌ 反面教材
pub struct Kernel {
    state: RwLock<AppState>,   // 所有状态在一个大锁里
}
```

问题：
- 锁粒度过粗，网络 IO 与 UI 查询互相阻塞；
- 无法表达"某状态下某操作不合法"（只能运行时 `if` 检查）；
- 无法对单个子系统做单测。

**采用的写法**：每个聚合根自己持有内部状态，用**消息传递 + 细粒度锁**暴露操作，用**类型状态机**表达合法性。

---

## 3. 平台抽象：`trait Platform`

内核需要宿主提供的能力分为三类：

```rust
// crates/synra-platform/src/lib.rs

/// 宿主提供的系统能力（打开 URL/文件、模拟按键、通知、剪贴板…）
#[async_trait]
pub trait PlatformCapability: Send + Sync + 'static {
    fn kind(&self) -> PlatformKind;

    /// 打开 URL（浏览器 / 默认应用）
    async fn open_url(&self, url: &Url) -> Result<()>;

    /// 用默认应用打开文件
    async fn open_path(&self, path: &Utf8Path) -> Result<()>;

    /// 在 Finder / 资源管理器中显示
    async fn reveal_path(&self, path: &Utf8Path) -> Result<()>;

    /// 模拟按键（快捷键面板场景）
    async fn simulate_keys(&self, keys: KeyChord) -> Result<()>;

    /// 系统通知
    async fn notify(&self, notification: Notification) -> Result<()>;

    /// 读剪贴板 / 写剪贴板
    async fn read_clipboard(&self) -> Result<ClipboardContent>;
    async fn write_clipboard(&self, content: ClipboardContent) -> Result<()>;

    /// 捕获当前 OS 选中的文本（桌面专属；不支持则返回 Err(Unsupported)）
    async fn capture_selection(&self) -> Result<String>;

    /// 获取当前活动窗口（未来：智能路由）
    async fn active_window(&self) -> Result<Option<WindowInfo>>;

    /// 设备能力自述（哪些能力可用），用于 capability 协商与 UI 提示
    fn capabilities(&self) -> &PlatformCapabilities;
}
```

关键设计点：

1. **`#[async_trait]` 是可接受的**（`async fn in trait` 已在 Rust 1.75+ 稳定，但 object-safety 限制使其无法直接用于 `dyn`；实现时优先用 `async-trait` 或 `trait_variant`，并在 ADR 里记录选择）。
3. **`capabilities()` 让内核能做能力探测**——例如"这个设备能执行 browser.open 吗"，进而影响意图路由与 UI 展示。
4. **`PlatformCapability` 不包含网络或文件系统**——那些是内核自己的职责（`synra-net`、`synra-store`），不属于"宿主差异"。

### 3.1 平台实现矩阵

| 实现                    | 目标                        | 说明                                                        |
| ----------------------- | --------------------------- | ----------------------------------------------------------- |
| `DesktopPlatform`       | Windows / macOS / Linux     | Tauri 实现：`opener`、`clipboard-manager`、`notification` 等  |
| `AndroidPlatform`       | Android                     | 通知、Intent 打开 URL、无障碍服务（模拟按键需要单独权限）      |
| `IosPlatform`           | iOS                         | `UIApplication.open`、通知、无模拟按键能力（返回 Unsupported） |
| `HeadlessPlatform`      | CLI / daemon / 测试          | 全部返回 `Unsupported` 或记录到日志（供 CLI 使用）             |
| `MockPlatform`          | 测试                        | `synra-testkit` 提供，记录调用供断言                          |

> `HeadlessPlatform` 的存在是**内核可 headless 的证明**。

---

## 4. 类型状态机：链路生命周期


### 4.1 设计

```rust
// crates/synra-core/src/link.rs

/// 链路的存在本身就意味着"已连接"。状态由类型参数表达。
pub struct Link<S: LinkState> {
    device_id: DeviceId,
    socket: SocketHandle,          // 仅在连接的链路中存在
    opened_at: Instant,
    trace: TraceContext,
    _state: PhantomData<S>,
}

pub struct Connecting;   // TCP 已发起，等 connect.ack
pub struct Ready;        // 握手完成，可以收发
pub struct Draining;     // 已发 close，等在途消息落地

impl Link<Connecting> {
    /// 唯一能从 Connecting 出来的方式：收到合法 ack 或超时失败
    pub async fn complete_handshake(
        self,
        ack: ConnectAck,
        timeout: Duration,
    ) -> Result<Link<Ready>> { /* ... */ }

    pub fn fail(self, reason: HandshakeError) -> FailedLink { /* ... */ }
}

impl Link<Ready> {
    pub async fn send(&self, frame: &Frame) -> Result<SendOutcome> { /* ... */ }
    pub async fn heartbeat(&self) -> Result<()> { /* ... */ }

    /// 关闭需要显式降级到 Draining
    pub async fn begin_close(self) -> Link<Draining> { /* ... */ }
}

impl Link<Draining> {
    /// 等待在途消息，随后销毁 socket
    pub async fn finish(self, grace: Duration) -> ClosedLink { /* ... */ }
}
```

**收益**：
- `Link<Ready>::send` 在类型上保证 socket 存在 → 消灭"connection is not open"运行时错误；
- `Connecting → Ready` 只有一条路径 → 消灭"`handshaking` 从未被赋值"这类状态空洞；
- 失败必须显式构造 `FailedLink` → 消灭"activate 抛错后卡在 `entering`"（[14](./14-plugin-lifecycle.md) §6 同理）。

### 4.2 但要注意：不要在类型状态机上过度设计

| 场景                     | 用类型状态机 | 用枚举 |
| ------------------------ | ------------ | ------ |
| 链路（生命周期长、状态少、转移严格） | ✅         |        |
| 插件实例（同样严格）      | ✅           |        |
| 传输会话（状态多达 7 个、转移复杂） |            | ✅（用枚举 + 私有构造，避免类型爆炸） |
| 配对握手（3 状态，但分支多） |            | ✅     |

> **判据**：状态数 ≤ 4 且"非法转移必须编译期禁止"→ 类型状态；否则枚举 + 私有字段 + 显式 `transition()` 方法。

### 4.3 不要用运行时 `if` 检查状态

```rust
if self.state == LinkState::Ready { self.socket.send(...) } else { Err(...) }

// ✅ 正确
impl Link<Ready> { pub async fn send(&self, ...) -> ... { self.socket.send(...) } }
```

---

## 5. 结构化并发与监督

### 5.1 任务分类

| 类别           | 例子                                        | 生命周期      | 失败策略                     |
| -------------- | ------------------------------------------- | ------------- | ---------------------------- |
| 引导任务        | 加载 store、恢复配对、启动监听               | 启动一次      | 失败 → 内核启动失败（显式）    |
| 长驻服务任务    | UDP 发现循环、TCP accept 循环、心跳 ticker   | 进程          | 失败 → 指数退避重启（有限次）  |
| 每链路任务      | 单链路的读循环、写队列、心跳                  | 链路          | 失败 → 标记链路失败，回收资源  |
| 每插件任务      | 插件的异步 host function、定时器              | 插件实例      | 失败 → 插件进入 `Faulted`，不影响其它 |
| 一次性任务      | 一次意图派发、一次文件传输                    | 请求          | 失败 → 返回结构化错误给调用方  |

### 5.2 监督器

```rust
// crates/synra-core/src/supervisor.rs

pub struct Supervisor {
    tasks: Mutex<HashMap<TaskId, TaskHandle>>,
    shutdown: CancellationToken,
}

pub struct TaskSpec {
    pub id: TaskId,                 // 稳定标识，便于重启与诊断
    pub restart: RestartPolicy,     // Never | Backoff { initial, max, max_attempts }
    pub critical: bool,             // critical 任务失败会触发内核降级
    pub factory: Box<dyn Fn(SupervisorCtx) -> BoxFuture<'static, Result<()>> + Send + Sync>,
}

impl Supervisor {
    /// 派生一个受监督任务；返回的 JoinHandle 不会被裸用
    pub fn spawn(&self, spec: TaskSpec) -> TaskId;

    /// 优雅关停：先取消，再等在途任务，最后强制超时
    pub async fn shutdown(&self, grace: Duration) -> ShutdownReport;
}
```

**规则**：

1. 内核里**禁止**直接 `tokio::spawn`（除 supervisor 内部）。用 `dylint` 自定义 lint 或 code review 强制。
2. 每个任务有**稳定 TaskId**（例如 `"discovery.udp"`、`"transport.link.{device_id}"`），出现在日志与诊断包里。
3. 长驻任务失败**必须**有退避重启，且重启次数进入指标（[18](./18-observability.md)）。
4. 关停顺序：先停发现与 accept（不再建立新链路），再停心跳，再 `Draining` 现有链路，最后 flush store。

### 5.3 背压与容量

每个会积压的地方都必须有**显式容量与背压策略**：

| 位置               | 容量策略                                        | 溢出行为                          |
| ------------------ | ----------------------------------------------- | --------------------------------- |
| 链路写队列          | 有界（例如 256 帧或 1 MiB）                      | 溢出 → 链路标记不健康并关闭        |
| 链路读循环          | 逐帧处理，处理时间受限（插件调用有超时）           | 超时 → 丢弃该帧 + 记录（或断链）    |
| 事件广播（给 UI）   | `tokio::sync::broadcast` 有界（例如 1024）        | 落后 → UI 收到 `Lagged` → 触发全量重取 |
| 插件 host call 并发 | 每插件有界（例如 8）                             | 排队，超时则拒绝                   |
| 传输分块           | 滑动窗口（例如 32 块未确认）                      | 按窗口节流                         |


---

## 6. 意图路由（`Intent`）—— 内核最核心的编排

> ★ **本节描述的东西实现得很完整，而它在生产里没有任何入口 —— 这是全系统最大的功能空洞。**
>
> 逐条核对（全仓 `crates/`、`apps/`、`tools/`）：
> * `Matcher`（`dispatch/matching.rs`，`:213` 的 `match_share`）—— **只有 `dispatch/tests.rs` 在用它**，
>   19 处全是 `Matcher::new(&mut set).match_share(...)`。
> * **控制协议根本传不了它**：`synra_control::Request` 的变体只有
>   `Hello` / `Status` / `Devices` / `SendText` / `SendFile` / `Plugins` / `Logs` / `Pair` / `PairAnswer`
>   —— **没有 `Intent*`**。守护进程实现的 `Kernel` trait 也没有任何 intent 方法
>   （只有 `version` / `status` / `devices` / `send_text` / `send_file` / `plugins` / `logs` / `pair` / `pair_answer`）。
> * CLI 的 args 里没有 intent 子命令（只有 `devices` / `status` / `send` / `plugin` / `logs` / `doctor` / `db`）。
> * app 侧有 `intent_dispatch` / `intent_snapshot` / `intent_decide` / `intent_cancel`
>   四个名字（在 `command.rs` 的 39 条契约里），四个都在 26 个未注册命令里。
>
> ★ **而守护进程收到一份分享时做的是：把它写进 inbox 目录。**
> `apps/synra-daemon/src/dispatch.rs` 的 `absorb` 落盘、校验文件名、拒绝越界路径 —— 这些都做得很仔细 ——
> 然后**没有任何东西去问插件"你能处理这个吗"**。
> 也就是说：**产品价值的落点（本节第一句自己写的）在端到端意义上不存在。**
> 一台设备发来的分享会变成对方硬盘上的一个文件，而不会变成一次动作。
>
> ★ 这与 [第 21 轮记下的那件事](./07-core-kernel.md) 是同一个形状但**不是同一个原因**：
> 那个是 `synra_core::Kernel` 没人构造（bus / supervisor / 审计没有驱动者）；
> 这个是**编排本身没有传输层**。修好 `Kernel` 的采用问题**不会**顺带修好这个 ——
> `Matcher` 的调用点需要在控制协议上新增一个请求、在 daemon 上实现它、并给它一个 `Board`。
>
> ★ **现在有入口了（已提交）。** 做法：
> * `synra_control::Request::IntentDispatch { mime, text }` + `Kernel::intent_dispatch(&self, mime, text)` ——
>   请求带的是**内容**而不是 `ShareItem`：`ShareItem.from` 是"哪台设备分享的"，
>   而本机 socket 上的调用方**不是一台设备**——只有内核知道自己是谁。
>   所以调用方发内容，内核补上 `from`，而不是让调用方自己编一个认证不了的 id。
> * 守护进程新增 `runtime/intent.rs`：`raise` → `Matcher::match_share` → `settle`。
>   `Runtime` 持有 `Arc<Mutex<Board>>`（`state.rs`），否则意图没地方存。
>   ★ 窗口是**关掉**而不是等完：`settle` 要 `now >= deadline`，而 deadline 的用途是绑定这一轮——
>   轮到这里时**每个插件都已经答过**，所以再等剩下的毫秒是为了睡而睡。
> * 答案里同时有 candidates 和整轮报告（`asked` / `handled` / `declined` / `offered` / `misbehaved`）——
>   ★ 这两者是为了分开**"什么都没装"与"全部崩了"**：从一个候选列表上看，它们一模一样。
> * CLI 新增 `synra share <TEXT> [--mime]`，它与 `send` 恰好相反：
>   `send` 指定设备并投递，`share` 什么都不指定、让内核去问每个插件。
> * 测试断言的是**候选人**（而不是"调用了").
>   ★ 第一版配置里我把 `can-handle` 的答案写成了 `pluginId` + `requiredCaps`，
>   而代码把它记为 **misbehaved**："can-handle returned missing field `handled`"——
>   正是这一声拒绝证明了配置不是真的插件行为。
> * ★ **现在路由能走到底了：`IntentSnapshot` / `IntentDecide` / `IntentCancel` 已加入控制协议，
>   `IntentDecide` 会 `choose` → `confirm` → `execute` → `conclude`，
>   而 `synra_core::dispatch::execute`（新文件 `dispatch/execute.rs`）是**全系统唯一跑插件动作的函数**。
>   它用自己的 `EXECUTE_BUDGET_MS`（5 s），而不是复用 `CAN_HANDLE_BUDGET_MS`（50 ms）：
>   后者要保证弹窗出现在用户还在看的时候，前者要完成用户要的动作。
> * ★ **三个契约/设计漏洞在这一轮被测试逼出来了**，它们都记在代码里：
>   1. **`Board` 没有回答"在哪里跑"的方法** —— ★ **已修复：新增 `Board::aim(id, target)`。**
>      一个没建议 target 的候选人会停在 `Choosing`（它就是"在哪里"这个问题），
>      而 `aim` 把它推到 `Confirming`——于是 `IntentDecide` 的 `target` 参数又回来了，而且真的做事。
>      ★ **而这两个问题共用一个状态是一个弱点，不是一个设计**：
>      `Choosing { candidates }` 同时表示"选哪个插件"（多个候选人）和"在哪里跑"（单个无 target 候选人），
>      状态本身不说自己在问什么，所以 `aim` 只能靠**候选人个数**反推。
>      它对多候选人**拒绝**（而不是拿第一个），因为那时问的是"哪个插件"，
>      把设备挂到用户没选的候选人上就是在替他选。真正的出路是把两个问题拆成两个状态，那是一次状态机重构。
>   2. **远程 target 不能当地执行。** `confirm` 之所以返回 `request_id`，
>      是因为常规下发是**发给另一台设备的信件**。实现在 target 不是本机时**坦白拒绝**
>      （`Unsupported`），而不是在用户**分享来源**的那台机器上跑他的动作。
>      ★ **它的信使已经存在，只是没人派。** `synra_protocol::event` 定义了
>      `INTENT_INVOKE` / `INTENT_RESULT` / `INTENT_CANCEL`（`event.rs:352-356`），它们进了
>      `ALL`、进了 `AckPolicy::Await`的表、`inbound.rs:459` 的 `is_kernel` 也把它们当内核事件——
>      而**全仓没有任何一处发或收**（命中全在定义、表和测试里）。
>      ★ 所以远程下发**不是契约缺失，是实现缺失**：四件事要做——
>      发信、收信后本地跑插件、回 `INTENT_RESULT`、按 `request_id` 做幂等；
>      而且它还需要 transport 线程拿到 `board` 与 `plugins`（参见下一条）。
>   3. **`execute` 拿不到被分享的东西** —— ★ **已修复**：`Proposal` 新增 `args`，
>      由插件从 `can-handle` 拿到的 item 里拄出它自己需要的，原样传给 `execute`。见 [09](./09-plugin-system.md) 的 `share` 契约注释。
>      ★ 而它能藏一轮的原因值得记：**`ScriptedBackend` 的索引键是 `work.to_string()`，
>      而 `Work` 的 `Display` 是 `interface.function` —— 每一次 `execute` 都落在同一个键上，配置看不见自己的输入。**
>      已改为把 payload 纳入键（`key_for`）；现在把 `args` 改回 `null` 会让**两个**测试变红，而修之前只有一个。
> * ★ **命令行现在能把它走完了。** `synra share <TEXT>` 开始，然后
>   `synra intent show|decide|cancel <ID>`（`--plugin` / `--target` 选填）。
>   在此之前，`share` 打印一个 id 而**没有第二个命令可以花它**——
>   路由可以从命令行开始，不能在命令行结束。
> * ★ **仍然没做的**：路由只到"候选人产生"为止。
>   `intent_decide`（用户选择 / 确认 / 取消）与真正调用插件都还没有，
>   而且这是刻意的："这个能处理吗"这个问题的副作用不该包含跑第三方代码。
>   进程内那条路（`Decision::Delivered`）也还没接上——它需要 inbox 把物料还回来。
>
> ★ 本轮没有动手做它，因为它是一次跨三层的纵向切片（control + daemon + CLI），
> 而不是一处接线；记在这里而不是留白，是因为**读本文的人会以为这一节描述的是能跑的东西**。

这是产品价值的落点（[01](./02-domain-and-flows.md) §1 的三层结构）。

### 6.1 数据模型

```rust
// crates/synra-core/src/intent.rs

/// 用户/系统产生的一个"意图"
#[derive(Debug, Clone, Serialize, Deserialize)]
#[serde(rename_all = "snake_case", tag = "kind")]
pub enum IntentSource {
    /// 系统分享入口（手机分享菜单）
    Share { item: ShareItem, source_app: Option<String> },
    /// 剪切板推送
    Clipboard { content: ClipboardContent },
    /// 插件 UI 主动触发（快捷键面板）
    PluginUi { plugin_id: PluginId, action: ActionName, params: Value },
    /// 本机动作（例如 CLI 直接调用）
    Direct { action: ActionName, params: Value, target: DeviceId },
}

/// 一个候选动作：某个插件声称它能处理这个意图
#[derive(Debug, Clone)]
pub struct IntentCandidate {
    pub plugin_id: PluginId,
    pub plugin_version: Semver,
    pub action: ActionName,
    /// 该动作需要宿主提供的能力（用于路由与校验）
    pub required_caps: Vec<Capability>,
    /// 插件建议的目标设备（None = 由用户/策略选择）
    pub suggested_target: Option<DeviceId>,
    /// 插件提供的展示信息（确认弹窗用）
    pub presentation: CandidatePresentation,
}

/// 意图的完整生命周期
#[derive(Debug)]
pub struct Intent {
    pub id: IntentId,
    pub trace: TraceContext,
    pub source: IntentSource,
    pub state: IntentState,
    pub created_at: Instant,
}

#[derive(Debug)]
pub enum IntentState {
    Matching { deadline: Instant, candidates: Vec<IntentCandidate> },
    AwaitingUserChoice { candidates: Vec<IntentCandidate> },
    AwaitingConfirm { candidate: IntentCandidate, target: DeviceId },
    Dispatching { candidate: IntentCandidate, target: DeviceId, request_id: RequestId },
    Executing { candidate: IntentCandidate, target: DeviceId, started_at: Instant },
    Succeeded { result: Value, elapsed: Duration },
    Failed { error: ErrorReport, elapsed: Duration },
    Cancelled { reason: CancelReason },
}
```

### 6.2 路由流程

```rust
impl Kernel {
    /// 唯一入口。所有跨设备动作都从这里进。
    pub async fn dispatch_intent(&self, source: IntentSource) -> Result<IntentId>;

    /// 查询意图状态（UI 用）
    pub fn intent(&self, id: IntentId) -> Option<IntentSnapshot>;

    /// 用户选择候选 / 确认 / 取消
    pub async fn resolve_intent(&self, id: IntentId, decision: IntentDecision) -> Result<()>;
}
```

内部流程（对应 [06](./05-system-architecture.md) §4 的第 ②–⑦ 步）：

```
1. 生成 IntentId + TraceContext
2. 规范化 source → ShareItem / ClipboardContent / …
3. 匹配阶段：
     - 查 Plugins 聚合根里"声明了 share 入口"的插件
     - 对每个插件：调用其 can_handle(shares) host function
         · 每次调用有 fuel + 时间上限（默认 50 ms，可配）
         · 超时/失败 → 记录并跳过（不影响其它插件）
     - 若有候选且总数 > 1 → 并发调用（受插件并发上限约束）
     - 收集候选 → IntentState::Matching → 到 deadline 或全部返回
4. 决策阶段（可配策略）：
     - Auto           → 若唯一候选且插件配置为自动
     - AskUser        → 发布事件 intent.candidates_ready，等 UI
     - Deny           → 无候选 → Failed(NoHandler)
5. 目标选择：
     - 若候选携带 suggested_target 且该设备在线 → 用它
     - 否则用策略（最近使用 / 用户选择 / 能力匹配）
6. capability 闸门（★ 关键）：
     - 校验候选的 required_caps ⊆ 该插件的 grants
     - 校验目标设备已安装该插件且版本兼容
     - 校验目标设备的 PlatformCapabilities 覆盖 required_caps
       （例如 iOS 不支持 simulate_keys → 提前拒绝，而不是远端失败）
7. 下发：
     - 构造 Envelope { event: "_plugin.{slug}.{action}", payload, request_id, trace_id }
     - Transfer/Transport 发送
     - 注册超时定时器
8. 回执：
     - 收到 _synra.intent.result（含 reply_request_id）
     - 匹配 IntentId → Succeeded/Failed → 发布事件
     - 超时 → Failed(Timeout)，保留 trace 便于诊断
9. 幂等：
     - request_id 作为去重键，重试时复用（不生成新的）
     - 接收端维护 request_id → result 缓存（TTL），重复请求直接返回缓存结果
```

### 6.3 关键设计点

| 点                        | 做法                                                                       | 解决什么                          |
| ------------------------- | -------------------------------------------------------------------------- | --------------------------------- |

---

## 7. 错误模型

### 7.1 分层错误类型

```rust
// crates/synra-protocol/src/error.rs —— 协议级错误（可跨设备传输）

#[derive(Debug, thiserror::Error, Serialize, Deserialize)]
#[serde(rename_all = "snake_case", tag = "code")]
#[non_exhaustive]
pub enum SynraError {
    #[error("device not found: {device_id}")]
    DeviceNotFound { device_id: DeviceId },

    #[error("device offline: {device_id}")]
    DeviceOffline { device_id: DeviceId },

    #[error("transport not open for {device_id}")]
    TransportNotOpen { device_id: DeviceId },

    #[error("transport handshake failed: {reason}")]
    HandshakeFailed { reason: String },

    #[error("frame too large: {len} > {max}")]
    FrameTooLarge { len: u32, max: u32 },

    #[error("capability denied: plugin {plugin_id} may not {capability}")]
    CapabilityDenied {
        plugin_id: PluginId,
        capability: String,
        declared: Vec<String>,
    },

    #[error("plugin not installed: {plugin_id}")]
    PluginNotInstalled { plugin_id: PluginId },

    #[error("plugin faulted: {plugin_id}: {reason}")]
    PluginFaulted { plugin_id: PluginId, reason: String },

    #[error("plugin exceeded quota: {plugin_id}: {quota}")]
    PluginQuotaExceeded { plugin_id: PluginId, quota: QuotaKind },

    #[error("no plugin can handle this intent")]
    NoHandler,

    #[error("transfer failed: {transfer_id}: {reason}")]
    TransferFailed { transfer_id: TransferId, reason: TransferReason },

    #[error("integrity check failed: {expected} != {actual}")]
    IntegrityMismatch { expected: String, actual: String },

    #[error("signature invalid for {plugin_id}")]
    SignatureInvalid { plugin_id: PluginId },

    #[error("protocol version unsupported: peer={peer}, local={local}")]
    ProtocolVersionUnsupported { peer: String, local: String },

    #[error("operation timed out after {elapsed_ms}ms")]
    Timeout { elapsed_ms: u64 },

    #[error("platform does not support {capability}")]
    Unsupported { capability: String },

    #[error("internal error: {message}")]
    Internal { message: String },
}
```

**错误码要求**：
- 每个 variant 有**稳定的数字码**（用于日志聚合与 UI 映射），定义在一个独立模块里；

### 7.2 内部分层

```rust
// 每一层有自己的错误，用 #[from] 向上转换
synra_net::Error        → 网络层（socket、帧、超时）
synra_store::Error      → 存储层（SQL、IO、迁移）
synra_plugin_host::Error → 插件层（wasm 实例化、闸门、配额）
        ↓
synra_core::Error       → 内核层（聚合根操作失败）
        ↓
apps 层错误             → 由 Tauri command 映射为可序列化响应
```

**规则**：
- 只有 `synra-protocol::SynraError` 可以**跨设备**传输；
- 内部错误可以携带 `source()` 链，但在边界（IPC / 网络）处必须**扁平化成 `SynraError`**；
- **不允许** `anyhow::Error` 出现在库 crate 的公开 API 里（只在 `apps/*` 的 `main` 与边界适配处使用）。

### 7.3 错误与用户可见性的映射

```rust
impl SynraError {
    pub fn user_visibility(&self) -> UserVisibility {
        match self {
            Self::TransportNotOpen { .. } | Self::HandshakeFailed { .. }
                if self.is_transient() => UserVisibility::Silent,
            Self::CapabilityDenied { .. }   => UserVisibility::Actionable,  // 提示用户去授权
            Self::PluginNotInstalled { .. } => UserVisibility::Actionable,
            Self::NoHandler                 => UserVisibility::Informational,
            _                               => UserVisibility::Error,
        }
    }
}
```

> Rust 版必须靠**类型分类**（`transient` / `actionable` / `informational`），而不是字符串。

---

## 8. 事件流（内核 → UI / 订阅者）

### 8.1 事件枚举

```rust
// crates/synra-core/src/events.rs

#[derive(Debug, Clone, Serialize)]
#[serde(rename_all = "snake_case", tag = "type", content = "payload")]
#[non_exhaustive]
pub enum KernelEvent {
    // —— 身份 ——
    IdentityChanged(IdentitySnapshot),

    // —— 发现 ——
    DiscoveryStarted { scan_id: ScanId },
    DiscoveryCandidateUpserted(DiscoveryCandidate),
    DiscoveryCandidateRemoved { device_id: DeviceId },
    DiscoveryFinished { scan_id: ScanId, found: usize },

    // —— 链路 ——
    DeviceCardUpdated(DeviceCard),         // UI 设备列表的唯一数据源
    LinkOpened { device_id: DeviceId, direction: Direction },
    LinkClosed { device_id: DeviceId, reason: CloseReason },
    LinkError { device_id: DeviceId, error: ErrorReport },

    // —— 配对 ——
    PairingRequested { from: DeviceId, initiator: PeerProfile },
    PairingAccepted { device_id: DeviceId },
    PairingRejected { device_id: DeviceId, reason: String },
    PairingCompleted { device_id: DeviceId },
    Unpaired { device_id: DeviceId, reason: UnpairReason },

    // —— 意图 ——
    IntentCreated(IntentSnapshot),
    IntentCandidatesReady { intent_id: IntentId, candidates: Vec<CandidateView> },
    IntentProgress { intent_id: IntentId, phase: IntentPhase },
    IntentFinished(IntentResult),

    // —— 传输 ——
    TransferProgress { transfer_id: TransferId, received: u64, total: u64 },
    TransferFinished { transfer_id: TransferId, ok: bool, reason: Option<String> },

    // —— 插件 ——
    PluginInstalled(PluginSummary),
    PluginInstallProgress { plugin_id: PluginId, stage: InstallStage },
    PluginInstallFailed { plugin_id: PluginId, error: ErrorReport },
    PluginActivated { plugin_id: PluginId },
    PluginDeactivated { plugin_id: PluginId, reason: DeactivateReason },
    PluginFaulted { plugin_id: PluginId, error: ErrorReport },
    PluginRemoved { plugin_id: PluginId },
    PluginPermissionRequested { plugin_id: PluginId, caps: Vec<Capability> },

    // —— 配置 ——
    ConfigChanged { key: ConfigKey },
}
```

### 8.2 订阅机制

```rust
impl Kernel {
    /// 订阅内核事件。返回的 receiver 有界；落后会收到 `Lagged`。
    pub fn subscribe(&self) -> broadcast::Receiver<KernelEvent>;

    /// 订阅状态快照变更（用于 UI 首屏与重连后的全量同步）
    pub fn snapshot(&self) -> KernelSnapshot;
}
```

**规则**：
1. UI **不轮询**；首屏拉一次 `snapshot()`，之后只消费事件；
2. 事件是**增量**，`snapshot()` 是**全量**；两者必须一致（用同一个内部视图构造，避免"事件与快照不一致"）；
3. 有界广播 + `Lagged` → UI 收到后**重新拉 `snapshot()`**（自愈）；
4. 事件必须 `Serialize`（给 IPC 用）且 `#[non_exhaustive]`（前端要处理未知事件类型，见 [10](./08-ipc-and-frontend.md) §3.4）。

### 8.3 为什么不用回调注册

```rust
kernel.on_device_found(|d| { ... });
```

问题：生命周期的管理责任在调用方；容易泄漏；顺序不确定。

**采用**：`watch`（状态）/ `broadcast`（事件）两个原语。UI 侧订阅是一次性的 `Receiver`，drop 即退订。

---

## 9. 内核构造与生命周期

```rust
// crates/synra-core/src/kernel.rs

pub struct KernelConfig {
    pub role: NodeRole,                 // Desktop | Mobile | Cli | Daemon
    pub platform: Arc<dyn PlatformCapability>,
    pub store: StoreConfig,             // 数据目录、SQLite 路径
    pub tuning: Tuning,                 // 端口、超时、并发（覆盖默认值）
    pub plugin_policy: PluginPolicy,    // 自动同步、默认权限、信任策略
}

pub struct Kernel { /* 各聚合根 */ }

impl Kernel {
    /// 构造内核：加载 store、恢复配对、但不启动网络
    pub async fn bootstrap(config: KernelConfig) -> Result<Arc<Self>>;

    /// 启动网络与服务（可在此后接入 UI）
    pub async fn start(self: &Arc<Self>) -> Result<()>;

    /// 优雅关停
    pub async fn shutdown(self: Arc<Self>, grace: Duration) -> ShutdownReport;
}
```

**启动顺序（为什么是这个顺序）**：

```
1. 初始化 tracing / metrics（最先，才能记录后续失败）               → synra-observe
2. 打开 store（SQLite + 迁移）                                     → synra-store
3. 加载/生成 Identity（device_id、display_name）                    → Identity
4. 加载 Trust（配对记录）                                          → Trust
5. 加载 Config                                                     → Config
6. 加载 Plugins（清单、启用状态、权限授予）                          → Plugins
7. 探测本机 IP 集合与网络接口                                       → Discovery
8. 启动 supervisor → 长驻任务：
     a. TCP accept 循环（先能收）
     b. UDP 发现监听（应答查询）
     c. mDNS 广告（若启用）
     d. 心跳 ticker（空转，等有链路）
     e. 重连 supervisor（按配对记录尝试恢复）
9. 发布 KernelEvent::IdentityChanged + 初始 DeviceCard 列表
   → UI 可以开始渲染（★ 与内核启动并行）
```

**关键**：第 8 步与"UI 首屏渲染"**并行**。UI 不需要等网络就绪。


---

## 10. 测试性设计

内核必须**可测**，因此：

| 设计点                     | 测试收益                                             |
| -------------------------- | ---------------------------------------------------- |
| `trait PlatformCapability` | 用 `MockPlatform` 断言"插件调用了 open_url"            |
| `trait Transport`          | 用 `MemoryTransport` 做双内核端到端测试（同进程、无网络） |
| `Clock` 抽象               | 用 `MockClock` 测超时/心跳，不需要真的 `sleep`         |
| `KernelConfig.tuning`      | 测试里把超时改小（1ms 而不是 6s），让测试快             |
| 事件流                     | 测试断言"产生了哪些事件"而不是"内部状态是什么"           |
| 纯函数内核                  | `synra-capability`、帧编解码、状态转移都是纯函数，易测   |

```rust
// crates/synra-testkit/src/lib.rs
pub struct KernelPair {
    pub a: Arc<Kernel>,
    pub b: Arc<Kernel>,
    /// 高级测试工具：把两边的传输对接，模拟"已配对"
    pub link: MemoryTransportLink,
}

impl KernelPair {
    /// 建立一个已配对的、链路 ready 的双内核环境
    pub async fn paired() -> Self;
    /// 推进虚拟时钟（触发心跳、超时）
    pub async fn advance(&self, d: Duration);
    /// 断言 A 向 B 发送了某事件
    pub async fn expect_event(&mut self, event: &str) -> Envelope;
}
```

见 [22 测试策略](./21-testing-strategy.md) §4。

---


| ---------------- | --------------------------------------------------- | --------------------------------------------- |
| 状态容器         | 30+ 个模块 + Vue ref + 4 个 Pinia store              | 8 个聚合根                                    |
| 状态权威         | 多处（main 与 renderer 各读一份配对文件）             | 单一（`Trust`）                                |
| 平台抽象         | 3 个 TS adapter，其中一个是桩                        | `trait PlatformCapability` + 5 个实现          |
| 链路状态         | 三层枚举，`handshaking` 从未赋值                     | 类型状态机 `Link<S>`                           |
| 任务管理         | 隐式（`setTimeout` / `setInterval` / promise 链）     | `Supervisor` + 稳定 TaskId                     |
| 背压             | 无                                                  | 每个队列有显式容量与溢出策略                    |
| 错误             | 裸字符串 + 关键词匹配抑制                            | 分层错误枚举 + 分类 + 稳定码                    |
| 事件             | 自定义 DOM 事件 + host event + 两套映射器             | 单一 `KernelEvent` 枚举                        |
| UI 同步          | 轮询 + 事件混合                                      | `snapshot()` + 增量事件（一致构造）             |
| 幂等             | 无                                                  | `request_id` 复用 + 接收端结果缓存              |
| trace            | 无（`traceId` 从不上线）                             | `trace_id` 随信封传播 + 全链路日志              |
| 意图路由         | 散落在 `register-plugin-inbound-transport` 的表驱动逻辑 | `Intent` 聚合根 + 显式状态机                  |
| headless         | 不可能                                              | `Kernel` + `HeadlessPlatform` 即可              |
