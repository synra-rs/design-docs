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
- [ ] ~40 个 Tauri command（见 §2.2）—— **12/39**。
      39 个名字确实存在，但只在 `crates/synra-ipc/src/command.rs` 的 `commands!` 表里；
      `src-tauri/src/main.rs:153-166` 只注册了 12 个（identity 2、devices 4、plugins 3、config 2、diag 1）。
      未实现的 27 个逐条列在 `synra-rust-app/src/ipc/registration.test.ts`
      的 `NOT_YET_REGISTERED` 里，**每条都带原因**（区分"等接线"与"决定不做"）。
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
- [ ] 前端 `event-bridge.ts`（未知事件 → 全量刷新、`seq` 断裂 → 全量刷新）——
      **逻辑是真的，接不上**。真名 `EventDispatcher`（`src/ipc/dispatcher.ts:62`）：
      未知事件 → `:137` `#refresh('unknown')`，`seq` 断裂 → `:122` `#refresh('gap')`，两个都有测试。
      ★ 但生产代码 **0 处引用**它，19 个测试全都自己 `new`；`src/kernel.ts:25` 把界面接在
      `new FakeKernel()` 上（该文件自己写着 "That is not written yet"）。
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
