# 10 · 能力模型与沙箱强制

> 本文定义 Rust 版 capability 系统的**语法、解析、三层闸门、资源限制与逃逸面收敛**。

---

## 1. 为什么不能只靠沙箱

常见误解："有 Wasm 沙箱就够了，不需要 capability。"

错。Wasm 沙箱解决的是**内存安全与代码执行隔离**（插件不能读宿主内存、不能直接 syscall）。它**不解决**：

- 插件能调用哪些 host function？（沙箱里所有 host function 都是"合法"的）
- 插件被允许访问哪个域名？
- 插件能读写哪个目录？
- 插件能发消息给哪台设备？

这些都是**授权问题**，必须由 capability 系统回答。沙箱与 capability 是**互补**的：

```
┌───────────────────────────────────────────────────────────┐
│  Wasm 沙箱：插件不能做什么（内存、syscall、无限循环）        │
├───────────────────────────────────────────────────────────┤
│  Capability：插件被允许做什么（host function 粒度）          │
├───────────────────────────────────────────────────────────┤
│  用户授权：用户同意插件做什么（安装/升级时确认）              │
└───────────────────────────────────────────────────────────┘
```

---

## 2. Capability 语法

### 2.1 形式文法

```
capability  := segment (":" segment)* qualifier?
segment     := [a-z][a-z0-9-]*
qualifier   := <受限目标>            ; 依 domain 而定
wildcard    := "*" | "**"            ; 仅作为完整 segment
```

BNF 展开（供解析器实现与 lint 使用）：

```
CAP        ::= PATTERN
PATTERN    ::= SEG (":" SEG)*
SEG        ::= NAME | "*" | "**"
NAME       ::= [a-z][a-z0-9-]{0,63}
```

> **规范约束**：
> - 总长度 ≤ 256 字符；
> - segment 数 ≤ 6（`domain:resource:action:q1:q2:q3`）；
> - 不允许空 segment（`ui::navigate` 非法）；
> - 不允许纯通配（`*` 单独非法——那等于"全部能力"）；
> - 不允许空白与特殊字符（除了 qualifier 的路径/域名/MIME 形态，见 §2.3）。

### 2.2 Domain 分类（权威表）

| domain     | 语义                 | 典型 capability                                   |
| ---------- | -------------------- | ------------------------------------------------- |
| `log`      | 日志                 | `log:*`、`log:debug`                              |
| `event`    | 事件总线             | `event:emit:_plugin.chat.*`、`event:subscribe:_synra.device.*` |
| `device`   | 跨设备               | `device:list`、`device:send:@synra-pc`、`device:broadcast` |
| `storage`  | 插件私有存储          | `storage:local:read`、`storage:local:write`、`storage:secure:read` |
| `network`  | 网络（宿主代理）      | `network:fetch:api.github.com`、`network:fetch:api.x.com:POST`、`network:websocket` |
| `fs`       | 文件系统（沙箱内）    | `fs:read:/data/**`、`fs:write:/cache/*`、`fs:watch` |
| `file`     | 大对象传输            | `file:transfer:send`、`file:transfer:receive`      |
| `platform` | 宿主系统能力          | `platform:open-url`、`platform:simulate-keys`、`platform:clipboard:read` |
| `action`   | 跨插件动作            | `action:declare:format-date`、`action:invoke:*:format-date` |
| `ui`       | **仅 UI 插件**：界面能力 | `ui:page`、`ui:navigate`、`ui:toast`、`ui:confirm`、`ui:notify`、`ui:modal` |
| `env`      | 只读环境信息（无闸门） | 不需要声明                                          |

**关键区分**：
- `device:send` 与 `file:transfer:send` 是**不同域**。前者是"发一帧消息"，后者是"发起一个传输会话"。
- `platform:*` 是**宿主系统能力**（打开浏览器、模拟按键），比 `device:*` 危险得多——因为它在本地执行系统操作。
- `ui:*` 只对 UI 插件有意义；headless 插件声明了也无效（装载期拒绝）。

### 2.3 Qualifier 语义（每个 domain 有自己的限定规则）

| domain    | qualifier 形态              | 匹配语义                                                     |
| --------- | --------------------------- | ------------------------------------------------------------ |
| `device`  | `@device-id` / `@self` / `*`  | 目标设备 id（`@` 前缀为设备 id 的显示别名约定）                 |
| `network` | `<host>` 或 `<host>:<METHOD>` | 域名精确匹配（**小写、去尾点、punycode 归一**）；METHOD 大写精确 |
| `fs`      | `<absolute-path-glob>`       | 沙箱内绝对路径（**先 canonicalize 再比较**，见 §4.3）           |
| `event`   | `<event-pattern>`            | 事件名 glob（`*` 单段，`**` 多段）                              |
| `action`  | `<name>`                     | 动作名精确或 glob                                              |
| `ui`      | `<pageKey>`                  | 页面 key                                                      |

**`network` 的规范化规则**（必须严格）：
1. 转小写；
2. 去掉末尾的点（`api.example.com.` → `api.example.com`）；
3. IDN 转 punycode（`例子.com` → `xn--fsqu00a.com`）；
4. 拒绝 IP 字面量（`192.168.1.1` 需要显式声明为 `ip:192.168.1.1`，见 §4.4）；
5. 通配只支持**前缀**形式 `*.example.com`（**不**支持 `api.*.com`）——后者是无意义且易错的。

**`fs` 的规范化规则**：
1. 相对于插件沙箱根解析；
2. `canonicalize()`（解析 `..`、symlink）；
3. 与授权路径的 canonical 形式做**前缀比较**（按路径组件，不是字符串前缀，避免 `/data` 匹配 `/database`）；
4. 拒绝在解析过程中逃出沙箱根（`PathEscape`）。

### 2.4 Capability 的三种声明位置

| 位置                       | 作用                                       | 何时校验       |
| -------------------------- | ------------------------------------------ | -------------- |
| manifest `capabilities[]`   | 插件**想要**什么                            | 安装/升级时询问用户 |
| `Grant.granted`             | 用户**授予**了什么                          | 装载期与调用期  |
| WIT 里 import 了哪个 interface | 插件**能调用**哪些函数（编译期）              | 编译期与装载期  |

**三者取交集**才是插件实际可用的能力：

```
实际可用 = manifest 想要的 ∩ 用户授予的 ∩ WIT import 的
```

装载期校验：

