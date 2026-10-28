# 2026-10-28 · 可延后复核的认证请求上下文

## 业务边界

共享接收层原来只向 `KernelEventCallback` 传递 Envelope。异步插件投递还需要保留收到消息时的
TLS 身份和持久授权代次，不能在出队时加载一份新授权，把旧消息重新变为可执行。

Core 现在提供 `peer::authenticated::RequestAuthority`，由完成 TLS 和 Hello 的认证链路取得，
保留实际本机、对端和原授权代次；内存或明文链路不能提供该上下文。它不含私钥，不能序列化为插件令牌。
原认证流和上下文 clone 共享拒绝状态：授权撤销、凭据替换、同证书重新配对或授权存储失败后，
旧上下文与旧连接均不能自行恢复；恢复数据库后只有新建立且重新核验的连接可继续。

接收层在调用内核事件回调前检查认证上下文，并把上下文与信封一起传给 App 和 daemon。
未来排队的消费者须保留该上下文，并在实际执行准入边界调用 `authorize(envelope)`，
同时复核来源、目标和原代次。来源或目标不匹配只拒绝该请求，不将其当作整个设备授权被撤销。
数据库锁只覆盖授权查询，不跨越网络 IO 或 guest 执行。

该上下文只证明设备授权，不代替插件身份、事件 namespace、capability、payload、去重或队列预算检查。
它不是请求截止时间，也不是连接存活或触摸板输入会话租约；这些状态必须由各自服务单独维护。

## 验证范围

- Core 全部 838 项库测试通过，包括新增 6 项认证上下文测试。
- 新增测试使用生产认证入口、真实文件 SQLite、双向配对和 loopback TLS，覆盖跨线程保留、
  伪造来源/目标、同密钥重新配对、数据库故障恢复、连接与上下文互相传播拒绝，以及明文链路拒绝。
- 原身份修复测试同时检查先前捕获的请求上下文被撤销，保留缺失本机登记的分支。
- 初次重新配对用例只撤销了一端，按现有契约被 ClaimRefused 拒绝；已纠正为两端明确撤销后重新同意，
  没有放宽配对校验。
- Core 和 daemon 的受影响目标严格 Clippy 通过；API guard 检查 463 个文件，无违规。

- daemon 的生产接收入口完成 1 项真实认证连接的请求/回复验证，回调检查实际来源、目标与授权。
- App 默认 Tauri 构建、全部目标及 official-plugin-tests 的严格 Clippy 通过；3 项认证传输测试通过，
  覆盖实际接收、拒绝及取消。前端与 Wasm 组件未改动，不重复已有 UI 验证。
- Core 与 daemon 的 Rust 1.88 全目标检查通过。

- App 默认 Tauri、全部目标及 official-plugin-tests 的 Rust 1.90 检查通过。
- 受影响 Rust 文件按项目已有的增量格式检查方式验证，无待应用格式改动；文档链接 73 个文件无问题。

## 阶段提交

| 仓库 | Commit | 完整 subject | 实测字符数 |
| --- | --- | --- | --- |
| Core | 5118475 | feat: retain authenticated request authority | 44 |
| App | 6732604 | refactor: pass authenticated request context | 44 |

上述提交均已签名并推送 main，实际检查 `%G? = G`、`%b` 为空、工作区为空且 upstream 比较为 `0 0`。
本记录使用 `docs: record deferred request authority`，实测 39 字符；均使用 subject，不加 body。
实际执行的计数命令为：

```powershell
[System.Globalization.StringInfo]::new('feat: retain authenticated request authority').LengthInTextElements
[System.Globalization.StringInfo]::new('refactor: pass authenticated request context').LengthInTextElements
[System.Globalization.StringInfo]::new('docs: record deferred request authority').LengthInTextElements
```

## 后续工作

本轮没有把 `device.send` 或远端插件事件服务标记为已实现。下一步使用上述上下文实现共享的
远端事件准入和有界投递，再接发送端、请求关联和执行结果；仍须独立检查插件授权与当前运行代次。
Starter 的跨设备复制、电脑打开链接和手机剪贴板、触摸板的原生输入仍未完成。
