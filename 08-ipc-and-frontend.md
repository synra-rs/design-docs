# 08 · IPC 契约与前端集成

> 📍 **归属：`synra-app` 仓库**（[README §2 分流表](./README.md)）。本文是**唯一一篇整篇进 app 仓库的文档**。
>
> 本文定义 Tauri 侧（第 2 环）与前端（第 3 环）之间的契约，以及前端如何从"业务宿主"退化为"渲染层"。
> 对应 [06 三环模型](./05-system-architecture.md) 的第 2↔3 环边界。
>
> ⚠️ **迁移到 app 仓库时**：文中所有 `./NN-*.md` 的相对链接要改成跨仓库形式——
> core 的文档在 `../../synra-core/design-docs/`（[README §2.1](./README.md) 的路径基准表）。

---

## 0. 与 core 的边界（先说清楚）

| 属于 core（`synra-core`）          | 属于本文（`synra-app`）                     |
| --------------------------------------- | ------------------------------------------------ |
| `KernelEvent` 枚举（事件的事实来源）      | 事件 → Tauri `emit` 的桥、前端的 `switch` 分发     |
| `SynraError` + 稳定错误码（错误的事实来源） | `IpcError` + `SuggestedAction`（面向 UI 的映射）   |
| `Kernel` 的公开 API（`bootstrap`/`start`/`subscribe`） | `#[tauri::command]` 的 ~40 个薄包装       |
| `ts-rs` 生成的类型                         | 消费生成物、`tauri-specta` 绑定                   |
| UI 插件的**会话令牌签发/校验**（[§12.6](./10-capability-and-sandbox.md)） | 具体 WebView 创建、CSP、`on_navigation`、`PluginUiOp` 的方法白名单**执行** |
| `trait UiHost`（UI 插件的抽象）             | `UiHost` 的 Tauri 实现                            |

> **判据**：如果一个功能在"没有 UI"的情况下也需要工作，它属于 core。
> 所以**令牌的签发**在 core（它是安全边界），**令牌的注入方式**（initialization script）在 app。

---

## 1. 原则

| # | 原则                                                                   |
| - | ---------------------------------------------------------------------- |
| 2 | **command 是请求/响应，event 是状态变更**；前端不轮询                      |
| 3 | **首屏用 `snapshot()`，之后只消费增量事件**；`Lagged` 时重新 snapshot      |
| 4 | **command 参数与返回值必须是 `SynraError` 可表达的错误**（不允许 panic 穿透） |
| 5 | **UI 插件的 IPC 面与宿主 UI 的 IPC 面是隔离的**（不同令牌、不同命令白名单）  |
| 6 | 前端**不持有**连接、插件实例、协议细节                                     |

---

## 2. Tauri Command 清单

### 2.1 命名规范

```
{domain}_{verb}          snake_case
例：identity_get、devices_scan_start、plugins_install、intent_dispatch
```

**分组**（对应内核聚合根）：

| 前缀        | 对应聚合根        | 说明                          |
| ----------- | ----------------- | ----------------------------- |
| `identity_` | `Identity`        | 本机身份                      |
| `devices_`  | `Discovery`+`Trust`+`Transport` | 设备视图（派生）    |
| `pairing_`  | `Trust`           | 配对                          |
| `intent_`   | `Intent`          | 意图路由                      |
| `transfer_` | `TransferSessions`| 文件传输                      |
| `plugins_`  | `Plugins`         | 插件                          |
| `config_`   | `Config`          | 配置                          |
| `diag_`     | `synra-observe`   | 诊断                          |

### 2.2 完整 Command 表

