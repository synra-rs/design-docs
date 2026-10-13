# 13 · 插件写法

> **本文的定位**：把"插件到底怎么写"这件事**具体到可讨论的程度**——作者写什么文件、manifest 长什么样、构建产物是什么、运行期谁调谁。
> 它是 [11 插件系统总设计](./09-plugin-system.md) 的**具体化**，也是 [16 插件 SDK](./15-plugin-sdk.md) 的**前置**。
>
> 🔴 **本文标注了 8 个尚未拍板的设计缺口（§11）**——这些是我们要讨论的东西，不是已经定好的。
> 标记约定：**✅ 已定** / **🟡 倾向** / **🔴 未定（需讨论）**

---

## 1. 一张图看三种插件形态

```
┌──────────────────────────────────────────────────────────────────────────────────┐
│  ① headless 插件（推荐，也是唯一"必须有"的部分）                                    │
│                                                                                  │
│   作者写：Rust crate                                                              │
│   产物：  plugin.wasm（Wasm 组件）                                                 │
│   跑在：  内核的 wasmtime 实例里                                                   │
│   能做：  匹配分享 / 执行动作 / 跨设备 send / 存储 / 网络（经内核代理）              │
│   不能：  渲染界面、访问 DOM、模拟按键（除非显式授权且平台支持）                      │
│                                                                                  │
│   例：github-bridge、clipboard-sync、shortcut-runner、file-drop                    │
└──────────────────────────────────────────────────────────────────────────────────┘

┌──────────────────────────────────────────────────────────────────────────────────┐
│  ② UI 插件（需要自定义界面时才用；**必须**同时有 headless 部分）                     │
│                                                                                  │
│   作者写：headless crate + ui/ 前端目录（Vue/React/原生 JS）                        │
│   产物：  plugin.wasm  +  ui.zip                                                  │
│   跑在：  Wasm 实例 + 一个隔离的 WebView（独立 origin）                             │
│   能做：  渲染页面，通过注入的 window.synra 调内核；调自己 headless 部分的 action     │
│   不能：  直连网络（CSP connect-src 'none'）、读写文件、模拟按键、管理其它插件         │
│                                                                                  │
│   例：chat（消息列表）、ps-shortcuts（快捷键面板）、settings-panel                  │
└──────────────────────────────────────────────────────────────────────────────────┘

┌──────────────────────────────────────────────────────────────────────────────────┐
│  ③ 纯 UI 插件 —— ❌ **明确不支持**（业主决策，[22 §3.1](./22-risks.md) Q-01）        │
│                                                                                  │
│   理由：没有 headless 部分 = 逻辑跑在 WebView 里 = 隔离最弱、无法 headless 复用、     │
│        能力模型要分裂成两套。所以每个插件**至少要有**一个 headless 部分               │
│        （哪怕只实现一个 can_handle）。                                             │
└──────────────────────────────────────────────────────────────────────────────────┘
```

**由此推出一条硬约束**（✅ 已定）：

> **`platform:simulate-keys` 这类高危能力只对 headless 部分开放，UI 部分永远不行。**
> UI 插件的方法白名单见 [08 §7.2](./08-ipc-and-frontend.md)。

---

## 2. 作者眼里的一个插件是什么

### 2.1 headless-only 插件（最简单）

```
synra-plugin-github-bridge/
├─ Cargo.toml                 # crate-type = ["cdylib"], 目标 wasm32-wasip2
├─ synra.json                 # ★ manifest（作者手写，构建时会规范化内联）
├─ src/
│  ├─ lib.rs                  # ★ 唯一入口：#[plugin] impl
│  └─ lib/
│     └─ github.rs            # 纯函数 helpers（可单测）
├─ icons/plugin.svg
└─ dist/                      # 构建产物（不进 git）
   ├─ plugin.wasm             #   Wasm 组件
   ├─ manifest.json           #   规范化后的 manifest
   └─ signature.json          #   ed25519 签名
```

### 2.2 hybrid 插件（有界面）

```
synra-plugin-chat/
├─ Cargo.toml
├─ synra.json                 # runtime: { headless: {...}, ui: {...} }
├─ src/
│  ├─ lib.rs                  # headless：can_handle / 收发消息 / 存储
│  └─ protocol.rs             # 消息编解码（纯函数）
├─ ui/                        # ★ 前端，独立的一份（自带 Vue，不与宿主共享）
│  ├─ package.json            #   vue / vite
│  ├─ index.html
│  ├─ vite.config.ts          #   base: './'（要在 synra-plugin:// 下工作）
│  └─ src/
│     ├─ main.ts              #   createApp + 从注入的 window.synra 读 env
│     └─ pages/Home.vue
└─ dist/
   ├─ plugin.wasm
   ├─ ui.zip                  #   ui/dist 打包，入口 index.html
   ├─ manifest.json
   └─ signature.json
```

### 2.3 三个模板（`synra-plugin new`）

| 模板        | 生成什么                                    | 用在哪                    |
| ----------- | ------------------------------------------- | ------------------------- |
| `headless`  | 只有 ①                                       | 动作类、协议类、同步类插件  |
| `hybrid`    | ① + ②                                        | 需要自定义页面的插件       |
| ~~`ui-only`~~ | **不提供**（③ 已否决）                        | —                         |

---

## 3. Manifest：作者要写什么

### 3.1 完整示例（hybrid）

```jsonc
// synra.json
{
  "$schema": "https://synra-app.dev/schema/manifest/v1.json",

  "id": "@synra-plugin/chat",
  "version": "1.0.0",
  "title": "Chat",
  "description": "跨设备聊天",
  "icon": "icons/plugin.svg",

  "publisher": { "id": "synra-official" },     // 签名时公钥指纹绑到这个 id

  "sdk": { "min": "1.0.0" },
  "wit": "synra:plugin@1.0.0",

  // ── ★ 运行时声明：插件的形态（[09 §2](./09-plugin-system.md)）──
  //     headless 必填；ui 可选。**没有** kind / isolation / target 这类字段。
  "runtime": {
    "headless": {
      "artifact": "dist/plugin.wasm",
      "maxMemoryMb": 32,                        // 建议值，宿主可收紧
      "callTimeoutMs": 2000
    },
    "ui": {
      "artifact": "dist/ui.zip",
      "entry": "index.html",
      "defaultPage": "home"
    }
  },

  // ── ★ 权限（必填，即使空数组）──
  "capabilities": [
    "log:*",
    "device:list",
    "device:send:*",
    "device:broadcast",
    "storage:local:read", "storage:local:write",
    "event:emit:_plugin.chat.*",
    { "cap": "network:fetch:api.github.com", "optional": true }
  ],

  // ── ★ 声明它能处理哪些分享（内核用它做预筛，避免盲目调所有插件）──
  "shareHandlers": [
    { "kinds": ["text", "url"], "description": "识别消息文本" }
  ],

  // ── ★ 声明它提供哪些动作（可被宿主 UI 或其它插件调用）──
  "actions": [
    {
      "name": "send-message",
      "args": {
        "type": "object",
        "properties": { "target": { "type": "string" }, "body": { "type": "string" } },
        "required": ["target", "body"]
      },
      "returns": { "type": "object", "properties": { "messageId": { "type": "string" } } }
    }
  ],

  // ── ★ 自己定义的事件（名字 + payload 形状）──
  //    命名空间 `_plugin.<slug>.` 由内核强制；名字与 schema 在这里声明一次
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

  // ── ★ 声明它要处理哪些入站事件（基础事件 + 自己的事件）──
  //    完整的基础事件目录见 [04 §6.3](./04-wire-protocol.md)
  "handles": [
    { "event": "_plugin.chat.*" },                          // 自己的（单段通配）
    { "event": "_synra.transfer.completed" },                // ★ 基础事件：文件收完
    { "event": "_synra.device.changed" },                    // ★ 基础事件：设备列表变化
    { "event": "_synra.share.received" }                     // ★ 基础事件：没人处理的分享
  ],

  // ── 🔴 定时任务声明（见 §10-1）──
  "timers": [
    { "name": "flush-outbox", "intervalMs": 30000 }
  ],

  "compose": [ { "id": "@synra-plugin/format", "version": "^1.0.0" } ],
  "pages": [ { "key": "home", "title": "聊天", "path": "/home" } ]
}
```

