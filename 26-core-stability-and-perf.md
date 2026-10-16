# 26 · 核心稳定性、封装与性能

> 本文把三条贯穿性要求落成**可检查的规则**：
> **底层核心要稳**（改动代价最小）、**封装要精致**（边界小且不可绕过）、**性能内建于设计**（不是事后优化）。
>
> 它约束**每一行内核代码**。前置：[03 架构基线](./03-architecture-baseline.md)（分层）、[11 设计原则](./11-design-principles.md)（必须避免什么）。
> 相关：[06 §7](./06-workspace-layout.md)（lint 与结构守卫）、[21 §8](./21-testing-strategy.md)（性能门禁）。

---

## 1. 为什么单独一篇

"核心要稳、封装要精致、性能要考虑"这三句话如果只是原则，落地时每个人理解不同。
所以本文把每条**变成一条规则 + 一个检查 + 一个反例**。

三者的关系：

```
        ┌─────────────────────────────────────────┐
        │ 底层核心要稳                              │
        │  = 最小依赖 + 最窄义务 + 显式稳定性等级      │
        └────────────────┬────────────────────────┘
                         │ 稳定意味着"不能为了性能改 API"
                         ▼
        ┌─────────────────────────────────────────┐
        │ 性能必须内建于设计                         │
        │  = 关键路径的类型选择在写代码前就定死          │
        └────────────────┬────────────────────────┘
                         │ 改动被限制在内核内部
                         ▼
        ┌─────────────────────────────────────────┐
        │ 封装要精致                                │
        │  = 内部类型不泄漏 → 优化不构成破坏性变更       │
        └─────────────────────────────────────────┘
```

**关键因果**：如果内部类型泄漏到公开 API（例如 `pub fn grants(&self) -> &HashMap<..>`），
那么"把 HashMap 换成更快的结构"就是**破坏性变更**。于是"核心要稳"与"性能要好"直接冲突。
**封装就是解开这个冲突的手段** —— 这也是本文最重要的结论。

---

## 2. 底层核心要稳

### 2.1 三条硬规则

| # | 规则                                                                   | 检查 |
| - | ---------------------------------------------------------------------- | ---- |
| S1 | **依赖只向下**，且底层 crate 的依赖集合尽量小（[03 §2](./03-architecture-baseline.md)） | `cargo tree -p synra-protocol` 里不出现 `tokio`/`wasmtime`/`rusqlite`；CI 守卫 |
| S2 | **每个 crate 的稳定性等级显式声明**（`STABILITY.md`）                        | [06 §7.2](./06-workspace-layout.md) |
| S3 | **算法/常量只有一处定义**（[11 A6](./11-design-principles.md)）               | 全仓 grep 字面量 |

`STABILITY.md` 的三层（沿用 `peon-burrow` 的做法）：

| 等级       | crate                                                                 | 承诺                                         |
| ---------- | --------------------------------------------------------------------- | -------------------------------------------- |
| **稳定**    | `synra-protocol`、`synra-capability`、`synra-plugin-api`                 | 遵守 SemVer；破坏性变更需 major + ADR + 迁移说明 |
| **半稳定**  | `synra-net`、`synra-store`、`synra-core`、`synra-plugin-host`            | 遵守 SemVer，但允许 minor 里扩展 trait（带默认方法） |
| **内部**    | `synra-observe`、`synra-platform`、`synra-testkit`、`apps/*`、`tools/*`   | 无承诺，随时可变                                |

> **为什么 `synra-core` 只是"半稳定"**：它是编排层，业务规则一定会长。
> 但它的**接口**（`Kernel` 的公开方法）要稳，**内部结构**（聚合根怎么组合）可以动。
> 这个区分必须写清楚，否则要么过度冻结（无法演进），要么随意改动（下游被反复打断）。

### 2.2 稳定层的"不能做什么"

| ❌ 稳定层不能                                   | 为什么                                            |
| ---------------------------------------------- | ------------------------------------------------- |
| 暴露具体容器类型（`HashMap`/`Vec`）在公开签名里    | 换实现就是破坏性变更（§3.2）                        |
| 暴露内部 trait 的具体实现类型                     | 同上                                              |
| 用 `pub` 字段                                   | 加一个字段就是破坏性变更；无法保持不变量             |
| 在公开 enum 上让下游必须穷尽 match（不加 `#[non_exhaustive]`） | 加一个变体就是破坏性变更              |
| 让公开函数签名里出现 `impl Trait` 之外的闭包返回类型 | 无法演进                                          |

### 2.3 `#[non_exhaustive]` 的分界（容易搞反）

| 类型位置                  | 要不要 `#[non_exhaustive]` | 理由                                              |
| ------------------------- | ------------------------- | ------------------------------------------------- |
| **面向插件/第三方的 enum**   | ✅ **要**                  | 外部 crate 会 match 它；加了才能加变体               |
| **内核内部的 enum**         | ❌ **不要**                | 不加时"新增变体"会在**编译期**指出所有需要处理的地方 —— 这是安全网 |
| 面向外部的 struct          | ✅ 要（并配构造函数）        | 外部不能字面量构造                                 |
| 内部 struct               | ❌ 不要                    | `..Default::default()` 会隐藏"忘了填字段"           |

