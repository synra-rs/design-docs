# 21 · 测试与验证策略

> 本文定义测试分层、协议一致性验证、恶意插件对抗测试、性能基准与验收方式。
> 目标：**每一类风险都有对应的自动化测试**，而不是上线后靠用户报 bug。
>
> 与 [11 设计原则](./11-design-principles.md) 的关系：那篇说"必须避免什么"，本文说"怎么证明没踩到"。

---

## 1. 分层

```
        ┌─────────────────────────────────────┐
        │  E2E（少量，慢）                      │
        │  · Tauri WebDriver 桌面端到端          │
        │  · 真机移动端                          │
        ├─────────────────────────────────────┤
        │  集成测试（中量）                     │
        │  · KernelPair：双内核 + 内存传输        │
        │  · 插件端到端（真 wasm 在真沙箱）        │
        │  · 安装 / 更新 / 回滚全流程             │
        │  · SQLite 迁移                         │
        ├─────────────────────────────────────┤
        │  单元测试（大量，快）                  │
        │  · 纯函数：capability 匹配、帧编解码、    │
        │    路径解析、错误分类、状态转移           │
        ├─────────────────────────────────────┤
        │  属性测试 / 模糊测试                   │
        │  · proptest + cargo-fuzz              │
        └─────────────────────────────────────┘
```

### 1.1 工具

| 用途       | 工具                                                                   |
| ---------- | ---------------------------------------------------------------------- |
| 测试运行器  | **`cargo nextest`**（并行、更好的输出、支持重试配置）                       |
| 属性测试    | **`proptest`**（策略化生成 + 收缩）                                       |
| 模糊测试    | **`cargo-fuzz`**                                                        |
| 基准       | **`criterion`**（本地真实耗时）+ **`iai-callgrind`**（CI 门禁，指令数稳定）   |
| 覆盖率      | `cargo llvm-cov`                                                        |
| 快照测试    | `insta`（JSON 输出、schema 迁移结果）                                      |
| 并发测试    | `loom`（关键同步原语）                                                    |
| Mock       | 自研 `synra-testkit`（不用 `mockall`，trait 都是自研的）                     |
| 桌面 E2E   | `tauri-driver` + WebDriver                                              |
| 前端测试    | Vitest（组件/单元）+ Playwright（可选）                                    |

---

## 2. 单元测试：必须覆盖的不变量

### 2.1 `synra-capability`（[10 §2–§3](./10-capability-and-sandbox.md)）

```rust
// 表驱动的边界用例
#[test_case("ui:*",              "ui:page",                        true  ; "单段通配")]
#[test_case("ui:*",              "ui:page:extra",                  false ; "星号不跨段")]
#[test_case("ui:**",             "ui:page:extra",                  true  ; "双星跨段")]
#[test_case("ui:**",             "device:list",                    false ; "域不匹配")]
#[test_case("network:fetch:**",  "network:fetch:api.x.com",        true  ; "host 限定")]
#[test_case("network:fetch:**",  "network:send:api.x.com",         false ; "resource 不匹配")]
#[test_case("network:fetch:api.x.com", "network:fetch:api.x.com:POST", false ; "段数不同")]
fn pattern_matching(pattern: &str, candidate: &str, expected: bool) { /* ... */ }

#[test]
fn bare_wildcard_is_rejected() {
    assert!(Capability::parse("*").is_err());
    assert!(Capability::parse("ui::navigate").is_err());
    assert!(Capability::parse("ui:").is_err());
}

proptest! {
    /// 精确匹配优先于通配
    #[test]
    fn exact_beats_wildcard(exact: Capability, other: Capability) { /* ... */ }

    /// grant ⊆ manifest 是装载的硬条件
    #[test]
    fn grant_must_be_subset_of_manifest(manifest: Vec<Capability>, grant: Vec<Capability>) { /* ... */ }
}
```

### 2.2 `synra-protocol` 的帧层（[04 §4](./04-wire-protocol.md)）

```rust
proptest! {
    #[test]
    fn frame_roundtrip(frame: Frame) {
        let mut buf = BytesMut::new();
        FrameEncoder::encode(&frame, &mut buf)?;
        let mut dec = FrameDecoder::new();
        dec.push(buf.freeze())?;
        prop_assert_eq!(dec.next_frame()?.unwrap(), frame);
    }

    /// ★ 任意字节流：要么产出合法帧，要么报错，绝不 panic、绝不越界
    #[test]
    fn decoder_never_panics_on_arbitrary_input(bytes: Vec<u8>) {
        let mut dec = FrameDecoder::new();
        let _ = dec.push(Bytes::from(bytes)).and_then(|()| dec.next_frame().map(|_| ()));
        // 允许 Err / Ok(None)，不允许 panic
    }

    /// 信封必须拒绝多余字段
    #[test]
    fn envelope_rejects_unknown_fields(extra_key: String, extra_val: String) {
        prop_assume!(!KNOWN_KEYS.contains(&extra_key.as_str()));
        let json = json!({ ...valid_envelope(), extra_key: extra_val });
        prop_assert!(serde_json::from_value::<Envelope>(json).is_err());
    }
}

/// ★ 表驱动：§4.2 的每一行都必须有一个用例
#[test_case(FrameError::Truncated,      &[0x00, 0x00, 0x00, 0x01]              ; "length < 2")]
#[test_case(FrameError::TooLarge,       &[(MAX_FRAME_LEN + 1).to_be_bytes()]    ; "length 超限")]
#[test_case(FrameError::UnknownKind,    &[0,0,0,3, 0xFF, 0x00, 0x00]            ; "未知 kind")]
#[test_case(FrameError::Malformed,      &[0,0,0,3, 0x01, 0x00, b'{']            ; "JSON 失败")]
#[test_case(FrameError::LengthMismatch, &[0,0,0,9, 0x01, 0x00, b'{',b'}']       ; "body 长度不符")]
fn frame_decode_rejects(expected: FrameError, bytes: &[u8]) { /* ... */ }

/// ★ 关键：任何错误都必须导致"不可继续"，而不是"跳过继续"（[11 A1]）
#[test]
fn decoder_is_poisoned_after_error() {
    let mut dec = FrameDecoder::new();
    let _ = dec.push(Bytes::from_static(&[0,0,0,3, 0xFF, 0x00, 0x00]));
    assert!(dec.is_poisoned(), "出错后解码器必须被标记为不可用");
}
```

### 2.3 路径解析（安全关键，[10 §4.3](./10-capability-and-sandbox.md)）

```rust
#[test_case("a/b.txt",                Ok("a/b.txt")            ; "正常")]
#[test_case("a/../b.txt",             Ok("b.txt")              ; "内部 ..")]
#[test_case("../../etc/passwd",       Err(PathError::Escape)   ; "逃逸")]
#[test_case("/etc/passwd",            Err(PathError::Absolute) ; "绝对路径")]
#[test_case("a/./b",                  Ok("a/b")                ; "当前目录")]
#[test_case("a//b",                   Ok("a/b")                ; "双斜杠")]
#[test_case("C:\\Windows\\System32",  Err(PathError::Absolute) ; "Windows 绝对路径")]
#[test_case("\\\\?\\C:\\x",           Err(_)                   ; "扩展路径前缀")]
fn resolve_path(input: &str, expected: Result<&str, PathError>) { /* ... */ }

/// 组件级前缀比较：/data 不匹配 /database
#[test]
fn prefix_compare_is_component_wise() { /* ... */ }

/// symlink 逃逸（unix）
#[cfg(unix)]
#[test]
fn symlink_escape_is_blocked() { /* ... */ }
```