### 3.2 每个字段的作用与「为什么需要它」

| 字段                        | 谁读它              | 作用                                                                 | 状态 |
| --------------------------- | ------------------- | -------------------------------------------------------------------- | ---- |
| `runtime.headless.artifact`  | 内核（装载）         | 显式指出 `.wasm` 路径。**只有这一个来源**，`verify` 与装载器读同一字段    | ✅   |
| `runtime.ui.artifact`        | app（承载）          | 显式指出 `ui.zip` 路径                                                 | ✅   |
| `runtime.*.maxMemoryMb` / `callTimeoutMs` | 内核（配额） | 插件的**建议值**；宿主按信任等级可收紧，不可放宽                          | ✅   |
| `capabilities[]`             | 内核（三层闸门）      | 插件要用的能力。可标 `optional`（缺了能降级运行）                        | ✅   |
| `shareHandlers[]`            | 内核（预筛）         | "这个插件可能处理哪些类型的分享" → 不调插件就能排除大半，降低匹配开销        | 🟡   |
| `actions[]`                  | 内核 + 其它插件       | 可被调用的动作及参数 schema → 跨插件组合与 UI 调用                       | ✅   |
| **`events[]`**               | 内核（契约 + 兼容检查）| **自己定义的事件名与 payload schema**。用于跨设备兼容性检查（[04 §6.6](./04-wire-protocol.md)）；`build` 会据此生成 Rust 常量（§4.3 ⑦） | ✅   |
| `handles[]`                  | 内核（事件路由）      | 要处理哪些入站事件（基础事件 + 自己的事件）。**没声明的不会被投递**          | ✅   |
| `timers[]`                   | 内核（调度器）        | 声明定时任务（§10-1 🔴）                        | 🔴   |
| `compose[]`                  | 内核（依赖图）        | 依赖其它插件及其版本约束 → 拓扑排序激活                                   | ✅   |
| `pages[]`                    | app（路由）          | UI 插件的页面清单                                                      | ✅   |

> **`events[]` 与 `handles[]` 的分工**：`events` 是**我说的**（我发出去的名字与形状），
> `handles` 是**我要听的**（基础事件的名字，或我自己/别人声明过的名字）。
> 两者都写时，说明这个插件既发也收同一个事件（例如聊天插件）。


**明确禁止出现在 manifest 里的字段**（这些是**部署决策**，不是插件属性）：

| ❌ 字段                        | 为什么禁止                                                    |
| ------------------------------ | ------------------------------------------------------------- |
| `kind: desktop \| mobile`       | 插件的**形态**由 `runtime` 表达；"跑在哪类设备"是宿主的事          |
| `isolation` / `runtime`（枚举）  | 隔离手段由宿主按形态决定，插件不能指定                             |
| `target: web \| electron \| …`  | 同上                                                          |
| `platform: win \| mac \| …`     | 平台可用性由 `platform.*` 能力在运行期查询，不是编译期声明          |

> **区分"插件属性"与"部署决策"**：
> - **插件属性**：它有 UI 吗？它要哪些能力？它处理什么输入？→ manifest 里声明。
> - **部署决策**：跑哪个线程？几个实例？多大内存？用什么隔离？→ 宿主决定（[09 §2.3](./09-plugin-system.md)）。

---

## 4. headless 部分：作者写的 Rust 代码

### 4.1 最小完整示例

```rust
// src/lib.rs
use synra_plugin_sdk::prelude::*;

#[plugin]
impl GithubBridge {
    // ─────────── 生命周期 ───────────

    /// 激活：注册动作、读配置、准备状态
    fn activate(config: ActivateConfig) -> Result<(), Error> {
        synra::log::info("github-bridge activated", &config.locale);
        // ✅ 允许：持久化状态
        synra::storage::local::set("enabled", &true)?;
        // ❌ 不允许：起线程 / 存全局可变状态（§10-3）
        Ok(())
    }

    /// 停用：释放资源（宿主会撤销订阅、销毁实例池）
    fn deactivate() {
        synra::log::info("github-bridge deactivated", &());
    }

    // ─────────── 分享匹配（★ 必须无副作用）───────────

    /// 判断能否处理一个分享项。
    /// ✅ 同步、纯计算、有 fuel 预算（默认 50ms）
    /// ❌ 不允许调用任何有副作用的 host function（SDK 在类型上挡住）
    fn can_handle(item: ShareItem) -> ShareDecision {
        if let Some(caps) = GITHUB_RE.captures(&item.text) {
            let url = normalize_github_url(&caps);
            if let Some(url) = url {
                return ShareDecision::handled("open-url", json!({ "url": url }));
            }
        }
        ShareDecision::unhandled()
    }

    // ─────────── 执行动作 ───────────

    /// 执行一个动作。这里可以调有副作用的 host function。
    fn execute(action: &str, args: Value) -> Result<Value, Error> {
        match action {
            // 由 can_handle 派发过来的（在**目标设备**上执行）
            "open-url" => {
                let url = args["url"].as_str().ok_or(Error::invalid_argument("url"))?;
                // ★ 这一步过 capability 闸门：platform:open-url
                synra::platform::open_url(url)?;
                Ok(json!({ "opened": url }))
            }

            // 由本插件的 UI 调过来的（§7.3 的 SelfInvokeAction）
            "send-message" => {
                let target = args["target"].as_str().ok_or(Error::invalid_argument("target"))?;
                let body   = args["body"].as_str().ok_or(Error::invalid_argument("body"))?;
                // ★ 过闸门：device:send:<target>
                synra::device::send(&target.into(), "chat.text", json!({
                    "channel": "default", "body": body
                }))?;
                // ✅ 持久化（跨实例共享的唯一方式）
                let id = new_message_id();
                synra::storage::local::set(&format!("out:{id}"), &json!({ "target": target, "body": body }))?;
                Ok(json!({ "messageId": id }))
            }

            other => Err(Error::unknown_action(other)),
        }
    }

    // ─────────── 入站事件（★ 详见 §4.3）───────────

    /// 内核收到匹配 `handles` 的事件时调用。
    /// 注意：**不是**"订阅回调"——它是一个显式的入口，实例可能不同。
    /// 返回 `Some` = 给发送方回执；`None` = 不回执（详见 §4.3 ③）。
    fn on_event(event: InboundEvent) -> Result<Option<Value>, Error> {
        match event.event.as_str() {
            "_plugin.chat.text" => {
                let body = event.payload["body"].as_str().ok_or(Error::invalid_argument("body"))?;
                synra::platform::notify("新消息", body)?;      // 过闸门：platform:notify
                Ok(Some(json!({ "received": true })))          // 回执
            }
            _ => Ok(None),
        }
    }

    // ─────────── 定时任务（🔴 见 §10-1）───────────

    fn on_timer(name: &str) -> Result<(), Error> {
        match name {
            "flush-outbox" => { /* 把 storage 里攒的 outbox 发出去 */ }
            _ => {}
        }
        Ok(())
    }
}

static GITHUB_RE: Lazy<Regex> = Lazy::new(|| {
    Regex::new(r"(?i)^(?:https?://)?(?:www\.)?github\.com/([\w.-]+)/([\w.-]+)$").unwrap()
});
```

