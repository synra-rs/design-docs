# 23 · 客户端库 `@synra/client`

> ★ **本文的实现清单已逐条对过源码（本次）。结论：核心几乎全部落地，缺的是"模板侧"两项和一项被上游卡住的测试。**
>
> 已经做好的（位置与本文写的略有出入，但要求都满足）：
> * `packages/client/src/surface.ts` 的 `SynraClient` 与各 namespace；
>   `http.ts` 的 `SynraFetchInit` / `SynraResponse`。
>   ★ 本文写的 `errors.ts`、`device.ts` **不存在** —— 错误类型与 `device.pick` 都在 `surface.ts` 里。
>   这是命名差异而不是缺失，但一张按文件名找东西的清单会让人以为它们没写，所以记在这里。
> * `global.ts`（本文写 `global.d.ts`，实际是 `.ts`）；`type-tests/consumer.ts` 是**不带任何 import**
>   的消费方文件 —— 它钉的是"`types: ["@synra/client"]` 一行就够拿到全局 `synra`"，
>   这件事包的自身构建是发现不了的。
> * `package.json`：**没有 `main`/`module`**，`exports` 只给 `types` —— 与本文一致。
>   ★ `files` 写的是 `["dist"]`，本文要求 `["*.d.ts"]`。两者不等价，但**后一道闸门在**：
>   `synra-js/scripts/check-shapes.ts` 会扫描 `dist/`，发现 `.js`/`.mjs`/`.cjs` 就报错。
>   真正保证"没有运行时"的是 `emitDeclarationOnly: true`（在 `packages/client/tsconfig.json` 里，
>   所以 `build: tsc -b` 等价于本文写的 `tsc --emitDeclarationOnly`），
>   `check-shapes` 检查的是**它确实生效了**，而不是相信它生效了。
> * `synra-app` 侧：`src/ipc/injection.ts` 末尾 `} satisfies SynraClient`（`:442`），
>   并且注释解释了为什么必须是 `satisfies` 而不是类型标注 ——
>   标注允许**多**一个字段，而多出来的字段在运行时是 `undefined`。
>   `isSynraError` / `isCapabilityDenied` / `isCode` 由宿主提供；
>   `supported` 列表与行为**来自同一张表**（`'network.fetch': false` 那一张），
>   所以"广告了却拒绝"不可能发生。
> * app 侧测试断言的是**精确键集**（`Object.keys(synra).sort()` 全等），比本文要求的"⊆"更严。
>
> ★ **没做的一项，以及它被卡住的真正位置**：
> 本文要求"形状断言测试（`tsd` 或 `expect-type`）：`SynraResponse` 字段与宿主实现逐一对齐"。
> 这个测试不存在 —— 但**不是因为缺测试工具**，而是因为**没有可对齐的宿主实现**：
> `injection.ts` 的 `BACKED` 表里 `'network.fetch': false`，宿主把它作为 `unsupported` 拒绝，
> 压根没有 `fetch` 实现。所以这一项真正的阻塞点是
> **[10 §4.4](./10-capability-and-sandbox.md) 记的那件事：全仓没有任何 HTTP 客户端**。
> 等宿主有了 `fetch`，`satisfies SynraClient` 会自动强制 `SynraResponse` 的形状 ——
> 那比另写一个 `tsd` 测试更强，因为它是在**唯一那份实现**上强制的。
> 把它写成"缺一个测试"会让人去找测试框架，而该找的是一条 HTTP 通路。
>
> ★ **另外两项没做（模板侧）**：`synra-plugin new` 生成的工程里**没有 `ui/` TypeScript 工程**。
> 现在的模板（`tools/synra-plugin-cli/src/template.rs`）写的是 `manifest.json`、`Cargo.toml`、
> `src/lib.rs`、`README.md`、`.gitignore`，以及一份 `ui/index.html` ——
> 界面是**内联 `<script type="module">` 的纯 HTML**，没有 `.ts` 入口。
> 所以"`ui/tsconfig.json` 默认含 `types: ["@synra/client"]`"和
> "`ui/package.json` 的 devDependencies 含 `@synra/client`"都无从谈起。
>
> ★ **而且这两项现在不该盲目补上，理由是它们会做出一个装不起来的脚手架：**
> `ui/package.json` 里写 `"@synra/client": "^0.1.0"` 意味着作者 `pnpm install` 要能从 registry 拿到它，
> 而 `@synra/client` 目前在 `synra-js` 里是 workspace 包、**尚未发布**。
> 更根本的是：一个只有 `tsconfig.json`、没有任何 `.ts` 输入的工程，
> `tsc` 会直接报 "No inputs were found" —— 所以补这两条真正意味着
> **把 UI 模板从"内联脚本的 HTML"改成"有 TS 入口 + 构建步骤的工程"**，
> 那是一次模板重设计（还牵动 manifest 的 `runtime.ui` 入口声明与 `verify` 的产物检查），
> 不是加两个文件。
> 依赖发布是**需要用户/线上操作**的事，按本次的目标先不做，在此记明。
> 真正待做的顺序是：先发布 `@synra/client` → 再把 UI 模板改成 TS 工程 → 最后这两条自然成立。

