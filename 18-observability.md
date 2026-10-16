# 18 · 可观测性与诊断

> 本文定义 Rust 版的日志、指标、追踪、审计与诊断包。

---

## 1. 三根支柱

```
┌─────────────────────────────────────────────────────────────────────┐
│  日志（Logging）        tracing + 结构化 JSON                        │
│  · 内核、网络、插件、IPC 全链路                                       │
│  · 按 level / target / trace_id 过滤                                 │
├─────────────────────────────────────────────────────────────────────┤
│  指标（Metrics）        自研轻量注册表（或 metrics crate）             │
│  · 连接数、传输速率、插件调用、配额拒绝、错误率                          │
│  · 内存中聚合 + 可导出                                                │
├─────────────────────────────────────────────────────────────────────┤
│  追踪（Tracing）        自研 span 上下文 + trace_id 随信封传播         │
│  · 一次用户操作 → 跨设备 → 全链路可查                                  │
└─────────────────────────────────────────────────────────────────────┘
```

**审计（Audit）** 是第四类（[10 §7](./10-capability-and-sandbox.md)）：插件能力的允许/拒绝记录，面向用户透明度，不属于传统可观测性。

---

## 2. 日志

### 2.1 `tracing` 配置

```rust
// crates/synra-observe/src/logging.rs

pub struct LogConfig {
    pub level: LevelFilter,             // 默认 Info
    pub format: LogFormat,              // Json（生产）| Pretty（开发）
    pub dir: Utf8PathBuf,
    pub file_prefix: String,            // "kernel"
    pub rotation: Rotation,             // Daily | Hourly | Size
    pub max_file_bytes: u64,            // 64 MiB
    pub retention_days: u32,            // 14
    pub console: bool,                  // daemon/CLI 为 true
    pub ansi: bool,
}

pub fn init(cfg: &LogConfig) -> Result<LogGuard, ObserveError> {
    let file_appender = tracing_appender::rolling::Builder::new()
        .rotation(cfg.rotation.into())
        .filename_prefix(&cfg.file_prefix)
        .max_log_files(cfg.retention_days as usize)
        .build(&cfg.dir)?;

    let (non_blocking, guard) = tracing_appender::non_blocking(file_appender);

    let subscriber = tracing_subscriber::registry()
        .with(cfg.level)
        .with(
            tracing_subscriber::fmt::layer()
                .json()
                .with_current_span(true)
                .with_span_list(true)
                .with_writer(non_blocking)
                .with_timer(UtcTime::rfc_3339())
        )
        // ★ 敏感信息过滤层（见 §2.3）
        .with(RedactionLayer::default())
        // ★ 插件日志加来源标记
        .with(PluginAttributionLayer::default());

    tracing::subscriber::set_global_default(subscriber)
        .map_err(|e| ObserveError::SubscriberInit(e.to_string()))?;

    Ok(LogGuard { _guard: guard })
}
```

### 2.2 字段规范

**每个 span 必须携带的字段**（用 `#[instrument]` 或显式 `span!`）：

| 字段              | 何时有            | 说明                            |
| ----------------- | ----------------- | ------------------------------- |
| `trace_id`        | 几乎所有操作       | 16 进制 32 位                    |
| `device_id`       | 设备相关           | 对端设备 id                      |
| `plugin_id`       | 插件相关           | 插件 id                          |
| `request_id`      | 报文相关           | 信封 id                          |
| `transfer_id`     | 传输相关           | 会话 id                          |
| `intent_id`       | 意图相关           | 意图 id                          |
| `task_id`         | supervisor 任务    | 稳定任务标识                     |

```rust
#[tracing::instrument(
    skip(self, frame),
    fields(
        device_id = %peer,
        event = %frame.event,
        request_id = %frame.request_id,
        trace_id = %frame.trace_id.to_hex(),
    )
)]
async fn send_frame(&self, peer: DeviceId, frame: Frame) -> Result<SendOutcome> { /* ... */ }
```

### 2.3 敏感信息过滤（必须）

**绝不允许进日志**：
- 剪贴板内容；
- 消息 payload 的正文；
- 文件内容；
- 密钥 / token；
- URL 的 query 与 fragment（可能含 token）；
- 插件的 secure storage 值。

**实现**：自定义 `Layer` 做字段名匹配 + 值截断：

```rust
pub struct RedactionLayer {
    /// 需要整体替换的字段名（大小写不敏感）
    redact_keys: HashSet<&'static str>,   // "payload", "body", "content", "token", "secret", ...
    /// 其余值的最大长度
    max_value_len: usize,                  // 默认 512
}

impl<S: Subscriber> Layer<S> for RedactionLayer {
    fn on_event(&self, event: &Event<'_>, _ctx: Context<'_, S>) {
        // 用 visitor 遍历字段：
        //  · 字段名命中 redact_keys → 值写为 "<redacted>"
        //  · 值是字符串且长度 > max_value_len → 截断并加 "…(len=N)"
        //  · URL 类型 → 去掉 query/fragment
    }
}
```

**要求**：这条有**测试**（[21 §6](./21-testing-strategy.md)）：给日志宏传 payload，断言输出里不含原文。

### 2.4 日志的用途分层

| 级别    | 用途                                | 默认保留 |
| ------- | ----------------------------------- | -------- |
| `ERROR` | 用户可感知的失败                      | 14 天    |
| `WARN`  | 异常但已恢复（重连、配额拒绝）         | 14 天    |
| `INFO`  | 关键状态变化（连接、安装、激活）        | 14 天    |
| `DEBUG` | 流程细节（仅开发/诊断时开启）           | 不落盘（除非开启） |
| `TRACE` | 帧级细节（仅开发）                    | 不落盘   |

**默认 `INFO`**。用户在设置里可以临时开 `DEBUG`（限时 30 分钟，自动回落）——这是重要的排障手段。

---

## 3. 追踪（跨设备）

### 3.1 `TraceContext`

```rust
// crates/synra-observe/src/trace.rs

#[derive(Debug, Clone, Copy, PartialEq, Eq, Hash, Serialize, Deserialize)]
pub struct TraceContext {
    pub trace_id: TraceId,
    /// 当前 span（用于父子关系可视化）
    pub span_id: SpanId,
    /// 采样决策（避免全量追踪的高开销）
    pub sampled: bool,
}

impl TraceContext {
    /// 新开一个子 span
    pub fn child(&self) -> Self;
    /// 从入站信封恢复（v2 协议携带 trace_id）
    pub fn from_envelope(env: &Envelope) -> Option<Self>;
}
```

### 3.2 随信封传播

v1 协议的信封是 7 字段白名单，**无法携带 trace_id**。因此：

| 场景                   | trace 行为                                                   |
| ---------------------- | ------------------------------------------------------------ |
| 本机操作               | 完整 trace                                                    |
| 出站到支持 v2 的对端     | `trace_id` 写入信封，对端**延续**同一 trace                     |
| 入站来自支持 v2 的对端    | 延续 trace                                                    |

**实现**：`Envelope` 内部**始终**有 `trace_id`；编码时按协商结果决定是否写入。

### 3.3 采样

全量追踪的日志量太大（一个文件传输有 80 个 chunk）。策略：

```rust
pub struct Sampler {
    /// 错误/慢请求总是采样
    pub always_on_error: bool,        // true
    pub always_slower_than: Duration, // 200ms
    /// 其余按概率
    pub base_rate: f64,               // 0.01（1%）
    /// 特定 trace 强制开启（用户点了"详细记录"）
    pub force: RwLock<HashSet<TraceId>>,
}
```

**用户可操作**：诊断页可以"对下一次操作开启详细追踪"（`force` 集合）。

### 3.4 跨设备追踪的 UI

