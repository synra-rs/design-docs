# 08 · Wasm 限额与 Trap 分类

状态：已实施，原生自动验收通过。范围：synra-plugin-host 的 Wasmtime 后端与对外 CallOutcome。

## 1. 重构前证据

[instance.rs:215](https://github.com/synra-rs/synra-core/blob/a75d399/crates/synra-plugin-host/src/engine/wasmtime_backend/instance.rs#L215) 先检查 memory_refused，再格式化整个 Wasmtime 错误链，依据是否包含 epoch/interrupt/fuel 选择 Timeout 或 FuelExhausted。否则返回普通 Trap。

这让错误上下文中的单词参与分类。普通 host failure 只要带这些词就可能被当作预算耗尽；升级后端或修改上下文文案也可能改变业务结果。没有 timeout/fuel 时的 unwrap_or(0) 又可能生成无真实配置依据的零预算诊断。

现有 [fuel/epoch/memory 测试](https://github.com/synra-rs/synra-core/tree/a75d399/crates/synra-plugin-host/src/engine/wasmtime_backend/tests) 是基础，可继续保留。当前 Cargo.toml 使用 Wasmtime 38 系列，具体可用错误类型以锁文件和本地依赖源码为准，本计划不假定未经验证的 API 名称。

## 2. 目标设计

### 2.1 分类只读取类型化原因

在后端边界对错误链中的类型化 trap、host call failure、资源限制拒绝进行分类。格式化文本仅写入诊断，不决定 code。若无法确认来源，保留为普通 Trap，不再通过关键词升级成预算错误。

CallOutcome 继续表达 timeout、fuel、memory 与普通 trap 的不同含义。必要时新增内部 FailureCause 作为统一分类输入，不让上层重复理解 Wasmtime 的错误包装。

### 2.2 每次调用记录实际生效预算

建立本次调用的 EffectiveBudget/LimitState：启用的时间限制、实际 fuel、实际内存 ceiling 及观察到的拒绝原因。调用开始时重置拒绝标志，实例池复用不得继承上一次结果。

内存、fuel 和 epoch 同时存在时，按可证明的实际失败来源分类。仅“曾尝试过内存增长失败”不应无条件覆盖随后无关的 trap；如果现有限制器无法判断因果，保留该事实为诊断并采用类型化 trap 的分类。

没有配置的限额不报成零。框架默认 ceiling 与调用指定 ceiling 分清，同时在错误中报告真正生效的上限。实例化失败和执行中失败仍有各自阶段信息。

## 3. 实施步骤

1. 检查当前锁定 Wasmtime 版本的 Trap/host error 包装方式和现有 invoke 路径，先以最小真实组件验证错误链下转型。
2. 在 `engine/wasmtime_backend/` 独立分类模块；将 error、effective budget 与观察状态作为输入，产出统一 CallOutcome。
3. 调整 store/limiter 在每次调用开始时清零并记录本次状态；保留现有 epoch ticker 和 fuel 重置逻辑，不顺手改变调度策略。
4. MissingExport、host call failure、guest trap、budget exceeded 分别走明确路径；移除 contains('fuel') 等字符串分支。
5. 更新 executor 到协议错误的映射，确认 App 插件 RPC 和 daemon 都保留相同分类和诊断。
6. 增加真实 Wasm 回归及实例复用序列，再整理原有预算相关注释。

## 4. 验收矩阵

| 场景 | 预期 |
| --- | --- |
| 普通错误文本包含 fuel/epoch/interrupt | 仍是原始普通 trap/host failure |
| 真实 fuel 耗尽 | FuelExhausted，预算值来自实际配置 |
| 真实 epoch 中断 | Timeout，时长来自有效时间限制 |
| 默认内存 ceiling 生效 | MemoryExceeded 报默认真实上限，不报 0 |
| 主动设置更小内存限制 | 报本次更小 ceiling |
| 先处理可恢复的内存增长拒绝，再正常调用同实例 | 拒绝记录、fuel、deadline 与内存 ceiling 都按新调用初始化 |
| 宿主强制终止之后再次调用插件 | 实例池丢弃已 trap 的实例，下一次从可用实例执行 |
| 无限额调用接在有限额调用后 | 不继承旧限制；测试有外部截止机制 |
| 内存拒绝后产生其他独立 trap | 按明确的因果规则归类，附带诊断 |
| App 与 daemon | 对同一 CallOutcome 返回一致错误类别 |

## 5. 风险和完成条件

后端版本升级可能改变可下转型的类型，分类模块应集中这种依赖并用真实组件测试锁住行为。不要在某类错误下转型失败后重新引入关键词 fallback。

测试执行带明确超时，避免错误的 epoch/fuel 设置让套件永久挂起。通过受影响 package 的真实后端测试、App RPC 映射回归后才算完成；ScriptedBackend 的模拟结果不能证明 Wasmtime 分类有效。

## 6. 落地方式

- `wasmtime_backend/failure.rs` 从错误链读取 `Trap::OutOfFuel`、`Trap::Interrupt`、宿主内存拒绝和 WIT 宿主调用错误。时间或 fuel 缺少对应预算时保留普通 trap。错误文本用于诊断。
- 每次进入 WIT 调用前建立有效预算。未指定 fuel 和时间时使用 Wasmtime 的解除限额值，未指定内存时恢复 16 MiB 默认上限。内存上限检查每个线性内存增长后的大小，已分配的内存不会收缩。
- 超出宿主配额直接从 Wasmtime 限制器返回 `MemoryCeilingExceeded`。如果线性内存声明或分配器最大值更严格或相等，guest 仍可以通过 `-1` 处理增长拒绝。可恢复拒绝只作为后续独立 trap 的附加诊断，并在下次调用清除。
- 实例化阶段沿用 `BackendError::Instantiate`。App 页面 `self.invoke`、本地执行和 daemon 远程执行共用 `CallOutcome::into_result`，保留协议错误码与预算诊断。

详见[实施记录](./IMPLEMENTATION.md)。