> 本文定义插件 UI 侧的**类型库** `@synra/client`：它属于 **`synra-js` JS monorepo**（[25](./25-js-monorepo.md)），
> 是一个 **`.d.ts` only 包**——编译后**只有类型声明，没有任何 `.js`**。
>
> 前置：[25 JS 包 monorepo](./25-js-monorepo.md)（仓库与包的划分）、[13 §5 UI 部分](./13-plugin-authoring.md)（`window.synra` 的运行时形状）、[08 §7](./08-ipc-and-frontend.md)（UI 插件的 IPC 与令牌）。
> 读者：插件作者 + `synra-app` 的实现者。

---

## 1. 一句话定义

> **`@synra/client` 是一个纯类型包**：它把"宿主注入的 `synra` 对象"的形状、全局声明、以及所有相关类型一次性给全。
> 插件作者**不 import 任何东西**就能写 `synra.xxx`；需要显式引用时用 `import type`。

```jsonc
// 插件的 tsconfig.json —— 一行搞定
{
  "compilerOptions": {
    "types": ["@synra/client"]
  }
}
```

```ts
// 不需要任何 import
synra.device.send({ to: 'dev_…', event: 'chat.text', payload: { body: 'hi' } })
synra.storage.set('theme', 'dark')
synra.log.info('mounted')
```

**关键性质**：

| 性质                | 说明                                                                        |
| ------------------- | --------------------------------------------------------------------------- |
| **零运行时**          | 包内没有任何 `.js`；`exports` 只给 `types`。不存在"两份实现打架"的问题            |
| **全局注入**          | 只要 `types` 里包含它，`synra` 与 `window.synra` 都有类型，**无需 import**        |
| **契约的唯一来源**     | 类型描述宿主注入的对象；**宿主按这个类型实现**，不是反过来                        |
| **不依赖任何运行时库** | 连 Vue 都不依赖（作者用什么框架与它无关）                                       |

### 1.1 为什么必须是"零运行时"

如果包里带实现（哪怕只是 `export const synra = globalThis.synra`），插件打包时会带上它，
于是出现**两份东西**：包里的那份，和宿主真正注入的那份。它们版本不一致时会打架
（类型对了、行为不对，或反之）。

**零运行时**从结构上消灭这个问题：**运行时对象的唯一来源就是宿主注入**。

> 因此：`device.onChange` / `device.pick` / `isSynraError` 这类需要**逻辑**的方法，
> 都是**声明为宿主必须提供**的（§4.4），而不是在包里实现。

### 1.2 这个包属于一个多包仓库

`@synra/client` 不是孤立的包，它和 `synra` / `@synra/protocol` / `@synra/testkit` 一起构成
**`synra-js` monorepo**（详见 [25](./25-js-monorepo.md)）：

```
synra-js/packages/
├─ synra/               # 类型汇总入口（作者一般不装，靠传递依赖解析）
├─ client/              # ★ 本文：@synra/client
├─ protocol/            # 线协议类型（Rust 生成）
└─ testkit/             # 插件作者本地的 mock 运行时（带实现）
```

---

## 2. 包形态

### 2.1 package.json

```jsonc
{
  "name": "@synra/client",
  "version": "1.0.0",
  "type": "module",
  "description": "Types for the Synra plugin UI runtime. Declaration files only — no runtime code.",
  "license": "MIT",
  "publishConfig": { "access": "public" },

  // ★ 没有 main / module —— 纯类型包（运行期导入会明确失败，这是有意的）
  "types": "./dist/index.d.ts",
  "exports": {
    ".":         { "types": "./dist/index.d.ts" },
    "./global":  { "types": "./dist/global.d.ts" },
    "./errors":  { "types": "./dist/errors.d.ts" }
  },

  // ★ 发布物（构建后 dist/ 里只有 .d.ts，见 [25 §7.2](./25-js-monorepo.md)）
  "files": ["dist"],
  "sideEffects": false,

  "scripts": {
    "stub": "unbuild --stub",
    "build": "unbuild",
    "test": "vitest run"
  },

  "dependencies": {
    "@synra/protocol": "workspace:^",   // 数据类型（会被内联进 .d.ts）
    "synra": "workspace:^"              // ★ 让 synra 成为传递依赖（§2.4）
  },
  "devDependencies": {
    "unbuild": "^3.6.1",
    "tinyglobby": "^0.2.17",
    "typescript": "^7.0.2",
    "vitest": "^5.0.3"
  }
}
```

> **构建用 `unbuild`**（与本机既有项目一致）。⚠️ 关键事实：`unbuild` **一定会产出 `.js` 空壳**
> （实测 1 字节），所以"零运行时"要靠 `build:done` hook 剥离 ——
> 完整配方、多入口的产出差异、以及 `respectExternal` 的选择见 [25 §7](./25-js-monorepo.md)。
>
> **`src/index.ts` 必须 `import './global'`**（side-effect import），否则 `types: ["@synra/client"]`
> 不会带上 `declare global`（[25 §7.3](./25-js-monorepo.md) 有实测对比）。

### 2.2 目录结构

```
packages/client/
├─ src/
│  ├─ index.ts             # 主入口：re-export 类型 + 引入 global 声明
│  ├─ global.ts            # ★ declare global（synra、window.synra、Unsubscribe）
│  ├─ surface.ts           # SynraClient 接口
│  ├─ namespaces/          # 10 个 namespace 的接口（纯类型）
│  │  ├─ env.ts  log.ts  event.ts  device.ts  storage.ts
│  │  └─ network.ts  transfer.ts  platform.ts  clipboard.ts  self.ts
│  ├─ http.ts              # ★ SynraResponse / SynraFetchInit（与宿主共用的形状契约）
│  └─ errors.ts            # SynraError 类型与判别式
├─ dist/                   # 构建产物：只有 .d.ts
└─ package.json
```

