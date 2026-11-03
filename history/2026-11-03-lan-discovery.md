# 2026-11-03 LAN发现与配对设备可达性

应用：`2d249cc96508948fb820494d73cafd7e46256cd4`；核心：`f773cbfba7659e5a72aeb8f8adc94bf5826aa660`。均已普通push到各自的 `origin/main`。

## 症状与证据

PC对安卓请求插件清单得到TCP连接拒绝（10061）；安卓主动扫描时PC不断记录 `lan heard 192.168.0.184:40779`，安卓端只有 `lan query`；双方已配对、应用打开仍显示离线。

`lan heard` 的端口是公告中的TCP监听端口，不是查询实际使用的UDP源端口。它证明收到了发现公告，不能证明TCP业务连接成功。

本机同时存在WLAN（192.168.0.194/21）与虚拟网卡Meta（198.18.0.1/30），因此不能假设有限广播一定走到WLAN，也不能从默认路由猜一个/24广播地址。

## 根因与修复

1. **UDP回程错误。** worker丢失 `recv_from` 的源地址，再从候选表取地址作回应目标；候选表保存的是TCP监听端口。现在接收结果分别保留公告处理结果和查询UDP回程地址，只有Query产生回应，Response/Offline不触发回应循环。
2. **重复扫描不应答。** 旧实现只从New/Updated提取回应目标，公告不变的第二次查询没有回应。现在每次非自身Query都回答。
3. **多网卡广播。** 旧注释声称255.255.255.255会经所有接口发出，代码没有枚举接口。现在通过 `if-addrs 0.15.0` 取得运行中的非点对点IPv4接口及其真实广播地址，去重并保留有限广播回退；绑定特定地址的socket只使用对应接口，避免loopback测试向外网接口发送。
4. **配对地址过期。** 应用监听器每次启动绑定随机TCP端口，同步计划、传输、远程调用却只读配对时保存的地址；入站配对记录也可能根本没有地址。现在共用peers地址解析，先确认已配对且未拉黑，再优先使用近期发现的监听地址，缺失时才尝试最后保存的地址。同步计划仅对未拉黑设备请求实时插件清单。
5. **入站临时端口覆盖。** 接收TCP连接时 `peer_addr` 是对方本次拨号的源端口，不是其监听端口。Host现在区分入站/出站；入站握手保留已发现监听端点，出站保留实际拨号目标。拨号后同时核对Hello中的设备ID与请求对象一致，拒绝回答成另一设备的端点。
6. **在线定义与刷新。** 会话按任务建立，空闲且已配对不应永久离线。设备卡现在参考近期发现或未过心跳期限的有效会话；保存的旧地址本身不代表在线。重复公告刷新发现记录；告别与90秒候选TTL届满通知界面，超时只发一次状态变化，重新发现可恢复在线。配对关系不因离线过期。

核心socket按发送/接收与网卡枚举拆开，worker按服务与循环拆开；应用设备模块分为卡片与扫描流程，公共调用路径不变，没有增加IPC字段或兼容旧协议。

## 回归与验证

先加入真实TCP监听器和独立UDP查询socket的回归，旧实现明确失败：回应目标为公告的TCP端口，而预期为UDP源端口。旧测试反而在公告的TCP端口绑定了UDP接收器，验证了错误的模型；已用真实协议流替换。

新增覆盖重复查询、Response/Offline不应答、发现过期/告别/重新上线、入站监听地址保留、Hello对象不符拒绝、入站配对无保存地址、未配对/拉黑不能借公告拨号。同步集成测试保存旧端点、关闭旧监听器、发现新监听器，通过真实TCP握手和插件清单应答，确认同一插件不再被错误判为离线且需要文件。所有新增网络等待有期限，端口由系统分配。

| 实际执行 | 结果 |
| --- | --- |
| `cargo test -p synra-core --lib --quiet` | 758项通过 |
| `cargo test --no-default-features --manifest-path src-tauri/Cargo.toml --quiet` | 378项通过 |
| `cargo check --manifest-path src-tauri/Cargo.toml --quiet` | 默认功能通过 |
| `pnpm test` | 68文件，568项通过 |
| `pnpm run lint` | 通过 |
| `pnpm run build` | facade、vue-tsc、生产构建通过 |
| `pnpm run shapes` | 15项通过 |
| `git diff --check` | 通过 |

额外 `cargo clippy -p synra-core --lib --no-deps -- -D warnings` **未通过**：未修改的 `link.rs:370` 文档后空行，`listener.rs:602` needless_borrow与`:605` let_unit_value。拆分时引入的wildcard导入已改为显式导入；所触及连接模块的两个已有missing_const_for_fn也已修正。没有关闭或抑制Clippy规则。

## 范围与复测

ADB当前没有连接设备。本轮证明的是Windows本机真实UDP/TCP回归，没有声称卓易通网络、无线接入点、Windows防火墙或安卓后台限制已通过实机验收。没有通过关闭防火墙或改配对记录绕过问题。

PC需重新启动包含新Rust代码的开发进程或使用新构建；安卓安装本轮包。两端打开后分别主动扫描，安卓应能收到回应并出现 `lan heard`；设备详情应显示当前监听地址，插件同步刷新计划应取得真实清单。若仍失败，新的同步错误会带出实际拨号地址，供与当前公告对照。

核心工作区原有未提交的 `crates/synra-store/src/db/migrations.rs` 保持原样，未暂存、未提交。本地构建仍使用该工作区，产物不是只由上述两个提交组成的纯净checkout。

## 提交凭证

| 仓库 | 提交 | Subject | 实测字符 |
| --- | --- | --- | --- |
| core | `f773cbf` | `fix: repair lan discovery reply routing` | 39 |
| app | `2d249cc` | `fix: resolve paired peers from fresh discovery` | 46 |

两次提交均只有subject、body为空。分别实际执行的字符校验：

```powershell
[System.Globalization.StringInfo]::new('fix: repair lan discovery reply routing').LengthInTextElements
[System.Globalization.StringInfo]::new('fix: resolve paired peers from fresh discovery').LengthInTextElements
```

core第一次SSH push连接被关闭，第二次普通push成功；app普通push成功。未使用force、跳过校验或改远端。

## Android复测包

`pnpm exec tauri android build --apk --target aarch64 --ci` 通过，仅含 `lib/arm64-v8a/libsynra_app.so`；`apksigner verify --verbose` 通过，v2签名有效。

已上传 [Synra-0.1.0-arm64-20261103-2d249cc.apk](ftp://192.168.0.201/Synra/Synra-0.1.0-arm64-20261103-2d249cc.apk)。先上传独立 `.uploading` 文件，检查远端大小并完整回读计算SHA-256，匹配后才改为最终文件名；保留旧包。

- 大小：51825164字节，约49.4 MiB。
- SHA-256：`41DFF39D895763C79D82CE4117EBD6C8ACCA79B9331D67527785E3E60B5FD10C`。
- 本地产物：应用仓库 `src-tauri/gen/android/app/build/outputs/apk/universal/release/app-universal-release.apk`。
- 构建仍有Tauri关于 `com.synra.app` 后缀的既有macOS命名建议；没有为消除提示而改变应用标识和数据目录。

本记录的提交subject为 `docs: record lan discovery and endpoint fixes`，实测45字符，校验命令：

```powershell
[System.Globalization.StringInfo]::new('docs: record lan discovery and endpoint fixes').LengthInTextElements
```
