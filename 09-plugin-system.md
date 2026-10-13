# 09 · 插件系统

> 这是本套文档**最重要的一篇**：它定义插件系统的整体形态——为什么分两轨、信任模型、以及各部分的边界。
> 阅读顺序建议：本文 → [10 能力与沙箱](./10-capability-and-sandbox.md) → [12 Wasm 宿主](./12-wasm-host.md) → [14 生命周期](./14-plugin-lifecycle.md) → [15 SDK](./15-plugin-sdk.md) → [16 分发](./16-plugin-distribution.md)。

---

## 1. 内核先行约束（写第一行代码之前必须定稿）

插件系统会**反向决定内核的形态**。以下约束在动手前必须锁定，否则后期要重构内核：

| # | 约束                                                                     | 影响的内核设计                                        |
| - | ------------------------------------------------------------------------ | ----------------------------------------------------- |
| 1 | 插件**没有**宿主指针、没有 `Arc<Kernel>`、没有文件句柄                        | 所有能力必须经 host function 转发（[07 §3](./07-core-kernel.md)） |
| 2 | 插件调用**必须可中断**（超时 / 配额 / 用户停止）                              | host function 必须是 `async` 且可取消                   |
| 3 | 插件调用**必须可审计**（谁在何时调了什么、结果如何）                           | `gate.rs` 是唯一入口，且带 trace                        |
| 4 | 插件状态**不能**与内核状态共享内存                                           | 插件只能通过 `storage` 能力读写自己的命名空间            |
| 5 | 插件的**事件订阅必须可退订且随实例销毁**                                       | 事件总线要支持按"订阅者 id"批量撤销                      |
| 6 | 插件之间的交互**只能**经内核（event / action），不能直接互调                    | 内核需要一个跨插件的 action 注册表                       |
| 7 | UI 插件**只能**通过 IPC 访问内核能力，不能直接访问内核内存或文件                  | 需要为 UI 插件设计独立的、受限的 IPC 面（[10](./08-ipc-and-frontend.md) §7） |

> 约束 2 是最容易被忽略的：如果 host function 是同步阻塞的，一个插件的死循环就会挂住内核线程。
> 在 wasmtime 里有两种中断手段：**fuel**（确定性指令计数）与 **epoch**（基于时间的抢占）。两者都要用（[13](./12-wasm-host.md) §4）。

---

## 2. 三类插件与双轨运行时

### 2.1 分类

```
                        ┌─────────────────────────────────┐
                        │         Plugin Manifests        │
                        └───────────────┬─────────────────┘
                                        │
          ┌─────────────────────────────┼─────────────────────────────┐
          │                             │                             │
┌─────────▼──────────┐      ┌───────────▼──────────┐      ┌───────────▼──────────┐
│  ① Headless 插件    │      │  ② UI 插件            │      │  ③ Hybrid 插件        │
│                     │      │                      │      │                      │
│ artifact: *.wasm    │      │ artifact: *-ui.zip   │      │ 两者都有              │
│ 执行位置: 内核内     │      │ 执行位置: WebView     │      │ 两者独立加载/授权      │
│ 接口: WIT 组件      │      │ 接口: window.synra    │      │                      │
│ 能力: host funcs    │      │ 能力: 内核 IPC        │      │                      │
│ 隔离: Wasm 沙箱     │      │ 隔离: iframe/WebView  │      │                      │
│                     │      │                      │      │                      │
│ 用途:               │      │ 用途:                 │      │ 用途:                │
│  · 分享匹配          │      │  · 自定义页面         │      │  · 聊天（UI + 协议）  │
│  · 动作执行          │      │  · 设置界面           │      │  · 文件浏览器         │
│  · 快捷指令          │      │  · 复杂配置           │      │  · 面板 + 后台逻辑    │
│  · 协议解析          │      │  · 数据可视化         │      │                      │
│  · 定时任务          │      │                      │      │                      │
└────────────────────┘      └──────────────────────┘      └──────────────────────┘
```

### 2.2 为什么必须分类

一个常见的简化是"插件不声明它跑在哪，全由宿主决定"。这个简化建立在**所有插件都是同一种代码**这个前提上。一旦引入 Wasm，前提就不成立了：

