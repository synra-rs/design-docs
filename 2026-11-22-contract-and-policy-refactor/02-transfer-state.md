# 02 · 传输状态与终态查询

状态：已接入外发传输的宿主终态、插件查询和 JS 契约。定向进度、worker 接管入口统一及实际网络收发验收仍在实施。优先级：P1。范围：Core 传输状态规则、App worker/查询接口、JS transfer 契约。

## 1. 当前证据

- [transfer.ts:146](https://github.com/synra-rs/synra-app/blob/f7c6d1d/src/ipc/injection/transfer.ts#L146) 在 worker 接管后将 transferring、0 字节写入 known；[status:166](https://github.com/synra-rs/synra-app/blob/f7c6d1d/src/ipc/injection/transfer.ts#L166) 发现任何缓存便直接返回。
- 同文件 onProgress 检查裸事件 type，Rust 的 [events.rs](https://github.com/synra-rs/synra-app/blob/f7c6d1d/src-tauri/src/events.rs) 则发送 KernelEventEnvelope。收到真实信封时无法更新缓存。
- [cleanup.rs:42](https://github.com/synra-rs/synra-app/blob/f7c6d1d/src-tauri/src/commands/transfers/cleanup.rs#L42) 移除活动传输后仅发布携带 transferId 的 TransferFinished。
- [direct.rs:82](https://github.com/synra-rs/synra-app/blob/f7c6d1d/src-tauri/src/commands/plugin_ui/transfer/direct.rs#L82) 仅查询活动列表；没有记录时返回 Unsupported，并用注释把终态保存责任交给 façade。该查询目前也未接收 ValidatedSession 做传输归属检查。
- [rpc.rs](https://github.com/synra-rs/synra-app/blob/f7c6d1d/src-tauri/src/commands/plugin_ui/rpc.rs) 的 optional Arc 决定 finish 同步发送还是后台发送，给同一个 RPC 形成两种时间语义。

重新运行的探针显示：真实格式进度信封回调 0 次；send 后调用 status 得到 transferring，实际 status RPC 调用 0 次。现有传输测试直接构造裸事件，未覆盖该边界。

## 2. 根因

活动面板需要移除已结束行，插件 API 需要查询操作结果；当前两者共用一个短生命周期列表。为补齐查询而放到页面的缓存没有可靠终态来源，也没有统一过期、所有权和重连机制。异步 worker 加入后，先前“send 返回时已经结束”的假设仍留在注释和缓存策略中。

## 3. 目标模型

### 3.1 宿主拥有可查询记录

定义类型化 TransferState 与最小 TransferRecord：标识、方向、所有者、状态、已确认字节数、总量、revision、结束时间及结构化失败原因。不要让客户端凭字符串猜测 terminal。

宿主记录区分 active 和 terminal：活动面板只展示 active；状态查询可读取有界 terminal 存储。首轮采用宿主进程内有界保留，容量和 TTL 作为具名服务策略，输入 now_ms 可测试。页面重载后可恢复；不承诺跨宿主重启恢复。如果未来需要落库，单独评估与已有传输日志的关系。

状态迁移规则与终态判定归 core；App 负责 worker 适配、插件会话归属和公开 DTO。不要把 Tauri token 或窗口句柄写入核心领域模型。

### 3.2 确定完成与失败的出口

- worker 在清理活动行之前提交真实终态；清理文件与保存业务结果是不同责任。
- 最终失败、取消、连接中断均生成可查询终态；TransferFinished 不再被解释为 completed。
- finished 事件携带最小最终状态或其 revision，客户端能按需查询；发布前写入记录，避免先收到事件后查不到。
- owner 在启动传输时登记。插件查询和进度订阅必须在宿主检查归属；主界面继续有管理视角。
- 未知或已过期 ID 返回明确的稳定原因，不能把“没找到”称作“不支持查询”。

### 3.3 收敛 API 时间语义

插件 send 统一表示已成功登记并由后台任务接管，返回 transferId；完成与失败通过 status/onProgress 获得。同步等待如仍有内部用途，使用显式独立方法，不通过 optional Arc 隐式选择。测试与生产使用同一任务执行接口，测试可注入受控 executor。

客户端只能把终态作为稳定缓存；非终态 status 查询刷新宿主，并处理 revision 与较晚事件的竞争。事件只用于及时通知，不能成为结果唯一来源。订阅先建立再读取快照，利用 revision 合并，覆盖快速完成和订阅晚到的情况。

## 4. 实施步骤与文件

1. 在 core 的传输模块定义状态与终态规则；在 `synra-ipc/src/dto/transfer.rs` 定义公开查询 DTO，明确旧状态字符串如何替换。
2. 在 App `commands/transfers/` 增加状态记录服务，统一 send_core、取消、接收与清理出口；复用现有 worker，避免再造传输协议。
3. 将插件传输归属加入启动上下文；`plugin_ui/transfer/direct.rs` 的 status 接收已验证会话；主界面读活动列表不受终态保留影响。
4. 移除以 optional Arc 区分时间语义的插件调用路径，更新 Rust 单测执行夹具。
5. 按 [01](./01-plugin-event-contract.md) 使用定向进度事件；拆分 transfer.ts 的上传编码、状态缓存和订阅责任。
6. 同步 JS 类型、testkit、facade 构建物及实际插件调用方；修正“send 等到完成”的旧说明。

## 5. 验收矩阵

| 场景 | 预期 |
| --- | --- |
| send 后不订阅，只轮询 | 按宿主变化到达真实终态，不固定在初始缓存 |
| worker 在订阅前完成 | status 或初始快照仍返回最终结果 |
| 只收到 finished | 查询能区分成功、失败、取消 |
| 页面重载 | 保留期内可查询本插件传输，无须旧页面缓存 |
| A 查询 B 的 transferId | 宿主拒绝，不泄漏 B 的任务详情 |
| RPC 响应晚于较新事件 | revision 防止旧快照覆盖新状态 |
| TTL/容量淘汰 | 明确过期或不存在，内存有上限 |
| 失败清理、取消与正常结束竞争 | 终态仅提交一次，worker 和临时文件不残留 |

## 6. 依赖与完成条件

依赖 [01](./01-plugin-event-contract.md) 的事件基础，与 [03](./03-upload-session.md) 的文件所有权转交共同验收。核心规则测试、宿主 worker 集成和真实序列化信封到客户端的回归均通过，才视为完成。不能仅删除前端缓存而保留“终态无处查询”的宿主行为。