```rust
// src-tauri/src/commands/mod.rs

// ─────────── identity ───────────
#[tauri::command] async fn identity_get(state: State<'_, AppState>) -> Result<IdentitySnapshot, IpcError>;
#[tauri::command] async fn identity_set_display_name(state: State<'_, AppState>, name: String) -> Result<(), IpcError>;

// ─────────── devices ───────────
/// 全量快照：设备卡片列表 + 本机身份 + 扫描状态
#[tauri::command] async fn devices_snapshot(state: State<'_, AppState>) -> Result<DevicesSnapshot, IpcError>;
#[tauri::command] async fn devices_scan_start(state: State<'_, AppState>, opts: ScanOptions) -> Result<ScanId, IpcError>;
#[tauri::command] async fn devices_scan_cancel(state: State<'_, AppState>, scan_id: ScanId) -> Result<(), IpcError>;
#[tauri::command] async fn devices_connect(state: State<'_, AppState>, device_id: DeviceId) -> Result<(), IpcError>;
#[tauri::command] async fn devices_disconnect(state: State<'_, AppState>, device_id: DeviceId) -> Result<(), IpcError>;
#[tauri::command] async fn devices_forget(state: State<'_, AppState>, device_id: DeviceId) -> Result<(), IpcError>;

// ─────────── pairing ───────────
#[tauri::command] async fn pairing_request(state: State<'_, AppState>, device_id: DeviceId) -> Result<RequestId, IpcError>;
#[tauri::command] async fn pairing_respond(state: State<'_, AppState>, request_id: RequestId, accept: bool, reason: Option<String>) -> Result<(), IpcError>;
#[tauri::command] async fn pairing_cancel(state: State<'_, AppState>, request_id: RequestId) -> Result<(), IpcError>;

// ─────────── intent ───────────
/// 唯一的下发入口（分享、剪切板、插件动作都走这里）
#[tauri::command] async fn intent_dispatch(state: State<'_, AppState>, source: IntentSourceDto) -> Result<IntentId, IpcError>;
#[tauri::command] async fn intent_snapshot(state: State<'_, AppState>, intent_id: IntentId) -> Result<IntentSnapshot, IpcError>;
#[tauri::command] async fn intent_decide(state: State<'_, AppState>, intent_id: IntentId, decision: IntentDecisionDto) -> Result<(), IpcError>;
#[tauri::command] async fn intent_cancel(state: State<'_, AppState>, intent_id: IntentId) -> Result<(), IpcError>;

// ─────────── transfer ───────────
#[tauri::command] async fn transfer_send_file(state: State<'_, AppState>, req: SendFileRequest) -> Result<TransferId, IpcError>;
#[tauri::command] async fn transfer_send_text(state: State<'_, AppState>, req: SendTextRequest) -> Result<TransferId, IpcError>;
#[tauri::command] async fn transfer_cancel(state: State<'_, AppState>, transfer_id: TransferId) -> Result<(), IpcError>;
#[tauri::command] async fn transfer_list(state: State<'_, AppState>) -> Result<Vec<TransferSnapshot>, IpcError>;

// ─────────── plugins ───────────
#[tauri::command] async fn plugins_list(state: State<'_, AppState>) -> Result<Vec<PluginSnapshot>, IpcError>;
#[tauri::command] async fn plugins_catalog_search(state: State<'_, AppState>, query: CatalogQuery) -> Result<Vec<CatalogItem>, IpcError>;
#[tauri::command] async fn plugins_install(state: State<'_, AppState>, source: InstallSourceDto) -> Result<PluginId, IpcError>;
#[tauri::command] async fn plugins_uninstall(state: State<'_, AppState>, plugin_id: PluginId, opts: RemoveOptionsDto) -> Result<RemoveReport, IpcError>;
#[tauri::command] async fn plugins_enable(state: State<'_, AppState>, plugin_id: PluginId) -> Result<(), IpcError>;
#[tauri::command] async fn plugins_disable(state: State<'_, AppState>, plugin_id: PluginId) -> Result<(), IpcError>;
#[tauri::command] async fn plugins_consent_respond(state: State<'_, AppState>, plugin_id: PluginId, granted: Vec<String>, denied: Vec<String>) -> Result<(), IpcError>;
#[tauri::command] async fn plugins_check_update(state: State<'_, AppState>, plugin_id: PluginId) -> Result<Option<UpdateInfo>, IpcError>;
#[tauri::command] async fn plugins_update(state: State<'_, AppState>, plugin_id: PluginId, to: String) -> Result<(), IpcError>;
#[tauri::command] async fn plugins_sync_plan(state: State<'_, AppState>, plugin_id: PluginId) -> Result<SyncPlanDto, IpcError>;
#[tauri::command] async fn plugins_sync_run(state: State<'_, AppState>, plan: SyncPlanDto) -> Result<SyncReport, IpcError>;
#[tauri::command] async fn plugins_audit(state: State<'_, AppState>, plugin_id: PluginId, limit: u32) -> Result<Vec<AuditEntryDto>, IpcError>;
#[tauri::command] async fn plugins_request_registry_search(state: State<'_, AppState>, query: String) -> Result<Vec<CatalogItem>, IpcError>;

// ─────────── UI 插件专用（不同于上表，见 §7）───────────
#[tauri::command] async fn plugin_ui_rpc(state: State<'_, AppState>, token: u128, method: String, args: serde_json::Value) -> Result<serde_json::Value, IpcError>;
#[tauri::command] async fn plugin_ui_ready(state: State<'_, AppState>, token: u128) -> Result<(), IpcError>;

// ─────────── config ───────────
#[tauri::command] async fn config_get(state: State<'_, AppState>) -> Result<ConfigSnapshot, IpcError>;
#[tauri::command] async fn config_set(state: State<'_, AppState>, patch: ConfigPatch) -> Result<(), IpcError>;

// ─────────── diagnostics ───────────
#[tauri::command] async fn diag_export(state: State<'_, AppState>, opts: DiagExportOptions) -> Result<Utf8PathBuf, IpcError>;
#[tauri::command] async fn diag_metrics(state: State<'_, AppState>) -> Result<MetricsSnapshot, IpcError>;
#[tauri::command] async fn diag_logs_tail(state: State<'_, AppState>, lines: u32) -> Result<Vec<String>, IpcError>;
```


### 2.3 错误传输

```rust
/// 所有 command 的 error 类型
#[derive(Debug, Serialize)]
pub struct IpcError {
    /// 稳定错误码（前端可 switch）
    pub code: ErrorCode,
    /// 人类可读信息（可本地化的消息键 + 参数）
    pub message: MessageKey,
    pub params: serde_json::Value,
    /// 结构化细节（含 SynraError 的字段）
    pub details: Option<serde_json::Value>,
    /// 是否值得重试
    pub retryable: bool,
    /// 该错误的可操作建议（UI 可以渲染成按钮）
    pub actions: Vec<SuggestedAction>,
}

#[derive(Debug, Serialize)]
#[serde(tag = "kind", rename_all = "snake_case")]
pub enum SuggestedAction {
    OpenSettings { section: String },
    RequestCapability { plugin_id: PluginId, capability: String },
    InstallPlugin { plugin_id: PluginId },
    ConnectDevice { device_id: DeviceId },
    Retry,
    ViewLogs { trace_id: String },
}
```

**为什么不用 `anyhow` 直接序列化**：前端需要**可编程**的错误（switch code、渲染按钮），而不是一段字符串。

```rust
impl From<synra_core::Error> for IpcError {
    fn from(e: synra_core::Error) -> Self {
        match e {
            synra_core::Error::Protocol(SynraError::CapabilityDenied { plugin_id, capability, .. }) => Self {
                code: ErrorCode::CapabilityDenied,
                message: MessageKey::new("error.capability_denied"),
                params: json!({ "pluginId": plugin_id, "capability": capability }),
                details: None,
                retryable: false,
                actions: vec![SuggestedAction::RequestCapability { plugin_id, capability }],
            },
            // ...
        }
    }
}
```

