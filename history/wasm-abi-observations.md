# Wasm ABI 修复前的调查快照

归档日期：2026-10-26；以下原文保留旧调查中的推断和相互矛盾的状态，不定义当前实现。
有关返回值槽位、Host 类型和启动 fuel 的更正见 [本次验证记录](./2026-10-26-wasm-abi.md)。

## 原 12 实现清单

## 10. 实现清单

> ★ 这一节在实现之前写就，长期整节未勾选。以下每一条都重新核对过源码；核对不到的保持未勾选，
> 并在原处写明"真实的机制是什么"或"缺的是哪一半"。名字对不上的按真实名字记，另加 `C`。

- [x] `WasmBackend` trait + `BackendKind` —— **C**：不在 `backend/mod.rs`，真实位置是
      `engine/types.rs:15`（`BackendKind`）与 `engine/types.rs:44`（`WasmBackend` trait）。
- [x] Engine 配置（§3 的安全基线）+ 组件模型链接 —— **A**：`engine/wasmtime_backend.rs:54`
      `MAX_WASM_STACK = 512 KiB`、`:60` `MEMORY_RESERVATION = 64 MiB`、`:63` 增长上限 `2 MiB`、
      `:186` `impl ResourceLimiter`、`:144-149` linker 定义。
- [ ] `backend/wasmi.rs`：轻量后端 —— **未做**，全仓无 `wasmi` 依赖。
      `tools/matrix` 自己也写着"no `wasmi` dependency exists"。
- [ ] `WasiGrant`：默认拒绝，只开 clocks/random —— **未做，且前提不成立**：
      `wit/synra-plugin.wit` 里**根本没有 WASI import**，所以没有可以收窄的时钟/随机能力面。
- [x] `WasmExecutor`：专用线程池 + 并发上限 —— **A/C**：不是 `Semaphore`，是
      `engine/executor.rs:219` 的 `ThreadPool`（`sync_channel` 有界队列）+ `:44` `MAX_THREADS = 64`，
      满了返回 `:198` `Saturated` 而不是增长。
      ★ **但它没有被用在真实调用路径上**：`synra-core/src/dispatch/plugins.rs` 的
      `instance.call()` 直接在调用线程上跑。整节"专用线程池"的防护在生产里不存在。
- [x] ★ `Enforcement::first_gap` —— **已接线**（第 81 轮）：
      `engine::check_enforceable(&Backend, &Quota)`，守护进程的 `build_set` 与 Tauri shell 各调一次。
      ★ **而上面那段分析是对的**：我确实先把它加在 `PluginSet::call` 里，
      结果一整套 dispatch 测试变红 —— 正如这里预言的那样。所以问题不是“补一行 if”，
      而是**这个问题该在哪里问**。
      ★ **而这个决定是**：`Enforcement` 继续表示“实现上诚实”（替身确实没有引擎），
      而安全论点在**内核选定引擎的那一刻**满足：一个无法约束它所运行东西的宿主
      **不得启动**，而且要说出缺的是哪一项。理由是：少一项限制是**安装**的事实，
      不是某一次调用的事实，而启动是唯一一个人能对它做事的时刻。
      ★ **而它今天是安静的，而这正是重点**：wasmtime 后端全都能强制、默认 quota 全都要求，
      所以它不会响。上面列的 `backend/wasmi.rs` 正是会让它响的那一个。
- [x] `EpochTicker`：默认 10ms —— **A/C**：`engine/executor.rs:38` `EPOCH_PERIOD = 10ms`、
      `:101` `EpochTicker`、`:161` `Drop`、`:361` `enforces_timeouts()`。
      ★ 但它不是"受监督任务"：`apps/synra-daemon` 里搜不到任何 ticker/epoch 监督代码。
