# Hybrid 安装包与实际 ZIP 验证

本轮接通作者 build、verify、tgz 与原生安装的产物格式；不代表官方插件的手机 UI、跨设备动作或鼠标输入已经实现。

## 当前契约

- build 输出完整 `dist/` 包：manifest 快照、完整 headless 组件及可选 UI ZIP；artifact 路径相对包根。
- hybrid 默认直接打包静态 `ui/`；框架构建由 Cargo metadata 显式声明 source、directory 和 argv command。
- 命令不自动安装依赖或选择包管理器；作者的本地程序只应写自己的 UI 资源目录。
- 组件、UI 和最终包全部校验后才整体替换输出；失败保留上一代及其签名，成功重建移除旧签名。
- 跨进程锁防止同项目并发构建；Windows 使用备份、重命名、失败恢复，下一轮恢复中断发布并清理可确认归属的过期暂存目录。
- 未知非空输出、其他项目/插件的标记、链接输出目录均拒绝覆盖；不承诺非空目录替换期间没有短暂缺口或断电持久性。

## 共用校验

原先 CLI 将 UI artifact 当目录，原生安装却要求 ZIP；解包器也未使用已有的完整 expansion plan。
现在 CLI build/verify 与原生安装复用 actual ZIP 校验：实际目录记录、入口、路径、类型、大小、压缩比和 CRC。
原生安装在签名校验后、引擎编译及安装写入前验证已读取的 UI 字节快照，不创建验证用临时文件。

ZIP 读取器会用名称索引合并重复记录，因此先有界扫描中央目录，再比较真实目录项与索引数量；
普通和 ZIP64（包括 legacy 字段未饱和的 ZIP64）均先执行数量预算，防止索引分配绕过上限。
文件入口先有界读取到内存快照，使预检与解码面对同一份字节；验证后修改源文件不能改变被解包的内容。
已有字节快照的原生 prepare 直接借用，不为验证再复制一份 ZIP。
UI 编码 ZIP 至多 32 MiB，解包内容至多 64 MiB、2000 个文件、4096 个目录项。
路径统一拒绝穿越、链接、Windows ADS/保留名/尾部别名、ASCII 大小写冲突及文件占用目录前缀；
runtime artifact 不能覆盖包元数据或派生的 ui 目录。嵌套 artifact 路径由 CLI 和原生安装共同支持。

解包只接受不存在或空的真实目录，使用 create_new 写文件，CRC 或 IO 失败时清理本轮文件及新建目录。
原生数据库提交失败也清理派生 UI 与嵌套空目录，保留失败前的安装版本。

## 已取得的证据

- UI 模块 113 项测试通过，覆盖实际 ZIP、重复记录、ZIP64、压缩炸弹、损坏 CRC、路径别名、覆盖拒绝与失败清理。
- 原生 install 32 项定向测试通过，含真实 SQLite、嵌套产物、缺失 UI 入口在编译前拒绝及版本记录失败后的完整清理。
- 实际 CLI verify 8 项集成测试通过；旧目录夹具改为真实 ZIP，并保留越界目录链接与字节预算拒绝验证。
- 作者链路 3 项真实测试通过：SDK guest、headless 和 hybrid；hybrid 经 CLI build、verify、pack，生产 tgz 摘要校验/解包、真实 Wasm 引擎检查和 SQLite 安装，核对最终 UI 内容。
- 失败 UI 重建保留旧 manifest、组件、ZIP 和签名；成功重建去掉旧签名，重复构建相同 UI 得到相同 ZIP。
- 中英文公共文档 ready 与构建通过；仅保留在文档工作分支，不合并部署分支。

阶段门禁已通过：
- `cargo test --locked --workspace -j 1`：全工作区测试。
- `cargo clippy --locked --workspace --all-targets -j 1 -- -D warnings`：全工作区严格 Clippy。
- `cargo +1.88.0 check --locked --workspace --all-targets -j 1`：完整最低 Rust 检查。
- Android ARM64 的 core、host 和真实 Wasm runtime 编译。
- App 337 项原生测试、默认功能 all-targets 严格 Clippy 与 Rust 1.90 检查。
- 公共文档 `pnpm ready`、`pnpm build`；110 个设计/公开文档链接检查无问题。

后续的 CLI 路径收口以 CLI 全 targets 测试（156 项单元及 8 项实际命令测试）、严格 Clippy 重检；
有界 ZIP 文件快照以 UI 113 项、install 32 项、core/UI/CLI 严格 Clippy 和 1.88 检查重检。
App 在快照改动后重跑原生调用方测试；其余无变化的通过结果复用，不反复执行整套门禁。
构建器随后显式固定 ZIP 时间戳，定向打包测试、CLI 严格 Clippy 和 1.88 检查通过，避免默认值受依赖特性影响。
清理已不需要的 daemon 开发产物 2.4 GiB，保留依赖缓存；构建器也在持锁后主动清理可确认归属的中断暂存目录。
完整日志留在 core target 下的 hybrid 系列日志及 App src-tauri/target 对应日志。

## 全量检查发现的独立问题

IPC 属性测试把“命令名加后缀”假定为不存在的命令，实际 `plugins_update` 加 `s` 是合法 `plugins_updates`。
修正为验证返回名称与查询精确相等，保留生成的 proptest 回归种子；IPC 123 项测试及严格 Clippy 已通过。
这不改变命令注册表或授权行为。

## 后续

两个官方插件仍需 Vue UI、真实宿主能力和手机到电脑的认证链路；触摸板还需电脑输入后端、会话租约与按钮释放验证。
不能把本轮包格式和识别器测试宣称为手机已能复制电脑选中、打开电脑链接或控制鼠标。
