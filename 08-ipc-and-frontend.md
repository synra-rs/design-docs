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

本机身份维护返回协议层的共享类型，不复制宿主私有 JSON 形状：
`CredentialSnapshot` 区分持久登记、独立的 `CredentialRecovery` 恢复元数据、当前进程加载的身份、待清理数量及重启要求；登记 ready 不表示原生密钥后端当前健康。
登记丢失时 enrollment 为 null，recovery 只提供最后登记的公开指纹与后端，设置页明确提示并要求确认后换钥；它不表示凭据可用或权限恢复。
`CredentialRepairReport` 表示新钥已提交，cleanup 是独立的 `CredentialCleanupReport`，清理失败不能抹掉换钥成功。
设置页确认框捕获当时展示的指纹，刷新不能悄悄更换待确认目标，后端拒绝失效指纹。
成功或失败后都重新读取身份与设备列表，因为失败前也可能已撤销旧授权；读取失败明确显示且可独立重试，
不能因重试刷新而再次换钥。操作结果保留在 store 中跨导航展示，应用重启后以新进程的实际身份重新读取。
状态不是私钥健康探测，不从界面快照产生授权；具体修复契约见 [17 §5](./17-storage-and-config.md)。
应用 Tauri 分发器在调用任何本应用命令之前，以真实 WebView label 选择 Host 或 Plugin 接口并查询共享命令注册表；
仅 main 可进入 Host 命令，其他 WebView 仅能进入已声明的插件桥接命令，之后仍须校验 token、权限及资源归属。
未知本应用命令默认拒绝；判断不使用调用参数中声称的窗口标签，不能仅依赖 main 窗口的插件权限配置或前端隐藏按钮。

配对命令共用 core 的 `PairingService`，任务返回 `synra-protocol::pairing::PairingSnapshot`。
显式导入表示用户批准经可信带外路径获得的目标证书，所选 device id 必须匹配证书；
应用从发现写入的设备表取得路由地址，路由和名称都不能产生授权。
`pairing_status` 返回运行时快照或 null（`Maybe`），快照中的 reply、errorCode、detail 为必有键的可空值。
前端只保存选择和表单输入，任务由共享 store 读取；`pairingUpdated` 触发权威重读，
读取串行合并并丢弃被新事件或命令失效的旧响应，避免旧快照覆盖新任务。
六位码提示、token 应答命令及旧配对事件已删除，无兼容入口。