```
设置 → 诊断 → 追踪
┌───────────────────────────────────────────────────────────────────┐
│  trace 7f3a…c21  （手机 → PC，耗时 412ms）                          │
│                                                                   │
│  10:23:41.002  [mobile] intent.dispatch    share=url github.com/…  │
│  10:23:41.004  [mobile] plugin.match       candidates=1            │
│  10:23:41.006  [mobile] capability.check   plugin=chat cap=device…  │
│  10:23:41.007  [mobile] transport.send     peer=laptop             │
│  10:23:41.198  [laptop] transport.recv     ← ★ 跨设备延续同一 trace   │
│  10:23:41.201  [laptop] gate.check         platform:open-url ok     │
│  10:23:41.402  [laptop] platform.open_url  url=github.com/…         │
│  10:23:41.409  [laptop] intent.result      ok                       │
│  10:23:41.414  [mobile] intent.finished    elapsed=412ms            │
└───────────────────────────────────────────────────────────────────┘
```

这需要：
1. 对端把它的 span 也写进**自己的日志**（带同一 `trace_id`）；
2. 诊断包**收集所有配对设备**的相关日志（可选：用户手动从每台设备导出，或经 `_synra.diagnostics.request` 拉取——**需要用户授权**）。

**第一版**：只做本机 trace。跨设备 trace 的展示作为 M5+ 的增强（但 trace_id 的**传播**要在 M1 就做，否则以后无法回溯）。

---

## 4. 指标

>
> ★ **而第 76 轮把这条链接通了**，并且选了 `status` 而不是新开一个读者：它是一个
> **已经存在、已经能用**的命令，而它的返回值是 `serde_json::json!`，所以加一个字段是**增量**的。
> ★ 而读者**直接序列化 `MetricsSnapshot`**，不重新塑形：那四个类型本来就 `derive(Serialize)`
> 并且 `rename_all = "camelCase"` —— 它们就是为了跨这道边界而写的。第一版把快照拆开重建成
> `{name, labels, value}`，那是**同一份数据的第二种拼法**：读者得同时懂两种，而没有任何东西
> 保持它们一致。
> ### ★ 而这一整组指标的真实状态（第 75 轮核对）
>
> 设计写的是一个带命名字段的 `Metrics` 结构；实现出来的是 `RingRecorder` + `SeriesKey`（名字 + 标签）。
> 两者形状不同，而**真正的问题不是形状**：这一组指标之前**一个生产者都没有** ——
> `RingRecorder` 写好、测试齐全，而全仓没有任何地方调 `counter_inc`。
>
> | 环节 | 状态 |
> | --- | --- |
> | 生产者 | **已有**（第 75 轮）：闸门记下 `gate_checks_total`、`gate_denied_total`、`gate_rate_limited_total` |
> | 传输 | **已通**（第 76 轮）：守护进程持有记录器 → `PluginSet::with_metrics` → `Authority` → `LinkerSession` → `HostCalls` |
> | 读者 | **已有**（第 76 轮）：`status` 的 `metrics` 字段报告记录器自己的 `MetricsSnapshot` |
> | `diag_metrics` | **仍然未做**：只在 `command.rs` 的表里，`synra-control` 的 `Kernel` trait 没有它，守护进程也没有 handler —— 但现在它背后的数据存在了 |
>
> ★ **而为什么从闸门开始**：它是唯一一个**已经知道这四个答案**的地方 —— 是否通过、
> 哪个插件、哪个能力，以及没通过时是**能力集还是限流器**说的不。在别处记一次，
> 都是对这里已经做出的决定的第二种意见。
>
> ★ 而标签用 `AuditOutcome::as_str()` 而不是另写一套名字：审计里的一行和指标里的
> 一个计数因此**不可能对同一件事说不同的话**。
>
### 4.1 指标清单

```rust
// crates/synra-observe/src/metrics.rs

pub struct Metrics {
    // ── 网络 ──
    pub links_open: Gauge,                        // 当前打开的链路数
    pub links_opened_total: Counter,              // 累计打开
    pub links_closed_total: CounterVec,           // 按 reason 分类
    pub link_handshake_ms: Histogram,             // 握手耗时
    pub frames_sent_total: CounterVec,            // 按 event 类型
    pub frames_received_total: CounterVec,
    pub frames_dropped_total: CounterVec,         // 按原因（过大/超时/限流）
    pub bytes_sent_total: Counter,
    pub bytes_received_total: Counter,
    pub reconnect_attempts_total: CounterVec,     // 按结果

    // ── 发现 ──
    pub scans_total: CounterVec,                  // 按结果
    pub scan_duration_ms: Histogram,
    pub candidates_found: Histogram,
    pub udp_packets_total: CounterVec,
    pub mdns_services_total: Counter,

    // ── 意图 ──
    pub intents_total: CounterVec,                // 按结果
    pub intent_duration_ms: Histogram,            // 按插件
    pub intent_match_duration_ms: Histogram,
    pub intent_no_handler_total: Counter,

    // ── 传输 ──
    pub transfers_total: CounterVec,              // 按 kind + 结果
    pub transfer_bytes: Histogram,
    pub transfer_duration_ms: Histogram,
    pub transfer_throughput_bps: Histogram,
    pub transfer_retries_total: Counter,

    // ── 插件 ──
    pub plugins_installed: Gauge,
    pub plugins_active: Gauge,
    pub plugin_install_total: CounterVec,         // 按 stage + 结果
    pub plugin_calls_total: CounterVec,           // 按 plugin + 接口
    pub plugin_call_duration_us: HistogramVec,
    pub plugin_fuel_consumed: HistogramVec,
    pub plugin_memory_bytes: GaugeVec,
    pub plugin_faulted_total: CounterVec,
    pub plugin_pool_hit_total: Counter,
    pub plugin_pool_miss_total: Counter,

    // ── 闸门 ──
    pub gate_checks_total: CounterVec,            // 按结果
    pub gate_denied_total: CounterVec,            // 按 plugin + capability
    pub gate_quota_exceeded_total: CounterVec,
    pub gate_rate_limited_total: CounterVec,

    // ── 存储 ──
    pub db_size_bytes: Gauge,
    pub db_write_duration_us: Histogram,
    pub plugin_data_bytes: GaugeVec,

    // ── 进程 ──
    pub memory_rss_bytes: Gauge,
    pub cpu_percent: Gauge,
    pub threads: Gauge,
    pub uptime_seconds: Counter,
}
```

### 4.2 实现选型

| 方案                         | 特点                          | 结论      |
| ---------------------------- | ----------------------------- | --------- |
| `metrics` crate + exporter    | 生态标准、facade 与实现分离    | **采用**  |
| `prometheus` crate            | 重量级                        | 不用      |
| `opentelemetry-rust`          | 完整但依赖重、变化快           | M6+ 可选  |

**采用 `metrics` crate（facade）+ 自研内存 recorder**：

```rust
pub struct RingRecorder {
    counters: DashMap<Key, AtomicU64>,
    gauges: DashMap<Key, AtomicI64>,
    histograms: DashMap<Key, Histogram>,
}
```

**为什么不用标准 exporter**：桌面应用不需要 HTTP scrape 端点。指标主要用于：
1. **诊断包**（导出快照）；
2. **设置页的"性能"面板**（实时展示）；
3. **开发期**（`--debug-metrics` 打印）。

### 4.3 基数控制（重要）

**问题**：如果 `frames_sent_total` 按 `event` 打标签，插件可以构造无限多的 event 名 → 指标基数爆炸 → 内存耗尽。

**措施**：
1. 标签值**必须**来自**受限集合**（已知事件类型；未知的归入 `other`）；
2. `plugin_id` 作为标签是安全的（插件数量有限）；
3. **绝不**把用户输入（payload 内容、URL）作为标签；
4. 每个指标有**基数上限**（超限后新标签归入 `overflow` 并计数）。

