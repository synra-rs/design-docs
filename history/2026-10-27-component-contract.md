# 组件验证入口与必需 headless 契约

本记录保存本轮实现和验证证据，不定义新的插件类型。
设计 13、15 与 22 的既定决策要求每个插件包含 Rust headless 组件；UI 是 hybrid 插件的补充。

## 修复的脱节

- CLI 原本检查完整组件，但原生 verifier 接受调用方提供的 imports 列表，安装还在签名校验前编译组件。
- ManifestContext 的可选 Implementation 布尔值只证明调用方填写了什么，不能证明实际导出或签名；ValidationLevel 也没有形成有效的二进制验证边界。
- 执行型 PluginSet 的直接装载没有检查完整 world，只有部分 lifecycle 导出的组件也能进入运行时。
- CLI 仍提供纯 UI 模板，公共 manifest 校验接受无 headless 的清单，与既定产品决策冲突。

## 当前实现

共享 `synra_plugin_host::component::inspect` 检查实际字节的组件编码、导入、完整 WIT 导出和签名及嵌套函数体。
大小上限复用引擎的 16 MiB 常量；错误在宿主边界分类，解析诊断最多保留 512 个 Unicode 字符。
检查不实例化组件、不执行初始化，也不生成本机机器码。
桌面真实组件测试还暴露上游 decoder 对重复导出类型身份的断言 panic；共享入口现将解析库的 unwind
转换为分类拒绝，不让半可信组件越过安装或装载边界。独立的实际组件回归覆盖此路径，未保留 panic 内容。

CLI 的 build 与 verify、原生安装 verifier、所有执行型 PluginSet 装载使用该入口。
Native Candidate 不再携带预解析 manifest 或调用方 imports；导入覆盖检查读取实际组件结果。
安装通过完整 verifier 后才交给所选引擎检查编译能力，安装拒绝前不会创建插件目录或版本记录。
ScriptedBackend 不执行 Wasm，保留为配额、生命周期等单元测试的独立接缝。

公共 manifest 校验统一要求 headless。CLI 只保留 headless、hybrid 两种模板，build plan 先执行公共校验；
CLI 参数、build、verify 和真实安装都有纯 UI 拒绝测试，中英文作者说明同步修正。
WIT 仅修正插件类型说明，函数与类型签名没有改变；全部生成资源已同步。

测试夹具从核心模块头或部分手写 lifecycle 组件改为生产 WIT 编码的完整组件。
无限循环组件仍在真实 Wasmtime 中验证指令预算、超时、实例恢复和对其他插件的隔离，没有放宽断言。

## 已有证据与阶段状态

- 全工作区 all-targets 严格 Clippy 已通过。
- API guard 扫描 454 个文件，无违规；六项 WIT 生成物只读检查通过。
- 缺少 on-event、错误参数类型、无导出组件、核心模块、超大输入与实际嵌套函数体错误均有拒绝测试。
- 安装测试使用文件 SQLite，并验证坏组件或签名在编译前拒绝、不留下安装目录与版本记录。
- 中英公共文档的 ready 与 VitePress build 通过，推送仍限工作分支。
- 全工作区测试通过；随后补充的 decoder 边界与 WIT metadata package 拒绝，共九项组件测试通过。
- 最终变更后 Host 默认配置 345 项、Core 安装相关 29 项及受影响 all-targets 严格 Clippy 通过。
- App 的 337 项测试、无默认配置与默认配置 all-targets 严格 Clippy、默认配置 MSRV 1.90 检查通过。
- 12 项 guards 通过，包含真实 App 构建和 Core workspace MSRV 1.88；最终 Core/Host all-targets MSRV 定向复查通过。
- Android ARM64 的 Core 与启用 Wasm 的 Host 检查通过，使用既定 NDK。
- SDK guest 已重新构建；真实 Host 的 SDK 回归通过，CLI 两项真实作者链路测试通过，含失败重建保留旧产物。
- 设计与公共文档共 109 个 Markdown 文件的本地链接检查通过；未修改的应用前端检查复用上一阶段结果。

桌面失效的 dev 包产物已通过 Cargo 包级清理移除 5.1 GiB，保留依赖缓存、源码与运行数据。
清理先核对目标的绝对路径、忽略规则和 dry-run；没有等待磁盘空间耗尽。

本轮不代表完整 UI 构建、手机插件界面、跨设备操作、原生输入、性能验收或 M2 完成。
Starter、Touchpad 仍按 [29](../29-official-plugins.md) 继续开发。

## 阶段提交

- Core：`71c628e`，`fix: enforce the plugin component contract`，实测 42 字符。
- App：`c1bbea1`，`test: use complete plugin component fixtures`，实测 44 字符。

以上提交为 GPG 签名、单行英文 subject、空 body；设计与中英作者文档在对应仓库单独提交。