### 2.4 错误分类（[11 A3](./11-design-principles.md)）

```rust
#[test]
fn transient_disconnects_are_silent() {
    for kind in [ConnectionReset, BrokenPipe, ConnectionAborted, UnexpectedEof] {
        let e = io::Error::from(kind);
        assert_eq!(DisconnectClass::from_io(&e), DisconnectClass::Transient);
        assert_eq!(UserVisibility::from(&e), UserVisibility::Silent);
    }
}

/// ★ 反向断言：不能靠错误消息里的关键词判定
#[test]
fn classification_ignores_error_message() {
    let e = io::Error::new(io::ErrorKind::PermissionDenied, "econnreset in message");
    assert!(matches!(DisconnectClass::from_io(&e), DisconnectClass::Io { .. }));
}
```

---

## 3. 协议一致性验证

### 3.1 Fixture 回放（逐字节）

协议是**全新设计**，所以 fixture 的作用是**锁定当前版本的线格式**
（防止无意间改动字节布局），而不是"对齐另一个实现"。

```
fixtures/wire/
├─ frame_hello.bin              + frame_hello.json     （期望解析结果）
├─ frame_hello_ack.bin
├─ frame_ping.bin
├─ frame_bye.bin
├─ frame_envelope_minimal.bin   （只有必填字段）
├─ frame_envelope_full.bin      （含 replyTo / ttlMs）
├─ frame_chunk.bin
├─ discovery_query.bin
├─ discovery_response.bin
├─ discovery_offline.bin
├─ transfer_request.json
├─ transfer_progress.json
└─ transfer_complete.json
```

```rust
#[test]
fn wire_fixtures_decode_exactly() {
    for f in load_fixtures("fixtures/wire") {
        assert_eq!(decode(&f.bytes)?, f.expected, "decoding {}", f.name);
    }
}

#[test]
fn wire_fixtures_encode_exactly() {
    for f in load_fixtures("fixtures/wire") {
        assert_eq!(encode(&f.expected)?, f.bytes, "encoding {}", f.name);
    }
}
```

> **fixture 变更规则**：某次改动导致 fixture 不匹配时，**必须**同时：
> ① 确认这是有意的线格式变更；② 更新 [04](./04-wire-protocol.md)；
> ③ 判断是否需要 bump `protocol`（[04 §9](./04-wire-protocol.md)）；④ 更新 fixture。
> **不允许**为了让测试变绿而直接重录 fixture。

### 3.2 双内核一致性

两个内核各跑一份完整实现，通过**内存传输**对接。
它验证的是：**同一个协议在两个独立实现里是否得到同样的结果**——
"自己与自己对"仍能发现"只有一个方向对"这类问题。

```rust
#[tokio::test]
async fn handshake_is_symmetric() {
    let pair = KernelPair::new().await;
    pair.a.connect(pair.b_id).await?;

    let a_view = pair.a.peer_capabilities(pair.b_id).await?;
    let b_view = pair.b.peer_capabilities(pair.a_id).await?;
    assert_eq!(a_view.protocol, b_view.protocol);
    // ★ 协商出的 Feature 交集必须一致：不能只有一方认为某能力可用
    assert_eq!(pair.a.agreed_features(pair.b_id), pair.b.agreed_features(pair.a_id));
}

#[tokio::test]
async fn ack_policy_is_same_in_both_directions() {
    // [11 A2] 的回归测试
    let pair = KernelPair::paired().await;
    let out = pair.a.send_and_await_ack(pair.b_id, event).await?;
    let inb = pair.b.send_and_await_ack(pair.a_id, event).await?;
    assert!(matches!(out, SendOutcome::Acked { .. }));
    assert!(matches!(inb, SendOutcome::Acked { .. }));
}
```

### 3.3 健壮性（fuzz + 故障注入）

| 目标                        | 方式                                                   |
| --------------------------- | ------------------------------------------------------ |
| 解码器不 panic               | `cargo-fuzz`：`FrameDecoder::next_frame`                 |
| 信封解析不 panic             | `cargo-fuzz`：`Envelope::inbound`                       |
| 归档解包不 panic             | `cargo-fuzz`：`unpack`（恶意 tar/zip 输入）              |
| 路径解析不逃逸               | `cargo-fuzz` + 属性测试                                 |
| 损坏帧导致断链（不是错位）      | 故障注入 `inject_corrupt_frame()` + 断言链路关闭           |
| 丢块后仍能完成传输             | 故障注入 `drop_chunk(n)` + 断言最终成功                   |
| 篡改块被检出                  | 故障注入 `corrupt_chunk(n)` + 断言 `DigestMismatch`      |
| 网络中断后重连                | 故障注入 `partition()` / `kill_peer()` + 断言恢复          |

> **不需要**"与外部实现互操作"这类测试——本项目没有需要兼容的外部实现。
> 协议正确性由 ① 逐字节 fixture ② 双内核一致性 ③ fuzz 与故障注入 三者覆盖。

---

## 4. 集成测试：`KernelPair`

`synra-testkit` 的核心工具，让"双内核 + 虚拟时钟 + 故障注入"成为一行代码。

```rust
#[tokio::test]
async fn pairing_then_message() {
    let pair = KernelPair::fresh().await;
    pair.a.request_pairing(pair.b_id).await?;
    pair.b.accept_pairing().await?;
    pair.settle().await;

    assert_eq!(pair.a.trust_level(pair.b_id), TrustLevel::Paired);
    assert_eq!(pair.b.trust_level(pair.a_id), TrustLevel::Paired);
}

#[tokio::test]
async fn stale_pairing_self_heals() {
    // A 删掉配对、B 仍以为配对着 → 下次连接必须自愈（[04 §7.3]）
    let pair = KernelPair::paired().await;
    pair.a.forget(pair.b_id).await?;
    pair.reconnect().await?;

    assert_eq!(pair.a.trust_level(pair.b_id), TrustLevel::Handshaked);
    assert_eq!(pair.b.trust_level(pair.a_id), TrustLevel::Handshaked, "对端必须被通知");
    // ★ 并且：连接是成功的，不是"连不上"
    assert_eq!(pair.a.link_state(pair.b_id), LinkState::Ready);
}

#[tokio::test]
async fn reconnect_after_peer_restart() {
    let mut pair = KernelPair::paired().await;
    pair.kill_b().await;
    pair.advance(Duration::from_secs(1)).await;
    assert_eq!(pair.a.link_state(&pair.b_id).await, LinkState::Closed);

    pair.restart_b().await;
    pair.advance(Duration::from_secs(4)).await;
    assert_eq!(pair.a.link_state(&pair.b_id).await, LinkState::Ready);
}

#[tokio::test]
async fn transfer_with_integrity() {
    let pair = KernelPair::paired().await;
    let data = random_bytes(3 * 1024 * 1024);
    let id = pair.a.transfer_send(&pair.b_id, data.clone()).await?;
    pair.run_until(id).await?;
    assert_eq!(pair.b.received(id).await.unwrap(), data);
}

#[tokio::test]
async fn intent_is_idempotent() {
    let pair = KernelPair::paired().await;
    let rid = pair.a.dispatch_once(intent()).await?;
    // 同一个 request_id 重发：对端不应执行两次
    pair.a.resend_same_request_id(rid).await?;
    assert_eq!(pair.b.executed_count(&plugin_id()), 1);
    assert_eq!(pair.a.last_outcome(&rid), Some(SendOutcome::Acked { .. }));
}

#[tokio::test]
async fn session_times_out_when_peer_is_silent() {
    // [11 A4] 的回归测试
    let pair = KernelPair::fresh().await;
    pair.b.make_silent();                          // 收到但不回应
    let err = pair.a.request_pairing(pair.b_id).await;
    assert!(matches!(err, Err(SessionError::Timeout)));

    pair.settle().await;
    // ★ 关键：发起端必须回到确定状态，不能永久"等待中"
    assert_eq!(pair.a.pairing_state(pair.b_id), PairingState::Idle);
}
```

