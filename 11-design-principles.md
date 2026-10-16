# 11 · 设计原则：必须避免什么

> 本文是**硬性约束清单**，不是"最佳实践建议"。
> 每一条都对应一个真实会发生的失败模式，以及一条**可执行的检查**。
>
> 用法：写代码前扫一遍；review 时对照检查；能自动化的已经进了 [06 §7.1 的结构守卫](./06-workspace-layout.md)。
>
> 标记：**必须避免**（违反就是设计错误）/ **检查**（怎么做才不会被违反）

---

## 0. 十条速查

| #  | 必须避免                        | 一句话后果                          |
| -- | ------------------------------- | ----------------------------------- |
| 1  | 帧解码遇到坏数据时"跳过并继续"     | 永久失去帧边界对齐，整条链路失效        |
| 2  | 可靠性语义因方向而异               | 某个方向的消息静默丢失                |
| 3  | 用错误消息字符串做判定             | 随平台漂移，且会误静默真实错误          |
| 4  | 会话没有超时                      | 对端不回时永久挂起                    |
| 5  | 激活失败没有回滚                   | 插件卡在中间态，既不能用也不能清理       |
| 6  | 多处各自维护同一份状态             | 状态漂移 + 竞态                      |
| 7  | 靠魔法字符串维持隐式契约            | 改一个字就静默失效                    |
| 8  | 静默删除用户数据                   | 用户数据无声消失                      |
| 9  | 类型里有、线上从不出现的字段         | 后来者无法判断它有没有用               |
| 10 | 在用户机器上执行构建/安装脚本        | 供应链风险                          |

---

## A. 协议与传输

### A1 · 帧解码不允许"跳过坏数据"

**必须避免**：解码器遇到超长帧或坏帧时，跳过头部继续尝试解析后面的字节。

**后果**：帧边界**永久失去对齐**。之后每一个"长度"字段都是错的，表现为
"这条链路莫名其妙全坏了，而且重启才好"。

**正确做法**：

```
任何一种结构性问题 → 返回错误 → 调用方**关闭链路**
（宁可断链重连，也不要带着错位继续跑）
```

**检查**（[21 §2.2](./21-testing-strategy.md)）：
- [x] `FrameDecoder::next_frame` 的**任何** `Err` 都导致链路关闭（代码审查 + 测试）——
      ★ 补上了缺的那一半。解码器侧本来就有测试（`frame.rs:1055`
      `poisoned_decoder_refuses_further_use`：一次错误后再用返回 `Poisoned`），
      但"**链路**会停下来"只有代码阅读、没有测试。新增
      `crates/synra-core/src/link.rs` 的
      `a_frame_the_decoder_refuses_ends_the_link`：直接往 socket 上写一个未知 kind 的帧，
      断言第一次读被拒、**第二次读也被拒**（不是"继续返回下一条"）。
      ★ 这个测试只能这样写：走 `send_frame` 的话**编码器先拒**（`BodyKindMismatch`），
      那样只能证明"发送方很小心"，证明不了接收方。对端不是编码器，对端是另一端上任意的东西。
- [x] 表驱动用例覆盖：太短 / 超长 / 未知 kind / JSON 失败 / 字段多余 / 长度不符 ——
      拆在两层，都在：
      `frame.rs`：`length_below_the_minimum_is_rejected`(910) /
      `length_above_the_ceiling_is_rejected_rather_than_buffered`(925) /
      `a_ceiling_breach_inside_scratch_is_also_rejected`(940) /
      `unknown_kind_bytes_are_rejected`(779，`#[case]` 表驱动) /
      `a_body_on_a_control_frame_is_rejected`(992) /
      `a_chunk_shorter_than_its_id_is_rejected`(1005) /
      `an_oversized_chunk_is_rejected`(1018) /
      `mismatched_body_shapes_cannot_be_encoded`(1136，`#[case]` 表驱动)；
      `envelope.rs:559` `unknown_fields_are_rejected`（字段多余）。
- [ ] `cargo-fuzz`：任意字节流不 panic、不无限消耗内存 —— **未做**。见
      [04 §11](./04-wire-protocol.md) 的同名条目：现在只有 proptest 覆盖任意字节不 panic，
      property 测试与 fuzz 的输入族不同，前者通过不能当作后者的结论。

---

### A2 · 可靠性语义不能因方向而异

**必须避免**：出站消息等 ACK，入站消息不等（或反之）。

**后果**：**"谁主动连的谁才有可靠性保证"**——一个从未被文档化的隐含语义。
结果是某个方向的消息静默丢失，而发送方认为"发送成功"。

**正确做法**：

