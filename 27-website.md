# 27 · 官网与使用文档

核对日期：2026-11-04。官网仓库为 [synra-rs/docs](https://github.com/synra-rs/docs)，
站点地址为 [synra-app.dev](https://synra-app.dev)。当前能力边界见 [CURRENT-STATE](./CURRENT-STATE.md)。

## 1. 读者与职责

| 文档 | 读者 | 内容 |
| --- | --- | --- |
| 官网与指南 | 用户 | 软件用途、安装当前构建、配对、插件管理、故障排查 |
| 插件作者手册 | 插件作者 | 组件与页面、清单、能力、构建、验证、签名和打包 |
| 参考 | 用户和集成者 | 当前错误、事件、路径与平台边界 |
| 设计仓库 | 实现者 | 契约、取舍、路线、历史证据 |

公开页面描述当前行为；能力缺口以清楚的限制说明，不把设计目标写成可用功能。
不链接内部设计仓库作为操作步骤，不保留虚构的商店、安装脚本或 TestFlight 渠道。

## 2. 内容结构

VitePress 当前包含 20 组中英文正文和双语首页，共 42 个公开 Markdown 页面。

| 区域 | 内容 |
| --- | --- |
| guide | 介绍、安装、上手、设备配对、插件、任务与传输、故障排查 |
| plugins | 首个插件、结构、清单、逻辑、UI、测试、签名打包、FAQ |
| reference | 能力、事件、错误码、数据位置、平台支持 |

英文位于根路径，简体中文位于 zh/。两端栏目和页面顺序一致。
保留 transfers、publishing、events 等既有 URL，导航文字根据当前内容改为任务与传输、签名与打包、事件。

仓库 README 与 CONTRIBUTING、写作规范均不发布为站点页面。

## 3. 排版与阅读体验

沿用现有主题、颜色、字体、内容宽度和响应式布局，使用 @synra/theme 的既有 VitePress 适配。
内容组织优先采用：

- 一段表达一个意思，以任务顺序组织指南。
- 二三级标题划分可单独阅读的步骤，右侧目录可定位。
- 相同属性的比较使用表格。
- 互换命令可使用 code-group，顺序步骤分别列出。
- tip/info/warning/details 使用明确且本地化的标题。
- 长命令在代码容器内横向滚动，正文不随之横向溢出。

中文页的菜单、主题、返回顶部与更新时间使用中文标签。
首页保留双设备演示的结构，演示内容改为明确配对及插件同步，避免演示尚未建立的系统分享工作流。

## 4. 技术事实的来源

| 内容 | 校准来源 |
| --- | --- |
| 配对、发现和任务 | core 服务及 app 的实际接入 |
| 插件能力 | WIT 标注、生成表、宿主实现和清单安装 |
| 页面客户端 | synra-js 类型与 app 注入实现 |
| 错误 | protocol ErrorCode、SynraError 与 IPC IpcError |
| 路径 | synra-store Paths 与具体宿主目录选择 |
| 平台 | 构建、适配器和实体设备验收分别记录 |

名称存在于类型或协议中，不表示所有生产路径均已接入。
明确区分公开事件目录、宿主 IPC 事件，以及插件的具体交付行为。
明文 TCP、零权限确认、动态应用监听端口和接收端逐次同意均在使用文档如实说明。

## 5. Schema

public/schema/ 保存公开 schema 与版本索引，types/frozen/ 保存经审查的结构摘要。
当前 schema 仍为开发契约，已有版本目录不能单独证明外部兼容冻结。

结构改动需要明确审查并更新快照；单纯文案修改不改结构摘要。
check-schema 校验 JSON、schema 标识、引用和结构摘要，不等同于自动解析所有 Markdown 示例，
也没有完整的跨仓库 Rust 字段同步 job。示例仍需对照实际清单与 CLI。

GitHub Pages 的响应能力以实际平台为准，不宣称已经配置了自定义 immutable 缓存头或 CORS 规则。

## 6. 本地验证

在 docs 仓库运行：

```sh
pnpm install --frozen-lockfile
pnpm dev
pnpm ready
pnpm build
pnpm preview
```

| 检查 | 当前入口 |
| --- | --- |
| ESLint | pnpm lint |
| Vue 与配置类型 | pnpm typecheck |
| 双语标题层次、容器、表格及代码 | pnpm check:structure |
| Schema 与摘要 | pnpm check:schema |
| 未定标记、内部设计链接与绝对承诺 | pnpm check:public |
| 静态渲染与站内死链 | pnpm build |

验证桌面与窄屏的首页、步骤页、长命令和表格。
浏览器截图证明布局，不证明插件真实执行或设备互通。
源码提交与站点部署状态分别报告，不以本地构建成功冒充线上发布完成。

## 7. 维护

行为变化时同时更新双语页面；代码块逐字一致，中文说明写在代码块外。
当前发布流程以仓库工作流为准，文档修改不另行触发手动发布或包发布。

具体写作约定和本地操作见 docs 仓库 CONTRIBUTING.md 与 .vitepress/writing-style.md。
本轮改动及验证见 [文档校准记录](./history/2026-11-04-documentation-refresh.md)。
