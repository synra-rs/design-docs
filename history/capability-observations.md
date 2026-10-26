# 能力系统旧实现观察

原 10 §8 的调查记录，归档于 2026-10-26；仅保存当时的源码观察、行号和失败证据。
其中的旧事件名称及“尚未实现”判断不定义当前规范，当前契约见 [10](../10-capability-and-sandbox.md)，状态见 [19](../19-roadmap.md)。

## 8. 实现清单

> ★ 这一节在实现之前写就，长期整节未勾选。以下每一条都重新核对过源码。
> **能力层（636-639、644、646、647）是真的、有真测试的；缺的是路径陷阱、网络守卫和恶意插件套件。**
> 名字对不上的按真实名字记（标 `C`）。

- [x] 解析器 + 常量 —— **C**：真实路径是 `crates/synra-capability/src/capability/syntax.rs`
      （不是 `synra-capability::syntax`）；`parse` 在 `:142`，`MAX_LEN = 256`(`:20`)、
      `MAX_SEGMENTS = 6`(`:17`) 与 §2.1 逐字相符；`capability/tests.rs:49-100` 覆盖
      `BareWildcard` / `EmptySegment` / `WildcardPrefix`。
      ★ 但"规范化（含各 domain 的 qualifier 规则）"被代码**明确放弃**了：
      `syntax.rs:316-322` 的注释说按 domain 分开做就是"五个解析器共用一个调用方"，
      `:323` 的 `is_qualifier_segment` 用一套字符类 `[A-Za-z0-9._@/*-]` 管所有 domain。
      没有 host 小写化、没有去尾点、没有 punycode，也没有 fs qualifier 的 canonicalize。
- [x] `*` / `**` 匹配 + 表驱动边界用例 + proptest —— **A**：
      `capability/matching.rs:50` 的 `matches`（一套算法同时处理 prefix 与 qualifier）、
      `:210` `glob_matches`；`capability/tests.rs:202-226` **逐行**搬了 §3.3 的全部十行
      （含 `:213` `ui:*` vs `ui:page:extra` = ❌、`:222` `network:fetch:api.x.com` vs `…:POST` = ❌）；
      proptest 在 `capability/proptests.rs` 与 `set/proptests.rs:32`
      `contains_does_not_depend_on_insertion_order`。
      模块名是 `capability::matching` 而不是 `pattern`，纯命名差异。
      ★ 本次另修：`matches` 原先**每调用分配两个 `Vec`**，已改 `SmallVec`，2 → 0 次分配
      （见 [11](../11-design-principles.md) 与 [26](../26-core-stability-and-perf.md)）。
- [x] `CapabilitySet`、包含关系 —— **C/A**：在 `set/limits.rs:29`（模块叫 `set` 不是 `grant`）；
      `contains`(`:126`)、`grants`(`:150`)、`iter`(`:175`)、`patterns`(`:183`)。
      ★ "通配展开（用于 UI 展示）"其实**不是展开**：`describe_all()`(`:193`) +
      `Capability::describe()`(`capability/consent.rs:15`) 把通配**渲染成一句话**
      （"read from any website"）。没有任何地方把 `network:fetch:**` 展开成具体能力列表。
- [x] `CapabilityDenied` 结构化错误（含被拒能力、**已授予**、**修复建议**）—— **三要素已完成**（第 74 轮）：
      新模块 `synra_capability::error::CapabilityDenied`（正是下面那句说缺的那个），
      而 `CallError::denied` 现在**接过会话自己的授予集**，`details` 是三个字段：
      `{"capability":…,"granted":[…],"suggestion":"…"}`。
      ★ **而建议是从两个集合推出来的，不是手写的**：它不只说“去要 X”，
      还会看插件**是否已经拥有同域的另一个能力** —— `storage:local:read` 与
      `storage:local:write` 是同一件事上的两个权限，而“用户已经信任这个插件做存储”与
      “什么都没授”是两个不同的问题。
      ★ 而它**不承诺会被授予**：授不授是用户的决定，一句把拒绝写成形式的话
      会让插件作者以为它只是流程。
      ★ 而被拒能力**无法解析时仍然给出拒绝**（只是没有域提示）：
      把一个坏拒绝变成“没有拒绝”是门控绝不能产生的结果。
      没有 `synra-capability::error` 模块，三样东西各答一半：
      `CapabilityError`(`syntax.rs:40`) 是**解析**错误；
      `GateRefusal::NotGranted{capability}`(`gate.rs:16-23`) 只带被拒的那个能力；
      面向插件的是 `CallError::denied`(`host_calls.rs:81`) + `ErrorCode::CapabilityDenied`
      + `AdmissionRefusal`(`synra-core/src/admission.rs:264`)。
      没有一个带上**已授予集合**或**修复建议**；最接近的 `admission.rs:266` 只是散文式地列出**缺失**项。