```rust
pub struct CardinalityGuard {
    max_series: usize,          // 默认 1000
    seen: DashSet<Key>,
    overflow_counter: Counter,
}
```

---

## 5. 审计（面向用户透明度）

（机制见 [10 §7](./10-capability-and-sandbox.md)。此处定义 UI 与保留策略。）

### 5.1 UI 形态

```
设置 → 插件 → Chat → 活动
┌─────────────────────────────────────────────────────────────────┐
│  最近 24 小时                                                    │
│                                                                 │
│  ✅ 14:23:01  device:send  →  laptop          6ms               │
│  ✅ 14:22:58  network:fetch → api.github.com  212ms             │
│  ⛔ 14:22:41  network:fetch → evil.com         ← 未授权，已拦截    │
│  ⚠️ 14:20:03  配额超限：host 调用 612/s > 500/s                  │
│                                                                 │
│  [查看完整历史] [清空记录]                                        │
└─────────────────────────────────────────────────────────────────┘
```

### 5.2 保留与清理

- 内存环形缓冲（最近 10000 条）+ DB（按 `audit_retention_days`）；
- 用户可以清空某个插件的审计；
- **审计记录不含 payload 内容**（[10 §7](./10-capability-and-sandbox.md)）。

### 5.3 审计的导出

审计是**诊断包的一部分**（用户可见、可审查），但要在导出时提示"包含你使用插件的记录"。

---

## 6. 诊断包

### 6.1 内容

```rust
pub struct DiagnosticBundle {
    /// 环境摘要
    env: EnvSummary,               // OS / 版本 / 架构 / 应用版本 / 内核版本
    /// 配置（脱敏）
    config: AppConfig,             // 无敏感字段（密钥相关全部排除）
    /// 设备与链路状态快照
    devices: DevicesSnapshot,
    /// 插件清单与状态（含审计计数，不含审计详情除非用户同意）
    plugins: Vec<PluginSnapshot>,
    /// 指标快照
    metrics: MetricsSnapshot,
    /// 日志（尾部 N 行 + 所有 ERROR 级别）
    logs: LogSlice,
    /// 最近的崩溃报告
    crashes: Vec<CrashReport>,
    /// 环境诊断（端口占用、权限、WebView 版本等）
    probes: Vec<ProbeResult>,
}
```

### 6.2 导出流程（必须用户确认）

```
设置 → 诊断 → 导出诊断包
  ┌──────────────────────────────────────────────────────┐
  │  将包含：                                             │
  │   ☑ 版本与平台信息                                    │
  │   ☑ 配置（已脱敏）                                     │
  │   ☑ 最近的错误日志（最多 14 天）                        │
  │   ☑ 性能指标                                          │
  │   ☑ 插件清单与状态                                     │
  │   ☐ 插件活动记录（含你调用过哪些能力）  ← 默认不勾        │
  │   ☐ 完整调试日志（可能含设备名）        ← 默认不勾        │
  │                                                      │
  │  ★ 不会包含：剪贴板内容、消息内容、文件内容、密钥        │
  │                                                      │
  │  保存到：~/Downloads/synra-diagnostic-<ts>.zip        │
  │                            [取消]  [导出]             │
  └──────────────────────────────────────────────────────┘
```

**要求**：
- 明确列出**包含什么**与**不包含什么**；
- 可选的敏感部分**默认关闭**；
- 导出后显示文件路径 + 大小 + SHA-256（便于用户核对与分享）。

### 6.3 环境诊断（`synra doctor` / 设置页）

```rust
pub struct ProbeResult {
    pub name: String,              // "tcp_port_available"
    pub status: ProbeStatus,       // Ok | Warn | Fail
    pub detail: String,
    pub suggestion: Option<String>,
}

/// 诊断项
pub async fn run_probes(paths: &Paths, cfg: &AppConfig) -> Vec<ProbeResult> {
    vec![
        probe_port_available(cfg.network.listen_port).await,       // 端口被占用？
        probe_discovery_port(cfg.network.discovery_port).await,
        probe_network_interfaces().await,                          // 有无可用网卡/是否全在 lo
        probe_firewall_hint().await,                               // 平台特定的防火墙提示
        probe_data_dir_writable(paths).await,
        probe_disk_space(paths).await,                             // < 500 MB 警告
        probe_clock_sanity().await,                                // 系统时间是否明显偏斜
        probe_webview_version().await,                             // WebView 是否过旧
        probe_keyring_available().await,                           // 密钥链是否可用
        probe_wasm_backend().await,                                // wasm 后端是否可用
        probe_plugin_integrity(&paths.plugins_dir).await,           // 已装插件 artifact 是否完整
    ]
}
```

> ★ **一次"诊断工具自己制造痕迹"的修复（本轮）。**
> `probe_disk_space` 用"写 1 MiB 再读回"来测可写性 —— 因为标准库没有可移植的剩余空间查询。
> 它在**成功**路径上会把探针文件删掉，**失败路径上没有**：
> `std::fs::write` 是**先创建文件再写内容**，所以"写失败"离开现场时已经留下了一个文件。
> 于是这个探针最该诊断的那种情况（磁盘写不进去）同时也是它**在用户数据目录里留垃圾**的情况，
> 每跑一次 `synra doctor` 一个。
>
> ★ **发现它的方式值得记下来：是那个残留文件本身报的案。**
> 仓库里出现了一个未跟踪的 0 字节文件 `apps/synra-cli/.synra-disk-probe` ——
> 来自一次写入被拒的运行。**产物就是 bug 报告。**
> 这也说明一件更一般的事：`.gitignore` 里没有它，所以它一出现就是可见的；
> 如果当初顺手把它加进忽略列表，这个 bug 就永远不会被发现。
>
> ★ 修复与测试的边界说清楚：失败路径的清理**没有可移植的复现方式**。
> 所有能让写失败的可移植手段（目录只读、探针路径是个目录）都在**文件被创建之前**就失败了，
> 所以现场没有东西可观察 —— 而一个不可能因正确原因变红的测试比没有测试更糟。
> 因此测试只覆盖可复现的那一半：**探测成功之后数据目录里什么都不剩**，
> 并用负控制验证过（去掉清理后它红，且打印出的正是残留的那个文件名）。
> 失败路径的那一行靠人工审查 + 上面的成因说明站住，不假装有测试。


---

## 7. 崩溃与 panic 处理

### 7.1 Panic hook

```rust
pub fn install_panic_hook(paths: &Paths, version: &str) {
    let crash_dir = paths.log_dir.join("crash");
    let default_hook = std::panic::take_hook();

    std::panic::set_hook(Box::new(move |info| {
        let report = CrashReport {
            at: SystemTime::now(),
            version: version.to_string(),
            location: info.location().map(|l| format!("{}:{}:{}", l.file(), l.line(), l.column())),
            message: info.payload().downcast_ref::<&str>().map(|s| s.to_string())
                .or_else(|| info.payload().downcast_ref::<String>().cloned()),
            thread: std::thread::current().name().map(String::from),
            backtrace: Backtrace::force_capture().to_string(),
            // ★ 当前 trace（如果是 panic 在某个 span 里）
            trace_id: current_trace_id(),
        };

        // 落盘（尽力而为；失败不能再 panic）
        let _ = write_crash_report(&crash_dir, &report);

        // 调用原 hook（让控制台输出保持）
        default_hook(info);
    }));
}
```