> **构建时不产出 `.js`**：`tsc` 配 `"emitDeclarationOnly": true`。
> CI 守卫：**`dist/` 下不允许出现 `.js` / `.mjs` / `.cjs`**（[25 §6](./25-js-monorepo.md)）。
> 这是"零运行时"承诺的可执行检查。

### 2.3 与 `@synra/protocol` 的分工

| 包                 | 内容                                                                          | 谁维护            |
| ------------------ | ----------------------------------------------------------------------------- | ----------------- |
| `@synra/protocol`   | 由 Rust 用 `ts-rs` **生成**的数据类型（`DeviceId` / 各 payload / `SynraErrorCode`…） | 生成（core 仓库）   |
| `@synra/client`     | 手写的注入对象契约（`SynraClient` + `declare global` + `SynraResponse`）           | 手写（JS 仓库）     |

**`@synra/client` 依赖 `@synra/protocol`，不依赖 app 的任何东西。**

### 2.4 `synra` 依赖：作者"一般不用安装"的机制

`@synra/client` 的 `dependencies` 里声明了 `synra`（[25 §3](./25-js-monorepo.md)）。
于是：**作者只要装了 `@synra/client`，`synra` 就已经在 `node_modules` 里**，
`import type { … } from 'synra'` 开箱可用——**不需要在 package.json 里显式安装它**。

```jsonc
// 插件作者的 ui/package.json —— 只装 client，不装 synra
{
  "devDependencies": {
    "@synra/client": "^1.0.0"
  }
}
```

```ts
// 但写 'synra' 也能解析（传递依赖）
import type { SynraError } from 'synra'
```

---

## 3. 全局类型注入

### 3.1 `global.d.ts` 的内容

```ts
// @synra/client/global.d.ts —— 只做声明
import type { SynraClient } from './surface'

declare global {
  /**
   * 宿主注入的 Synra 客户端。
   *
   * ★ 由宿主的 initialization script 注入；在**插件的 UI 上下文**里必然存在。
   * 宿主自己的 UI 页面里它**不存在**（宿主走 Tauri command，不用这个对象）。
   */
  const synra: SynraClient

  interface Window {
    /** 与全局 `synra` 是同一个对象。用于"可能不在插件里"的代码做安全判空。 */
    readonly synra?: SynraClient
  }

  /** 一次事件订阅的退订函数 */
  type Unsubscribe = () => void
}

export {}
```

### 3.2 三种写法都受支持（同一类型）

| 写法                          | 何时用                                                                  |
| ----------------------------- | ----------------------------------------------------------------------- |
| `synra.device.send(…)`         | **默认用法**。不用 import、不用判空                                        |
| `window.synra?.device.send(…)` | 当同一份代码可能在"插件页"与"非插件环境"（预览、单测、宿主页）都跑时          |
| `globalThis.synra`             | 需要绕过局部变量遮蔽时（不推荐，但类型上支持）                               |

**要求**：

| # | 要求                                                                     |
| - | ------------------------------------------------------------------------ |
| 1 | `types: ["@synra/client"]` 必须**开箱即用**（不需要额外 `include`、triple-slash） |
| 2 | 全局 `synra` 与 `window.synra` 是**同一类型**（不允许形状不一致）               |
| 3 | `global.d.ts` 不导出任何运行时值                                           |
| 4 | 类型里不出现 `any`；不确定的地方用 `unknown`，且必须有办法收窄                   |

### 3.3 关于显式 import（重要）

因为包内没有运行时值，**不要写** `import { synra } from '@synra/client'`
——那是在 import 一个不存在的运行时绑定，打包器会报错或产出 `undefined`。

需要显式引用时，用 `import type`：

```ts
import type { SynraClient, Unsubscribe, SynraError } from '@synra/client'
import type { DeviceTarget, TransferId } from '@synra/protocol'
import type { SynraError as Err2 } from 'synra'          // 等价（§2.4）

const off: Unsubscribe = synra.device.onChange(d => { /* d 自动是 DeviceTarget[] */ })
```

> 想要"带 import 的写法"时，用 `globalThis.synra`（全局命名空间里，类型可用且**不需要 import**）。

### 3.4 五层类型来源

```
synra-core/crates/synra-protocol    （Rust 类型 = 数据层的唯一真相源）
        │  ts-rs 生成 → commit 到 JS 仓库
        ▼
@synra/protocol                           （数据类型）
        │  被引用
        ▼
@synra/client                             （注入契约 + declare global）
        │  tsconfig types / import type
        ▼
synra-app 的注入脚本                  （★ 按同一类型实现，保证两侧字段一致）
        │
        ▼
插件作者的 ui/src/**                        （业务代码，零 import 也能写）
```

---

## 4. 导出面（全部是类型）

### 4.1 `SynraClient`

