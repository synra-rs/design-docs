# 03 · 架构基线

> 本文定义**三环模型、状态归属、分层与依赖方向**。
> 它是所有实现文档的地基：任何人想加一个模块，先在这里找到它属于哪一环、状态归谁。
> 细节分别见 [05 系统架构](./05-system-architecture.md)（进程/线程）与 [07 领域内核](./07-core-kernel.md)（领域模型）。

---

## 1. 三环模型

```
╔═══════════════════════════════════════════════════════════════════════════════╗
║  第 3 环 · 表示层（Presentation）  ——  synra-app 仓库                     ║
║                                                                               ║
║   Tauri WebView                                                               ║
║     ├─ 宿主 UI：Home / Plugins / Devices / Settings                            ║
║     └─ 插件 UI：每个 UI 插件一个隔离的渲染上下文（独立 origin）                   ║
║                                                                               ║
║   与内核的唯一通道：Tauri IPC（command 调用 + event 订阅），类型由 Rust 生成      ║
╚═══════════════════════════════════╦═══════════════════════════════════════════╝
                                    ║  IPC（强类型，可审计）
╔═══════════════════════════════════▼═══════════════════════════════════════════╗
║  第 2 环 · 宿主壳（Host Shell）  ——  synra-app 仓库                       ║
║                                                                               ║
║   Tauri 应用                                                                   ║
║     ├─ 窗口 / 菜单 / 托盘 / 通知 / 深链 / 文件关联                              ║
║     ├─ command 注册（薄适配层，不含业务）                                        ║
║     ├─ 平台能力实现（trait Platform 的 GUI 侧部分）                              ║
║     ├─ UI 插件承载（WebView 创建、CSP、令牌注入）                                ║
║     └─ 内核生命周期管理（启动 / 关停 / 单实例）                                   ║
╚═══════════════════════════════════╦═══════════════════════════════════════════╝
                                    ║  直接函数调用（同进程）
╔═══════════════════════════════════▼═══════════════════════════════════════════╗
║  第 1 环 · 内核（Kernel）  ——  synra-core 仓库                            ║
║                                                                               ║
║   synra-core        领域模型、聚合根、状态机、编排                              ║
║   synra-protocol    线协议（帧、信封、事件、会话）—— 无 IO                       ║
║   synra-capability  capability DSL（解析、匹配、授权集）—— 纯函数               ║
║   synra-net         发现（UDP/mDNS）、传输（TCP）、心跳、重连                    ║
║   synra-plugin-host Wasm 沙箱、能力闸门、插件生命周期                            ║
║   synra-plugin-ui   UI 插件 artifact 管理（不含渲染）                            ║
║   synra-store       SQLite / 文件沙箱 / 密钥                                     ║
║   synra-platform    trait Platform + 各平台实现                                 ║
║   synra-observe     tracing / metrics / 诊断包                                  ║
║                                                                               ║
║   全部为纯 Rust，无 UI 依赖，可单独测试，可被 CLI / daemon 复用                   ║
╚═══════════════════════════════════════════════════════════════════════════════╝
```

### 1.1 三条铁律

| # | 铁律                                                              | 由什么保证                                      |
| - | ----------------------------------------------------------------- | --------------------------------------------- |
| 1 | **内核不知道 UI 存在。** 内核 crate 不得依赖 `tauri` / `wry` / 任何 GUI 库 | 物理隔离（两个仓库）+ CI 守卫（`cargo tree`）      |
| 2 | **前端不知道网络存在。** 前端不持有 socket、不解析协议，只与内核 IPC 对话   | `trait` 边界 + CSP                             |
| 3 | **插件不能直接碰内核。** 插件只能通过沙箱边界（Wasm host function / UI IPC）交互 | Wasm 沙箱 + 会话令牌                     |

---

## 2. 依赖方向

```
synra-protocol        ← 无内部依赖（只有 serde / bytes / 时间）
synra-capability      ← 无内部依赖
      │
synra-plugin-api      ← protocol + capability；WIT 生成的绑定 + 纯类型
synra-store           ← protocol
synra-observe         ← 无（tracing/metrics facade）
      │
synra-net             ← protocol + observe
synra-plugin-host     ← plugin-api + capability + observe
synra-plugin-ui       ← plugin-api + store
      │
synra-platform        ← trait 定义（无实现依赖）
      │
synra-core            ← 以上全部（唯一"知道全局"的 crate）
      │
apps/{cli,daemon}     ← synra-core + synra-platform（HeadlessPlatform）
tools/synra-plugin-cli ← plugin-api + wit-codegen
```

**禁止的依赖方向**（CI 守卫强制，见 [06 §7.1](./06-workspace-layout.md)）：

