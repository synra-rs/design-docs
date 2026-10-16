# 16 · 插件分发

> 本文定义插件从发布者到用户设备的**完整信任链**，以及设备之间的**同步协议**。

---

## 1. 分发的三条路径

```
┌────────────────────────────────────────────────────────────────────────────┐
│  路径 A：Registry 安装（公开来源）                                           │
│                                                                            │
│  发布者 ──sign──▶ npm 兼容 registry ──GET──▶ 用户设备 ──verify──▶ 安装       │
│                       │                                                    │
│                       └─ packument（元数据 + 公开签名 + 摘要）                │
└────────────────────────────────────────────────────────────────────────────┘

┌────────────────────────────────────────────────────────────────────────────┐
│  路径 B：设备间同步（已配对设备）                                             │
│                                                                            │
│  设备 A（已装） ──file.transfer.* + sha256──▶ 设备 B                        │
│                    │                                                       │
│                    └─ B ★ 必须重新执行完整 verify（不因为是"内部"就跳过）      │
└────────────────────────────────────────────────────────────────────────────┘

┌────────────────────────────────────────────────────────────────────────────┐
│  路径 C：本地开发（仅 dev）                                                  │
│                                                                            │
│  开发者 ──synra-plugin build──▶ 本地已构建 artifact ──dev 通道──▶ 宿主       │
│                    │                                                       │
│                    └─ ★ 生产构建里完全禁用 Local 来源                        │
└────────────────────────────────────────────────────────────────────────────┘
```

**关键原则**：**没有"可信来源"这一说**。无论从哪来，都要走同一套 `Verifying` 流程（[14 §2.3](./14-plugin-lifecycle.md)）。
唯一的差别是**信任等级**（影响默认配额与自动更新），**不是**是否校验。

---

## 2. 包格式与摘要

### 2.1 Artifact 类型

| artifact    | 文件              | 上限       | 摘要算法  |
| ----------- | ----------------- | ---------- | --------- |
| Headless    | `plugin.wasm`     | 32 MiB     | SHA-256   |
| UI          | `ui.zip`          | 16 MiB     | SHA-256   |
| Manifest    | `manifest.json`   | 256 KiB    | SHA-256（含在签名内） |
| 整个包       | `<name>.synra`    | 64 MiB     | SHA-256   |

### 2.2 摘要的表达

使用 **SRI（Subresource Integrity）** 风格，与 npm 生态兼容：

```
sha256-<base64(sha256(bytes))>
```

```rust
#[derive(Debug, Clone, PartialEq, Eq, Serialize, Deserialize)]
#[serde(transparent)]
pub struct SriDigest(String);       // "sha256-47DEQpj8HBSa+/TImW+5JCeuQeRkm5NMpJWZG3hSuFU="

impl SriDigest {
    pub fn compute(bytes: &[u8]) -> Self;
    pub fn parse(s: &str) -> Result<Self, DigestParseError>;
}
```

> **新增**：`dist.synra.signature`（见 §3）。

---

## 3. 信任链与签名

### 3.1 为什么必须签名

- `_synra.plugin.installed` 事件里有 `signature` 字段，但**从不填充**；
- 任何能连上你设备的人（或能篡改 registry 响应的人）都能推送任意代码并被安装执行；
- 用户在用户机上跑 `pnpm install` + `build`，等于执行任意 npm 包脚本。

在新的架构下（分发**已构建的 wasm**），签名的必要性更强：wasm 一旦执行，就能调用所有被授予的 capability。

### 3.2 签名方案

```
算法：Ed25519（快速、小签名、广泛支持）
被签名内容：规范化后的 manifest + 各 artifact 的 SRI 摘要

signature.json:
{
  "version": 1,
  "publisher": { "id": "synra-official", "pubkey": "<base64 ed25519 pubkey>" },
  "algorithm": "ed25519",
  "signedAt": "2026-07-22T10:00:00Z",
  "subject": {
    "manifest": "sha256-...",
    "artifacts": {
      "plugin.wasm": "sha256-...",
      "ui.zip": "sha256-..."
    }
  },
  "signature": "<base64 ed25519 signature over canonical subject>"
}
```

**规范化（canonicalization）**：签名对象是**确定性序列化**的 `subject`（字段顺序固定、无空白、UTF-8）——用一个专门的 `canonical_json` 函数，**不用** `serde_json::to_string`（后者顺序依赖结构体字段序，跨版本可能变）。

```rust
pub fn canonical_signing_payload(subject: &SigningSubject) -> Vec<u8> {
    // 1. 手工按固定顺序构造 JSON（或用 BTreeMap 保证键序）
    // 2. 不使用 serde_json::to_string_pretty
    // 3. 不包含 signature 字段本身
    // 4. 字符串先做 NFC 归一化
}
```

### 3.3 发布者身份与信任

```rust
pub struct Publisher {
    pub id: String,                  // "synra-official" / "imba97"
    pub name: Option<String>,
    pub pubkey: VerifyingKey,        // ed25519
    pub url: Option<Url>,
}

/// 本地信任列表（用户的信任决策）
pub struct PublisherTrustStore {
    trusted: HashMap<String, PubkeyFingerprint>,     // publisher id → 已信任的公钥指纹
    revoked: HashSet<PubkeyFingerprint>,
}

impl PublisherTrustStore {
    /// 判断一个签名属于哪个信任等级
    pub fn classify(&self, publisher: &Publisher, plugin_id: &PluginId) -> PluginTrust {
        if is_builtin(plugin_id) { return PluginTrust::Builtin; }
        match self.trusted.get(&publisher.id) {
            Some(fp) if fp == &publisher.fingerprint() => PluginTrust::SignedTrustedPublisher,
            Some(_) => PluginTrust::Untrusted,   // ★ publisher id 相同但公钥变了 → 不可信
            None => PluginTrust::SignedUnknownPublisher,
        }
    }
}
```