```rust
// ✅ 面向插件：加了 non_exhaustive，将来能加变体
#[non_exhaustive]
pub enum BuiltinEvent {
    TransferCompleted(TransferDone),
    DeviceChanged { devices: Vec<DeviceSnapshot> },
    // 将来加 HostForeground 不破坏插件编译
}

// ✅ 内核内部：不加，于是"新增一个 Inbound 变体"会让所有 match 报错，
//    强迫实现者处理它 —— 这正是我们想要的
pub enum Inbound {
    MemberOffline { device: DeviceId },
    PairRequest { /* … */ },
    // 新增变体 → 编译错误 → 必须处理
}
```

> ⚠️ **搞反的代价**：给内部 enum 加 `#[non_exhaustive]` 会强迫内部代码写 `_ => {}`，
> 于是**新增状态时被静默忽略** —— 这是最难发现的缺陷之一（正是 [11 C2](./11-design-principles.md) 说的"状态机空洞"）。

---

## 3. 封装要精致

### 3.1 "精致"的定义

不是"文件多、抽象多"，而是**边界小、不可绕过、内部可自由改动**：

| 特征              | 表现                                                        |
| ----------------- | ----------------------------------------------------------- |
| **边界小**         | 公开项最少；能在内部的东西绝不 `pub`                            |
| **不可绕过**       | 唯一的入口强制不变量（`Envelope::outbound` / `gate_host_fn!`）  |
| **内部可动**       | 换实现不改公开签名 —— 于是性能优化不构成破坏性变更               |
| **错误自解释**     | 类型里能看出"不这么做会怎样"（`Link<Connecting>` 没有 `send`）  |

### 3.2 四条硬规则

| # | 规则                                                                | 反例（不要这样）                                   |
| - | ------------------------------------------------------------------- | ------------------------------------------------ |
| E1 | **公开签名里不出现具体容器**：用 `&[T]` / `&str` / 迭代器 / 自定义只读类型 | `pub fn grants(&self) -> &HashSet<Capability>`    |
| E2 | **不导出内部表示**：`pub struct` 的字段全私有 + 构造函数 + 只读访问器      | `pub struct Grants { pub exact: HashSet<_> }`      |
| E3 | **不变量只有一个构造入口**                                            | 允许调用方 `Envelope { .. }` 字面量构造             |
| E4 | **`impl Trait` 返回**：不泄漏具体类型                                 | `pub fn peers(&self) -> impl Iterator<Item = &Peer> + '_` |

```rust
// ❌ 泄漏内部表示：换容器 = 破坏性变更
pub struct CapabilitySet { pub exact: HashSet<Capability>, pub wild: Vec<Capability> }

// ✅ 封装：内部随便换（HashSet → 有序 Vec → bitset），签名不变
pub struct CapabilitySet {
    exact: HashSet<Capability>,     // 私有
    wild: SmallVec<[Capability; 4]>,
}

impl CapabilitySet {
    pub fn contains(&self, cap: &Capability) -> bool;     // 只读接口
    pub fn iter(&self) -> impl Iterator<Item = &Capability> + '_;   // E4
    pub fn len(&self) -> usize;
}
```

> **E1 的实际收益**：`CapabilitySet` 的内部结构**已经改过一次**（见 §5.2：从"线性扫 wild 列表"
> 改成"按域索引 + 精确/通配分离"）。如果当初 `pub` 了 `HashSet`，这次优化就是破坏性变更。

### 3.3 错误也必须封装

| ❌ 不要                                        | ✅ 要                                              |
| --------------------------------------------- | -------------------------------------------------- |
| `pub enum Error { Io(std::io::Error), Json(serde_json::Error) }` | 内部错误类型私有；对外的 `SynraError` 只有**稳定的错误码** |
| 让调用方 match 具体 IO 错误                       | `DisconnectClass` + `UserVisibility`（[04 §7.6](./04-wire-protocol.md)） |

否则"换掉 `serde_json` 换 `sonic-rs`"（性能优化）会变成破坏性变更。

---

## 4. 性能内建于设计

### 4.1 热路径清单（先定义"哪里算热"）

只有这些地方值得为性能牺牲可读性。**其它地方一律以清晰为先。**

| # | 热路径                              | 频率量级                        | 预算 |
| - | ----------------------------------- | ------------------------------- | ---- |
| 1 | 帧编解码（信封 / 二进制块）            | 每帧（**每秒几千～几万**）          | < 5 µs / 1 KiB；< 20 µs / 256 KiB |
| 2 | 信封解析与事件路由                     | 每帧                            | 路由查表 O(1) |
| 3 | capability 匹配                      | 每次 host call                  | < 200 ns（**无分配**） |
| 4 | host function 调用（含闸门 + 审计）     | 每次插件调用                     | < 20 µs |
| 5 | 事件入队/出队                        | 每事件                          | O(1) |
| 6 | 传输块路径                            | 每块（**吞吐关键**）              | 零拷贝，> 100 MB/s |
| 7 | 逻辑时钟 / 状态转移                    | 每操作                          | 无分配 |
| 8 | 插件实例获取（池命中）                  | 每次插件调用                     | < 5 µs |