> ★ **一次"崩溃报告自己把自己覆盖掉"的修复（本轮）。**
> `CrashStore::write` 用 `crash-{at_ms}.json` 命名报告文件 —— 文件名只带**毫秒时间戳**。
> 于是同一毫秒内的两次崩溃写到同一个路径，**后一次覆盖前一次**。
>
> ★ **发现它的方式：把一个集成测试跑了很多遍。**
> `crates/synra-observe/tests/panic_hook.rs` 在 15 次里失败 2 次，断言是
> `every panic should have produced its own report: 2 of 3`。
> 这个测试自己的文档注释里写着，它之所以只有一个 `#[test]` 就是为了避免
> "在负载高的机器上偶尔失败的那种竞态" —— 而它自己正是一个。
> 单次运行看不出来，`cargo test --workspace` 那次也没复现；**只有循环跑才现形**。
>
> ★ **后果不是"少了一个文件"。** `startup_verdict` 靠**数报告条数**决定是否进安全模式。
> 条数少了就是：崩溃循环到了阈值却没进安全模式，于是下次启动**又把这个插件拉起来**，
> 而用户需要用来停用它的那个设置界面就在这次启动后面。
> 换句话说，**防止崩溃循环的机制被一个文件名打败了**。
> 一个紧凑的激活循环在一毫秒内 panic 三次完全正常，所以这不是理论情况。
>
> ★ 修复：名字改为 `crash-{at_ms}.json`、`crash-{at_ms}-1.json`、`-2` …
> 并且用 `create_new(true)` 而不是"先判断存在再写" ——
> 后者是两步，中间可以被另一个线程插进来，而 panic hook 跑在**发生 panic 的那个线程**上，
> 所以两个线程同时崩溃是真实情况而不是假想。`create_new` 在名字被占用时**原子**失败，这才是循环正确的依据。
> （负控制时把后缀去掉会**死循环**，因为 `create_new` 永远失败 —— 这恰好证明后缀是承重的。）
>
> ★ 验证：修好之后同一个集成测试跑 **40 次，0 失败**（修之前 15 次里 2 次失败）。
> 另加一个单元测试 `three_crashes_in_the_same_millisecond_are_three_reports`，
> 用**同一个时间戳**写三次（时间戳不同的话对着坏版本也会通过，等于什么也没证明），
> 断言文件数、条数、以及安全模式确实被触发。

| 场景                         | 处理                                                     |
| ---------------------------- | -------------------------------------------------------- |
| 内核任务 panic               | supervisor 捕获（`catch_unwind` + JoinHandle 错误）→ 重启或降级 |
| 插件调用中的宿主 panic         | `catch_unwind` → 销毁该实例 + 返回错误（[12 §7.2](./12-wasm-host.md)） |
| 主线程 panic                 | 进程退出；下次启动时检测到崩溃报告 → 提示用户导出诊断包      |
| 数据库损坏                   | 启动时 `PRAGMA integrity_check`；失败 → 用备份或重建 + 提示  |

### 7.3 启动时的崩溃检测

```
启动时读 logs/crash/ 里最近 5 分钟内的报告：
  · 有 → UI 显示"上次运行异常退出" + [查看详情] [导出诊断包] [忽略]
  · 同一位置连续崩溃 3 次 → 进入安全模式（禁用所有插件启动）
```

**安全模式**很重要：如果某个插件导致崩溃，用户需要一个途径进入应用来禁用它。

---


| ---------------- | ----------------------------------------------- | ------------------------------------------- |
| 日志             | 前端 `console.log`（生产环境用户看不到）           | `tracing` 结构化 JSON 落盘 + 轮转            |
| 日志位置         | 无（Electron dev 是终端）                         | 统一 `logs/`，UI 可查看                       |
| trace            | `traceId` 类型存在但**从不上线**                   | `trace_id` 随信封传播 + 全链路 span            |
| 跨设备关联        | ❌                                              | ✅（v2 对端；v1 标注 trace 断裂）              |
| 指标             | ❌                                              | ✅ `metrics` + 内存 recorder + 基数保护        |
| 插件调用可见性    | ❌                                              | ✅ 指标 + 审计                                |
| 用户可见的插件活动 | ❌                                              | ✅ 审计 UI                                    |
| 诊断包           | ❌                                              | ✅ 带明确的内容清单 + 脱敏                      |
| 环境诊断         | ❌                                              | ✅ `probe_*` 系列 + 可操作建议                 |
| 崩溃报告         | ❌                                              | ✅ panic hook + 安全模式                      |
| 敏感信息过滤      | ❌（`console.log(payload)` 直接打印）             | ✅ `RedactionLayer` + 测试                    |

---

## 9. 实现清单

> ★ 这一节在实现之前写就，长期整节未勾选。逐条核对源码后：**7 条真做了一半或全部，9 条从未开工。**
> 最要紧的两条写在前面：**日志系统完全没有初始化**，**崩溃报告没有任何生产者** ——
> 两者都是"类型齐了、接线没做"，而测试都是绿的。

