# 固定开发工具链与独立 MSRV 验证

日期：2026-10-26。对应 [19 D6](../19-roadmap.md)、[06 §7.0](../06-workspace-layout.md)
和 [ADR-022](../20-adr.md)。

## 配置与边界

core 与 app 各自根目录新增 rust-toolchain.toml，固定相同完整 Rust 版本、minimal profile、
Clippy 和 rustfmt；版本只在各自可独立使用的仓库文件维护，不在主题文档重复抄写。
这次固定的编译器与之前本机 stable 的 rustc 版本和 commit hash 一致，不借此重新格式化整个仓库。
本地安装命令与 CI 开发安装步骤均为在相应仓库目录执行
`rustup toolchain install --no-self-update`，app 的父目录配置同样适用于 src-tauri。

core 的检查与 guards job、app 的 shell job 均从仓库文件选择开发工具链，
Android target 安装也在 core 目录执行，避免为全局 stable 安装后却由另一编译器检查。
app 增加独立 MSRV job；两仓库的 MSRV 安装步骤用 Python 标准库 tomllib
以二进制模式读取各自 Cargo.toml 的 workspace.package.rust-version，
通过 job output 传递给显式 `cargo +<msrv>` 检查。
声明仅从两段值补成同一最低版本的三段值，不提高最低版本。
core 检查整个 workspace 全部目标，app 检查无默认特性的全部目标；
相关 Cargo 主检查使用 --locked，避免验证期间改写依赖解析。

未修改两个 release 工作流，也未修改现有 workflow_call 触发与发布编排。
本轮没有触发远端 CI；已有入口仅在发版工作流调用时执行，不为验证本改动创建 tag。

## 回归发现的内存流缺陷

全量测试中一次取消用例等待终止帧超时，促使复核共享内存流。
读取原先把条件变量唤醒后队列仍为空直接当成超时，忽略了无数据通知与虚假唤醒；
关闭状态使用另一互斥锁，也存在检查关闭与进入等待之间丢失通知的窗口。

新增受控唤醒用例在保留旧等待逻辑时立即失败，修正后通过。
内存流现在将队列和关闭状态放在同一互斥锁中，通过条件谓词等待，不因通知重置剩余超时。
端点关闭会唤醒读者与已被满队列阻塞的写者，缓冲字节仍可读完再返回 EOF；
空读写立即返回，不等待缓冲条件。
原有完整取消与确认断言保留，没有以重试单测或放大业务超时作为修复。
这项传输原语修复独立于工具链配置提交，排查过程另记入 core 的 NOTES。

## 验证

actionlint 1.7.12 对两个修改后的 CI 工作流通过；仅检查目标工作流，
没有修改发布 YAML 来消除检查问题。
直接从工作流读取实际 setup 命令，在 Git Bash 执行并核对 job output、
开发工具链仍保持固定版本以及显式 MSRV 对应的 Cargo 版本。
Windows Python 仅做 Unix 换行适配，不将此验证当成 Ubuntu runner 的执行记录。

固定版本下完成 core workspace 全量测试，含 core 815、store 286、protocol 244、
IPC 123、daemon 256 项单测及其他 workspace 包、集成测试和 doctest。
workspace 全部目标 Clippy 通过；内存流修正后按影响范围复核 core 全部目标 Clippy。
真实 Wasm feature 测试 352 项通过、1 项原有 canonical ABI 字符串返回夹具用例仍忽略，
不声称该忽略用例已经覆盖。

应用后端 325 项测试通过，开发版本默认 Tauri 检查和无默认特性全部目标 Clippy 通过；
最低版本检查使用独立 target 目录，避免在同一构建缓存中反复切换编译器。
未改动前端、IPC 形状、JS 包或依赖解析，复用上一阶段未受影响的前端与 JS 验证。

feature/target 矩阵先完成 7 行、4 行明确跳过、0 失败：Android 协议、传输、存储、内核与真实引擎宿主，
以及无引擎宿主和独立 CLI 均通过；未实现的四种组合保持显式 SKIP。
内存流修正后另按影响范围复核 Android 内核编译，当前源码通过。
12 项架构守卫通过，包括 core 最低版本全部目标检查、应用调用方构建、101 份文档链接与数量检查；
应用最低版本也在内存流修正后再次定向编译通过。Rust 改动的格式及各仓库 diff 空白检查通过。

## 剩余范围

当前配置和命令在本机 Windows 上验证，未触发 GitHub runner，也未完成 macOS/iOS 运行。
本机仅找到 Android NDK，工程指向的 SDK 与 adb 不存在，不能验证原生 Keystore 或设备行为。
D1 的 Android 后端与设备运行、整库或平台恢复及完整业务链路继续保留，
D2–D5 尚有未完成项；本阶段不构成完整目标完成，也不包含发布工作。
