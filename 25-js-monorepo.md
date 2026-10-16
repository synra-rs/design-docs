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
│  ├─ theme/                    # ★ @synra/theme —— 样式真相源：包根 token + ./unocss ./vitepress ./css（[28](./28-app-ui.md)）
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
| **`@synra/theme`**   | ✅          | 插件作者的 `devDependencies`、`synra-app`、`docs` | 样式真相源：包根 token（**零依赖**）+ 子路径适配 `./unocss` / `./vitepress` / `./css`（[28 §2 R2](./28-app-ui.md)） |

**"有运行时？"这一列是关键**：前三个是纯类型（零运行时），有实现的是 `@synra/testkit`（注入对象的替身）
与 `@synra/theme`（构建期工具与映射函数——**没有消费者会 import 它的运行期代码**）。
CI 对前三个有"不允许产出 `.js`"的守卫（§6）。

> ★ **`@synra/theme` 的包根必须是零依赖的**，而这不是洁癖（[28 §2 R2](./28-app-ui.md)）：
> 官网、内联反闪屏、以及任何"只要颜色不要工具类"的消费者都从包根取东西。所以 `unocss` 只能挂在
> **`./unocss` 子路径**上，且必须声明成 `peerDependenciesMeta.optional`——写成 `dependencies`
> 就等于让包根也依赖它，**每个非 UnoCSS 消费者都要为 UnoCSS 付一次装包与体积**。
> 这条有守卫（§6 守卫 9），另一条守卫管"只有 `./unocss` 那层能 import `unocss`"。

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

### 2.2 `@synra/protocol`：**手写镜像**，由脚本核对

> ★ **本节的标题与那张图描述了一条不存在的生成路径。**
> 图上写的 `cargo test -p synra-protocol --features ts-export`（ts-rs 导出）**跑不起来**：
> `synra-protocol` 没有 `[features]` 段，也没有 `ts_export` 模块。
> 而**这一点 `packages/protocol/src/generated/wire.ts` 的文件头自己就写着**
> —— 那份文件头是对的，本节是错的。
>
> 实际机制：`src/generated/*.ts` 是**手写**的镜像，`pnpm --filter @synra/protocol sync`
> 只加头注释（`scripts/gen-protocol-types.ts`），真正保证它没写错的是
> `scripts/check-shapes.ts` 与 Rust 的逐项对照（§6 守卫 3/5）。
>
> 所以下面这张图应当读作**曾经设想过的**机制，而不是现行机制：

```
synra-core/crates/synra-protocol/**   （Rust 类型 = 唯一真相源）
        │  ~~cargo test -p synra-protocol --features ts-export~~（这条命令不存在）
        ▼
    ✗ 没有 target/ts-export/ 这个目录
        │
        ▼
synra-js/packages/protocol/src/generated/*.ts   （★ 手写，commit 进仓库）
        │  scripts/check-shapes.ts 与 Rust 逐项对照 + check-schema 的冻结摘要
        ▼
    CI 失败 = 内核改了协议而 TS 镜像没跟上
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
    "shapes": "esno scripts/check-shapes.ts",
    "sync": "pnpm --filter @synra/protocol sync",
    "release": "bumpp -r"
  }
}
```

> ★ **`scripts/**` 的 `.ts` 一律用 `esno` 跑**（`esno scripts/check-shapes.ts` 这种），与
> `synra-rust-app` / `docs` 一致 —— [27 §5.6](./27-website.md) 更是把 `esno` 写成了那本仓库的约定。
> 本文先前写的是 `tsx`，而这个仓库一度是**唯一**用它的那个。
>
> ★ **说清这次换的是什么**：`esno@4` 的依赖里就是 `tsx`（`esno` 是它的一层包装），
> 所以换的是**名字与一致性，不是运行时** —— `pnpm exec esno --version` 打印的是 `tsx v4.23.15`。
> 收益是"三个仓库的脚本用同一个命令跑"，而不是性能或能力。
>
> ★ **顺带删掉了四个脚本的 `#!/usr/bin/env tsx`。** 它们是装饰：那些文件在 git 里是 `100644`
> （**没有可执行位**），也就是从来不能被 `./scripts/x.ts` 那样直接执行 —— 一直都是经 `pnpm run` 跑的。
> 而一个指向 `tsx` 的 shebang 会让"这个文件可以自己跑"看起来成立，同时把一个已经删掉的运行时又写回了仓库。
> `docs` / `synra-rust-app` 的脚本都没有 shebang，现在这里也没有。

`@synra/testkit` **也跟同一版本**（它是 client 的替身，语义上必须对齐）。

