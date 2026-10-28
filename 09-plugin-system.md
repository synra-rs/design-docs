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

## 2. 两种插件形态与双轨运行时

### 2.1 分类

| 形态 | 产物 | 执行位置 |
| --- | --- | --- |
| Headless | `plugin.wasm` | 内核的 Wasm 沙箱 |
| Hybrid | `plugin.wasm` 与 `ui.zip` | 逻辑在 Wasm，界面在隔离 WebView |

只支持 headless 和 hybrid 两种插件形态；UI 是 hybrid 的界面部分，不是可独立安装的第三类插件。
每个插件都必须包含 headless 组件，即使业务逻辑很少。UI 只能经受限 IPC 调用内核，
不能将平台动作等 headless 能力搬入 WebView。此约定与 [13 §1](./13-plugin-authoring.md) 和 [22 Q-01](./22-risks.md) 一致。

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
    // headless 必需；ui 可选；两个都有 = hybrid
  }
}
```

### 2.3 划分原则：宿主决定"怎么跑"，插件声明"能不能跑"

插件声明**形态**（headless / hybrid），但**不声明**：

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

    // —— event:emit:*；接收由 manifest.handles 声明 ——
    event-emit: func(event: string, payload: string) -> outcome;

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
    // ★ **已修复：`can-handle` 的答案现在可以带 `args`。**
    //
    // 契约没有改（`execute` 的签名不变），而是把传递补上了：
    // `Proposal`（代码里 `can-handle` 的答案类型）新增了 `args: serde_json::Value`（`#[serde(default)]`，
    // 所以不发这个字段的旧插件行为不变），`into_candidate` 把它原样传给 `PlannedAction.args`，
    // 而 `execute` 把它作为 `args-json` 发出去。
    //
    // 为什么由**插件**而不是孿主填：只有插件知道自己的动作需要什么（打开链接要 URL，
    // 加到阅读列表要 URL + 标题）；孿主代填就是在猜每个插件的接口。
    //
    // 为什么这不是绕过闸门：args 回到的是**写它的那个插件**，而闸门审的是能力不是参数；
    // 插件无法通过对自己撒谎来提权，而它拷贝的那份 item 本来就是孿主给的。
    //
    // ★ **而且这个缺陷能藏一轮，是因为配置看不见自己的输入。**
    // `Work` 的 `Display` 是 `interface.function`（`share.execute`），而 `ScriptedBackend` 用 `work.to_string()` 当索引键——
    // 于是**每一次 `execute` 都落在同一个键上**，配置无法区分参数，也无法发现 `null`。
    // 已改为 `key_for(work)`（`engine/scripted.rs`），把 payload 纳入键；现在参数对不上的调用会落到 fallback，而不是惄惄匹配。
    // ★ 教训：**一个看不见自己输入的配置，是一个断言不了输入的测试。**
}
```

**关键设计点**：

1. **所有跨边界数据是字符串（JSON）**，而不是复杂类型。理由：WIT 的复杂类型演进会破坏兼容；JSON 在两侧都易于扩展。
   - ⚠️ 代价：序列化开销。**但**capability 闸门与审计需要结构化数据，所以**内核侧会解析**这些 JSON。
2. **`outcome` 是一等类型**，不是抛异常。Wasm 里没有异常，trap 只用于"不可恢复的 bug"。
3. **`env-*` 无闸门**，其余**全部**经闸门。
4. **headless 使用 manifest `handles` 与必需的 `lifecycle.on-event(event-json) -> outcome`**。
   > 2026-10-27 决策：移除没有稳定实例归属的 `event-subscribe` WIT import，
   > 由宿主选择当前实例池中的实例投递；接收仍需具体事件的 `event:subscribe:<name>` 授权。
   > 本地 `event-emit` 成功表示入队，处理结果由有界诊断记录；不自动回复、不重试。
   > UI 的 JS 事件流订阅保持独立，详见 [13 §10-5](./13-plugin-authoring.md)。
5. **`can-handle` 返回 JSON 字符串而不是 `outcome`**：因为"不处理"不是错误。
   > 匹配调用经 `Work::allows_side_effects`、StoreData 和共享 HostCalls 进入纯上下文，
   > 持有授权的插件也不能在匹配阶段执行副作用；真实 Rust guest 已覆盖拒绝与正常执行，
   > 证据见 [作者链路记录](./history/2026-10-26-rust-authoring.md)。

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
    /// 独立 WebView，独立 origin（完整插件 id 的摘要域名，见 08 §7.3）
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

### 5.5 设计令牌：插件界面怎样看起来还是同一个产品

隔离是**安全**属性，视觉一致是**表现**属性。§5.1–§5.4 只讲了前者，结果是"插件界面长什么样"无人负责，
最后必然是每个插件一套视觉语言。

#### 为什么唯一能共享的只有值

插件界面跑在**自己的 WebView，或独立 origin 的 sandbox iframe** 里。这条边界挡住了插件读宿主状态，
同时也意味着：

- 宿主的**组件实例无法共享**。Vue 组件、响应式对象、antd 的 `ConfigProvider` —— 没有一个能跨 realm。
- 宿主的 **antd 插件够不着**。所以"要不要给插件 antd"不是一个可选项：插件只能**自己 bundle 一份**，
  而宿主既拦不住也不该拦。

能跨边界的只有**数据**。CSS 自定义属性是平台本来就有的机制 —— 属性设在文档根上，嵌套文档的 `var()` 就能解析到。

**所以契约是：插件拥有它的结构，宿主拥有它的值。**

```
宿主 WebView
  └── 插件 WebView / iframe          （自己的 origin、自己的 realm）
        ├── 插件自己的样式            —— 结构由它全权决定
        └── var(--synra-color-primary) —— 一个它没有选择的取值
