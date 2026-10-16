# 17 · 存储与配置

> 本文定义内核的持久化：SQLite schema、KV、插件沙箱目录、迁移、密钥管理。
> 对应 [06 §3.7](./06-workspace-layout.md) 的 `synra-store` crate。

---

## 1. 数据目录布局

```
<data-dir>/                              # 平台相关，见 §1.1
├─ kernel.db                             # SQLite（WAL 模式）
├─ kernel.db-wal / kernel.db-shm
├─ keys/
│  ├─ install.id.key                     # 本机 device_id 的加密密钥（可选）
│  └─ plugin-secrets.db                  # 插件 secure storage（加密）
├─ plugins/                              # 插件 artifact（只读内容）
│  └─ <plugin-id>/
│     ├─ .staging-<ts>/                  # 安装中的临时目录
│     ├─ 1.2.0/
│     │  ├─ plugin.wasm
│     │  ├─ ui/
│     │  ├─ manifest.json
│     │  ├─ signature.json
│     │  ├─ source.json
│     │  └─ .installed                   # 标记 + 摘要 + 时间
├─ plugin-data/                          # 插件运行时可写数据（沙箱根）
│  └─ <plugin-id>/
│     ├─ files/                          # fs:read/write 的根
│     └─ cache/
├─ cache/
│  ├─ wasm/                              # 编译产物缓存
│  │  └─ <sha256>-<backend>-<wasmtime-ver>.cwasm
│  ├─ catalog/                           # registry packument 缓存
│  │  └─ <registry-hash>/<package>.json
│  └─ diagnostics/                       # 导出的诊断包
├─ logs/
│  ├─ kernel.<date>.log                  # 结构化 JSON 日志（轮转）
│  └─ crash/<ts>.json                    # panic / crash 报告
└─ config.json                           # 少量启动前必须读的配置（见 §4.4）
```

### 1.1 平台数据目录

| 平台    | 数据目录                                          | 依据                                    |
| ------- | ------------------------------------------------- | --------------------------------------- |
| Windows | `%APPDATA%\synra\`（Roaming）或 `%LOCALAPPDATA%`   | Roaming 用于配置，Local 用于缓存         |
| macOS   | `~/Library/Application Support/dev.synra-app.app/`     | 标准位置                                 |
| Linux   | `$XDG_DATA_HOME/synra/`（默认 `~/.local/share/synra/`） | XDG Base Directory                 |
| Android | `Context.filesDir` / `Context.noBackupFilesDir`    | Tauri 提供                               |
| iOS     | `Application Support/`（`Library/Application Support/`） | 备份 + 不备份两处分工             |

**分工原则**：
- **数据**（DB、插件、插件数据）→ 会被备份；
- **缓存**（编译产物、catalog、日志）→ 不备份，可随时重建；
- **密钥** → 走 OS keychain（§5）而不是文件。

```rust
pub struct Paths {
    pub data_dir: Utf8PathBuf,
    pub cache_dir: Utf8PathBuf,
    pub log_dir: Utf8PathBuf,
    pub plugins_dir: Utf8PathBuf,
    pub plugin_data_dir: Utf8PathBuf,
    pub db_path: Utf8PathBuf,
}
```

> Rust 版只有一套 `Paths`，由 `synra-store` 提供，全应用统一。

---

## 2. SQLite Schema

### 2.1 为什么选 SQLite

| 方案                | 优点                     | 缺点                          | 结论        |
| ------------------- | ------------------------ | ----------------------------- | ----------- |
| **SQLite（rusqlite）** | 事务、查询、成熟、单文件  | 需要 FFI（bundled 编译）        | **采用**    |
| sled / redb         | 纯 Rust、快              | 无查询语言、生态较小            | 备选        |


**采用 `rusqlite` + `bundled` feature**（自带 SQLite，不依赖系统库）：

```toml
rusqlite = { version = "0.3x", features = ["bundled", "serde_json", "chrono", "hooks"] }
```

### 2.2 迁移

```rust
// crates/synra-store/src/db.rs

use rusqlite_migration::{Migrations, M};

pub fn migrations() -> Migrations<'static> {
    Migrations::new(vec![
        M::up(include_str!("../schema/0001_init.sql")),
        M::up(include_str!("../schema/0002_plugins.sql")),
        M::up(include_str!("../schema/0003_transfers.sql")),
        M::up(include_str!("../schema/0004_audit.sql")),
        M::up(include_str!("../schema/0005_intents.sql")),
    ])
}