> ⚠️ **注意 `Lazy<Regex>`**：这是**不可变的**一次性初始化，可以安全地在池化实例之间"各自持有一份"。
> 但 `static mut` / `OnceCell<Mutex<_>>` 这类**可变**全局**必须禁止**（§10-3）——
> 因为实例池里的不同实例有各自独立的线性内存，写进去的东西不共享。

### 4.2 内核会调用你实现的这些方法

`#[plugin] impl` 的完整方法集（**插件的全部"入口"就在这里**——作者写的逻辑都是这些方法被调用时执行的）：

| 方法                                             | 谁触发                     | 必需 | 能在里面做什么                                    | 约束 |
| ------------------------------------------------ | -------------------------- | ---- | ------------------------------------------------- | ---- |
| `activate(config)`                                | 内核（安装后/启用时/启动时）  | ✅    | 读配置、注册 action、写持久化状态                    | 不许起线程、不许存可变全局（§10-3） |
| `deactivate()`                                    | 内核（停用/卸载/关停）        | 🟡   | 释放逻辑资源（宿主会撤销订阅并销毁实例池）             | 必须快速返回 |
| `can_handle(item) -> ShareDecision`               | 内核（收到分享时）            | 🟡   | **只做判断**，返回"我能处理成哪个 action"              | **必须无副作用**；有 fuel 预算（默认 50 ms） |
| `execute(action, args) -> Value`                  | 内核（本机执行动作时）         | ✅    | 干实事：调 host function（过闸门）                    | 可中断；有超时与配额 |
| **`on_event(event) -> Option<Value>`**            | **内核（收到匹配的事件时）**   | 🟡   | **事件被调用后的逻辑**（见 §4.3）                     | 与 `execute` 同级：可调 host function、有超时 |
| `on_timer(name)`                                  | 内核（定时器到点）            | 🟡   | 周期性任务                                          | 🔴 未定（§10-1） |

**必需性**（🟡 = 声明了才需要）：manifest 里声明了 `shareHandlers` 就必须有 `can_handle`；
声明了 `handles` 就必须有 `on_event`；声明了 `actions` 就必须有 `execute`。
**声明的入口必须实现，实现的入口必须声明**——`synra-plugin verify` 会双向检查。

> ⚠️ **最容易误解的一点**：插件不是"配置驱动的脚本"。
> **作者写的是真实的 Rust 代码**（`activate` / `execute` / `on_event` 的**函数体**），
> 编译成 `.wasm` 后由内核在沙箱里调用。"声明"（manifest）只决定**内核在什么条件下调用哪个入口**，
> 不替代逻辑本身。

### 4.3 声明一个事件 + 事件被调用后的逻辑

这是作者最常问的问题。完整走一遍：**谁声明、谁发、内核怎么路由、作者的回调在哪执行、怎么回执。**

#### ① manifest 里声明"我处理这些事件"

```jsonc
// synra.json
{
  // 声明式路由：内核只把匹配的事件投给我（★ 见 §10-5 的设计讨论）
  "handles": [
    { "event": "_plugin.chat.text" },                    // 精确
    { "event": "_plugin.chat.*" },                       // 单段通配
    { "event": "file.transfer.complete",                 // 数据面事件 + 条件
      "match": { "meta.kind": "attachment" } }
  ],
  "capabilities": [
    { "cap": "event:emit:_plugin.chat.*" },              // 发事件的能力（按命名空间授权）
    { "cap": "device:send:*" },
    { "cap": "platform:notify" }
  ]
}
```

#### ② Rust 里实现 `on_event`

```rust
// src/lib.rs
use synra_plugin_sdk::prelude::*;

#[plugin]
impl ChatPlugin {
    /// ★ 内核收到匹配 `handles` 的事件时调用这个。
    ///
    /// `event` 里带着"谁发的、发给谁、要不要回执"，所以作者不需要自己解析信封。
    fn on_event(event: InboundEvent) -> Result<Option<Value>, Error> {
        // 按事件名分支。★ 这里是普通 Rust 代码：可以 if/loop/调 host function
        match event.event.as_str() {
            // 来自另一台设备的聊天消息
            "_plugin.chat.text" => {
                let body = event.payload["body"]
                    .as_str()
                    .ok_or(Error::invalid_argument("body"))?;

                synra::log::info("收到消息", &json!({ "from": event.from, "len": body.len() }));

                // ① 有副作用的事：过闸门（platform:notify）
                synra::platform::notify("新消息", body)?;

                // ② 想回执 → 返回 Some（内核按 `reply_to` 发回；见 ③）
                Ok(Some(json!({ "received": true })))
            }

            // 传过来的文件已收完
            "file.transfer.complete" => {
                let transfer_id = event.payload["transferId"].as_str().unwrap_or_default();
                // 读内核交付的落盘路径（transfer 能力的产物）
                let path = synra::transfer::received_path(transfer_id)?;
                synra::log::info("文件已落地", &json!({ "path": path }));
                Ok(None)                       // 不回执
            }

            _ => Ok(None),                     // 声明过的通配可能兜住别的名字
        }
    }
}
```

#### ③ 三种回执方式（作者按需选）

| 想做什么                       | 怎么做                                              | 结果 |
| ------------------------------ | --------------------------------------------------- | ---- |
| **同步回一个值给发送方**         | `Ok(Some(value))`                                    | 内核按信封的 `reply_to` 发一条结果事件（[04 §5.2](./04-wire-protocol.md)） |
| **不需要回执**                  | `Ok(None)`                                          | 内核只记审计，不发包 |
| **稍后异步回 / 主动发给别人**     | `Ok(None)` + 后续调 `synra::device::send(...)`        | 由插件自己发起新事件（`event:emit` 能力） |
| **处理失败**                    | `Err(Error::...)`                                    | 内核发结构化错误（如果 `reply_to` 存在），并记审计；**不会静默失败** |

#### ④ 全链路（从 UI 点击到回调执行）

```
[设备 A] 插件 UI 按钮
   │ synra.device.send({ to: 'dev_b', event: '_plugin.chat.text', payload: { body: 'hi' } })
   ▼
[设备 A] 内核  → 过闸门 event:emit:_plugin.chat.*  → 装信封（trace_id / request_id）
   │ TCP
   ▼
[设备 B] 内核
   ├─ 解帧 → 校验信封 → 事件是 `_plugin.chat.text`
   ├─ ★ 查路由表：谁声明了 handles 匹配这个事件？（manifest 装载时就建好的索引）
   │     → chat-plugin 命中
   ├─ 从实例池取一个实例 → 调 Wasm: on_event(InboundEvent { event, from, payload, reply_to })
   │     · 每次 host call 过闸门（platform:notify 已授权 → 通过）
   │     · 有 fuel / epoch / 超时（插件卡住只影响这一次调用）
   └─ 返回 Some(value) → 按 reply_to 发结果事件回设备 A
   ▼
[设备 A] 内核 → UI：结果事件（带同一个 trace_id）
```

#### ⑤ 为什么是"声明 + 回调"而不是"运行时订阅"

| 方案                          | 为什么不用 / 为什么用                                                       |
| ----------------------------- | ------------------------------------------------------------------------- |
| `event::subscribe(cb)` 注册回调 | ❌ **在池化模型下不成立**：回调需要一个**稳定的目标**，而池里有多个实例，各自独立内存。谁收到？ |
| **manifest `handles` + `on_event`** ✅ | ✅ 内核在装载时就建好路由索引；投递时任选一个池实例即可；可审计、可限流、可离线分析 |
| 插件自己轮询拉取                | 实时性差（要靠 timer 唤醒）。可作为逃生舱，不作默认                            |

#### ⑥ synra 自带哪些事件（基础事件）

**内核提供"能力面"，插件提供"语义"。** 所以基础事件说的是"发生了什么"（设备上线、文件收完、收到分享），
而不预测"这意味着什么"（一条聊天消息、一张发票）。

