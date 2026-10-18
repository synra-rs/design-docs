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

> ★ **容器定了**：`.synra` 是一个 **zip**，内容就是安装器接受的目录（`manifest.json`、manifest 命名的 artifact、
> 有签名时的 `signature.json`）。解包**复用** `synra_plugin_ui::artifact::unpack` 的同一套规则（`..`/绝对路径/
> 符号链接拒绝、条目数与大小上限、压缩比），因为那些规则是关于**归档**的、不是关于 UI 的；
> `ArchiveLimits::package()` 就是本节 64 MiB / 32 MiB 那一组数字。发送侧 `package_directory` 把已安装目录压成包，
> 并**跳过已解包的 `ui/`**（包里带的是 `ui.zip`，`ui/` 是本机的产物）。
> 本轮只到"装得进去"：`install::from_package` 有测试，接收端的同步回调仍发/收单个 wasm，尚未接上。
>
> ★ **接上了（设备间同步）**：`plugins_sync_run` 现在把 `plugins/<id>/` 打成 `.synra` 再发；接收侧（窗口与 daemon）
> 在 `on_finished` 里先 `install::from_package`，成功后才 `record_received`，失败回 `SyncOutcome::refused`。
> 收件箱里的名字也从 `plugin-<id>.wasm` 改成 `plugin-<id>.synra`。

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
    "manifest": "<sha256, 64 hex digits>",
    "artifacts": {
      "plugin.wasm": "sha256-<base64, no padding>",
      "ui.zip": "sha256-<base64, no padding>"
    }
  },
  "signature": "<base64 ed25519 signature over canonical subject>"
}
```

> ★ **两种编码，而且是有意不同的 —— 2026-10-18 实测踩过一次。**
>
> **manifest 用 hex**（`synra_plugin_api::manifest_digest`，64 个十六进制字符），**artifact 用 SRI**
> （`sha256-` + 无 padding 的 base64），因为前者是"一份文本文件的摘要"、后者是"一个按内容寻址的 blob"。
> 本节的示例原先两边都写成 `sha256-...`，实现按 hex 签 manifest、按 SRI 签 artifact，于是**一个真签名的包会
> 被报成"签名覆盖了别的 manifest"** —— 而两边的摘要都不可能匹配，因为连编码都不同。
>
> 这条被两个测试钉住：`a_signed_package_with_a_genuine_signature_installs`（正控制，签对了必须装上）与
> `a_signed_package_whose_signature_covers_other_bytes_is_refused_at_the_signature`。
> 教训是**比较一方算出来的摘要时，要和签名方用同一个函数**，不要"顺手"用另一个 crate 的 SRI 助手。

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

> ★ **`synra-plugin pack` 产出的就是这种 tarball**：`package/` 前缀，里面是 `manifest.json`、manifest 点名的
> artifacts、以及 `signature.json`（有签名时）；mode 固定 `0644`、mtime 固定 0，所以同样的输入两次打包字节相同（注册表的
> 摘要不会因为构建时钟而变）。只打包 manifest 点名的文件，不把源码/测试带进分发。
> ★ **pack 先跑 `verify`**：与安装时同一套检查，任何一项失败就不出包 —— 一个已经存在的 tarball 比一份被跳过的报告更容易被
> 信任。
> ★ **有签名就核对签名**：存在 `signature.json` 时，pack 用安装侧同一个 `sign::verify` 核对 manifest 与每个 artifact 的摘要；
> 不匹配就不出包。这里传的是**空 trust store** —— 只问“这把钥匙签的是不是这些字节”，不问“用户信不信这个钥匙”，后者是
> 安装那台设备的决定。
> ★ **`--registry <url>` 只做形状校验**：`https`、只有 scheme+host（无路径）、`http` 仅允许 loopback（本地 Verdaccio）；
> 通过后打印 `npm publish --registry <url> <file>`，**不上传**。账号、scope、策略属于发布目标，不属于归档。
> 发布：`npm publish dist/@acme-hello-1.2.3.tgz`（或把同一个 tarball 上传到自建/兼容 registry）。**仍然缺**的是
> `publish` 子命令：账号、scope、registry 策略都属于发布目标，不属于归档。


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

> ★ **回退有了**：`plugins.registry` 可以是**一个 URL 或 URL 数组**（数组即按顺序尝试的镜像）。packument 与搜索结果按顺序
> 请求；tarball 先取服务 packument 的那个镜像，再按**同一路径**尝试其它已配置镜像（一个把 tarball 指到未配置 origin 的
> packument 仍然被拒）。只有网络错误、404/408/429/5xx 会换下一个；401/403 直接拒绝，不再问别人。
> **校验失败不回退**：下载后走的是**一次** `install_directory`，签名/摘要失败不会触发任何镜像重试 —— 回退只发生在字节
> 到达之前。设置页仍只编辑一个 URL；数组写在配置文件里。
> ★ **设置页能编辑镜像列表了**：每一行一个 URL，可增删；保存时 1 个写**字符串**（旧配置的形状，不因为打开页面就改形），
> 多个写数组，全部清空写 `unset`。列表顺序就是尝试顺序。


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

> ★ **解析与解包已经有了（`synra-registry`）**：`Packument`（`dist-tags` 的 `latest` 决定版本，**不**按 key 排序猜；
> `versions[v].dist.tarball`；内联 manifest 用契约自己的 `Manifest` 解析）与 `extract_tgz`（gzip + tar；只接受普通文件与
> 目录，拒绝 `..`/绝对路径/符号链接/硬链接与超限；失败按写入清单清理，不递归删除）。npm 的 `package/` 前缀由调用方传入
> `strip_prefix`，不带前缀的包会被拒，而不是解到错误的深度。
> **仍然缺**：registry 客户端本身 —— 地址配置、`plugins_catalog_search`、`InstallSourceDto::Registry`，以及把下载的
> tarball 接到 `from_directory`。
> ★ **第一版客户端接上了**：配置键 `plugins.registry`（base URL；缺失时 `Unsupported`，不猜默认）。`plugins_install`
> 的 `Registry` 分支取 packument → `dist-tags.latest` 或指定版本 → `dist.tarball`（**必须同 origin**，否则拒绝：一个让
> 设备去取别处的 packument 是在索取未被配置的权限）→ `synra-fetch::fetch_bytes` → `synra_registry::extract_tgz`（`package/`
> 前缀）→ 现有的 `from_directory` 管线。`http` 只允许 loopback（本地镜像/测试），公网必须 `https`。
> **仍然缺**：catalog 搜索端点、镜像/回退策略（§4.2）、设置页里的 registry 配置与安装 UI。
> ★ **配置与入口有了**：设置页新增 Registry 一节（写 `plugins.registry`；清空 = `unset`，即关闭 registry 安装）。插件页的
> 安装框上多了一个 **From a directory / From the registry** 的切换：模式是控件而不是对文本的猜测（`@acme/hello` 与
> `C:\plugins\hello` 都是“带斜杠的字符串”）。registry 模式把 `{source:'registry', pluginId}` 交给 `plugins_install`。
> **仍然缺**：catalog 搜索、镜像/回退、以及 registry 安装前的权限预览（内联 manifest 还没显示）。
> ★ **权限预览有了**：`plugins_preview`（`pluginId` + 可选 `version`）读 packument 的内联 manifest，返回
> `InstallPreviewDto{ pluginId, version, title, description?, publisher, publisherName?, capabilities, headless, hasUi,
> sizeBytes? }`。插件页的 registry 安装先调它，弹一个确认框（名称、版本、发布者、capabilities、runtime、发布者声称的体积），
> 确认后才下载。它是**问题**不是批准：预览是发布者的说法，签名在字节上校验，装完仍然默认关闭。
> **仍然缺**：catalog 搜索、镜像/回退。
> ★ **搜索接上了**：`plugins_catalog_search`（已声明、现注册）请求 `{base}/-/v1/search?text=keywords:synra-plugin%20<词>&size=N`，
> 把 npm 的 `objects[].package` 映射成已有的 `CatalogItem`（名称、版本、描述、publisher.username、是否已装）；名字不是
> 插件 id 的结果被丢掉 —— 一个按关键词返回的索引仍可能给出这台设备装不了的东西。`capability_count` 在搜索阶段是 0：
> 搜索响应没有 manifest，真正的权限列表在预览里显示。插件页的搜索结果带 **Preview**，走的还是同一个确认框。
> **仍然缺**：排序/分页、镜像与回退（§4.2）。
> ★ **分页与排序有了**：`CatalogQuery` 多 `offset`（翻页，映射到 npm 的 `from`）与 `sort`（`CatalogSort`：`Relevance`
> 是索引自己的顺序，`Name` 在本地按标题排序 —— npm 搜索没有 `sort` 参数，所以只提供搜索端点能兑现的两种）。
> 插件页用 20 条一页，最后一页满才显示 **Load more**；下一页的 offset 是已显示条数，重复的结果按 pluginId 去重。

### 4.4 静态 index（可选）

一个桶（S3 / R2 / 静态站点）不能回答 packument，但能托管文件。`synra-registry` 因此接受一种更小的文档，并解析成与
`Packument` 相同的结构：

```jsonc
{
  "name": "@acme/hello",
  "latest": "1.2.0",                 // 可选；缺省时取列表最后一项
  "publishers": {                    // 可选：目录声明的发布者公钥指纹
    "@acme": "AB12 CD34 EF56 7890"
  },
  "versions": [
    {
      "version": "1.2.0",
      "tarball": "https://static.example.com/@acme/hello-1.2.0.tgz",
      "integrity": "sha512-…",        // 可选，仍是声明
      "manifest": { /* 内联 manifest */ }
    }
  ]
}
```

★ 为什么要第二种形状：路径 A 假设 npm 兼容，而静态托管是作者最便宜的发布方式；解析成同一个 `Packument` 后，安装路径
（下载 tarball → 解包 → 校验 → 写盘）**不需要知道**源是数据库还是桶。
★ 为什么 `latest` 缺省取**最后一项**而不是比较版本字符串：这个 crate 没有 semver 实现，文本比较会把 `1.10.0` 排在
`1.9.0` 前面；写 `latest` 是诚实的说法，回退顺序则由作者决定。
★ `publishers` 是**目录的声明**，不是验证：预览对话框在下载 tarball 之前就能显示它，安装后的指纹提示可以拿它做第二个
对照；真正的判断仍然是对字节验签。两者不一致时不是一个自动结论，而是一条值得给人看的事实。
> ★ **不一致时说出来**：安装完成后，若目录声明的指纹与 `InstallReport.fingerprint` 不同，报告里多一条 warning，句子同时
> 写出两个值；界面把它跟在“装上但处于关闭状态”后面显示。它不是拒绝：两份文档都出自作者，任何一份都可能过期；但沉默会
> 浪费目录提供的唯一第二个意见。
★ 仍会拒绝：空列表、同一版本列两次、`latest` 指向不存在的版本（文档说一件事、列另一件事时不做猜测）。
★ **作者不用手写它**：`synra-plugin pack --index <base>` 在 tarball 旁边写 `<pluginId>.json`（如 `dist/@acme/hello.json`），
`tarball` 是 `{base}/{文件名}`，`manifest` 取自 manifest 文件，`publishers` 取自刚刚核对过的签名（无签名则为空）。手写的 index
是对同一份包的第二种描述，两份会漂移。
> ★ **发第二个版本是追加，不是覆盖**：目标 index 已存在时读取它，保留其他版本的条目，替换同版本条目、更新 `latest`、合并
> `publishers`。同一版本重打包替换原条目（解析器拒绝重复版本）；读不懂的现有文件拒绝覆盖 —— 一个工具不该用自己看不懂的
> 内容去换掉它没写的版本历史。
> ★ **tarball 可以相对**：`pack --index`（不带值）写相对文件名，客户端按 index 自身的 URL 解析；带值时写
> `{base}/{文件名}`。作者在打包时通常不知道桶的公开地址，而“上传这两个文件”应该是发布的最小动作。
> 客户端拒绝 `..`（不规范化）：一个爬出自己目录的 index 是在请求作者没打算发布的东西。
> ★ **发之前可以自查**：`synra-plugin publish --index <file>` 读 index → 定位每个 tarball → 用安装侧的 `extract_tgz` 解包 →
> 有 `signature.json` 就用安装侧同一个 `sign::verify` 核对 → 检查 manifest 点名的 artifacts 都在。它**不上传**：桶是否公开、
> CDN 是否配置属于托管方；它只回答“装得上的形状对不对”。绝对 URL 的 tarball 需要 `--base` 才能映射回本地文件。
> ★ **发之后可以回读**：`publish --verify --base <url>` 再 GET `{base}/{pluginId}.json` 与每个 tarball，用同一个
> `parse_static_index` 解析、按 SHA-256 比对字节；任何版本不同都点名。网络走宿主的 `NetworkGuard`（https；http 仅 loopback），
> 所以“宿主拒的地址”在这里也会被拒。
> ★ **失败时说“重传哪个”**：回读不一致的版本会在末尾列出 `re-upload <version> <local file> -> <url>`，作者不需要再从
> index 反推路径。
> ★ **上传是作者自己的命令**：`synra-plugin upload --index … --base … --command 'aws s3 cp {file} {url}'` 按 index 顺序
> 上传 index 与每个 tarball。模板按空白切分、占位符逐 token 替换、**不经过 shell**；`{file}`/`{url}` 各成为一个参数（路径
> 含空格也一样）。凭据与工具属于作者，这个命令只提供两个它算得准的值：哪个文件、哪个 URL。
**仍然缺**：客户端还没有把 `plugins.registry` 指向静态 index（目前只解析，不接网络）；`index.json` 的缓存与 ETag 也
留给接入时处理。
> ★ **接上了**：`plugins.registry` 里以 `static:` 开头的地址表示静态 index（如 `static:https://bucket.example/plugins`）。
> 客户端按 `{base}/{pluginId}.json` 取文档并 `parse_static_index`；tarball 用文档给的完整 URL（可以是另一个 CDN 主机，仍受
> https/loopback 策略约束）。**搜索**只在 npm 兼容的地址上做：静态 index 没有索引可搜，全部是静态时 `plugins_catalog_search`
> 返回 `Unsupported`，而不是把 404 说成“没有匹配”。
> ★ **条件请求**：文档的 `ETag`/`Last-Modified` 按 URL 缓存；下一次带 `If-None-Match`/`If-Modified-Since`，304 直接复用上次
> 解析结果（`synra-fetch` 也因此不再把 304 当重定向）。进程内缓存，靠验证器续期，不设 TTL。
> ★ **`Cache-Control: max-age`**：有寿命的条目过期后不再发送验证器，直接重新读取；304 携带新的 `max-age` 时续期。
> `no-store`/`no-cache` 被当作“没有寿命”，即每次都重新验证 —— 对这个客户端而言是保守的读法。


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
      + `publisher.rs` ed25519 验签；测试 `verify_tests.rs`。
      ★ **而"校验"曾经只做了一半，2026-10-18 补上另一半。** 宿主验的是**签名本身**，从不问它签的是不是
      **这个包** —— 于是「拿一个真签名的包、换掉 `plugin.wasm`、保留 `signature.json`」能通过全部 11 步。
      CLI 那边反而三样都比了（manifest 摘要、artifact 集合、`covers`），也就是同一条规则两份实现、
      其中一份漏了一格。现在比较在 `SigningSubject::mismatch`（`publisher.rs`）里，宿主与 CLI 都调它；
      `VerifyError::SignatureCoversOtherBytes` 让流水线在 `Stage::Signature` 停下，并点名是哪个文件不一致
      （"the signature covers X, the package has Y"），因为"签名无效"会把作者送去重新签名一个可能没问题的包。
      三条测试挡着这件事：签名真但字节不同、签名后改过 manifest、签名不覆盖新增的 artifact。
