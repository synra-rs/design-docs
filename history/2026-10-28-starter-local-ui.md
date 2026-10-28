# 2026-10-28 · Starter 本机诊断 UI 联调

本轮记录本机 UI 的开发过程，未完成 Starter 两端能力或触摸板。后续已完成签名提交和固定版本同步，见[阶段提交记录](./2026-10-28-signed-plugin-phase.md)。

## 已验证的链路

- Starter 增加 Vue 页面，使用发布的 `@synra/client@0.0.11` 类型；已移除本地 sibling overrides。
- 页面只调用自身 `debug`，参数为 `{}`；单请求在途、8 秒等待提示、8 条脱敏记录，页面离开后忽略迟到结果。
- 10 项 Vitest、类型检查、ESLint 和 Vite 构建通过。
- 当前 Core CLI 生成包含真实 component、manifest 和 UI ZIP 的 `dist/`。
- App 的 opt-in `official-plugin-tests` 用 `SYNRA_STARTER_PACKAGE` 指向外部生成的包，经过生产安装、明确授予 `action:declare:debug`、启用和激活。
- 同一测试解析实际窗口 URL，读取解压后的入口与相对 JS 资源，经 `self.actions` 与 `self.invoke` 调用真实 Wasmtime 中的 Starter，验证返回宿主 SDK/形态/语言/主题、非法参数、错误令牌及关窗撤销。

## 真实联调发现并修正的问题

1. 将 scoped id 放在 URL authority 会被标准 URL 解析改变；文件 stem 也不等于实际入口。改为完整 id 的摘要 origin 和真实 entry，并覆盖路径编码、相对资源及 HTTPS 映射。
2. 窗口 label 保留 `@` 不符合 Tauri 字符限制，改为摘要 label。
3. Starter 原先没有激活注册动作，页面调用会被 ownership 检查拒绝；现在通过 SDK 注册 `debug`，仅声明对应的具体权限。
4. 安装器只认可 capability 表里的字面通配符，误拒绝合法的具体 `action:declare:debug`。识别规则改为已知 capability family 的匹配，未知 family 仍拒绝。
5. 共享加载器在持有 store guard 时执行 guest activation；注册动作会再次锁同一数据库。加载 API 改收共享 mutex，并只在读取记录时持锁，App、daemon 启动/启用/更新同步调整。
6. UI 种子把插件要求的 SDK floor 当成宿主实现版本；改为共享 Host SDK 常量。Rust 宿主同时对齐 `ui.theme` 和默认语言 `en`。
7. Tauri 对 Windows/Android 的摘要 origin 视为 remote，必须显式允许插件页面的两个 RPC 入口；权限组由共享 registry 生成，避免开放 Host 命令。

## 本轮检查

- Core 832 项、插件宿主 392 项、daemon 265 项测试通过。
- 应用默认桌面构建的 352 项测试通过，包含实际 Tauri ACL 解析：Host 命令不能由插件窗口调用，插件 RPC 只允许对应窗口和 origin。
- Starter 独立原生安装/RPC gate 通过；作者 CLI 校验 5 个真实导入和 3 个 UI ZIP 文件，0 warning、0 failure。
- 应用、Core/插件宿主/daemon 的 all-targets 严格 Clippy 通过；Starter 本机和 Wasm Clippy、3 项 Rust 测试通过。
- Core 受影响模块及调用方 Rust 1.88、应用默认桌面构建及测试目标 Rust 1.90、Starter 本机及 Wasm Rust 1.88 检查通过。
- UI 10 项测试、类型检查、ESLint、Vite 构建通过；应用 8 项 shapes 检查、Core API guard、文档链接检查通过。

检查后通过 Cargo 按 package 清理已用完的 App/daemon 开发产物，保留依赖缓存、源码、安装包和检查日志。
Cargo 报告删除 419 个 daemon 文件及 3098 个 App 文件；磁盘实际可用空间从 19,277,733,888 字节增至 26,261,770,240 字节。

## 尚未完成的验收

真实 Wasmtime/RPC 测试没有渲染 WebView，也没有证明手机到电脑的传输。仍需完成窗口渲染、CSP/IPC、窗口状态与权限撤销核对、两端配对和目标选择、电脑选中文本捕获到手机剪贴板，以及电脑打开链接。

SDK、CLI、README 和 CI revision pins 的后续同步及 UI CI 步骤已经完成，见[阶段提交记录](./2026-10-28-signed-plugin-phase.md)；本记录中的早期环境与测试时点保留用于追溯。

触摸板仍只有纯手势识别和 Wasm 脚手架，未接手机 UI、认证输入会话或原生鼠标后端。
