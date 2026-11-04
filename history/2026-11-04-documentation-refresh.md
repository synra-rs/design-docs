# 2026-11-04 文档与 README 校准

## 范围与事实来源

核对 core、app、synra-js、两个独立插件、design-docs 和 VitePress docs 七个仓库。
根目录的 `app-pre-refactor` 与 `synra-rust-app` 同源、refs 完全一致且无独有提交，
只是落后 61 个提交的旧检出；核对后已删除，需要旧状态时可从该仓库的 `0f5afb1` 检出。
构建产物、缓存和历史日期记录不改写为当前能力。
实现基线及主要行为见 [CURRENT-STATE](../CURRENT-STATE.md)。

核对依据包括 Cargo workspace、应用配置和监听入口、插件 Host 适配、WIT、SDK/CLI 参数与模板、
TypeScript 及注入实现、插件清单、数据 Paths、错误枚举、事件桥和本轮已有验收记录。

## 调整

- 各仓库 README 改为用途、能力、使用入口和相关项目，移除过时文件数量、已完成的下一步和长篇排错过程。
- 本地构建、检查及实际插件验收进入各自 CONTRIBUTING。应用命令数量守卫仍保留，改读开发文档。
- 设计仓库增加当前实现入口，专题链接到现状；更新 UI 进展、路线与官网说明，旧阶段快照明确标注时间背景。
- VitePress 20 组中英文正文全部校准，保留既有 URL、主题、内容宽度与响应式结构。
- 修正虚构安装渠道、配对共享密钥、自动剪贴板设置、任意中断续传、全平台可用等不符合当前实现的表述。
- 作者文档改用无 WASI 模块与 CLI 组件编码，固定源码 SDK/CLI，区分声明、授权、宿主实现与平台能力。
- 参考区分 IPC 与插件事件、SynraError 与 IpcError，修正错误可重试分类及真实数据路径。
- 首页继续使用双设备演示，内容改为明确配对和插件同步，中英文一致；中文导航辅助标签补齐。
- 浏览器发现原 logo/favicon SVG 不存在；复用应用现有 PNG 图标并修正引用，消除资源 404。
- Schema 结构保持原样，开发快照与发布兼容承诺分开说明。
- 补齐当时已实现但未落文档的插件能力：注册表安装与预览、发布者密钥决定、每日更新检查与四种策略、
  调用记录、保留天数与配额。应用 README、内核 README、当前状态和路线状态表同步补充。
- 作者文档的标题自定义 id 改用自动锚点：VitePress 支持 `{#id}`，但 `doc-links` 守卫按标题文本计算锚点，
  自定义 id 会让链接检查失败。中文链接改指向中文锚点，构建产物已确认 `id` 与 `href` 对应。

## 组织依据

README 按项目作用、使用方式与帮助入口组织，长开发说明分流到专门文件，参考
[GitHub README 指引](https://docs.github.com/en/repositories/managing-your-repositorys-settings-and-features/customizing-your-repository/about-readmes)。
排版使用现有标题、表格和有标题容器，遵循仓库写作规范及
[VitePress 1.x Markdown 扩展](https://vuejs.github.io/vitepress/v1/guide/markdown)。

## 验证

应用 shapes 已通过 15 项，JS check 的类型、消费方、15 项形状和作者示例检查通过。
VitePress lint、类型、双语结构、schema、公开内容和生产构建通过；最终修订后再次核对。
浏览器已检查中文快速上手的桌面和 390px 视口，正文层次、目录及提示容器沿用原主题。
VitePress 本地开发页已检查中文快速上手页的导航、目录、提示容器和正文宽度；桌面截图显示侧栏、正文与右侧目录保持当前主题的阅读层次。
补充后的插件页在 1440 桌面和 390px 移动视口重新截图：新增的三张表格不溢出，标题层级与提示容器沿用原主题。
`cargo run -q -p guards` 12 项全部通过，其中文档链接检查覆盖 175 个文件；`pnpm check`/`build` 复跑通过。

本轮不重跑未修改运行逻辑的全量 Rust 或插件业务测试，不将文档与浏览器检查当成实体双设备验收。
core 原有 migrations.rs 改动保留，不纳入本轮暂存与提交。

## 工作区清理

工作区根目录只保留七个仓库和 `artifacts/`。本轮删除的都是可重建或已被取代的本地产物：

| 删除项 | 原因 |
| --- | --- |
| `probe-android/`（399 MB） | Tauri 2 安卓构建探针，只有 4 个源文件，其余为 `target/`；无文档引用，能力已由应用本体覆盖 |
| `.scratch2/`（6.2 MB） | 多轮会话日志与截图；其中三处被历史记录引用，引用处已注明目录已删除 |
| 根目录 `*.log`、`*.txt`（13 个） | 构建、测试与 WebSocket 输出转储 |
| 根目录 `synra-*.png`（6 个） | 2026-10 的旧界面截图，已被 `artifacts/20261104-docs/` 取代 |
| 根目录两个 APK（63 MB） | 2026-10-16/18 的构建，被 `artifacts/20261104-touchpad/` 的 2026-11-04 交付副本取代 |

`artifacts/` 保留：`20261104-touchpad/` 是带校验表与提交回执的交付包，
`20261104-docs/` 是本轮文档截图，`tools/` 是验证所用的已安装插件 CLI。
