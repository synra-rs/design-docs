# 本地插件事件服务

## 契约与实现

旧 WIT 的 `event-subscribe` 没有可用的 guest 回调入口，且稳定回调目标与池化实例冲突。
现确定使用 manifest `handles` 与必需的 `lifecycle.on-event(event-json) -> outcome`，
移除动态 headless 订阅 import；UI JS 事件流不受此决策影响。

两个原生宿主共用 core 的 EventService：guest 发射时入有界队列，单个 worker 经普通
PluginSet 调用另一个当前实例，不在 host call 内重入 PluginSet。入队成功与处理完成分开，
不自动发送远端回复、不重试。上下文身份和 request/trace 由宿主提供，guest 数据保持在 payload 下。

入队与执行都重查持久化启用、信任、活动版本、精确 manifest consent 和具体事件能力。
执行证明绑定 runtime 与装载代次，同版本卸载重载也使旧发射任务失效。
接收其他包的事件不等于冒充其他包；发射仍强制属于自己的完整包命名空间。

单条 payload 为 64 KiB，队列 128 项、8 MiB，等待期限 5 秒；因果深度 8，
单链发射和投递各 32 次。每插件事件与 handler 各 16 项，缓存 32 个装载代次，
诊断保留 64 条，不保存 payload。处理计数接入已有 RingRecorder，标签只有有限的结果分类。
服务随所属 runtime 停机；队列清空，已执行的调用按原预算结束，服务不复用于其他 runtime。

payload 使用受限 Draft 2020-12，实际检查类型、required、数值边界、嵌套、数组及额外属性。
schema 位置拒绝引用、format、正则，字符串类型要求 maxLength；schema 与 payload
均限制深度、节点数、编码大小。属性名与 const 数据不作为 schema 关键词解释。
依赖 jsonschema 固定版本且关闭默认外部解析功能；上游实现见
[官方仓库](https://github.com/Stranger6667/jsonschema)。

## 验证证据

- 插件 API 的 188 个测试通过，包含真实 payload 校验、Unicode、数值等价、编码预算及敏感值不回显。
- 首批 daemon 5 个实际组件与文件 SQLite 测试通过，覆盖跨 guest 投递、上下文与 trace、撤权、同版本重载和循环发射。
- 首批 core 5 个测试通过，覆盖因果并发预算、过期、资源回收及有界诊断与指标。
- 随后补充显式停机、runtime 销毁和临时数据库释放顺序，以及阻塞接收端的队列压力测试；daemon 的 6 个真实链路测试复验通过，临时目录均显式关闭成功。
- workspace 的 all-targets 严格 Clippy 与 WIT 六份生成物只读检查通过，文档链接及 API 边界扫描通过。
- workspace 回归的 core 823 个单测与性能预算测试通过；发现的三个失败测试目标修正后定向复验通过（daemon 265、host 383、SDK 6），其余通过结果继续复用。
- 桌面无窗口和默认 Tauri 配置的 all-targets 严格 Clippy 均通过；无窗口 Rust 337 个测试通过，含真实组件投递与无执行上下文的冒充拒绝。
- 公开文档的中英作者说明与事件页已同步，ready 和站点构建通过；保留在文档工作分支，不触发 main 的发布流程。
- 架构守卫 12 项通过，包含 core 声明 MSRV 1.88 的全 workspace/all-targets 检查、实际 sibling app 构建与 122 份文档链接检查。
- App 默认配置在声明 MSRV 1.90 上的 all-targets 检查通过；Android ARM64 的 core 与启用 Wasm 的 host 检查通过，使用既定 NDK。
- 新 WIT 对应的实际 Rust SDK guest 已重新构建，真实 host 的单项 opt-in 测试通过；CLI 的两项真实组件门禁通过，包含新脚手架构建、验证与失败重建保留原产物。

## 清理与未完成范围

通过 Cargo 的包级 dev 清理移除桌面应用旧产物 3.6 GiB，保留依赖缓存、其他 profile、源码和运行数据。
core 阶段测试和最终定向 lint 通过后，再清理 daemon 与 CLI 的 dev 包产物 21.7 GiB；
清理前确认无构建进程并核对 Cargo dry-run，只删除 Cargo 管理的目标目录内容。
阶段结束后清理不再使用的 host dev 包产物 3.5 GiB，本轮合计约 28.8 GiB；
其他包的依赖缓存、release 产物和运行数据保留。
新事件测试显式确认 worker Arc 释放和临时 SQLite 目录关闭成功，避免仅等待 handler 完成就尝试删除仍打开的文件。

本轮仍未完成远端 device.send、系统事件源与 UI 事件的生产接线、跨插件动作、请求回复和进行中宿主 IO 的统一取消。
ManifestContext 的旧可选 Implementation 布尔校验也仍需清理；它不能替代实际组件的 WIT 验证。
Starter、Touchpad 的完整需求见 [29](../29-official-plugins.md)。两个插件的 SDK/CLI git pin 已统一到
`9c7c78b9788e5357ccb4537c9ff1f8d3abedea58`，补齐必需的 guest 入口；无 handles 时明确拒绝调用。
两仓格式、严格 native/Wasm Clippy、测试（Starter 3、Touchpad 18）、MSRV 1.88、实际组件构建和 verify 均通过。
这些适配不代表手机界面、跨设备链路或原生鼠标输入已完成。
M2、D1/D2 及整体目标继续开发中，此记录不是完成声明。

## 阶段提交

所有提交使用英文单行 subject、空 body 和 GPG 签名；已推送对应开发分支。

- core：`9c7c78b`，本地事件服务、payload 校验、WIT、daemon 与实际链路测试。
- app：`1ba0db4`，桌面事件服务接线与真实组件测试。
- docs 工作分支：`57c2c62`，中英作者说明与事件命名空间；未推送 main。
- Starter：`60b0ebd`，SDK/CLI 固定版本与 guest 入口。
- Touchpad：`fdd1e0d`，SDK/CLI 固定版本与 guest 入口。