```rust
// src-tauri/src/commands/mod.rs

// ─────────── identity ───────────
#[tauri::command] async fn identity_get(state: State<'_, AppState>) -> Result<IdentitySnapshot, IpcError>;
#[tauri::command] async fn identity_set_display_name(state: State<'_, AppState>, name: String) -> Result<(), IpcError>;
#[tauri::command] fn identity_credentials(state: State<'_, AppState>) -> Result<CredentialSnapshot, IpcError>;
#[tauri::command] async fn identity_repair(state: State<'_, AppState>, expected_fingerprint: String) -> Result<CredentialRepairReport, IpcError>;
#[tauri::command] async fn identity_cleanup(state: State<'_, AppState>) -> Result<CredentialCleanupReport, IpcError>;

// ─────────── devices ───────────
/// 全量快照：设备卡片列表 + 本机身份 + 扫描状态
#[tauri::command] async fn devices_snapshot(state: State<'_, AppState>) -> Result<DevicesSnapshot, IpcError>;
#[tauri::command] async fn devices_scan_start(state: State<'_, AppState>, opts: ScanOptions) -> Result<ScanId, IpcError>;
#[tauri::command] async fn devices_scan_cancel(state: State<'_, AppState>, scan_id: ScanId) -> Result<(), IpcError>;
#[tauri::command] async fn devices_connect(state: State<'_, AppState>, device_id: DeviceId) -> Result<(), IpcError>;
#[tauri::command] async fn devices_disconnect(state: State<'_, AppState>, device_id: DeviceId) -> Result<(), IpcError>;
#[tauri::command] async fn devices_forget(state: State<'_, AppState>, device_id: DeviceId) -> Result<(), IpcError>;

// ─────────── pairing ───────────
#[tauri::command] fn pairing_invite(state: State<'_, AppState>) -> Result<String, IpcError>;
#[tauri::command] async fn pairing_listen(state: State<'_, AppState>, device_id: String, invitation: String) -> Result<PairingSnapshot, IpcError>;
#[tauri::command] async fn pairing_connect(state: State<'_, AppState>, device_id: String, reply: String) -> Result<PairingSnapshot, IpcError>;
#[tauri::command] fn pairing_status(state: State<'_, AppState>) -> Result<Option<PairingSnapshot>, IpcError>;
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
#[tauri::command] async fn transfer_respond(state: State<'_, AppState>, transfer_id: TransferId, accept: bool) -> Result<(), IpcError>;
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
#[tauri::command] async fn plugins_update(state: State<'_, AppState>, plugin_id: PluginId) -> Result<InstallReport, IpcError>;
/// 重试一个 faulted 插件的装载，**不改任何开关**：它已经 enabled + trusted，失败的是启动。
#[tauri::command] async fn plugins_reload(state: State<'_, AppState>, plugin_id: PluginId) -> Result<PluginSnapshot, IpcError>;
#[tauri::command] async fn plugins_sync_plan(state: State<'_, AppState>, plugin_id: PluginId) -> Result<SyncPlanDto, IpcError>;
#[tauri::command] async fn plugins_sync_run(state: State<'_, AppState>, plan: SyncPlanDto) -> Result<SyncReport, IpcError>;
#[tauri::command] async fn plugins_publishers(state: State<'_, AppState>) -> Result<Vec<PublisherDto>, IpcError>;
#[tauri::command] async fn plugins_publisher_decide(state: State<'_, AppState>, publisher_id: String, trust: String) -> Result<(), IpcError>;
#[tauri::command] async fn plugins_preview(state: State<'_, AppState>, plugin_id: String, version: Option<String>) -> Result<InstallPreviewDto, IpcError>;
#[tauri::command] async fn plugins_audit(state: State<'_, AppState>, plugin_id: PluginId, limit: u32, offset: Option<u32>) -> Result<Vec<AuditEntryDto>, IpcError>;
#[tauri::command] async fn plugins_audit_clear(state: State<'_, AppState>, plugin_id: PluginId) -> Result<(), IpcError>;
#[tauri::command] async fn plugins_request_registry_search(state: State<'_, AppState>, query: String) -> Result<Vec<CatalogItem>, IpcError>;

// ─────────── UI 插件专用（不同于上表，见 §7）───────────
// ★ 实际签名与设计初稿有三处不同，都是被运行时事实逼出来的（2026-10-18）：
//   1. `token` 是 **十进制字符串**：`u128` 超出 JS 数字精度，数字形式会静默丢低位；
//   2. `plugin_ui_ready` 收 **label**：`validate` 要声明者与版本，而这两样只有宿主自己的 `hosted_ui` 知道；
//   3. `plugin_ui_rpc` 用 token 反应 `hosted_ui` 找到那一行，所以页面的声明与宿主的记录必须对上。
#[tauri::command] async fn plugin_ui_rpc(state: State<'_, AppState>, token: String, method: String, args: serde_json::Value) -> Result<serde_json::Value, IpcError>;
#[tauri::command] async fn plugin_ui_ready(state: State<'_, AppState>, label: String) -> Result<(), IpcError>;

// ─────────── config ───────────
#[tauri::command] async fn config_get(state: State<'_, AppState>) -> Result<ConfigSnapshot, IpcError>;
#[tauri::command] async fn config_set(state: State<'_, AppState>, patch: ConfigPatch) -> Result<(), IpcError>;

// ─────────── diagnostics ───────────
#[tauri::command] async fn diag_export(state: State<'_, AppState>, options: DiagExportOptions) -> Result<DiagExportResult, IpcError>;
#[tauri::command] async fn diag_probes(state: State<'_, AppState>) -> Result<ProbeReportDto, IpcError>;
#[tauri::command] async fn diag_metrics(state: State<'_, AppState>) -> Result<MetricsSnapshot, IpcError>;
#[tauri::command] async fn diag_logs_tail(state: State<'_, AppState>, lines: Option<u32>) -> Result<Vec<String>, IpcError>;
/// 删掉会让下次启动进入安全模式的崩溃报告；"忽略"必须是持久的，否则下次启动会再问一遍。
#[tauri::command] async fn diag_crash_dismiss(state: State<'_, AppState>) -> Result<(), IpcError>;
/// 崩溃报告本身，最新在前；设置页的“查看详情”按需读它。
#[tauri::command] async fn diag_crashes(state: State<'_, AppState>) -> Result<Vec<CrashReport>, IpcError>;
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

插件页面的 self.invoke 和经 CallError 返回的直接宿主调用共享错误适配：顶层 code 仍用 IPC 的 ErrorCode，params.reason 保留消息，
details.pluginCode 保留原始插件或宿主调用错误码，details.pluginDetails 承载有值时的 WIT details。
合法 JSON 转为对应 JSON 值；空字符串和非 JSON 文本保留为字符串；缺失则不写 pluginDetails，JSON null 则显式写 null。
详情始终嵌套，不能覆盖顶层 code、retryable 或动作；未知插件码分类为 internal，但原始码仍可诊断，notImplemented 分类为 unsupported。
这套页面规则的验证见[错误详情记录](./history/2026-10-26-error-details.md)。
本地与远程 intent 使用同一插件错误分类；SynraError 的可选 JSON details 可供插件 UI 类型读取，
通用 IPC 错误在 details.context 保留该上下文、details.traceId 保留原操作 trace，不把详情展开为顶层权限或错误分类。
成功 JSON 与错误在重复请求及数据库恢复中的验证见[意图结果记录](./history/2026-10-26-intent-outcomes.md)。

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
      case 'pairingUpdated':        void store.pairing.refresh(kernel); break
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

> ★ **2026-10-18：枚举已经落地，位置与名字都与上面这段初稿略有不同。** 它在
> `synra-ipc/src/plugin_ui.rs`，线名是点分的（`env.get`、`device.list`、`storage.set`…），`parse` 只认 `ALL` 里的
> 名字；上面的变体之外还补了 `DeviceBroadcast`、`DevicePick`、`StorageQuota`、`TransferOnProgress`、`SelfInvoke`、
> `SelfActions`（客户端的 namespace 里有，这份初稿没列全）。是**变体**只表示"可寻址"，不表示"这个构建实现了它"。

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

宿主的 `open_request` 读取 manifest、签发令牌并预留窗口；创建失败或窗口销毁时撤销会话。
完整插件 id 不直接放进 URL authority 或窗口 label：`@scope/name` 会被标准 URL 解析成用户信息和路径。

每个插件用完整 id 的 SHA-256 派生独立 origin，十六进制摘要分成两个 32 字符 DNS label：

```text
synra-plugin://p<digest前32字符>.<digest后32字符>.localhost/<entry>#<defaultPage>
```

`entry` 是 ZIP 内真实文件路径，保留目录和扩展名，并通过 URL 路径分段编码；`defaultPage` 可选，作为 hash 路由。
相对资源以该 HTML 地址解析，不使用文件 stem 或页面 key 代替入口文件。
窗口 label 使用完整摘要前缀和独立文档 id，只包含 Tauri 接受的字符，不包含私有 RPC nonce。
同一有效授权代次复用当前文档；替换后的窗口使用新 label，旧窗口的延迟销毁不会撤销新会话。

插件必须已启用且已同意授权才可打开 UI；创建时从同一数据库读事务取得权限与持久授权代次。
RPC、ready 及资源读取均重检当前代次，停用、授权变更、活动版本切换和同版本 reload 使旧页面失效，
同版本重装也不能恢复旧 nonce。资源请求以框架提供的 WebView label 绑定所属页面。
后台检查请求销毁失效窗口，失败保留记录重试；拒绝旧调用不依赖原生窗口是否已关闭。
只有显式 ready 报告能标记监听器已安装，授权检查、资源读取和清理扫描本身不改变 readiness。
验证记录见[UI 授权代次](./history/2026-10-28-ui-authority.md)。

Windows/Android 显式使用 HTTPS 自定义协议映射，对应
`https://synra-plugin.p<前32字符>.<后32字符>.localhost/<entry>`。
导航校验把原生协议和这个明确选择的映射归一化，只允许同一文档的 fragment 变化，拒绝更换插件、文件、query 或外部地址。
`window.open` 始终拒绝。

Tauri 将带摘要域名的页面视作 remote origin；须为 `plugin-*` 窗口显式配置该 HTTPS origin 的 capability。
原生构建从共享 `CommandRegistry` 生成两组 app-command 权限：主窗口只有 Host 入口，插件页面只有
`plugin_ui_ready` 和 `plugin_ui_rpc`；运行时 invoke handler 仍校验调用方 interface，RPC 再校验会话和具体权限。

