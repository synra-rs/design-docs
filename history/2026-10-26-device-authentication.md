# 共享设备认证通道与守卫误报修复（2026-10-26）

对应 [19 D1、D5](../19-roadmap.md)、[04 §7.2](../04-wire-protocol.md)、[17 §5](../17-storage-and-config.md)。
本记录保存本次代码事实及验证范围，不替代目标契约。

## 共享认证实现

`synra-net::authentication` 已加入正式 workspace，采用与独立实验相同的 rustls、rcgen 与 ring provider。
它接收已连接的 TCP socket、本机凭据及已核验的对端身份；只有完成 TLS 1.3、双向证书验证、
`synra/1` ALPN 协商及完整证书固定校验后，才返回可读写的认证流。
认证失败销毁传入的 socket，不提供取回明文连接的接口。

- 生成证书的 DNS 名称包含规范化设备 UUID；恢复或读取公开凭据时，以显式 Unix 时间检查有效期及设备名称。
- 本机恢复还校验证书与 PKCS#8 私钥匹配，缺失、错误或不匹配均返回错误，不生成替代密钥。
- 生成的证书有效期为 3650 天，起始时间提前一天容纳时钟偏差；到期、换证或换钥需要明确重新核验，不静默更新固定凭据。
- 公开身份提供完整证书与 SHA-256 指纹；解析候选证书不等于用户批准，批准事实由配对服务持有。
- 只接受明确批准的完整证书，拒绝仅由已批准证书签出的派生证书；对端设备 id 来自该绑定，不能由 Hello 覆盖。
- 单项证书或密钥最多 16 KiB，入站批准列表最多 256 项，拒绝空列表和重复设备或凭据；关闭会话恢复、票据与早期数据。
- 握手总预算最多 10 秒，每次底层读写重新应用剩余时间，持续发送不完整记录不能重置预算。
- rustls 应用明文及发送缓冲设为 64 KiB；这不表示整个进程或所有 TLS 结构总共只占 64 KiB。
- 握手成功恢复原有应用读写超时，后续读写只通过认证流进行。

新增测试经过共享模块的正式入口和真实 TCP，覆盖凭据恢复后重连、双向业务通信、线路密文、
设备 id 冒用、同 id 换钥、未批准与匿名客户端、明文、TLS 1.2、缺失或错误 ALPN、
派生证书拒绝、输入边界、证书有效期、静默对端与不完整记录超时。
派生证书测试同时确认标准 TLS 握手已通过，再断言完整证书固定校验拒绝连接。

## API 守卫误报

首次工作区测试在 `api-guard` 集成测试失败：旧规则逐行匹配 `std::io::Error`，
将私有 helper、函数体及测试构造错误的语句当成公开 API，同时维护一份固定误报清单。

外部错误规则现改为解析 Rust 声明，只检查公开签名、字段和类型；私有实现、函数体及外部 trait 实现不参与该规则。
回归测试覆盖多行签名、嵌套错误类型、受限可见性、测试模块、标准 IO trait 和无法解析的输入，
并验证带空白的限定类型路径不能绕过检查。集成测试直接要求实际守卫成功，不再接受误报豁免。
错误模板检查也改为读取属性语法，避免 rustfmt 换行使规则被漏读。
现有容器及枚举规则、分级政策尚未整体修订，D5 仍未完成。

## 已验证范围

| 命令或检查 | 结果 |
| --- | --- |
| `cargo clippy -p synra-net --all-targets -- -D warnings` | 通过 |
| `cargo test -p synra-net authentication --quiet` | 19 个认证测试通过 |
| `cargo clippy -p api-guard --all-targets -- -D warnings` | 通过 |
| `cargo test -p api-guard --quiet` | 25 个单测及 3 个集成测试通过 |
| `cargo test --locked --workspace --quiet` | 修正守卫后通过；保留 1 个既有忽略用例 |
| `cargo clippy --locked --workspace --all-targets -- -D warnings` | 通过 |
| `cargo run --locked -q -p guards` | 12 项通过，含调用方构建、MSRV 1.88、文档链接及可检查的计数 |
| app：`cargo check --manifest-path src-tauri/Cargo.toml` | 默认 Tauri feature 编译通过，依赖锁文件已同步 |
| app：`cargo test --locked --no-default-features --manifest-path src-tauri/Cargo.toml --quiet` | 317 个测试通过 |
| `cargo check --locked -p synra-net --target aarch64-linux-android --target-dir spike/device-authentication/target-android` | 通过 |
| `cargo test --locked -p synra-net --target aarch64-linux-android --target-dir spike/device-authentication/target-android --no-run` | Android 测试程序链接通过 |

Android 命令沿用[认证实验](../../synra-rust-core/spike/device-authentication/README.md)记录的 NDK r28b、API 24 环境变量。
未在 Android 设备或模拟器运行本模块，不据此声称移动端运行行为已通过。
本次未改前端或 Wasm 执行逻辑，不额外重跑未受影响的前端检查；修改的 Rust 文件单独检查格式。

## 未完成边界

共享模块目前只提供已批准身份之间的认证能力；两个宿主仍使用原有 TCP 建链，尚未接入该模块。
安全密钥后端、凭据持久化与丢失恢复策略、首次带外核验及双向确认、配对取消与撤销、
现存连接关闭和应用 Hello 设备 id 复核仍需继续实现，因此 D1 不算完成。
本次没有发布操作。