---

## 3. 类型生成（Rust → TypeScript）

### 3.1 工具选型

| 工具        | 特点                                                    | 评价              |
| ----------- | ------------------------------------------------------- | ----------------- |
| **`ts-rs`** | 派生宏 `#[derive(TS)]`，生成 `.d.ts` 到指定目录，成熟稳定   | **采用**（主）     |
| `specta`    | 与 tauri-specta 集成，能直接生成 command 绑定             | 可选（见 §3.3）    |
| `typeshare` | 面向多语言（Swift/Kotlin/TS）                            | 若需要移动端原生类型时补充 |

**决定**：用 `ts-rs` 生成数据模型；用 **`tauri-specta`** 生成 command/event 的类型安全绑定（避免手写 `invoke("devices_snapshot")` 字符串）。

### 3.2 生成流程

```rust
// crates/synra-protocol/src/lib.rs
#[derive(Debug, Clone, Serialize, Deserialize, TS)]
#[ts(export, export_to = "../../../ui/generated/")]
#[serde(rename_all = "camelCase")]
pub struct IdentitySnapshot {
    pub device_id: DeviceId,
    pub display_name: String,
    pub platform: PlatformKind,
    pub app_version: String,
    pub sdk_version: String,
}
```

```make
# justfile
types:       ; cargo test --workspace --features ts-export export_bindings
types-check: ; cargo test --workspace --features ts-export export_bindings && git diff --exit-code ui/generated/
```

**CI 门禁**：`types-check` 必须通过 → **Rust 类型变了但没重新生成 TS 会导致 CI 失败**。

### 3.3 前端调用形态

```ts
// src/src/ipc/client.ts
import { commands, events } from '../generated/bindings'   // ← tauri-specta 生成

// 用法：完全类型安全
const snapshot = await commands.devicesSnapshot()          // 返回 Result<DevicesSnapshot, IpcError>
if (snapshot.status === 'error') { handle(snapshot.error) }

// 事件订阅：类型安全
const unlisten = await events.deviceCardUpdated.listen(({ payload }) => {
  devicesStore.upsert(payload)
})
```

---

## 4. 事件通道

### 4.1 三种事件

| 通道                    | 用途                                       | 背压                        |
| ----------------------- | ------------------------------------------ | --------------------------- |
| `kernel://event`         | `KernelEvent` 的增量事件                     | Tauri emit（丢弃策略由前端处理） |
| `kernel://log`           | 内核日志（tracing 的 ring buffer 尾部）       | 节流（默认 200ms 批量）       |
| `plugin-ui://{nonce}`    | 发给特定 UI 插件 WebView 的事件              | 每插件独立通道                |

### 4.2 前端事件处理模式

```ts
// src/src/kernel/event-bridge.ts
export function installKernelEventBridge(kernel: KernelClient, store: RootStore) {
  const unlisten = events.kernelEvent.listen(({ payload }) => {
    switch (payload.type) {
      case 'device_card_updated':   store.devices.upsert(payload.payload); break
      case 'pairing_requested':     store.pairing.pushRequest(payload.payload); break
      case 'intent_candidates_ready': store.intent.showCandidates(payload.payload); break
      case 'plugin_install_progress': store.plugins.setProgress(payload.payload); break
      case 'plugin_faulted':        store.plugins.markFaulted(payload.payload); break
      // ...
      default:
        // ★ 未知事件类型（Rust 加了新事件但前端未更新）
        //   → 触发一次全量 snapshot（自愈），而不是崩溃
        void store.refreshAll()
    }
  })
  return unlisten
}
```

**要求**：
- 事件 payload 用 `#[serde(tag = "type", content = "payload")]` → 前端可辨识；
- **`Lagged` 处理**：如果 Tauri 的事件通道丢事件（极端情况下），前端要能检测（例如事件里带单调递增序号）→ 重新 snapshot。

### 4.3 事件序号（防丢事件的保险）

```rust
#[derive(Debug, Clone, Serialize)]
pub struct KernelEventEnvelope {
    /// 单调递增序号（每次内核发布事件 +1）
    pub seq: u64,
    pub at: i64,
    #[serde(flatten)]
    pub event: KernelEvent,
}
```

前端：`if (envelope.seq !== lastSeq + 1) { void refreshSnapshot() }`。


---

## 5. 前端状态管理（从 Pinia stores 迁移）

### 5.1 现状与目标

| 旧 store / 状态源                | 新归属                                     |
| -------------------------------- | ------------------------------------------ |
| `stores/app-shell.ts`（UI 折叠等） | ✅ **保留**（纯 UI 状态）                    |
| `stores/lan-discovery.ts`         | ❌ 删除 → 内核 `devices_snapshot` + 事件     |
| `stores/pairing.ts`               | ⚠️ 部分保留（弹窗可见性/UI）+ 内核为权威      |
| `stores/paired-reconnect.ts`      | ❌ 删除 → 内核 `ReconnectSupervisor`         |
| `composables/use-plugin-catalog`  | ⚠️ 保留（UI 分页/搜索）+ 内核 `plugins_list`  |
| `composables/use-connect-page`    | ❌ 删除 → 内核 `devices_*` / `pairing_*`    |
| `lib/paired-devices-storage`      | ❌ 删除 → 内核 store                        |
| `lib/plugin-registry-preferences` | ❌ 删除 → 内核 config                       |
| `lib/pairing-outbound-pending`    | ❌ 删除 → 内核 `PairingSession`             |

**判据**（[03 §5](./03-architecture-baseline.md) 的状态归属表）：跨设备、跨重启、被权限约束 → 内核；纯展示、纯本地交互 → 前端。

