# 25 · JS 包 monorepo（`synra-js`）

> 本文定义**面向插件的一部分 JavaScript/TypeScript 包**的仓库形态：
> 谁在哪个包、怎么发布、`synra` 为什么"一般不用安装"、以及生成类型与形状一致性如何保证。
>
> 前置：[23 `@synra/client`](./23-client-library.md)（类型库契约）。
> 相关：[README §2](./README.md)（仓库拓扑）、[ADR-016](./20-adr.md)（为什么拆仓库）。

---

## 1. 为什么单独一个仓库

插件作者需要的东西全是 TS 声明与少量开发期工具，它们：

| 特征                              | 后果                                              |
| --------------------------------- | ------------------------------------------------- |
| 独立于 Tauri 应用发布节奏           | 修一个类型不该触发一次桌面安装包构建                   |
| 被**第三方**依赖（插件作者）         | 需要一个干净、可单独测试、可独立版本化的仓库            |
| 有自己的一套工具链（pnpm + tsc + vitest） | 放进 app 仓库会让 app 的 CI 变成两个项目拼在一起 |

所以：**第三个仓库**。

```
D:/Projects/
├─ synra-core/        # 内核（Cargo workspace）
├─ synra-app/         # Tauri 应用（桌面 + 移动 + 宿主 UI）
└─ synra-js/           # ★ 本文：TS 包 monorepo（插件作者用的东西）
```

> 这与 [ADR-016](./20-adr.md) 不冲突：那里论证的是"**内核**与**宿主**必须分开"；
> 这里把"**给第三方的 TS 契约与工具**"再单独拎出来，因为它有完全不同的消费者与发布节奏。

---

## 2. 包含哪些包

```
synra-js/
├─ package.json                 # 私有根包（不发布）
├─ pnpm-workspace.yaml          # packages/*
├─ pnpm-lock.yaml
├─ packages/
│  ├─ synra/                    # ★ 汇总入口（consignment，见 §3）
│  ├─ client/                   # @synra/client —— 类型库（.d.ts only）
│  ├─ protocol/                 # @synra/protocol —— 线协议类型（Rust 生成）
│  └─ testkit/                  # @synra/testkit —— 插件作者的本地 mock 运行时
├─ scripts/
│  ├─ gen-protocol-types.ts     # 从 core 仓库拉生成物 → packages/protocol
│  └─ check-shapes.ts           # 形状一致性守卫（§6）
└─ .github/workflows/           # 只跑 Node 相关 CI
```

### 2.1 包一览

| 包                  | 有运行时？ | 谁依赖它                     | 作用                                                       |
| ------------------- | ---------- | ---------------------------- | ---------------------------------------------------------- |
| **`synra`**          | ❌          | 传递依赖（作者一般不用装）      | 汇总对外类型面：re-export `@synra/client` + `@synra/protocol` |
| **`@synra/client`**  | ❌          | 插件作者的 `devDependencies`   | 注入对象契约 + `declare global` + 错误与 HTTP 形状            |
| **`@synra/protocol`**| ❌          | `@synra/client`、宿主、其它工具 | 线协议数据类型（**从 Rust 生成**）                            |
| **`@synra/testkit`** | ✅          | 插件作者的 `devDependencies`   | `createMockClient()`：开发期/单测用的假 `synra`               |

**"有运行时？"这一列是关键**：前三个是纯类型（零运行时），只有 `@synra/testkit` 有实现。
CI 对前三个有"不允许产出 `.js`"的守卫（§6）。