**关键规则**：

1. **publisher id 与公钥绑定**。如果 `@synra-plugin/chat` 的发布者 id 是 `synra-official`，但公钥换了 → **视为 Untrusted**（防止 id 伪造 / 密钥轮换未告知）；
2. **密钥轮换**需要一个显式的"ephemeral key + 主密钥签名"机制（v2 再做，第一版只支持一把密钥）；
3. **撤销列表**（revoked）可以来自内置的撤销清单（随应用更新）或用户手动。

### 3.4 TOFU（Trust On First Use）

第一次安装一个发布者的插件时：

```
UI 显示：
  ┌────────────────────────────────────────────────┐
  │  首次安装来自 "imba97" 的插件                    │
  │                                                │
  │  发布者公钥指纹：                               │
  │    A1B2 C3D4 E5F6 7890 ...                     │
  │                                                │
  │  来源：registry.npmjs.org/@imba97/synra-plugin-…│
  │                                                │
  │  ☐ 信任此发布者的后续插件（不再询问）             │
  │                                                │
  │         [取消]            [安装]                │
  └────────────────────────────────────────────────┘
```

- 用户可以"只信任这一次"或"信任此发布者"；
- 信任后写入 `PublisherTrustStore`（publisher id + 公钥指纹）；
- **指纹要显示**（用户可以与自己知道的对比，类似 SSH 首次连接）。

### 3.5 撤销与安全公告

```
内置撤销清单（随应用更新分发）：
  {
    "revoked": [
      { "plugin": "@bad/plugin", "versions": ["1.0.0", "1.0.1"], "reason": "恶意代码", "at": "..." },
      { "publisher": "evil-actor", "pubkeyFingerprint": "...", "reason": "..." }
    ]
  }
```

行为：
- 已安装的被撤销版本 → **立即停用**（Faulted with reason `Revoked`）+ 通知用户；
- 尝试安装被撤销版本 → 拒绝，附原因；
- 撤销清单通过 Tauri updater 或 registry 的 `security-advisories` 端点获取（离线时用内置的）。

---

## 4. Registry 协议

### 4.1 元数据（packument）扩展

在 npm 兼容 packument 上增加 `synra` 字段：

```jsonc
{
  "name": "@synra-plugin/chat",
  "dist-tags": { "latest": "1.2.0" },
  "versions": {
    "1.2.0": {
      "name": "@synra-plugin/chat",
      "version": "1.2.0",
      "dist": {
        "tarball": "https://registry.example.com/@synra-plugin/chat/-/chat-1.2.0.tgz",
        "integrity": "sha512-...",
        "synra": {
          "signature": { /* signature.json 的内容或它的 URL */ },
          "manifestDigest": "sha256-...",
          "artifacts": {
            "plugin.wasm": "sha256-...",
            "ui.zip": "sha256-..."
          },
          "capabilities": ["device:send:*", "storage:local:read"],   // 便于搜索/预检
          "runtime": { "headless": true, "ui": true },
          "size": { "wasm": 1234567, "ui": 89012 }
        }
      },
      "synra": { /* 完整 manifest 内联（便于预检与 catalog 搜索）*/ }
    }
  }
}
```

**内联 manifest 的价值**：安装前就能展示权限、体积、截屏等（不下载 artifact）。


```rust
pub struct RegistryConfig {
    pub default: Url,                                 // https://registry.npmjs.org
    pub mirrors: Vec<RegistryMirror>,                  // 按顺序尝试
    pub overrides: Vec<(PackagePattern, Url)>,         // 精确 > 通配
}

pub struct RegistryMirror {
    pub url: Url,
    pub enabled: bool,
    pub priority: u8,
}
```

**优先级**：
```
1. overrides 中精确匹配的包
2. overrides 中通配匹配的包
3. mirrors 中按 priority
4. default
```

**回退**：一个 registry 失败（超时/404/校验失败）→ 尝试下一个，**但**：
- 如果失败原因是**签名/摘要校验失败** → **不回退**（可能是中间人攻击，回退等于跳过校验）；
- 只对"网络错误 / 404 / 429"回退。


### 4.3 Catalog 搜索

```rust
pub struct CatalogQuery {
    pub text: Option<String>,
    pub capabilities: Vec<Capability>,      // 只显示需要某能力的插件
    pub runtime: Option<RuntimeKind>,
    pub publisher: Option<String>,
    pub sort: CatalogSort,                  // Relevance | Downloads | Updated | Name
    pub page: u32,
    pub per_page: u32,
}

pub struct CatalogItem {
    pub plugin_id: PluginId,
    pub version: Semver,
    pub title: String,
    pub description: Option<String>,
    pub icon_url: Option<Url>,
    pub publisher: PublisherSummary,
    pub trust: PluginTrust,
    pub runtime: RuntimeFlags,
    pub capabilities_summary: Vec<String>,
    pub size_bytes: u64,
    pub installed: Option<InstalledSummary>,     // 本机状态
    pub update_available: bool,
    pub rating: Option<f32>,                     // v2
}
```