```rust
pub fn verify_effective_caps(
    manifest: &PluginManifest,
    grant: &Grant,
    wasm_imports: &ImportSet,      // 从 wasm 模块读取的 import 列表
) -> Result<EffectiveCaps, VerifyError> {
    // 1. manifest 声明必须语法合法
    // 2. 用户授予必须是 manifest 的子集
    // 3. wasm import 的 interface 必须被授予覆盖
    //    （例如 import 了 host.network-fetch，但没授予任何 network:fetch:* → 拒绝装载）
    // 4. 反向：授予了但 wasm 没 import → 警告（不拒绝）
}
```


---

## 3. 三层闸门

```
┌─────────────────────────────────────────────────────────────────────────┐
│  第一层 · 编译期（插件作者本地 / CI）                                     │
│                                                                         │
│  · synra-plugin-cli verify：                                          │
│      - capability 字符串语法                                             │
│      - manifest 与 WIT world 对齐（import 的 interface 是否有对应能力）    │
│      - wasm 不得 import 未授权的能力                                      │
│      - 体积 / 依赖 / 禁用导入检查                                        │
│  · Rust SDK 的宏：`#[capability("device:send")]` 标注在调用点，          │
│    SDK 检查它是否在 manifest 声明集合内（编译期报错）                      │
│                                                                         │
│  失败 → 插件无法构建/无法发布                                             │
├─────────────────────────────────────────────────────────────────────────┤
│  第二层 · 装载期（内核，安装/激活时）                                      │
│                                                                         │
│  · manifest schema 校验 + 依赖图无环                                      │
│  · capability 语法 + domain 适用性（如 headless 不许有 ui:*）              │
│  · wasm 模块的 import 集合 ⊆ 用户授予的能力                                │
│  · WIT 版本与宿主兼容                                                     │
│  · 签名校验（[16](./16-plugin-distribution.md) §3）                        │
│                                                                         │
│  失败 → 安装被拒绝，或激活被拒绝（状态可见，附原因）                        │
├─────────────────────────────────────────────────────────────────────────┤
│  第三层 · 调用期（内核，每一次 host function 调用）                        │
│                                                                         │
│  · 解析被调用的能力标识（domain:resource[:qualifier]）                     │
│  · 与 Grant.granted 匹配（精确优先，然后 glob）                            │
│  · 配额检查（§5）                                                         │
│  · 审计记录（[18](./18-observability.md) §5）                             │
│  · 拒绝 → 返回结构化错误给插件 + 通知用户（可配置）                          │
└─────────────────────────────────────────────────────────────────────────┘
```

### 3.1 第三层是唯一必经路径（用类型强制）

> ★ **先记两处本次核对出来的、比下面这一节更要紧的实情。** 它们都属于"机制齐全、没有人调用"这一类，
> 而且都落在上面那张三层图里：
>
> **（一）第二层的 import 检查原先没有任何生产调用者 —— 本次已接上。**
> 图上写的是「wasm 模块的 import 集合 ⊆ 用户授予的能力」。
> 实现它的函数是 `host_surface::undeclared_imports`（有测试），
> 而用它的管线 `synra_plugin_host::verify::verify` **全仓没有生产调用者** ——
> 于是"导入了一个 manifest 从未声明的 host function"的 wasm 会被正常装载并运行，
> 唯一的症状是之后某次调用在闸门被拒，而那时已经离装载很远了，作者也无从据以修改。
> **本次接上**：`PluginSet::load` 在编译之后用
> `undeclared_imports(compiled.imports(), &spec.grants)` 检查并拒绝
> （`PluginSetError::UndeclaredImport`）。
> * 检查的是**组件自己报告的 import**（编译期从类型里读出来），不是 manifest 声明的东西 ——
>   这是"声明"与"事实"的区别，也是它只能放在编译之后的原因：字节编译完之前不存在 imports。
> * 为此 `CompiledArtifact` 增加了 `imports()`（默认空）——内核持有的是
>   `Box<dyn CompiledArtifact>`，不能为了拿到 imports 就去依赖某个具体引擎。
> * 拒绝时的消息**点名那个 import 而不是那个 capability**：作者需要知道删掉或声明哪一个 import；
>   对用户点名 capability 会暗示"改个设置就能跑"，而事实相反。
> * 测试双向：未覆盖 → 拒绝且名字对得上；同一 import 配上 capability → 正常装载。
>   负控制把检查关掉后测试红（"should refuse the load, got Ok(())"）。
> * ★ 这是**第 1 层**的检查（imports ⊆ manifest），不是第 2 层写的那个（imports ⊆ **用户授予**）——
>   后者要等下面（二）解决之后才有意义，而那时这段代码不用改：它比的一直是 `spec.grants`。
>
> **（二）per-capability 的用户同意，端到端没有实现；装载器把 manifest 的声明直接当作授权。**
> 图上第 2 层写的是「wasm 模块的 import 集合 ⊆ **用户授予的能力**」，第 3 层与 §7 的审计都假定存在一份
> "用户授予了什么"的记录，表 `plugin_grants` 就是为它建的。实际情况：
> * `crates/synra-store/src/db/sqlite/plugins.rs:150` 的 `plugin_grants()` 是**读者**，
>   而它**只在测试里被调用**（`tests.rs:375/387/402`）。
> * **没有任何生产代码写这张表** —— 全仓 `INSERT INTO` 里没有 `plugin_grants`。
> * 于是 `apps/synra-daemon/src/runtime/plugins.rs:178` 的
>   `let grants = grants_of(&manifest)?;` 就是全部：**插件声明什么能力，就拿到什么能力**。
>   唯一的门槛是 `should_run()` = `enabled && trusted`，一个**每插件一个布尔**。
> * 装载路径也**完全没有调用 verify 管线**（`load_one` 里没有任何 `verify` 调用）——
>   本次接上的是其中 import 覆盖这一条，签名/摘要/发布者信任那几步仍然没有调用者。
> * app 侧也没有同意界面：`plugins_consent_respond` 在未注册清单里，
>   前端 13 个 TS 文件里 `consent` 只出现在 `fake-kernel.ts` 这个夹具里。
>
> ★ **为什么这一条不能就地修**：把 `load_one` 改成读 `plugin_grants` 并取交集，
> 在**没有任何写入方**的今天会让每个插件的授权集变空 —— 插件全部停止工作。
> 也就是说当前"用 manifest 当授权"是**承重的**，不是一处可以单独打补丁的疏忽。
> 要修它得先有同意流程（安装 → 展示能力 → 记录决定 → 写 `plugin_grants`），
> 而那是一条完整的纵向功能，涉及内核、IPC（`plugins_consent_respond`）、前端三处。
> 记在这里而不是改一行，是因为改一行会让整件事从"不安全"变成"不能用"。


**问题**：如何保证"所有 host function 都过闸门"？人工审查不可靠。

**做法**：host function 的注册**只能**通过一个宏，该宏强制要求 capability 描述符：

```rust
// crates/synra-plugin-host/src/host_funcs/mod.rs

