# 14 · 插件生命周期

> 本文定义插件从"用户输入一个来源"到"被卸载"的完整状态机。

---

## 1. 完整状态机

```text
                              ┌─────────────┐
                              │  Available  │  ← 在 Registry 里可见（未安装）
                              └──────┬──────┘
                                     │ install(source)
                              ┌──────▼──────┐
                              │  Resolving  │  解析来源（registry / git / url / local）
                              └──────┬──────┘
                     fail ──────────┤
                                     │ resolved { artifact_url, manifest_preview, signature }
                              ┌──────▼──────┐
                              │ Downloading │  下载 artifact（带进度、可取消、可断点）
                              └──────┬──────┘
                     fail ──────────┤
                                     │ downloaded { bytes, sha256 }
                              ┌──────▼──────┐
                              │  Verifying  │  签名 + 摘要 + manifest schema + WIT 兼容 + 依赖图
                              └──────┬──────┘
                     fail ──────────┤
                                     │ verified
                              ┌──────▼──────┐
                              │  Awaiting   │  等待用户确认权限（新装或权限变更时）
                              │  Consent    │
                              └──────┬──────┘
                     reject ────────┤
                                     │ consented { grant }
                              ┌──────▼──────┐
                              │  Installing │  落盘（原子）+ 写 store + 可选 AOT 预编译
                              └──────┬──────┘
                     fail ──────────┤
                                     │ installed
                              ┌──────▼──────┐
                              │  Installed  │  ← 已安装，未启用
                              └──────┬──────┘
                                     │ enable
                              ┌──────▼──────┐
                              │ Activating  │  装载 wasm/UI + 校验有效能力 + 调 activate()
                              └──────┬──────┘
                     fail ──────────┤
                                     │ activated
                              ┌──────▼──────┐
                              │   Active    │  ← 插件可用
                              └──┬───────┬──┘
                                 │       │
                    disable ─────┘       └───── fault（配额/崩溃/健康度）
                              ┌──────▼──────┐        ┌─────────────┐
                              │ Deactivating│        │   Faulted   │
                              │ (调 deactivate)      └──────┬──────┘
                              └──────┬──────┘               │
                                     │                      │ 用户可重试 / 停用
                              ┌──────▼──────┐               │
                              │  Installed  │ ◀─────────────┘
                              └──────┬──────┘
                                     │ uninstall
                              ┌──────▼──────┐
                              │ Removing    │  调 deactivate（若 active）+ 删 artifact + 清 store
                              └──────┬──────┘
                                     │
                              ┌──────▼──────┐
                              │  Available  │
                              └─────────────┘

任意状态 ──cancel/timeout/error──▶ Failed { stage, error, retryable }
```

### 1.1 Rust 状态类型

```rust
// crates/synra-plugin-host/src/lifecycle.rs

/// 生命周期中的插件。每个阶段是不同的类型 → 非法转移无法编译。
pub enum PluginSlot {
    Available(AvailablePlugin),
    Resolving(ResolvingPlugin),
    Downloading(DownloadingPlugin),
    Verifying(VerifyingPlugin),
    AwaitingConsent(AwaitingConsentPlugin),
    Installing(InstallingPlugin),
    Installed(InstalledPlugin),
    Activating(ActivatingPlugin),
    Active(ActivePlugin),
    Deactivating(DeactivatingPlugin),
    Faulted(FaultedPlugin),
    Removing(RemovingPlugin),
    Failed(FailedPlugin),
}

impl PluginSlot {
    /// 只读快照（供 UI）
    pub fn snapshot(&self) -> PluginSnapshot;
    /// 该状态是否可被用户取消
    pub fn is_cancellable(&self) -> bool;
}

/// 失稳状态必须携带足够信息，以便重试或诊断
pub struct FailedPlugin {
    pub plugin_id: Option<PluginId>,        // 可能在解析出 id 之前就失败
    pub source: InstallSource,
    pub stage: FailStage,
    pub error: ErrorReport,
    pub retryable: bool,
    pub attempts: u32,
    pub last_failure_at: Instant,
}
```

> **关键**：`Failed` 是一个**真实状态**，不是"回到 Available 并记一条日志"。

---

## 2. 各阶段详解

### 2.1 Resolving（解析来源）