```ts
export interface SynraClient {
  /** 该插件的 id（宿主注入时已知，作者不需要判断"我是谁"） */
  readonly pluginId: string
  readonly version: string

  /** 该插件**已被授予**的能力（注入时的同步快照） */
  readonly capabilities: readonly string[]

  // ── 无闸门（只读环境信息）──
  readonly env: EnvNamespace

  // ── 受能力闸门约束 ──
  readonly log: LogNamespace
  readonly event: EventNamespace
  readonly device: DeviceNamespace
  readonly storage: StorageNamespace
  readonly network: NetworkNamespace
  readonly transfer: TransferNamespace
  readonly platform: PlatformNamespace
  readonly clipboard: ClipboardNamespace

  /** ★ UI → 本插件 headless 部分（[13 §7.3](./13-plugin-authoring.md)） */
  readonly self: SelfNamespace

  /** 错误判定便捷函数（§4.4：由**宿主**实现） */
  readonly isSynraError: (e: unknown) => e is SynraError
  readonly isCapabilityDenied: (e: unknown) => e is SynraError & { code: 'capability_denied' }

  /** 告诉宿主"页面已挂载完成"（加载态切换与超时判定）。同步、无副作用。 */
  ready(): void
}
```

### 4.2 各 namespace

```ts
export interface EnvNamespace {
  hostKind(): 'desktop' | 'mobile'
  platform(): 'windows' | 'macos' | 'linux' | 'android' | 'ios'
  locale(): string
  darkMode(): boolean
  appVersion(): string
  sdkVersion(): string
  /**
   * 渲染这个页面的 WebView 引擎，以及它报告的版本。
   *
   * ★ **只用于渐进增强**：一个页面必须在它所支持的最弱引擎上可用，不能把 `kind === 'webview2'` 当作功能是否存在
   * 的条件。`version` 为 `null` 表示引擎没有在 UA 里报版本（例如 WebKitGTK），不是"版本未知但可以当新版"。
   */
  webview(): WebviewInfo
  /** 宿主**平台**是否支持该能力（例：iOS 无 simulate_keys）——与 `can()` 不是一回事 */
  platformSupports(capability: PlatformCapability): boolean
  /** 宿主**版本**是否实现了某个客户端方法 */
  hostSupports(feature: ClientFeature): boolean
  /** 本插件是否**被授予**该能力 */
  can(capability: string): boolean
}

export type WebviewKind = 'webview2' | 'wkwebview' | 'webkitgtk' | 'android-webview' | 'unknown'
export interface WebviewInfo {
  kind: WebviewKind
  /** 引擎没有报版本时为 `null`，不是猜一个 */
  version: string | null
}

export interface LogNamespace {
  trace(msg: string, ctx?: Record<string, unknown>): void
  debug(msg: string, ctx?: Record<string, unknown>): void
  info(msg: string, ctx?: Record<string, unknown>): void
  warn(msg: string, ctx?: Record<string, unknown>): void
  error(msg: string, ctx?: Record<string, unknown>): void
  child(prefix: string): LogNamespace
}

export interface EventNamespace {
  emit(event: string, payload?: unknown): Promise<void>
  /** ★ 一定返回退订函数——否则插件页面卸载时会泄漏 */
  subscribe<T = unknown>(pattern: string, handler: (payload: T) => void): Unsubscribe
}

export interface DeviceNamespace {
  /** 已配对且未屏蔽的设备；保留离线项，通过 online 表达连接状态。需要 device:list。 */
  list(): Promise<DeviceTarget[]>
  send(req: { to: DeviceId; event: string; payload?: unknown }): Promise<void>
  broadcast(req: { event: string; payload?: unknown }): Promise<void>
  /** 设备集合变化（宿主侧包了一层 subscribe + 合并去重） */
  onChange(handler: (devices: DeviceTarget[]) => void): Unsubscribe

  /**
   * ★ 调起**宿主提供的选设备 UI**，让用户挑一台设备。
   *
   * 为什么需要它：插件作者自己画一个设备选择器，会与宿主的视觉/行为不一致；
   * 而且在"多台 PC"场景下，"这个按钮发给谁"是一个**用户决策**，不该由插件猜。
   *
   * 语义：
   *   · 用户选中       → resolve(deviceId)
   *   · 用户取消       → resolve(null)
   *   · 宿主尚未实现该能力 → reject(SynraError{ code: 'unsupported' })
   *     （所以作者可以先 `env.hostSupports('device.pick')` 再决定是否用）
   */
  pick(options?: PickOptions): Promise<DeviceId | null>

  /** 宿主是否支持某个客户端方法（用于渐进增强，见 §4.5） */
  readonly supported: readonly ClientFeature[]
}

export interface PickOptions {
  title?: string
  /** 只列出装了某插件的设备 */
  requiresPlugin?: PluginId
  /** 只列出具备某能力的设备 */
  requiresCapability?: string
}

export interface StorageNamespace {
  // 重载：给了 fallback 则返回值非可选
  get<T = unknown>(key: string): Promise<T | undefined>
  get<T>(key: string, defaults: { fallback: T }): Promise<T>
  set<T = unknown>(key: string, value: T): Promise<void>
  delete(key: string): Promise<void>
  keys(prefix?: string): Promise<string[]>
  quota(): Promise<{ usedBytes: number; maxBytes: number }>
}

export interface NetworkNamespace {
  /** ★ 唯一的出网路径（CSP `connect-src 'none'` 会让直接 fetch 失败） */
  fetch(input: string, init?: SynraFetchInit): Promise<SynraResponse>
}

export interface TransferNamespace {
  send(req: { to: DeviceId; file: Blob; fileName: string; contextId?: string }): Promise<TransferId>
  status(id: TransferId): Promise<TransferStatus>
  onProgress(id: TransferId, handler: (p: TransferProgress) => void): Unsubscribe
}

export interface PlatformNamespace {
  openUrl(url: string): Promise<void>
  notify(req: { title: string; body: string }): Promise<void>
  // ★ 没有 simulateKeys、没有 fs：
  //   UI 插件永远不能做这两件事（[13 §5.2](./13-plugin-authoring.md)）；
  //   需要时经 `synra.self.invoke()` 交给 headless 部分。
}

export interface ClipboardNamespace {
  read(): Promise<string>
  write(text: string): Promise<void>
}

export interface SelfNamespace {
  /**
   * 调本插件 headless 部分声明的 action。
   * ★ 这是 UI 拿到高危能力的**唯一**途径（例如模拟按键必须由 headless 做）。
   */
  invoke<Args = unknown, Result = unknown>(action: string, args?: Args): Promise<Result>
  /** 列出本插件 headless 声明了哪些 action（用于动态渲染按钮） */
  actions(): Promise<ActionDescriptor[]>
}
```

