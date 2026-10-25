# 04 · 线协议

> 本文定义 Synra 设备之间的**线上协议**：发现、建链、信封、事件、会话。
> 它是**新设计**，不存在"兼容旧格式"的约束——所有字段、编码、语义都由本文规定，实现必须与本文一致。
>
> 协议定义在 `synra-core/crates/synra-protocol/`（无 IO 依赖的纯数据与规则）。
> 任何线上行为都能在本文找到依据；本文没写的，不允许实现。

---

## 1. 设计约束（为什么协议长这样）

| # | 约束                                                     | 对协议的影响                                                     |
| - | -------------------------------------------------------- | ---------------------------------------------------------------- |
| 1 | **同一份实现对桌面与移动**，不写三遍                        | 没有"平台特定的字段"；发现与传输只有一套语义                       |
| 2 | **信封字段可审计**，不允许插件凭空造字段                     | 信封是固定字段的结构体，`deny_unknown_fields`                      |
| 3 | **大对象不能撑爆信封**                                     | 数据面与会话分离；信封只承载控制信息                               |
| 4 | **每一帧都要能定位到一次用户操作**                            | 信封携带 `trace_id`，全链路贯通                                    |
| 5 | **接收端必须能拒绝坏数据而不损坏链路**                       | 帧层显式长度与上限，超限即断链（不是跳过）                          |
| 6 | **可靠性语义只有一种**，不因方向而异                         | 出站与入站使用同一套 ACK 与超时策略                                |
| 7 | **不能有"幽灵字段"**（类型里有、线上从不出现）                | 本文出现的每个字段必须有生产者与消费者，并被一致性测试覆盖           |
| 8 | **插件不能碰到协议**                                       | 插件只说"我要发这个事件"，信封由内核构造                            |

---

## 2. 分层

```
┌──────────────────────────────────────────────────────────────┐
│ 应用层   Intent / Share / Clipboard / Plugin Action           │
├──────────────────────────────────────────────────────────────┤
│ 事件层   EventName（_synra.* / _plugin.<slug>.* / 数据面名）    │
├──────────────────────────────────────────────────────────────┤
│ 信封层   Envelope（固定字段；固定 8 个 + 2 个可选）              │
├──────────────────────────────────────────────────────────────┤
│ 会话层   TransferSession / PairingSession / IntentSession      │
├──────────────────────────────────────────────────────────────┤
│ 帧层     Frame：u32 BE 长度 + u8 kind + body                   │
├──────────────────────────────────────────────────────────────┤
│ 传输层   TCP（设备间）· 本地 socket（宿主 ↔ 内核控制面）           │
├──────────────────────────────────────────────────────────────┤
│ 发现层   UDP 广播 + mDNS（同一份报文模型）                       │
└──────────────────────────────────────────────────────────────┘
```

---

## 3. 标识与类型

`synra-protocol` 里的所有标识都是**新类型（newtype）**，不裸用 `String`。这条是硬性的：
它让"把 transfer_id 当成 request_id 传"这类错误在编译期就不可能。

```rust
// crates/synra-protocol/src/ids.rs

/// 所有 ID 都是 UUIDv7（时间有序，便于 SQLite 索引与日志排序）
macro_rules! uuid_newtype {
    ($name:ident, $prefix:literal) => {
        #[derive(Debug, Clone, Copy, PartialEq, Eq, Hash, PartialOrd, Ord, Serialize, Deserialize)]
        #[serde(transparent)]
        pub struct $name(pub Uuid);

        impl $name {
            pub fn new() -> Self { Self(Uuid::now_v7()) }
            pub fn is_nil(&self) -> bool { self.0.is_nil() }
        }

        impl std::str::FromStr for $name {
            type Err = IdError;
            fn from_str(s: &str) -> Result<Self, IdError> {
                // 唯一构造路径：必须是合法 UUID。不存在"兜底生成"的分支。
                Uuid::parse_str(s).map(Self).map_err(|_| IdError::NotUuid)
            }
        }
    };
}

uuid_newtype!(DeviceId,   "dev");
uuid_newtype!(RequestId,  "req");
uuid_newtype!(TransferId, "xfr");
uuid_newtype!(IntentId,   "int");
uuid_newtype!(ScanId,     "scan");
uuid_newtype!(SessionId,  "ses");

/// 128-bit trace id（W3C trace context 兼容：32 位十六进制）
#[derive(Debug, Clone, Copy, PartialEq, Eq, Hash)]
pub struct TraceId(pub u128);

impl TraceId {
    pub fn new() -> Self { Self(rand::random()) }
    pub fn to_hex(self) -> String { format!("{:032x}", self.0) }
}

// ★ 上面那两行原本是 `#[derive(Serialize, Deserialize)]` + `#[serde(transparent)]`，
// 而它们与**自己上一行的注释相矛盾**：那句说的是"32 位十六进制"，
// 而 `transparent` 把它序列化成一个**128 位的 JSON 数字**。三件事同时错了：
//
// 1. `serde_json::Value` 直接拒绝：`number out of range` —— 所以任何经过 `Value`
//    的路径（比如 IPC 转换）都会在遇到 trace 时失败；
// 2. JavaScript 的 `JSON.parse` **不报错，而是静默地四舍五入** ——
//    一个"差不多对"的 trace id 比没有更糟：它把错的步骤串在一起，而看起来像是对的；
// 3. `synra-js` 对外发布的是 `export type TraceId = string`（`wire.d.ts`）——
//    **Rust 侧和 TypeScript 侧对同一字段的类型说法不一致**。
//
// 现在它自己实现 serde：序列化为 32 位十六进制字符串，反序列化时校验
// 非空、≤ 32 位、全部是十六进制。这也才是 W3C trace context 的写法。
impl Serialize for TraceId { /* serialize_str(self.to_hex()) */ }
impl<'de> Deserialize<'de> for TraceId { /* from_str_radix(text, 16) */ }

/// 插件标识：由 manifest 声明，全局唯一
#[derive(Debug, Clone, PartialEq, Eq, Hash, PartialOrd, Ord, Serialize, Deserialize)]
#[serde(try_from = "String", into = "String")]
pub struct PluginId(String);          // 规则见 [15 §2.2](./15-plugin-sdk.md)

/// 插件在事件名里的短名：`@synra-plugin/chat` → `chat`
#[derive(Debug, Clone, PartialEq, Eq, Hash, Serialize, Deserialize)]
#[serde(try_from = "String", into = "String")]
pub struct PluginSlug(String);        // 字符集 [a-z0-9-]，长度 1..=48
```

**为什么用 UUIDv7**：时间有序。这让 SQLite 的主键索引不碎片化，日志按时间排序不需要额外列，
且 `TransferId` 的调试输出可以直接看出先后。

---

## 4. 帧层

### 4.1 格式

```
┌──────────────────┬──────────┬──────────┬─────────────────────────┐
│ u32 BE length    │ u8 kind  │ u8 flags │ body（length - 2 字节）   │
└──────────────────┴──────────┴──────────┴─────────────────────────┘
     4 字节            1 字节     1 字节
```

```rust
pub const FRAME_HEADER_LEN: usize = 6;
pub const MAX_FRAME_LEN: u32 = 1024 * 1024;      // 含 header，整个帧

#[derive(Debug, Clone, Copy, PartialEq, Eq)]
#[repr(u8)]
pub enum FrameKind {
    /// 信封（UTF-8 JSON）
    Envelope = 0x01,
    /// 二进制数据块
    Chunk    = 0x02,
    /// 心跳（body 为空）
    Ping     = 0x03,
    Pong     = 0x04,
    /// 建链握手
    Hello    = 0x05,
    HelloAck = 0x06,
    /// 主动关闭（body 是原因码 + 可选文本）
    Bye      = 0x07,
}

#[derive(Debug, Clone)]
pub struct Frame {
    pub kind: FrameKind,
    pub flags: u8,                  // 保留，当前恒为 0
    pub body: FrameBody,
}

#[derive(Debug, Clone)]
pub enum FrameBody {
    Envelope(Envelope),
    Chunk(ChunkFrame),
    Empty,
    Hello(Hello),
    HelloAck(HelloAck),
    Bye(Bye),
}

pub struct ChunkFrame {
    pub chunk_id: u32,              // 会话内序号
    pub data: Bytes,                // 裸字节，上限 MAX_CHUNK_LEN
}

pub const MAX_CHUNK_LEN: usize = 256 * 1024;
```

### 4.2 解码规则（必须严格）

| 情况                              | 行为                                                              |
| --------------------------------- | ----------------------------------------------------------------- |
| `length < 2`                      | `FrameError::Truncated` → **关闭链路**                             |
| `length > MAX_FRAME_LEN`          | `FrameError::TooLarge` → **关闭链路**（不跳过、不尝试重同步）        |
| `kind` 未知                       | `FrameError::UnknownKind` → 关闭链路（**不允许忽略**）              |
| JSON 解析失败                     | `FrameError::Malformed` → 关闭链路                                 |
| 信封字段不合法 / 有多余字段        | `FrameError::InvalidEnvelope` → 关闭链路                           |
| body 长度与声明不符               | `FrameError::LengthMismatch` → 关闭链路                            |

> **设计理由**（写进代码注释）：这是数据面最容易被写错的地方。
> 一个"宽容"的解码器会试图跳过坏帧并继续——但那样会**永久失去帧边界对齐**，
> 之后所有帧都解析失败，表现为"这条链路莫名其妙全坏了"。
> **宁可断链重连，也不要带着错位继续跑。** 断链的代价是几秒重连，错位的代价是整条链路不可用且难排查。

### 4.3 编解码的实现要求

```rust
/// 解码器持有跨帧的缓冲；它必须是"要么整体成功、要么报错并且拒绝继续"的
pub struct FrameDecoder { /* buf: BytesMut，见 [26 §5.4](./26-core-stability-and-perf.md) */ }

impl FrameDecoder {
    /// ★ 传入 `Bytes`（从 socket 读到的共享切片）——**不复制**。
    /// 传 `&[u8]` 会强迫内部 `extend_from_slice`，在吞吐路径上是一次纯浪费。
    pub fn push(&mut self, input: Bytes) -> Result<(), FrameError>;

    /// 返回 Ok(None) 表示"数据还不够，等更多字节"。
    /// 任何 Err 都意味着调用方**必须**销毁这个解码器并断开链路。
    ///
    /// 返回的 `Fragment` **借用缓冲**（零拷贝）；缓冲在它存活期间不被复用。
    pub fn next_frame(&mut self) -> Result<Option<Fragment<'_>>, FrameError>;

    /// 缓冲已完全消费 → 复用（重置长度，不清零）。见 §5.4 的回收策略。
    pub fn recycle(&mut self);
}

