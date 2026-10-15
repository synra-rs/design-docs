# 15 · 插件 SDK 与作者契约

> 本文定义插件作者看到的世界：manifest、构建工具链、Rust/TS SDK、以及"从零到发布"的完整路径。

---

## 1. 插件作者的三种选择

| 路径            | 语言           | 产物                    | 适用场景                          |
| --------------- | -------------- | ----------------------- | --------------------------------- |
| **Rust headless** | Rust           | `plugin.wasm`           | 逻辑、动作、协议解析（**推荐**）    |
| **Rust hybrid**  | Rust + HTML/JS | `plugin.wasm` + `ui.zip` | 需要自定义界面                     |
| **TS UI-only**   | TypeScript/Vue | `ui.zip`                | 只需要一个界面，逻辑很轻（如设置面板） |

**不推荐**：纯 JS 的 headless 插件。
理由：headless 的价值在于"跑在内核里、可被内核调度、有强隔离"。纯 JS 做不到（要么同 realm 无隔离，要么 WebView 里被挂起）。

> **对旧插件作者的迁移路径**：旧插件（Vue SPA + `defin` 逻辑混在一起）迁移到 Rust 版时：
> - UI 部分 → 保留为 `ui.zip`（Vue 组件基本能复用）；
> - 逻辑部分 → 重写为 Rust headless（这是必须的，因为逻辑要从 WebView 迁到内核）。

---

## 2. Manifest 规范

### 2.1 文件位置与形式

- 位置：包根的 `synra.json`（或 `package.json` 的 `synra` 字段，二者取其一，**推荐独立文件**）；
- 编码：UTF-8，无 BOM；
- 构建产物里**必须**内联一份规范化后的 `manifest.json`（供内核校验，不依赖包管理器元数据）。

### 2.2 Schema

```jsonc
{
  "$schema": "https://synra-app.dev/schema/manifest/v1.json",

  // ── 身份 ──
  "id": "@synra-plugin/chat",        // 必填。规则见 §2.3
  "version": "1.2.0",                // 必填。semver
  "title": "Chat",                   // 必填（UI 显示名；可本地化，见 §2.5）
  "description": "跨设备聊天",         // 可选
  "icon": "assets/icon.svg",         // 可选。相对包根；内核会校验存在性

  // ── 作者与许可 ──
  "publisher": {
    "id": "synra-official",          // 必填（用于信任与签名）
    "name": "Synra Official",        // 可选
    "url": "https://synra-app.dev"       // 可选
  },
  "license": "MIT",
  "repository": "https://github.com/synra/plugin-chat",

  // ── 兼容性 ──
  "sdk": {
    "min": "1.0.0",                  // 必填：宿主 SDK 最低版本
    "preferred": "1.2.0"             // 可选
  },
  "wit": "synra:plugin@1.0.0",       // 必填：使用的 WIT 世界版本

  "runtime": {
    "headless": {
      "artifact": "dist/plugin.wasm",  // 相对包根
      "stateful": false,               // 是否可在实例间保留内存状态
      "hints": ["cpu-heavy"]           // 可选软提示（宿主可忽略）
    },
    "ui": {
      "artifact": "dist/ui.zip",
      "entry": "index.html",           // zip 内的入口
      "defaultPage": "home"            // 可选：默认页面 key
    }
    // 至少填一个；两个都填 = hybrid
  },

  // ── ★ 权限（必填，即使为空数组）──
  "capabilities": [
    { "cap": "log:*" },
    { "cap": "device:list" },
    { "cap": "device:send:*" },
    { "cap": "device:broadcast" },
    { "cap": "storage:local:read" },
    { "cap": "storage:local:write" },
    { "cap": "event:emit:_plugin.chat.*" },
    { "cap": "event:subscribe:_plugin.*.*" },
    { "cap": "ui:page" }
  ],

  // ── 声明的动作（可被其它插件或宿主调用）──
  "actions": [
    {
      "name": "send-message",
      "args": { "type": "object", "properties": { "target": { "type": "string" }, "body": { "type": "string" } }, "required": ["target", "body"] },
      "returns": { "type": "object", "properties": { "messageId": { "type": "string" } } },
      "description": "发送一条消息"
    }
  ],

  // ── 声明的分享入口（参与意图匹配）──
  "shareHandlers": [
    { "kinds": ["text", "url"], "description": "识别消息文本" }
  ],

  // ── ★ 自己定义的事件（名字 + payload schema）──
  //    命名空间 `_plugin.<slug>.` 由内核强制；这里声明一次，
  //    `synra-plugin build` 会据此生成 Rust 常量，作者不写字符串（[13 §4.3 ⑦](./13-plugin-authoring.md)）
  "events": [
    {
      "name": "_plugin.chat.text",
      "payload": {
        "type": "object",
        "required": ["body"],
        "properties": {
          "body":     { "type": "string", "maxLength": 65536 },
          "sentAtMs": { "type": "integer" }
        },
        "additionalProperties": false
      }
    },
    { "name": "_plugin.chat.typing",
      "payload": { "type": "object", "properties": {}, "additionalProperties": false } }
  ],

  // ── ★ 要处理的入站事件（基础事件 + 自己的事件）──
  //    基础事件目录见 [04 §6.3](./04-wire-protocol.md)；没声明的不会被投递
  "handles": [
    { "event": "_plugin.chat.*" },
    { "event": "_synra.transfer.completed" },
    { "event": "_synra.device.changed" }
  ],

  // ── 依赖（compose）──
  "compose": [
    { "id": "@synra-plugin/format", "version": "^1.0.0", "optional": false }
  ],

  // ── 网络白名单（与 capabilities 里的 network:* 对应；便于 UI 展示）──
  "network": {
    "outbound": [
      { "host": "api.github.com" },
      { "host": "*.example.com", "methods": ["GET"] }
    ]
  },

  // ── UI 页面（UI/hybrid 插件）──
  "pages": [
    { "key": "home", "title": "聊天", "path": "/home", "icon": "chat" },
    { "key": "settings", "title": "设置", "path": "/settings" }
  ],

  // ── 配额建议（宿主可忽略或收紧）──
  "resources": {
    "preferredMemoryMb": 32,
    "preferredTimeoutMs": 2000
  },

  // ── 本地化（可选）──
  "locales": {
    "zh-CN": { "title": "聊天", "description": "跨设备聊天" },
    "en":    { "title": "Chat", "description": "Cross-device chat" }
  }
}
```