**catalog 实现**：由于我们要兼容 npm registry，搜索能力受限（npm 的搜索 API 较弱）。
方案：
1. **主来源**：npm registry 的搜索端点（`/-/v1/search?text=keywords:synra-plugin`）；
2. **可选**：自建 catalog 服务（聚合 + 缓存 + 评分）；


---

## 5. 设备间同步

### 5.1 同步触发

| 触发                     | 默认行为                                            |
| ------------------------ | --------------------------------------------------- |
| 本机安装插件完成           | 通知已配对设备（**不自动推**，见 §5.2）                |
| 用户在某设备点"同步到…"     | 立即推送                                            |
| 用户在新设备上启用某插件     | 检查其它设备是否有 → 提示拉取                         |
| 定时（可选）              | 检查所有设备插件清单是否一致（默认关闭）               |

> Rust 版明确为**策略化**：`autoSync: { mode: NotifyOnly | AutoPush | Off }`，默认 `NotifyOnly`（通知 + 一键同步）。
> 理由：自动推送插件到用户所有设备是**重大安全决策**，不应默认开启。

### 5.2 同步协议

```
① 发起端 A ──▶ B：_synra.plugin.installed.query
                    payload: { pluginId }
                    （envelope request；超时 5s；幂等）

② B ──▶ A：_synra.plugin.installed.query.reply
              payload: { pluginId, version?, digest?, active? }

③ A 判断：
     · B 无 → 需要推送
     · B 有相同 digest → 跳过（skipped）
     · B 有不同版本 → 按冲突策略（§5.3）

④ A ──▶ B：file.transfer.request
              payload: { transferId, kind: 'plugin-bundle',
                         pluginId, version, byteLength, sha256,
                         syncSessionId }

⑤ A ──▶ B：file.transfer.chunk × N
              payload: { ..., chunkIndex, totalChunks }
              ★ 使用会话级 ACK + 滑动窗口（[04 §6](./04-wire-protocol.md)）
              ★ v2 时用二进制帧，v1 时用 base64

⑥ B 每收一块：校验 + 累积 + 定期回 file.transfer.progress
             缺块 → file.transfer.nack { missingIndexes } → A 重传

⑦ A ──▶ B：file.transfer.complete
              payload: { transferId, kind, totalChunks, sha256 }

⑧ B：
     · 校验 sha256（★ 必须）
     · 解包（含归档安全限制）
     · ★ 执行完整的 Verifying 流程（签名、manifest、imports、WIT…）
     · 安装（原子）
     · 按自身策略决定是否启用
     · 回 _synra.plugin.sync.result { ok, installedVersion, activated, error? }

⑨ A 更新同步状态（成功/失败/跳过），UI 显示
```

### 5.3 冲突解决

```rust
#[derive(Debug, Clone, Copy, PartialEq, Eq, Serialize, Deserialize)]
#[serde(rename_all = "snake_case")]
pub enum SyncConflictPolicy {
    /// 不降级：双方各自保留自己的版本（★ 默认）
    NeverDowngrade,
    SourceWins,
    /// 保留较新版本
    KeepNewest,
    /// 每台设备独立，不做一致性保证
    Independent,
    /// 询问用户
    Ask,
}
```

**默认 `NeverDowngrade`** 的理由：
- 自动降级可能让设备上的插件**回退到有漏洞的版本**；
- 用户对"我的设备上的软件被远程降级"会非常抗拒。

### 5.4 同步的安全要求

| - | -------------------------------------------------------------------- | ------ |
| 1 | 接收端**必须**重新校验签名与摘要                                        | ❌     |
| 2 | 接收端**必须**校验整包 sha256                                          | ❌     |
| 3 | 接收端**必须**校验 manifest 与 capability                              | ❌     |
| 4 | 接收端**必须**做归档安全限制（大小/条目/路径穿越）                        | ❌     |
| 5 | 传输**必须**有会话级 ACK（否则"成功"是假的）                             | ❌     |
| 6 | `syncSessionId` **必须**用于隔离多轮同步（防跨轮混块）                    | ❌（从不填） |
| 7 | 接收端的决策（是否启用）**必须**由本地策略决定，不由发起端指定              | ❌（发起端隐含决定） |
| 8 | 同步**不能**覆盖用户在接收端的授权决策（grant 是本地权威）                  | ❌（无 grant） |

### 5.5 同步的可观测性

每次同步产生一条记录：

```rust
pub struct SyncRecord {
    pub id: SyncId,
    pub plugin_id: PluginId,
    pub version: Semver,
    pub direction: SyncDirection,
    pub peer: DeviceId,
    pub started_at: SystemTime,
    pub finished_at: Option<SystemTime>,
    pub outcome: SyncOutcome,       // Succeeded | Skipped { reason } | Failed { error }
    pub bytes_transferred: u64,
    pub trace_id: TraceId,
}
```

UI（设置 → 插件 → 同步历史）展示最近 N 条。

---

## 6. 更新检查

```rust
pub struct UpdateCheck {
    /// 检查哪些插件
    pub targets: UpdateTargets,      // All | Specific(Vec<PluginId>)
    /// 是否包含预发布版本
    pub allow_prerelease: bool,
    /// 并发上限
    pub concurrency: u32,
}

pub struct UpdateInfo {
    pub plugin_id: PluginId,
    pub current: Semver,
    pub available: Semver,
    pub capability_diff: ConsentDiff,        // 是否会更需要权限
    pub artifact_size: u64,
    pub changelog_url: Option<Url>,
    pub requires_consent: bool,              // capability_diff 非空
    pub trust_change: Option<TrustChange>,   // 发布者公钥变了？
}
```