| # | 禁止                                                     | 理由                                    |
| - | -------------------------------------------------------- | --------------------------------------- |
| 1 | 任何 `crates/*` 依赖 `tauri` / `wry` / `tao`               | 铁律 1                                  |
| 2 | `synra-protocol` 依赖 `tokio` / `rusqlite` / `wasmtime`    | 协议是纯数据，必须能被任何环境消费         |
| 3 | `synra-net` 依赖 `synra-plugin-*`                          | 网络不关心插件                           |
| 4 | `synra-plugin-api` 依赖 `synra-core`                        | 插件契约必须与内核实现解耦                 |
| 5 | `crates/*` 被 `apps/*` 反向依赖                             | 结构                                    |
| 6 | 任何 crate 直接 `tokio::spawn` 长驻任务                      | 必须经 [07 §5 的 supervisor](./07-core-kernel.md) |

---

## 3. 平台抽象

内核需要宿主提供的"系统能力"通过一个 trait 注入，**不是**在业务里写 `if platform == ...`。

```rust
// crates/synra-platform/src/lib.rs

/// 宿主提供的系统能力。**实现方**：桌面（Tauri）、Android、iOS、headless。
#[async_trait]
pub trait Platform: Send + Sync + 'static {
    fn kind(&self) -> PlatformKind;

    // ── 打开与展示 ──
    async fn open_url(&self, url: &Url) -> Result<()>;
    async fn open_path(&self, path: &Utf8Path) -> Result<()>;
    async fn reveal_path(&self, path: &Utf8Path) -> Result<()>;

    // ── 输入模拟（能力可用性由 capabilities() 声明）──
    async fn simulate_keys(&self, chord: KeyChord) -> Result<()>;

    // ── 通知 ──
    async fn notify(&self, n: Notification) -> Result<()>;

    // ── 剪切板 ──
    async fn read_clipboard(&self) -> Result<ClipboardContent>;
    async fn write_clipboard(&self, c: ClipboardContent) -> Result<()>;
    /// 捕获当前系统选中的文本（桌面专属；不支持则 Unsupported）
    async fn capture_selection(&self) -> Result<String>;

    // ── 能力自述（用于路由决策与 UI 禁用态）──
    fn capabilities(&self) -> &PlatformCapabilities;
}
```

### 3.1 实现矩阵

| 实现                | 目标                    | 说明                                                   |
| ------------------- | ----------------------- | ------------------------------------------------------ |
| `DesktopPlatform`   | Windows / macOS / Linux | Tauri 侧实现（`opener` / `clipboard-manager` / `notification`） |
| `AndroidPlatform`   | Android                 | Intent 打开 URL、通知；模拟按键需无障碍服务（可能不可用） |
| `IosPlatform`       | iOS                     | `UIApplication.open`、通知；**无模拟按键**（返回 `Unsupported`） |
| `HeadlessPlatform`  | CLI / daemon / 测试      | 全部返回 `Unsupported` 或记录到日志                      |
| `MockPlatform`      | 测试                    | `synra-testkit` 提供，记录调用供断言                      |

**要点**：

1. **不支持的能力返回 `Err(Error::Unsupported)`，不静默 no-op。** 与 [01 §4 P10](./01-charter.md)（默认拒绝）一致。
2. **`capabilities()` 让内核能提前决策**：例如 iOS 上不支持 `simulate_keys`，那么目标是 iOS 的
   "模拟按键"意图应该在**下发前**就被拒绝并给出明确提示（[02 §3 步骤 6](./02-domain-and-flows.md)），
   而不是让插件白跑一趟。
3. **`HeadlessPlatform` 的存在是"内核可 headless"的证明**（[01 G3](./01-charter.md)）。

---

## 4. 状态归属（唯一权威）

**这是本文最重要的一节。** 每个状态只有一个所有者；其余都是**从它派生的视图**。

| # | 状态                              | 所有者（聚合根）      | 生命周期 | 持久化 |
| - | --------------------------------- | ------------------- | -------- | ------ |
| 1 | 本机 `device_id` / `display_name` / profile | `Identity`     | 进程     | ✅     |
| 2 | 候选设备（UDP/mDNS/探测结果）        | `Discovery`         | 会话     | ❌     |
| 3 | 本机 IP 集合（用于自我抑制）          | `Discovery`         | 会话     | ❌     |
| 4 | 已配对设备记录                       | `Trust`             | 进程     | ✅     |
| 5 | 待接受的配对请求                     | `Trust`             | 会话     | ❌     |
| 6 | 链路状态（`Connecting`/`Ready`/…）   | `Transport`         | 会话     | ❌     |
| 7 | 链路的重连调度状态                    | `Transport`         | 会话     | ❌     |
| 8 | 进行中的意图                         | `Intent`            | 会话     | ⚠️ 可选 |
| 9 | 幂等去重表（`request_id` → 结果）     | `Intent` / `Transfer` | 会话   | ❌     |
| 10 | 插件清单 / 安装记录 / 启用状态        | `Plugins`           | 进程     | ✅     |
| 11 | 插件授权（grant / deny）             | `Plugins`           | 进程     | ✅     |
| 12 | 插件运行实例 / 实例池                 | `Plugins`           | 会话     | ❌     |
| 13 | 插件 artifact（磁盘）                | `Plugins`（落盘）+ `PluginUi`（UI 部分） | 持久 | ✅ |
| 14 | 传输会话 / 组装缓冲 / 续传位图         | `Transfer`          | 会话     | ⚠️ 可选 |
| 15 | 用户配置                            | `Config`            | 进程     | ✅     |
| 16 | 审计记录                            | `Observe`（环形 + DB） | 进程   | ⚠️ 有保留期 |
| 17 | **UI 视图状态**（侧栏折叠、主题）      | **前端**             | 会话     | ❌     |
| 18 | **插件页面内的视图状态**              | **插件自己**         | 会话     | ❌     |