- [x] host fn 都过闸门 —— **C，且 CI 检查曾经是空转的**。
      两个宏全仓无定义；真实机制更强：`build.rs` 每次从 `wit/synra-plugin.wit` 重新生成
      `capability_table.rs`，`host_calls.rs:374-380` 拒绝表外一切调用，
      `host_surface.rs` 有三个测试（免检白名单 ≤10 条且每条必须是事实读取 / 危险清单点名 / 两表恰好划分全集）。
      ★ `tools/guards` 里那条标着 "10 §3.1" 的守卫**曾经永远不可能失败**：
      它 grep 的 `ungated_host_fn!` 只出现在它自己内部，它白名单的
      `crates/synra-plugin-host/src/host_funcs/env.rs` 连目录都不存在。
      **本次已重写**为直接读生成表、要求每个 `None` 条目属于 `env-` / `platform-capabilities` /
      `plugin-id` 三族，并加了"解析出 0 条即报错"的自检；负控制（把 `network-fetch` 改成 `None`）确认它会红。
- [x] 路径解析 + canonicalize + 组件级前缀比较 —— **C/A**：真名 `PluginDataRoot`
      （`synra-store/src/sandbox.rs:99`，不是 `PluginSandbox`）；`resolve()`(`:198`) 先
      canonicalize 父目录(`:229`)、逃逸在 `:236` 被拒、判定用 `paths.rs` 的 `is_within`
      （按 `Path::components()` 逐段比较，`:522` 有 proptest）。真实逃逸测试（含 symlink）在 `:474`。
      ★ **"TOCTOU 安全（句柄式）"没有做到**：返回的是 `PathBuf` 而不是已打开的 `File`/`Dir` 句柄。
      模块自己的注释（`sandbox.rs:14-17`）就写明这是已知差距，`cap-std` 也**不是依赖**。
- [x] Windows 路径陷阱（保留名、ADS、设备命名空间）—— **已实现**（第 70 轮）：
      `synra_platform::guard`。3 个新拒绝理由（`reservedName`/`alternativeStream`/`deviceNamespace`）。
      ★ **而三条各自对应一个“内核与文件系统对同一个名字理解不同”**：
      * `NUL`/`CON`/`COM1`… 在 Windows 上**先于文件系统被解析** —— 以为写了 `<root>/NUL` 实际什么都没写；
        而且 **扩展名在设备名之后**被看，所以 `CON.txt` 也是控制台；尾部的点和空格也会被剥掉。
      * `notes.txt:secret` 是 NTFS 的**隐藏副流**，而在别的平台只是一个普通字符 ——两种读法不同。
        它也是绕过扩展名检查的方法：`report.exe:harmless.txt` 的扩展名是 `txt`。
      * `\\?\C:\data\..\..\Windows` **关掉了 Windows 本来会做的路径清理**，
        所以 `..` 是字面意义的一个目录名。两个看起来一样的路径**不指同一个地方**——
        所以本设计的“用 dunce 规范化”对安全检查是**错的**，它改成了**拒绝**。
      ★ **而短路径（`PROGRA~1`）和大小写不敏感本来就是 fail-closed**：
      `is_within` 用 `Path::components()` 逐段比较，所以 `C:\Data` 与 `c:\data` 被判为**不同**——
      两个方向都是拒绝，也就不会放行任何东西。这一条仍由 `is_within` 负责，不改。
      `dunce` / 保留名（`CON`/`NUL`/`AUX`）/ 8.3 短路径（`PROGRA~1`）/ `\\?\` verbatim 处理。
      只有三处 `cfg!(windows)` 测试字面量。
      ★ 而且有一个**方向相反的**性质：`is_within` 用 `Path::components()` 的 `PartialEq`，
      在 Windows 上**大小写敏感**、且把 `\\?\` 当不同 Prefix 组件，所以 `C:\Data` 与 `c:\data`
      会被判为不同。这是 fail-closed（安全），但与 §4.3 要求的"比较时大小写不敏感"正相反。
- [x] `NetworkGuard`：scheme 白名单 + host 规范化 + DNS 后 IP 校验 + 重定向逐步校鮌 + SSRF 拒绝表 ——
      **已实现**（已提交）：`synra-core/src/network.rs`。四步按 `10 §4.4` 的顺序：
      `check_url`（scheme + host）、`check_resolved`（IP 策略）、`next_hop`（逐步重校），
      `IpPolicy` 六个字段与 `Cidr` 都在，默认全 `false`。
      ★ **而第三步为什么是参数而不是调用**：这个 crate 不做 DNS，
      而一个会解析域名的守卫是一个**离线就测不了**的守卫。
      ★ **七个绕过都有对应的测试**，而两个最典型的做了阴性对照：
      * `https://api.example.com@evil.example/` —— 取 `@` **之后**的部分；取之前的会把
        授权给一台机器然后连另一台。对照失败。
      * **每一个**解析出的地址都要过，不是第一个：对照失败。
      * `169.254.169.254`（云元数据）、`::ffff:127.0.0.1`（换了家族的环回）、
        `api.example.com.evil.example`、`/33` 的 CIDR、超限重定向 —— 各有一条。
      * 证据：**55 suites / 2820 passed**（+22），clippy、7 guard（含兄弟 app 构建）均绿。
      ★ **但它还没有生产调用方**：`network-fetch` 本身仍然拒绝（仓库里没有
      HTTP 客户端）。所以这是**把守卫先建好**，而不是“拥有了护拦”。