```
id ::= scoped | unscoped
scoped   ::= "@" scope "/" name
unscoped ::= name
scope    ::= [a-z0-9][a-z0-9-]{1,38}
name     ::= [a-z0-9][a-z0-9-]{0,63}

推荐：@synra-plugin/<name>（官方）
也允许：@<your-scope>/synra-plugin-<name>
```

**收紧的部分**：必须有 scope（避免命名空间冲突）、长度上限、保留字（`synra`、`core`、`system` 不可作 scope）。

**wire slug**（事件前缀用）：

```rust
/// `@synra-plugin/chat` → `chat`；`@imba97/synra-plugin-notes` → `notes`
pub fn to_wire_slug(id: &PluginId) -> PluginSlug;
```


### 2.4 Manifest 校验（装载期 + CLI verify）

| 检查                                     | 层         | 失败                       |
| ---------------------------------------- | ---------- | -------------------------- |
| JSON Schema 合法                          | 两者        | 拒绝                       |
| `id` 与包名一致                           | CLI        | 拒绝                       |
| `version` 是合法 semver                   | 两者        | 拒绝                       |
| `runtime` 至少一个                        | 两者        | 拒绝                       |
| 声明的 artifact 存在                       | 两者        | 拒绝                       |
| capability 语法合法                        | 两者        | 拒绝                       |
| headless 插件不得声明 `ui:*`               | 两者        | 拒绝                       |
| UI-only 插件不得声明 headless 专属能力       | 两者        | 拒绝                       |
| `network.outbound` 与 `network:*` 能力一致  | 两者        | 拒绝                       |
| `actions[].name` 无重复                    | 两者        | 拒绝                       |
| `events[].name` 在该插件内唯一              | 两者        | 拒绝                       |
| `events[].name` 前缀等于自己的 wire slug     | 两者        | 拒绝（命名空间越权）           |
| `events[].payload` 是顶层 object schema     | 两者        | 拒绝                       |
| `events[].payload` 里无 `$ref`/`format`/正则 | `verify`   | 拒绝                       |
| `events[].payload` 的 `maxLength`/`maximum` 有界 | `verify` | 拒绝                     |
| `handles[].event` 是"基础常量 / 自己声明过的名字 / 合法通配"三者之一 | `verify` | 拒绝（🔴 见 [13 §10-5](./13-plugin-authoring.md)） |
| 声明的入口与实现的方法**双向**对应            | 两者        | 拒绝（有 `handles` 必须有 `on_event`，反之亦然） |
| `pages[].key` 无重复                       | CLI+UI     | 拒绝                       |
| `compose` 无环                             | 装载期      | 拒绝                       |
| `sdk.min` ≤ 宿主版本                       | 装载期      | 拒绝（附升级提示）           |
| `wit` 与宿主兼容                           | 装载期      | 拒绝                       |
| **与已配对设备的 `events` 声明兼容**          | 装载期      | 拒绝激活，明确指出与哪台设备的哪个版本冲突（[04 §6.6](./04-wire-protocol.md)） |
| 图标文件存在且类型合法                       | CLI        | 警告                       |

