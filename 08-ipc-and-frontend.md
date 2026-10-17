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
/// 列出 board 上持有的全部 intent（裸数组，`Many`）—— 首页"received"面板要的是它，而它一直不存在
#[tauri::command] async fn intent_list(state: State<'_, AppState>) -> Result<Vec<IntentSnapshot>, IpcError>;
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
2. CSS 用 UnoCSS，配置与规则见 [28](./28-app-ui.md)；**CSS 输出兼容性靠 Vite 的 `build.cssTarget`**（见下方注）；
3. 运行时探测（`WebviewKind`）用于**渐进增强**，不用于**基本功能**；
4. CI 至少在 WebView2 + WebKitGTK（Docker）+ WKWebView（macOS runner）上跑视觉/功能回归。

> ★ **本条原写的是"UnoCSS + 显式 `targets`"，而 UnoCSS 里没有这个旋钮。**
> `presetWind3` 的选项继承自 `preset-mini`（`important` / `dark` / `preflight` 一类），没有 `targets`；
> 而 `synra-app` 的 `vite.config.ts` 当时只设了 `target: 'es2022'`，没有 `cssTarget`。
> 也就是说这条要求的**意图是真的（别输出新 CSS 给弱的 WebView），指的那个机制不存在**。
> 落点改为 Vite 的 `build.cssTarget`，规则编号 [28 §4 R15](./28-app-ui.md)。

### 6.2 前端工具链

