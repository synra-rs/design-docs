# Synra · 设计文档

> 版本：`v1.1-draft`
> 日期：2026-10-25
> 文档目录：`D:/Projects/synra-rust/design-docs/`（独立仓库 `synra-rs/design-docs`）
> 性质：**设计文档（Design-First）**。本目录不含实现代码，只定义契约、边界、取舍与路线。

---

## 0. 这个项目是什么

**Synra 是一个跨设备动作编排框架。** 它把「一台设备上的一个动作」变成「另一台设备上的一次执行」，
而**具体做什么由插件定义**。

```
手机看到一条 GitHub 仓库名  →  分享给 Synra  →  插件识别并补全
                                              →  确认后用电脑的浏览器打开
```

- 桌面端与移动端都是**宿主**，都平等地既能发起也能执行。
- 宿主提供**能力**（打开浏览器、模拟按键、读写文件、通知…），**插件**决定"什么输入该触发哪个能力"。
- 插件在**内核的沙箱**里运行，能力由**内核强制**（不是靠约定）。

> ⚠️ **这是一个全新项目。**
> 本目录所有设计都是"从零设计"的结果，**没有任何为兼容历史格式或历史数据而存在的分支**。
> 协议从第一版就是第一版（[04 §9](./04-wire-protocol.md)）；不需要双版本、兼容模式、迁移脚本。

---

## 1. 当前目录形态

```
D:/Projects/synra-rust/          # 各仓库并排检出
├─ design-docs/                   # ★ 本仓库（synra-rs/design-docs，已推送）
├─ synra-rust-core/               # Rust workspace、daemon、CLI 与工具
├─ synra-rust-app/                # Tauri 应用
├─ synra-js/                      # 插件 UI 类型、客户端与主题
└─ docs/                          # 官网、用户文档与 schema
```

> 目录名带 `synra-rust-` 前缀是**设计期遗留**，与仓库名（`synra-core` / `synra-app`）不同。
> 位置按业主指示**不动**；GitHub 上的仓库名才是权威。

**目标形态**（五个独立仓库，对齐本机生态里 `peon-burrow` / `peon-hall` 的做法）：

```
D:/Projects/
├─ synra-core/     →  github.com/synra-rs/synra-core      内核（Cargo workspace）
├─ synra-app/      →  github.com/synra-rs/synra-app       Tauri 应用（桌面 + 移动）
├─ synra-js/   →  github.com/synra-rs/synra-js   插件 UI 的类型库 + 样式真相源（TS 包 monorepo）
├─ design-docs/    →  github.com/synra-rs/design-docs    ★ 本仓库：设计文档（内部）
└─ docs/           →  github.com/synra-rs/docs            官网 + 用户文档 + schema（对外）
```

> ⚠️ **这里曾经短暂地是六个**：样式真相源一度独立成 `synra-unocss` 仓库，后来业主要求它同时给
> **三个消费者**（宿主、插件作者、官网）用，而其中官网只取 token 层——于是它落回 `synra-js` 的
> `packages/theme`，包名 `@synra/theme`，**外部 UI 的适配一律走子路径**（[28 §1.3、§2 R2](./28-app-ui.md)）。
> 记在这里是因为"六个"这个念头很自然：**它被否决的理由不是麻烦，是消费者拿到的东西不同**
> （`./unocss`、`./vitepress`、`./css` 是三个入口，而不是三个包）。

### 1.1 `design-docs` 与 `docs` 是**两个不同的仓库**

这是最容易混的一点，务必分清：

| | **`design-docs`**（本仓库） | **`docs`**（另一个仓库） |
| --- | --- | --- |
| 读者 | **实现者**（我们自己） | **用户 / 插件作者** |
| 内容 | 目标架构、协议契约、ADR、风险、路线图 | 官网首页、使用指南、命令参考、插件作者手册 |
| 语气 | "为什么这样设计、有哪些取舍、哪里还没定" | "怎么装、怎么用、这个参数是什么" |
| 是否发布到站点 | ❌ **不上站点**（[27 §3](./27-website.md) 会说明理由） | ✅ 就是站点本身 |
| 权威性 | 设计决策的唯一权威 | 面向用户的事实，与实现保持一致 |
| 技术栈 | 纯 Markdown | VitePress + Netlify |
| 域名 | — | `synra-app.dev` |

