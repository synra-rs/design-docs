# Rust Guest 作者链路与实际导入检查

日期：2026-10-26；对应 [ADR-024](../20-adr.md)、[SDK](../15-plugin-sdk.md)、
[构建决策](../13-plugin-authoring.md) 和 [M2](../19-roadmap.md)。

## 从真实编译暴露的问题

原 SDK 顶层 Outcome、LogLevel 与 DeviceTarget 来自宿主模型，
与 wit-bindgen 生成 Host 函数真正使用的类型不同；原夹具还调用了生成 enum 不存在的 ok/err 方法。
现在直接公开同一份 WIT 生成类型、Guest traits 与 export 宏，SDK 不再直接依赖宿主 manifest、协议、签名或 serde；生成器的编译依赖不作为 Guest 运行依赖。
公开类型测试将函数指针赋给真实生成 Host 函数类型，避免只证明宿主模型可构造。

原夹具没有参与编译，其注释把 unsafe、目标和外部 wasm-tools 都当作阻塞。
实际修正导出后，保留 forbid unsafe 即可编译，没有放宽工作区安全规则；
目标安装也可完成，不能继续把旧推断当作环境结论。

wasm32-wasip2 真实产物虽是组件，却引入 wasi:cli、wasi:io、wasi:clocks 等接口，
与默认不提供 WASI 的宿主不相容，实例化明确失败。
改为 wasm32-unknown-unknown core module，使用 CLI 固定的 wit-component 读取 SDK 嵌入元数据并生成组件，
无需外部 wasm-tools、WASI adapter 或直接系统能力。

真实组件还暴露了宿主只报告接口容器名的缺陷：PluginSet 的授权检查需要实际函数名，
类型接口本身不应被当作能力请求。
宿主现在从准确版本的 Host 接口提取函数并转成闸门的规范名称，
只跳过值类型，未知导入和资源仍然保留以便拒绝；真实资源导入有独立负例。

## CLI 构建与失败语义

原模板只有旧式 C 导出，没有实现 WIT world；构建也只运行 Cargo，未将实际输出写到 manifest 路径。
新模板实现 lifecycle、share 和 actions traits，最后调用 SDK export 宏；
二进制名对齐 synra-plugin，crate 名仍是 synra-plugin-cli。

构建从 Cargo compiler-artifact JSON 中选择当前项目的 cdylib，不猜库名或 target 路径，
读取受体积限制的 core module，编码与校验后才原子替换 manifest artifact。
拒绝越界目标、缺失或多个候选产物、无效模块与缺少导出；旧输出不能让失败构建通过。

真实模板测试保留生成 Rust 源，只以 Cargo patch 解析本地未发布 SDK，
使用不同库名与独立 target 目录，经过实际 CLI 构建、PluginSet 装载与调用。
成功编译但没有插件导出、Rust 编译失败两种重建都返回失败，旧组件字节保持原样。
SDK 尚未发布，不能把测试中的本地依赖覆盖描述为已验证 crates.io 分发。

## 测试与检查

SDK 真实 Guest 测试经过公开导出、组件编码、生产后端与 Host 闸门；
验证读取返回、Unicode 参数、匹配阶段拒绝已授权的副作用、执行允许、
重复调用清理、结构化失败和实际 Host 次数。
CLI 调用方测试还经过生产 PluginSet，确认缺少授权时在装载拒绝，
具备所需授权后可以激活与往返字符串。

普通单测不递归启动 Cargo；显式 rust-guest-tests 门禁读取刚构建的 fixture，
缺少输入即失败，不使用 ignore。
CI 新增相同构建与测试步骤，没有改变工作流触发或发布编排；
本机实际执行不作远端 runner 已完成的声明。

本阶段验证结果：

| 范围 | 结果 |
| --- | --- |
| core workspace | `cargo test --locked --workspace` 全部单测、集成与 doctest 通过，含 core 815、daemon 256、Host 356、CLI 151、SDK 6 项单测 |
| 严格 lint | workspace 全部目标 Clippy 通过，另对 SDK、CLI、Host 的作者链路特性全部目标执行严格 Clippy |
| 独立真实引擎 | `cargo test --locked -p synra-plugin-host --features wasm-runtime`：356 通过、0 忽略 |
| Rust 作者链路 | 直接从 CI YAML 提取步骤并在本机 Git Bash 执行，SDK fixture release 构建、Host 真实 Rust 用例 1 项及 CLI 真实构建/调用用例 2 项通过 |
| 最低 Rust | core 守卫的完整 workspace 全部目标检查、Rust 1.88 的 SDK fixture Wasm 目标检查、应用最低版本全部目标无默认特性检查均通过 |
| 应用后端 | 无默认特性 325 项测试与默认 Tauri 检查通过，未修改前端或 IPC 形状，复用未受影响的相关验证 |
| 平台与独立工具 | Android 内核与真实引擎宿主交叉编译、无默认特性的独立作者 CLI 编译通过 |
| 架构与文档 | 12 项架构守卫通过，含实际调用方构建、最低版本、文档链接与数量；本轮 Rust 模块格式与 diff 空白检查通过 |
| CI 语法 | actionlint 对修改后的检查工作流通过，仅新增作者链路步骤与时限，没有修改发布 YAML 或触发条件 |

阶段全量门禁完成后未再修改业务逻辑；
文档证据补全后只复核文档链接、数量和空白，不重复全量 Rust 测试。
Android 此处仅证明交叉编译，不证明链接或设备运行；本轮没有运行发版或远端 CI。

## 当前边界

本次完成 Rust headless 作者链路与宿主装载边界；
完整本地安装、用户同意与真实操作系统动作、完整 CLI verify 二进制检查、UI 构建、
作者高层封装、SDK 独立分发时的 WIT 资源与路径验证、移动设备运行和性能验收仍待完成。
匹配纯度由宿主逐次调用的上下文强制，SDK 类型或 lint 提示不能代替安全边界。
历史的构建与纯度推断保留于 [原观察](./rust-authoring-observations.md)；
整体 D1–D5 与各里程碑状态仍见路线图，不将本阶段标记为完整目标完成。