- [x] `CallBudget` + `PluginInstance::call`：fuel + epoch + memory 三层 ——
      **★ 已修：三层现在都真的会触发**。原先 `Engine::increment_epoch()` 在全仓 **0 命中** ——
      本 crate 自己的 `Epoch` 原子量与 wasmtime engine 的 epoch 计数器**没有任何连接**：
      store 被告知"你的 deadline 是第 25 tick"，ticker 也确实把 `Epoch` 从 0 推到 25，
      而 wasmtime 自己的计数器永远是 0。后果是一个**阻塞在 host call 里**的 guest
      （不消耗指令，所以 fuel 不减少）**没有任何东西能中断它**，而"墙钟超时是唯一能停下
      阻塞在 host 调用里的 guest 的那一层"正是它的存在理由。
      修法：`Epoch` 增加 observers（`advance()` 通知它们），`WasmtimeBackend::with_epoch`
      把 `engine.increment_epoch()` 注册进去 —— 两个计数器从此是同一个动作，调用方不需要记得两边都推。
      另外 `WasmtimeBackend::new` 原先等于 `with_epoch(kind, Epoch::new())`，即**一个没有 ticker 的私有 epoch**，
      正是隔壁构造函数注释里称为"impossible"的那种失败；现在它自己起一个 ticker 并持有。
      测试：`an_engine_sharing_an_epoch_is_joined_to_it`、`an_engine_with_its_own_epoch_advances_it`、
      `an_observer_moves_when_the_epoch_does`、
      `every_observer_is_called_and_none_needs_the_lock_to_be_free`（后者守的是 observer 里读 epoch 不死锁）。
      两处负控制都做了：去掉 observe 注册 → 测试红；把 `new()` 改回无 ticker → 测试红。
      ★ 仍未做的：`EpochTicker` 在 `apps/synra-daemon` 里**没有监督者** —— 它靠 `Drop` 停止，
      而没有任何地方检查它还活着。
- [x] `catch_unwind` 包住 Wasm 调用 —— **已补上**。原先全仓 `catch_unwind` 0 命中，
      而 `Cargo.toml` 的 `[profile.release] panic = "unwind"` 注释给的理由恰恰是
      "插件 panic 必须能被 catch_unwind 捕获" —— 为一个不存在的机制做的配置。
      现在边界在 `synra-core/src/dispatch/plugins.rs` 的 `run_on`：
      捕获、**丢弃该实例**（panic 穿过的 store 状态无法信任）、返回 `CallOutcome::Trap`。
      测试与负控制见 `dispatch/tests.rs` 的
      `a_plugin_that_panics_the_host_does_not_stop_the_kernel`（去掉 catch 即失败）。
      ★ 未做的一半：**panic hook 接入 tracing** —— `synra-observe` 里没有 tracing 初始化（见 [18](../18-observability.md)）。
- [ ] `gate_host_fn!` / `ungated_host_fn!` 宏 —— **未做，且这两个宏从未存在**。
      真实机制更强：`build.rs` 从 `wit/synra-plugin.wit` 的 `@ungated` / `@capability("…")` 注释
      **每次构建重新生成** `src/generated/capability_table.rs`。
      见 [10](../10-capability-and-sandbox.md) 与 [11](../11-design-principles.md) 的同名条目。