- [x] `PublisherTrustStore`（TOFU + 公钥指纹绑定 + 撤销）—— **A，但从不持久化**。
      `publisher.rs:619-710` 的 `PublisherTrustStore{trust/forget/revoke/classify}`，
      测试 `publisher_tests.rs:276-346`、`verify_tests.rs:184-232`。
      ★ **纯内存**：所有生产调用点都传 `PublisherTrustStore::new()`（`run.rs:207` 等），
      也就是**默认谁都不信任、撤销集永远为空**，每次进程启动都回到这个状态。
      ★ **持久化了（migration v12）**：`publishers` 表（publisher_id / fingerprint / pubkey / trust / first_seen_ms /
      decided_ms）。安装时从表构建 store（`trusted` 进 trusted map，`revoked` 进 revoked set）；安装成功后把签名发布者
      记成 `seen`，`INSERT ... DO NOTHING` 保证一次安装既不会把 key 升成 trusted、也不会把用户的决定重置。
      撤销的 key 现在真的会在 `publisher-trust` 阶段被拒（daemon 测试 `a_revoked_publisher_is_refused_even_with_a_valid_signature`）。
      **仍然缺**：没有命令/界面把 `seen` 变成 `trusted`/`revoked`，也没有指纹确认 UI —— 表只能被安装写入，用户还不能做决定。
      ★ **命令与列表有了**：`plugins_publishers`（Many，返回 `PublisherDto{ publisherId, fingerprint, trust, firstSeenMs,
      decidedMs }`）与 `plugins_publisher_decide`（`publisherId` + `trust` ∈ `seen|trusted|revoked`）。插件页新增
      **Publishers** 区：每个 key 显示 id、指纹、状态，带 “Trust key” / “Revoke key”；命令只改已经存在的行 —— 一个
      没被任何签名见过的 key 不能被“信任”，否则界面能信任一个从未出现过的公钥。
      **仍然缺**：安装签名包后没有即时的指纹确认步骤（用户要自己去列表里点）。
      ★ **即时确认有了**：`install::Installed` 带 `signed_by: Option<SignedPublisher>{ publisher_id, fingerprint }`，
      `InstallReport` 多 `publisherId` / `fingerprint`（都可以缺失 = 无签名）。插件页在安装成功后、列表重读之后，
      如果该 key 还没被 `trusted`，显示一个带指纹的提示（Trust this key / Not now）。不是模态：插件是关的、没有东西
      在跑，指纹可以按自己的节奏比对。**仍然缺**：没有“永远不要这个发布者”的一键拒绝（要去列表里 Revoke）。
      ★ **一键拒绝也有了**：提示里三个答案 —— Trust this key / Never trust this publisher / Not now；后两个都关闭提示，
      但只有前者写 `trusted`，中间那个写 `revoked`（之后这个 key 的包会在 `publisher-trust` 阶段被拒）。
      ★ **决定：只有暗色一套 token**（28 §6 R22）。设置页把 `ui.theme` 印出来并说明“只有一套调色板”，不再画一个永远只有