**检查频率**：默认每 24 小时（可在设置里关闭）。
**离线**：跳过，下次联网时检查。


| 条件                                    | 是否自动更新 |
| --------------------------------------- | ------------ |
| 信任等级 < `SignedTrustedPublisher`       | ❌ 仅通知     |
| capability 有新增/放宽                    | ❌ 需确认     |
| 发布者公钥变了                            | ❌ 需确认 + 警告 |
| 当前版本在撤销列表中                      | ✅ 强制更新（或停用） |
| 其余情况                                 | ✅ 自动（若策略为 AutoMinor/AutoAll） |

---

> ★ **而第 79 轮把第一行做成了**：`synra_plugin_host::catalog`：
> `CatalogCache`（`catalog_cache_dir()` 下的 **7 天** TTL）+ `CatalogClient`（按镜像优先级依次尝试，全部失败时回退到缓存）。
>
> ★ **而 TTL 过期不是“没有”**：`Cached::Stale` **仍然把文档交出来**，
> 只是标上年龄。这正是这一行要求的（“用缓存的 packument + 提示离线”）——
> 一个过期就拒绝交出的类型会把“网络慢”变成“目录是空的”。
> ★ **而一个损坏的条目是错误，不是缺失**：吞掉腐败会让本模块的系统性 bug
> 表现为“永远没有缓存”，而没有任何地方说条目存在且读不了。
> ★ **而它抓到了一个真的 Windows 陷阱**：插件名通常由字母数字 `@` `/` `.` 组成，
> 而**全字母数字的名字转义后还是它自己** —— 一个叫 `CON`、`NUL`、`COM1` 的插件
> 会写出 `CON.json`，而在 Windows 上这不是文件：名字在查文件系统**之前**就被解析成设备，
> 写入去了控制台。现在所有缓存文件都带 `packument-` 前缀，没有任何插件名能产生设备名。
>
> ★ **而第 80 轮把它接到了命令上**：`Request::CatalogSearch` + `Kernel::catalog_search`
> + 守护进程的 `runtime::catalog::search` + `synra plugin search <package>`。
> 之前 `plugins_catalog_search` 只在 `command.rs` 的表里 —— **又一个声明了而没人实现的命令**。
> ★ 而它回答里**带着这份目录从哪来**：`origin` 是 `registry`（带镜像）或
> `cache`（带年龄与 `stale`），因为“最新版是 1.2.3”这句话在刚回答的注册表和八天前的缓存里
> 是**两件事**；而没有仿站回答且什么都没缓存时它返回 `DeviceOffline`，不是空列表。
>
> ★ **而这一轮顺手抓到了一个真的依赖崩溃**：`ureq-proto` 0.6.4 在响应头过多时
> 会把东西 push 进一个 **已满的 `ArrayVec<_, 16>`**（`util.rs:178`，其注释写的就是“满了就 panic”）。
> 它是这么被发现的：**解析不了的主机干净地返回错误，而能解析的主机走到了 bug** ——
> 这正是 `synra-fetch` 自己的测试从来没见过的原因：它们全部走 `127.0.0.1:<port>` 的本进程档位。
> 现在 `Client::fetch` 把它包起来**变成一条拒绝**，并且拒绝里就写着“客户端 panic 了”：
> 一句“网络失败”会把下一个读者送去查网络，而**服务端能送回多少个头不是插件能选的**。
>
## 7. 离线与降级

| 场景                       | 行为                                                     |
| -------------------------- | -------------------------------------------------------- |
| Registry 不可达             | 用缓存的 packument（TTL 7 天）；提示"离线模式"              |
| 镜像全部失败                | 失败并提示；不静默回退到"不校验"                            |
| 设备间同步时对端离线         | 排队；对端上线时提示（可配置自动）                          |
| 签名无法校验（缺公钥）       | 拒绝安装，提示"该插件的发布者未知，需要用户显式信任"          |
| 撤销清单更新失败            | 用内置的旧清单；在 UI 上提示"安全清单可能不是最新"            |

**明确不做**：**任何形式的"跳过校验"开关**。
用户可以：
- 选择不安装某个插件；
- 显式信任某个发布者（TOFU）；
- 但不能"关闭签名校验"。


---

## 8. 实现清单

> ★ 这一节在实现之前写就，长期整节未勾选。逐条核对源码后：**签名与摘要那一块是真的且有真测试**；
> **分发/registry 那一块的核心（HTTP 客户端）根本不存在**；同步协议只有一个"接收方义务"表，没有流程。
> 名字对不上的按真实名字记。

- [x] `SriDigest`（SRI 解析）—— **C，且"兼容读取"被明确排除**：
      `crates/synra-protocol/src/session.rs:393` 的 `SriDigest`，`TryFrom<String>` 在 `:438`。
      ★ **无 `compute()`**；★ `DigestAlgorithm`(`:364`) **只有 `Sha256`**，
      `shasum`/SHA-1 全仓 0 命中 —— 兼容读取被有意排除，不是漏了。
      ★ CLI 另有一份**私有** `sri_digest()`（`sign.rs:201`），注释说与 `SriDigest` 的 padding 约定不同 ——
      所以同一个概念现在有两份实现。
