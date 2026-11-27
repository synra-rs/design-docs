# 实施记录

记录日期：2026-11-23。计划范围仍是本目录 01–11，以下阶段结果不代表整体完成。

2026-11-27 更新了 04、07、11 的联合证据，并接通设备事件、外发状态及进度、独立上传会话和自定义事件。详细提交及验证见第三至第十批。

## 当前进度

| 项目 | 状态 | 已有证据 / 剩余工作 |
| --- | --- | --- |
| 01 事件投影 | 宿主投影与自动验收已接入 | 设备、传输及自定义主题通过会话定向投递。退订、撤权和恢复已覆盖。自定义事件缺口按订阅报告，实体平台验收归 11 |
| 02 传输状态 | 外发终态查询及定向进度已接入 | Core 状态机、确认字节、App 有界终态及资源归属、JS 类型已接入，worker 接管入口已统一。真实 TCP 回环收发通过，实体设备验收归 11 |
| 03 上传会话 | 已接入 | 核对窗口会话、串行提交 offset，上传文件和预算一并移交给 worker。取消、过期、启动残留及权限失效有自动覆盖 |
| 04 testkit | 已实施并自动验收 | 授权、方法支持、平台分支、拒绝、错误、设备事件、自定义主题、传输状态和网络范围均有共同场景 |
| 05 版本策略 | 统一语义已接入 | 清单、SDK 最低版本、安装、同步、注册表及 App/daemon 更新共用 SemVer 解析和优先级规则 |
| 06 平台按键 | 已实施并通过 Windows 实机验收 | 解析、授权、支持查询及失败清理有自动覆盖。其余桌面环境的实际效果归 11 |
| 07 错误边界 | App 与 testkit 自动验收通过 | 真实 serde 错误及非法输入经实际 façade 或 testkit 调用路径验证，完整宿主 UI 展示归 11 |
| 08 Wasm 分类 | 待实施 | 仍需结构化 trap 分类及真实组件验收 |
| 09 注册表 | 已实施并自动验收 | 共享源值、严格配置读写、有限原因、预览与发现页已接入；最终宿主联合验收见 11 |
| 10 宿主环境 | 待实施 | 仍需宿主事实与布局分离及响应式回归 |
| 11 联合验收 | 部分实施 | 已联合验证错误、授权、注册表、设备投影及传输进度和终态。真实 TCP 两端及跨页面传输状态已通过，实体平台场景待补齐 |

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

## 第五批 · 设备事件定向投递

Core `df4cb59`、JS `4a15235` 和 App `d9cd62d` 已签名推送。Core 登记 `event.poll`、`event.resync` 和 `device.changed` 主题。JS 为 `event.subscribe` 和 `device.onChange` 提供可选 `onError`，testkit 以同一主题及 `device:list` 授权驱动设备变化，两种实现均隔离单个回调的异常。

App 通过现有 `plugin_ui_rpc` 投递。宿主从 Tauri 取得真实调用窗口，并校验窗口、令牌、插件授权和会话。设备事件与 `device.list` 共用已配对、未屏蔽设备的投影，保留离线项。每页最多登记 64 个模式和一个等待中的读取，读取至多等待 20 秒，积压最多 8 帧。缺序时由宿主重读设备快照；窗口销毁、会话失效和最后一次退订会释放队列并唤醒读取。

管理事件定向给主窗口。插件 capability 继续限制通用事件监听，设备投影沿已授权的 RPC 返回。该方式使用有界等待，沿用移动桥已有的调用链。脚本守卫从参数类型识别 Tauri 注入的 `Webview`，继续核对前端实际传入的字段。Core 生产者的真信封和 12 条模式向量进入 App 测试及 shapes，每次检查都会从 Rust 重建。

| 验证 | 实际结果 |
| --- | --- |
| Core IPC | 128 项通过，lib Clippy 以 `-D warnings` 通过 |
| JS | 全量 240 项通过，其中 testkit 76 项。lint、类型、15 项 shapes 和文档示例通过 |
| App 前端 | 79 文件、701 项通过，契约守卫修正后独立复跑注册表 5 项通过。生产 build 和 21 项 shapes 通过，lint 保留 298 条既有 UnoCSS 警告 |
| App Rust | 395 项无默认功能测试与默认 Tauri check 通过，包含 3 项会话投影回归 |
| App 原生 ACL 与嵌入资源 | 使用 `--features tauri/custom-protocol` 执行 `commands::plugin_ui::tests::native_acl`，两项通过 |