一个选项的控件；`ui.theme` 仍是内核记录，插件通过 `env-dark-mode` 读到它。
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
  ★ **第 147 轮没有改签名，而是找到了一条更好的路**：`handle_within` **本来就返回** `Served`，
  所以不需要回调 —— 需要的是让 `Served` **带上**那些完成的传输。
  现在 `Served.finished: Vec<FinishedTransfer>`（`path` / `byte_length` / `bundle`），
  而 `Decision::Finished` 那一支往它里面推一条 —— 三个事实一起走，其中 `bundle` 说出这是哪一类。
  ★ **证据来自已有的测试**：第 145 轮那条真实传输的测试里，监听端线程的 `join()` **就返回 `Served`**，
  所以它现在断言 `served.finished` 有一条、4096 字节、且 `bundle` 为 `None`（发的是附件）。
  一个值从「文件落地」走到了「调用方手里」，而这条路径是被一次真实传输证明的。
  ★ **仍然缺的最后一段**：`listener.rs` 里那个**连接线程**仍然把 `Served` 丢掉
  （每连接一个线程，它只打印 `ended`）。要真正写库，要么那个线程回调，要么把累加器交出去 ——
  而它需要 `Send + Sync + 'static`，这是下一轮的形状问题。
  ★ 于是那三件等待中的东西（`SyncRecord` / `sync_log` / 转换）**仍然**没有生产调用者，
  但缺的已经不是「值到不了调用方」，而是「连接线程没有把它交给调用方」。
  ★ **第 148 轮把最后一段接上了：连接线程现在把值交出去。**
  `listener::serve` 多了一个 `&Arc<dyn Fn(&FinishedTransfer) + Send + Sync>` 参数 ——
  它必须是 `Send + Sync + 'static`，因为持有它的是那个连接线程（借用到不了那里）。
  而 `FinishedTransfer` 多了 `peer`：**逐条带上**，因为「谁发的这个文件」是关于**文件**的事实，
  而 `Served::peer` 是关于**连接**的。
  ★ **而 daemon 那一头没有接通，原因是一个具体的作用域事实，不是懒惰**：
  在 `serve` 里唯一的 `Arc<Mutex<SynraDb>>` 叫 `shared_db`，而它在 **L637 已经被 `remembered` 取走**
  （一个活得更久的持有者），同时 `db` 在这个作用域是一个**值**（L381 就 move 进了 `shared_db`）。
  所以写那一行需要一个**并列的第二个 Arc**，或在它被消费前先 clone 一份 —— 那是一处签名上的一行改动，
  是下一轮的第一步。
  ★ 于是回调目前**说出**收到的是什么，而不是假装写进了库。
  **一个说「not recorded yet」的闭包比一个看似写库的闭包诚实。**
  ★ **第 149 轮把那一行改了，三件孤立的东西第一次真的合作。**
  在 `shared_db` 还手上的地方取**第二个句柄**（`history_db`），因为第一个句柄在 L637 已被
  `remembered` 取走。而写库的活儿抽成了函数 `record_received`，**理由是可测性**：
  回调活在监听器的连接循环里，那个循环**从不返回** —— 只在它内部发生的事**测不到**。
  搬到函数里之后，一个内存库加一个手搭的 `FinishedTransfer` 就能验证它，
  而这是「接线能编译」与「收到的包变成一行」之间的区别。
  ★ **两条测试**：`a_received_bundle_becomes_a_row_of_history`（一行、插件对、按插件也能查回来）
  与 `an_attachment_is_not_a_synchronisation`（一张截图不是一次同步 —— 一份填满它们的「历史」
  会用错误的行回答「同步过什么」）。
  ★ 于是 `SyncRecord`（第 133 轮）、`sync_log` 表（第 135 轮）、两者的转换（第 136 轮）
  **终于都有了生产调用者**，路径是：`Decision::Finished` → `Served::finished` → 连接线程的回调 →
  `record_received` → `SyncRecord::to_row` → `record_sync_log`。
  ★ 仍缺的：那次传输**之后**的事 —— 安装。而版本号诚实地是 `None`：它在传输的元数据里，
  ★ **第 151 轮把「安装」接上了。** `record_received` 现在也调 `insert_plugin` ——
  于是 `synra-store` 里写好了、测过了、而**没有任何生产代码调用过**的两个函数
  （`insert_plugin`、`plugin_is_installed`）第一次有了调用者。
  ★ **`trusted: false` 是这一行里唯一真正重要的判断。** `PluginRow::trusted` 的文档说它回答的是
  「用户是否确认过它能做什么」，而一个刚刚到达的文件**没有被任何人确认过**。
  `enabled: true` 回答的是另一个问题（「它是否被允许运行」）—— 而 store 把它们做成两列
  正是为了这种情况：*它可以运行，而还没有人说过它能做什么*。
  ★ **而「收到就装」是一个决定，不是唯一的行为**，理由写进了代码：一个到达的文件本可以
  留在 inbox 里等人批准，但这个内核**没有任何地方记录**「到了但没装」——
  `plugins` 是唯一说「一个插件存在」的表，而一个躺在目录里、没有行的包对每个读者都是不可见的。
  ★ 测试三条：历史、对端持有、以及「装上了但不可信」（`health` 是 `unknown`，因为它是**被观察**的，
  不是被假设的）。
  ★ 仍缺的：**归档**。doc 15 的包是 `plugin.wasm` + manifest 的归档，所以这一轮装的是
  「一个以插件命名的文件」；真正的解包、manifest 校验与签名验证要等包格式。
  ★ **第 152 轮做了发送侧的第一半**：`TransferSender::open_bundle(path, chunk_size, plugin_id, version)`，
  而 `kind` 与 `meta` 从 `request()` 里**硬编码的常量**变成了**字段**。
  ★ **为什么这是一个真缺口**：`request()` 把 `kind` 与 `meta` 写死为附件，
  于是一个插件包会**以一张截图的身份到达** —— 而接收端**正是**据此决定它落在哪里、以及它是什么
  （第 139 轮的 `bundle` 字段、第 151 轮的安装）。一条在接收端看起来完全正常的错。
  两条测试互为对照：`open_bundle` 说是插件包，`open` 仍然是附件 ——
  没有第二条，把默认值改错会让**每一次**普通发送都变成插件包。
  ★ **而改这个字段时，一条注释变成了假话**：`request()` 的旧文档写着
  *「Why the kind is always `Attachment`」* —— 而它现在不是了。那五行已删。
  **一个说「永远是」的注释，在它的前提消失以后是最危险的一种。**
  ★ 仍缺的：真正调用 `open_bundle` 的人。`send_file`（`session/send/awaiting.rs`）走的是 `open`，
  而 daemon 需要一条把包推给对端的路径 —— 那才会让第 134 轮的 `SyncTable` 第一次被调用。
  ★ **第 153 轮把发送那一半做出来了**：`synra_core::send_bundle`（`session/send/awaiting.rs`），
  与 `send_file` 同一层、同一条路 —— `TransferSender::open_bundle` 与 `open` 的区别只在请求里说的是什么。
  ★ **四个步骤因此抽成了 `run_transfer`**：它们是**这次传输**的，不是**载荷**的 ——
  一个包和一张截图走同一条路，而唯一不同的那件事已经在 sender 里了。抽取之后 `send_file` 的函数体是一行。
  ★ **`send_bundle` 比 `send_file` 少一个参数**（没有 `chunk_size`），而这是一个判断：
  包是 daemon 代表插件发的，两者对「一块该走多少字节」都没有看法，所以分块大小是**内核的**。
  （clippy 的 `too_many_arguments` 正是这样把它从 8 个压到 7 个的。）
  ★ **而它目前既没有调用者、也没有测试。** 调用它需要 `synra-control` 的 `Kernel` 加一个方法，
  而那个 trait 有**三个**实现（两个测试替身加 daemon 的 `Runtime`）—— 那是下一轮的事。
  ★ **第 154 轮把那条链接完了**：`Kernel::send_bundle`（traits）→ daemon 的 `Runtime` → `core::send_bundle`。
  而 daemon 那 142 行的传输函数**搬进了私有方法 `send_a_file`**（`impl Runtime`），
  两个 trait 入口各是一行 —— 校验、拨号、报告都是同一套，只有请求里说的是什么不同。
  ★ 搬移用了**括号配对**找闭合行，而不是「第一个四空格的 `}`」——
  第 144 轮就是后者切错了位置、文件变成未闭合。`send_file` 在 L194-335，一次就对了。
  ★ **`Kernel::send_bundle` 有默认实现，而 `send_file` 没有**，理由写进了 trait：
  *「这个内核不发插件包」对同 crate 里的测试替身是一个**正确**的回答* ——
  一个必需的方法会逼每个替身声称一个它没有的实现，而一个对自己没做的事给出貌似合理回答的替身，
  正是本仓库反复清除的那种谎。
  ★ **端到端测试**（`a_bundle_arrives_as_a_bundle`，跑了三次）+ 全量 1122 条：
  一个包从 `Runtime::send_bundle` 出发、经传输、在对端被认成**插件包**（第 139 轮的 `bundle` 字段）——
  而在这一轮之前，它会以一张截图的身份到达。
  ★ 仍缺的：发送端**记下**「对端现在持有我推的这个版本」。`SyncTable::apply_result`（第 134 轮）
  ★ **第 161 轮把两张表接上了**：`SyncTable::from_rows`（core）从 `sync_records` 的行装载。
  它们回答同一个问题（谁持有什么），一张在内存、一张在磁盘 —— 而在此之前**它们之间没有路**：
  一个从库里读出行的宿主没有地方放它读到的东西。
  ★ 三个判断：**单向**（库是记录，一张能写回的表会是它的第二个作者）；
  **空版本/空摘要变成 `None` 而不是 `Some("")`**（后者会被读成「对端持有版本 ''」这个断言）；
  **读不出来的行跳过而不是整体拒绝**（标识来自一台可能跑别的构建的设备，
  一张拒绝装载的表会把其余的历史一起带走）。
  ★ **而这一轮撞出一个真缺陷，第 162 轮把它修了**：`RemoteState::has_it` 原本是 `self.version.is_some()`，
  所以「摘要已知而版本未知」那一行 —— 正是**接收端**写下的形状（第 150 轮）—— 会让它说「**没有**」。
  而它其实有：摘要说得出它持有哪些字节。第一版断言写的是 `assert!(receiver.has_it())`，
  它失败了，而失败说明的正是这件事。当时只把它记下来，**而没有去断言一个可疑的答案**。
  ★ **第 162 轮改了它**：`version.is_some() || digest.is_some()`。改之前先看遍了调用者 ——
  只有两处，都在测试里，**生产代码零调用者** —— 所以这个修正是安全的。
  ★ 而后果值得写下来：旧答案会把「对端持有这些字节」读成「它没有这个插件」，
  于是下一次同步会把同一个包**每一轮都再推一遍**。一个错误的 `has_it` 不是少一个功能，是多一条无限循环。
  ★ 上一轮撤掉的那条断言现在**加回来了**，并且通过 —— 而它有明确的失败条件（改回 `version.is_some()` 即失败）。
  ★ **第 155 轮把发送端那一半也接上了**：`send_a_file` 在成功返回前写一行 `sync_records` ——
  与第 150 轮的接收端那一半对称，而两行回答的是同一个问题的两侧。
  ★ **只在一个 `Verified` 事件上记，而那是精确判据**：`events` 只在接收端真的说了「我校验过了」
  之后才带上它（`await_confirmation` 的返回值）。一次「发出去了但没听到确认」的传输
  不该声称对端持有什么 —— 那样的行会让**下一次同步以为两端一致**。
  ★ 端到端测试因此多了三条断言：版本是 `1.2.0`（发送端知道它推的是哪个版本，
  而接收端那一行是空串 —— 它只知道传输的元数据里有什么）、对端是监听端的标识、摘要是接收端确认过的那个。
  ★ **而 `SyncTable`（第 134 轮的内存类型）仍然没有调用者** —— 这是一件该说清的事，不是遗漏：
  `SyncTable` 与 `sync_records` 回答**同一个问题**（谁持有什么），一个是内存的，一个是持久化的。
  daemon 用的是后者（它本来就有一个数据库）。**两个类型回答一个问题，是重复** ——
  要不要让 `SyncTable` 变成 `sync_records` 的缓存，或者干脆删掉它，是一个该明确决定的设计问题。
  仍在等它 —— 而接收端那一半（`sync_records`）在第 150 轮已经通了。
  记在这里，而不是让下一轮以为它与 `send_file` 一样被走过了。
  ★ 于是链条现在只差两端之间那一段：`Kernel::send_bundle` → `Runtime` → `core::send_bundle`，
  而它一旦接上，第 134 轮的 `SyncTable` 就有了第一个真实的调用点。
  而监听器不保留元数据，所以不猜。
  ★ **第 150 轮接上了第二张表**：`sync_records`（第 134 轮建的）也第一次有了写者。
  它与历史回答的是**两个不同的问题**：「发生了什么」与「对端现在有什么」。
  后者由 `(plugin_id, peer_device)` 主键回答「两端是否一致」，而不是罗列发生过的事。
  ★ **而做的过程中发现一个真的丢失**：`Received::Verified { digest, .. }` 里有摘要 ——
  接收端**校验过**它 —— 而 `Decision::Finished` 的产生处用 `..` 把它丢掉了。
  摘要是唯一能回答「两端是否持有同样的字节」的东西，所以它现在一路带到写库处：
  `Received::Verified` → `Decision::Finished` → `Served::finished` → 回调 → `sync_records`。
  ★ **版本号诚实地是空串**：它随传输的元数据走，而一次 `Decision` 带的是**文件证明过**的东西，
  那个是摘要。一行写着「这个对端持有这些字节」的空版本行是**真的** —— 比从文件名编一个版本好。
  ★ 仍缺的：那次传输**之后**的事 —— 安装。而版本号诚实地是 `None`：它在传输的元数据里，  ★ 本轮的代价：22 次工具调用，其中大部分花在**猜函数边界**上 ——
  我三次把一个新变量插进了 `start_loops`，而那个闭包一直在 `serve` 里（`start_loops` 在 L130 前就结束了）。
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