- [x] `canonical_signing_payload` + 跨版本稳定性测试 —— **A/C**：
      真名 `SigningSubject::canonical_bytes`（`synra-plugin-api/src/publisher.rs:220`）。
      ★ 测试是真的：`publisher_tests.rs:70`（定序无空白）、`:94`（同内容不同插入序 → 同字节）、
      `:103`（转义）、`:113`（空 map）—— 跨版本稳定性成立。
- [x] `signature.json` 的生成（CLI）与校验（宿主）—— **A**：
      生成 `run.rs:248`；校验 `plugin-host/src/verify/mod.rs`（`Stage::Signature`）
      + `publisher.rs:363` ed25519 验签；测试 `verify_tests.rs:105-170`/`:529`。
- [x] `PublisherTrustStore`（TOFU + 公钥指纹绑定 + 撤销）—— **A，但从不持久化**。
      `publisher.rs:619-710` 的 `PublisherTrustStore{trust/forget/revoke/classify}`，
      测试 `publisher_tests.rs:276-346`、`verify_tests.rs:184-232`。
      ★ **纯内存**：所有生产调用点都传 `PublisherTrustStore::new()`（`run.rs:207` 等），
      也就是**默认谁都不信任、撤销集永远为空**，每次进程启动都回到这个状态。
- [x] 撤销清单的内置分发与生效（停用已装的被撤销版本）—— **已实现**（第 78 轮）：
      `synra_plugin_api::revocation`（上面那个文档格式）+ 守护进程的加载路径。
      ★ **两个行为均已落地**：已安装的被撤销版本被**拒绝加载并写入** `enabled = false`
      （不只是跳过——否则下次启动会再试一次）；`must_disable()` 扫描已装集合，
      也就是"尝试安装被撤销版本也拒绝"那一半。
      ★ **而清单来自数据目录旁的 `revocations.json`**（缺失则为空并记录）——
      "随应用更新分发"意味着**离线情形就是内置情形**，而一份只能从网络拿到的清单
      会让离线机器根本没有撤销数据。
      ★ **而版本是字面比较（精确字符串）而非 semver**：设计点名的是确切版本，
      把它当范围解释会**扩大**一条条目的含义（点名 `1.0.0` 会连 `1.0.0+build` 一起禁掉）。
      代价是反向的风险：拼写不同（`v1.0.0`）会**漏掉**，而漏掉的撤销就是插件继续跑。
      ★ **而第 93 轮把它的“误伤”也守上了**：`revoking_one_plugin_leaves_the_others_alone`。
      撤销会**禁用一行**，而一个匹配得太宽的撤销（比如只比版本、不比插件名）
      会禁用清单根本没点名的插件 —— 用户看到自己安装并授权的插件停止工作，
      理由还是关于一个不是它的版本，而他无从判断发生了什么。
      所以断言不是“被撤销的那个被禁用了”，而是**另一个没有**。
      ★ 取反对照：把匹配改成只看版本 ⇒ 红，
      报出 `{"@bad/plugin": false, "@good/plugin": false}`。
      这正是 `must_disable()` 是"扫描"而不是"用到时检查"的原因：**漏掉在用到时是看不见的**。
- [x] Registry 客户端（packument 解析、镜像优先级、**校验失败不回退**）—— **C/一半，且"客户端"那半是 B**。
      `plugin-host/src/registry.rs:525` `Packument::parse`、`:263-325` 镜像优先级、
      `:713` `check_integrity`、`:120-140` `allows_fallback`
      （`IntegrityMismatch` 不可回退 —— 规则正确且有测试 `registry_tests.rs:248-263`）。
      ★ **没有任何 HTTP 客户端** —— 模块注释自己写明 "the fetching is not here"；
      全仓没有 `RegistryConfig` 的生产调用者。解析与策略有，取件没有。
- [x] `CatalogQuery` / `CatalogItem`（单一实现）—— **A**：
      `synra-ipc/src/dto/plugin.rs:88`/`:112`；TS 侧只有一份生成物
      `synra-js/packages/protocol/src/generated/ui.ts:232`/`:251`。"Electron/Capacitor 双份合并掉"成立。
