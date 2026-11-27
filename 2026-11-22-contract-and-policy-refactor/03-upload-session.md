# 03 · 分块上传会话与资源所有权

状态：上传状态、会话归属、预算及 worker 移交已接入。联合发布证据见[实施记录](./IMPLEMENTATION.md)。优先级：P1。范围：App 插件上传服务；状态规则和限额策略归 Core。

## 1. 当前证据与具体失败路径

[chunked.rs:108](https://github.com/synra-rs/synra-app/blob/f7c6d1d/src-tauri/src/commands/plugin_ui/transfer/chunked.rs#L108) 先在全局 map 锁内取得 path、total、next，解锁写文件，再次加锁把 written 设为 next。两个 chunk 可以同时读到相同 written，各自通过上限检查并完成追加，随后把相同 next 写回。文件长度与账面长度从此不一致；finish/abort 也可能在追加之间取走条目。

[types.rs](https://github.com/synra-rs/synra-app/blob/f7c6d1d/src-tauri/src/commands/plugin_ui/transfer/types.rs) 只保存 plugin_id、路径、目标与计数，没有页面会话、authority generation、过期时间、上传状态和全局预算。检查了 uploads 与 discard_upload 的生产调用方，未找到过期清扫和页面关闭回收；[transfer.ts:160](https://github.com/synra-rs/synra-app/blob/f7c6d1d/src/ipc/injection/transfer.ts#L160) 却把这些当作 abort 失败后的保证。

这些是源码路径推导，本轮未运行并发 Rust 复现。当前单次文件/块大小上限是必要约束，不是要删除的“魔法数字”；缺的是上传实例和资源总量的完整模型。

## 2. 目标模型

### 2.1 上传是带身份和状态的资源

建议 UploadSession 包含：uploadId、owner plugin、owner session/generation、目标、预期总量、已提交 offset、lastActivity、状态和受控临时文件所有权。

状态迁移为 receiving → handingOff → transferred，或 receiving/handingOff → failed/aborted/expired。transferred 表示文件已交由发送 worker 管理，不表示远端接收成功。远端结果属于 [02](./02-transfer-state.md)。

每次调用既要通过现有 authority 校验，也要验证资源属于本次会话。不能让同插件新开的页面继承已失效页面的 uploadId。

### 2.2 每个上传独立串行执行

全局 map 仅用于取得上传句柄，不跨磁盘 IO 持有全局锁。每个上传使用自己的同步边界，将状态检查、offset 校验、写盘和计数提交视为一个操作；finish/abort 与 chunk 使用同一同步边界。

chunk 请求增加明确 offset。仅接受 offset 等于已提交位置的追加；乱序和重复返回结构化 expectedOffset。首轮不实现自动重传和任意断点续传，避免以“重试友好”为由静默重复追加。写入部分失败时上传进入 failed 并清理，不允许客户端用不确定 offset 继续。

### 2.3 明确所有权转交

finish 在同一上传状态保护下检查实际文件长度和预期长度，进入 handingOff。只有 worker 登记成功后，临时文件清理责任才转移给传输任务；任何启动失败均由上传服务回收。重复 finish 不能产生两个传输。

abort/expire/revoke 幂等；若 worker 已接管，上传 abort 不再删除 worker 文件。业务取消走传输服务，不能两个服务各自尝试删除同一个活跃文件。

## 3. 预算、清扫与可观测性

- 保留单文件和单块限制，并增加每会话/每插件并发数、宿主临时存储总预算；限额在 begin 时预留，在结束时释放。
- 具体默认值在实现时根据现有传输预算和内存/磁盘占用确定，集中到 UploadPolicy 并写明单位。首轮不把它们全部暴露为设置。
- TTL 由 lastActivity 和注入 now_ms 判断；接入现有宿主维护循环，使用可取消清扫，不新增独立无界线程。
- 页面关闭、会话失效、撤权、插件停用及宿主正常退出调用同一个 revoke_owner/close 接口。
- 启动残留清扫只处理服务明确拥有的 spool 子目录和可识别记录，不递归删除用户选择的路径。清理失败保留诊断和后续回收标识，不能仅吞异常。

## 4. 实施步骤与文件

1. 将 `plugin_ui/transfer/types.rs` 和 `chunked.rs` 拆成会话、追加、接管、清扫责任模块，RPC 仅解析参数和委托。
2. 将通用状态转换及预算算法放入无 UI 依赖的 core 模块；App 保留会话 token、路径和 worker 适配。先复用现有相关模块，确有复用前不新建通用 crate。
3. 在窗口关闭、authority revoke、插件生命周期及宿主 shutdown 路径调用清理入口。
4. 更新 transfer.begin/chunk/finish 的 IPC DTO 和 JS 上传循环；由 begin 回答块上限，客户端不再独立维护一份不可核对的限制。
5. 与 [02](./02-transfer-state.md) 统一 worker 注册接口，消除 optional Arc 造成的两种 finish 行为。
6. 盘点旧的单次 base64 transfer.send 实际调用方；若已无生产使用，移除旧内部操作及专用 4 MiB 分支。若仍使用，则通过同一上传/接管服务，不保留独立清理实现。

## 5. 验收矩阵

| 场景 | 预期 |
| --- | --- |
| 同一上传两个同时到达的 chunk | 文件和 written 始终一致；重复 offset 至多一个成功 |
| 不同上传并发 | 一个慢磁盘上传不占用整个注册表锁 |
| chunk 与 finish/abort 竞争 | 状态串行，终止后不能继续写入或恢复条目 |
| chunk 部分写入后失败 | 上传失败并回收，不把部分写入计为完整块 |
| worker 启动失败/立即结束 | 文件责任只转交一次，临时文件与预算最终归零 |
| 页面关闭、撤权、generation 更新 | 原会话资源立即失效并进入清理 |
| TTL、配额、启动残留 | 有界拒绝与清理；测试注入时间，不依赖 sleep |
| 任意错误路径 | 只删除服务拥有的路径，诊断能关联 uploadId |

## 6. 完成条件

必须使用可控制的 barrier 或故障注入验证并发交错，不能只串行调用 chunk 再检查计数。磁盘测试使用独立临时目录并设置测试截止时间。实际接管和会话清理路径通过后，才可删除客户端注释中尚不存在的保证或将其改成已验证行为。
