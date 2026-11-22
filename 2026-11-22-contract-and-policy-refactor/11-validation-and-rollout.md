# 11 · 跨仓库验收、证据与分批交付

状态：验收计划及本轮只读证据。此文中的构建与测试命令尚未在本轮执行，不能作为通过记录。

## 1. 现有验证的盲区

- App 的 [injection-capabilities.test.ts](https://github.com/synra-rs/synra-app/blob/f7c6d1d/tests/ipc/injection-capabilities.test.ts) 主要用“调用不抛异常”代表支持。订阅返回了函数但未正确匹配和交付，仍能通过这种断言。
- [injection-transfer.test.ts:143](https://github.com/synra-rs/synra-app/blob/f7c6d1d/tests/ipc/injection-transfer.test.ts#L143) 手工发裸 transferUpdated；生产 Rust 发事件信封，两侧没有由同一序列化样例贯通。
- testkit 的 implements SynraClient 与协议 shapes 证明结构，不能证明授权、状态和副作用语义。
- 原生录制后端只能证明请求序列，不能证明 Windows/macOS/Linux 实际输入效果。

因此本轮同时计划加强验证，但不通过为每行 helper 写镜像测试来增加数量。

## 2. 本轮已经完成的只读验证

执行日期：2026-11-22。源码提交见 [总览](./README.md)。执行方式：用本地 TypeScript transpileModule 将目标源模块转为内存 CommonJS，递归加载相对运行时依赖，直接调用导出函数；未写入测试或生产文件。Node 实际运行时为本机 26.13.0；该版本仅记录调查环境，不修改项目 engines。

| 探针 | 实际结果 | 证明范围 |
| --- | --- | --- |
| testkit hostFeatures=[] 后订阅 | advertised=false，调用返回 function | 支持查询与执行不一致 |
| testkit 授予 platform:read-clipboard | capability_denied，要求 platform:clipboard:read | 授权拼写偏离真实宿主 |
| stub device.list 后调用两次 | recorded=1，lastArgs 为 `<stubbed>` | 记账统计的是设置 stub，遗漏真实调用 |
| façade 订阅 wanted.event 后注入无关信封 | receivedUnrelated=1 | pattern 未参与匹配 |
| 订阅挂接完成后调用退订 | offCalls=0 | 底层监听未释放 |
| asSynraError/isSynraErrorValue(null) | 两者均抛 null.code 读取异常 | 边界处理不是总定义 |
| asSynraError(new Error('transport failed')) | code=unsupported，feature=unknown | 未知失败被误归类 |
| 发送后收到 `{seq,at,event}` 传输事件 | 新增进度回调=0 | 真实信封层级不匹配 |
| send 后 status | transferring、receivedBytes=0，status RPC=0 次 | 非终态缓存阻止刷新 |
| 同一传输事件改成裸对象 | 新增回调=1 | 排除该探针中监听器未挂接的干扰 |

上述 probes 使用模拟 Bridge 驱动真实 façade，验证的是对应模块的行为，不代表完整 Tauri 原生桥、资源 ACL 或跨设备传输已经端到端运行。Rust 上传竞态、版本边界和平台输入本轮只完成源码审查。

复现这些行为时，在现有 Vitest 中直接加载对应模块，按以下步骤建立回归即可；不要复制错误算法作测试对象：

1. 由 createMockClient 构造上述 hostFeatures/grants，断言真实方法与调用记录。
2. Bridge.listen 使用可手工 resolve 的 Promise，返回带计数的 off，分别在 resolve 前后退订。
3. 使用 core 真正序列化的事件信封驱动 façade，保留不同事件/不同资源作为负例。
4. send 的 bridge 按 method 回答 begin/chunk/finish/status，分别统计 status 调用和订阅结果；模拟 worker 在订阅前完成。
5. 用表驱动 unknown 错误输入，不只测合法对象。

## 3. 首批必须建立的跨边界测试

### 3.1 Rust 生产者到 TypeScript 消费者

Rust 使用真实 DTO 与 serde，经 Rust 工具标准输出产生语义夹具；前端消费经过核对的快照，不能自行重写一个“看起来一样”的对象。快照存于 JS/App 的测试目录并记录来源提交，不在只允许既定文件类型的 core 仓库新增 JSON，也不加入 JS/Python helper。

跨仓库通过已审查提交或受控发布的夹具快照串联，不在核心 Cargo workspace 引入兄弟仓库源码路径。联合检查记录各仓库 SHA。夹具含事件、错误、终态和授权样例，字段省略、空集合、错误分支都包括。

### 3.2 Façade 与 testkit 同场景断言

App 的测试可使用 testkit 作为 devDependency，场景以工厂注入不同实现。断言公开结果、错误类别、回调与清理次数，不断言内部私有结构。core 的能力匹配向量为授权语义提供第三方校准，避免 façade 和 mock 一起错而互相证明。

### 3.3 真正驱动资源与两端

上传并发使用 barrier 控制交错和独立临时目录；传输使用真实 worker、loopback 两端和截止时间；Wasm 使用真实组件；平台输入另列实体环境证据。不用 sleep 等待概率成功，不以相同的 stub 回答发送端和接收端来声称端到端通过。

## 4. 各项最小验收与命令

命令在对应仓库执行，使用该仓库工具链和当前已解析依赖。改依赖后按项目流程更新 lock，再执行 locked 检查。先跑受影响包，阶段末运行联合检查；不用每改一行重跑全矩阵。

| 项目/改造项 | 实施后检查入口 |
| --- | --- |
| core 版本 05 | cargo test --locked -p synra-plugin-api；-p synra-registry；-p synra-plugin-host；-p synra-core（按改动选包） |
| core 事件/状态 01、02 | cargo test --locked -p synra-ipc；-p synra-core；受影响 daemon/宿主集成测试 |
| core 原生键盘 06 | cargo test --locked -p synra-platform；目标平台构建与实体环境操作 |
| Wasm 08 | cargo test --locked -p synra-plugin-host --features wasm-runtime；按现有 CI 补充 feature 矩阵 |
| App Rust 所有相关项 | cargo test --no-default-features --manifest-path src-tauri/Cargo.toml |
| App Tauri 编排 | cargo check --manifest-path src-tauri/Cargo.toml，覆盖默认 feature 的 shell 与窗口代码 |
| App TS | pnpm test；pnpm run lint；pnpm run build；pnpm run shapes |
| JS testkit 04 | pnpm --filter @synra/testkit test，随后 pnpm test；pnpm run check；pnpm run lint；需要发布产物时 pnpm run build |
| core 阶段检查 | cargo run --locked -p guards -- .；cargo run --locked -p synra-error-guard -- .；相关 Clippy、文档检查和 CI 矩阵 |

现有 App shapes 会重建并比对提交的 `src-tauri/inject/facade.js`，见 [facade 检查](https://github.com/synra-rs/synra-app/blob/f7c6d1d/scripts/checks/facade.ts)。修改 injection 必须更新构建物；不能只改源文件便认为已进入 Rust 宿主。

## 5. 实施批次与交付记录

| 批次 | 内容 | 进入下一批的条件 |
| --- | --- | --- |
| A | 07、09、最小联合夹具；04 的权限/记账修正 | 错误/业务原因稳定，回归先红后绿 |
| B | 01、02、03 及 04 对应模型 | 定向事件、终态查询、资源接管与清理共同通过 |
| C | 05、06、08、10 独立收敛 | 各领域业务入口与平台边界有证据 |
| D | 全部联合场景和文档同步 | 无临时双路径，契约、构建物、依赖提交一致 |

每项实施可拆多个小提交，但生产调用方与接口变更必须处于同一可工作的批次。提交记录包含各仓库 SHA、命令、退出结果、平台和未覆盖边界。测试结果不能只记录“通过”，也不把旧历史中的测试数量当本批结果。

## 6. 文档与范围控制

实施后分别更新相应专题：事件/状态更新 08、23；能力和 testkit 更新 10、25；版本更新 14、16；按键和宿主环境更新 28 及实际平台说明；错误和预算更新 12、18。

CURRENT-STATE 仅记录实际完成且已验证的变化，计划继续保留本目录。源码发生业务行为变化时同步用户文档；不要为了让所有文档看起来一致而重写历史证据。

保持通用 task 审批与取消、host/plugin 边界、能力 gate 和资源限额。不在本轮顺手实现未支持的插件业务，也不删除合理的平台分支、资源上限或显式 unsupported。

## 7. 总体验收定义

全部条目完成必须同时满足：旧补丁路径已删除或归入唯一责任模块；公开结果和错误由权威类型表达；实际宿主与 testkit 通过共同用例；会话失效与失败路径资源归零；生成物和依赖可复现；原生部分单独记录平台证据。

如果某个平台或真实组件尚未验证，只标记对应边界待验收，不用前端单测或源码存在替代结论。整体计划在实施完成前始终保持“计划”状态。
