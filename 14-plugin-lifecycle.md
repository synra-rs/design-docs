# 14 · 插件生命周期

> ★ **先说一件最重要的实情，它决定了本文剩下部分的读法：这套生命周期在生产里没有入口。**
>
> 本次逐条核对源码后确认：**一个插件无法在生产中被安装，因此也永远不会被装载。**
> 证据链是闭合的，每一环都单独查过：
>
> 1. `plugins` 表**唯一的写入方**是 `synra-store/src/sqlite/plugins.rs:53` 的 `insert_plugin`，
>    而它的调用者只有 `tests.rs`（两处）。**没有任何生产代码插入一个插件行。**
> 2. 迁移脚本从不种子化一行 —— `INSERT INTO plugins` 在全仓只出现在 `insert_plugin` 里。
> 3. `set_plugin_trusted`（`:118`，用户"我信任它"的那一步）**同样只有测试调用**；
>    它本身还是一条 `UPDATE ... WHERE plugin_id = ?1`，即**只能改已存在的行**。
> 4. app 的 `plugins_enable` → `set_plugin_enabled`（UPDATE），没有行时返回 `PluginNotInstalled`。
>    也就是说界面上的"启用"按钮对不存在的行无能为力，这是对的；**问题是没有东西负责创建那行**。
> 5. 本文 §1 那张 13 状态机的实现 `PluginSlot`（`plugin-host/src/lifecycle/pipeline.rs`）
>    连同它的测试都在，**但没有任何生产驱动者**。
> 6. 于是守护进程 `load_installed`（它要求行满足 `enabled && trusted`）读到的 `db.plugins()`
>    **只能是空的** —— 所以"插件宿主"在端到端意义上是不可达的。
>
> ★ 这解释了此前几轮分别记下的几件事，它们不是各自独立的问题，而是同一个洞的不同侧面：
> `plugin_grants` 没有写入方（没有同意流程就没有决定可记；★ 2026-10-18 起 daemon 侧有了 `plugin consent`）、
> 装载期 verify 管线没有调用者（★ 2026-10-18 起有了：守护进程的 `plugin install` 走它，见 §2.5 末尾）、`session: None` 导致闸门从未运行、
> 审计环没有生产者、以及 app 侧 26 个未注册命令里那一整组 `plugins_*`。
> 也就是说：**插件子系统的每一块都被很好地单独测试了，而它们之间没有一条路。**
>
> ★ 为什么本轮没有补上完整安装路径：它意味着实现一条完整的纵向功能
> （来源解析 → 下载 → 校验 → 同意 → 落盘 → 写库 → 启用 → 装载），
> 其中"下载"还需要一个**全仓都不存在的 HTTP 客户端**（见 [10](./10-capability-and-sandbox.md) §4.4）。
> 记在这里，是因为**本文其余部分描述的是一个尚未接线、但已逐块实现的设计**，读者需要知道这一点。
>
> ★ **但是第一步本轮补上了，而且它是纯增量的：`record_discovered_plugins`。**
> 守护进程启动时扫描 `plugins/`，把**数据库还不认识的**插件目录记成"已安装、未启用、未信任"。
> 这一条通路此前完全不存在 —— `insert_plugin`（`plugins` 表唯一的写入方）的调用者**只有测试**，
> 于是 `db.plugins()` 永远是空，于是 `load_installed` 永远什么都不装载，
> **无论磁盘上放了什么**。
>
> 几个刻意的决定：
> * **只记录、不信任。** 插件目录对以该用户身份运行的任何东西都是可写的，
>   所以"磁盘上有这个目录"不能等于"用户批准了它" —— 那会把文件系统变成权限边界。
>   `should_run` 是 `enabled && trusted`，两个都不写，所以它出现在列表里但不会运行。
> * **两层目录。** 插件 id 含斜杠（`@acme/hello`），实际布局是 `plugins/@acme/hello/`。
>   只看直接子目录会看到 `@acme` 并报"没有 manifest.json" ——
>   把一次完好的安装报成损坏的。这不是假设：第一版就是这么写的，被
>   `discovering_a_plugin_twice_records_it_once` 抓住了。
>   现在的规则是"**含 manifest.json 的目录才是插件**，其 id 是相对 plugins 目录的路径"，
>   与 `load_one` 解析插件用的是同一条规则，两者不会对"哪个目录是插件"产生分歧。
> * **幂等。** 每次启动都会跑；重复插入会撞主键，而每次启动都打印"记录失败"会淹没真正的问题。
> * **纯增量。** 用户已经启用并信任的插件重启后原样保留 ——
>   一个会重插或重置标志的协调逻辑会**每次开机把所有插件静默关掉**，
>   而这种 bug 表现为"我的插件老是自己关掉"。
>   测试 `discovering_does_not_overwrite_a_decision_the_user_has_made` 守这一条。
> * **坏目录要说出来。** 一个既没有 manifest 也没有子目录的目录是"误放的"或"拷了一半的"，
>   用户需要知道自己属于哪一种，所以原因是带回的而不是丢掉的。
>
> 负控制：把插入时的 `false, false` 改成 `true, true`，测试立刻红（"a discovered directory must not be enabled"）。
>
> ★ 仍然缺的下一步：**信任**。`set_plugin_trusted` 依旧只有测试调用，
> 也就是说插件现在能被看见、能被启用，但**没有任何路径能让它变成 `trusted`**，
> 因此仍然不会装载。缺的是本文 §2.4 的 `AwaitingConsent` 那一步：
> 展示它要的能力、记录每一条决定（写 `plugin_grants`）、再置 `trusted`。
> 那一步同时会给 `plugin_grants` 补上它一直缺的写入方。
>
> ★ **那一步本轮补上了：`plugins_consent_respond`（app 侧）。**
> 两处代码：
> * **store 的写入方** `SynraDb::record_grants`（`synra-store/src/sqlite/plugins.rs`）。
>   `plugin_grants` 此前有 schema、有索引、有行类型、有读者，**就是没有写入方** ——
>   全仓每一句 `INSERT INTO plugin_grants` 都在测试里。
>   `decided_ms` 与 `manifest_digest` 是**每条决定**的而不是每插件的，因为两者回答的都是
>   "用户同意的是哪一份文档"：一个升级后多声明了能力的插件，其 digest 不同，
>   于是旧行明显早于磁盘上现在的 manifest —— 这就是让"能力悄悄变多"可被发现而不是隐形的东西。
>   每行 upsert，所以改主意是覆盖而不是堆积历史（历史在审计里，这张表是当前状态）。
> * **那个动作** `plugins_consent_respond(plugin_id, granted)`。
>   规则：**manifest 声明的每一条都被授予，才算 trusted**；拒绝任何一条就不算。
>   ★ 这比设计文档暗示的更严，而且是刻意的 —— 见 [15](./15-plugin-sdk.md) 里对
>   "支持可选性"那一条的更正：`CapabilityDeclaration` 只有 `{ cap, reason }`，
>   **没有 `optional`**，所以没有"作者说少了它也能用"这回事，
>   任何猜测哪条可以不要的规则都是凭空发明。
>   一个缺了声明能力的插件跑起来，正是同意机制存在的意义所在。
> * 无法读取 manifest 时**拒绝**而不是当作"什么都没声明"：
>   后者会记下一组空决定，与"用户全部拒绝"无法区分，还会把插件标成已信任 ——
>   而没有任何东西被展示给任何人。
>
> 负控制两条，都对：把记录改成写空数组 → `granting_every_capability…` 红；
> 把 `trusted` 改成恒为 true → `declining_a_capability…` 红（`right: "awaiting_consent"`）。
>
> ★ **整条链现在通了**：目录被发现（上一轮）→ 出现在列表里且状态是 `awaiting_consent`
> → 用户回应 → 决定写入 `plugin_grants`、`trusted` 置位 → `should_run` 为真 → 装载。
> 路径上**每一个环节此前都是断的**，而现在只剩最后一跳没验证：
> 让一个真的 wasm 插件走完它（受夹具与环境限制，见 [12](./12-wasm-host.md)）。
>
> ★ 一个仍然存在的不一致（记下来而不是藏着）：装载器 `load_one` 仍然用
> `grants_of(&manifest)`，**没有读 `plugin_grants`**。
> 也就是说用户现在可以拒绝某条能力，而装载路径照样按 manifest 授权。
> 这不是可以顺手改的一行：一旦取交集，任何**没有**做过同意的插件（例如手工放进目录、
> 从未打开过设置界面的）授权集就是空的，会全部停止工作。
> 正确的下一步是在装载时**检查**是否存在与当前 `manifest_digest` 匹配的决定，
> 不匹配就拒绝装载并说明原因（"需要重新确认"），而不是静默取交集。
> 那需要一个"拒绝并让用户去确认"的交互闭环，比一行交集大。

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
- 高风险能力（`platform:simulate-keys`、`platform:read-clipboard`、`fs:write`）用醒目样式；
- 显示"这个插件来自哪"（registry + 包名 + 版本 + 发布者签名状态）；
- 升级时只显示**差异**（避免用户疲劳 → 一路点同意）；
- 用户拒绝后，插件保持 `Available`（不是 `Failed`），并能看到"被拒绝的能力"。