pub fn open(path: &Utf8Path) -> Result<Connection, StoreError> {
    let mut conn = Connection::open(path)?;

    // ── 性能与可靠性设置 ──
    conn.pragma_update(None, "journal_mode", "WAL")?;      // 并发读写
    conn.pragma_update(None, "synchronous", "NORMAL")?;    // WAL 下安全且快
    conn.pragma_update(None, "foreign_keys", "ON")?;
    conn.pragma_update(None, "busy_timeout", 5000)?;
    conn.pragma_update(None, "temp_store", "MEMORY")?;
    conn.pragma_update(None, "mmap_size", 268435456)?;      // 256 MiB

    // ── 迁移 ──
    migrations().to_latest(&mut conn)?;

    Ok(conn)
}
```

**规则**：
- 迁移脚本**只增不改**（已发布的迁移不可编辑）；
- 破坏性变更（删列）要有独立的、可选的 `compact` 步骤（不自动执行）。

### 2.3 表清单（汇总）

| 表                       | 内容                                       | 生命周期     |
| ------------------------ | ------------------------------------------ | ------------ |
| `identity`               | 本机 device_id / display_name / profile     | 永久         |
| `paired_devices`         | 已配对设备记录（含 peer features）           | 永久         |
| `devices_seen`           | 见过的设备（用于显示名缓存、排序）             | 可清理       |
| `plugins`                | 插件主记录（信任、启用、健康度）              | 永久         |
| `plugin_versions`        | 各版本 artifact（摘要、manifest、指纹）       | GC 可清      |
| `plugin_grants`          | 授权（granted / denied + manifest 指纹）     | 永久         |
| `plugin_actions`         | 插件声明的动作                                | 随插件       |
| `plugin_deps`            | 依赖关系                                     | 随插件       |
| `plugin_audit`           | 闸门审计（环形，可清）                        | 可清理       |
| `sync_records`           | 跨设备同步历史                                | 可清理       |
| `transfer_sessions`      | 传输会话（用于续传）                          | TTL 清理     |
| `intents`                | 意图（可选持久化，用于离线排队）               | TTL 清理     |
| `config`                 | 配置 KV                                     | 永久         |
| `schema_migrations`      | 迁移版本（由 rusqlite_migration 管理）         | 永久         |

### 2.4 关键表结构（节选）

```sql
-- 0001_init.sql
CREATE TABLE identity (
  id                 INTEGER PRIMARY KEY CHECK (id = 1),   -- 单行表
  device_id          TEXT NOT NULL UNIQUE,
  display_name       TEXT NOT NULL,
  platform           TEXT NOT NULL,
  app_version        TEXT NOT NULL,
  profile_json       TEXT NOT NULL DEFAULT '{}',
  created_at         INTEGER NOT NULL,
  updated_at         INTEGER NOT NULL
);

CREATE TABLE paired_devices (
  device_id            TEXT PRIMARY KEY,
  display_name         TEXT NOT NULL,
  paired_at            INTEGER NOT NULL,
  last_seen_at         INTEGER,
  last_resolved_host   TEXT,            -- IPv4/IPv6 字面量
  last_resolved_port   INTEGER,
  peer_features_json   TEXT NOT NULL DEFAULT '[]',
  peer_pubkey          TEXT,            -- 若对端有能力签名（[16 §3]）
  trust_level          INTEGER NOT NULL DEFAULT 1,
  notes                TEXT
);
CREATE INDEX idx_paired_last_seen ON paired_devices(last_seen_at DESC);

CREATE TABLE devices_seen (
  device_id       TEXT PRIMARY KEY,
  display_name    TEXT,
  platform        TEXT,
  first_seen_at   INTEGER NOT NULL,
  last_seen_at    INTEGER NOT NULL,
  seen_count      INTEGER NOT NULL DEFAULT 1
);
CREATE INDEX idx_seen_last ON devices_seen(last_seen_at DESC);

-- 0003_transfers.sql
CREATE TABLE transfer_sessions (
  transfer_id       TEXT PRIMARY KEY,
  kind              TEXT NOT NULL,               -- plugin-bundle | attachment
  direction         TEXT NOT NULL,               -- inbound | outbound
  peer_device_id    TEXT NOT NULL,
  plugin_id         TEXT,                        -- kind=plugin-bundle 时
  plugin_version    TEXT,
  total_bytes       INTEGER,
  total_chunks      INTEGER,
  chunk_size        INTEGER NOT NULL,
  sha256            TEXT,
  state             TEXT NOT NULL,               -- Requested|Streaming|...
  received_bitmap   BLOB,                        -- ★ 用于断点续传（哪些块已收）
  bytes_received    INTEGER NOT NULL DEFAULT 0,
  staging_path      TEXT,                        -- 部分内容的落盘位置
  started_at        INTEGER NOT NULL,
  updated_at        INTEGER NOT NULL,
);
CREATE INDEX idx_transfers_expires ON transfer_sessions(expires_at);

-- 0004_audit.sql
CREATE TABLE plugin_audit (
  id            INTEGER PRIMARY KEY AUTOINCREMENT,
  at            INTEGER NOT NULL,
  trace_id      TEXT NOT NULL,
  plugin_id     TEXT NOT NULL,
  plugin_version TEXT NOT NULL,
  capability    TEXT NOT NULL,
  outcome       TEXT NOT NULL,                   -- allowed|denied|quota|rate_limited
  reason        TEXT,
  duration_us   INTEGER,
  summary_json  TEXT NOT NULL DEFAULT '{}'       -- ★ 不含 payload 内容
);
CREATE INDEX idx_audit_plugin_time ON plugin_audit(plugin_id, at DESC);
CREATE INDEX idx_audit_time ON plugin_audit(at DESC);