### 4.1 判定规则

> 凡是「跨设备 / 跨重启 / 被权限约束」的状态 → **内核**
> 凡是「纯展示 / 纯本地交互」的状态 → **前端**

### 4.2 `DeviceCard` 是派生视图（不是状态）

```
DeviceCard = f(Discovery, Trust, Transport, Identity)
```

| 输入                    | 提供什么                             |
| ----------------------- | ------------------------------------ |
| `Discovery`             | 地址（host / port）、mDNS 里的名字提示 |
| `Trust`                 | 是否已配对、最后解析到的地址            |
| `Transport`             | 链路就绪？连接中？上次断链原因？         |
| `Identity`              | 排除自己                             |

**禁止**：任何模块自己维护一份"设备列表"。
`DeviceCard` 每次从这四个聚合根算出来（可以缓存，但缓存必须是派生缓存，不是第二权威）。

> **为什么强调这条**：如果"配对列表"和"连接列表"各自维护，就会出现
> "配对里有、连接里没有"、"两边名字不一样"这类不一致，且没人说得清哪个对。
> 唯一权威 + 派生视图从结构上消灭了这个问题。

---

## 5. 一次动作的完整路径

以"手机分享 → 桌面执行"为例（[02 §3](./02-domain-and-flows.md) 的旅程在架构上长什么样）：

```
① UI 事件（WebView）
     用户点"确认"
② 单次 IPC：invoke("intent_decide", { intentId, target })
     ★ 强类型，类型由 Rust 生成（[08 §3](./08-ipc-and-frontend.md)）
③ 内核：Intent 聚合根
     · capability 闸门（三件事：插件授权 / 目标已装 / 目标平台支持）
     · 构造 Envelope（trace_id 从这里诞生）
     · 交给 Transport
④ 传输：内核内 async task（结构化并发，受 supervisor 管理）
     · 帧编码 → TCP
⑤ 对端内核：同样的 ③④ 反向
     · 解码 → 信封校验 → 事件路由 → 闸门 → 调插件的 execute()
⑥ 事件推送：内核 → WebView（KernelEvent，带 trace_id）
     前端只消费事件，不轮询
```

**从 UI 到网络只有 3 跳，中间全是强类型 Rust。**

对照一下"如果按老思路做"会多出什么（这些正是 [11](./11-design-principles.md) 要避免的）：

| 老思路的做法                              | 代价                                    |
| ----------------------------------------- | --------------------------------------- |
| 业务逻辑写在 composable 里，靠多个 ref 串   | 状态无权威、竞态、组件重渲染难以预测        |
| 平台差异用 `if (isDesktop) ... else ...`    | 每加一个平台改很多处                       |
| 事件靠自定义 DOM 事件 + 各处 `on/off`        | 泄漏、顺序不确定、跨进程两套映射             |
| 错误靠字符串匹配来判断"是否该静默"            | 会误静默真实错误                          |

---

## 6. 分层职责速查（加模块时先查这里）

| 你想加的东西                     | 放哪                                              |
| -------------------------------- | ------------------------------------------------- |
| 新的线上事件                      | `synra-protocol`（先改 [04](./04-wire-protocol.md)） |
| 新的 capability 域               | `synra-capability`（先改 [10](./10-capability-and-sandbox.md)） |
| 新的平台能力                     | `trait Platform` + 各实现（先改本文 §3）              |
| 新的领域状态                     | 一个聚合根里（先改本文 §4）                          |
| 新的 Tauri command               | `synra-app/src-tauri/commands`（先改 [08](./08-ipc-and-frontend.md)） |
| 新的前端 store                   | `synra-app/src/stores`（必须是只读镜像或纯 UI 状态） |
| 新的插件能力（host function）      | `synra-plugin-host/host_funcs`（必须过闸门，[10 §3.1](./10-capability-and-sandbox.md)） |
| 新的长驻后台任务                  | `Supervisor` 里的一个 `TaskSpec`（[07 §5](./07-core-kernel.md)） |
| 新的持久化表                     | `synra-store` 的迁移（先改 [17](./17-storage-and-config.md)） |
| 新的指标 / 日志字段               | `synra-observe`（先改 [18](./18-observability.md)） |
