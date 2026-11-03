# 插件权限崩溃修复验收

应用 `14aaf70`、协议 `51c3ca1`，1280×720 浏览器预览。截图使用真实页面，数据来自隔离假内核；空拒绝列表已按 Rust serde 规则省略。

- [已授权插件查看](./01-granted-without-denied.jpg)：没有 denied 字段仍显示两个已允许权限，可进入修改。
- [首次决策](./02-first-decision.jpg)：没有 denied 字段，新增项未预选，主动作可用。
- [全量同意之后](./03-after-allow-all.jpg)：同意结果省略 denied，页面仍正常切回查看模式。
- [浏览器控制台](./browser-console.json)：本次读取共13条debug，0 warning、0 error；包括关闭和再次打开权限页。

保存、部分授权、失败重试、草稿与事件刷新另有行为测试。本目录不代表卓易通真机验收；完整验证范围见 [实施记录](../../../history/2026-11-03-plugin-permission-crash.md)。