原生 ACL 用例分别检查桌面解析结果及 Android 权限清单的静态解析，确认插件窗口可调用页面 RPC，管理快照与通用事件监听受限。普通开发构建的 ACL 用例通过，但同组既有 CSP 用例找不到嵌入 HTML，因为 Tauri 配置了开发服务器。启用原生嵌入资源后整组通过。Android 设备运行仍待验证。设备快照恢复已覆盖缺序，连接中断后重连、传输进度和插件自定义事件仍待接入。

## 第六批 · 宿主保留外发传输终态

Core `e9c26a5`、JS `72bdcfa` 和 App `8d59a77` 已签名推送。

Core 从经过校验的确认位图和发送分块计划计算确认字节数，包含重传和短尾块。外发状态从等待到传输中，再由 worker 提交完成、取消或结构化失败，重复确认和终态后的更新无法增加 revision。App 在同一注册表登记 worker、记录归属并保存终态。管理视图继续读取活动项。终态在本次宿主进程中最多保留 512 条，保留期 10 分钟。

插件页面的 `status` 每次向宿主读取。宿主校验当前发送授权及发起插件，重开页面可以读到同一插件的近期终态。不存在、已淘汰或属于其他插件的 ID 得到相同的 `transfer_not_found` 原因。客户端按宿主确认的字节数展示进度。JS 镜像包含分块计数、revision、结束时间及类型化失败。

外发取消只发信号，由 worker 提交终态和移除临时资源。Core 的实际发送协议回归覆盖重传、短尾块、错误摘要、取消和完成确认。App 的内存链路测试核对发送端和接收端的完整数据、完成与取消记录，后台 worker 的拨号失败用例验证新页面可以查询终态。实体设备或真实网络两端的联合用例仍待执行。

| 验证 | 实际结果 |
| --- | --- |
| Core | 传输 42 项、发送协议 7 项、IPC 129 项通过。全库 Clippy 报告 60 项既有错误，编译器定位均在此次未修改的文件 |
| JS | 240 项通过，含 testkit 76 项。lint、消费者类型及 15 项 shapes 通过，JS 与 Rust 对照 41 个 DTO |
| App | 全量前端 79 文件、702 项，原生无默认功能 396 项，默认 Tauri 编译，以及生产 build、21 项 shapes 和 lint 通过。lint 保留 298 条既有 UnoCSS 排序警告 |
| App 定向复跑 | IPC 299 项、最终 Rust 传输 61 项通过 |

JS 的递归测试由 Windows 上 Rollup 的 Node 诊断子进程拖住。终止了该次验证进程，分别运行剩余包及脚本测试并核对各组结果，依赖与测试配置保持原样。传输的实时订阅仍需接入 01 的会话投影，旧 RPC 的可选 Arc 分支随上传所有权改造收敛。

## 第七批 · 上传会话与 worker 文件所有权

Core `1ca9082`、JS `f9ca99b` 和 App `f8a7f43` 已签名推送。

Core 管理上传状态与预算准入。开始时登记插件、会话、声明大小和目标。每个上传用独立锁保护检查 offset、写入文件与提交计数，重复或乱序追加返回 `expectedOffset`。部分写入失败会关闭并清理。准备接管时同时核对已提交偏移和文件实际长度。

App 的宿主预算为 256 MiB、32 个槽位，单文件最多 64 MiB，每个页面 2 个槽位，每个插件 4 个槽位。开始后返回 1 MiB 块上限。客户端按宿主返回的大小切分，逐块校验偏移确认。临时文件及其预算凭证由同一个对象持有，worker 接管后继续占用预算，直到文件清理结束。清理失败时记录上传 ID，保留预算和文件位置，交给现有维护循环重试。

窗口关闭、插件授权版本变化和退出调用会话清理。宿主用单调时钟检查 2 分钟上传空闲期。在已取得桌面单实例锁或移动端单进程启动时，清扫专用 spool 中可识别的 `upload-UUID` 一文件目录。服务按已知文件路径、非递归地清理。

公开 `transfer.send()` 使用分块会话和统一 worker 入口。原单次 base64 RPC 的调用点只有 Rust 测试，已移除该内部操作及 4 MiB 分支。`transfer.finish` 在后台任务接管后返回，同一个入口覆盖生产和测试。

并发回归用屏障或受控写入验证重复偏移、跨上传独立写盘、部分写入失败和 abort 与 worker 移交竞争。另有真实 RPC 测试覆盖授权变更后无需页面再调用即可回收、无权拒绝和会话重开后查询 worker 终态。预算测试包括接管时继续计费、worker 拒绝后归零、过期和启动残留清扫。