**CSP（由自定义协议 handler 注入响应头）**：

```
Content-Security-Policy:
  default-src 'none';
  script-src 'self';
  style-src 'self' 'unsafe-inline';
  img-src 'self' data: blob:;
  font-src 'self';
  connect-src ipc: https://ipc.localhost;
  frame-src 'none';
  object-src 'none';
  base-uri 'none';
  form-action 'none';
```

**仅允许本机 IPC 是关键**：`ipc:` 是原生 WebKit 的 Tauri 传输，HTTPS 地址是这些插件窗口在
Windows/Android 上显式选择的本机协议映射。主窗口使用 `ipc: http://ipc.localhost`。
这不允许普通 HTTP、WebSocket 或插件 origin 的直接连接；真正的网络请求仍走
`synra.network.fetch`（内核代理 + 闸门 + SSRF 防护）。Tauri command ACL、调用方界面和会话权限继续逐层校验。
设置为 `connect-src 'none'` 会连 Tauri 自己的传输一起阻断，并触发带告警的 postMessage 回退。
iframe 若经父窗口转发，可继续使用 `connect-src 'none'`；原生独立 WebView 使用上述本机 IPC 例外。

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

> **2026-10-29：宿主界面与插件功能分离。** 首页移除分享与剪贴板入口，设备页移除直接发送文字和文件，接收文件只显示元数据；
> 保留设备/插件管理、通用任务状态、授权和取消，以及示例插件标识和调用记录。清理对应状态、弹窗、文案和过时测试，
> 四页检查结果及验收证据见[宿主边界记录](./history/2026-10-29-host-ui-boundary.md)，当前规范见 `28 R56`。

> **2026-10-29 续：插件页健康提示和窄屏控件修正。** 预览健康状态对齐内核 `good`，异常仅显示一条本地化配额提示；
> 安装输入在小屏独占一行，卡片操作按钮按需换行。373 项前端测试及构建、契约检查通过，浏览器验证中英文的主内容区均不横向溢出。
> 记录与原生交互验收边界见[插件页复核记录](./history/2026-10-29-plugin-page-refinements.md)。

> **2026-10-29：界面信息层级与中英文切换分批实现。** 说明文字使用更低对比度的共享次级色；
> 设置页折叠高级内容并按需轮询性能数据。语言默认跟随系统，可保存中文或英文，原生托盘、文件通知及插件环境同步采用语言规则。
> 已完成检查与下一轮待处理细节见[本轮记录](./history/2026-10-29-interface-locales.md)，用户要求今日停止，整体目标保持暂停。

> **2026-10-28：主程序改为受视口约束的独立滚动框架。** 侧栏菜单与主内容分别滚动，品牌及设备信息保持可见；
> 窄栏容纳组件原生宽度，小屏底部导航占独立行。真实浏览器尺寸与键盘验证见
> [布局验收记录](./history/2026-10-28-application-layout.md)，约束见 `28 §4 R54`。

> **2026-10-28：Windows 插件窗口与 IPC 策略修复。** Open 改为异步 Tauri 命令，验收入口经过前端实际 IPC；
> 主窗口组件样式与插件本机 IPC 的 CSP 已修正，空配置保留必填数组，本机旧开发库已备份修复。
> 具体原因、测试及真实界面验收边界见[修复记录](./history/2026-10-28-plugin-window-repairs.md)。

> **2026-10-26：插件错误详情贯通 Wasm 与页面 RPC。** 引擎保留可选 details，页面保留原始错误码和详情；
> 生产 self.invoke 经过实际 SQLite、PluginSet 与 Wasm 的回归见[错误详情记录](./history/2026-10-26-error-details.md)。

> **2026-10-26：应用首次配对交互已接入共享受限服务。** 邀请导出、邀请导入与应答展示、应答导入、
> 任务重读和取消经过应用命令与真实 TLS 验证；全局事件流和 store 保留导航期间的任务，
> 设计、门禁证据与平台限制见[本轮记录](./history/2026-10-26-app-first-pairing.md)。

> **2026-10-22：CLI 发布者密钥决定已补齐（19 §8.238）。** `plugin publisher list` 查询已验证签名的发布者，
> `publisher decide ID seen|trusted|revoked --fingerprint FP` 只决定显示的密钥。界面列表与安装后的指纹提示同步传入
> fingerprint；共享内核验证输入，数据库更新原子比较指纹，未知发布者或错误指纹不写入。信任不代替插件权限授权。

> **2026-10-22：CLI 更新入口已补齐。** `plugin registry`、`policy`、`check`、`update` 通过控制 socket 写配置或执行操作，
> 手动检查不自动安装，手动更新也不继承新增权限的旧授权。命令和校验细节见 `19 §8.237`。

> **2026-10-22：daemon 也会后台检查和自动更新。** 两个宿主共用 core 的策略、源设置读取和每日间隔规则。
> daemon 把正负结果存入同一张 `plugin_update_checks`，CLI 的插件列表带更新版本和检查时间；
> app 的自动安装也新增源设置变化与候选版本撤销检查，拒绝发生在停止旧实例之前。见 `19 §8.236`。

> **2026-10-22：更新策略已连接真实安装。** 设置页四档策略写入 `plugins.auto_update`，默认 `notify-only`；
> `auto-minor`/`auto-all` 只安装相同可信签名者且无新增权限的稳定新版本。旧实例迁移、停止与新实例加载复用手动更新流程，
> 版本与权限决定原子提交，拒绝时保留旧版本及提示。实现与验证见 `19 §8.235`。

> **2026-10-22：更新检查结果与时间已持久化。** `plugin_update_checks` 绑定已安装版本，正负结果都记录；
> 失败保留上次答案并在下一轮重试，重启不重置 24 小时间隔。`PluginSnapshot.update` 与 `updateCheckedAtMs`
> 通过 `pluginUpdated` 事件一起更新，插件页直接读共享 store，移除了页面自己的更新表与额外的 mount 读取。
> `plugins_updates` 仍提供只读列表，但页面首屏已由 `plugins_list` 一次带齐。
> `Cancellation.wait` 使用条件变量，可立即结束后台的五分钟等待。实现、测试与剩余边界见 `19 §8.234`。

