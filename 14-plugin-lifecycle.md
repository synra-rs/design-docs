# 14 · 插件生命周期

> 本文定义安装记录、安装更新任务、运行实例及失败恢复的目标契约。
> 2026-10-25 修订：三类状态分别建模，不再以一个 `PluginSlot` 覆盖整个插件。
> 当前实现尚未完成该重构，见 [19 §0.2](./19-roadmap.md)。早期调查已移入 [历史观察](./history/lifecycle-observations.md)。

## 1. 三类状态与目录视图

| 对象 | 拥有的事实 | 不应承担的状态 |
| --- | --- | --- |
| 安装记录 | 当前版本、启用意图、授权、签名者、来源与安装时间 | 下载进度、实例是否存活 |
| 安装或更新任务 | 任务 id、目标版本、验证材料、进度、取消、失败阶段与重试信息 | 旧版本的运行健康度 |
| 运行实例 | 运行版本、启动、运行、停止、故障及资源句柄 | Registry 是否有更新 |
| 目录视图 | 可发现的包、可安装版本、可用更新 | 本机安装及运行事实 |

```text
安装更新任务：Resolving → Downloading → Verifying → AwaitingConsent → Installing → Succeeded
             任一未提交阶段 → Cancelled / Failed { stage, error, retryable }
             已提交但未完成恢复 → Recovering → Succeeded / Failed { committed: true, ... }

运行实例：Stopped → Activating → Active → Deactivating → Stopped
                    └──────────→ Faulted ←─────────┘
          Faulted → Activating（显式重试）/ Stopped（停用）

安装记录：只在安装提交、授权、启用意图修改或卸载提交时变更；重启后从持久记录重建。
```

**并存场景**：v1 正在运行时可以下载和验证 v2；v2 下载失败只结束该任务，不将 v1 标为 `Faulted`。
新装被拒后不产生安装记录；更新被拒后保留旧安装记录和实例。任务失败必须可见，不能只留日志。
每个插件的提交、卸载、撤销和权限修改由共享运行时协调；可并行准备，不得并发提交冲突操作。
失败后的重试创建新任务并关联前次任务；提交后的恢复必须按提交记录处理，不能盲目重做迁移。

### 1.1 类型与快照边界

下面是职责示意，不是现有 Rust API：

```rust
struct InstalledPlugin { /* id, version, enabled, grants, signer, source */ }
struct PluginOperation { /* operation_id, target_version, stage, result */ }
struct PluginInstance { /* plugin_id, running_version, state, handles */ }
struct PluginSnapshot {
    installed: Option<InstalledSnapshot>,
    operation: Option<OperationSnapshot>,
    runtime: RuntimeSnapshot,
}
```

类型状态可以限制一次任务内的 `Verified → Prepared → Committed` 转移，
不再强迫目录条目、安装记录和运行实例共用一个线性枚举。
UI 快照由共享运行时组合，必须同时表达“v1 运行中，v2 下载失败”等真实场景。
各阶段的历史实现笔记已归档；实现进度统一维护于 [19 §0.2](./19-roadmap.md)。

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
- 高风险能力（`platform:simulate-keys`、`platform:read-clipboard`、`fs:write`）用醒目样式；
- 显示"这个插件来自哪"（registry + 包名 + 版本 + 发布者签名状态）；
- 升级时只显示**差异**（避免用户疲劳 → 一路点同意）；
- 用户拒绝后，任务结束为取消并展示被拒绝的能力；新装不产生安装记录，更新保留旧版本及其运行状态。

### 2.5 Installing（安装落盘）

