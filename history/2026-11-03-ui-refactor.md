# 2026-11-03 界面流程重构实施记录

应用提交：`1228d7454be1de1981a2f94181f764d01d56f62c`。主题提交：`a82475494fa7663665ae269ae507eb50a6d7006e`、`0eeb959dfcfb0411b8fb7a4eeca536e28f1b56fa`。均已推送各仓库 `origin/main`。

方案与原审查见 [13](../ui-design/13-ui-refactor-plan.md)，实际画面见 [本轮截图](../ui-design/evidence/2026-11-03-refactor/README.md)。原有后端配对修复继续作为基线，本轮没有修改Rust源码、IPC契约或增加兼容旧流程的分支。

## 交付结果

| 路径 | 本轮行为 |
| --- | --- |
| 浏览插件 | 默认摘要列表；待处理和打开入口常显；列表和详情复用更多菜单。筛选和滚动不随详情展开而丢失 |
| 单个插件 | 桌面最大640px的详情抽屉，窄屏全宽；概览、权限、最近调用分开。通知和全局查找按ID进入同一详情 |
| 权限 | 已有决定先查看再修改；首次决策直接编辑。用途和完整范围同处一行，新增不预选；部分授权、取消、失败重试、外部事件和草稿保留有行为测试 |
| 本地安装 | 独立对话框，以原生插件包/文件夹选择为主，手动路径放高级展开。无虚构本地预检，失败保留路径；成功后直接进入权限决策 |
| 发现 | 一个注册表输入支持名称或准确ID，按钮/非组合态Enter提交。结果详情后安装，无第二个ID输入框；未配置、初始、加载、无结果、失败分别呈现 |
| 配对 | A显示本次码并等待/撤回，B显示码并接受/拒绝/拉黑；不从弹窗消失推断配对成功。全局弹层移出主布局网格，避免打开时挤动导航和内容 |
| 弹层冲突 | 配对优先，安装/权限/同步暂停展示而保留草稿，完成后恢复。只协调表现层，不建立后端配对队列 |
| 四页一致性 | 标题20px、正文14px、元数据13px；普通控件32px、紧凑28px，触控规则使用 `any-pointer: coarse`。去掉后代按钮强制高度、中文两字按钮插空格、巨大空容器及高强调诊断气泡 |
| 全局查找 | 使用组件库输入框，空词提供页面/设置入口；支持Ctrl/Cmd-K和 `/`，保留组合态Enter/Escape与选中项ARIA关联 |

## 结构与性能

- 以Vue组件、composable和现有Pinia状态分工；相同菜单、动作执行、错误转换、输入法判断及浏览器测试夹具复用，没有引入新的UI框架或通用表单配置系统。
- `PluginsPage.vue` 从约1500行降到约200行。首页、设置、诊断、假内核、长测试与shapes脚本拆开；检查 `src` 和 `scripts` 的TS/Vue/CSS文件，均不超过300行。
- 插件详情、安装、发布者界面按需加载，审计只在对应标签活动时读取；没有给每个摘要行预先挂载管理表单或执行检查更新。
- 诊断控制台移除第二个内核订阅，`events.ts` 明确处理 `debugWire`，避免每帧进入未知事件处理并重读设备快照。测试验证诊断帧会到达本地控制台流、不会产生额外IPC读取。
- 浏览器预览保留独立假内核，新增五项插件和配对两种角色的样例；守卫限制预览数据在假内核内部组合，Native仍选择真实桥接。

最终生产构建的部分输出如下。它们是本轮构建结果，不是整体性能提升或内存回落的证明。

| 文件 | 原始大小 | gzip |
| --- | --- | --- |
| 主入口 | 207.06kB | 65.66kB |
| 插件页 | 47.61kB | 14.20kB |
| 插件详情，按需加载 | 55.14kB | 17.19kB |
| xterm，首次打开诊断才加载 | 330.95kB | 83.01kB |

## 实际验证

| 检查 | 本轮结果 |
| --- | --- |
| 应用 `pnpm test` | 65个文件、554项通过 |
| 应用 `pnpm run lint` | 通过 |
| 应用类型检查和Vite生产构建 | 通过，包含在最终Tauri构建的before-build流程 |
| 应用 `pnpm run shapes` | 15项通过；保留命令名称/参数、夹具隔离、注入产物、对话框命名等守卫，尺寸守卫改为共享主题规则 |
| 主题 `pnpm --filter @synra/theme build` / `test` | 构建通过，81项测试通过，包含对比度和适配器约束 |
| 主题ESLint | `pnpm exec eslint packages/theme/src packages/theme/test` 通过 |
| Windows | `pnpm exec tauri build --debug --no-bundle` 通过，产出调试EXE |
| Android | `pnpm exec tauri android build --debug --apk --target aarch64 --ci` 通过，APK内只有 `arm64-v8a` Native库 |

浏览器实际检查：390×844、768×844、1024×844、1280×720、1904×844；包括五项插件、长英文名、待授权、更新提示、权限、两类安装入口、配对A/B、全局查找、首页与设置。中等/宽屏主内容的 `scrollWidth` 与 `clientWidth` 相等；英文390px设置也为375px/375px。桌面搜索图标实测28×28px，输入框边框为0且使用共享字体。

两张配对截图来自独立的浏览器样例，显示同一固定测试码，只证明角色、按钮和布局；没有把它们当成真实双端网络配对。部分授权、拒绝/拉黑、重复提交、安装失败保留路径、弹层打断保留草稿、更新、移除和完整发布者指纹另由挂载/状态测试验证。

## 本地产物与范围

均为调试产物，用于构建校验，未执行安装或启动真实应用。APK保留调试内容，不能作为发版包体积基线。

| 产物，相对应用仓库 | 字节数 | SHA-256 |
| --- | --- | --- |
| `src-tauri/target/debug/synra-app.exe` | 74374656 | `88A7107C98A4A3CCB4399DB34B09A8DFE926D7868B1316367AB8D7CD05164087` |
| `src-tauri/gen/android/app/build/outputs/apk/universal/debug/app-universal-debug.apk` | 1072544483 | `F4421F405D2443F47911E6ECAD33585EFDE54F1A48292BF635E10A40AEEB901D` |

当前ADB没有连接设备，Windows真实窗口、卓易通Android WebView中的双端配对、系统选择器、触摸/软键盘、200%缩放、1000项及长期资源压力尚未实测。本轮没有修改Rust/IPC，未重跑Rust业务测试；没有复用历史测试数量声称本轮通过。`synra-rust-core` 原有未提交的 `migrations.rs` 改动保留原样，未纳入本轮提交；本地产物以该本地工作区依赖构建。

## 提交凭证

三个代码提交的body均为空，均使用普通push到对应 `origin/main`，没有创建标签或强制推送。

| 仓库 | 提交 | subject | 实测字符数 |
| --- | --- | --- | --- |
| synra-js | `a824754` | `feat: unify control density across input modes` | 46 |
| synra-js | `0eeb959` | `fix: refine compact controls and touch targets` | 46 |
| synra-app | `1228d74` | `refactor: rebuild the host management interface` | 47 |

实际执行的计数命令：

```powershell
[System.Globalization.StringInfo]::new('feat: unify control density across input modes').LengthInTextElements
[System.Globalization.StringInfo]::new('fix: refine compact controls and touch targets').LengthInTextElements
[System.Globalization.StringInfo]::new('refactor: rebuild the host management interface').LengthInTextElements
```