/// 注册一个受闸门保护的 host function。
/// 宏展开后生成的包装函数是唯一能与 wasmtime 链接的形式。
macro_rules! gate_host_fn {
    (
        name: $wit_name:literal,
        capability: $cap:expr,          // ★ 必填：CapabilityPattern
        handler: $handler:path
    ) => {{
        // 生成的函数体：
        //   1. 从 Store 取出 PluginContext（含 grants / quota / audit）
        //   2. 计算实际能力标识（可从参数派生出 qualifier）
        //   3. ctx.gate.check(&resolved_cap)?;
        //   4. ctx.quota.consume(...)?;
        //   5. ctx.audit.record(...);
        //   6. 调用 handler
        //   7. 把 Result 映射为 WIT outcome
        // 任何一步失败都不会调用 handler
    }};
}

/// 无闸门的 host function（只允许用于 env 只读信息）
macro_rules! ungated_host_fn {
    (name: $wit_name:literal, reason: $why:literal, handler: $handler:path) => {{ /* ... */ }};
}
```

**约束**：
- `ungated_host_fn` 的 `reason` 是**必填文档字符串**，且**只允许在 `host_funcs/env.rs` 里使用**（用 lint 或 `#[cfg]` 强制）；
- CI 检查：`grep -c 'ungated_host_fn' src/host_funcs/` 的结果必须等于 `env.rs` 里的数量。

### 3.2 能力标识的动态解析

有些能力标识需要**从参数派生** qualifier：

```rust
#[gate_host_fn(
    name = "device-send",
    capability = CapabilityPattern::parse("device:send:*"),   // 模板
    handler = handlers::device_send
)]
async fn device_send_gated(
    ctx: &mut PluginCtx,
    target: String,
    event: String,
    payload: String,
) -> Result<Outcome> {
    // 从参数派生实际能力标识
    let actual = Capability::new("device", "send", Some(Qualifier::Device(target.clone())));
    ctx.gate.check(&actual)?;      // ★ 用实际标识校验，而不是模板
    // ...
}
```

**要求**：模板 `device:send:*` 表示"这个方法会按 target 校验"；`gate.check` 用**派生后的**实际标识。CI 检查每个 `gate_host_fn` 的 handler 里**恰好有一次** `ctx.gate.check`（用 AST 检查或 lint）。

### 3.3 匹配语义（精确优先，**且与顺序无关**）

`contains` 在**每次 host call** 上执行，所以它是热路径 —— 类型选择由 [26 §5.2](./26-core-stability-and-perf.md) 规定：
**无分配、O(1) 快速否定、结果不依赖匹配顺序。**

```rust
// crates/synra-capability/src/set.rs

/// ★ 字段全私有：内部结构可以自由优化（换容器不算破坏性变更，[26 §3.2](./26-core-stability-and-perf.md)）
pub struct CapabilitySet {
    /// 精确能力，按域分桶（域的个数很少 → SmallVec 优于 HashMap）
    exact: SmallVec<[(DomainId, SmallVec<[Capability; 4]>); 8]>,
    /// 通配模式，**按 specificity 预排序**（更具体的在前）
    patterns: SmallVec<[Pattern; 8]>,
    /// 域位域：用于"这个集合里根本没有该域的能力"的 O(1) 快速否定
    domain_mask: u32,
}

impl CapabilitySet {
    /// ★ 无分配。语义**不依赖**插入顺序，也不依赖"先匹配到哪个通配"。
    pub fn contains(&self, cap: &Capability) -> bool {
        // 最常见的情况：该域完全没授权 → 一次位运算就否定
        if !self.domain_mask.has(cap.domain()) {
            return false;
        }
        if self.exact_for(cap.domain()).contains(cap) {
            return true;
        }
        self.patterns.iter().any(|p| p.matches(cap))
    }

    // ── 只读访问器（不暴露容器类型，[26 §3.2 E1](./26-core-stability-and-perf.md)）──
    pub fn iter(&self) -> impl Iterator<Item = &Capability> + '_;
    pub fn len(&self) -> usize;
    pub fn is_empty(&self) -> bool;
}

/// `*` 匹配单段，`**` 匹配多段（含零段）
pub struct Pattern { segments: SmallVec<[Segment; 6]> }

impl Pattern {
    /// 段数相等时逐段匹配；`**` 可吞任意多段
    pub fn matches(&self, cap: &Capability) -> bool;
    /// ★ 用于排序：段数多 + 通配少 = 更具体
    fn specificity(&self) -> u32;
}
```

**两条硬性要求**：

| # | 要求                                                        | 检查 |
| - | ----------------------------------------------------------- | ---- |
| 1 | `contains` **零分配**                                        | [26 §6.2](./26-core-stability-and-perf.md) 的分配计数断言 |
| 2 | `contains` 的结果**与插入顺序、通配顺序无关**                    | proptest：打乱 `CapabilitySet` 的构造顺序 → 所有 `contains` 结果不变 |

> **关于第 2 条**：`patterns` 被预排序按 specificity 排列，是为了"如果同时有 `ui:*` 和 `ui:page` 两个模式，
> 谁是'生效的那个'"在 UI 展示与审计里有**确定答案**。但**匹配结果本身**必须与顺序无关 ——
> 只要任一模式涵盖该能力就是命中。

**为什么键的哈希算法也要挑**：`Capability` 的内容来自**插件 manifest**（半可信）。
用无随机种子的快速哈希会让恶意 manifest 构造大量碰撞，把 `exact_for()` 的查找退化成 O(n)。
所以这里用**分桶 + 线性扫小数组**（容量天然很小，没有哈希攻击面），
而不是 `HashSet`。这也是 [26 §4.3](./26-core-stability-and-perf.md) "不受信任的键不要用无种子哈希"的应用。


**必须覆盖的边界用例**（proptest + 显式表驱动）：

