# 2026-11-03 插件权限页渲染崩溃修复

应用：`14aaf705c44f9c67ed1b29e66fb4dbb588b519af`；协议与检查脚本：`51c3ca19c89c870aef6b037572c0d3bb1a764c22`。均已推送对应仓库 `origin/main`。

## 原因与修复

真实 `PluginSnapshot` 的 `requested`、`denied` 使用 Rust serde 的 `skip_serializing_if = "Vec::is_empty"`；空列表会从 JSON 中省略。TypeScript 却把 `denied` 声明成必填，组件直接展开、读取长度并调用 `includes`，因此触发渲染异常。旧测试与浏览器预览一直显式提供空数组，未覆盖真实形态。

新增的缺省字段用例在修复前复现了 `props.plugin.denied is not iterable` 与未处理渲染错误。修复后，权限列表只从声明的 `requested` 读取；缺省表示没有请求，不再通过历史允许/拒绝记录猜测请求。拒绝列表缺省按空列表处理，首次决策、已授权查看、保存结果与外部事件均使用同一语义。

`RouterView`、组件更新、`subTree`、`parentNode` 报错属于本次排查的级联症状：优先修复权限渲染异常，再验证关闭、切页与重开。未改动 Vue 内部或通过隐藏控制台警告掩盖异常。本轮回归与浏览器路径均未再出现这些错误；没有声称覆盖所有可能的 Vue 内部异常。

## 同类排查

| 位置 | 发现和处理 |
| --- | --- |
| `PluginSnapshot.denied` | 修正为可选，组件显式处理缺省；新增首次、全量同意、无请求、保存后重开和事件刷新回归 |
| `UpdateInfo.addedCapabilities` | Rust 空数组省略而 TS 声明必填；改为可选。现有 `v-for` 可处理缺省，新增无新增权限的更新详情用例 |
| `IntentSnapshot.candidates` | 同样修正为可选；现有模板迭代可处理缺省 |
| 安装、移除、同步、配置 | 核对 warnings、leftovers、failures、set、unset 等字段；原有可选声明和调用方处理保留，纳入统一检查 |
| 预览内核 | 插件列表、响应与事件按真实快照规则省略空 requested/denied，防止预览继续掩盖问题 |
| 协议守卫 | 对 Rust DTO 中可以省略的字段检查 TS `?`，包含 Option 字段；用真实契约和把可选字段改回必填的负向用例验证 |
| 配对检查 | 原检查仍读取协议层的旧任务结构；改读 IPC 的真实配对 DTO，同时检查阶段、结果和可省略字段。删除内核中不存在的 PairRefusal 类型出口 |
| Node 警告 | 预览配置只在有 window 时读取 window.localStorage；Node 测试不触碰实验性全局 storage，保留浏览器拒绝存储时的既有处理 |

按职责拆开原约650行的界面契约与953行检查脚本；本轮修改、新增的 TS/Vue 文件均不超过300行。仍从 `@synra/protocol` 导入边界类型，没有在宿主复制 DTO 或改变 Rust 授权契约。

## 本轮验证

| 检查 | 结果 |
| --- | --- |
| 应用前端完整测试 | 66个文件、560项通过；最终测试输出无 warning、error 或 unhandled 文本 |
| 应用 lint、vue-tsc 与生产构建 | 通过 |
| 应用 shapes | 15项通过 |
| `cargo test --no-default-features --manifest-path src-tauri/Cargo.toml` | 371项通过 |
| `cargo check --manifest-path src-tauri/Cargo.toml` | 默认功能原生壳检查通过 |
| synra-js `tsc -b`、消费方类型检查、全库 lint | 通过 |
| 协议包测试、scripts测试 | 分别3项、56项通过；后者包含10项缺省字段契约检查 |
| synra-js shapes | 15项通过，包含字段可省略性 |
| 浏览器实际操作 | 查看全量授权插件、编辑/取消、首次授权、全选保存、关闭与重开通过；控制台13条debug，0 warning、0 error |
| Android Release | `pnpm exec tauri android build --apk --target aarch64 --ci` 通过；APK签名验证通过，仅含arm64-v8a Native库 |

截图和控制台原始记录见 [验收证据](../ui-design/evidence/2026-11-03-permissions-fix/README.md)。浏览器使用假内核，但现在其插件快照确实省略空字段；真实形态的依据来自 Rust serde 声明。未连接卓易通设备完成安装和权限流程实测。

## 复测安装包

已上传 [Synra-0.1.0-arm64-20261103-14aaf70.apk](ftp://192.168.0.201/Synra/Synra-0.1.0-arm64-20261103-14aaf70.apk)，保留原有版本文件。上传后回读远端完整内容，大小与SHA-256均与本地一致。

- 大小：51722156字节，约49.3 MiB。
- SHA-256：`3A0F8AE1E1ABA1E28C8EDDBA59A4D7D90560D1626811525C21535218CD4CA5C7`。
- 本地产物：应用仓库的 `src-tauri/gen/android/app/build/outputs/apk/universal/release/app-universal-release.apk`。
- 本轮未修改Rust源码；core原有未提交的 `crates/synra-store/src/db/migrations.rs` 保持原样，本地产物依赖该工作区。

## 提交凭证

| 仓库 | 提交 | subject | 实测字符数 |
| --- | --- | --- | --- |
| synra-js | `51c3ca1` | `fix: honor omitted fields in IPC contracts` | 42 |
| synra-app | `14aaf70` | `fix: handle omitted plugin permission lists` | 43 |

两个提交均 body 为空，普通 push 到对应 `origin/main` 成功。实际执行的计数命令：

```powershell
[System.Globalization.StringInfo]::new('fix: honor omitted fields in IPC contracts').LengthInTextElements
[System.Globalization.StringInfo]::new('fix: handle omitted plugin permission lists').LengthInTextElements
```