完整目录（含 payload 形状）见 [04 §6.3](./04-wire-protocol.md)。作者最常用的十条：

| 事件                              | 什么时候触发                                | 典型用途 |
| --------------------------------- | ----------------------------------------- | -------- |
| ★ `_synra.device.changed`          | 已配对设备的在线/信息变化（**全量快照**）       | 刷新设备列表；无状态处理 |
| ★ `_synra.transfer.completed`      | 传输完成且**摘要已校验**                     | 拿落盘路径做后续处理 |
| ★ `_synra.share.received`          | 收到分享，但**没有任何插件声明处理它**          | "通用接收"型插件的兜底入口 |
| `_synra.transfer.failed`           | 传输失败/中止                              | 提示用户 + 清理 |
| `_synra.intent.completed`          | 本机发出的意图在远端执行成功                  | 关掉 loading、显示结果 |
| `_synra.intent.failed`             | 本机发出的意图失败（带结构化错误）             | 显示可操作错误 |
| `_synra.pairing.changed`           | 被对方解除配对 / 新配对成功                  | 清理本地缓存、提示用户 |
| `_synra.clipboard.received`        | 对端推来剪切板内容（**默认需用户开启**）        | 聊天类插件的快速输入 |
| `_synra.plugin.capability-changed` | 用户改了本插件的授权                        | 重新渲染权限相关的 UI |
| `_synra.host.background`           | 宿主即将进后台（移动端）                     | 尽快持久化状态 |

用法就是写进 `handles`，然后在 `on_event` 里 match：

```jsonc
// synra.json
{ "handles": [ { "event": "_synra.transfer.completed" },
               { "event": "_synra.device.changed" },
               { "event": "_plugin.chat.text" } ] }
```

```rust
fn on_event(e: InboundEvent) -> Result<Option<Value>, Error> {
    match e.builtin()? {
        // ★ 基础事件是**强类型**的：内核知道它们的形状，直接给你结构体
        BuiltinEvent::TransferCompleted(t) => {
            let path = t.local_path.ok_or(Error::unsupported("无落盘路径"))?;
            process_file(&path, &t.meta)?;
            Ok(None)
        }
        BuiltinEvent::DeviceChanged { devices } => {
            // 全量快照，所以插件不需要自己维护差量（实例池里也没地方存）
            synra::log::info("设备列表更新", &json!({ "count": devices.len() }));
            Ok(None)
        }
        // 插件自定义事件是裸 JSON —— 内核不认识它的形状
        BuiltinEvent::None => match e.event.as_str() {
            "_plugin.chat.text" => { /* serde 反序列化 e.payload */ }
            _ => Ok(None),
        },
        _ => Ok(None),
    }
}
```

**三条关键区别**（作者必须知道）：

|                    | 基础事件（`_synra.*` / `file.transfer.*`）        | 自定义事件（`_plugin.<你的 slug>.*`）            |
| ------------------ | ---------------------------------------------- | --------------------------------------------- |
| 谁定义 payload      | **内核**（Rust 类型，强类型给你）                  | **你**（manifest 里的 JSON Schema）              |
| 能不能自己发         | 只有白名单两条（[04 §6.5](./04-wire-protocol.md)） | ✅ 只能发自己命名空间下的                          |
| 跨设备兼容谁负责      | 内核（`#[non_exhaustive]` + Feature）             | **你的 schema**：安装时对比双方声明，不兼容就拒绝激活（[04 §6.6](./04-wire-protocol.md)） |

> **所以"作者也可以写自己的"是**：在 manifest 的 `events` 里声明名字 + payload schema，
> 在 `handles` 里声明要收哪些，在 `on_event` 里 match。命名空间 `_plugin.<slug>.` 由内核强制，
> 你无法（也不需要）伪造别人的事件。

#### ⑦ 这条路径上仍然 🔴 未定的三件事

见 §10-5：**`on_event` 的返回类型**（回执 vs void）、
**高频事件是否需要批量投递**（例如 `_synra.transfer.progress` 与每块 `file.transfer.chunk`）、
**`handles` 的匹配语法与 `capabilities` 的 `event:emit:*` 是否复用同一套解析**。

> **一个必须先承认的耦合**：事件名（无论基础的还是自定义的）是**跨插件/跨设备的字符串契约**。
> 这与 [11 D1](./11-design-principles.md)（不靠魔法字符串接线）有张力。
> 缓解方向：
> ① 基础事件名全部是 SDK 常量（`synra::builtin::TRANSFER_COMPLETED`），**作者不写字符串**；
> ② 自定义事件名在 manifest 里声明一次，Rust 侧用 SDK 生成的 `const`（`synra::events::CHAT_TEXT`，由 `synra-plugin build` 从 manifest 生成）；
> ③ `verify` 检查 `handles` 里的名字是"已知基础常量 / 自己声明过的名字 / 合法通配"三者之一（🔴 待定）。

#### ⑧ 谁写 Rust、谁不写（澄清一个常见误解）

| 插件的部分            | 作者写什么语言                                              | 作者的机器上需要什么                                        |
| --------------------- | --------------------------------------------------------- | --------------------------------------------------------- |
| **headless（逻辑）**   | **Rust**（`on_event` / `execute` / `can_handle` 的函数体）    | Rust 工具链 + `wasm32-wasip2` target + `synra-plugin` CLI   |
| **UI（界面）**         | TypeScript / Vue 等                                        | Node + pnpm；**不写也不编译 Rust**                            |

也就是说："作者不接触 Rust"**只对 UI 部分成立**——UI 部分消费的是 `@synra/client` 这个
**纯类型包**（[23](./23-client-library.md) / [25](./25-js-monorepo.md)），它只是把宿主注入的
`window.synra` 描述成类型，与 Rust 无关。


而**插件的逻辑部分就是 Rust 代码**：作者在本机写好、编译成 `.wasm`、签名、发布
（§4.5 构建链）。**用户机器上永远不编译任何东西**（[ADR-007](./20-adr.md)）——
这才是我原意要说的事，与"作者不写 Rust"是两件事。

### 4.4 SDK 的 9 组能力面（作者能调什么）

```rust
synra::env       // 只读：sdk_version / host_kind / locale / dark_mode / can(cap)
synra::log       // trace/debug/info/warn/error
synra::event     // emit
synra::device    // list / send / broadcast
synra::storage   // local::{get,set,delete,keys} + quota
synra::network   // fetch（宿主代理，过闸门 + SSRF 防护）
synra::transfer  // send / status（大对象传输）
synra::platform  // open_url / open_path / reveal / simulate_keys / notify / clipboard
synra::action    // declare / invoke / list（跨插件）
```

（详细签名与闸门规则见 [12 能力模型](./10-capability-and-sandbox.md) 与 [16 SDK](./15-plugin-sdk.md)。）

### 4.5 构建链

```
src/*.rs
   │  cargo build --target wasm32-wasip2 --release
   ▼
target/wasm32-wasip2/release/<crate>.wasm        ← 核心模块（或已是组件，视 Rust 版本）
   │  wasm-tools component new（若产出的是核心模块）  🔴 见 §10-8
   ▼
dist/plugin.wasm                                 ← Wasm 组件，实现 wit/synra-plugin.wit 的 world
   │  synra-plugin verify
   │    ├─ manifest schema
   │    ├─ capability 语法 + domain 适用性
   │    └─ ★ wasm 的 import 集合 ⊆ manifest.capabilities（[10 §2.4](./10-capability-and-sandbox.md)）
   ▼
dist/plugin.wasm（已校验）
   │  synra-plugin sign
   ▼
dist/signature.json
```

**关键约束**（`verify` 会拦）：

