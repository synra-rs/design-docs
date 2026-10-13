# Synra · 设计文档

> 版本：`v1.0-draft`
> 日期：2026-07-22
> 文档目录：`D:/Projects/synra-rust/design-docs/`（**未来会独立成一个仓库**）
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

## 1. 当前目录形态（设计期）

```
D:/Projects/synra-rust/            # ← 设计期的容器（三者未来各自独立成仓库）
├─ design-docs/                    # ★ 本文档集；未来独立发一个仓库
├─ synra-rust-core/                # 内核仓库骨架（目前只有 README.md，尚未 git init）
└─ synra-rust-app/                 # 应用仓库骨架（目前只有 README.md，尚未 git init）
```

**未来的目标形态**（对齐本机生态里 `peon-burrow` / `peon-hall` 的做法）：

```
D:/Projects/
├─ synra-rust-core/                # 独立仓库 1：Rust workspace（内核 + CLI + 工具）
├─ synra-rust-app/                 # 独立仓库 2：Tauri 应用（桌面 + 移动 + 前端）
└─ synra-design-docs/              # 独立仓库 3（或并入 core 的 design-docs/）
```

---

## 2. 已拍板的关键决策（速查）

| # | 决策                                                       | 落点 |
| - | ---------------------------------------------------------- | ---- |
| 1 | **拆两个仓库**：`synra-rust-core` + `synra-rust-app`          | §3、[ADR-016](./20-adr.md) |
| 2 | **插件只做双轨**（headless Wasm + UI WebView），不做"纯 UI 插件" | [09 §2](./09-plugin-system.md)、[13 §1](./13-plugin-authoring.md)、[ADR-002](./20-adr.md) |
| 3 | **不做集群 / 选主**，保持对等直连                              | [ADR-011](./20-adr.md)、[19 §7](./19-roadmap.md) |
| 4 | **前端用标准 Vite + pnpm + Vitest**                          | [ADR-012](./20-adr.md) |
| 5 | **当前无 macOS 环境**：M0 只做 Android 的 spike，iOS 推迟      | [19 §0.1](./19-roadmap.md)、[R-14](./22-risks.md) |
| 6 | **协议是全新设计**，不兼容任何历史实现                          | [04 §9](./04-wire-protocol.md) |
| 7 | **核心要稳 / 封装要精致 / 性能内建于设计**（三条贯穿性要求落成可检查规则） | [26](./26-core-stability-and-perf.md) |

---

## 3. 仓库拓扑（已决策：拆两个仓库）

沿用本机生态的做法（`peon-burrow` ↔ `peon-hall`）：**内核与宿主是两个独立 git 仓库，并排放在同一个父目录下**。

```
synra-rust-core/                   # 内核 + CLI + 工具（Cargo workspace）
├─ Cargo.toml                       #   resolver = "3", edition 2024, MSRV 见 §8
├─ STABILITY.md                     #   稳定性分级（哪些 crate 遵守 SemVer）
├─ design-docs/                     #   本文档集中归 core 的部分
├─ crates/                          #   11 个内核 crate
│  ├─ synra-protocol/               #     线协议（无 IO）
│  ├─ synra-capability/             #     capability DSL（纯函数）
│  ├─ synra-net/                    #     发现 + 传输
│  ├─ synra-store/                  #     SQLite + 文件沙箱 + 密钥
│  ├─ synra-observe/                #     tracing / metrics / 诊断包
│  ├─ synra-plugin-api/             #     插件契约（纯类型 + WIT 绑定）
│  ├─ synra-plugin-host/            #     ★ Wasm 沙箱 + 能力闸门
│  ├─ synra-plugin-ui/              #     UI 插件 artifact 管理
│  ├─ synra-platform/               #     Platform trait + 各平台实现
│  ├─ synra-core/                   #     ★ 领域内核 + 编排
│  └─ synra-testkit/                #     测试夹具（publish = false）
├─ apps/{synra-cli,synra-daemon}/    #   薄宿主（也是"内核不依赖 UI"的验收工具）
├─ tools/{synra-plugin-cli,wit-codegen}/
├─ wit/synra-plugin.wit             #   ★ 插件接口定义（单一真相源）
└─ spike/                           #   M0 的技术验证（[19 §0.1](./19-roadmap.md)）

synra-rust-app/                    # Tauri 应用（桌面 + 移动 + 前端）
├─ package.json / vite.config.ts     #   标准 Vite + pnpm（[ADR-012](./20-adr.md)）
├─ design-docs/                      #   本文档集中归 app 的部分
├─ src/                              #   前端：宿主 UI + 插件 UI 运行时
└─ src-tauri/                        #   ~40 个 command + Platform 实现 + UI 插件承载
```

### 3.1 为什么拆

