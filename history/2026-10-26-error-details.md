# Wasm 与插件页面的错误详情

日期：2026-10-26；对应 [12 的返回边界](../12-wasm-host.md#10-当前实现与验证边界)、[08 的错误传输](../08-ipc-and-frontend.md#23-错误传输)及 [Starter 调试需求](../29-official-plugins.md#22-首批功能)。

## 发现与契约

上一阶段的命令错误已进入 HostCalls，但 Wasm 导出将 Outcome::Err 转为 CallOutcome::Failed 时仅复制 code 与 message。
即使 guest 原样转发了宿主 details，引擎也会丢弃；页面 self.invoke 与部分直接宿主调用的错误适配再次丢失可选详情。
真实组件回归先复现 details 丢失，再修复生产转换；没有通过为测试单独组装成功答案来绕过缺口。

CallOutcome::Failed 与 Scripted::Failed 均显式保存 details: Option<String>；引擎按 WIT 原样传递，不解析文本、不把空字符串当作缺失。
五种 outcome 导出覆盖成功、无详情、空字符串、JSON 对象、JSON null 和非 JSON 文本，重复调用同时验证 post-return 清理。
插件主动报告失败仍不是失常，不降低健康度。

页面共用 plugin_error 适配器，IPC 顶层分类保持 ErrorCode，原始码放入 details.pluginCode。
details.pluginDetails 在源详情存在时写入；有效 JSON 解码为 JSON 值，其他字符串原样保留。
未知插件码仍可诊断，notImplemented 明确分类为 unsupported；嵌套详情即使包含 code 也不能覆盖 IPC 分类。
本次没有变更 WIT、SDK 导出或 Rust IPC 错误结构。
前端移除把 JSON details 误写成字符串的手写 IpcError，直接引用协议生成类型；不再单独维护 params、details 和 actions 的字段形状。

## 验证方式

App 新增可审查的 WAT fixture，测试通过生产 self.invoke、真实 SQLite 动作声明、PluginSet 和真实 Wasmtime 引擎返回拒绝。
覆盖缺失与空详情、JSON null、非 JSON、Unicode 对象、数组、自定义错误码、notImplemented 及详情中的同名 code；检查最终 IPC JSON 和插件健康度。
直接 Host CallError 使用同一适配器，另有字段保留与分类不被详情覆盖的回归。
页面注入层经 self.invoke 的回归保留同一个结构化拒绝对象；这是可注入 bridge 的测试，不替代真实 WebView 或手机验证。
WAT 组装器仅为 App 开发依赖，固定为 Core 锁文件中已使用的版本；生产依赖和引擎版本不变。

## 阶段门禁

Core 全工作区 3,613 项测试、全目标严格 Clippy 通过；真实 Rust SDK guest 的定向测试另有 1 项通过。
SDK 与 WIT 源码未变，复用上一阶段从源码构建的 guest 产物执行本轮已修改的宿主回归；未重复构建该 guest。
12 项仓库守卫通过，包含 Core MSRV 1.88、App 调用方、118 份文档链接与可检验计数。

App 无默认特性全量 334 项 Rust 测试、全目标严格 Clippy、默认 Tauri 编译与 MSRV 1.90 全目标编译通过。
前端 347 项测试、ESLint、vue-tsc、Vite 构建及 8 项 shapes 检查通过；注入脚本前后 SHA-256 相同。
Vite 仍提示部分压缩 chunk 大于 500 kB；本轮记录该包体提示，没有调整阈值或宣称完成包体性能优化。
新增 Rust 模块完整检查 rustfmt，已有 Rust 文件仅格式化本轮修改范围；构建任务限制单个 Cargo job。
日志保留在各仓库忽略的 target 目录；本轮没有发版、打 tag 或部署。

## 剩余工作

远程 intent 当前仍把非成功 outcome 缩减为状态字符串；本轮没有宣称其错误码、消息和详情已贯通。
后续应统一 local/remote intent 的结构化错误、去重回复和重启后结果，避免再在宿主里增加各自映射。
Starter 两端请求回复、手机 UI、选区捕获与触摸板原生输入继续以 [29](../29-official-plugins.md) 为准；本轮不等于插件功能或 M2 完成。

本节保留该阶段状态；后续本地及远程意图、重复回复、数据库恢复和共享分类修复见[意图结果记录](./2026-10-26-intent-outcomes.md)。