| 维度             | Headless（Wasm）        | UI（ESM bundle）        |
| ---------------- | ----------------------- | ----------------------- |
| 能渲染界面        | ❌                      | ✅                      |
| 能访问 DOM        | ❌                      | ✅（在隔离上下文内）      |
| 内存上限可控      | ✅（线性内存硬上限）      | ❌（JS 堆）              |
| CPU 配额可控      | ✅（fuel / epoch）       | ❌                      |
| 可中断            | ✅                      | ⚠️ 只能销毁 WebView      |
| 执行位置          | 内核内（任意线程）         | WebView（UI 线程）        |
| 崩溃影响          | 隔离（trap → 实例销毁）   | 隔离（WebView 崩溃）      |
| 启动成本          | 低（预编译后可 ~0）       | 中（WebView 创建）        |
| 跨平台一致        | ✅ 完全一致              | ⚠️ WebView 引擎差异       |

**结论**：这不是"部署细节"，而是**插件的本质属性**。因此新版 manifest **必须**声明：

```jsonc
{
  "synra": {
    "runtime": {
      "headless": { "artifact": "dist/plugin.wasm", "wit": "synra:plugin@1.0.0" },
      "ui":       { "artifact": "dist/ui.zip", "entry": "index.html" }
    }
    // 至少提供一个；两个都有 = hybrid
  }
}
```

### 2.3 划分原则：宿主决定"怎么跑"，插件声明"能不能跑"

插件声明**形态**（headless / ui），但**不声明**：

- 跑在哪个线程 / 哪个 wasm 实例（宿主决定，用实例池）；
- 用多少内存（宿主按插件信任等级分配）；
- 是否预热（宿主决定）；
- 部署在哪个平台（一份 artifact 全平台跑）。

即：

| 决策           | 谁决定 | 类比                                       |
| -------------- | ------ | ------------------------------------------ |
| 插件有 UI 吗    | 插件   | 这是产品设计，不是部署                     |
| 逻辑在哪执行     | 宿主   | 引擎、线程、实例池                          |
| 资源配额        | 宿主   | 按信任等级 / 用户设置                       |
| 是否隔离        | 总是   | 没有"不隔离"这个选项                        |

---

## 3. 信任模型

### 3.1 信任等级

```rust
// crates/synra-plugin-host/src/trust.rs

#[derive(Debug, Clone, Copy, PartialEq, Eq, PartialOrd, Ord, Serialize, Deserialize)]
#[serde(rename_all = "snake_case")]
pub enum PluginTrust {
    /// 未签名 / 签名无效 → 默认不安装
    Untrusted = 0,
    /// 签名有效但来源不在白名单（用户已确认安装）
    SignedUnknownPublisher = 1,
    /// 签名有效且发布者在用户信任列表
    SignedTrustedPublisher = 2,
    /// 官方 / 内置插件
    Builtin = 3,
}
```

### 3.2 信任等级决定什么

| 维度           | Untrusted | SignedUnknown | SignedTrusted | Builtin |
| -------------- | --------- | ------------- | ------------- | ------- |
| 能否安装        | ❌        | ✅（需确认）   | ✅            | ✅      |
| 默认授权能力    | —         | 仅 `log` / `ui` | 按 manifest 提议，用户确认 | 全部按 manifest |
| 自动更新        | —         | ❌            | ✅            | ✅      |
| 内存上限        | —         | 64 MiB        | 256 MiB       | 1 GiB   |
| 单次调用 fuel   | —         | 中等          | 高            | 无限制  |
| 网络能力默认    | —         | ❌            | 询问          | ✅      |
| 文件系统能力    | —         | ❌            | 询问          | ✅      |

### 3.3 授权不是一个布尔值

```rust
/// 用户对某个插件授予的能力集合（持久化）
#[derive(Debug, Clone, Serialize, Deserialize)]
pub struct Grant {
    pub plugin_id: PluginId,
    pub plugin_version: Semver,
    pub granted: CapabilitySet,
    /// 用户拒绝过的能力（不再反复询问）
    pub denied: CapabilitySet,
    pub granted_at: SystemTime,
    /// 授权指纹：插件升级后 capability 变了是否要重新确认
    pub manifest_fingerprint: Sha256Digest,
}
```

