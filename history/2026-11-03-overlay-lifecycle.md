# 2026-11-03 同步与抽屉生命周期修复

应用提交：`942b68370ec35392752783e9ada2ff6f341b34d3`，已推送 `origin/main`。本轮不修改Rust源码或IPC契约。

## 根因与复现

从插件详情选择“同步到设备”，同步窗口先出现，随后消失；日志等入口也无法显示。浏览器复现确认：同步窗口仍处于打开状态，但其祖先 `.ant-drawer-content-wrapper` 已经 `display:none`。

原因是 `PluginSync` 声明在 `PluginDetail` 的抽屉正文内部，且内联挂载。弹层协调器选中同步后隐藏详情抽屉，同时隐藏了抽屉里面的同步窗口。同步仍登记为打开，却已没有可点击的关闭入口，形成展示死锁。

另外，抽屉设置 `get-container=false` 会进入inline绝对定位，受页面容器约束；日志入口的显示优先级5低于详情20和同步50，点击后把自身登记为打开并隐藏气泡，却无法取得展示权。这是“其他窗口也打不开”的另一个直接原因。

新增真实Portal回归在修复前分别确认了“同步DOM仍位于抽屉下”和“点击日志后aria-hidden仍为true”，没有只断言一个open属性。

## 变更

- 同步与详情抽屉成为同级工作流，保留同一个插件和操作状态。隐藏详情不再隐藏或销毁同步；关闭同步恢复原详情。
- 抽屉和模态框使用组件库默认body Portal；两个抽屉关闭push。页面布局不承担浮层定位，不再把独立工作流声明在会被隐藏的容器正文中。
- 日志的显式打开优先级110，允许临时查看并在关闭后恢复原待办；气泡层级1100、日志层1200，使入口不被默认1000的模态遮住。配对等后台等待状态不会因此被取消。
- 全局查找优先级90，高于普通管理流程、低于配对100。关闭查找恢复详情；保留已有工作流草稿。
- 日志按实际可见状态测量终端。控制台样式改为具有debug命名空间的CSS导入，避免多个外部scoped样式在真实组件测试中出现描述符转译错误，并为纯CSS导入提供类型声明。
- 已有行为测试读取Portal中的实际DOM和子组件；没有把Teleport全部stub掉。新增3项覆盖同步关闭重开、日志打断恢复、目标详情移除后可打开发布者抽屉。

## 本轮验证

| 检查 | 结果 |
| --- | --- |
| 完整前端测试 | 68个文件、568项通过 |
| lint、vue-tsc与生产构建 | 通过 |
| shapes | 15项通过 |
| `cargo test --no-default-features --manifest-path src-tauri/Cargo.toml` | 373项通过 |
| 默认功能 `cargo check --manifest-path src-tauri/Cargo.toml` | 通过 |
| 文件尺寸 | 本轮修改、新增的TS/Vue/CSS文件均不超过300行 |
| 浏览器 | 1280×720、390×844；详情→同步→日志→恢复同步→恢复详情，以及Ctrl-K查找后恢复详情 |

桌面实际测得抽屉为fixed且不位于main内部；同步窗口不位于抽屉内部。main始终x=240、宽1040，横向滚动量0；390px下body宽390、main横向滚动量0。日志稳定显示后焦点位于日志面板。截图与尺寸记录见 [证据](../ui-design/evidence/2026-11-03-overlays/README.md)。

浏览器使用假内核，Pixel 8同步目标为样例。截图证明窗口与布局，不代表已进行真实插件传输；未在Windows WebView2原生窗口或卓易通设备完成安装与同步实测。core原有未提交的 `crates/synra-store/src/db/migrations.rs` 保持原样，本地产物依赖该工作区。

## Android复测包

`pnpm exec tauri android build --apk --target aarch64 --ci` 通过；签名校验通过，仅含arm64-v8a Native库。已上传 [Synra-0.1.0-arm64-20261103-942b683.apk](ftp://192.168.0.201/Synra/Synra-0.1.0-arm64-20261103-942b683.apk)，保留原有文件。上传后回读完整文件，SHA-256与本地一致。

- 大小：51726484字节，约49.3 MiB。
- SHA-256：`1475922B8094257052F1EFF737BE8C719055BF74DCAFD6D745CF77CE5CA44090`。
- 本地产物：应用仓库 `src-tauri/gen/android/app/build/outputs/apk/universal/release/app-universal-release.apk`。

## 提交凭证

`942b683`：`fix: keep overlays outside suspended drawers`，实测44字符，body为空，普通push至应用仓库 `origin/main` 成功。

```powershell
[System.Globalization.StringInfo]::new('fix: keep overlays outside suspended drawers').LengthInTextElements
```