```rust
#[derive(Debug, Clone, Serialize, Deserialize)]
#[serde(rename_all = "camelCase", tag = "kind")]
pub enum InstallSource {
    /// npm 兼容 registry
    Registry { package: String, version: VersionReq, registry: Option<Url> },
    /// git 仓库
    Git { url: String, reference: Option<String>, subdir: Option<String> },
    /// 直接 URL（指向 .wasm / .zip / tarball）
    Url { url: Url, digest: Option<Sha256Digest> },
    /// 本地路径（开发态；生产环境默认禁用，见 §2.1.2）
    Local { path: Utf8PathBuf },
}
```

#### 2.1.1 各来源的解析器

```rust
#[async_trait]
pub trait SourceResolver: Send + Sync {
    fn kind(&self) -> SourceKind;
    async fn resolve(&self, source: &InstallSource) -> Result<ResolvedArtifact, ResolveError>;
}

pub struct ResolvedArtifact {
    pub manifest: PluginManifest,
    pub artifacts: Vec<ArtifactRef>,      // wasm + ui.zip
    pub signature: Option<PluginSignature>,
    pub source_identity: SourceIdentity,  // 用于信任判断（registry+package+version / git commit）
    /// 是否需要构建（**新版永远为 false**，见下）
    pub requires_build: bool,
}
```

#### 2.1.2 明确不做的事：不在用户机上构建

| ------------------------------------- | ---------------------------------------- |
| `pnpm install --prod` 在用户机         | ❌ **完全不做**                           |
| `pnpm run build` / `synra-sdk build` 在用户机 | ❌ **完全不做**                     |
| 分发 npm tarball（含源码与依赖）        | ✅ 只分发**已构建、已签名**的 artifact      |

理由（[05 P8](./11-design-principles.md)）：
1. **供应链风险**：安装脚本可以执行任意代码；
2. **不可复现**：用户机的 Node/pnpm 版本、网络、缓存不同；
3. **慢**：npm install 可能几分钟；
4. **攻击面**：`postinstall` 脚本是 npm 生态的著名问题。

**因此**：`InstallSource::Local` **只允许在开发模式**（`cfg(debug_assertions)` 或显式的 dev 开关），且本地目录必须**已经构建好**（`synra-plugin build` 由作者在本地跑）。生产构建里 `Local` 直接返回 `Error::LocalInstallDisabled`。

#### 2.1.3 Registry 协议

```
GET {registry}/{package}                  → packument（元数据）
  · 读取 versions[{version}].dist.tarball
  · 读取 versions[{version}].dist.integrity（SRI）
  · 读取 versions[{version}].dist.shasum
  · 读取 versions[{version}].synra（可选：manifest 内联，便于预检）
GET {tarball_url}                         → 下载
```


```rust
pub struct RegistryConfig {
    pub default: Url,                              // https://registry.npmjs.org
    pub overrides: Vec<(PackagePattern, Url)>,      // 精确 & 通配
}

impl RegistryConfig {
    /// 优先级：overrides 精确 > overrides 通配 > default
    pub fn resolve(&self, package: &str) -> &Url;
}
```

### 2.2 Downloading（下载）

```rust
pub struct DownloadPlan {
    pub artifacts: Vec<ArtifactDownload>,
    pub total_bytes: Option<u64>,
}

pub struct ArtifactDownload {
    pub url: Url,
    pub expected_digest: Sha256Digest,
    pub max_bytes: u64,             // 上限（防 DoS）
    pub timeout: Duration,
}
```

**要求**：
1. **大小上限**（wasm ≤ 32 MiB，ui.zip ≤ 16 MiB）；
2. **流式哈希**（边下边算 sha256，不 buffer 全部到内存）；
3. **进度事件**（每 64 KiB 或每 200ms 发一次，节流）；
4. **可取消**（`CancellationToken`）；
5. **原子落盘**：下载到 `<target>.partial-<ts>` → 校验 → `rename`；
6. **重定向**：最多 5 跳，且**每一跳都校验 scheme/host**（防 SSRF，[10 §4.4](./10-capability-and-sandbox.md)）；
7. **HTTPS 强制**（`Url`/`Git` 来源不允许明文 http，除非显式 dev flag）。

### 2.3 Verifying（校验）—— ★ 最关键的阶段