**不在热路径上**的：插件安装、配对、UI 快照、配置读写、诊断。
这些**不要**为了性能写复杂代码（[11 D3](./11-design-principles.md)：清晰优先）。

### 4.2 五条硬规则

| # | 规则                                                             | 检查方式 |
| - | ---------------------------------------------------------------- | -------- |
| P1 | **热路径不分配**（除显式声明的必要分配）                              | `dhat` 或自建分配计数器断言 |
| P2 | **能借用就不拥有**：`&str` / `&[u8]` / `Cow<str>` / `RawValue`        | clippy `needless_pass_by_value`（已 deny） |
| P3 | **能预计算就不每次算**：路由表、解析后的 capability 集、interned 事件 id | 代码审查 + 基准 |
| P4 | **能零拷贝就不复制**：`bytes::Bytes` / `BytesMut` / `RawValue`        | 基准 + 审查 |
| P5 | **每个队列有界，每个 `join` 有预算**（[11 C3](./11-design-principles.md)） | 构造点必须有容量参数 |

### 4.3 数据结构的选型表（**写代码前就定死**）

| 场景                    | 选                                                                | 为什么不选替代 |
| ----------------------- | ----------------------------------------------------------------- | -------------- |
| 帧缓冲 / 二进制块         | **`bytes::Bytes`（共享切片）· `bytes::BytesMut`（写侧）**             | `Vec<u8>` clone 是深拷贝；`Bytes` 引用计数 O(1) |
| 事件名 / 标识符字符串      | **`CompactStr`**（≤ 24 字节内联，SSO）                              | `String` 每帧一次堆分配；`Arc<str>` 原子操作更贵 |
| 短小列表（≤ 8 项）        | **`SmallVec<[T; N]>`**                                             | `Vec` 有堆分配；`Box<[T]>` 不可增 |
| 只读的固定集合            | **`Box<[T]>`** 或 `&[T]`                                           | `Vec` 多了 capacity 字段，语义也不对 |
| 不受信任键的 map          | **`std::collections::HashMap`（SipHash）** 或 `foldhash`             | `FxHash` 对**攻击者可控的键**有碰撞 DoS 风险 |
| 元组索引的小状态          | 具名 struct                                                       | 元组可读性差、易错位 |
| 位图（传输进度）           | **`bitvec`** 或自建 `Vec<u64>` + 位运算                           | `Vec<u8>` 浪费 8×，且 `received_bitmap: Vec<u8>` 语义含糊 |
| 热路径 JSON              | **`serde_json::value::RawValue`**（透传）+ 需要时才解析              | `serde_json::Value` 会建整棵树（§5.1） |
| 需要枚举小集合成员         | 枚举 + `bitflags` / `u32` 位域                                     | `HashSet<Feature>` 在热路径上是浪费 |

> **为什么"不受信任键"这一行很重要**：`Feature` / `Capability` 这类集合的键来自**网络或插件**
> （攻击者可控）。用无种子哈希（`FxHash`/`ahash` 的固定种子）会让攻击者构造大量碰撞，
> 把 O(1) 查找变成 O(n) —— 这是一个真实的 DoS 面。
> **热路径上用快速哈希，但键必须可信；键不可信时用 SipHash 或带随机种子的实现。**

### 4.4 依赖选型（只加真正需要的）

| crate                | 用途                              | 为什么是它 |
| -------------------- | --------------------------------- | ---------- |
| `bytes` `1.x`         | 帧缓冲、二进制块                    | 引用计数切片，零拷贝的标准解 |
| `compact_str` `0.9`   | 事件名、ID、短字符串                 | SSO，避免每帧堆分配 |
| `smallvec` `1.x`      | 短列表（候选插件、通配能力）           | 避免堆分配 |
| `bitvec` `1.x`        | 传输进度位图                        | 语义明确 + 支持位运算 |
| `serde_json`（`raw_value` feature） | payload 透传            | `RawValue` 零拷贝 |
| `foldhash` `0.1`      | 可信键的热路径哈希                   | 比 SipHash 快，且明确"仅用于可信键" |
| `dashmap` `6.x`       | 并发只读为主的表（设备表）             | 免全局锁；但**只用于读多写少** |

> ⚠️ **每加一个依赖要在 PR 里说明"为什么 std 不行"**。依赖是长期的维护成本，
> 也是"核心要稳"的敌人（每个依赖都在动）。

---

## 5. 已经识别出的设计级修正（本文的直接产物）

以下五处是审计现有设计时发现的**真实性能/封装问题**。它们必须在写代码前定下来。

### 5.1 `Envelope.payload` 不该是 `serde_json::Value`

| 现状                            | 问题                                                              |
| ------------------------------- | ----------------------------------------------------------------- |
| `payload: Option<serde_json::Value>` | ① 解析时建整棵 `Value` 树（每个字段一次分配 + `String` 分配）；② 转发时要重新序列化 |

**修正**：改用 **`Box<serde_json::value::RawValue>`**（透传的原始 JSON）。