### 5.2 前端 store 结构（目标）

```
src/src/stores/
├─ ui.ts            # 壳层（侧栏折叠、主题、窗口尺寸）        ← 纯 UI
├─ devices.ts       # 设备卡片列表（镜像内核快照 + 增量事件）   ← 只读镜像
├─ pairing.ts       # 配对弹窗队列与交互态                    ← UI + 内核权威
├─ plugins.ts       # 插件列表、安装进度、授权对话框           ← 只读镜像 + UI
├─ intent.ts        # 意图候选、确认弹窗、执行进度             ← 只读镜像 + UI
├─ transfers.ts     # 传输进度                                ← 只读镜像
├─ config.ts        # 配置（双向：读镜像 + 写命令）             ← 镜像 + 命令
└─ diagnostics.ts   # 诊断页数据                              ← 按需拉取
```

**关键约束**：`devices` / `plugins` / `intent` / `transfers` 这些 store 是**只读镜像**：
- 它们**不**包含业务逻辑（不判断"该不该连"、"该不该安装"）；
- 它们**只**：缓存快照、应用增量事件、暴露给组件；
- 任何动作都通过 `commands.*` 发起，然后**等待事件**更新状态（不做乐观更新，除非有明确需求）。

```ts
// src/src/stores/devices.ts
export const useDevicesStore = defineStore('devices', () => {
  const cards = ref<DeviceCard[]>([])
  const scanning = ref(false)
  const byId = computed(() => new Map(cards.value.map(c => [c.deviceId, c])))

  function applySnapshot(s: DevicesSnapshot) {
    cards.value = s.cards
    scanning.value = s.scanning
  }

  function upsert(card: DeviceCard) {
    const i = cards.value.findIndex(c => c.deviceId === card.deviceId)
    if (i >= 0) cards.value[i] = card
    else cards.value.push(card)
  }

  function remove(deviceId: string) {
    cards.value = cards.value.filter(c => c.deviceId !== deviceId)
  }

  return { cards, scanning, byId, applySnapshot, upsert, remove }
})
```

### 5.3 组件层几乎不变

`components/**` 的展示组件（`PluginCard`、`DeviceDiscoveryList`、`PairingRequestDialog`、`AppShellLayout`…）**保留结构**，只改数据来源：

| 旧写法                                         | 新写法                                       |
| ---------------------------------------------- | -------------------------------------------- |
| `const store = useLanDiscoveryStore()`          | `const devices = useDevicesStore()`            |
| `store.peers`                                   | `devices.cards`                                |
| `store.connectToDevice(id)`                     | `await commands.devicesConnect(id)`            |
| `useSynraSystemEnvelope().send(...)`             | `await commands.intentDispatch({ kind: 'direct', ... })` |

**迁移量估计**：`pages/` 4 个文件 + `components/discovery/` 4 个 + `components/plugins/` 3 个 + `composables/` 8 个中约 5 个被删除。

---

## 6. 前端构建与平台差异

### 6.1 WebView 引擎矩阵

| 平台    | WebView            | 需要处理的差异                                     |
| ------- | ------------------ | -------------------------------------------------- |
| Windows | WebView2 (Chromium) | 最接近"标准"；支持 importmap 等新特性               |
| macOS   | WKWebView (WebKit)  | 部分 CSS/API 差异；`backdrop-filter`、`dialog` 等    |
| Linux   | WebKitGTK           | **差异最大**：CSS 支持较弱、iframe 隔离较弱           |
| Android | System WebView      | Chromium 内核，版本随设备（可能较旧）                |
| iOS     | WKWebView           | 与 macOS 类似                                       |

**策略**：
1. 使用 **Baseline Widely Available** 的 CSS/JS 特性（避免 `:has()` 等新特性作为必需）；
2. CSS 用 UnoCSS + 显式 `targets`（不用最激进的 preset）；
3. 运行时探测（`WebviewKind`）用于**渐进增强**，不用于**基本功能**；
4. CI 至少在 WebView2 + WebKitGTK（Docker）+ WKWebView（macOS runner）上跑视觉/功能回归。

### 6.2 前端工具链

| 用途     | 选型                                        | 说明                                   |
| -------- | ------------------------------------------- | -------------------------------------- |
| 构建     | **Vite**（保留，Tauri 官方模板）              | 不用 Vite+（Rust 侧已接管工具链职责）    |
| 框架     | Vue 3 + `vue-router` + Pinia                 | 保留（组件层可复用）                    |
| 样式     | UnoCSS                                       | 保留                                    |
| 测试     | Vitest（组件/单元）+ Playwright（E2E，可选）   | 保留 + 增加                            |
| 类型     | TypeScript（版本随生态）                      | 生成物来源是 Rust                       |

> 这是一个可以商量的决策（记录在 [21 ADR-009](./20-adr.md)）。

---

## 7. UI 插件的 IPC 面（独立且受限）

**这是与宿主 UI 完全不同的通道**。理由见 [10 §6](./10-capability-and-sandbox.md)（会话令牌）。

### 7.1 通道设计

宿主 UI 用**具名 command**（`devices_snapshot` 等 ~40 个）。
UI 插件用**单一 command + method 字符串**：

```rust
#[tauri::command]
async fn plugin_ui_rpc(
    state: State<'_, AppState>,
    token: u128,
    method: String,
    args: serde_json::Value,
) -> Result<serde_json::Value, IpcError> {
    // 1. 校验会话令牌（★ 插件不能伪造 plugin_id）
    let session = state.ui_sessions.validate(token)?;

    // 2. 方法白名单（★ 插件只能调这些）
    let op = PluginUiOp::parse(&method)
        .ok_or(IpcError::unknown_method(&method))?;

    // 3. 闸门（用 session.claimed_plugin_id 与 session.grants）
    state.gate.check_ui_op(&session, &op, &args)?;

    // 4. 执行
    state.execute_ui_op(&session, op, args).await
}
```