> ★ **`@synra/theme` 不在这条版本线上**（[28 §2 R1](./28-app-ui.md)）。
> 同版本的唯一理由是"它们互相 re-export，版本错位会产生'类型来自不同代次'的错误"——
> 而它与那四个包之间**没有依赖边**（这是可以检查的，§6 守卫 6）。
> 把两者绑在一起，会让"改一支紫的明度"触发一次协议包的版本号滚动——
> 而协议包的版本号是插件作者用来判断"我的类型要不要跟着升"的东西，**它不该被一次配色改动搅动**。
> 代价说清楚：§6 的版本守卫从"四个包同版本"改成**两组**，并新增两条依赖关系守卫。

### 4.2 发布

| 包                  | `publishConfig.access` | 发布顺序（依赖优先）                |
| ------------------- | ---------------------- | ---------------------------------- |
| `@synra/protocol`    | `public`               | 1                                  |
| `@synra/client`      | `public`               | 2                                  |
| `synra`              | `public`               | 3                                  |
| `@synra/testkit`     | `public`               | 4                                  |
| `@synra/theme`       | `public`               | **与上面四个无依赖关系**（§4.1 的 ★），所以 `publishOrder()` 按目录顺序把它排在任意位置——这是对的：它一条内部依赖边都没有，排在哪都不影响安装 |

发布**不允许**跳过任何一步（发布脚本按拓扑排序执行，失败即中止；重跑可续）。

> ★ **它排在最后只是目录顺序的巧合，不是依赖要求**：`@synra/theme` 不依赖那四个，那四个也不依赖它。
> 所以第 4.2 节那个"拓扑排序"对它其实无事可做——**这也是"独立版本线"的直接结果**。

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
| core → client                 | **手写镜像**（`packages/protocol/src/generated/*.ts`）+ `check-shapes.ts` 与 Rust 对照（§2.2） |
| client → app                  | app 依赖 `@synra/client`（**dev** 依赖即可，类型在编译期用完）  |
| app 不能写自己的 `SynraClient`   | CI 守卫：禁止 app 里出现 `interface SynraResponse` 之类的重复声明 |

> **app 用 `devDependencies` 还是 `dependencies`**：因为 `@synra/client` 零运行时，
> 它**不需要**进 app 的运行期依赖（不会被打包进产物）。用 `devDependencies` 即可。

---

## 6. CI 守卫

`synra-js` 仓库的 CI（Node 侧）：

| # | 守卫                                                  | 怎么实现                                                |
| - | ----------------------------------------------------- | ------------------------------------------------------- |
| 1 | **类型包零运行时**                                      | `dist/` 下出现 `.js`/`.mjs` 即失败；另加 `package.json` 里不得有 `main`/`module`/`exports.import`。★ **名单是三个纯类型包，`@synra/theme` 不在名单里**（它有运行时，而且必须有） |
| 2 | `synra` 与 `@synra/client` **没有值导出**                | ★ **入口**（`dist/index.d.ts`）里不得出现不带 `type` 的 `export { ... }`。仓库开了 `verbatimModuleSyntax`，所以"类型还是值"由编译器强制区分 |
| 3 | **形状一致性**                                          | `scripts/check-shapes.ts`：命名约定 / 错误码 / 帧类型 / 常量 / DTO 字段**逐项与 Rust 对照**（内核仓库不在旁边时跳过并说明） |
| 4 | 生成物未被手改                                          | `pnpm sync && git diff --exit-code packages/protocol`      |
| 5 | 生成物与 core 一致                                      | 同守卫 3 的那次对照（CI 把 `synra-core` checkout 到 `synra-rust-core`——**仓库里面**，理由见 §6.1 的 ★） |
| 6 | **一组四个**包版本一致（`synra` / `@synra/client` / `@synra/protocol` / `@synra/testkit`）；`@synra/theme` **独立版本线** | 读前四个 `package.json` 的 `version` 比较；`theme` 单独一条线（§4.1 的 ★） |
| 7 | `types: ["@synra/client"]` 可用                         | `scripts/link-dist.ts` 铺出安装布局 + `tsc -p packages/client/type-tests/tsconfig.json` |
| 8 | 文档里的类型片段能编译                                    | 把 [23](./23-client-library.md) 里的代码块抽出来做类型检查（防文档漂移）★ **第 122 轮核实：已实现** —— `scripts/check-doc-snippets.ts`，挂在根 `package.json` 的 `docs:types`，实跑输出 `· doc-snippets: the complete authoring example in 23 §8 compiles` |
| 9 | **`theme` 的包根零依赖 / `unocss` 是 optional peer**      | 包根本身不出现 `from 'unocss'`；`package.json` 里 `unocss` 在 `peerDependenciesMeta` 中 `optional: true`，且**不在** `dependencies`（[28 §2 R2](./28-app-ui.md)） |
| 10 | **只有 `./unocss` 那一层能 import `unocss`**               | 除 `packages/theme/src/unocss/**` 以外，任何文件出现 `from 'unocss'` 即失败；负控制：在 `src/css/` 里加一行 import → 必须报红 |
| 11 | **标签与版本一致**（发布前，不在 `check` 里）                 | `scripts/check-release-tag.ts`：`GITHUB_REF_NAME` 去掉 `v` 之后必须等于**每个**会发布的包的版本。空标签、空工作区、任何一个包对不上都是失败（3 条正/负向测试在 `check-release-tag.test.ts`） |
| 12 | **每个声明的入口都真的存在**                               | `check-shapes.ts`：`main` / `module` / `types` / `exports.*` 里写的每一个路径都要在包目录里存在。★ **它第一次跑就抓到了 `@synra/testkit`**，见下面的 ★ |