**`KernelPair` 必须提供的能力**：

| 能力                            | 用途                                          |
| ------------------------------- | --------------------------------------------- |
| 内存传输（无网络）                | 快、可重复、CI 友好                             |
| 虚拟时钟 `advance(d)`             | 测超时/心跳/退避，不真的 sleep                    |
| 故障注入                          | `kill` / `partition` / `drop_chunk` / `corrupt_chunk` / `make_silent` / `delay` |
| 事件断言                          | `expect_event_a(…)` / `expect_event_b(…)`       |
| 状态查询                          | `link_state` / `trust_level` / `pairing_state`   |
| 完整内核 API                      | 不是简化版（否则测不出真问题）                     |

---

## 5. 恶意插件对抗测试（安全关键）

**每个用例是一个故意的恶意 Wasm 插件（或恶意 UI 插件）。**
这是整套测试里最重要的一组：安全属性在"写错"时功能测试照样全绿，只有对抗测试能发现。

> ### ★ 而这张表的 29 条，**真实状态是什么**（第 71 轮逐条核对）
>
> 下面那句“未做：没有”已经过时了 —— 大多数用例在之后的轮次里被逐个覆盖，
> 只是**分散在各自的模块里**。按“未做”去写会写出已经存在的测试，而比那更糟的是它会**掩盖真正没有的那几条**。
>
> | 用例 | 状态 | 证据 |
> | --- | --- | --- |
> | 1 死循环 | **✓ 真实 guest**（第 72 轮） | `a_guest_that_loops_for_ever_is_stopped_by_its_instruction_budget` |
> | 3 内存上限 | **✓ 真实 guest**（第 72 轮） | `a_guest_cannot_grow_past_the_ceiling_its_budget_names` |
> | 2 超时 | **✓ 真实 guest（第 73 轮）**：死循环的 guest 被 deadline 截断 | `a_wall_clock_budget_arms_a_deadline_on_the_store` |
> | 4、5 栈 / 越界 | **配置被证明装上了** ⚠ | 见下面 |
> | 6 panic | ✓ | `a_plugin_that_panics_the_host_does_not_stop_the_kernel` |
> | 7 未声明的 host function | ✓ | `IMPORTS_UNKNOWN` 档位 + linker 的导入表 |
> | 8–10 路径逃逸 / symlink | ✓ | `check_path` 的 `OutsideSandbox`、`is_within`、`expand.rs` **直接拒绝 symlink** |
> | 11–13 scheme / 环回 / 169.254 | ✓ | `synra_core::network` 的守卫测试（各有一条） |
> | 14 重定向到白名单外 | ✓ | `a_redirect_to_a_host_nobody_granted_is_refused_at_that_hop` |
> | 15 DNS 重定向 | ✓ | `FixedResolver`：只交出已校验的地址 |
> | 16 事件命名空间 | ✓ | `EventName::check_emittable_by` |
> | 17 `device.send` 未授权 | ✓ | gate 的 `CapabilityDenied` 测试 |
> | 18 高频调用 | ✓ | `RateKind` 五维，`rate_kind()` 分类 |
> | 19 超大 payload | ✓ | `PayloadTooLarge`（包体与帧两处） |
> | 20 递归调 action | **未实现** ✗ | `action_invoke` 在两个宿主里都拒绝 |
> | 21–24 伪造 manifest / 摘要 / 签名 / 公钥 | ✓ | `verify`、`DigestMismatch`、`BadSignature`、`Untrusted` |
> | 25–27 zip bomb / tar 穿越 / symlink | ✓ | `artifact::expand::plan`：条目数、单文件、总量、深度、symlink 全有测试 |
> | 28 宿主 panic | ✓ | `catch_unwind` 的测试 |
> | 29 `can_handle` 产生副作用 | ✓ | `HostCalls::pure()` + 纯会话拒绝副作用的测试 |
>
> ★ **而第 96 轮把那修复的另一半也钉住了**：关掉 `timeout: None` 的 deadline 只有在
> `timeout: Some(_)` **仍然会开**、仍然能停下真客体时才是对的。
> `a_real_guest_that_loops_is_stopped_by_the_wall_clock_not_by_zero` 跑一个真的死循环，
> 燃料给足以跑过时钟，所以停下它的必须是**墙上的钟**；
> 并且断言结果里的 `after` **就是预算的那个时长**（250ms）而不是零 ——
> `after: 0ns` 正是第 95 轮那个 bug 的签名。
> ★ 取反对照：把燃料改成 1000⇒红，报 `got FuelExhausted { fuel: 1000 }`，
> 证明这条测试分得清两层限制。
> ★ **而第 95 轮把它做成了最强形式：真的客体，穿过 Kernel**：
> `crates/synra-core/src/dispatch/tests.rs` 的 `a_real_guest_that_loops_is_stopped_without_costing_the_others`。
> 前一条证的是**内核对结果的处理**，引擎自己的测试证的是**引擎会产生这些结果**，
> 但没有人把一个**真客体穿过 Kernel** 跑过 —— 而那才是用户拥有的性质。
> 现在一个真的 `(loop (br))` 被装进 `PluginSet`，引擎真的数指令并真的停下它，
> 而旁边那个真插件**连续两次**照常工作。
> ★ **而它当场抓到了一个真的 bug**：预算里 `timeout: None` 的调用
> 会保留**实例化时为防 start 函数跑飞而设的 `set_epoch_deadline(1)`**，
> 于是执行器下一次 epoch 推进就中断它，`classify` 报出 `Timeout { after: 0ns }` ——
> **一个根本没有时限的预算被判为超时**。现在 `None` 会把 deadline 真正卸下。
> ★ **而它解释了一个旧的“偶发”**：第 75/76 轮记录过一次无法复现的失败，
> 当时只能写“只出现在全工作区跑的时候”。原因就是它：中断需要 epoch 恰好在调用飞行时推进，
> 而那只在并行负载下自己发生。新测试故意**先**推进 epoch 再调用，所以不是竞态。
> ★ **而“卸下”该用什么值也是跑出来的**：`u64::MAX` 在 wasmtime 内部
> **直接 panic**（`attempt to add with overflow`，store.rs），因为它把 deadline 加到当前 epoch 上。
> 改用 `1 << 40`（约三百年，且不会溢出）。
> ★ **而第 94 轮把那句关键的“内核仍可用 + 其它插件仍可用”补上了**：
> `crates/synra-core/src/dispatch/tests.rs` 的
> `every_way_a_plugin_can_misbehave_leaves_the_others_working`。
> 原来只有两条隔离测试（一条为 trap、一条为 host panic），而脚本档位能表达 **7 种**失败。
> 现在一张表跑六种：报告失败、trap、燃料耗尽、内存上限、超时、host panic。
> ★ **而断言包括“之后仍可用”**：每一种失败后再发一次共享，
> 它必须仍然被那个没坏的插件服务 —— 一个“恢复了但把 set 弄坏了”的调度
> 会通过上面每一条断言，然后在下一次共享上失败。
> ★ **而它证什么、不证什么写在测试里**：它证**内核的处理**（给定这个结果，
> 其他插件仍被问、用户仍拿到提示），**不证引擎会产生这些结果**（那是第 72
> 轮的燃料/内存真档位测试）。两半同属一个性质，缺一不可。
> ★ **而取反对照时第一次打在了错的分支上**：我先改的是 `match_share` 的
> `Err(PluginSetError)` 臂，而 trap/超时走的是 `Ok(CallOutcome)` 在 `take_answer` 里分类。
> 那一版取反对照**绿了** —— 而一个通过的取反对照并不证明测试是空的，
> 它证明的是**取反对照打错了地方**。改到正确的分支后报出 `a trap: both were asked`。
> ★ **而第 72 轮把其中两条从“只在档位上”变成了真的**：用 `deactivate: func()`（**无参数、无返回值**）
> 做档位。它的 canonical lift 就是一个没有签名的 core 函数，所以手写 `.wat` **四行就够** ——
> 而指令预算和内存上限是**引擎**的性质而不是接口的，在 `activate` 里死循环和在 `deactivate` 里死循环
> 测的是**同一个限制**。于是：
>
> * **燃料**：一个真正死循环的 guest 被停下，结果是 `FuelExhausted { fuel: 100_000 }`；
>   阴性对照（让 guest 直接返回）⇒ 测试红。
> * **内存**：一个真正一直 `memory.grow` 的 guest，
>   在被拒后自己 `unreachable`（因为被拒的 `memory.grow` **不 trap，只返回 -1**），
>   结果是 `MemoryExceeded { limit }`，而断言比的是**调度给的那个数**：把预算改成 4 MiB
>   而期望 1 MiB ⇒ 红（它不会因为引擎自己的 16 MiB 上限而假绿）。
>
> ★ 而一个死档位被删了：原来那个把循环 lift 成 `activate` 的 `SPINS`
> **没有任何人调用**。一个死档位比没有档位更糟：它读起来像覆盖。
>
> ★ **而 `string -> string` 这条路还是带着证据地 `#[ignore]` 着**（第 73 轮）：试了两种写法，
> 引擎对每一种都给了确切的回答，而这两句话现在写在 `#[ignore]` 里：
>
> * 把返回区指针当**前置参数**：`lowered parameter types [I32, I32] do not match parameter types
>   [I32, I32, I32]` —— 也就是说降下来的参数**只有输入串**，没有返回指针；
> * 把返回区地址当**平的 i32 结果**：编译过，运行时 trap `expected 1 result(s), got 0`。
>
> ★ 而它的断言也被换了：原来是 `assert!(!text.is_empty() || text.is_empty())` ——
> **一个对任何字符串都成立的断言**，所以即使档位修好了它也什么都证不了。
> 现在它比的是那个 21 个字符的答案本身。
>
> ★ 而另两个限制测试（超时、内存）**之前一直围绕一个 trap 在断言**：
> 它们用的档位同样有 ABI 错误，所以那次调用立刻 trap，而它们断言的 deadline/ceiling
> 是在**一次根本没跑起来的调用**上被装上的。现在它们用那个真正会死循环的 guest。
>
> #### ⚠ 1–5 的真实状态，比“只在档位上”更精确也更值得记
>
> **全仓只有两个 `#[ignore]`，而它们正是证明真实 guest 能跑通的那两个**，都在 `wasmtime_tests.rs`：
>
> * `:270` `a_call_reaches_the_guest_and_its_answer_comes_back`
> * `:306` `a_guest_that_loops_for_ever_is_stopped_by_its_instruction_budget`
>
> 属性里写的理由是：**手写的 `.wat` 档位不符合 canonical ABI 对 `string` 参数与返回值的要求**，
> 而修好它的路径是**用 `bindgen!` 从 WIT 文件生成档位**。换句话说：查找、实例化、
> 安装内存上限、装配 epoch 都有测试，**而“一次完整的调用真的进了 guest 并回来了”没有**。
>
> ★ 而 `a_wall_clock_budget_arms_a_deadline_on_the_store` 与
> `a_memory_ceiling_is_installed_before_the_guest_runs` 是**真断言**：它们断言实例实际被给了多少 tick、
> 以及上限是否装上，而不是断言一个档位的回答。所以这一行的真实状态是：
> **配置被证明装上了，没被证明对一个真会死循环的 guest 生效。**
>
> ★ **而一个错误的归因在这里被纠正了**：这两个 `#[ignore]` 一度被说成“因为缺 `wasm32-wasip2` 而阻塞”。
> `unsafe`/target/`wasm-tools` 那三个理由是 `crates/synra-plugin-sdk/tests/guest/world_fixture.rs` 的，
> 而那个文件**根本不参与编译** —— 它自己写明了“没有 `#[ignore]`，因为编译不了的测试不是被跳过的测试”。
> 两件事各自成立，不是一件。
>

