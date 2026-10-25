# 本机设备凭据持久化（2026-10-26）

对应 [19 D1](../19-roadmap.md)、[17 §5](../17-storage-and-config.md)，
接续[共享设备认证通道](./2026-10-26-device-authentication.md)。本记录描述本次代码事实与验证边界。

## 实现与恢复规则

- `synra-store::secrets::WindowsCredentialBackend` 直接访问当前 Windows 用户的原生 Credential Manager，
  使用目标平台依赖 keyring 3.6.3 的 Windows 后端，不通过可全局替换的默认构造器选择 mock。
  原生访问在进程内串行化，限制单项 2560 字节；错误只返回固定分类文本，不格式化可能包含敏感字节的库错误。
- 私钥账号按公开证书 SHA-256 指纹区分，与插件命名空间隔离；不同登记不能覆盖同一私钥槽位。
  SQLite 迁移 18 增加单行公开登记表，只含本机 id、证书、后端种类和就绪状态，没有私钥字段。
- `synra-core::device_credentials::{enroll, load}` 提供共享完整流程：原子认领公开登记、释放数据库锁、
  写入并读回密钥、验证 id、有效期与证书私钥匹配，再重检本机身份和公开记录后提交。
  数据库写入接入现有耗时观测；不在数据库锁中执行密钥 IO 或加密验证。
- 已有配对而没有登记时拒绝初次登记；内存后端不允许用于持久设备身份。
  `load` 永远不生成身份，重复 `enroll` 只恢复已登记的同一凭据，不换钥掩盖错误。
- 写入前失败、丢弃写入或损坏数据会保留未就绪记录并拒绝认证；写入成功后报错或数据库提交失败时，
  下一次可读回并验证同一私钥后完成登记；缺失、错钥、后端改变或本机 id 改变均拒绝继续。
- 初次 `Identity::load` 改为原子创建后读取已存赢家，避免两个连接同时启动时后写入者覆盖已经登记凭据的设备 id。

## 验证范围

定向验证已通过：共享凭据服务 12 个测试、SQLite 凭据边界 2 个测试、初次身份认领回归测试及原有身份测试。
故障注入覆盖写入前后错误、损坏或丢失写入、只读数据库提交失败、外部 IO 期间身份改变，
并使用同一磁盘数据库的两个独立连接验证进行中的登记不会被竞争者覆盖。

Windows 原生测试使用独立随机命名空间并清理测试密钥，验证二进制密钥跨后端对象读取、删除幂等及大小限制。
共享服务的原生测试关闭数据库并重建后端，再恢复同一凭据，经生产认证通道和真实 TCP 双向通信；
同时检查 SQLite 主文件、WAL 和 SHM 不包含完整私钥 DER，删除密钥后确认加载与再次登记均拒绝自动换钥。

阶段门禁结果：

| 命令或检查 | 结果 |
| --- | --- |
| 定向编译及 `cargo clippy -p synra-core -p synra-store --all-targets -- -D warnings` | 通过 |
| `cargo test --locked --workspace --quiet` | 已执行到 platform，发现下述既有测试断言问题；此前各模块通过，包括 core 759 个测试 |
| `cargo test --locked -p synra-platform --quiet` | 修正断言后 80 个测试通过，保留触发样本 |
| 从失败点之后续跑其余 11 个包的测试 | 通过，包括 store 263 个测试；不重复此前已通过的包 |
| `cargo test --locked --workspace --doc --quiet` | 通过 |
| `cargo clippy --locked --workspace --all-targets -- -D warnings` | 通过 |
| `cargo run --locked -q -p guards` | 12 项通过，含调用方构建、MSRV 1.88、89 份文档链接及计数检查 |
| app：`cargo check --manifest-path src-tauri/Cargo.toml` | 默认 Tauri feature 编译通过，锁文件同步 keyring 依赖 |
| app：`cargo test --locked --no-default-features --manifest-path src-tauri/Cargo.toml --quiet` | 317 个测试通过 |
| `cargo check --locked -p synra-core --target aarch64-linux-android --target-dir spike/device-authentication/target-android` | 共享服务及存储依赖交叉编译通过，不含 Windows 原生后端 |
| 修改的 17 个 Rust 文件单独 `rustfmt --check`，以及 `git diff --check` | 通过 |

其余包的续跑命令为：

```text
cargo test --locked -p synra-plugin-api -p synra-plugin-cli -p synra-plugin-host -p synra-plugin-sdk -p synra-plugin-ui -p synra-protocol -p synra-registry -p synra-store -p synra-testkit -p synra-testkit-alloc -p wit-codegen --quiet
```

Android 编译沿用[认证实验](../../synra-rust-core/spike/device-authentication/README.md)记录的 NDK r28b、API 24 环境变量。
本次没有前端改动，不额外重跑前端检查。

## 全量测试发现的既有断言问题

平台属性测试生成 `C:\tmp\nul.exe`，生产规则先按保留设备名正确拒绝，
原断言却仅接受可执行扩展名拒绝，因此误报失败；现允许这两种明确拒绝原因，保持“不能打开”的安全属性。
触发样本保存在 `synra-platform/proptest-regressions/proptests.txt`，没有修改生产拒绝规则或删除失败样本。

## 未完成边界

两个宿主还没有调用这套登记服务或接入认证传输，不能据此声称生产流量已经加密。
Android 等其他平台原生后端、加密文件降级、显式身份修复、首次核验、对端凭据绑定、配对取消与撤销及 Hello 复核仍待实现。
Windows 原生测试通过不代表 Android 运行通过，也不代表密钥后端具备 TLS 握手总时限。
本次仅开发与验证，不涉及发布。