| 用途     | 选型                                        | 说明                                   |
| -------- | ------------------------------------------- | -------------------------------------- |
| 构建     | **Vite**（保留，Tauri 官方模板）              | 不用 Vite+（Rust 侧已接管工具链职责）    |
| 框架     | Vue 3 + `vue-router` + Pinia                 | 保留（组件层可复用）                    |
| 组件     | **antd**，配置**只能**来自 `@synra/theme/antd` | 2026-10-16 定；[28 §2 R51–R53](./28-app-ui.md)。`antd` 在 `@synra/theme` 里是 **optional peer**，官网不为它付装包 |
| 样式     | UnoCSS（配置与全部样式规则见 [28](./28-app-ui.md)） | 保留；`@synra/theme/unocss` 提供宿主与插件共用的 token（★ 这条原来写的是旧包名 `@synra/unocss`） |
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
- [ ] ~40 个 Tauri command（见 §2.2）—— **22/40**（2026-10-18）。
      ★ `plugins_install` 是第 22 条，它进得来是因为**安装管线搬进了内核**：`synra_core::install` 现在是
      daemon 与 Tauri 壳共用的那一份（校验顺序、staging、回滚、写行），两个宿主各自只提供"这台机器能做什么"
      （`HostFacts`）。在此之前，安装能力只有 CLI 有，而界面的插件页连"装一个"都无处可点。
      它同时是 `InstallSourceDto` 的第一个消费者：`Path` 走管线，`Registry` 被**按名字**拒
      （`Unsupported`，理由是没有取件客户端）—— 而不是报一个从未发生过的网络错误。
      ★ 而这个数字刚刚落后过一轮：第 104 轮把 `diag_metrics` 移了出去，
      而下面的散文记下了那件事、**这一行没有**。一个数字比一段散文更容易被忘，
      而它正是读者会当真的那个：“剩下十九条”与“剩下十八条”对一个想知道还差多少的人是两个答案。
      ★ **而这一轮是 21/40：`intent_list` 进了契约（§2.2），因为界面一直在假装它存在。**
      抓出它的不是人眼，是一条**新守卫**：`scripts/check-shapes.ts` 的 `command-arguments`
      读 `shell.rs` 的 wrapper 签名（参数名、`Option<T>` 之外都必填、参数名按 `#[tauri::command]` 的默认规则转 camelCase），
      再与每个 call site 逐键对照。它上线第一次运行就报了两处 —— 而这正是"漏参数"这类错误的形状：
      **假内核忽略参数**（`invoke(command, _args?)`）、**契约表里根本没有参数那一半**
      （`commands!` 每行只有 域名/接口/返回形状/是否可变）、旧守卫**只比名字**，三处一起把它藏住了。
      * `DevicesPage.vue` 调 `devices_scan_start` 没传 `opts`：`#[tauri::command]` 在命令体之前反序列化参数，
        所以拒绝发生在**桥接层**，客户端把"不是 `IpcError` 的 rejection"归类成 `error.bridgeFailed`，
        用户读到 "The interface could not reach the kernel."，而真正的原因躺在 `params.detail`（按 R34 界面不渲染它）。
        现在界面传 `opts`，并且真的扫得动了 —— 见下面那两条。
      * ★ **而这一轮最后把扫描做成了真的：发现能力搬进了窗口自己。** 之前那条注释写着"discovery 属于守护进程，
        shell 只差半个内核"，其中**一半已经过期**：`synra-control::Request::Devices{scan_ms}` 早就有按需扫描
        （daemon 侧 `runtime/devices.rs::scan` 会设 `scan_until_ms` 并等出窗口，CLI 的 `synra devices --scan-ms` 在用），
        而 `synra-core` 的 worker 本来就读这个截止时间（`tick(..., on_demand)`）。
        真正挡住的是**两个宿主有两个数据目录**：窗口的是 Tauri 的 `app_data_dir`（`%APPDATA%\com.synra.app`），
        守护进程的是 `SYNRA_DATA_DIR` / 临时目录 —— 让守护进程扫，扫到的设备写进**它自己的** `devices_seen`，
        而界面读的是本进程那张表，结果是"扫到三台、界面一台都没有"，比拒绝更糟。
        所以按 [05 §2.1](./05-system-architecture.md) 的划分（桌面端是**同进程**内核）把 `DiscoveryService` 接进 shell：
        `src-tauri/src/discovery.rs` 绑两个 socket、用 `synra_core::worker::run_recording` 起一条被 supervisor 管的线程、
        把听到的写进**本进程**的 `devices_seen`；`devices_scan_start` 于是做三件事 —— 记下 `scanning`（running 的语义）、
        发布截止时间并等出 `SCAN_PASSIVE` 的窗口、清掉它再作答。命令标了 `#[tauri::command(async)]`：
        Tauri 对**同步**函数加这个属性会把它放到 sync threadpool（宏里就叫这个名字），窗口在问网络的时候照样重绘。
        * ★ 顺带修掉了 `synra-core` 里一个**"读者有、写者有、却永远空"的表**：`run_recording` 的回调本来把
          `host.usable(...)`（`current ∧ displayable`，而 `displayable` 从 `Handshaked` 起）交出去，
          于是一个只是**广播了自己**的设备永远不会被交到回调手里 —— 而 `devices_seen` 自己的文档写的是
          "a sighting of a device this kernel has not paired with"。现在交出去的是**这一 tick 真正吸收到的**候选
          （`Absorbed::New/Updated`），`Unchanged`/`Offline` 不算发现，否则就是每个间隔给每台设备写一行它已经写过的东西。
        * ★ 还有一处**不能不说清的诚实问题**：`DeviceAnnouncement` 要求非零 `port`（`validate` 拒绝 0），
          而这个构建**没有传输层** —— 窗口不在任何端口上接受连接。所以它宣告的是它**真的绑了**的那个端口
          （发现响应端口），并把这句写进模块头：对端若来拨号，遇到的是发现响应器而不是传输层，
          这是"这个构建没有那半边"的事实，不是"一个谁都没占的地址"。等 shell 有了传输层，这里换成那个端口。
      * `HomePage.vue` 调 `intent_snapshot` 既没传 `intentId`、又读 `answer.value.intents`：
        这不是缺参数而是**缺一条命令**。`Board::all()` 一直在（它存在的理由是持久化要存"变了的"，包括刚结束的），
        缺的是名字 —— 所以 `intent_list`（`Many`，**裸数组**）进了契约、shell 与假内核，那个面板现在读列表本身。
      * 参数那一侧还有个小的：cached 扫描曾经留下一个 `ScanState{0,0,0}`，而 `scanning` 的语义是
        **正在进行的扫描**（store 判 `scan !== undefined`）—— 于是按钮变永久"Scanning"、进度条永远不动，
        直到进程重启。cached 扫描现在什么都不留，而**网络扫描**在等待期间留下的是真的那条（带 `total_ms`），
        作答前清掉。
      * 假内核现在按 `shell.rs` 的必填键**拒绝**（消息与 `tauri::Error::InvalidArgs` 同形），
        而且 cached 与网络两种模式真的分开了；那张必填表本身也被同一个脚本**双向**核对，
        所以"假内核比真外壳宽松"这个方向不会再回来。
      ★ **而第 85 轮把两个宿主的 `Host` 实现逐方法对了一遍**（守护进程的 `KernelHost` vs shell 的 `DesktopHost`）：
      **26 个方法里只有 1 个不同 —— `platform`**：守护进程拒绝，shell 真做。
      那是**故意的**（无窗口宿主没有剪贴板与通知），而且 `synra-platform` 已经写了
      `HeadlessPlatform` “先过守卫再拒绝”的语义。
      ★ 这个扫描值得记下来，因为它在本会话里**三次抓到了真的不对称**：
      审计 sink 只有守护进程有（第 82 轮补上）、引擎能力检查只有守护进程有（第 81 轮补上）。
      现在它们对齐了，而这条以后可以当作“两边一致”的基线来重跑。
      > ★ **一个归零的进度条：`devices_scan_start` 从来没有扫描过。**
      > 它写一个 `ScanState { elapsed_ms: 0, found: 0 }` 就返回，而全仓**没有任何代码更新它**，
      > 所以界面画的是一条永远不会动的进度条。旧注释自己反对这件事：
      > “giving it a running state with a deadline would show a progress bar that cannot progress — which is
      > worse than an honest finished one” —— 而它对**每一次非 cached 扫描**都在这么做。
      > * ★ **为什么 shell 扫不了**：`synra_core::DiscoveryService` 在库里（`worker.rs`，
      >   `bind_on`/`run_recording`），但它要**宣告一个端口**，而 shell 不在任何端口上服务。
      >   发现属于**接受连接的进程**，也就是 daemon；它已经在扫了（`serve.rs:346` 的线程活到进程结束），
      >   并把听到的写进**同一张 `devices_seen`**—— 所以列表是新的。
      > * ★ **缺的是“立即扫一次”**：daemon 有 `on_demand` 截止时间，但 `synra-control` 的 `Request`
      >   **没有对应的请求**，所以这条路完全不存在。现在非 cached 扫描会**拒绝并说明原因**
      >   （`Unsupported`，reason 里同时点名两半）；cached 扫描仍然可用，因为它真的只读已知。
      > * ★ **待办，而且第一版写错了。** 我先写的是“给控制协议加 `Request::Scan`，
      >   daemon 据此设 `on_demand`，shell 转发”—— **shell 根本不会说控制协议**：
      >   `src-tauri` 的 `Cargo.toml` 里没有 `synra-control`，全仓 0 处 `Request::`（只有 `diag.rs:266` 一句注释提到它）。
      >   控制协议是 **CLI 对 daemon** 的，不是 shell 对 daemon 的。
      > * ★ **真正的原因在 [05 §2.1](./05-system-architecture.md)**：设计里桌面端的签名是
      >   `Kernel（Arc<Kernel>）` **同进程**，`DiscoveryService` 和 `TransportRegistry` 都在里面。
      >   shell 现在两者都没有；它手里的 `Host` 能 `listen_on`（`host/kernel.rs:160`）但从没调过。
      >   所以 scan 不是差一个请求，是差半个内核。
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
      39 个名字确实存在（**当时**；现在是 40 个，见上），但只在 `crates/synra-ipc/src/command.rs` 的 `commands!` 表里；
      `src-tauri/src/main.rs` 注册了 13 个（identity 2、devices 4、plugins 3、config 2、diag 2）。
      未实现的 26 个（**当时**；现在是 19 个）逐条列在 `synra-rust-app/src/ipc/registration.test.ts`
      的 `NOT_YET_REGISTERED` 里，**每条都带原因**（区分"等接线"与"决定不做"）。
      ★ 本次新增的是 `diag_logs_tail`：它原本在"未实现"清单里，理由是
