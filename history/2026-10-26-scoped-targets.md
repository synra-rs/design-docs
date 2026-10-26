# 设备目标与跨插件动作的窄范围授权

日期：2026-10-26；对应 [10 §2.8](../10-capability-and-sandbox.md#28-设备目标与跨插件动作的范围)、[M2](../19-roadmap.md) 和 [Starter](../29-official-plugins.md)。

## 发现与契约

device-send、action-invoke 与 action-declare 原先把 WIT 的族标签直接作为每次调用所需权限。
授予某台设备或某个插件动作仍会因缺少整个函数族而被拒绝；action 的单个星号也不能表达插件与动作两个限定段。
本轮将导入可用性与具体操作授权分开，不授予比 manifest 或用户决定更宽的权限。

设备发送每次校验 device:send:<规范设备 UUID>，不把界面名称或 @self 解析成另一种权限身份。
目标不是规范小写带连字符 UUID 时报告 badArgument；事件仍使用完整插件命名空间并检查所有权。
动作调用每次校验 action:invoke:<完整 @scope/name>:<动作名>，相同包名但不同 scope 不能互相授权。
支持具体插件所有动作、任何插件某个动作及完整双星号范围等显式授权，沿用共享 capability 匹配器。
WIT 的 action-invoke 标签改为 action:invoke:*:*；单限定段 action:invoke:* 不能覆盖实际调用，不做旧标签兼容。
动作注册的族标签改为 action:declare:*，每次检查 action:declare:<实际名称>；不把裸 action:declare 解释为全动作授权。

动作名限定为 1–64 个 ASCII 字母、数字、点、下划线和连字符，不能将请求参数中的星号、冒号或路径当成权限模式。
名称校验由 synra-plugin-api 拥有，manifest、HostCalls 的运行时声明与跨插件调用复用。
名称错误为 badArgument；权限不足为 capabilityDenied，详情保留所需具体范围。
没有该函数族权限时仍先报告 capabilityDenied；范围授权先于速率预算及宿主效果，拒绝进入审计。

## 实现与验证

共享 HostCalls 在实际参数上授权，host_surface 的 permits 与 undeclared_imports 只判断导入范围可用性。
WIT 单源同步六项派生资源；本轮无 ABI 签名变化，没有手改生成文件。
实际设备发送、动作声明和动作调用通过生产 WIT 元数据组装成组件，经真实 Wasmtime 及类型化 linker import 进入 HostCalls。
同一实例切换当前 LinkerSession，验证窄授权成功、不同设备或插件/动作拒绝、撤销后拒绝及显式通配范围。
记录宿主到达次数，拒绝必须不产生宿主效果；这些回归的效果端是记录宿主，不声称已执行真实设备投递或跨插件服务。

单元回归另覆盖设备别名和非规范格式、动作通配符及分隔符注入、跨 scope 同名包、外来事件所有者、速率为零时仍先检查权限，以及审计分类。
manifest 回归经过真实 JSON 读取及 schema 级校验；CLI 集成回归使用实际 verify 进程、真实组件字节和窄范围 manifest，并拒绝缺少动作限定段的授权。

## 阶段门禁与清理

Core 全工作区 3,611 项测试与全目标严格 Clippy 通过，包含新增的 2 项 manifest/名字回归、5 项范围闸门回归及 3 项真实 Wasm 回归。
WIT 六项派生资源的只读检查通过；SDK guest 从当前源码重新构建，真实宿主回归 1 项、CLI 的 guest 装载与脚手架构建回归 2 项通过。
Android aarch64 的 Core 与真实 Wasm host 编译通过；没有据此声明设备端运行已通过。
12 项仓库守卫通过，包括实际 App 调用方、Core MSRV 1.88 全目标、121 份文档的链接与可检验计数。
App 无默认特性全量 335 项 Rust 测试、全目标严格 Clippy、默认 Tauri 编译与 MSRV 1.90 全目标编译通过。
前端及 JS 类型契约未变，复用上一阶段的前端测试、lint、构建和类型检查，不重复全量 UI 测试。
新增 Rust 模块及本轮修改范围按固定工具链格式化；没有整体格式化旧文件，Cargo 重型任务保持单个 job 串行执行。

按用户要求在仍有空间时检查并整理旧产物；发现 16 个已不用的 host 测试变体，仅二进制和调试符号即约 4.5 GiB。
按筛选结果和显式文件清单删除都被工具策略拒绝，未执行这两次删除；改用 Cargo 管理的包级清理。
确认相关进程结束、target 归属和忽略规则，审查 dry-run 后清理 synra-plugin-host 的 dev profile：20,673 个文件、20.3 GiB。
该包本轮的新缓存也一并清除，后续需要时重建；保留其他依赖包、release profile、源码、运行数据及本轮验证日志。
清理之后没有改动已验证的源码，不因清理生成缓存而重复运行通过的全部检查。

## 剩余工作

两个生产宿主的 device_send 与 action_invoke 效果后端仍明确拒绝未实现功能；本轮只完成它们共用的范围闸门与名字约束。
生产事件投递、订阅回调、普通请求/回复、运行时注册与 manifest 声明集合的核对、跨插件执行深度/超时预算、手机 UI 和触摸板原生输入仍按路线图推进。
更换调用会话的撤销回归不等于已经中断执行中的网络或系统副作用。
未发版、打 tag 或部署。