## 第 163 轮：版本随包一起旅行

★ 那行 `record_sync` 的版本曾经是**空串**，理由写得很清楚：元数据止步于接收端，而一次 `Decision` 带的是
*文件证明过*的东西，那个是摘要。**那个理由在第 163 轮不再成立** —— 于是它成了假话，
而这是这份文档要盯着的事：一个说法在它的前提消失以后，读起来仍然像设计说明。

★ 现在的路是：`TransferMeta::PluginBundle { version }` → `version_of`（与 `bundle_of` 并列的一个纯函数）
→ `Receiving::version` → `Decision::Finished` → `FinishedTransfer::version` → `record_sync`。
于是那一行说「这个对端持有这些字节，而它们是 1.2.0」——下一次比较不必再查一次。

★ 而**它的第一次尝试是空的**：那个 `version_of` 被插进了 `Receiving` 的字段列表里
（`functions are not allowed in struct definitions`），因为单行锚点 `/// Which plugin this bundle belongs to…`
的**第一个**匹配就在结构体内部。把它移到结构体之前以后，`Receiving` 的文档又被顶开了 ——
**这是本会话第五次踩同一个坑**：把一个 item 插到另一个 item 之前，会把附着在后者上的东西抢走。

## 第 164 轮：一个完成的传输说得出它回应的是哪个请求