### 4.3 与运行时形状的**故意差异**

类型描述的是"给作者的友好视图"，与宿主注入的原始形状有几处刻意的不同：

| 运行时形状                 | 类型（本库）               | 为什么                                     |
| -------------------------- | -------------------------- | ------------------------------------------ |
| `env` 是一组属性            | `env` 是**方法**            | 宿主侧是 IPC 代理，方法能明确"这是一次调用"    |
| 只有 `event.subscribe`      | 额外提供 `device.onChange`  | 见 §4.4                                     |
| `fetch` 返回原生 `Response` | 返回 `SynraResponse`        | 见 §4.6（**与宿主同一类型**）                 |
| `self.invoke` 返回 `unknown`| 泛型 `<Args, Result>`       | 作者按 `ActionDescriptor` 断言                |

### 4.4 便捷方法与判定函数：为什么由**宿主**提供

`device.onChange` / `device.pick` / `onProgress` / `isSynraError` 都需要**运行期逻辑**。
本包零运行时，所以有三个选择：

| 选项                                    | 优点                          | 缺点                            |
| --------------------------------------- | ----------------------------- | ------------------------------- |
| **A. 声明为宿主必须提供的方法** ✅         | 类型与运行时一致；本包零运行时   | 宿主注入脚本要实现它们（各几行）    |
| B. 声明为可选（`onChange?:`），宿主按需提供 | 灵活                          | 作者要判空；能力不一致             |
| C. 移到另一个带实现的包                    | 本包纯粹                       | 又一个包（但 `testkit` 确实需要，见 [25](./25-js-monorepo.md)） |

**选 A**：运行时只有宿主一个提供者，让它实现最简单（它本来就有 `event.subscribe`，包一层即可）。
**这条决定了本包能始终保持零运行时。**

### 4.5 渐进增强：`ClientFeature` 与 `env.hostSupports`

`device.pick` 这类"预留但可能未实现"的能力，需要一个**能力自述**机制，
否则作者只能靠"调用失败再看错误"来探测（体验差，且日志里全是预期内的失败）。

```ts
export type ClientFeature =
  | 'device.onChange'
  | 'device.pick'
  | 'transfer.onProgress'
  | 'event.subscribe'
  | 'network.fetch'
  | 'clipboard.read'

// 用法：渐进增强
const devices = await synra.device.list()
const canPick = synra.device.supported.includes('device.pick')

async function chooseTarget(): Promise<DeviceId | null> {
  if (canPick) return synra.device.pick({ title: '在哪台设备执行？' })
  return null // 展示插件自己的显式选择控件；不能自动改用第一台
}
```

**规则**：

| # | 规则                                                                 |
| - | -------------------------------------------------------------------- |
| 1 | `supported` 是**同步快照**（跟 `capabilities` 一样），渲染时可判断         |
| 2 | 未实现的方法**必须存在**且**必须明确拒绝**（`code: 'unsupported'`），不能是 `undefined` |
| 3 | `supported` 里没有的项，作者不应调用；调了也必须得到 `unsupported` 而不是崩溃 |

> **为什么"方法必须存在"**：如果宿主返回的对象缺少某个方法，作者写 `synra.device.pick(...)`
> 会得到 `TypeError: not a function`——一个**不可诊断**的错误。
> 保留方法 + 明确 `unsupported` 让"这个宿主版本不支持"变成一个可处理的正常情况。

### 4.6 `network.fetch` 的形状（拍板：自定义，但与宿主**同一类型**）

**决定**：不用原生 `Response`，用自定义的 `SynraResponse`。原因：经 IPC 拿不到真正的
`Response` 实例（它绑定在 fetch 的流式语义上），硬造一个会让人误以为能 `body.getReader()`。

```ts
// @synra/client/src/http.ts —— ★ 这个文件是"两侧共同的形状契约"
export interface SynraFetchInit {
  method?: 'GET' | 'POST' | 'PUT' | 'PATCH' | 'DELETE' | 'HEAD'
  headers?: Record<string, string>
  /** 请求体：字符串或二进制 */
  body?: string | Uint8Array
  /** 请求超时（内核会强制；超出直接 reject `timeout`） */
  timeoutMs?: number
}

export interface SynraResponse {
  readonly status: number
  readonly statusText: string
  /** 头字段名统一小写（与 HTTP/2、`fetch` 一致） */
  readonly headers: Readonly<Record<string, string>>
  readonly url: string
  readonly ok: boolean                 // 2xx
  readonly redirected: boolean

  // ★ 三种读取方式互斥（读完一次就没了），与 fetch 语义一致但**明确**
  text(): Promise<string>
  json(): Promise<unknown>
  bytes(): Promise<Uint8Array>
}

/** 出网相关的错误码（属于 SynraErrorCode，见 §5） */
export type NetworkErrorCode =
  | 'scheme_not_allowed'
  | 'host_not_allowed'
  | 'ip_not_allowed'
  | 'redirect_blocked'
  | 'response_too_large'
```