- [x] host functions（env/log/event/device/storage/network/transfer/platform/action）——
      **A/C，且端到端不可达**。`src/generated/capability_table.rs:19-55` 有 35 个函数、
      `engine/linker.rs:153-341` 覆盖九种签名。
      ★ 文档列的 10 组里 **`fs` 不存在**（合同没有文件系统接口）。
      ★ 更重要的：`wasmtime_backend.rs` 的 `instantiate` 恒为 `session: None`，而 `linker::run`
      在 session 缺失时对**任何** guest 调用都返回拒绝 —— 所以 35 个 host function 在端到端意义上
      全都不可达，唯一的 `impl Host` 是测试桩（`host_calls_tests.rs:49`）。
      ★ **这意味着更要紧的一件事：能力闸门在生产里从未运行过。**
      `instantiate` 之后没人装 session，所以 `run` 在第一道 `let Some(session) = … else` 就返回了 ——
      闸门（capability 检查）根本没走到，审计也一条没记。
      闸门是整套能力模型所依赖的机制，而它是**静止的**。
      ★ **本次完成了前置的两块**（都不改变运行时行为，但缺了它们无法继续）：
      1. `PluginInstance::set_session(LinkerSession)`，带默认空实现 ——
         有 host function 要授权的后端重写它；没有的后端（`ScriptedBackend` 从表里答，够不着 guest）
         不必为此写一个空函数体。
         不变量仍然是安全方向：**忘了调用它得到的是"插件调不了 host"，不是"插件能不经检查地调"**。
      2. `LinkerSession` 从 `linker.rs` **搬到了 `engine/session.rs`，且不再受 `wasm-runtime` 门控**。
         原先它连带被 gated，于是 `set_session` 在无引擎的构建里**没法写出参数类型**。
         一个描述"这次调用被允许做什么"的类型，不该和恰好执行它的引擎一起被编译掉。
         测试 `session_tests.rs` 跑在**默认 feature** 下（281 条里那 4 条），
         其中 `the_session_type_exists_without_an_engine` 就是钉这件事的。
      ★ **仍未做的，以及为什么它不是"补一行"**：
      ★ **上面列的三条本轮全部做完了，所以闸门现在真的会跑。** 做法如下：
      * **`Arc<dyn Host>` 进了 `PluginSet`**，且 `PluginSet::new(backend, host)` 是**必填**的：
        没有诚实的默认值——无 session 的实例会拒绝每一次调用，
        而那个失败会读成"插件有问题"而不是"孿主没说效果该去哪"。
      * **同时抽出了 `Authority`**（grants + `Arc<Budget>` + `Arc<Mutex<Vec<AuditEntry>>>`），
        `Loaded` 持有它。这样抽的理由有两个：
        budget 和审计属于**插件**而不是实例（每实例一份会把限额乘以实例上限），
        而且"这个插件能做什么"是一句**不需要 `.wasm` 就能测**的话。
        测试 `a_session_carries_the_plugins_own_authority` 和
        `two_instances_of_one_plugin_share_their_rate_and_their_record` 就是这两句话（后者用指针相等，
        因为"同一份限额"正好是副本不是的东西）。
      * **审计现在有了写入者**：`LinkerSession.audit` 是共享句柄，
        `PluginSet::call` 在每次调用后把它排入 `synra_observe::audit::AuditRing`，
        并暴露 `audit()` / `audit_for()` / `refusals_for()`。
        环本来就有界、有测试、有查询——只是没有写入者；
        而之前唯一的写入路径是 `Kernel::record_action`，那个类型没有任何孿主构造（见 [07](../07-core-kernel.md) 开头）。
      * **生产的 `impl Host` 在守护进程里**：`apps/synra-daemon/src/runtime/plugin_host.rs`。
        它服务 `env-*`（`sdk-version` / `host-kind` / `locale` / `dark-mode` / `can`）、`log`、`devices`，
        其余一律以 **`NOT_IMPLEMENTED`** 拒绝。
        拒绝不是占位：插件按 code 分支，而编造一个看起来合理的答案才是唯一更糟的选择。
        `can` 读的是 **`plugin_grants` 里用户真正授予的集合**，所以它与门的判定一致；
        `devices` 读已配对且 `is_usable` 的行——不把离线设备当成诺言。
      * ★ **一处契约缺陷记在这里**：`platform_capabilities` / `locale` / `dark_mode`
        返回**值**而不是 `Result`，所以孿主**无法拒绝**。
        前者答空列表（这个构建没有平台集成，空是真的），
        后两者读配置。一个**不可失败的问题无法区分"没答案"与"没有"**，
        而插件两者都分不出来——这是契约本身的问题，不是这份实现的。
      * 仍然卡住的只有**端到端**：两条真跑 wasm 的测试仍是 `#[ignore]`，
        所以"闸门跑了"是靠实例上的 session 证明的，不是靠一个 guest 跑完证明的。
      * `PluginSet` 需要持有一个 `Arc<dyn Host>`，`run_on` 需要为每次调用构造 session
        （`LinkerSession::new` 要 `Arc<quota::Budget>`，而 `Loaded` 只带 `CapabilitySet`，**没有 budget**）；
      * 需要一个**生产用的 `impl Host`** —— 27 个方法。`synra-core` 的依赖够（platform/observe/store/net 都在），
        但其中 `secret_*` 只有 `MemoryBackend`、`fetch` **全仓没有任何 HTTP 客户端**，
        storage 也没有插件的表：**这三组现在无法诚实实现**，只能拒绝；
      * 审计要真的有用，session 的条目必须**活得比实例久**（进内核的 ring），
        而那不是这里的 `Mutex<Vec<AuditEntry>>` 能给的 —— 共享句柄该长什么样，
        要等有调用方的那一步再定，而不是在这里猜（所以本次**没有**加 `with_audit` 这类没有调用方的构造器）。
      * 端到端的验证还被夹具卡住：两条真跑 wasm 的测试都是 `#[ignore]`，
        原因是手写 `.wat` 与 canonical ABI 对不上，出路是 `bindgen!` 生成的夹具 ——
        而那需要 `wasm32-wasip2` 与 `wasm-tools`（环境依赖，见本节末尾）。