| 模式                  | 候选                       | 结果 | 理由                        |
| --------------------- | -------------------------- | ---- | --------------------------- |
| `ui:*`                | `ui:page`                   | ✅   | 单段通配                    |
| `ui:*`                | `ui:page:extra`             | ❌   | 段数不同                    |
| `ui:**`               | `ui:page:extra`             | ✅   | 多段通配                    |
| `ui:**`               | `device:list`               | ❌   | 首段不同                    |
| `network:fetch:**`    | `network:fetch:api.x.com`   | ✅   |                             |
| `network:fetch:**`    | `network:send:api.x.com`    | ❌   | 第二段不同                  |
| `network:fetch:api.x.com` | `network:fetch:api.x.com:POST` | ❌ | 段数不同（无通配）      |
| `ui:page`             | `ui:page`                   | ✅   | 精确                        |
| `log:*`               | `log:debug`                 | ✅   |                             |
| `*`                   | 任何                        | **拒绝** | 纯通配在语法层就非法      |

### 3.4 通配的"授权升级"风险

```rust
// ❌ 危险：用户授予 `network:fetch:*`，等于授予全部域名
// ✅ 推荐：manifest 里逐域名声明，UI 上逐域名确认
```

**规则**：
- 用户界面上，**通配授权必须展开显示**（"允许访问任意网站"而不是"允许网络"）；
- `*` 级别的授权需要**二次确认**（输入插件名 / 勾选显式警告）；
- 插件升级若把精确能力换成通配能力 → **强制重新确认**。

---

## 4. 逃逸面收敛（各能力的硬边界）

### 4.1 `device:send` 的边界

```rust
// 校验顺序（任一失败即拒绝）
1. target 是合法 DeviceId 形态
2. target 在授权集合内（精确或 glob）
3. target 不是 self（防自环）
4. 目标设备已配对（除非授权里有 device:send:*:*:unpaired 这种特例——**不提供**）
5. 目标设备在线（离线 → 返回 DeviceOffline，可选择排队）
6. 事件名合法（§4.5）
7. payload 大小 ≤ 限制（默认 256 KiB，与帧上限一致）
8. 速率限制（§5）
```

### 4.2 `event:emit` 的边界

```
1. 插件只能 emit 自己命名空间下的事件：`_plugin.<self-slug>.*`
   ★ 这是硬约束，不是 capability 能突破的
   （否则插件可以伪造别的插件的事件）
2. 例外：可以使用内核提供的"系统事件发射"白名单（如 file.transfer.*），
   但这些事件的内核侧处理器会校验来源
3. event 名长度 ≤ 256，字符集受限
4. payload 大小 ≤ 限制
```


### 4.3 `fs` 的边界（capability-based 路径）

**不使用字符串前缀比较**。采用"解析后比较路径组件"：

```rust
// crates/synra-store/src/sandbox.rs

pub struct PluginSandbox {
    root: Utf8PathBuf,          // 已 canonicalize
    grants: Vec<Utf8PathBuf>,   // 已 canonicalize 的授权路径
}

impl PluginSandbox {
    /// 把插件提供的相对路径解析为绝对路径，并校验在授权范围内
    pub fn resolve(&self, rel: &str, mode: AccessMode) -> Result<Utf8PathBuf, PathError> {
        // 1. 拒绝绝对路径（插件只能给相对路径）
        if Path::new(rel).is_absolute() { return Err(PathError::Absolute); }
        // 2. 拼接 + lexical normalize（处理 .. 与 .）
        let joined = self.root.join(rel);
        let lexical = lexical_normalize(&joined);
        // 3. ★ canonicalize（解析 symlink）；不存在的父目录用最近的已存在祖先
        let canonical = canonicalize_best_effort(&lexical)?;
        // 4. 校验 canonical 在 root 内（按组件比较）
        if !canonical.starts_with(&self.root) { return Err(PathError::Escape); }
        // 5. 校验在某个授权的 grant 路径内
        if !self.grants.iter().any(|g| canonical.starts_with(g)) {
            return Err(PathError::NotGranted);
        }
        // 6. 校验访问模式（read/write）与 grant 匹配
        Ok(canonical)
    }
}
```


| 平台    | 陷阱                                                       | 处理                                              |
| ------- | ---------------------------------------------------------- | ------------------------------------------------- |
| Windows | 短路径 `PROGRA~1`、大小写不敏感、UNC 路径 `\\?\C:\...`、保留名（`CON`/`NUL`/`AUX`） | 用 `dunce` crate 规范化；比较时大小写不敏感；拒绝保留名 |
| macOS   | HFS+/APFS 默认大小写不敏感（可配）；Unicode 归一化（NFC/NFD） | 比较时大小写不敏感 + Unicode 归一化                 |
| Linux   | symlink、hard link、`..`                                    | `canonicalize` 已处理 symlink；拒绝跨设备 hardlink 检查（可选） |
| 全部    | TOCTOU（检查后替换）                                        | 用**已打开的文件句柄**操作（`cap-std` 风格），而不是"检查路径再打开" |

> **最后一条最重要**：`resolve()` 返回路径后调用方再 `File::open` 之间存在 TOCTOU 窗口。
> 正确做法是 `resolve()` 返回**已经打开的 `File` / `Dir` 句柄**（capability-based 设计），后续操作基于句柄。
> 实现建议：用 `cap-std` crate 的 `Dir` 抽象，或自行封装 `openat` 语义。

### 4.4 `network` 的边界（最容易被忽视的攻击面）

```
校验顺序：
1. URL scheme ∈ {https}（+ 可配置的 http 白名单，默认仅 https）
   ★ 拒绝 file: / data: / ftp: / javascript: 等
2. host 规范化（§2.3）后在授权集合内
3. ★ DNS 解析后校验 IP：
     · 解析出的所有 IP 都必须通过"IP 策略"（§4.4.1）
     · 防止 DNS 重绑定：解析后**用该 IP 连接**，Host header 用原域名
4. 重定向：
     · 最多 5 跳
     · ★ 每一跳都重新执行 1–3
     · 跨域重定向需重新校验（不能"首跳白名单 → 后续任意"）
5. 请求/响应体大小上限
6. 超时（连接 + 总时长）
7. 并发数上限（每插件）
```

#### 4.4.1 IP 策略（防 SSRF）

```rust
pub struct IpPolicy {
    /// 是否允许私网地址
    pub allow_private: bool,             // 默认 false
    /// 是否允许环回
    pub allow_loopback: bool,            // 默认 false
    /// 是否允许链路本地（169.254.0.0/16 —— 云元数据服务！）
    pub allow_link_local: bool,          // 默认 false
    /// 是否允许 IPv6 唯一本地地址
    pub allow_ula: bool,                 // 默认 false
    /// 显式允许的 CIDR（例如局域网内自建服务）
    pub allow_cidrs: Vec<IpNet>,
    /// 显式拒绝的 CIDR（优先级最高）
    pub deny_cidrs: Vec<IpNet>,
}
```