> ★ **2026-10-18：这一步现在有写入方了（daemon 侧）。**
>
> `Request::PluginConsent { plugin_id, granted }` → `Kernel::plugin_consent`，CLI 是
> `synra plugin consent <id> --allow <cap>...`。它做四件事，每件都有理由：
>
> * **读磁盘上的 manifest** —— 决定是针对**一份文档**做的，所以摘要从当前字节算，而不是从数据库里抄；
>   更新后摘要变了，那些 row 会显得比 manifest 旧，装载时就会要求重新确认（见 §2.6）。
> * **granted ⊆ 声明** —— 点名了一个插件没要的能力就拒（`InvalidArgument`）。通常是笔误，而若不是笔误，
>   就是客户端在替用户扩大决定。
> * **每个声明写一行**（允许/拒绝），不是只写允许的：缺一行等于"没有决定"，而装载器要的是每一条都有决定。
> * **trusted = 至少允许一条** —— 全拒不是批准。`should_run` 是 `enabled && trusted`，所以全拒之后即使再来一次
>   `enable` 也不会运行，这正是设计里"完全拒绝"应有的样子。
>
> 仍然缺的：**升级差异**（`ConsentDiff`）没有实现 —— 现在重新确认是全量重来，不是只问新增的那几条。
>
> ★ **2026-10-18（第二轮）：规则搬进内核了，因为它有两个答案。**
>
> 上一轮之后发现：app 的 `plugins_consent_respond` 规定"**每一条**声明都授予了才算 trusted"，而 daemon 是
> "**至少一条**"。装载器两边都照已授予的行建能力集，于是**同一个用户决定**在一个宿主里得到一个跑不起来的插件、
> 在另一个里得到一个带着子集运行的插件 —— 而用户看到的只有一个开关。
>
> 现在规则在 `synra_core::consent::decide`（纯函数，无 IO）：每个声明一行、granted ⊆ 声明、摘要取当前字节、
> **trusted = 至少允许一条**。选"部分同意可以运行"是因为它正是 `Consent::is_partial()` 与能力闸门存在的原因：
> 授予集合是权威，没授予的能力在闸门被拒，而不是让整个插件不跑 —— 后者会让用户看到自己"同意了两条"的插件
> 永远不工作，且没有任何设置解释原因。
>
> 两个宿主现在都只问这个函数，`plugin_consent` / `plugins_consent_respond` 各自只剩"把决定写进库"。
> 断言在两处：`partial_consent_loads_the_plugin_with_only_what_was_allowed`（daemon，检查装载器拿到的
> 能力集里**没有**被拒的那条）与 `declining_one_capability_records_the_denial_and_still_trusts`（app）。

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