```rust
// 由**事件类型**决定 AckPolicy，不由链路方向决定
pub fn ack_policy(event: &EventName) -> AckPolicy {
    if events::NO_AUTO_ACK.contains(&event.as_str()) { AckPolicy::FireAndForget }
    else { AckPolicy::Await }
}
// ★ 出站与入站调用同一个函数
```

**检查**：
- [x] `ack_policy` 只有一个实现 —— 全仓 `grep 'fn ack_policy'` 只有
      `crates/synra-protocol/src/event.rs:515` 一处，没有第二个实现。
- [ ] ★ 出站/入站**都没有**调用它 —— 这一条没有做到，而且比"漏了"更值得记：`AckPolicy` 在整个
      workspace 里的引用**全部集中在那一个文件里**（`grep AckPolicy` 只命中 `event.rs`，12 处）。
      也就是说这个策略现在是自洽的、被测试过的、但**悬空的**：没有生产代码问它
      "这条要不要回执"。会话层各走各的路（`session/send/awaiting.rs` 的等待是传输确认，
      与事件级 ack 策略不是一回事）。
      为什么值得单独标出来：一份"只有一个实现"的绿色检查**看起来**这条已经完成了 ——
      而它只覆盖了"没有第二个实现"，没覆盖"有人用它"。检查项要能失败，得写成
      "`ack_policy` 在 `event.rs` 之外至少被调用一次"。
- [ ] 测试：同一事件在出站与入站路径上都得到 `Acked` —— 依赖上一条，先没有。

---

### A3 · 不用错误消息字符串做判定

**必须避免**：

```rust
// ❌ 这类判定会随平台与运行库版本漂移
let transient = msg.to_lowercase().contains("econnreset")
             || msg.to_lowercase().contains("socket hang up");
```

**后果**：两个方向都会错——
① 新版运行库改了消息文本 → 判定失效；
② 一个**真实**的错误恰好包含某个关键词 → 被静默掉，用户看不到。

**正确做法**（[04 §7.6](./04-wire-protocol.md)）：

```rust
impl DisconnectClass {
    pub fn from_io(e: &std::io::Error) -> Self {
        use std::io::ErrorKind::*;
        match e.kind() {
            ConnectionReset | BrokenPipe | ConnectionAborted | UnexpectedEof => Self::Transient,
            _ => Self::Io { raw_os_error: e.raw_os_error() },
        }
    }
    /// "是否让用户看见"是**类型上的结论**，不是字符串匹配
    pub fn user_visibility(self) -> UserVisibility { /* ... */ }
}
```

**检查**：
- [x] 全仓 `grep -ri 'contains("econn\|contains("socket\|contains("epipe'` 为空 —— 实测 0 命中
      （`crates` + `apps` + `tools` 全部 `.rs`）。
- [x] `UserVisibility` 的每个分支有测试 —— `error.rs` 里 `UserVisibility::` 共 8 处引用，
      分支与用例都在同一文件内。

---

### A4 · 每个会话必须有超时

**必须避免**：配对请求发出去后无限等待；传输会话永不过期。

**后果**：
- UI 上有个"黄点"永久转圈，用户只能重启 App；
- 半途中断的传输会话永久占内存（每一次失败都泄漏一点）。

**正确做法**（[04 §8.1](./04-wire-protocol.md)）：

```rust
pub struct SessionTimeouts {
    pub pairing: Duration,          // 30s
    pub intent: Duration,           // 30s
    pub transfer_idle: Duration,    // 60s
    pub transfer_total: Duration,   // 30min
    pub result_cache: Duration,     // 5min（幂等重试用）
}
```

并且**发起方在本端超时后必须清除自己的"等待中"状态**——不能只依赖对端回应。

**检查**：
- [x] 每个 `Pending*` / `Awaiting*` 状态都有超时来源 —— 全仓这类状态只有一个：
      `pairing/desk.rs` 的 `PendingPairing`，它带 `deadline_ms`，由 `pairing_window_ms` 推出，
      `sweep_expired(now_ms)` 负责清退。
      等待侧的 `session/send/awaiting.rs` 不持有状态，它收一个 `deadline: Instant` 参数，
      每个循环都拿 `Instant::now() >= deadline` 判一次。
- [~] ★ `SessionTimeouts` —— **下面这段判断有一半是过时的**（第 83 轮核对）。
      `synra-protocol/src/session.rs` 的模块注释写着"每个会话都有超时，它们都在一个地方：
      `SessionTimeouts`"。**但全仓 grep 现在有 9 个宿主侧调用点**：
      `synra-core/src/listener.rs:271`、`synra-core/src/inbound.rs:623`、
      `apps/synra-daemon/src/serve.rs:238`、`apps/synra-daemon/src/runtime/commands.rs:497,515`，
      以及四个测试。也就是说这个"唯一权威的超时配置"**确实被生产代码读取了** ——
      下面那句"11 处命中，全部在 `session.rs` 内部"写于守护进程连接那一半被实现之前。
      `SessionTimeouts`、`Default`、`validate()` 与 L934-960 的自测仍然在 `session.rs` 里，
      但那不再是全部。