>
> ### ★ 而现在可以写了 —— 下一轮要用的事实在这里，不要重新调查
>
> **1. 客户端依赖离线可用。** `ureq 3.4.2` 已在本机 registry 缓存里，带 `rustls` + `gzip`；
> 已用一个临时 crate 验证 `cargo build --offline` **能解析并编译**。TLS 栈（`rustls`/`ring`/`aws-lc-rs`/`webpki-roots`）也全在缓存里。
>
> **2. 放在哪里：新建一个宿主侧 crate，不是 `synra-net`。**
> * `synra-net` 的自述是「两台设备之间、socket 已经建立之后的事」，而且它**不做 IO**；HTTP 客户端两条都违背。
> * 放进 `synra-core` 会把 **TLS 拖进内核的依赖图**，而 `synra-core` 是要向 Android 交叉编译的（见 `matrix`）。
> * 所以：一个只有宿主依赖的 crate，内核的依赖图不变。
>
> **3. DNS 重定向防御是可实现的，不是口号。** `ureq` 有 `Resolver` trait（`unversioned/resolver.rs:29`），
> 经 `Agent::with_parts(config, connector, resolver)`（`agent.rs:130`）装入。于是可以做成：**解析 → 守卫校验每一个地址
> → 只把通过的地址交给 ureq**，而证书仍然是对**原域名**验的。
> 把 URL 直接换成 IP 是错的（证书校验会失败，或更糟的：被绕过），而守卫里已经写下了这一点。
>
> **4. 自动重定向必须关掉。** 设计要求**每一跳都重新校验**，而 `ureq` 的 `max_redirects` 是自己跟：
> 能限次数，不能每跳都问 `NetworkGuard::next_hop`。所以要 `max_redirects(0)` +
> `max_redirects_will_error(false)`，然后**自己跟**，每跳调一次 `next_hop`。
>
> **5. 响应体上限有现成的。** `BodyWithConfig::limit(u64)`（`body/mod.rs:502`）。
> 而 `request-json` 的**具体形状文档没定**（和 intent 的 payload 一样），所以它要像
> `synra_core::intent::remote` 那样，**把决定写在模块文档里**，而不是散在代码里。
>
> ★ **已实现（已提交）**：新 crate `crates/synra-fetch`。`Client::fetch` 把上面五条全部落了地：
> 解析 → `check_resolved` → `FixedResolver` 只交出通过的地址（证书仍验原域名）→
> `max_redirects(0)` + 自己的重定向循环（每跳过 `next_hop`）→ 响应体限额在**读的时候**生效。
> * ★ **测试真的开了 socket**：一个 `TcpListener` 写固定响应，因为这个模块的每一个判断都是关于
>   一个**即将发出的请求**，而把传输换成桩就只能测桩自己对守卫的理解。
> * ★ **两个阴性对照**：让客户端自己跟重定向（→ 5 个测试红）；
>   把响应体上限改回 `u64::MAX`（→ `a_body_larger_than_the_limit...` 红）。
> * ★ **而它捕到了两件事**：
>   1. `ureq` 的 `limit(n)` 在剩余为 0 时读一次就报错，而 `read_to_string` 总会多读一次看结尾
>      —— 所以**恰好等于上限的响应体会被拒**。传 `limit + 1` 才让“超过才拒”名副其实。
>   2. 我的第一版重定向测试转向了**同一台主机的另一个端口**，而它**通过了守卫**
>      —— 因为能力授权的是 **host**（它的拼写就是 `network:fetch:<host>`），端口不在授权里。
>      是测试错了而不是守卫错了；现在改成转向 `localhost`，并另加一条**断言端口确实不在授权里**。
>
> ★ **守卫现在有生产调用方了**：守护进程的 `KernelHost::fetch` 已接上 `Client`。
> * 授权的 host 从**已记录的授权里读**：`network:fetch:<host>`，而 `network:fetch:api.github.com:POST`
>   里的方法不归守卫管（守卫管的是去哪里）。
> * ★ **而 `network:fetch:*` 授权不等于“任何 host”**：`*` 是**门**对整个族的拼写，
>   把它当主机名就是为一台叫 `*` 的机器建守卫，把它当“任意”就是 SSRF 守卫本身要防的能力。
>   所以它一个 host 也到不了，拒绝码是 `capabilityDenied`（去要一个具体的）。
> * ★ **客户端缓存带着它建立时的 host 列表**，列表一变就重建 ——
>   否则**用户撤回了一个 host，插件还能通过缓存的客户端继续访问它**。有一条测试就是这个：
>   授予 → Ok，撤回 → **必须**拒绝（如果缓存只按 plugin 作 key，这条会红）。
> * ★ **而一条旧测试当场失败了**，正如它该失败：`the_groups_this_build_does_not_implement_refuse_with_one_code`
>   把 `fetch` 列在“本构建未实现”里。现在它不在那个名单里了，并且像 storage 一样
>   **被正面断言**（拒绝码是 capability 而不是 notImplemented），所以“从名单里删掉”
>   不会被误认为“把功能删掉了”。
>
> **仍然没做**：桌面 shell 的 `DesktopHost::fetch`；`network-websocket`；
> 以及一个真实的 HTTPS 请求——测试跑在本机 `TcpListener` 上，因为本轮不能联网。
      整个 workspace **没有任何 HTTP 客户端**（搜 `reqwest`/`hyper::`/`ureq`/`isahc` 只有 `Host::fetch`
      的 trait 声明与测试桩）。`ErrorCode::IpNotAllowed`/`RedirectBlocked` 有码无生产者。