```rust
pub struct Envelope {
    // …
    /// 原始 JSON 文本（已通过语法校验，未解析成树）
    ///
    /// ★ 为什么用 RawValue：内核在大多数情况下**不需要理解 payload**——
    /// 它只需要校验信封、路由、转发。用 RawValue 可以：
    ///   ① 解析时零分配（`RawValue::from_string` 只做校验）
    ///   ② 转发时直接拼接，不重新序列化
    ///   ③ 只有真正要读字段的一方（插件）才解析一次
    #[serde(skip_serializing_if = "Option::is_none", default)]
    pub payload: Option<Box<RawValue>>,
}
```

并且**尽量借用**：`&RawValue` 可以直接指向帧缓冲里的那片 ASCII 字节（验证它是合法 JSON 之后）：

```rust
/// 从帧缓冲零拷贝解析。生命周期绑定到缓冲，所以缓冲不能在处理完之前回收。
pub fn inbound_ref<'a>(raw: &'a RawValue, local: DeviceId) -> Result<EnvelopeRef<'a>, EnvelopeError>;
```

**带来的连锁好处**：

| 位置                     | 变化                                                    |
| ------------------------ | ------------------------------------------------------- |
| 事件路由                  | 不解 JSON 就能路由（事件名与信封字段是扁平结构）             |
| 转发（A → B → C 或 host → UI） | 直接把 `RawValue` 塞进新信封，`bytes` 层面就是拼接           |
| 插件边界                  | 插件 SDK 在跨 Wasm 时做**一次**解析（它必须解析才能用）        |
| UI 边界（IPC）             | 作为**字符串**传给前端，前端 `JSON.parse` 一次               |

> **谨慎点**：`RawValue` 只保证是**合法 JSON 语法**，不保证符合你的 schema。
> 所以 schema 校验发生在**消费方**（插件 SDK 或内核的强类型基础事件转换），
> 且错误要能定位到"哪个字段不符合哪个 schema"（[11 A6](./11-design-principles.md)：不能有幽灵字段，也不能有幽灵校验）。

### 5.2 `CapabilitySet::contains` 必须无分配且不受顺序影响

| 现状                                                    | 问题 |
| ------------------------------------------------------- | ---- |
| `exact: HashSet` → 命中；否则**线性扫** `wild: Vec`         | 通配多时变慢；顺序可能影响结果（先匹配到哪个粗粒度通配） |
| `HashSet` 的键来自**插件 manifest**（半可信）                | 见 §4.3 的碰撞 DoS 考量 |

**修正**：

```rust
pub struct CapabilitySet {
    /// 精确能力，按域分桶（domain 数量很少 → SmallVec 优于 HashMap）
    exact: SmallVec<[(DomainId, SmallVec<[Capability; 4]>); 8]>,
    /// 通配能力，**按 specificity 预排序**（更具体的在前）
    wild: SmallVec<[Capability; 8]>,
    /// 位域快照：用于"这个插件有没有某域能力"的快速否定（O(1)）
    domain_mask: u32,
}

impl CapabilitySet {
    /// ★ 无分配；语义不依赖插入顺序或匹配顺序
    pub fn contains(&self, cap: &Capability) -> bool {
        if !self.domain_mask.has(cap.domain()) {
            return false;                       // 最常见的情况：快速否定
        }
        if self.exact_for(cap.domain()).contains(cap) { return true; }
        self.wild.iter().any(|w| w.covers(cap))
    }
}
```

**规则**：**`contains` 的结果不许依赖匹配顺序**（更具体的通配存在时，结果必须与只有它时相同）。
这一条要有 proptest 覆盖（打乱插入顺序 → 结果不变）。

### 5.3 事件路由要预计算，不要每帧靠字符串

| 现状                                                    | 问题 |
| ------------------------------------------------------- | ---- |
| `EventName(CompactString)`，路由时 match 字符串             | 每帧多次字符串比较；通配匹配更贵 |
| `Inbound::Plugin { slug, logical: String, payload }`      | `logical` 每帧分配 |

**修正**：

```rust
/// ★ 事件 id：装载期把"已知事件名"与"通配规则"编号
#[derive(Debug, Clone, Copy, PartialEq, Eq, Hash)]
pub struct EventId(u16);

pub struct EventRegistry {
    /// 名字 → id（只在装载/收到新名字时查询一次）
    by_name: HashMap<EventName, EventId, foldhash::fast::RandomState>,
    /// ★ 路由表：id → 订阅它的插件
    routes: Vec<SmallVec<[PluginHandle; 2]>>,
    /// 通配规则（少数），命中后回填 by_name 缓存
    wildcards: SmallVec<[WildcardRoute; 8]>,
    /// 预热缓存（LRU 上限，防止插件伪造海量事件名导致内存增长）
    resolved: LruCache<EventName, EventId>,
}
```

| 收益                          | 说明                                                    |
| ----------------------------- | ------------------------------------------------------- |
| 热路径 O(1) 查表               | 第一次见到的名字查一次通配，之后缓存                        |
| `logical` 不再分配             | 用 `&str` 借用帧缓冲（与 §5.1 的 `RawValue` 同一思路）      |
| 通配数量有上限                  | `wildcards` 有界 → 防止 manifest 用海量通配拖慢路由         |
| 未知事件名不会无限增长内存        | `resolved` 是**有界** LRU（[11 C3](./11-design-principles.md)） |