### 7.2 允许的方法（`PluginUiOp`）

```rust
#[derive(Debug, Clone, Copy)]
pub enum PluginUiOp {
    // env（无闸门）
    EnvGet,
    EnvCan,

    // log
    LogWrite,

    // event
    EventEmit,
    EventSubscribe,          // 订阅在 WebView 侧由 IPC 事件通道承载
    EventUnsubscribe,

    // device
    DeviceList,
    DeviceSend,

    // storage
    StorageGet, StorageSet, StorageDelete, StorageKeys,

    // network
    NetworkFetch,

    // clipboard
    ClipboardRead, ClipboardWrite,

    // platform（受约束）
    PlatformOpenUrl, PlatformNotify,

    // transfer
    TransferSendBlob,        // 从 WebView 传 Blob（经 Tauri 大文件通道）
    TransferStatus,

    // 与本插件 headless 部分通信
    SelfInvokeAction,

    // 页面生命周期（宿主 UI 需要知道插件页面何时就绪）
    PageReady, PageNavigate,
}
```

**明确不允许**（即使插件声明了对应 capability）：

| 不允许的方法                          | 理由                                           |
| ------------------------------------- | ---------------------------------------------- |
| `PlatformSimulateKeys`                 | UI 插件（第三方 JS）**永远**不能模拟按键          |
| `FsRead` / `FsWrite`                   | UI 插件通过 `NetworkFetch` 与 `TransferSendBlob` 即可；直接 FS 风险过高 |
| `PluginInstall` / `PluginEnable`       | 插件不能管理其他插件                             |
| `PluginConsentRespond`                 | 插件不能替用户授权                               |
| 任何 `devices_*` 管理命令                | 插件不能扫描、配对、忘记设备                     |
| `ConfigSet`                            | 插件不能改全局配置（只能用自己的 storage）        |

> **设计理由**：UI 插件跑在 JS 环境里，是**最不可信的插件形态**。它的 IPC 面必须是**最小可用集**。
> headless 插件（Wasm）可以有更丰富的能力（例如 `fs`、`simulate-keys`），因为 Wasm 沙箱提供了强隔离。

### 7.3 插件 WebView 的创建

```rust
// src-tauri/src/plugin_ui.rs

pub async fn open_plugin_page(
    app: &AppHandle,
    plugin_id: &PluginId,
    page_key: &str,
) -> Result<WebviewWindow, PluginUiError> {
    // 1. 校验插件已激活且是 UI/hybrid
    let session = app.state::<AppState>();
    let plugin = session.plugins.require_active_ui(plugin_id)?;

    // 2. 签发 UI 会话令牌
    let nonce = session.ui_sessions.issue(plugin_id.clone(), plugin.version.clone(), plugin.grants.clone());

    // 3. 构建插件页面 URL
    //    ★ 独立 origin：synra-plugin://<plugin-id>/<page-key>/
    //      通过 Tauri 的自定义协议 handler 提供文件，并强制 CSP
    let url = format!("synra-plugin://{}/{}", plugin_id, page_key);

    // 4. 创建 WebView，注入初始化脚本（含 token，闭包捕获，不上 window）
    let window = WebviewWindowBuilder::new(app, format!("plugin-{}-{}", plugin_id, page_key), WebviewUrl::CustomProtocol(url))
        .title(plugin.title.clone())
        .initialization_script(build_plugin_init_script(nonce))
        .on_navigation(move |url| {
            // ★ 限制导航范围：只允许插件自己的 origin 与白名单外链
            url.scheme() == "synra-plugin" || is_external_link(url)
        })
        .build()?;

    Ok(window)
}
```

**CSP（由自定义协议 handler 注入响应头）**：

```
Content-Security-Policy:
  default-src 'none';
  script-src 'self';
  style-src 'self' 'unsafe-inline';
  img-src 'self' data: blob:;
  font-src 'self';
  connect-src 'none';         ← ★ 插件不能直接发网络请求，必须走 synra.network.fetch
  frame-src 'none';
  object-src 'none';
  base-uri 'none';
  form-action 'none';
```

**`connect-src 'none'` 是关键**：它让 UI 插件**无法绕过内核**直接 `fetch`。
如果插件确实需要网络，走 `synra.network.fetch`（内核代理 + 闸门 + SSRF 防护）。

---

## 8. 迁移路径（前端逐步切换）

```
阶段 1：搭骨架
  · 新建 src/（app 仓库；从旧 apps/frontend 复制组件层）
  · 建立 ipc/client.ts + generated/bindings
  · 用一个假内核（返回静态 snapshot）跑通壳层与页面

阶段 2：设备页切换
  · devices store 接内核 snapshot + 事件
  · 删除 stores/lan-discovery、stores/paired-reconnect
  · pairing 走内核命令

阶段 3：插件页切换
  · plugins store 接内核
  · 删除 plugins/host/**（装载逻辑全在内核）
  · 插件 UI 走 WebView + plugin_ui_rpc

阶段 4：清理
  · 删除 composables 中被内核接管的（use-connect-page、use-paired-auto-connect…）
  · 删除 lib/ 下的协议与存储实现
  · 删除 bootsrap/**（内核负责）
```

**每个阶段都要可运行**（不能有"全绿才能跑"的阶段）。

---

## 9. 实现清单

> ★ 这一节在实现之前写就，长期整节未勾选。逐条核对源码后：**这一节目前没有一条可以勾。**
> 三样东西是真的（`IpcError` 三件套、`KernelEventEnvelope` 带 `seq`、`EventDispatcher` 的 gap/unknown 逻辑），
> 但三样都缺"接上去"的那一半；其余九条是从未开工。
> 下面写清每一条的**真实状态**与真实名字。