| # | 约束                                                          | 为什么                                     |
| - | ------------------------------------------------------------- | ------------------------------------------ |
| 1 | `manifest.capabilities` ⊇ wasm 的 host import 集合              | 否则插件能调用未声明的能力（这是编译期就能拦住的越权） |
| 2 | headless 不许声明 `ui:*`，UI 侧不许声明 headless 专属能力         | 能力面按形态区分                            |
| 3 | 不许 import WASI 的 `sockets` / `filesystem` preopens           | 网络与文件必须经内核（可审计、可限流）        |
| 4 | 产物 ≤ 32 MiB；`ui.zip` 解压后 ≤ 256 MiB、条目 ≤ 4096           | 解包炸弹 / DoS                              |

---

## 5. UI 部分：作者写的前端代码

### 5.1 它跑在哪

```
synra-app 的主 WebView（宿主 UI）
  ┌─────────────────────────────────────────────────────────────────┐
  │  src/  （宿主自己的 Vue 应用）                                    │
  │    pages/{home,plugins,devices,settings}                          │
  │                                                                  │
  │    ┌───────────────────────────────────────────────────────────┐ │
  │    │ 插件页面（独立 WebView 或 sandbox iframe，独立 origin）      │ │
  │    │  origin: synra-plugin://<plugin-id>/                       │ │
  │    │  CSP:    default-src 'none'; script-src 'self';            │ │
  │    │          connect-src 'none'; object-src 'none'             │ │
  │    │                                                            │ │
  │    │  ┌──────────────────────────────────────────────────────┐ │ │
  │    │  │ 插件的 ui.zip 内容（自带一份 Vue，不与宿主共享）         │ │ │
  │    │  │   index.html + assets/*.js + assets/*.css             │ │ │
  │    │  │   注入的 window.synra（冻结的 API 对象，见 §5.2）       │ │ │
  │    │  └──────────────────────────────────────────────────────┘ │ │
  │    └───────────────────────────────────────────────────────────┘ │
  └─────────────────────────────────────────────────────────────────┘
```

> **为什么不让插件共享宿主的 Vue**：
> 如果让宿主把 `vue` 通过 importmap / 全局变量"借"给插件，会引入一整套复杂度
> （vendor chunk 的稳定性、裸 specifier 的解析、打包器对导出的改名、各 WebView 引擎的差异），
> 而且这些复杂度**只在某些平台上暴露**，很难测全。
>
> 本设计让插件**自带运行时**：多几百 KB 下载，换来**独立 origin 的彻底隔离**与"任何平台行为一致"。
> 对绝大多数插件这是一笔划算的交易（UI 插件的下载量远小于它的逻辑插件部分）。

### 5.2 注入的 API（`window.synra`）

> ★ **作者不该直接裸用这个对象。** 请用类型库 **`@synra/client`**（[23](./23-client-library.md)）：它是一个 **`.d.ts` only 包**（零运行时），
> 在 `tsconfig.json` 里写 `"types": ["@synra/client"]` 之后，**不用 import 任何东西**就能写 `synra.xxx`。
>
> 下面列的是**宿主注入的运行时形状**——类型库描述的正是它，而宿主必须按类型实现。

```ts
// ui/src/main.ts
import { createApp } from 'vue'
import App from './App.vue'

// ★ 不 import 宿主的任何运行时模块。
//   `synra` 由宿主的 initialization script 注入为全局对象（token 在闭包里，不在对象上）。
//   类型来自 `@synra/client`（[23](./23-client-library.md)）——在 ui/tsconfig.json 里加一行：
//       "types": ["@synra/client"]
//   之后**不需要任何 import** 就能写 synra.xxx。

console.log(synra.pluginId, synra.capabilities)   // 只读

createApp(App).mount('#app')
```

**形状**：完整定义见 [23 §4](./23-client-library.md)。这里只列"作者会用到什么"的概览：

```ts
// 概念概览（权威定义在 @synra/client）
interface SynraClient {
  readonly pluginId: string
  readonly version: string
  readonly capabilities: readonly string[]        // 已授予的能力（同步快照）

  readonly env: EnvNamespace                      // 只读环境（hostKind / platform / locale / can / platformSupports）
  readonly log: LogNamespace
  readonly event: EventNamespace                  // emit / subscribe
  readonly device: DeviceNamespace                // list / send / broadcast / onChange
  readonly storage: StorageNamespace              // get / set / delete / keys / quota
  readonly network: NetworkNamespace              // fetch（唯一的出网路径）
  readonly transfer: TransferNamespace            // send / status / onProgress
  readonly platform: PlatformNamespace            // openUrl / notify
  readonly clipboard: ClipboardNamespace          // read / write
  readonly self: SelfNamespace                    // invoke / actions（调本插件的 headless）

  readonly isSynraError: (e: unknown) => e is SynraError
  readonly isCapabilityDenied: (e: unknown) => e is SynraError & { code: 'capability_denied' }

  ready(): void                                   // 页面生命周期
}
```

**明确没有的东西**（安全考量）：

| 没有                     | 为什么                                                  |
| ------------------------ | ------------------------------------------------------- |
| `platform.simulateKeys`   | UI 插件永远不能模拟按键                                   |
| `fs.*`                    | UI 插件用 `network.fetch` 与 `transfer` 即可；直接 FS 风险过高 |
| 任何 `plugin.*` 管理方法   | 插件不能安装/启用/停用其它插件，也不能替用户授权            |
| `config.set`              | 插件只能读写自己的 storage，不能改全局配置                 |
| `localStorage` / `indexedDB` | WebView 层面禁用（无持久化 profile），持久化只走 `storage`  |
| 直接的 `fetch`            | CSP `connect-src 'none'` 会拦住                          |

### 5.3 UI 侧的构建

```
ui/src/** ──vite build（base: './'）──▶ ui/dist/{index.html, assets/*}
   │  synra-plugin build
   ▼
dist/ui.zip          （zip 内路径 < 256 字符，拒绝符号链接，禁止 ../ 穿越）
```

**构建期校验**（写进 `verify`）：

| 检查                                                | 为什么                              |
| --------------------------------------------------- | ----------------------------------- |
| `base: './'`（产物里是相对路径）                       | 要在 `synra-plugin://` 自定义协议下工作 |
| 无 `eval` / `new Function`（CSP 无 `unsafe-eval`）     | 否则页面里脚本直接报错                |
| 无内联 `<script>` 之外的远程 URL 引用                   | CSP `script-src 'self'` 会拦          |
| 产物里不出现 `http://` / `https://` 的 API 调用         | `connect-src 'none'` 会拦；必须走 `synra.network.fetch` |

---

## 6. 一张图看数据流（分享 → 远端执行）

```
 手机（发起端）                                                         PC（目标端）
─────────────────────────────────────────────────────────────────────────────────────────
 用户分享 "imba97/smserialport"
        │
        ▼
 ┌──────────────────┐
 │ 内核 ShareIntake  │ ① 规范化 → ShareItem{kind: Text, text: "imba97/smserialport"}
 └────────┬─────────┘
          │ ② 预筛：查 manifest.shareHandlers（声明了 text/url 的插件）
          │     ★ 不调任何插件就能排除掉大半
          ▼
 ┌──────────────────┐
 │  gateway          │ ③ 对每个候选调 can_handle(item)
 │  （能力闸门）      │     · fuel 预算 50ms + 同步
 └────────┬─────────┘     · SDK 类型上禁止有副作用的调用（audit 为空是断言）
          │                 ┌────────────────────────────┐
          ├────────────────▶│ Wasm 实例（github-bridge）  │
          │                 │  can_handle("imba97/smser…") │
          │◀────────────────│  → handled("open-url",       │
          │   候选            │      {url:"https://gith…"})  │
          ▼                 └────────────────────────────┘
 ┌──────────────────┐
 │ 决策              │ ④ 唯一候选 + 插件配置为"需确认" → 弹确认框
 └────────┬─────────┘    用户确认 → IntentState::Dispatching
          │ ⑤ capability 闸门（下发前一次性校验三件事）
          │     · 该插件被授予 device:send:<target>
          │     · 目标设备已安装该插件且版本兼容
          │     · 目标设备的 PlatformCapabilities 覆盖所需能力
          ▼
 ┌──────────────────┐
 │ synra-net         │ ⑥ Envelope { event: "_plugin.github-bridge.open-url",
 └────────┬─────────┘                request_id, trace_id, payload:{url} }
          │
          └──────────────── TCP（端口 32100）──────────────────────────▶
                                                                        ▼
                                                            ┌──────────────────────┐
                                                            │ 内核入站 + 事件路由    │
                                                            │  按 handles/动作归属  │
                                                            │  找到 github-bridge    │
                                                            └──────────┬───────────┘
                                                                       │ ⑦ 闸门：platform:open-url
                                                                       ▼
                                                            ┌──────────────────────┐
                                                            │ Wasm 实例             │
                                                            │  execute("open-url")  │
                                                            └──────────┬───────────┘
                                                                       │ ⑧ synra::platform::open_url(url)
                                                                       ▼
                                                            ┌──────────────────────┐
                                                            │ 宿主壳（Tauri）        │
                                                            │  Platform 实现 → 浏览器 │
                                                            └──────────────────────┘
```