### 2.5 本地化

`title` / `description` / `pages[].title` 支持本地化：

```jsonc
"title": "Chat",
"locales": { "zh-CN": { "title": "聊天" } }
```

内核解析时按 `env.locale` 选择，回退到默认值。**不做**复杂的 i18n 框架（插件自己处理内容级本地化）。

---

## 3. 构建工具链：`synra-plugin` CLI

### 3.1 命令集

```bash
# 脚手架
synra-plugin new my-plugin --template headless        # Rust headless
synra-plugin new my-plugin --template hybrid          # Rust + Vue UI
synra-plugin new my-plugin --template ui-only         # 纯 Vue UI

# 构建
synra-plugin build                # 按 manifest 的 runtime 声明构建对应产物
synra-plugin build --release
synra-plugin build --no-ui        # 只构建 headless

# 校验（提交前必跑）
synra-plugin verify
  ├─ manifest schema + 一致性
  ├─ capability 语法 + domain 适用性
  ├─ wasm imports ⊆ capabilities        ← ★ 关键检查
  ├─ 无禁用 import（sockets / fs preopens）
  ├─ ui.zip 结构 + CSP 兼容检查
  ├─ 体积上限
  └─ WIT 版本兼容
synra-plugin verify --strict        # CI 用：警告也当错误

# 签名
synra-plugin sign --key ~/.synra/keys/dev.key
synra-plugin keygen --out ~/.synra/keys/dev.key

# 发布
synra-plugin pack                  # 生成 .synra 包（tarball）
synra-plugin publish               # 上传到 registry

# 开发
synra-plugin dev                   # 监听变更 → 重建 → 热重载到运行中的宿主
synra-plugin dev --host 127.0.0.1:39001   # 指定宿主
synra-plugin test                  # 在沙箱里跑契约测试
```

### 3.2 `build` 做什么

```rust
// tools/synra-plugin-cli/src/cmd/build.rs

pub async fn build(opts: BuildOptions) -> Result<BuildReport> {
    let manifest = Manifest::load(&opts.cwd)?;

    let mut report = BuildReport::default();

    // ── headless：cargo build --target wasm32-wasip2 ──
    if let Some(headless) = &manifest.runtime.headless {
        let status = Command::new("cargo")
            .args(["build", "--target", "wasm32-wasip2", "--release"])
            .env("RUSTFLAGS", "-C link-arg=-zstack-size=1048576")
            .status()?;
        // 产物：target/wasm32-wasip2/release/<crate>.wasm
        // → 复制到 dist/plugin.wasm
        // → 可选：wasm-opt / strip
        // → ★ 生成 component（wasm-tools component new）
        report.artifacts.push(headless_artifact);
    }

    // ── UI：vite build → zip ──
    if let Some(ui) = &manifest.runtime.ui {
        // 1. 在 ui/ 目录跑 vite build（输出到 dist/ui/）
        // 2. ★ 校验产物不含外部网络引用（connect-src 'none'）
        // 3. 校验不含内联 eval（CSP 兼容）
        // 4. 打包为 dist/ui.zip
        report.artifacts.push(ui_artifact);
    }

    // ── manifest 内联（规范化后）──
    write_normalized_manifest(&opts.cwd, &manifest)?;

    // ── 自检 ──
    let verify = verify(&opts.cwd, VerifyMode::Normal).await?;
    report.verify = verify;

    Ok(report)
}
```

