# 05 · 统一版本解析、排序与升级策略

状态：计划。优先级：P2。范围：core 插件清单、注册表、同步和更新；App/daemon 调用方同步调整。

## 1. 当前存在的规则

| 实现 | 证据 | 当前问题 |
| --- | --- | --- |
| plugin_sync::compare_versions | [decision_policy.rs:159](https://github.com/synra-rs/synra-core/blob/a75d399/crates/synra-core/src/plugin_sync/decision_policy.rs#L159) | 预发布仅保留有/无，数字解析失败变 0，build metadata 参与向量比较 |
| plugin-host registry::compare_versions | [integrity.rs:14](https://github.com/synra-rs/synra-core/blob/a75d399/crates/synra-plugin-host/src/registry/integrity.rs#L14) | 同形简化实现，用于缺 latest 时挑默认版本 |
| plugin-api manifest::compare_versions | [version.rs:178](https://github.com/synra-rs/synra-core/blob/a75d399/crates/synra-plugin-api/src/manifest/version.rs#L178) | 自写解析和校验；u64 解析失败被 filter_map 丢弃，校验又未统一拒绝溢出 |
| synra-registry::is_newer | [packument/mod.rs:172](https://github.com/synra-rs/synra-core/blob/a75d399/crates/synra-registry/src/packument/mod.rs#L172) | 使用 semver，但非法版本回退为字符串不同即更新，无法证明候选更高 |

`1.0.0-rc.1` 与 `1.0.0-rc.2` 被前两套比较为相等；`1.0.0+build.9` 与 `1.0.0` 会因额外向量元素而排序不同。这些不只是重复代码，而是相同包在同步、安装、更新路径得到不同结论。

另外，semver 库的完整对象排序是否包含 build metadata，不能凭使用库名就认定等于升级优先级；实现应显式采用版本 precedence 并用 build-only 样例验证。

## 2. 目标规则

### 2.1 一个解析器与版本值

在依赖层次允许的最低公共模块建立 PluginVersion，建议归 `synra-plugin-api::manifest`：该 crate 已拥有版本合法性契约，plugin-host 与内核也在使用。采用 workspace 已有 semver 依赖封装，保持外部只暴露领域 API，避免公开泄漏第三方类型。

所有入口先解析，再使用同一个 precedence 比较。major/minor/patch、预发布标识与 build metadata 的处理只定义一次。溢出、空标识、数字前导零等按同一合法性规则拒绝，不变成 0，也不丢弃字段。

### 2.2 比较和业务策略分离

- 排序只回答版本优先级，不决定是否自动安装。
- latest tag 的解析、版本范围选择、稳定版策略、主版本限制分别是命名策略。
- build metadata 不改变升级优先级；同版本不同 bytes 仍依据现有 digest 冲突规则决定，不改称版本升级。
- NeverDowngrade、KeepNewest、SourceWins、Ask 等同步策略继续明确表达。对于不可比较的输入返回结构化错误或人工处理结果，不偷偷通过字符串差异绕过降级保护。
- 非 SemVer 的日期式版本若确有生产需求，必须独立声明版本方案；首轮插件版本保持 SemVer，不继续保留无判定依据的 fallback。

## 3. 实施步骤

1. 搜索 `compare_versions`、`check_version`、`is_newer`、VersionReq 及直接版本字符串比较，建立调用方列表，包含 CLI、App 和 daemon。
2. 在 plugin-api 添加统一封装与结果错误，保留适合调用者的命名方法，例如 parse/compare_precedence；先写边界用例。
3. 将 manifest 校验、SDK min 判断、install replacement/automatic、plugin_sync、plugin-host 的默认挑选迁入统一规则。
4. 调整 synra-registry 的升级比较使用相同规则；保留其协议层的 tag/range 选择职责，不让注册表反向依赖运行时内核。
5. 如果现有 crate 依赖方向不允许引用 plugin-api，先确认最小公共归属；不得引入循环依赖，也不为这一个函数直接新建大型“公共工具”层。
6. 将 bool 升级判定改成可传播解析失败的结果；App 与 daemon 显示正确诊断，不将失败当作“没有更新”或“有更新”。
7. 删除旧比较器、对外重复 re-export 和为旧 fallback 专设的分支，同步相关说明。

## 4. 验收表

| 输入 | 比较/策略结果 |
| --- | --- |
| 1.10.0 与 1.9.0 | 前者更高 |
| 1.0.0-rc.2 与 1.0.0-rc.10 | 后者更高 |
| 1.0.0-alpha 与 1.0.0 | 正式版更高 |
| 1.0.0+build.1 与 1.0.0+build.9 | 升级优先级相同 |
| 1.0.0-01、空标识、超范围数字 | 明确解析失败，不降成零或截断 |
| registry latest 比已安装旧 | 不提供自动降级 |
| 缺 latest 且包含多个预发布 | 采用明确默认选择策略，排序不依赖迭代顺序 |
| 相同版本、不同 digest | 仍按同步冲突策略处理 |
| 任意有效 a、b、c | 比较满足自反、反对称、传递性 |

## 5. 风险与完成条件

已有本地不规范版本可能从“勉强可用”变为明确拒绝。先扫描测试 fixture、模板及实际安装记录样例，列出需要修正的数据；未发布 Rust/App 直接更新，不额外维护宽松比较器。对外插件作者文档和 JS 发行说明同步描述合法版本格式。

验收必须在安装、同步、注册表更新三个业务入口验证同一组边界，不能仅让新的比较器单测通过。根工作区与受影响 package 检查见 [11](./11-validation-and-rollout.md)。