> **缓存的键仍然不可信**（对端可发任意事件名）→ 所以用**随机种子**的哈希 + **有界**缓存。
> 这是"快速哈希只用于可信键"规则的具体应用（§4.3）。

### 5.4 帧解码的缓冲复用策略

| 现状                                        | 问题 |
| ------------------------------------------- | ---- |
| `FrameDecoder { buf: BytesMut }`，`decode(&mut self, input: &[u8])` | 传入切片 → 必须先 `extend_from_slice`（复制一次）；每帧解析后 `buf` 如何回收没有规定 |

**修正**：明确"锚定 + 冻结 + 回收"的策略。

```rust
impl FrameDecoder {
    /// ★ 传入 Bytes（从 socket 读到的共享切片），避免复制
    pub fn push(&mut self, input: Bytes) -> Result<(), FrameError>;

    /// 返回的 Fragment 借用缓冲（零拷贝）。缓冲在 Fragment 存活期间不被复用。
    pub fn next_frame(&mut self) -> Result<Option<Fragment<'_>>, FrameError>;

    /// 缓冲已完全消费 → 回收复用（不清零，只重置长度）
    pub fn recycle(&mut self);
}
```

| 规则                                                     | 为什么 |
| -------------------------------------------------------- | ------ |
| 每连接一个解码器，缓冲**随连接结束释放**                       | 不共享，避免锁与串味 |
| 缓冲**上限**（`MAX_FRAME_LEN` + 头）· 超限即断链              | 否则对端可以撑爆内存（[11 A1](./11-design-principles.md)） |
| 用 `BytesMut::split_to` / `freeze` 而不是 `to_vec`           | 零拷贝 |
| 缓冲超过阈值（如 256 KiB）后**重新分配**而不是无限增长           | 避免长期持有大缓冲 |

**关键性能收益**：一个 3 MB 的文件传输，如果每块都 `to_vec`，就是 3 MB 的额外复制与分配；
用 `Bytes` 共享切片则是几十次引用计数操作。

### 5.5 传输块的"零拷贝"必须贯穿

| 环节                          | 要求                                                    |
| ----------------------------- | ------------------------------------------------------- |
| socket 读                      | `Bytes`（不是 `Vec<u8>`）—— 由 tokio 的 `read_buf` 提供    |
| 解码                           | `Fragment` 借用缓冲（§5.4），不复制                       |
| 路由 / 闸门                     | 只读元数据，不触碰 `data`                                 |
| 写盘                           | `tokio::fs::File::write_all(&fragment.data)` —— 无中间态  |
| 摘要                           | `hasher.update(&fragment.data)` —— 流式，不缓冲整包         |
| 跨设备转发（A → B → C，未实现）    | **必须**用 `Bytes` 直接转发，不许 `to_vec`                 |

> 审计要求：`crates/synra-net` 与 `synra-protocol` 的传输路径里
> **不允许出现 `to_vec()` / `Vec::from(&[u8])` / `.clone()` 作用于块数据**（CI grep 守卫）。

---

## 6. 把规则变成检查

| 规则组      | 自动化                                                        | 人工 |
| ----------- | ------------------------------------------------------------- | ---- |
| S（稳定性）  | `cargo-deny`（依赖方向）、`STABILITY.md` 与 crate 列表一致性检查   | ADR 评审 |
| E（封装）    | `cargo public-api` 快照 diff（公开面变化必须显式批准）、`missing_docs` deny | PR 审查 |
| P（性能）    | 基准门禁（[21 §8](./21-testing-strategy.md)）+ **分配计数断言** + grep 守卫 | 热路径审查 |

### 6.1 `cargo public-api` 快照（这是"核心要稳"最直接的守卫）

```bash
# CI：每 crate 一份公开 API 快照，变化必须在 PR 里显式更新
cargo public-api --simplified -p synra-protocol > api/synra-protocol.txt
git diff --exit-code api/ || {
  echo "公开 API 变化。若是故意的，请更新 api/ 快照并在 PR 描述里说明理由。"
  exit 1
}
```

**为什么需要它**：SemVer 承诺靠人记是不可靠的。
它会抓到三类意外破坏：① 公开签名里混进了具体类型；② 忘了 `pub(crate)`；③ 无意间扩大公开面。

### 6.2 分配计数断言（性能回归的早期警报）

```rust
#[test]
fn hot_paths_do_not_allocate() {
    let counter = AllocCounter::install();
    // ① 帧解码（信封）
    decoder.push(Bytes::from_static(envelope_frame_bytes))?;
    let _ = decoder.next_frame()?;
    assert_eq!(counter.allocations(), 0, "frame decode must not allocate");
    // ② capability 匹配
    counter.reset();
    let _ = grants.contains(&cap);
    assert_eq!(counter.allocations(), 0, "capability match must not allocate");
}
```

比基准测试更早报警（基准要跑到 CI 的统计门禁才失败，分配计数是确定的）。