### 3.3 Rust 插件模板

```toml
# Cargo.toml
[package]
name = "synra-plugin-chat"
version = "1.2.0"
edition = "2024"

[lib]
crate-type = ["cdylib"]

[dependencies]
synra-plugin-sdk = "1.0"
serde = { version = "1", features = ["derive"] }
serde_json = "1"

[profile.release]
opt-level = "s"        # 体积优先
lto = true
codegen-units = 1
panic = "abort"        # ★ guest 侧可以 abort（trap 由宿主捕获）
strip = true
```

```rust
// src/lib.rs
use synra_plugin_sdk::{plugin, ShareItem, ShareDecision, Outcome, Error};

struct ChatPlugin {
    // 插件自己的状态（注意：池化要求无状态 → 状态放 storage）
}

#[plugin]
impl ChatPlugin {
    /// 生命周期：激活
    fn activate(config: ActivateConfig) -> Result<(), Error> {
        synra::log::info("chat plugin activated", &config.locale);
        Ok(())
    }

    /// 生命周期：停用
    fn deactivate() {
        synra::log::info("chat plugin deactivated", &());
    }

    /// ★ 匹配：判断能否处理一个分享项（必须无副作用）
    fn can_handle(item: ShareItem) -> ShareDecision {
        // 示例：识别 GitHub 仓库名或链接
        if let Some(caps) = GITHUB_RE.captures(&item.text) {
            let url = normalize_github_url(&caps);
            return ShareDecision::handled("open-url", json!({ "url": url }));
        }
        ShareDecision::unhandled()
    }

    /// 执行一个动作
    fn execute(action: &str, args: serde_json::Value) -> Outcome {
        match action {
            "open-url" => {
                let url = args["url"].as_str().ok_or_else(|| Error::invalid_argument("url"))?;
                // ★ 这一步在目标设备上执行：调用宿主的系统能力（过闸门）
                synra::platform::open_url(url)?;
                Ok(json!({ "opened": url }))
            }
            _ => Err(Error::unknown_action(action)),
        }
    }
}

static GITHUB_RE: Lazy<Regex> = Lazy::new(|| {
    Regex::new(r"(?i)(?:https?://)?(?:www\.)?github\.com/([\w.-]+)/([\w.-]+)|^([\w.-]+)/([\w.-]+)$").unwrap()
});
```

### 3.4 UI 插件模板（Vue）

```
my-plugin-ui/
├─ synra.json
├─ package.json
├─ vite.config.ts
├─ index.html
├─ src/
│  ├─ main.ts
│  ├─ App.vue
│  └─ pages/
│     ├─ Home.vue
│     └─ Settings.vue
└─ dist/ui.zip            ← 构建产物
```

```ts
// ui/src/main.ts
import { createApp } from 'vue'
import App from './App.vue'

// ★ 不 import 宿主的任何运行时模块。
//   `synra` 由宿主的 initialization script 注入到全局；类型来自 `@synra/client`（[23](./23-client-library.md)）。
//   在 ui/tsconfig.json 里加一行即可，无需 import：
//       "types": ["@synra/client"]
createApp(App).mount('#app')
```

```vue
<!-- ui/src/pages/Home.vue -->
<script setup lang="ts">
import { onMounted, onUnmounted, ref } from 'vue'
// 只 import 类型；`synra` 是全局的（见上）
import type { DeviceTarget, Unsubscribe } from '@synra/client'

const devices = ref<DeviceTarget[]>([])
const selected = ref('')
const input = ref('')
const error = ref<string | null>(null)
let off: Unsubscribe | undefined

onMounted(async () => {
  synra.ready()                                   // 告诉宿主页面已就绪
  devices.value = await synra.device.list()
  selected.value = devices.value[0]?.deviceId ?? ''
  off = synra.device.onChange(d => { devices.value = d })
})

onUnmounted(() => off?.())

async function send() {
  try {
    await synra.device.send({ to: selected.value, event: 'chat.text', payload: { body: input.value } })
    input.value = ''
  } catch (e) {
    // ★ 结构化错误：判别靠 e.kind / e.code，不解析 message（[11 D1](./11-design-principles.md)）
    error.value = synra.isCapabilityDenied(e) ? '缺少权限：device:send' : String(e)
  }
}
</script>
```