- [x] `CompileCache`：键含 wasm hash + backend + wasmtime 版本 —— **A/C**：
      `engine/cache.rs:65-91` 的 `CacheKey{artifact_digest, backend, engine_version, target}`。
      ★ `CompileCache` 只被自己的测试调用；没有任何后端 override `precompile`/`load_precompiled`
      （`types.rs:629/639` 的默认实现恒返回 `BackendUnavailable`），所以 `.cwasm` 永远不会被写出来。
- [~] 编译任务超时与输入大小限制 —— ★ **大小限制已有**（第 113 轮），**超时仍未做**：
      `MAX_COMPONENT_BYTES = 16 MiB`（`engine/wasmtime_backend.rs`，由 `engine` 再导出），在 `compile` 里
      **在引擎看到任何字节之前**检查。它与 CLI 的 `MAX_WASM_BYTES` **同值**，而不是重新选一个数字：
      第 116 轮把它变成了**一处**：常量从 `wasmtime_backend`（在 `wasm-runtime` feature 后）
      移到 `engine/mod.rs`，于是 `synra-plugin-cli` 能直接引用它 —— 它本就不需要开那个 feature。
      ★ 而两个同值字面量的代价写在那里了：`verify` 放行、宿主拒绝，
      于是一个检查告诉作者“你的插件没问题”而内核不同意。
      ★ 而它乍看像重复：CLI 拦的是**命令行传入**，registry 拦的是**远程包**，
      而宿主拦的是**任何到达它的路** —— manifest 指向的文件、被改过的缓存。Wasmtime 的解析器不是资源限制器。
      ★ 测试断言的是**拒绝的措辞而非拒绝本身**：16 MiB 的零不是组件，引擎也会拒绝它，
      所以只有报文同时写出两个数字（16777217 / 16777216）才能证明这是**本层**拦下的。
      ★ **而超时那一半第 116 轮查过了 wasmtime 的源码，结论是它在本进程内做不到**：
      wasmtime 38 没有任何编译期限或取消接口（只有 `precompile_*`，它只是提前编译，不带 deadline）；
      `config.rs` 里的 epoch 是**执行期**的，编译在宿主线程上跑，不经过它。而 Rust 线程**不能被取消**，
      所以“超时就丢掉那个线程”是一个漏线程的假超时。两条真路：子进程隔离（重，且不在本轮范围），
      或者接受**大小上限就是本进程内唯一的守卫**。后者是现状，而且现在写明了为什么它是现状。