### 5.1 headless 插件

| #  | 攻击                                             | 期望结果                                       |
| -- | ------------------------------------------------ | ---------------------------------------------- |
| 1  | 死循环（纯计算）                                   | `FuelExhausted`；内核继续正常工作                |
| 2  | 死循环 + 阻塞 host 调用                            | epoch 中断；`Timeout`                           |
| 3  | 分配 4 GiB 内存                                   | 内存限制 trap；实例销毁；**宿主内存不增长**        |
| 4  | 栈溢出（深递归）                                   | trap；实例销毁                                  |
| 5  | 越界内存访问                                      | trap；**不影响宿主**                             |
| 6  | panic（guest `unreachable`）                       | trap；返回错误（不杀死内核）                      |
| 7  | 调用未声明的 host function                         | **装载期**拒绝（imports ⊄ capabilities）          |
| 8  | 读 `/etc/passwd`                                  | `PathNotGranted` / `Escape`                     |
| 9  | `../` 逃出沙箱                                    | `Escape`                                        |
| 10 | symlink 逃逸                                      | `Escape`                                        |
| 11 | `network.fetch("file:///etc/passwd")`              | `SchemeNotAllowed`                              |
| 12 | `network.fetch("http://localhost:8080/admin")`     | `IpNotAllowed`（环回）                           |
| 13 | `network.fetch("http://169.254.169.254/")`         | `IpNotAllowed`（链路本地 / 云元数据）              |
| 14 | `network.fetch` 后重定向到白名单外                   | 重定向被拦截                                     |
| 15 | DNS 重绑定（首次白名单 IP，第二次内网）               | 用解析后的 IP 连接 + 每跳重校验 → 拦截             |
| 16 | `event.emit("_plugin.other.something")`            | `EventNamespaceViolation`                       |
| 17 | `device.send` 到未授权设备                          | `CapabilityDenied`                              |
| 18 | 高频调用 host（1e6/s）                             | 令牌桶限流 `RateLimited`                         |
| 19 | 发送超大 payload（> 256 KiB）                       | `PayloadTooLarge`                               |
| 20 | 递归调用 action（A→B→A）                           | `ActionChainTooDeep`                            |
| 21 | 虚假 manifest（声明 1 个能力，wasm import 10 个）     | 装载期拒绝                                      |
| 22 | 篡改的 wasm（摘要不匹配）                           | `DigestMismatch`                                |
| 23 | 伪造签名                                          | `BadSignature`                                  |
| 24 | 有效签名但发布者公钥变了                             | `Untrusted`（id 与公钥绑定）                     |
| 25 | zip bomb（`ui.zip` 解压到 10 GB）                   | `ArchiveTooLarge`                               |
| 26 | tar 里的路径穿越条目（`../../x`）                    | 拒绝                                            |
| 27 | tar 里的 symlink 条目                              | 拒绝                                            |
| 28 | host function 里宿主 panic                          | `catch_unwind` 捕获；实例销毁；内核存活            |
| 29 | `can_handle` 产生副作用                             | 契约测试失败（audit 非空）                         |