| 验证 | 实际结果 |
| --- | --- |
| Core | 上传状态 2 项、IPC 129 项通过，IPC Clippy 通过，错误守卫扫描 1098 个文件无违规 |
| JS | 240 项通过，lint、消费者类型、15 项 shapes 及文档示例通过，48 个 DTO 对齐 |
| App Rust | 406 项通过，含并发上传、实际磁盘清理与后台接管。默认 Tauri 编译和两个可选插件测试入口编译通过 |
| App 前端 | 79 个文件、703 项通过，其中 8 项覆盖上传和外发状态。生产 build、21 项 shapes 和 lint 通过，保留 298 条既有 UnoCSS 排序警告 |

## 第八批 · 会话定向传输进度与快照恢复

Core `3242f31`、JS `0085251` 和 App `25331ac` 已签名推送。宿主按 `file:transfer:send` 与发起插件身份投递 `transfer.progress`，只传递公开状态。`transfer.onProgress` 使用传输 ID 登记，随后读取初始状态。通用主题订阅和单任务订阅使用同一页面读取循环。JS 按 `revision` 合并查询、实时事件和恢复快照，已退订或关闭页面的回调不再触发。

宿主检测管理事件队列漏帧时要求客户端恢复。页面序列缺口、宿主恢复信号及可重试 IPC 失败会触发 `event.resync`，由宿主返回授权范围内当前状态和序列水位。可重试读取至多退避两次，失效会话走结构化错误回调。恢复使用当前有效会话与仍在保留期内的记录。

Core 夹具包含真实 `transfer.progress`、快照恢复和宿主漏帧信号。App 和 testkit 对照同一 `peer_aborted` 终态及其 `too_large` 原因。App Rust 用真实后台 worker 拨号失败验证终态投影、归属隔离和恢复。前端首次复测在退订后等待另一轮读取，改为在退订前确认最后一帧回调已完成。

| 验证 | 实际结果 |
| --- | --- |
| Core | IPC 129 项通过，lib 和生产者 Clippy 以 `-D warnings` 通过，错误守卫扫描 1098 个文件无违规 |
| JS | 240 项通过，其中 testkit 76 项。lint、消费者类型、15 项 shapes 和示例编译通过 |
| App Rust | 408 项无默认功能测试及默认 Tauri 编译通过。嵌入资源模式下的 Windows ACL、CSP 及 Android 权限清单检查共 2 项通过 |
| App 前端 | 79 个文件、706 项通过。最终定向复跑 28 项，生产构建和 21 项 shapes 通过。lint 保留 298 条既有 UnoCSS 排序警告 |
| 文档 | 154 个 Markdown 文件，零链接问题 |

## 第九批 · TCP 回环收发与页面重开回归

App `fef06c0` 增加 Windows ARM64 上的实际 TCP 回环场景，收发宿主各用独立状态，端口由系统分配。发送端走插件上传 RPC 和实际 worker，接收端用 Core 的生产连接处理器和默认接收策略。用例上传 1,048,593 字节，逐次核对宿主确认偏移，并在收件箱核对落盘内容。发送端终态确认五个协议分块及全部字节，`transfer.status` 与定向事件一致。

完成后关闭原页面，旧令牌的轮询被拒。新页面沿用已安装插件身份并取得新流 ID，重新订阅和恢复得到同一个传输终态。测试端点是同机两个回环网络端点，实体设备 WebView 和局域网仍需另行验收。

定向 Rust 3 项、无默认功能 Rust 409 项和默认 Tauri 编译通过。前端 79 个文件、706 项，生产构建、21 项 shapes 及 lint 通过，保留 298 条既有 UnoCSS 排序警告。文档检查 154 个文件、零链接问题。

## 第十批 · 自定义事件定向投递与缺口通知

Core `9c19b45`、JS `e41c83d` 和 App `2df9c83` 均为签名提交。本地事件沿用执行、声明和发射授权校验，远端事件沿用入站校验。Core 事件 worker 复核来源及期限后通知宿主观察者。App 观察者通过弱引用使用当前宿主，沿独立队列和页面 RPC 投递 `_plugin.<full-plugin-id>.<topic>`。公开信封包含序列、时间、完整事件名和 JSON 对象。精确订阅在登记时检查具体 `event:subscribe:<event>`，通配订阅对每次具体投递复核。每次投递也重新校验页面会话及当前权限。

客户端恢复时带上已消费的序列。宿主重读状态型快照，并报告哪些现存逻辑订阅错过了自定义事件。`onGap` 收到其所处的序列范围，新事件继续走原订阅。

真实 Wasm 组件通过生产宿主发出已声明事件，获准的页面取得完整主题和 JSON payload，另一插件的通配订阅无法接收。队列溢出只通知受影响的订阅，设备快照仍可读取。App 和 testkit 使用同一 Rust 序列化夹具验证精确授权、通配筛选和公开载荷。