> ★ **2026-10-18：上面第 1–5 步有了第一个实现，而它的几处与伪代码不同。**
>
> 实现是 `synra-daemon/src/runtime/install.rs`（`from_directory`），入口是控制通道的
> `Request::PluginInstall` → `Kernel::plugin_install` —— 也就是 `synra plugin install <目录>` 第一次真的做事。
> 差异每一条都是刻意的：
>
> * **没有 `<version>` 这一层**：现存布局是 `plugins/<plugin-id>/`（`load_one` 与 `record_discovered_plugins` 都按
>   这条规则解析目录），换成带版本号的目录是另一件事，不该被"接上安装路径"顺手改掉。
> * **staging 在叶子旁边**：`plugins/@acme/.hello.installing-<ts>` → rename 成 `plugins/@acme/hello`。临时名只能用
>   **叶子**（`hello`）而不是 id（`@acme/hello`）—— 后者会拼出两级路径，第一版就是错的，测试当场报 `os error 3`。
> * **先读字节再写**：每个 artifact 只读一次，哈希与落盘用同一份字节。读两次（一次算摘要、一次复制）会留下一个
>   窗口，写下去的是没人校验过的字节。
> * **没有 `source.json` / `.installed` 标记**：来源与时间在 `plugins` 行上（时间戳由 `insert_plugin` 写），
>   再写一份文件就是同一事实的第二个作者。
> * **失败即回滚**：目录已就位而写库失败时删掉目录。反过来会让一行指向从未写下的文件，
>   `load_installed` 会把它报成"插件坏了"，而不是"安装失败了"。

