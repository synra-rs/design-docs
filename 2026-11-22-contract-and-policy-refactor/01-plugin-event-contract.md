# 01 · 插件事件投影与订阅生命周期

状态：设备投影、会话投递和订阅生命周期已接入。传输进度、插件自定义事件及连接中断后的恢复待实施。优先级：P1。实际结果见[实施记录](./IMPLEMENTATION.md)。

## 1. 当前证据

- [event.ts:30](https://github.com/synra-rs/synra-app/blob/f7c6d1d/src/ipc/injection/event.ts#L30) 将参数命名为 `_pattern`，订阅 `KERNEL_EVENT_CHANNEL` 后直接调用 handler，没有名称匹配、payload 投影或插件作用域判断。
- 同文件 `.then(off)` 在已挂接情况下只返回 off，没有保存；返回的退订函数只把 active 改为 false。已挂接后退订不会释放底层监听。
- [device.ts:53](https://github.com/synra-rs/synra-app/blob/f7c6d1d/src/ipc/injection/device.ts#L53) 订阅 `device.changed` 并读取 `payload.devices`；[events.rs](https://github.com/synra-rs/synra-app/blob/f7c6d1d/src-tauri/src/events.rs) 发布的是包含 seq、at、event 的 IPC 信封。
- [capabilities.ts](https://github.com/synra-rs/synra-app/blob/f7c6d1d/src/ipc/injection/capabilities.ts) 将 event.subscribe 和 device.onChange 标为 true。现有支持性测试主要检查调用不抛异常，没有验证收到正确事件。

只读探针订阅 wanted.event 后投递无关信封，handler 被调用 1 次；完成挂接再退订，底层 off 调用 0 次。

## 2. 根因与责任边界

主界面管理事件与插件公开事件不是同一种契约。给插件端增加一次 `payload.event` 解包仍不够：完整管理快照可能包含插件不应读取的设备、任务或其他插件信息。前端过滤只能改善行为，不能承担宿主授权。

主窗口继续通过现有中央 dispatcher 消费管理事件。插件页面改用宿主投影后的定向流：宿主确认页面会话、授权和资源所有权后，决定可交付的事件及字段。

## 3. 目标设计

### 3.1 明确插件事件契约

在 core 的 IPC 层定义独立插件事件信封，至少包含序列号、事件名和允许公开的 payload；不复用带主界面 DTO 的 KernelEvent。JS 对应类型同步自权威契约。

建立显式投影表：管理事件 → 插件主题 → 所需授权 → 可见字段。设备变化使用与 device.list 同一构建器生成 DeviceTarget 集合，保留离线项；传输变化只包含该插件获准访问的 transferId。插件自定义事件沿完整插件身份及已有核心事件命名规则处理。

`device.changed` 这样的现有局部别名必须在实施时盘点所有调用方，选定一个公开名称并一次更新；不得同时在多个函数里散落新旧名称适配。主题表及匹配样例随类型一起审查。

### 3.2 每个页面一个订阅协调器

- 仅一条物理监听，每个逻辑订阅登记 pattern、handler、错误回调和清理状态。
- pattern 解析遵守核心事件模式语义；不能直接拿冒号分隔的能力匹配器代替点分隔的事件模式。
- 宿主以当前有效会话登记订阅并定向发送，撤权、窗口关闭和 authority generation 失效时撤销。
- 客户端统一解码插件信封，依据名称匹配后仅交付公开 payload。
- seq 中断后重读受影响资源的快照；不对有缺口的历史流继续假装连续。

同步返回 Unsubscribe 的 API 可以保留；增加可选的结构化失败通知入口，使异步挂接失败可报告。先完成类型和实际插件调用方盘点再确定参数形式，不用吞异常维持“调用成功”的外观。

### 3.3 清理是确定性行为

订阅状态至少区分 attaching、active、disposed。挂接前退订，则挂接成功后立即调用 off；挂接后退订，则立即调用保存的 off；重复退订幂等。最后一个逻辑订阅退出时按协调器策略释放物理监听。页面 dispose 统一清空。

## 4. 实施步骤与涉及文件

1. 在 `synra-core/crates/synra-ipc/src/event/` 定义插件信封、投影 DTO 和主题标识，明确资源可见性。
2. 在 App 的 `commands/plugin_ui/` 增加会话订阅服务；检查 `ipc_access.rs`、Tauri capability 配置与 Android 桥，确认插件页面不能通过底层 API绕回完整管理流。当前代码仅证明注入 API 的直通行为，不据此宣称所有原生 ACL 已失效。
3. 将 `events.rs` 的管理分发与插件定向投影分开，复用数据构建器，不复制一套设备卡逻辑。
4. 将 `src/ipc/injection/event.ts` 按协调器、解码和匹配职责拆分；device.onChange 与 transfer.onProgress 使用该基础设施。
5. 更新 `packages/client/src/surface.ts`、协议镜像、testkit、注入 facade 构建物和调用方说明。
6. 补充下表的行为回归，再移除直接订阅管理通道的注入实现。

## 5. 验收场景

| 场景 | 必须满足 |
| --- | --- |
| 两个不同 pattern | 各自只收到匹配主题的 payload，不能收到管理信封 |
| 两个插件、不同授权 | 一个插件不能读取另一个插件的传输、内部诊断或未授权资源 |
| 设备在线转离线 | list 与 onChange 的设备集合语义一致，不静默删除离线设备 |
| 挂接前、挂接后退订 | 底层清理恰好一次，退订后无回调 |
| 异步挂接失败 | 可观察的结构化失败，不冒充订阅已生效 |
| 缺序、重连、旧页面失效 | 重读快照，旧会话停止收消息 |
| 重复挂载与卸载 | 监听数恢复到初始值 |

## 6. 依赖、风险和完成条件

错误出口与 [07](./07-error-boundary.md) 对齐；传输投影由 [02](./02-transfer-state.md) 补齐。不要为完成事件基础设施顺手实现未支持的 device.send 或 event.emit。

完成条件是宿主定向投递、真实信封消费和生命周期回归均通过。仅通过 TypeScript 检查、返回一个函数或客户端过滤测试，不足以标记本项完成。