```

#### 约定

| 项 | 决定 |
| -- | ---- |
| 前缀 | `--synra-` |
| 命名 | **与 antd v5 的 token 名一一对应** —— 作者查 antd 文档就能对上，不学第二套词 |
| 注入范围 | 品牌与语义色、背景与容器、文字、描边、字体与字号、间距、圆角、阴影、动效、深浅模式与密度 |
| manifest 是否声明用了哪些 | **不声明**。令牌是 API：稳定，改一个就是破坏性变更并 bump 版本，不是逐个插件协商的东西 |
| 声明式 UI（JSON Schema → 宿主渲染） | **不做**。它表达不了聊天、文件浏览器、图表，而为这些开特例的 schema 比它限制掉的自由更糟 |

> ★ **2026-10-16 更正："与 antd v5 的 token 名一一对应"这句已经不成立，而它的**意图**用另一种方式保留。**
>
> **发生了什么**：本节写下时假设宿主用 antd；后来 [28](./28-app-ui.md) 把宿主的视觉语言整个换掉了
> （UnoCSS + 一套语义命名的 token），而**组件库这一层直到 2026-10-16 才定下来用 antd**。
> 于是两份文档之间出现了一个真实矛盾：本节按 antd 的键名立契约，而实现里的名字是
> `--synra-screen` / `--synra-ink` / `--synra-primary-ink` / `--synra-line-strong` 这类**语义名**
> —— 它们已经发布（`@synra/theme@0.0.8`），而且每个对比度都是**算出来的**（[28 §6 R24](./28-app-ui.md)）。
>
> **结论：命名以 `@synra/theme` 为准，而"antd 的名字"由适配器提供。**
> `@synra/theme/antd`（[28 §2 R52](./28-app-ui.md)）把我们的 token 映射成 antd v5 的 theme token，
> **并导出这张映射表**。于是本节那句"作者查 antd 文档就能对上，不学第二套词"**照旧成立**：
> 用 antd 写插件的作者拿到的就是 antd 的键名，只是它们的**取值**来自同一份真相源。
>
> **一串连带更正**：
>
> | 本节原文 | 现在 |
> | -------- | ---- |
> | "命名与 antd v5 一一对应" | 语义命名（`--synra-*`），对应关系由适配器导出的映射表给出 |
> | "主题只有一个来源，产出：宿主自己的 antd `ConfigProvider` 主题、注入插件上下文的 `:root` 块、`synra.theme.current()`" | **仍然成立**，只是第一样的产出者明确为 `@synra/theme/antd` —— 这条现在是三个消费者共享一份 token 的另一种说法（[28 §1.1](./28-app-ui.md)） |
> | "antd 的 `boxShadow` 是逗号分隔的多层值" | 仍然成立，但**我们的浮起层不靠阴影**（[28 R53](./28-app-ui.md)）：适配器把 antd 的阴影关掉，浮起由玻璃表达 |
> | "完整变量清单见 `synra-core` 的 `NOTES/2026-02-ui-design-tokens.md`" | **那份清单是 antd 命名时代的产物**，需要与实现对齐（已记在 [28 §16.1](./28-app-ui.md) 的清单里） |
>
> **没有变的一点**：插件**仍然拿不到宿主的 antd**。边界是进程/realm 级的，本节上面那段论证不受影响 ——
> 变的只是"两边怎么长得像"：以前靠相同的键名，现在靠同一份 token 加一张映射表。

完整变量清单见 `github.com/synra-rs/synra-core` 的 `NOTES/2026-02-ui-design-tokens.md`。

#### 代价，如实写

| | |
| -- | -- |
| **解决** | 颜色、圆角、间距、字号、深浅 —— 宿主改一处，所有插件跟着动 |
| **不解决** | 插件用 antd 5.20、宿主用 5.24 时组件内部仍有细微差异。组件内部是插件的​事，令牌层够不到 |
| **接受** | 插件可以完全不理会令牌。令牌让一致变**容易**，不是变**强制**。用户看到的偏差是插件的选择，而那确实是它的选择 |

#### 三个容易做错的地方

**1. 阴影必须拆层，不能整体拷贝。**
antd 的 `boxShadow` 是逗号分隔的多层值；CSS 变量的值里出现逗号会在 `var()` 展开时被当成分隔符。
要么每层一个变量，要么用单层近似。

**2. 令牌变了 ≠ 界面重绘。**
antd v5 的样式在 JavaScript 里生成并缓存，改自定义属性**不会**让已渲染组件重算派生样式。
所以注入是**两个通道**，第二个不是可选的：

```ts
// 1. 值：写进插件文档根的 CSS 自定义属性
// 2. 通知：让插件重建它缓存的东西
synra.theme.subscribe((tokens) => { /* 重建 ConfigProvider */ })
synra.theme.current()          // 首屏取值，避免竞态
```

**3. 属性往哪里写，取决于隔离层级。**
L1（独立 WebView）由宿主直接写进它加载的文档；L2/L3 的 iframe **有自己的文档树、不继承父级**，
需要宿主注入 `<style>` 或由插件在启动时套用宿主交给它的令牌对象 —— 这两个层级**连初始值都依赖第二个通道**。

#### 对宿主代码的含义

主题必须只有**一个**来源，由它产出三样东西：宿主自己的 antd `ConfigProvider` 主题、注入插件上下文的 `:root` 块、
以及 `synra.theme.current()` 返回的对象。两个来源必然漂移，而漂移正是这一节要消灭的东西。

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


> ★ **存储有表了，而之前它一个字节都存不了。**
>
> 6 月份就写在这里的 `storage-get/set/delete/keys/quota` 五个函数，
> 能力目录里的 `storage:local:read` / `storage:local:write`，一直都在；
> 而 **仓库里每一个宿主对这五个都回 `notImplemented`**，存储层也没有表。
> 于是一个想记住一个设置的插件，唯一的办法是**每次运行都重新问用户**。
>
> * `plugin_storage(plugin_id, key, value, updated_ms)`，主键是 `(plugin_id, key)` —— 两个插件都叫
>   `settings` 时是两份设置，共享键空间会让装一个无关插件改变另一个读到什么。
> * `ON DELETE CASCADE`，因为行上面那一行没了之后，没有任何查询能把残留值归给任何东西。
> * ★ **计算的是值的字节数**（`LENGTH(CAST(value AS BLOB))`）而不是行或页的大小：
>   后者会让同一个插件在 SQLite 升级后因为看不到的原因超配额。
> * ★ **而前缀是前缀，不是模式**：`LIKE` 会把 `%` 和 `_` 当通配符，而插件的键可以含它们。
>   已转义（`ESCAPE '\'`）。阴性对照：改回拼接，`a_prefix_containing_a_wildcard_is_a_prefix_rather_than_a_pattern`
>   以 `not every key` 失败。
> * 证据：**55 suites / 2796 passed**（+11），clippy、5 guard、matrix 均绿。
>
> ★ **而它顺手暴露了一件事**：存储层**没有任何删除插件行的方法**
> （无 `remove_plugin`），所以级联只能在 SQL 层面断言。卸载路径仍然不存在。
>
> ★ **两半都完了（已提交）。** daemon 的 `KernelHost` 实现了四个操作加
> `storage_used`，`storage-set` 在写之前检配额。
> * ★ **配额只有一个来源，而不是两个。** trait 原来的方法是
>   `storage_quota() -> (used, allowed)`，但 **allowed 不是宿主能知道的**：它属于配额，而配额就是
>   `HostCalls` 里那个 `CallBudget` 的来源。两边各报一个数就是**两个来源**，而它们会在
>   第一次有人收紧配额时分歧。现在宿主只回答 **usage**，上限由 budget 提供。
> * ★ **而这修正了一个只能证明“数字原样返回”的测试**：
>   `a_storage_quota_is_a_pair_of_numbers_a_plugin_can_read` 断言的 `[128,1024]` 是 **stub 自己的两个常量**。
>   两个真正的数字取代了它，断言现在是关于**每一半各自从哪里来**。
> * ★ **写入巿径是 `used - replaced + new`，不是 `used + new`**：用更短的值重写一个键
>   是在**收紧**存储，不看被替换掉什么的检查会拒绝一次让情况变好的写入。
>   阴性对照：改成 `used + new`，该测试以
>   `would store 17 bytes, and its limit is 12` 失败。
> * ★ **而它用了一个自己的错误码**：`CallError::QUOTA_EXCEEDED = "quotaExceeded"`。
>   设计叫来就发明了 `PluginQuotaExceeded` 这个名字，而它之前只能归到 `rateLimited`
>   —— 那会告诉插件“你太快了，等一下”，而它真正该做的是存少一点。
> * 证据：**55 suites / 2798 passed**，clippy、5 guard 均绿。
>
> ★ **而它暴露了一个空转的测试与一个真实的缺口。**
> `a_storage_quota_is_a_pair_of_numbers_a_plugin_can_read` 断言的是 **stub 自己的两个常量**——
> 双重回答 `storage_quota` 时给出一对写死的数字，于是测试验证的只是"双重编造的数字原样返回"。
> 而真正缺的那一半是：**app 的 `DesktopHost` 仍然对五个函数全部拒绝**，
> 所以桌面 shell 里的插件依然存不下任何东西。

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
3. 调用方必须声明 `action:invoke:<完整 @scope/name>:<name>`。允许通配授权：`action:invoke:*:format-date`；实际参数不能为通配符，动作名须为 1–64 个 ASCII 字母、数字、点、下划线或连字符；manifest 与运行时共用校验；
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
| 4 | UI 插件是否允许直接使用 Vue                                      | 允许（注入 Vue 运行时） / 只允许原生 DOM                 | **允许**（生态与开发效率）。★ 注意这不是"Vue 实例不共享"这么窄：**任何组件实例、任何运行时都跨不过隔离边界**，宿主也不能把自己的 antd 交给插件。视觉一致改由设计令牌承担（§5.5） |
| 5 | headless 插件是否支持网络直接访问（WASI socket）                   | 支持 / 只经内核代理                                   | **只经内核代理**（可审计、可限流） |
| 6 | 插件能否声明定时任务                                             | 支持（内核调度） / 不支持                             | **支持**（`on-tick`），但配额严格 |
| 7 | 插件状态是否持久化（Memento）                                     | 自动快照 / 插件自己用 storage                          | **插件自己用 storage**（简单、可测） |