- [ ] 同步协议 9 步流程 + `_synra.plugin.sync.result` —— **C/一半**：
      事件名存在（`protocol/src/event.rs:363`，入站白名单 `inbound.rs:470`）；
      7 个接收方义务 `ReceiverDuty::ALL`（`plugin-sync.rs:263-284`，测试断言 `== 7`）。
      ★ **没有 9 步流程，也没有任何编排者** —— `synra-core/src/plugin_sync.rs` 只被自己和
      `plugin_sync_tests.rs` 使用，**事件名没有发送方**。
  ★ **第 130 轮做出了接收端那一半**：`ReceiverCheck`（`plugin_sync.rs`）按 §5.2 ⑧ 的顺序推进，
  每一步都要求前一步**已经做过**，跳步或错序都是一个具名的 `SyncError::OutOfOrder`，
  而且被拒的步骤**不留痕迹**（`performed` 只记真正做过的）。走完七步得到的 `SyncOutcome` 里
  `performed` 等于 `ReceiverDuty::ALL` —— 不是构造时填的，是走出来的。
  ★ **仍然没有的**：事件的**发送方**（`_synra.plugin.sync.result` 至今无人发出）、transfer 层的接线
  （④–⑦）、以及 A 侧第 ⑨ 步。所以本行**保持未勾**，缺的是另外两半。
  ★ **第 131 轮把发送方补上了**：`synra_core::sync_result(request, from, outcome, at)`（`messaging.rs`）
  构造 ⑧ 的回复，载荷是`SyncResult`（`ok` / `installedVersion` / `activated` / `error`）。
  它是 `reply_to`，所以带着请求的 id 与 trace —— 发起端要能把结局对回到它发起的那个传输上。
  ★ **内部类型不上线**：`SyncOutcome` 记的是这台设备自己做了哪些义务、顺序如何；
  对方需要四个事实就够了，给它一份它看不到证据的清单不是线上格式。`SyncResult::of` 做这个转换。
  ★ 两条测试：成功的四个字段，以及**失败的不能被看成成功** —— 后者才是这条回复存在的理由。
  而 transfer 层的接线（④–⑦）与 A 侧第 ⑨ 步仍未做，所以本行**保持未勾**。
  ★ **第 132 轮补上 ⑨ 的第一半**：`SyncTable::apply_result(device, plugin_id, result, pushed_digest)`。
  这张表能存能查，但从写下来那天起**只有测试在写**，所以「A 更新同步状态」没有实现；
  表和回复都在，缺的是把它们接起来的那一行。现在 `record` 有了第一个生产调用者。
  ★ **失败时不动这张表，这是语义不是省事**：一次失败的同步没有改变对端持有什么。
  写成「什么也没有」会让下次推送判断成「它需要」而重复传输；写成「新版本」更糟 ——
  表会声称对端有一个它从未收到的包。**失败属于历史（§5.5 的 `SyncRecord`），不属于「对端现在有什么」。**
  ★ 摘要从**参数**来而不是从回复来：回复里没有摘要，也不需要 —— 发起端推的就是那个包，摘要是它自己算的。
  ★ 同时把 `SyncResult` 补成可往返（`Deserialize`）：上一轮只做了「出」，一个只能序列化的线上类型是半成品。
  ★ **仍然没有的**：daemon 侧的编排者（真正调用 `apply_result` 的地方）、失败的历史记录
  （`SyncRecord` 仍只有四个字段，文档要九个）、transfer 层接线（④–⑦）。
  ★ 另外 `ReceiverDuty::ALL` 的顺序（digest → signature → manifest → archive-limits → …）与 §5.2 ⑧ 的
  行文（先解包、再走 Verifying）**不一致**。本轮按枚举自己的顺序实现，因为类型注释写明那就是「执行顺序」；
  这个分歧记在这里，没有擅自改任何一边。
- [x] `SyncConflictPolicy`（默认 `NeverDowngrade`）—— **C/一半**：
      真名 `ConflictPolicy`（`plugin_sync.rs:89`），`decide()`(`:176-228`) 实现了 NeverDowngrade 语义，
      测试 `plugin_sync_tests.rs:33-118`。
      ★ **"默认"只是文档注释**：没有 `impl Default`，也没有任何配置来源；
      `decide` 的**唯一调用者是测试**。
- [x] `syncSessionId` 的真实使用（跨轮隔离）—— **A**：
      `SyncSessionId`(`plugin_sync.rs:34`)；`SyncSession::absorb`(`:449-455`) 对他轮 chunk 返回
      `SyncError::WrongSession`；测试 `plugin_sync_tests.rs:157-199`。隔离是真的，而且是结构性的。