> ★ **2026-10-18：传输的四条命令已经全部注册**（`transfer_list`、`transfer_cancel`、`transfer_send_file`、
> `transfer_send_text`，壳侧注册总数 29），发送路径端到端可调。明细在 `19 §8.6`：注册表 `Transfers`、
> `announce_transfer`、`snapshot_from`、`address_of`、`send_over`；测试 5 条（未配对拒绝 ×2、死地址不留行、
> 文本拒绝不留文件、内存链路端点）。
> **仍未做**：worker（拨号与循环在命令线程里跑，两次发送会排队）与接收侧（`listener::serve` 要
> `Host`/`PairingDesk`/`Router`，那三样在 daemon 里）。
> ★★★★ **上面两句都已过时（2026-10-18 晚，同一会话的后续）**：
> (1) **发送不再占命令线程** —— 两条命令改成 `async` + 后台线程（`shell.rs` 的 `blocking` 辅助函数），窗口线程不再被占用、
> 两次发送并行，而**返回值仍然 await**，所以拒绝依旧以 `Err` 返回而不是变成事后方事件；
> (2) **接收侧已经实现** —— `src-tauri/src/listener.rs` 的 `spawn(app)` 用 `Host::new` / `listener::bind` /
> `PairingDesk::new` / `Router::new(state.plugins, state.board)` 起线程跑 `serve`，`on_finished` 里对 bundle 调
> `record_sync` —— 所以 `sync_records` 第一次有了生产写入方。`Router` 不是 daemon 那 660 行编排，而是
> `synra_core::dispatch::orchestrate::Router`（两个 `Arc`，壳里本来就有）。
> **壳侧注册总数现在是 30**（多出的是 `plugins_sync_plan`）。仍未做的只有：接收侧的端到端测试（配方在 `19 §8.7`）。
> ★ **2026-10-18 晚：`plugins_sync_plan` 不再是"读一张没人写的表"。** 它现在对每台已配对设备问一次插件清单
> （`_synra.plugin.list.request/response`，答半在 core 的 listener），把答复替换进 `sync_records`，再用答复里的本插件状态
> 喂 `decide`；`online` 因此表示"它答复了"。安装管线也第一次写 `plugin_versions`，否则 `active_plugin_version`
> 在生产里永远是 `None`。细节与四段改动在 `19 §8.8`、`16 §第 178 轮`；运行器随后落地（第 180 轮），插件卡现在也调用了它。
> ★ **2026-10-18 更晚：接收侧不再只有半段。** `record_received` 从 daemon 搬进了 `synra_core::plugin_sync`，
> 壳的 `on_finished` 现在调的是同一个函数：写历史、写对端持有什么、写 `plugins` 行，并**回一个 `SyncOutcome`** ——
> 在此之前壳只写 `sync_records` 且返回 `None`，发起端等不到结果。细节在 `16 §第 179 轮`。
> ★ **而 `plugins_sync_run` 也注册了（壳侧 34 条）。** 它接一份计划，对每台缺这个插件的设备拨号、`send_bundle`、
> 读回 `SyncResult`，再用 `verdict` 决定 `sync_records` 能不能写：没确认、被拒绝、或说装了却不给版本，都不写。
> 为此 `SynraDb::active_plugin` 多返回了 artifact 的文件名。细节在 `16 §第 180 轮`、`19 §8.8`。
> **同步链仍未完成的一件事是包格式**：接收端写行，但收到的是单个 wasm，没有 manifest 与目录可以落到 `plugins/<id>/`。
> ★ **worker 的第一步被一个细节挡住（2026-10-18）**：把两条发送命令改成 `async` + `spawn_blocking` 是最省力的修法，
> 但那要求把 `AppState` move 进去 —— 而 `AppState` **只有 `#[derive(Debug)]`，没有 `Clone`**（`state.rs:39`），
> `tauri::State` 又借不着 `'static`。两条路：让 `AppState` 变成可 `Clone`（字段大多已是 `Arc`/`Mutex`，
> 但要逐个确认 `SynraDb`/`Paths` 是否可克隆），或者 `manage(Arc<AppState>)` 并让每条命令先 `Arc::clone`。
> **后者更稳**：它不要求每个字段都可克隆，只把已有的那一个 `Arc` 交给闭包。
> ★ **界面语义的决定（2026-10-18，不需要改契约）**：有人会问"改成后台后，拨号失败还怎么告诉用户"。
> 答案已经在 DTO 里 —— `TransferSnapshot` 有 **`abort_reason: Option<String>`** 与 `state`。所以后台 worker 的失败
> 走 `announce_transfer(state="failed", abort_reason=Some(...))` + `TransferFinished`，前端的 `intents` store 本来就在
> 消费这两个事件。**命令的返回值因此只承诺"已排队"，而不是"已送达"** —— 那是后台化的真正代价，也是它必须写进
> 命令文档注释的一句话（今天的注释说"阻塞"；改后台时那里要改成"排队"）。