### 6.3 grep 守卫

| 禁止的写法                                          | 范围 | 理由 |
| -------------------------------------------------- | ---- | ---- |
| `to_vec()` / `Vec::from(` 作用于块数据                | 传输路径 | 破坏零拷贝（§5.5） |
| `pub HashMap` / `pub HashSet` / `pub Vec` 在公开签名里 | 稳定层 crate | 违反 E1 |
| `pub` 字段（除 POD 型极小类型且经 ADR 批准）              | 稳定层 crate | 违反 E2 |
| `std::sync::Mutex` 跨 `await`                        | 全部 | [11 C4](./11-design-principles.md) |
| `rayon` / `std::thread::spawn` 在 `crates/*`           | 内核 | 并发模型统一走 tokio + supervisor |

---

> ★ **而第 90 轮把同一个问题问到了 `tools/guards` 自己身上**：
> 它有 **7 个守卫、一个测试都没有** —— 也就是说没有人证明过它们**能失败**。
> 而这个文件里已经出过一个正好发生过的例子：第 5 个守卫曾经扫描
> `ungated_host_fn!`（**全仓不存在的宏**）并且放行一个**不存在的文件**，于是数月里每次都报 `ok`。
> 它自己的注释就是结论：*"a missing check is a gap someone may notice,
> while a green check is evidence that the gap has been closed."*
> ★ 现在每个守卫都有一条**让它失败**的测试，在一棵合成的树上跑：
> GUI 依赖、协议层的运行时、超长的 `main.rs`（正好 61 行，边界两侧都断言）、
> `paths.rs` 之外的 `directories::`、未闸门的动作、**表不存在**、以及**表解析为空**。
> ★ 而为了能测试依赖类守卫（它们跑 `cargo tree`，需要网络），匹配那一半被抽成了
> `banned_lines()` —— **因为那才是能静默出错的一半**：一个比较永远不匹配的守卫，
> 看起来和一个没有违规的守卫完全一样。
> ★ **而其中两个还各做了取反对照**：把窄主比较改成一个永远达不到的上限
> ⇒ 它的测试红；把门控守卫的“哪些族可以不闸门”改成 `if false`（**它取代的那个守卫的形状**）⇒ 红。
> ★ **而第 89 轮发现过滤器读的还不是“规则”，而是“某个例子”**：
> 它读的是 `holds io::Error`（外部错误类型规则的**一个实例**）和 `exposes BTreeMap`
> （具体容器规则的**一个实例**）。所以一个拿 `HashMap` 当具体容器、或 `rusqlite::Error`
> 当外部错误类型的签名，**会被报告然后被忽略** —— 名字的过滤器只能读到别人想到的那一个。
> ★ 现在过滤器用一份 `RULE_FRAGMENTS`（四个消息片段，一条规则一个），
> 而 `the_filter_reads_every_rule_the_tool_can_report` 拿它和**工具自己的 `#[error(...)]` 模板**双向对账**：
> 每个模板都必须被某个片段覆盖，每个片段也必须覆盖某个模板。
> 前者捉“工具能报而测试不读”，后者捉“片段陈旧到什么都读不到”。
> ★ 两个方向各做了一次取反对照：删掉一个片段 ⇒ 前者报错；
> 把片段换成一句没人打印的话 ⇒ 后者报错。
> ★ **而第 88 轮发现了一个比“缺少守卫”更糟的形状**：`api-guard` 确实报了第二条规则
> （稳定层 crate 的公开 enum 缺 `#[non_exhaustive]`），而**没有任何测试读它**：
> `stability.rs` 的过滤器只看 `holds io::Error` / `exposes BTreeMap`。
> 于是那一条规则被报了 **5 次**，而其中 **4 次是本会话自己加的**
> （`UpdatePolicy`、`ChangeSize`、`Cached`、`Origin`）。**一个没人读的守卫产出的就是这个**：
> 你写了违反规则的代码，而它照样绿。现在过滤器读两条规则，五个 enum 全部加上了属性，
> 而加上它们之后守护进程的 `match` 确实开始报错 —— 那正是这个属性在做事。
> ★ **而那两条已记录的偏差也换了锤法**：它们原本按 `path:line` 钉住，
> 而行号在**四轮里移了四次**（467 → 520 → 533 → 590），每次都是因为在一个无关的私有函数
> **上面加了东西**。现在改成按**工具输出的条目文本**匹配 —— 那才是规则真正关心的东西，而它不会因为文件被改动而移动。
> ★ **而这五条在第 85 轮逐条核过，结论是：其中只有一条能按写的样子写成 grep**。>
>>
> | 条目 | 真实状态 |>
> | --- | --- |>
> | `to_vec()` / `Vec::from(` 作用于块数据 | **grep 分不出来**：传输路径里唯一的 `to_vec()` 是 `session/send/transfer.rs:53` 的 **bitmap**，不是块数据。要区分它们需要知道变量的来历，而不是看一行。 |>
> | `pub HashMap`/`pub Vec` 在公开签名里 | **已覆盖**，但在 `api-guard` 而不是 `tools/guards`：它报 `BTreeMap`、`io::Error` 这类**外部类型出现在承诺里**。 |>
> | `pub` 字段（除 POD） | 同上，也在 `api-guard` 的规则里。 |>
> | `std::sync::Mutex` 跨 `await` | **不是 grep**：需要跨 await 点的流分析。一个行内正则会把所有持有锁的函数都报出来。 |>
> | `rayon` / `std::thread::spawn` 在 `crates/*` | **与代码矛盾**：`engine/executor.rs` 的**有界线程池**是 [12](./12-wasm-host.md) 明确赞同的设计（`MAX_THREADS = 64` + `sync_channel`），而 `listener.rs:114` / `synra-control/endpoint.rs:659` 也各自 spawn。按字面写会把已经被认可的设计报成违规。 |>
>>
> 也就是说这张表的意图分成三类：**已经被另一个工具覆盖**（#2、#3）、**本身不是 grep**（#4）、>
> **与现有代码矛盾而需要先做一个决定**（#5）。真正剩下可以写的只有 #1，而它需要知道>
> 哪些变量是块数据 —— 那不是 grep，是一个关于类型的约定（比如让块走一个 newtype）。>
## 7. 反例：这些"优化"是错的

