# 研究依据与可迁移结论
## 2. 研究依据与可迁移结论

### 2.1 Node Modules Inspector / Devframe

`node-modules-inspector` 的公开仓库把产品定位为“查看已安装依赖的交互式检查器”，同时提供静态构建、CLI 报告和 MCP 接口。它的 UI 不是普通内容站，而是一个开发者工作台：左侧保持项目上下文，主区承载可扫描的数据，工具面板承载操作和诊断。仓库当前前端依赖中包含 `@xterm/xterm`、`@xterm/addon-fit`、`devframe` 和 `@devframes/agentic`，说明终端、按容器适配尺寸和工具面板是其产品结构的一部分，而不是装饰性组件。

对 Synra 有价值的不是“黑底终端”本身，而是下面的工作台模型：

| 参考做法 | Synra 的迁移方式 |
|---|---|
| 左侧项目上下文持续可见 | 桌面持续显示当前设备和一级导航；窄窗口改为底部导航 |
| 主区只展示当前工作范围 | 首页、设备、插件和设置各自有明确任务，不把所有入口堆在首页 |
| 工具面板独立于主业务页面 | `synra://wire` 作为全局线缆监视器，打开后不改变后台任务 |
| 终端按会话和状态管理 | 先保持只读流；后续如增加交互会话，必须有明确会话 ID、来源和终止动作 |
| 命令、消息、终端共享一个工作台 | 搜索、通知、待处理确认和终端入口统一使用全局协调器 |
| 详情按需打开 | 插件、设备和诊断对象先给摘要，完整字段进入 Drawer 或详情面 |

Devframe 的终端插件进一步展示了几条实用规则：终端既可以是只读流，也可以是可输入的 PTY；会话有状态、重启、终止、移除和尺寸变化；外部入口可以通过会话 ID 聚焦某个终端。Synra 当前只需要只读诊断流，但应提前把“日志流”和“终端会话”分开建模，避免将来把后台命令误做成普通日志。

### 2.2 跨平台和交互规范

- **Apple HIG** 强调熟悉、清晰、可恢复和明确反馈；动效要短、准、有目的，并且不能成为传达信息的唯一方式。见 [Design principles](https://developer.apple.com/design/human-interface-guidelines/design-principles/)、[Motion](https://developer.apple.com/design/human-interface-guidelines/motion) 和 [Feedback](https://developer.apple.com/design/human-interface-guidelines/feedback)。
- **Material 3** 将动效视为表达空间关系和交互物理感的系统，而不是每个组件自行决定。Synra 不直接引入 Material 组件，但应保留“统一 motion scheme”的思路。见 [Material Motion](https://m3.material.io/styles/motion)。
- **Fluent 2** 把响应式拆成重新定位、缩放、重排、显示/隐藏和重新架构，并强调统一间距节奏。见 [Layout](https://fluent2.microsoft.design/layout)。
- **WCAG 2.2** 要求键盘焦点可见、指针目标有足够尺寸，并要求拖拽功能提供不依赖拖拽的替代操作。见 [Focus Appearance](https://www.w3.org/WAI/WCAG22/Understanding/focus-appearance)、[Target Size](https://www.w3.org/WAI/WCAG22/Understanding/target-size-minimum) 和 [Dragging Movements](https://www.w3.org/WAI/WCAG22/Understanding/dragging-movements)。
- **Carbon、Fluent 和 Ant Design 的图标规范**都强调图标要简单、可识别、风格统一；图标按钮不能依赖过小的图形本身来提供命中范围。见 [Carbon Icons](https://www.carbondesignsystem.com/building-blocks/foundations/icons/guidelines)、[Fluent Iconography](https://fluent2.microsoft.design/iconography/) 和 [Ant Design Icons](https://ant.design/docs/spec/icon/)。

### 2.3 终端实现依据

Synra 已经使用 xterm.js。其官方 API 提供光标样式、滚动回看、屏幕阅读器模式、选择、链接处理、主题、滚动缓动和 `FitAddon` 尺寸适配。相关实现约束见 [ITerminalOptions](https://xtermjs.org/docs/api/terminal/interfaces/iterminaloptions/)、[Terminal API](https://xtermjs.org/docs/api/terminal/classes/terminal/)、[Using addons](https://xtermjs.org/docs/guides/using-addons/) 和 [Link handling](https://xtermjs.org/docs/guides/link-handling/)。
