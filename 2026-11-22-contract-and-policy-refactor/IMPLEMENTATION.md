# 实施记录

记录日期：2026-11-23。计划范围仍是本目录 01–11，以下阶段结果不代表整体完成。

2026-11-27 更新了 04、07、11 的联合证据，并开始 01 的事件契约。详细提交及验证见第三批、第四批。

## 当前进度

| 项目 | 状态 | 已有证据 / 剩余工作 |
| --- | --- | --- |
| 01 事件投影 | Core 契约已交付 | 已有独立信封、设备字段投影和权威匹配修复。宿主定向流、授权撤销、协调器和联合回归待接入 |
| 02 传输状态 | 待实施 | 仍需宿主事实、终态查询及真实 worker 两端验证 |
| 03 上传会话 | 待实施 | 仍需并发状态转换、所有权移交及会话失效清理 |
| 04 testkit | 基础管线已交付 | 授权、方法支持、平台分支、拒绝和错误守卫已有共同场景。事件、传输模型及网络范围的联合验收继续补齐 |
| 05 版本策略 | 待实施 | 仍需调用链统一和 semver 边界测试 |
| 06 平台按键 | 待实施 | 仍需结构化模型及原生平台验证 |
| 07 错误边界 | App 与 testkit 自动验收通过 | 真实 serde 错误及非法输入经实际 façade 或 testkit 调用路径验证，完整宿主 UI 展示归 11 |
| 08 Wasm 分类 | 待实施 | 仍需结构化 trap 分类及真实组件验收 |
| 09 注册表 | 已实施并自动验收 | 共享源值、严格配置读写、有限原因、预览与发现页已接入；最终宿主联合验收见 11 |
| 10 宿主环境 | 待实施 | 仍需宿主事实与布局分离及响应式回归 |
| 11 联合验收 | 部分实施 | 已联合验证错误、授权和注册表夹具。事件与终态的消费、真实传输两端和实体平台场景待补齐 |

## 第一批：错误边界和真实序列化

core 已签名提交并推送 `09c99013df371c5502caf7e0eccb05504d8df020`。该提交将 SuggestedAction 的 payload 字段按 camelCase 序列化，并提供 `synra-ipc/examples/error_contract.rs`，从真实 DTO 和 serde 输出所有 ErrorCode 的重试策略、六类建议动作、IPC 参数和插件 trace/数组诊断。

发现的是实际契约缺陷：旧 serde 属性只重命名枚举 tag，`plugin_id`、`device_id`、`trace_id` 与 JS 公开类型不一致。新增 Rust 客户端形状测试先失败，修改序列化属性后通过；未保留两套未发布格式。

App 工作区已建立 `src/ipc/errors`，由宿主 KernelClient 与插件 namespace 共用。对 unknown 校验完整必填字段和权威错误码，未知失败转为 internal；明确缺 API 才返回 unsupported。IPC params 与 details 保留在明确结构中；诊断有深度、节点和文本上限，并对敏感字段和 URL 凭据脱敏。读取对象时不调用 getter、toString 或任意序列化钩子，校验时提取快照，防止二次读取已变化属性导致再次抛异常。建议动作按类型复制，保留其业务 payload。

`tests/fixtures/error-contract.json` 来自上述 Rust producer，`pnpm run shapes` 会重新执行 producer 并比较语义输出，fixture 只存于 App。测试从 namespace 的 bridge rejection 和宿主命令入口消费该样例，不用手写 TS 对象代替 Rust 生产者。

### 已执行的验证

| 命令 / 用例 | 实际结果 | 证明范围 |
| --- | --- | --- |
| core `cargo test --locked -p synra-protocol -p synra-ipc --quiet` | 退出 0；244 + 125 个单元测试，另外 13 个集成测试 | 协议和 IPC 的当前测试范围 |
| core 对 protocol、ipc lib 与 error_contract example 的 Clippy，`-D warnings` | 退出 0 | 修改的 Rust 边界及 producer |
| App 新增错误边界用例 | null、畸形字段、未知 code、getter、代理、循环对象等先暴露缺陷，修复后通过 | 不可信错误归一化及真实桥适配 |
| App `pnpm test` | 退出 0；74 个文件、616 个测试 | 当前前端完整测试集 |
| App `pnpm run lint` | 退出 0；0 error，297 条既有 UnoCSS 排序 warning | 保留现有 lint 强度，未将 warning 计为无警告通过 |
| App `pnpm run build` | 退出 0 | façade 重建、vue-tsc 与生产 Vite 构建 |
| App `pnpm run shapes` | 退出 0；18 项检查 | 包括真实错误输出与提交 façade freshness |
| core `cargo run --locked -p synra-error-guard -- .` | 退出 0；1090 个文件，无违规 | 错误边界守卫 |
| core `cargo run --locked -p doc-links -- . ../design-docs` | 退出 0；153 个文件，无问题 | 本地 Markdown 链接 |

注册表批次随后完成 App 原生 392 个测试和默认 Tauri 编译，core 的兄弟 App 构建与 MSRV 1.88.0 检查也已通过，后续结果见下一节。07 的 testkit 部分、完整宿主交互及其余计划不能由这些测试推断为完成。