pub struct FrameEncoder;

impl FrameEncoder {
    /// 编码不做任何"尽力而为"的截断：超限就返回错误。
    pub fn encode(frame: &Frame, out: &mut BytesMut) -> Result<(), FrameError>;
}
```

**为什么帧层就必须零拷贝**（[26 §5.5](./26-core-stability-and-perf.md)）：

| 环节             | 用 `to_vec` 的代价                              | 用 `Bytes`/`Fragment` 的代价 |
| ---------------- | ---------------------------------------------- | -------------------------- |
| 一个 3 MB 的传输  | ≈ 12 MB 额外复制（读 + 解码 + 写盘 + 摘要各一次） | 几十次引用计数操作            |
| 每秒几千帧的信封   | 每帧 1～2 次堆分配 + 内存拷贝                     | 0 次分配（热路径零分配断言，[26 §6.2](./26-core-stability-and-perf.md)） |


**测试要求**（[21 §2.2](./21-testing-strategy.md)）：

- `proptest` 覆盖"任意字节流 → 解码器要么返回合法帧，要么报错，绝不 panic、绝不越界"；
- 显式的表驱动用例覆盖 §4.2 的每一行；
- `cargo-fuzz` 目标：`decode` 对随机输入不 panic、不消耗无限内存。

---

## 5. 信封层

### 5.1 结构

```rust
#[derive(Debug, Clone, Serialize, Deserialize)]
#[serde(rename_all = "camelCase", deny_unknown_fields)]
pub struct Envelope {
    /// 本次发送的唯一标识。ACK 与请求-应答都用它关联。
    pub request_id: RequestId,

    /// 事件名（见 §6）
    pub event: EventName,

    /// 目标设备
    pub target: DeviceId,

    /// 来源设备。发送端必须填自己的真实 id，接收端必须校验。
    pub from: DeviceId,

    /// 这是对哪一条请求的应答（应答类事件必填）
    #[serde(skip_serializing_if = "Option::is_none", default)]
    pub reply_to: Option<RequestId>,

    /// 全链路追踪 id
    pub trace_id: TraceId,

    /// 事件相关的数据。大小上限见 §5.4。
    ///
    /// ★ 类型是 `RawValue` 而不是 `serde_json::Value` —— 见 [26 §5.1](./26-core-stability-and-perf.md)。
    /// 理由：内核在大多数情况下**不需要理解 payload**（只需校验信封、路由、转发），
    /// 用 RawValue 可以做到"解析零分配 + 转发不重新序列化"。
    /// 只有消费方（插件 SDK / 强类型基础事件转换）才解析。
    #[serde(skip_serializing_if = "Option::is_none", default)]
    pub payload: Option<Box<RawValue>>,

    /// 毫秒时间戳（发送端时钟；仅用于日志与排序，**不用于超时判定**）
    pub sent_at_ms: i64,

    /// 该帧的存活时间。接收端超时后丢弃。`None` = 不过期。
    #[serde(skip_serializing_if = "Option::is_none", default)]
    pub ttl_ms: Option<u32>,
}
```

**这是完整的字段清单——没有第 11 个字段。** 新增字段必须同时改本文与 `Envelope`，并走 §9 的协议演进流程。

**借用版本（接收路径用这个）**：

```rust
/// 零拷贝的接收视图：`payload` 与 `event` 都借用帧缓冲（用 `CompactStr` 承载内联短串）。
///
/// 生命周期绑定到帧缓冲 —— 所以缓冲不能在处理完之前回收（[26 §5.4](./26-core-stability-and-perf.md)）。
/// 需要跨 `await` 或跨任务持有 payload 时，显式调用 `to_owned()` 转换。
#[derive(Debug, Clone)]
pub struct EnvelopeRef<'a> {
    pub request_id: RequestId,
    pub event: EventName,                 // 解析时按需分配一次（见 §6.8 的事件 id 缓存）
    pub target: DeviceId,
    pub from: DeviceId,
    pub reply_to: Option<RequestId>,
    pub trace_id: TraceId,
    pub payload: Option<&'a RawValue>,    // ★ 零拷贝：指向帧缓冲里的 JSON 文本
    pub sent_at_ms: i64,
    pub ttl_ms: Option<u32>,
}

impl EnvelopeRef<'_> {
    /// 需要跨越帧缓冲的生命周期时使用
    pub fn to_owned(&self) -> Envelope;
}
```

### 5.2 不变量

| # | 不变量                                                          | 校验位置   |
| - | --------------------------------------------------------------- | ---------- |
| 1 | `from` 必须是合法 UUID，且**必须**等于链路的对端 id（防伪造）      | 传输层     |
| 2 | `target` 必须是本机 id，否则 `CONNECT_INVALID` 级错误并断链        | 传输层     |
| 3 | `request_id` 不允许重复（同一链路上 5 分钟窗口内）                 | 传输层     |
| 4 | `reply_to` 若非空，必须指向一个**本机发出过**且未过期的 `request_id` | 会话层     |
| 5 | `trace_id` 非 nil                                                | 信封构造   |
| 6 | `payload` 序列化后 ≤ `MAX_PAYLOAD_LEN`（256 KiB）                 | 信封构造   |
| 7 | 时间戳与超时判定**无关**（用本地单调时钟）                          | 会话层     |

### 5.3 构造与解析的单一入口

```rust
impl Envelope {
    /// 发送端唯一构造入口。调用方不能自己拼字段。
    ///
    /// payload 接受 `&impl Serialize`：调用方传自己的类型，内核序列化**一次**到 RawValue。
    /// 也可以直接传 `Option<Box<RawValue>>` 做转发（不重新序列化）。
    pub fn outbound<P: ?Sized + Serialize>(
        event: EventName,
        target: DeviceId,
        from: DeviceId,
        payload: Option<&P>,
        trace: TraceContext,
    ) -> Result<Self, EnvelopeError>;

    /// 应答。
    pub fn reply_to<P: ?Sized + Serialize>(
        request: &Envelope,
        event: EventName,
        payload: Option<&P>,
        local: DeviceId,
    ) -> Result<Self, EnvelopeError>;

    /// 接收端解析（拥有版）。`raw` 是**已通过语法校验**的 JSON 文本。
    pub fn inbound(raw: &RawValue, local: DeviceId) -> Result<Self, EnvelopeError>;

    /// 接收端解析（**零拷贝借用版**，热路径用这个）。
    /// 生命周期绑定到帧缓冲（[26 §5.4](./26-core-stability-and-perf.md)）。
    pub fn inbound_ref<'a>(raw: &'a RawValue, local: DeviceId) -> Result<EnvelopeRef<'a>, EnvelopeError>;
}
```

> **为什么入参是 `&impl Serialize` 而不是 `serde_json::Value`**：
> 传 `Value` 会强迫调用方**先建一棵 JSON 树**（每个字段一次分配），内核再把它序列化一遍。
> 传 `&impl Serialize` 则是一次序列化直接写进输出缓冲。
> 这条路也是 [26 §5.1](./26-core-stability-and-perf.md) 的结论：**能少一次分配就少一次**。


> **为什么强制单一入口**：信封的 10 个字段里，有 4 个（`from`/`target`/`trace_id`/`reply_to`）
> 是**调用方不该自己决定**的。让它们只能由 `Envelope::outbound` / `reply_to` 填，
> 就从类型上消灭了"某个调用点忘了填 trace_id"这类问题。

### 5.4 大小上限

| 项                       | 上限      | 超过时                                                |
| ------------------------ | --------- | ----------------------------------------------------- |
| `payload` 序列化后        | 256 KiB   | `EnvelopeError::PayloadTooLarge`；提示改用 `transfer`   |
| JSON 嵌套深度             | 32 层     | `EnvelopeError::TooDeep`                              |
| 单帧总长                  | 1 MiB     | `FrameError::TooLarge`（见 §4.2）                      |
| 单块（`ChunkFrame.data`） | 256 KiB   | 编码端拒绝；接收端断链                                  |

---

## 6. 事件层

### 6.1 命名空间

| 前缀                    | 谁可以发                                | 语义                              |
| ----------------------- | --------------------------------------- | --------------------------------- |
| `_synra.`               | 仅内核                                   | 系统事件（配对、意图回执、插件同步…） |
| `_plugin.<slug>.`       | 只有该 slug 的插件（内核强制）             | 插件事件                          |
| **数据面裸名**           | 仅内核                                   | `file.transfer.*`（见 §8）         |
| `synra.internal.`       | 仅内核进程内（不经网络）                   | 控制面 / 宿主内部                  |

```rust
#[derive(Debug, Clone, PartialEq, Eq, Hash)]
pub struct EventName(CompactString);

impl EventName {
    /// 构造系统事件
    pub fn system(logical: &str) -> Result<Self, EventNameError>;

    /// 构造插件事件（内核侧调用；插件自己不能指定 slug）
    pub fn plugin(slug: &PluginSlug, logical: &str) -> Result<Self, EventNameError>;

    /// 数据面事件
    pub fn data_plane(logical: DataPlaneEvent) -> Self;

    pub fn is_host_only(&self) -> bool;
    pub fn plugin_scope(&self) -> Option<(&PluginSlug, &str)>;
}
```

**硬约束（内核强制，不是约定）**：

```
插件 emit 的事件，其 `_plugin.<slug>.` 部分**必须**等于该插件自己的 slug。
否则返回 CapabilityDenied（理由：伪造其它插件的事件）。
```

### 6.2 两类事件：内核的和插件的

**这是必须先分清的一件事。** 事件名按"谁来定义、谁保证 payload 形状"分两类：

|             | **内核基础事件**                                       | **插件自定义事件**                       |
| ----------- | ---------------------------------------------------- | ---------------------------------------- |
| 谁定义名字    | **内核**（本节的常量表；插件不能新增）                     | 插件作者（在 manifest 里声明）             |
| 谁定义 payload | **内核**（Rust 类型；插件直接拿到解析好的结构）             | 插件作者（manifest 里的 JSON Schema）      |
| 前进兼容性    | 由内核负责（`#[non_exhaustive]` + Feature，[§9](#9-协议演进)） | 由**声明对比**保证（§6.6）                  |
| 谁能发        | 内核（少数白名单事件允许插件发，见 §6.5）                   | 该插件自己（命名空间强制）                   |
| 命名          | `device.*` / `file.transfer.*` / `_synra.*`             | `_plugin.<自己的 slug>.*`                  |

> **设计原则**：内核提供的是**能力面**（"有设备上线了"、"文件收完了"、"有分享进来了"），
> 插件提供的是**语义**（"这是一条聊天消息"、"这是一份发票"）。
> **内核不预测插件的语义**，所以它不为插件的业务事件做任何约定。

### 6.3 内核基础事件目录（权威）