| 验证 | 实际结果 |
| --- | --- |
| Core | IPC 130 项和事件服务 5 项通过。IPC lib 与生产者 Clippy 通过，错误守卫扫描 1098 个文件无违规 |
| JS | 240 项通过，其中 testkit 76 项。lint、消费者类型、15 项 shapes 和文档示例通过 |
| App Rust | 定向会话回归 8 项、无默认功能 411 项和默认 Tauri 编译通过 |
| App 前端 | 定向 30 项，全量 79 个文件、708 项通过。生产构建与 21 项 shapes 通过。lint 保留 298 条既有 UnoCSS 排序警告 |
| 文档 | 154 个 Markdown 文件，零链接问题 |

JS 的默认脚本测试进程在 Windows 上长时间没有输出，终止后以单 worker 线程池复跑，76 项全部通过。App 首轮前端全量在并行编译期间有一项设置页 5 秒超时。编译结束后单独复跑，708 项全部通过，沿用原有超时设置。

## 第十一批 · 统一插件版本语义

Core `aaf1efb` 提供统一的版本值与业务比较入口。App `03df068` 将本地和注册表更新检查、插件包打开流程迁入同一规则。两个提交均已签名。

Core 的 `PluginVersion` 和 `PluginVersionReq` 封装工作区已有的 SemVer 库。清单用同一规则验证插件版本、SDK 最低版本和依赖范围。旧清单测试先复现了数字预发布前导零和主版本溢出被接受的行为，迁移后均被拒绝。已扫描 starter 和 touchpad 清单，当前的 0.2.6 和 0.2.0 符合规则。

比较只使用 SemVer 优先级。构建元数据作为精确版本身份保留，同优先级的不同字节继续遵循同步策略。`NeverDowngrade` 和 `KeepNewest` 对不可比较的版本给出拒绝原因。`SourceWins` 与人工确认仍按既有直接选择语义执行。手动替换须提高优先级，自动更新另外检查稳定版、主版本策略、签名及授权。

npm 默认安装采用发布者的 `latest`，精确版本、显式标签、范围请求有各自的选择规则。静态索引缺少 `latest` 时选择最高优先级，优先级相同按完整版本字符串升序选第一项。App 和 daemon 在版本无效时报告检查失败，保留上次成功的检查记录。App 打开插件包时显示版本错误原因。

| 验证 | Windows ARM64 实际结果 |
| --- | --- |
| Core | plugin-api 185、plugin-host 346、registry 26、core lib 779、daemon 更新 19 项通过。启用 `wasm-runtime` 的三 crate lib Clippy 以 `-D warnings` 通过 |
| App Rust | 无默认功能 412 项和默认 Tauri 编译通过。真实回环注册表覆盖同优先级与无效版本 |
| App 前端 | 全量 79 个文件、709 项通过。生产构建和 21 项 shapes 通过。lint 保留 298 条既有 UnoCSS 排序警告 |
| 守卫 | 错误守卫扫描 1098 个文件无违规，文档链接检查 154 个文件零问题 |

plugin-host 默认特性下的严格 Clippy 报告既有的 `epoch::ticks_for` 未使用导入。启用使用该导入的 `wasm-runtime` 后通过。App 首轮契约检查在 Cargo 编译排队时超时，编译结束后单独复跑，21 项全部通过。

## 第十二批 · 对齐插件出网范围

Core `aeef961` 收敛原生范围与传输入口。JS `c3a98c9` 对齐 testkit 的授权及 HTTPS 检查。App `9828400` 复用共享客户端缓存，验证页面请求及原生 RPC。三个源码提交均已签名。

testkit 的回归先复现了一个主机只有 GET 授权时，额外的族通配符仍能放开 POST。Core 和 testkit 统一为从明确指定该主机的授权中匹配方法，该请求现在返回 `capability_denied`。Core 在请求及每次重定向前解析实际 URL。App 和 daemon 共用按插件缓存客户端的实现，授权变化后读取新的范围，并发预算继续按插件共享。

App 的原生 `plugin_ui_rpc`、注入 façade 和 JS testkit 使用同一份 18 项用例。检查绝对 URL、实际主机、端口、用户信息、IPv6、方法限定、授权拒绝以及 HTTPS 策略。原生用真实持久化授权调用宿主，确认获准的回环地址在 IP 校验阶段被拦截。testkit 用明确的响应夹具验证该请求已通过范围检查。重定向另用 Core 的实际 HTTP 回环端点检查后续目标和方法。