| 理由                    | 说明                                                                       |
| ----------------------- | -------------------------------------------------------------------------- |
| **内核可独立发布与消费** | `synra-protocol` / `synra-capability` / `synra-plugin-api` 有独立价值         |
| **发布节奏解耦**         | 内核发 crates.io，应用发安装包（Tauri bundler），互不阻塞                       |
| **CI 更轻**             | 内核 CI 不需要 Node/WebView/NDK；应用 CI 不需要跑 wasmtime 的 feature 矩阵      |
| **物理强制内核边界**     | 应用仓库的代码**不可能**被内核 crate 依赖（反之亦然）——比"文档铁律"硬              |
| **移动端生成物体积**     | `gen/android` + `gen/apple` 不进内核仓库                                     |

### 3.2 跨仓库的接口（唯三接触点）

| # | 接触点                              | 形态                                                       |
| - | ----------------------------------- | ---------------------------------------------------------- |
| 1 | Rust API（`synra-core`）             | 版本依赖；`Kernel` 的公开方法                                |
| 2 | TypeScript 类型                      | core 侧用 `ts-rs` 生成 → 发 npm 包（或 app 侧 build script 生成） |
| 3 | 插件 UI 的类型库（`@synra/client`）  | **`.d.ts` only（零运行时）**；描述注入给插件 WebView 的全局 `synra`（[23](./23-client-library.md)） |

**除此之外没有第四个**。应用不得直接依赖 `synra-net` / `synra-store` / `synra-plugin-host` 的内部类型。

**跨仓库引用约定**：写「仓库名 + 路径」的**纯文本**，并给本地相对链接（GitHub 上跨仓库链接会失效）。

| 从                                 | 到 core                   | 到 app                    |
| ---------------------------------- | ------------------------- | ------------------------- |
| `synra-rust-core/design-docs/*.md`  | 同仓库 → `./`             | `../../synra-rust-app/…`  |
| `synra-rust-app/design-docs/*.md`   | `../../synra-rust-core/…` | 同仓库 → `./`             |

---

## 4. 文档分流表（哪篇进哪个仓库）

| #   | 文档                                          | 归属     | 内容                                                       |
| --- | --------------------------------------------- | -------- | ---------------------------------------------------------- |
| 01  | [设计纲要](./01-charter.md)                    | core     | 目标 / 非目标、设计原则、成功判据、术语                      |
| 02  | [领域模型与用户旅程](./02-domain-and-flows.md)   | core     | 领域对象、三条主流程、业务规则清单                            |
| 03  | [架构基线](./03-architecture-baseline.md)       | core     | 三环模型、状态归属、分层与依赖方向                            |
| 04  | [线协议](./04-wire-protocol.md)                 | core     | 帧 / 信封 / 事件 / 会话 / 传输 / 发现 / 演进                  |
| 05  | [系统架构](./05-system-architecture.md)         | core     | 进程与线程模型、Tauri 边界、部署形态、关键决策                 |
| 06  | [Workspace 与 Crate 规划](./06-workspace-layout.md) | core | core 仓库的目录、依赖图、feature、lint、CI 守卫               |
| 07  | [领域内核](./07-core-kernel.md)                 | core     | 聚合根、类型状态机、监督、意图路由、错误模型                   |
| 08  | [IPC 契约与前端集成](./08-ipc-and-frontend.md)   | **app**  | Tauri command/event、生成式类型、前端 stores、UI 插件 IPC      |
| 09  | [插件系统](./09-plugin-system.md)               | core     | 双轨运行时、信任模型、UI 插件承载的抽象边界                    |
| 10  | [能力模型与沙箱强制](./10-capability-and-sandbox.md) | core | capability DSL、三层闸门、配额、逃逸面收敛                   |
| 11  | [设计原则：必须避免什么](./11-design-principles.md) | core | 从真实工程事故中提炼的硬性约束                            |
| 12  | [Wasm 宿主](./12-wasm-host.md)                  | core     | 组件模型、wasmtime 配置、中断、实例池、故障隔离                |
| **13** | ★ [**插件写法**](./13-plugin-authoring.md)   | core     | **作者写什么代码 / manifest / 构建产物 / 运行期谁调谁** + 待讨论缺口 |
| 14  | [插件生命周期](./14-plugin-lifecycle.md)         | core     | 安装 → 校验 → 激活 → 降级 → 卸载的状态机                     |
| 15  | [插件 SDK 与作者契约](./15-plugin-sdk.md)        | core     | Rust SDK、WIT 世界、CLI、打包与签名规范                       |
| 16  | [插件分发](./16-plugin-distribution.md)          | core     | Registry、签名与信任链、跨设备同步、冲突解决                    |
| 17  | [存储与配置](./17-storage-and-config.md)         | core     | SQLite schema、文件沙箱、密钥、GC                             |
| 18  | [可观测性与诊断](./18-observability.md)          | core     | tracing、指标、跨设备 trace、诊断包、崩溃处理                   |
| 19  | [实施路线图](./19-roadmap.md)                    | core     | M0–M6 + M0 的技术验证 spike，每步标仓库归属                    |
| 20  | [架构决策记录（ADR）](./20-adr.md)               | core     | ADR 全集                                                     |
| 21  | [测试与验证策略](./21-testing-strategy.md)       | core     | 分层测试、恶意插件对抗、性能基准（app 侧另列）                   |
| 22  | [风险与开放问题](./22-risks.md)                  | core     | 风险登记、决策台账、技术天花板                                  |
| **23** | ★ [**类型库 `@synra/client`**](./23-client-library.md)  | **js** | **`.d.ts` only（零运行时）**；`types: ["@synra/client"]` 一行拿到全局 `synra` 与 `window.synra` |
| 24  | [文档编写约定与维护](./24-migration-map.md)      | core     | 约定、评审机制、文档如何随设计演进                               |
| **25** | ★ [**JS 包 monorepo**](./25-js-monorepo.md)             | **js**   | `synra` / `@synra/client` / `@synra/protocol` / `@synra/testkit` 的仓库形态、发布、版本、CI 守卫 |
| **26** | ★ [**核心稳定性、封装与性能**](./26-core-stability-and-perf.md) | core | 三条贯穿性要求的**可检查规则**：稳定层义务、封装四律、热路径零分配、数据结构选型 |