★ 上一轮让**版本**走到落库处；这一轮让**请求的标识**走到 listener。
路与 `digest`、`version` 完全一样（这是第四次走它）：`Envelope::request_id` → `Receiving::request_id`
→ `Decision::Finished` → listener 的解构。

★ **为什么是它**：`messaging::sync_result` 要的是**整个请求信封**（`&Envelope`），
所以一个回复必须先说得清它回应**哪一个**请求 —— 而 `Decision::Finished` 在此之前\只有路径、长度、
插件、摘要与版本，没有一样能指回那个请求。

★ listener 这一轮用 `..` 收下它而**还没有发回复**：下一个缺的是一件只有它知道的事 ——
`on_finished` 回调**不返回值**，所以接收端装了什么（daemon 那一侧的决定）到不了握手它的那一侧。
那要改回调的签名，是下一轮的形状问题。

★ 而这一轮**一次就做对了**，用的是上一轮学到的那两件事：锚点全是单行的，
新加的东西不插到别的 item 的前面。三个文件、857 条测试、一次编译通过。

## 第 165 轮：一个还没有答案的设计问题

★ 想做「接收端回一个结果」，于是去看 `messaging::sync_result`：它要**整个请求信封**，
所以之前的第 164 轮先把 `request_id` 送了过去。而再往下，`SyncOutcome` 有一个**必填**字段：

```rust
pub session: SyncSessionId,   // Which round.
```

★ **而传输从来没有传过它**：`TransferSender::request()` 里 `sync_session_id: None` 从第 145 轮起就是这样，
第 152 轮我读过那一行。也就是说**设计里「一轮同步」这个概念，在实现的传输里没有对应物**。

★ 所以「接收端回一个结果」卡住的地方不是接线，而是一个**决定**：一轮同步的标识由谁造、什么时候造、
以及它是否该随传输走。而 `send_bundle` 已经有 7 个参数，再加一个会越过 clippy 的阈值 ——
所以那个决定同时会影响签名怎么组织（把 `plugin_id`/`version`/`session` 合成一个 `BundleRef`？）。

★ **没有猜**：编一个 `SyncSessionId` 能让代码编译，而它会让「哪一轮」这个字段变成一个装饰。

## 第 166 轮：那个设计问题有答案了 —— 一轮同步有了名字

★ 上一轮停在一个决定上。这一轮做了它：**发起端造那个标识，而且它必须随传输走。**

理由有两条，而第二条是决定性的：一轮同步由一侧发起，而只有那一侧能命名它 ——
两台设备同时推送会各自造一个对方从未听过的标识，于是**谁的回复都配不上任何一轮**。
接收端的职责是把它**复述回来**，而这正是 `TransferRequest::sync_session_id` 携带它的原因。

★ 而实现时撞到一个类型上的事实：协议里的字段是 `Option<SessionId>`（protocol 的类型），
而 core 有自己的 `SyncSessionId`。**两个类型指的是同一件事，而只有一个装得进包。**
所以 `SyncRound.session` 是 `SessionId`，而两个类型之间的桥是「同一个标识、两处类型」，
不是编造一个值 —— 这与「编一个 `SyncSessionId` 让代码编译」是两件不同的事。

