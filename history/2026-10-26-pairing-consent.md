# 受限配对同意交换（2026-10-26）

对应 [19 D1](../19-roadmap.md)、[04 §7.2](../04-wire-protocol.md)、[ADR-017](../20-adr.md)。
本次实现共享网络层的首次核验基础，不代表两个宿主已经采用安全配对流程。

## 核验与会话边界

`synra-net::authentication::pairing` 提供完整公开证书的邀请编码与解析，以及只交换用户同意的专用通道。
邀请为 ASCII 文本 `synra-pair:1:<device UUID>:<base64url 无填充的完整证书 DER>`，不包含私钥或业务授权。
总长上限 2048 字节，编码在分配证书字符串前检查大小，解析在解码前检查输入大小、ASCII、格式和版本，
并复用正式凭据入口验证设备名称与证书有效期；过大的完整证书拒绝生成邀请，不截断。
当前生成的设备证书可往返解析；尚未实现二维码显示、摄像头扫描或邀请导入 UI。

双方必须从预期设备的可信显示或传递渠道取得完整邀请，并由本地用户确认其来源；
扫描此类显示是后续 UI 的可选传递方式，不能把发现广播或当前未信任网络连接中收到的邀请自动当成批准。
专用通道复用正式 mTLS、设备 id 名称验证、完整证书固定校验、禁用恢复与 early data 及握手总时限。
只有预先核验的完整证书能进入这一通道，同一设备 id 换钥仍拒绝；配对自身也拒绝。

配对 ALPN 为 `synra/pairing/1`，业务 ALPN 为 `synra/1`，双向替换均拒绝。
用户同意的会话关联采用 [RFC 9266 §2](https://www.rfc-editor.org/rfc/rfc9266.html)：
固定标签 `EXPORTER-Channel-Binding`、空 context、32 字节输出，经 rustls 正式 exporter 接口取得。
它关联已经通过 mTLS 的这次会话，身份认证仍由标准 TLS 与带外批准的证书固定校验提供，不自行设计认证摘要或握手。

通道没有通用 Read/Write 或原始 socket getter，`confirm` 消耗它且不能重复调用。
确认记录为四字节大端长度前缀与最多 1024 字节的严格 JSON，字段为 `version`、`from`、`target`、`binding`、`confirmed`；
版本固定为 1，拒绝未知字段、身份不符和其他会话的 binding，先检查长度再分配接收缓冲。
握手后用户窗口最多 60 秒，确认 IO 的总预算不超过剩余窗口，每次实际 socket 读写都重设剩余时限，
并在产生同意证据前再次检查总预算与窗口；不完整 TLS 记录不能延长等待。

本地拒绝发送负记录后立即关闭，不等待对端用户；消息发送或接收失败也不会产生同意证据。
立即取消可能与对端写入竞争，对端可能收到负记录或连接错误，不能假定取消消息一定送达。
成功或失败均关闭本次配对连接，遵守 [RFC 9266 §4.1](https://www.rfc-editor.org/rfc/rfc9266.html) 的单次机制使用与交换结束后关闭要求。
业务需另建认证连接，不能把配对连接升级成业务流。

`MutualConsent` 由成功的双方正向确认产生，含双方公开身份及本次会话 binding，不能公开构造、Clone 或反序列化。
它仅代表两端在此会话表达了用户同意；不保存数据库、不返回配对成功结果，也不授予业务权限。
后续共享服务仍需将任务与会话关联，验证取消、撤销、身份和期限，并在双方持久化完成后确认授权。

## 验证范围

新增 19 个测试，网络场景均经真实 loopback TCP 和生产通道入口，验证邀请往返、无效及过期邀请、超长完整证书，
双方身份与 exporter 一致、同密钥重连后 binding 改变、旧会话确认重放拒绝、发送者与目标及版本和 schema 检查，
超大或空帧、无效 JSON、断开、本地与远端拒绝、无需远端回答的立即取消、到期提示、零预算、同 id 换钥与自配对拒绝，
业务与配对 ALPN 双向隔离，以及成功交换后连接关闭。
标准 exporter 测试还独立通过 rustls 客户端接口计算结果，与正式服务器发出的记录比较；
慢速不完整 TLS 记录测试通过完成信号停止发送方，验证整个确认阶段确实在总预算内终止。

| 命令或检查 | 结果 |
| --- | --- |
| `cargo check -p synra-net --lib` | 通过 |
| `cargo clippy -p synra-net --all-targets -- -D warnings` | 通过 |
| `cargo test -p synra-net --lib authentication::tests::pairing --quiet` | 新增 19 个测试通过 |
| `cargo clippy --locked --workspace --all-targets -- -D warnings` | 通过 |
| `cargo test --locked --workspace --quiet` | 通过 |
| `cargo run --locked -q -p guards` | 12 项通过，含真实应用调用方、MSRV 1.88、90 份文档链接及计数检查 |
| app 默认 Tauri 构建及无默认 feature 的 Rust 测试 | 通过，317 个测试；锁文件同步网络层依赖 |
| Android：`cargo test --locked -p synra-net --target aarch64-linux-android --target-dir spike/device-authentication/target-android --no-run` | 测试程序链接通过，未运行 |

Android 使用[技术实验](../../synra-rust-core/spike/device-authentication/README.md)记录的 NDK r28b、API 24 配置。
本次没有前端或 Wasm 执行路径修改，不额外重跑对应专门检查；修改的 Rust 文件单独检查格式。
最终修改的 10 个 Rust 文件单独通过 `rustfmt --check`，各仓库 `git diff --check` 通过。
工作区门禁后只增强了测试夹具，让真实 TLS 入口消费编码并解析后的带外邀请；重新通过该模块的 Clippy、19 个测试及 Android 测试程序链接，未重复无关工作区测试。

## 尚未实现

对端身份与配对任务的持久绑定、双方持久化完成确认、取消与撤销的提交重检、宿主接入及 Hello 身份复核仍待实现。
两个宿主仍调用原有普通 TCP、token 和六位显示码流程，不能声称生产配对或业务流已经得到这次保护。
旧显示码文档中错误的身份认证承诺已移除；由公开 UUID 计算的代码不能证明对端身份，token 也不能代替 mTLS。
Android 只链接测试程序，未在设备或模拟器运行；其他平台密钥后端及身份修复流程也仍待实现，因此 D1 保持开发中。