> 编号到 26；**23 与 25 是 JS 仓库的文档**（类型库与包 monorepo），其余在 core（[08](./08-ipc-and-frontend.md) 与 [13 §5](./13-plugin-authoring.md) 归 app）。

### 4.1 需要**拆成两份**的内容

| 位置                                        | core 部分                              | app 部分                                    |
| ------------------------------------------- | -------------------------------------- | ------------------------------------------- |
| [09 §5 UI 插件的承载](./09-plugin-system.md) | `trait UiHost` 抽象 + 令牌签发/校验      | WebView/iframe 创建、CSP、`on_navigation`     |
| [08](./08-ipc-and-frontend.md)               | 只需知道"内核通过命令暴露面"             | **整篇**（command 清单、类型生成、stores）     |
| [21 §5.2](./21-testing-strategy.md) UI 插件对抗 | —                                    | 整节                                        |
| [21 §10](./21-testing-strategy.md) 手工清单    | 内核相关项                              | UI / 安装包 / 真机相关项                      |

### 4.2 路线图的仓库归属

| 里程碑 | 主要仓库 | 另一仓库的并行工作                                                  |
| ------ | -------- | ------------------------------------------------------------------- |
| M0     | **两个** | core：workspace/CI/**技术验证 spike**；app：前端骨架（假内核）+ Tauri 壳层 |
| M1     | core     | app：用假内核渲染四页                                                 |
| M2     | core     | app：设备页接线                                                      |
| M3     | core     | app：插件页、授权、审计 UI                                            |
| M4     | **app**  | core：平台能力补齐、插件 UI artifact 服务                              |
| M5     | **app**  | core：移动端 wasm 后端验证                                            |
| M6     | 两个     | —                                                                   |

### 4.3 测试的仓库归属

| 测试类型                     | 仓库 | 说明                                  |
| ---------------------------- | ---- | ------------------------------------- |
| 协议编解码 / 一致性            | core | [21 §2–3](./21-testing-strategy.md)    |
| 内核集成（`KernelPair`）       | core | [21 §4](./21-testing-strategy.md)      |
| 恶意插件对抗（headless）        | core | [21 §5.1](./21-testing-strategy.md)    |
| 敏感数据                      | core | [21 §6](./21-testing-strategy.md)      |
| UI 插件对抗（CSP/跨 origin）   | app  | [21 §5.2](./21-testing-strategy.md)    |
| 前端行为回归                   | app  | [21 §7](./21-testing-strategy.md)      |
| 桌面 E2E（`tauri-driver`）     | app  | [21 §10](./21-testing-strategy.md)     |

---

## 5. 阅读路径

- **想理解产品要做什么** → 01 → 02
- **想理解系统长什么样** → 03 → 05 → 06
- **要写 core 代码** → 03 → 06 → 07 → 04
- **要写 app / 前端** → 05 → 08 → 13
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
| Wasmtime | `48.x`             | 组件模型 + WASI 0.2/0.3                                |
| tokio    | `1.x`              | 异步运行时与结构化并发                                   |
| rusqlite | `0.3x`（`bundled`） | 本地持久化                                             |
| quinn    | `0.11.x`           | 可选传输（[ADR-004](./20-adr.md)）                      |

> ⚠️ **不写 `rust-toolchain.toml`**（会让 CI 的 MSRV job 失效，[ADR-014](./20-adr.md)）。
