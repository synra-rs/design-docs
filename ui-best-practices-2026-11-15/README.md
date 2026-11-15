# Synra 跨端界面最佳实践（2026-11-15）

> 这是一组按主题拆分的研究和设计方向文档，用于指导 Synra 下一轮 UI 更新。它补充现有 [`ui-design`](../ui-design/) 文档，不替代已经完成的实现记录和交互契约。

这组文档的核心结论是：Synra 当前不需要再增加一套组件，而需要统一工作台层级、状态反馈、跨端重排、图标语言和动效节制。`node-modules-inspector` / Devframe 提供了终端和开发者工作台的参考，Apple HIG、Material 3、Fluent 2、Carbon、Ant Design 和 WCAG 提供跨平台交互、动效、图标与可访问性依据。

## 文档目录

| 文档 | 内容 |
|---|---|
| [研究依据与可迁移结论](./01-research.md) | Node Modules Inspector、Devframe、Apple、Material、Fluent、WCAG、图标和 xterm.js 研究 |
| [目标体验、视觉语言与跨端布局](./02-visual-and-cross-platform.md) | 目标体验、表面层级、颜色、字体、Icon 规则、响应式和跨端交互 |
| [交互模式](./03-interaction-patterns.md) | 页面头部、摘要/详情、Drawer、Modal、搜索、反馈层级 |
| [动效规范](./04-motion.md) | 动效原则、时间 token、页面/任务/弹层/终端动效和 reduced-motion |
| [终端与 `synra://wire`](./05-terminal.md) | 终端定位、布局、视觉、暂停/复制/搜索、数据和性能约束 |
| [四个页面的改版方向](./06-page-directions.md) | 首页、设备、插件、设置的具体调整建议 |
| [实施、验收与边界](./07-implementation-and-acceptance.md) | 分阶段计划、验收清单和明确不做的事情 |
| [文档关系与研究链接](./08-references.md) | 与现有 UI 文档的关系和外部参考链接 |
| [布局稳定与渐进披露](./09-layout-stability-and-disclosure.md) | 父级 hover/键盘/触摸动作区、稳定槽位、顶部通知层和防止页面跳动 |

## 推荐阅读顺序

1. 先读[研究依据与可迁移结论](./01-research.md)，理解为什么采用工作台和终端模型。
2. 再读[目标体验、视觉语言与跨端布局](./02-visual-and-cross-platform.md)和[交互模式](./03-interaction-patterns.md)，确定页面骨架。
3. 然后读[交互模式](./03-interaction-patterns.md)、[布局稳定与渐进披露](./09-layout-stability-and-disclosure.md)和[动效规范](./04-motion.md)，确定动作收纳、稳定布局和反馈方式。
4. 再读[终端与 `synra://wire`](./05-terminal.md)，确定诊断工具的表现方式。
5. 最后按[四个页面的改版方向](./06-page-directions.md)和[实施、验收与边界](./07-implementation-and-acceptance.md)拆实施批次。

## 版本边界

- 目录日期表示这一版研究和设计方向的时间点，后续 UI 研究应创建新的日期目录。
- 本版不要求替换 Vue、Ant Design Vue、UnoCSS、`@synra/theme` 或现有 IPC/内核契约。
- 本版不把 Node Modules Inspector 的依赖图、市场首页或终端样式直接复制到 Synra。
- 本版优先改善信息层级、状态可见性和跨端一致性，再决定是否需要新增组件或能力。
