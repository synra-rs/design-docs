# 2026-10-28 · 主程序独立滚动布局

## 参考与决策

参考 [Microsoft NavigationView](https://learn.microsoft.com/en-us/windows/apps/develop/ui/controls/navigationview)
的稳定、自适应导航结构，以及 [MDN overflow](https://developer.mozilla.org/en-US/docs/Web/CSS/Reference/Properties/overflow)
对尺寸约束与键盘可达性的说明。本程序采用固定视口框架、区域内滚动；使用
[overscroll-behavior](https://developer.mozilla.org/en-US/docs/Web/CSS/Reference/Properties/overscroll-behavior)
限制滚动串联。实现仍使用现有 token、UnoCSS 和组件库。

原 screen shortcut 仅包含 min-h-dvh；Grid 子项随内容长高，原有 overflow-y-auto 没有受限高度。
1280×720 时文档与侧栏均约 1082px，整页出现滚动，侧栏设备信息位于视口之外。

框架现通过 fixed inset-0 约束视口，主行列使用 minmax(0,1fr)，子区域允许收缩。
侧栏品牌和设备信息不收缩，菜单内部滚动；主内容单独滚动，提供键盘入口、可见焦点与稳定滚动条占位。
路由切换重置主内容滚动位置。小屏由独立底部导航行和剩余高度内容区组成。

另发现原窄栏 72px 无法完整容纳组件库默认 80px 菜单；窄栏改为 96px，保留两侧内边距并居中对齐图标。
没有覆盖组件内部样式或改写全局主题。规则加入 `28 §4 R54`。

## 当前浏览器验收

本轮截图与 DOM 测量均来自应用同一套 Vue 界面的本地预览，使用现有 FakeKernel 演示数据。
Settings 中的身份、自启能力提示来自模拟桥；本记录只对实际测量到的布局与交互作结论。

| 步骤 | 情况 | 结果 |
| --- | --- | --- |
| 1 | 修改前 1280×720，文档高约 1082px | 失败：整页滚动，侧栏随页面变长 |
| 2 | 修改后同尺寸，文档 1280×720，内容 scrollHeight 约 1082px | 通过：主内容内部滚动，侧栏设备信息底边 708px |
| 3 | 内容区 PageDown 与跨页导航 | 通过：内容滚动到 362px，文档仍为 0；切换路由回到顶部 |
| 4 | 1024×560 的 Home、Devices、Plugins、Settings | 通过：文档均保持 1024×560，无全局横向或纵向溢出 |
| 5 | 900×560 窄栏 | 通过：96px 侧栏容纳 80px 菜单，图标完整且对齐 |
| 6 | 390×844 小屏 | 通过：主区域 800px、底栏 44px，文档保持 844px；主区域能滚到 2244px 末尾 |
| 7 | 1280×180 导航压力场景 | 通过：44px 高菜单区域包含 180px 内容，滚动到 136px；文档为 0，侧栏信息底边 168px |

180px 高度只用于导航溢出压力验证，不改变原生窗口 560px 的最小高度。
键盘测试验证了实际滚动边界；不据此宣称完整无障碍合规。
小屏是浏览器尺寸验证，原生 Windows 像素截图和手机真机仍未在本轮验收。

截图与机器读数保存在本地 `.scratch2/layout-audit/`：before/after、最小桌面、窄栏、小屏与导航压力场景。
这是本轮新采集的证据，没有将旧截图作为当前验收。

## 工程验证

- 前端 39 个文件、347 项测试通过；后续改动仅涉及布局类与图标对齐，浏览器再次验证。
- 全量前端 lint 无警告；类型检查、Vite 构建和 8 项 shape guards 通过。
- 普通桌面开发程序重新构建成功，界面资产已嵌入。
- 构建仍提示既有单 JS chunk 超过 500kB，以及 Cargo 的已知 PDB 命名冲突；未更改告警阈值。
- 未改 Rust 业务代码，未重复运行未受影响的 Core/daemon 全量测试；未发版或部署。

## 提交记录

| 仓库 | Subject | 实测字符数 |
| --- | --- | --- |
| App | `fix: keep application panes within the viewport` | 47 |
| Design | `docs: define independent application scrolling` | 46 |

使用 PowerShell `[System.Globalization.StringInfo]::new('<subject>').LengthInTextElements` 实测，
按会话要求使用英文单行 subject、无 body、GPG 签名并推送。
