# 2026-10-29 · 第七批：扫描取消与任务归属

应用：`821dc9a`，`fix: complete device scan cancellation`，38字符、GPG签名、无body，已推送main。

本批补齐第六批设备界面依赖的扫描生命周期，不增加页面结构或更换组件库。当前规范与下一步见[11](../ui-design/11-current-progress.md)。

## 1. 原因与调度核对

旧取消只清除共享扫描标记，发现窗口仍等待到期；旧扫描完成无条件清除标记和截止值，并缺少结束通知。界面按钮恢复并不能证明后台结束。

第六批要求核对shell的同步函数调度。本次确认`devices_scan_start`原本已有`#[tauri::command(async)]`，Tauri会异步调度同步函数；没有发现需要修改的主线程调度缺陷，shell.rs未改动。

## 2. 内核契约

- `ScanTask`同时持有对外快照和本任务Cancellation；扫描准入检查/占位在同一个临界区，已有任务时返回带running ID的rate_limited。
- 取消在同一把锁中校验scanId并克隆令牌，释放锁后唤醒；取消不提前删除任务，重复取消仍只作用于匹配任务，过期ID返回invalid_argument。
- 发现等待复用已有Cancellation的条件变量，移除20ms睡眠轮询；预先取消的任务不发布发现需求。
- `DemandWindow`在正常退出或异常展开时compare_exchange释放自己发布的截止值；任务退出守卫只清除匹配scanId，在清除后发布ScanUpdated，再由既有全局事件入口读取权威快照。
- 取消没有终止进程级发现服务；它释放本次主动发现窗口，周期发现仍遵守既有调度，已经发出的UDP包不能撤回。
- 任务指标区分ran/cancelled/failed并保留候选数量、时长；Discovery对象销毁时请求后台工作线程退出。

缓存模式和零timeout不占用或打断主动扫描。默认窗口、协议上限及原有等待时长规则未变；没有引入协议DTO、兼容分支或新的页面订阅/轮询。

## 3. 界面与预览

取消被接受但快照仍有原任务时显示“正在等待扫描停止”，真实结束后才显示“扫描已停止”。读取失败不凭空宣称停止。

本地操作代数随新扫描ID变化；新的扫描清除旧取消反馈。旧取消响应晚到时，无论替代任务仍在运行还是已经结束，都不能把替代任务标成停止。

预览使用不同任务ID、拒绝过期取消与重叠扫描，缓存读取保留在途扫描；取消在下一个计时器回合结束并发出通知。900ms演示时序和立即返回的开始命令仍不同于Native，不拿预览时长推断真实性能。

## 4. 验证与修正过程

Rust迭代先通过无默认特性的tests编译，再运行设备命令定向测试：30项通过；涉及真实Discovery服务、内存SQLite、跨命令取消、原任务退出前拒绝重叠、正常/取消/异常结束通知和清理。

最终阶段检查通过：

```text
cargo test --locked --no-default-features --lib --manifest-path src-tauri/Cargo.toml -j1
  356 passed / 0 failed，4.64s测试时间
cargo check --locked --manifest-path src-tauri/Cargo.toml -j1
cargo clippy --locked --no-default-features --all-targets --manifest-path src-tauri/Cargo.toml -j1 -- -D warnings
rustfmt --edition 2024 --config skip_children=true --check
  src-tauri/src/state.rs
  src-tauri/src/discovery.rs
  src-tauri/src/discovery/tests.rs
  src-tauri/src/commands/devices.rs
  src-tauri/src/commands/devices/tests.rs
```

Rust调用显式设置`CARGO_BUILD_JOBS=1`。本批库单测全部通过；没有运行产生所有crate-type链接产物的完整cargo test、默认特性运行测试或移动端构建。

早期检查暴露found目标字段为u32、取消临界区显著Drop收紧lint，均修正后通过；全workspace的`cargo fmt --all --check`暴露已有大量格式差异，没有全仓重排，只格式化本次修改的五个Rust文件并检查通过。

前端局部类型与lint先修正，78项设备/预览/契约/事件定向测试通过；最终435项/45文件全量测试、全量ESLint、生产构建和8项shapes通过。覆盖新任务隔离、取消晚到、停止等待与预览拒绝路径。

生产主JS为867.14kB（gzip269.46kB），CSS18.11kB；构建仍有既有500kB块提示，Node仍有localStorage实验警告。本批没有根据包体数字宣称运行时性能提升。

本批没有重新进行浏览器截图、Native窗口操作、真实双设备握手或取消配对；第六批截图只属于当时的前端验证。

## 5. 构建空间与后续

本批检查后D盘可用2,676,797,440字节（约2.49GiB），没有重建桌面EXE，现有EXE仍是第五批配对流程版本。

此前自动审批两次拒绝删除无用staticlib，仅返回`blocked by policy`；没有绕过或再次尝试删除。两个路径仍为同一份约2.12GiB硬链接产物：

```text
synra-rust-app/src-tauri/target/debug/synra_app.lib
synra-rust-app/src-tauri/target/debug/deps/synra_app.lib
```

用户手动清理请求仍未收到完成回复；后续Native链接前必须重新核对空间。设置组织/局部失败、全局接收确认反馈及其他插件管理操作恢复流程继续开发，不发版、不打标签、不部署。
