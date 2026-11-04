# 2026-11-04 · Touchpad 输入会话与移动端返回

本轮按用户选择补齐触摸板页面、Wasm 手势与电脑原生输入链路，不以界面动画冒充电脑操作。

## 实现

- Touchpad 0.2 为独立 Vue 页面与 stateful Wasm：手机选择已配对电脑，显示等待、已连接和结束状态，提供五种手势说明。
- 完整触点快照进入 Rust recognizer，再通过受 `device:input` 约束的 `input-session` 导入进入共享服务。
- 发起端同时检查具体 `device:send:<UUID>` 范围。接收端检查配对、屏蔽、插件启用/可信、授权代次及独立 `platform:control-pointer` 权限。
- WIT 的 `@receiver-capability` 声明接收侧权限，代码生成器将其纳入清单许可集合；它不替代导入门控，也不增加可绕过确认的原生函数。
- 电脑每次显示宿主确认框，允许后才创建 OS pointer adapter 并回复接受。宿主在控制期间提供停止按钮。
- 输入复用 Hello 与 PeerLink；严格校验会话、授权代次、递增序号、整包命令及 delta。当前原生接收后端使用 Windows Enigo，其他系统明确拒绝。
- 输入队列最多 64 项，合并相邻移动/滚动的累加量，不越过按钮边界；每包至多 16 个操作。发送上限 60 Hz，心跳 250 ms，接收租约 1 s。
- 页面状态续租为 750 ms。关闭、隐藏、取消、断连、权限变化及队列异常结束控制；电脑独立释放所持按钮，不依赖手机最后一个 up。
- 停止与原生执行通过同一状态锁排序；新会话取得硬件所有权前，前一会话必须先完成释放。
- 拨号完成后重新检查取消/授权，旧连接不会产生晚到的确认请求；正常退出宿主时停止所有控制，并有界等待原生按键释放。
- 控制端以同一 Rust 单调时钟计算最近 120 项“宿主入队到执行 ACK”的 p50/p95/p99；不含之前的页面 IPC/Wasm 排队，完整触控采样延迟仍待性能验收。常规成功 frame/status 保留指标，不逐帧写持久审计；拒绝、错误及会话管理仍审计。

## Android 页面承载

插件使用独立 `PluginActivity`，不复用 `singleTask` 的 MainActivity。宿主在 WebView 外提供 48 dp 的“返回 Synra”栏，系统返回也结束该 Activity。
两种插件共用此入口，网页加载或执行失败不影响返回。失焦/销毁同时撤回输入控制和页面令牌。

Android 工程的源码、资源与 Gradle wrapper 纳入版本控制，生成的 Wry/Tauri 桥接、SDK 路径、签名配置及构建缓存继续忽略。
本地没有 release keystore 配置时仍可配置和构建 debug 工程。

## 已完成的验证

| 层 | 结果 |
| --- | --- |
| 共享内核 | 770 项 core、124 项 IPC、93 项 platform、156 项 CLI、393 项 plugin-host、6 项 SDK、56 项 codegen 测试通过 |
| Rust lint | core/platform/plugin-host/IPC/codegen 的 lib Clippy，以 `--no-deps -- -D warnings` 通过；修正原 listener/link 的三项既有问题 |
| 输入双端 | 12 项相关测试，真实 TCP/Hello：同意前无 OS adapter、拒绝、取消、严格序号、重放、撤权、源页面租约、接收租约、晚到拨号与退出时松键 |
| 应用壳 | 381 项默认无 Tauri 测试通过，默认功能 cargo check 通过 |
| 前端 | 574 项测试、lint、类型与生产构建、15 项 shapes 通过；57 个命令已注册，前端调用其中 46 个 |
| JS | 类型/消费方/文档示例、15 项 shapes、lint 通过；设备类型允许发现阶段的 desktop/headless，而不伪造远端 OS |
| Touchpad | 22 项 Rust 测试、7 项 Vue 测试、native 与 wasm32 Clippy、fmt、CLI 组件构建与 verify 通过 |
| Starter | desktop 发现类型回归在内的 19 项 UI 测试、类型、lint、构建通过 |
| 实际插件包 | 生产 `.synra` 打包、文件安装、显式权限批准、启用、页面 nonce/RPC、真实 Wasm、双端 TCP、接收同意、五种手势及拖拽中停止控制通过 |
| 页面布局 | 390×748 CSS viewport 无横向溢出；五种说明在首屏，触摸区域 354×413 CSS px；浏览器采集 0 warning / 0 error |