| 验证 | Windows ARM64 实际结果 |
| --- | --- |
| Core | fetch 22、plugin-host 346、daemon 插件宿主 15、网络守卫 22 项通过。fetch 和 plugin-host lib Clippy 在 `wasm-runtime` 下以 `-D warnings` 通过。错误守卫扫描 1099 个文件无违规 |
| JS | testkit 78 项，其余包与脚本 164 项通过。lint、消费者类型、15 项 shapes 与文档示例通过 |
| App Rust | 无默认功能 409 项与默认 Tauri 编译通过。用例覆盖真实 RPC、权限替换后的客户端复用和现有 IP 策略 |
| App 前端 | 80 个文件、727 项通过。生产构建和 21 项 shapes 通过。lint 有 298 条既有 UnoCSS 排序警告 |
| 文档 | 154 个 Markdown 文件，零链接问题 |

首次插件宿主全量测试暴露了 WIT 合同夹具依赖族通配授权。给测试请求补上明确的主机授权后通过。Core 和 App 的锁文件继续使用各自已锁定的 `url` 修订版。

App 前端两次使用默认 5 秒超时的全量运行分别有 3 项和 1 项挂载超时。涉及的三个文件使用默认超时单独复跑，28 项通过。最终用 `pnpm test --maxWorkers=1 --pool=threads --testTimeout=15000 --reporter=dot` 完成全量验证。

## 第十三批 · 结构化平台按键

Core `f1ab82c` 提供类型化按键、平台执行和清理逻辑，更新了 SDK、CLI 与宿主使用的 WIT 说明。App `e698533` 在 Wasm 宿主入口接入解析和按键诊断。两个提交均已签名。

App 和 daemon 在 Wasm 宿主边界将参数解析为 `KeyChord`。主键与修饰键从同一组值解析，重复修饰键、未知主键、空段和旧平台语法返回 `badArgument`。WIT 的参数仍是字符串，使用规范语法后通过同一宿主入口。已核对当前工作区内的 starter 和 touchpad 源码，旧 Windows 热键写法的迁移对照见 [SDK 指南](../15-plugin-sdk.md)。

Windows 和 macOS 从按键值生成原生键事件，Linux 从相同值生成 X11 键名。宿主选区复制沿同一执行路径。执行将主键和修饰键的按下与释放分开记录，按逆序清理，并保留每项释放错误及最初的按下错误。支持查询同时检查宿主策略及当前桌面后端可用性，具体调用继续通过插件能力门禁。

Windows ARM64 启动专用 Win32 接收窗口。用户聚焦后，测试程序用当前 `synra-platform` 的 `CommandPlatform` 发送按键，接收窗口从实际系统消息队列捕获以下顺序。

| 组合 | 按下和释放 |
| --- | --- |
| `Ctrl+C` | Ctrl↓ C↓ C↑ Ctrl↑ |
| `Ctrl+V` | Ctrl↓ V↓ V↑ Ctrl↑ |
| `Ctrl+Shift+S` | Ctrl↓ Shift↓ S↓ S↑ Shift↑ Ctrl↑ |

最初的回归测试复现了旧检查接受 `Ctrl++`。实机测试首轮没有前台焦点，随后发现托管接收程序与原生库架构不匹配。换用 ARM64 原生接收程序并由用户聚焦后，全部序列通过。

| 验证 | 实际结果 |
| --- | --- |
| Core | `synra-platform` 101 项、插件宿主 346 项、SDK 6 项及 1 项文档测试、daemon 插件宿主 16 项通过。平台 lib 的 Clippy 以 `-D warnings` 通过。WIT 产物检查通过，错误守卫扫描 1103 个文件无违规 |
| App Rust | 无默认功能 410 项和默认 Tauri 编译通过。新增回归核对宿主将无效组合映射为 `badArgument` |
| App 前端 | 80 个文件、727 项通过。单 worker 前端测试使用一次性的 `--testTimeout=15000`。生产构建和 21 项 shapes 通过。lint 有 298 条既有 UnoCSS 排序警告 |
| macOS 与 Linux | `synra-platform` 在 `aarch64-apple-darwin` 和 `x86_64-unknown-linux-gnu` 上的 Rust 类型检查通过。首次 macOS 检查在绑定 SQLite 的 C 编译阶段失败，随后仅在交叉检查命令中设置 `LIBSQLITE3_SYS_USE_PKG_CONFIG=1`，使用系统 SQLite 绑定。应用内实际效果归 11 |
| 文档 | 154 个 Markdown 文件，零链接问题 |

## 仍需交付

08、10 继续实施。11 仍缺实体设备间的网络收发及其余平台操作证据。
