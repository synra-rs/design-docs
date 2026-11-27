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
| 06 平台按键 | 已实施，通过 Windows 实机及 Alpine X11 验收 | 解析、授权、支持查询及失败清理有自动覆盖。Linux 事件在 ARM64 虚拟机中验证，macOS 原生效果待登记 |
| 07 错误边界 | App 与 testkit 自动验收及 Windows 原生 WebView 验收通过 | 真实 serde 错误经实际 façade 与 testkit 验证，Starter 的参数诊断与撤权提示见第十六批 |
| 08 Wasm 分类 | 已实施并自动验收 | 真实组件核对 fuel、epoch、内存上限、独立 trap 和复用。App RPC 与 daemon 远程执行保留一致诊断 |
| 09 注册表 | 已实施，通过自动验收及 Windows 原生 WebView 验收 | 严格配置读写、静态源预览与 npm 重试见第十七批。后台读取和更新检查顺序分别见第十九、二十及二十二批 |
| 10 宿主环境 | 已实施，待移动端实机验收 | Windows WebView 实测布局与关闭，Chrome 验证移动 UA 下的浏览器预览。Android 返回已有自动化和 ACL 回归 |
| 11 联合验收 | 本机联合检查通过，待设备实测 | Windows 真实插件页面已验证诊断和撤权。真实 TCP 两端及跨页面传输状态已通过，移动端、实体收发及其他原生效果待登记 |

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

## 第十四批 · Wasm 失败分类与调用预算

Core `4a25b84` 将失败分类集中到 Wasmtime 后端的 `failure.rs`。App `19845ea` 补充真实 Wasm 组件的页面 RPC 验收。两个提交均已签名。
调用按错误类型识别 fuel、epoch 和宿主内存上限。非 result 型 WIT 宿主错误的代码与消息保留在 trap 诊断中，其中的预算关键词不会改变分类。

宿主超限由限制器直接产生类型化 trap。线性内存声明或分配器施加更严或相等的限制时，`memory.grow` 仍可返回 `-1`。真实组件先遇到内存拒绝、再触发除零，结果保留除零 trap 和拒绝诊断。复用测试在同一个组件实例上依次执行可恢复拒绝、无显式限额的有限计算与增长、独立 trap，验证 fuel、deadline、默认内存边界和拒绝记录的重置。

缺省的每块线性内存增长上限为 16 MiB，本次显式配额可以替换该上限。实例化错误单独返回。App 页面 RPC 和 daemon 远程执行分别用真实组件验证共享错误转换，两条路径都按插件配额返回 `plugin_trapped` 和带 `limit` 的 `memoryExceeded` 诊断。

| 验证 | Windows ARM64 实际结果 |
| --- | --- |
| Core | 插件宿主 395 项、调度 39 项、daemon 246 项通过。插件宿主 lib 在 `wasm-runtime` 下的 Clippy 以 `-D warnings` 通过 |
| App Rust | 无默认功能 411 项通过，默认 Tauri 编译通过 |
| App 前端 | 80 个文件、727 项通过。生产构建与 21 项契约检查通过。lint 保留 298 条既有 UnoCSS 排序警告 |
| 守卫 | 错误守卫扫描 1104 个文件无违规。154 个 Markdown 文件零链接问题 |

前端首轮在 Rust 编译期间有一个 settings 测试文件未能启动 worker。该文件用默认测试超时单独复跑，16 项通过。最后在 Rust 编译结束后使用 `pnpm test --maxWorkers=1 --pool=forks --testTimeout=15000 --reporter=dot` 完成全部 727 项。

## 第十五批 · 宿主事实、视口与系统返回

App `bd83c92` 已签名。主窗口从既有设备快照读取 Rust 提供的平台事实，插件注入与 Wasm 宿主使用同一来源。启动时修正旧设备资料中的平台，保留设备 ID 与名称。底部导航、图标 rail 和展开侧栏各由视口控制。Android 主窗口注册 Tauri 系统返回事件，退出权限限定在 Android 主窗口。设备和设置的初始化各自驱动，迟到的原生注册会释放监听。

Windows ARM64 使用嵌入本轮生产前端的 Tauri 测试构建和独立数据目录。测试构建临时放宽窗口下限，产品最小窗口仍为 1024×560。直接测量 WebView 的实际视口，标题栏始终存在，页面均无全局横向溢出。