> ★ **而第 104 轮又把一条从“未实现”移了出去**：`diag_metrics`。
> 它在 `registration.test.ts` 里的理由是“**`AppState` 后面没有 metrics 沉沿**” —— 这句话是真的，
> 而且是**唯一**的障碍：命令本身只是读一个已经存在的记录器。
> 现在壳拥有自己的 `RingRecorder`（与日志环同一个故事），
> 它同时被 `.with_metrics()` 交给插件集的闸门 —— 所以报告出来的就是闸门真的数过的。
> ★ **而两个测试分别钉住两半**：空壳报告**空列表**而不是报错（“没有”与“查不到”是两件事），
> 而按闸门的方式计一笔后，命令必须看得到它及其**标签** ——
> 一个读另一个空记录器的命令会永远返回空列表，而那是诊断命令最能误导的形状。
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
      所有 DTO 与命令绑定一个都没生成。★ **而 2026-10-18 起"CI 那半"不再是零**：core 与 app 都有了
      `.github/workflows/ci.yml`（core：clippy / test / MSRV / guards / matrix；app：test / build / shapes 与
      `--no-default-features` 的 shell）。**但它跑的是测试与守卫，不是生成物 diff** —— `types-check` 仍然不存在，
      因为没有生成物可 diff。所以这一条的状态从"没有 CI"变成"有 CI，缺的是生成这一步"。