- [x] `InstancePool` + 空闲回收 + 池指标 —— **A/C**：真名 `pool::InstancePool`
      （不是 `PluginPool`），`pool/instance.rs:278` `evict_idle(now_ms)`、`:317` `PoolReport`。
      ★ `evict_idle` 只被测试调用，没有任何扫描器；`synra-observe` 里也**没有池指标**。
      ★ **2026-10-19：池指标有了，扫描器仍然没有。** `plugin_pool_hit_total` / `plugin_pool_miss_total` 在
      `PluginSet::call` 里按“调用前 `live` 是否已有这个 handle 的实例”记一次；`evict_idle` 依旧只有测试在调 ——
      池的**回收**仍然没有生产触发点。
- [x] 健康状态机 + 降级/停用策略 —— **A/C**：真名 `quota::Health`
      （不是 `PluginHealth`/`HealthState`），`quota/budget.rs:25` 四档 + `:16/:19` 阈值 +
      `:152-159` 降级配额，`quota/tests.rs:290-364` 有完整阶梯测试。
            ★ **而第 114 轮把它接上了**：`Budget::record_misbehaviour(now_ms)` 与 `record_refusal` 共用
      同一个台阶（`note_bad_call`），而 `PluginSet::call` 在客体 misbehaviour 后调它。
      ★ **而回归测试的形状值得一说**：它与拒绝那条阶梯用**同一组配件**（`PERSISTENT_REFUSALS` 次/窗口、
      `in_window`），所以两条路径看得出是**一个机制**；另一条测试取反对照：单次 misbehaviour
      **不**改变 health —— 阶梯只对**持续**坏的窗口起作用。
      ★ **而那个缺口第 115 轮补上了**：`PluginSet::health_of(id)`，而 `plugins_value()`
      现在同时报 **两个**健康：`health`（数据库那行，安装时记下、跨重启存在）与
      `liveHealth`（阶梯**现在**的一档：`good`/`overreaching`/`degraded`/`faulted`）。
      ★ **两个字段而不是一个，因为它们是两件事**：一个跨重启存在的记录，
      和一个只在本进程里成立的事实。它们不一致的时候正是最值得看的时候。
      ★ 而本轮又犯了一次第 111 轮的毛病**：我只读了 `state.rs` 前两条 `plugins` 命中，
      得出“Runtime 没有这个字段”，于是又加了一个 —— **而它本来就在第 90 行**。
      这次是编译器在提交前拦下的，而上一次是我自己 grep 到的 ——
      **两次都是“看了一部分就下结论”**。
      ★ **2026-10-19：窗口侧的健康也有生产者了。** 契约里的 `PluginHealthDto`（档位、句子、准入/拒绝、生效配额）此前
      没有任何代码构造，壳的每张卡都是 `health: None`。现在 `PluginSet::report_of` 返回 `QuotaReport`，壳的 `snapshot()`
      映射成 DTO，卡片在档位不是 `good` 时显示警告（`faulted` 用 error），正文是配额自己的 `explain()` 句子。
      与 daemon 的 `liveHealth` 是同一档、同一个 budget。