#### A. 内核内部事件（插件**不能**收到，也不能发）

这些是协议机制本身。它们出现在 §4/§7 的帧层与握手流程里，**不会**投递给插件的 `on_event`。

```rust
pub mod kernel_internal {
    // ── 传输控制（帧层直接处理；不发 ACK，见 §7.4）──
    pub const HELLO:     &str = "device.hello";
    pub const HELLO_ACK: &str = "device.hello.ack";
    pub const ACK:       &str = "device.ack";
    pub const PING:      &str = "device.ping";
    pub const BYE:       &str = "device.bye";
    pub const ERROR:     &str = "device.error";        // 协议级错误（非业务错误）

    // ── 会话控制（由内核的会话状态机消费）──
    pub const PAIR_REQUEST:  &str = "device.pairing.request";
    pub const PAIR_RESPONSE: &str = "device.pairing.response";
    pub const UNPAIR:        &str = "device.pairing.unpair";
    pub const INTENT_INVOKE: &str = "_synra.intent.invoke";
    pub const INTENT_RESULT: &str = "_synra.intent.result";
    pub const INTENT_CANCEL: &str = "_synra.intent.cancel";

    // ── 插件同步（内核之间的元数据交换，不经过插件）──
    pub const PLUGIN_QUERY:       &str = "_synra.plugin.query";
    pub const PLUGIN_QUERY_REPLY: &str = "_synra.plugin.query.reply";
    pub const PLUGIN_SYNC_RESULT: &str = "_synra.plugin.sync.result";

    /// 不触发自动 ACK 的事件（其余一律自动 ACK）
    pub const NO_AUTO_ACK: &[&str] = &[HELLO, HELLO_ACK, ACK, PING, BYE, ERROR];
}
```

#### B. 插件可见的基础事件（插件可以 `handles`）

**这些是内核提供给插件的"能力面"。** ★ 标记的四条是绝大多数插件会用到的。

```rust
pub mod builtin {
    // ── 设备与链路（★ device.changed 是最常用的入口）──
    /// ★ 已配对设备的在线/离线/信息变化。payload 见下方表。
    pub const DEVICE_CHANGED: &str = "_synra.device.changed";
    /// 某台设备离线（更细粒度；想只关心离线时用这个）
    pub const DEVICE_OFFLINE: &str = "_synra.device.offline";
    /// 某台设备的显示名变了
    pub const DEVICE_RENAMED: &str = "_synra.device.renamed";
    /// 配对关系变化（被对方解除、或新配对成功）
    pub const PAIRING_CHANGED: &str = "_synra.pairing.changed";

    // ── 分享与意图（★ share.received 是"分享类插件"的入口）──
    /// ★ 本机收到一个分享项，且**没有任何插件声明处理它时**才发（见下方说明）
    pub const SHARE_RECEIVED: &str = "_synra.share.received";
    /// 本机发起的意图，远端执行完成
    pub const INTENT_COMPLETED: &str = "_synra.intent.completed";
    /// 本机发起的意图最终失败（含结构化错误）
    pub const INTENT_FAILED: &str = "_synra.intent.failed";

    // ── 传输（★ transfer.completed 是"文件类插件"的入口）──
    /// ★ 一个传输会话成功完成且已校验摘要（插件在这里拿落盘路径）
    pub const TRANSFER_COMPLETED: &str = "_synra.transfer.completed";
    /// 传输失败/中止（含原因码）
    pub const TRANSFER_FAILED: &str = "_synra.transfer.failed";
    /// 传输进度（低价值、高频；插件要订阅需显式声明）
    pub const TRANSFER_PROGRESS: &str = "_synra.transfer.progress";

    // ── 剪切板 ──
    /// 对端推送过来的剪切板内容（本机写入剪贴板后发）
    pub const CLIPBOARD_RECEIVED: &str = "_synra.clipboard.received";

    // ── 插件生命周期（发给**所有**插件，用于跨设备状态同步）──
    /// 某个插件在本机安装成功
    pub const PLUGIN_INSTALLED: &str = "_synra.plugin.installed";
    /// 某个插件被更新（从 oldVersion → newVersion）
    pub const PLUGIN_UPDATED: &str = "_synra.plugin.updated";
    /// 某个插件被卸载
    pub const PLUGIN_UNINSTALLED: &str = "_synra.plugin.uninstalled";
    /// 某个插件的授权发生变化（用户改了权限）
    pub const PLUGIN_CAPABILITY_CHANGED: &str = "_synra.plugin.capability-changed";

    // ── 定时与生命周期 ──
    /// 宿主进入前台（移动端恢复时）
    pub const HOST_FOREGROUND: &str = "_synra.host.foreground";
    /// 宿主即将进入后台/休眠（插件应尽快持久化状态）
    pub const HOST_BACKGROUND: &str = "_synra.host.background";
    /// 网络接口变化（Wi-Fi 漫游、插拔网线）
    pub const NETWORK_CHANGED: &str = "_synra.network.changed";
}
```

**各自的 payload（内核保证的形状）**：

| 事件                          | payload                                   | 说明 |
| ----------------------------- | ----------------------------------------- | ---- |
| `_synra.device.changed`        | `DeviceSnapshot[]`                          | 全量快照（不是增量）。插件不必自己维护差量 |
| `_synra.device.offline`        | `{ deviceId, lastSeenAtMs }`                | —    |
| `_synra.device.renamed`        | `{ deviceId, displayName }`                 | —    |
| `_synra.pairing.changed`       | `{ deviceId, paired: bool, reason?: string }` | —  |
| `_synra.share.received`        | `ShareItem`                                 | **只在无插件声明处理该分享时**才发（见下） |