写下反面清单，避免为了性能把核心搞复杂。

| ❌ 不要                                                  | 为什么 |
| ------------------------------------------------------- | ------ |
| 在非热路径用 `unsafe`（例如为了跳过一次边界检查）            | 收益 < 维护成本；`unsafe` 只允许在沙箱边界那一处（[12](./12-wasm-host.md)） |
| 为了省一次 clone 就把 `Arc<Mutex<T>>` 传进业务逻辑          | 把锁语义泄漏到业务层，破坏封装与可测性 |
| 用 `FxHash` 处理对端可控的键                              | 碰撞 DoS（§4.3） |
| 把 `serde_json::Value` 全换成 `simd-json`                 | 换个 JSON 库是全局风险，收益在 payload 场景有限；先用 `RawValue` 拿到 80% 收益 |
| 为"未来可能的并发"预先引入无锁结构                          | 复杂度立刻付出，收益是假设的（[11 D3](./11-design-principles.md)） |
| 手写 `impl Iterator` 只为了少一次分配                       | 可读性代价高；先看 profiler 结论 |
| 把 trait 全改成 `dyn` 以减少单态化膨胀                       | 热路径上 `dyn` 的间接调用与优化屏障代价通常更大；只有冷路径才考虑 |

---

## 8. 实现清单

> ★ 这一节在实现之前写就，长期整节未勾选。逐条核对源码后：**5 条真做了，5 条缺失**。
> ★ 另外要单独说明：**§4.1 的每一个性能预算都只是散文**。全仓无 `benches/` 目录、无 `criterion`、
> 无 `#[bench]`（已 grep 确认），所以帧编解码 < 5 µs、路由 O(1)、capability < 200 ns、
> host function < 20 µs、传输 > 100 MB/s、插件池命中 < 5 µs **全部没有测量**。
> 唯一被测的是**分配次数**，不是时间。

- [x] `STABILITY.md`：三层 + 每 crate 归属 —— **A，且本次修好了两个漏网的**。
      三层在 `STABILITY.md:12-14`，逐 crate 归属；`api_guard::CRATES`(`tools/api-guard/src/lib.rs:134`) 镜像之。
      ★ `synra-control` 与 `synra-plugin-sdk` **两张表都没有** —— 即没有等级、没有规则。
      而守着这件事的测试当时断言的是 `CRATES.len() >= 13`（表的转述，不是与工作区的比对），
      所以看不见第 14、15 个成员。已改成**读 `Cargo.toml`**，任何 `crates/` 下未分类的成员都会红；
      两个 crate 已按 `Internal` 落表（保守选择：`Internal` 不承诺任何东西，日后**升级**不破坏任何人）。
- [ ] `api/*.txt` 快照 + CI diff 守卫 —— **未做，且选型已被明确放弃**：
      无 `api/` 目录、无快照文件、全仓无 `cargo public-api`、无 CI。
      `STABILITY.md:101-103` 自己写明改用 `tools/api-guard` 的文本扫描。
      ★ 那条替代方案**能抓什么、抓不到什么**要记清楚：抓得住"公开签名里出现具体容器"
      "稳定层 pub enum 缺 `#[non_exhaustive]`""公开错误持有第三方类型"；
      **抓不到 re-export、模块树可达性、pub 字段**。
      而 `STABILITY.md` 声称它会报"a public struct whose fields are public" ——
      `Violation` 只有四个变体，**没有 PublicField**，`public_fn_or_field` 的字段分支要求
      `line.starts_with("pub ")`，缩进的字段行永远匹配不上，所以 `pub struct S { pub count: u32 }`
      对它是隐形的。**这是一处过度声明，应当改掉或补上实现。**