- [x] `tracing` 初始化（JSON + 轮转 + 非阻塞 writer）—— **★ 本次实现了，但格式是文本而不是 JSON，原因值得记**。
      原状：全仓 0 处 `tracing_subscriber`/`tracing_appender`/`set_global_default`；
      而 `synra-observe/Cargo.toml` **声明了 `tracing` 与 `tracing-subscriber` 两个依赖，
      没有任何源文件使用它们** —— 死依赖。日志层该有的东西都在（级别枚举、verbosity 窗口、
      计数、`RedactingWriter`），**但没有任何一行被写出去**，因为没有东西装 subscriber。
      下游后果是具体的：守护进程的 `synra logs` 读 `data_dir/logs` 下**没有任何代码创建过**的文件，
      所以它永远回答"没有行"，而 CLI 的 `--follow` 与 `--trace` 没有东西可过滤。
      **本次实现**：`logging::RotatingFile`（`logging/file.rs`）+ `logging::init`（`logging/init.rs`）。
      `LogConfig` 收目录/级别/大小上限/保留份数，`init` 返回 `LogGuard`（持有 writer，drop 时 flush）。
      写入链是 `tracing` → `RedactingWriter` → `RotatingFile`，**脱敏发生在落盘前的那一环**，
      所以策略是"出去的路上做一次"，而不是每个调用点记得小心。
      在守护进程里**先于插件宿主构建**安装（装在之后就不记录构建它时的失败）；
      打不开日志只报告不终止 —— 一个写不了诊断文件的主机仍然能搬文件，为它拒绝启动是拿用户的实际工作换记录。
      ★ **格式：文本而不是 JSON**，而且这不是偷懒：
      * `tracing-subscriber` 在这个工作区只启用了 `["fmt", "env-filter", "registry"]`，**没有 `json`**；
      * 更要紧的是守护进程的读取端（`reports.rs:152` 的 `logs_value`）**按空白分隔的 token 找级别**
        （`is_debug_or_trace`，`reports.rs:145` 的注释专门解释过为什么不能用 `contains(" DEBUG ")`）。
        JSON 里级别是 `"level":"DEBUG"`，不是 token，那个过滤器会一行都认不出来。
      也就是说**文档要 JSON、读取端要文本**，两者是冲突的，而读取端是有测试的真代码。
      先把写入端做成文本（可离线、与现有读取端一致），并把冲突记在这里，而不是加一个 feature 去换格式。
      ★ 仍未做的一半：**"非阻塞 writer"没有** —— 写入是同步的，没有 `tracing-appender` 的
      `non_blocking`。轮转是**按大小**（`DEFAULT_MAX_BYTES = 8 MiB`，`DEFAULT_KEEP = 7`），不是按天。
      测试在 `crates/synra-observe/tests/logging.rs`（独立二进制，因为 `try_init` 是进程级的），
      且**只有一个 `init` 测试**：两次安装会互相冲突，两个测试各装一次就是竞态。
      它断言的是"行到了磁盘上"以及"**`hunter2` 没有到磁盘上**"—— 后者是脱敏唯一能失败的地方。
      负控制：把 `redact_line` 改成原样返回 → 测试红（"the secret reached the disk"）。
      另加一条守护进程侧的路由守卫
      `the_log_reader_and_the_log_writer_agree_on_the_directory`：
      写入端取 `Paths::log_dir()`、读取端手写 `data_dir.join("logs")`，
      今天两者相等只是两行独立代码的巧合；`Paths` 一旦改变存放位置，
      `synra logs` 会永远返回空列表**而不报任何错** —— 这条测试就是钉住这个巧合。
      ★ **读取端现在是共享的**：`synra_observe::logging` 暴露
      `tail` / `newest_file` / `is_debug_or_trace` / `LogTail`，
      守护进程的 `runtime/reports.rs` 不再自己实现这三件事。
      原因是有**两个**读者：守护进程答 `synra logs`，shell 往诊断包里放日志摘录。
      两套实现会漂移，而漂移是不可见的 —— 两个读者对"哪些行是 debug"意见不一致，
      要等有人对比两份输出才会被发现。
       ★ **本次补上了 `--verbose` 缺的那一半（它此前是空头承诺）。**
       原状：`with_max_level` 在 subscriber 构建时就把级别钉死，而 `tracing` 每进程只允许一个 subscriber ——
       于是**这个进程在它有生之年写不出一行 debug**，`synra logs --verbose` 从一个永远不含 DEBUG 的文件里
       把 DEBUG 过滤掉，结果与不加 `--verbose` 完全一样。**没有任何东西失败，标志只是什么也没做。**
       修法分两层：
       * `logging/init.rs`：级别判定从常量改成**每次事件问一次**（`DynamicLevel` 这个 `Layer`），
         它读的是 `Verbosity::effective()` —— 于是 `Verbosity` 这个**早就写好、测试齐全、零生产调用者**
         的类型终于有了调用者。窗口（`VERBOSE_WINDOW`，30 分钟）比"提升"本身同样重要：
         一个不会过期的提升就是一台没人看的机器上一直开着的 debug 日志，那是磁盘被填满的经典方式。
       * `runtime/reports.rs` 的 `logs_value`：请求里带 `verbose` 就把级别提到 `Debug`，
         带 `trace` 就提到 `Trace`（跟一个 id 是更窄也更刻意的请求，值得更多细节），
         并在回答里回 `raised` 与 `level` —— **因为第一次读确实不会有新的 debug 行**，
         不说明的话用户会认为这个标志没用（而那正是它此前的样子）。
       * `Runtime` 新增 `verbosity: Option<Arc<Verbosity>>`：`None` 表示"日志没跑起来"
         （文件打不开的守护进程照样回答这个命令），此时它**不能声称自己提升过**。
       测试三层：`logging.rs` 的集成测试在同一次运行里**先写一行 raise 之前的 debug、再写一行之后的**，
       断言前者不在磁盘上、后者在（只断言后者的话，一个无条件写 debug 的实现也会通过，而那正是要防的）；
       守护进程侧断言的是 **logger 自己的状态**，因为那才是决定 debug 行写不写的东西；
       加一条"没有 logger 时不谎报"的用例。
       负控制：把 gate 改成恒 `true` → 集成测试红（"a debug line was kept before anything asked for one"）；
       把 `raise_for_a_while` 改成空操作 → 守护进程测试红。
       ★ 顺带记一个自己踩的坑：新加的两个测试最初用了与**已有测试相同的临时目录名**
       （`logs-verbose` / `logs-trace`），而 `TempDir::new` 会先删掉那个目录 ——
       于是它们互相删对方的日志文件，表现为两个**原本通过**的测试变红。
       是测试套件抓住的，不是我看出来的：同一进程里并行跑的测试共享 `temp_dir`，名字必须唯一。
      ★ **shell 现在也有自己的日志**：`AppState::start_logging`（独立于 `AppState::new`，
      因为 `tracing` 每进程只允许一个 subscriber，而 `AppState::new` 也被测试调用），
      `LogGuard` 由 `AppState` 持有 —— 持有而不是泄漏，因为 drop 才会 flush。
- [x] 字段名命中 + 长度截断 + URL 去 query + 测试 —— **A/C**：
      `RedactionLayer` 不存在（**不是 `Layer<S>` 实现**）；真实机制是 `Redactor`(`redact.rs:154`)
      + `RedactingWriter`(`logging/writer.rs:16`)，一个 `Write` 包装器。
      三项性质都是真的且有测试：字段名黑名单(`:187`)、长度截断(`:222`)、URL 去 query(`:261`)
      + proptest(`:542`/`:593`)。
      ★ 但 `logging/tests.rs:18` 的 `a_payload_never_reaches_the_output` 注释说"the test the design document
      asks for by name"，实际是手写 `format!("event=… payload=…")` 直接喂 `redact_line` ——
      **没有 tracing 宏、没有 subscriber**（因为不存在），所以文档要求的 **JSON 格式完全没有测试**。
- [ ] `PluginAttributionLayer`（插件日志加 `plugin_id`）—— **未做**，0 命中。
- [x] `TraceContext` + `trace_id` 随信封传播 —— **A/一半**。
      `TraceContext`(`trace/identity.rs:78`) + `Sampler` 有；`from_envelope`/`continue_from` 存在且有测试。
      ★ **v1 的 `trace_break` 标注 0 命中**，断链标注不存在。
- [x] `Sampler`（错误/慢请求总是采样 + 用户强制开启）—— **A**：
      `trace/sampler.rs:14` + `should_record(context, failed, elapsed_ms)`(`:160`)，
      含 always/nothing/everything 构造 + 单测与 proptest。
- [x] `RingRecorder` + `CardinalityGuard` —— **A/C**：`metrics/recorder.rs:108` + `:18`，含测试与 proptest。
      ★ 文档选定的 **`metrics` facade 从未使用**：`Cargo.toml` 声明了依赖，0 处
      `metrics::counter!`/`gauge!`/`histogram!`。**自研 recorder 有，facade 没有。**