- [ ] ★ 而 `ack_policy`（A2）**确实仍然悬空**，只是原因比"没人调它"更深：
      全仓搜 `AckPolicy`/`ack_policy`，`event.rs` 之外**只有一条文档注释**。
      原因是**根本没有发送路径可供询问**：`synra-net` 整个 crate
      （`Backoff` / `Liveness` / `WriteQueue`）**在两个工作区里一个消费者都没有**，
      而 `PeerLink::send` 直接 `write_all` 到 socket。
      要接 `AckPolicy`，先要有一条走 `synra-net` 的写入路径 ——
      那是一个架构决定（写入线程），不是一行 `if`。
      ★ 而第 83 轮顺手修掉了同一处的真缺陷：`PeerLink` 设了**读**超时而**没有写**超时，
      于是一个不再读的对端不产生任何错误，只是**调用线程永远不返回**
      （TCP 不会把一个满窗口报成错误），而 `synra-core/src/peer/link.rs` 早就从同一个
      `flags.timeout` 设了写超时。两个实现里的一个现在追上了另一个。
- [x] 测试：对端完全不回应时，本端在超时后回到确定状态 ——
      `pairing/desk.rs:474` `a_waiter_whose_window_closed_is_told_the_request_expired`：
      配对窗口关掉后，等待方收到的是"请求已过期"，不是挂住。

---

### A5 · 传输的"成功"必须经过校验

**必须避免**：收到所有块就算成功；摘要字段可选；块内容走 base64 进 JSON。

**后果**：
- 传输破损但报告成功；
- 33% 的带宽浪费在 base64 上；
- 接收端缺块时静默丢弃，发送端永远不知道。

**正确做法**（[04 §8.4](./04-wire-protocol.md)）：

| 要求                                    | 检查                                  |
| --------------------------------------- | ------------------------------------- |
| `digest` 必填且必须校验                    | [ ] 协议里 `digest` 不是 `Option`       |
| 块走二进制帧（`ChunkFrame`），不经 JSON     | [ ] 代码里没有 `chunk_base64`           |
| 滑动窗口而不是逐块 await                    | [ ] 有一个显式的窗口参数，默认 32        |
| 缺块回 `progress`（位图），只重传缺的         | [ ] 测试：丢一块后仍能完成              |
| 会话有 TTL                                | [ ] 见 A4                              |
| 帧级 ACK ≠ 会话级 ACK                     | [ ] 代码里区分这两种确认                |

---

### A6 · 不允许"幽灵字段"

**必须避免**：类型里定义了字段，但线上从不发送（或从不读取）。

**后果**：后来者看到字段会以为它有用，基于它写逻辑；或者以为某个能力已经实现。
最坏的情况是**安全机制看起来存在**——例如一个 `signature` 字段，从不填充也从不校验，
于是"签名"这件事在文档里是有的，在行为上是没有的。

**正确做法**：**线上出现的每个字段都必须有生产者、消费者，以及测试覆盖。**
暂时不实现的机制不要放进协议类型（可以放在 feature flag 后面的独立类型里）。

**检查**：
- [ ] [04](./04-wire-protocol.md) 的 `Envelope` 字段表 = 代码里的字段
- [ ] 每个字段都有至少一个构造点与一个读取点
- [ ] 未实现的能力（如集群）用 `#[cfg(feature = "...")]` 隔离，而不是留一个空字段

---

## B. 安全

### B1 · 插件必须在内核强制的沙箱与闸门里

**必须避免**：插件与宿主共享同一个执行环境（同一个 JS realm / 同一个地址空间）；
把权限校验交给插件自己或交给打包期的约定。

**后果**：插件可以
- 读取宿主的任意内存 / 全局对象；
- 绕过校验直接调用底层能力；
- 无限分配内存或死循环，让宿主不可用。

**正确做法**（[09](./09-plugin-system.md)、[10](./10-capability-and-sandbox.md)、[12](./12-wasm-host.md)）：

| 层   | 手段                                                          |
| ---- | ------------------------------------------------------------- |
| 隔离 | headless 插件跑 Wasm（线性内存 + 无 syscall）；UI 插件跑独立 origin 的 WebView |
| 能力 | 三层闸门：编译期（`import` ⊆ manifest）+ 装载期（校验）+ 调用期（每次检查） |
| 资源 | 内存硬上限 + fuel + epoch 中断 + 速率限制 + 配额                |
| 崩溃 | 插件 trap 只影响本次调用；宿主 panic 用 `catch_unwind` 兜住       |