> ★ **2026-10-19：桌面壳第一次真正装载了插件，而这条路上有两个"测试全绿"的假象。**
> ① `with_plugins_or_refuse` 建了一个**空的** `PluginSet` 就交给窗口：插件被列出、能被启用、**永远不可调用**，
>    所以 `intent_dispatch` 匹配不到任何东西、卡片永远是 `unavailable`。
> ② 装载器按 `plugins_dir.join(plugin_id)` 找文件，而安装器写的是 `plugins/acme+hello`（`sanitise_component`）——
>    两个宿主的夹具都用手写的两层路径，于是**装得上、列得出、加载不了**三个事实同时成立而没有一条测试发现。
>    守护进程的目录扫描有同样的问题，并且它用目录名当 id；现在它从 manifest 的 `id` 取标识并要求目录名与
>    `sanitise_component(id)` 一致，不一致就报出该用的名字。
> 装载逻辑本身搬进了 `synra_core::plugin`（`load_installed` / `load_plugin` / `LoadedSummary`），
> 两个宿主共用一份；壳在 `app_state` 里装 panic hook、取 `StartupVerdict`，safe mode 时一个都不装。
> **用户看得到原因**：`PluginSnapshot.stateReason` 是新增字段（Rust DTO → `@synra/protocol` 镜像 → 卡片上的告警），
> 所以"崩溃 3 次所以这次没启动任何插件"与"这个构建没有引擎"不再都显示成 `unavailable`。
> 启用 / 同意 / 更新之后**就地装载**，停用 / 卸载之后**就地卸载**——开关不再需要一次重启才生效。
> 装载现在也真的调用插件的 `lifecycle.activate`，卸载前调用 `deactivate`：失败时插件被移出集合并把原因写进摘要，
> 所以"编译成功"不再等于"可路由"。摘要里有拒绝的卡片状态是 **`faulted`**（页面本来就有这条红色分支，此前从不触发），
> 没有拒绝的才是 `unavailable`——"插件坏了"与"这个构建没有引擎"不再是同一个词。
> 目录发现（`record_discovered_plugins`）也搬进了 `synra_core::plugin`，壳在建引擎之前扫自己的 `plugins/`：
> 一个作者拷进窗口数据目录的插件不再是"窗口看不见、守护进程看得见"。发现写下的行仍是 `enabled = false,
> trusted = false`——发现是可见性，不是批准。
> 壳侧测试 +5（启动装载、安全模式卡片、失败原因、开关/同意/卸载、目录发现），core 9 条 loader 测试，daemon 的同一批测试改到安装器的真实路径上。
>
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
- [ ] ~40 个 Tauri command（见 §2.2）—— **45/47**（2026-10-19；本轮把 `diag_probes` 接上）。
      ★ `plugins_install` 是第 22 条，它进得来是因为**安装管线搬进了内核**：`synra_core::install` 现在是
      daemon 与 Tauri 壳共用的那一份（校验顺序、staging、回滚、写行），两个宿主各自只提供"这台机器能做什么"
      （`HostFacts`）。在此之前，安装能力只有 CLI 有，而界面的插件页连"装一个"都无处可点。
      它同时是 `InstallSourceDto` 的第一个消费者：`Path` 走管线，`Registry` 被**按名字**拒
      （`Unsupported`，理由是没有取件客户端）—— 而不是报一个从未发生过的网络错误。
      ★ **2026-10-19：壳的 registry 安装早就有，只是这一行写的是旧状态。** `plugins_install` 把 `Registry` 交给
      `commands/registry.rs`：packument 或 static index → 镜像候选 → 取 tarball → **现在先比对 `dist.integrity`**（没有
      integrity 就拒绝）→ `extract_tgz` → 同一个安装管线。daemon 侧另有 §8.122 的 npm packument 管线（CLI 用），两条实现
      有重复，统一到核心层是下一步。
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
      > * ★ **而它现在有调用方了（2026-10-19）**：插件卡的 “Recent calls” 折叠区在展开时读这条记录，拒绝行高亮并带
      >   `reason`；“没有记录”与“读不到”分别是空态与 `ErrorNotice`。顺序由表的查询保证（`at_ms DESC, id DESC`，
      >   新→旧），`pages/plugins/actions.ts` 不再反转。
      > * ★ **清空也接上了**：`plugins_audit_clear`（`Unit`）同时删 `plugin_audit` 的行与 `PluginSet` 环里的记录，
      >   只清一个插件；界面上它只在那一节有行时出现，且不弹确认。
      > * ★ **读的也是表了**（同日更晚）：`plugins_audit` 走 `audit_trail_for(plugin, offset, limit)` 而不是环，所以重
      >   启之后的历史仍然在；没有引擎的构建也能答。`offset` 是**可选**参数（第一次请求不带它），"查看完整历史"因此是
      >   沿同一索引的下一页而不是重新查询。命名的解析（`AuditOutcome::from_stable_name`）放在 `synra-observe`，
      >   表的拼写因此只有一份，未知名字被报成拒绝而不是猜成“allowed”。
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
      > ★ **2026-10-19：这两个也做完了，上面那句假话被删掉了。** `plugin install` 已有 `Request::PluginInstall`；本轮新增
      > `Request::PluginRemove { plugin_id, remove_data }`、`Kernel::plugin_remove` 与守护进程实现（先量字节、删行、删
      > artifact，`--data` 才删 sandbox），CLI 的 `plugin remove <id> [--data]` 真的发请求，prose 输出一句话
      > “removed …; N bytes freed; its data was kept/deleted”。`dispatch` 里那个兜底分支现在只覆盖“未来新增、还没有 request
      > 的 plugin action”，并如实说这一点。
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
- [ ] **插件 UI 宿主的第一片：`plugin_ui_ready` 在没有 token 时能说什么**（2026-10-18 查清，留给下一轮的决定）。
      已确认的事实：`synra-plugin-ui` 的 `session` 模块**是完整的**（`UiSessionRegistry::{issue_with, issue_placeholder,
      revoke, revoke_plugin, revoke_all, validate}`、`UiSessionToken`、`SESSION_LIFETIME_MS`、`ValidatedSession`），
      所以"令牌怎么签发/校验"不需要写。缺的只有两件：
      * `AppState` 里没有 `ui_sessions` 字段（`synra-plugin-ui` 已是依赖）；
      * `plugin_ui_ready` / `plugin_ui_rpc` 两条命令没注册。
      **而真正的阻塞在注入层**：`src/ipc/injection/index.ts` 的 `ready()` 调用 `plugin_ui_ready` 时**不带任何参数**，
      因为 `InjectionContext` 里根本没有 token/nonce 字段。所以现在注册这条命令，它只能：
      (a) 接受一个**可选** token，没有就拒绝（`InvalidArgument`），等于"注册了一条永远拒绝的命令"；
      (b) 无参记录"某个插件 UI 说它准备好了"，但那是一个**没有身份的声明**，等于把校验推迟到 `plugin_ui_rpc`；
      (c) 连同注入层一起做：`initialization_script` 把 token 注入到 `InjectionContext`（`tauri-shell` 下，只能在默认
          feature 的 `cargo check` 里验证编译）。
      **倾向 (c)**：它才不会留下一条半功能命令；代价是这一轮改动的可验证性从"完全"降到"编译 + 单元测试"。
      不管选哪个，(a)/(b) 都必须在命令的文档注释里写清"还没有 token"这件事，否则它就是一条看起来正常的假命令。
      ★ **2026-10-18 晚些时候又看清楚一层，方案 (c) 本身也不够**：`UiSessionRegistry::validate` 的签名是
      `validate(nonce, claimed_plugin: &PluginId, running_version: &str, now_ms)` —— 它要**声明者**和**运行版本**，
      两者都是宿主侧的事实，token 里没有；而 `initialization_script` 只能把 token 交进页面，交不了"这个 WebView
      属于谁"。所以第一片不是"加一条命令"，而是**宿主记住它给哪个插件开了哪个窗口**：
      `AppState` 里一张 `hosted_ui: HashMap<label, (plugin_id, version, nonce)>`，由**创建 WebView 的那段代码**写入；
      之后 `plugin_ui_ready(label)` 才能查表、`validate`、记录就绪。
      这张表在没有 WebView 创建代码之前只能是空的，而"表永远是空的"命令和 (a) 一样是假的 —— 所以这一片必须与
      **WebView 创建**一起做，成为一轮完整的工作，而不是两半。
      ★ **2026-10-18：中间那两块已经落地，注册与窗口创建仍然待做。**
      已完成（`synra-app` 已推送）：`AppState.ui_sessions` 与 `AppState.hosted_ui`（`HostedUi { plugin_id, version,
      nonce, ready }`）、`commands::plugin_ui::plugin_ui_ready(state, label)`，以及三条测试（未登记标签拒绝 /
      令牌不认识拒绝且不标记就绪 / 有效会话标记就绪）。
      两处刻意：两个拒绝**用同一个错误码**（区分它们等于让页面能枚举哪些标签存在）；先复制三个字段、**释放表锁**再
      `validate`（持两把锁而第三方可能反序获取就是造死锁 —— clippy 的 `significant_drop_tightening` 两次指出来）。
      ★ **为什么还没注册 `plugin_ui_ready`**：`src/ipc/injection/index.ts` 的 `ready()` 现在**不带参数**，注册后
      `shapes` 的 `command-arguments` 会立刻报"缺 label"，而那正是对的 —— 页面的 label 只能由创建它的窗口知道。
      所以注册要等窗口创建，两者是同一件事。下一步：`WebviewWindowBuilder`（`synra-plugin://` + CSP +
      `initialization_script` 注入 token 与 label），然后 wrapper + `generate_handler!` + 清单移动一起提交。
      ★ **2026-10-18 晚些：窗口与策略都做完了，剩下的是"文件从哪里来"。**
      已落地并推送（`synra-app`）：`open_request`（读 manifest、申请令牌、写 `hosted_ui`、算 `synra-plugin://<id>/<page>`）、
      `initialization_script(label, token)`（token 是**字符串**，因为 `u128` 超出 JS 数字精度）、`ui_file(root, path)`
      （`check_path` + `is_within` 双重守卫，`../identity.toml` 有测试拒绝）、`split_ui_url`（按**宿主打开过的 id**
      最长前缀切分 —— URL 解析器会把 `@acme/hello` 切成 host `@acme`）、`plugin_ui_open` 命令并**注册**（契约先加了
      `"plugin_ui_open" => PluginUi, Host, Unit, true`，插件接口不可达）。
      **缺的是一件设计决定，不是代码**：`synra-plugin://` handler 要读的是**解压后的 UI bundle**，而
      `synra-plugin-ui` 的 `artifact/expand.rs` 只提供**限额与条目**（`ArchiveLimits`、`expands_too_far`），没有一个
      把 zip 落到 `plugins/<id>/ui/` 的东西 —— 也就是说**安装时没有解压 UI 包**。所以 handler 的落点取决于三选一：
      (a) 安装时解压（`synra_core::install` 里加一步，与 wasm artifact 同处），目录 `plugins/<id>/ui/`；
      (b) 首次打开时解压（懒，但要考虑并发与失败重试）；
      (c) 直接从 zip 读（每次请求解一个条目，I/O 换实现简单）。
      **倾向 (a)**：它与现有安装管线的失败语义一致（装不上就是装不上），而且 `ui_file` 现在就能直接吃那个目录。
      ★ **2026-10-18 再查一步，代价变清楚了**：`synra-plugin-ui` 的 `Cargo.toml` **没有 `zip` 依赖**，
      `expand.rs` 里也没有任何 `zip::` 调用 —— 它解析条目是手写的，只做**限额判定**，不做落盘。所以 (a) 的第一步不是
      "加一步解压"，而是**先引入一个解压实现**（`zip` crate，或者把现在手写的条目解析扩成能写文件），并给它配上
      zip-slip 防护（条目名同样要过 `check_path`，落盘路径同样要 `is_within`）。这是一轮的工作量，不是十行。
      `plugin_ui_ready` 那一条已经完成并注册（2026-10-18），`shapes` 的"调用未注册命令"从 1 变成 **0**。
      剩下真正缺的只有一件事：**页面文件从哪里来**。
      ★ **责任归属是明确的，不需要再决定**：`01` 把产物定义为 `plugin.wasm`（headless）与 **`ui.zip`**（UI）；
      `06` 的第 295 行把 `synra-plugin-ui` 描述为"UI 插件 artifact 的**解包/校验/目录管理**、提供 UI 加载 URL"。
      所以解压实现就该落在 `synra-plugin-ui`，而"放在哪里"（`plugins/<id>/ui/`）与"谁触发"（安装管线）只是实现细节。
      缺的确实只是一个 zip 读取器。
      ★ **2026-10-18 晚：读取器写完了，接线卡在一个回滚细节上。**
      `synra-plugin-ui` 现在有 `zip` 依赖（`default-features = false` + 只开 `deflate` —— 别的后端都是会跑在攻击者输入上
      的代码），以及 `artifact::unpack(archive, dest, limits)` 与 `bundle_dir(plugin_dir)`：
      条目名先过 `check_path`、join 后再查 `starts_with(dest)`、单文件/累计/条目数三重限额。三条测试（正常解压、
      `../identity.toml` 被拒且外面什么都没写、超限被拒）。MSRV 1.88 上 `cargo check` 通过，`guards` 的 MSRV 项也绿。
      **接进 `synra_core::install` 的真正阻塞**：安装管线失败时靠 `discard(&staging, &files)` 回滚，而它是**按文件列表**
      删除的；解压出来的 `ui/` 是一棵**目录树**，`discard` 不认它，而 `remove_dir_all` 是守卫只允许沙箱调用的操作。
      所以要么 `unpack` 返回它写了哪些路径（让 `discard` 逐文件删），要么给 `discard` 一个"删除自己刚建的目录"的
      受控入口。**这是安装器的回滚语义问题，不是十行接线** —— 决定了再动手，否则失败安装会留下一棵无主目录树。