> ★ **本次核对发现：这一列说的没错，而代码违反过它 —— 已经修好。**
>
> 守卫原先只查两件事：`package.json` 里没有运行时入口、`dist/` 里没有 `.js`。
> 两件都通过，**而 `packages/protocol/src/index.ts` 照样从入口导出了 13 个值**：
> `isDelta`、`MAX_COUNTED_GAP`、`SequenceTracker`、`FrameKind`、`FRAME_HEADER_LEN`、
> `MAX_FRAME_LEN`、`MAX_CHUNK_LEN`、`MAX_PAYLOAD_LEN`、`FRAME_LENGTH_OVERHEAD`、
> `DEFAULT_TCP_PORT`、`DISCOVERY_PORT`、`MDNS_SERVICE`、`PROTOCOL_VERSION`。
>
> 后果不是文档不整齐，而是**一个作者照着类型写 `import { SequenceTracker } from '@synra/protocol'`
> 会类型检查通过、运行时解析不到** —— `undefined` 或者一个打包错误，没有任何地方提前提醒。
> "零运行时"这句话在类型层面是假的。
>
> ★ **而这个包自己的测试早就知道。** `packages/protocol/test/events.test.ts` 的注释写着：
> "`@synra/protocol` 是一个**纯类型**包，所以它按设计没有运行时入口，按名字 import 会解析失败"
> —— 然后它从源码路径导入。**测试知道这些名字拿不到，入口却照样把它们许诺出去。**
>
> ★ **修复分两步，因为第一版只做对了一半：**
> 1. **守卫改成检查公开面**，而不是整个 `dist`：消费方只能通过 `exports` 指的那个文件到达这个包，
>    所以看的是 `dist/index.d.ts` 里的 `export { ... }`（不带 `type`）。
>    仓库开了 `verbatimModuleSyntax`，所以"类型还是值"由编译器强制区分，不是靠自觉。
>    按整个 `dist` 查会把"内部模块有个值、入口只带出类型"这种无害情况也判成失败，逼人去加白名单。
> 2. **`protocol` 不再导出值**，`SequenceTracker` / `isDelta` / `MAX_COUNTED_GAP` 搬到
>    `synra-rust-app/src/ipc/sequence.ts`，连同它们的测试。
>    理由是这一列本身：`protocol` 是纯类型，而**宿主**是真有运行时的应用程序。
>
> ★ **第一步只改 `protocol` 时，类型检查立刻抓到了一个我没找到的消费者**：
> 应用的 `src/ipc/dispatcher.ts:30` 正是 `import { isDelta, SequenceTracker } from '@synra/protocol'`。
> 我先前的排查只搜了常量名（`MAX_FRAME_LEN` 等），**没有搜 `isDelta` / `SequenceTracker`**，
> 于是得出了"没有任何东西导入它们"的错误结论。**是 `vue-tsc` 纠正了我，不是我纠正了自己** ——
> 这也说明把这件事交给编译器而不是靠 grep 是对的。
> （`dispatcher.ts` 的 `EventDispatcher` 至今**没有生产调用者**，所以这个坏导入从来没在运行时暴露过。）
>
> ★ 顺带更正 `SequenceTracker` 的注释：它写着"只持有最后一个序号，**不含运行时**"，
> 而这句话出现在一个**类的定义**上方。所有者换了之后那句话才成立。

### 2.2 `@synra/protocol`：生成物，不手写

```
synra-core/crates/synra-protocol/**   （Rust 类型 = 唯一真相源）
        │  cargo test -p synra-protocol --features ts-export   （ts-rs 导出）
        ▼
生成到 core 仓库的 target/ts-export/*.ts
        │  pnpm --filter @synra/protocol sync   （scripts/gen-protocol-types.ts 复制并加头注释）
        ▼
synra-js/packages/protocol/src/generated/*.ts   （★ commit 进仓库）
```

**为什么 commit 生成物而不是发布时生成**：

| 理由                                  | 说明                                              |
| ------------------------------------- | ------------------------------------------------- |
| 插件作者的 **UI 代码**只用发布包（不写 Rust，不编译 Rust） | 发布包必须自带完整类型，且 `types` 一行生效 |
| 让"类型变了"在 review 里可见              | 生成物进 diff，评审能看到线上契约的变化                 |
| 避免发布流程依赖两个仓库的 CI 同时可用       | 发布是纯 Node 侧动作                                |