- [ ] 测试：死循环（fuel）、阻塞 host（epoch）、内存炸弹、trap 恢复、并发隔离 ——
      ★ **五项里 0 项**由一个真跑 wasm 的测试覆盖：
      fuel 的唯一真用例 `wasmtime_tests.rs:306` 被 `#[ignore]`；
      epoch **完全无测试**（`:334` 只断言"deadline 被写进去了"）；
      内存炸弹无 guest 测试（`:354` 只断言 ceiling 已安装）；
      trap 恢复只有 `ScriptedBackend` 桩；并发隔离测的是线程池，不跑 wasm。
      本 crate 里唯一执行真 wasm 的两个测试都被 `#[ignore]`（`wasmtime_tests.rs:270, 306`）。
      ★ **2026-10-19 更正 fuel 那一行**：`last_fuel_consumed()` 现在由 `PluginInstance` 暴露，wasm 后端在调用前后各读
      一次 `store.get_fuel()`；断言它的测试是 `a_call_reaches_the_guest_and_its_answer_comes_back`（**没有** `#[ignore]`），
      外加一个被 fuel 停下的 guest 断言恰好花了预算那么多 —— 所以“fuel 只有被 ignore 的用例”这句不再成立。
      仍然被 ignore 的两条（`wasmtime_tests.rs:270, 306`）的注解值得照抄，它比"没做"更有用：
      两条的注解都是
      "the hand-written fixture does not match the canonical ABI for a `string` parameter and result;
      the lookup itself now works, and a fixture built by `bindgen!` from the WIT file is what would close this"。
      也就是说**缺的是夹具而不是被测代码**：手写的 `.wat` 与 canonical ABI 对不上，
      而 `bindgen!` 从 `wit/synra-plugin.wit` 生成的夹具才是出路。
      ★ 这条路径**依赖环境**：`synra-plugin-sdk/tests/guest/world_fixture.rs` 自己写明
      `wasm32-wasip2` 目标与 `wasm-tools` 都没有安装。
      所以这一条要真正勾上，先要有那个目标 —— 属于"需要环境"的一类，本次不碰。

## 原 21 对抗测试调查

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


## 原 21 Host 调用夹具调查

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

## 原 12 fuel 与 epoch 调查

### 4.1 为什么 fuel 不够，还需要 epoch

| 场景                                        | fuel 能挡住？ | epoch 能挡住？ |
| 燃料 / `fuel` | 指令数 | ★ **而“不限”必须真的给燃料**：每个 store 都开着 `consume_fuel` 且从 **0** 开始，
  所以“预算里没提金额”如果什么都不做，客体会立刻报 `FuelExhausted { fuel: 0 }`
  —— **一个没有指令上限的预算被判为没指令了**。现在 `None` 给 `u64::MAX`。 |
| 实例复用 | 池 | ★ **而一次超预算不能把插件变成永久坏掉的**：一个 **trap 过的 component instance 不能再次进入**，
| 内存上限 | 字节 | ★ **而这一层的 `None` 不是“无限”，而是“最后一道底”**：预算不提上限时，
  实例仍被初始化时的 16 MiB 拦着。这是故意的 —— `execute_budget()` 就不提内存上限，
  所以一个动作只被时间约束，而这 16 MiB 是阻止恶意 `execute` 吃掉整台机器的最后手段。
  ★ **而它被钉住了**，因为这个不一致看得见：另两层的 `None` 都是“无限”，
  “顺手改一致”就会**拆掉沙箱的最后一道底**。 |
| 内存结果里的数字 | 字节 | ★ **而它报的必须是真正拦住客体的那个数**：`MemoryExceeded { limit }` 原本报
  `budget.max_memory_growth.unwrap_or(0)`，所以一个撞上 16 MiB 墙的客体会听到“你的上限是 0”。
  现在报 store 里真正生效的值。**一个描述错东西的数字比没有数字更糟**，因为作者会去改它。 |
  而 `PluginSet` 原来只在 **panic** 时丢弃实例。于是一次燃料耗尽会让之后每一次调用都报
  `Trap { "cannot enter component instance" }` —— 用户的唯一办法是重启。
  现在任何 misbehaviour 都会丢弃实例，与 panic 路径同一条规则；代价是下一次调用重建一次实例，而池就是干这个的。 |
| ------------------------------------------- | ------------- | -------------- |
| 插件里的死循环（纯计算）                      | ✅            | ✅             |
| 插件调用 host function，host 阻塞（如 IO）     | ❌（guest 没消耗指令） | ✅      |
| 插件分配巨大内存                              | ❌            | ❌（需 memory limiter） |
| 插件在 host function 里递归调用回 guest         | ⚠️ 部分        | ✅             |

**所以三者都要**：fuel + epoch + memory limiter。