- [x] **事件 → store 的四个读者**（2026-10-18 落地，第四个是配置）。
      ★ **第四个 store：`settings`。** 内核现在发布 `ConfigUpdated(Box<ConfigSnapshot>)`（整份快照，不是补丁 ——
      补丁是**写入方**的输入，不是结果的描述），前端 `events.ts` 把它交给 `useSettingsStore`，设置页是它的读者。
      不注册的话，一条配置事件会被当成"未知类型"，触发一次**与配置无关的设备重读**。
      ★ **接这个 store 时撞出一个真实缺陷，值得单独记下**：`SettingsPage.vue` 原来把 `config_get` 的返回值当作
      键值表（`config.value = configAnswer.value`），而它实际是 `{ values, overridden }`。于是 `config['theme']`
      **永远是 `undefined`**，两个偏好项在界面上从来只显示"未设置"。**类型标注是错的**（写成
      `Record<string, unknown>`）—— 一个错误的类型断言让 `vue-tsc` 通过，并且把一整块功能藏了十几轮。
      教训：**对着 IPC 边界手写的类型标注，和守卫里的"手抄一份 DTO"是同一种漂移**；`ConfigSnapshot` 现在来自
      `@synra/protocol`，错形状会在编译期失败。
      ★ **这个缺陷有两层，第二层被第一层遮住**：形状修好之后，页面仍然读不到值——它的键名是 `theme`，而内核用点分
      路径 `ui.theme`（`messages/settings.ts` 里甚至写着"页面的键名不是内核的"，但页面照旧用短名）。现在有
      `configKey(key)` 一处映射，并有一条断言"`ui.theme: dark` 会显示成 dark"。
      **单个错误的类型断言可以同时掩盖两层缺陷** —— 它让编译器替一个错误的世界观背书，而两处错误都在它后面。
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
- [x] `PluginUiOp` 枚举 + 方法白名单 + `plugin_ui_rpc` —— **第一片做完了**（2026-10-18）。
      枚举在 `synra-ipc/src/plugin_ui.rs`：`ALL` + `as_str` + `parse`，线名是点分的（`device.list`），
      而设计里明确**不允许**的方法（`platform.simulateKeys`、`fs.read/write`、`plugins_install`、`plugin_consent`、
      任何 `devices_*`、`config.set`）**不是变体** —— 所以"不存在"与"不允许"给同一个答案，页面没法用它枚举宿主能力。
      `plugin_ui_rpc(token, method, args)` 用 token 反查 `hosted_ui`，校验会话，再分派；
      **`device.list` 是第一个有实现的方法**（四个字段：`deviceId`、`displayName`、`platform`、`online`），
      其余在表面上但返回 `Unsupported` —— 一个空列表或一个假答案会让作者按不存在的功能写代码。
      ★ **`storage.*` 也接上了**（同日更晚）：`get/set/delete/keys/quota` 走同一通道，grant 用会话的
      `CapabilitySet` 检查，配额用 `PluginSet::quota`、写入按 `used - replaced + new` 判断，超限是 `PayloadTooLarge`。
      门面编码为 JSON，读到 headless 半的裸字符串则原样返回。
      ★ **桌面动作也接上了**：`clipboard.read/write` 与 `platform.openUrl/notify`；`CommandPlatform` 提到
      `AppState.platform` 与插件宿主共享（通知限流是每实例的）。同时修正了注入面里过时的能力名
      （`clipboard_read` → `platform:read-clipboard`）。
      ★ **出网也接上了**：`network.fetch` 走 `PluginSet::host` 的 `fetch`（与 Wasm 半同一个客户端与 in-flight
      限制）；`synra-fetch::Response` 补了 `statusText/url/redirected`，门面把纯文本 body 变成互斥的
      `text/json/bytes`。`timeoutMs` 与字节 body 在门面与 RPC 两处拒绝。
      ★ **插件卡的"打开"入口接上了**：`PluginSnapshot.hasUi` 读 manifest 的 `runtime.ui`，与 `hybrid` 分开
      （只有 UI 的插件有页面、没有 headless 半）；按钮只在 `hasUi` 时出现，点击调 `plugin_ui_open`。
      `check-shapes` 的 `command-arguments` 也修了：`app: tauri::AppHandle` 与 `state` 一样是宏注入的，不该被当成必填键。
      ★ **`self.actions` / `self.invoke` 接上了**：先有 `Host::action_declare`（写 `plugin_actions`，schema 必须是
      JSON）与 `action_list`，RPC 再按"action 属于本插件"检查后以 `Work::InvokeAction` 调 `PluginSet::call`；
      失败码保留插件自己的，超时是 `PluginTimeout`，其余是 `PluginTrapped`。`command-names` 守卫也修了：只认
      `kernel` / `client` / `bridge` 的 `call`/`invoke`，不再把 `self.invoke('clear')` 当成命令。
      ★ **导航范围限制做了**：`plugin_ui_open` 装 `on_navigation`，只允许打开的页面本身（含 `#` 片段），
      `window.open` 一律 `Deny`。理由：初始化脚本挂在 WebView 上，跳到同 bundle 的另一页会保留本页的 token 与授权。
      ★ **两处顺带修掉的、比枚举更重要的事**：
      1. `open_request` 原来用 `issue_placeholder` 签令牌，而它是**计数器**（注册表自己的文档写着 not for production）——
         一个页面能猜到另一个页面的令牌。现在用 `uuid::Uuid::new_v4().as_u128()`（122 位随机）；
      2. 会话的 grants 原来是 `CapabilitySet::new()`（空集），于是 `env.can` 对用户批准过的能力也答 false。
         现在从 `plugin_grants` 里读 `granted = 1` 的行；一个解析不了的 grant 直接拒绝开页，而不是拿一个猜出来的集合。
      ★ 前端同时改了两处：`InjectionContext` 多一个 `token`（十进制字符串），`device.list` 不再调 **Host** 命令
      `devices_snapshot`，而是走 `plugin_ui_rpc`（`method: 'device.list'`）—— 这是 `Interface::allows` 那条分离规则
      第一次被真正遵守，此前注入面自己越过了它。
      ★ **注入也接上了（同日更晚）**：`src/ipc/injection/bootstrap.ts` 是第二个 Vite 构建入口
      （`pnpm run build:facade`，配置 `vite.inject.config.ts`），产出一份 IIFE `src-tauri/inject/facade.js`（约 8.5 KB）。
      壳 `include_str!` 它，`initialization_script` 生成 `(function(seed){ <facade>; SynraInjection.boot(seed) })(<seed>)`。
      seed 的字段由宿主在页面加载前决定（含 `granted` / `denied`、locale、dark、`sdk.preferred`、platform、hostKind、
      appVersion）。★ **seed 是参数而不是全局**：`window.__SYNRA_PLUGIN__` 会被页面引入的任意脚本读到，而设计文档
      要求 token 在闭包里；作为参数它只活在构造门面的那次调用中。
      **生成物为什么提交**：Rust 构建不依赖 Node；`scripts/check-shapes.ts` 的 `facade-fresh` 用 Vite 的 `build()`
      在临时目录重建并逐字节比较（CRLF 归一化），所以"提交了过期产物"是一个红门而不是一次事故。
      **仍然缺的**：分块/流式大文件通道、接收端的接受/拒绝 UI。

      ★ **入站传输的进度也接上了**：`listener::serve_with` 用 `ProgressCallback` 上报接受与每个块，壳把接收中的传输写进
      同一张 `Transfers` 表；`FinishedTransfer.transfer_id` 让完成时能关掉那行（announce `completed` → `TransferFinished`）。

      ★ **`transfer.*` 第一版**：base64 走 `plugin_ui_rpc`（不新增 Tauri 命令），内核解码到临时文件后复用
      `transfers::transfer_send_file`；上限 4 MiB，`send` 在传输结束后返回，门面缓存最终状态供晚到的
      `onProgress` 回放。理由与限制见 19 §8.10。

      ★ **`env.platformSupports` 接上了真实答案**：seed 增加 `platformCapabilities`，由
      `CommandPlatform::supported_actions()` 生成（`simulate_keys` 只在策略允许时出现），门面同步查表。
      ★ **剪贴板能力拼写统一了**：以生成契约为准（`platform:read-clipboard`/`write-clipboard`），
      `PlatformAction::capability` 与 `admission` 改用平铺名；`canonical_spelling` 在 manifest 读取与 store v11
      grant 迁移两处把旧拼写改成正名（不在匹配器里教第二遍）。