```rust
/// 安装提交示意；rename 与 SQLite 不是同一个崩溃原子事务，恢复契约见 §3.1
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


| 方面 | 早期方案 | 当前目标 |
| --------------- | --------------------------------------------- | --------------------------------------- |
| 原子性          | Electron 有 staging；Capacitor **无**          | ✅ 两端统一 staging + rename             |
| 完整性校验      | ❌（`dist.shasum` 只存不校验）                  | ✅ sha256 必校                           |
| store 并发      | ❌ 整文件 read-modify-write，无锁              | ✅ SQLite 事务                           |
| 失败清理        | ⚠️ 部分（Capacitor 失败 rm）                    | ✅ staging 清理 + 记录                   |
| 已存在同版本    | ⚠️ 直接覆盖                                    | ✅ 幂等：摘要相同则跳过，不同则报错        |

### 2.6 激活与资源清理

激活读取已提交的安装记录，并为目标版本创建独立运行实例，不消费或删除安装记录。
共享运行时检查授权、签名者撤销、依赖与资源配额后执行以下步骤：

1. 将运行实例设为 `Activating`，创建仅管理本次资源的清理 guard。
2. 装载 headless 组件，校验有效能力并创建实例池，在 fuel、超时和取消约束下调用 `activate`。
3. 对 hybrid 校验 UI artifact 并登记受限 UI 会话；不提供纯 UI 激活分支。
4. 事件订阅、action 和定时任务随本次实例登记，成功后才对外发布 `Active`。
5. 失败时撤销本次实例池、订阅、UI 令牌、action 和定时任务，发布 `Faulted` 及具体原因。

`ActivationGuard` 的 `Drop` 只能兜底同步资源清理。需要等待的退出动作由显式、可取消且有超时的清理流程完成。
它不恢复旧版本、不回退插件数据，也不撤销插件已经产生的外部副作用；数据保证见 §3.1。
失败后安装记录仍然存在，用户可以重试、停用或卸载，不允许留下可接收调用的半激活实例。

### 2.7 Deactivating（停用）

停用只改变运行实例；用户明确停用时同时持久化 `enabled=false`，更新过程中的临时停机不改变启用意图。

1. 阻止该实例接收新调用，取消在途任务并限时等待。
2. 在 fuel 和超时约束下调用 `deactivate`；失败也继续撤销本次实例的全部资源。
3. 销毁实例池、订阅、action、定时任务和 UI 令牌，发布 `Stopped`；清理异常另行携带原因。
4. 保留安装记录、版本和用户数据；不得用“从槽位移除”同时表达停机与卸载。

发布者撤销和用户停用共用停止资源的实现，但撤销理由、授权事实和用户启用意图分别保留。

### 2.8 Faulted（故障态）

触发条件：
- 健康度模型判定（[12 §7.3](./12-wasm-host.md)）；
- 连续 trap / 配额持续超限；
- `activate` 失败；重试必须是另一次明确的启动尝试；
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
    // 6. 按用户明确选择删除或保留 plugin-data/<sanitised-id>/，默认保留
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

1. 下载、验证与确认属于更新任务，准备期间旧版本可以继续运行。
2. 仅在签名者、来源与原授权仍有效且权限未增加或放宽时，按策略继承授权；保留原启用意图，不自动启用原本停用的插件。
3. 权限新增、放宽或发布者身份变化时要求相应确认；拒绝不影响旧安装。
4. 提交前重检撤销、授权、源设置与取消状态，再停止旧实例写入并执行受限迁移；校验失败不进入提交。
5. 按 §3.1 完成切换与恢复，先停旧后起新；分别报告任务提交和实例激活的结果。


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

### 3.1 提交、迁移与恢复保证

安装提交与激活成功分别报告。最低保证如下，不能统称为“更新失败自动回滚”：

| 阶段或故障 | 必须保证 | 可见结果 |
| --- | --- | --- |
| 下载、验证、授权或提交前重检失败 | 旧安装与数据不变；仍可运行的旧实例继续服务 | 任务失败或取消，当前版本不变 |
| 数据迁移失败 | 只丢弃本次工作副本，不修改现用数据；尝试恢复因切换而停止的旧实例并单独报告恢复失败 | 更新失败，旧版本与原数据保留 |
| 文件与数据库提交中断 | 启动前按持久提交记录完成或撤回切换，核对版本、授权与数据代次 | 恢复完成后再加载实例；不能加载混合代次 |
| 提交成功但新版本激活失败 | 保留新安装及对应数据，清理本次运行资源；不自动换回旧代码读取新数据 | 安装成功且激活失败，提供重试或停用入口 |
| 插件已经产生外部副作用 | 不承诺自动撤销文件导出、网络请求或平台动作 | 如实报告已完成阶段及失败原因 |

**迁移目标设计**：迁移前停止实例写入并等待在途调用退出，对插件持久数据创建工作副本；
迁移只允许读写该副本，不授予网络、平台动作或其他外部写入能力。
迁移完成后校验结果，在同一提交计划中绑定新 artifact、授权和数据代次。
不能隔离的持久数据后端在迁移前明确拒绝，不能转而让迁移直接修改现用数据。
该副本是一次操作的恢复手段，不是为未发布版本保留历史格式兼容。

文件 rename 与 SQLite 事务不能组成一个共同的崩溃原子事务。
共享运行时必须有持久操作记录，记录准备、切换、提交和恢复所需的旧新路径及版本；
提交前准备好并持久化恢复材料，启动时先恢复未完成操作，再加载插件，最后清理过期副本。
取消仅在提交点前生效；进入切换后完成提交或恢复，再报告最终结果。

**当前差距**：当前流程可先调用旧实例的 `migrate` 再替换版本，尚未提供上述数据隔离与持久恢复协议。
这是一项待开发的契约，不把现有迁移称为数据可回滚。任务见 [19 §0.2](./19-roadmap.md)。

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

以下 SQL 是目标字段职责示意，实际 schema 由存储实现维护。安装更新任务另有持久操作记录，
用于任务诊断及 §3.1 的恢复；运行实例的 `Active` 不作为跨重启事实保存。

```sql
-- crates/synra-store/schema/0002_plugins.sql

