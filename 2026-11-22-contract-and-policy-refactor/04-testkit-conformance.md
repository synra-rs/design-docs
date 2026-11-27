# 04 · Testkit 行为与能力语义收敛

状态：已实施并通过共同场景验收。覆盖授权、支持、错误、设备和自定义事件、外发传输及网络范围。优先级：P1。实际结果见[实施记录](./IMPLEMENTATION.md)。

## 1. 改造前的证据

核心文件为 [packages/testkit/src/index.ts](https://github.com/synra-rs/synra-js/blob/622a2ef/packages/testkit/src/index.ts)。类型上 implements SynraClient 只能证明签名完整，不能保证行为相同。

| 位置 | 当前行为 | 与真实契约的差异 |
| --- | --- | --- |
| clipboard.read/write，571/582 行 | platform:clipboard:read/write | App 检查 platform:read-clipboard/write-clipboard |
| event.subscribe，305 行 | 不读取 features | hostSupports 返回 false 时调用仍成功 |
| stub，202 行及各方法开头 | 设置 stub 记为调用，命中则直接返回 | 实际调用参数/次数不记录，并跳过若干授权检查 |
| device.list，329 行 | 仅返回 online | client 明确保留已配对且未屏蔽的离线设备 |
| env.can 与 gate，270/674 行 | 查询精确或纯星号；执行另做前缀通配 | 同一个 mock 的查询和执行就会不一致，纯 * 也不符合核心能力语法 |
| device.supported，385/709 行 | 工厂构造后 defineProperty 补值 | 直接 new MockSynraClient 和工厂返回的支持列表不同 |
| network.fetch/hostOf，459/720 行 | 正则提取 authority | 端口、用户信息等混入能力主机名，不能代表 core 的 URL 范围判断 |
| transfer.status，516 行 | 任意 ID 返回 completed | 不存在和真实完成无法区分 |

本轮只读复现了前三项：未支持的订阅成功；真实剪贴板授权被拒；设置 stub 后真实调用两次，callCount 为 1 且参数是 `<stubbed>`。

## 2. 目标边界

testkit 是可控的协议替身，模拟授权、支持、调用记录和可观察状态。它不实现真实网络、Wasm、系统输入或传输字节。对这些外部结果使用显式 fixture/stub，但默认行为不能伪造成功结果。

`hostSupports`、`platformSupports`、`can` 分别回答方法实现、平台能力、插件授权。缺授权不能改写为平台不支持；查询不能宣称真实宿主没有提供的能力。App 当前 BACKED 统一表本身不是问题，关键是由行为测试证明表中 true 对应完整可用语义。

## 3. 目标结构与调用流程

将单文件拆为 options/profile、调用记录、授权策略、stub 响应、事件/设备、存储/网络、传输等职责模块。公开入口仅保留类型、工厂和类导出。

每个调用采用固定顺序：

1. 根据宿主 profile 判断 API 是否实现。
2. 根据生产宿主实际顺序检查平台支持和授权；排序约定写成用例，不由各方法临时选择。
3. 记录真实调用及参数；拒绝也记录一次。
4. 通过策略后执行 stub 或默认模型；记录结果或失败，不把设置 stub 当作业务调用。

stub 仅替换响应，不隐式绕过策略。若测试确实需要不受策略的故障注入，使用名字明确的低层测试接口，不复用普通 stub。

## 4. 能力和支持模型

使用一份构造时 profile 生成 features、device.supported 与执行门禁，直接构造和工厂构造完全一致。当前 App 支持哪些方法与“mock 可以模拟哪些方法”分开，提供显式 app profile 和自定义 profile；不靠“最容易用到的三个”作隐式宿主事实。

授权名称遵循 core DSL。能力匹配语义以 core 的解析与匹配用例为权威：`*` 单段、`**` 多段、限定符规则和非法语法都要核对，不能以通用字符串前缀替代。App 的 env.can 和本地预检查同样进入此一致性测试；最终授权始终由宿主强制。

首轮不新增通用运行时包，不把函数实现塞入类型包。两个仓库需要的少量纯逻辑分别放在各自责任模块，以共享语义向量和差分测试约束。若以后维护成本证明值得独立包，再单独提案。

## 5. 状态与数据修正

- device.list 与 watcher 使用相同集合过滤规则，保留离线项；设备选择的默认 fixture 必须显式，不能掩盖用户选择逻辑。
- 传输 fixture 按 ID 建模状态；reportProgress 同时推进状态。未知 ID 返回稳定失败，不能自动完成。
- HTTP 默认没有响应 fixture 时明确失败；如果需要空响应，则通过 fixture 指定一个字节来源，text/json/bytes 从同一内容解析，避免空文本和 `{}` 同时被称为同一响应。
- URL scope 的测试以标准 URL 解析和核心允许语义为准，覆盖 hostname、端口、IPv6、用户信息、非法协议，不将正则捕获直接当能力字符串。
- 对时间、ID 和外部响应保留可注入控制点，避免真实 sleep 和网络。

## 6. 实施顺序与验收

1. 先增加能在当前实现上失败的权限、支持、stub 记账和离线设备用例。
2. 建立调用管线，修正构造 profile、错误出口及每种方法的授权顺序。
3. 按责任拆分原文件，在不改变导出入口的前提下迁移现有测试。
4. 跟随 [01](./01-plugin-event-contract.md)、[02](./02-transfer-state.md)、[07](./07-error-boundary.md) 更新事件、传输和错误模型。
5. 按 [11](./11-validation-and-rollout.md) 用同一语义场景驱动 App facade 与 testkit，并核对 core 产生的授权/事件样例。

验收至少覆盖：支持与否 × 授权与否 × 平台支持与否；stub 成功/失败及准确记账；构造入口一致；离线设备；未知传输；URL 范围；事件模式与退订。验证行为和结果，不要求两端内部代码结构相同。

## 7. 行为变化与交付条件

修正后依赖“stub 自动跳过权限”或“任意传输都已完成”的旧测试可能失败，这是需要迁移的测试假设，不能通过保留隐藏开关维持。JS 对外变更按发布流程说明；实际插件 fixture 同步更新。所有通用断言通过后，才把 mock 视为当前宿主契约的可靠替身。