**两者都必须存在，不能合并**：
设计文档里有大量"还没定 / 我们决定不做 / 这个限制的代价是什么"的内容，
对用户发布是有害的（把内部权衡当承诺）；而用户文档里的"怎么用"对实现者也不够。

> **交叉引用约定**：设计文档引用用户文档时写
> `synra-rs/docs` 仓库的 `guide/xxx.md`（纯文本 + 链接），反之亦然。

---

## 2. 已拍板的关键决策（速查）

| #  | 决策                                                         | 落点 |
| -- | ------------------------------------------------------------ | ---- |
| 1  | **拆五个仓库**：core / app / js / design-docs / docs             | §3、[ADR-016](./20-adr.md) |
| 2  | **插件只做双轨**（headless Wasm + UI WebView），不做"纯 UI 插件"   | [09 §2](./09-plugin-system.md)、[13 §1](./13-plugin-authoring.md)、[ADR-002](./20-adr.md) |
| 3  | **不做集群 / 选主**，保持对等直连                                | [ADR-011](./20-adr.md)、[19 §7](./19-roadmap.md) |
| 4  | **前端用标准 Vite + pnpm + Vitest**                            | [ADR-012](./20-adr.md) |
| 5  | **当前无 macOS 环境**：M0 只做 Android 的 spike，iOS 推迟         | [19 §0.1](./19-roadmap.md)、[R-14](./22-risks.md) |
| 6  | **协议是全新设计**，不兼容任何历史实现                            | [04 §9](./04-wire-protocol.md) |
| 7  | **核心要稳 / 封装要精致 / 性能内建于设计**                        | [26](./26-core-stability-and-perf.md) |
| 8  | **官网 = 用户文档 = schema 托管**，同一个仓库（`docs`），VitePress + Netlify | [27](./27-website.md) |
| 9  | **域名 `synra-app.dev`**；包标识符 **`dev.synra-app.app`**        | [22 Q-11](./22-risks.md) |
| 10 | **App UI 只有一份样式真相源**：`@synra/theme`（在 `synra-js/packages/theme`）——包根是零依赖的 token，外部 UI 走子路径（`./unocss` / `./vitepress` / `./css`），三个消费者都从它取 | [28](./28-app-ui.md) |
| 11 | **设备连接需要持久身份与认证加密**；局域网不作为信任边界 | [04 §7.2、§8.2](./04-wire-protocol.md)、ADR-017 |
| 12 | **共享运行时持有完整业务流程**；daemon 与应用只做宿主适配 | [03 §4](./03-architecture-baseline.md)、[07 §9.1](./07-core-kernel.md)、ADR-018 |
| 13 | **安装记录、安装更新任务、运行实例分别建模** | [14 §1](./14-plugin-lifecycle.md)、ADR-019 |
| 14 | **资源清理、安装提交、数据迁移与恢复分别定义保证** | [14 §3.1](./14-plugin-lifecycle.md)、ADR-020 |
| 15 | **未发布阶段允许直接重构，不承担旧格式兼容或内部 API 冻结承诺** | [26 §2](./26-core-stability-and-perf.md)、ADR-021 |
| 16 | **固定开发工具链，MSRV 独立验证；按改动范围选择验证层级** | [06 §7.0](./06-workspace-layout.md)、[21 §1.2](./21-testing-strategy.md)、ADR-022 |
| 17 | **规范、决策、当前计划与历史记录分开维护** | [24 §2.4](./24-migration-map.md)、ADR-023 |

第 11–17 项于 2026-10-25 修订，决策理由见 [20 ADR](./20-adr.md)。
**它们是目标契约，不是已实现声明**；当前差距和验收统一列于 [19 §0.2](./19-roadmap.md)。
历史记录保留原编号于 [history](./history/README.md)，不作为当前待办或测试结果。

