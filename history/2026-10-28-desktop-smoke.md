# 2026-10-28 · Starter 真实桌面窗口验收

## 契约与入口

自动安装、授权、启用和 RPC 测试仍不能证明 WebView 能加载真实 HTML、JS 与注入桥。
App 新增显式 `--starter-ui-smoke <dist>` 开发入口，仅在 desktop debug 构建且启用
`official-plugin-tests` 时编译。启动检查早于数据库和监听器初始化，要求
`com.synra.ui-smoke` 标识及空 deep-link schemes；拒绝在正常应用 profile 中运行。

空插件列表的独立 profile 经生产安装、最小 debug 授权、启用和原生 `plugin_ui_open`
打开 Starter。已有插件时拒绝自动覆盖；之后可不带该参数重新启动验收实例。
这个入口不属于发布构建，也不替代真实手机到电脑的功能验收。

## 真实启动发现的问题

首轮 Windows 启动在窗口出现前栈溢出。进程 75736 的本机 crash dump 显示
`0xc00000fd`，故障点为 `__chkstk`，栈上调用者为 Cranelift 的
`constructor_simplify`；PE header 的主线程 stack reserve 为 1048576 字节。
这发生在实际 Starter 组件的编译阶段，测试线程中通过的链路无法覆盖这个条件。

共享 Wasmtime 后端把 `Component::new` 放入显式 8 MiB 栈的 scoped worker。
backend clones 共用编译准入锁，每个 engine 同时最多一个编译 worker；没有额外任务队列，
借用原始有界输入，不复制整个组件。线程创建与 panic 转为结构化错误。
guest 的 512 KiB Wasm 栈上限不变，所有宿主通过同一个编译入口获得修复。

真实 Starter 集成测试改在 1 MiB 调用线程内完成生产安装、授权、启用、资源与 RPC、
撤销和重装流程，防止只在较大测试线程栈上验证。

## 验收记录

- 共享 Wasm 宿主 392 项 runtime 测试通过，另无 runtime 配置的 345 项通过。
- App 358 项测试通过，包含 1 MiB 调用线程上的真实 Starter 生命周期与 RPC。
- Core runtime 和 App all-targets 严格 Clippy 通过；Core Rust 1.88、App Rust 1.90 检查及实际桌面二进制构建通过。
- Windows x64 实例使用独立 profile 成功安装、授权、启用并创建实际 WebView。
  可访问性树确认 `https://synra-plugin.<digest>.localhost/index.html` 页面及真实 Vue 内容，平台 windows、应用版本 0.1.0。
- 对现有“运行诊断”按钮执行键盘操作，页面显示真实 Wasm 返回的 `@synra-plugin/starter`、
  插件 0.1.0、WIT 1.0.0、宿主 SDK 0.1.0、desktop、en 和浅色；请求记录为 `#1 ok 9.2 ms`。
  这是一次实际页面调用的观测，不作为分位数性能验收。
- 窗口截图分别报 `FrameArrived timed out` 和 `window capture timed out`；
  页面内容和真实交互已确认，像素级视觉排版仍未验收。
- 插件窗口的实际关闭已确认，随后关闭 owned 测试进程；按包清理不再使用的 App 与 daemon dev 产物，
  实际合计释放 8577937408 字节（约 8 GiB），保留依赖缓存、日志和独立验收 profile。
  页面可访问性证据保存在 App `src-tauri/target/ui-smoke-accessibility.txt`。

新增文件通过 API guard（461 个文件）和设计文档链接检查，README 变化通过 App 8 项 shapes 检查。
本机解锁后阶段签名提交已完成，见[提交记录](./2026-10-28-signed-plugin-phase.md)；等待期间没有启动第二个签名进程。

此结果不代表电脑选中文本、手机剪贴板、电脑打开链接或触摸板已经完成，也不代表手机真机验收通过。