- [ ] §4.1 的指标清单全部接入 —— **零接入**：没有任何生产代码记录指标；
      ★ **而第 141 轮核实：「零接入」这个词不成立。** `host_calls.rs` 里有 4 处生产记录：
      `gate_checks_total{result=allowed|denied|rate_limited}`、`gate_denied_total{capability}`、
      `gate_rate_limited_total{plugin}`（`self.count(...)`，`host_calls.rs:470-479`），
      管道从 `PluginSet::with_metrics` → `session` → `HostCallSession::with_metrics` 一路是通的，
      而 `host_calls.rs:337` 的注释解释了为什么从门开始：*「它是唯一已经知道答案的地方 ——
      是权限集说了不，还是限流器说了不」*。
      ★ 所以本行仍不勾，但缺的不是「一个都没有」，而是**清单上的其余项**（网络、发现、意图、传输…）。
      一个说「零接入」的行会让人从零开始做一遍已经做完的那部分。
      ★ **而本轮差点就这么做了**：我先查了 §4.1 清单、再查 recorder API、再查注入点，
      第三次之后才问最直接的那个问题 ——「全仓有没有人在记指标」。**那个问题应该第一个问。**
      ★ **第 142 轮接上了第一个直方图**：`host_call_ms{import=…}`，在 `record()` 里 ——
      `elapsed_ms` 本来就在它手上（它进了 audit 条目），只是**没有任何生产代码用过** `histogram_observe`，
      全仓只有 recorder 自己的测试调它。没有调用者的直方图是一个**形状**而不是一次测量。
      ★ 而它被记在 `record()` 的**最前面**：一次被拒绝的调用也花了它花的时间，
      只由成功调用构成的分布会用一个错误的问题的答案回答「这个插件有多慢」。
      ★ 测试做过**变异验证**：把那一行注释掉 → `FAILED` 并报出 `the duration was not recorded`；恢复 → 通过。
      这一条值得记，因为断言形式是 `is_some()`（`MetricsSnapshot` 没有 `keys()`，所以看不到实际 series），
      弱断言更需要证明它能失败。
      ★ 于是本行的状态是：门的三项 + 一个直方图已接入；缺的仍是清单上的**其余项**
      （网络、发现、意图、传输…）。
      ★ **第 143 轮接了意图组的两项**（`daemon` 侧的 `intent_dispatch`，那里 `self.metrics` 已在手）：
      `intents_total{outcome=ok|refused}` 与 `intent_no_handler_total`。
      ★ **而第一版测试失败了，失败本身有价值**：它假设「空插件集 → 路由失败」，但一个空插件集
      **也会成功地把意图建立起来**。所以「路由成功」与「有人回答」是两件事 ——
      这正是清单里 `intent_no_handler_total` 存在的理由，而 `intents_total{outcome=ok}` 会把两者算成一件。
      一个没人能处理的意图和一个被处理了的意图，对用户是两种处境。
      ★ 计时那两项（`intent_duration_ms` / `intent_match_duration_ms`）仍未接：它们要墙钟，
      而这个方法的 `now_ms` 是**逻辑**时间。
      ★ 本行的状态：门三项 + 一个直方图 + 意图两项；缺的仍是网络、发现、传输那几组。
      ★ **第 156 轮接了发现组**：`scans_total{result=ran|unavailable}`、`candidates_found`、
      `scan_duration_ms` —— 落点是 daemon 的 `Runtime::scan`，那里 `self.metrics` 已在手。
      而在此之前，这一组**一次都没记过**（全仓 `scans_total` 零匹配）。
      ★ **`unavailable` 那一条是刻意的**：一台没有发现套接字的机器是一个用户会问的状态，
      而一个只在 daemon 自己的日志里出现的 `return`，`status` 是看不见的。
      ★ 而「扫了但没有候选」与「扫到了三个」是用户读 `devices` 时的两种处境，只有一种是问题 ——
      一个从不被记录的计数会让两者变成同一件事。
      ★ 测试断言的是「**记了**」而不是「记了哪一个」：这个 fixture 有没有发现套接字是它自己的事，
      而两条路径各记一条。
      ★ 本行的状态：门三项 + 直方图两项（host 调用、意图）+ 传输两项 + **发现三项**；
      缺的是网络那一组（`links_open`、`frames_*`、`bytes_*`、`reconnect_attempts_total`…）。
      ★ **第 157 轮接了网络组的第一项**：`frames_sent_total{event}`，落点是 `Host::send`。
      而 `Host` 因此多了一个**可选的记录器**（`with_metrics`）—— 与插件宿主同一个理由：
      *一个不记账的 host 是 host，不是坏的*，而一次**因为指标写不进去**而失败的发送，
      会是两端都看不见、也处理不了的理由拒绝掉的消息。
      ★ **记在发送之后，而不是之前** —— 这不是细节：被链路拒绝的帧从未离开，
      把它算进去会让这台设备说它讲了比实际更多的话。
      ★ **而这一项之所以值得接**：`Host` 原本只有一个手工的 `sent: u64`（注释写着「for a report」），
      它回答「说了多少」却说不出「说了什么」—— 而后者才是排查时要问的。
      ★ 测试直接加在**既有的端到端测试**（`an_envelope_travels_between_two_connected_hosts`）里，
      它本来就走完了 dial → 握手 → session → envelope，并且已经断言了 `sent() == 1`。
      ★ 本行的状态：门三项 + 直方图两项 + 传输两项 + 发现三项 + **网络一项**；
      缺的是 `links_open`、`bytes_*`、`reconnect_attempts_total` 等。
      ★ **第 157-158 轮接了网络组的两项**：`frames_sent_total{event}`（`Host::send`，记在**发送之后** ——
      被链路拒绝的帧从未离开）与 `links_opened_total`（`Host::adopt` 里 session 打开处 ——
      拨入与接入**共同**经过那里，所以一个计数覆盖两侧）。
      ★ 而 `Host` 因此多了一个**可选的记录器**（`with_metrics`），理由与插件宿主相同：
      *一个不记账的 host 是 host，不是坏的*。
      ★ **这两项曾经一度消失，而教训比指标本身重要**：一个 `git filter-branch` 进程在仓库里活动
      （它的工作目录 `.git-rewrite/` 一直在变），在我提交**之前**重置了工作树 ——
      于是 `git add -A` 提交的**只有那个临时目录**，而两次提交的标题（`feat: a sent frame is counted…`）
      看着完全正常。**一个提交看起来成功、内容却是别的东西**，与这个仓库一直在清理的那些失败同型。
      ★ 现在的做法：只用 `git add <我改的文件>`，改完**立刻**提交，并在提交后 grep 一次源码确认改动真的在里面。
      ★ 本行的状态：门三项 + 直方图两项 + 传输两项 + 发现三项 + 网络两项；
      ★ **第 159 轮接了接收侧**：`frames_received_total{event}`，在 `listener` 处理 `Decision::Delivered`
      的那一支 —— 那是唯一知道「是哪一个事件」的地方。它与发送侧分开，因为
      *说得比听到的多* 与 *听到的比说得多* 是两种不同的处境。
      ★ **而这一项有一个诚实的缺口**：调用它的那一支只有 `listener::handle_within` 会产生，
      而 `host/tests.rs` 的端到端测试走的是 `Host::accept` + `link.receive`，**完全绕过它** ——
      第一版断言就写在那里，它失败了，而失败说明的正是这件事。所以现在的测试测的是
      `Host::count_frame_received` **本身**，而「它在 listener 里被调用」目前只有编译期证据。
      ★ **工程上的一条硬教训，比指标重要**：这个仓库里有一个进程会**周期性重置未提交的工作树**
      （它改写历史，`.git-rewrite/` 就是它的工作目录）。所以「改完再测试再提交」的老做法会丢改动：
      我曾两次写出内容完全正常的提交标题，而 `git show --stat` 显示改动**根本不在里面**。
      现在的做法是：**改动、编译、测试、提交放在同一条命令里**，并在提交后立刻 `git show --stat` 核对。
      ★ 本行的状态：门三项 + 直方图两项 + 传输两项 + 发现三项 + 网络三项（发送、链路、接收）。
      ★ **第 160 轮把三个计数收成同一个写法**（`Host::count_frame_sent` / `count_frame_received` /
      `count_link_opened`）。这不是整洁问题：上一轮我刚在方法文档里写下
      *「`metrics` 是 `pub(super)`，而『一帧怎么记』该有一个答案，而不是每个看得见它的模块各有一份」* ——
      而当时**发送**那一处仍然是调用点自己去够字段的。**注释说一套、代码做另一套**，
      是这份文档一直在清理的东西，只是这次的作者是我。
      ★ 而 `links_closed_total{reason}` 这一轮**没做**，因为查下来 host 模块里
      **没有关闭链路的函数** —— 一个没有自然落点的指标不该被按在某个顺手的地方。
      ★ 本行的状态不变：门三项 + 直方图两项 + 传输两项 + 发现三项 + 网络三项。
      缺的是 `bytes_*`、`reconnect_attempts_total`、`links_close_total{reason}` 等。
      ★ **第 144 轮接了传输组的两项**：`transfer_bytes` 与 `transfer_duration_ms{kind=attachment}`，
      在 `send_file` 的成功返回处（`self.metrics` 已在手，与意图那两项同一个落点）。
      ★ **而 `transfers_total{outcome}` 故意没接，理由值得写下来**：`send_file` 有多条 early return
      （路径不存在、地址解析不了、握手失败），只在成功路径记的计数器会统计「成功了的那些」，
      而对用户正在问的那些一言不发 —— 与意图那条注释里说的同一个错。它需要**每个出口经过同一处**，
      而这是 `impl Kernel for Runtime` 的 trait 方法，改不了名、包不了壳。
      ★ 本轮**试过**结构手术（把函数体塞进一个立即调用的闭包，好让所有出口汇聚），
      脚本定位闭合行时切错了位置、文件变成未闭合，于是 `git checkout` 回退、改成两处小改动。
      **一次失败的改法换一个更小的改法，比硬推下去便宜。**
      ★ **而第 145 轮把那条测试写出来了**：一次**真实的**文件传输 —— 一个监听端在一个线程里 accept 并
      handle，主线程用 `send_file` 拨它（`runtime/tests.rs` 的 `a_file_that_reaches_the_other_side_is_measured`）。
      两个断言：`transfer_bytes{kind=attachment}` 与 `transfer_duration_ms{kind=attachment}` 都出现了。
      ★ **而它是网络测试，所以跑了三次**（第 95 轮的教训：一条只在全量跑里失败的测试，看起来像偶发）：
      三次都过，全量 196 条也过。
      ★ **写它的过程中撞到两件真事，都写进了代码注释**：
      ① **两端的 flags 必须一致** —— 监听端用 `no_timeout`，而 `Runtime` 自己的 host 不是，
      于是一端设了截止、另一端没有，而它们对「对话什么时候结束」的看法就不一致
      （`transport/tests.rs` 早就用一段注释记过这件事，我撞了同一堵墙）。
      ② **`bind(host, 0)` 报回来的地址不总是能拨的** —— 端口 0 让操作系统挑，而它可能报未指定地址；
      拨那个在 Windows 上不是「拨本机」。`transport/tests.rs` 为此有一个 `dialable` helper，
      这条测试做了同样的事而不依赖另一个测试模块的私有函数。
      ★ 本行的状态：门三项 + 直方图两项（host 调用、意图）+ 传输两项；缺的是网络、发现那几组。
      `RingRecorder` 只出现在 `synra-observe` 自己的测试里。§4.1 的指标名中只有
      `frames_sent_total`/`links_open` 出现过，且只是 `metrics/tests.rs` 的测试夹具。