### 环境与失败尝试

- App 第一次生产构建因链接的 `@synra/theme` 旧 dist 中的数字 zIndex 配置而失败。检查实际解析主题后确认源码已正确；执行 `pnpm --filter @synra/theme build` 重建本地依赖后，App build 通过，未添加样式绕行分支。
- 本机是 Windows ARM64，PATH 中的 LLVM 路径不存在，ring 因缺 clang 构建失败。使用官方 LLVM 23.1.6 ARM64 压缩包解压到系统临时目录，仅向验证进程的 PATH 加入 bin；未修改仓库 Rust 工具链、系统安装或全局 PATH。随后原生测试与默认构建通过。
- `cargo fmt --all --check` 报告大量既有格式差异。本批不批量重写整个仓库，改动文件分别格式化与检查；该全量命令的失败不记为通过。
- core deletion guard 原来按旧平铺文件名认所有者，误报已移动到目录的 sandbox/plugin_files 模块；兄弟 App 检查仍找旧仓库名。现已按模块所有权及真实兄弟目录修正，并验证相似目录名不能越权；历史失效链接单独修复，检查保持启用。

## 第二批：注册表源、业务原因及验证基础

core `bf0ee5d6d5bf9fe7522a8f167d775a11384fcb84`、JS `8212e179a82857355fad6fba9c842f7f25758d09`、App `d6f04e6d3a0f80b10c2e704646980d651c689f96` 已签名并推送。App 的初始错误边界提交为 `97db1bf77877a9c1b75eb50989a86fcc5d9285dc`。

### 配置读写与责任

| 入口 | 实际处理 |
| --- | --- |
| 设置页 → App config_set | 保留当前单源 string、多源 array 的写入形式；提交前严格验证整个值，坏元素拒绝整份 patch |
| App 安装、预览、搜索、更新 | transport::bases 返回 core RegistrySource，保留镜像顺序；不再维护 App 私有 Base 结构或手工拆 scheme/port |
| App 本地更新发现 | 使用相同 RegistryConfig 判断是否配置源，数组和损坏配置不再因 String-only 读取而被当作缺失；本地版本策略继续归 05 |
| App 更新缓存、core registry_setting | 保留原始序列化值作为来源身份，用于拒绝配置变更前的旧结果；这一用途不代替领域解析 |
| daemon plugin_settings → Sources | CLI 数组写入与空列表取消配置沿用 daemon 的数据库；验证和更新下载复用同一个 core 源值 |
| daemon catalog | 既有命令使用自身 RegistryConfig 的默认索引策略，不消费 plugins.registry；本批没有把它与 App 配置文件合并 |

core 的 RegistryConfig/RegistrySource 拒绝非字符串项、空白镜像、无效 URL、凭据、查询/片段及远程 HTTP，区分空配置与坏配置；错误携带镜像位置。npm tarball 通过标准 URL 的 origin 比较，默认端口、主机大小写和 IPv6 的等价写法不再依赖文本前缀；静态文件的查询参数使用独立的下载路径和网络授权规则。

RegistryProblem 定义 notConfigured、invalidConfiguration、searchUnsupported，放在 IpcError.details.registry。原始网络 code、retryable、参数和动作继续传到页面；HTTP 408、429、5xx 保持可重试，搜索端点 404 与静态源明确给出 searchUnsupported。发现页通过有限状态展示，设置页保留坏项并显示具体字段和位置；切换配置会使旧查询失效。搜索不受支持时可按当前输入的插件 ID 进入已有预览和安装确认流程。

预览内核的两个搜索入口复用同一实现，并按相同原因拒绝缺配置或坏配置。`registry_contract.rs` 使用真实解析器、DTO 和 serde 产生 14 组设置向量及三类业务错误，保存于 App 夹具目录；shapes 重跑该 producer。组件用这些 Rust 样例验证英文/中文、替换 message key、设置入口和按 ID 预览。真实回环 HTTP 验证 404/408/429/503/403、镜像回退、空结果，以及完整性失败后下一镜像零连接。

### 本批自动验收

| 命令 / 检查 | 实际结果 |
| --- | --- |
| core registry、ipc 测试 | 26 + 126 个测试，退出 0；protocol 的本批 244 个测试另见第一批 |
| daemon runtime::updates | 18 个真实更新场景，退出 0 |
| core registry、ipc、daemon 的 lib/examples Clippy | `--no-deps -- -D warnings`，退出 0 |
| guards 单元测试及 all-targets Clippy | 14 个测试，退出 0；Clippy 无警告 |
| core guards | 12 项通过，包含真实兄弟 App 构建及 MSRV 1.88.0 workspace all-targets 检查 |
| core error guard / doc-links | 1093 个源码文件无违规；154 个 Markdown 文件零链接问题 |
| App 全量前端 | `pnpm test --maxWorkers=1`：77 个文件、645 个测试通过；搜索别名补充回归随后 51 个相关测试通过 |
| App Rust | `cargo test --locked --no-default-features --manifest-path src-tauri/Cargo.toml --quiet`：392 个测试通过 |
| App Tauri | `cargo check --locked --manifest-path src-tauri/Cargo.toml` 退出 0 |
| App lint / build / shapes | 退出 0；包含 vue-tsc、Vite、注入脚本和 19 项 shapes。lint 保留既有 UnoCSS 排序 warning；no-default-features Rust 留有既有 record_received unused import warning |
| JS pnpm test | protocol 3、theme 85、testkit 64、脚本 76 个测试通过；现有 testkit 的通过数不代表 04 的语义已经修复 |
| JS check / lint / protocol build | 退出 0；消费者类型、文档片段及 15 项 shapes 通过，40 个 Rust DTO 字段与镜像一致 |