**每条都要有"内核仍然可用"的断言**，而不只是"这个调用被拒了"：

```rust
#[tokio::test]
async fn malicious_plugin_loop_does_not_harm_kernel() {
    let host = PluginHostFixture::install("malicious-loop").await;
    let err = host.call("can_handle", args).await.unwrap_err();
    assert!(matches!(err, HostError::FuelExhausted { .. }));

    // ★ 关键：内核仍然服务
    assert!(host.kernel().snapshot().await.is_ok());
    // ★ 并且：其它插件仍可用
    assert!(host.call_other_plugin("healthy", "can_handle", args).await.is_ok());
}
```

### 5.2 UI 插件

| #  | 攻击                                          | 期望结果                                    |
| -- | --------------------------------------------- | ------------------------------------------- |
| 1  | `fetch('http://evil.com')`                      | 被 CSP `connect-src 'none'` 阻止             |
| 2  | `window.parent.document.body`                   | 跨 origin 被阻止                             |
| 3  | `localStorage.setItem(...)`                     | 无持久化（独立 profile），不影响宿主            |
| 4  | 用其它插件的 token 调 IPC                        | 拒绝（token 与 plugin_id 绑定校验）            |
| 5  | 调 `plugin_ui_rpc("plugin_install", ...)`        | 方法白名单拒绝                                |
| 6  | 调 `plugin_ui_rpc("platform_simulate_keys", ...)` | 方法白名单拒绝（UI 插件永不支持）              |
| 7  | `eval` / `new Function`                          | 被 CSP `script-src 'self'`（无 `unsafe-eval`）阻止 |
| 8  | 停用后继续调 IPC                                 | 拒绝（token 已撤销）                          |
| 9  | 导航到外部 URL                                   | `on_navigation` 拦截                          |
| 10 | 无限循环（阻塞自己的 UI 线程）                      | ★ WebView 层面无法中断——**已知限制**，见下       |

> **第 10 条是 UI 插件的真实限制**：WebView 里的 JS 无法被内核中断。缓解：
> ① 插件 WebView 独立进程（Chromium 系）→ 崩溃不影响宿主；
> ② UI 提供"这个插件无响应，强制关闭"；
> ③ 文档明确说明 UI 插件隔离**弱于** headless 插件（[11 B1](./11-design-principles.md)）。
> **诚实记录限制比假装没有更好。**

---

## 6. 敏感数据测试

```rust
/// ★ 日志里绝不能出现 payload
#[test]
fn logs_never_contain_payload() {
    let cap = TestSubscriber::capture();
    tracing::subscriber::with_default(cap.clone(), || {
        let payload = json!({ "password": "hunter2", "token": "secret-abc" });
        tracing::info!(?payload, "sending");
    });
    let out = cap.output();
    assert!(!out.contains("hunter2"));
    assert!(!out.contains("secret-abc"));
}

/// ★ 诊断包不含密钥
#[test]
fn diagnostic_bundle_excludes_secrets() { /* ... */ }

/// ★ 审计记录只记能力，不记内容
#[test]
fn audit_records_capability_not_content() { /* ... */ }
```

---

## 7. 前端与集成回归（app 仓库）

| 项                        | 方式                                                        | 状态（2026-10-18） |
| ------------------------- | ----------------------------------------------------------- | ------------------ |
| 组件与 store 单元测试       | Vitest                                                      | ✅ 170 条：三个 store 的 `apply` 契约、`actions.ts` 的参数形状、`events.ts` 的接线 |
| 页面挂载测试               | Vitest + `jsdom` + `@vue/test-utils`                        | ✅ 四页各有一份 `*.mount.test.ts`（共 13 条）；默认环境仍是 `node` |
| 事件桥接（含未知事件兜底）   | Vitest：注入未知 `KernelEvent.type` → 断言触发全量刷新          | ✅ 用**虚构**类型名，因为测的是接线而非内核枚举 |
| Lint                       | `eslint .`（CI 已跑）                                        | ✅ 0 问题；`--fix` 与 `command-arguments` 守卫曾互相打架，见 08 §9 |
| UI 插件隔离               | §5.2 的对抗用例                                 | ⬜ 插件 UI 宿主尚不存在 |
| 桌面端到端                | `tauri-driver`：启动 → 渲染四页 → 配对 → 分享 → 执行           | ⬜ 未做 |
| WebView 引擎兼容          | CI 在 WebView2 / WebKitGTK / WKWebView 上跑视觉回归             | ⬜ 未做 |

★ **一个被自己写错、随后纠正的结论**（2026-10-18）：我曾在这里写下"`shell.rs` 只在 `tauri-shell` feature 下编译，
而本机与 CI 都跑 `--no-default-features`，所以改动它无法在工作区内验证"，并据此推迟了发现事件的接线。**这是错的**：
本机 `cargo check --manifest-path src-tauri/Cargo.toml`（默认 feature）**能通过** —— 需要 WebKitGTK 的是 **CI 的
Ubuntu runner**，不是这台开发机。

正确说法是：

* **测试**只在 `--no-default-features` 下跑（失败原因清楚、跑得快），所以 `shell.rs` 里**没有**测试；
* **编译**两种 feature 都能在本机验证：改 `shell.rs` 之后必须跑 `cargo check`（默认），否则那种错误只有在真正的桌面
  构建里才会暴露；
* 因此"要改 `shell.rs` 的活做不了"是假限制。真正的限制是**它没有测试**，只能用类型与编译护栏。

教训：**先验证限制本身，再按限制做决定** —— 一个没被验证过的约束，可能一直在无谓地缩小可选方案。