- [ ] 设置页的"性能"面板（实时指标）—— **缺失**：`synra-rust-app/src/pages` 只有
      Devices/Home/Plugins/Settings。
- [ ] 审计 UI（时间线 + 拒绝高亮 + 清空）—— **UI 仍然没有；但"后端无生产者"这句只对一半了**。
      ★ **仓库里有两个审计环，只有一个有生产者。**
      * `PluginSet` 的环（`crates/synra-core/src/dispatch/plugins.rs:312`）**有**：每个实例被装上 session，
        调用记录了能力与结果，`drain_audit` 把 session 记下的排进环，`audit_for`/`refusals_for` 读回来。
        命令 `plugins_audit` 已经在 shell 里注册，读的就是这个环 —— 所以审计 UI 现在**有真数据可显示**。
      * `Kernel` 的环（`kernel.rs:676`）**没有**：`record_action` 的唯一调用者是 `kernel_tests.rs:380`，
        而 `synra_core::Kernel` 本身没有任何宿主构造它。这一半仍然是"只有测试往里填数的空壳"。
      `AuditRing` + `PluginActivity::of`(`audit.rs:384`) 能出摘要，app 里仍然没有任何 audit 组件。
- [x] `DiagnosticBundle` + 导出流程（内容清单确认）—— **A，而且本次让"debug 日志"这一节真的有内容了**。
      `diagnostics/records.rs:94` + `BundlePlan::build`(`plan.rs:190`) +
      `DiagExportOptions{include_audit, include_debug_logs, log_lines}` 三项默认 `false`
      （`synra-ipc/src/dto/config.rs:74-82`）+ CLI `doctor --bundle`(`args.rs:264`)。
      ★ 原状：`diag_export` 里 `sources.log_lines = Some(Vec::new())` ——
      **一节被"包含"但内容是空的**。这正是半成品功能的样子：同意页与文件就一节没有内容的章节达成了一致，
      而分享了这个包的用户以为自己已经把日志发出去了。
      守它的测试 `asking_for_the_debug_log_carries_the_section_too` 只断言这一节**被列出**，
      所以它从始至终都是绿的。
      **本次**：shell 自己也有了日志（`AppState::start_logging`），
      `diag_export` 用共享读取器（`synra-observe::logging::tail`）填 `log_lines`，
      并且是 `verbose = true` —— 用户要的就是 debug 日志，安静过滤会恰好把他要的藏起来。
      新测试 `the_debug_log_section_carries_the_lines_the_shell_wrote`
      真的写一份日志再要求它出现在 bundle **文件**里；负控制（改回空列表）确认会红，
      失败信息里就是 `"logs":{"tail":[],...}` 那个空章节。
      另加 `the_debug_log_section_is_empty_rather_than_absent_when_there_is_no_log`，
      区分"这一节是空的"与"这一节不存在"—— 与审计那节同一个判断。
      ★ 审计那一节仍然是空列表，因为**仍然没有审计写入方** —— 两者情况不同，所以答案也不同。
- [x] `run_probes` 系列（§6.3）—— **A，全部实现且已接线**。
      ★ **更正一处我自己写错的注记。** 上一轮我在这里写了"§6.3 列的 11 个探针只实现了 6 个左右，
      `probe_webview_version`/`probe_wasm_backend`/`probe_keyring_available` 未见"。
      **那是错的**，而且是逐条核对就能发现的那种错：三个探针都在
      `crates/synra-observe/src/doctor/host.rs`（`:55`/`:99`/`:85`），
      而 `run_probes`(`doctor/mod.rs:54`) **12 次 `push` 覆盖了 §6.3 的每一个探针**
      外加一个 `probe_udp_port_available`：
      端口可用（TCP + UDP）/ 网卡 / 防火墙提示 / 数据目录可写 / 磁盘空间 / 时钟偏差 /
      WebView 版本 / 密钥链可用 / wasm 后端可用 / 插件目录。
      我是从一份子代理报告里转述的，没有自己打开 `run_probes` 看 ——
      这正是我在别人代码里反复找的那类问题（**断言的是说法，不是那个说法所指的东西**），
      写在文档里同样成立。教训记在这里而不是悄悄改掉。
      剩下的确实缺的只有一半：**设置页的诊断展示**。app 的 `src/pages` 只有
      Devices/Home/Plugins/Settings 四个，没有任何 doctor/探针的调用方。
      探针本身不是没人用的 —— `synra doctor`（CLI，已注册）就是它们的调用方
      （`DoctorReport::run` → `run_probes`），所以这半是"少一个界面"，不是"少一套实现"。
