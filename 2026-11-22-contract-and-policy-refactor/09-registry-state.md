# 09 · 注册表配置与结构化工作流状态

状态：已实施，通过自动验收及 Windows 原生 WebView 验收。优先级：P1。范围：App 注册表命令、插件发现界面、共享 IPC 类型。实际提交、读写入口及验证见[实施记录](./IMPLEMENTATION.md)。

## 1. 当前证据

[registry-discover.vue:65](https://github.com/synra-rs/synra-app/blob/f7c6d1d/src/components/plugins/registry-discover.vue#L65) 用 message.includes('no registry is configured') 判断是否展示去设置入口。

[registry/transport.rs:13](https://github.com/synra-rs/synra-app/blob/f7c6d1d/src-tauri/src/commands/registry/transport.rs#L13) 实际将英文说明写进 params.why，message 来自 IpcError::unknown(Unsupported) 的翻译 key。配置为空时又使用另一句 names no registry。前端匹配的字段和内容都不是稳定业务契约。

同文件数组解析用 filter_map 丢弃非字符串项，再过滤空 URL。`[123]` 会变成“未配置”，而 `[123, 'https://example.test']` 会只使用有效项，隐藏配置损坏。该结论直接来自解析路径，本轮未运行该 Rust 场景。

[failureOf](https://github.com/synra-rs/synra-app/blob/f7c6d1d/src/features/plugins/actions.ts#L46) 保留 message，但把 params 转成展示 detail，业务原因在到达组件前也没有类型化表达。

## 2. 目标行为

| 状态 | 界面与操作 |
| --- | --- |
| 没有配置源 / 明确配置空列表 | 显示未配置，引导进入插件设置 |
| 配置类型或 URL 非法 | 显示具体配置项错误，引导修正，不伪装成未配置 |
| 已配置但源不支持搜索 | 说明该源能力限制，可保留按 ID 安装等实际可用入口 |
| 网络故障 | 保留原始 code、retryable、诊断与重试操作 |
| 完整性或验证失败 | 明确拒绝，不能通过换源绕过验证 |
| 正常搜索无结果 | 显示空结果，不展示配置缺失提示 |

## 3. 目标设计

### 3.1 严格配置解析

将配置读取与解析分离。使用 RegistrySource/RegistryConfig 领域值，明确 npm 与 static source 的类型、URL 结构和镜像顺序。保留现有 string/array 形式是否必要，依据真实设置写入方决定；开发期若统一为一种结构，则读写方同批更新。

数组中存在不合法元素时整体报错，包含 field/index 等可定位信息；空列表仅表示明确关闭或未配置。不要一边保存原数据、一边在读取时悄悄修复。URL 解析尽量复用已有标准解析器和源适配器，npm/static 的差异由源类型承载，不在 UI 复制。

### 3.2 稳定原因与显示文案分离

定义 RegistryProblem 的有限原因，例如 notConfigured、invalidConfiguration、searchUnsupported。它属于注册表工作流，不扩大为所有错误共用的字符串池。

当前失败可继续使用粗粒度 ErrorCode，但在有类型的 details 中携带注册表问题、字段和建议动作；核心 IpcError 与 JS 形状同步。ErrorCode 仍是失败主分类，业务原因仅细分可执行操作，不能覆盖原始网络或验证失败。

前端 failureOf 保留该结构，useCatalog 输出可判别工作流状态，组件通过有限状态分支展示。message 只用于翻译，不做 includes、startsWith 或正则业务判断。

## 4. 实施步骤

1. 盘点 `REGISTRY_KEY` 的所有写入和读取：设置页、registry transport、插件安装/搜索/更新、daemon 对应源配置。记录两宿主配置差异，不假定必须共享同一设置文件。
2. 在合适的 core 配置/注册表模块定义可复用解析规则；App 保留设置来源和 IpcError 映射。源选择/镜像允许回退规则继续复用现有实现。
3. 定义 RegistryProblem DTO 和建议动作映射，同步 synra-ipc 与 JS protocol，遵循现有类型同步流程。
4. 更新 transport::bases，严格拒绝坏元素，明确未配置与非法配置的区别。
5. 更新 `features/plugins/actions.ts`、catalog.ts 和 registry-discover.vue，删除英文句子匹配。
6. 更新预览内核和测试 fixture，使预览与真实命令返回相同结构；切换语言不改变状态判断。

## 5. 验收场景

- 配置缺失、空字符串、空数组分别按目标约定处理，去设置入口可用。
- 数字、对象、混合数组、非法 URL 返回具体配置错误，不被过滤后继续执行。
- npm/static 源各自支持的行为与错误保持区别。
- 网络、无搜索端点、无结果、包校验失败不会误判为未配置。
- 中英文、替换 message key、修改说明句子均不改变工作流状态。
- 设置修改后的新查询立即使用新配置，旧响应不能覆盖新的结果。
- 镜像回退仍遵守完整性失败不可绕过的约束。

## 6. 完成条件

按真实 Rust 命令序列化错误驱动组件测试，不能在前端 fixture 中直接塞入那句英文来证明提示有效。关联 [07](./07-error-boundary.md) 的错误归一化，并在 [11](./11-validation-and-rollout.md) 记录后端与 UI 的联合验收。