**检查**（[21 §5](./21-testing-strategy.md)）：
- [ ] 恶意插件套件覆盖：死循环、内存炸弹、越权调用、伪造事件、路径逃逸、SSRF
  > ★ **这个套件仍然不存在**，但本轮在其中一项上找到了真问题。
  > **「伪造事件」的防御写好了、测试了、并且在自己的文档里声明了它在做什么**：
  > `EventName::check_emittable_by` 的注释写着"The host calls this on every outbound plugin event.
  > It is the difference between 'plugins have their own namespace' being true and it being a sentence in a document"——
  > 而 **孿主从来没调过它**，六个调用点全在它自己的测试模块里。
  > 于是两件事都是允许的：插件发另一个插件命名空间的事件（`_plugin.notes.*`），
  > 以及插件发孿主事件（`_synra.device.changed`）。本轮接上了（`event-emit` / `device-send` / `device-broadcast` 三处）。
  > ★ 而且接上后立刻发现**测试配置用的事件名 `a.b` 根本不合法**（语法只收
  > `_synra.` / `_plugin.` / `synra.internal.` / `device.` / `file.transfer.` 五个命名空间）——
  > 在没人校验之前，一个名字是不是事件名没有人问。
  > * 另一项发现：**SSRF 的代码存在但没有校验器**——`SchemeNotAllowed` / `HostNotAllowed` /
  >   `IpNotAllowed` 三个码只出现在 `error.rs` 的枚举、可重试表和描述表里，没有任何地方产生它们。
- [x] host function 的**唯一**注册方式 —— ★ 这两条写的机制不存在：全仓 `grep 'gate_host_fn!'`
      与 `grep 'ungated_host_fn!'` 都是 **0 命中**。
      真正的机制是更强的：`crates/synra-plugin-host/build.rs` 在**每次构建时**从
      `wit/synra-plugin.wit` 重新生成 `src/generated/capability_table.rs`。
      负控制：手工把表里 `#log` 改成 `None`，重新 `cargo test` —— 文件被 build.rs 改回，
      测试全绿。**表想漂移都漂不了**，这比"宏是唯一注册方式"更难违反。
- [x] 无 gate 集合的白名单守卫 —— `host_surface.rs` 里三个测试已经守着这条，且早于本次核对存在：
      `the_ungated_list_is_short_and_every_entry_is_a_read`（≤10 条，且每条必须是 `env-*` /
      `platform-capabilities` / `plugin-id`）、`nothing_dangerous_is_ungated`（点名清单）、
      `the_gated_and_ungated_lists_partition_the_table`（两张表恰好划分全集）。
      ★ 文档写的是"ungated 只允许出现在 `env.rs`"，而实际的免检集合是 7 个入口，
      多出 `platform-capabilities` 与 `plugin-id`。就性质而言没问题（都是"我是什么"的事实读取），
      但规则写一种、事实是另一种，规则就会被下一个人按自己以为的方向"修正"。
      白名单测试已经把这个集合钉住了，所以这里改的是文档，不是代码。
- [ ] 单测断言"被拒的调用**没有产生副作用**"（audit 里只有拒绝记录）

---

### B2 · 默认拒绝

**必须避免**：白名单为空时"允许全部"；未知输入按"最宽松"解释；缺字段按"最宽松"默认值。

**正确做法**：

| 场景               | 默认                                                     |
| ------------------ | -------------------------------------------------------- |
| `fs` 路径          | 只能访问显式授权的子目录，且**先 canonicalize 再比较**        |
| `network` 目标     | 只能访问白名单 host；scheme 只允许 https；IP 策略默认拒私网    |
| 平台能力           | 未声明的能力一律拒绝；高危能力默认**不可授予**                |
| 事件发送           | 插件只能发自己命名空间下的事件                               |
| UI 插件的方法白名单  | 未列出的方法一律拒绝                                        |
| 归档解包           | 超出条目/大小/深度/路径限制一律拒绝；symlink 条目直接拒绝       |

**检查**：
- [x] 每个白名单的"空"状态有测试（断言拒绝而不是允许）—— `capability/set/tests.rs` 里
      空集合的用例断言的是**拒绝**；`config/tests.rs:174`、`tests.rs:429` 也明确写着
      "一个白名单会让新设置进不来"。
- [x] 路径比较是**组件级**（`/data` 不匹配 `/database`）—— `synra-store/src/paths.rs:237`
      `is_within` 按 `Path::components()` 逐段比较，注释直接点名这个坑；
      用例在同文件 L366/L399-403/L515/L531/L537。
      另一条独立证据：`synra-capability` 的 `tests.rs:268` 断言
      `fs:read:/database` **不**被 `fs:read:/data*` 覆盖。