-- 0005_intents.sql（可选：离线排队）
CREATE TABLE intents (
  intent_id     TEXT PRIMARY KEY,
  trace_id      TEXT NOT NULL,
  source_json   TEXT NOT NULL,
  state_json    TEXT NOT NULL,
  target_device TEXT,
  plugin_id     TEXT,
  created_at    INTEGER NOT NULL,
  updated_at    INTEGER NOT NULL,
  expires_at    INTEGER                        -- NULL = 不过期（排队）
);
CREATE INDEX idx_intents_state ON intents(state_json);
```

### 2.5 事务与并发

```rust
pub struct Store {
    /// 写连接（单写者）
    write: Mutex<Connection>,
    /// 读连接池（WAL 允许多读者）
    read: r2d2::Pool<SqliteConnectionManager>,
}

impl Store {
    /// 写操作：自动重试 SQLITE_BUSY（配合 busy_timeout）
    pub fn write<T>(&self, f: impl FnOnce(&mut Connection) -> Result<T, StoreError>) -> Result<T, StoreError>;

    /// 读操作：从池取
    pub fn read<T>(&self, f: impl FnOnce(&Connection) -> Result<T, StoreError>) -> Result<T, StoreError>;
}
```

**规则**：
- 写操作**短小**（不做网络 IO、不调用插件），避免长事务；
- 需要"读-改-写"的逻辑必须在**一个事务**里（或用一个内存锁保护）；
- 批量写（如审计）用**异步批量提交**（攒 100 条或 500ms flush）。

---

## 3. 插件沙箱目录

```rust
// crates/synra-store/src/sandbox.rs

pub struct PluginDataRoot {
    root: Utf8PathBuf,          // <data-dir>/plugin-data/<plugin-id>/
}

impl PluginDataRoot {
    /// 创建并返回已 canonicalize 的根
    pub fn ensure(plugin_id: &PluginId) -> Result<Self, StoreError>;

    /// 解析插件提供的相对路径（[10 §4.3](./10-capability-and-sandbox.md)）
    pub fn open_dir(&self, sub: &str) -> Result<cap_std::fs::Dir, PathError>;
}
```

**使用 `cap-std`（推荐）**：

```toml
cap-std = "3"
```

`cap_std::fs::Dir` 提供 capability-based 文件访问——**没有路径穿越的可能**，因为所有操作都相对于已打开的目录句柄（消除 TOCTOU）。

```rust
// 例子：插件请求写 /cache/tmp.txt
let dir = sandbox.open_dir("cache")?;              // 已授权的子目录
let mut file = dir.create("tmp.txt")?;             // ★ 无法逃出 cache/
file.write_all(data)?;
```

> 这比"解析路径 + 检查前缀 + 再打开"的做法**根本性地**更安全。

### 3.1 沙箱目录的可见结构

```
plugin-data/<plugin-id>/
├─ files/        # 插件通过 fs:* 能力可读写（也可被用户通过 UI 浏览/清理）
├─ cache/        # 可随时清理
└─ .meta.json    # 大小统计、最后访问时间（用于配额与 GC）
```

**关键**：`files/` 是**用户可见可管理**的（"这个插件用了 12 MB"→ 可清理）。

### 3.2 UI 插件的存储

UI 插件**不能**直接访问文件系统（[08 §7.2](./08-ipc-and-frontend.md)）。
它的存储通过：
- `synra.storage.*`（KV，与 headless 插件共用同一套 `storage` 能力）；
- `synra.transfer.sendBlob`（大对象，走传输会话）。

**不允许**：`localStorage` / `IndexedDB`（在插件 WebView 里可以通过 CSP/权限禁用，或直接用无持久化的 WebView profile）。

---

## 4. 配置

### 4.1 分层

```
┌───────────────────────────────────────────────────────┐
│  编译期默认值（Rust const，见 synra-protocol::defaults）│
├───────────────────────────────────────────────────────┤
│  config.json（启动前必须可用，少量关键项）               │
├───────────────────────────────────────────────────────┤
│  DB `config` 表（其余全部配置）                          │
├───────────────────────────────────────────────────────┤
│  环境变量（开发/CI 覆盖，SYNRA_* 前缀）                   │
├───────────────────────────────────────────────────────┤
│  CLI 参数（仅 CLI/daemon）                               │
└───────────────────────────────────────────────────────┘
优先级：CLI > 环境变量 > DB > config.json > 默认值
```

### 4.2 `config.json`（引导配置）

**为什么需要它**：有些配置在**打开 DB 之前**就要知道（例如 DB 路径、日志级别）。

```jsonc
{
  "dataDir": null,             // null = 平台默认
  "cacheDir": null,
  "logLevel": "info",          // trace|debug|info|warn|error
  "logFormat": "json",         // json|pretty
  "listenPort": 32100,
  "discoveryPort": 32101,
  "enableDiscovery": true
}
```

### 4.3 DB `config` 表（完整配置）

```sql
CREATE TABLE config (
  key        TEXT PRIMARY KEY,
  value      TEXT NOT NULL,      -- JSON
  updated_at INTEGER NOT NULL
);
```

配置项（按域）：

```rust
#[derive(Debug, Clone, Serialize, Deserialize, PartialEq)]
#[serde(rename_all = "camelCase", default)]
pub struct AppConfig {
    // ── 网络 ──
    pub network: NetworkConfig,
    // ── 发现 ──
    pub discovery: DiscoveryConfig,
    // ── 插件 ──
    pub plugins: PluginConfig,
    // ── 同步 ──
    pub sync: SyncConfig,
    // ── 修剪 / 隐私 ──
    pub privacy: PrivacyConfig,
    // ── UI（前端镜像，内核只存不改）──
    pub ui: UiConfig,
    // ── 更新 ──
    pub update: UpdateConfig,
}