```rust
pub struct VerifyPipeline {
    steps: Vec<Box<dyn VerifyStep>>,
}

pub enum VerifyStep {
    /// 1. artifact 摘要与 registry/来源声明一致
    Digest,
    /// 2. 签名有效（ed25519，见 [16](./16-plugin-distribution.md) §3）
    Signature,
    /// 3. 发布者是否在用户信任列表（→ 决定信任等级）
    PublisherTrust,
    /// 4. manifest schema 校验
    ManifestSchema,
    /// 5. capability 字符串语法
    CapabilitySyntax,
    /// 6. WIT world 版本与宿主兼容
    WitCompatibility,
    /// 7. wasm 的 import 集合 ⊆ 声明的 capability（[10 §2.4](./10-capability-and-sandbox.md)）
    ImportsWithinCapabilities,
    /// 8. wasm 不得 import 禁用接口（sockets / fs preopens 等）
    NoForbiddenImports,
    /// 9. 依赖图无环
    DependencyAcyclic,
    /// 10. 体积与条目数上限（解包炸弹防护）
    ArchiveLimits,
    /// 11. 平台适用性（例如声明需要 simulate-keys 的插件在 iOS 上警告）
    PlatformApplicability,
}
```

**每一步失败都要给出可操作的原因**：

```rust
#[derive(Debug, thiserror::Error)]
pub enum VerifyError {
    #[error("artifact digest mismatch: registry says {expected}, computed {actual}")]
    DigestMismatch { expected: Sha256Digest, actual: Sha256Digest },

    #[error("signature verification failed for publisher {publisher}: {reason}")]
    BadSignature { publisher: String, reason: String },

    #[error("plugin is unsigned; this registry/host requires a signature")]
    Unsigned,

    #[error("manifest field `{field}` is invalid: {reason}")]
    InvalidManifestField { field: String, reason: String },

    #[error("capability `{cap}` is not valid for a {runtime} plugin: {reason}")]
    CapabilityNotApplicable { cap: String, runtime: String, reason: String },

    #[error("plugin requires WIT {required}, host supports up to {supported}")]
    WitIncompatible { required: String, supported: String },

    #[error("plugin imports `{import}` but declares no capability covering it")]
    UndeclaredImport { import: String },

    #[error("plugin imports forbidden interface `{import}`")]
    ForbiddenImport { import: String },

    #[error("dependency cycle: {cycle:?}")]
    DependencyCycle { cycle: Vec<PluginId> },

    #[error("archive exceeds limits: {detail}")]
    ArchiveTooLarge { detail: String },

    #[error("plugin's UI artifact is missing but `runtime.ui` is declared")]
    MissingUiArtifact,
}
```


```rust
pub struct ArchiveLimits {
    pub max_entries: usize,             // 默认 4096
    pub max_total_uncompressed: u64,    // 默认 256 MiB
    pub max_single_file: u64,           // 默认 64 MiB
    pub max_compression_ratio: f64,     // 默认 200（防 zip bomb）
    pub max_path_depth: usize,          // 默认 16
}
```

**路径穿越防护**：解包时每个条目路径必须 `canonicalize` 后仍在目标目录内（拒绝 `../`、绝对路径、symlink 指向外部）。**symlink / hardlink 条目直接拒绝**。

### 2.4 AwaitingConsent（等待用户确认）

```rust
pub struct ConsentRequest {
    pub plugin_id: PluginId,
    pub title: String,
    pub publisher: PublisherInfo,
    pub trust: PluginTrust,
    /// 每个能力的展示信息（人类可读）
    pub requested: Vec<CapabilityRequest>,
    /// 相对于已授予版本的差异（升级场景）
    pub diff: Option<ConsentDiff>,
}

pub struct CapabilityRequest {
    pub capability: Capability,
    /// 人类可读说明（SDK 可提供本地化描述，或内核生成）
    pub description: String,
    /// 风险等级（用于 UI 配色）
    pub risk: RiskLevel,
    /// 是否是可选项（插件在缺它时能降级运行）
    pub optional: bool,
}

pub enum ConsentDiff {
    /// 升级：新增了这些能力
    Added(Vec<CapabilityRequest>),
    /// 升级：移除了这些能力（无需确认，但要展示）
    Removed(Vec<Capability>),
    /// capability 从精确变成了通配（★ 必须重新确认）
    Widened(Vec<(Capability, Capability)>),
}
```