**必须有的守卫**（[§6](#6-ci-守卫)）：JS 仓库里的生成物与 core 当前导出**必须一致**，
否则 JS 仓库的 CI 失败（core 改了协议但没同步过来）。

### 2.3 文件头（生成物必须标注，防止被手改）

```ts
/* eslint-disable */
/**
 * ⚠️ 自动生成，请勿手动修改。
 *
 * 来源：synra-core/crates/synra-protocol
 * 生成：cargo test -p synra-protocol --features ts-export
 * 同步：pnpm --filter @synra/protocol sync
 */
```

CI 守卫：**任何生成文件被手改 → 重新同步后 `git diff` 不为空 → 失败**。

### 2.4 `@synra/testkit`：唯一带实现的那个

插件作者在本地要能"没有宿主也跑起来"：

```ts
import { createMockClient } from '@synra/testkit'

const synra = createMockClient({
  capabilities: ['device:list', 'device:send'],
  devices: [{ deviceId: 'dev_a', displayName: '我的 PC', online: true }],
  platformCapabilities: ['open_url', 'notify'],       // 模拟 iOS（无 simulate_keys 等）
  hostFeatures: ['device.onChange'],                  // 模拟"宿主未实现 device.pick"
})

// 调用被记录，供断言
await synra.device.send({ to: 'dev_a', event: 'chat.text', payload: { body: 'hi' } })
expect(synra.calls).toContainEqual({ ns: 'device', method: 'send', args: { /* … */ } })

// 让某次调用失败
synra.stub('platform', 'openUrl', { kind: 'synra-error', code: 'capability_denied', /* … */ })
```

`MockSynraClient extends SynraClient`（类型来自 `@synra/client`），所以
**mock 与真实注入对象是同一类型**——作者写测试时不会"测过了但真机不对"。

> `testkit` 是 devDependency，**永远不会进入插件产物**（作者只用它跑测试）。
> 它也是"零运行时"原则的例外：**它不是注入对象的实现，而是它的替身**，两者不冲突。

---

## 3. `synra` 汇总包：为什么"一般不用安装"

### 3.1 目标

作者想写 `import type { SynraError } from 'synra'` 这种**不带 scope 的短路径**，
又不想在 `package.json` 里多装一个包。

### 3.2 机制（consignment + 传递依赖）

```
packages/synra/
├─ package.json         # name: "synra"，private: false，"version": <与 client 同步>
├─ src/index.ts         # 只有 re-export type，没有任何运行时值
└─ dist/index.d.ts      # 生成物（只有 .d.ts）
```

```ts
// packages/synra/src/index.ts —— 全部是 type-only re-export
export type * from '@synra/client'
export type * from '@synra/protocol'

// ★ 显式重导出可能重名的/关键的几个，给 IDE 更好的提示
export type { SynraClient, SynraError, SynraErrorCode, SynraResponse } from '@synra/client'
export type { DeviceId, TransferId, DeviceTarget } from '@synra/protocol'
```

```jsonc
// packages/client/package.json 的 dependencies（见 [23 §2.1](./23-client-library.md)）
{
  "dependencies": {
    "@synra/protocol": "workspace:^",
    "synra": "workspace:^"          // ★ 让 synra 成为传递依赖
  }
}
```

于是：

```
作者装 @synra/client
   └─ node_modules/@synra/client
   └─ node_modules/@synra/protocol      （传递）
   └─ node_modules/synra                （传递）★ 于是 'synra' 可解析
```

| 事实                                  | 结果                                          |
| ------------------------------------- | --------------------------------------------- |
| 作者 `package.json` 里只有 `@synra/client` | `import type … from 'synra'` 仍然**可解析**       |
| `synra` 是传递依赖                      | 版本由 `@synra/client` 决定，作者不会装错版本     |
| `synra` 零运行时                        | 它不会出现在产物里（作者只 `import type`）         |
| 想显式安装也可以                         | `devDependencies: { "synra": "^1.0.0" }`        |

> ⚠️ **这是有意的"隐式可用"**：代价是"依赖关系不那么显式"。
> 收益是作者少一个包、少一个版本要对齐。为了让行为可预测，**三个包版本必须同步发布**（§4）。

### 3.3 禁止的事

| ❌ 不允许                                      | 为什么                                            |
| --------------------------------------------- | ------------------------------------------------- |
| 在 `synra` 里放**运行时实现**                    | 会破坏"零运行时"，且与注入对象形成两份实现            |
| 在 `synra` 里定义**新的类型**                    | 会造成"同一类型两处定义"，[11 D4](./11-design-principles.md) |
| 让 `synra` 成为**唯一**入口（禁止直接用 `@synra/client`） | tsconfig 的 `types` 需要包名稳定，且 `testkit` 依赖 client |
| 让 `synra` 出现在**插件的 dependencies**（运行时依赖） | 它已经零运行时，进 dependencies 只会让人以为它有用   |

---

## 4. 版本与发布

### 4.1 三个包**同步版本**

`synra` / `@synra/client` / `@synra/protocol` 采用**同一个版本号**（`fixed` 模式），
由 `bumpp -r` 一次 bump 全部（与本机既有 monorepo 的约定一致）。

理由：它们互相 re-export，版本错位会产生"类型来自不同代次"的诡异错误，
而这种错误极难诊断。

```jsonc
// 根 package.json
{
  "name": "synra-js",
  "private": true,
  "packageManager": "pnpm@11.x",
  "scripts": {
    "build": "pnpm -r build",
    "test": "pnpm -r test",
    "check": "tsc -b && pnpm run shapes",
    "shapes": "tsx scripts/check-shapes.ts",
    "sync": "pnpm --filter @synra/protocol sync",
    "release": "bumpp -r"
  }
}
```

`@synra/testkit` **也跟同一版本**（它是 client 的替身，语义上必须对齐）。

### 4.2 发布

| 包                  | `publishConfig.access` | 发布顺序（依赖优先）                |
| ------------------- | ---------------------- | ---------------------------------- |
| `@synra/protocol`    | `public`               | 1                                  |
| `@synra/client`      | `public`               | 2                                  |
| `synra`              | `public`               | 3                                  |
| `@synra/testkit`     | `public`               | 4                                  |

发布**不允许**跳过任何一步（发布脚本按拓扑排序执行，失败即中止；重跑可续）。

---

## 5. 与两个 Rust 仓库的关系

```
synra-core ──────────► synra-js ──────────► 插件作者
   （协议真相源）              （生成 + 手写契约）         （tsconfig / devDeps）
        │                          ▲
        │                          │ 注入脚本按同一类型实现
        ▼                          │
synra-app ────────────────────┘
   （宿主：注入 window.synra）
```

| 方向                          | 机制                                                        |
| ----------------------------- | ----------------------------------------------------------- |
| core → client                 | `ts-rs` 生成 → `pnpm sync` → commit（§2.2）                    |
| client → app                  | app 依赖 `@synra/client`（**dev** 依赖即可，类型在编译期用完）  |
| app 不能写自己的 `SynraClient`   | CI 守卫：禁止 app 里出现 `interface SynraResponse` 之类的重复声明 |

> **app 用 `devDependencies` 还是 `dependencies`**：因为 `@synra/client` 零运行时，
> 它**不需要**进 app 的运行期依赖（不会被打包进产物）。用 `devDependencies` 即可。

---

## 6. CI 守卫

`synra-js` 仓库的 CI（Node 侧）：

| # | 守卫                                                  | 怎么实现                                                |
| - | ----------------------------------------------------- | ------------------------------------------------------- |
| 1 | **类型包零运行时**                                      | `dist/` 下出现 `.js`/`.mjs`/`.cjs` 即失败                  |
| 2 | `synra` 与 `@synra/client` **没有值导出**                | 检查 `dist/*.d.ts` 里没有 `export declare const/function/class`（只允许 `export type` / `export interface`） |
| 3 | **形状一致性**                                          | `scripts/check-shapes.ts`：断言 `SynraResponse` / `SynraError` 的字段集合与快照一致（字段增删必须显式更新快照，进 review diff） |
| 4 | 生成物未被手改                                          | `pnpm sync && git diff --exit-code packages/protocol`      |
| 5 | 生成物与 core 一致                                      | 从 core 仓库取当前导出，比对（跨仓库 job 或 nightly）           |
| 6 | 三个包版本一致                                          | 读三个 `package.json` 的 `version` 比较                       |
| 7 | `types: ["@synra/client"]` 可用                         | 一个 fixture 工程，`tsc --noEmit` 必须通过，且断言 `synra.device.send` / `synra.device.pick` 的类型 |
| 8 | 文档里的类型片段能编译                                    | 把 [23](./23-client-library.md) 里的代码块抽出来做类型检查（防文档漂移） |

`synra-app` 仓库的对应守卫：

| # | 守卫                                                       |
| - | ---------------------------------------------------------- |
| 1 | 注入脚本 `satisfies SynraClient`（漏字段/多字段 → 编译失败）   |
| 2 | 注入对象暴露的键集合 ⊆ `SynraClient` 的键集合                 |
| 3 | 未实现的能力必须在**明确拒绝**（`unsupported`）并缺席 `supported` |
| 4 | 禁止在本仓库重复声明 `SynraResponse` / `SynraError`           |

---

## 7. 工具链与构建（unbuild）

### 7.1 选型

| 用途       | 选择                                                    | 依据 |
| ---------- | ------------------------------------------------------- | ---- |
| 包管理      | **pnpm workspaces**（`packages/*`）                      | [ADR-012](./20-adr.md)；与本机既有 monorepo 一致 |
| 构建       | **`unbuild`**（`build.config.ts` + `unbuild --stub`）      | 与本机既有项目（`initx` / `triggerix` / `ext-kit` 等）同一套约定 |
| 测试       | **Vitest**                                                | 同上 |
| 形状断言    | `tsd` / `expect-type`                                     | §6 守卫 3 |
| 版本与发布   | `bumpp -r` + 拓扑排序发布脚本                               | 本机既有 monorepo 的约定 |

**为什么用 unbuild**：

1. 它把**声明打包成自包含的 `.d.ts`** —— 对类型包尤其重要（§7.3）；
2. `unbuild --stub` 让 monorepo 内部联调**不需要重新构建**（指向 `src` 直接跑）；
3. 本机生态里已经有一致写法（`mergeDefaultBuildConfig` 共享配置），新仓库不必另发明一套。

> **不用 Vite+ 的 `vp` 统一 CLI**：本仓库没有前端应用（只有类型包与一个测试替身），
> 引入额外工具链没有收益（与 [ADR-012](./20-adr.md) 一致；ADR-012 约束的是 **app 仓库**）。

### 7.2 ⚠️ 实测事实：`unbuild` 一定会产出 `.js`

这是设计这个仓库时**必须知道的**。实测（unbuild 3.6.1）：

```
$ unbuild
i Building probe
Generated an empty chunk: "index"          ← ★ 类型包没有运行时值
√ Build succeeded for probe
  dist/index.mjs (total size: 1 B)          ← ★ 产出 1 字节的空壳 .js
  dist/index.d.ts
```

**结论**：`unbuild` **无法**只产出 `.d.ts`。所以"零运行时"必须靠
**`build:done` hook 显式剥离运行时文件**来实现，并且有 CI 守卫（§6 守卫 1）。

```ts
// packages/client/build.config.ts
import { defineBuildConfig } from 'unbuild'
import { rm } from 'node:fs/promises'
import { glob } from 'tinyglobby'

export default defineBuildConfig({
  // ★ 单入口 + 在 index.ts 里 `import './global'`，
  //   这样 declare global 会被内联进同一个 dist/index.d.ts（见 §7.3）
  entries: ['src/index'],

  declaration: true,
  clean: true,

  rollup: {
    // ★ 把 @synra/protocol 的类型也内联进来，产出**自包含**的 .d.ts
    dts: { respectExternal: false },
  },

  hooks: {
    // ★ 剥离 unbuild 必然产出的空壳 .js —— 这是"零运行时"的落地手段
    'build:done': async () => {
      const runtime = await glob(['dist/**/*.{js,mjs,cjs}'], { cwd: process.cwd() })
      await Promise.all(runtime.map(f => rm(f, { force: true })))
      if (runtime.length === 0) throw new Error('expected unbuild to emit runtime files; recipe changed?')
    },
  },
})
```

> **最后那个 `throw` 是有意的**：如果将来 unbuild 版本改了行为（不再产出 `.js`），
> 我们要知道，而不是静默地留着一个不再需要的 hook。

### 7.3 实测:三种 dts 产出的差别（决定了 `respectExternal` 与入口设计）

| 配置                            | 产出                                              | 评价 |
| ------------------------------- | ------------------------------------------------- | ---- |
| `entries: ['src/index']`，默认 `respectExternal: true` | 自包含（内部模块内联），但**外部包保持 `import`** | 作者需要能解析 `@synra/protocol` |
| **同上 + `respectExternal: false`** ✅ | **完全自包含**，`@synra/protocol` 的类型也内联进来 | ★ 采用：作者少一层依赖解析 |
| `entries: [index, global]`（多入口） | 同时产出 `index.d.ts` + `global.d.ts` + **`.d.mts` 各一份** | 多出重复文件；且 `declare global` 落在 `global.d.ts` |

**采用方案**：**单入口**，并在 `src/index.ts` 里用 **side-effect import** 把全局声明拉进来：

```ts
// packages/client/src/index.ts
import './global'                       // ★ 只为让 declare global 进入同一个 d.ts
export type * from './surface'
export type * from './namespaces/env'
// … 其余 namespace
export type { SynraError, SynraErrorCode, SuggestedAction } from './errors'
export type { SynraFetchInit, SynraResponse } from './http'
```

实测产出（正是我们想要的形态）：

```ts
// dist/index.d.ts  ← 单个自包含文件
interface SynraClient { /* … */ }

declare global {
  const synra: SynraClient
  interface Window { readonly synra?: SynraClient }
  type Unsubscribe = () => void
}

export type { SynraClient, SynraError, /* … */ }
```

于是 `types: ["@synra/client"]` **一行生效**——已用 `tsc` 端到端验证：全局 `synra`、
`window.synra`、`Unsubscribe` 均可用，且**类型是真实的**（把 `pluginId` 写成 `pluginID` 会报 `TS2551`，
证明不是 `any`）。

> **`.d.mts` 的处理**：单入口下 unbuild 会同时产出 `index.d.ts` 与 `index.d.mts`（内容相同）。
> 因为 `exports` 只给一个 `types` 路径，可以只保留 `.d.ts`。
> 若不想看到重复文件，在同一个 `build:done` hook 里一并删掉 `.d.mts`。

### 7.4 各包的 `build.config.ts`

```ts
// packages/protocol/build.config.ts —— 生成物，同样是纯类型
export default defineBuildConfig({ entries: ['src/index'], declaration: true, clean: true })

// packages/synra/build.config.ts —— 纯 re-export
export default defineBuildConfig({
  entries: ['src/index'],
  declaration: true,
  clean: true,
  rollup: { dts: { respectExternal: false } },   // 把 client + protocol 的类型都内联
})

// packages/testkit/build.config.ts —— ★ 唯一有运行时的包
export default defineBuildConfig({
  entries: ['src/index'],
  declaration: true,
  clean: true,
  rollup: {
    esbuild: { minify: true },
    inlineDependencies: false,   // 不要把 @synra/client 的（零运行时）类型包内联
    dts: { respectExternal: true },
  },
})
```

> **`testkit` 需要 `inlineDependencies: false`**：它依赖 `@synra/client`（零运行时），
> 内联只会产生空引用。参考本机 `bigseek-js` 的同类注释（"不内联依赖"的理由）。

### 7.5 完整 package.json 约定

#### 纯类型包（`synra` / `@synra/client` / `@synra/protocol`）

```jsonc
{
  "name": "@synra/client",
  "version": "1.0.0",                     // ★ 四个包同版本（§4.1）
  "type": "module",
  "description": "Types for the Synra plugin UI runtime. Declaration files only — no runtime code.",
  "license": "MIT",
  "publishConfig": { "access": "public" },

  // ── 入口：★ 没有 main / module ──
  "types": "./dist/index.d.ts",
  "exports": {
    ".": { "types": "./dist/index.d.ts" }
  },

  // ── 发布物 ──
  "files": ["dist"],

  // ── 脚本 ──
  "scripts": {
    "stub": "unbuild --stub",
    "build": "unbuild",
    "test": "vitest run"
  },

  // ── 依赖 ──
  "dependencies": {
    "@synra/protocol": "workspace:^",
    "synra": "workspace:^"                // ★ 让 synra 成为传递依赖（§3.2）
  },
  "devDependencies": {
    "unbuild": "^3.6.1",
    "tinyglobby": "^0.2.17",              // build.config 的 hook 里用
    "typescript": "^7.0.2",
    "vitest": "^5.0.3"
  }
}
```

**逐字段说明**：

| 字段                            | 怎么填                                          | 为什么                                                       |
| ------------------------------- | ----------------------------------------------- | ------------------------------------------------------------ |
| `type: "module"`                 | 固定                                            | 与本机所有项目一致；`build.config.ts` 也能被解析                  |
| `main` / `module`                | **不写**                                        | 没有运行时入口。写了会指向被剥离的文件 → `MODULE_NOT_FOUND`        |
| `types`                          | `./dist/index.d.ts`                             | 支撑 `types: ["pkg"]` 与 `import type` 两种用法                  |
| `exports["."].types`             | 同上                                            | 现代解析器（`moduleResolution: bundler`）走 `exports`             |
| `exports["."].default`           | 见下                                             | 见 §7.6                                                       |
| `files`                          | `["dist"]`                                      | 发布物只有 `dist/`（里面已被 hook 保证只有 `.d.ts`）                |
| `publishConfig.access`           | `"public"`                                      | scoped 包默认 private，必须显式声明                              |
| `dependencies`                   | `workspace:^`                                   | pnpm workspace 协议；发布时自动替换为具体版本号                    |
| `peerDependencies`               | 纯类型包**不需要**                                | 见下方说明                                                     |
| `devDependencies`                | 构建/测试工具 + 类型                                     | 不进消费者产物                                               |

> **纯类型包为什么不需要 `peerDependencies`**：peer 的语义是"消费者必须提供这个运行时"。
> 类型包没有运行时，它的依赖只有一个作用——**让 `.d.ts` 能被解析**。
> 而 `respectExternal: false` 已经把这些类型**内联**进 `dist/index.d.ts`，
> 所以严格来说 `@synra/protocol` 对 `@synra/client` 的消费者**连解析都不需要**。
>
> 那为什么还留 `dependencies`？两个理由：① 生成后仍可能保留少量 `import`；
> ② **类型包版本必须与它描述的协议版本对应**，这个约束用 `dependencies` 表达最清楚。
> 因此：**用 `dependencies`，不用 `peerDependencies`**。

#### `@synra/testkit`（唯一有运行时）

```jsonc
{
  "name": "@synra/testkit",
  "version": "1.0.0",
  "type": "module",
  "license": "MIT",
  "publishConfig": { "access": "public" },

  "main": "./dist/index.mjs",
  "module": "./dist/index.mjs",
  "types": "./dist/index.d.ts",
  "exports": {
    ".": {
      "types": "./dist/index.d.ts",
      "import": "./dist/index.mjs"
    }
  },
  "files": ["dist"],

  "scripts": {
    "stub": "unbuild --stub",
    "build": "unbuild",
    "test": "vitest run"
  },

  "dependencies": {
    "@synra/client": "workspace:^",
    "synra": "workspace:^"
  },
  "devDependencies": {
    "unbuild": "^3.6.1",
    "typescript": "^7.0.2",
    "vitest": "^5.0.3"
  }
}
```

> **`testkit` 是 devDependency**（作者只用它跑测试），所以它**可以**有运行时。
> 它的 `MockSynraClient extends SynraClient`（类型来自 `@synra/client`），
> 保证"测过了"与"真机行为"用同一套类型约束。

#### 插件作者的 `ui/package.json`（模板生成）

```jsonc
{
  "name": "synra-plugin-<slug>-ui",
  "private": true,
  "type": "module",
  "scripts": {
    "dev": "vite",
    "build": "vite build",
    "test": "vitest run",
    "typecheck": "vue-tsc --noEmit"
  },
  "dependencies": {
    "vue": "^3.5.40"
  },
  "devDependencies": {
    "@synra/client": "^1.0.0",       // ★ 只装这一个；synra 由传递依赖提供
    "@synra/testkit": "^1.0.0",      // 本地开发/单测的假 synra
    "vite": "^7",
    "vue-tsc": "^3"
  }
}
```

> 作者**不需要**装 `@synra/protocol` / `synra`——前者被 `@synra/client` 内联，
> 后者是传递依赖（[23 §2.4](./23-client-library.md)）。

### 7.6 `exports` 的条件设计（纯类型包）

```jsonc
"exports": {
  ".": {
    "types": "./dist/index.d.ts"
  }
}
```

**只给 `types`，不给 `default`**：

| 场景                          | 行为                                                    |
| ----------------------------- | ------------------------------------------------------- |
| `import type { X } from '@synra/client'` | 走 `types`，正常                                    |
| `tsconfig` 的 `types: [...]`    | 走 `types` 字段（不经 `exports`），正常                    |
| 运行期 `import '@synra/client'` | 解析失败 → **明确报错**，而不是拿到一个空模块                |

> **不给 `default` 是有意的**："这个包没有运行时"应该在**第一次尝试运行期导入时立刻暴露**，
> 而不是变成一个静默的空对象（那会导致更难查的问题）。这是 [11 D1](./11-design-principles.md) 的思路：
> 让错误在最早、最明确的地方出现。

### 7.7 本地联调

| 场景                                  | 做法                                                                 |
| ------------------------------------- | -------------------------------------------------------------------- |
| `synra-js` 内部改包                | `pnpm -r stub` → 各包 `dist` 指向 `src`，无需重建                        |
| app 仓库要用未发布的改动                  | app 的 `pnpm-workspace.yaml` 用 `overrides` + `link:` 指向本地目录（见本机 `pnpm-local-overrides` 的做法） |
| 验证"作者装上去能不能用"                 | CI 的 fixture 工程（§6 守卫 7）从**打包产物**（`pnpm pack`）安装，而不是链源码 |

> ⚠️ **fixture 必须用 `pnpm pack` 的产物**，否则测不到 `files` 字段与 `exports` 条件是否正确——
> 这是最常见的"本地好用、发布就坏"。

---

## 8. 实现清单

- [ ] 建仓库骨架：根 `package.json` / `pnpm-workspace.yaml` / `tsconfig.json` / `.gitignore` / CI
- [ ] 共享构建配置：`scripts/build.common.ts`（`mergeDefaultBuildConfig`，与本机生态一致）
- [ ] 纯类型包的 `build:done` hook（剥离 `.js`；§7.2）
- [ ] `packages/protocol`：`sync` 脚本 + 文件头 + 生成物目录
- [ ] `packages/client`：见 [23 §10](./23-client-library.md)
- [ ] `packages/synra`：type-only re-export + 被 `@synra/client` 依赖（§3.2）
- [ ] `packages/testkit`：`createMockClient` / `stub` / `calls`
- [ ] `scripts/check-shapes.ts` + 形状快照
- [ ] CI 的 8 条守卫（§6）
- [ ] 四包版本同步（`fixed` 模式 + `bumpp -r`）
- [ ] 发布脚本（拓扑排序，可续跑）
- [ ] fixture 工程：从 `pnpm pack` 产物安装并跑 `tsc --noEmit`
- [ ] 本地联调脚本（`stub` + app 侧 `overrides`）