**必须拒绝的地址**（默认）：
- `127.0.0.0/8`、`::1`
- `10.0.0.0/8`、`172.16.0.0/12`、`192.168.0.0/16`
- `169.254.0.0/16`（**云元数据服务 `169.254.169.254`** —— SSRF 的经典目标）
- `0.0.0.0/8`、`224.0.0.0/4`（组播）、`240.0.0.0/4`（保留）
- IPv6：`fc00::/7`、`fe80::/10`、`::ffff:0:0/96`（IPv4 映射）


### 4.5 事件名与 payload 的边界

```rust
pub fn validate_event_name(plugin_slug: &PluginSlug, event: &str) -> Result<()> {
    // 1. 长度 ≤ 256
    // 2. ASCII 可见字符，无空白
    // 3. 如果是 `_plugin.x.y` 形式，x 必须 == 自己的 slug
    // 4. 如果以 `_synra.` 开头 → 除非在系统白名单里，否则拒绝
    //    （白名单：file.transfer.* 这类内核处理的事件）
    // 5. 不允许 `synra.internal.host.` 前缀（host-only，插件不得使用）
}
```

**payload 限制**：
- 序列化后 ≤ 256 KiB（与帧上限一致；超出则提示用 `file.transfer`）；
- 不允许循环引用（JSON 天然不支持，但 Rust 侧要防止栈溢出——用 `serde_json` 的默认深度限制）；
- 深度限制（默认 32 层）。

### 4.6 `platform:*` 的边界（最危险的能力）

| capability                  | 风险                                   | 额外要求                                    |
| --------------------------- | -------------------------------------- | ------------------------------------------- |
| `platform:open-url`         | 钓鱼（打开恶意站点）                     | 显示确认弹窗（可配置为"仅 https"）           |
| `platform:open-path`        | 执行任意文件（如果打开的是可执行文件）      | ★ 拒绝可执行扩展名；或强制"仅用默认应用打开"且不在 PATH 中 |
| `platform:simulate-keys`    | **最强的能力**：可模拟任意按键            | ★ 独立的、默认关闭的授权；UI 上红色警示        |
| `platform:clipboard:read`   | 读取密码等敏感内容                       | 权限 + 用户可见指示器（"某插件正在读剪贴板"）  |
| `platform:clipboard:write`  | 污染剪贴板                               | 权限                                        |
| `platform:notify`           | 骚扰                                    | 速率限制                                    |
| `platform:reveal-path`      | 信息泄露（揭示路径存在性）                | 路径必须在沙箱内                             |

> **`platform:simulate-keys` 需要特别设计**：它能让插件控制用户的其他应用。要求：
> - 默认**不可授予**（需要用户在设置里显式开启"允许插件模拟按键"）；
> - 每次调用在 UI 上有可见指示（托盘图标 / 通知）；
> - 只能在用户前台交互期间调用（不能后台静默连按）；
> - 有速率上限。

---

## 5. 配额与限流

```rust
pub struct Quota {
    // 静态配额（按信任等级）
    pub max_memory_bytes: u64,
    pub max_instances: u32,
    pub max_storage_bytes: u64,
    pub max_open_transfers: u32,

    // 动态限流（令牌桶）
    pub host_calls_per_sec: u32,        // 默认 500
    pub host_calls_burst: u32,          // 默认 1000
    pub events_per_sec: u32,            // 默认 100
    pub network_requests_per_sec: u32,  // 默认 10
    pub network_concurrent: u32,        // 默认 4
    pub device_sends_per_sec: u32,      // 默认 50
    pub fs_ops_per_sec: u32,            // 默认 100

    // 单次调用
    pub call_fuel: u64,
    pub call_timeout: Duration,
}
```

**限流的实现**：

```rust
pub struct RateLimiter {
    buckets: EnumMap<RateKind, TokenBucket>,
}

impl RateLimiter {
    /// 消耗一个令牌；不足则返回 RetryAfter
    pub fn try_consume(&mut self, kind: RateKind, n: u32) -> Result<(), RateLimited>;
}
```

**违规处理策略**：

```
单次超限        → 本次调用失败（PluginQuotaExceeded / RateLimited）
持续超限（10s 内 > 20 次） → 记入插件健康度，UI 显示"该插件请求过于频繁"
健康度过低      → 插件进入 Degraded（降配额）→ 仍持续 → Faulted（停用）
```

**必须有 UI 透明度**：
- 设置页能看到每个插件的：内存占用、调用次数、被拒次数、最近一次拒绝原因；
- 用户可以手动调高某个插件的配额（信任的插件）。

---

## 6. UI 插件的会话令牌

UI 插件跑在 WebView 里，它通过 IPC 调内核。**必须防止**：

- 插件伪造 `pluginId`；
- 插件调用另一个插件的能力；
- 插件在被停用后继续调用。

**方案**：

```rust
/// 内核为每个已激活的 UI 插件签发的会话令牌
pub struct UiSessionToken {
    pub plugin_id: PluginId,
    pub plugin_version: Semver,
    /// 随机 128-bit，不持久化
    pub nonce: u128,
    /// 该会话的有效授予快照（激活时冻结）
    pub grants: CapabilitySet,
    pub issued_at: Instant,
}

pub struct UiSessionRegistry {
    sessions: RwLock<HashMap<u128, UiSessionToken>>,
}

impl UiSessionRegistry {
    pub fn issue(&self, plugin_id: PluginId, version: Semver, grants: CapabilitySet) -> u128;
    pub fn revoke(&self, nonce: u128);
    /// 每次 IPC 调用都校验
    pub fn validate(&self, nonce: u128, claimed_plugin: &PluginId) -> Result<&UiSessionToken>;
}
```

**注入方式**：

```rust
/// Tauri 侧：为插件 WebView 创建独立的 initialization script
/// 把 token 注入为不可枚举的闭包变量（不是 window 上的属性）
let init_script = format!(
    r#"(function() {{
        const token = "{nonce}";
        const invoke = (method, args) => window.__TAURI__.core.invoke('plugin_ipc', {{ token, method, args }});
        Object.defineProperty(window, 'synra', {{
            value: Object.freeze({{ /* 只有方法，没有 token */ }}),
            writable: false, enumerable: false, configurable: false
        }});
    }})();"#
);
```

**要点**：
- token **不出现在 `window` 上**（闭包捕获），插件代码无法读取它；
- 每个插件 WebView 用自己的 token；
- 插件停用 → `revoke(nonce)` → 该 WebView 的所有 IPC 立即失败；
- 插件重新激活 → 新 token（旧的失效）。