**规则**：
- 插件升级后如果 **capability 集合变了** → 重新询问（`manifest_fingerprint` 不匹配）；
- 如果只是代码变了、capability 不变 → 沿用授权（避免骚扰）；
- 用户可以在设置里查看"每个插件被授予了什么、实际用过什么"（审计，[18](./18-observability.md) §5）。

---

## 4. 插件与内核的接口（不使用闭包）

接口形态必须是**函数调用**，而不能是"传一个对象给插件"。原因：跨沙箱无法传递闭包/对象引用——Wasm 与宿主是两个地址空间，只能交换值。

```
插件（Wasm 组件）                    内核（宿主）
─────────────────                    ─────────────
synra:plugin/host.log          ──▶   Logger::log  （闸门：log:*）
synra:plugin/host.events.emit  ──▶   EventBus::emit（闸门：event:emit:*）
synra:plugin/host.device.send  ──▶   IntentRouter  （闸门：device:send[:target]）
synra:plugin/host.storage.get  ──▶   PluginStore   （闸门：storage:local:read）
synra:plugin/host.transfer.*   ──▶   TransferSessions（闸门：file:transfer:*）
synra:plugin/host.platform.*   ──▶   PlatformCapability（闸门：platform:open-url 等）
synra:plugin/host.env.*        ──▶   只读（无闸门）
```

```wit
// wit/synra-plugin.wit （节选）
package synra:plugin@1.0.0;

interface types {
    enum log-level { trace, debug, info, warn, error }
    record device-target { id: string, name: string, kind: device-kind }
    enum device-kind { desktop, mobile, headless }
    record envelope { event: string, target: string, payload: string }
    variant outcome { ok(string), err(error) }
    record error { code: string, message: string, details: option<string> }
}

interface host {
    use types.{log-level, device-target, envelope, outcome};

    // —— 无闸门（只读环境信息）——
    env-sdk-version: func() -> string;
    env-host-kind: func() -> string;
    env-locale: func() -> string;
    env-dark-mode: func() -> bool;
    env-can: func(capability: string) -> bool;

    // —— log:* ——
    log: func(level: log-level, message: string, ctx: option<string>);

    // —— event:emit:* / event:subscribe:* ——
    event-emit: func(event: string, payload: string) -> outcome;
    event-subscribe: func(pattern: string) -> outcome;   // 回调经 guest export

    // —— device:list / device:send[:target] / device:broadcast ——
    device-list: func() -> list<device-target>;
    device-send: func(target: string, event: string, payload: string) -> outcome;
    device-broadcast: func(event: string, payload: string) -> outcome;

    // —— storage:local:* / storage:secure:* ——
    storage-get: func(key: string) -> outcome;
    storage-set: func(key: string, value: string) -> outcome;
    storage-delete: func(key: string) -> outcome;
    storage-keys: func(prefix: option<string>) -> list<string>;
    storage-quota: func() -> tuple<u64, u64>;

    // —— network:fetch:<host>[:<method>] ——
    network-fetch: func(request-json: string) -> outcome;   // 宿主代理，返回 JSON
    network-websocket: func(url: string) -> outcome;

    // —— fs:* ——
    fs-read: func(path: string) -> outcome;
    fs-write: func(path: string, data: list<u8>) -> outcome;
    fs-list: func(path: string) -> outcome;
    fs-stat: func(path: string) -> outcome;

    // —— file:transfer:* ——
    transfer-send: func(kind: string, target: string, meta-json: string) -> outcome;
    transfer-status: func(transfer-id: string) -> outcome;

    // —— platform:*（宿主系统能力）——
    platform-open-url: func(url: string) -> outcome;
    platform-open-path: func(path: string) -> outcome;
    platform-reveal-path: func(path: string) -> outcome;
    platform-simulate-keys: func(chord: string) -> outcome;
    platform-notify: func(title: string, body: string) -> outcome;
    platform-read-clipboard: func() -> outcome;
    platform-write-clipboard: func(text: string) -> outcome;
    platform-capabilities: func() -> list<string>;

    // —— action:*（跨插件动作）——
    action-declare: func(name: string, schema-json: string) -> outcome;
    action-invoke: func(target-plugin: string, name: string, args-json: string) -> outcome;
    action-list: func() -> string;   // JSON 数组
}

world plugin {
    import host;

    // guest 导出：插件实现的生命周期与入口
    export lifecycle;
    export share;
    export actions;
}

interface lifecycle {
    use types.{outcome};
    /// 插件被激活（首次或重新启用）
    activate: func(config-json: string) -> outcome;
    /// 插件被停用，必须释放所有资源
    deactivate: func();
    /// 内核要求插件自我检查（版本迁移等）
    migrate: func(from-version: string, to-version: string) -> outcome;
    /// 定时器 / 异步任务回调
    on-tick: func(tick-json: string) -> outcome;
}

interface share {
    use types.{outcome};
    /// ★ 匹配：判断能否处理一个分享项（必须无副作用）
    can-handle: func(item-json: string) -> string;   // JSON: {"handled":false} | {"handled":true,"action":...,"presentation":...}
    /// 执行一个动作
    execute: func(action: string, args-json: string) -> outcome;
}
```