> ★ **2026-10-19：卸载也有 CLI 入口了。** `Request::PluginRemove { plugin_id, remove_data }` →
> `Kernel::plugin_remove`：守护进程先用 `PluginFiles::size_of` 量字节（artifact，加 `--data` 时的 sandbox），删行，
> 再删目录，答案带 `bytesFreed` 与 `dataKept`。CLI `synra plugin remove <id> [--data]` 因此从"解析了但什么都没做"变成
> 真的卸载；prose 输出一句话而不是 JSON 字段名。默认保留数据这一点没变 —— 与窗口的对话框是同一个默认值。
> * **落点是 `enabled=false, trusted=false`**：第 5 步写的是"安装"，不是"批准"。同意仍然是 `plugins_consent_respond`
>   （app 侧）与 `plugin enable` 那两步。
>
> 仍然没有的：`PluginSlot` 仍然没有驱动者（状态机与数据库行是同一件事的两份描述，尚未合一）、
> registry 安装（没有 HTTP 客户端）、以及 app 侧的 `plugins_install` 命令。
>
> ★ **2026-10-18（第二轮）：第二条门也查签名了。**
>
> `plugins/` 有两条入口：`plugin install`（跑完 11 步）与**手工放进去一个目录**（启动时的
> `record_discovered_plugins`，作者开发时就是这么做的）。后者原来"有 manifest 就记成已安装"，于是
> **一个被换掉 `plugin.wasm`、签名是真的但覆盖的是别的字节的包，会变成"已安装插件"** —— 用户随后看到的同意提示，
> 是关于一份没人签过的文档。现在扫描时若目录里有 `signature.json`，就验签 + 比对 subject 的摘要；
> 没有签名的目录仍然照记（那是作者的本地构建），只是它照样是 untrusted，同意仍是闸门。
>
> 代价是一次扫描每个新目录多几个哈希、不编译；收益是"签了名的包不能声称自己是另一个包"。

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

> ★ **2026-10-18：卸载的阻塞点不是 UI，是删除职责；而上面第 6 步的路径写错了。**
>
> **路径。** 实现的布局是 `plugins/<id>/`（artifact）与 **`plugin-data/<sanitised-id>/`**（用户数据）**两个目录**，
> 不是"`<data>/plugins/<id>/` 里面还装着用户数据"。`Paths::plugins_dir` 与 `Paths::plugin_data_dir` 是两个成员，
> `plugin_data_dir_for` 会对 id 做 sanitise。所以"删插件、留数据"在布局上成立 —— 这正是第 6 步要的默认值。
>
> **删除职责。** 结构性守卫 `only the sandbox may delete a directory tree` 只允许
> `crates/synra-store/src/sandbox.rs` 调 `remove_dir_all`，而 `plugins/<id>/` 不属于沙箱（沙箱管的是 plugin-data）。
> 于是卸载卡在**所有权**上而不是缺一个函数：要删插件目录，必须先有一个明确命名的所有者。
>
> **决定。** 在 `synra-store` 里新增 `PluginFiles`，与 `Sandbox` 并列，只做一件事：删掉某个插件 id 的 artifact
> 目录，以及（可选）它的 data 目录。守卫的允许名单加这一条，理由写在同一行。三条约束：
>
> * id 先 `sanitise_component`，再断言结果仍在 `plugins_dir` 之下（`is_within` 已存在）；
> * 只认这两个目录，不接受任意路径；
> * data 目录**只在 `remove_data` 为真时**删，默认保留 —— 第 6 步的"询问"就是它的默认值。
>
> 之后三步（store 的 `remove_plugin`、命令 `plugins_uninstall`、页面按钮）按这个形状写。
>
> **四步均已落地**（2026-10-18）：`crates/synra-store/src/plugin_files.rs`（含三条测试）、
> `SynraDb::remove_plugin`（事务，含一条断言两张表都干净）、`commands::plugins::plugins_uninstall`
> （先记录后目录，含两条）、页面上的 `Remove` + `APopconfirm`。
> ★ **2026-10-19：那一行确认换成了一个能问的对话框。** 页面不再用 `APopconfirm`（它的确认句是“数据留在磁盘上”，
> 所以 `remove_data` 恒为 `false`），改成 `AModal` + “Also delete its data” 勾选：不勾是保留（默认、可逆），勾了才把
> `remove_data: true` 传给命令。对话框用 `get-container="false"` 留在组件里，测试才看得见那个勾选框；每次打开都把勾选
> 重置为“不删”，记住上次的破坏性选择等于第二次访问默认删数据。
> ★ **对话框显示数据目录路径**：`PluginSnapshot` 增加了 `data_path`，壳用 `Paths::plugin_sandbox_dir` 构造它 ——
> 与卸载本身同一个函数，否则“数据在哪”会有两个答案。
> ★ **2026-10-19：`RemoveReport` 回来了。** 契约里这条命令一直是 `One`，而壳返回 `()`、界面按 `null` 读 —— 声明与实现
> 差了一个形状，而且没有任何守卫检查返回值。现在 `plugins_uninstall` 返回 `RemoveReport{bytes_freed, data_kept,
> leftovers}`：字节数在删除**之前**量（artifact 目录 + `remove_data` 为真时的 sandbox），`data_kept` 是
> `!remove_data`，`leftovers` 恒空 —— `PluginFiles::remove` 要么整棵删掉要么拒绝，没有“删了一半”的状态。
> 页面把 `bytes_freed` 格式化后放进报告句（`removedKept`/`removedDeleted`），`formatBytes` 同时从 HomePage 抽到
> `src/format.ts`，两个页面共用一个实现。
> ★ 这一节的两项（删除前询问、删除后报告）都已闭环。

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
