# Synra 设计文档

这里维护 Synra 的架构契约、设计取舍和实施证据。
Synra 由应用、Rust 内核、JS 库和独立插件协作，让一台设备上的操作在另一台设备完成。

**先看[当前实现状态](./CURRENT-STATE.md)**，再进入对应专题。
面向用户和插件作者的操作文档位于 [synra-app.dev](https://synra-app.dev/zh/)。

## 项目分工

| 仓库 | 作用 |
| --- | --- |
| [synra-core](https://github.com/synra-rs/synra-core) | Rust 内核、设备通信、插件运行时、SDK 与作者工具 |
| [synra-app](https://github.com/synra-rs/synra-app) | Tauri 应用、通用管理、确认与插件页面承载 |
| [synra-js](https://github.com/synra-rs/synra-js) | 客户端与协议类型、测试工具和主题 |
| [docs](https://github.com/synra-rs/docs) | VitePress 官网、使用文档与 JSON Schema |
| [Starter](https://github.com/synra-rs/synra-plugin-starter) | 独立的宿主能力与诊断示例 |
| [Touchpad](https://github.com/synra-rs/synra-plugin-touchpad) | 独立的手机触摸板插件 |
| 本仓库 | 规范、决策、路线与历史证据 |

各仓库独立管理，并排检出用于联调。目录名 `synra-rust-core`、`synra-rust-app`
与远端仓库名不同，不代表代码仍在单体仓库中。

## 阅读入口

| 主题 | 文档 |
| --- | --- |
| 产品与架构 | [01 项目章程](./01-charter.md)、[02 领域与流程](./02-domain-and-flows.md)、[03 架构基线](./03-architecture-baseline.md)、[05 系统架构](./05-system-architecture.md) |
| 通信与核心 | [04 线协议](./04-wire-protocol.md)、[06 Workspace](./06-workspace-layout.md)、[07 核心](./07-core-kernel.md)、[08 IPC 与前端](./08-ipc-and-frontend.md) |
| 插件运行 | [09 插件系统](./09-plugin-system.md)、[10 能力](./10-capability-and-sandbox.md)、[12 Wasm](./12-wasm-host.md)、[14 生命周期](./14-plugin-lifecycle.md) |
| 插件开发与分发 | [13 作者流程](./13-plugin-authoring.md)、[15 SDK](./15-plugin-sdk.md)、[16 分发](./16-plugin-distribution.md)、[29 独立插件](./29-official-plugins.md) |
| 界面与 JS | [23 客户端库](./23-client-library.md)、[25 JS 仓库](./25-js-monorepo.md)、[28 App UI](./28-app-ui.md)、[UI 设计](./ui-design/README.md) |
| 质量与运维 | [11 设计原则](./11-design-principles.md)、[17 存储与配置](./17-storage-and-config.md)、[18 可观测性](./18-observability.md)、[21 测试](./21-testing-strategy.md)、[26 开发契约](./26-core-stability-and-perf.md) |
| 决策与计划 | [19 路线图](./19-roadmap.md)、[20 ADR](./20-adr.md)、[22 风险](./22-risks.md)、[24 迁移映射](./24-migration-map.md)、[27 官网](./27-website.md) |

## 如何理解状态

- [CURRENT-STATE.md](./CURRENT-STATE.md) 记录按源码核对的当前能力和边界。
- 专题文档描述契约与设计；其中的目标、建议和阶段验收不等同于已经实现。
- [ADR](./20-adr.md) 保留决策沿革，后续替代决策优先。例如当前明文 TCP 取代了早期 mTLS 方案。
- [history](./history/README.md) 保留当时的实现、测试与失败证据，不把旧测试数量作为当前结果。
- 工具版本和依赖以各仓库工具链文件、Cargo.toml、package.json 与锁文件为准。

## 维护方式

行为变化时同时更新源码所属仓库说明、当前状态和对外文档。
使用相对链接串联本仓库文档；跨仓库使用明确的 GitHub 链接。
新增状态记录写明日期、验证入口和边界，历史证据保持原时间背景。