★ **而三个参数变成了一个值**：`plugin_id` + `version` + `session` 合成了 `SyncRound`。
这不仅是因为 `send_bundle` 到了 clippy 的参数上限 —— 它们在构造上就属于一起：
一轮同步是**关于一个插件的一个版本**的，而一个与它携带的包不一致的声明是没人能处理的报文。
**给那个东西起名字，比给 lint 加一个 allow 诚实。**

★ `TransferSender::request()` 不再写死 `sync_session_id: None`，所以一个包到了对端**说得出自己是哪一轮** ——
这是「接收端回一个结果」欠的最后一件事（第 133 轮的 `sync_result` 与第 134 轮的 `apply_result` 都在等它）。

## 第 167 轮：轮次走到了 listener

★ 上一轮让发起端命名一轮同步；这一轮让**那个名字到达 listener** —— 路与 `digest`、`version`、`request_id`
完全一样（这是第五次走它），而它读的是 `TransferRequest::sync_session_id`：
**接收端复述发起端选的值，而不是自己造一个**（一个自己造的轮次，是发起端从未听过的一轮）。

★ 于是 listener 手里现在齐了：`request_id`（回复给谁）、`sync_session_id`（哪一轮）、`plugin_id` 与 `version`
（哪个插件的哪个版本）、`digest`（哪些字节）、以及 `Served::peer`（哪个设备）。

★ **而「回一个结果」还差最后一件**：`on_finished` 回调**不返回值**，所以接收端装了什么
（那个决定发生在 daemon 那一侧，第 151 轮）到不了持有 link 的这一侧。
改它的签名是下一轮的事 —— 而那是唯一还缺的东西。

## 第 168 轮：一个回复不需要那个请求

★ 上一轮说「只差 `on_finished` 的返回值」。而改它要先看清楚一件事：**回复由谁造、用什么造。**
`messaging::sync_result` 要一个 `&Envelope`，而 listener 手里没有 —— 请求的字节被那次传输消费掉了，
活下来的是它已经留下的东西（`request_id`、对端、trace、ttl）。

★ 于是给那组事实起了个名字：`ReplyContext`，以及 `Envelope::reply_parts(context, event, at, payload)`。
`reply_to` 改为复用同一条路 —— **一条路，而不是一个规则加一个例外**。
`ReplyContext::of(request, local)` 的 `local` 是参数而不是从请求里读的字段，而这正是它是函数的原因：
**谁在回答从来不在问题里**。第一版漏了它，那会让一个回复以**提问方**的名义发出。

★ **而我差点改错一处正确的东西。** 看 `reply_to` 时我以为它是个 bug（`request_id: RequestId::new()`），
准备「修」它 —— 而紧接着的一行是 `reply_to: Some(request.request_id)`：每个信封有自己的标识，
而回复用 `reply_to` 指出它回应谁。**这是本会话第三次险些修掉一个正确的东西**，
而三次都是靠读完那一整块才停手的。

★ **这一轮的执行很糟：四次失败。** 一次把新结构插进了 `impl Envelope` 里（第六次踩同一个坑），
一次被那个重置工作树的进程清掉了写入，两次让 `messaging.rs` 的签名与函数体不一致 ——
最后靠**回退那一个文件**、只留下 protocol 的新增才收住。交付是好的，过程是浪费的。

## 第 169 轮：结果构造器收下了那个 context

★ `messaging::sync_result` 从 `(request: &Envelope, from, outcome, at)` 变成
`(context: ReplyContext, outcome, at)` —— 少了一个参数，而**少掉的那个正是调用方拿不到的那个**。

★ 上一轮的教训直接兑现了：**签名与函数体一起改**，两个测试调用点也在同一个脚本里改掉。
上一轮就是让它们不一致，才连错两次。这一轮一次编译通过、908 条测试、clippy 干净。

★ 于是「接收端回一个结果」只差最后一件：`on_finished` 回调**不返回值**，
所以接收端装了什么（那个决定在 daemon 那一侧）到不了持有 link 的这一侧。
那一步要同时改 daemon 的闭包与 listener 的调用点 —— 而它现在只需要一个 `SyncOutcome`。

## 第 170 轮：一个回复还缺什么，现在是清楚的

★ 想把「接收端回一个结果」做完，于是把回复需要的东西对着 `ReplyContext` 数了一遍。
listener 手里已经有的：`request_id`（回给谁）、`target`（对端 = `FinishedTransfer::peer`）、
`sync_session_id`、`plugin_id`、`version`、`digest`。**缺两样：trace，和「谁在回答」。**

★ trace 这一轮补上了（`Decision::Finished` 带上 `Inbound::trace`，第七次走那条「把事实送到 listener」的路）。
而 `from` 缺的是另一类东西：它是**本机标识**，不属于这次传输，而 `Host::identity` 是 `pub(super)` ——
listener 在别的模块，看不见。所以那要么是 `Host` 上的一个访问器，要么是 `FinishedTransfer` 再带一个字段 ——
**而那是下一轮的第一个决定**。

★ **而查 `ReceiverDuty` 时得到一个更重要的结论**：它的七个变体是**检查义务**
（`PackageDigest`、`Signature`、`Manifest`、`ArchiveLimits`、`SessionComplete`、`LocalActivation`、`LocalGrants`），
**里面没有「安装」**。而 daemon 只做了安装 —— 所以一个诚实的 `SyncOutcome` 里，
`performed` 是**空的**，`is_fully_checked()` 是 `false`。
**那不是缺陷，那是事实**：这个内核还没有运行过 ⑧ 的那七项检查，而一个声称做过的 outcome 会让 ⑨ 说谎。

## 第 171 轮：接收端真的回了一个结果

★ 第 133 轮的 `messaging::sync_result` 从那时起没有生产调用者，而这一轮它有了：
接收端在检查并安装之后，把结果发回给发起端。第 134 轮的 `SyncTable::apply_result` 等的也是它。

★ **而做的过程中发现第 148 轮的一个错误。** 那一轮我把回调放在 `handle` **返回之后**执行 ——
那时连接已经结束、`link` 已经 drop，所以**结果永远发不出去**。
回调现在移进了 `handle_within` 内部：**回复属于问它的那条连接**。

★ **`performed` 是空的，而那是事实。** 它列的是 ⑧ 的七项检查义务
（`PackageDigest`、`Signature`、`Manifest`、`ArchiveLimits`、`SessionComplete`、`LocalActivation`、`LocalGrants`），
而这个内核**一项都还没有跑** —— 它做的是安装。`activated: true` 是真的
（安装时启用了，第 151 轮），而 `is_fully_checked()` 会是 `false`。
**一个声称检查过的 outcome 会让 ⑨ 写下一件没发生的事**，而那是这份文档一直在挡的东西。

★ **`FinishedCallback` 是一个类型别名**，而不是三处各写一遍那个读不出来的签名 ——
clippy 的 `type_complexity` 是对的，而它的答案与第 166 轮的 `SyncRound` 是同一个：**给那个东西起名字**。

★ **这一轮执行得不好：六次失败、十七次调用。** 脚本在中途 assert 中止（于是后半段的改动从未写入）、
缩进不匹配让三处调用点漏掉、同一处被插了两次。**根因是我把一个改动拆成了太多步** ——
而第 169 轮已经证明：一次做完、一次编译，是最好的形状。

## 第 172 轮：发起端把那个结果读掉了

★ 上一轮接收端会回一个结果。这一轮**发起端把它读掉** —— 而理由是硬的：
`await_confirmation` 停在接收端的 `Complete` 上，而结果是在**那之后**才发的。
不读它，它就留在流里，成为下一个人读到的「关于一个已经结束的轮次的消息」。
**一个没人读的回复不是没被回答的问题，而是对下一个问题的错误回答。**