**关键设计点**：

1. **所有跨边界数据是字符串（JSON）**，而不是复杂类型。理由：WIT 的复杂类型演进会破坏兼容；JSON 在两侧都易于扩展。
   - ⚠️ 代价：序列化开销。**但**capability 闸门与审计需要结构化数据，所以**内核侧会解析**这些 JSON。
2. **`outcome` 是一等类型**，不是抛异常。Wasm 里没有异常，trap 只用于"不可恢复的 bug"。
3. **`env-*` 无闸门**，其余**全部**经闸门。
4. **`event-subscribe` 采用 guest export 回调**（WIT 的 resource/回调），内核持有 guest 的回调表。
5. **`can-handle` 返回 JSON 字符串而不是 `outcome`**：因为"不处理"不是错误。

> **为什么是"订阅 + 拉取"而不是"响应式对象"**：Wasm 里没有 Vue，也没有任何"宿主可观测对象"能跨沙箱共享。
> - 插件订阅 `device.*` 事件（内核推送变更）；
> - 插件需要时调 `device-list()` 拉当前快照。

---

## 5. UI 插件的隔离方案

UI 插件跑在 WebView 里，**天然比 Wasm 弱**。分层处理：

### 5.1 三层隔离（按平台能力退化）

| 层级 | 手段                                   | 隔离强度 | 可用平台                          |
| ---- | -------------------------------------- | -------- | --------------------------------- |
| L1   | **独立 WebView + 独立 origin**          | 最强     | 桌面（Tauri 多 WebView）、WebView2 / WKWebView |
| L2   | **sandbox iframe + `srcdoc` + 独立 origin** | 强   | 所有（含 Linux WebKitGTK）          |
| L3   | **sandbox iframe + 同 origin**          | 弱       | 兜底（WebKitGTK 老版本）            |

```rust
pub enum UiIsolation {
    /// 独立 WebView，独立 origin（如 synra-plugin://<plugin-id>/）
    SeparateWebview { origin: String },
    /// sandbox iframe，宽松 CSP
    SandboxedIframe { csp: Csp },
    /// 降级：同 origin iframe（仅当平台不支持前两者）
    SameOriginIframe,
}

impl UiIsolation {
    /// 按平台选择最强可用方案
    pub fn best_for(platform: PlatformKind, webview_kind: WebviewKind) -> Self;
}
```

### 5.2 UI 插件的能力面（`window.synra`）

UI 插件**不直接**访问内核。Tauri 的 IPC 面（`invoke` / `event`）对插件是**独立且受限的**：