- [x] `SyncRecord` + 同步历史 UI 数据 —— **C/一半**：
      `SyncRecord`(`plugin_sync.rs:602-611`) 存在**但只有四个字段**（`plugin_id`/`peer`/`action`/`at_ms`），
  ★ **第 133 轮把字段补齐了**：`SyncRecord` 现在有 `id`（新类型 `SyncId`）、`version`、
  `direction`（新类型 `SyncDirection`）、`started_at_ms`、`finished_at_ms`、`verdict`、
  `bytes_transferred`、`trace_id`，加上既有的 `plugin_id`/`peer`/`action`。
  ★ **两个刻意的偏离，都记在这里**：文档写 `SystemTime`，实现用**有符号毫秒** —— 仓库里每个时间戳
  都是这个形状，而这条记录还要与它们比较、要能进 SQLite；文档写 `SyncOutcome`，实现叫 `SyncVerdict`，
  因为 `SyncOutcome` 在这个模块里已经是一个**不同**的东西（接收端做过哪些义务的清单）。一个是意图，一个是命名冲突。
  ★ **三态而不是布尔**：`Succeeded` / `Skipped { reason }` / `Failed { error }` ——
  跳过意味着双方已经一致、不需要任何人做什么，失败意味着有人得看一眼。合成一个 `ok: false` 会把两件事说成一件。
  ★ **`finished` 返回 `Result`**：一条记录只能有一个结局，收两次尾会**静默覆盖**第一次 ——
  「先失败后成功」读起来就是成功。这是它返回 `Result` 而不是 `self` 的全部理由，也有测试。
  ★ 仍然没有的：持久化、IPC DTO、前端消费，以及真正创建这些记录的编排者。
  ★ **第 134 轮发现那句话里的「持久化」指错了对象。** `sync_records` 表**从第一个迁移就存在**，
  而且**没有任何 Rust 代码读过或写过它**。但它的形状是每 `(plugin_id, peer_device)` **一行**、
  主键就是这两列 —— 那是**对端现在持有什么**（`plugin_sync::SyncTable` / `RemoteState`），
  不是本轮的日志（`SyncRecord`）。两个东西同名不同物，**表是给前者准备的**：
  内存里的表在进程结束时就没了，重启之后每台设备都像什么都没有，而那正是设计第 ③ 步要避免的重复传输。
  ★ 现在 `SynraDb::sync_records()` / `record_sync(...)`（`sqlite/sync.rs`）读它、写它（upsert），
  两条测试：一次往返，以及**同一对端第二轮是更正那一行而不是多加一行** —— 一台设备两行会让
  「对端已经有了吗」有两个答案。
  ★ **而 `SyncRecord` 的历史仍然没有持久化**：表记的是当前状态，日志要另有一张表。
  所以「持久化」这句现在要分成两半读：状态那半做完了，历史那半没有。
  ★ **第 135 轮把那半也做了**：迁移 **v10** 建了 `sync_log` 表，`SynraDb::record_sync_log()` /
  `sync_log(plugin_id, limit)` 读写它，四条测试。
  ★ **形状照 v9（`intents`）的先例，而且理由写在同一个地方**：*整份文档 + 查询用的投影列*。
  `document` 装内核自己序列化的那一轮，旁边的列只保留**历史页面真正要问的两个问题**
  （最新的在前、按插件过滤）—— 一个为了找某个插件而解析每一行的查询，会随着同步次数变慢。
  ★ **两张表的分工现在写清楚了**：`sync_records` 每个 `(plugin_id, peer_device)` 一行、每轮**替换**，
  回答「对端已经有了吗」；`sync_log` 每轮**追加**，回答「发生了什么、什么时候」。
  ★ 而 `record_sync_log` 也是 upsert：一轮写两次（开始一次、结束一次），第二次是**同一轮**，
  两行会让历史把一轮显示两遍、其中一遍说它从未结束。
  ★ 仍然没有的：core 侧的转换（`SyncRecord` ↔ 这一行，需要给两个枚举加 serde）、
  IPC DTO、前端消费，以及真正写这些行的编排者。
  ★ **第 136 轮把 core 侧的转换做了**：`SyncRecord::to_row()` / `from_row()`（`plugin_sync.rs`），
  `SyncVerdict` / `SyncAction` / `SyncDirection` / `SyncId` / `SyncRecord` 各自加了 serde。
  ★ **写入时投影列与文档同源**（都从同一个 `self` 取），**读回时以 `document` 为准** ——
  与文档不一致的列只可能来自有人手工改库，而那时相信哪一半都不对；让列赢会让历史显示一轮
  从未发生过的同步。坏文档是**错误**而不是猜测，有测试。
  ★ **往返测试走遍三种结局**：`Succeeded` / `Skipped { reason }` / `Failed { error }` 携带的内容不同，
  只测一种会漏掉另一种的字段。
  ★ 仍然没有的：IPC DTO、前端消费，以及真正写这些行的编排者。
  ★ **第 137 轮做了一个查证，结果值得写下来**：daemon **没有 transfer 的接收侧** ——
  它只有发送（`commands.rs` 的 `send_file`），入站路径处理的是**意图**（`runtime/intent.rs` 的 `dispatch`）。
  ★ **而第 138 轮把这句话纠正了：说过头了。** 接收侧**存在于 core** 的 `inbound.rs` ——
  流式写盘、逐块校验、`Decision::Finished { path, byte_length }`，daemon 只是 `pub use` 它。
  真实缺口比那句话**小也更精确**：
  ★ **`Decision::Finished` 不带「这是一个插件包」的信息。** 唯一的消费者（`listener.rs:331`）
  只 `println!` 一句 `received N bytes into <path>` —— 于是一个插件包和一张截图在代码里**没有区别**。
  而 `proposed_name` 的注释自己承认了这一点：*「plugin bundle 还不是这个内核会安装的东西」*。
  ★ **下一轮的形状已经读清**（读清了再写，这是第 136→137 轮验证过的做法）：
  `Receiving`（`session/receive.rs:65`，只有 `receiver` 与 `trace_id` 两个字段，手写 `Debug`）
  应当在 `accept` 时从 `TransferMeta::PluginBundle` 记住 `Option<PluginId>`，
  `Decision::Finished` 带上它，消费者才能区分这两件事。改的是三个文件、一条测试。
  ★ **还有一件必须承认的事**：第 137 轮的 `SyncInbox` 与真实接收路径**形状不同** ——
  真实路径是**流式写盘**，不是「整包在内存的块里」。所以 `SyncInbox` / `ReceiverCheck` 更像是 ⑧ 的
  ★ **第 139 轮把那件「该明确决定的设计问题」做了一半，而且是最实的一步**：
  传输完成时不再只知道一个路径。`Receiving`（`session/receive.rs`）在**接受提案时**从
  `TransferMeta::PluginBundle { plugin_id, version }` 记住 `Option<PluginId>`，
  `Decision::Finished` 带上它，`listener` 于是能说出区别：
  *「a plugin bundle for X, which this kernel does not install yet」* 而不是和一张截图同一句话。
  ★ **为什么字段在 `Receiving` 上**：接受提案时才知道这件事，而完成时手上只剩一个路径 ——
  从 `plugin-<id>.wasm` 反推文件名会在有人发一个叫 `plugin-notes.wasm` 的文件时错掉。
  ★ **第 140 轮把那条欠下的测试补上了，而补它的方式是让它可以被测试。**
  判断本身只是一个纯函数 —— 读元数据的一个字段 —— 却被写在 `Receiving::accept` 里，
  而那个函数需要一整个信封、一个真摘要和一个**可写的目录**才能跑。
  抽成 `bundle_of(&TransferMeta)` 之后，两条测试不需要信封、不需要摘要、不碰磁盘：
  插件包 → `Some(plugin_id)`，附件 → `None`。**一个纯问题不该需要一个文件系统才能回答。**