> ★ **而它当时没有生产调用方 —— 因为守护进程把整组 `platform:*` 一律拒绝了。**
> `KernelHost::platform` 原来对每一个平台动作都答 `notImplemented`，
> 而 `synra-platform` 实际上实现了整条受守护的路径（`desktop.rs` / `host.rs` 都调 `check_url`/`check_path`）。
> ★ **而 `HeadlessPlatform` 的注释早就说明了为什么那不对**：
> “一个传了 `javascript:` 的插件应该被告知**它的地址被拒了**，
> 而不是“这个孿主没有浏览器”。前者是作者能修的错误，后者是关于安装的事实。”
> 现在 `KernelHost::platform` 委派给 `HeadlessPlatform`：先跑守卫，再拒绝，
> 并把 `PlatformError` 按原因映到 `badArgument`（插件的错）或 `notImplemented`（安装的事实）。
> ★ 而且 `Policies` 的构造器与访问器也没有调用方 —— 政策永远是默认值（严格的那个）。
> 这是安全的，但意味着**没有任何地方能改它**；记在这里而不是当成已完成。
      `guard.rs:21` 的 `check_url` 是 `platform:open-url` 的 scheme/host 存在性检查，不做解析。
      ★ 同时也说明：**SSRF 这一类漏洞目前无法被触发，因为还没有东西会去发请求** —— 这是"没有功能"而不是"有漏洞"。