#[derive(Debug, Clone, Serialize, Deserialize, PartialEq)]
#[serde(rename_all = "camelCase", default)]
pub struct PluginConfig {
    pub auto_sync: SyncMode,                      // notify_only (默认) | auto_push | off
    pub default_registry: String,                 // https://registry.npmjs.org
    pub registry_mirrors: Vec<RegistryMirrorConfig>,
    pub require_signature: bool,                  // ★ true，且不提供关闭开关
    pub allow_local_install: bool,                 // false（生产）
    pub default_quota: QuotaTier,                 // standard | relaxed
    pub max_installed: u32,                       // 默认 100
    pub auto_update: UpdatePolicy,                // notify_only (默认) | auto_minor | auto_all | never
    pub keep_old_versions: u32,                   // 默认 2
}

#[derive(Debug, Clone, Serialize, Deserialize, PartialEq)]
#[serde(rename_all = "camelCase", default)]
pub struct PrivacyConfig {
    /// 是否收集匿名使用统计（默认关闭）
    pub telemetry: bool,                          // false
    /// 剪贴板同步是否默认开启
    pub clipboard_sync_enabled: bool,             // false
    /// 是否记录插件网络访问审计（默认开启，本地）
    pub plugin_network_audit: bool,               // true
    /// 审计保留时长
    pub audit_retention_days: u32,                // 默认 7
    /// 日志保留时长
    pub log_retention_days: u32,                  // 默认 14
}
```

### 4.4 配置变更通知

```rust
impl Config {
    /// 应用一个补丁，成功后发布事件
    pub async fn patch(&self, patch: ConfigPatch) -> Result<ConfigSnapshot, ConfigError> {
        // 1. 校验（各域的 validate）
        // 2. 在事务里写 DB
        // 3. 计算变化集
        // 4. 通知各子系统（例如网络端口变了 → 重启监听）
        // 5. 发布 KernelEvent::ConfigChanged
    }
}
```

**关键**：某些配置变更需要**副作用**（重启监听、重载插件策略）。这些副作用必须由内核处理，不是"改完就完"。

---

## 5. 密钥与敏感数据

### 5.1 密钥分类

| 密钥                       | 用途                            | 存储                                    |
| -------------------------- | ------------------------------- | --------------------------------------- |
| 设备身份密钥（可选）         | 握手 challenge/response、对端认证 | OS keychain（若可用），否则加密文件       |
| 插件 secure storage         | 插件存的 token/密码              | 每插件独立，加密，密钥在 keychain          |
| 发布者私钥                  | 只属于作者，**不在用户设备上**    | 作者的机器 / CI secret                    |

### 5.2 后端选择

```rust
#[async_trait]
pub trait SecretBackend: Send + Sync {
    async fn get(&self, key: &SecretKey) -> Result<Option<SecretBytes>, SecretError>;
    async fn set(&self, key: &SecretKey, value: &[u8]) -> Result<(), SecretError>;
    async fn delete(&self, key: &SecretKey) -> Result<(), SecretError>;
    fn kind(&self) -> SecretBackendKind;
}

pub enum SecretBackendKind {
    /// Windows Credential Manager
    WindowsCredential,
    /// macOS Keychain
    MacKeychain,
    /// Linux Secret Service (gnome-keyring / kwallet)
    LinuxSecretService,
    /// Android Keystore / EncryptedSharedPreferences
    AndroidKeystore,
    /// iOS Keychain
    IosKeychain,
    /// 加密文件（兜底：无 OS keychain 时的 Linux 环境）
    /// ★ 密钥派生自用户密码或存在一个 0600 的密钥文件
    EncryptedFile,
}
```

**降级链**：OS keychain → 加密文件（`EncryptedFile`）。

**必须告知用户**：当使用 `EncryptedFile` 时，UI 上提示"当前系统没有可用的密钥链，敏感数据以本地加密文件保存"。

### 5.3 插件 secure storage

```
key 布局（DB）：
  plugin_secrets(plugin_id, key, nonce, ciphertext, created_at, updated_at)

加密：
  · 每插件一个数据密钥（DEK），随机 32 字节
  · DEK 由 OS keychain 里的主密钥（KEK）包装后存在 DB
  · 使用 XChaCha20-Poly1305（AEAD）
  · 每次写入用新随机 nonce