> ★ **守卫 12 是"发布之后才会发现"的那一类里最便宜的一个，而它当场就抓到了一条。**
>
> `@synra/testkit` 的 `main` / `module` / `exports.import` 三处都写着 `./dist/index.mjs`，
> 而 `tsc -b` 产出的是 `dist/index.js` —— 运行期入口指向一个**不存在的文件**（实测：
> 按声明 import 得到 `ERR_MODULE_NOT_FOUND`，而 `dist/index.js` 有 4 个导出）。
>
> **为什么一直没有东西发现它**：`types` 是对的，所以类型检查全绿；`pnpm pack` 也照常成功
> （它只按 `files` 挑文件，不验证入口）；而 `scripts/link-dist.ts` 的名单只覆盖三个纯类型包
> （那份名单的存在理由是 `types: [...]` 的解析方式，与这里的运行时包无关）。失败落在使用者的
> 第一次 `import`，而那已经是我们发出去之后了 —— **它本来会跟着第一次发版一起出去。**
>
> 判据特意写成"声明的路径必须存在"而不是"扩展名必须是 `.js`"：后者会把 `.mjs` / `.cjs`
> 这类合法的选择也判成错，而那是另一件事。

> ★ **守卫 9 与 10 是"两层"这个设计的承重墙**，它们是同一个决定的两种失败方式：
> 包根依赖 `unocss`（9 失败）会让**官网**为一个它不用的工具付一次体积；而 `unocss` 的 import
> 散落到 `./css` 或 `./vitepress`（10 失败）会让"子路径"名存实亡——那时分离只存在于目录结构上，
> 消费者看到的仍然是一个必须装 `unocss` 的包。**一条管"谁被拖累"，一条管"分离是不是真的"。**

> ★ **本次核对修正了三处，它们都会让照做的人做错：**
> * **守卫 2** 原本写的是"检查 `dist/*.d.ts` 里没有 `export declare const/function/class`"。
>   按**整个 dist** 查会把"内部模块里有个值、入口只用 `export type` 带出类型"这种**无害**情况
>   也判成失败，逼人去加白名单；而消费方根本到不了那个内部模块。
>   实现改成只看入口。本轮正是这条守卫发现了 `@synra/protocol` 的 13 个值导出。
> * **守卫 6** 原本写"三个包"，而可发布的是**四个**（`testkit` 也是）——
>   一条只守三个包的守卫会漏掉第四个，而它的版本错开同样会让 `workspace:^` 改写成错的区间。
> * **守卫 3** 原本写"与**快照**一致"。实际做法是与 **Rust 对照**，比快照强：
>   快照只能发现"变了"，对照能发现"错了"。这一条也不是"没按文档做"，
>   而是同一目的下更强的做法，所以按实际改了描述。
>
> ★ **第 124 轮给守卫 3 的解析器补了测试，并因此找到一个真缺陷。**
>
> `check-shapes.ts` 靠正则把两侧都当文本读，所以每条对照都取决于"正则找到了东西"。此前 `scripts/`
> 只有 `publish.test.ts`，四个纯解析函数（`rustConvention` / `errorCode` 两个 / `evaluateProduct`）一个测试都没有。
> 现在 `scripts/check-shapes.test.ts` 有 **11 条**，共 27 条。
>
> **找到的缺陷**：`rustConvention` 取声明**之前最近的**那个 `rename_all`，注释里的理由是"它紧邻声明"。
> 这在类型**有**约定时成立，在类型**没有**约定时不成立 —— 那时最近的是**上一个类型**的，
> 于是它被当成这个类型的。后果有两个，第二个更糟：调用方那句"这个类型没有 `rename_all`"的失败
> **在这种情况下永远到不了**；而如果邻居的约定恰好和表里写的一样，守卫会把它报成**一致**。
>
> 现在的规则是：**只有当这个约定与声明之间没有别的类型声明时，它才属于这个类型**。
>
> ★ **而这个缺陷目前是潜伏的**：真实 crate 里表上 13 个类型**各自都有**约定，所以改前改后
> `shapes agree (10 checks)` 都是绿的。它会在有人新加一个不带 `rename_all` 的类型时发作 ——
> 而那正是加这条守卫的时刻。补测试的价值不是"现在红了"，是**它本来不会红**。

`synra-app` 仓库的对应守卫：