---

## 4. Rust SDK（`synra-plugin-sdk`）

### 4.1 结构

```text
crates/synra-plugin-sdk/
├─ src/
│  ├─ lib.rs
│  ├─ prelude.rs         # `use synra_plugin_sdk::prelude::*;`
│  ├─ macros.rs          # #[plugin] 属性宏
│  ├─ modules/           # 9 个 namespace 的薄封装
│  │  ├─ log.rs
│  │  ├─ event.rs
│  │  ├─ device.rs
│  │  ├─ storage.rs
│  │  ├─ network.rs
│  │  ├─ fs.rs
│  │  ├─ transfer.rs
│  │  ├─ platform.rs
│  │  ├─ action.rs
│  │  └─ env.rs
│  ├─ types.rs           # ShareItem / ShareDecision / Outcome / Error 等
│  └─ testing.rs         # feature = "testing"：内存 mock 后端
└─ bindings/             # wit-bindgen 生成（构建时）
```

### 4.2 `#[plugin]` 宏做什么

```rust
#[plugin]
impl ChatPlugin {
    fn can_handle(item: ShareItem) -> ShareDecision { ... }
    fn execute(action: &str, args: Value) -> Outcome { ... }
    fn activate(config: ActivateConfig) -> Result<(), Error> { ... }
    fn deactivate() { ... }
}
```

展开为：
1. `wit_bindgen::generate!` 的 `export!` 调用（把 `can_handle` / `execute` / `activate` / `deactivate` 映射到 WIT 导出）；
2. 生成 JSON 序列化的适配层（WIT 用 string，Rust 用类型）；
3. **编译期检查**：
   - 函数签名与 WIT 世界一致；
   - `can_handle` **不得**调用任何有副作用的 host function（用 trait bound 或 lint 检查）；
   - 返回值类型实现 `Serialize`。

### 4.3 9 个 namespace 的 Rust 形态

```rust
pub mod log {
    pub fn trace(msg: &str, ctx: impl Serialize) { /* host.log("trace", ...) */ }
    pub fn debug(msg: &str, ctx: impl Serialize) { /* ... */ }
    pub fn info(msg: &str, ctx: impl Serialize) { /* ... */ }
    pub fn warn(msg: &str, ctx: impl Serialize) { /* ... */ }
    pub fn error(msg: &str, ctx: impl Serialize) { /* ... */ }
}

pub mod event {
    pub fn emit(event: &str, payload: impl Serialize) -> Result<(), Error>;
    /// 注册一个处理器（内核推送时调用）
    pub fn subscribe(pattern: &str, handler: impl Fn(&str, &[u8]) + 'static);
}

pub mod device {
    pub fn list() -> Result<Vec<DeviceTarget>, Error>;
    pub fn send(target: &DeviceId, event: &str, payload: impl Serialize) -> Result<(), Error>;
    pub fn broadcast(event: &str, payload: impl Serialize) -> Result<(), Error>;
}

pub mod storage {
    pub mod local {
        pub fn get<T: DeserializeOwned>(key: &str) -> Result<Option<T>, Error>;
        pub fn set<T: Serialize>(key: &str, value: &T) -> Result<(), Error>;
        pub fn delete(key: &str) -> Result<(), Error>;
        pub fn keys(prefix: Option<&str>) -> Result<Vec<String>, Error>;
    }
    pub fn quota() -> Result<Quota, Error>;
}

pub mod network {
    pub fn fetch(req: FetchRequest) -> Result<FetchResponse, Error>;
}

pub mod fs {
    pub fn read(path: &str) -> Result<Vec<u8>, Error>;
    pub fn read_to_string(path: &str) -> Result<String, Error>;
    pub fn write(path: &str, data: &[u8]) -> Result<(), Error>;
    pub fn list(path: &str) -> Result<Vec<DirEntry>, Error>;
    pub fn stat(path: &str) -> Result<FileStat, Error>;
}

pub mod transfer {
    pub fn send(meta: TransferMeta, target: &DeviceId) -> Result<TransferId, Error>;
    pub fn status(id: &TransferId) -> Result<TransferStatus, Error>;
}

pub mod platform {
    pub fn open_url(url: &str) -> Result<(), Error>;
    pub fn open_path(path: &str) -> Result<(), Error>;
    pub fn simulate_keys(chord: &str) -> Result<(), Error>;
    pub fn notify(title: &str, body: &str) -> Result<(), Error>;
    pub fn read_clipboard() -> Result<String, Error>;
    pub fn write_clipboard(text: &str) -> Result<(), Error>;
    pub fn capabilities() -> PlatformCapabilities;
}

pub mod action {
    pub fn declare(name: &str, schema: ActionSchema) -> Result<(), Error>;
    pub fn invoke(plugin: &PluginId, name: &str, args: impl Serialize) -> Result<Value, Error>;
}

pub mod env {
    pub fn sdk_version() -> String;
    pub fn host_kind() -> PlatformKind;
    pub fn locale() -> String;
    pub fn dark_mode() -> bool;
    pub fn can(capability: &str) -> bool;
}
```