```ts
// 注入到插件 WebView 的全局对象 window.synra（类型见 @synra/client，[23](./23-client-library.md)）
declare const synra: {
  readonly pluginId: string
  readonly version: string
  readonly capabilities: readonly string[]      // 该插件已被授予的能力（只读）

  // ── 无闸门 ──
  env(): Promise<EnvInfo>
  can(capability: string): boolean

  // ── 受闸门约束 ──
  log(level: LogLevel, message: string, ctx?: unknown): void
  event: { emit(event: string, payload: unknown): Promise<void>; subscribe(pattern: string, handler: (p: unknown) => void): () => void }
  device: { list(): Promise<DeviceTarget[]>; send(target: string, event: string, payload: unknown): Promise<void> }
  storage: { get<T>(key: string): Promise<T | undefined>; set(key: string, value: unknown): Promise<void>; ... }
  network: { fetch(input: string, init?: RequestInit): Promise<Response> }
  clipboard: { read(): Promise<string>; write(text: string): Promise<void> }
  platform: { openUrl(url: string): Promise<void>; ... }
  transfer: { send(file: File | Blob, target: string): Promise<TransferId>; ... }

  // ── 仅宿主 UI 可用（插件不可见，防止越权）──
  // (不存在)
}
```

**关键**：

- 注入的 `synra` 对象是**冻结的**（`Object.freeze`），且**不暴露任何内核代理对象**；
- 每个方法都是一次 `invoke`，携带**内核签发的会话令牌**（[12](./10-capability-and-sandbox.md) §6）；
- **不注入任何"共享对象"**：同 realm 的对象可以被反射探查，等于把内核结构暴露给插件。

### 5.3 UI 插件与 headless 插件的通信

一个 hybrid 插件的 UI 部分要与自己的 headless 部分对话：

```
[UI 插件 WebView]                            [内核 Wasm 沙箱]
   synra.self.invoke(action, args)        ──▶  闸门：校验 UI 插件的会话令牌
                                                校验 action 属于本插件的 headless 部分
                                                按 headless 的 grant 再过一次闸门
                                                调用 Wasm 组件的 execute()
                     ◀── 结果 / 事件推送 ──
```

即 UI 部分要走**自己 headless 的 action**，而不是"什么都能直接调"：
能力面由 manifest 划分（[13 §7.3](./13-plugin-authoring.md)），
而 `synra.self.invoke` 是 UI 拿到高危能力的**唯一**途径（[13 §5.2](./13-plugin-authoring.md)）。

### 5.4 关于 Linux WebKitGTK 的诚实说明

Linux 桌面（WebKitGTK）的 iframe 隔离弱于 Chromium。缓解措施：

- 强制 `sandbox` 属性 + 严格 CSP（`default-src 'none'; script-src 'self' 'unsafe-inline'`）；
- 禁用 `allow-same-origin`（除非必须）；
- 不给插件访问 `localStorage` 的能力（改用内核 storage 能力）；
- 明确**在文档中告知**：Linux 上 UI 插件隔离强度低于 Windows/macOS，建议只安装信任来源的 UI 插件。

> **诚实优于承诺**：不要在文档里说"所有平台都有强隔离"。

---

## 6. 插件的资源管理

### 6.1 实例池

```rust
// crates/synra-plugin-host/src/pool.rs

pub struct InstancePool {
    engine: Arc<Engine>,
    /// 每插件一个池
    pools: DashMap<PluginId, PluginPool>,
    config: PoolConfig,
}

pub struct PoolConfig {
    /// 每插件最大并发实例
    pub max_instances: usize,          // 默认 4
    /// 空闲多久回收
    pub idle_ttl: Duration,            // 默认 5 min
    /// 是否预热（激活时预创建 1 个）
    pub warmup: bool,                  // 默认 true
}

/// 一次调用的"租借"
pub struct InstanceLease {
    instance: PooledInstance,
    _permit: OwnedSemaphorePermit,
}

impl InstanceLease {
    /// 在实例上执行一次导出函数调用，带 fuel 与超时
    pub async fn call<T: ComponentExport>(
        &mut self,
        args: T::Args,
        budget: CallBudget,
    ) -> Result<T::Result, HostError>;
}

pub struct CallBudget {
    /// 指令数上限（确定性）
    pub fuel: u64,
    /// 墙钟超时（非确定性，用于兜底）
    pub timeout: Duration,
    /// 允许的最大内存增长
    pub max_memory_growth: u64,
}
```