- [x] **协议 handler 做了，CSP 受框架限制**（2026-10-18）。
      `builder()` 里 `register_uri_scheme_protocol("synra-plugin", …)`（Tauri 2.12 的方法名是
      `register_uri_scheme_protocol`，handler 收的是 `UriSchemeContext` 而不是 `AppHandle` —— 猜一次就错，读
      `tauri-2.12.1/src/app.rs:2267` 一次就对），handler 是 `commands::plugin_ui::serve`：
      已知 id 取自 `hosted_ui`（页面只能点名宿主开过窗口的插件）→ `split_ui_url` → `ui_file` →
      `200`（带 content-type）/ **`403`**（路径规则拒绝）/ **`404`**（包缺文件）。
      ★ **2026-10-28 修正早期的“继承”推断**：Tauri 的全局 CSP 用于主程序资产，自定义协议须为自己的响应附上策略。
      主窗口只允许 `ipc: http://ipc.localhost` 连接；协议 handler 的 200 响应带 `PLUGIN_CSP`，
      插件连接只允许 `ipc: https://ipc.localhost`，完整规则见 §7.3。普通网络直连仍被拒绝。
      但 `'self'` 在 `synra-plugin://` 这个 origin 下具体意味着什么，只能由一次真实运行确认 —— 这是本仓库唯一无法用
      `cargo check` 回答的问题，记在这里而不是假装它已解决。
