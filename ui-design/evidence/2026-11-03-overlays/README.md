# 同步与日志弹层验收

应用 `942b683`，浏览器真实截图；设备与插件来自隔离假内核。本目录验证共享管理组件，不表示GitHub样例或Touchpad已经完成真实同步传输。

| 截图 | 内容 |
| --- | --- |
| [01 详情](./01-detail.jpg) | 1280×720，详情浮于页面上方，主区保留原位置 |
| [02 同步恢复](./02-sync-restored.jpg) | 同步窗口持续可见，关闭日志后可回到该窗口 |
| [03 同步期间打开日志](./03-console-during-sync.jpg) | 日志显示并取得焦点，原同步保留待恢复 |
| [04 全局查找](./04-search-over-detail.jpg) | Ctrl-K在详情打开时也能显示，关闭后恢复详情 |
| [05 手机同步](./05-sync-390.jpg) | 390×844，同步窗口完整可达 |
| [06 手机日志](./06-console-390.jpg) | 390×844，日志仍可打开 |

[尺寸记录](./viewport-checks.json) 核对body、main、横向滚动和Portal位置。[浏览器console](./browser-console.json) 本次采集0 warning、0 error。Native与业务传输边界见 [实施记录](../../../history/2026-11-03-overlay-lifecycle.md)。