- [ ] `KernelEventEnvelope`（带 `seq`）+ Tauri emit 桥 —— **有了**（2026-10-18）。
      `KernelEventEnvelope` + `SequenceTracker` 是真货（`crates/synra-ipc/src/event.rs`、`:243`）。
      ★ **emit 桥不存在**那句曾经是真的：`src-tauri` 下 0 处 `emit`，envelope 没有生产者，页面画出快照后再不更新。
      ★ **现在有生产者了**：`synra-rust-app/src-tauri/src/events.rs` 订阅 `AppState.events`（内核自己的
      `EventBus`），把每个 delivery 变成 `KernelEventEnvelope` 并 `emit` 到 `kernel-event` —— 与
      `src/ipc/channel.ts` 的 `KERNEL_EVENT_CHANNEL` 同一个字符串。接线时发现两件事：
      * **总线把序号扔掉了**：`emit` 返回它，`Delivery` 却只带事件，于是订阅方无法判断"少了几个"。
        现在 `Delivery` 带 `first_seq`（队列连续，`first_seq + i` 就是第 i 个的序号），`missed` 说明断口在
        `first_seq` 之前 —— 这正是总线一直在数数的理由，只是此前没有交给读者。
      * **发布什么形状**：发的是 **wire** 事件（`synra-ipc::event::KernelEvent`），不是内核内部的
        `synra_core::KernelEvent`。界面是这条流唯一的消费者，用 wire 形状就不需要一层枚举映射。
      ★ **历史，保留下来因为它解释了这条为什么重要**：前端那一半更早一轮就补上了 —— `src/ipc/tauri.ts` 的
      `TauriKernel` 实现 `invoke`/`listen`（用 `@tauri-apps/api` 而不是 `window.__TAURI__` 全局），
      `kernel.ts` 按 `window.__TAURI_INTERNALS__` 选择真内核还是假内核。而 `DevicesPage.vue` 与 `injection.ts`
      真的调用了 `subscribe`，所以当时的后果是：**页面画出快照、然后再也不更新，而页面看起来是正常的。**
      ★ 三个命令开始发布：`plugins_enable` / `plugins_disable` / `plugins_consent_respond`（`PluginUpdated`）
      与 `plugins_install`。测试 `enabling_a_plugin_tells_the_interface` 断言的是**序号来自总线**（第一次是 1）。
      ★ 仍然缺的：后端 `EventDispatcher` 在生产代码里**依然只有测试在 `new` 它**（页面走的是 `kernel.subscribe`），
      以及设备/意图/传输那几类事件还没有发布点 —— 桥接的是通道，不是每一类事实。
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
      ★ **2026-10-18：接上了，于是"只有测试在 new 它"不再成立。** `src/events.ts` 的 `startEvents` 在 `main.ts`
      挂载之后启动：订阅 `KERNEL_EVENT_CHANNEL` → `EventDispatcher.accept` → 设备 store 的 `apply`；
      未知类型走 `refresh`（重读 `devices_snapshot`），失败的重读**不**调用 `snapshotTaken` ——
      把没拿到的快照标记成"已完整"正是这条链要消灭的那种静默过期。
      测试 `src/events.test.ts` 四条：订阅了那个通道、设备卡片进 store、未知类型触发一次重读、`stop()` 真的解绑。
      ★ **仍然缺的：插件/意图/传输 store 还没有 `apply`**，所以那几类事件目前只会触发一次重读 ——
      推送的通道通了，但"每一类事实"还没用完它。
      ★ **2026-10-18（第二轮）：设备侧有了第一个发布点。** `devices_forget` → `DeviceRemoved`：
      忘记一台设备之后，列表上的那张卡片不该等到页面重开才消失。测试 `forgetting_a_device_tells_the_interface`
      断言序号来自总线、且事件点名的是那一台。**扫描**也有了：`devices_scan_start` 在开始时就发布 `ScanUpdated`（结束那半仍是快照 —— 发现数是数据库的回答，不是那个函数的）。仍然没有发布点的是**发现**（`DeviceCardUpdated`，
      `DeviceCardUpdated` 由 discovery worker 产生，而它现在还没有总线）。