---

### B3 · 敏感数据不进日志

**必须避免**：把 payload / 剪贴板内容 / URL 的 query / 密钥直接写进日志或诊断包。

**正确做法**（[18 §2.3](./18-observability.md)）：
一层 `RedactionLayer`：
- 命中字段名（`payload` / `body` / `content` / `token` / `secret`…）→ 替换为 `<redacted>`；
- 其余字符串值超过 512 字符 → 截断；
- URL 类型 → 去掉 query 与 fragment。

**检查**：
- [x] 测试：给日志宏传一个含 `hunter2` 的 payload，断言输出里**不含** `hunter2` ——
      `synra-observe/src/redact.rs:314-315` 就是这条，用例字面量是
      `("secret", "hunter2")` 与 `("password", "hunter2")`。
- [x] 测试：诊断包里不含密钥 —— `synra-ipc` 的 `tests.rs:470`
      `a_diagnostic_export_defaults_to_the_least_intrusive_bundle` 断言
      `DiagExportOptions::default()` 是最不侵入的一档。
      ★ 提醒一句这条的**边界**：它断言的是默认档位，不是"任何档位都不含密钥"。
      高侵入档位是否含密钥，这条检查没有覆盖。
- [x] 审计记录**不含 payload 内容**（只有"调用了哪个能力"）——
      `synra-observe/src/audit.rs:437` `a_record_has_no_place_to_put_a_payload`。
      这个保证是**构造性**的：`AuditEntry` 的字段只有
      `at_ms` / `plugin` / `capability` / `outcome` / `elapsed_ms`，
      没有任何一个能装下内容，所以不是一个可能被忘记的约定。
      `AuditEntryDto` 同样只有 `at_ms` / `capability` / `allowed` / `reason` /
      `elapsed_ms` / `trace_id`。另有 `audit.rs:758`
      `a_rendered_record_is_a_destination_not_a_body` 守着渲染层。

---

### B4 · 插件产物必须可验证

**必须避免**：trust-on-install（没有签名、没有摘要校验）；在用户机器上执行安装/构建脚本。

**后果**：任何能投递一个包的人都能在用户设备上执行任意代码。

**正确做法**（[16](./16-plugin-distribution.md)）：

| 要求                                 | 检查                                     |
| ------------------------------------ | ---------------------------------------- |
| 签名强制（ed25519），且**不提供跳过开关** | [ ] 安装流程里没有 "skip verification" 分支 |
| 摘要必校验（与来源声明一致）             | [ ] 篡改测试：改一个字节 → 必须拒绝          |
| 发布者 id 与公钥**绑定**                | [ ] id 相同但公钥变了 → 视为不可信           |
| 归档安全（条目/大小/路径穿越/炸弹）        | [ ] 每个限制都有一个恶意 fixture            |
| 安装 = 下载 + 校验 + 解包，**不构建**      | [ ] `InstallSource::Local` 只在 dev 下可用  |

---

## C. 状态与并发

### C1 · 唯一权威

**必须避免**：多个模块各自维护"同一份状态"的副本并互相同步。

**后果**：状态漂移、竞态、"两边显示不一样"、没人说得清哪个对。

**正确做法**（[03 §4](./03-architecture-baseline.md)）：
每个状态只有一个所有者（聚合根），其余都是**从它派生的视图**。
`DeviceCard` 是派生的，不是存储的。

**检查**：
- [x] 全仓搜索：不存在第二个 `HashMap<DeviceId, _>` 表示"设备状态" —— `HashMap<DeviceId` 全仓
      **只有一处命中**：`pairing/desk.rs:97` 的 `Mutex<HashMap<DeviceId, Entry>>`。
      看了：那是**配对台账**，`Entry` 装的是 `PendingPairing` 加一个一次性应答通道，
      生命周期是"用户还没回答的那几十秒"。它不是设备状态的第二份拷贝 ——
      设备状态仍然是 store 里的那份。
      ★ 所以这条的**措辞**有问题：它写的是"不存在 `HashMap<DeviceId, _>`"，
      而真正要禁的是"存在第二份**设备状态权威**"。照字面读，一个完全正当的配对台账也会被判不合格；
      按字面去改，则要为了过检查去换掉一个无关的 key 类型。
- [ ] 派生视图的构造只依赖聚合根的公开读取接口
- [ ] 前端 store 都是**只读镜像**（不判断业务规则，只缓存快照 + 应用增量事件）

---

### C2 · 非法状态无法表示