**UI 要求**：
- 通配能力必须**展开显示**（"访问任意网站"）；
- 高风险能力（`platform:simulate-keys`、`platform:clipboard:read`、`fs:write`）用醒目样式；
- 显示"这个插件来自哪"（registry + 包名 + 版本 + 发布者签名状态）；
- 升级时只显示**差异**（避免用户疲劳 → 一路点同意）；
- 用户拒绝后，插件保持 `Available`（不是 `Failed`），并能看到"被拒绝的能力"。

### 2.5 Installing（安装落盘）

```rust
/// 原子安装：全部成功才可见
pub async fn install_atomically(
    plan: InstallPlan,
    paths: &PluginPaths,
) -> Result<InstalledPlugin, InstallError> {
    //    <plugins>/<plugin-id>/<version>/
    //      ├─ plugin.wasm
    //      ├─ ui/                     (UI artifact 解开)
    //      ├─ manifest.json            (规范化后的 manifest)
    //      ├─ signature.json           (签名原文)
    //      ├─ source.json              (来源信息，用于更新检查)
    //      └─ .installed               (标记文件，含 sha256 与安装时间)
    // 2. 写入 staging 目录：<plugins>/<plugin-id>/.staging-<ts>/
    // 3. 校验 staging 内容完整性
    // 4. rename staging → <version>（原子）
    // 5. 写 store（SQLite 事务）
    // 6. 可选：AOT 预编译（异步，不阻塞安装完成）
}
```


| --------------- | --------------------------------------------- | --------------------------------------- |
| 原子性          | Electron 有 staging；Capacitor **无**          | ✅ 两端统一 staging + rename             |
| 完整性校验      | ❌（`dist.shasum` 只存不校验）                  | ✅ sha256 必校                           |
| store 并发      | ❌ 整文件 read-modify-write，无锁              | ✅ SQLite 事务                           |
| 失败清理        | ⚠️ 部分（Capacitor 失败 rm）                    | ✅ staging 清理 + 记录                   |
| 已存在同版本    | ⚠️ 直接覆盖                                    | ✅ 幂等：摘要相同则跳过，不同则报错        |

### 2.6 Installed / Activating / Active

```rust
pub async fn activate(&self, plugin_id: &PluginId) -> Result<(), ActivateError> {
    // 1. 从槽位取出（类型上保证当前是 Installed）
    let slot = self.slots.remove(plugin_id)?;         // Installed(..)
    let activating = slot.into_activating()?;          // → Activating

    let guard = ActivationGuard::new(self, plugin_id);

    // 3. 检查依赖（compose）
    let order = self.deps.activation_order(plugin_id)?;
    for dep in order { self.ensure_active(dep).await?; }

    // 4. 装载
    match activating.manifest.runtime {
        RuntimeDecl::Headless { .. } => {
            // a. 从编译缓存加载（或编译）
            let compiled = self.compile_cache.load_or_compile(&activating)?;
            // b. 校验有效能力（manifest ∩ grant ∩ imports）
            let caps = verify_effective_caps(&activating.manifest, &activating.grant, &compiled.imports())?;
            // c. 创建实例池
            let pool = self.pools.create(plugin_id, compiled, caps, &activating.quota)?;
            // d. 调 activate()（带 fuel/epoch）
            let lease = pool.acquire().await?;
            lease.call_activate(&activating.config).await?;
            // e. 注册事件订阅、action 声明、定时任务
        }
        RuntimeDecl::Ui { .. } => {
            // a. 校验 ui artifact 完整性
            // b. 创建 UI 会话令牌（[10 §6](./10-capability-and-sandbox.md)）
            // c. 注册路由（供 UI 插件列表）
        }
        RuntimeDecl::Hybrid { .. } => { /* 两者都做 */ }
    }

    // 5. 成功 → 转入 Active，并"提交"guard
    let active = activating.into_active(runtime_handle);
    self.slots.insert(plugin_id.clone(), PluginSlot::Active(active));
    guard.commit();
    Ok(())
}

/// ★ 失败时自动回滚，绝不留下半激活状态
struct ActivationGuard<'a> {
    host: &'a PluginHost,
    plugin_id: PluginId,
    committed: bool,
}

impl Drop for ActivationGuard<'_> {
    fn drop(&mut self) {
        if self.committed { return; }
        // 回滚：销毁实例池、撤销事件订阅、撤销 UI 令牌、移除 action 声明
        // 状态 → Faulted { stage: Activating, error }
        self.host.rollback_activation(&self.plugin_id);
    }
}
```