- [x] **同意面板（插件页）** —— **做完了**（2026-10-18）。
      `PluginSnapshot.requested` 落地（`crates/synra-ipc/src/dto/plugin.rs`，`#[serde(default)]`），由 app 的
      `snapshot()` 从 manifest 的 `capabilities` 填；TS 侧在 `@synra/protocol` 的 `ui.ts` 同步。
      页面为每个声明渲染一个勾选框（初始态来自 `granted`），提交 → `plugins_consent_respond` → **用返回的卡片替换
      那一条**，不重读：返回的就是内核刚记下的。
      ★ **没有"全部允许"按钮** —— 把逐项同意压缩成一次点击，正是这套能力模型存在理由的反面。
      ★ 同一张卡片上还有：安装（路径）、启用开关（`plugins_enable`/`disable`；成功后由 `PluginUpdated` 事件让列表
      自己更新 —— 开关**不**自己改列表，同一个变化只留一份描述）。
      ★ **2026-10-18：先失败一次，随后做成了，两个教训都记下来。**
      **失败的那次**：装了 `jsdom` + `@vue/test-utils`、加了 `@vitejs/plugin-vue`（不加时报的是
      "content contains invalid JS syntax"，指向模板里的 `{{ }}`，与真实原因无关），前两条测试通过、其余全挂；
      于是把 `ant-design-vue` 整体 stub 成"渲染 slot 的 div"，列表**整个不渲染**：`AList` 的内容来自 `renderItem`
      **作用域插槽**，普通 stub 不调用它。当时判断"需要一套认识插槽的 stub 层"，于是回退。
      **做成的这次**：真组件留着，只补 jsdom 没有的三个 API —— `matchMedia`、`ResizeObserver`、
      `IntersectionObserver`。真正的原因不在 antd，而在一句**问错问题的守卫**：
      `if (!('matchMedia' in window))` —— jsdom 给 `window` 一个**占位**的 `matchMedia`（存在但不是函数），
      所以守卫认为"已经有了"，什么都不装；前两次挂载恰好没走到那条路径，第三次才炸。
      **教训**：一个问错问题的守卫比没有守卫更糟 —— 它让前两条测试通过，并把原因藏起来。
      现在每个测试都无条件安装这些 API（`src/pages/plugin-page.mount.test.ts`）。
      **覆盖**：挂载即读列表、卡片文字到达 DOM、显示 `requested` 而非 `granted`、拒绝渲染 `ErrorNotice` 且 `code` 正确
      （断言**组件 prop** 而不是在文本里找 code）、以及**store 里放进一张卡片屏幕就出现**（页面读 store、`events.ts`
      写 store 的接合点）。**不覆盖**点击：那是 antd 的行为，而点击产生的**参数**由
      `src/pages/plugins/actions.test.ts` 断言。全局环境仍是 `node`，需要 DOM 的文件用
      `// @vitest-environment jsdom` 自己声明。
      ★ **四个页面都挂上了**（`plugin-page`、`devices-page`、`home-page`、`settings-page` 四个 `*.mount.test.ts`）。
      写这四个文件时踩了四个坑，每一个都**表现为"测试写得不对"而不是"代码有问题"** —— 记下来，下一个写挂载测试的人
      不必重踩：
      1. **`matchMedia` 在 jsdom 里是占位而不是函数**。`if (!('matchMedia' in window))` 会认为已经存在、什么都不装，
         于是前几次挂载恰好成功、后面某一次才炸在 `window.matchMedia is not a function`。要**无条件赋值**。
      2. **`useRouter()` 需要真的 router**。空路由表 + `createMemoryHistory` 就够，但少了它会在任何断言之前抛出。
      3. **按命令名作答**，不要一刀切 `{ ok: true, value: [] }`。多调用页面里，返回值形状错了会炸在离出错调用很远的
         地方 —— 设备页第一次就这么错的（`{ self, paired, visible }` 根本不是 `DevicesSnapshot`）。
      4. **断言组件 prop，不要断言文本里的 code**。通知组件渲染的是翻译后的句子；而对 DTO **不存在的字段**断言
         （`INTENT.summary`）会以 `undefined` **通过**。
      仍然**不覆盖**的是点击：那是 antd 的行为，而点击产生的**参数**由 `src/pages/plugins/actions.test.ts` 断言。