**两侧一致性是硬要求**（这正是"不能有字段不统一"的落地）：

```ts
// ★ 宿主的注入脚本必须用同一个类型来实现 fetch
//   synra-app 的注入脚本：
import type { SynraFetchInit, SynraResponse } from '@synra/client'

async function fetchImpl(input: string, init?: SynraFetchInit): Promise<SynraResponse> {
  // …经 IPC 掉内核，返回**恰好**这个形状
  return { status, statusText, headers, url, ok, redirected, text, json, bytes }
}
```

| 保证机制                                                          | 在哪 |
| ----------------------------------------------------------------- | ---- |
| 类型定义只有一处（`@synra/client/src/http.ts`）                     | 本文 §4.6 |
| 宿主注入脚本 `import type` 同一个类型                               | [08 §7.2](./08-ipc-and-frontend.md) |
| 集成测试断言"注入对象返回的形状与类型字段**逐一匹配**"                 | [21 §7](./21-testing-strategy.md) |
| 字段名漂移检测（用 `tsd`/`expect-type` 做形状断言，编译期失败）        | [25 §6](./25-js-monorepo.md) |
| **没有第二个地方声明 `SynraResponse`**                             | CI 守卫（grep） |

> ⚠️ **不要在两个仓库里各写一份同名接口**。这就是 [11 D4](./11-design-principles.md)
> （契约只有一个来源）在 JS 侧的落地：**类型在包，实现按类型**。

---

## 5. 错误模型（零运行时）

因为不能用 `class`（那会产生 `.js`），错误用**判别式结构**表达。

### 5.1 类型

```ts
// errors.ts —— 全部是类型

export interface SynraError {
  /** ★ 判别字段：所有 Synra 错误都有它 */
  readonly kind: 'synra-error'
  readonly code: SynraErrorCode
  readonly message: string
  readonly retryable: boolean
  readonly actions: readonly SuggestedAction[]
  readonly traceId?: string
  readonly details?: unknown
}

/** 与 [04 §7.6](./04-wire-protocol.md) 对齐；由 @synra/protocol 生成 */
export type SynraErrorCode =
  | 'capability_denied'
  | 'plugin_not_installed'
  | 'device_offline'
  | 'not_paired'
  | 'transport_not_open'
  | 'transfer_failed'
  | 'integrity_mismatch'
  | 'payload_too_large'
  | 'timeout'
  | 'rate_limited'
  | 'unsupported'
  | 'invalid_argument'
  | NetworkErrorCode                  // scheme/host/ip/redirect/too-large
  | 'internal'

/** 用户可执行的建议——UI 据此渲染按钮 */
export type SuggestedAction =
  | { kind: 'open_settings'; section: string }
  | { kind: 'request_capability'; capability: string }
  | { kind: 'install_plugin'; pluginId: string }
  | { kind: 'connect_device'; deviceId: DeviceId }
  | { kind: 'retry' }
  | { kind: 'view_logs'; traceId: string }
```

### 5.2 两种判定方式

**① 结构判别（纯类型，零运行时）**——主路径：

```ts
try {
  await synra.platform.openUrl(url)
} catch (e) {
  // ★ 判别字段收窄：不需要任何 import、不需要运行时函数
  if (typeof e === 'object' && e !== null && (e as SynraError).kind === 'synra-error') {
    const err = e as SynraError
    if (err.code === 'capability_denied') return { state: 'need_permission', capability: err.message }
    if (err.retryable) return { state: 'retry_later', actions: err.actions }
  }
  throw e
}
```

**② 宿主注入的便捷判定**（类型在本包声明，实现由宿主提供）：

```ts
if (synra.isCapabilityDenied(e)) { /* ... */ }
if (synra.isSynraError(e) && e.retryable) { /* ... */ }
```

> **为什么两种都要有**：① 保证**零运行时**，且在 Node 单测里也能判；
> ② 给作者一行写法。两者形状一致，不冲突。
> 这也是 [11 D1](./11-design-principles.md)（不靠字符串判定）的落地：**判别靠 `kind` + `code`**。

---

## 6. 可用性判断（不给运行时 helper）

`isAvailable()` / `assertAvailable()` 需要函数体，所以本包不提供。推荐写法：

```ts
// 判断当前是否在插件上下文
if (typeof window.synra !== 'undefined') { /* 在插件里 */ }

// 需要"不在就抛可读错误" → 作者自己写在自己的 ui/src/lib/ 里：
export function requireSynra(): SynraClient {
  const s = window.synra
  if (!s) throw new Error('此页面必须在 Synra 插件上下文中运行（window.synra 不存在）')
  return s
}
```

**理由**：这类 helper 是**项目级**的（不同插件想要不同的错误文案与降级策略），
放进类型库只会让包膨胀，且仍然需要运行时。

> 想在开发期拿到"完整的假 `synra`"（带实现）→ 用 `@synra/testkit`（[25 §2.4](./25-js-monorepo.md)）。

---

## 7. 三个"能不能"的语义（最容易混，必须在类型注释里写清）