**要点**：

- **实例复用**：Wasm 实例的创建有成本（实例化 + 可能的数据段初始化）。池化让"调用一次 `can_handle`"的成本从毫秒级降到微秒级。
- **但实例有状态**：插件可能在实例内存里存了状态。因此**必须**：
  - 要么保证插件的所有状态都在 `storage` 能力里（推荐，写进 SDK 约定）；
  - 要么在复用前调用 `deactivate`/`activate`（代价高）；
  - 要么标记"有状态插件"不允许池化（manifest 可声明 `stateful: true`）。
- **`fuel` 与 `timeout` 都要有**：fuel 保证确定性（同输入同消耗），timeout 兜底（防止 host function 阻塞导致 fuel 不减少）。

### 6.2 配额

```rust
pub struct PluginQuota {
    pub max_memory_bytes: u64,          // 线性内存上限
    pub max_table_elements: u32,
    pub max_instances: u32,
    pub max_open_transfers: u32,
    pub max_storage_bytes: u64,
    pub max_events_per_second: u32,     // 事件发射速率限制
    pub max_host_calls_per_second: u32, // 防止插件高频调用内核
    pub call_fuel: u64,
    pub call_timeout: Duration,
}
```

**配额不是"防恶意"的银弹**（Wasm 沙箱已阻止最坏情况），它的作用是：

1. 防止**有 bug 的插件**耗尽资源影响其它插件；
2. 给用户一个可理解的"这个插件用太多了"的反馈；
3. 在 UI 上展示插件资源使用（透明度）。

违规处理：

```
配额超限 → 本次调用失败（PluginQuotaExceeded）
         → 记入插件健康度
         → 连续 N 次超限 → 插件进入 Faulted 状态（[14](./14-plugin-lifecycle.md) §6）
         → 通知用户
```

---

## 7. 插件间交互

### 7.1 两种机制

| 机制         | 语义               | 适用                              | 闸门要求                     |
| ------------ | ------------------ | --------------------------------- | ---------------------------- |
| **Event**    | 广播，发布/订阅     | 松耦合通知（"用户登录了"）          | `event:emit:<pattern>` / `event:subscribe:<pattern>` |
| **Action**   | 定向调用，请求/响应  | 强耦合能力复用（"帮我格式化日期"）   | `action:declare:<name>` / `action:invoke:<plugin>:<name>` |

### 7.2 Action 注册表

```rust
pub struct ActionRegistry {
    actions: RwLock<HashMap<(PluginId, ActionName), ActionDecl>>,
}

pub struct ActionDecl {
    pub plugin_id: PluginId,
    pub name: ActionName,
    pub schema: ActionSchema,       // JSON Schema（args / returns）
    pub since_version: Semver,
    pub deprecated: bool,
}
```

**规则**：

1. 插件必须在 manifest 里**声明**它能提供的 action（不能只靠运行时 `action-declare`）；
2. 运行时 `action-declare` 只是**注册具体实现**，且名字必须在 manifest 声明集合内；
3. 调用方必须声明 `action:invoke:<plugin>:<name>`。允许通配：`action:invoke:*:format-date`；
4. **跨插件调用有超时与配额**：防止链式调用无限扩散（A → B → C → A）；
5. **调用链深度上限**（默认 8），超过则拒绝并报 `ActionChainTooDeep`。

### 7.3 Compose（依赖）

`compose` 被实现为**依赖图**（而不是一个简单的 id 列表）：

```rust
pub struct PluginDependencyGraph {
    /// plugin_id → 它依赖的插件
    deps: HashMap<PluginId, Vec<PluginId>>,
}

impl PluginDependencyGraph {
    /// 启用某个插件时，先按拓扑序启用依赖
    pub fn activation_order(&self, target: PluginId) -> Result<Vec<PluginId>>;
    /// 检测环（manifest 校验阶段）
    pub fn detect_cycles(&self) -> Vec<Vec<PluginId>>;
    /// 某个插件被禁用时，哪些插件需要级联禁用
    pub fn dependents(&self, target: PluginId) -> Vec<PluginId>;
}
```