---

## 7. 一张图看运行期对象关系

```
                        ┌───────────────────────────────────────────────┐
                        │  synra-core（内核进程）                    │
                        │                                               │
   ┌──────────────┐     │  ┌─────────────────┐   ┌───────────────────┐  │
   │ synra-net    │◀────┼──│ synra-core      │──▶│ synra-plugin-host │  │
   │ 发现/传输     │     │  │ IntentRouter    │   │                   │  │
   └──────────────┘     │  │ EventBus        │   │  ┌─────────────┐  │  │
                        │  └────────┬────────┘   │  │ Engine      │  │  │
                        │           │            │  │ (wasmtime)  │  │  │
                        │           ▼            │  └──────┬──────┘  │  │
                        │  ┌─────────────────┐   │         │         │  │
                        │  │ gate.rs  ★闸门   │◀──┼─────────┤         │  │
                        │  └─────────────────┘   │         ▼         │  │
                        │                        │  ┌─────────────┐  │  │
                        │                        │  │ InstancePool│  │  │
                        │                        │  │ per plugin  │  │  │
                        │                        │  └──────┬──────┘  │  │
                        │                        └─────────┼─────────┘  │
                        └──────────────────────────────────┼────────────┘
                                                           │ host functions
                        ┌──────────────────────────────────┼────────────┐
                        │  Wasm 实例 × N（每个插件一个池）    │            │
                        │  ┌────────────────────────────┐  │            │
                        │  │ 线性内存（每实例独立）        │◀─┘            │
                        │  │  代码 + 数据段 + heap        │               │
                        │  │  ❌ 实例之间的可变状态不共享   │               │
                        │  └────────────────────────────┘               │
                        └───────────────────────────────────────────────┘
                                              ▲
                                              │ ⑨ SelfInvokeAction
                        ┌─────────────────────┴─────────────────────────┐
                        │  synra-app（Tauri）                       │
                        │  ┌──────────────────────┐  ┌────────────────┐ │
                        │  │ commands/（~40 个）   │  │ plugin_ui.rs    │ │
                        │  │  plugin_ui_rpc ★令牌  │─▶│  创建 WebView   │ │
                        │  └──────────┬───────────┘  │  CSP / origin   │ │
                        │             │              │  注入 window.synra│ │
                        │             ▼              └────────┬───────┘ │
                        │  ┌──────────────────────┐           │         │
                        │  │ 宿主 UI（宿主自己的 Vue）│           │         │
                        │  └──────────────────────┘           │         │
                        │  ┌──────────────────────────────────▼───────┐ │
                        │  │ 插件 WebView（独立 origin，自带 Vue）      │ │
                        │  │  只能通过 window.synra → plugin_ui_rpc     │ │
                        │  └──────────────────────────────────────────┘ │
                        └───────────────────────────────────────────────┘
```

### 7.3 关键点：UI → headless 的两条路

| 场景                          | 走哪条路                                     | 闸门                                       |
| ----------------------------- | -------------------------------------------- | ------------------------------------------ |
| UI 想发消息给别的设备           | `synra.device.send(...)`（**直接**）           | `device:send:<target>`                     |
| UI 想做一件需要 headless 逻辑的事 | `synra.self.invoke("send-message", args)`     | 内核校验该 action **属于本插件**，再按 headless 的 grant 校验 |

**为什么要有第二条路**：UI 插件是**最不可信的插件形态**（JS、在 WebView 里、无法被内核中断）。
所以：
1. 高危能力（`fs:*`、`platform:simulate-keys`）**只从 headless 出口**——UI 想用就必须"请 headless 帮忙"，
   而 headless 的行为是**可审计、可限流、可中断**的；
2. UI 侧的能力面因此可以做到很小（§5.2 的"明确没有的东西"）。

---

## 8. 三类交互的时序图

### 8.1 分享匹配（内核主动调插件）

```
内核 ShareIntake          gateway(闸门)         Wasm 实例(GithubBridge)
      │                        │                        │
      │ can_handle(item)       │                        │
      ├───────────────────────▶│                        │
      │                        │ 检查：该调用是否只需 host:env
      │                        │ 配额：fuel=预算, timeout=50ms
      │                        ├───────────────────────▶│
      │                        │                        │ 纯计算
      │                        │◀───────────────────────┤
      │                        │ 审计：Outcome::Ok       │
      │◀───────────────────────┤                        │
      │ ShareDecision::Handled │                        │
```

**断言（测试）**：`can_handle` 执行完后 **audit 表里除了"调用本身"不应有任何 host call 记录**。
这是"匹配必须无副作用"的**可执行定义**（[21 §8.3.3](./21-testing-strategy.md)）。

### 8.2 UI 调 headless（`SelfInvokeAction`）

```
插件 WebView           app: plugin_ui_rpc       core: gate        Wasm(GithubBridge)
    │                        │                     │                    │
    │ synra.self.invoke(     │                     │                    │
    │   "send-message", a)   │                     │                    │
    ├───────────────────────▶│                     │                    │
    │  (token 闭包捕获)       │ 校验 token → pluginId│                    │
    │                        │ 方法白名单：SelfInvokeAction               │
    │                        ├────────────────────▶│                    │
    │                        │                     │ ① action 属于本插件？
    │                        │                     │ ② headless 有 device:send 授权？
    │                        │                     ├───────────────────▶│
    │                        │                     │                    │ execute(...)
    │                        │                     │                    │ device::send(...)
    │                        │                     │                    │ storage::set(...)
    │                        │                     │◀───────────────────┤
    │◀───────────────────────┴─────────────────────┤                    │
    │ { messageId }                                │                    │
```

### 8.3 跨设备：UI 直接发（不走 headless）

```
插件 WebView         app: plugin_ui_rpc      core: gate        synra-net
    │                       │                    │                │
    │ synra.device.send(    │                    │                │
    │   "laptop","chat.text",p)                  │                │
    ├──────────────────────▶│                    │                │
    │                       ├───────────────────▶│                │
    │                       │  闸门：device:send:laptop              │
    │                       │  （UI 侧的 grant 快照，不是 headless 的）│
    │                       ├───────────────────────────────────▶│
    │                       │                    │                │ 发帧
    │◀──────────────────────┴────────────────────┤                │
```

> ⚠️ **这里有一个能力模型问题**（🔴 见 §10-4）：UI 侧与 headless 侧的 grant 是同一套还是两套？
> 目前文档里是**同一套**（manifest 只声明一次 `capabilities`），但这样 UI 侧就间接得到了
> `fs:*` / `platform:simulate-keys` 这些"声明了但 UI 不该有"的能力——靠 §5.2 的**方法白名单**兜住。
> 是否要改成"manifest 里分开声明 `runtime.headless.capabilities` 与 `runtime.ui.capabilities`"？**这是决策点。**

