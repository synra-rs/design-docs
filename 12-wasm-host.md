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
    config.epoch_interruption(true);           // ★ Wasm 执行期间的墙钟中断

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

默认不提供任何 WASI 能力，当前 WIT world 也没有 WASI 导入。
SDK 使用 wasm32-unknown-unknown core module 与内置组件编码，避免 wasip2 std 自动引入 cli/io/clocks。
系统交互均经 synra:plugin/host 的能力闸门，日志用 Host log，网络与文件不能绕过宿主。

若将来引入时钟、随机数等接口，必须另行设计显式授权与资源边界；
不能把“只读”当作自动授权，也不能由编译器或 adapter 顺带暴露 WASI 系统能力。

### 3.2 组件初始化预算

组件初始化发生在首个业务调用之前，也可能执行 guest 构造函数，因此必须配置独立的有限预算。
当前启动上限为 1,000,000 fuel、1 秒 epoch deadline 和每个线性内存 16 MiB；
这些是当前策略值，不是移动端负载测量结果，后续调用按调用预算重新设置限制。
启动期间尚无授权会话，Host 调用仍须默认拒绝。

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

### 4.1 fuel、epoch 与 Host IO 的边界

| 场景 | 指令 fuel | epoch deadline | 其他必要限制 |
| --- | --- | --- | --- |
| Guest 纯计算或无限循环 | 可终止 | 可终止 | 调用并发与队列上限 |
| 同步 Host IO 阻塞 | 不能终止 | 不能抢占 Host Rust 代码 | IO 自身的总截止时间、取消与资源上限 |
| Guest 内存增长 | 不代替内存限制 | 不代替内存限制 | ResourceLimiter |
| Host 再次进入 Guest | 只约束正在执行的 Wasm | 只约束正在执行的 Wasm | 重入规则、递归与并发上限 |

epoch 依赖持续推进的引擎计数，只在 Wasm 执行期间检查，不能替代同步 Host 操作的超时。
IO 必须由宿主提供自己的 deadline 与取消机制，异步适配还须明确取消后的提交和资源清理语义；
把 Wasm 放到另一个线程不会自动终止那个线程中正在阻塞的 IO。
当前纯计算 epoch 用例不作为 Host IO 有界终止的证明。

每次业务调用都重新设置 fuel 与 deadline，未指定时不能继承构造函数或上一调用的有限额度。
无指令上限使用足够大的 fuel，无时限使用不会在当前 epoch 加法中溢出的相对值；
没有指定内存额度时仍保留初始化内存限制。
发生 trap 等异常后由实例池丢弃实例，不能将无法再次进入的组件当作健康实例复用。

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

### 5.1 WIT 类型与返回值所有权

生产 `wit/synra-plugin.wit` 是组件签名的唯一来源，Host 注册和类型由 Wasmtime bindgen 生成，
禁止另维护签名、参数数量或返回形状表来证明自身一致。
适配器只负责将类型值交给共享 HostCalls，并保留各参数的业务含义；
outcome 使用 WIT variant，枚举、record、tuple、list 与 option 保持相应类型，不以任意字符串或默认值替代。

Guest 入口须在执行前匹配完整参数与结果类型；调用后取得宿主持有的返回值，并执行 canonical post-return。
类型不符或 post-return 失败均作为失败处理，不能丢弃结果或返回空字符串伪装成功。
Host 调用计数只记录实际导入调用的尝试，包括拒绝，不把一次 Guest 入口调用计为一次 Host 调用。

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

## 10. 当前实现与验证边界

Wasm ABI 的当前实现以生产 WIT 与真实组件测试为准，旧逐轮观察归档于
[调查快照](./history/wasm-abi-observations.md)，修复证据见 [2026-10-26 记录](./history/2026-10-26-wasm-abi.md)。

| 范围 | 当前实现或验收边界 |
| --- | --- |
| Host 绑定 | `engine/wasmtime_backend/bindings.rs` 从生产 WIT 生成类型与注册，`engine/linker/host.rs` 实现全部 Host trait 并经过共享闸门 |
| Guest 调用 | `engine/wasmtime_backend/invoke.rs` 在执行前校验参数与结果类型，读取字符串或 outcome，并执行 post-return |
| 初始化 | 独立启动 fuel、epoch deadline 与内存限制，正常构造和无限循环构造均有真实组件测试 |
| 返回与清理 | 五种 outcome 入口覆盖成功、失败与重复调用，错误 code、message、可选 details 完整保留，post-return 失败不能报告成功 |
| 真实 Host 边界 | 覆盖无会话、无权限、允许读取及匹配阶段禁止写入，并检查宿主实际效果和调用计数 |
| 类型不符 | 错误 Host 签名在链接时拒绝，错误 Guest 返回类型在执行 guest 指令前拒绝 |
| 完整目标 | 这些测试不等于 Rust SDK 构建、安装、路由、授权和平台动作全链路验收，也不证明 Android/iOS 运行或性能预算达标 |

AOT、安全预编译加载、其他引擎后端、平台运行和性能验收仍须各自提供证据；
编译通过、配置存在与测试通过不能互相代替。
整体待办继续以 [19](./19-roadmap.md) 为准，不将本次 ABI 修复记作 M2 或完整开发目标完成。

WIT 的 details 是可选字符串；引擎不解析或重写，须区分缺失、空字符串、JSON null、JSON 文本及非 JSON 文本。
主动报告 Failed 不算插件失常；上层可以分类错误，但不能丢弃原始插件错误码。
Wasm 与页面 RPC 的当前传输规则及回归见[错误详情记录](./history/2026-10-26-error-details.md)。
CallOutcome 到协议错误的分类现由共享宿主层拥有，页面、本地及远程意图复用；
预算拒绝与主动失败保留不同类别，原始插件码和详情留在结构化上下文，见[意图结果记录](./history/2026-10-26-intent-outcomes.md)。