### 4.4 错误处理

```rust
#[derive(Debug, thiserror::Error)]
pub enum Error {
    #[error("capability denied: {capability}")]
    CapabilityDenied { capability: String, declared: Vec<String> },

    #[error("quota exceeded: {quota}")]
    QuotaExceeded { quota: String },

    #[error("rate limited: retry after {retry_after_ms}ms")]
    RateLimited { retry_after_ms: u64 },

    #[error("device not found: {device_id}")]
    DeviceNotFound { device_id: String },

    #[error("device offline: {device_id}")]
    DeviceOffline { device_id: String },

    #[error("invalid argument: {detail}")]
    InvalidArgument { detail: String },

    #[error("path not granted: {path}")]
    PathNotGranted { path: String },

    #[error("host not supported: {capability}")]
    Unsupported { capability: String },

    #[error("internal: {message}")]
    Internal { message: String },
}
```

**关键**：错误是**结构化的**，插件作者可以 `match` 出 `CapabilityDenied` 并降级，而不是解析字符串。

---

## 5. 测试与开发体验

### 5.1 本地测试（无宿主）

```rust
// SDK 的 testing feature 提供内存后端
#[cfg(test)]
mod tests {
    use synra_plugin_sdk::testing::*;

    #[test]
    fn recognizes_github_repo_shorthand() {
        let host = MockHost::builder()
            .grant("log:*")
            .device("laptop", DeviceTarget { id: "…".into(), name: "Laptop".into(), kind: DeviceKind::Desktop })
            .build();

        host.with(|| {
            let decision = ChatPlugin::can_handle(ShareItem::text("imba97/smserialport"));
            assert!(decision.handled);
            assert_eq!(decision.action.unwrap(), "open-url");
        });

        // 断言没有调用任何有副作用的 host function
        assert!(host.audit().is_empty());
    }

    #[test]
    fn open_url_requires_capability() {
        let host = MockHost::builder().build();   // 无 platform:open-url 授权
        host.with(|| {
            let err = ChatPlugin::execute("open-url", json!({ "url": "https://x.com" })).unwrap_err();
            assert!(matches!(err, Error::CapabilityDenied { .. }));
        });
    }
}
```

### 5.2 契约测试（`synra-plugin test`）

在**真实沙箱**里跑一遍：

```
1. 用 wasmtime 实例化插件的 wasm
2. 校验 imports ⊆ manifest capabilities
3. 调用 can_handle 一批 fixture 输入，检查无副作用（audit 为空）
4. 调用 execute，检查返回结构
5. 测 fuel 消耗（是否在预算内）
6. 测内存峰值
7. 输出报告
```

### 5.3 `synra-plugin dev` 热重载

```
1. 监听源文件（cargo watch 语义）
2. 变更 → 重建 → 写 dist/plugin.wasm
3. 通过本地 socket 通知运行中的宿主（`synra-plugin dev` 注册的端口）
4. 宿主：停用该插件 → 重新装载新 artifact → 激活
5. 插件 UI（若有）通过 WebView reload 生效
```


---

## 6. 发布

### 6.1 打包格式

```
<name>-<version>.synra         # 实际是 tar.gz
├─ manifest.json               # 规范化后的 manifest
├─ plugin.wasm                 # headless artifact（若有）
├─ ui.zip                      # UI artifact（若有）
├─ signature.json              # ed25519 签名
├─ LICENSE
├─ README.md
└─ CHANGELOG.md
```