---

## 9. 一个可讨论的具体例子：PS 快捷键插件

这是个能把所有部件都用上的例子（也是旧文档里的原始场景）。

```
手机（UI 插件页）                         PC（headless 执行）
┌──────────────────────────┐            ┌───────────────────────────────┐
│ ps-shortcuts 的 ui.zip     │            │ ps-shortcuts 的 plugin.wasm    │
│                           │            │                               │
│  ┌─────┬─────┬─────┐      │            │  execute("simulate", {        │
│  │撤销 │重做 │魔棒 │      │            │    chord: "Ctrl+Z" })          │
│  └──┬──┴─────┴─────┘      │            │    → platform::simulate_keys() │
│     │ 点击                │            │      ★ 过闸门 platform:simulate-keys
│     ▼                     │            │      ★ 需要用户在**本机**显式开启该能力
│ synra.self.invoke(        │            │         （默认不可授予，红色警示）│
│   "simulate",             │            │                               │
│   {chord:"Ctrl+Z"})       │            │                               │
└──────────┬───────────────┘            └───────────────▲───────────────┘
           │ 过闸门：action 属于本插件                      │
           └────────────── 跨设备 ────────────────────────┘
```

**这个例子暴露的问题**（值得讨论）：

| # | 问题                                                                   | 我的倾向                                            |
| - | ---------------------------------------------------------------------- | --------------------------------------------------- |
| 1 | 按键要在 100ms 内生效——跨设备往返 + Wasm 调用 + 闸门，够快吗？              | 够（LAN RTT 1ms + host call 20μs），但需要实测        |
| 2 | `simulate_keys` 需要 PC 上开辅助功能/无障碍权限，插件怎么知道"能不能用"？    | `synra::platform::capabilities()` + `env::can()`；UI 侧显示禁用态并给出引导 |
| 3 | 面板上的按钮列表是硬编码还是配置？                                        | `pages` + `storage`（插件自己管配置）                 |
| 4 | 如果用户连了两台 PC，按钮怎么知道发给谁？                                  | 需要"选择目标设备"的通用 UI（内核提供，见 [08 §2.2](./08-ipc-and-frontend.md) 的 `devices_snapshot`） |
| 5 | 长按连发（比如"按住连续撤销"）怎么办？                                     | UI 侧节流 + 每次 `self.invoke`；或需要一个"流式"能力（🔴 未设计） |

---

## 10. 🔴 尚未拍板的设计缺口（讨论清单）

按"影响面 × 现在不定下来的代价"排序。

### 10-1 ★ 定时任务的载体

**问题**：`timers` 声明了 `flush-outbox` 每 30s 跑一次。但插件是**池化、可能无实例**的——
谁来触发？触发时用哪个实例？

**选项**：

| 选项                                            | 优点                          | 缺点                                              |
| ----------------------------------------------- | ----------------------------- | ------------------------------------------------- |
| A. 内核调度器持有定时器，到点调 `on_timer`        | 简单；与实例池解耦             | 插件无法自己做复杂调度；内核要有定时任务注册表        |
| B. 插件在 `activate` 里调 `synra::timer::every()` | 更灵活                        | 需要"活的实例"；与池化冲突（实例可能已被回收）        |
| C. 不支持定时任务，让插件用 `handles` 响应外部事件 | 最简单，能力面最小              | 有些插件真的需要（outbox 重试、心跳、缓存清理）       |

**倾向**：**A**（内核调度 + `on_timer` 入口）。理由：与池化一致、可配额、可观测。
**需要确认**：定时任务的最小间隔（防插件高频占用内核）、是否允许插件动态增删定时器。

---

### 10-2 ★★ `can_handle` 的批量与能力边界

**问题**：现在 `can_handle(item) -> ShareDecision` 是**一个 item 一次调用**。如果有 10 个候选插件 × 1 个 item，
就是 10 次 Wasm 调用。如果有 5 个 item（多选分享）就是 50 次。

**选项**：

| 选项                                       | 优点                     | 缺点                              |
| ------------------------------------------ | ------------------------ | --------------------------------- |
| A. 保持单项（现在）                          | 简单、SDK 好用            | 调用次数 = O(插件数 × item 数)      |
| B. 批量：`can_handle(items) -> Vec<Decision>` | 一次调用处理多个           | 有副作用风险（插件可能跨 item 推断）  |
| C. 批量 + 强制纯函数（只允许 env import）      | 批量的效率 + 无副作用       | 需要 WIT 层面拆出一个"纯接口"        |

**倾向**：先 **A**，`shareHandlers` 预筛（§3.1）把插件数降下来。
**但需要确认**：预筛的准确率——如果预筛太粗（比如所有声明 `text` 的插件都进），A 的调用数还是很高。

**关联的更深问题**：**"纯函数"这个约束怎么在类型上表达？**
现在只有文档说"can_handle 不许有副作用"。真正做的话，WIT 里应该给 `can_handle` 一个**只 import 了 `env` 的独立接口**——
这样宿主在实例化时就能保证它调不到任何有副作用的东西。**这是否要做？**（我倾向要做，它是 [21 §8.3.3](./21-testing-strategy.md) 那条"安全属性必须显式断言"的编译期版本。）

---

### 10-3 ★★ 实例池 vs 插件状态（最容易埋坑的地方）

**问题**：池化的前提是**无状态**。但插件作者很自然想写：

```rust
// 作者会很自然地这么写 —— 这是个坑
static OUTBOX: OnceCell<Mutex<Vec<Message>>> = OnceCell::new();

fn execute(...) {
    OUTBOX.get().unwrap().lock().push(msg);   // 写进实例 A 的内存
}
fn on_timer(...) {
    for m in OUTBOX.get().unwrap().lock().drain(..) { send(m) }   // 可能在实例 B 里跑 → 空！
}
```

不同实例有**各自的线性内存**，所以这段代码会**偶发失效**（取决于哪个实例被租出来）。

**选项**：

| 选项                                              | 优点                        | 缺点                                        |
| ------------------------------------------------- | --------------------------- | ------------------------------------------- |
| A. manifest 声明 `stateful: true` → 不池化（每调用新实例） | 语义明确                | 每次调用都要实例化（慢）；且"新实例"意味着**状态丢失**，作者还是得用 storage |
| B. SDK 在类型上禁止可变全局（`static mut` / `OnceCell<Mutex>` lint） | 从根上避免坑 | 需要自定义 lint；作者可能用 `unsafe` 绕过（但 workspace 里 `forbid(unsafe_code)`） |
| C. 每插件一个长驻实例（不池化、不复用池）              | 作者体验最好                | 失去池化的收益；N 个插件 = N 个长驻实例的常驻内存 |
| D. A + B 组合                                       | 双保险                      | 复杂度                                          |

**倾向**：**B +（A 作为逃生舱）**。
- 默认：SDK 禁止可变全局；状态**只能**经 `storage`（跨实例天然一致）；
- 逃生舱：真的需要内存态缓存的插件可以声明 `stateful: true`，宿主给它一个**长驻实例**
  （于是它自己的进程内状态是一致的，但要接受"内核重启即丢失"）。

**需要确认**：
1. 是否接受"默认禁止可变全局"（这会影响插件作者的写法习惯）；
2. `stateful: true` 是"长驻实例"还是"每调用新实例"？（我倾向**长驻实例**，否则声明它没意义）

---

### 10-4 ★ 两套能力声明还是一套

**问题**（见 §8.3 的 ⚠️）：manifest 现在只有一套 `capabilities`。
UI 部分与 headless 部分的能力面**本质上不同**（§5.2 vs §4.4），但声明是共享的。

**选项**：