---

## 8. 性能基准与门禁

### 8.1 基准清单

| 基准                              | 目标       | 门禁            |
| --------------------------------- | ---------- | --------------- |
| 帧编解码（信封，1 KiB）             | < 5 μs     | 回归 > 20% 失败  |
| 帧编解码（二进制块，256 KiB）        | < 20 μs    | 同上            |
| capability 匹配（10 条授权）         | < 200 ns   | 同上            |
| capability 解析（含规范化）          | < 500 ns   | 同上            |
| host function 调用（含闸门 + 审计）   | < 20 μs    | 同上            |
| Wasm 实例获取（池命中）              | < 5 μs     | 同上            |
| 内核启动到 `start()` 返回            | < 200 ms   | 同上            |
| 插件激活（编译缓存命中）              | < 50 ms    | 同上            |
| 一次本机意图（匹配 + 执行）           | < 20 ms    | 同上            |
| 一次跨设备意图（本机回环）            | < 50 ms    | 同上            |
| 传输吞吐（256 KiB 块，本机）          | > 100 MB/s | 同上            |
| 内存（10 个插件激活 + 空闲）          | < 120 MiB  | 同上            |
| 桌面冷启动到可交互                   | < 500 ms   | 同上            |

### 8.2 CI 门禁

```rust
// benches/gate.rs —— iai-callgrind：指令数比墙上时间稳定
iai_callgrind::library_benchmark_group!(
    name = hot_paths;
    benchmarks = [capability_match, frame_decode_envelope, gate_check]
);
```

**双轨**：`iai-callgrind` 做 CI 门禁（确定性强，回归检测可靠），
`criterion` 做本地真实耗时与优化验证。

### 8.3 零分配断言（比基准更早的警报）

热路径的**硬性要求是零分配**（[26 §4.2 P1](./26-core-stability-and-perf.md)）。
基准测试要跑到统计门禁才失败，而分配计数是**确定性**的 —— 它能在"性能还没退化到门禁线"时就报警。

```rust
use synra_testkit::AllocCounter;

#[test]
fn frame_decode_does_not_allocate() {
    let counter = AllocCounter::install();
    let mut dec = FrameDecoder::new();
    dec.push(Bytes::from_static(ENVELOPE_FRAME))?;
    let _ = dec.next_frame()?;
    assert_eq!(counter.allocations(), 0, "帧解码必须零分配（[26 §5.4]）");
}

#[test]
fn capability_match_does_not_allocate() {
    let counter = AllocCounter::install();
    let _ = GRANTS.contains(&cap);
    assert_eq!(counter.allocations(), 0, "capability 匹配必须零分配（[26 §5.2]）");
}

#[test]
fn event_routing_does_not_allocate() {
    let counter = AllocCounter::install();
    let _ = registry.route("_synra.device.changed");
    assert_eq!(counter.allocations(), 0, "事件路由必须零分配（[26 §5.3]）");
}
```

**覆盖清单**（[26 §4.1](./26-core-stability-and-perf.md) 的 8 条热路径）：
帧编解码、信封解析、事件路由、capability 匹配、事件入队/出队、传输块路径、状态转移、实例池获取。

> ⚠️ **不要滥用**：只对"声明为零分配"的函数断言。给非热路径加零分配断言会把设计钉死，
> 与 [26 §4.1](./26-core-stability-and-perf.md)（非热路径以清晰为先）矛盾。

---

## 9. 覆盖率要求

| crate               | 行覆盖 | 分支覆盖 | 说明                          |
| ------------------- | ------ | -------- | ----------------------------- |
| `synra-capability`   | ≥ 95%  | ≥ 90%    | 安全关键，纯函数，应全覆盖       |
| `synra-protocol`     | ≥ 90%  | ≥ 85%    | 协议正确性关键                  |
| `synra-net`          | ≥ 80%  | ≥ 70%    | IO 密集，部分路径难覆盖          |
| `synra-plugin-host`  | ≥ 85%  | ≥ 75%    | 安全关键                       |
| `synra-core`         | ≥ 80%  | ≥ 70%    | 编排逻辑                       |
| `synra-store`        | ≥ 75%  | —        | 落盘与迁移                     |
| `synra-platform`     | ≥ 60%  | —        | 大量平台分支，用 mock 覆盖关键路径 |
| `apps/*`             | —      | —        | 薄层，不强制                    |

> 覆盖率是**下限**，不是目标。安全关键路径应该**穷尽**（表驱动 + proptest），而不是"达到 95%"。

---

## 10. 手工 / 探索性测试清单

自动化成本过高或不可靠的，留在这里。

### 10.1 内核相关（core 仓库）

| 项                              | 平台       | 频率    |
| ------------------------------- | ---------- | ------- |
| 企业 WiFi（UDP 被拦）下的发现回退  | 桌面 + 移动 | 每里程碑 |
| 睡眠/唤醒后重连                  | 桌面       | 每里程碑 |
| 系统时间跳变（手动改系统时间）      | 桌面       | 一次    |
| 磁盘满时的行为                    | 桌面       | 一次    |
| 无网卡 / 只有 lo 时的行为          | 桌面       | 一次    |
| 系统代理存在时的网络行为           | 桌面       | 一次    |
| 防火墙拦截端口时的用户提示          | Windows    | 一次    |

### 10.2 应用相关（app 仓库）

| 项                              | 平台        | 频率    |
| ------------------------------- | ----------- | ------- |
| 真机 Android：分享菜单 → Synra → 执行 | Android  | 每里程碑 |
| 真机 iOS：分享扩展 → Synra         | iOS         | 每里程碑 |
| 移动端切后台 30 分钟后回前台        | Android/iOS | 每里程碑 |
| 多显示器 / 高 DPI 的 UI 渲染        | 桌面        | 一次    |
| 安装包与自动更新流程               | 三平台       | 发版前  |

**结果记录**：每项做成 checklist 条目，附环境信息与截图。

---

## 11. 生态约定（沿用本机既有项目已验证的做法）