- [ ] `IpcError` + `ErrorCode` + `SuggestedAction` + `From<core::Error>` ——
      **四分之三**。`IpcError` 在 `crates/synra-ipc/src/error.rs:125`，`:127` 转出 protocol 的 `ErrorCode`，
      `:144` `SuggestedAction`，转换是 `:221` 的固有方法 `IpcError::from_synra_error(&SynraError)`。
      ★ **`From<core::Error>` 不存在**：全仓 0 处 `impl From<..> for IpcError`，
      且 `synra-ipc/Cargo.toml` 只依赖 protocol/capability/observe —— `synra_core::Error` 根本不在作用域里。
      > ★ **而其中四个 intent 命令的理由，原来是错的。**
      > `synra-rust-app/src/ipc/registration.test.ts` 原来写的是
      > “needs the intent board behind AppState” —— 而 **board 已经在 `AppState` 里了**
      > （`state.rs:57 pub board: Arc<Mutex<Board>>`，`:119` 构造）。理由错的方向正好是**会代价的那个**：
      > 读者一查，看到 board 已经在了，就以为这件事快好了。
      > * 真正缺的是**编排本身**：`synra-daemon/src/runtime/intent.rs` 里的
      >   `raise` → `Matcher::match_share` → `settle` → `choose` → `confirm` → `execute` → `conclude`
      >   **住在一个二进制 crate 里**，所以第二个孿主（桌面 shell）**无法复用**，
      >   只能把大约 250 行抄一遍 —— 而那正是设计反对的事。
      > * ★ 它除了 `synra_control::Refusal` 之外**没有任何守护进程依赖**，所以“搬进库”是一次
      >   干净的移动（换一个域内错误类型，两个孿主各自映射到 `Refusal` / `IpcError`），
      >   而不是一次重写。