---

## 3. 仓库拓扑（五个仓库）

沿用本机生态的做法（`peon-burrow` ↔ `peon-hall`）：**各仓库独立，并排放在同一个父目录下**。

### 3.1 `synra-core` — 内核

```
synra-core/
├─ Cargo.toml                      #   resolver = "3", edition 2024, MSRV 见 §8
├─ STABILITY.md                    #   ★ 稳定性分级（[26 §2](./26-core-stability-and-perf.md)）
├─ api/*.txt                       #   ★ cargo public-api 快照
├─ crates/
│  ├─ synra-protocol/              #     线协议（无 IO：不依赖 tokio/rusqlite/wasmtime）
│  ├─ synra-capability/            #     capability DSL（纯函数、零分配匹配）
│  ├─ synra-net/                   #     发现 + 传输
│  ├─ synra-store/                 #     SQLite + 文件沙箱 + 密钥
│  ├─ synra-observe/               #     tracing / metrics / 诊断包
│  ├─ synra-plugin-api/            #     插件契约（纯类型 + WIT 绑定）
│  ├─ synra-plugin-host/           #     ★ Wasm 沙箱 + 能力闸门
│  ├─ synra-plugin-ui/             #     UI 插件 artifact 管理
│  ├─ synra-platform/              #     Platform trait + 各平台实现
│  ├─ synra-ipc/                   #     ★ 宿主壳 ↔ 界面：命令 / 事件 / 错误的契约
│  ├─ synra-control/               #     ★ 命令行客户端 ↔ 同机内核的控制通道
│  ├─ synra-core/                  #     ★ 领域内核 + 编排
│  ├─ synra-testkit/               #     测试夹具（publish = false）
│  └─ synra-testkit-alloc/         #     计数全局分配器（publish = false）
├─ apps/{synra-cli,synra-daemon}/   #   薄宿主（"内核不依赖 UI"的验收工具）
├─ tools/{synra-plugin-cli,wit-codegen,guards,matrix,api-guard}/
├─ wit/synra-plugin.wit            #   ★ 插件接口定义（单一真相源）
├─ fixtures/wire/ · benches/ · fuzz/
└─ spike/                          #   M0 的技术验证，不进 workspace，结论见 RESULTS.md
```