| # | 守卫                                                       |
| - | ---------------------------------------------------------- |
| 1 | 注入脚本 `satisfies SynraClient`（漏字段/多字段 → 编译失败）   |
| 2 | 注入对象暴露的键集合 ⊆ `SynraClient` 的键集合                 |
| 3 | 未实现的能力必须在**明确拒绝**（`unsupported`）并缺席 `supported` |
| 4 | 禁止在本仓库重复声明 `SynraResponse` / `SynraError`           |

### 6.1 发布流水线：一个文件，只有标签触发

`.github/workflows/release.yaml` 是**唯一**的 workflow，触发源**只有** `push: tags: ['v*']`。

```
tag v*  →  标签与版本一致  →  取内核（可缺）  →  install(frozen)
        →  check  →  test  →  lint  →  build
        →  publish:workspace --publish --provenance
```

> ★ **这里曾经有一步"生成物未被手改"，它被删掉了 —— 因为它不可能通过。**
>
> 那一步是 `pnpm --filter @synra/protocol sync` + `git diff`，从旧 `ci.yml` 搬过来的。两条理由各自都足够：
> ① `sync` 现在**不是生成器**：它按设计打印说明并 `exit 1`（§2.2 记录的"那条导出路径不存在"），
> 所以它在 `set -e` 下当场失败；② "重新生成再 diff"在生成器不存在时是不可能的。
>
> 真正的检查在 `check` 里：`shapes` 拿镜像与内核**逐字段对照** —— 那正是 `sync` 的说明让人去跑的那一个。
> **而它之所以能被发现，是因为这一步在 CI 里跑过一次**：本地 `pnpm run check` 不碰它。

| 决定 | 理由 |
| ---- | ---- |
| **只有一个文件** | 检查与发布共用同一串步骤，不必维护两份会分叉的步骤列表 |
| **只有标签触发** | 一次发布 = 一个标签。**代价写清楚：PR 与 push 到 main 都不再有任何检查**，坏提交要到打标签时才发现，而那时它已经在 main 上了。要恢复就再加一个 `ci.yaml`，不是往这里加 `pull_request`——那会让两件事共用一次运行，而它们的权限要求不同（发布要 `id-token: write`） |
| **`pnpm/action-setup` 不写 `version`** | 版本由 `package.json#packageManager` 单一来源决定；两处都写会得到 `Multiple versions of pnpm specified` 而直接失败 |
| **OIDC 而不是 `NPM_TOKEN`** | `permissions: id-token: write` 走 npm Trusted Publisher，仓库里没有长期 token。⚠️ **npm 后台的 "Workflow filename" 必须写成 `release.yaml`**（不是 `.yml`），否则 OIDC 校验被拒，而报错是 401、看起来像 token 过期 |
| **标签守卫排在最前** | 它是这里唯一一条"发出去就收不回来"的规则：到了 registry 的版本只能 deprecate |
| **`--provenance` 只在 CI 给** | 它需要 OIDC 提供方；本机带上会失败，而本地 `--publish` 是"发到一半失败后继续"的口子（§8 的 `--from`），必须留着 |

> ★ **顺带修掉一个"检查了很多、其实一项都没查"的洞。**
>
> 守卫 3/5 是拿 TypeScript 与 Rust **逐项对照**的，而它们一直在找**仓库旁边**的内核
> （`resolve(jsRoot, '..')`）。旧 `ci.yml` 用 `actions/checkout` 把内核放到 `path: synra-rust-core`——
> 那是**仓库里面**，因为 checkout 拒绝把文件放到工作区之外。所以 CI 里那句
> "the Rust protocol crate was not found … the checks that compare against Rust were skipped"
> 每次都在打印，而绿色的摘要和"从未对照过"的摘要在输出里长得一样。
>
> 现在 `check-shapes.ts` 的 `kernelCandidates()` 按顺序找三个位置（`SYNRA_CORE_ROOT` → 旁边 → 里面），
> 三条测试钉住它（含"里面有"这一条，也就是 CI 的布局）。**本地布局本来就能跑，所以这个洞只在 CI 里存在**——
> 这也是它活了这么久的原因。

---

## 7. 工具链与构建（选型记录：**最终没有用 `unbuild`**）

