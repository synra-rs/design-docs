# 10 · 宿主环境与响应式布局分离

状态：计划。优先级：P2。范围：App 启动环境、布局和返回/退出处理。

## 1. 当前证据

[App.vue:83](https://github.com/synra-rs/synra-app/blob/f7c6d1d/src/App.vue#L83) 通过移动 UA 或 maxTouchPoints > 0 推断 phoneLike，再结合 Tauri 标志和 64rem 媒体查询得到 isMobile。随后同一个 isMobile 同时决定布局、全屏和双击退出。

窄桌面窗口、触屏 Windows 设备、Android 横屏都可能触发与真实宿主平台不一致的路径。当前媒体查询把匿名函数注册为监听器，却在卸载时 removeEventListener('change', apply)，移除的不是同一个函数。

onMounted 中等待原生监听注册后才继续设备与设置读取；挂接失败或组件已卸载时，启动与清理也需要独立处理。此项为代码结构风险，不能据此认定所有移动构建当前均会失败。

## 2. 目标模型

### 2.1 宿主事实

HostEnvironment 来自可信宿主适配：browser/desktop/mobile、platform、可用窗口动作和返回机制。复用已存在的 Rust 平台事实及 Tauri 检测函数，主窗口与插件注入环境使用同一含义。

浏览器预览显式使用 browser profile，不把缺失宿主 API 当成移动设备。UA 仅用于已有的 WebView 渐进增强或显示诊断，不作为授予原生生命周期行为的依据。

### 2.2 视口布局

ViewportLayout 只描述 compact/wide、导航呈现方式和断点变化。maxTouchPoints 可帮助选择交互目标尺寸，但不能决定 mobile host。断点保留为设计系统的具名布局常量，不因是一个数值就抽成用户配置。

桌面窄窗口可以显示紧凑布局，仍保留桌面关闭语义；Android 宽屏可以显示较宽内容，仍使用移动端返回策略。rail 只从布局派生，避免多个 watch 和 apply 同时维护同一事实。

### 2.3 返回和退出状态

抽出 useHostBack/等价协调器：先关闭当前对话框，再处理实际导航栈，最后按宿主策略退出。双击退出窗口使用单一具名常量；状态通过注入 now_ms 验证，不由多个定时器和时间戳各自判断。

`src/app/back.ts` 目前按 path/query/hash 判断是否主路由，这可保留为页面语义辅助；不能将“非主路由”直接等同“router.back 一定有可返回历史”。处理直接深链进入子状态时的确定导航出口。

原生 PluginActivity 的系统返回与主窗口返回分开承载，复用语义而不重复抢占事件。CSS safe area 仍处理显示，不作为原生 API 失败时一切正常的证明。

## 3. 实施步骤与涉及文件

1. 盘点 App.vue 对 isMobile/rail/isTauri 的全部消费方，列出哪些需要平台事实、哪些只需要视口。
2. 在 `src/app/` 建立宿主环境适配与 viewport composable，复用现有 `ipc/tauri.ts` 检测和 Rust 环境数据，不新建第二套 OS 字符串推断。
3. 以具名 handler 注册媒体查询，并让注册函数返回 disposer；处理初始化后立即卸载、异步注册晚到的情况。
4. 将原生全屏、返回/关闭挂接移到宿主生命周期协调器。失败单独诊断，设备/设置初始化不依赖该监听成功。
5. 更新 App.vue，仅组合布局、状态读取和生命周期服务，避免继续添加新的平台 if 分支。
6. 同步浏览器预览、主窗口与插件页面环境说明，检查 `tests/app/back.test.ts` 和 mounted App 测试。

## 4. 验收矩阵

| 宿主/情形 | 预期 |
| --- | --- |
| Windows 触屏且宽窗口 | 桌面关闭语义，布局随宽度变化 |
| Windows 非触屏且窄窗口 | 紧凑布局，不自动全屏，不套用移动退出 |
| Android 横屏/大屏 | 移动返回不受断点变化取消 |
| browser 移动 UA | 可预览紧凑界面，不调用原生窗口 API |
| 弹窗打开、内部导航、根页 | 按明确优先级返回，无重复退出 |
| 直接进入带 query/hash 的页面 | 没有历史时仍有确定出口 |
| 注册前卸载、重复挂载 | 媒体和原生监听均释放一次，无晚到泄漏 |
| 原生 API 注册失败 | 诊断可见，设备/设置初始化继续进行 |
| 退出时间窗口边界 | fake clock 确认临界点行为，不依赖真实 sleep |

## 5. 风险与完成条件

Android 的全屏、系统返回和 Activity 行为必须在实际构建中验证；jsdom 的 matchMedia 和模拟 Tauri 只能验证编排。移动构建先更新 dist 再打包，避免用旧前端验证新逻辑。

完成条件包括桌面触屏/窄窗口场景及 Android 返回的独立证据。不能把把 phoneLike 提取成一个工具函数视为完成，必须消除平台与布局的混用。