> * ★ **已完成（已提交）**：编排现在在 `synra-core/src/dispatch/orchestrate.rs`，
>   对外是 `dispatch` / `snapshot` / `decide` / `cancel` / `Router`，错误类型是 `RouteError`（带
>   `synra_protocol::ErrorCode` + 一句人话）。守护进程的 `runtime/intent.rs` 只剩下一层映射：
>   `RouteError` → `Refusal`，一行。桌面 shell 可以用同一套代码把它映射到 `IpcError`，
>   而不必把 250 行抄一遍。
> * ★ **而且这次移动顺手暴露了一个死掉的守卫**：`host_calls_tests.rs` 里的
>   `every_action_shaped_name_is_one_the_contract_declares` **从写下那一刻起就没有跑过**——
>   一次插入把新测试放在了 `#[test]` 与它的函数之间，于是属性重复、
>   而原来那个函数**一个属性都没有**。编译器只是很小声地说了一句 warning，
>   而 `cargo test` 输出里看不出少了一条 —— 直到这次重构让它变成了错误。
- [ ] ~40 个 Tauri command（见 §2.2）—— **18/39**。
      > ★ **18/39：`plugins_audit` 也注册了。** 它原来的理由是“the audit ring has no production
      > writer yet”—— 写下时是真的：`AuditRing` 有界、有测试、可查询，而**没有人往里写**。
      > 现在写入者存在（`PluginSet` 为每个实例装 session，调用后把它记下的排进 ring），
      > 所以命令读的是**真的记录**。
      > * ★ **两处契约缺口被记下**：`AuditEntryDto` 有 `trace_id`，而 `AuditEntry`
      >   **没有 trace**（session 没拿到 trace），所以永远是 `None` —— 是**记录**的缺口，不是映射的。
      >   `AuditEntry::new` 的字段里也确实没有可以当 trace 的东西。
      > * ★ **而一个“清单”差点被我删错**：`commands/mod.rs` 的 `pub use`
      >   列表看起来无人使用且缺了七个命令，我删了它——而 `diag/tests.rs` 确实在用
      >   其中一个。**我的 grep 模式里漏了 `config_set`，而我没有跑测试就提交。**
      >   已恢复并补全，且在注释里写明它**不是清单**——清单是 `main.rs` 的 `generate_handler!`，
      >   而核对它的是 `src/ipc/registration.test.ts`。
      >
      > ★ **一个报告里的四个常量值，和一个与注释相反的值。**
      >
      > `diag_export` 把每个插件的 `version` 写成 `String::new()`、`active` 写成 `false`、
      > `had_refusals` 写成 `false`。后两个不是省略，是**声明**：尤其 `had_refusals: false` 告诉维护者
      > “这个插件从没被拒绝过”—— 而用户发包精压缩包的时候，往往就是在问“为什么不行”，
      > 而拒绝通常就是答案。三个现在都从 `PluginSet`（`version`/`contains`/`refusals_for`）和
      > `paired_devices`（`is_usable`）读。
      >
      > ★ **而更重的是 `devices.rs` 里那一行**：`card_from_paired` 的注释写着
      > “a paired device with no open session is **offline** rather than absent”，而紧跟着的代码是 `online: true`。
      > **注释是对的。** 所以仪表板对一台关了一周的笔记本亮着绿灯，用户第一次知道
      > 不对是发送失败的时候。现在用 `SessionRegistry::live_for`，它回答的正是卡片在问的问题。
      > 已有阴性对照：改回 `true` 会让 `a_paired_device_with_no_session_is_offline` 变红。
      > ★ **现在是 17/39：四个 intent 命令已注册**（`commands/intent.rs`，
      > `main.rs` 的 `generate_handler!`）。它们之前卡在的不是接线，而是**编排住在二进制里**：
      > 现在在 `synra_core::dispatch::orchestrate`，两个孿主各自只写一层映射。
      > ★ **而且这次还发现库返回的是无类型 JSON，这对第二个孿主是不够的**：
      > 界面的 `IntentCandidate` **必须有一个设备**，而内核的候选人**可能对“在哪里跑”没有意见**（那正是“在哪里”这个问题的来源）。
      > 所以 `snapshot` 现在返回类型化的 `IntentView` / `CandidateView`，而 `to_json()` 只是**其中一种**渲染。
      > ★ **两处契约不匹配被记下而不是抹平**：
      > 界面的 `Share` **没有 MIME**（shell 发 `text/plain`）；界面的 `IntentPrompt` 要求一个消息键和选项列表，
      > 而内核是用**状态**来问问题的 —— shell 没有把状态翻译成一句自己写的问题，
      > 而是直接传了 `None`（见 `intent_snapshot` 的注释）。
      > ★ 与之无关但同样是"看得见、跑不了"的一处：**CLI 的 `plugin` 子命令里，四个里只有一个能做任何事。**
      > `plugin_request`（`synra-cli/src/session.rs`）对除 `list` 外的每个 action 都返回 `None`，
      > 而 `question_for` 返回 `None` 会让 CLI 打出**"this command needs a running kernel, and none is reachable"**
      > —— 一句在内核正常运行时**为假**的话，它把用户送去检查错误的东西。
      > `plugin install/enable/disable/remove` 四个都出现在 `every_documented_command_exists` 的清单里，
      > 参数测试也只断言"能解析"。
      > ★ **本轮修好两个**：新增 `Request::PluginSetEnabled`，守护进程真的写库
      > （`set_plugin_enabled` 在 store 里存在、有测试、**在生产里从来没有调用方**），
      > CLI 的 `enable`/`disable` 真的发请求。
      > ★ **剩下两个仍然不做，但现在说的是真话**：`dispatch` 对 `Command::Plugin` 给出
      > "installing needs a registry client, and removing needs a decision about the plugin's stored data
      > that nothing has made yet"，而不是那句关于内核的假话。
      39 个名字确实存在，但只在 `crates/synra-ipc/src/command.rs` 的 `commands!` 表里；
      `src-tauri/src/main.rs` 注册了 13 个（identity 2、devices 4、plugins 3、config 2、diag 2）。
      未实现的 26 个逐条列在 `synra-rust-app/src/ipc/registration.test.ts`
      的 `NOT_YET_REGISTERED` 里，**每条都带原因**（区分"等接线"与"决定不做"）。
      ★ 本次新增的是 `diag_logs_tail`：它原本在"未实现"清单里，理由是
      "no log ring is behind AppState" —— 那条理由**过期了**（shell 现在有自己的日志，
      见 [18 §9](./18-observability.md)），于是命令从清单移到了注册表。
      这正是那些理由的用处：一份没有原因的未做清单会烂成没人能据以行动的东西。
      它读的是 **shell 自己的**日志（`AppState::start_logging` 写的），与 CLI 通过控制协议读的
      **内核**日志是两个进程的两份文件 —— 一个把两者混起来的命令会给用户看错的那份，而且看不出来。
      测试 `the_log_tail_reads_the_shells_own_log_including_the_debug_lines`
      断言两件不同的事：行**回来了**，以及 **debug 行也回来了**（安静过滤会只返回 INFO，
      而调用方正是为了看细节才跑这个诊断命令的）。
      另加 `the_log_tail_is_empty_rather_than_an_error_when_nothing_has_been_written`：
      全新安装还没写过日志，把它报成失败会让一个正常的安装看起来是坏的。
      ★ 两处原本守不住这件事的检查，本次都处理了：
      `command.rs` 里那条测试比的是"注册表 vs 同文件的手写列表"（两份同源拷贝互相比对），
      它同名断言的那句话——"文档里有而这里没有 = 前端会调而 shell 没有"——它**看不见**；
      `scripts/check-shapes.ts` 的 `checkCommandNames` 也只看同一张注册表。
      新加的 `registration.test.ts` **直接读 `command.rs` 解析出 39 个名字**，
      再与 `main.rs` 的 `generate_handler!` 两个方向比对，负控制确认会红。
- [ ] `ts-rs` 派生 + `tauri-specta` 绑定生成 + CI `types-check` —— **未做，且选型换了**。
      全仓 0 处 `ts-rs`/`tauri-specta`；真实机制是 `synra-ipc/src/ts_export.rs` **手写**生成两个 TS union
      （只导出 `ErrorCode` 与 `SuggestedAction`），靠 `serde_json::to_value` 反推变体名。
      所有 DTO 与命令绑定一个都没生成。CI 那半彻底没有：core 与 app 都没有 `.github/`，也没有 `justfile`。
- [ ] `KernelEventEnvelope`（带 `seq`）+ Tauri emit 桥 —— **一半**。
      `KernelEventEnvelope` + `SequenceTracker` 是真货（`crates/synra-ipc/src/event.rs:193`、`:243`）。
      ★ **emit 桥不存在**：`synra-rust-app` 下所有 `.rs` 搜 `emit`/`Emitter`/`listen`
      只命中 `build.rs` 的一句注释 —— envelope 没有生产者。
      ★ **本次补上了前端那一半，于是这个缺口现在是"一半对一半"而不是"两头都没有"。**
      新增 `synra-rust-app/src/ipc/tauri.ts` 的 `TauriKernel`：实现 `Bridge` 的
      `invoke` / `listen`，用 `@tauri-apps/api`（**不是** `window.__TAURI__` 全局，
      理由见那个文件的头部：全局是注入到**页面**里的，而这个页面是宿主界面，
      将来要托管第三方插件 UI；一个"只在某功能还没做时才安全"的决定不算决定）。
      `kernel.ts` 现在按 **`window.__TAURI_INTERNALS__` 是否存在**选择
      `TauriKernel` 还是 `FakeKernel` —— 这一句此前是注释里的承诺（"is not written yet"），
      现在是有两个分支、且两个分支都有测试的代码。
      ★ **但仍然不出声**：`src-tauri` 下依然 0 处 `emit`，
      所以 `TauriKernel.listen('kernel-event', …)` 会订阅到一个**永远不触发的通道**。
      后果具体：`DevicesPage.vue:194` 与 `injection.ts:270` 都真的调用了 `subscribe`，
      所以打包后的应用会画出快照、然后**再也不更新** —— 而页面看起来是正常的。
      补上它需要 shell 侧有一个事件源（本机内核的发现/传输回路）并把 envelope `emit` 到 WebView，
      那不是这一轮的范围，记在这里。