> ★ **本节原本通篇按 `unbuild` 来写，包括逐包的 `build.config.ts`。而实现选的是 `tsc -b`。**
> 本次核对后把决策补在这里，并把后面的实测保留为**依据**而不是现行方案 ——
> 那些数字正是决定不用它的原因，删掉就等于把结论和理由一起丢了。
>
> **实际用的**：四个包一律 `tsc -b`，纯类型包靠 `packages/*/tsconfig.json` 里的
> `emitDeclarationOnly: true` 达到"不产出 `.js`"。没有任何 `build.config.ts`，
> `unbuild` 也**从未进过 `pnpm-lock.yaml`**（所以不是"用过又删了"）。
>
> **为什么偏离本节的原选择**：因为本节 §7.2 的实测本身就给出了理由 ——
> `unbuild` 对没有运行时值的包也会产出 `dist/index.mjs`（1 字节的空壳），
> 于是必须再加一个 `build:done` 钩子把它删掉，然后再加一条 CI 守卫确认删掉了。
> 那是三步，且每一步都能忘记跑。
> `emitDeclarationOnly` 让它**从来不存在**：一步，且 `check-shapes.ts` 检查的是这一步确实生效
> （§6 守卫 1）。**实测发现的问题是真的，只是解法比"产出后删除"更短。**
>
> 代价说清楚：`unbuild` 会把声明**打包成自包含的 `.d.ts`**（§7.3），而 `tsc` 产出的是多个
> 互相 import 的文件 —— 也就是说**发布包里的 `.d.ts` 是散的**。
> 这对 `types: ["@synra/client"]` 没有影响（TypeScript 会顺着 import 读），
> 但如果将来改成"单文件 `.d.ts`"是需求，本节 §7.3 的实测就是重新评估的起点。

### 7.1 选型

| 用途       | 选择                                                    | 依据 |
| ---------- | ------------------------------------------------------- | ---- |
| 包管理      | **pnpm workspaces**（`packages/*`）                      | [ADR-012](./20-adr.md)；与本机既有 monorepo 一致 |
| 构建       | **`tsc -b` + `emitDeclarationOnly`**（~~`unbuild`~~，见本节开头的决策记录） | §7.2 的实测：`unbuild` 会为纯类型包产出空壳 `.js` |
| 测试       | **Vitest**                                                | 与本机既有项目一致 |
| 形状断言    | **`check-shapes.ts` 直接与 Rust 对照**（~~`tsd` / `expect-type`~~） | §6 守卫 3：对照比快照强，快照只能发现"变了" |
| 版本与发布   | `bumpp -r`；**拓扑发布脚本尚缺**（§8 清单第 11 条）          | 本机既有 monorepo 的约定 |