### 6.2 Registry 发布（npm 兼容）

```jsonc
// package.json（用于发布到 npm 兼容 registry）
{
  "name": "@synra-plugin/chat",
  "version": "1.2.0",
  "files": ["manifest.json", "plugin.wasm", "ui.zip", "signature.json", "README.md", "LICENSE"],
  "synra": { /* manifest 内联，便于预检与搜索 */ }
}
```

`synra-plugin publish`：
1. 跑 `verify --strict`；
2. 打包；
3. 签名（若无密钥则提示 `keygen`）；
4. 上传到 registry（`PUT` 或 npm publish）；
5. 打印安装命令与校验摘要。

### 6.3 密钥管理

```bash
synra-plugin keygen --out ~/.synra/keys/dev.key
# 生成 ed25519 密钥对；私钥权限 0600，公钥写入 manifest 的 publisher.pubkey
# ★ 建议：CI 里用环境变量 SYNRA_PLUGIN_KEY 提供私钥，不落盘
```

**信任与验证链**见 [16 插件同步与分发](./16-plugin-distribution.md) §3。

---

## 7. 从旧插件迁移的检查清单

对一个旧 Vue 插件，迁移步骤：

- [ ] 写 `synra.json`：把 `package.json` 的 `synra` 字段 + `entries.ui` 转成新的 `runtime.ui`
- [ ] 提取逻辑：把 `src/lib/**` 与 `src/index.ts` 里与 `PluginBridge` 交互的部分，重写为 Rust headless（`can_handle` + `execute`）
- [ ] 保留 UI：`pages/**` 与 `components/**` 基本可复用；把"注入 bridge"换成全局 `synra`（[23](./23-client-library.md)）
- [ ] 替换 API：
  | 旧 | 新 |
  | -- | -- |
  | `bridge.usePairedDevices()` → `pairedDevices` ref | `await synra.device.list()` + `synra.event.subscribe('_synra.device.*', ...)` |
  | `bridge.send({ event, target, payload })` | `await synra.device.send(target, event, payload)` |
  | `bridge.broadcast({ event, payload })` | `await synra.device.broadcast(event, payload)` |
  | `bridge.fetch(input)` | `await synra.network.fetch(input)`（**受闸门**） |
  | `bridge.readFile({ path })` | `await synra.fs.read(path)`（**沙箱内**） |
  | `bridge.useClipboard()` | `await synra.clipboard.read()/write()` |
  | `bridge.usePreferences()` | `await synra.storage.get/set()` |
  | `useSynraPluginEnvelope()` | `synra.event` |
  | `defineComponent` + `SynraPlugin` 类 | WIT `lifecycle.activate/deactivate` |
- [ ] 移除 `provide` 包装与任何 bridge 对象（宿主只注入全局 `synra` 门面）
- [ ] 把 capability 声明从字符串数组改成 `{ cap: "..." }` 数组（支持可选性）
- [ ] 跑 `synra-plugin verify --strict`
- [ ] 在真实宿主里 `synra-plugin dev` 验证

---

## 8. 实现清单

> ★ 这一节在实现之前写就，长期整节未勾选。逐条核对源码后：**合同、生成器、类型库是真货**；
> 薄封装、属性宏、打包、发布、模板都没有。**"verify 通过"与"部署会被拒"之间目前有一道缝**（见 755）。

- [x] `wit/synra-plugin.wit` 完整定义（host + lifecycle + share + actions）—— **A**：
      `wit/synra-plugin.wit:88` host，以及 lifecycle、share、actions，`:352` `world plugin`。
- [x] `tools/wit-codegen`：生成 Rust 宿主绑定与 SDK 绑定 —— **A**：
      `tools/wit-codegen/`（`parse/{contract,gate,signature,table}.rs`、`emit.rs`）；
      `crates/synra-plugin-host/build.rs` 每次构建重新生成 `capability_table.rs` 与 `host_trait.rs`。
      ★ SDK 侧**另走一条路**：`synra-plugin-sdk/src/lib.rs:67` 用 `wit_bindgen::generate!`，
      所以"生成器负责 SDK 绑定"只说对了一半。