**core 仓库不含本套设计文档** —— 设计文档在 [`design-docs` 仓库](https://github.com/synra-rs/design-docs)。

### 3.2 `synra-app` — Tauri 应用

```
synra-app/
├─ package.json / vite.config.ts    #   标准 Vite + pnpm（[ADR-012](./20-adr.md)）
├─ src/                             #   宿主 UI
│  ├─ pages/{home,plugins,devices,settings}/
│  ├─ components/ · stores/ · ipc/
│  └─ generated/                    #   ★ 生成物，CI 有 types-check
└─ src-tauri/                       #   ~40 个 command + Platform 实现 + UI 插件承载
   ├─ tauri.conf.json               #   identifier = dev.synra-app.app
   └─ src/inject/                   #   ★ 注入 window.synra（必须 satisfies SynraClient）
```

### 3.3 `synra-js` — 插件 UI 的类型库与样式

```
synra-js/packages/
├─ synra/          # 类型汇总入口（作者一般不装，靠传递依赖解析）
├─ client/         # @synra/client —— .d.ts only（零运行时）
├─ protocol/       # @synra/protocol —— 线协议类型（Rust 生成）
├─ theme/          # ★ @synra/theme —— 样式真相源：包根零依赖 token + ./unocss ./vitepress ./css
└─ testkit/        # @synra/testkit —— 插件作者的 mock 运行时
```

详见 [23](./23-client-library.md)、[25](./25-js-monorepo.md) 与 [28](./28-app-ui.md)。

> ★ **`@synra/theme` 是一个"层"，所以它是子路径而不是包**：宿主界面、插件界面、官网都要同一套 token，
> 但**它们要的东西不一样**——宿主与插件要 UnoCSS 的 preset，官网要 VitePress 的品牌映射，
> 内联反闪屏只要 CSS 变量。于是包根保持**零依赖**，`unocss` 只挂在 `./unocss` 子路径上、并且是
> **optional peer**（[28 §2 R2、R50](./28-app-ui.md)）。
>
> ⚠️ 这也意味着它的**版本线不与那四个类型包相同**（[25 §4.1](./25-js-monorepo.md)）：
> 让"改一支紫的明度"触发一次协议包的版本号滚动，对插件作者是纯噪音。

### 3.4 `design-docs` — 设计文档（本仓库）

```
design-docs/
├─ README.md · 01-charter.md · … · 28-app-ui.md   # 28 篇（见 §4 分流表）
├─ history/                                  # 开发与调查记录归档，不是当前规范
└─ 无构建：纯 Markdown，不上站点（理由见 §1.1）
```

### 3.5 `docs` — 官网 + 用户文档 + schema

```
docs/
├─ .vitepress/                # VitePress 配置与主题
├─ guide/ · reference/        # 英文内容（面向用户 / 插件作者）
├─ zh/                        # 简中（与英文结构镜像，CI 强制）
├─ public/
│  ├─ schema/                 # ★ JSON Schema（https://synra-app.dev/schema/…）
│  └─ CNAME                   #   synra-app.dev
├─ netlify.toml               #   构建、重定向、缓存头
└─ scripts/                   #   结构校验等
```

详见 [27 官网设计](./27-website.md)。

### 3.6 为什么拆

| 理由                    | 说明                                                                       |
| ----------------------- | -------------------------------------------------------------------------- |
| **内核可独立发布与消费** | `synra-protocol` / `synra-capability` / `synra-plugin-api` 有独立价值         |
| **发布节奏解耦**         | 内核发 crates.io，应用发安装包，类型库发 npm，站点发 Netlify —— 互不阻塞         |
| **CI 更轻**             | 内核 CI 不需要 Node/WebView/NDK；站点 CI 不需要 Rust 工具链                    |
| **物理强制内核边界**     | 应用仓库的代码**不可能**被内核 crate 依赖（反之亦然）——比"文档铁律"硬              |
| **移动端生成物体积**     | `gen/android` + `gen/apple` 不进内核仓库                                     |
| **设计文档不能对用户发布** | 里面有"还没定 / 决定不做 / 这个限制的代价"，发出去会被当成承诺（§1.1、[27 §3](./27-website.md)） |
| **schema 与用户文档同源** | schema 与描述它的页面必须一起改，放两个仓库必然漂移（[27 §2](./27-website.md)）    |

> ⚠️ **这一节曾经有过第 8 条理由（"样式真相源两端都要装，所以它该独立成仓库"），它已被删除。**
> 业主要求样式同时给**三个**消费者用（宿主、插件作者、官网），而三个要的东西不一样——
> 于是它落回 `synra-js` 的 `packages/theme`，用**子路径**分开（[28 §1.3、§2 R2/R50](./28-app-ui.md)）。
> 一条"该拆仓库"的理由因此不成立：**要分开的是入口，不是仓库。**
>
> ⚠️ **而"发布节奏解耦"这一条多了一个例外，写在这里而不是藏起来**：官网自
> [28 §15](./28-app-ui.md) 起消费 `@synra/theme`（用 `preflight: 'tokens'`），
> 所以**站点的构建依赖一个已发布的包版本**。它仍然不阻塞别人发布，
> 但"改一个 token，全站与应用同时变色"不再是自动的——官网要等一次版本升级。
> 这是业主要"官网也一起用那个库"的代价，换来的是**一条守不住的规则消失了**（[28 §13.3](./28-app-ui.md)）。

### 3.7 跨仓库的接口

| # | 接触点                              | 形态                                                       |
| - | ----------------------------------- | ---------------------------------------------------------- |
| 1 | Rust API（`synra-core`）             | 版本依赖；`Kernel` 的公开方法                                |
| 2 | TypeScript 类型                      | core 的 `ts-rs` 生成 → `synra-js` 的 `@synra/protocol`   |
| 3 | 插件 UI 的形状契约（`@synra/client`）  | **`.d.ts` only**；app 的注入脚本按它实现（[23](./23-client-library.md)） |
| 4 | 插件契约（WIT）                      | `wit/synra-plugin.wit` → 插件 SDK 与宿主实现                  |
| 5 | **JSON Schema（公开 URL）**           | `docs` 仓库的 `public/schema/**` → `https://synra-app.dev/schema/…`，被插件 manifest 的 `$schema` 引用 |
| 6 | **样式真相源（`@synra/theme`）**       | **构建期**依赖，四个入口：包根（零依赖 token）、`./unocss`、`./vitepress`、`./css`。宿主与官网用 `overrides: link:`，插件作者用版本依赖；**官网只取 token 层**（`preflight: 'tokens'`，[28 §2 R2、§15 R47](./28-app-ui.md)） |

**没有第七个。** 应用不得直接依赖 `synra-net` / `synra-store` / `synra-plugin-host` 的内部类型。

**跨仓库引用约定**：写「仓库名 + 路径」的**纯文本**，并给 GitHub 链接（本地相对路径在跨仓库时不存在）。

| 从                       | 指向                    | 写法 |
| ------------------------ | ----------------------- | ---- |
| 任一仓库                  | `synra-core`             | ``synra-rs/synra-core`` 的 `crates/synra-protocol/` |
| 任一仓库                  | `synra-app`              | ``synra-rs/synra-app`` 的 `src-tauri/src/inject/` |
| 任一仓库                  | `synra-js`           | ``synra-rs/synra-js`` 的 `packages/client/`（样式在 `packages/theme/`） |
| 任一仓库                  | `design-docs`（本仓库）   | 本地相对链接（`./` 或 `../`） |
| 设计文档 → 用户文档        | `docs`                   | ``synra-rs/docs`` 的 `guide/installation.md` |
| 用户文档 → 设计文档        | `design-docs`            | ❌ **不要链**（用户不该读内部设计，[27 §3](./27-website.md)） |

> **本地相对链接只在同一仓库内有效。** 设计文档之间用 `./` 相对链接；
> 跨到 `synra-core` / `synra-app` / `synra-js` / `docs` 时一律用 GitHub URL 或纯文本仓名。



---

## 4. 文档分流表（哪篇进哪个仓库）

| #   | 文档                                          | 归属     | 内容                                                       |
| --- | --------------------------------------------- | -------- | ---------------------------------------------------------- |
| 01  | [设计纲要](./01-charter.md)                    | `synra-core` | 目标 / 非目标、设计原则、成功判据、术语                      |
| 02  | [领域模型与用户旅程](./02-domain-and-flows.md)   | `synra-core` | 领域对象、三条主流程、业务规则清单                            |
| 03  | [架构基线](./03-architecture-baseline.md)       | `synra-core` | 三环模型、状态归属、分层与依赖方向                            |
| 04  | [线协议](./04-wire-protocol.md)                 | `synra-core` | 帧 / 信封 / 事件 / 会话 / 传输 / 发现 / 演进                  |
| 05  | [系统架构](./05-system-architecture.md)         | `synra-core` | 进程与线程模型、Tauri 边界、部署形态、关键决策                 |
| 06  | [Workspace 与 Crate 规划](./06-workspace-layout.md) | `synra-core` | core 仓库的目录、依赖图、feature、lint、CI 守卫               |
| 07  | [领域内核](./07-core-kernel.md)                 | `synra-core` | 聚合根、类型状态机、监督、意图路由、错误模型                   |
| 08  | [IPC 契约与前端集成](./08-ipc-and-frontend.md)   | `synra-app` | Tauri command/event、生成式类型、前端 stores、UI 插件 IPC      |
| 09  | [插件系统](./09-plugin-system.md)               | `synra-core` | 双轨运行时、信任模型、UI 插件承载的抽象边界                    |
| 10  | [能力模型与沙箱强制](./10-capability-and-sandbox.md) | `synra-core` | capability DSL、三层闸门、配额、逃逸面收敛                   |
| 11  | [设计原则：必须避免什么](./11-design-principles.md) | `synra-core` | 从真实工程事故中提炼的硬性约束                            |
| 12  | [Wasm 宿主](./12-wasm-host.md)                  | `synra-core` | 组件模型、wasmtime 配置、中断、实例池、故障隔离                |
| **13** | ★ [**插件写法**](./13-plugin-authoring.md)   | `synra-app` | **作者写什么代码 / manifest / 构建产物 / 运行期谁调谁** + 待讨论缺口 |
| 14  | [插件生命周期](./14-plugin-lifecycle.md)         | `synra-core` | 安装 → 校验 → 激活 → 降级 → 卸载的状态机                     |
| 15  | [插件 SDK 与作者契约](./15-plugin-sdk.md)        | `synra-core` | Rust SDK、WIT 世界、CLI、打包与签名规范                       |
| 16  | [插件分发](./16-plugin-distribution.md)          | `synra-core` | Registry、签名与信任链、跨设备同步、冲突解决                    |
| 17  | [存储与配置](./17-storage-and-config.md)         | `synra-core` | SQLite schema、文件沙箱、密钥、GC                             |
| 18  | [可观测性与诊断](./18-observability.md)          | `synra-core` | tracing、指标、跨设备 trace、诊断包、崩溃处理                   |
| 19  | [实施路线图](./19-roadmap.md)                    | `synra-core` | M0–M6 + M0 的技术验证 spike，每步标仓库归属                    |
| 20  | [架构决策记录（ADR）](./20-adr.md)               | `synra-core` | ADR 全集                                                     |
| 21  | [测试与验证策略](./21-testing-strategy.md)       | `synra-core` | 分层测试、恶意插件对抗、性能基准（app 侧另列）                   |
| 22  | [风险与开放问题](./22-risks.md)                  | `synra-core` | 风险登记、决策台账、技术天花板                                  |
| **23** | ★ [**类型库 `@synra/client`**](./23-client-library.md)  | `synra-js` | **`.d.ts` only（零运行时）**；`types: ["@synra/client"]` 一行拿到全局 `synra` 与 `window.synra` |
| 24  | [文档编写约定与维护](./24-migration-map.md)      | `synra-core` | 约定、评审机制、文档如何随设计演进                               |
| **25** | ★ [**JS 包 monorepo**](./25-js-monorepo.md)             | `synra-js` | `synra` / `@synra/client` / `@synra/protocol` / `@synra/testkit` 的仓库形态、发布、版本、CI 守卫 |
| **26** | ★ [**核心稳定性、封装与性能**](./26-core-stability-and-perf.md) | `synra-core` | 三条贯穿性要求的**可检查规则**：稳定层义务、封装四律、热路径零分配、数据结构选型 |
| **27** | ★ [**官网与文档站**](./27-website.md)                | `docs` | 站点结构、双语策略、Netlify 部署、**schema 托管与冻结**、写作规范 |
| **28** | ★ [**App UI 规范**](./28-app-ui.md)                  | `synra-app` + `synra-js` + `docs` | **三个消费者**（宿主 / 插件 / 官网）的同一套 token 与 **53 条规则**：`@synra/theme` 的多入口形状（`./unocss` / `./antd` / `./vitepress` / `./css`）、断点、字体、色彩与材质、组件层（antd）的绑定方式、文案、无障碍下限、强制执行与守不住的部分 |

> **归属列用的是真实仓库名**（`synra-core` / `synra-app` / `synra-js` / `docs`）。本仓库（`design-docs`）本身是设计文档的家，不再单列一列。
>
> ⚠️ **28 是唯一一篇跨三个仓库的文档**：宿主界面在 `synra-app`，token 的真相源在 `synra-js` 的
> `packages/theme`（§3.3，因为插件作者在仓库外，拿不到 `synra-app` 里的东西），而**官网也是它的消费者**
> （`docs`，只取 token 层）。见 [28 §1.1–§1.3、§15](./28-app-ui.md)。

### 4.1 需要**拆成两份**的内容

| 位置                                        | `synra-core` 部分                        | `synra-app` 部分                              |
| ------------------------------------------- | -------------------------------------- | ------------------------------------------- |
| [09 §5 UI 插件的承载](./09-plugin-system.md) | `trait UiHost` 抽象 + 令牌签发/校验      | WebView/iframe 创建、CSP、`on_navigation`     |
| [08](./08-ipc-and-frontend.md)               | 只需知道"内核通过命令暴露面"             | **整篇**（command 清单、类型生成、stores）     |
| [21 §5.2](./21-testing-strategy.md) UI 插件对抗 | —                                    | 整节                                        |
| [21 §10](./21-testing-strategy.md) 手工清单    | 内核相关项                              | UI / 安装包 / 真机相关项                      |

### 4.2 路线图的仓库归属

| 里程碑 | 主仓库        | 并行工作                                                        |
| ------ | ------------- | --------------------------------------------------------------- |
| M0     | **三个**      | `synra-core`：workspace/CI/**技术验证 spike**；`synra-app`：Tauri 壳层 + 假内核四页；`synra-js`：包骨架与类型生成管线 |
| M1     | `synra-core`  | `synra-app`：用假内核渲染四页；`synra-js`：`@synra/protocol` 生成物首次发布 |
| M2     | `synra-core`  | `synra-app`：设备页接线                                             |
| M3     | `synra-core`  | `synra-app`：插件页、授权、审计 UI；`synra-js`：`@synra/client` 定稿   |
| M4     | `synra-app`   | `synra-core`：平台能力补齐、插件 UI artifact 服务；`docs`：站点骨架与 P0 页面 |
| M5     | `synra-app`   | `synra-core`：移动端 wasm 后端验证                                   |
| M6     | `synra-core`  | `synra-app`：发布准备；`docs`：schema 冻结与全站内容；`synra-js`：v1 发布 |

### 4.3 测试的仓库归属

| 测试类型                          | 仓库          | 说明                                  |
| --------------------------------- | ------------- | ------------------------------------- |
| 协议编解码 / 一致性                 | `synra-core`  | [21 §2–3](./21-testing-strategy.md)    |
| 内核集成（`KernelPair`）            | `synra-core`  | [21 §4](./21-testing-strategy.md)      |
| 恶意插件对抗（headless）             | `synra-core`  | [21 §5.1](./21-testing-strategy.md)    |
| 敏感数据                           | `synra-core`  | [21 §6](./21-testing-strategy.md)      |
| UI 插件对抗（CSP / 跨 origin）       | `synra-app`   | [21 §5.2](./21-testing-strategy.md)    |
| 前端行为回归                        | `synra-app`   | [21 §7](./21-testing-strategy.md)      |
| 桌面 E2E（`tauri-driver`）          | `synra-app`   | [21 §10](./21-testing-strategy.md)     |
| **类型库形状一致性 / 零运行时**        | `synra-js`    | [25 §6](./25-js-monorepo.md)（11 条守卫；发布流水线见 §6.1） |
| **UI 样式与 token 一致性**             | `synra-app` + `synra-js` + `docs` | [28 §13](./28-app-ui.md)（11 + 14 条守卫，3 条守不住的） |
| **站点：双语言结构 / 死链 / schema**   | `docs`        | [27 §7](./27-website.md)（10 条守卫）   |

---

## 5. 阅读路径

- **想理解产品要做什么** → 01 → 02
- **想理解系统长什么样** → 03 → 05 → 06
- **要写 core 代码** → 03 → 06 → 07 → 04
- **要写 app / 前端** → 05 → 08 → 13
- **要写 App UI（宿主或插件）** → 08 → 28 → 23
- **要写插件作者文档** → 13 → 15 → 12
- **负责插件系统** → 09 → 10 → 12 → 13 → 14 → 15 → 16
- **负责排期 / 评审** → 19 → 22 → 20 → 11

---

## 6. 文档约定

- **MUST / SHOULD / MAY** 按 RFC 2119 语义使用。
- **命名**：
  - 线格式字段名 = `camelCase`（Rust 结构体用 `#[serde(rename_all = "camelCase")]`）。
  - Rust 内部字段 = `snake_case`。
  - 标识符 `synra_*` / `Synra*` 为**建议命名**，实现时可微调，但语义不可变。
- **写作风格**：**结论 → 理由 → 反例（踩过的坑）**，关键处标 ⚠️。
  目标读者是"下一个接手的人 / AI 协作者"，要求"看完即可着手写"。
- **状态标记**：✅ 已定 / 🟡 倾向 / 🔴 未定（待讨论）。**未定的设计必须显式标 🔴**，
  不允许把"还没想清楚"写成"就是这样"。
- **每篇设计文档尽量包含**：目标 / 契约 / 流程 / 失败模式 / 验收。
- 交叉引用用相对链接 + `§` 章节号（例如 `[04 §7](./04-wire-protocol.md)`）。

---

## 7. 与 `D:/Projects` 既有 Rust 生态的对齐

本机已有多个成熟的 Rust 项目，**我们主动采用它们的既有约定，而不是自己发明**（详见 [06 §9](./06-workspace-layout.md)）：

| 项目                       | 借什么                                                                          |
| -------------------------- | ------------------------------------------------------------------------------- |
| `peon/peon-burrow`          | ★ **双仓库拓扑**、文档结构（项目层 + `design/` + `decisions/`）、CI 守卫、"等就绪不要 sleep" |
| `clash-verge-rev`           | `[workspace.lints.clippy]` 严格 lint 集、`panic = "unwind"`                        |
| `harbor`                    | 发布编排（`cargo harbor`，拓扑排序 + 可续跑 + CI `--plan`）                          |
| `cargo-bumpp`               | 版本与 tag（`cargo bumpp`，Conventional Commits，**无 CHANGELOG 文件**）             |
| `bigseek`                   | 跨语言测试向量文件锁定语义；wasm 体积的实测记录方法                                  |
| `crate-plugin-kit` / `bmux` | "unsafe 收在一个岛"的纪律、ABI 契约独立成 crate、能力名只做加法演进（[ADR-013](./20-adr.md)） |

**三个生态空白**（需要自己趟，见 [22 R-01b](./22-risks.md)）：
① 没有任何项目用过 `wasmtime` / 组件模型；② 没有项目做过 Tauri 移动端；③ 没有项目做过 UDP 广播 / 局域网发现。
→ 所以 [19 §0.1](./19-roadmap.md) 把 M0 排成一组技术验证 spike。

---

## 8. 版本锚点

| 组件     | 版本               | 说明                                                  |
| -------- | ------------------ | ----------------------------------------------------- |
| Tauri    | `2.12.x`           | 桌面 + 移动统一外壳，channel/event IPC                   |
| Rust     | edition 2024       | `async fn in trait`、`let-else`、RPITIT                |
| **MSRV** | **`1.88`**         | 写在 `Cargo.toml` 的 `[workspace.package] rust-version`  |
| Wasmtime | **`38.x`**         | 组件模型 + cranelift（以 `Cargo.toml` 实锁版本为准）       |
| tokio    | `1.x`              | 异步运行时与结构化并发                                   |
| rusqlite | **`0.40`**（`bundled`） | 本地持久化。`bundled` 是**移动端的前提**，不是可选项（见 spike 4） |
| quinn    | `0.11.x`           | 可选传输（[ADR-004](./20-adr.md)）                      |

> 开发与 CI 主检查的目标工具链在 `rust-toolchain.toml` 固定；MSRV 单独显式选择编译器验证。
> 此约定取代 ADR-014，尚待实现，见 [06 §7.0](./06-workspace-layout.md)、[ADR-022](./20-adr.md)。
>
> ⚠️ **Wasmtime 早先在此处写的是 `48.x`，workspace 实际锁的是 `38`。**
> 本文改为记录实际值：一份比现实更响亮的版本锚点，正是它本该防住的那种漂移。
