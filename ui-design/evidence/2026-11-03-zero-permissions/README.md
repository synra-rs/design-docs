# 零权限插件确认样例

应用 `76b5865`、内核 `3eb3845` 的浏览器假内核样例，1280×720。名称Notes仅作样例；它与Touchpad具有相同的零权限、尚未确认状态，不代表真实Touchpad执行了触控功能。

- [确认前](./01-needs-approval.jpg)：无需权限、运行确认说明与“允许运行”入口同时可见。
- [确认后](./02-approved.jpg)：状态变为运行中，确认按钮消失，没有生成能力授权项。
- [浏览器console](./browser-console.json)：记录本次Vite调试消息；开发服务的ResizeObserver提示和验证边界见 [实施记录](../../../history/2026-11-03-zero-permission-approval.md)。