- [ ] `synra-plugin-sdk`：9 个 namespace 薄封装 + `Error` —— **未做**。
      `synra-plugin-sdk/src/lib.rs` **只有** `wit_bindgen` 生成的绑定 + `synra-plugin-api` 三个类型的 re-export；
      没有 9 个 namespace 薄封装，也没有 `Error` 类型。
      ★ 文档点名的 **`fs` namespace 在合同里根本不存在** —— 合同没有文件系统接口；
      真实分层是"裸绑定 + `synra-plugin-api` 边界类型"。
- [ ] `#[plugin]` 属性宏（含编译期检查）—— **未做**：全仓 `Cargo.toml` grep `proc-macro` **0 命中**，
      工作区没有任何 proc-macro crate。
      ★ 而且这里有一个**已知的硬冲突**，不是"还没写"那么简单：
      `synra-plugin-sdk/tests/guest/world_fixture.rs:17-20` 自己写明 `export!` 需要 `unsafe`，
      而工作区 `unsafe_code = "forbid"` **不能**用局部 `allow` 放开。
      要落地要么把宏放进一个独立 crate（像 `synra-testkit-alloc` 那样把例外集中到一处），
      要么改变对这个 fixture 的处理方式 —— 这是一次设计选择，先得做。
- [ ] `testing` feature：`MockHost` + audit 断言 —— **未做**：
      `synra-plugin-sdk/Cargo.toml` 连 `[features]` 段都没有；无 `MockHost`。
      `host_calls_tests.rs:49` 的 `Stub` 是 crate 内 `#[cfg(test)]`，**插件作者拿不到**。
- [ ] `tools/synra-plugin-cli`：new / build / verify / sign / pack / publish / dev / test ——
      **4/9**：`args.rs:35-44` 只有 `New`/`Build`/`Verify`/`Sign`，缺 `pack`/`publish`/`dev`/`test`。
- [ ] `verify` 的完整检查项（§2.4 + §3.1）—— **一半，而且有一处静默失效**。
      CLI 有：manifest 校验（`verify/check.rs:75`，`ValidationLevel::Schema`）+ 体积/文件数 + import 名单。
      ★ `verify/target.rs:38` 的 `wasm_imports` **恒为空向量**（诚实降级为 warn），
      所以 **import 检查在生产中从不运行**；
      ★ 更要紧的是：**CLI 里没有"import ⊆ 已声明的 capability"这一 §3.1 的核心检查** ——
      只有宿主侧 `plugin-host/src/verify/mod.rs:64` 的 `Stage::ImportsWithinCapabilities` 有。
      后果是作者本地 `verify` 得到的"通过"，并不包含部署时会被拒的那一条 ——
      一个本地全绿的插件可能在装载期被拒，而 CLI 没有告诉他为什么。
- [ ] Rust headless 模板 + Rust hybrid 模板 + TS UI-only 模板 —— **C/半个**：
      `template.rs:36-43` 的 `TemplateKind{Headless,Ui,Hybrid}` 三套确实存在并被测试（`validate_manifest`）。
      ★ **headless 模板不是组件**：`template.rs:386-403` 产出的是 `cdylib` +
      `#[no_mangle] extern "C" fn synra_plugin_activate()`，与 WIT 要求的
      `export!("synra:plugin/lifecycle@1.0.0")` **不匹配** —— 生成的插件装上去，
      `compile` 会因缺导出被拒。模板能过它自己的测试，因为它测的是 manifest。
      ★ UI 模板是单个裸 `index.html`，无 TS/Vue/`package.json`/`tsconfig.json`。
- [x] `@synra/client` 类型库（`.d.ts` only）—— **A**：
      `synra-js/packages/client/package.json:10-14` 的 `exports` **只有**
      `"types": "./dist/index.d.ts"`，无 `main`/`module`；`src/global.ts:27` 声明全局。
- [ ] 打包格式 `.synra` + registry 发布流程 —— **未做**：无 `.synra` 格式、无 `publish` 命令；
      `sign` 只写 `signature.json`（`run.rs:248`）。
- [ ] 迁移文档 + 一个真实的迁移示例（把旧 chat 插件迁过来）—— **未做，且前提不成立**：
      见 §7 的说明，仓库里没有旧插件可以迁。
