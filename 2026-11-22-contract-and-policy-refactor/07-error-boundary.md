# 07 · 注入边界错误归一化

状态：App 与 testkit 已通过错误形状和分类的共同自动验收。Windows 原生 WebView 已验收 Starter 的参数诊断与撤权提示。优先级：P1。实际提交和验证见[实施记录](./IMPLEMENTATION.md)。

## 1. 当前证据

[errors.ts:60](https://github.com/synra-rs/synra-app/blob/f7c6d1d/src/ipc/injection/errors.ts#L60) 仅根据 candidate.code 是 string 就强转为 SynraError；其余异常一律转成 unsupported('unknown')。candidate 为 null 时会读取 null.code。类型守卫也缺少 null 检查，未检查 retryable。

同文件构造错误使用 `as unknown as SynraError`，部分 message key 为 camelCase，部分为 snake_case。虽然当前有些字段已经满足类型，强转仍会掩盖将来契约变化。[testkit 的错误守卫](https://github.com/synra-rs/synra-js/blob/622a2ef/packages/testkit/src/index.ts#L626) 又采取不同校验条件。

真实契约见 [SynraError](https://github.com/synra-rs/synra-js/blob/622a2ef/packages/protocol/src/generated/wire.ts#L146) 与 [IpcError](https://github.com/synra-rs/synra-js/blob/622a2ef/packages/protocol/src/generated/ui-activity.ts#L20)：code、message、retryable 必填，diagnostic details 与建议动作有各自作用。

只读探针确认：asSynraError(null) 和 isSynraErrorValue(null) 都抛异常；new Error('transport failed') 被转换为 unsupported，原始故障无法从分类判断。

## 2. 目标不变量

- 类型守卫对任何 unknown 都安全返回 boolean，不抛异常。
- 接收并校验完整必填结构；code 以权威 ErrorCode 集合为准，不把任意字符串当可信分类。
- unsupported 仅用于明确缺失的方法或平台能力；未知失败映射为 internal，保留经限长和脱敏的诊断来源。
- IPC params、details、actions 不丢失；diagnostic details 不能覆盖权威 code 或其他必填字段。
- message key 由一个映射约定产生，不按调用者偏好拼写；UI 翻译和机器分支分别使用 message 与 code/结构化原因。
- 错误归一化不是序列化任意对象；不递归读取可能有副作用的属性或泄漏整个请求、token、文件内容。

## 3. 目标结构

在 App injection 内建立窄职责错误边界模块，包含总定义的 guard、具类型构造器、IPC → SynraError 适配、unknown → internal 归一化和受控诊断提取。

优先使用明确返回类型或 satisfies，而不是 `as unknown as`。输入类型不可信时运行时验证；内部已知构造无需重复验证。类型包不新增运行时函数，testkit 通过相同语义用例保证行为一致。

建议对业务可判定原因使用拥有者定义的类型，例如注册表缺配置；不要让通用 normalizeError 猜测 message 的内容。注册表工作流见 [09](./09-registry-state.md)，Wasm trap 来源分类见 [08](./08-wasm-failure-classification.md)。

## 4. 实施步骤与文件

1. 盘点 `injection/errors.ts`、所有 `.catch(asSynraError)` 等价路径、client.ts 的 IpcError 使用和 testkit guard；建立现有 ErrorCode/message key 对照。
2. 先加边界输入表：null、undefined、primitive、Error、合法 IPC/Synra 错误、缺字段对象、未知 code，当前缺陷用例必须失败。
3. 添加具类型构造器，移除无需存在的双重强转；保留 actions、retryable 与 traceId 等公开信息。
4. IPC 错误的 params 转为明确诊断字段或约定结构，避免强转过程中丢失；不把 params 展开到顶层。
5. 更新 facade 各 namespace 使用统一出口，保留原始失败类别，不增加 catch-return-empty 的成功兜底。
6. testkit 守卫按同一公共类型要求更新；若增加错误原因 DTO，由 Rust 源和 JS 类型同步定义。
7. 同步翻译 key、文档、注入构建物和现有错误断言，核查实际 UI 展示。

## 5. 验收矩阵

| 输入/路径 | 结果 |
| --- | --- |
| null、undefined、数字、字符串 | guard 为 false；归一化为可诊断 internal，不抛二次异常 |
| 只有 code 的对象 | 不冒充完整 SynraError |
| 合法 IpcError | code、retryable、actions 和诊断保留 |
| 未知 code | 明确 internal/协议异常，不变为 unsupported |
| 明确缺 API | unsupported 且带 feature |
| 无授权 | capability_denied 且带准确 capability |
| 连接或序列化异常 | 分类能说明异常，不宣称方法不存在 |
| 循环对象、过长文本、敏感字段 | 诊断有界，不因提取诊断失败再抛异常 |
| façade 与 testkit | 对公共样例有相同分类和 guard 结果 |

## 6. 完成条件

回归覆盖真实 bridge rejection，经 namespace 到插件 catch 的完整路径；仅测试构造器不足。未知错误不得静默吞掉，诊断也不能让敏感 token 流向日志。本项可以最先独立交付，为事件、传输和注册表提供稳定失败出口。