| 约定                                | 理由                                                          |
| ----------------------------------- | ------------------------------------------------------------- |
| **等就绪，不要 sleep**                | flaky 通常意味着"等就绪的方式不对"；加重试只会把它藏起来             |
| **端口一律用 `0`**，从 `local_addr()` 取 | 固定端口在 CI 并行时互撞，是 flaky 的主要来源                       |
| **测试不访问外网**                    | registry 测试用本地 HTTP server 伪造；其他用回环                    |
| **不用外部命令生成证书**               | 用 `rcgen` 现场生成 → TLS 断言**永远会跑**，而不是"缺工具就跳过"       |
| **用 `tempfile`，不碰真实用户目录**     | `Paths::for_test(dir)` 注入                                     |
| **安全属性必须显式断言**               | "连接失败就算通过"是反模式（[§5](#5-恶意插件对抗测试安全关键)）          |
| **断言路径上不允许 `sleep`**           | 只用"等到条件成立或超时"的 helper；`Delay` 只出现在 mock 剧本里        |

---

## 12. 测试基础设施清单

> ★ 这一节在实现之前写就，长期整节未勾选。逐条核对源码后，它是**两类东西混在一起**：
> **已经做了的测试**（夹具、协议 fixture、敏感数据断言、迁移测试）与
> **仍然缺的基础设施**（nextest 配置、benches、fuzz、覆盖率门禁、性能门禁、justfile）。
> 前一类里几条只是**位置和文档写的不一样**；后一类是**一个都没有**，而它们曾经共有一个缺失的前提：CI。
>
> ★ **那个前提在 2026-10-18 补上了一半**：`synra-core` 与 `synra-app` 各有一个 `checks` workflow，跑的是
> clippy / test / MSRV / guards / matrix（app 另加 test / build / shapes）。所以下面几行里"没有 CI"之类的理由
> 已经过期 —— 缺的不再是"能不能自动跑"，而是**每一项自己的门槛**（覆盖率没有测、基准不存在、fuzz 不存在）。

- [ ] `cargo nextest` 配置（超时、retries、test groups）—— **未做**：
      `.config/nextest.toml` 与 `justfile` 都不存在。测试现在靠 `cargo test` 跑，
      而 `SynraDb` 那条 `rusqlite` 的 `Connection` 是 `Send` 不是 `Sync`，
      所以"超时 / retries"这类配置确实还没到需要的时候。
- [x] `synra-testkit`：夹具 + 故障注入 —— **3/4，且这一条是清单里最完整的**。
      `KernelPair`(`src/pair/`)、`MemoryTransport`/`MemoryTransportLink`(`src/transport.rs`)、
      `MockClock`/`SystemClock`/`Deadline`(`src/clock.rs`) 都在，加一个计数分配器。
      故障注入是真的：`KernelPair::inject(Side, Fault)`、`cut()`、`attempts()`、`dropped()`。
      ★ **`MockPlatform` 全仓 0 命中** —— 这是四样里唯一没有的。
      ★ 另外 `KernelPair` 有更深的缺口，不在这条清单里但更要紧：
      **它在 `synra-testkit` 之外 0 处使用**，22 条测试跑的是它自己的 `FakeKernel`，
      而且 `PeerLink` 持的是具体 `TcpStream`、`MemoryTransport` 搬的是已解码的 `Frame`，
      两者类型上不兼容（详见 [04 §11](./04-wire-protocol.md)）。夹具测的是夹具自己。
- [x] `fixtures/wire/**`：协议 fixture —— **A，只是不在这个目录**。
      真实位置 `crates/synra-protocol/tests/wire_fixtures.rs`（261 行），
      而且做法比"录一遍编码器的输出"更硬：**手写字节**、拆成具名片段、
      三向断言（片段加起来等于文档布局 / 编码器产出这些字节 / 解码器还原同一帧）。
      文件顶部的注释解释了为什么不能"录"：录只能证明代码等于它自己。
- [ ] `benches/`：criterion + iai-callgrind 双轨 —— **未做**，目录不存在。
      ★ 与之相关的是 [26 §4.1](./26-core-stability-and-perf.md) 的**每一个性能预算都只是散文**：
      全仓无 `benches/`、无 `criterion`、无 `#[bench]`。唯一被测的是**分配次数**，不是时间。
- [ ] `fuzz/`：帧解码、信封解析、归档解包、路径解析、capability 解析 —— **未做**，目录不存在。
      现在只有 proptest 覆盖任意字节不 panic；property 测试与 fuzz 的输入族不同，前者通过不能当作后者的结论。
      ★ **而第 99 轮定位到了其中最具体的一条，并且它已经写在代码里**：
      `host_calls_tests.rs` 的 `a_pure_session_refuses_the_calls_that_act` 自己写着——
      *“它证明了拒绝本身有效，而**不证明有任何东西会构造出这样一个 session**”*，
      而 `Work::allows_side_effects` 到 store 那个标志的**两行是“没有任何测试走到的部分”**。
      ★ **而原因比“漏了一条测试”更有意思**：引擎测试里没有任何一个客体调用过主机函数。
      `instantiate` 不接受 session（store 的 `session: None`），只有 `PluginSet` 在每次调用前
      `set_session`。所以要走到那两行，需要一个**其 `can-handle` 会调用侧效应型主机函数的真客体**，
      加上一个带 grants 的 session（`LinkerSession::new`）与一个 audit ring（`with_audit`）——
      因为客体只要忽略返回值就能继续跑，**拒绝只能从审计里看到**。
      ★ 现成的两个配件已经能拼出它：`ANSWERS` 给了 `string -> string` 的 can-handle ABI，
      `IMPORTS` 给了 import instance 的写法；而侧效应函数选 `platform-open-url`（`func(url: string) -> outcome`，
      单个字符串）比 `log`（六个参数）省事。
      ★ **而第 100 轮写到了哪一步，以及卡在哪里**（配件已回退，因为半成品比没有更糟）：
      WAT 的骨架跑通了 —— import instance、`core func (canon lower ...)`、
      `(with "host" (instance ...))`、以及**为什么内存必须自己一个模块**（降低一个带
      `string` 参数的导入必须指定 memory，而那块内存属于客体，所以不能在它里）。
      ★ **而第 101 轮把它缩到了一行，并且推翻了上一轮的猜测**：
      用一组取反对照把它逼到了一个结论 —— **字符串参数本身没问题**：
      `(func (param "url" string))`（无返回值）能通过验证，报的是后面那个类型不匹配；
      而**任何返回 `outcome`（variant）的导入都会被 wasmtime 的校验器拒绝**，报
      `instance not valid to be used as import`。
      ★ **而它在三种写法下都在同一个偏移（0x5e）上报同一句话**：
      内联写在 `result` 里、写在 instance 类型内部、以及把 instance 写成命名 `(type $host_t (instance ...))`。
      这三种都失败，而且**失败与 core 层无关**：
      把 core instance 里的降低函数换成普通 core 函数（仍保留 variant 导入）依然报同一句。
      ★ **而这不像是 wasmtime 的缺陷**：真实工具链不把类型写在文本里，而是用
      `wasm-tools component embed` 把 WIT 类型**嵌进组件的类型段**，再由 `component new` 生成组件。
      手写路径下这一步无法表达，而 `wasm-tools` 不在离线缓存里。
      ★ 所以下一步不是“再换一种写法”，而是二选一：引入 `wasm-tools`；
      或者把对照放在**已有的组件配件**上——即用 `wasm-runtime` 真档位测试里已有的
      组件产物，而不是手写一个。
      `platform-open-url: func(url: string) -> outcome`，而 `outcome` 是一个 `variant`（`ok(string)` / `err(error)`，
      `error` 是一个带 `option<string>` 的 record）。客体导入的类型必须与 linker 定义的一致，
      而**类型写成一个命名 `$out` 引用时 wasm-tools 报
      `instance not valid to be used as import`**；内联形式的写法没有试到通。
      ★ **而真实工具链不这么写**：它用 `wasm-tools component embed/new` 从 WIT 生成配件，
      类型由 WIT 算出来，手写只是本工作区没有那个工具时的替代。所以下一步有两条路：
      ①把这个 variant 用内联类型写出来；②引入 `wasm-tools`（当前离线缓存里没有）生成配件。
      ★ **而本轮没有写它**，因为配件与 host 存根都是真活，写到一半比不写更糟。
- [ ] `crates/synra-plugin-host/tests/malicious/`：§5.1 的 29 个用例 —— **未做**，目录不存在。
      这是三份文档里最长的测试清单项，目前一个都没有。
      孤立的对抗测试是有的（归档穿越/symlink/体积、签名吊销、令牌伪造、装载期 import 覆盖），
      但 §5.1 那句关键的"**内核仍可用 + 其它插件仍可用**"没有任何测试断言。
- [ ] UI 插件对抗测试（app 仓库）—— **未做，而且前提不成立**：
      `synra-rust-app/src` 13 个 TS 文件里，`plugin_ui*` 只出现在
      `src/ipc/registration.test.ts` 的"未实现"清单里。
      没有插件 origin、没有令牌注入、没有 `on_navigation`，所以**没有可攻击的表面**。
      这一条要等 `plugin_ui_rpc` 那批命令先存在。
- [x] 敏感数据测试（日志、诊断包、审计）—— **A，散在两处而不是一个套件里**。
      日志：`synra-observe/src/logging/tests.rs` 的 `hunter2` 用例（8 处命中同一套策略）；
      审计：`crates/synra-observe/src/audit.rs:437` `a_record_has_no_place_to_put_a_payload`
      —— 而且这个保证是**构造性**的：`AuditEntry` 的字段没有一个能装内容；
      诊断包：`synra-rust-app/src-tauri/src/commands/diag/tests.rs:38`
      `the_exclusions_are_written_into_the_file`，加上本轮新加的
      `the_debug_log_section_carries_the_lines_the_shell_wrote`。
- [x] SQLite schema 迁移测试（本项目自己的 schema 版本演进）—— **A，而且相当扎实**。
      `crates/synra-store/src/db/tests.rs` 覆盖：版本号从 1 连续（`the_migration_list_is_a_sequence_from_one`）、
      空库识别（`schema_state_recognises_an_empty_database`）、
      更新的库被拒而不是被迁（`a_newer_database_is_refused_rather_than_migrated`）、
      空库跑完每一步（`applying_to_an_empty_database_runs_every_step`）、
      失败语句点名是哪条迁移的哪句（`a_failed_statement_names_the_migration_and_the_statement`）、
      已发布的迁移不被改写（`a_published_migration_is_never_edited`）、
      以及列清单与迁移 schema 一致（`records.rs:766`）。
      ★ 实际迁移数是 **7** 个（`migrations.rs` 的 `version: 1..=7`），不是 [17 §2.2](./17-storage-and-config.md) 写的 5 个。
- [ ] 覆盖率门禁（CI）—— **未做**：CI 有了（2026-10-18），但**没有任何地方测量覆盖率**，所以还没有一个可以
      设成门槛的数字。加一个 `cargo llvm-cov` 的 job 是第一步，第二步才是选一个阈值。
- [ ] 性能门禁（CI，iai-callgrind）—— **未做**：CI 有了，但基准本身仍然不存在（见上一条 `benches/`），
      所以门禁没有可比较的东西。
- [x] 手工 checklist（§10）—— **A**：§10「手工 / 探索性测试清单」在本文 L562 起，已经写在文档里。
      这一条要的是"清单存在"，它存在。
- [x] `just docs-check`（文档链接与锚点校验）—— **已有**（第 102 轮）
      ★ **而第 107 轮加了第二种文档检查：数量断言**。`tools/doc-claims` 读出
      “**全仓 N 命中/处 `X`**”这类句子，在代码里数一遍，不一致就报。
      它之所以存在，是因为**这类句子已经烂了两次**：`GcTask` 被说成“0 命中”而实际有 1，
      `config.json` 被说成“无人读取的路径常量”而实际已是 9 文件、533 行测试的模块。
      ★ **而它对自己也不客气**：跑的过程里它发现自己的**测试名字也在被数**（用真名做 fixture），
      于是 fixture 全换成不存在的名字；它也分不清**“原状：0 命中”**（历史）与当下，
      所以那类句子被跳过。最后：**11 条声明，0 条不一致，3 条无法核查**（模式而非名字）。
      ★ **而它当场找到了一条真的**：第 11 文档说 `remove_dir_all` “42 处，全部在测试里”，
      而实际是 **71 处**，且生产侧确实有调用点（`sandbox.rs` 删自己的 cache 与 root）。该行已重写。，但**不是 `just`**：
      `synra-rust-core/tools/doc-links`，一个无依赖的 Rust 工具，
      `cargo run -q -p doc-links -- <目录>...`。它检查：文件不存在、锚点不存在（含同文件 `#section`）。
      ★ **而它第一次跑就报了 110 个问题，几乎全是假的**：`/zh/plugins/manifest` 这种路径是
      **站点根相对**的，而不是相对于当前文档 —— 把它们当成后者就会在几乎每一行上报错。
      **一个什么都报的检查与什么都不报的一样没用**，而且更坏：诚实的反应是不再读它的输出。
      于是解析器学会了四种形状（直接路径、补 `.md`、`index.md`、`public/`），而剩下的 **2** 个是真的：
      `08` 指向一个已改名的文件（`09-ui-plugin-model.md` → `09-plugin-system.md`），
      `24` 的例子链接指向 `synra-app/design-docs/`（迁移后已不存在）。两处已修。
      ★ **而工具自己也被真语料找出一个 bug**：`strip_code` 把行内代码段抹成空白，
      对**链接**是对的（反引号里的链接是例子），对**标题**是错的 —— GitHub 的锚点保留反引号里的词，
      所以 `### 10-2 ★★ `can_handle` 的边界` 的锚点含 `can_handle`。抹掉它会把**就在页面上的标题**报成断链。
      现在两条路径分开，并有一条用真实标题写的测试。
      ★ **而第 103 轮把它变成了守卫的第 8 条**，而不是一个靠人记得跑的工具：
      为此 `doc-links` 拆成了 lib + 薄前端（与 `api-guard` 同一形状），`guards` 直接调它的函数。
      ★ **而它只能是“第三种答案”**：文档在**三个**仓库里（本仓库 + 两个兄弟），
      而只 clone 本仓库是正常的。兄弟不在时诚实的答案是 **`SKIP` 并说出原因**，
      而不是 `ok` —— 后者等于说“链接都查了、都好”，而实际一行都没读。
      ★ **而它第一次跑就在本仓库里找到 3 条断链**（之前手动跑没发现，因为当时只指向了两个文档仓库）：
      `NOTES/` 和 `spike/` 下的文件用 `../../` 指向仓库根下的路径，而它们本身就在根下一层 ——
      **多一层 `../` 的错误读起来和对的一模一样**，而且它们写于本会话。现在：**81 篇、0 问题**，而守卫一起跑（`cargo run -q -p guards`）。
      ★ 仍可单独跑：`cargo run -q -p doc-links -- ../design-docs ../docs` —— 目前 **71 篇、0 问题**。
      它**不检查外部链接**：那需要网络，而那正是“每次提交都能跑”与“想起来才跑”的区别。
      ★ 这一条现在比写它的时候更有用：本轮与上一轮我在这套文档里**改了大量行号与文件名引用**，
      而没有任何东西校验它们。链接与锚点是会烂的，而烂掉的链接指向的是**看起来对、实际不存在**的位置。