> **为什么不能"把宿主对象交给插件"**：同 realm 的对象可被反射探查，插件的"权限"就退化成"你不知道这个对象上还有什么"。本设计的做法是：插件只拿到一个冻结的门面，每个方法都是一次独立校验的 IPC。

---

## 7. 审计

每次闸门检查都产生一条审计记录（在内存环形缓冲 + 可选落盘）：

```rust
#[derive(Debug, Serialize)]
pub struct AuditEntry {
    pub at: SystemTime,
    pub trace_id: TraceId,
    pub plugin_id: PluginId,
    pub plugin_version: Semver,
    pub capability: Capability,
    pub outcome: AuditOutcome,       // Allowed | Denied { reason } | QuotaExceeded | RateLimited
    pub duration: Duration,
    /// 调用摘要（不含敏感数据；例如 network 只记 host，不记 body）
    pub summary: AuditSummary,
}
```

**UI 用途**（设置 → 插件 → 详情）：
- "这个插件最近调用过什么"（时间线）；
- "它申请过但被拒绝的能力"（用户可据此决定是否授权）；
- "网络访问记录"（访问过哪些域名）。

**隐私要求**：
- 审计记录**不包含 payload 内容**（避免记录密码、消息内容）；
- 环形缓冲有上限（默认 10000 条）；
- 用户可清空。

---

## 8. 实现清单

> ★ 这一节在实现之前写就，长期整节未勾选。以下每一条都重新核对过源码。
> **能力层（636-639、644、646、647）是真的、有真测试的；缺的是路径陷阱、网络守卫和恶意插件套件。**
> 名字对不上的按真实名字记（标 `C`）。

- [x] 解析器 + 常量 —— **C**：真实路径是 `crates/synra-capability/src/capability/syntax.rs`
      （不是 `synra-capability::syntax`）；`parse` 在 `:142`，`MAX_LEN = 256`(`:20`)、
      `MAX_SEGMENTS = 6`(`:17`) 与 §2.1 逐字相符；`capability/tests.rs:49-100` 覆盖
      `BareWildcard` / `EmptySegment` / `WildcardPrefix`。
      ★ 但"规范化（含各 domain 的 qualifier 规则）"被代码**明确放弃**了：
      `syntax.rs:316-322` 的注释说按 domain 分开做就是"五个解析器共用一个调用方"，
      `:323` 的 `is_qualifier_segment` 用一套字符类 `[A-Za-z0-9._@/*-]` 管所有 domain。
      没有 host 小写化、没有去尾点、没有 punycode，也没有 fs qualifier 的 canonicalize。
- [x] `*` / `**` 匹配 + 表驱动边界用例 + proptest —— **A**：
      `capability/matching.rs:50` 的 `matches`（一套算法同时处理 prefix 与 qualifier）、
      `:210` `glob_matches`；`capability/tests.rs:202-226` **逐行**搬了 §3.3 的全部十行
      （含 `:213` `ui:*` vs `ui:page:extra` = ❌、`:222` `network:fetch:api.x.com` vs `…:POST` = ❌）；
      proptest 在 `capability/proptests.rs` 与 `set/proptests.rs:32`
      `contains_does_not_depend_on_insertion_order`。
      模块名是 `capability::matching` 而不是 `pattern`，纯命名差异。
      ★ 本次另修：`matches` 原先**每调用分配两个 `Vec`**，已改 `SmallVec`，2 → 0 次分配
      （见 [11](./11-design-principles.md) 与 [26](./26-core-stability-and-perf.md)）。
- [x] `CapabilitySet`、包含关系 —— **C/A**：在 `set/limits.rs:29`（模块叫 `set` 不是 `grant`）；
      `contains`(`:126`)、`grants`(`:150`)、`iter`(`:175`)、`patterns`(`:183`)。
      ★ "通配展开（用于 UI 展示）"其实**不是展开**：`describe_all()`(`:193`) +
      `Capability::describe()`(`capability/consent.rs:15`) 把通配**渲染成一句话**
      （"read from any website"）。没有任何地方把 `network:fetch:**` 展开成具体能力列表。
- [ ] `CapabilityDenied` 结构化错误（含被拒能力、**已授予**、**修复建议**）—— **三要素只做到第一个**。
      没有 `synra-capability::error` 模块，三样东西各答一半：
      `CapabilityError`(`syntax.rs:40`) 是**解析**错误；
      `GateRefusal::NotGranted{capability}`(`gate.rs:16-23`) 只带被拒的那个能力；
      面向插件的是 `CallError::denied`(`host_calls.rs:81`) + `ErrorCode::CapabilityDenied`
      + `AdmissionRefusal`(`synra-core/src/admission.rs:264`)。
      没有一个带上**已授予集合**或**修复建议**；最接近的 `admission.rs:266` 只是散文式地列出**缺失**项。
- [x] host fn 都过闸门 —— **C，且 CI 检查曾经是空转的**。
      两个宏全仓无定义；真实机制更强：`build.rs` 每次从 `wit/synra-plugin.wit` 重新生成
      `capability_table.rs`，`host_calls.rs:374-380` 拒绝表外一切调用，
      `host_surface.rs` 有三个测试（免检白名单 ≤10 条且每条必须是事实读取 / 危险清单点名 / 两表恰好划分全集）。
      ★ `tools/guards` 里那条标着 "10 §3.1" 的守卫**曾经永远不可能失败**：
      它 grep 的 `ungated_host_fn!` 只出现在它自己内部，它白名单的
      `crates/synra-plugin-host/src/host_funcs/env.rs` 连目录都不存在。
      **本次已重写**为直接读生成表、要求每个 `None` 条目属于 `env-` / `platform-capabilities` /
      `plugin-id` 三族，并加了"解析出 0 条即报错"的自检；负控制（把 `network-fetch` 改成 `None`）确认它会红。
- [x] 路径解析 + canonicalize + 组件级前缀比较 —— **C/A**：真名 `PluginDataRoot`
      （`synra-store/src/sandbox.rs:99`，不是 `PluginSandbox`）；`resolve()`(`:198`) 先
      canonicalize 父目录(`:229`)、逃逸在 `:236` 被拒、判定用 `paths.rs` 的 `is_within`
      （按 `Path::components()` 逐段比较，`:522` 有 proptest）。真实逃逸测试（含 symlink）在 `:474`。
      ★ **"TOCTOU 安全（句柄式）"没有做到**：返回的是 `PathBuf` 而不是已打开的 `File`/`Dir` 句柄。
      模块自己的注释（`sandbox.rs:14-17`）就写明这是已知差距，`cap-std` 也**不是依赖**。