> Rust 的 `Drop` + `?` 运算符天然提供这个保证。

### 2.7 Deactivating（停用）

```rust
pub async fn deactivate(&self, plugin_id: &PluginId, reason: DeactivateReason) -> Result<()> {
    let slot = self.slots.remove(plugin_id)?;      // Active(..)
    let deactivating = slot.into_deactivating()?;

    let result = async {
        // 1. 停止接收新的调用（闸门对该插件返回 Deactivated）
        // 2. 取消该插件的所有在途任务（host call、定时器、事件处理）
        // 3. 等待在途调用完成（有超时，默认 5s）
        // 4. 调 guest 的 deactivate()（有 fuel/epoch 上限）
        // 5. 销毁实例池
        // 6. 撤销事件订阅与 action 声明
        // 7. 撤销 UI 会话令牌（WebView 侧 IPC 立即失效）
        // 8. （可选）快照状态
    }.await;

    match result {
        Ok(()) => { self.slots.insert(id, Installed(deactivating.into_installed())); Ok(()) }
        Err(e) => {
            // ★ deactivate 失败不阻塞停用（插件已经不可用了）
            warn!("plugin deactivate failed: {e}; forcing deactivation");
            self.slots.insert(id, Faulted(..));
            Ok(())   // 对用户而言"停用成功"
        }
    }
}
```

**语义决定**：`deactivate` 失败**不阻塞**停用。理由：用户点了"停用"，插件就必须停止工作——即使它不愿意。但失败要记录，并且**下一次激活前必须清理干净**。

### 2.8 Faulted（故障态）

触发条件：
- 健康度模型判定（[12 §7.3](./12-wasm-host.md)）；
- 连续 trap / 配额持续超限；
- `activate` 失败且重试次数用尽；
- wasm 编译失败（缓存损坏）。

Faulted 状态：
- 插件**不可用**（闸门拒绝所有调用）；
- UI 显示醒目的错误卡片 + 原因 + "重试" / "停用" / "卸载" 按钮；
- 保留 artifact（便于诊断与重试）；
- **不自动复活**（避免 crash loop）。

### 2.9 Removing（卸载）

```rust
pub async fn remove(&self, plugin_id: &PluginId, opts: RemoveOptions) -> Result<RemoveReport> {
    // 1. 若 Active → 先 deactivate
    // 2. 检查依赖：若有其它插件依赖它 → 拒绝（或按 opts.cascade 级联停用）
    // 3. 删除 artifact 目录（所有版本，或保留最近 N 个）
    // 4. 删除编译缓存
    // 5. 删除 store 记录（安装记录 + 授权 + 健康度）
    // 6. 询问是否删除插件数据（`<data>/plugins/<id>/`，用户数据）
    //    ★ 默认保留（避免误删），但明确告知路径
    // 7. 撤销 UI 令牌、路由
    // 8. 返回报告（删了什么、留了什么）
}
```


---

## 3. 更新（Update）

```rust
pub struct UpdatePlan {
    pub from: Semver,
    pub to: Semver,
    pub capability_diff: ConsentDiff,
    pub requires_restart: bool,
}

impl PluginHost {
    /// 检查更新（不下载）
    pub async fn check_update(&self, plugin_id: &PluginId) -> Result<Option<UpdateInfo>>;

    /// 执行更新（内部走完整生命周期的 install 流程）
    pub async fn update(&self, plugin_id: &PluginId, to: Semver) -> Result<UpdateReport>;
}
```