| 实际视口宽度 | 布局及窗口结果 |
| --- | --- |
| 1100px | 展开侧栏 240px。WebView 模拟 5 个触点，仍为桌面宿主 |
| 826px | 图标 rail 96px，显示桌面标题栏 |
| 666px | 底部导航覆盖视口宽度。WebView 原报 10 个触点，探针临时覆写为 0 时也保留桌面标题栏。点击关闭后原生窗口隐藏，进程继续运行 |

App 的 Rust 无默认功能 412 项和默认 Tauri 编译通过。带 `tauri/custom-protocol` 的原生资源和 ACL 用例 2 项通过。主窗口及返回逻辑的重点回归 17 项通过，前端全量 81 个文件、736 项通过。生产构建和 21 项 shapes 通过，lint 有 297 条既有 UnoCSS 排序警告。

默认开发协议的原生资源测试缺少嵌入的 HTML。构建生产前端后启用 `tauri/custom-protocol` 复跑，原生两项通过。前端首轮全量有一个独立的终端关闭焦点测试失败。该文件以默认测试超时单跑 10 项通过，随后在原生构建结束后用单 worker 的 forks 复跑全量通过。

## 第十六批 · 联合验收基线复核

Windows ARM64 核对 Core `4a25b84`、App `bd83c92` 和 JS `c3a98c9`。三个源码仓库与各自的上游主分支一致。

| 检查 | 本轮结果 |
| --- | --- |
| Core `cargo run --locked --quiet -p guards` | 12 项通过，包括兄弟 App 编译、Rust 1.88.0 MSRV 和 198 个设计及公开文档的链接。文档计数中 4 项可核对且一致，3 项无对应检查入口 |
| Core `cargo run --locked --quiet -p matrix` | 4 项通过、7 项跳过、0 项失败。Android 协议与传输、桌面无引擎插件宿主、独立 CLI 编译通过 |
| Core `cargo test --locked -p synra-ipc -p synra-registry -p synra-plugin-api` | 341 项通过 |
| JS `pnpm test`、`pnpm run check`、`pnpm run lint` | 共 242 项测试通过，其中 testkit 为 78 项。消费者类型、15 项契约检查和文档示例通过 |
| 独立 Starter | `15ed578` 的 0.2.6 包使用锁定的 SDK 和 CLI 修订版 `e0c5481` 构建。清单、完整 WIT 组件、10 个实际导入和 3 个 UI 归档文件共 4 项检查通过 |
| App 真实包回归 | `cargo test --locked --no-default-features --features official-plugin-tests --manifest-path src-tauri/Cargo.toml --lib` 全量 413 项通过，覆盖真实包安装、激活、正常调用、撤权和页面会话更新 |
| App 完整检查 | 默认 Tauri 编译通过。前端 81 个文件、736 项通过，生产构建及 21 项契约检查通过。lint 为 0 错误、297 条既有 UnoCSS 排序警告 |

本机为 Rust 1.99.1 安装了 `aarch64-linux-android` 标准库。Android 持久化、核心和启用 Wasmtime 的插件宿主三个配置缺少 NDK C 工具链。其余四个跳过项是矩阵预留的无插件核心、wasmi、AOT 和无 TCP 配置。

真实包回归首轮在激活处复现授权夹具漂移。Starter 0.2.6 已导入 `platform-open-path`，App 的验收授权清单尚未包含它。App `1b7cb2f` 补入这项明确的测试授权，撤权用例从同一清单移除 `device:list`，复跑覆盖其余授权及拒绝行为。该签名提交也校准了两个 Starter 验收入口的适用范围。

Windows 真 WebView 使用同一 Starter 0.2.6 包及独立 App 数据目录。通过生产管理 IPC 安装、授予已知能力、启用和打开页面。页面实际渲染 Wasm 诊断，宿主形态为 `desktop`。Wasm 的参数拒绝沿注入 façade 返回 `invalid_argument`，`details.context.pluginCode` 保留 `badArgument`。通过生产 IPC 撤销 `device:list` 并重新打开插件，旧 WebView 退出，新页面返回 `capability_denied` 并显示「设备列表未获授权」提示。