- [ ] Windows 路径陷阱处理（短路径、保留名、UNC）—— **未做**：全仓 0 处
      `dunce` / 保留名（`CON`/`NUL`/`AUX`）/ 8.3 短路径（`PROGRA~1`）/ `\\?\` verbatim 处理。
      只有三处 `cfg!(windows)` 测试字面量。
      ★ 而且有一个**方向相反的**性质：`is_within` 用 `Path::components()` 的 `PartialEq`，
      在 Windows 上**大小写敏感**、且把 `\\?\` 当不同 Prefix 组件，所以 `C:\Data` 与 `c:\data`
      会被判为不同。这是 fail-closed（安全），但与 §4.3 要求的"比较时大小写不敏感"正相反。
- [x] `NetworkGuard`：scheme 白名单 + host 规范化 + DNS 后 IP 校验 + 重定向逐步校鮌 + SSRF 拒绝表 ——
      **已实现**（已提交）：`synra-core/src/network.rs`。四步按 `10 §4.4` 的顺序：
      `check_url`（scheme + host）、`check_resolved`（IP 策略）、`next_hop`（逐步重校），
      `IpPolicy` 六个字段与 `Cidr` 都在，默认全 `false`。
      ★ **而第三步为什么是参数而不是调用**：这个 crate 不做 DNS，
      而一个会解析域名的守卫是一个**离线就测不了**的守卫。
      ★ **七个绕过都有对应的测试**，而两个最典型的做了阴性对照：
      * `https://api.example.com@evil.example/` —— 取 `@` **之后**的部分；取之前的会把
        授权给一台机器然后连另一台。对照失败。
      * **每一个**解析出的地址都要过，不是第一个：对照失败。
      * `169.254.169.254`（云元数据）、`::ffff:127.0.0.1`（换了家族的环回）、
        `api.example.com.evil.example`、`/33` 的 CIDR、超限重定向 —— 各有一条。
      * 证据：**55 suites / 2820 passed**（+22），clippy、7 guard（含兄弟 app 构建）均绿。
      ★ **但它还没有生产调用方**：`network-fetch` 本身仍然拒绝（仓库里没有
      HTTP 客户端）。所以这是**把守卫先建好**，而不是“拥有了护拦”。
>
> ### ★ 而现在可以写了 —— 下一轮要用的事实在这里，不要重新调查
>
> **1. 客户端依赖离线可用。** `ureq 3.4.2` 已在本机 registry 缓存里，带 `rustls` + `gzip`；
> 已用一个临时 crate 验证 `cargo build --offline` **能解析并编译**。TLS 栈（`rustls`/`ring`/`aws-lc-rs`/`webpki-roots`）也全在缓存里。
>
> **2. 放在哪里：新建一个宿主侧 crate，不是 `synra-net`。**
> * `synra-net` 的自述是「两台设备之间、socket 已经建立之后的事」，而且它**不做 IO**；HTTP 客户端两条都违背。
> * 放进 `synra-core` 会把 **TLS 拖进内核的依赖图**，而 `synra-core` 是要向 Android 交叉编译的（见 `matrix`）。
> * 所以：一个只有宿主依赖的 crate，内核的依赖图不变。
>
> **3. DNS 重定向防御是可实现的，不是口号。** `ureq` 有 `Resolver` trait（`unversioned/resolver.rs:29`），
> 经 `Agent::with_parts(config, connector, resolver)`（`agent.rs:130`）装入。于是可以做成：**解析 → 守卫校验每一个地址
> → 只把通过的地址交给 ureq**，而证书仍然是对**原域名**验的。
> 把 URL 直接换成 IP 是错的（证书校验会失败，或更糟的：被绕过），而守卫里已经写下了这一点。
>
> **4. 自动重定向必须关掉。** 设计要求**每一跳都重新校验**，而 `ureq` 的 `max_redirects` 是自己跟：
> 能限次数，不能每跳都问 `NetworkGuard::next_hop`。所以要 `max_redirects(0)` +
> `max_redirects_will_error(false)`，然后**自己跟**，每跳调一次 `next_hop`。
>
> **5. 响应体上限有现成的。** `BodyWithConfig::limit(u64)`（`body/mod.rs:502`）。
> 而 `request-json` 的**具体形状文档没定**（和 intent 的 payload 一样），所以它要像
> `synra_core::intent::remote` 那样，**把决定写在模块文档里**，而不是散在代码里。
>
> ★ **已实现（已提交）**：新 crate `crates/synra-fetch`。`Client::fetch` 把上面五条全部落了地：
> 解析 → `check_resolved` → `FixedResolver` 只交出通过的地址（证书仍验原域名）→
> `max_redirects(0)` + 自己的重定向循环（每跳过 `next_hop`）→ 响应体限额在**读的时候**生效。
> * ★ **测试真的开了 socket**：一个 `TcpListener` 写固定响应，因为这个模块的每一个判断都是关于
>   一个**即将发出的请求**，而把传输换成桩就只能测桩自己对守卫的理解。
> * ★ **两个阴性对照**：让客户端自己跟重定向（→ 5 个测试红）；
>   把响应体上限改回 `u64::MAX`（→ `a_body_larger_than_the_limit...` 红）。
> * ★ **而它捕到了两件事**：
>   1. `ureq` 的 `limit(n)` 在剩余为 0 时读一次就报错，而 `read_to_string` 总会多读一次看结尾
>      —— 所以**恰好等于上限的响应体会被拒**。传 `limit + 1` 才让“超过才拒”名副其实。
>   2. 我的第一版重定向测试转向了**同一台主机的另一个端口**，而它**通过了守卫**
>      —— 因为能力授权的是 **host**（它的拼写就是 `network:fetch:<host>`），端口不在授权里。
>      是测试错了而不是守卫错了；现在改成转向 `localhost`，并另加一条**断言端口确实不在授权里**。
>
> ★ **而它还在等一个宿主**：`KernelHost::fetch`/`DesktopHost::fetch` 仍然拒绝，
> 所以插件还是发不出请求。下一步是把 `Client` 接到两个宿主上。
      整个 workspace **没有任何 HTTP 客户端**（搜 `reqwest`/`hyper::`/`ureq`/`isahc` 只有 `Host::fetch`
      的 trait 声明与测试桩）。`ErrorCode::IpNotAllowed`/`RedirectBlocked` 有码无生产者。