- [x] 插件 WebView 初始化脚本（token 闭包注入）—— **部分做了**（2026-10-18）：
      `commands::plugin_ui::initialization_script(label, token)` 生成脚本、`open_request` 组装、`plugin_ui_open` 建窗口；
      **仍缺**协议 handler（`register_uri_scheme`）与 CSP，所以页面还打不开。
      不是"没实现闭包"，是还没有可以闭包的东西。
- [x] 导航范围限制（`on_navigation`）—— **做了**：`plugin_ui::stays_on_page(candidate, opened)` 只允许打开时的那一页
      （忽略 `#` 之后的片段）；另一页、另一插件 origin、`https`/`file`/`data` 都被拒（测试覆盖前缀陷阱与外部 scheme）。
      ★ **窗口关闭即会话结束**：`on_window_event` 的 `Destroyed` 调 `plugin_ui::forget_window`，删除 `hosted_ui` 条目并
      `revoke` 对应 nonce；`CloseRequested` 可能被取消，所以选 `Destroyed`。此前 `hosted_ui` 只增不减，一个关掉的页面仍能
      通过 `plugin_ui_ready` 与协议 handler 得到回答。
- [x] **一个插件一个页面**：`plugin_ui::window_label(id)` 从 id 派生窗口 label（`plugin-@acme__hello`）；`plugin_ui_open`
      先找已有窗口，`show` + `set_focus` 后返回，不再建第二个窗口、也不再发第二个令牌。为此 `tauri` 启用 `unstable`
      feature —— `Manager::get_window` 在 2.12 里就在这个 feature 后面。
      ★ **检查与占位在同一把锁下**：`open_request` 返回 `OpenDecision::{Open, AlreadyOpen}`，在 `hosted_ui` 的锁里插入；
      若插入发现已有条目，把旧条目放回并 `revoke` 这次多发的 nonce，返回 `AlreadyOpen`。建窗失败时 `plugin_ui_open` 调
      `forget_window`，否则一次失败会让插件永远“已打开”却没有窗口可聚焦。
- [ ] 前端 stores：`ui`/`devices`/`pairing`/`plugins`/`intent`/`transfers`/`config`/`diagnostics` —— **1/8**。
      只有 `src/stores/devices.ts`，其余七个没有文件。
      ★ 现存这一个确实是忠实的只读镜像：`:70` 整体替换快照、`:82/:94/:99` 只记录、
      `:133` 的 `switch` 只认三个事件且 `default` 返回 `false`，不含业务规则。
- [ ] WebView 引擎兼容矩阵（WebView2 / WKWebView / WebKitGTK / Android / iOS）+ 探测 —— **探测做了，矩阵没做**。
      2026-10-22：`env.webview()` 落地（`@synra/client` 的 `WebviewKind`/`WebviewInfo` + 注入门面的 `detectWebview`）。
      引擎由**平台**决定（Windows→WebView2、macOS/iOS→WKWebView、Linux→WebKitGTK、Android→系统 WebView），版本从 UA
      读，读不到就是 `null`——WebKitGTK 的 `AppleWebKit/…` 是 WebKit 的版本而不是 GTK 引擎的，拿它冒充引擎版本会让
      最低版本判断悄悄错掉。测试覆盖六种 UA 与一个未知平台。
      **仍然缺的是矩阵本身**：CI 还没有在 WebView2 + WebKitGTK（Docker）+ WKWebView（macOS runner）上跑视觉/功能回归。
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