- [x] 前端 `event-bridge.ts`（未知事件 → 全量刷新、`seq` 断裂 → 全量刷新）——
      **逻辑是真的，也仍然是"接不上"**。真名 `EventDispatcher`（`src/ipc/dispatcher.ts`）：
      未知事件 → `#refresh('unknown')`，`seq` 断裂 → `#refresh('gap')`，两个都有测试。
      ★ **本轮之后它的状态变了，值得更新**：`src/kernel.ts` 不再固定接在
      `new FakeKernel()` 上，而是按环境选择；而 `TauriKernel` 提供真实的那一半。
      所以"生产代码 0 处引用"这句话现在只对 `EventDispatcher` 成立 ——
      **它仍然只有测试在 `new` 它**（`DevicesPage` 调的是 `kernel.subscribe`，
      走的不是 `EventDispatcher`）。这是又一个"写好、测过、没人调用"，
      而且它与上面那条是同一个洞的两端：没有事件生产者，所以接上也没有意义。
      ★ 顺带修掉一处真实违反：`DevicesPage.vue` 与 `injection.ts` 曾经从
      **`fake-kernel.ts`** 里 import 事件通道常量 —— 生产代码读夹具。
      常量移到 `src/ipc/channel.ts`，并新增守卫
      `fake-quarantine`（`scripts/check-shapes.ts`）：只有 `kernel.ts` 可以提到那个假实现。
      负控制：让 `DevicesPage.vue` 再 import 一次 → 守卫红并点名该文件。
- [ ] `PluginUiOp` 枚举 + 方法白名单 + `plugin_ui_rpc` —— **未做**。
      `PluginUiOp` 全仓 0 处；`plugin_ui_rpc` 只作为注释出现在 `command.rs:39`。
      `command.rs` 里 `Interface::{Host,Plugin}`、`:59` `allows()`、`:375` `plugin_allow_list()` 是真的且有测试，
      **但 app 侧 0 处调用**。
      ★ `injection.ts` 的 `BACKED` 布尔表不是这个方法白名单：它答的是"这个功能实现了没有"，
      不是"这个调用允许不允许"。
- [ ] UI 插件自定义协议 handler（`synra-plugin://`）+ CSP（`connect-src 'none'`）—— **未做**。
      全仓 0 处 `synra-plugin://`、`register_uri_scheme`、`WebviewWindowBuilder`。
      `plugin-ui/src/artifact/serving.rs` 只有 `check_path` 与 `content_type_for`，不产响应头。
      ★ 唯一的 CSP 是 `src-tauri/tauri.conf.json:22` 的 app 级 CSP，那是**宿主主窗口**的，不是插件页的。
- [ ] 插件 WebView 初始化脚本（token 闭包注入）—— **未做**：全仓 0 处 `initialization_script`，
      0 处 `Object.freeze`。`InjectionContext` 里**根本没有 token/nonce 字段** ——
      不是"没实现闭包"，是还没有可以闭包的东西。
- [ ] 导航范围限制（`on_navigation`）—— **未做**，全仓 0 命中。
- [ ] 前端 stores：`ui`/`devices`/`pairing`/`plugins`/`intent`/`transfers`/`config`/`diagnostics` —— **1/8**。
      只有 `src/stores/devices.ts`，其余七个没有文件。
      ★ 现存这一个确实是忠实的只读镜像：`:70` 整体替换快照、`:82/:94/:99` 只记录、
      `:133` 的 `switch` 只认三个事件且 `default` 返回 `false`，不含业务规则。
- [ ] WebView 引擎兼容矩阵（WebView2 / WKWebView / WebKitGTK / Android / iOS）+ 探测 —— **未做**。
      `WebviewKind` 只存在于设计文档（[09](./09-ui-plugin-model.md)、本文 §384），代码 0 处。
      ★ 比"没有"更糟的是：`src/ipc/injection.ts` 原先把 `platform`/`hostKind`/`appVersion`
      当成**硬编码常量**暴露（`platform: () => 'windows'`），在 Linux/macOS/Android 上就是撒谎。
      一个按 `platform()` 分支的插件在所有平台都会走 Windows 那条路，而且**不会报错** ——
      一个看起来合理的错答案和正确答案在调用点无法区分。本次已改成从 `InjectionContext` 读取，
      并加了会失败的测试（三个平台问出三个不同答案）。
- [ ] 前端迁移四阶段（§8），每阶段可独立运行 —— **前提不成立**。
      `synra-rust-app` 首个提交只含 `.gitattributes`/`.gitignore`/`README.md`；
      `git log --all -- "*lan-discovery*"` 无输出。§8 要删的那些文件**在本仓库从未存在过** ——
      这是从零构建，不是迁移，"每阶段可独立运行"没有指涉对象。
      这一条应当改写成"从零构建的四个里程碑"，或者删掉。