★ 它作为一个 `SessionEvent::SyncResult` 出现，而不是一个返回值：这条链路上的一切都是用事件列表描述的，
而结果正是在同一个链路、同一次传输里发生的事。

★ **而第一次实现让一次成功的传输失败了。** 接收端发完结果就结束它那一侧（它没有更多要收的），
于是发起端读到的是**关闭** —— 而 `SessionError::Link(Closed)` 被 `?` 传播出去，
`send_bundle` 返回 `TransportNotOpen`，第 154 轮那条端到端测试立刻红了。

★ 修法是**一行**：一个已经关闭的连接**没有结果可读**，而那不是这次传输的失败。
`Ok(None)`。**这一行是必要的，而不是防御性的** —— 它是「对端说完了」与「读到一个错误」之间的区别，
而只有一条真实的端到端测试能发现它。857 条测试里有那一条，而它正是在这里起作用。

★ **而这一轮的最后一段仍然缺着**：发起端把结果**读进来**了，但还没有**用它**
（第 134 轮的 `SyncTable::apply_result` 仍在等一个调用者）。那是下一步。

## 第 173 轮：那一行现在记的是对端**装上的**版本

★ 第 155 轮写 `sync_records` 那一行时，能填的只有「**我推过去的**版本」—— 因为那时没有回复可听。
而第 172 轮之后有了：结果里带着 `installed_version`。**那是两个不同的问题**，
而一个拒绝了包的设备，在旧写法下仍然会被记成「持有它」。

★ 三条规则，来自第 134 轮的 `SyncTable::apply_result`（同一件事的持久化副本）：
回复说 `ok: false` → **不写**（它什么也没装）；回复说 `ok` 却没给版本 → **不写**（回复自相矛盾）；
**没有回复** → 用推过去的版本（第 155 轮那个判断仍然是唯一可用的答案）。

★ **而它的证据有一个诚实的缺口。** 端到端测试仍然通过，因为接收端装上的版本与推过去的是同一个
（它从传输的元数据里读，第 163 轮）—— 所以「用装上的」这一段在**成功的传输里逻辑等价**。
它只在接收端**拒绝**时不同，而制造那件事需要一条会拒绝的接收端，这里的脚手架做不到。
`ok: false` 与「ok 却没有版本」两条分支因此没有测试，而它们是这一轮真正新加的东西。

★ 于是 ⑨ 现在是完整的：发起端发一个包、接收端回它做了什么、发起端把那个答案写进
「谁持有什么」。**而 `SyncTable` 那个内存类型仍然没有宿主** —— 这一轮用的是它的持久化对偶，
而两者的关系（缓存、还是两个宿主各自的实现）仍是一个没有回答的设计问题。

## 第 174 轮：两张脸，一个契约

★ 上一轮留下一个没有回答的问题：`SyncTable`（内存）与 `sync_records`（SQLite）是什么关系。
这一轮去查那个一直写在文档里的理由 —— **「一个内核没有 SQLite 也可以用它」—— 而它是错的**：
`synra-core` 从第 136 轮起就依赖 `synra-store`（`SyncRecord::to_row` 返回 store 的行类型）。
**没有那种宿主。** 那句理由是我在第 157 轮推断出来的，而它一直没被验证过。

★ 真正的理由是：一个是纯逻辑、一个是持久化，而**没有任何东西保证两边一致**。
而「一致」可以被测 —— 所以现在有一条测试（`the_two_faces_of_one_question_agree`）：
同一组事实写进库，再装进内存的那张，两边的答案必须一样。

★ 它同时钉住第 162 轮修过的语义：**摘要也算「持有」**。
库里那一行的版本是空的（接收端写下的形状），而两个实现都必须把它读成「有」，
否则下一次同步会把同一个包再推一遍。这条测试改回 `version.is_some()` 就会失败。

★ **而它不删 `SyncTable`。** 「两个实现」不是一个必须消除的重复 —— 单元格逻辑与持久化
本来就是两件事，而它们之间的契约是可以被钉住的。删掉它需要删掉它的测试，而那是我不做的事。

## 第 175 轮：`ReceiverCheck` 为什么接不上（而这一轮**没有**接上）

★ 上一轮推翻了一个推断出来的理由。这一轮查 `ReceiverCheck` —— 第 130 轮写的那个「接收端要按顺序履行的七项义务」，
它从那时起没有生产调用者。**而它接不上，原因是结构性的**：

`ReceiverCheck::verify_digest` 与 `check_session_complete` 都要一个 **`SyncSession`** —— 那套块记账是
第 130 轮**为同步**写的。而真实的传输路径用的是 **`TransferReceiver`**（流式写盘）。
**两套块记账，形状不同**：第 137 轮对 `SyncInbox` 的那个判断，对这里是同一个事实。

★ 所以七项里有**三项**这一侧真的能做 —— `check_room`（磁盘空间）、`decide_activation`、
`keep_local_grants` —— 而另外四项与 `SessionComplete` 要一个这个内核还没有的东西。
把它们中的任何一项记成「做过」，就是让 ⑨ 写下一件没发生的事。

★ **而这一轮试图把那三项接上，失败了四次编译。** 改动横跨一个函数的签名、一个闭包的捕获、
三处测试调用点与一处类型（`plan.data_dir` 是 `String` 而不是 `PathBuf`）——
而脚本在中途 assert 中止、`write_text` 从不执行，于是每一轮编译看到的都是**上一轮那棵半改的树**。
最后**回退**了那一个文件。

★ **记下的是判断，不是代码**：那三项**可以**接，而接之前该先把它们的调用点想清楚 ——
那是一个函数签名的事，不是一次插入的事。而 `ReceiverCheck` 的其余四项**在传输路径统一块记账之前接不上**，
那是一个真正的设计决定，而不是一个遗漏。

## 第 176 轮：那三项检查接上了 —— 而 `performed` **仍然是空的**

★ 第 175 轮想做而失败的那件事，这一轮做成了：daemon 在装之前真的跑
`check_room`（磁盘空间）、`decide_activation`、`keep_local_grants`，而 outcome 用 `ReceiverCheck::finish` 收口。

★ **而 `performed` 依然是空的，而我没有查出为什么。** `ReceiverCheck` 对一个**被拒绝**的步骤不留痕迹
（第 130 轮的设计），所以空着有两种可能：三项之中有的失败了（`check_room` 会问目录有多少空余，
而那个探针在第 130 轮有过一个 Windows 上的坑），或者 `finish` 没有把它们带过来。

★ **而我写的那条断言是错的，撤掉了。** 它断言 `performed` 非空 —— 而它失败了。
一个为了通过而写的断言比没有断言更糟，所以那里现在只留下调用，**把问题写在原地**。

★ **这一轮的机械失败值得单独记下，因为它是可修的**：我的脚本把 `write_text` 放在最后，
于是中途一次 `assert` 失败就**一个字都没落盘** —— 而下一轮编译看到的还是那棵半改的树，
我修的是**上一个错误的影子**。第 175 轮因此浪费了四次编译，而这一轮又浪费了几次。
**正确的形状是：先把所有替换的前提验证一遍，再写一次。**

★ 而两个真的发现，都是关于借用与所有权的：
① `plan.data_dir.clone()` 写在闭包里会**逃逸** —— 表达式里提到 `plan` 就会让 `'static` 闭包捕获那个借用，
即使 `clone()` 的结果是 owned 的；要把它挪到闭包**外**。
② 而更省的一条是**用它旁边已经有的那个 owned 值**（`data_dir`）—— 而它对 `listener::serve` 只能**借用**，
不能移走。