- [x] **意图的成功路径有测试了**（2026-10-18 补完，含两次自我纠正）。
      原先只有失败路径：没有引擎时拒绝、`announce` 对面板不认识的 id 静默。要测成功路径就得有 `PluginSet`，而
      `PluginSet::new(backend, host)` 的两个参数都不便宜：
      * `Backend` 的替身是 `synra_plugin_host::engine::scripted::ScriptedBackend::new()`（无参）；
      * **`Host` 没有默认方法**，原来那份"全部拒绝"的实现是 core 的 dispatch 测试私有的 ~170 行。
      **做法**：把那 170 行提到 `synra-plugin-host/src/testing.rs`（`RefusingHost`），**不**放在 `synra-testkit` ——
      testkit 依赖 core，而 core 的测试要用它，那是 Cargo 禁止的环；`synra-plugin-host` 定义 `Host` 且不依赖 core，
      是唯一没有环的位置。core 侧改成 `use synra_plugin_host::testing::RefusingHost as Refusing;`，别名让几十处
      调用一处不改。
      **app 侧不需要"能匹配的插件"**：`AppState::with_plugins` 这个 builder 本来就有，一个**真实但空**的集合就够。
      夹具 `engine_state()` 因此是这三行。
      ★ **一次靠失败学到的东西**：我按"无匹配 ⇒ 报错"写了 `expect_err`，测试失败——因为 `intent_dispatch` 在
      没有任何插件匹配时**仍然会创建意图**（状态是"没有候选"），那是一个不同于"这个构建不能跑插件"的答案。
      测试于是改成断言真正发生的事，并顺带覆盖了 `announce` 在成功路径上的行为。
      ★ **另一个反直觉的教训**：先写断言、让失败告诉你行为是什么，比先假设行为再构造夹具便宜得多 —— 这一条省掉了
      "造一个能匹配的插件"那一整轮。