| 选项                                                              | 优点                    | 缺点                                |
| ----------------------------------------------------------------- | ----------------------- | ----------------------------------- |
| A. 一套 `capabilities`，UI 侧靠**方法白名单**兜住                     | 简单；作者不用想两遍       | UI 侧的拒绝是"运行期才暴露"，作者体验差 |
| B. 分两套：`runtime.headless.capabilities` + `runtime.ui.capabilities` | 精确；作者一眼看出 UI 能用什么 | manifest 更啰嗦；授权 UI 要展示两组   |
| C. 一套 + 明确的"UI 可见能力子集"规则（在 schema 里标注）               | 折中                     | 作者还是要查表                       |

**倾向**：**B**（虽然啰嗦，但**声明即契约**——作者写 manifest 时就知道 UI 能做什么，而不是跑起来才发现被拒）。
**需要确认**：授权弹窗是否要分两组展示（"逻辑部分需要" / "界面部分需要"）。

---

### 10-5 ★★ 入站事件的分发模型

**问题**：这是整套设计里**最不确定**的一块。

内核收到 `_plugin.chat.text` 或 `file.transfer.complete`，要交给插件的 `on_event`。
但**插件是池化的**，"交给哪个实例"没有天然答案。而且"订阅"这个概念在池化模型下不成立
（订阅意味着一个**稳定的回调目标**）。

**选项**：

| 选项                                                          | 语义                             | 优点                          | 缺点                                              |
| ------------------------------------------------------------- | -------------------------------- | ----------------------------- | ------------------------------------------------- |
| A. **声明式路由**（manifest `handles` + 内核调用 `on_event`）    | "我看到符合条件的事件就调你"        | 与池化一致；可审计、可限流       | 插件不能"运行时动态订阅"；每个事件都要过一次 Wasm 调用 |
| B. **订阅回调**（`event::subscribe` + guest callback）          | 传统发布/订阅                     | 作者熟悉                       | 与池化冲突（回调目标不稳定）；需要长驻实例           |
| C. **显式拉取**（内核把待处理事件排队，插件在 `on_timer` 或下次调用时拉） | 插件自己掌握节奏          | 与池化完全兼容；天然批处理        | 实时性差（要靠 timer 唤醒）                          |
| D. A + 需要长驻的插件用 `stateful: true`                        | 混合                            | 兼顾                          | 两套语义                                          |

**倾向**：**A**（声明式路由）作为默认，**D** 作为逃生舱。
具体形态：
```
manifest.handles = [ { event: "_plugin.chat.*" }, { event: "file.transfer.complete", kind: "attachment" } ]
                    ↓ 内核收到匹配事件
                Wasm: on_event(InboundEvent { event, from, payload, reply_to })
                    ↓ 需要回执时
                返回 Result<Option<Value>, Error>  → 内核按 reply_to 发回
```

**需要确认的问题**：
1. `on_event` 返回 `Option<Value>`（回执）还是 void（只能自己用 `device::send` 回）？
2. 事件量大时（比如每块 `file.transfer.chunk`）会不会把内核拖垮？需要"批量投递"吗？
3. `handles` 的匹配语法和 `capabilities` 里的 `event:emit:*` 是两套还是复用？

---

### 10-6 ★ 定时任务的间隔下限与配额

**问题**：如果允许 `intervalMs: 100`，一个插件就能每 100ms 唤醒一次 Wasm 实例——
N 个这样的插件会持续消耗 CPU。

**倾向**：
- 下限 **1000ms**（1 秒）；更短的用 timer 没意义（该用事件）；
- 每个插件的定时任务数 ≤ 8；
- 定时任务的调用也消耗 `host_calls_per_sec` 配额；
- 移动端后台时**暂停**定时任务（省电）。

**需要确认**：是否允许"只在有网/有设备在线时才跑"（条件定时器）？

---

### 10-7 ★ `stateful: true` 的确切语义

**问题**（与 §10-3 关联）：声明 `stateful: true` 之后：

| 子问题                                     | 我的倾向                                 |
| ------------------------------------------ | ---------------------------------------- |
| 是"每调用新实例"还是"长驻实例"？              | **长驻实例**（每调用新实例的话，状态会丢，声明它就没意义） |
| 内核重启后状态怎么办？                        | **丢失**，插件必须自己用 `storage` 持久化   |
| 能否池化？                                   | 不能（只有一个实例，串行处理）              |
| 内存上限要不要更严？                          | 要（长驻 = 常驻内存，按 1/4 配额）           |
| 是否影响 `on_event` / `on_timer` 的语义？      | 不影响（本来就是内核调它）                   |
| 移动端后台挂起后怎么恢复？                     | 需要一次 `activate` 或一个"恢复"钩子（🔴 未设计） |

**需要确认**：`stateful` 是不是"逃生舱"就够，还是需要更细的（比如"只有缓存是内存态、其余无状态"）。

---

### 10-8 ★ 构建链的确切工具（wasm32-wasip2 的产出形态）

**问题**：Rust 的 `wasm32-wasip2` target 在不同版本下产出不同：
- 较新版本 `cargo build --target wasm32-wasip2` 直接产出**组件**；
- 较早/某些配置下产出**核心模块**，需要 `wasm-tools component new` + adapter。

**这是一个 spike 项**（[19 §0.1](./19-roadmap.md) spike 2/3 顺带验证），但**现在就要定文档口径**：

**倾向**：
- 文档写"`cargo build --target wasm32-wasip2 --release` 产出 Wasm 组件"；
- `synra-plugin build` **自动探测**：如果产出是核心模块，自动跑 `wasm-tools component new`；
- **锁定 `wasm-tools` / `wit-bindgen` 的版本**（写进 `synra-plugin-cli` 的 `Cargo.toml`，并在 `verify` 时检查插件是用哪个版本构建的）。

**需要确认**：是否允许插件作者用自己的 `wasm-tools` 版本（可能导致组件不兼容）。

---

## 11. 给讨论用的三个"最该先定"的问题

如果只能先定三个，我建议是：

```
① §10-5 入站事件的分发模型
   为什么最优先：它决定插件是"事件驱动的对象"还是"被调用的纯函数"。
   选 A/D 还是 B，会反向影响 §10-3（状态）与 §10-1（定时任务）的设计。

② §10-3 实例池 vs 状态
   为什么次优先：这是"插件作者会不会踩坑"的最大来源。
   它同时决定 §10-7（stateful 的语义）。

③ §10-4 两套能力还是一套
   为什么第三：它决定 manifest 的形状，而 manifest 是**最早要被冻结**的东西
   （插件作者要照着它写，改一次就要所有作者改一遍）。
```

> 其余 5 个（§10-2、§10-6、§10-7、§10-8）都可以在 ①②③ 定了之后自然推导出来。

---

## 12. 相关文档

| 想深入什么                          | 去哪                                                          |
| ----------------------------------- | ------------------------------------------------------------- |
| 为什么是双轨（headless + UI）         | [09 §2](./09-plugin-system.md)                                 |
| capability 语法与三层闸门            | [10](./10-capability-and-sandbox.md)                           |
| wasmtime 配置、fuel/epoch、实例池     | [12](./12-wasm-host.md)                                        |
| 安装 / 更新 / 卸载的状态机            | [14](./14-plugin-lifecycle.md)                                 |
| SDK 的 9 个 namespace 与 CLI          | [15](./15-plugin-sdk.md)                                       |
| 签名 / registry / 跨设备同步          | [16](./16-plugin-distribution.md)                              |
| **UI 侧的类型库 `@synra/client`** ★   | [23](./23-client-library.md)                                   |
| UI 插件的 IPC、CSP 与令牌             | [08 §7](./08-ipc-and-frontend.md)                              |
| 契约（WIT world）的完整定义            | `synra-core/wit/synra-plugin.wit`（M2 产出）                |