- [x] 各 RateKind 的令牌桶 —— **C**：`RateLimiter` 确实存在
      （`synra-platform/src/guard.rs:200`）但它是**每个 `PlatformAction` 的冷却/小时窗**限流器，与 `RateKind` 无关。
      文档要的令牌桶在 `synra-plugin-host/src/quota/`：`RateKind`(`limits.rs:13`)、
      `TokenBucket::try_consume`(`ledger.rs:79`)、`Budget::consume`(`budget.rs:237`)，
      `quota/proptests.rs:18` 有 proptest。性质完全成立，只是名字和 crate 换了 —— **是两套独立机制**。
       > ★ **而 `Quota` 的十二个字段里，只有“速率”那五个真的被强制。** 逐条核对：
       > 被强制的：`host_calls_per_sec` / `host_calls_burst`（以及其余四个 kind，
       > 经 `Quota::rate_for`(`limits.rs:130`) → `Budget::consume`(`budget.rs:237`)，
       > `host_calls.rs:374-390` 的 `rate_kind()` 负责分类）。
       > **没被强制的**：`max_memory_bytes`、`max_instances`、`max_storage_bytes`、`max_open_transfers`、
       > `call_fuel`、`call_timeout_ms`、`network_concurrent`。
       > * ★ 根因是 `CallBudget::from_quota`(`engine/types.rs:233`) **在生产里从来没有调用方**——
       >   只有 `scripted.rs:691` 的 `enforcement` 和一个测试用。它的注释写着
       >   "Every field is taken from the quota rather than chosen here, so the number a user sees in a settings
       >   screen is the number the sandbox enforces"，**而那句话当时只有三分之一是真的**。
       > * 已修的是 `call_fuel` / `call_timeout_ms` / `max_memory_bytes`：`share.execute` 现在用
       >   `PluginSet::quota()` → `CallBudget::from_quota`，而不再是一个固定的 5 秒超时 + `fuel: None`。
       >   ★ 那个 `fuel: None` 值得单独说：**它意味着执行一个动作时完全没有指令上限**，只有墙钟。
       > * 仍然没强制的：`max_instances`（实例上限走的是 `PluginSpec.limit`，**不是** quota —— 两个来源）、
       >   `max_storage_bytes` 与 `max_open_transfers`（这两个守的功能本身还没实现，所以“没强制”是对的）、
       >   `network_concurrent`（无并发请求这回事）。
       > * ★ **为什么之前看不出来**：速率**是**被强制的，而“一个能做事的 quota”看起来就像“一个能用的 quota”。
- [x] `UiSessionRegistry`：签发/校验/撤销 —— **A，但"注入"那半是 B**。
      名字完全一致：`synra-plugin-ui/src/session/registry.rs:17`，`issue_with`(`:46`)、
      `validate`(`:140`，要求 nonce + plugin_id + version 三者一致)、`revoke`(`:113`)、
      过期清扫(`:232`)；proptest `session/proptests.rs:66`
      `a_revoked_nonce_is_indistinguishable_from_an_unknown_one`。
      ★ **Tauri initialization script 注入：2026-10-18 已做了一半**——注册表在 core、`AppState.ui_sessions` 在壳、
      `initialization_script(label, token)` 生成脚本并由 `plugin_ui_open` 使用。**仍缺**协议 handler 与 CSP。
      （`synra-observe/src/audit.rs:179`，不是 `AuditLog`）。`MEMORY_CAPACITY = 10_000` 与 §7 一致；
      无 payload 是**构造性**的（`AuditEntry` 的字段没有一个能装内容，`audit.rs:437` 有断言）；
      UI 查询接口齐全（`for_plugin`/`refusals_for_plugin`/`PluginActivity::of`/`clear_plugin`/`clear`）。
      ★ 但**生产代码从不写它**：`Kernel::record_action` 只在测试里被调用（见 [18](../18-observability.md)）。
- [x] 装载期 `verify_effective_caps`：manifest ∩ grant ∩ wasm imports —— **A/C**：
      同名函数在 `set/interface.rs:67`，四步齐全：grant ⊆ manifest(`:74-80`)、
      每个 import interface 被覆盖(`:83-87`)、effective 取交集(`:92-97`)、
      已授权但未 import → 警告而不拒绝(`:100-104`)；测试 `set/tests.rs:206,219,231,246,260`。
      ★ 签名漂移：文档写 `(&PluginManifest, &Grant, &ImportSet)`，真实是
      `(&CapabilitySet, &CapabilitySet, &ImportSet)` —— 两者都已解析过，
      "manifest 声明必须语法合法"是在构造 `CapabilitySet` 时强制的，不在这个函数里。
- [ ] 恶意插件测试套件（[21](../21-testing-strategy.md) §5）—— **未做**：没有
      `PluginHostFixture::install("malicious-loop")` 之类的套件，没有 `tests/malicious*`，
      **也没有任何测试断言 §5.1 那句"内核仍可用 + 其它插件仍可用"**。
      29 条里约三分之一有孤立的对抗测试（归档穿越/symlink/体积、签名吊销、令牌伪造、装载期 import 覆盖），
      但 SSRF 那 11-15 行**无法测**（没有 fetch 实现可攻击），§5.2 的 UI 插件 10 行整组没测。