**必须避免**：用 `enum` + 运行时 `if` 表示"这个操作在当前状态下不合法"。

**后果**：忘记检查 → 运行时错误；检查写错 → 状态机出现空洞
（例如某个状态从来没被赋值过，或从某个状态出不来）。

**正确做法**：**状态数 ≤ 4 且非法转移必须编译期禁止 → 类型状态机。**

```rust
pub struct Link<S: LinkState> { /* socket 只存在于已连接的链路里 */ }

impl Link<Connecting> {
    pub async fn complete(self, ack: HelloAck) -> Result<Link<Ready>>;
    pub fn fail(self, reason: HandshakeError) -> FailedLink;
}
impl Link<Ready> {
    pub async fn send(&self, frame: &Frame) -> Result<SendOutcome>;  // socket 必然存在
}
```

状态多（> 4）且转移复杂时用 `enum` + **私有字段** + 显式 `transition()`，而不是裸 `enum`。

**检查**：
- [ ] `Link` / 插件槽位用类型状态机
- [ ] 不存在"声明了但从未被赋值的状态"
- [x] 状态机的 `match` 里没有 `_ => unreachable!()` 兜底 —— 全仓
      `grep '_\s*=>\s*unreachable!'` 0 命中。

---

### C3 · 每个队列有上限，每个 await 有超时或取消源

**必须避免**：无界队列；没有取消源的读循环；`await` 一个可能永不返回的东西。

**后果**：
- 无界队列 → 内存增长到 OOM（而且是缓慢的，很难定位）；
- 无取消源的 `await` → **关停卡住**、进程不退出；
- 而这类问题**事后改造代价极高**（要动整个调用链）。

**正确做法**：

| 位置         | 上限                                       | 溢出行为                       |
| ------------ | ------------------------------------------ | ------------------------------ |
| 链路写队列    | `WRITE_QUEUE_CAPACITY`（256 帧）             | 链路判为不健康并关闭             |
| 内核事件广播  | 有界 `broadcast`                            | 订阅者落后 → `Lagged` → 全量重取 |
| 插件 host call | 每插件并发上限（`Semaphore`）                | 排队 + 超时                     |
| 传输分块      | 滑动窗口                                    | 节流                           |
| 审计记录      | 环形缓冲（10000 条）+ DB 保留期              | 覆盖最旧                       |
| 指标标签基数   | `CardinalityGuard`（每指标 1000 series）     | 归入 `overflow`                |

**检查**：
- [x] 所有 `mpsc` / `broadcast` / `DashMap` 的构造点都有容量参数 —— `tokio` 的
      `mpsc::channel()` / `broadcast::channel()` 无参形式全仓 **0 命中**。
      ★ 唯一的无参构造是 `pairing/desk.rs:189` 的 `std::sync::mpsc::channel()`：
      它不是异步队列，而是"用户还没回答时的一次性应答"，发送端只有这一个、
      收一个值就不可能再发，并且 `pairing_window_ms` 已经给了截止时间。
      严格按字面（"`mpsc` 都要有容量"）它不合格；按字面去改要么换个类型、要么加一个恒定容量，
      都是在为一个不存在的泄漏风险写代码。所以这里改文档。
- [ ] 全仓搜索 `recv().await` / `read().await`：每一个都能被 `CancellationToken` 打断
- [ ] 关停测试：在"正在传输 + 正在调用插件"时关停，必须在 grace 内退出
- [x] **内核里禁止 `std::sync::Mutex` 跨 `await`** —— `std::sync::Mutex` 全仓 2 处命中，
      都在测试文件里（`desktop_tests.rs`、`host_calls_tests.rs`）。生产代码零命中。
      配对台账用的是 `parking_lot::Mutex`，且 `desk.rs:195` 在阻塞前显式 `drop(entries)` ——
      注释写的正是这条规则。

---

### C4 · 锁与所有权

**必须避免**：`Arc<Mutex<BigState>>` 一把大锁保护所有状态。

**后果**：网络 IO 与 UI 查询互相阻塞；无法表达"某状态下某操作不合法"；
无法对单个子系统做单测。

**正确做法**：每个聚合根自己持有内部状态，用**消息传递 + 细粒度锁**暴露操作；
长操作（网络、Wasm 调用、文件 IO）在锁外做。

**检查**：
- [ ] 不存在跨越整个 `Kernel` 的锁
- [ ] 临界区里没有 `await`（除了专门的异步锁）
- [ ] 每个聚合根能被单独构造与测试

---

## D. 可维护性

### D1 · 不靠魔法字符串维持接线

**必须避免**：一个模块用"约定好的错误字符串 / 事件名字符串"与另一个模块通信。

**后果**：改一个字就静默失效，且**没有任何编译期或测试期信号**。