- [x] **事件 → store 的三个读者**（2026-10-18 落地）。
      `src/events.ts` 是界面里**唯一**订阅 `KERNEL_EVENT_CHANNEL` 的地方：它按序号判定（应用 / 忽略重复 / 重读
      快照），再分给三个 store —— `devices`（`deviceCardUpdated`、`deviceRemoved`）、`plugins`（`pluginUpdated`）、
      `intents`（`intentUpdated`、`intentFinished`、`transferUpdated`、`transferFinished`）。页面是读者：不再自己
      订阅，也不持有副本（`plugins` 页原来既订阅又持有 `ref`，两处都去掉了）。
      **契约**：每个 store 的 `apply(event)` 返回 `boolean` —— `false` 不是"什么都没发生"，而是"我放不下这个"，
      调用方据此重读快照。这让"未知类型"与"不需要处理"在类型上分开，而不是靠调用方猜。
      ★ 只有一条路没有事件：**卸载**。`pluginRemoved` 不存在（不存在的插件没有卡片），所以命令成功之后页面调
      `plugins.forget(id)` 再重读一次 —— 先忘掉，即使随后的读取失败，那张卡片也已经不在了。
      ★ 三处 `applySnapshot` 只用于**首次读取**（`devices_snapshot` / `plugins_list` / `intent_list`），此后只有事件。
      ★ 一条测试因此改了含义：`events.test.ts` 里"未知类型触发重读"曾拿 `pluginUpdated` 当例子，而它现在是已知
      类型；现在用了一个**虚构**的类型名，因为这条测的是"本文件没注册处理器"，是接线的性质而非内核枚举的性质。
      ★ **2026-10-18（补）：意图的两类事件有发布者了。** `intent_dispatch` / `intent_decide` / `intent_cancel`
      成功后，用私有 `announce(state, id)` 从**面板**读回快照再发 `IntentUpdated` —— 从面板读回而不是就地拼，
      因为面板才是 `intent_list` 会回答的东西，拼一份就是同一意图的第二份描述。
      两个细节：决定**被拒绝**时不发（什么都没变，发布未变的状态等于说"发生了"）；面板答不出来时**静默**
      （竞态修不了，而事件不是用来发错误的 —— 少播报一次代价是过期列表，编一次代价是错误列表）。
      **传输的两类仍然没有发布者**：没有传输子系统，`transfer_*` 四条命令也仍未注册。
- [x] **发现（`DeviceCardUpdated`）的发布点 —— 扫描结束时播报卡片**（2026-10-18 落地）。
      位置是 `devices_scan_start`：`discovery.scan(budget)` 返回、`scanning` 清掉之后，读一次 `devices_snapshot`
      并把**每张卡片**逐条 publish 成 `DeviceCardUpdated`。三个理由：
      * 卡片由 `devices_snapshot` 构建，所以事件里的卡片与下一次列表里的卡片是同一个东西，不会漂移；
      * **一台设备一条事件**，而不是一条"列表变了"——新出现、掉线、改名是三个不同的事实，一条"变了"会把它们
        压成一次全量重读；
      * 顺序是"先 `ScanUpdated`（开始），后卡片（结束）"，界面先画进度、再拿结果。
      ★ **没有直接测试**：现有扫描测试都走 `cached_only`，而那条 `return` 在发布之前；要覆盖它需要一个真的
      绑定套接字的扫描，而 `synra-core` 已经测过同一件事。保护来自代码形状（发布用的是 `devices_snapshot`）。
      ★ **worker 的即时播报：做完了**（2026-10-18，按方案 (a)）。形状是：
      `Discovery::start` 多收一个 `Announce = Arc<dyn Fn(&str) + Send + Sync>`，回调里
      `for id in record(&db, candidates) { announce(&id) }`；`shell.rs` 构造闭包，捕获 `db` / `host` / `events`
      三个 `Arc` 克隆，用 `commands::devices::card_for(db, host, id)` 造卡后 `events.emit(DeviceCardUpdated)`。
      **卡片仍然只有一个描述**：`card_for` 复用的是 `devices_snapshot` 用的那两个函数（`card_from_paired` /
      `card_from_seen`），所以方案 (b) 的"两份实现会漂移"没有发生。`card_for` 挂在 `#[cfg(feature = "tauri-shell")]`
      下 —— 只有壳用它，无默认 feature 下它就该不存在。
      ★ **验证**：默认 feature（含 `shell.rs`）与 `--no-default-features`（含测试）**都要跑 `cargo check`**。这一点
      上一轮被我误判成"本机做不到"，记在 [21 §7](./21-testing-strategy.md) 里。
      —— 以下是当初判断"不是一行"的调查记录，保留下来是为了看清最后是怎么绕过它的：
      worker 走 `discovery::start` → `synra_core::worker::run_recording(db, callback)`，回调里只有
      `record(db, candidates)`，**没有总线**。wire 事件里"设备被发现"只有 `DeviceCardUpdated(Box<DeviceCard>)`，
      而 worker 手里只有"刚看到的这一台"。
      **两个形状只能选一个**：(a) 宿主给一个回调 `announce(device_id)`，宿主读库、造卡片、发 wire 事件；
      (b) worker 直接造卡片。代价：卡片构造出现两份，而这张卡片是设备页的全部内容，漂移的后果是同一台设备在
      两个入口长得不一样。倾向前者 —— 卡片的唯一作者仍是 `devices_snapshot`。
      ★ app 的 `EventBus` 装的是 **wire** 事件（`synra_ipc::event::KernelEvent`），不是 core 的 —— 所以
      "worker 发 `DeviceSeen`、桥再翻译"这条路需要**第二条总线**，不推荐。