```ts
// ① 我（插件）被授予了这个能力吗？→ 决定"我要不要走这条路"
if (synra.env.can('platform:open-url')) { /* 可以尝试 */ }

// ② 宿主平台本身支持这个能力吗？→ 决定"按钮要不要禁用"
if (synra.env.platformSupports('simulate_keys')) { /* 显示按钮 */ }
else { /* 显示"当前平台不支持" */ }

// ③ 同步读快照——渲染时的条件分支（不用 await）
const caps = synra.capabilities
const feats = synra.device.supported
```

| 判断                        | 数据来源     | 同步？ | 用途                |
| --------------------------- | ------------ | ------ | ------------------- |
| `synra.capabilities`         | 注入时快照    | ✅      | 渲染时的条件分支      |
| `synra.env.can(cap)`         | 授权目录      | ✅      | 与授权一致的检查点    |
| `synra.env.platformSupports` | 宿主平台能力  | ✅      | 平台差异导致的禁用态  |
| `synra.device.supported`     | 宿主实现能力  | ✅      | 渐进增强（§4.5）     |

> **为什么都是同步的**：如果异步，每个组件都要 `await`，渲染逻辑会很难写。
> 宿主在注入时就知道这些信息，直接同步给它。

---

## 8. 作者体验：完整示例

```vue
<!-- ui/src/pages/Home.vue -->
<script setup lang="ts">
import { onMounted, onUnmounted, ref } from 'vue'
// 只 import 类型；`synra` 本身是全局的，不需要 import
import type { DeviceTarget, Unsubscribe } from '@synra/client'
// ★ `DeviceId` 不在 `@synra/client` 里 —— 它是线协议的类型，从 `@synra/protocol` 来。
//   本文原来的示例把它写进了 `@synra/client` 的 import，照抄的作者会拿到
//   "no exported member 'DeviceId'"。改这一行的是编译器，不是校对：
//   `synra-js/scripts/check-doc-snippets.ts` 会把这个完整示例抽出来 `tsc --noEmit`。
import type { DeviceId } from '@synra/protocol'

const devices = ref<DeviceTarget[]>([])
const target = ref<DeviceId | null>(null)
const error = ref<string | null>(null)
let off: Unsubscribe | undefined

onMounted(async () => {
  synra.ready()                                    // 告诉宿主"页面已就绪"
  devices.value = await synra.device.list()
  off = synra.device.onChange(d => { devices.value = d })
})

onUnmounted(() => off?.())

async function pickTarget() {
  // 宿主支持时使用原生选择器，否则由下面的 select 控件明确选择。
  if (synra.device.supported.includes('device.pick'))
    target.value = await synra.device.pick({ title: '在哪台设备执行？' })
}

async function openOnLaptop(url: string) {
  if (!target.value) return
  try {
    await synra.self.invoke('open-url', { url, to: target.value })
  } catch (e) {
    error.value = synra.isCapabilityDenied(e) ? '缺少权限：platform:open-url' : String(e)
  }
}
</script>
<template>
  <label for="target">在哪台设备执行</label>
  <select id="target" v-model="target">
    <option :value="null">请选择目标设备</option>
    <option v-for="device in devices" :key="device.deviceId" :value="device.deviceId">
      {{ device.displayName }} · {{ device.online ? '在线' : '离线' }} · {{ device.deviceId }}
    </option>
  </select>
</template>
```

模板里的 `ui/tsconfig.json`（由 `synra-plugin new` 生成）：

```jsonc
{
  "compilerOptions": {
    "target": "ES2022",
    "module": "ESNext",
    "moduleResolution": "bundler",
    "strict": true,
    "types": ["@synra/client"]          // ★ 一行：全局 synra 与 window.synra 都有类型
  },
  "include": ["src", "vite.config.ts"]
}
```

---

## 9. 待确认（🔴）

| # | 问题                                                      | 状态 / 倾向                                                     |
| - | --------------------------------------------------------- | -------------------------------------------------------------- |
| 1 | `network.fetch` 返回自定义形状                              | ✅ **已拍板**：自定义；两侧**同一类型**（§4.6）                     |
| 2 | `device.pick`（宿主选设备 UI）                              | ✅ **已拍板**：做，预留；未实现时返回 `unsupported`（§4.2/§4.5）      |
| 3 | `ClientFeature` 的初始清单（§4.5 那 6 项是否够）              | 🟡 先这 6 项，按需加（只增不改）                                     |
| 4 | `pick` 是弹宿主模态，还是"就地展开一个设备列表"               | 🔴 未定——影响沉浸度；倾向**宿主模态**（可复用、视觉一致）              |
| 5 | `synra` 汇总包的实际形态（consignment vs 独立）               | 🟡 consignment + `@synra/client` 传递依赖（[25 §3](./25-js-monorepo.md)） |

---

## 10. 实现清单

`@synra/client`（`synra-js` 仓库）：

- [x] `surface.ts`：`SynraClient` 与 10 个 namespace 的接口
      （错误类型与 `device` 也在这里，本文另写的 `errors.ts` / `device.ts` 不存在）
- [x] `global.d.ts`：`declare global { const synra; interface Window { synra? }; type Unsubscribe }`
      —— 实际文件是 `src/global.ts`，内容覆盖了这条要求