- [x] `install_panic_hook` + `CrashReport` 落盘 —— **★ 本次补上了缺的那一半**。
      `CrashReport`(`crash.rs:54`)、`CrashStore::write`(`:254`) 原本就是真的，
      但 **`install_panic_hook` 不存在，全仓 0 处 `std::panic::set_hook`/`take_hook`** ——
      于是整条链路是**通的、但干的**：报告类型、存储、按位置计数、阈值、以及查询它的装载路径全都在，
      而**永远没有人写过一份报告**。后果具体：安全模式触发不了（它数的那份文件不存在），
      一个在激活时崩溃的插件每次都只会崩，用户拿不到任何可读的记录。
      **本次实现**：`synra_observe::crash::install_panic_hook(store, version)`，
      在 `serve.rs` 里**先于任何会崩的东西**安装（在插件宿主构建之前 —— 装在之后就不记录构建它时的失败），
      version 用 `env!("CARGO_PKG_VERSION")` 即守护进程自己的版本而不是 observe 层的。
      ★ 它**串联**而不是替换上一个 hook：替换掉就会让装了这个 hook 的构建在终端里**静默 panic**，
      那比它要记录的故障更糟。先写报告、再跑原 hook，所以写失败也还有普通消息。
      ★ 测试放在 `crates/synra-observe/tests/panic_hook.rs`（独立二进制），
      因为 `set_hook` 是**进程级**的 —— 放进 `src/` 的测试会改掉同一二进制里所有其它测试报告失败的方式。
      ★ 而且那里**只有一个 `#[test]`**：两次安装会互相替换，两个测试各装一次就是一个
      "大多数运行能过、机器一忙就失败"的竞态 —— 正是这个仓库一直在找并点名的那类东西。
      测试跑 5 次确认稳定；负控制（把写报告那行去掉）确认会红（"the hook wrote 0 reports"）。
      至此崩溃管线是端到端通的：panic → hook → 落盘 → 按位置计数 → `verdict` → 安全模式 → 不装载插件。
- [x] 启动时崩溃检测 + 安全模式 —— **A，且本次把"它要门控的行为"接上了**。
      `StartupVerdict::SafeMode` + `startup_verdict`(`crash.rs:190`) + `SAFE_MODE_THRESHOLD`
      + 真实测试(`:474`/`:495`/`:525`)，并接进 `Kernel::startup_verdict`(`kernel.rs:358`)。
      ★ 原状：`StartupVerdict::allows_plugins()`(`:160`) **只在测试里被调用**
      （`crash.rs:443`/`:469`/`:490`、`kernel_tests.rs:365`）——
      全仓没有任何插件加载路径查询它。后果具体而不抽象：一个在激活时崩溃的插件，
      第三次之后仍会被第四次启动，而用户要关掉它所需要的设置页**始终打不开** ——
      而这正是 `SafeMode` 自己的文档注释里写的"必须存在的那个出口"。
      **本次接上**：`apps/synra-daemon/src/runtime/plugins.rs` 的 `load_installed` 现在收
      `&StartupVerdict`（不是 `bool`，这样拒绝时能说出**是哪个位置**在反复崩），
      `serve.rs` 在装载前用 `CrashStore::verdict(now)` 得到判定再传进去；
      `LoadedSummary` 新增 `safe_mode: Option<String>`，**独立于 `refused`** ——
      两者是不同的陈述：`refused` 说的是"这些插件失败了"，
      `safe_mode` 说的是"守护进程在看过任何一个插件之前就决定了不启动它们"，
      把后者塞进前者等于把责任推给一个**从未被检查过**的插件。
      状态报告（`reports.rs`）也跟着暴露 `safeMode`，否则客户端看到的是
      `loaded: false` + 空的 `notLoaded`，读起来就是"这台机器没装插件"—— 与事实相反，且会把人引去重装。
      测试 `safe_mode_loads_nothing_and_says_why` 走**真实路径**（写三份崩溃报告 → 问 store 要判定 →
      装载），而不是手工构造 `SafeMode` 枚举 —— 后者只能证明"我编的那个值返回 false"。
      负控制：把判定那行改成 `if false`，测试立刻红（"safe mode loaded something"）。
      至此这条链路是通的**且有水**：`install_panic_hook` 本次已实现（见上一条），
      panic → 落盘 → 计数 → `verdict` → 不装载插件，整条都有测试。
- [x] DB `integrity_check` + 损坏恢复路径 —— **检查有，恢复没有**：
      `SynraDb::health`(`sqlite/mod.rs:174`) + `interpret_integrity` + 测试(`db/tests.rs:249-279`)；
      ★ `health.rs:42` 只提示"restore from a backup"，**无任何恢复代码**。
- [x] `synra logs --trace <id>` —— **★ 本次把丢掉的 id 接了回来**。
      原状：`channel.rs:108` 把 `--trace <id>` 翻译成 `verbose: true`，**id 被丢掉**；
      `Request::Logs` 里根本没有 trace 字段；daemon 的 `logs_value` 只按 level 过滤。
      而守这件事的测试 `a_trace_filter_asks_for_the_verbose_log` 只断言 `verbose` 翻转，
      注释却写着 "the filter is applied to what comes back" —— **没有任何代码应用它**。
      后果：用户想跟一条用户动作的日志，拿到的是整份日志，而且**没有任何提示说过滤被忽略了**；
      一份安静的日志的快照和一份什么都没发生的实时日志，在屏幕上是一样的。
      **本次**：`Request::Logs` 增加 `trace: Option<String>`，
      `channel.rs` 透传而不是折叠，`logs_value` 增加一个**子串**过滤
      （级别用 token 匹配是因为它是固定词表且有固定位置；trace id 是不透明字符串，
      出现在 logger 决定渲染的地方，没有位置可锚定，token 匹配会一行都找不到）。
      响应里回显 `trace`，这样调用方能区分"这条 trace 没有日志"和"过滤在传递路上丢了"——
      正是丢掉 id 造成的那种混淆。
      测试：`a_trace_filter_carries_the_id_and_asks_for_the_verbose_log`（CLI 侧，
      断言 id 真的在请求里）与 `a_trace_filter_keeps_only_the_lines_that_mention_it`（daemon 侧，
      四条日志里挑出两条、另一条 trace 不在里面、不存在的 id 得到空列表而不是全部）。
      负控制：把过滤那行去掉 → 测试红，并且失败信息直接把四条日志打出来。
      ★ 仍未做的：**日志行里现在还没有 trace id** —— 过滤是对的，但要有东西可过滤，
      得先把 `TraceContext` 作为 span 字段挂上去（`TraceContext` 与 `RedactingWriter` 都在，缺的是插桩）。
- [x] `synra logs --follow` —— **★ 本次改成明确拒绝，而不是假装**。
      原状：`args.rs:237` 定义了这个标志，`args.rs:573`/`:586` 断言它**能解析**，
      而**全仓从未读取它**。命令照常发一个普通 `logs` 请求、打印尾部、退出。
      一个要求"跟随"的用户会一直以为自己看的是实时日志 —— 而这是**看不见的失败**，
      因为安静日志的快照和一个什么都没发生的实时日志，输出完全一样。
      **本次**：`dispatch` 里在连接之前拒绝，并说明原因是**功能未实现**而不是坏掉：
      `Request` 是"一个连接一个请求"，要流式推送需要另一套协议 —— 这**不是接线缺口而是缺功能**，
      而把这件事说清楚才是诚实的答案。替代方案（在这里循环、每秒重问一次）会是一个
      **静默轮询**的 `--follow`，那是与这个标志所承诺的不同的东西。
      测试 `following_the_log_is_refused_rather_than_answered_with_a_snapshot`
      同时断言消息里有 `--follow` 与 `not implemented` —— 只断言 `Failed` 会把用户引去找一个没在跑的内核。
- [x] `synra doctor`（CLI）—— **A**：`Command::Doctor`(`args.rs:80`) +
      `DoctorReport::run`(`doctor.rs:76`) 调 `run_probes`(`:83`) + `--failures-only`/`--bundle`。

## 第 165 轮：一次重传被记下来了

★ `transfer_retries_total{kind}`，来源是 `send_a_file` 手里已有的 `resends` —— 那个数一直有
（`Resend` 事件产生的），只是从来没有被记进指标；它只出现在给用户的报告里。

★ **只在真的发生重传时记**：一个永远加零的计数器是一条没人需要的序列。

★ **而它的证据是诚实的缺口**：没有测试覆盖这一行。重传发生在一次报告丢失时，
而**制造那件事**需要一条会丢包的链路 —— 这里的测试脚手架做不到。
所以这一行目前的证据是「编译通过 + 857 条既有测试」，而不是「它被验证过」。
