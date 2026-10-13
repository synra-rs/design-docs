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

### 7.2 崩溃恢复

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

- [ ] `tracing` 初始化（JSON + 轮转 + 非阻塞 writer）
- [ ] `RedactionLayer`（字段名命中 + 长度截断 + URL 去 query）+ 测试
- [ ] `PluginAttributionLayer`（插件日志加 `plugin_id`）
- [ ] `TraceContext` + `trace_id` 随信封传播（v2）+ v1 的 `trace_break` 标注
- [ ] `Sampler`（错误/慢请求总是采样 + 用户强制开启）
- [ ] `metrics` facade + `RingRecorder` + `CardinalityGuard`
- [ ] §4.1 的指标清单全部接入
- [ ] 设置页的"性能"面板（实时指标）
- [ ] 审计 UI（时间线 + 拒绝高亮 + 清空）
- [ ] `DiagnosticBundle` + 导出流程（内容清单确认）
- [ ] `run_probes` 系列（§6.3）+ 设置页诊断展示
- [ ] `install_panic_hook` + `CrashReport` 落盘
- [ ] 启动时崩溃检测 + 安全模式
- [ ] DB `integrity_check` + 损坏恢复路径
- [ ] `synra logs --follow --trace <id>`（CLI）
- [ ] `synra doctor`（CLI）
