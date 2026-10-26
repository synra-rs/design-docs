# 本地与远程意图的完整结果

日期：2026-10-26；对应 [路线图 M2](../19-roadmap.md)、[页面错误传输](../08-ipc-and-frontend.md#23-错误传输)及 [Starter 的请求回复](../29-official-plugins.md#22-首批功能)。

## 实际缺口

生产 decide 与 accept_remote 把非成功 Wasm outcome 缩减成 failed 等状态字符串，丢失插件消息、详情及运行预算。
重复回复把持久化 JSON 结果取为字符串，导致对象、数组、数字及 null 变成空文本；失败重放重新拼接错误也会丢失结构。
接收端还重新生成 trace，导致两端诊断不能按同一操作关联。

## 当前契约

共享宿主层的 CallOutcome::into_result 与 reported_failure 统一分类，页面不再另写一份插件错误码映射。
主动拒绝保留原始 pluginCode 和可选 pluginDetails；时间、fuel、内存及 trap 使用各自执行错误类别和预算字段。
SynraError 增加可选 JSON details；Rust 内部使用 Box，避免把较大的 JSON 枚举内联到每一个 Result。
第一次内联实现触发 Clippy result_large_err，报告 Err 至少 152 B；改为冷错误路径的间接存储后严格检查通过。
IPC 在原诊断中保留 context 和 traceId；插件 UI 的 SynraError 类型也可读取 details。
TypeScript 协议文件是由形状脚本核对的手写镜像，不是自动生成文件；本轮同步镜像并运行消费者类型检查，WIT 不变。

远程 InvokeResult 改为带 status 的成功或失败变体：

```json
{"status":"succeeded","result":{"selected":"文本"},"elapsedMs":2}
```

```json
{"status":"failed","error":{"code":"unsupported","message":"refused by guest","retryable":false,"traceId":"operation trace","details":{"pluginCode":"notImplemented"}},"elapsedMs":2}
```

成功值只解析一次；非 JSON 的成功文本保留为字符串。失败必须有完整 SynraError，未知变体和矛盾字段直接拒绝。
Board 直接接收 Result<JSON, SynraError>，不再组合成功标志、文本和可选错误，也不为入库再次序列化及解析结果。
不保留旧的 ok/result/error/errorCode 格式分支；本地 Decided 的 result 为 JSON 值，error 为结构化错误。
接收工作保留原请求 trace；重复请求沿用已记录 trace，不接受新的信封覆盖它。发送端用本地记录的 trace 归一化错误上下文。
完成结果通过原有 Board 及数据库记录复用；重复回复和恢复回复不重新执行动作，保留原执行耗时。

## 验证方式与环境处理

新组件以生产 WIT 编码，在真实 Wasmtime 中执行；经过生产 decide、accept_remote、apply_result 和两个 Board。
覆盖对象、数组、null、数字、JSON 字符串、空文本、普通文本及结构化拒绝，分别执行本地和远程路径。
Stateful guest 在第二次执行时直接 trap，确保重复请求及数据库恢复的回复不能靠重新运行 guest 得到相同答案。
接收侧 SQLite 真实关闭重开并恢复 Board，再投递原请求；改变重复信封的 trace 也不能覆盖首次操作。
这些是进程内双 Board、真实 Wasm 和文件数据库恢复测试，不是两个独立宿主进程或手机的整体验收。

首次定向命令漏掉 --lib，额外链接集成测试时遇到磁盘耗尽，随后 lib 链接也报告 no space on device。
确认自有进程终止、target 位于当前工作区且为忽略的 Cargo 产物后，补齐缺失缓存标识并审查 dry-run；
按 synra-core 包清理 30,026 个文件、约 43.5 GiB，保留依赖缓存、源码及运行数据，改用 --lib 后新回归通过。
随后在仍有空间时主动清理 synra-app 开发 profile 的旧产物：10,094 个文件、约 11.6 GiB；不清理 release profile 或依赖缓存。
后续按用户要求持续整理不用的产物，并在大型链接前检查剩余空间，不再等到不足才清理。

## 阶段门禁

Core 全工作区 3,601 项测试、全目标严格 Clippy 通过；新真实 Wasm 与 SQLite 重放回归包含在全量中。
第一次全量运行揭示旧 API 守卫的错误发布前提，按 [D5](./2026-10-26-unpublished-api.md) 修正。
随后全量发现 daemon 的等待结果仍断言空字符串，现按新契约断言 JSON null；定向验证后全量通过。
测试总数与上轮不同，包含删除过时兼容性规则测试及新增结果链路回归，不是漏跑工作区。
真实 Rust SDK guest 定向测试另有 1 项通过；SDK 与 WIT 未变，复用原有源码构建产物，不重新构建 guest。
12 项仓库守卫通过，包含 Core MSRV 1.88 全目标、App 实际调用方及 Core、设计库和公开文档库共 120 份文档的链接。
Android aarch64 的 Core 与真实 Wasm host 编译通过；此项只证明可编译，不代表设备或移动端验收。

App 无默认特性全量 335 项 Rust 测试、全目标严格 Clippy、默认 Tauri 编译及 MSRV 1.90 全目标编译通过。
前端 347 项测试、ESLint、vue-tsc、Vite 构建及 8 项 shapes 检查通过。
Vite 仍提示部分压缩 chunk 超过 500 kB；本轮未调整阈值或宣称解决包体性能问题。
JS 协议与 client 定向构建、完整类型检查、公共消费者类型检查、14 项 shapes、文档示例类型检查及 ESLint 通过。
Rust 修改范围与新增模块符合 rustfmt，复用源码及环境未变的已通过检查；Cargo 构建始终限制单个 job。
验证日志保留在忽略的 target 目录；只进行开发提交，没有发版、tag 或部署。

## 剩余边界

本轮修复已有意图执行链的结果与诊断，不代表 device.send、事件订阅回调、跨插件动作授权或普通插件请求回复已全部实现。
进行中执行的真正取消、撤销竞态及平台副作用的结果不确定性仍待统一处理。
数据库记录尚未落盘时的崩溃窗口及原生副作用的 exactly-once 保证不在本轮验收范围内。
Starter 手机界面、选区捕获及触摸板输入会话继续按 [29](../29-official-plugins.md) 推进，不据此声明 M2–M6 完成。