> ★ **上面那张图的第一步现在存在了**（已提交）：内核收到同伴的分享后会调 `Matcher`，候选进意图流程。
> 但 **第四步（无候选时发 `_synra.share.received` 给插件）仍然没做**：
> 那需要向插件投递事件，而 `EventDispatcher` 到今天仍然没有生产调用方。
| `_synra.intent.completed`      | `IntentOutcome`                             | [§8.3](#83-意图) |
| `_synra.intent.failed`         | `IntentOutcome`（`ok: false`）               | 同上 |
| `_synra.transfer.completed`    | `{ transferId, kind, byteLength, digest, localPath?, meta }` | 摘要**已校验**过才发 |
| `_synra.transfer.failed`       | `{ transferId, reason: TransferAbortReason }` | [§8.4](#84-文件传输) |
| `_synra.transfer.progress`     | `TransferProgress`                          | 高频；默认不投递 |
| `_synra.clipboard.received`    | `{ text?, mime, fromDeviceId }`             | 敏感：默认需用户开启 |
| `_synra.plugin.*`（四条）       | `{ pluginId, version?, oldVersion?, capabilities? }` | 发给所有插件，用于状态同步 |
| `_synra.host.*`                | `{}`                                        | 无 payload（或留扩展位） |
| `_synra.network.changed`       | `{ kind: 'up'\|'down'\|'changed' }`          | —    |

> **为什么 `_synra.device.changed` 发全量快照**：增量事件要求订阅方自己维护状态，
> 而池化的插件实例各有独立内存（[13 §10-3](./13-plugin-authoring.md)）——**它没有地方存差量**。
> 全量快照让插件可以**无状态**地处理。

**关于 `_synra.share.received` 的一个重要语义**：

```
内核收到分享
   ├─ 1. 按 manifest.shareHandlers 预筛 → 得到候选插件
   ├─ 2. 逐个调 can_handle（纯计算，有 fuel 预算）
   │
   ├─ 有候选 → 走正常的"意图"流程（[02 §3](./02-domain-and-flows.md)）
   │           ★ 不发 `_synra.share.received`
   │
   └─ 无候选 → ★ 此时才发 `_synra.share.received`
               （给"通用接收型"插件一个兜底机会，比如"存到电脑"插件）
```

这解决了一个真实的矛盾：**"匹配失败"本身是一个事件**，
但不能让每个插件都为每次分享跑一遍（否则一个总返回 Unhandled 的插件会白白唤醒）。
所以它只在**预筛 + 匹配全失败**后发出。

#### C. 数据面事件（插件经 `transfer` API 用，不能自己发）

```rust
/// 数据面（裸名，不带 `_synra.` 前缀）
#[derive(Debug, Clone, Copy, PartialEq, Eq, Serialize, Deserialize)]
#[serde(rename_all = "snake_case")]
pub enum DataPlaneEvent {
    FileTransferRequest,
    FileTransferChunk,
    FileTransferComplete,
    FileTransferAbort,
    FileTransferProgress,
}
```

> **数据面为什么用裸名，且插件不能直接发**：`file.transfer.*` 是**内核能力**（分块、窗口、摘要校验、续传）。
> 插件通过 SDK 的 `transfer` API 使用它；**直接 emit 会被闸门拒绝**——
> 否则插件可以绕过内核的传输状态机，破坏组装缓冲与摘要校验。

### 6.4 插件自定义事件

#### 命名规则（内核强制，不是约定）

```
_plugin.<slug>.<logical…>

  slug     必须等于该插件 manifest 的 slug（字符集 [a-z0-9-]，长度 1..=48）
  logical  插件自己定，允许 `.` 与 `*`（`*` 只用于 handles 的匹配，不用于发送）
```

**硬约束**：

| # | 规则                                                                     | 违反时                         |
| - | ------------------------------------------------------------------------ | ------------------------------ |
| 1 | 插件只能发 `_plugin.<自己的 slug>.*`                                        | `CapabilityDenied`（伪造他人事件） |
| 2 | 插件不能发 `_synra.*` / `device.*` / `file.transfer.*`                      | `CapabilityDenied`             |
| 3 | 发送还需要 `event:emit:<具体名或通配>` 能力                                 | `CapabilityDenied`             |
| 4 | 接收需要在 manifest `handles` 里声明（否则内核不投递）                        | 静默不投递（记审计）             |
| 5 | 收到的事件名若不在自己的 `handles` 里，**不会**被调用                          | —                              |

> **规则 2 与"基础事件"的关系**：插件**可以**发少数几个基础事件
> （例如"我想让内核广播我更新了"），但那些在 §6.5 的白名单里。
> 白名单之外一律拒绝。

#### payload 的 JSON Schema

插件在 manifest 里为自己的事件声明 payload 形状：

```jsonc
// synra.json
{
  "events": [
    {
      "name": "_plugin.chat.text",
      "payload": {
        "type": "object",
        "required": ["body"],
        "properties": {
          "body":      { "type": "string", "maxLength": 65536 },
          "replyToId": { "type": "string" },
          "sentAtMs":  { "type": "integer" }
        },
        "additionalProperties": false
      }
    },
    {
      "name": "_plugin.chat.typing",
      "payload": { "type": "object", "properties": {}, "additionalProperties": false }
    }
  ]
}
```

约束：

| # | 约束                                                                 | 落点 |
| - | -------------------------------------------------------------------- | ---- |
| 1 | Schema 必须是**顶层 object**（不能是 string/number/array）               | [15 §2](./15-plugin-sdk.md) |
| 2 | `additionalProperties: false` 强制（不允许"顺手多塞字段"）                 | 同上 |
| 3 | `maxLength` / `maximum` 必须有界（否则等于没有约束）                       | 同上 |
| 4 | 序列化后 ≤ 256 KiB（[§5.4](#54-大小上限)）                                | 内核 |
| 5 | Schema 里不能出现 `$ref` / `format` / 正则（避免求值成本与实现差异）          | `verify` |
| 6 | 事件名在**同一插件内**唯一                                                | `verify` |

### 6.5 插件可以发送的内核事件（白名单）

极少数基础事件允许插件主动发（用于"插件想引起内核注意"）：

| 事件                             | 语义                                       | 需要的能力 |
| -------------------------------- | ------------------------------------------ | ---------- |
| `_synra.diagnostics.request`      | 请求生成诊断包（用户点"报告问题"时用）         | `diagnostics:create` |
| `_synra.plugin.capability-request`| 请求用户授予更多能力（插件 UI 里"申请权限"按钮） | 无需（内核必须弹窗让用户确认） |

> **为什么只有两条**：白名单越大，"插件能替内核说话"的面就越大。
> 需要新增时，必须在 [04](./04-wire-protocol.md) 与 [10](./10-capability-and-sandbox.md) 里都加一条，
> 并说明"为什么不能通过 `self.invoke` 或 `platform.*` 达成"。

### 6.6 前进兼容：双方的 schema 必须兼容

跨设备插件事件是**契约**：A 设备的插件发 `_plugin.chat.text`，B 设备的同名插件收。
两台设备上的插件版本可能不同，所以**安装/激活阶段就要检查兼容性**。

```
内核在"装载插件"时：
  1. 收集本机所有已装插件的 `events` 声明（名字 → schema）
  2. 对每个已配对设备，已知对方插件的 events 声明（随 plugin.sync 元数据交换）
  3. 检查"我会发的" ⊆（对方会收的 ∩ 兼容的）
        · 名字不存在       → 该事件不可发（发出去对方是 Unknown，无害但无用）
        · 名字存在但 schema 不兼容 → ★ 拒绝激活，并给出明确原因
```

**什么叫"兼容"**（刻意保守）：

| 变化                                    | 判定 | 理由 |
| --------------------------------------- | ---- | ---- |
| 新增**可选**字段                          | ✅ 兼容 | 旧接收方忽略它（`additionalProperties: false` 只约束**发送方**的 schema） |
| 新增**必填**字段                          | ❌ 不兼容 | 旧接收方拿不到这个字段 |
| 删除字段                                 | ❌ 不兼容 | 旧发送方还在发，新接收方校验失败 |
| 改字段类型 / 改 `maxLength` 变小            | ❌ 不兼容 | — |
| 放宽约束（`maxLength` 变大、可选变必填→必填变可选） | ✅ 兼容 | 旧数据仍然合法 |

**不兼容时的行为**（三选一，按 §6.6.1 的规则）：

| 场景                     | 行为                                                     |
| ------------------------ | -------------------------------------------------------- |
| **本机主动发**的方向不兼容  | 该事件**不可发**。`verify` 与激活时都报错，明确指出与哪台设备、哪个版本冲突 |
| **接收**方向不兼容         | 仍可接收（内核只做结构校验）；把不兼容的字段**记入审计** |
| 双方都升级了但中间版本缺失 | 按"不可发"处理，UI 提示"对方插件版本过低，该功能不可用"       |

#### 6.6.1 为什么是"拒绝激活"而不是"运行时报错"

| 方案                    | 问题                                                       |
| ----------------------- | ---------------------------------------------------------- |
| 运行时校验 + 报错         | 用户已经点了按钮才失败，且错误发生在**对端设备**上（用户看不到对端日志） |
| **装载期检查 + 明确拒绝** ✅ | 在最早、最可控的地方暴露；错误信息能直接指向"与哪台设备的哪个插件版本不兼容" |

这是 [11 D1](./11-design-principles.md)（让错误在最早最明确的地方出现）的又一次应用。

> 🔴 **未定**：`plugin.sync` 元数据里要不要带**完整的 events schema**？
> 带 → 能做上面这套检查，但元数据变大；不带 → 只能做名字级检查。
> 倾向：带**名字 + schema 摘要（hash）**，只有摘要不一致时才去拉完整 schema。

### 6.7 语义化的分发入口

内核不按字符串分发业务事件，而是把它们解析成语义枚举：

```rust
#[derive(Debug, Clone)]
pub enum Inbound {
    /// 传输控制（帧层已消费掉大部分；这里只剩需要业务逻辑的）
    MemberOffline { device: DeviceId },
    DisplayNameChanged { device: DeviceId, name: String },

    /// 配对
    PairRequest  { from: PeerProfile, request: RequestId },
    PairResponse { accepted: bool, reason: Option<String>, request: RequestId },
    Unpair       { reason: Option<String> },

    /// 意图
    IntentInvoke(IntentInvocation),
    IntentResult(IntentOutcome),
    IntentCancel { intent: IntentId, reason: String },

    /// 插件同步
    PluginQuery { plugin: PluginId },
    PluginQueryReply { plugin: PluginId, version: Semver, digest: SriDigest },
    PluginSyncResult(SyncOutcome),

    /// 数据面
    Transfer(TransferMessage),

    /// ★ 内核基础事件（[§6.3 B](#63-内核基础事件目录权威)）——已解析成强类型
    Builtin(BuiltinEvent),

    /// 插件事件（路由给插件，见 [13 §4.3](./13-plugin-authoring.md)）
    /// ★ payload 是借用帧缓冲的 RawValue，`logical` 也借用 —— 热路径零分配（§6.8）
    Plugin { slug: PluginSlug, logical: CompactStr, payload: Option<&'a RawValue> },

    /// 未知但格式合法的事件（内核记录并忽略，**不报错、不断链**）
    Unknown { event: EventName, payload: Option<&'a RawValue> },
}

/// 内核基础事件的强类型视图（插件拿到的是这些，不是裸 JSON）
#[derive(Debug, Clone)]
pub enum BuiltinEvent {
    DeviceChanged { devices: Vec<DeviceSnapshot> },
    DeviceOffline { device: DeviceId, last_seen_at_ms: i64 },
    DeviceRenamed { device: DeviceId, display_name: String },
    PairingChanged { device: DeviceId, paired: bool, reason: Option<String> },
    ShareReceived { item: ShareItem },
    IntentCompleted(IntentOutcome),
    IntentFailed(IntentOutcome),
    TransferCompleted(TransferDone),
    TransferFailed { transfer: TransferId, reason: TransferAbortReason },
    TransferProgress(TransferProgress),
    ClipboardReceived { text: Option<String>, mime: String, from: DeviceId },
    PluginInstalled { plugin: PluginId, version: Semver },
    PluginUpdated { plugin: PluginId, from: Semver, to: Semver },
    PluginUninstalled { plugin: PluginId },
    PluginCapabilityChanged { plugin: PluginId, capabilities: Vec<Capability> },
    HostForeground,
    HostBackground,
    NetworkChanged { kind: NetworkChangeKind },
}
```

> **为什么插件拿到强类型而不是裸 JSON**：
> 内核基础事件的 payload 是**内核自己定义**的（§6.3 B 的表），所以它当然能给出强类型。
> 只有**插件自定义事件**的 payload 才是 `RawValue`（内核不认识它的形状，见 §6.8）。
> 这让"用基础事件"的插件不需要手写反序列化与字段名拼写。

> **`Unknown` 这一档是刻意的**：新版设备可以给旧版设备发它不认识的事件。
> 那不是错误（可能是功能不存在或版本不同），记录 + 忽略即可。
> 这与 §4.2 的"帧层任何异常都断链"不矛盾——**帧层是结构，事件层是内容**。
> 结构坏了没法继续，内容不认识完全可以继续。

### 6.8 事件路由：预计算 + 有界缓存

事件路由在**每帧**上执行（[26 §4.1](./26-core-stability-and-perf.md) 的热路径 #2），
所以**不能每帧靠字符串比较**。设计见 [26 §5.3](./26-core-stability-and-perf.md)。

```rust
/// ★ 事件 id：装载期给"已知事件名"与"通配规则"编号
#[derive(Debug, Clone, Copy, PartialEq, Eq, Hash)]
pub struct EventId(u16);

pub struct EventRegistry {
    /// 名字 → id。**只在首次见到某个名字时查询一次**（之后走 `resolved` 缓存）
    ///
    /// ★ 键来自对端（不可信）→ 用**随机种子**的哈希，避免碰撞 DoS（[26 §4.3](./26-core-stability-and-perf.md)）
    by_name: HashMap<EventName, EventId, foldhash::fast::RandomState>,

    /// ★ 路由表：id → 订阅它的插件。O(1) 查表，不需要字符串比较
    routes: Vec<SmallVec<[PluginHandle; 2]>>,

    /// 通配规则（数量**有界**：manifest 里 `handles` 的通配条数有上限）
    wildcards: SmallVec<[WildcardRoute; 8]>,

    /// 已解析名字的有界 LRU（防止对端发海量陌生事件名导致内存增长，[11 C3](./11-design-principles.md)）
    resolved: LruCache<EventName, EventId>,
}

impl EventRegistry {
    /// 装载插件时建立索引（把 manifest 的 `handles` 编译进 `routes`/`wildcards`）
    pub fn register(&mut self, plugin: PluginHandle, handles: &[HandleRule]) -> Result<()>;

    /// 热路径：一帧事件 → 要投递的插件集合。**O(1) 查表 + 零分配**（返回借用切片）
    pub fn route<'a>(&'a mut self, event: &str) -> Routed<'a>;

    /// 卸载插件时撤销路由（不能留下悬挂 target）
    pub fn unregister(&mut self, plugin: PluginHandle);
}
```

| 设计点                        | 为什么                                                        |
| ----------------------------- | ------------------------------------------------------------- |
| `EventId` 是 `u16`             | 内核知道的事件名有界（基础事件 + 已装插件声明的事件）；查表只需一次数组索引 |
| `routes` 用 `Vec` 索引而非 map   | id 是稠密小整数 → 数组比哈希表快且无攻击面                        |
| `resolved` 是**有界** LRU       | 对端可以发任意事件名 → 无界缓存等于内存泄漏（[11 C3](./11-design-principles.md)） |
| `wildcards` **有界**            | 否则一个插件用上百条通配就能拖慢所有事件的路由                      |
| `route()` 返回借用              | 热路径零分配（[26 §6.2](./26-core-stability-and-perf.md) 的分配断言覆盖它） |
| `unregister` 必须存在           | 卸载插件后残留路由会让事件投给不存在的实例（静默丢失）               |

---

## 7. 传输层

### 7.1 端口与常量

```rust
// crates/synra-protocol/src/limits.rs
pub mod limits {
    /// 设备间主连接的默认端口
    pub const DEFAULT_TCP_PORT: u16 = 32100;
    /// UDP 发现端口
    pub const DISCOVERY_PORT: u16 = 32101;
    /// mDNS 服务类型
    pub const MDNS_SERVICE: &str = "_synra._tcp.local";

    /// 建链握手超时
    pub const HELLO_TIMEOUT: Duration = Duration::from_secs(6);
    /// 心跳间隔
    pub const HEARTBEAT_INTERVAL: Duration = Duration::from_secs(10);
    /// 心跳超时（超过这个时间没有任何入站活动即判定链路死亡）
    pub const HEARTBEAT_TIMEOUT: Duration = Duration::from_secs(25);
    /// 等 ACK 超时
    pub const ACK_TIMEOUT: Duration = Duration::from_secs(6);

    /// 帧与 payload 上限（见 §4/§5）
    pub const MAX_FRAME_LEN: u32 = 1024 * 1024;
    pub const MAX_CHUNK_LEN: usize = 256 * 1024;
    pub const MAX_PAYLOAD_LEN: usize = 256 * 1024;

    /// 写队列容量（帧数）。溢出 → 该链路判为不健康并关闭。
    pub const WRITE_QUEUE_CAPACITY: usize = 256;
    /// 单帧处理时间上限（防止一个慢处理器饿死读循环）
    pub const MAX_FRAME_PROCESSING: Duration = Duration::from_millis(250);

    /// 请求去重窗口
    pub const REQUEST_DEDUP_WINDOW: Duration = Duration::from_secs(300);
    /// 应答缓存保留时长（幂等重试）
    pub const REPLY_CACHE_TTL: Duration = Duration::from_secs(300);
}
```

**所有这些值只有一个定义处**，且都是 `const`；运行期可通过配置覆盖（`Tuning`），
但默认值就在这里。**不允许在其它文件里再写一遍字面量。**

### 7.2 建链握手

**目标契约（2026-10-25，[ADR-017](./20-adr.md)）**：TCP 建连后先建立经过认证的加密通道，
再在该通道内交换 `Hello` / `HelloAck`。重连必须校验配对时绑定的对端身份公钥；
`device_id`、地址、显示名、challenge 回显和配对 token 均不能单独证明对端身份。
若使用 token 表达配对代次，它只能在受保护通道中传输，不能代替身份认证。

首次配对只能进入权限受限的配对流程，必须通过成熟方案提供的带外核验方式确认双方身份，
核验结果绑定本次安全会话及双方身份密钥。完成双方确认与持久化前，不接受意图执行、文件传输或插件同步。
禁止静默信任新密钥、按设备名自动接受证书，或在认证失败后退回明文。
认证加密库、身份凭据格式和配对核验方案须先完成桌面与 Android 技术验证，再补齐具体字段与测试向量；
不得自行用 nonce、摘要或 challenge 拼接一个认证协议。

下面的图和 payload 是**安全通道内的应用层协商示意**，不描述密码学握手。
共享网络层已实现已批准身份间的 mTLS 通道、设备 id 与证书校验、完整证书固定校验及握手上限
（[实现与验证](./history/2026-10-26-device-authentication.md)）。两个宿主的业务拨号和共享 listener 已接入认证入口，
本机凭据初始化失败时拒绝联网，旧 token 不会被转成凭据授权；CLI 首次配对已接入受限任务服务，
应用首次配对与两个宿主的显式换钥修复均已接入共享流程；记录级登记和身份元数据丢失恢复已接入，沿革不产生授权（[恢复验证](./history/2026-10-26-registration-recovery.md)）；Android 后端及平台恢复仍待完成，
因此尚不满足完整契约，实施与验收见 [19 §0.2](./19-roadmap.md)。

共享层现另有受限配对通道的实现验证：双方通过可信带外路径交换含完整证书的邀请，
再以固定证书的 mTLS 证明私钥持有；发现广播或在当前未信任连接中收到邀请不等于带外批准。
本次用户同意通过 [RFC 9266](https://www.rfc-editor.org/rfc/rfc9266.html) 的 TLS exporter 绑定到双方证书所在的安全会话，
不将自制短码、UUID 摘要或 exporter 回显单独当成身份认证。
配对使用独立 `synra/pairing/1` ALPN，拒绝业务 ALPN；同一个配对机制实例依次交换有界的用户同意记录和持久化回执，
两阶段均检查版本、阶段、双方身份与原始 TLS 会话绑定，不暴露通用业务 IO，完成或失败后关闭，业务流另行建链。
`MutualConsent` 只证明本次双方同意，不能据此授权；共享服务先持久化同意，再交换双方存储成功的回执，
最后原子复核任务与身份后产生独立代次的授权，后续每个业务请求都要重新核对其捕获的授权代次。
取消或撤销必须使原任务无法提交，同证书重新配对也不能恢复旧连接的授权。
回执不能保证两台设备原子提交；一侧最终提交失败时，本机继续拒绝业务，不从未完成日志自动补发授权。
具体流程、测试范围及剩余限制见[初次核验](./history/2026-10-26-pairing-consent.md)与[持久绑定验证](./history/2026-10-26-peer-trust.md)。
旧六位显示码不能证明身份、私钥持有或本次会话，应用和 CLI 生产入口已整体替换；显式换钥会撤销旧绑定，两端必须重新核验。

共享的 `synra_core::peer::authenticated::AuthenticatedPeer` 现将认证通道、Hello 协商和业务链路组成一个入口：
本机 profile 必须匹配本机证书，Hello 与 HelloAck 声明的 id 必须匹配实际 TLS 身份及当前授权代次，
接收方复核后才能发 HelloAck，发起方拒绝非 `Current` 的配对状态；token 不参与身份认证。
TLS 与 Hello 共用不超过 `HELLO_TIMEOUT` 的总预算，业务每帧也有总 IO 预算，TLS 内部的部分记录读取不能重置预算。
Hello 读取预取到的后续帧保留在同一个 decoder 中，交付每个帧前复核授权，包含缓存中的信封和文件块；
IO 返回后也要复核，避免等待期间撤销后交付数据，数据库检查失败使本链路持续拒绝，恢复必须建立新连接。
实现与边界见[认证业务入口验证](./history/2026-10-26-authenticated-peer.md)和[宿主接入验证](./history/2026-10-26-host-authentication.md)。
宿主克隆 `DialConfig` 后在 Host 锁外建立 TLS、校验当前授权并协商 Hello，完成后才短暂持锁记录 session；
出站必须指定目标设备 id 和对应凭据，地址只作为路由，不能单凭一个裸地址选择要信任的身份。
缺少本机原生后端或持久凭据时继续提供本地功能并拒绝业务建链，不静默换钥、回退内存密钥或明文。

```
发起方                                    接收方
  │                                          │
  │ TCP connect                              │
  ├─────────────────────────────────────────▶│
  │ 认证加密握手；首次配对限制业务权限         │
  │◀────────────────────────────────────────▶│
  │ Frame{ kind: Hello, body: Hello{         │
  │   app: "synra",                          │
  │   protocol: 1,                           │
  │   device_id, display_name,               │
  │   platform, app_version,                 │
  │   features: [...],                       │
  │   intent: Fresh | Paired,                │
  │   challenge: [u8; 32]                    │
  │ }}                                       │
  ├─────────────────────────────────────────▶│
  │                                          │ 校验 app / protocol / target
  │                                          │ 决定 intent: Fresh | Paired
  │                                          │ （本地是否已有配对记录）
  │                                          │ 计算 challenge_response
  │ Frame{ kind: HelloAck, body: HelloAck{   │
  │   device_id, display_name, platform,     │
  │   features: [...],                       │
  │   intent: Fresh | Paired,                │
  │   peer_is_paired_locally: bool,          │
  │   challenge_response: [u8; 32]           │
  │ }}                                       │
  │◀─────────────────────────────────────────┤
  │ 校验 challenge_response                   │
  │ → 应用层协商完成；业务权限另查配对状态     │
```

```rust
#[derive(Debug, Clone, Serialize, Deserialize)]
#[serde(rename_all = "camelCase", deny_unknown_fields)]
pub struct Hello {
    pub app: String,                    // 固定 "synra"
    pub protocol: u16,                  // 见 §9
    pub device_id: DeviceId,
    pub display_name: String,
    pub platform: PlatformKind,
    pub app_version: String,
    pub features: Vec<Feature>,
    pub intent: LinkIntent,
    pub challenge: [u8; 32],
}

#[derive(Debug, Clone, Copy, PartialEq, Eq, Serialize, Deserialize)]
#[serde(rename_all = "snake_case")]
pub enum LinkIntent { Fresh, Paired }

#[derive(Debug, Clone, Serialize, Deserialize)]
#[serde(rename_all = "camelCase", deny_unknown_fields)]
pub struct HelloAck {
    pub device_id: DeviceId,
    pub display_name: String,
    pub platform: PlatformKind,
    pub features: Vec<Feature>,
    pub intent: LinkIntent,
    /// 对端（接收方）本地是否把发起方记为已配对。
    /// ★ 每个实现都必须发这个字段——它是"陈旧配对自愈"的依据。
    pub peer_is_paired_locally: bool,
    pub challenge_response: [u8; 32],
}
```

### 7.3 配对授权不一致

本机已撤销或不存在对端授权时，业务连接必须在受保护入口拒绝，不能为了发送 `Fresh` 提示而放行未批准身份。
重连只接受已绑定的确切证书及当前授权代次；对端声明、旧 token 和 `HelloAck` 字段不能恢复授权。

发起方无法认证对端时报告认证失败，保留本机已有绑定，等待用户明确撤销或重新配对；
不能从未认证的断链、替换证书或超时推断“对方已解除配对”，更不能因此自动删除可信绑定。
本机主动解除配对会原子删除授权与列表记录、取消该身份的未完成任务，并使旧业务连接的后续请求被拒绝。
同证书重新配对也产生新代次，旧连接不能复活；首次或重新配对必须走 [§8.2](#82-配对) 的受限流程。

### 7.4 心跳与 ACK

```
心跳：
  每 HEARTBEAT_INTERVAL 发 Frame{ kind: Ping }（body 空，5 字节）
  收到 Ping → 立即回 Pong
  任一入站帧都刷新 last_activity
  now - last_activity > HEARTBEAT_TIMEOUT → 判链路死亡

ACK（可靠投递）：
  除 events::NO_AUTO_ACK 之外的所有信封事件，接收端自动回一条：
      event: "device.ack"
      reply_to: <原 request_id>
      target: <原 from>
  发送端按 request_id 匹配；ACK_TIMEOUT 内未收到 → SendOutcome::Failed
```

```rust
#[derive(Debug, Clone, PartialEq, Eq)]
pub enum SendOutcome {
    /// 对端已确认收到
    Acked { rtt: Duration },
    /// 已写入链路缓冲，未等 ACK（调用方选了 FireAndForget）
    Sent,
    /// 未能发出
    Failed(SendFailure),
}

pub enum SendFailure {
    NotConnected,
    QueueFull,
    AckTimeout { waited: Duration },
    /// 幂等重试时，对端返回了上次的结果
    Deduplicated,
}
```

**两种发送策略，由事件类型决定——不由方向决定**：

```rust
pub enum AckPolicy {
    /// 不等 ACK。只用于心跳类与可丢的进度通知。
    FireAndForget,
    /// 等 ACK（默认）。所有业务事件、控制事件、配对、意图、同步都用它。
    Await,
}
```

> **这是一条重要的纠正**：可靠投递不能因为"这条链路是对方主动连的"就退化。
> 出站与入站必须用同一个 `AckPolicy` 判定函数，否则会出现"某方向的消息静默丢失"。

### 7.5 幂等

```
发送端：重试时**复用同一个 request_id**（不生成新的）
接收端：
  · 收到已知 request_id → 不重新执行，直接回上次的应答（缓存 REPLY_CACHE_TTL）
  · 收到未知 request_id → 执行
```

这是"点两次按钮只开一个浏览器标签"的保证。

### 7.6 断链分类

```rust
#[derive(Debug, Clone, Copy, PartialEq, Eq)]
pub enum DisconnectClass {
    /// 对端正常关闭（收到 Bye）
    RemoteClosed,
    /// 本地主动关闭
    LocalClosed,
    /// 瞬时网络抖动：不打扰用户，直接进入重连
    Transient,
    /// 协议错误（帧损坏、握手失败、目标不匹配）
    ProtocolViolation,
    /// 心跳超时
    HeartbeatTimeout,
    /// 其它 IO 错误
    Io { raw_os_error: Option<i32> },
}

impl DisconnectClass {
    /// 从 IO 错误映射——**用错误类型，不用错误消息里的关键词**
    pub fn from_io(e: &std::io::Error) -> Self {
        use std::io::ErrorKind::*;
        match e.kind() {
            ConnectionReset | BrokenPipe | ConnectionAborted | UnexpectedEof => Self::Transient,
            _ => Self::Io { raw_os_error: e.raw_os_error() },
        }
    }

    /// 是否应该让用户看见
    pub fn user_visibility(self) -> UserVisibility {
        match self {
            Self::Transient | Self::RemoteClosed => UserVisibility::Silent,
            Self::HeartbeatTimeout | Self::Io { .. } => UserVisibility::Degraded,
            Self::ProtocolViolation => UserVisibility::Error,
            Self::LocalClosed => UserVisibility::Silent,
        }
    }
}
```

> **不要用错误消息里的字符串做判定。** `econnreset` / `socket hang up` 这类关键词会随平台与
> 运行库版本变化，而且会把"用户可见的错误"错误地静默掉。用 `ErrorKind` 与 `raw_os_error`。

### 7.7 重连

```rust
pub struct ReconnectPolicy {
    /// 退避序列（用尽后转为指数退避）
    pub backoff: Vec<Duration>,                 // 默认 [1s, 3s, 8s, 20s]
    pub exponential: ExponentialBackoff,        // 之后 base 30s、上限 5min、抖动 ±20%
    /// 触发条件（任一满足即尝试）
    pub triggers: ReconnectTriggers,
}

pub struct ReconnectTriggers {
    /// 已配对设备掉线
    pub on_disconnect: bool,        // true
    /// 该设备被重新发现（★ 比定时轮询有效得多）
    pub on_rediscovered: bool,      // true
    /// 网络接口变化（网卡切换、Wi-Fi 漫游）
    pub on_network_change: bool,    // true
    /// 宿主回到前台（移动端）
    pub on_foreground: bool,        // true
}
```

**重连由内核负责**（不是 UI），且**事件驱动优先于定时**：
设备重新出现在局域网里时立即重试，而不是死等下一次退避。

---

## 8. 会话层

### 8.1 三类会话

| 会话              | 标识        | 状态机                                                                 | 持久化          |
| ----------------- | ----------- | ---------------------------------------------------------------------- | --------------- |
| `PairingSession`  | `RequestId` | `Sent → Answered → Completed \| Rejected \| Expired`                     | 结果入配对表     |
| `IntentSession`   | `IntentId`  | `Matching → Decided → Dispatching → Executing → Done \| Failed \| Expired` | 可选（离线排队） |
| `TransferSession` | `TransferId`| `Requested → Streaming → Assembling → Verified → Persisted \| Failed \| Aborted` | 可选（断点续传） |

**每个会话必须有超时**：

```rust
pub struct SessionTimeouts {
    pub pairing: Duration,          // 默认 30s
    pub intent: Duration,           // 默认 30s
    pub transfer_idle: Duration,    // 默认 60s（无新块则中止）
    pub transfer_total: Duration,   // 默认 30min
    /// 终态在内存里保留多久（供幂等重试返回缓存结果）
    pub result_cache: Duration,     // 默认 5min
}
```

### 8.2 配对

首次配对通过独立 `synra/pairing/1` mTLS 通道完成，不依赖已有业务连接。
双方先通过可信带外路径交换完整公开证书，导入邀请的动作就是明确批准该身份，解析文本本身不构成批准。
CLI 与 daemon 的入口现为 `pair invite`、`pair listen DEVICE --invitation ...`、
`pair connect DEVICE --reply ...`、`pair status` 和 `pair cancel REQUEST_ID`；应用界面尚待接入此流程。

发起方先导出 `synra-pair:1:...`；接收方明确导入后生成有时限的 `synra-pair-reply:1:PORT:...`，
再经可信带外路径交还发起方导入。应答中的端口及发现地址只用于路由，TLS 仍须证明持有邀请中确切证书的私钥。
两次导入均须核对所选设备 id 与证书身份，禁止把另一设备的邀请应用到当前选中的设备。
双方同意、TLS exporter 绑定与持久回执均由同一受限通道机制处理，不另外设计认证算法。

**规则**：

| # | 规则                                                                       |
| - | -------------------------------------------------------------------------- |
| 1 | 所选设备 id、可信带外邀请和实际 TLS 证书身份必须一致，拒绝自配对及替换已有绑定 |
| 2 | 每个共享网络身份只保留一个活动任务，取消中的任务退出前不得复用运行时槽位 |
| 3 | 握手前先持久预留任务代次、双方公开证书与到期时间，此时 binding 为 NULL，不产生业务授权 |
| 4 | TLS 成功后仅能绑定原预留任务，取消、撤销、过期或更换本机登记后，晚到连接均不得重新领取任务 |
| 5 | 双方持久回执完成后，原子提交本机授权代次及列表记录，失败回滚两者，旧 token 不参与授权 |
| 6 | 时限覆盖等待连接、TLS、同意和持久回执；数据库锁等待后及提交前也复核单调时钟预算 |
| 7 | 完成或失败后关闭专用连接，业务另行认证；重启不得从预留或未完成日志推断授权或恢复旧 socket |

> 身份密钥变化、丢失或被撤销后，旧绑定不得继续授权；需要明确重新配对。
> 取消任务阻止尚未提交的授权；已经提交时须如实拒绝取消，解除已生效配对是单独的撤销动作。
> 发现广播只是候选地址提示，不是认证凭据。私钥保存和轮换见 [17 §5](./17-storage-and-config.md)。

任务快照区分 Listening、Connecting、Confirming、Cancelling 及终态，快照本身不能授权。
应答、任务代次、取消与边界测试见[生产首次配对入口](./history/2026-10-26-first-pairing.md)。

### 8.3 意图

```rust
#[derive(Debug, Clone, Serialize, Deserialize)]
#[serde(rename_all = "camelCase", deny_unknown_fields)]
pub struct IntentInvocation {
    pub intent_id: IntentId,
    pub plugin_id: PluginId,
    pub action: String,
    /// ★ boxed `RawValue`：内核可校验形状但不必解析成树（[26 §5.1](./26-core-stability-and-perf.md)）
    pub args: Box<RawValue>,
    /// 发起方给的展示信息（接收方可用来做确认弹窗）
    pub display: Option<IntentDisplay>,
}

#[derive(Debug, Clone, Serialize, Deserialize)]
#[serde(rename_all = "camelCase", deny_unknown_fields)]
pub struct IntentOutcome {
    pub intent_id: IntentId,
    pub ok: bool,
    #[serde(skip_serializing_if = "Option::is_none", default)]
    pub result: Option<Box<RawValue>>,
    #[serde(skip_serializing_if = "Option::is_none", default)]
    pub error: Option<ErrorReport>,
    pub elapsed_ms: u64,
}
```

**接收方的校验顺序**（任一失败即回 `ok: false` + 结构化错误，**不执行**）：

```
1. 信封校验（§5.2）
2. 发起方是已配对设备？（IntentInvoke 不接受未配对来源）
3. 该 plugin_id 在本机已安装且已激活？
4. 该 action 属于该插件声明的 actions？
5. 该插件的 grant 覆盖该 action 所需的能力？
6. 本机 PlatformCapabilities 覆盖所需能力？（否则提前失败，不让插件白跑）
7. 幂等检查（request_id / intent_id）
8. 交给插件 execute()
```

### 8.4 文件传输

```rust
#[derive(Debug, Clone, Serialize, Deserialize)]
#[serde(rename_all = "camelCase", deny_unknown_fields)]
pub struct TransferRequest {
    pub transfer_id: TransferId,
    pub kind: TransferKind,
    pub byte_length: u64,
    pub chunk_size: u32,                // 发送方决定，接收方校验一致性
    pub digest: SriDigest,              // ★ 整包摘要，必填
    #[serde(skip_serializing_if = "Option::is_none", default)]
    pub sync_session_id: Option<SessionId>,
    #[serde(flatten)]
    pub meta: TransferMeta,
}

#[derive(Debug, Clone, Serialize, Deserialize)]
#[serde(rename_all = "camelCase", tag = "kind")]
pub enum TransferMeta {
    #[serde(rename = "plugin-bundle")]
    PluginBundle { plugin_id: PluginId, version: Semver },
    #[serde(rename = "attachment")]
    Attachment { file_name: String, mime_type: Option<String>, context_id: Option<String> },
}

#[derive(Debug, Clone, Serialize, Deserialize)]
#[serde(rename_all = "camelCase", deny_unknown_fields)]
pub struct TransferProgress {
    pub transfer_id: TransferId,
    /// 已确认收到的块序号（可能不连续，用位图表达）
    pub received_bitmap: Vec<u8>,
    pub bytes_received: u64,
}

#[derive(Debug, Clone, Serialize, Deserialize)]
#[serde(rename_all = "camelCase", deny_unknown_fields)]
pub struct TransferComplete {
    pub transfer_id: TransferId,
    pub digest: SriDigest,              // 必须与 request 里的一致
}

#[derive(Debug, Clone, Serialize, Deserialize)]
#[serde(rename_all = "camelCase", deny_unknown_fields)]
pub struct TransferAbort {
    pub transfer_id: TransferId,
    pub reason: TransferAbortReason,
}

#[derive(Debug, Clone, Copy, PartialEq, Eq, Serialize, Deserialize)]
#[serde(rename_all = "snake_case")]
pub enum TransferAbortReason {
    UserCancelled,
    DigestMismatch,
    TooLarge,
    IoError,
    Timeout,
    PeerRejected,
}
```

**控制面走信封，数据面走 `ChunkFrame`**：

```
信封（JSON）                          ChunkFrame（二进制）
  file.transfer.request   ─────▶
                          ◀─────    （接收方 ready，隐含在 chunk 的滑动窗口里）
  file.transfer.chunk × N ─────▶    ★ 每块一个新 request_id 是正常的，
                                      但属于同一个 transfer_id
                          ◀─────    file.transfer.progress（每 N 块或每 500ms）
  file.transfer.complete   ─────▶
                          ◀─────    file.transfer.complete（确认）或 abort
```

**传输的硬性要求**：

| # | 要求                                                              | 为什么                                         |
| - | ----------------------------------------------------------------- | ---------------------------------------------- |
| 1 | `digest` 必填且**必须校验**                                        | 否则"传输成功"是假的                            |
| 2 | 块内容走二进制帧（不经 JSON/base64）                                | base64 有 33% 膨胀 + 编解码开销                  |
| 3 | 滑动窗口（默认 32 块未确认），**不是逐块 await**                     | 逐块串行会让吞吐受 RTT 限制                      |
| 4 | 缺块时接收方回 `progress`（带位图），发送方只重传缺的                  | 支持断点续传的基础                              |
| 5 | 会话有 TTL（`transfer_idle` / `transfer_total`）                    | 半途中断的会话不能永久占内存                      |
| 6 | 接收方的"已收"状态可持久化（大文件续传）                             | 中断后可恢复                                    |
| 7 | 会话级 ACK 与帧级 ACK 是两回事（前者确认"块已入组装缓冲"，后者确认"帧已收到"） | 帧 ACK 不代表数据被接受                          |
| 8 | 完整发送只有在收到同一 `transfer_id` 且摘要匹配的完成确认后才返回成功 | 块写入 socket、部分事件或等待超时均不能证明交付成功 |

两个宿主必须共用完整发送流程，包含提议、受确认约束的发送窗口、完成、最终确认和终止处理；
进度回调报告接收方已确认的块，不能用本机写入数代替接收进度。
未确认块数必须始终受窗口上限限制，收到一份 progress 只释放其中新增确认的槽位；
重复报告不能延长空闲预算，报告不得确认尚未发送的块或撤回已经确认的块。
取消与总时限覆盖等待最终确认，超时和拒绝返回明确错误，失败写入后的链路不得继续重试帧。
协议没有独立的 ready/accept 消息；拒绝通过 `file.transfer.abort` 表达，不能自行假定已获接收同意。
当前实现窗口上限为 4，正常报告不再保存整份缺块列表，实际重传才记录重传事件；
此选择已通过小缓冲区与缺块测试，尚不代表完成吞吐或延迟分位数调优。
实现与验收范围见[共享完整发送流程](./history/2026-10-26-transfer-workflow.md)。

---

## 9. 协议演进


```
protocol 字段是一个 u16，第一版为 1。

同 major（都是 1）内的演进：
  · 新增 Feature（§9.2） → 不改 protocol 号
  · 新增可选字段         → 不改 protocol 号
  · 新增事件名           → 不改 protocol 号
  · 修改已有字段的语义/类型 → ★ 必须 bump protocol 号

protocol 号不同：
  · 握手时发现对端 protocol 高于自己 → 完成握手，但只发「双方都支持」的 Feature
  · 发现对端 protocol 低于自己       → 同上
  · 若是**无法互操作**（例如 major 不同，用高 8 位表示 major）→ 回 HelloAck 之外的
    Error 帧（reason: ProtocolVersionUnsupported），并断开；UI 提示"需要升级对方设备"
```

```rust
#[derive(Debug, Clone, Copy, PartialEq, Eq, Hash, Serialize, Deserialize)]
#[serde(rename_all = "kebab-case")]
#[non_exhaustive]
pub enum Feature {
    /// 二进制块帧（若对端不支持，回落到 base64-in-JSON —— 只为极老的中间版本保留）
    BinaryChunks,
    /// 传输会话进度位图与缺块重传
    TransferResume,
    /// 插件事件的路由元数据
    PluginRouting,
    /// 意图结果的结构化错误
    StructuredIntentErrors,
    /// 集群中继（当前未实现，见 [19 §7](./19-roadmap.md)）
    ClusterRelay,
}

/// 链路级协商结果
#[derive(Debug, Clone)]
pub struct PeerCapabilities {
    pub protocol: u16,
    pub features: HashSet<Feature>,
}

impl PeerCapabilities {
    pub fn supports(&self, f: Feature) -> bool { self.features.contains(&f) }
}
```

> **`#[non_exhaustive]` + 只增不改**：新 Feature 永远是新增而不是修改，所以"双方取交集"这个规则
> 在所有版本组合下都成立。这就是"不需要兼容层"的原因——**协议从一开始就允许共存，而不是靠兼容模式**。

### 9.1 不允许的做法

| ❌ 不允许                                              | 为什么                                                     |
| ------------------------------------------------------ | ---------------------------------------------------------- |
| 在信封里加"版本"字段来分支逻辑                            | 版本在握手协商一次，之后链路的行为是确定的，不需要每帧判断      |
| 类型里定义但从不发送的字段                                | 幽灵字段让后来者无法判断"到底有没有用"                        |
| 靠错误消息字符串做判定                                    | 见 §7.6 的理由                                             |
| 让接收端"猜测"字段含义                                    | 宁可让 `deny_unknown_fields` 拒绝，也不要猜                   |

---

## 10. 发现层

### 10.1 UDP 广播

```
查询（广播）：
  payload = "SYNRA_DISCOVERY" + "\n" + JSON{ protocol, device_id, display_name, port }

  发送：对每个 IPv4 网卡算出的广播地址 + 255.255.255.255
  重发：每 400ms 一次，共 3 次（覆盖一次性丢包），总预算见 §10.3
  接收：忽略 source_device_id == 本机 的报文

响应（单播回查询来源）：
  payload = JSON{
    device_id, display_name, platform, app_version,
    protocol, features, port,                       ← ★ port 总是发
  }
  准入条件：JSON 可解析 且 app == "synra" 且 device_id != 本机
```

**与"裸 magic 字符串"的区别（设计理由）**：

| 设计点                          | 本设计                                       | 为什么                                                    |
| ------------------------------- | -------------------------------------------- | --------------------------------------------------------- |
| 查询报文含 JSON                  | ✅（magic + 换行 + JSON）                     | 一次往返就能拿到对端信息，减少一次探测                        |
| `protocol` / `features` 参与发现  | ✅                                           | 发现阶段就知道能力，握手时不必再协商一遍                       |
| 未知字段                         | **忽略**（不拒绝）                            | 发现报文的演进必须允许"对方多说了一点"                        |

### 10.2 离线公告

```
payload = "SYNRA_OFFLINE" + "\n" + JSON{ device_id, timestamp }
发送：3 次，间隔 150ms（进程退出前尽力而为）
接收：忽略自身；标记该设备离线并发布事件
```

### 10.3 扫描预算

```rust
pub struct ScanBudget {
    /// 总预算（UI 的"扫描中"最多转这么久）
    pub total: Duration,            // 默认 2500ms
    /// UDP/mDNS 收包窗口
    pub passive: Duration,          // 默认 1200ms
    /// TCP 单次探测超时
    pub probe_timeout: Duration,    // 默认 800ms
    /// 探测并发
    pub probe_concurrency: usize,   // 默认 32
}
```

> **探测并发与超时只在一处定义**（`ScanBudget::default()`），
> 不允许各平台各写一个值——那正是"同一份代码在不同端行为不同"的来源。

### 10.4 自我发现抑制（只有一层）

```
本机身份判定（唯一一处）：
    candidate.device_id == self.device_id
 || candidate.ip ∈ self.local_ip_set          （网卡地址集合，含 IPv6）

★ 不允许在各处再各自加一层过滤。
```

> 那种"补丁叠加"的结果是：**没人说得清哪一层在哪生效**，而且漏一层就出现"自己发现自己"。
> 新设计把判据收敛成上面两行，放在 `Discovery::admit()` 里。

### 10.5 可显示性（重要决策）

```
一个候选设备进入 UI 列表的条件（TrustLevel >= Handshaked）：
    完成过一次成功的 Synra 握手
    （即：TCP 建链 + Hello/HelloAck 交换 + challenge 校验通过）

UDP / mDNS 只能提供「候选地址」，永远不能提供「身份」。
```

```rust
#[derive(Debug, Clone, Copy, PartialEq, Eq, PartialOrd, Ord)]
pub enum TrustLevel {
    /// 只在 UDP/mDNS 里听说过。**不进 UI 列表**，仅作为探测目标存在。
    Advertised = 0,
    /// 完成过一次握手。可显示；可发起配对。
    Handshaked = 1,
    /// 已配对。
    Paired = 2,
}
```

> **这条是安全边界**：如果"发现了就能显示"，那么同网段任何人广播一条 UDP 报文
> 就能往用户设备列表里塞条目。必须要求一次真实的协议握手才算"存在"。

---

## 11. 协议实现清单（给实现者的 checklist）

`crates/synra-protocol/`：

> ★ 这一节在实现之前写就，之后长期没有跟着代码走，所以它曾经整体是未勾选状态，而其中大部分工作早已完成。
> 下面每一个勾选都是**这一次重新核对过的**：文件存在、被点名的类型存在、被点名的性质有测试。核对不到的那几条
> 保持未勾选，因为"文件建了"不等于"清单要求的东西在里面"。

- [x] `ids.rs`：6 个 newtype + `TraceId` + `PluginId` / `PluginSlug`，`FromStr` 严格校验
- [x] `frame.rs`：`FrameKind` / `Frame` / `FrameDecoder` / `FrameEncoder`；**超限即错误**（§4.2）
- [x] `envelope.rs`：`Envelope` + `outbound` / `reply_to` / `inbound` 三个入口 + 不变量校验
- [x] `event.rs`：`EventName` + 常量表 + `NO_AUTO_ACK` + `DataPlaneEvent`
- [x] `inbound.rs`：`Inbound` 语义枚举 + 从 `Envelope` 的解析（含 `Unknown` 兜底）
- [x] `handshake.rs`：`Hello` / `HelloAck` / `LinkIntent` + 陈旧配对自愈规则
- [x] `session.rs`：三类会话的状态机类型 + `SessionTimeouts` + 配对/意图/传输三类会话的 payload 类型
      （`PairRequest` / `PairResponse` / `IntentInvocation` / `IntentOutcome` / `IntentDisplay` /
      `TransferRequest` / `TransferComplete` / `TransferAbort`）
- [x] `discovery.rs`：查询/响应/离线公告的编解码 + `ScanBudget`
- [x] `limits.rs`：§7.1 的全部常量
- [x] `feature.rs`：`Feature` / `PeerCapabilities` / 协商 —— `Feature` 与协商在 `handshake.rs`，
      `PeerCapabilities` 为该类型上的 `required` / `offered` 集合与 `Hello` / `HelloAck` 的交集计算
- [x] `error.rs`：`SynraError` + 稳定错误码 + `user_visibility()`
- [x] **无 IO 依赖**：`cargo tree -p synra-protocol` 不出现 `tokio` / `wasmtime` / `rusqlite`
      （实测依赖只有 `bytes` / `compact_str` / `rand_core` / `serde` / `serde_json` / `smallvec` /
      `thiserror` / `time` / `uuid`）

测试（[21](./21-testing-strategy.md)）：

- [x] proptest：帧编解码往返、任意字节流不 panic、信封 `deny_unknown_fields`
- [x] 表驱动：§4.2 的每一行、§5.2 的每一条不变量
- [ ] cargo-fuzz：`FrameDecoder::next_frame`、`Envelope::inbound_ref`/`inbound` —— **未做**。
      现在只有 proptest 覆盖任意字节不 panic。property 测试与 fuzz 覆盖的输入族不同（property 按生成器走，
      fuzz 按覆盖反馈走），所以前者通过**不能**当作后者的结论，这里保持未勾选。
- [x] 帧解码热路径零分配断言 —— `crates/synra-protocol/tests/hot_path_alloc.rs`，用
      `synra-testkit-alloc` 的计数分配器实测：整帧一次读到时走 `next_frame` 的快路径，**0 次分配**；
      `EventName::parse` 也是 0 次。断言是 0，不是"实测值"。
      ★ 这条之所以值得记一笔，是因为它先给了两个**错的**答案：计数器没有装 `#[global_allocator]`，
      所有断言都读到 0，看着全绿；装上之后读出"每帧 1 次分配"，差点被当成真实结论写进
      `lib.rs` 的设计原则。真相是那 1 次是首次触碰的一次性成本，再加一轮预热就归零。
      三个错误结论有同一个来源 —— **只测一次、只测自己看得见的部分**。所以文件里现在有
      `the_counter_counts_an_allocation_that_definitely_happens` 这一条：它断言一个**确定会发生**的分配
      被计数到了。没有它，一个恒读 0 的计数器能让上面每一条都"通过"。
- [x] capability 匹配零分配 —— **已实现，已修正**。`crates/synra-core/tests/hot_paths.rs` 里本来有
      9 条零分配断言，全部通过、全部无效：它同样没有装 `#[global_allocator]`，读的一直是没装上的
      计数器。装上之后真相是：帧解码 0、capability 匹配 2、路由 2、门禁 3、查不到 5。
      capability 匹配是唯一值得马上改的（它跑在系统每一次权限检查上），已改成 `SmallVec`，
      **2 → 0**；门禁顺带 3 → 1。
      剩下三条（路由 2、门禁 1、查不到 5）**断言的是实测值而不是 0**，并且测试名已经改成
      `*_stays_within_its_budget` —— 名字不能说零分配而断言说不是。去掉它们要动 `RouteOutcome`
      的公开 payload 和静态表的预解析，属于设计变更，另开一轮做，不在这里偷偷改到能过为止。
- [ ] 双内核端到端（`KernelPair`）：全部会话走通、断链重连、幂等重试、传输校验 ——
      **★ harness 是存在的，而且很完整；缺的是"用它跑真内核"。**
      先纠正一处：本轮我一度以为 `KernelPair` 不存在（在 `src/` 下按文件找 `pair.rs`，没找到）。
      它在 `crates/synra-testkit/src/pair/`，是一个目录（`mod.rs`/`harness.rs`/`expectation.rs`/
      `helpers.rs`/`record.rs`/`tests.rs`）。**这是我看错了，不是代码的问题。**
      已有的能力恰好就是这一条要的：`KernelPair<K>` 把两个内核用一个 `MemoryTransportLink` 接起来、
      共用一块 `MockClock`（`advance` 推时间，不用 sleep），并且带
      **断链**（`cut()`）、**故障注入**（`inject(Side, Fault)`）、
      **重试计数**（`attempts`）、**丢弃计数**（`dropped`）、
      **有序期望**（`Expectation` + `check`，缺失/顺序/事件不匹配各有区分）。
      `pair/tests.rs` 22 条测试把这些都覆盖了。
      ★ **但全仓 `KernelPair`/`PairedKernel`/`MemoryTransport` 在 `synra-testkit` 之外 0 处使用** ——
      那 22 条测试跑的是它自己的 `FakeKernel`。也就是说：**这个夹具测的是夹具自己**。
      它证明了 harness 能用，没有证明**真内核**经过它能把三类会话走通、能断链重连、能幂等重试、
      能校验传输。
      ★ **为什么不用的原因不是"没人写测试"，而是缺一个缝，值得写清楚：**
      * `PeerLink` 持有一个**具体的 `TcpStream`**（`crates/synra-core/src/link.rs:128`、`:155`），
        它自己从字节流里解码帧（内部持 `FrameDecoder`）。
      * `MemoryTransport` 搬运的是**已经解码好的 `Frame`**（`transport.rs:228` `send(frame)`、
        `:237` `receive() -> Option<Frame>`），不是字节。
      * 全仓**没有任何 `Transport` / `Stream` trait**（`grep 'trait Transport|trait Stream'` 0 命中）。
      两者**在类型上就不兼容**：一个在帧层，一个在字节层，而链路层把 socket 写死成了 `TcpStream`。
      所以这一条的前置不是"写端到端测试"，而是**先决定那个缝长什么样**：
      要么给 `PeerLink` 一个字节流抽象（`TcpStream` 实现它，内存链路也实现它），
      要么把 `MemoryTransport` 改成面向字节。这是一个设计决定，两边的现有测试都在守着自己那一侧。
      所以这一条保持未勾选，措辞也对齐一下：
      它不是"要造一个 `KernelPair`"，而是"**要先长出那条缝，再拿现有的 `KernelPair` 去跑 `Kernel`**"。
      `synra-testkit` 的模块注释（`src/lib.rs:14-16`）其实已经写明了它的用途 ——
      "so two kernels can be linked in one process with no sockets. This is what makes the end-to-end tests of §4.3
      possible without a network" —— 而 §4.3 的那些端到端测试还没有写，因为**没有 socket 可换**。
      ★ **而第 83–84 轮各撞到了一次"没有这条缝"的代价，两件事都值得记在这里：**
      * **同一件事被实现了两遍，然后漂移了。** `synra-core/src/peer/link.rs` 从 `flags.timeout` 设了**写**超时，
        而同 crate 的 `PeerLink`（就是上面那个 `link.rs`）**只有读超时**。于是一个不再读的对端
        不产生任何错误 —— 调用线程**永远不返回**，TCP 也不会把满窗口报成错误。
        漂移之所以看不见，正是因为**没有一份共同契约去跑这两个实现**。
      * **`synra-net` 整个 crate 没有消费者。** `Backoff` / `Liveness` / `WriteQueue`
        在两个工作区里 0 处使用（`Backoff` 的命中全是 `synra-core` 自己的 supervisor 枚举变体），
        因为字节层的链路要排队就得先有**写入线程** —— 又是一个缝的决定。
        代价是 `AckPolicy` 无处可问：全仓搜 `AckPolicy`，`event.rs` 之外只有一条文档注释。
      ★ **而第 86 轮把这条缝长出来了**：`synra_core::link::LinkStream`
      （`Read + Write + Send + Debug`）加 `link::memory::MemoryStream`，`PeerLink` 改持 `Box<dyn LinkStream>`。
      ★ **而它为什么能在不改调用方的前提下加上**：`new(stream: impl LinkStream + 'static)`，
      而 `TcpStream` 实现了它 —— 所以守护进程、listener、测试都不必改。
      ★ **而方法返回的是 `LinkError` 而不是 `std::io::Error`**，这是为了绕开 `api-guard`
      的规则（对外部 crate 错误类型的承诺就是对它版本的承诺）。
      ★ 三个行为是故意做成像 socket 的，因为不会阻塞的流测不出 socket 的问题：
      **有界缓冲**（满了就等待然后失败）、**超时返回 `WouldBlock`**（于是 `is_a_timeout` 把它认成
      “安静”而不是“断开”）、**一端 drop 后另一端读到 EOF**。
      ★ **而它自己就是证据**：两个 `PeerLink` 在内存上交换了一个真的 envelope（带帧、
      带 target/from 校验）。
      ★ **而第 87 轮让它跑起了真工作**：`session/tests.rs` 的 `connected()` 从
      “开 `TcpListener`、connect、accept”改成 `MemoryStream::pair()`，
      **13 条会话测试全部跑在内存上** —— 提议传输、窗口内写块、读报告、确认，每一步都不再需要 socket。
      ★ **而耗时没有变**：两边都是 6.41s（那一位数来自一条故意等超时的用例，`HELLO_TIMEOUT` 那条），
      所以这次换接不是用性能换来的。
      下一步才是把它接到 `synra-testkit` 的两个 `Kernel` 上。
      * 两条合起来说明：**这条缝不只是"能不能换成内存链路"**，它同时决定了
        ①同一套失败语义能不能只写一遍，②`synra-net` 里那些已经写好、测好的决策能不能被用上。
        在它长出来之前，每修一处链路问题都是在两个实现之间做人工同步 —— 第 84 轮就是第二次。
