# 12 · Wasm 插件宿主

> 本文定义 headless 插件的执行环境：wasmtime 配置、组件模型、资源限制、中断机制、实例池与故障隔离。
> 对应 [06 §3.5](./06-workspace-layout.md) 的 `synra-plugin-host` crate。

---

## 1. 为什么是 Wasm + 组件模型

| 方案                        | 隔离  | 类型安全 | 多语言 | 启动 | 生态       | 结论        |
| --------------------------- | ----- | -------- | ------ | ---- | ---------- | ----------- |
| **Wasm 组件模型（WIT）**     | 强    | ✅ 强     | ✅     | 低   | 成长中      | **采用**    |
| Wasm core module + WASI 0.2 | 强    | ❌ 手写 ABI | ✅   | 低   | 成熟        | 备选        |
| 独立进程 + JSON-RPC          | 强    | ❌        | ✅     | 高   | 成熟        | 太重（插件很小） |
| Native dylib（`dlopen`）     | 无    | ❌        | ✅     | 低   | 成熟        | **不可接受**（无隔离） |
| eBPF-like 沙箱              | 强    | ❌        | ❌     | —    | —          | 平台不支持   |

**选组件模型的理由**：
1. WIT 提供**接口级别的类型安全**，插件与宿主的 ABI 由工具生成，不手写；
2. 组件模型支持**接口演进**（`@1.0.0` / `@1.1.0`），这对插件生态是刚需；
3. `wit-bindgen` 支持 Rust / C / Go / Python / JS(componentize) 等多种 guest 语言；
4. wasmtime 对组件模型支持成熟（[wasmtime 48.x](https://github.com/bytecodealliance/wasmtime/releases/tag/v48.0.0)）。

**代价**：
- 组件模型的工具链比 core module 复杂；
- 二进制体积略大（组件封装开销）；
- 某些 WASI preview1 库还没有组件化版本（可用 adapter 过渡）。

> 这个取舍记录在 [21 ADR-003](./20-adr.md)。

---

## 2. 后端抽象（支持 wasmtime / wasmi 切换）

```rust
// crates/synra-plugin-host/src/backend/mod.rs

/// 编译产物（不同后端类型不同）
pub enum CompiledPlugin {
    Wasmtime(wasmtime::component::Component),
    Wasmi(wasmi::Module),
}

pub trait WasmBackend: Send + Sync + 'static {
    fn kind(&self) -> BackendKind;

    /// 编译（可能很慢 → 异步 + 可缓存）
    fn compile(&self, wasm: &[u8]) -> Result<CompiledPlugin, BackendError>;

    /// 预编译（AOT）：生成平台相关字节码，供下次直接加载
    fn precompile(&self, wasm: &[u8]) -> Result<Vec<u8>, BackendError>;
    fn load_precompiled(&self, bytes: &[u8]) -> Result<CompiledPlugin, BackendError>;

    /// 实例化：注入 host functions 与配额
    fn instantiate(
        &self,
        compiled: &CompiledPlugin,
        ctx: PluginCtx,
        limits: &ResourceLimits,
    ) -> Result<Box<dyn PluginInstance>, BackendError>;
}

#[derive(Debug, Clone, Copy, PartialEq, Eq)]
pub enum BackendKind {
    /// 完整 wasmtime + Cranelift JIT（桌面：最快）
    WasmtimeCranelift,
    /// wasmtime + Winch（更快的编译，慢一点的执行）
    WasmtimeWinch,
    /// wasmtime + pulley 解释器（无 JIT，移动端 / iOS）
    WasmtimePulley,
    /// wasmi 纯解释器（体积最小）
    Wasmi,
}

impl BackendKind {
    /// 按目标平台选择默认后端
    pub fn default_for(target: PlatformKind) -> Self {
        match target {
            PlatformKind::Desktop => Self::WasmtimeCranelift,
            // iOS 禁止 JIT → 必须解释器或 AOT
            PlatformKind::Ios => Self::WasmtimePulley,
            // Android 允许 JIT，但包体积敏感 → Winch 是折中
            PlatformKind::Android => Self::WasmtimeWinch,
            PlatformKind::Cli => Self::WasmtimeCranelift,
        }
    }
}
```

**关键点**：
- **编译与实例化分离**：编译一次，多次实例化（实例池的基础）；
- **AOT 预编译**：桌面端可以在**安装时**预编译（用户等一次），之后启动零编译成本；移动端**必须**AOT 或解释器；
- 后端可切换让"体积敏感"场景能选 `wasmi`（~1 MB vs wasmtime 的 5–15 MB）。

---

## 3. Engine 配置（安全基线）

```rust
// crates/synra-plugin-host/src/engine.rs

pub fn build_engine(kind: BackendKind, cfg: &EngineConfig) -> Result<Engine> {
    let mut config = wasmtime::Config::new();

    // ── 组件模型 ──
    config.wasm_component_model(true);

    // ── 中断机制（两者都要）──
    config.consume_fuel(true);                 // ★ 确定性指令预算
    config.epoch_interruption(true);           // ★ 墙钟抢占（兜底 host 阻塞）

    // ── 资源上限 ──
    config.max_wasm_stack(512 * 1024);         // Wasm 栈上限（不是宿主栈）
    config.memory_reservation(64 * 1024 * 1024);
    config.memory_reservation_for_growth(2 * 1024 * 1024);
    config.memory_guaranteed_dense_image_size(0);

    // ── 关闭不需要的特性（缩小攻击面 + 减小体积）──
    config.wasm_threads(false);                // 不允许线程（隔离更简单）
    config.wasm_reference_types(true);         // 组件模型需要
    config.wasm_simd(false);                   // 除非插件需要
    config.wasm_relaxed_simd(false);
    config.wasm_bulk_memory(true);             // 组件模型需要
    config.wasm_multi_memory(false);
    config.wasm_tail_call(false);
    config.wasm_gc(false);                     // 打开会显著增大体积
    config.wasm_function_references(true);
    config.wasm_custom_page_sizes(false);

    // ── 编译策略 ──
    config.cranelift_opt_level(match kind {
        BackendKind::WasmtimeCranelift => OptLevel::Speed,
        BackendKind::WasmtimeWinch => OptLevel::None,
        _ => OptLevel::None,
    });

    // ── 安全：不允许 guest 触发宿主 panic ──
    config.wasm_backtrace(false);              // 关闭以减小体积（诊断时另开）
    config.native_unwind_info(false);

    // ── WASI ──
    // ★ 只允许插件需要的 WASI 接口；默认不提供文件系统与网络
    //   通过 component linker 的实例化策略控制（见 §3.1）

    Engine::new(&config)
}
```

### 3.1 WASI 的授予策略

**默认不提供任何 WASI 能力**。插件需要的所有系统交互都通过 `synra:plugin/host` 的 host functions（它们过闸门）。

```rust
pub struct WasiGrant {
    /// 是否允许时钟（插件需要时间戳时）
    pub clocks: bool,                  // 默认 true（只读，无害）
    /// 是否允许随机数
    pub random: bool,                  // 默认 true（无害）
    /// 是否允许标准输出（用于调试日志）
    pub stdout: bool,                  // 默认 false（改用 host.log）
    /// 是否允许文件系统预打开目录（**默认空**）
    pub preopens: Vec<(Utf8PathBuf, Utf8PathBuf)>,
    /// 是否允许环境变量
    pub env: Vec<(String, String)>,    // 默认空
    /// 是否允许 socket
    pub sockets: bool,                  // ★ 默认 false，且**建议永不开启**
}
```

> **为什么 sockets 建议永不开启**：如果插件能直接建 socket，capability 的 `network:fetch:<host>` 就形同虚设（插件可以绕过内核直连任意地址）。
> 即使 WASI sockets 支持 DNS 白名单，也无法做"重定向逐步校验"与"IP 策略"。**网络必须经内核代理**。

---

## 4. 中断与超时（两层防护）

```rust
// crates/synra-plugin-host/src/instance.rs

pub struct CallBudget {
    /// 指令预算（确定性）
    pub fuel: u64,
    /// 墙钟超时（非确定性，兜底）
    pub timeout: Duration,
    /// 内存增长上限
    pub max_memory_growth: u64,
}

impl PluginInstance {
    pub async fn call<T: Export>(
        &mut self,
        args: T::Args,
        budget: CallBudget,
    ) -> Result<T::Result, HostError> {
        // ── 层 1：fuel ──
        // 设置本次调用的指令预算。耗尽时 wasmtime trap with OutOfFuel。
        self.store.set_fuel(budget.fuel)?;

        // ── 层 2：epoch（墙钟）──
        // 计算 deadline 对应的 epoch 值；epoch ticker 任务每隔 epoch_period 递增。
        let deadline_epoch = self.compute_deadline_epoch(budget.timeout);
        self.store.set_epoch_deadline(deadline_epoch);

        // ── 层 3：内存增长限制 ──
        self.limiter.reset_growth_counter(budget.max_memory_growth);

        // 执行（★ 注意：不能直接 block_in_place 在一个 worker 上，
        //   否则并发插件会互相饿死；见 §4.2）
        let result = tokio::task::block_in_place(|| {
            self.binding.call_execute(&mut self.store, args)
        });

        match result {
            Ok(v) => Ok(v),
            Err(e) if e.is_out_of_fuel() => Err(HostError::FuelExhausted { budget: budget.fuel }),
            Err(e) if e.is_epoch_deadline() => Err(HostError::Timeout { after: budget.timeout }),
            Err(e) if e.is_memory_growth_exceeded() => Err(HostError::MemoryExceeded),
            Err(e) => Err(HostError::WasmTrap(e)),
        }
    }
}
```

### 4.1 为什么 fuel 不够，还需要 epoch

| 场景                                        | fuel 能挡住？ | epoch 能挡住？ |
| ------------------------------------------- | ------------- | -------------- |
| 插件里的死循环（纯计算）                      | ✅            | ✅             |
| 插件调用 host function，host 阻塞（如 IO）     | ❌（guest 没消耗指令） | ✅      |
| 插件分配巨大内存                              | ❌            | ❌（需 memory limiter） |
| 插件在 host function 里递归调用回 guest         | ⚠️ 部分        | ✅             |

**所以三者都要**：fuel + epoch + memory limiter。

### 4.2 不要在异步 worker 上直接执行 Wasm

wasmtime 的执行是**同步阻塞**的。如果在 tokio worker 线程上直接跑：

```rust
// ❌ 会阻塞 tokio worker，导致其它任务的调度延迟
let result = binding.call(&mut store, args);
```

有两种正确做法：

| 做法                          | 说明                                                       | 代价                     |
| ----------------------------- | ---------------------------------------------------------- | ------------------------ |
| **专用线程池**（推荐）          | 一组专用 OS 线程执行 Wasm；`tokio::sync::oneshot` 拿结果    | 线程数 = 并发插件调用上限  |
| `block_in_place` + 多线程运行时 | 让 tokio 把该 worker 换出去（tokio 自动补齐 worker）         | 依赖多线程 runtime       |

**推荐专用线程池**：

```rust
pub struct WasmExecutor {
    /// 专用线程池（与 tokio 的 worker 池分离）
    pool: rayon::ThreadPool,
    /// 并发上限
    semaphore: Arc<Semaphore>,
    /// epoch ticker
    epoch_handle: EpochTicker,
}

impl WasmExecutor {
    pub async fn run<F, R>(&self, f: F) -> R
    where F: FnOnce() -> R + Send + 'static, R: Send + 'static
    {
        let permit = self.semaphore.clone().acquire_owned().await.unwrap();
        let (tx, rx) = tokio::sync::oneshot::channel();
        self.pool.spawn(move || {
            let _permit = permit;
            let _ = tx.send(f());
        });
        rx.await.expect("executor thread panicked")
    }
}
```

**并发上限**（`Semaphore`）必须存在，否则 N 个插件同时调用会创建 N 个线程。

### 4.3 epoch ticker

```rust
pub struct EpochTicker {
    handle: JoinHandle<()>,
    cancel: CancellationToken,
}

impl EpochTicker {
    /// 每 `period` 递增一次所有插件的 epoch
    pub fn start(engine: Engine, period: Duration) -> Self {
        // 默认 period = 10ms
        // → 超时精度 10ms，对插件调用（通常 < 50ms）足够
        // 更小 → 精度高但唤醒开销大
    }
}

impl Drop for EpochTicker {
    fn drop(&mut self) { self.cancel.cancel(); }
}
```

**注册为受监督任务**（[07 §5.2](./07-core-kernel.md)）：如果 ticker 死了，所有超时都会失效 → 必须重启。

---

## 5. Host Function 的实现模式

```rust
// crates/synra-plugin-host/src/host_funcs/network.rs

gate_host_fn!(
    name = "network-fetch",
    capability = CapabilityPattern::parse("network:fetch:*"),
    handler = network_fetch
);

async fn network_fetch(
    ctx: &mut PluginCtx,
    request_json: String,
) -> Result<Outcome, HostError> {
    // 1. 解析请求
    let req: FetchRequest = serde_json::from_str(&request_json)
        .map_err(|e| HostError::InvalidArgument { detail: e.to_string() })?;

    // 2. ★ 派生实际能力标识并过闸门
    let host = normalize_host(&req.url)?;                    // §10 §2.3 的规范化
    let cap = Capability::network_fetch(&host, req.method.as_deref());
    ctx.gate.check(&cap)?;

    // 3. ★ 配额与限流
    ctx.quota.consume(RateKind::NetworkRequestsPerSec, 1)?;
    ctx.quota.acquire_permit(RateKind::NetworkConcurrent).await?;

    // 4. ★ SSRF 防护（scheme / DNS / IP / 重定向）
    let response = ctx.network.fetch_guarded(req, &ctx.quota.ip_policy).await?;

    // 5. 审计（不含 body）
    ctx.audit.record(AuditSummary::NetworkFetch {
        host: host.clone(),
        method: req.method.clone(),
        status: response.status,
    });

    // 6. 返回（响应体大小受限）
    Ok(Outcome::ok(serde_json::to_string(&response)?))
}
```

**统一模式**（每个 host function 都遵循）：

```
解析参数 → 派生能力标识 → gate.check → quota.consume → IP/路径/事件名校验
        → 执行业务 → 审计 → 序列化返回
```

**代码生成的辅助**：可以用宏把"解析 + gate + quota + audit"这部分模板化，只留业务逻辑给 handler。

---

## 6. 实例池与状态管理

### 6.1 池化前提：插件必须无状态（或状态在 storage）

```wit
// manifest 声明
{ "runtime": { "headless": { "artifact": "...", "stateful": false } } }
```

- `stateful: false`（默认，推荐）→ 实例可池化复用；
- `stateful: true` → 每次调用创建新实例（慢），或要求插件实现 `snapshot`/`restore`（[14 §7](./14-plugin-lifecycle.md)）。

### 6.2 池实现

```rust
pub struct PluginPool {
    compiled: Arc<CompiledPlugin>,
    idle: ArrayQueue<PooledInstance>,
    total: AtomicUsize,
    config: PoolConfig,
}

impl PluginPool {
    pub async fn acquire(&self) -> Result<InstanceLease, PoolError> {
        // 1. 尝试从 idle 取（无锁）
        if let Some(inst) = self.idle.pop() {
            // ★ 复用前校验实例健康（epoch 未过期、无累积 fuel 问题）
            if inst.is_healthy() { return Ok(InstanceLease::new(inst)); }
            // 不健康 → 丢弃，继续
        }
        // 2. 若 total < max_instances → 新建
        // 3. 否则等待（有超时）
    }
}
```

### 6.3 空闲回收

```
每 30s 扫描所有池：
  · 空闲 > idle_ttl（5 min）的实例 → 销毁，但保留至少 1 个（若 warmup）
  · 长期未使用的插件的整个池 → 销毁（插件保持"已启用"但无实例）
```

**指标**：池大小、命中率、创建次数、平均租借时长（[18](./18-observability.md)）。

---

## 7. 故障隔离

### 7.1 可能的故障与影响面

| 故障                          | Wasm 层表现          | 影响面                          | 处理                                    |
| ----------------------------- | -------------------- | ------------------------------- | --------------------------------------- |
| 无限循环                       | `OutOfFuel` / epoch  | 仅本次调用                       | 返回错误，记入健康度                      |
| 栈溢出                         | trap                 | 仅本次调用                       | 返回错误 + 记录（可能是插件 bug）          |
| 越界内存访问                    | trap                 | 仅本次调用                       | 返回错误（**Wasm 保证不影响宿主**）         |
| 内存超限                       | `MemoryLimit` trap   | 仅本次调用                       | 销毁实例，返回错误                        |
| 调用不存在的导出                | 链接错误              | 装载期就发现                     | 拒绝装载                                 |
| 恶意 host function 参数         | 校验失败              | 仅本次调用                       | 返回 `InvalidArgument`                   |
| 插件 panic（Rust guest）        | trap（`unreachable`） | 仅本次调用                       | 返回错误；若频繁则 Faulted                 |
| **宿主 bug 导致 panic**         | —                    | **可能杀死整个进程**              | ★ 见 §7.2                               |

### 7.2 宿主 panic 的防护

Wasm 隔离保护的是"guest 不能破坏宿主"。但**宿主自己的 bug** 仍可能 panic。措施：

1. **`catch_unwind` 包住所有 wasmtime 调用**：

```rust
let result = std::panic::catch_unwind(AssertUnwindSafe(|| {
    executor.run_blocking(|| binding.call(&mut store, args))
}));
match result {
    Ok(r) => r,
    Err(panic) => {
        // 记录 panic（含 backtrace）
        error!(?panic, plugin = %plugin_id, "host panicked during wasm call");
        // 实例不可信了 → 销毁
        Err(HostError::HostPanic)
    }
}
```

2. **内核设置 panic hook**：把 panic 写进日志与诊断包，而不是丢弃；
3. **关键任务里的 panic 视为任务失败**（由 supervisor 决定是否重启）；
4. **`panic = "abort"` 不适用于本场景** —— 会让一次插件调用中的宿主 bug 杀死整个应用。使用 `panic = "unwind"`（默认）。

> 这是一个重要取舍：`panic = "abort"` 能减小体积，但在"跑第三方代码"的场景里代价太大。

### 7.3 健康度模型

```rust
pub struct PluginHealth {
    /// 滑动窗口内的调用统计
    window: SlidingWindow,
    /// 当前状态
    state: HealthState,
}

pub enum HealthState {
    Healthy,
    /// 近期失败率偏高 → 降低配额
    Degraded { since: Instant, reason: DegradeReason },
    /// 持续失败 → 已停用
    Faulted { since: Instant, reason: FaultReason },
}

impl PluginHealth {
    /// 每次调用后更新；返回建议的动作
    pub fn record(&mut self, outcome: CallOutcome) -> HealthAction {
        // 规则（示意）：
        //  · 连续 5 次 trap → Degraded
        //  · 连续 20 次 trap，或 1 分钟内 > 50% 失败 → Faulted
        //  · 内存超限 3 次 → Faulted（这个插件太大了）
        //  · 配额超限 → Degraded（降配额）+ 用户通知
        //  · 成功 → 逐步恢复（连续 100 次成功从 Degraded 回 Healthy）
    }
}
```

---

## 8. 编译与缓存

### 8.1 编译时机

| 时机            | 桌面                              | 移动                              |
| --------------- | --------------------------------- | --------------------------------- |
| 安装时（AOT）    | ✅ 预编译并缓存                    | ✅ **必须**（避免运行时编译卡顿）    |
| 首次激活         | 从缓存加载（~ms）                  | 从缓存加载                          |
| 缓存未命中       | 编译（几十 ms ~ 几百 ms）+ 提示     | 编译（可能几秒）+ 进度提示            |

```rust
pub struct CompileCache {
    dir: Utf8PathBuf,          // <data-dir>/cache/wasm/<hash>.cwasm
    index: SqliteIndex,        // hash → { compiled_at, backend_kind, wasmtime_version }
}

impl CompileCache {
    /// 缓存键必须包含所有影响编译结果的因素
    fn key(&self, wasm_sha256: &Sha256Digest, backend: BackendKind) -> String {
        format!("{}-{}-{}", wasm_sha256, backend, WASMTIME_VERSION)
    }
}
```

**必须**在缓存键里包含 **wasmtime 版本**与**后端类型**，否则升级 wasmtime 后会加载不兼容的预编译产物。

### 8.2 编译的 DoS 风险

Wasm **验证与编译本身**是攻击面（一个精心构造的模块可以让编译器消耗大量内存/时间）。

措施：
1. **限制输入大小**：`.wasm` ≤ 上限（例如 32 MiB，压缩后 ≤ 8 MiB）；
2. **在独立任务里编译**，带超时（例如 30s），超时则拒绝；
3. **限制编译内存**（wasmtime 的 `Config` 有相关选项，或用一个受限的线程）；
4. **编译失败不缓存**，但记录失败原因（避免重复攻击）；
5. 移动端 AOT 在**安装时**做，且有进度与可取消。

---


| ---------------- | ------------------------------------- | ------------------------------------------- |
| 执行位置         | WebView 主 JS context（"宿主决定"）     | 内核内 Wasm 实例（headless）/ 独立 WebView（UI） |
| 隔离             | 无（同 realm）                        | Wasm 线性内存 + 无 syscall                   |
| 中断             | 无法中断                               | fuel + epoch + memory limit                 |
| 内存上限         | 无                                    | 硬上限                                      |
| 实例复用         | module instance（V8 缓存）             | 显式实例池                                  |
| 编译             | V8 解析 JS                             | AOT 预编译 + 缓存（键含版本）                 |
| host 调用        | JS Proxy（约 0.3 μs/次）               | Wasm host function（约 1–10 μs/次，含闸门）   |
| 崩溃影响         | 拖垮整个 SPA                          | 仅本次调用                                  |
| 宿主 panic 防护   | N/A                                   | `catch_unwind` + 实例销毁                    |
| 移动端           | 同 WebView                            | pulley/winch AOT（无 JIT）                   |

> **性能预期**：Wasm host call 比 JS Proxy 慢一个数量级（1–10 μs vs 0.3 μs）。但**绝对数量级仍是微秒**，对插件逻辑（通常做 IO 而非密集计算）无实质影响。
> 真正的性能收益在别处：**不再有跨 realm 的数据序列化、不再有 Vue 响应式开销**。

---

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
      ★ 未做的一半：**panic hook 接入 tracing** —— `synra-observe` 里没有 tracing 初始化（见 [18](./18-observability.md)）。
- [ ] `gate_host_fn!` / `ungated_host_fn!` 宏 —— **未做，且这两个宏从未存在**。
      真实机制更强：`build.rs` 从 `wit/synra-plugin.wit` 的 `@ungated` / `@capability("…")` 注释
      **每次构建重新生成** `src/generated/capability_table.rs`。
      见 [10](./10-capability-and-sandbox.md) 与 [11](./11-design-principles.md) 的同名条目。
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
        而之前唯一的写入路径是 `Kernel::record_action`，那个类型没有任何孿主构造（见 [07](./07-core-kernel.md) 开头）。
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
- [ ] 编译任务超时与输入大小限制 —— **未做**：`wasmtime_backend.rs:566-572` 的 `compile`
      只检查"是不是空的"。CLI 侧有 `MAX_WASM_BYTES`、registry 侧有 `MAX_PACKUMENT_BYTES`，宿主侧没有。
- [x] `InstancePool` + 空闲回收 + 池指标 —— **A/C**：真名 `pool::InstancePool`
      （不是 `PluginPool`），`pool/instance.rs:278` `evict_idle(now_ms)`、`:317` `PoolReport`。
      ★ `evict_idle` 只被测试调用，没有任何扫描器；`synra-observe` 里也**没有池指标**。
- [x] 健康状态机 + 降级/停用策略 —— **A/C**：真名 `quota::Health`
      （不是 `PluginHealth`/`HealthState`），`quota/budget.rs:25` 四档 + `:16/:19` 阈值 +
      `:152-159` 降级配额，`quota/tests.rs:290-364` 有完整阶梯测试。
      ★ **触发源不对**：降级由**速率限制拒绝**驱动，不是 `CallOutcome::Trap` / `MemoryExceeded`。
      也就是说"5 次 trap → 降级"这条路径没有实现 —— 一个反复 trap 的插件**永远不会**被降级。
- [ ] 测试：死循环（fuel）、阻塞 host（epoch）、内存炸弹、trap 恢复、并发隔离 ——
      ★ **五项里 0 项**由一个真跑 wasm 的测试覆盖：
      fuel 的唯一真用例 `wasmtime_tests.rs:306` 被 `#[ignore]`；
      epoch **完全无测试**（`:334` 只断言"deadline 被写进去了"）；
      内存炸弹无 guest 测试（`:354` 只断言 ceiling 已安装）；
      trap 恢复只有 `ScriptedBackend` 桩；并发隔离测的是线程池，不跑 wasm。
      本 crate 里唯一执行真 wasm 的两个测试都被 `#[ignore]`（`wasmtime_tests.rs:270, 306`）。
      ★ **但是 `#[ignore]` 的理由值得照抄下来，它比"没做"更有用**：
      两条的注解都是
      "the hand-written fixture does not match the canonical ABI for a `string` parameter and result;
      the lookup itself now works, and a fixture built by `bindgen!` from the WIT file is what would close this"。
      也就是说**缺的是夹具而不是被测代码**：手写的 `.wat` 与 canonical ABI 对不上，
      而 `bindgen!` 从 `wit/synra-plugin.wit` 生成的夹具才是出路。
      ★ 这条路径**依赖环境**：`synra-plugin-sdk/tests/guest/world_fixture.rs` 自己写明
      `wasm32-wasip2` 目标与 `wasm-tools` 都没有安装。
      所以这一条要真正勾上，先要有那个目标 —— 属于"需要环境"的一类，本次不碰。