基础检查修复分别提交 core `6665cbe6da48793f6a465f9ae9fbc2735052ff8b` 与 JS `187552cc9f4fe9036b056f57b6ac9c190eab7343`。JS 守卫与测试共享递归 Rust 源遍历，修复六个嵌套 DTO 省略字段突变用例的失败；同批仅修复主题中三处 lint 格式问题。没有放宽断言或添加忽略名单。

本机同时运行 Rust 和多个 DOM 测试时，默认并发暴露多项 5 秒超时。改用一次验证进程的 `--maxWorkers=1` 后通过，测试超时阈值和仓库测试配置未修改。成功响应形状测试显式配置预览注册表；不把未配置产生的错误放行为空数组。Windows 接受的回环套接字继承非阻塞模式，测试服务端已显式改为有读写截止时间的阻塞连接。

## 第三批：testkit 执行策略与授权一致性

App 脱敏修复 `b7c014d`、JS 执行管线 `46ca207`、Core 授权生产者 `d95ae8b`、App 客户端一致性 `93e13dc` 均已签名并推送。

testkit 由一个执行管线记录实际调用、检查方法支持、授权和平台支持，再执行默认模型或响应 stub。支持查询与构造快照共用 profile，真实 App 缺失的方法明确返回 unsupported。设备列表保留离线设备，设备选择与 HTTP 成功需要明确 fixture，传输状态按 ID 保存。授权拒绝使用正式剪贴板权限。前 9 项回归在旧实现上有 8 项失败，迁移后全部通过。测试迁移说明见 [25 §2.4](../25-js-monorepo.md)。

Core 运行实际 CapabilitySet 生成 28 条向量。App 的本地检查和 testkit 同时消费该快照，验证单段及多段通配、限定符、跨域拒绝和语法错误。App shapes 重新运行生产者。共同场景也核对剪贴板的授权和平台组合、方法可用性以及 Core 错误码的全部重试策略。平台分支由测试 Bridge 模拟，原生效果单独验收。

| 验证 | 实际结果 |
| --- | --- |
| App 脱敏修复 | 前端 77 文件、649 个测试通过。截断 URL、Authorization 与 FTP 凭据回归已覆盖 |
| JS 执行管线 | 全量 237 个测试通过，其中 testkit 73 个。lint、check、testkit 构建及 Node 实际导入退出 0 |
| Core 授权生产者 | capability 91 个、IPC 126 个单测通过。生产者运行和示例 Clippy 退出 0 |
| App 联合场景 | IPC 28 文件、281 个测试通过。全量 78 文件、684 个测试通过 |
| App 发布门槛 | 392 个无默认功能 Rust 测试与默认 Tauri check 退出 0。生产 build、20 项 shapes、lint 均通过。lint 有 298 条既有 UnoCSS 排序警告，Rust 保留既有未使用导入警告 |

全量前端第一次使用 forks 时有 2 个 worker 启动超时。线程池以相同的逐文件隔离和单 worker 设置复跑，通过完整测试集合。生成的 façade 已提交。

## 第四批：插件事件契约起点

Core `1706aad` 修复 `HandlePattern` 丢弃通配符后面字面段的问题。回归先失败，修正后 245 个协议单测、13 个集成测试通过，Clippy 退出 0。`7faca0e` 定义独立的 `PluginEventEnvelope` 与设备投影，`device.changed` 只序列化配对且未屏蔽的设备，并保留离线成员。IPC 128 个单测、producer 和 Clippy 退出 0。两个提交均已签名推送。宿主定向投递与最后一次退订、窗口销毁和撤权的清理仍待实现。

JS `0a1df4e` 已签名推送。公开设备类型引用协议镜像，testkit 按 Core 的点分模式检查语法和深度，为同一个回调的每次订阅保留独立的清理。JS 全量 239 个测试通过，其中 testkit 75 个。lint、消费者类型、15 项 shapes 和文档示例通过。Core 事件生产者已能生成真实信封和 12 条匹配向量，接入 App 消费回归和持续一致性检查归下一步。

## 仍需交付

继续推进 01 的各页授权和订阅生命周期，再接 02、03 及 04 的事件和传输场景。05、06、08、10 继续实施。11 仍缺事件与终态共同场景、真实 worker 两端和平台操作证据。