```

**能力**：`storage:secure:read` / `storage:secure:write`（与 `storage:local` 分开授权）。

### 5.4 密钥轮换与备份

- **不提供**"导出密钥"功能（避免误传）；
- 设备身份密钥可以**重新生成**（后果：其它设备需要重新配对）；
- 插件 secure storage 的数据在**卸载插件**时询问是否删除。

---

## 6. 数据清理与 GC

| 数据                       | 策略                                                       |
| -------------------------- | ---------------------------------------------------------- |
| 编译缓存                   | LRU，上限（默认 512 MiB）；过期（wasmtime 版本变化）的立即删 |
| catalog 缓存               | TTL 7 天                                                   |
| 审计记录                   | 保留 `audit_retention_days`（默认 7 天）                     |
| 日志                       | 保留 `log_retention_days`（默认 14 天），单文件上限 64 MiB   |
| 传输会话（含部分内容）       | `expires_at` 过期即删（含 staging 文件）                     |
| 意图（离线排队）             | 完成后 24h 清理；或用户手动清                                |
| 诊断包                     | 保留最近 10 个                                              |
| `devices_seen`             | 保留最近 500 条                                            |

```rust
pub struct GcTask {
    pub schedule: Duration,      // 默认 1 小时
}

impl GcTask {
    /// 作为受监督任务运行（[07 §5.2](./07-core-kernel.md)）
    pub async fn run(&self, paths: &Paths, store: &Store, cfg: &Config) -> Result<GcReport>;
}
```

  ★ **已实现的部分（第 66 轮）**：意图的 24 小时保留。
  `Runtime::collect_intents` 做扫描，守护进程在**自己的线程**上每小时跑一次（首次等 **1 分钟**，
  因为设计要求 GC 在**空闲时**跑，而刚启动的守护进程正在迁移数据库、读插件、认领目录）。
  等待是**切片的**，所以停机不会等完整个间隔。
  ★ 而 `collect_intents` 之前**没有调用方** —— 一条写在文档里的保留规则，而没有东西跑它。
  现在有了，并且它的时间表是**参数**，所以测试用毫秒而不是等一小时。

  ★ **而 `devices_seen` 的 500 条上限也有了（第 67 轮）**：同一个扫描周期里一并跑。
  它是这一节里**唯一真正在填满**的目标：写它的是发现线程，而在共享网络上（办公室、
  咖啡馆、火车）守护进程每几分钟就听到一台新设备，一直到它停下来。
  ★ 排序里的**标识符作为第二键**不是装饰：同一毫秒听到的两台设备
  如果没有它，保留哪一台取决于 SQLite 恰好返回的顺序 —— 两次扫描可能保留不同的行，
  而任何测试都无法对它断言任何事。

> ### ★ 而这张表里每一行的**真实状态**（第 68 轮逐条核对）
>
> | 目标 | 真实状态 |
> | --- | --- |
> | 意图（离线队列） | **已实现**（第 62–66 轮）：存、读回、重启不丢、定时清理 |
> | `devices_seen` 500 条 | **已实现**（第 67 轮） |
> | 日志 | **已有，但不是这里写的那样**：`RotatingFile` 按 **8 MiB** 轮转并保留 N 份，而不是单文件 64 MiB 上限。总量有界，所以没有按天数删除的需要 |
> | 崩溃报告 | **已有**：`CrashStore::prune`，有测试 |
> | 诊断包 | **本轮之前没有调用方**：`BundlePlan` 写好、测试齐全（含两道脱敏和摘要），而生产代码里 **0 个** `BundlePlan::plan`。第 68 轮接上了 `synra doctor --bundle` |
> | 编译缓存（LRU / 512 MiB） | **没有对象**：全仓 **0 处** `wasmtime::Cache`，也没有任何代码读写 `wasm_cache_dir()` |
> | catalog 缓存（TTL 7 天） | **没有对象**：`catalog_cache_dir()` 存在，而 `Paths` 之外无人读写它 |
> | 审计记录（7 天） | **已实现**（第 69 轮）：`record_audit` + `forget_audit_before`，守护进程通过 `with_audit_sink` 写入 |
> | 传输会话 | **没有写入方**：`transfer_sessions` 表空着 |
>
> ### ★ 而 `plugin_audit` 的写入方，来自一个回调（第 69 轮）
>
> 它自己的注释早就写了它是什么：**内存里是一个环，而持久的那半才让界面的
> “最近活动”活过重启**。环一直在被写（每次调用后 `drain_audit`），而表从来没人写 ——
> 所以用户看到的审计是**本进程**最近一万次调用，一重启就空了。
>
> ★ **而写入方为什么是回调而不是一个库句柄**：`synra-core` 决定“这次调用该被审计”，
> 而**宿主**才有数据库 —— 它不依赖 `rusqlite`、也不知道路径。这和 `decide` 返回一个
> 自己发不出去的信封、由守护进程发出去，是同一个分法。
>
> ★ 而它**连一条测试都当场推翻了自己**：第一版测试跑一次 scripted 匹配并断言 sink
> 收到了东西，而它什么都没收到 —— `ScriptedBackend` 从表里取答案，**没有 guest 跑起来**，
> 也就没有 host 函数被调用、没有东西被审计。现在它直接驱动 `drain_audit` ——
> **这一轮改的那一个函数** —— 并且断言 sink 看到的和环持有的是**同一列记录**。
>
> ★ **所以“实现 GC”在这里已经不是一件事**：两行完成、两行由别的机制守住、
> **四行根本没有对象**。后四行写收集器只会得到永远返回 0 的函数 ——
> 而它们真正需要的是**先有东西去写**。
>

**关键要求**：
- GC **不删除用户数据**（`plugin-data/files/`）；
- GC 不删除 `pinned` 版本；
- GC 报告删了什么（写入日志 + 可查）；
- GC 在**空闲时**运行（不与应用启动争 IO）。

---

> ### ★ 而这一节里**三张表一直是空的**（第 62 轮查出，并修了一张）
>
> 迁移建了 `plugin_audit`、`transfer_sessions`、`intents` 三张表，而在**两个工作区里都没有任何代码**
> 读或写它们 —— 搜 `(FROM|INTO|UPDATE|TABLE)` + 表名只命中 `migrations.rs` 里的 `CREATE TABLE`。
> 于是：
>
> | 表 | 状态 | 后果 |
> | --- | --- | --- |
> | `intents` | **已修**，见下面 | 之前意图只在内存 `Board` 里，**重启就丢** |
> | `plugin_audit` | 仍空：审计写在内存环里 | 需要先决定它该不该落盘 |
> | `transfer_sessions` | 仍空：传输不写库 | 本节的“`expires_at` 过期即删”目前无对象 |
>
> ★ **所以“实现 GC”这件事本身要先分清楚**：对着一张没人写的表做收集，
> 是一个**不可能失败**的功能 —— 正是这个仓库在别处一直在清理的形状。
> 所以 `plugin_audit` 和 `transfer_sessions` 的收集器**没有写**：写进去只会是一个永远返回 0 的函数。
>
> ★ 而 **`devices_seen` 的保留（500 条）也还没写**：它是唯一真有数据的表，
> 但这一轮没顾上，记在这里。
>
> ### ★ `intents` 到位了（第 62–64 轮）
>
> 1. **存**：`save_intent` / `load_intents` / `forget_finished_intents` / `intent_for_request`。
> 2. **而原来的表格子装不下一个意图** —— 迁移 9 重建了它：
>    `document` 存内核自己序列化的整个意图，旁边的列是**不解析每一行**就能回答的两个查询：
>    保留清理（按 `state`）和幂等查找（按 `origin` + `request`）。
>    旧表的列是在类型存在**之前**设计的，装不下 trace、`Choice`、候选列表、
>    拒绝理由、派发的 `request_id` —— 用它存出来的意图会**忘记谁来回答**，
>    而那比不存更糟，因为它看上去像存成功了。
> 3. **`intents_by_request` 是 UNIQUE**：`Board::raise` 在**内存里**拒绝重复请求，
>    而持久化的意义正是重启不能丢掉这件事。边计的 id 是**新生成**的，
>    所以主键拦不住重复 —— 这个索引把它从板子记得的规则变成数据库强制的。
> 4. **读回**：`Intent` 及其嵌套类型现在有 serde，`Board::restore` / `all` 把它放回去；
>    `restore` **不是** `raise` —— 后者会另取 id 和 trace，于是重发的请求会跑第二次。
>
> ★ **而守护进程已经接上了（第 65 轮）**：`Runtime::restore_intents` 在**绑定套接字之前**
> 跑（和数据库先迁移同一个理由：客户端连上来时，板子必须已经是它声称的那个），
> 而 `checkpoint_intent` 在三条修改路径上被调用。于是**重启不再丢意图**。
>
> | 位置 | 为什么在这里 |
> | --- | --- |
> | 绑定套接字**之前** | 一个连上来就能问的内核，板子必须已经是它声称的那个。否则重发的请求会被当成新的，而动作跑第二次 |
> | 每次变化**之后** | 存的是板子现在的样子，而不是另一个关于它的意见 |
>
> ★ **写入失败只记日志，不拒绝** —— 因为动作**已经发生了**。
> 一个在插件跑完之后回答“没成功”的命令，是在告诉用户什么都没发生而实际发生了。
> 这是守护进程里唯一一处把失败留成日志而不变拒绝的地方，理由是它发生在**不可撤销点之后**。
>
> ★ **而它当场就捕到了一个自己写的错**：第一版读的是 `answer["intentId"]`，
> 而 `Routed::to_json` 的键叫 `intent`。缺失的键取 `as_str()` 是 `None` ——
> 所以那行代码**永远不会被执行**，持久化静默地什么都不做。阴性对照：
> 把那两行删掉 → `an_intent_survives_a_restart` 和 `a_restart_does_not_bring_back_an_intent_twice` 变红。
>
> ★ **凭什么说 `intents` 之前是死的**：内存 `Board` 是 `HashMap<IntentId, Intent>`，
> 而全仓没有 `load_intents`、也没有任何 `INTO intents` —— 所以进程一退，意图全部消失。
> 这也是为什么本节的“离线排队”当时不可能成立。
## 7. 备份与恢复

| 数据              | 备份方式                                    | 说明                        |
| ----------------- | ------------------------------------------- | --------------------------- |
| DB（identity/配对/插件记录） | SQLite 在线备份 API（`VACUUM INTO`） | 用户可一键导出              |
| 插件 artifact      | 不备份（可从 registry 重装）                  | 只在 `source.json` 里记录来源 |
| 插件数据 `files/`   | 用户手动（或"导出插件数据"功能）              | 体积可能大                   |
| 密钥              | 不备份（安全考虑）                            | 丢失即重新配对               |

**导入/导出格式**（`.synrabackup`，zip）：
```
manifest.json           # 备份元数据（版本、时间、包含什么）
kernel.db               # SQLite dump
config.json
plugins-manifest.json   # 已装插件清单（id + version + source），用于恢复后重装
```

**恢复流程**：
1. 校验备份文件版本兼容；
2. 恢复 DB（新连接，替换）；
3. 提示用户"需要重新安装以下插件"（因为有 artifact）；
4. 一键重装（从记录的 source）。

---

## 8. 实现清单

> ★ 这一节在实现之前写就，长期整节未勾选。逐条核对源码后：**只有路径、SQLite 接线、沙箱逃逸防护三样是真的**，
> 其余九条是"类型有、实现没有"或完全缺失。名字对不上的按真实名字记。

- [x] `Paths` + 目录创建 —— **A/半个**。`Paths` 存在且字段私有（`crates/synra-store/src/paths.rs:32`），
      `ensure_directories`（`:172`）有真实创建测试（`:440-469`）。
      ★ **"平台相关"缺失**：`Paths::new(data_dir, cache_dir)` 把目录当**参数**收，
      `Path::discover` 不存在、`directories` 也不是依赖 —— 平台解析根本没有。
      ★ **"权限"完全缺失**：全仓 0 处 `set_permissions`/`PermissionsExt`，没有 0700/0600。
- [x] `rusqlite` + `bundled` + WAL —— **C**：`rusqlite 0.40` + `bundled` 在
      `crates/synra-store/Cargo.toml`；WAL 在 `db/migrations.rs:228`。
      ★ `rusqlite_migration` **不是依赖**，用手写 `MIGRATIONS` 常量 + `db::apply` 取代。
- [ ] 5 个初始迁移脚本 + CI 对"空库/旧库"各跑一次 —— **实际是 9 个**迁移（第 106 轮核对：写这一行时是 7 个，后来又加了两个）
      （`db/migrations.rs:18-209`），不是 5 个。空库有真实 SQLite 测试（`sqlite/tests.rs:18`）；
      "旧库"只在纯模型层覆盖（`db/tests.rs:173` 用闭包而非真库）；
      ★ **CI 不存在**（无 `.github/`），所以"各跑一次"这半没有。
- [ ] `Store::write` / `Store::read` + SQLITE_BUSY 重试 —— **未做**。
      **没有 `Store` 类型**。真实的是 `SynraDb`（`sqlite/mod.rs:115`）：持**单个** `Connection`，
      没有 write/read 闭包，**没有 SQLITE_BUSY 重试** —— 只有一条 `busy_timeout=5000` pragma。
      §2.5 的整个形状不存在。
- [~] 审计的落盘 —— **两个宿主都有了**（第 69、81轮）：
      `SynraDb::record_audit` + `PluginSet::with_audit_sink`，守护进程与 Tauri shell 各装一个 sink。
      ★ **而 shell 那一半是这一轮才补上的**：`with_audit_sink` 写给了守护进程，
      而它是**唯一**调用者 —— 所以桌面 shell 里插件的审计只在内存环里，
      **随进程结束而消失**，而迁移文件里写的正是“持久那半才能让‘最近活动’活过重启”。
      ★ **而异步批量提交仍然没做**：现在是每次调用一次写入，
      而设计要的是批量。它是**性能**而不是正确性的差别，所以留在这里而不勾。
- [x] `PluginDataRoot` + 沙箱逃逸防护 —— **A，但"句柄式"是 B**。
      真名 `PluginDataRoot`（`synra-store/src/sandbox.rs:99`）；`resolve()`(`:198`) 先 canonicalize
      父目录(`:229`)、逃逸在 `:236` 被拒、判定用 `paths.rs` 的 `is_within`；真实逃逸测试含 symlink(`:474`)
      + proptest(`:705`)。
      ★ **`cap-std` 不是依赖**，实现是 canonicalize-then-compare，模块自己的注释（`sandbox.rs:14-17`）
      就写明 cap-std 才是目标。**"消除 TOCTOU"没有做到**。
- [ ] 配置分层（config.json + DB + 环境变量）+ `ConfigPatch` + 副作用处理 ——
      ★ **第 106 轮核对：这一行已经过时了。** 写它的时候"五层里两层有"，现在是四层：
      文件那层是一个**完整模块**（`crates/synra-core/src/config/`，9 个文件、533 行测试），
      默认值与覆盖分开（`is_overridden`）、**原子应用**（`apply(set, unset)`：要么全生效要么全不）、
      `load_json` **读一个不信任的文件**（未知键**保留**而不是丢弃）、`to_json` 写回，而 `kernel.rs` 已经在用它。
      ★ **而这两句仍然成立**，所以它们留在这里而不是被我一起删掉：
      `ConfigPatch` 只存在于 IPC DTO（`synra-ipc/src/dto/config.rs:32`）**且无消费方**（`synra-ipc` 之外两处命中都只是注释）；
      环境变量只有 `SYNRA_DATA_DIR`（`apps/synra-cli/src/session.rs:32`），无 `Config::patch`，无副作用处理
      （`KernelEvent::ConfigChanged` 只带一个计数）。**缺的是消费方与那一层环境变量，不是配置本身。**
- [ ] `SecretBackend` 五平台实现 + 降级链 + 用户提示 —— **枚举齐全，实现为零**。
      trait 在 `secrets/backend.rs:18`，六个 kind 齐（`kind.rs:10-25`），
      **但只有一个实现：`MemoryBackend`（`backend.rs:63`）**。五个平台 keychain、`EncryptedFile` 全 0 个。
      `choose_backend`(`:123`) 是按 strength 取最大 —— 是**偏好排序而不是降级链**，且没有可降级目标。
- [ ] 插件 secure storage（XChaCha20-Poly1305 + 每插件 DEK）—— **完全缺失**：
      全仓 0 处 `chacha`/`aead`/`DEK`；`plugin_secrets` 表不在任何迁移里。
- [ ] `GcTask` + 清理策略 + GC 报告 —— ★ **两个类型仍缺失，但"清理策略"已经不是了**：
      ★ **而第 110 轮两个类型都写了**：`crates/synra-daemon/src/runtime/intents.rs` 的
      `GcReport`（带 `failures`）与 `GcTask`（调度作为一个值），`serve.rs` 的 `collect_loop` 改取 `GcTask`。
      ★ **而这一行本身就是证据**：拿掉声明后不到一轮，`doc-claims` 守卫就报了
      `GcReport` “文档说 0、代码有 7” —— 而它不过是在做它该做的事。
      旧的一句保留在这里，因为它记录了之前的状态 —— **当时**：`GcReport` 全仓 0 命中；
      `GcTask` 只有 **1** 命中且是一句注释。
      ★ **而“当时”两个字就是关键**：不写它，`doc-claims` 会把这句历史当成
      当下的声明去数，于是报告一个**准确叙述过去**的句子是错的。工具对此的规则是
      看数字前面有没有 `原状`/`当时`/`修前`。（`apps/synra-daemon/src/serve.rs:36`）。
      而 `runtime::intents::sweep` 已经在跑（每小时一次，带 `Retention { intents_ms, audit_ms }`），
      第 92 轮还有一条**只允许它动自己那三张表**的测试。**缺的是那两个类型与一份可读的报告，不是清理本身。**
      ★ **写入那一半本轮做了：`SynraDb::backup_to` + `synra db backup <path>`。**
      用 `VACUUM INTO ?1`（**绑定参数**，不是拼字符串：路径来自用户，
      引号会结束 SQL 字面量），而不是拷贝文件 ——
      活数据库还有 WAL，只拷主文件得到的是一个**可能打不开**的库。
      目标已存在时**拒绝而不覆盖**：调用方通常是在做危险操作**之前**备份的用户，
      而工具无法知道那个文件是不是他想保留的那个。
      测试断言的是**备份能被引擎打开并 migrate**，而不是"文件存在" ——
      后者对一个没有写完的副本也成立。
      **还没做的：恢复（导入）与导出。**
      `db/health.rs:42` 只**报告**损坏并提示"restore from a backup"，没有恢复路径。
- [ ] 磁盘配额检查（安装前检查可用空间）—— **缺失**：`doctor/storage.rs:66` 的
      `probe_disk_space` 是**诊断探针**，不是安装前检查；`Quota::max_storage_bytes` 是单插件运行时配额。
      ★ **第 128 轮补上了这块缺的能力，但没补上这道闸**，所以本行仍然不勾：
      **缺的不是"检查"而是"可读的数字"** —— 探针一直能说"这个目录写得进去"，却不能说**还剩多少**，
      因为标准库没有可移植的空闲空间查询，而"free some disk space"是用户唯一能执行的建议。
      现在有 `doctor::free_space(path) -> Option<u64>`（`doctor/storage.rs`），探针的成功详情报告真实数字
      （`… ; N bytes are free`），读不到时才退回"this host did not report the free space"。
      ★ **它靠 `sysinfo` 而不是 FFI**：工作区 `unsafe_code = "forbid"`，所以 `GetDiskFreeSpaceExW` / `statvfs`
      这条路在本仓库里写不出来；`sysinfo` 读挂载表是安全 Rust，且已在离线缓存里（0.39.6），所以这条**不是**被环境挡住的。
      ★ 取**最长匹配的挂载点**：`/` 是所有路径的前缀，取第一个匹配会把根文件系统的剩余空间报给另一个挂载卷上的路径 ——
      一个真实但说错盘的数字，比没有数字更糟。这与第 124 轮那条 `rename_all` 的教训是同一个。
      ★ **而闸门仍未接**：安装路径还没有拿 `free_space` / `is_low_disk_space` / `LOW_DISK_BYTES`
      在解包前拒绝一次安装。能力有了，用它的人还没有 —— 这正是本轮不该顺手勾掉本行的原因。
      ★ 另外，本轮为了跑全工作区测试先撞上了 `LNK1180` / `LNK1285`：清掉 `C:\cargo-target\**\*.pdb`
      释放 4.57 GB 后 63 个套件 3025 条全过。**那是磁盘，不是代码。**
