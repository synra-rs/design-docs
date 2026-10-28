# 2026-10-28 · 插件窗口与 IPC 修复

## 真实使用暴露的问题

Starter 的源码根目录声明了 ui.zip，但不是完整安装产物；应安装 CLI 生成并验证通过的 dist。
本机原有开发库又缺少新增的 plugins.authority_revision 和 plugin_authority_clock，
即使授权已完成，Open 仍在读取授权快照时失败，通用提示隐藏了实际的 SQLite 错误。
本次先做 SQLite 一致性备份，再在单个事务内补齐本机缺失结构与当前触发器；
校验原有 25 张表的 53 条记录、外键和完整性，设备身份、插件与权限记录未改动。
这是一次本地开发环境修复，没有给未发布数据格式添加兼容分支。

主窗口另有独立的配置契约问题：Rust 序列化省略空 overridden 数组，前端却把它当必填数组展开。
ConfigSnapshot 现在总是输出该字段，空配置也可往返序列化；没有在前端吞掉格式错误。

## Windows 窗口与样式

同步 Tauri command 调用 WebviewWindowBuilder 会在 Windows 上死锁，只创建窗口外壳。
此前 smoke 从 setup 直接调用 Rust wrapper，虽然验证了页面与 Wasm RPC，却没有覆盖前端真实 IPC 的调度线程。
Open 现为异步 command；开发 smoke 通过主文档的 Tauri invoke 打开插件，等待返回后显示验收结果，
并检查组件库动态 style 是否被浏览器接受。该入口仍只允许隔离的 debug 验收 profile。
参考 [Tauri 的 Windows 已知限制](https://docs.rs/tauri/latest/tauri/webview/struct.WebviewWindowBuilder.html#method.new)。

主窗口原本允许内联样式，但 Tauri 自动添加的样式 nonce 使浏览器忽略 unsafe-inline，
当前组件库插入的样式又未携带 nonce，因此样式消失。仅关闭 style-src 的自动 CSP 改写，
保留显式样式策略、脚本保护和其他限制；原生测试检查实际配置与嵌入的 HTML。

## IPC 控制台报错

connect-src 'none' 阻断了 Tauri 自己的 fetch 传输；框架随后回退 postMessage，
所以页面能显示、调用能成功，但控制台仍出现 CSP 错误和回退警告。
现对主窗口允许 ipc: 与 http://ipc.localhost，对使用 HTTPS 映射的插件窗口允许 ipc: 与 https://ipc.localhost。
不允许通配 HTTP、WebSocket 或插件自身 origin 的直连，ACL、界面边界、会话和 capability 校验保持生效。
参考 [Tauri CSP 配置示例](https://v2.tauri.app/security/csp/)。

Starter HTML 声明有效的内联 SVG favicon，避免浏览器额外请求 favicon.ico。
DevTools 自动请求的 .well-known/appspecific/com.chrome.devtools.json 继续被连接策略阻止；
这是开发者工具探测，不为消除其警告放开插件网络。

## 验证与边界

- Core IPC 124 项测试及严格 Clippy、Rust 1.88 检查通过；提交前再次定向验证 2 项配置序列化测试。
- App 69 项插件 UI 测试和 20 项无 Tauri 配置测试通过，包含真实 Starter、Wasm、页面授权与原生 ACL。
- 原生 CSP 测试精确检查本机 IPC 白名单，同时确认脚本保护和实际嵌入样式策略。
- App all-targets 严格 Clippy、Rust 1.90 检查、8 项 shape guards 及普通开发版构建通过。
- Starter 18 项 UI 测试、类型检查和 lint 通过；CLI 重建 dist 并验证 4 项通过、无警告或失败。
- 直接检查打包后的 HTML，确认 favicon 为有效 SVG data URL，没有新增图标网络资源。
- 用户已确认 Windows 页面显示；新 IPC 策略下控制台的完整复测仍需重开页面确认，不能以单元测试替代。

正常开发可执行文件已恢复为普通配置；没有发版、打 tag 或部署。
已安装 Starter 的 favicon 要经正常本地包更新取得，新构建的 dist 仍为开发版本。

收尾时只清理两个桌面运行用不到的 Windows 静态库产物，保留 exe、DLL、调试信息和依赖缓存；
实际释放 4549730304 字节，磁盘剩余 10792316928 字节。

## 提交记录

| 仓库 | Subject | 实测字符数 |
| --- | --- | --- |
| Core · ba2c776 | `fix: serialize empty configuration overrides` | 44 |
| App · 5b13c6f | `fix: align plugin windows with native IPC` | 41 |
| Starter · d71773a | `fix: provide an inline plugin favicon` | 37 |
| Design | `docs: record plugin window repair findings` | 42 |

计数使用 PowerShell `[System.Globalization.StringInfo]::new('<subject>').LengthInTextElements`，
提交使用英文单行 subject、无 body、GPG 签名；按本次会话授权推送。