> **不用 Vite+ 的 `vp` 统一 CLI**：本仓库没有前端应用（只有类型包与一个测试替身），
> 引入额外工具链没有收益（与 [ADR-012](./20-adr.md) 一致；ADR-012 约束的是 **app 仓库**）。
>
> 以下 §7.2–§7.6 是**当时的实测与设计**，保留为上面那个决策的证据。

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
  "version": "1.0.0",                     // ★ 这四个包同版本（§4.1；`@synra/theme` 独立一条线，不在此列）
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
| `files`                          | `["dist", "!dist/.tsbuildinfo"]`                 | 发布物只有 `dist/`，而**构建缓存要排除**：`tsc -b` 把它写在 `dist/.tsbuildinfo` 里，而 npm 的 `files` 白名单会把它一起打进去。实测：`@synra/theme` 的包因此从 44.9 kB 降到 28.7 kB（解包 140 kB → 76 kB），而**五个包都中过这一枪**。取反模式在 npm 与 pnpm 的打包器下都生效（用 `pnpm pack` 出的真实 tarball 验过） |
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
    "@synra/theme": "^1.0.0",        // ★ 样式真相源（独立版本线，见 §4.1）；preset 从它取
    "unocss": "^66",                 // @synra/theme 的 optional peer；区间以它的 peerDependencies 为准
    "vite": "^7",
    "vue-tsc": "^3"
  }
}
```

> 作者**不需要**装 `@synra/protocol` / `synra`——前者被 `@synra/client` 内联，
> 后者是传递依赖（[23 §2.4](./23-client-library.md)）。
>
> ★ **`@synra/theme` 是 devDependency，而它的产出进产物**：它是**构建期**工具（preset 与映射函数），
> 不会被运行期 import；但它生成的 CSS、以及**字体文件**会进插件产物（[28 §5 R17、R18](./28-app-ui.md)）。
> 这两件事不矛盾，却很容易被"devDependency 不进产物"这条经验读错，所以写在这里。
> **注意 import 的是子路径**：`import { synraConfig } from '@synra/theme/unocss'`——
> 包根是零依赖的 token 层，`unocss` 只在子路径上（§2.1 的 ★、守卫 9/10）。

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
| `synra-app` 要用 `@synra/theme` 的未发布改动 | 同上，`overrides` 里加一行 `'@synra/theme': link:../synra-js/packages/theme` —— `synra-app` 的 `pnpm-workspace.yaml` 已有 `@synra/client` / `@synra/protocol` 两行先例（同一目录下），且那份注释解释了为什么用 `link:` 而不是 `packages:` 通配 |
| `docs` 要用 `@synra/theme` 的未发布改动 | 同上一行的写法（指向同一个目录）。**这是官网成为消费者之后的必然结果**（[28 §15](./28-app-ui.md)）：本地必须 `link:`，否则改一个 token 要发一次包才看得见效果 |
| 验证"作者装上去能不能用"                 | CI 的 fixture 工程（§6 守卫 7）从**打包产物**（`pnpm pack`）安装，而不是链源码 |

> ⚠️ **fixture 必须用 `pnpm pack` 的产物**，否则测不到 `files` 字段与 `exports` 条件是否正确——
> 这是最常见的"本地好用、发布就坏"。

---

## 8. 实现清单

> ★ **本次逐条对过 `synra-js` 的实际内容。最大的问题是第 7 节整节描述了一个仓库从未使用的工具链。**
>
> ### §7 写的是 `unbuild`，而这个仓库用的是 `tsc`
>
> 证据：
> * **`unbuild` 从来不是依赖** —— `pnpm-lock.yaml` 里一次都没出现过（所以不是"用过又删了"）。
> * **没有任何 `build.config.ts`**，而 §7.4 逐包给了一份。
> * `scripts/` 下没有 `build.common.ts`，也没有 `mergeDefaultBuildConfig`。
> * 四个包的 `build` 都是 `tsc -b`，纯类型包靠 `tsconfig.json` 里的 `emitDeclarationOnly: true`
>   达到"不产出 `.js`"，**没有 `build:done` 钩子**。
>
> 这不只是措辞过时：§7.2 的标题是"⚠️ **实测事实**：`unbuild` 一定会产出 `.js`"，
> §7.3 是"实测：三种 dts 产出的差别"，§7.4 给每个包一份 `build.config.ts`，§7.6 讲 `exports` 条件设计 ——
> **这些"实测"是关于一个不在依赖图里的工具的**，而按 §7.4 去找 `build.config.ts` 的人会一无所获。
>
> ★ 值得注意的是：**结论仍然成立，路径不成立**。"纯类型包不能产出 `.js`"是对的，
> 但保证它的不是 `build:done` 里那句 `rm` 或 `unbuild` 的配置，而是 `emitDeclarationOnly` ——
> 也就是说，**是编译器而不是构建后处理在保证这件事**。清单第 2、3 条因此不是"没做"，
> 而是被一个更可靠的做法取代了：少一步就是少一个会忘记跑的地方。
>
> ### §2.2 的生成机制也是假的
>
> §2.2 画的是 `cargo test -p synra-protocol --features ts-export`（ts-rs 导出）→ `target/ts-export/*.ts`
> → `pnpm sync` 复制。**那条命令跑不起来**，而且**这一点 `packages/protocol/src/generated/wire.ts`
> 的文件头自己就写着**：`synra-protocol` 没有 `[features]` 段，也没有 `ts_export` 模块，
> 导出路径从来没有存在过。真实情况是 `src/generated/*.ts` 是**手写镜像**，
> `pnpm sync` 只加头注释、`check-shapes.ts` 负责核对。
> 所以 §2.2 的标题"生成物，不手写"与它引用的文件头互相矛盾 —— 文件头是对的。
>
> ### 已经做好、且与本文一致的
>
> §2 的四个包、`pnpm-workspace.yaml`、`tsconfig.base.json`、`.github/workflows/`（当时是 `ci.yml`，
> 现在是 §6.1 的 `release.yaml`）、`scripts/{check-shapes,gen-protocol-types,link-dist}.ts`、生成物目录与头注释、
> `testkit` 的实现（64 条测试）、四包版本一致（本轮补上了守卫）、
> `types: ["@synra/client"]` 的 fixture 检查（§7 之外的清单第 12 条）。
>
> ★ **清单第 12、13 条的"机制"与本文写的不同，但要求都满足**：
> 本文要求"从 `pnpm pack` 产物安装并跑 `tsc --noEmit`"与"`stub` + app 侧 `overrides`"；
> 实际用的是 `scripts/link-dist.ts` —— 它把已构建的包按**安装后的样子**铺一份出来，
> 因为 `types: ["@synra/client"]` 是 TypeScript 按**类型根下的目录**解析的，不是走模块解析，
> 而 workspace 里的 `node_modules/@synra/client` 是指向源码的符号链接、没有 `dist`。
> 这个做法比 `pnpm pack` 快，而且测的是同一件事。
>
> ★ **而第 122 轮核对过这两项，结论是"缺"这句话已经过期了一半**：
>
> **守卫 8 存在**：`scripts/check-doc-snippets.ts`，挂在根 `package.json` 的 `docs:types`
> （`esno scripts/check-doc-snippets.ts`）。本轮实际跑过，输出是
> `· doc-snippets: the complete authoring example in 23 §8 compiles`。
> 所以这条不是"没有"，是**没有人把它写下来**。
>
> **清单第 11 条的"拓扑排序"存在，而且比这句话写的更细**：`scripts/publish.ts` 的 `publishOrder()`
> 有 6 条测试（依赖在前、**稳定**、遇环拒绝而不是硬破、私有包不参与、工作区外的依赖忽略、
> 真实工作区的清单能排出顺序），并且默认是**彩排** —— 只有 `--publish` 才真的发布。
>
> ★ **而"可续跑"在第 123 轮补上了，形式是 `--from`**：`pnpm run publish:workspace -- --from synra`
> 会输出 `resuming at synra: 3 of 4 package(s)`，并从那个包开始列出要发的包；实测过这条路径。
>
> ★ **它不去查 npm 上已发布的版本，是故意的**："哪些已经出去了"是操作者手里的事实，不是这个脚本能在离线
> 查出来的东西；把它变成一次网络查询，会让"彩排"不再等价于"发布"。
>
> ★ **而两种"读不懂就当没给"的读法都被拒绝了，因为它们正好是危险的那一种**：
> `--from @synra/typo`（不在顺序里）与 `--from`（没带值）都**报错并以 1 退出**，而不是"没匹配到，那就全发"。
> 实测：`--from @synra/typo` 输出拒绝理由、退出码 1、**一个包也没发**。因为那种读法会在操作者以为自己在
> **续发**的时刻重发整个工作区，而发出去的版本是不能撤回的。四条新测试针对这一点，并且做过变异测试：
> 把拒绝改成"返回全部顺序"，那三条拒绝测试全部失败。现在 `scripts` 共 **16** 条测试。
>
> ★ 本轮同时把 `synra-js` **整个跑了一遍**，作为"库开发结束"的证据之一：`pnpm -r test` 四个包
> （protocol 3 条、testkit 64 条、另两个没有测试文件）、`pnpm run test:scripts` 16 条、
> `pnpm run check` 四项全过（`tsc -b`、`link-dist` 铺出 3 个已构建包、`shapes agree (10 checks)`、
> 以及上面那条文档代码块），`pnpm run lint` 干净。**这个仓库在此前多轮验证里从没被跑过** —— 不是因为跑不了，
> 而是因为我一直只跑 Rust 侧；node 与 pnpm 在离线环境里是有的，181 个依赖也都在本地。

- [x] 建仓库骨架：根 `package.json` / `pnpm-workspace.yaml` / `tsconfig.json` / `.gitignore` / CI
      —— 实际是 `tsconfig.base.json` + `tsconfig.json`，其余一致
- [x] 共享构建配置：`scripts/build.common.ts`（`mergeDefaultBuildConfig`，与本机生态一致）
      ★ **不存在，而且不需要**：没有任何包用 `unbuild`（见上面的核对），
      四个包一律 `tsc -b`。这条是被取代而不是被遗漏 —— 没有构建后处理，也就没有共享配置可言
- [x] 纯类型包的 `build:done` hook（剥离 `.js`；§7.2）
      ★ **不存在，而且比它更好**：`tsconfig.json` 的 `emitDeclarationOnly: true` 让 `.js` 从来不产生，
      而不是产出后再删。少一步就是少一个会忘记跑的地方，而 `check-shapes.ts` 检查的是
      **它确实生效了**（§6 守卫 1）
- [x] `packages/protocol`：`sync` 脚本 + 文件头 + 生成物目录
      ★ 文件头是对的（"手写镜像，由脚本核对"），**§2.2 描述的 `ts-rs` 导出路径是假的**
- [x] `packages/client`：见 [23 §10](./23-client-library.md)
- [x] `packages/synra`：type-only re-export + 被 `@synra/client` 依赖（§3.2）
- [x] `packages/testkit`：`createMockClient` / `stub` / `calls`
- [x] `scripts/check-shapes.ts` + 形状快照
      ★ 核对方式是**与 Rust 对照**（命名、错误码、帧类型、常量、DTO 覆盖），而不是快照文件；
      在没有内核仓库时那几项会跳过并说明。这比快照强：快照只能发现"变了"，对照能发现"错了"
- [x] CI 的 **11** 条守卫（§6）—— **1、2、3、4、5、6、7、9、10、11 已实现**（9/10 随 `@synra/theme` 落地，
      11 是发布前的标签守卫）；**8（文档代码块能编译）没有**
      ★ **而"守卫 9/10 落地"这件事顺带暴露了一个更值得记的洞**：`scripts/` 里的代码**不在任何构建工程里**
      （每包的 `tsconfig.json` 只收自己的 `src/`），而 eslint 不做类型检查 —— 所以那里的类型错误
      **没有任何东西会发现**。`tsconfig.lint.json`（那个"给 eslint 读"的工程）里躺着两条：
      `gen-protocol-types.ts` 少了 `node:fs` 的 import（**一跑就 ReferenceError**），
      以及 `publish.ts` 两个 throw 漏了 `OrderError` 的第二个参数（于是 `error.involved` 是 `undefined`）。
      第一条会让发布流水线在"生成物"那一步崩掉；两条都已修。
      ★ **教训**：`tsc -b` 绿不等于仓库里的 TypeScript 是干净的 —— 它只检查它被指到的那些文件。
- [x] **发布流水线：一个 `release.yaml`，只有 `v*` 标签触发**（§6.1）
      ★ 旧的 `ci.yml` 已删除，它那串检查（check / test / lint / 内核对照）全部并进这一个文件；
      **"生成物未被手改"那一步没有并过来 —— 它不可能通过**（理由见 §6.1 的 ★）
      ★ **顺带修好一个洞**：守卫 3/5 在 CI 里从来没跑过（内核 checkout 到了仓库里面，而脚本只找旁边），见 §6.1 的 ★
- [ ] **`synra` 的依赖声明与它的公开面不一致**（本轮彩排时发现，尚未决定怎么处理）：
      `packages/synra/src/index.ts` 是 `export type * from '@synra/client'` / `'@synra/protocol'`，
      所以它的 `.d.ts` 里有这两个外部 import；而 manifest 把它们放在 **`devDependencies`** 里
      （`publishConfig` 之外的唯一一处）。后果：**单独 `npm i synra` 的消费者拿到的是解析不了的类型**，
      而 §3.3 的表格把"想显式安装也可以"列为一个选项。
      修法各有代价，需要一次决定：① 写进 `dependencies`/`peerDependencies` → 与 `@synra/client`
      形成**环**，而 `publishOrder()` 遇环是**拒绝**（这是它明确的设计）；② 把 `synra` 的声明做成自包含
      （需要打包器，而本仓库用的是 `tsc -b`）；③ 承认它只能作为传递依赖，改掉文档里那个选项
- [x] 四包版本同步（`fixed` 模式 + `bumpp -r`）
      ★ 本轮补上了守卫（`check-shapes.ts` 的"4 publishable packages share version"）。
      原来只有 `bumpp -r`，而"记得跑它"不是一条守卫
      ★ **而 `@synra/theme` 让这条守卫从"四个包"变成"一组四个"**（§4.1 的 ★）：
      它进这个仓库、但**不进这条版本线**——因为它与那四个包之间没有依赖边。
      于是 §6 多了两条守卫（9：包根零依赖 / optional peer；10：只有 `./unocss` 能 import `unocss`），
      而这两条**不是为版本号服务的**，是为"两层"这个形状服务的（§6 的 ★）。
- [~] 发布脚本（拓扑排序，可续跑）—— **排序已有**（第 91 轮），**续跑还没有**。
      `scripts/publish.ts`（`publishOrder()` + `readWorkspace()`），`pnpm run publish:workspace`。
      ★ **而默认不发布**：不加 `--publish` 只打印顺序，因为发布**不可撤回**（到了注册表
      的版本只能 deprecate），而演练正是一次发布前最需要的东西。
      ★ **而顺序是一个纯函数**，因为它是能静默出错的那一半：按目录顺序发布看起来
      和按依赖顺序完全一样，直到有人装了它。测试既跑合成图（链、钻石、循环、私有、外部依赖）
      也跑**真工作区**（从 `packages/*/package.json` 读），所以依赖变了而顺序没跟上会红。
      ★ **而环不是被打破而是被拒绝**：环没有合法顺序，两种硬出一个的做法都更糟
      —— 先发一个依赖缺失的版本（正是这个脚本存在的原因），或者每次顺序不同（演练就预测不了发布）。
      ★ **而写测试时它抓到了我一个错误的信念**：我写了 `@synra/testkit` 是私有的，
      而它实际上是**要发布的**（§2.4 明说：插件作者的 `devDependencies`，唯一带运行时的那个）。
      于是测试改成断言**它的位置**（最后一个，因为它依赖另外三个），而不是它不在列表里。
      ★ **而“可续跑”这一半确实还没有**：发到一半失败后重跑会重发已发的版本，
      而已发布的版本发不了第二次。所以这一条保持未勾选。
      而 `@synra/client` 依赖 `synra` / `@synra/protocol`，发布顺序错了作者会装到不存在的版本
- [x] fixture 工程：从 `pnpm pack` 产物安装并跑 `tsc --noEmit`
      ★ 机制换成 `scripts/link-dist.ts` + `packages/client/type-tests/`，测的是同一件事（见上面的说明）
- [x] 本地联调脚本（`stub` + app 侧 `overrides`）—— 同上，实际是 `link-dist.ts`
- [ ] **`packages/theme`（`@synra/theme`）**：`exports` 四个入口（`.` / `./unocss` / `./vitepress` / `./css`）+
      包根 token 数据 + 两条自检测试（preflight 发出的变量名集合 == theme 的颜色键；成对颜色的 WCAG 比值）。
      规则见 [28 §2、§13.2](./28-app-ui.md)；本文这边要动的是 §2.1 的表、§4.1 的版本线、§4.2 的发布表、
      §6 守卫 6 与新增的 9/10、§7.5 的插件模板、§7.7 的本地联调