首轮验收只给主 WebView 配置调试参数，同一 WebView2 环境创建插件页时返回 `ERROR_INVALID_STATE`。复跑在独立检出中为两个窗口传入一致的本地调试参数，保留生产 IPC、插件权限和页面会话路径。本机常规可执行文件恢复后通过哈希比对。见 [WebView2 环境选项](https://learn.microsoft.com/en-us/microsoft-edge/webview2/reference/win32/webview2-idl?view=webview2-1.0.4258.31)。

前端全量使用单 worker 的 forks 池和一次性的 15 秒测试超时。最终文档链接检查为 198 个文件零问题，错误守卫检查 1104 个文件零违规。

## 第十七批 · Windows 注册表界面验收

Windows ARM64 使用第十六批的独立 Tauri 构建与独立配置目录，覆盖 [09](./09-registry-state.md) 的生产设置、注册表 IPC 和插件发现界面。嵌入的生产前端与原生命令来自 App `bd83c92`，截至 `1b7cb2f` 的差异只有验收测试与贡献说明。Core 和 JS 验收基线分别为 `4a25b84` 和 `c3a98c9`。注册表 HTTP 端点在本机回环端口提供固定响应，并记录搜索及预览请求。

| 场景 | WebView 和原生命令结果 |
| --- | --- |
| 空源列表 | 搜索显示配置提示，点击「Open settings」进入插件设置。注册表服务未收到请求 |
| 非法源 | 在设置页保存非法地址时，界面显示 `plugins.registry` 和绝对 URL 要求。通过原生配置命令提交包含数字的镜像数组时，返回 `invalid_argument` 和 `invalidConfiguration`，镜像索引为 1。两次拒绝后已保存的列表仍为空 |
| 静态源 | 从设置页保存静态源后，搜索说明该源的能力限制。点击按 ID 预览后，服务只收到静态索引请求，实际界面显示 Starter 0.2.6 清单 |
| npm 源 | 从设置页保存 npm 源后执行三次搜索，依次得到空结果、HTTP 503 后的重试入口，以及点击重试后显示的 `@local/registry-proof` 条目 |

探针首轮在外部更新源后立刻提交搜索，未观察到空结果提示。最终复跑从设置页提交源，确认设置已保存后进入插件发现页。验证结束后恢复独立配置目录中原有的源和界面语言。

## 第十八批 · 移动 UA 浏览器预览

在 Windows ARM64 上以 Chrome `158.0.8127.38` 的独立浏览器配置，加载 App `1b7cb2f` 对应的生产前端。使用 Android 移动 UA 和 5 个触点，在同一页面依次将实际视口设为 390、826 和 1100 CSS px。

三种宽度分别显示 390px 底部导航、96px 图标栏和 240px 展开侧栏。从底部导航点击设置能打开设置页面。三次均保持浏览器预览形态，未显示原生标题栏，也未调用浏览器全屏 API。未观察到页面横向溢出、未处理的 JavaScript 异常或原生 API 错误。该批验收覆盖 [10](./10-host-environment.md) 的移动 UA 浏览器条目。

## 第十九批 · 注册表搜索与设置并行

App `a6ccc2b` 已签名并推送。Core `4a25b84` 与 JS `c3a98c9` 是本批所用基线。

在旧版 Windows ARM64 Tauri 测试构建中，回环 npm 源收到搜索请求后扣住 HTTP 响应。随后提交的 `config_get` 直到释放该响应才完成。搜索和设置命令共享窗口线程，导致发现页等待慢镜像时也阻塞了设置。

分页入口 `plugins_catalog_search` 和简化入口 `plugins_request_registry_search` 共用一个后台阻塞执行路径。搜索参数、序列化和错误映射沿用原有契约。使用两个回环源复跑，分别验证两个入口在旧 HTTP 响应仍挂起时完成设置读取、保存新源，并从新源取得条目。新版构建同时通过缺失配置、坏镜像、静态预览、npm 空结果和 HTTP 503 重试的原生回归。

新增的发现页挂载测试从设置 store 接收 `configUpdated`，验证较早源的迟到响应未进入列表，下一次界面搜索显示新源条目。新旧两套原生试验都使用隔离的 App 数据目录。常用的 Windows App 二进制通过哈希比对确认一致。

App 默认 Tauri 编译、无默认功能的 412 项 Rust 测试、21 项契约检查和 TypeScript 检查通过。前端首轮全量出现一项既有调试控制台关闭焦点测试失败。该文件以默认超时单跑 10 项通过，随后按单 worker、15 秒超时全量复跑，81 个文件、737 项通过。lint 为零错误、297 条既有 UnoCSS 排序警告。变更 Rust 文件的格式检查通过，仓库级检查报告其他文件的既有格式差异。

## 第二十批 · 注册表预览与更新检查

App `06c6f32` 已签名并推送。使用 Core `4a25b84`、JS `c3a98c9` 和独立 Starter `15ed578` 的 0.2.6 构建。

Windows ARM64 原生复现 `plugins_preview` 和 `plugins_check_update` 在等候回环源的元数据期间阻塞设置读取。两条命令现与搜索共用后台阻塞线程池。更新检查读取一次源配置，从该快照发起请求，保存检查结果时仍由数据库核对已安装版本与当前源。安装和更新共用本次解析的源列表处理元数据和下载。缓存采用同目录临时文件及原子替换。

实际 WebView 分别验证四个读取入口。回环旧源响应仍挂起时，设置可以读写，新请求即时使用新源。更新检查先保存新源的 Starter `0.2.8`，释放旧源的 `0.2.7` 后，旧检查返回 `invalid_argument` 并保留新源检查记录。两个搜索入口、缺失配置、坏配置、静态预览、npm 空结果和 HTTP 503 重试也已复测。

默认 Tauri 编译、变更文件格式检查、31 项注册表前端和 21 项契约检查通过。首次 Rust 验收未设置 `SYNRA_STARTER_PACKAGE`，官方包用例报告缺少路径。指定已构建的 Starter `dist` 后，启用 `official-plugin-tests` 的 Rust 全量 413 项通过。原生探针首次调试请求超时，延长探针的 IPC 等待后完整复跑通过。测试构建与常用 Windows App 使用不同的可执行文件和数据目录，常用可执行文件的哈希校验通过。

## 第二十一批 · Linux X11 按键事件

从 Core `4a25b84` 的检出快照和锁文件构建 `aarch64-unknown-linux-musl` 探针。Windows ARM64 上使用 Rust `1.99.1`、Zig `0.17.1` 的 C 编译器以及 Alpine 的静态运行库完成链接。在 QEMU `11.1.0` 的 Alpine `3.24.4` ARM64 虚拟机运行，使用 Linux `6.18.60`、Xvfb `21.1.26`、xdotool `4.20260303.1` 和 xev `1.2.7`。

探针调用生产 `CommandPlatform`。默认策略拒绝按键且支持查询不报告该动作。显式开启策略、未设置 `DISPLAY` 时返回 `Unsupported`。在 Xvfb `:101` 中聚焦独立 xev 窗口后，支持查询成功，接收队列的 14 个事件与下表完全一致。

| 组合 | xev 捕获顺序 |
| --- | --- |
| `Ctrl+C` | Control_L↓ c↓ c↑ Control_L↑ |
| `Ctrl+V` | Control_L↓ v↓ v↑ Control_L↑ |
| `Ctrl+Shift+S` | Control_L↓ Shift_L↓ S↓ S↑ Shift_L↑ Control_L↑ |

虚拟机内的源码编译较慢，最终使用宿主交叉构建的探针，并在来宾侧核对二进制 SHA-256。首轮 X11 就绪查询挂起，复跑使用单独的显示、关闭 GLX 的 Xvfb 和有截止时间的就绪查询。结果覆盖 Linux X11 后端的键事件投递和释放。

## 第二十二批 · 更新检查请求顺序

实现提交为 Core `80332f1` 和 App `3d9e069`。

Windows ARM64 原生复现同一镜像的两次更新检查乱序返回。后发请求先保存 Starter `0.2.8`，较早请求释放后曾把已保存的记录覆盖为 `0.2.7`。App 与 daemon 现在在读取元数据前从共享 SQLite 数据库领取请求顺序。保存时在同一条语句中校验当前安装版本、源配置及已有检查顺序。失败重试只更新同一次检查的记录。

原生复跑将隔离的第 17 版数据库迁到第 18 版，确认旧版本检查记录保留并得到默认顺序 0。随后让同源两次请求乱序返回，较早请求返回 `invalid_argument`，新记录仍为 `0.2.8`。另复跑了换源时旧检查被拒绝的场景。SQLite 回归通过两条连接分配顺序，覆盖系统时钟回拨后较早结果仍不能覆盖较新结果。

Core、SQLite 存储和 daemon 的 Rust 测试、错误边界检查、App 默认 Tauri 编译、413 项 App Rust 测试、31 项注册表前端测试和 21 项契约检查通过。Windows 原生探针使用独立的可执行文件和数据目录。验证后复核了日常 App 可执行文件的 SHA-256。

## 第二十三批 · 自动安装沿用检查结果

实现提交为 Core `727802b` 和 App `12a3ef1`。

App 后台检查直接交给自动安装流程处理已登记的请求及选中的注册表版本，下载沿用原元数据、镜像顺序和索引地址。App 和 daemon 在下载及切换前核对该检查是否仍有效。Core 最终在写入活动版本的 SQLite 事务内核对检查身份，已替换的检查无法产生版本切换，写入被拒时沿用安装器的回滚路径。

定向开发测试通过，包含 Core 自动安装 17 项、daemon 更新 19 项、存储替换 5 项及 App 更新 11 项。新增用例覆盖两次检查选择相同版本且时间戳相同的情况，验证旧检查既不能触发下载，也不能在 Core 提交版本。正常安装只发出一次元数据请求和一次归档下载。默认 Tauri 编译和 Core 错误守卫通过。

## 第二十四批 · 事件、传输和更新的开发检查

实现提交为 Core `4b93edc` 和 App `1276cfc`。

传输快照先检查所有权和订阅范围，再构造状态。上传配额、终态移交和事件注册分别在各自状态变更完成后释放锁。Core 的上传准入、关闭及版本大小判定可在常量上下文执行。注册表网络错误转换借用原始拒绝信息，按既有语义传递错误码和重试建议。

在修复前后的严格 Clippy 报告中核对诊断，并与审查基线比对源码来源。本轮处理了实施后新增的报错。Core 与 App 的全目标检查仍报告审查基线中的诊断，包含通配导入和未使用导入。

Core 781 项、App 413 项及 daemon 更新 19 项 Rust 开发测试通过。Core 最低 Rust 版本 1.88.0 的编译、App 默认 Tauri 编译及 daemon 单独执行的全目标严格 Clippy 检查通过。

## 第二十五批 · 进度订阅初始快照恢复

实现提交为 App `855c6f1` 和 JS `dbff85a`。

App 在注册进度订阅成功后继续轮询事件。初始状态查询遇到可重试失败时，沿用事件恢复的 100 ms 和 300 ms 间隔进行两次重试。重试期间仍接收进度，迟到的旧快照按 revision 丢弃。重试耗尽后退订并报告结构化失败。页面关闭或主动退订会终止后续查询。

testkit 在授权通过后拒绝当前 App 不支持的 `transfer.send` 分组选项 `contextId`，错误为 `unsupported`。选项校验先于响应 stub，拒绝会记录在对应调用中。客户端类型注释说明了该选项需要宿主支持。

三项回归在修复前复现了订阅提前结束和 mock 返回传输 ID。修复后 App 的 69 项定向测试、testkit 的 78 项测试通过，并重新生成了 App 注入产物。App 的 21 项和 JS 的 15 项契约检查、两仓类型检查、改动文件的 ESLint 检查及默认 Tauri 编译均通过。

## 第二十六批 · 诊断键名和引号凭据脱敏

实现提交为 App `205318f` 和 JS `adb7e79`。

App 与 testkit 在复制诊断对象时先脱敏键名，再按长度预算保留。脱敏后的同名键保留第一项，避免碰撞导致整个对象的诊断丢失。错误文本中带引号的敏感字段沿用原有字段规则，转义引号和截断值也会被整体遮盖。

四条共同回归通过实际 façade 的桥接拒绝和 testkit 的失败 stub 驱动。在修复前复现带鉴权 URL 键名和带引号凭据残留，修复后保留脱敏地址及可用诊断。App 的 90 项定向测试和 testkit 的 78 项测试通过。两仓的类型检查、App 的 21 项和 JS 的 15 项契约检查、改动文件的 ESLint 及默认 Tauri 编译均通过。App 注入产物已重新生成。

## 仍需交付

10 仍需 Android 系统返回、全屏和 Activity 的实机证据。11 的实体收发、Android 和鸿蒙卓易通测试场景见[移动端验收记录](./11-validation-and-rollout.md#8-移动端和设备间实测记录)。macOS 原生按键仍需实测。