**更新语义**：
2. 若 capability 不变 → 沿用授权，自动启用新版本；
3. 若 capability 新增/放宽 → 进入 `AwaitingConsent`（[§2.4](#24-awaitingconsent等待用户确认)）；
4. 切换版本时**先停旧后起新**（不并行，避免两个版本同时操作）；


```rust
pub enum AutoUpdatePolicy {
    /// 不自动（用户手动检查）
    Never,
    /// 自动检查，提示用户确认
    NotifyOnly,                 // 默认
    /// 同一 major 内自动更新
    AutoMinor,
    /// 全部自动（仅 Builtin / TrustedPublisher）
    AutoAll,
}
```

---

## 4. 跨设备同步（插件分发）


```rust
pub struct SyncPlan {
    pub plugin_id: PluginId,
    pub version: Semver,
    pub peers: Vec<DeviceId>,
    pub direction: SyncDirection,
}

pub enum SyncDirection {
    /// 本机有，推给对端
    Push,
    /// 对端有，拉到本机
    Pull,
}
```

流程：

```
1. 版本预检：向对端发 `_synra.plugin.installed.query`（envelope request，超时 5s）
   ← 对端回 { pluginId, version, digest }
2. 若对端已有相同 digest → 跳过（skipped）
3. 若对端没有或 digest 不同：
     a. 本机构造传输计划（wasm + ui.zip + manifest + signature）
     b. 经 file.transfer.* 传输（★ 用会话级 ACK 与 sha256 校验）
     c. 对端执行完整的 Verifying 流程（★ 不因为是"已配对设备"就跳过校验）
     d. 对端安装 → 按自身策略决定是否启用
     e. 对端回 `_synra.plugin.sync.result { ok, installed_version, activated }`
4. 冲突（对端有更新版本）：
     · 策略 `keep-newest`：保留较新版本
     · 策略 `ask`：UI 询问用户
     · 策略 `never-downgrade`（★ 推荐默认）：不降级，双方各自保留
```


| ---------------------------------------------- | -------------------------------------- |
| `complete.sha256` 从不校验                       | 必校验                                  |
| 非原生平台直接丢弃入站包                          | 平台无关                                |
| 无 `_synra.plugin.sync.result` 回执              | 实现                                    |
| 自动同步未实现（手动）                            | 策略化（默认通知 + 一键同步）             |
| 无冲突解决                                       | `never-downgrade` 为默认                 |
| `syncSessionId` 从不使用                          | 用于隔离多轮同步（防跨轮混块）             |
| 两个平台两套 store 实现                           | 内核统一                                |

---

## 5. 依赖（compose）与级联

```rust
pub struct DependencyGraph {
    /// plugin → 它依赖的 plugins
    deps: HashMap<PluginId, BTreeSet<PluginId>>,
    /// plugin → 反向依赖
    dependents: HashMap<PluginId, BTreeSet<PluginId>>,
}

impl DependencyGraph {
    /// 按拓扑序返回需要先激活的插件
    pub fn activation_order(&self, target: &PluginId) -> Result<Vec<PluginId>, DepError>;
    /// 环检测（安装期调用，拒绝安装成环的插件）
    pub fn detect_cycles(&self) -> Vec<Vec<PluginId>>;
    /// 某个插件停用/卸载时受影响的插件
    pub fn affected(&self, target: &PluginId) -> BTreeSet<PluginId>;
}
```


```rust
pub struct CascadeDeactivation {
    /// 待级联停用的插件
    pub victims: Vec<PluginId>,
    /// 撤销窗口
    pub undo_deadline: Instant,      // 默认 5s
    cancel: CancellationToken,
}

impl PluginHost {
    /// 用户停止一个被依赖的插件 → 提示 + 启动撤销窗口
    pub async fn deactivate_with_cascade(&self, id: PluginId) -> Result<CascadeHandle>;
    /// 用户在窗口内点了"撤销"
    pub fn undo_cascade(&self, handle: CascadeHandle) -> Result<()>;
}
```

**版本约束**（依赖必须带版本要求，否则"装错版本"只能在运行期暴露）：

```jsonc
{
  "compose": [
    { "id": "@synra-plugin/format", "version": "^1.2.0", "optional": false }
  ]
}
```

---

## 6. 状态持久化

```sql
-- crates/synra-store/schema/0002_plugins.sql

CREATE TABLE plugins (
  plugin_id           TEXT PRIMARY KEY,
  title               TEXT NOT NULL,
  description         TEXT,
  publisher           TEXT,
  trust               INTEGER NOT NULL,          -- PluginTrust
  source_json         TEXT NOT NULL,             -- InstallSource
  installed_version   TEXT,                      -- 当前安装的版本（可空 = 未安装）
  active              INTEGER NOT NULL DEFAULT 0,
  health_state        TEXT NOT NULL DEFAULT 'healthy',
  health_reason       TEXT,
  installed_at        INTEGER,
  updated_at          INTEGER,
  auto_update         TEXT NOT NULL DEFAULT 'notify_only'
);

CREATE TABLE plugin_versions (
  plugin_id      TEXT NOT NULL,
  version        TEXT NOT NULL,
  artifact_root  TEXT NOT NULL,                  -- 相对 plugins 根
  wasm_digest    TEXT,
  ui_digest      TEXT,
  manifest_json  TEXT NOT NULL,
  signature_json TEXT,
  manifest_fp    TEXT NOT NULL,                  -- capability 指纹（用于授权比对）
  installed_at   INTEGER NOT NULL,
  last_used_at   INTEGER,
  pinned         INTEGER NOT NULL DEFAULT 0,     -- 防 GC
  PRIMARY KEY (plugin_id, version)
);

CREATE TABLE plugin_grants (
  plugin_id           TEXT NOT NULL,
  granted_json        TEXT NOT NULL,             -- CapabilitySet
  denied_json         TEXT NOT NULL,
  manifest_fp         TEXT NOT NULL,
  granted_at          INTEGER NOT NULL,
  PRIMARY KEY (plugin_id)
);

CREATE TABLE plugin_actions (
  plugin_id     TEXT NOT NULL,
  name          TEXT NOT NULL,
  schema_json   TEXT NOT NULL,
  since_version TEXT NOT NULL,
  deprecated    INTEGER NOT NULL DEFAULT 0,
  PRIMARY KEY (plugin_id, name)
);

CREATE TABLE plugin_deps (
  plugin_id        TEXT NOT NULL,
  dep_plugin_id    TEXT NOT NULL,
  version_req      TEXT NOT NULL,
  optional         INTEGER NOT NULL DEFAULT 0,
  PRIMARY KEY (plugin_id, dep_plugin_id)
);

CREATE INDEX idx_plugins_active ON plugins(active);
CREATE INDEX idx_plugin_versions_used ON plugin_versions(plugin_id, last_used_at DESC);
```

**规则**：
- `pinned` 的版本不会被 GC；
- `manifest_fp` 是 capability 集合的稳定指纹 → 升级时判断是否需要重新确认。

---

## 7. 快照与恢复（Memento）


| 方案                          | 优点                     | 缺点                              | 采用 |
| ----------------------------- | ------------------------ | --------------------------------- | ---- |
| 内核自动快照插件内存            | 对插件透明                | Wasm 实例内存不可序列化             | ❌   |
| 插件自己用 `storage` 持久化      | 简单、可测、跨平台         | 插件作者要写代码                    | ✅   |
| 插件实现 `snapshot`/`restore`   | 表达力强                  | 复杂、易错、需要 WIT resource       | ⚠️ 后续 |

**决定**：第一版**只提供 `storage`**，不提供自动快照。文档明确告知插件作者"状态要自己存"。

---


| ------------------ | ----------------------------------------------- | ---------------------------------- |
| 状态模型           | 文档 6 态（未实现）；代码两套互不映射的枚举        | 13 个真实状态类型 + `Failed` 是一等状态 |
| 非法转移           | 运行时无检查 → 卡死                              | 类型状态机（编译期禁止）             |
| 失败回滚           | ❌（`activate` 抛错 → 卡在 `entering`）           | ✅ `ActivationGuard` + `Drop`        |
| 安装原子性         | 仅 Electron 有 staging                          | ✅ 统一                             |
| 完整性校验         | ❌                                              | ✅ sha256 必校 + 签名                |
| 归档安全           | ❌（无大小/条目/路径穿越防护）                    | ✅ 全套限制                          |
| 用户确认           | ❌（capabilities 不传递，无授权概念）             | ✅ `AwaitingConsent` + 差异展示       |
| 依赖图             | ❌（`compose` 未实现）                           | ✅ 拓扑排序 + 环检测 + 级联撤销窗口    |
| 数据删除           | ⚠️ 静默删除坏插件目录                            | ✅ 显式询问 + 报告                    |
| 跨设备同步         | ⚠️ 手动、无校验、单向                            | ✅ 策略化、必校验、双向                |
| 平台一致性         | ❌（Electron 与 Capacitor 两套 store 实现）       | ✅ 内核统一                          |
| 重启恢复           | ⚠️（靠 preferences，语义不一致）                  | ✅ store 里的 `active` 字段           |