"
  ★ 而这两条互为对照，所以不需要额外的变异测试：判断永远返回 `None` 会让第一条失败，
  永远返回 `Some` 会让第二条失败 —— 而「把所有传输都当成插件包」正是没有第二条时会发生的错。
"
  ★ `synra-core` 全量 **650** 条通过（648 + 2），clippy 干净。
  **清单化**（把六件事说出来、按序要求），而不是能插进流式接收的东西。要不要把它们接上去，
  还是让流式路径自己履行那六条义务，是一个**该明确决定**的设计问题，不是一句「还没接线」。
  所以第 130 轮那个 `ReceiverCheck` 不是「缺一个触发者」，而是**连入站事件都没有**。
  ★ **以及 ④–⑦ 的库层补上了**：`SyncInbox`（`plugin_sync.rs`）
  —— 会话知道哪一块到了、检查序列知道整包要验什么，而**字节**没有地方放。
  它是刻意无聊的版本：块在内存里、完成时按 **index 而不是到达顺序** 拼装。
  落盘与 socket 循环是 daemon 的事，一个假装做了它们的类型正是本仓库反复清除的东西。
  ★ **记账没有重写**：`SyncSession::absorb` 已经回答「这一块是不是新的、在不在范围内、属不属于这一轮」，
  重写会给出两个答案，而两个答案就是这样开始漂移的。
  ★ 错轮次的块被**计数**（`refused_chunks`）而不是静默丢掉：非零是「有人往这台设备发了一个它不在里面的轮次」
  的唯一证据；而越界与错轮次分开计数 —— 合成一个会让「有人在发错轮次」这个信号被数错冲淡。
  ★ 仍然没有的：daemon 的 socket 接收循环（④–⑦ 的 I/O 那半）、IPC DTO、前端消费、编排者。
  ★ **第 146 轮想接上「收到一个插件包就写一条同步记录」，结论是：那个位置做不到。**
  `Decision` 定义在 `synra-core/src/inbound.rs`，而消费它的 `Decision::Finished` 在
  `synra-core/src/listener.rs` 里 —— 而那里是 **`handle_within` 内部**，那个函数的参数是
  `host / stream / inbox / deadline / desk / router`：**没有数据库**。所以「写一条 `sync_log`」
  在收到包的那一刻没有可写的地方，需要改 `handle_within` 的签名（生产调用点与测试调用点各一处）。
  ★ **而本轮的方向一开始是错的**：我按 `serve_one` 去找，以为它接收 daemon 的 kernel ——
  那是 `synra-control/src/endpoint.rs` 的**控制端点**方法，与这里无关。同名方法在两个 crate 里，
  而我猜了六次文件，而不是一次看清模块结构。**一次 `grep -r` 就能定案的事，我用了六个命令。**
  ★ 三件已经存在却仍无生产调用者的东西，仍然无调用者，而且现在知道**为什么**：
  `SyncRecord`（第 133 轮）、`sync_log` 表（第 135 轮）、两者之间的转换（第 136 轮）——
  它们等的是一个**能写库的接收点**，而接收点的形状（`handle_within`）里没有库。
  ★ 下一轮第一步：改 `handle_within` 的签名，让它在完成时回调一次 ——
  两处调用点（`serve.rs` 与 `transport/tests.rs`），加上一条测试。
      文档要的是 `version`/`direction`/`startedAt`/`finishedAt`/`outcome`/`bytesTransferred`/`traceId`。
      ★ 无 `Default`、无持久化、无 IPC DTO、无前端消费。
- [x] 自动更新门槛表（下面那张表）—— **已实现**（第 77 轮）：
      `synra_plugin_api::update`。`UpdatePolicy`（默认 `NotifyOnly`）、`UpdateFacts`、`UpdateBlocker`、
      `UpdateAction`，以及 `decide(policy, &facts)`。
      ★ **而表里没写的两件事，这里得自己决定并且写了理由**：
      * **撤销压过一切**，包括策略。已安装版本被撤销时不是“询问”而是“必须”——
        否则下载失败时会继续跑一个已被撤回的版本，而那正是这一行存在的原因。
      * **阻碍项全部报出，不只报第一个。**它们常常同时成立，只报一个会让用户修完一个、
        重试、再发现下一个 —— 而更糟的是把“发布者公钥变了”藏在一条权限消息后面。
      ★ **而它故意不做任何动作**：不拉取、不安装、不调度。一个会行动的决策函数没有网络就无法测试。
      （`dto/plugin.rs:72-83`，含 `needs_new_consent`/`added_capabilities`）；
      无 `UpdateCheck` 类型、无调度器、无门槛表。grep `auto_update`/`update_policy` **0 命中**。
- [ ] 离线降级行为 + UI 提示 —— **未做**：`synra-rust-app/src` 里 `offline` 只出现在
      **设备卡片**（`DevicesPage.vue:145`、`stores/devices.test.ts`），与插件目录/降级无关。
