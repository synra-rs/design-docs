# 2026-10-29 · 第一批宿主界面基础整理

应用提交：`051bd07`，`refactor: polish host interface foundations`，GPG签名、43字符、无body，已推送应用main。
目标依据：[专业软件完成度](../ui-design/00-product-quality-bar.md)。沿用现有页面形式，处理构图、默认占位、状态和操作入口，不按Folo模板强行重建。

## 1. 已实现的行为

| 区域 | 改动 | 保留的契约 |
| --- | --- | --- |
| 壳层/四页 | 共用PageHeader，主内容上限1024px，14px正文，清除标题默认margin；列表项目与页头对齐 | 视口内主区/侧栏独立滚动，窄栏和手机底部导航 |
| 首页 | 紧凑连接面、局部空状态短句、任务命名；按真实传输对端显示名称 | 原有决策、拒绝、取消与记录读取；没有伪造完成历史 |
| 设备 | 配对按需展开；恢复运行任务/错误主动展开；收起保留目标和草稿 | 邀请导出、listen/connect、取消、扫描和取消配对调用保持原语义 |
| 插件 | 安装/查找/权限管理/发布者按需展开，常用和异常入口保留在外 | 安装和授权不合并，新增权限/健康异常仍可见，审计仍按需查询 |
| 表单/反馈 | 安装失败保留输入，成功收起后归还焦点；移除Modal移出flex内容流 | 默认保留插件数据；所有既有管理动作仍有入口 |
| 可达性 | 给插件开关/打开、设置开关补可访问名称；页头按钮手机至少44px | 未声称所有既有小按钮和开关触控面积已完成整改 |

使用就地展开而非统一Drawer，是本批最小且足够的结构调整；未引入新依赖或改组件库。
`force-render`保留表单/现有测试与草稿；展开本身不查询审计或注册表。以后若性能测量表明渲染有问题，再调整挂载策略。
列表项目的 `paddingInline: 0` 是对组件默认24px内边距的结构性调整，避免标题、输入和记录各自错位；没有复制配色或添加第二份主题。

## 2. 测试与验证

1. 先进行类型检查和受影响文件lint，局部执行首页、设备、插件、设置及locale测试。
2. 初次94项中1项失败：旧测试模糊匹配含Install的按钮，新增入口后匹配到了展开按钮；改用明确提交控件。
3. 新增4项行为测试覆盖按需展示/无额外读取、配对草稿保留、恢复任务自动可见、安装失败后重试与焦点归还；原有传输测试增加真实对端显示断言。
4. 新测试曾因脱离document的可见性判断和误选分段控件radio失败；使用 `attachTo: document.body`、具名文本输入并始终卸载后通过，经验记入AGENT。
5. 阶段全量：42文件、**377项测试通过**；`eslint . --max-warnings 0`、`pnpm run build`、`pnpm run shapes`通过。
6. shapes仍为8项，47个前端调用命令、67个调用点，对齐58个Native wrapper；没有增加IPC面。
7. `CARGO_BUILD_JOBS=1 cargo build --locked -j1 --manifest-path src-tauri/Cargo.toml` 成功，用42.44秒更新开发EXE。没有Rust源码改动，未重复未变的Rust全量测试。

日志在工作区 `.scratch2/ui-phase-a-{all-tests,lint,build,shapes,native}.log`，属于本地验证产物，不作为版本库依赖。
已有警告仍存在：生产JS主块约837.56kB（gzip261.17kB），Node测试环境localStorage实验性提示，以及Cargo bin/lib PDB同名提示；未通过放宽门禁或屏蔽警告处理。

## 3. 实际浏览器检查

使用本轮本地假内核界面与真实组件，检查390px中英文、768px窄栏、1024px最小桌面及默认桌面视口。
检查了主区 `scrollWidth/clientWidth`，实际字段和菜单、键盘Enter展开、配对和安装草稿收起后恢复；语言最终恢复“跟随系统”，视口覆盖已清除。
这些是浏览器界面证据，没有执行真实设备配对、插件安装、权限变更或删除用户数据，没有声称原生窗口交互通过。

截图目录：[2026-10-29-ui-foundation](./2026-10-29-ui-foundation/)。其中：

- [手机首页中文](./2026-10-29-ui-foundation/home-mobile-cn.jpg)、[英文](./2026-10-29-ui-foundation/home-mobile-en.jpg)。
- [手机插件中文](./2026-10-29-ui-foundation/plugins-mobile-cn.jpg)、[英文](./2026-10-29-ui-foundation/plugins-mobile-en.jpg)。
- [手机安装英文](./2026-10-29-ui-foundation/install-mobile-en.jpg)、[手机设备中文](./2026-10-29-ui-foundation/devices-mobile-cn.jpg)。
- [1024px插件管理](./2026-10-29-ui-foundation/plugin-management-desktop-cn.jpg)。
- [1280px插件默认页](./2026-10-29-ui-foundation/plugins-desktop-cn.jpg)、[1280px首页](./2026-10-29-ui-foundation/home-desktop-cn.jpg)。

尺寸实测：390px下首页中英文、设备中文、插件/安装英文、设置英文的主区均为 `clientWidth=scrollWidth=375`；
1024px设置/插件管理为769/769；768px插件窄栏为657/657；恢复默认1280px后为1025/1025。
375等数值包含滚动条稳定占位的影响，不误当作390px视口设置失败。

## 4. 资源清理与待续

Native构建完成后立即清理两个未使用的 `synra_app.lib` 路径；实际回收约2.12GiB，不把可能的硬链接文件长度相加计算。
保留开发EXE、PDB、依赖与增量缓存；清理后D盘可用约4.43GiB。
当前任务与后续范围见[11](../ui-design/11-current-progress.md)，本批不代表整套UI目标完成。