- [x] 热路径零分配断言（§6.2）—— **A，但类型改名，而且两份断言原先都是空转的**。
      `AllocCounter` 不存在；真实名是 `CountingAllocator` / `CounterGuard` / `AllocationBudget` /
      `AllocationCount`，在 **`synra-testkit-alloc`**（不是 `synra-testkit`）。
      ★ 两处零分配测试（`synra-core/tests/hot_paths.rs`、`synra-protocol/tests/hot_path_alloc.rs`）
      **原先都没有声明 `#[global_allocator]`** —— `CountingAllocator` 只是一个类型，
      没有任何二进制把它装上去，所以计数器恒读 0，**每一条零断言都"通过"了**。
      装上之后的真相：帧解码 0、capability 匹配 2、路由 2、门禁 3、查不到 5。
      capability 匹配已改 `SmallVec`（2 → 0），门禁顺带 3 → 1（详见 [11](./11-design-principles.md)）；
      剩下三条**断言实测值而不是 0**，测试名也改成了 `*_stays_within_its_budget`。
      两份文件现在都带一个**阳性对照**：断言一个确定会发生的分配**被计数到了** ——
      没有它，一个恒读 0 的计数器能让所有断言一起"通过"。
- [ ] grep 守卫脚本（§6.3）接进 `just check` —— **未做**：无 `justfile`、无 xtask、无 CI。
      `tools/guards` 是一个 Rust 二进制，含 5 项检查，**其中没有一项是 §6.3 表里的条目**；
      §6.3 里已实现的两行（公开签名含容器、pub 字段）其实在 api-guard，而后者只实现了一半（见上）。
      ★ **这 5 项里原本有 1 项永远不可能失败**：`guard_host_functions_are_gated` grep 的
      `ungated_host_fn!` 只出现在它自己内部，它白名单的文件连目录都不存在。已重写为读生成表，
      负控制确认会红。另外 4 项都做了负控制：`directories`、`protocol 无 IO`、`bin 瘦`、
      `内核不依赖 GUI` **都能真的失败**。
- [x] `synra-protocol`：`payload: Option<Box<RawValue>>` —— **A/一半**：
      `envelope.rs:142` 精确。★ **`inbound_ref` / `EnvelopeRef` 全仓 0 命中** ——
      真实 API 是 `Envelope::inbound(&RawValue, DeviceId)`(`:245`)/`inbound_any`(`:256`)，
      返回**拥有**的 `Envelope` 而不是借用的视图。借用视图没有构建。
- [x] `CapabilitySet` 内部结构 + 顺序无关 proptest —— **A**：
      按域分桶 `SmallVec<[(Domain, Vec<Capability>); 8]>` + `SmallVec<[Capability; 8]>` patterns
      + `DomainMask`；`contains`(`limits.rs:126`) 走 mask→bucket→any，**没有依赖顺序的提前退出**；
      顺序无关 proptest 在 `set/proptests.rs:32`。
      字段名是 `patterns` 不是 `wild`，掩码是新类型 `DomainMask` 不是裸 `u32`。
- [x] 事件注册表（`EventId` + 路由表 + 有界缓存）—— **A/C**：
      `EventId(u16)`(`routing.rs:52`) + `EventRegistry`(`:157`)，`routes: Vec<Vec<PluginKey>>`
      （稠密 id 直接索引）、`wildcards` 受 `MAX_WILDCARD_RULES` 约束、
      `by_name` 用带种子的 `RandomState` + 硬上限 `MAX_DISTINCT_NAMES = 1024`。
      ★ 文档写的 `LruCache<EventName, EventId>` 被"到顶即丢"取代 —— 同样有界，且不需要一个缓存依赖。
- [x] `FrameDecoder::push(Bytes)` / `next_frame()` / `recycle()` —— **A/三分之二**：
      `next_frame()`(`frame.rs:627`) 与 `recycle()`(`:726`) 精确；
      `push` 收 `&Bytes` 而不是 `Bytes` —— 同样避免复制。
      ★ **没有 `Fragment` 类型**：`next_frame` 返回拥有型 `Frame`，其 body 是同一 allocation 的
      `Bytes` 切片(`:669`)，零拷贝成立。
- [ ] 传输路径零拷贝的 grep 守卫（§5.5）—— **未做**，该守卫不存在。
- [ ] 依赖选型表落进 `Cargo.toml` + PR 模板 —— **一半，且 PR 模板没有**。
      7 个里 4 个已落：`bytes`、`compact_str 0.9`、`smallvec 1`、`serde_json raw_value`。
      `bitvec` / `foldhash` / `dashmap` **不在任何 `Cargo.toml`**（手写 `ProgressBitmap` 代替 bitvec、
      `std::hash::RandomState` 代替 foldhash）。无 `.github/` → 无 PR 模板。
- [ ] ★ 另外记一条**不在清单里但属于本文范围**的：根 `Cargo.toml` 的
      `[profile.release] panic = "unwind"` 注释写着"插件 panic 必须能被 `catch_unwind` 捕获"，
      而 `catch_unwind` 在全仓出现 **0 次** —— 为一个不存在的机制做的配置。
      **本次已实现**：边界在 `synra-core/src/dispatch/plugins.rs` 的 `run_on`
      （捕获 → 丢弃该实例 → 返回 `CallOutcome::Trap`），测试与负控制见
      `dispatch/tests.rs` 的 `a_plugin_that_panics_the_host_does_not_stop_the_kernel`。