CREATE TABLE plugins (
  plugin_id           TEXT PRIMARY KEY,
  title               TEXT NOT NULL,
  description         TEXT,
  publisher           TEXT,
  trust               INTEGER NOT NULL,          -- PluginTrust
  source_json         TEXT NOT NULL,             -- InstallSource
  installed_version   TEXT NOT NULL,             -- 当前已提交的安装版本
  enabled             INTEGER NOT NULL DEFAULT 0, -- 用户启用意图，不表示实例存活
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

**决定**：插件通过 `storage` 持久化业务状态，不提供 Wasm 内存的自动快照或恢复。
§3.1 的持久数据工作副本用于一次更新的提交恢复，不等同于运行内存快照。

---


| 方面 | 早期方案 | 当前目标 |
| ------------------ | ----------------------------------------------- | ---------------------------------- |
| 状态模型           | 安装、任务与实例混在同一个枚举 | 三类状态分别拥有事实，快照组合展示 |
| 非法转移           | 运行时无检查 → 卡死                              | 类型状态机（编译期禁止）             |
| 激活资源清理       | ❌（`activate` 抛错 → 卡在 `entering`）           | guard 兜底与显式限时清理；版本和数据恢复另见 §3.1 |
| 安装提交           | 仅 Electron 有 staging | staging、持久操作记录与恢复，见 §3.1 |
| 完整性校验         | ❌                                              | ✅ sha256 必校 + 签名                |
| 归档安全           | ❌（无大小/条目/路径穿越防护）                    | ✅ 全套限制                          |
| 用户确认           | ❌（capabilities 不传递，无授权概念）             | ✅ `AwaitingConsent` + 差异展示       |
| 依赖图             | ❌（`compose` 未实现）                           | ✅ 拓扑排序 + 环检测 + 级联撤销窗口    |
| 数据删除           | ⚠️ 静默删除坏插件目录                            | ✅ 显式询问 + 报告                    |
| 跨设备同步         | ⚠️ 手动、无校验、单向                            | ✅ 策略化、必校验、双向                |
| 平台一致性         | ❌（Electron 与 Capacitor 两套 store 实现）       | ✅ 内核统一                          |
| 重启恢复           | ⚠️（靠 preferences，语义不一致） | 从启用意图和授权重建实例，先恢复未完成提交 |