- [x] `http.ts`：`SynraFetchInit` / `SynraResponse`（★ 两侧共用）
- [x] `errors.ts`：`SynraError` / `SynraErrorCode` / `SuggestedAction`（纯类型）—— **在 `surface.ts` 里**
- [x] `device.ts` 里 `pick` + `supported: ClientFeature[]` —— **在 `surface.ts` 里**
- [x] `package.json`：**没有 `main`/`module`**；`exports` 只给 `types`；`files` 是 `["dist"]` 而非 `["*.d.ts"]`，
      但 `scripts/check-shapes.ts` 扫 `dist/` 里有没有 `.js`/`.mjs`/`.cjs`，后一道闸门在
- [x] 构建：`tsc --emitDeclarationOnly` —— 以 `tsconfig.json` 的 `emitDeclarationOnly: true` + `build: tsc -b` 实现，等价
- [x] **CI 守卫**：`dist/` 下不存在 `.js` / `.mjs` / `.cjs` —— 在 `synra-js/scripts/check-shapes.ts`
- [x] 形状断言测试（`tsd` 或 `expect-type`）：`SynraResponse` 字段与宿主实现逐一对齐
      ★ **换成了更强的做法，而不是省略**：实现走 `satisfies SynraClient`，所以形状是在**唯一那份实现**上被
      强制的，而不是在另一个断言文件里再写一遍。（`network.fetch` 本身仍未实现 —— 见下面的说明。）
- [x] **守卫 8：文档里的类型片段能编译**（`synra-js/scripts/check-doc-snippets.ts`）
      抽 §8 的**完整示例**到临时工程跑 `tsc --noEmit`（匹配已构建的声明、`types: ["@synra/client"]`、
      `vue` 用一个只声明三个名字的 shim）。
      只抽**完整示例**而不是每个代码块：其余大多是片段（两行调用、接口草图、故意写错的拼写），
      而片段不是程序 —— 一个需要猜作者意图的守卫，报的是它自己的猜测。
      设计文档仓库不在旁边时**跳过并说明**（同 `check-shapes.ts` 的跨仓库比对）。
      负控制：把示例里的 `isCapabilityDenied` 换成一个不存在的名字 → 守卫红并点名行号。
      ★ 它第一次运行就抓到了三个真错，见上面那条。
      ★ **被上游卡住，不是缺测试工具**：`injection.ts` 的 `BACKED` 表里 `'network.fetch': false`，
      宿主根本没有 `fetch` 实现，所以没有可对齐的形状。
      等宿主有了 `fetch`，`satisfies SynraClient` 会自动强制这件事 —— 那比另写一个 `tsd` 测试更强。

`synra-app` 侧（保证两侧字段一致）：

- [x] 注入脚本 `import type { SynraClient } from '@synra/client'`，
      并用 `satisfies SynraClient`（或显式标注）实现——**漏字段/多字段在编译期就报错**
      （`src/ipc/injection.ts:442`，并且注释写明了为什么是 `satisfies` 而不是类型标注）
- [x] `device.pick` 未实现时返回 `SynraError{ code: 'unsupported' }`，且不出现在 `supported` 里
      —— `supported` 与行为来自同一张 `BACKED` 表
- [x] `isSynraError` / `isCapabilityDenied` 由宿主提供
      ★ **这一条本来标错了，而且是我自己批错的。** 它此前被标成已完成，实际上：`SynraClient` 只声明了
      `isSynraError` 与 `isCode`，宿主也只实现了这两个。`isCapabilityDenied`（本文 §4.1 列了它，
      §5.2 与 §8 的示例都在用）和 `ready()`（§4.1）**两边都没有**。
      我当时看到 `injection.ts` 里一句提到 `isCode` 的注释就接受了它，而没有去数它到底声明了哪几个 ——
      正是我这几轮在别人代码里反复找到的那种错。
      ★ **拉出它的是编译器**：本轮新增的守卫 8 把 §8 的完整示例抽出来跑 `tsc --noEmit`，
      第一次运行就报了三个错：错的 `DeviceId` import、`isCapabilityDenied` 不存在、`ready` 不存在。
      修法：`surface.ts` 加上两个成员，`injection.ts` 实现它们（`ready()` 调契约里已有的
      `plugin_ui_ready`），`testkit` 的 `MockSynraClient` 也补上 —— 后者是编译器自己列出来的。
- [x] 测试：注入对象暴露的键集合 ⊆ `SynraClient` 的键集合
      —— 实际断言的是**精确相等**，比这条更严
- [x] 测试：`types: ["@synra/client"]` 的 fixture 工程里 `synra.device.send` / `synra.device.pick` 类型正确
      —— `packages/client/type-tests/consumer.ts`，**不含任何 import**，所以它只会因为
      `types` 那一行失效而失败

文档 / 模板侧：

- [x] [13 §5.2](./13-plugin-authoring.md) 指向本文与 [25](./25-js-monorepo.md)
- [ ] `synra-plugin new` 生成的 `ui/tsconfig.json` 默认含 `"types": ["@synra/client"]`
      ★ **没做，且现在不该单独做**：模板的界面是内联脚本的 `ui/index.html`，没有 `.ts` 入口；
      一个没有输入的 `tsconfig.json` 会让 `tsc` 报 "No inputs were found"。
      真正的做法是把 UI 模板改成 TS 工程，见本节开头
- [ ] `ui/package.json` 的 `devDependencies` 含 `@synra/client`（+ 可选 `@synra/testkit`）
      ★ **没做，阻塞在发布上**：`@synra/client` 还是 workspace 包、尚未发布，
      写进 `devDependencies` 会让生成的工程 `install` 失败 —— 那比没有这条更糟