**正确做法**：用**显式枚举**代替字符串契约。

```rust
// ❌ 探测结果与合并逻辑之间靠 error 字符串沟通
Ok(vec![ProbeResult { ok: false, error: Some("PROBE_EMBEDDED_IN_DISCOVERY".into()) }])

// ✅
pub enum ProbeOutcome {
    Reachable { peer: PeerInfo },
    /// 该目标已在发现阶段被探测过，无需重复
    AlreadyEmbedded,
    Unreachable { reason: String },
}
```

**检查**：
- [ ] 全仓搜索字符串比较：只允许出现在①协议字段值校验 ②用户可见文案 ③测试断言里
- [ ] 事件名、命令名、错误码都是常量或枚举，不是散落的字面量

---

### D2 · 不静默删除用户数据

**必须避免**：发现某个插件 artifact 不完整就**静默删除**目录与记录；
"清理"任务顺手删掉用户可能想要的东西。

**后果**：用户数据无声消失，且**没有任何提示**。

**正确做法**（[14 §2.9](./14-plugin-lifecycle.md)）：

| 场景           | 行为                                                         |
| -------------- | ------------------------------------------------------------ |
| 检测到损坏的安装 | 标记为 `Broken`，**保留** artifact 便于诊断；UI 显示并提供"清理"按钮 |
| 卸载插件        | **明确询问**是否删除插件数据；并**报告**删了什么、留了什么          |
| 更新失败        | 自动回滚到旧版本，**不删**新版本（供诊断）                        |
| GC             | 只删缓存与过期会话；**永不**碰 `plugin-data/<id>/files/`           |
| 无法完成的操作   | 返回 `Err`，不"尽力而为"                                       |

**检查**：
- [x] 代码里没有"检测到问题就 `remove_dir_all`"的路径 —— `remove_dir_all` 全仓 42 处命中，
      逐个看过上下文：**全部在测试代码里**。生产侧的两处是 `endpoint.rs` 的 `TempDir`
      （`std::env::temp_dir()` 下的测试目录，`Drop` 里清理自己），其余在
      `synra-observe/src/crash.rs` 的 `#[cfg(test)] mod tests`（L369 起）。
      也就是说这条规则在生产代码里成立 —— 但成立的原因是**没有人在生产里调用它**，
      而不是因为有守卫拦着。加一条守卫仍然是值得的，因为这条禁令目前靠的是自觉。
- [ ] 卸载流程有测试断言"删除前有询问、删除后有报告"
- [x] GC 的测试断言“用户数据未被触碰” —— **已有**（第 92 轮）：
      `apps/synra-daemon/src/runtime/tests.rs` 的 `the_sweep_takes_only_what_it_is_for`。
      ★ **而它比试试看更强在两点**：它用 `counts()`（**产品自己的机制**，
      `status` 报的就是它）对比扫前扫后，所以不会因为一份手维护的清单而通过；
      并且它把“**不该动的表**”列出来（identity / paired_devices / plugins / plugin_grants / config…），
      所以将来新增一个收集目标会**因为出现在差异里**而红，而不是因为没被提到。
      ★ 而断言的**顺序也是它的一部分**：取反对照时发现，把一个收集目标指向
      `paired_devices` 会先触发“收集到了 0 条”而**永远走不到那条会点名受损表的断言**。
      现在具体断言在前：一个会真删的错误目标给出
      `the sweep changed \`paired_devices\`, which it is not for`（`Some(1)` → `Some(0)`）。

---

### D3 · 类型与代码结构要对齐职责

**必须避免**：一个包/模块持续膨胀，成为"什么都在里面"的地方。

**后果**：新增功能倾向于"再加一个模块"，而不是"放到正确的层"。
半年后没人敢重构它。

**正确做法**：按**聚合根**切分（[07 §2](./07-core-kernel.md)），
模块只能通过定义好的接口交互；新增东西前先在 [03 §6](./03-architecture-baseline.md) 找到它的位置。

**检查**：
- [ ] 每个 crate 有明确的"职责与不变量"一节（[06 §3](./06-workspace-layout.md)）
- [ ] 结构守卫：内核不依赖 GUI、协议不依赖 IO、bin ≤ 60 行
- [ ] 任何"我想在这里加个模块"的冲动，先回答"它属于哪个聚合根"

---

### D4 · 类型契约只有一个来源

**必须避免**：同一个数据结构在两个地方各写一遍（例如 Rust 写一遍、TypeScript 手写一遍）。

**后果**：两边漂移；错误只在运行时暴露。

**正确做法**：
- Rust 类型是**唯一真相源**；
- TypeScript 类型由 `ts-rs` **生成**；
- CI 校验"生成物与当前 Rust 类型一致"（不一致则构建失败）。