- [x] **lint 与守卫的分歧**（2026-10-18）。
      `pnpm run lint` 早就在 `package.json` 里，却从未进过 CI；第一次跑是 **17 error / 25 warning**，几乎全在最近几轮
      写的文件里。真正有价值的是里面藏着的一处**修复器与检查器互相打架**：
      `eslint --fix` 的 `object-shorthand` 把 `{ pluginId: pluginId }` 改写成 `{ pluginId }`，而 `check-shapes` 的
      `command-arguments` 只认冒号形式 —— 于是"格式化过的、完全正确的调用"被守卫读成"漏传参数"。
      修的是**守卫**：扫描前先把简写展开（`{ pluginId }` → `{ pluginId: pluginId }`）。第一版还错了 ——
      正则把分隔逗号一起吃掉，`{ pluginId, removeData }` 只展开第一个，守卫把自己刚改写的第二个参数报成缺失；
      改成**前瞻**保留分隔符后才对。
      `lint` 现在在 CI 的 `interface` job 里（Test 与 Build 之间），规则与检查器终于在同一处运行。
      两个教训：**(a)** 一条规则和一个检查器意见不一致，只有它们**在同一处运行**时才看得见；
      **(b)** 自动修复会破坏检查，属于最难察觉的一类缺陷 —— 单独看每个工具都是对的。
- [x] **卸载（插件页 + 命令）** —— **做完了**（2026-10-18），四层各有一条测试。
      决定写在 [14 §2.9](./14-plugin-lifecycle.md)：`PluginFiles` 是与 `Sandbox` 并列的第二个删除所有者，
      只删"某个 id 的 artifact 目录 + `remove_data` 为真时的 data 目录"。
      store 的 `remove_plugin` 在一个事务里删 `plugins` 行与 `plugin_grants` 行；命令按 **先记录、后目录** 的顺序
      —— 目录先删而记录还在，装载器会把不存在的目录报成"插件损坏"；反过来只是残留文件，下次启动会被重新发现。
      ★ 页面上的确认句**必须说出数据去哪了**（"removed and its data stays on disk"），且 `remove_data` 恒为
      `false`：确认说数据会留下，却传 `true`，就是让按钮和它旁边那句话互相矛盾。
      ★ **没有事件**：界面唯一认识的插件事件是 `PluginUpdated`，而它携带一张卡片 —— 已经不存在的插件没有卡片；
      页面在命令返回后自己重读列表。
      `RemoveOptionsDto` / `RemoveReport` 两个 DTO 仍然没有被用到：报告里要的 `bytes_freed` 需要一个测量，
      而这一版只回答"删了/没删"。
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
      `WebviewKind` 只存在于设计文档（[09](./09-plugin-system.md)、本文 §384），代码 0 处。
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
