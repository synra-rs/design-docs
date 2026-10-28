# 2026-10-28 · 认证远端插件事件接收

## 当前实现

Core 的 EventService 已支持远端插件事件准入，与本机事件共用队列、worker、实际 WIT on-event、
插件运行代次检查和接收授权。App、daemon 的认证接收回调均接入共享实现，监听器保留完整信封和
原 RequestAuthority；接收准入不持有 PluginSet 锁，正在运行的 guest 不阻塞对端取得入队回执。

设备身份以 TLS 和 Hello 建立的上下文为准，接收服务的本机身份也必须与该上下文一致。
senderPluginId 从完整插件事件 namespace 取得，payload 中的 from、senderPluginId、traceId 不覆盖宿主上下文。
远端发送宿主仍须验证自己 guest 的身份、事件声明及发送授权；设备认证不等于对某个插件代码的远程证明。
接收插件仍须声明匹配的 handles，满足自己的 payload 条件，并持有具体 event:subscribe 授权。
事件订阅不赋予平台操作权限，平台调用继续通过独立 capability 闸门。

原设备授权在投递与 guest 执行准入前重新核验。排队期间撤权或同证书重新同意不会恢复旧任务；
订阅被撤销时不执行该 guest。已经开始的 guest 仍按自己的调用预算完成，本轮不承诺回滚已发生的副作用。
触摸板的独立控制会话、断连清理和租约仍须实现，不能用普通事件的五秒生命周期代替。

## 回执与资源约束

`_synra.plugin.event.admission` 回复原 requestId、trace 和实际来源/目标，
payload 为 status=accepted 或 status=refused（携带结构化 error）。
accepted 只表示入队或在去重窗口中已入队；事件可能无订阅者、过期、撤权或执行失败，不能把该回执显示为远端动作成功。
执行诊断保持有界，记录请求、trace、目标及错误码，不记录事件 payload 或 guest 返回值。

| 资源 | 当前上限及行为 |
| --- | --- |
| 待处理队列 | 128 项，与本机事件共享；另有最多一项正在处理 |
| 排队 payload | 8 MiB；单项还受共享插件 payload 字节、深度和结构预算约束 |
| 排队生命周期 | 本地单调时钟 5 秒；信封给出更短 ttl 时取更短值，不减两台设备的 sentAt |
| 去重记录 | 每个运行时最多 1024 条，保留 60 秒；满时拒绝新请求，不提前淘汰仍有效的记录 |
| 重复请求 | 按来源设备和 requestId 查找，复核原授权代次及完整信封摘要；一致时仅重发回执 |
| 冲突及新授权 | 同 ID 不同内容拒绝；同 ID 换授权代次也拒绝，不能悄悄重新执行 |
| 准入失败 | 不占去重条目；队列释放容量后可重新提交原先明确被拒绝的请求 |
| 回执传输 | 只在请求连接上回复；不拨回、不写入业务结果 LateQueue |

去重是内存窗口，重启后清空；没有 exactly-once 保证，也不自动重试不确定是否已入队的消息。
业务请求/结果、业务持久化幂等及实时输入序号需要各自的契约。

## 验证

Core 全部 838 项、daemon 全部 273 项库测试通过，包含 7 项新增远端事件用例和 1 项真实监听分发用例。
测试经过真实文件 SQLite、loopback TLS、生产认证入口、实际 WIT 组件和 daemon 原生宿主：

- guest 忙时仍可取得入队回执，解除阻塞后实际投递，保留 UUID、完整插件 id、requestId、trace 与嵌套 payload。
- 相同消息只执行一次；用后续屏障消息确认先前重复没有潜伏在队列中，冲突 ID 明确拒绝。
- 排队时撤销设备或插件订阅、同密钥重新同意后旧任务拒绝，新连接也不能把旧 requestId 当新请求执行。
- 满队列明确拒绝，释放容量后原被拒请求可提交；130 次真实 guest 处理成功。
- 1024 条回执满额后拒绝新 ID，原重复请求仍能取得回执，不静默提前淘汰。
- 非插件事件、回复帧、非 object payload、零生命周期和没有插件引擎分别拒绝。

压力用例发现测试夹具仅用单字节编码次数，第 80 次生成非法 UTF-8，触发 PluginTrapped；
已将夹具改为十进制字符串，没有扩大产品配额或绕过错误。该回归由 130 次实际 guest 处理覆盖。
认证 fixture 预置已批准事实，不把它冒充首次配对 UI 或原生密钥后端验收。

Core、daemon 严格 Clippy 通过，API guard 检查 464 个文件无违规；Rust 1.88 全目标检查通过。
App 默认 Tauri、全部目标及 official-plugin-tests 的严格 Clippy 通过，2 项 App 实际 guest 事件测试通过。
前端和插件 SDK/WIT 未变化，不重复 UI 测试或更新 SDK/CLI 固定版本。
App 默认 Tauri、全部目标及 official-plugin-tests 的 Rust 1.90 检查通过。
受影响 Rust 文件的增量格式检查无待应用改动，文档链接检查 74 个文件无问题。

## 阶段提交

| 仓库 | Commit | 完整 subject | 实测字符数 |
| --- | --- | --- | --- |
| Core | b8ca503 | feat: admit authenticated remote plugin events | 46 |
| App | 3411e1e | feat: admit remote events in the desktop host | 45 |

上述提交已签名并推送 main，实际检查 `%G? = G`、`%b` 为空、工作区为空且 upstream 比较为 `0 0`。
本记录使用 `docs: record remote plugin event admission`，实测 42 字符，subject 后不加 body。
实际执行的计数命令为：

```powershell
[System.Globalization.StringInfo]::new('feat: admit authenticated remote plugin events').LengthInTextElements
[System.Globalization.StringInfo]::new('feat: admit remote events in the desktop host').LengthInTextElements
[System.Globalization.StringInfo]::new('docs: record remote plugin event admission').LengthInTextElements
```

## 剩余工作

WIT 与 JS 的 device.send、device.broadcast 发送端仍未实现，不能将本轮接收能力视为手机插件已经可用。
后续继续补发送准入、认证发送、结构化拒绝和业务结果关联；Starter 的复制、打开链接及手机剪贴板，
触摸板的手机 UI 和原生输入会话仍待完成。
启用跨设备发送前还须加入贯通同一 trace 的跨设备因果预算，不能在每次接收时重新开始循环控制。