**级联的语义**：
- 级联执行有**取消窗口**（操作后短暂时间内可撤销）——实现为"待执行的级联任务"而不是立即执行；
- 环检测在**安装期**就做（拒绝安装会形成环的插件），而不是启用期。

---

## 8. 插件系统的模块结构

```text
crates/synra-plugin-host/
├─ lib.rs
├─ trust.rs          # 信任等级与授权（Grant）
├─ manifest.rs       # manifest 解析与校验（含依赖图、capability 语法）
├─ engine.rs         # wasmtime Engine 配置（fuel/epoch/limits/编译策略）
├─ backend/
│  ├─ mod.rs         # trait WasmBackend（wasmtime / wasmi）
│  ├─ wasmtime.rs
│  └─ wasmi.rs
├─ instance.rs       # 单实例封装（Store、内存、状态）
├─ pool.rs           # 实例池与租借
├─ gate.rs           # ★ 能力闸门（唯一入口，带审计与配额）
├─ host_funcs/
│  ├─ mod.rs         # 统一的注册与宏
│  ├─ env.rs
│  ├─ log.rs
│  ├─ event.rs
│  ├─ device.rs
│  ├─ storage.rs
│  ├─ network.rs
│  ├─ fs.rs
│  ├─ transfer.rs
│  ├─ platform.rs
│  └─ action.rs
├─ lifecycle.rs      # 安装/激活/停用/卸载状态机
├─ registry.rs       # 已安装插件索引（内存 + store）
├─ verify.rs         # 签名、WIT 版本、体积、禁用导入校验
├─ health.rs         # 插件健康度（超限、崩溃、错误率）
└─ error.rs
```

---

## 9. 与其他部分的关系

| 想深入什么                       | 去哪                                                     |
| -------------------------------- | -------------------------------------------------------- |
| capability 语法、三层闸门、配额   | [10](./10-capability-and-sandbox.md)                      |
| Wasm 宿主：引擎配置、fuel/epoch、实例池 | [12](./12-wasm-host.md)                            |
| **作者视角**：写什么代码、manifest、构建、运行期谁调谁 | [13](./13-plugin-authoring.md)      |
| 安装 / 校验 / 激活 / 卸载的状态机  | [14](./14-plugin-lifecycle.md)                            |
| SDK（Rust 侧）与 CLI             | [15](./15-plugin-sdk.md)                                  |
| 签名、registry、跨设备同步         | [16](./16-plugin-distribution.md)                         |
| UI 插件的类型库 `@synra/client`   | [23](./23-client-library.md)                              |
| UI 插件的 IPC、CSP、会话令牌       | [08 §7](./08-ipc-and-frontend.md)                          |

---

## 10. 待决问题（需要在实现前拍板）

| # | 问题                                                            | 选项                                                | 建议                    |
| - | --------------------------------------------------------------- | --------------------------------------------------- | ----------------------- |
| 1 | 移动端是否内嵌 wasmtime                                          | 内嵌（pulley/winch） / wasmi / 不支持 headless 插件   | 内嵌 + feature 可切 wasmi（ADR-005） |
| 2 | Wasm 组件模型（WIT）vs 传统 core module + WASI                     | 组件模型 / core module                              | **组件模型**（类型安全、接口演进） |
| 3 | 跨边界用 JSON 还是 WIT 结构化类型                                  | JSON / 结构化 / 混合                                 | **混合**：控制面结构化，payload JSON |
| 4 | UI 插件是否允许直接使用 Vue                                      | 允许（注入 Vue 运行时） / 只允许原生 DOM                 | **允许**（生态与开发效率），但 Vue 实例由插件自己管理，不与宿主共享 |
| 5 | headless 插件是否支持网络直接访问（WASI socket）                   | 支持 / 只经内核代理                                   | **只经内核代理**（可审计、可限流） |
| 6 | 插件能否声明定时任务                                             | 支持（内核调度） / 不支持                             | **支持**（`on-tick`），但配额严格 |
| 7 | 插件状态是否持久化（Memento）                                     | 自动快照 / 插件自己用 storage                          | **插件自己用 storage**（简单、可测） |