完整包验收使用 app 的显式 `touchpad-plugin-tests` feature，设置 `SYNRA_TOUCHPAD_PACKAGE` 为独立插件的 CLI-built dist。
默认测试替换的是最后一层 pointer adapter，记录命令和释放；上层安装、页面 RPC、Wasm、授权与 TCP 均为生产入口。
它证明控制链路，不证明物理鼠标已执行。

## 实机边界与构建提示

现场没有 adb 设备或已有 AVD，因此 Android/卓易通的返回、系统手势及连续触控仍需实体设备复测。
Windows 专用测试窗口已建立，但 Computer Use 连续两次无法抓取该窗口，报 capture timeout；没有继续向未观察到的桌面发送鼠标输入。
Windows 原生输入后端已编译，未把记录 adapter 的通过写成真实鼠标验收。

Android 构建已验证 `PluginActivity` 的 Kotlin/Manifest 接入。构建仍有原有 AGP legacy DSL、生成的 Wry deprecated API、JDK 21 编译 Java 8 及 `.app` 标识符提示；未静默压制这些提示。
Windows 构建仍有既存 bin/lib 同名 PDB 提示，涉及 Android `.so` 名称，不在本轮单独改名。

本轮提交排除用户原有 `synra-store/src/db/migrations.rs` 的迁移 17。首次两次签名等待 Pinentry 超时；2026-11-04 上午解锁后，已完成 GPG 签名提交和推送。
Touchpad 的 SDK、Cargo.lock 和 CI 作者 CLI 统一固定到 `ffec0a2dd3833346bccc6f75bb49447c4e2cc9eb`，临时本地覆盖已移除。
重新执行 locked Rust 检查、原生/wasm32 Clippy、22 项测试、Git 同版本 CLI 构建与 verify，均通过；manifest、Wasm、UI ZIP 与已交付包逐字节一致。

| 仓库 | 提交 | 内容 |
| --- | --- | --- |
| synra-core | ffec0a2 | 同意后启用的输入会话与 Windows 适配器 |
| synra-app | 581c90a | 通用控制确认、停止与 Android 插件返回 |
| synra-js | be4191f | 输入快照类型与平台能力声明 |
| synra-plugin-touchpad | f95d63e、a65ea16 | 手机页面、手势与真实输入通道，CI 所需 TOML 格式 |
| synra-plugin-starter | 9422c34 | 识别发现阶段的 desktop 类型 |
| synra-app | 3759c3a | `ai-docs/good-practices` 经验总结与索引 |

可复用方法另见应用仓库的 [开发经验](https://github.com/synra-rs/synra-app/blob/main/ai-docs/good-practices/2026-11-04-cross-device-development.md)。

## 用户复测

当前本地安装入口拒绝重复 id；已有 Touchpad 0.1 时，先在插件详情卸载旧版本并保留数据，再安装本轮 0.2 包。

1. 电脑和手机均更新宿主，均安装并启用 Touchpad 0.2，重新检查它新增的权限。
2. 在手机选择在线 Windows 电脑并连接；电脑未同意前手机保持等待，不能控制鼠标。
3. 电脑允许本次后验证单指移动/轻点、双指滚动/轻点和双击按住拖拽。
4. 拖拽中分别验证手机返回、切后台、断网及电脑停止；松键后再次连接必须重新得到电脑允许。
5. 分别打开 Starter、Touchpad，验证原生返回栏和系统返回均能回到 Synra。

## 交付文件

三份开发验证文件已上传至 `ftp://192.168.0.201/Synra/`，临时名上传后逐个完整 RETR 回读，大小及 SHA-256 均匹配才改为最终名；既有安装包保留。
本地副本位于 `D:/Projects/synra-rust/artifacts/20261104-touchpad/`，含使用说明、校验表和 FTP 校验记录。

| 文件 | 字节 | SHA-256 |
| --- | --- | --- |
| Synra-0.1.0-arm64-20261104-touchpad.apk | 52075200 | E1FB585E125519849603685866D9DCD80DB3E0695ADC543A7D22B1DB7B7A4DC2 |
| Synra-0.1.0-windows-x64-20261104-touchpad.exe | 43147776 | 39E835DD510D69852B1BD9C2173BAD3547B0CC27BF6F55A6D8B1D4C215C45032 |
| Synra-Touchpad-0.2.0-20261104.synra | 91590 | CC91D1F041370AD4A66E0F3E1223B52FF1AE7AEAAF55DEA285440736AC83F69C |

APK 的 v2 签名验证通过，只有 `lib/arm64-v8a/libsynra_app.so`；打包 manifest 确认 PluginActivity 为非导出 standard Activity。
Windows PE 的 Machine 为 x64，release EXE 构建成功。