**检查**：
- [ ] `src/generated/` 是生成物，且 CI 有 `types-check`（生成后 `git diff --exit-code`）
- [ ] 前端没有任何手写的 IPC 请求/响应类型

---

### D5 · 不在用户机器上执行构建或安装脚本

**必须避免**：安装插件时跑 `npm install` / `cargo build` / 任何来自包的脚本。

**后果**：供应链攻击面——一个包的 `postinstall` 就能在用户机器上执行任意代码。

**正确做法**：[ADR-007](./20-adr.md)。安装 = 下载 + 校验 + 解包。
插件作者在**自己的机器/CI**上构建好产物并签名。

**检查**：
- [x] 安装路径里没有子进程调用（`Command::new("npm"|"cargo"|"pnpm")`）—— 全仓 0 命中。
- [ ] `InstallSource::Local` 在 release 构建里返回错误 —— 名字对不上：代码里是
      `InstallSourceDto`（`synra-ipc/src/dto/plugin.rs:127`，一个 **DTO**），
      没有叫 `InstallSource` 的安装源类型。DTO 是给前端的形状，不做发布构建的分支。
      要么这条该写 `InstallSourceDto`，要么**内核里根本没有"从本地路径安装"这个入口**
      —— 后者若是前者，那这条禁令是靠"没有这条路"满足的，而不是靠检查。

---

## E. 性能与体验

> **这一节只讲"用户能感知的性能"。** 内核内部的**类型选择、分配、零拷贝、预计算**等
> 设计级性能约束单独成篇：[26 核心稳定性、封装与性能](./26-core-stability-and-perf.md)。
> 两篇的分工：本文说"用户会看到什么变慢"，26 说"代码该怎么写才能在热路径上不慢"。

### E1 · 首屏不等后端

**必须避免**：UI 要等所有子服务初始化完成才能渲染。

**后果**：冷启动时间 = 最慢的那个子系统的初始化时间。

**正确做法**（[07 §9](./07-core-kernel.md)）：
内核启动与 UI 加载**并行**。UI 先渲染骨架，然后：
1. 拉一次 `snapshot()`（全量）
2. 之后只消费增量事件

**检查**：
- [ ] 冷启动到可交互 < 500ms（[21 §8](./21-testing-strategy.md)）
- [ ] UI 在"内核尚未就绪"时能渲染（用假内核的测试）

---

### E2 · 传输不做逐块串行

**必须避免**：`for chunk in chunks { send(chunk).await? }`。

**后果**：吞吐受 RTT 限制（一个 5 MB 的包 ≈ 80 个块 → 至少 80 个 RTT）。

**正确做法**：滑动窗口（默认 32 块未确认）。

**检查**：
- [ ] 传输实现里有显式的窗口参数
- [ ] 吞吐基准 > 100 MB/s（本机，[21 §8](./21-testing-strategy.md)）

---

### E3 · 移动端后台可达性要如实（不承诺做不到的事）

**必须避免**：在产品文案里承诺"手机随时立刻收到"。

**后果**：用户遇到"没收到"时认为是 bug，而它其实是操作系统限制。

**正确做法**（[22 R-03](./22-risks.md)）：
- 桌面侧可以常驻（托盘）→ 保证发起端不丢；
- 移动侧提供"排队 + 上线后送达"的接口，平台实现按各系统机制补；
- **UI 上的语义是"已排队，对方上线后自动送达"**，而不是"已送达"。

**检查**：
- [ ] 没有任何文案承诺"实时到达移动端"
- [ ] 有"排队 → 对端上线 → 自动送达"的端到端测试

---

## F. 把原则变成检查

| 原则组   | 自动化方式                                                          |
| -------- | ------------------------------------------------------------------- |
| A（协议） | 表驱动用例 + proptest + fuzz + `KernelPair` 集成测试                   |
| B（安全） | 恶意插件套件（[21 §5](./21-testing-strategy.md)）+ CI 守卫 + 敏感数据测试 |
| C（状态） | 关停测试 + 并发压力测试 + 类型状态机的穷尽 `match`                       |
| D（维护） | CI 守卫（[06 §7.1](./06-workspace-layout.md)）+ `types-check` + grep 守卫 |
| E（性能） | `criterion` / `iai-callgrind` 基准门禁 + 手工清单                       |

**未自动化的部分**（需要 review 检查）：

- [ ] 新增的"等待"是否有超时来源
- [ ] 新增的"状态副本"是否有唯一权威
- [ ] 新增的"字符串契约"是否是枚举
- [ ] 新增的"清理逻辑"是否会碰用户数据
- [ ] 新增的"承诺"是否在能力范围内