> ★ **而它当时没有生产调用方 —— 因为守护进程把整组 `platform:*` 一律拒绝了。**
> `KernelHost::platform` 原来对每一个平台动作都答 `notImplemented`，
> 而 `synra-platform` 实际上实现了整条受守护的路径（`desktop.rs` / `host.rs` 都调 `check_url`/`check_path`）。
> ★ **而 `HeadlessPlatform` 的注释早就说明了为什么那不对**：
> “一个传了 `javascript:` 的插件应该被告知**它的地址被拒了**，
> 而不是“这个孿主没有浏览器”。前者是作者能修的错误，后者是关于安装的事实。”
> 现在 `KernelHost::platform` 委派给 `HeadlessPlatform`：先跑守卫，再拒绝，
> 并把 `PlatformError` 按原因映到 `badArgument`（插件的错）或 `notImplemented`（安装的事实）。
> ★ 而且 `Policies` 的构造器与访问器也没有调用方 —— 政策永远是默认值（严格的那个）。
> 这是安全的，但意味着**没有任何地方能改它**；记在这里而不是当成已完成。
      `guard.rs:21` 的 `check_url` 是 `platform:open-url` 的 scheme/host 存在性检查，不做解析。
      ★ 同时也说明：**SSRF 这一类漏洞目前无法被触发，因为还没有东西会去发请求** —— 这是"没有功能"而不是"有漏洞"。
- [x] 各 RateKind 的令牌桶 —— **C**：`RateLimiter` 确实存在
      （`synra-platform/src/guard.rs:200`）但它是**每个 `PlatformAction` 的冷却/小时窗**限流器，与 `RateKind` 无关。
      文档要的令牌桶在 `synra-plugin-host/src/quota/`：`RateKind`(`limits.rs:13`)、
      `TokenBucket::try_consume`(`ledger.rs:79`)、`Budget::consume`(`budget.rs:237`)，
      `quota/proptests.rs:18` 有 proptest。性质完全成立，只是名字和 crate 换了 —— **是两套独立机制**。
       > ★ **而 `Quota` 的十二个字段里，只有“速率”那五个真的被强制。** 逐条核对：
       > 被强制的：`host_calls_per_sec` / `host_calls_burst`（以及其余四个 kind，
       > 经 `Quota::rate_for`(`limits.rs:130`) → `Budget::consume`(`budget.rs:237`)，
       > `host_calls.rs:374-390` 的 `rate_kind()` 负责分类）。
       > **没被强制的**：`max_memory_bytes`、`max_instances`、`max_storage_bytes`、`max_open_transfers`、
       > `call_fuel`、`call_timeout_ms`、`network_concurrent`。
       > * ★ 根因是 `CallBudget::from_quota`(`engine/types.rs:233`) **在生产里从来没有调用方**——
       >   只有 `scripted.rs:691` 的 `enforcement` 和一个测试用。它的注释写着
       >   "Every field is taken from the quota rather than chosen here, so the number a user sees in a settings
       >   screen is the number the sandbox enforces"，**而那句话当时只有三分之一是真的**。
       > * 已修的是 `call_fuel` / `call_timeout_ms` / `max_memory_bytes`：`share.execute` 现在用
       >   `PluginSet::quota()` → `CallBudget::from_quota`，而不再是一个固定的 5 秒超时 + `fuel: None`。
       >   ★ 那个 `fuel: None` 值得单独说：**它意味着执行一个动作时完全没有指令上限**，只有墙钟。
       > * 仍然没强制的：`max_instances`（实例上限走的是 `PluginSpec.limit`，**不是** quota —— 两个来源）、
       >   `max_storage_bytes` 与 `max_open_transfers`（这两个守的功能本身还没实现，所以“没强制”是对的）、
       >   `network_concurrent`（无并发请求这回事）。
       > * ★ **为什么之前看不出来**：速率**是**被强制的，而“一个能做事的 quota”看起来就像“一个能用的 quota”。
- [x] `UiSessionRegistry`：签发/校验/撤销 —— **A，但"注入"那半是 B**。
      名字完全一致：`synra-plugin-ui/src/session/registry.rs:17`，`issue_with`(`:46`)、
      `validate`(`:140`，要求 nonce + plugin_id + version 三者一致)、`revoke`(`:113`)、
      过期清扫(`:232`)；proptest `session/proptests.rs:66`
      `a_revoked_nonce_is_indistinguishable_from_an_unknown_one`。
      ★ **Tauri initialization script 注入完全没有**：`synra-rust-app/src-tauri/Cargo.toml`
      **根本没有依赖 `synra-plugin-ui`**，全仓 0 处 `initialization_script`。注册表实现了，app 没用它。
- [x] 环形缓冲 + 无 payload + UI 查询接口 —— **C/A**：真名 `AuditRing`
      （`synra-observe/src/audit.rs:179`，不是 `AuditLog`）。`MEMORY_CAPACITY = 10_000` 与 §7 一致；
      无 payload 是**构造性**的（`AuditEntry` 的字段没有一个能装内容，`audit.rs:437` 有断言）；
      UI 查询接口齐全（`for_plugin`/`refusals_for_plugin`/`PluginActivity::of`/`clear_plugin`/`clear`）。
      ★ 但**生产代码从不写它**：`Kernel::record_action` 只在测试里被调用（见 [18](./18-observability.md)）。
- [x] 装载期 `verify_effective_caps`：manifest ∩ grant ∩ wasm imports —— **A/C**：
      同名函数在 `set/interface.rs:67`，四步齐全：grant ⊆ manifest(`:74-80`)、
      每个 import interface 被覆盖(`:83-87`)、effective 取交集(`:92-97`)、
      已授权但未 import → 警告而不拒绝(`:100-104`)；测试 `set/tests.rs:206,219,231,246,260`。
      ★ 签名漂移：文档写 `(&PluginManifest, &Grant, &ImportSet)`，真实是
      `(&CapabilitySet, &CapabilitySet, &ImportSet)` —— 两者都已解析过，
      "manifest 声明必须语法合法"是在构造 `CapabilitySet` 时强制的，不在这个函数里。
- [ ] 恶意插件测试套件（[21](./21-testing-strategy.md) §5）—— **未做**：没有
      `PluginHostFixture::install("malicious-loop")` 之类的套件，没有 `tests/malicious*`，
      **也没有任何测试断言 §5.1 那句"内核仍可用 + 其它插件仍可用"**。
      29 条里约三分之一有孤立的对抗测试（归档穿越/symlink/体积、签名吊销、令牌伪造、装载期 import 覆盖），
      但 SSRF 那 11-15 行**无法测**（没有 fetch 实现可攻击），§5.2 的 UI 插件 10 行整组没测。