## 第 177 轮：`performed` 为什么是空的 —— 答案与撤掉

★ 第 176 轮留下一个问题，而这一轮查清了它。**那三项检查不是独立的，它们是七项序列里的第 4、6、7 项**：

| 检查 | 要求先完成 |
|---|---|
| `check_room` | `ArchiveLimits`、`Manifest` |
| `decide_activation` | `SessionComplete` |
| `keep_local_grants` | `LocalActivation` |

而 daemon 一项前置都没做，所以**三项全部被拒** —— 而「一个被拒绝的步骤不留痕迹」（第 130 轮的设计）
正是 `performed` 空着的原因。**那个 outcome 没有声称做过任何检查，而那是对的。**

★ 于是第 176 轮接上的那三项是**死代码**：它们永远返回 `OutOfOrder`，只让每个包多跑一次磁盘探针。
这一轮撤掉了它们。

★ 而**那个问题被钉成了一条测试**（`the_receiver_checks_cannot_be_run_out_of_order`）：
在序列之外调用这三项，得到的是 `OutOfOrder` 而不是「做过」；而三次拒绝之后 `performed` 仍然是空的。
把 `require` 改成总是 `Ok` 就会失败。

★ **而它再次印证了第 141 与第 175 轮的结论**：⑧ 的那七项接不上，不是因为没人写调用者，
而是因为**序列的第 1 项就要归档与 `SyncSession`，而传输路径没有它们**。
这是一个设计决定（统一块记账、定义包格式），不是接线工作。

## 第 178 轮：表终于有了写者，而计划问的是现在

★ `plugin_versions` 是这份文档里反复缺席的那张表：`active_plugin_version` 有读者、有测试，而**生产代码里没有写者**。
`install::from_directory` 写 `plugins` 行与文件，却从不写版本行；`versions.rs` 的模块文档甚至写着
"the installer wrote it"。于是 `plugins_sync_plan` 在真实安装之后对任何插件都答 `PluginNotInstalled` ——
一个测试全绿、生产恒为空的功能。

★ 这一轮把四段接上：

1. **写**：`record_plugin_version`（同一事务：先清旧的 active，再 upsert；旧行保留给回滚），
   以及 `install::from_directory` 里的调用。失败时连 `plugins` 行一起撤销，所以它收 `&mut SynraDb`。
2. **答**：`local_list` 把 `plugins` 与 `plugin_versions` 的 active 行 join 成 `PluginList`（没有 active 行的插件不进清单），
   `answer_plugin_list` 造答复；listener 收到请求就地回复，`Decision::Delivered` 因此带上信封。
3. **问**：`ask_plugin_list` 发请求并按 deadline 读答复，`MemoryStream` 两端有测试。
4. **计划**：`plugins_sync_plan` 拨号问每台已配对设备，`replace_sync_for_peer` 用答复替换该对端的全部行，再 `decide`。

★ **两个判断值得记下**：

- **一次活的答复是完整的**，所以它用替换而不是 upsert：卸载掉的插件不会出现在答复里，而 upsert 看不见「没有」，
  旧行会留到永远。
- **`online` 的意思变了**：从「它有地址」变成「它答复了」。没答复的对端仍计入「需要发」（安全方向），
  而界面终于能区分「能拨但没答」与「答了但没有」。

★ 仍缺的是 `plugins_sync_run`：`send_bundle` 在 core 里、调用者在 daemon，壳里还没有一个
「按计划发一个包并读回结果」的运行器。

## 第 179 轮：接收那一半只有一个实现

★ 第 178 轮最后说「壳的 listener 收到包不安装」。这一轮发现真正的问题不是「壳少写了一段」，而是**同一件事有两个实现**：
daemon 的 `record_received` 写三张表（历史、对端持有什么、`plugins` 行）并返回 `SyncOutcome`；
壳的闭包只写第二张并返回 `None`。于是壳收到一个包**不回 `SyncResult`**，发起端等到 deadline；也没有历史、没有安装行。

★ 所以 `record_received` 搬进了 `synra_core::plugin_sync`（收 `&SynraDb` 与 `&FinishedTransfer`，返回 `Option<SyncOutcome>`），
两个宿主各一行调用。壳的 listener 现在真的回结果了。

★ 搬的过程中有两个**已经存在但没被看见**的错误：

- **`activated` 与它旁边那行矛盾**：daemon 的 outcome 走 `ReceiverCheck::finish`，而 `decide_activation` 要求
  `SessionComplete` —— 传输路径永远不做，所以 `activated` 是 `false`；可它刚写的行是 `enabled = true`。
  现在 outcome 直接按那一行说 `true`，而 `performed` 仍然是空的（七项检查一项没跑，`is_fully_checked()` 是 false）。
  这三件事第一次彼此一致。
- **一条挂空的文档注释**：`serve_control` 的文档在函数消失之后留在文件里，紧贴着 `record_received` 的文档，
  于是 rustdoc 把它算进了后者的说明。删函数时一起删掉了。

★ 还有一处**版本**：第 150 轮说接收行「版本诚实地是空串」，因为元数据到不了接收端；第 163 轮已经让它到了
（`FinishedTransfer::version`）。历史行现在也记下版本，而不是继续留空。

★ 仍然没有的是**真正的解包**：收下的是单个 `plugin-<id>.wasm`，不是带 manifest 的包目录，而安装管线要的是目录。
两个宿主现在做的是同一件「写行」的事，而它们都还没有做「校验包并落到 `plugins/<id>/`」——那要等包格式。

## 第 180 轮：计划有人执行了

★ 第 178 轮说这条线「剩下的是让 `sync_records` 有新鲜数据」，第 179 轮让接收侧回了一个结果。这一轮把**发送端**补上：
`plugins_sync_run` 接一份 `SyncPlanDto`，对每台 `needs_artifact` 的设备拨号、`core::send_bundle`、读结果。

★ 前置是一件小事，但它决定了这个命令能不能开工：`plugin_versions.artifact` 存了文件名（相对 `plugins/<id>/`），
而唯一的读者 `active_plugin_version` 只返回 `(version, digest)`。所以加了 `SynraDb::active_plugin` →
`ActiveVersion { version, artifact, digest }`，并让旧的读者变成它的一层投影 —— **两个读者不会对「哪个版本生效」有两个答案**，
这是 `SyncTable` / `sync_records` 那次的同一个教训。

★ **`verdict(events, pushed)` 是这一轮真正新写下的规则**，三个答案而不是两个：

| 事件 | 结论 | 写不写 `sync_records` |
| --- | --- | --- |
| 没有 `Verified` | `Unconfirmed` | ❌ 一次没被确认的传输说不出对端有什么 |
| `SyncResult` 且 `ok: false` | `Refused` | ❌ 它什么都没装 |
| `SyncResult` 且 `ok`、有版本 | `Installed(版本)` | ✅ 记**接收端说的**版本 |
| `SyncResult` 且 `ok`、无版本 | `Refused` | ❌ 自相矛盾；补上我推的版本就是编造 |
| 有 `Verified`、无 `SyncResult` | `Installed(pushed)` | ✅ 旧构建不发结果时的唯一事实 |

★ 最后一行是刻意的：远端可能跑一个不送 `SyncResult` 的版本，而字节确实被确认了 —— 拿「我推的版本」记下来，
比「什么都没发生」好。它也是唯一允许用**本机**版本的地方。

★ 于是命令注册数从 33 变成 34，`plugins_sync_run` 从 `NOT_YET_REGISTERED` 移出。**仍然没有的**：接收端真正的解包
（见第 179 轮最后一段）——所以一次同步的终点是「对端说它装了」，不是「对端能跑」。
