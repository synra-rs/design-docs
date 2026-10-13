# 06 · Workspace 与 Crate 规划

> 本文定义 **`synra-core` 仓库**（[README §1](./README.md)）的物理结构：有哪些 crate、谁依赖谁、哪些 feature gate 控制编译期形态。
> 目标：**内核可单独编译、单独测试、单独部署**；宿主（`synra-app`）是它的一个消费者，且**不在本仓库里**。
>
> ⚠️ **拆仓库的影响**：`src-tauri` 与 `gen/`（即 Tauri 桌面/移动）**不在本仓库**（见 [README §1](./README.md)）。
> 本仓库的 `apps/` 下只有 `synra-cli` 与 `synra-daemon`——它们只是内核的**薄宿主**，与内核同仓库有明确理由（见 §4.2）。

---

## 1. 顶层目录结构（`synra-core`）

```text
synra-core/                   # ← Cargo workspace 根 = git 仓库根
├─ Cargo.toml                      # workspace 根
├─ Cargo.lock
├─ README.md / LICENSE (MIT)
├─ STABILITY.md                    # ★ 稳定性分级（哪些 crate 遵守 SemVer）
├─ rustfmt.toml / deny.toml
├─ .cargo/config.toml              # 目标平台别名、链接参数
├─ justfile + xtask/               # 统一任务入口 + 结构守卫
├─ design-docs/                        # 本文档集里归 core 的部分（见 README §2 分流表）
├─ crates/
│  ├─ synra-core/                  # ★ 领域内核：模型、聚合根、状态机、编排
│  ├─ synra-protocol/              # ★ 线协议：信封、事件、帧、版本协商
│  ├─ synra-net/                   # ★ 网络：发现、传输、心跳、重连
│  ├─ synra-plugin-api/            # ★ 插件契约（纯类型 + WIT 绑定，无实现）
│  ├─ synra-plugin-host/           # ★ 插件宿主：Wasm 沙箱、闸门、生命周期
│  ├─ synra-plugin-ui/             # UI 插件 artifact 管理（不含渲染）
│  ├─ synra-store/                 # 持久化：SQLite、文件沙箱、密钥
│  ├─ synra-capability/            # capability DSL：解析、匹配、授权集
│  ├─ synra-observe/               # tracing / metrics / 诊断包
│  ├─ synra-platform/              # Platform trait + 各平台实现
│  └─ synra-testkit/               # 测试夹具（publish = false）
├─ apps/
│  ├─ synra-cli/                   # CLI（复用内核）
│  └─ synra-daemon/                # 无 UI 守护进程（可选形态）
├─ tools/
│  ├─ synra-plugin-cli/            # 插件作者工具：new/build/verify/sign/publish
│  └─ wit-codegen/                 # WIT → Rust/TS 绑定生成
├─ wit/                            # 插件接口定义（组件模型）
│  ├─ synra-plugin.wit             # 主世界
│  └─ deps/                        # 依赖的 WIT 包（wasi 等）
├─ spike/                          # M0 的 8 个技术验证（[19 §0.1](./19-roadmap.md)）
└─ testharness/
   └─ fixtures/                    # 协议编解码 fixture（[21 §3](./21-testing-strategy.md)）
```

> **`ui/` 不在这里**。前端（宿主 UI + 插件 UI 运行时）在 `synra-app` 仓库。
> core 只**生成** TypeScript 类型（`ts-rs`）与 UI 插件的运行时**形状描述**，不包含 Vue 代码。

---

## 2. Crate 依赖图

```text
                      ┌──────────────────┐
                      │  synra-protocol  │  ← 无内部依赖（只有 serde/bytes/时间）
                      └────────┬─────────┘
                               │
              ┌────────────────┼─────────────────┐
              │                │                 │
     ┌────────▼───────┐  ┌─────▼──────────┐  ┌───▼──────────────┐
     │ synra-capability│  │ synra-store    │  │ synra-observe    │
     │ （DSL 与匹配）  │  │ （持久化）      │  │ （日志/指标）     │
     └────────┬───────┘  └─────┬──────────┘  └───┬──────────────┘
              │                │                 │
              └────────────────┼─────────────────┘
                               │
                      ┌────────▼─────────┐
                      │  synra-plugin-api │  ← WIT 生成的绑定 + 纯类型（无实现）
                      └────────┬──────────┘
                               │
     ┌─────────────────────────┼───────────────────────────┐
     │                         │                           │
┌────▼──────────────┐  ┌───────▼────────────┐  ┌───────────▼──────────┐
│ synra-plugin-host │  │ synra-plugin-ui    │  │  （插件 SDK 侧）       │
│ （wasmtime 沙箱） │  │ （artifact 管理）   │  │  synra-plugin-sdk     │
└────────┬──────────┘  └───────┬────────────┘  └──────────────────────┘
         │                     │
         └──────────┬──────────┘
                    │
           ┌────────▼─────────┐
           │   synra-net      │  ← 只用 protocol + observe
           └────────┬─────────┘
                    │
           ┌────────▼─────────┐
           │   synra-core     │  ← 编排所有子系统（唯一"知道全局"的 crate）
           └────────┬─────────┘
                    │
     ┌──────────────┼──────────────┬──────────────────┐
     │              │              │                  │
┌────▼──────────┐ ┌─▼───────────┐ ┌▼──────────────┐ ┌─▼──────────────┐
│synra-platform │ │ synra-cli   │ │ synra-daemon  │ │synra-plugin-cli│
│（trait+实现）  │ │ （薄宿主）   │ │ （薄宿主）     │ │ （插件作者工具） │
└───────────────┘ └─────────────┘ └───────────────┘ └────────────────┘
        └──────────────────── 本仓库边界 ────────────────────┘

                 ╔═══════════════ 另一个仓库 ═══════════════╗
                 ║  synra-app                          ║
                 ║   src-tauri/  ── 依赖 synra-core（版本依赖）║
                 ║   src/        ── 消费生成的 TS 类型        ║
                 ╚══════════════════════════════════════════╝
```

> **依赖方向的关键**：`synra-app` 依赖 `synra-core`，**反之永远不成立**。
> 这是拆仓库（[ADR-016](./20-adr.md)）的最大收益——它由**物理隔离**保证，而不是靠评审纪律。

### 2.1 依赖方向铁律

| # | 规则                                                                       | 强制手段                                        |
| - | -------------------------------------------------------------------------- | ----------------------------------------------- |
| 1 | `synra-core` 及以下**不得**依赖 `tauri` / `wry` / `tao` / 任何 GUI crate      | `cargo-deny` banned list + CI 检查 `cargo tree`  |
| 2 | `synra-protocol` 不得依赖 `synra-core`（协议是纯数据）                        | 依赖图方向                                       |
| 3 | `synra-plugin-api` 不得依赖 `synra-core`（插件契约与内核解耦）                 | 同上                                             |
| 4 | `synra-net` **不得**依赖 `synra-plugin-*`（网络不关心插件）                    | 同上                                             |
| 5 | 任何 crate 不得直接 `tokio::spawn` 长时间任务，必须经 `synra-core` 的 supervisor | code review + 自定义 clippy 规则（后续）         |
| 6 | `apps/*` 不得被 `crates/*` 依赖                                              | workspace 结构                                   |

> 铁律 1 是"内核可 headless"的保证；铁律 3 是"插件契约可独立发布"的保证。

---

## 3. 各 Crate 职责与不变量

### 3.1 `synra-protocol`（无内部依赖）

**职责**：线协议的唯一真相源。

```
src/
├─ lib.rs
├─ version.rs        # ProtocolVersion, 兼容策略
├─ envelope.rs       # Envelope 结构与白名单校验
├─ event.rs          # 事件名常量 + 前缀规则 + 解析/构造
├─ frame.rs          # 帧格式（长度前缀 + 类型 + payload）
├─ codec/
│  ├─ json.rs        # v1 控制面编解码
│  └─ binary.rs      # v2 数据面编解码（可选）
├─ discovery.rs      # 发现报文结构（UDP）
├─ file_transfer.rs  # file.transfer.* payload 类型
├─ error.rs          # SynraError 枚举 + 稳定错误码
└─ ids.rs            # DeviceId / RequestId / TransferId / TraceId 新类型
```

**不变量**：

- 所有 ID 是**新类型**（newtype），不裸用 `String`：`DeviceId(Uuid)`、`RequestId(Uuid)`、`TransferId(Uuid)`、`TraceId(u128)`；
- 所有 payload 类型 `#[non_exhaustive]`，便于协议扩展；
- 序列化统一 `serde(rename_all = "snake_case")`；
- 版本协商逻辑在 `version.rs`，**不在**各个 payload 里。

### 3.2 `synra-capability`

**职责**：capability 字符串的解析、规范化、匹配、授权集运算。

```
src/
├─ lib.rs
├─ syntax.rs      # 解析 domain:resource:action[:qualifier]
├─ pattern.rs     # glob 匹配（* / **），精确优先
├─ grant.rs       # CapabilitySet 类型与包含关系
└─ error.rs       # CapabilityDenied 结构化错误
```

**不变量**：
- **纯函数、无 IO、无 async**；
- 解析一次，缓存为内部表示（避免每次调用重新解析字符串）；
- 匹配语义有**穷尽的属性测试**（proptest），尤其是 `*` vs `**` 的边界。

### 3.3 `synra-net`

**职责**：局域网发现与设备间可靠传输。

```
src/
├─ lib.rs
├─ discovery/
│  ├─ mod.rs        # DiscoveryService：广播、监听、过期、去重
│  ├─ packet.rs     # 报文编解码
│  └─ mdns.rs       # 可选 mDNS 后端（feature）
├─ transport/
│  ├─ mod.rs        # TransportRegistry：链路表 + 生命周期
│  ├─ link.rs       # 单链路类型状态机（Connecting→Handshaking→Ready→…）
│  ├─ framing.rs    # 读写循环、背压、最大帧
│  ├─ heartbeat.rs  # 心跳与超时检测
│  ├─ reconnect.rs  # 退避重连策略
│  └─ quic.rs       # 可选 QUIC 传输（feature）
├─ tls.rs           # 可选加密（feature）
└─ probe.rs         # TCP 单发探测（配对前）
```

**不变量**：
- 链路的"已就绪"状态与 socket 所有权绑定（类型状态机，见 [08](./07-core-kernel.md) §4）；
- 所有超时/间隔都是**常量集中定义**，可被配置覆盖，但不散落在逻辑里；
- 网络任务全部由 `synra-core` 的 supervisor 派生，`synra-net` 不自行 spawn 顶级任务。

### 3.4 `synra-plugin-api`

**职责**：插件与宿主之间的契约。**纯类型 + WIT 生成物，零实现逻辑**。

```
src/
├─ lib.rs
├─ bindings/        # wit-bindgen 生成的宿主侧绑定
├─ manifest.rs      # PluginManifest（Rust 侧结构）
├─ shared.rs        # 跨宿主的共享类型（ShareItem / IntentRequest / ...）
└─ version.rs       # API 版本与兼容矩阵
```

**不变量**：此 crate 可以被插件作者单独依赖（作为 `synra-plugin-sdk` 的底层），**不得**拉入 wasmtime / tokio / sqlite。

### 3.5 `synra-plugin-host`（本次重构的重点）

**职责**：插件的装载、沙箱执行、能力闸门、生命周期、实例池。

```
src/
├─ lib.rs
├─ engine.rs        # wasmtime Engine 配置（fuel/epoch/limits/compiler 策略）
├─ instance.rs      # 单插件实例：线性内存、store、状态
├─ pool.rs          # 实例池与空闲回收
├─ host_funcs/      # 暴露给插件的 host functions（每个都过 capability 闸门）
│  ├─ mod.rs
│  ├─ log.rs
│  ├─ event.rs
│  ├─ device.rs
│  ├─ storage.rs
│  ├─ network.rs
│  ├─ platform.rs
│  └─ misc.rs
├─ gate.rs          # 调用期闸门（capability 校验 + 审计 + 配额）
├─ lifecycle.rs     # 安装/激活/停用/卸载状态机
├─ registry.rs      # 已安装插件索引（内存 + store 持久化）
├─ verify.rs        # manifest 校验、签名校验、WIT 版本匹配
└─ error.rs
```

**不变量**：
- 插件代码永远拿不到 `&Kernel`、`Arc<...>` 或任何宿主指针——它只有 WIT 定义的函数调用；
- 每次 host function 调用**必须**先过 `gate.rs`，无例外（用宏强制，见 [12](./10-capability-and-sandbox.md) §3.3）；
- 单个插件的 panic/超时/内存超限**不得**影响其它插件或内核。

### 3.6 `synra-core`

**职责**：领域模型与编排。是唯一"知道全局"的 crate。

```
src/
├─ lib.rs
├─ kernel.rs        # Kernel 结构体 + 构造 + 生命周期
├─ identity.rs      # 本机身份（device_id / display_name / profile）
├─ discovery.rs     # 发现聚合：可用设备视图（合并发现 + 配对 + 链路状态）
├─ trust.rs         # 信任：配对记录、握手编排、解除配对
├─ link.rs          # 链路聚合视图（对 UI 暴露的"设备卡片"）
├─ intent.rs        # ★ 意图路由：分享/动作的匹配、决策、下发、回执
├─ envelope.rs      # 事件路由与订阅（对内核内部与插件）
├─ transfer.rs      # 文件传输会话编排
├─ plugin.rs        # 插件子系统门面（委托 plugin-host + plugin-ui）
├─ config.rs        # 配置聚合与变更通知
├─ supervisor.rs    # 任务监督：派生、重启策略、优雅关停
└─ events.rs        # 内核对外事件流（供 IPC 推送）
```

**不变量**：
- 每个子系统有**唯一的聚合根**，状态不外泄（[08](./07-core-kernel.md) §2）；
- 所有对外状态变更通过 `events.rs` 发布，UI 不轮询；
- `Kernel` 用 `Arc` 共享，但**内部可变状态全部走消息/锁**，禁止 `unsafe`。

### 3.7 `synra-store`

```
src/
├─ lib.rs
├─ db.rs            # SQLite 连接池 + WAL + 迁移
├─ schema/          # 迁移脚本（embedded）
├─ repo/
│  ├─ device.rs     # 已配对设备
│  ├─ plugin.rs     # 插件安装记录
│  ├─ transfer.rs   # 传输会话（用于续传）
│  └─ config.rs     # 配置 KV
├─ blob.rs          # 大对象（插件 artifact、缓存）文件存储
├─ sandbox.rs       # 插件沙箱目录管理（canonicalize + 前缀校验）
└─ secret.rs        # 密钥与敏感数据（OS keychain 或加密文件）
```

### 3.8 其余

| crate               | 职责                                                                 |
| ------------------- | -------------------------------------------------------------------- |
| `synra-plugin-ui`   | UI 插件 artifact 解包/校验/目录管理、提供 UI 加载 URL、协调 WebView 创建（不含渲染） |
| `synra-platform`    | `Platform` trait + `desktop` / `android` / `ios` / `headless` 实现      |
| `synra-observe`     | `tracing` 初始化、日志轮转、指标注册、诊断包导出                        |
| `synra-testkit`     | 假传输（内存）、真 Wasm 测试插件、协议 fixture 回放、临时 store          |

---

## 4. `apps/` 与 `tools/`（本仓库）

### 4.1 ⚠️ `synra-desktop` / `synra-mobile` **不在本仓库**

它们属于 **`synra-app`**（[README §1](./README.md)、[ADR-016](./20-adr.md)）。
本仓库只提供内核，并通过 [README §1.3](./README.md) 的两个接触点（Rust API + 生成的 TS 类型）被它消费。

> **为什么 CLI / daemon 在 core，而 desktop / mobile 不在**：
>
> | | 放在 core | 放在 app |
> |---|---|---|
> | CLI / daemon | ✅ 它们**只是内核的另一个宿主**，是"内核不依赖 UI"的验收工具；与内核同版本演进最自然；`peon-burrow` 就是 7 个 crate + `[[bin]] burrow` 在同一仓库 | ❌ 会引入跨仓库的 CLI 发布流程 |
> | desktop / mobile | ❌ 会把 Tauri / WebView / NDK 依赖拖进内核仓库的 CI 与依赖图 | ✅ 有独立的发布节奏（安装包）、独立 CI（Node + NDK + Xcode）、独立的移动端生成物 |
>
> **判据**：宿主是否**只依赖内核的公开 API**且**没有平台特定的重型依赖**？CLI/daemon 是；Tauri 应用不是。

`synra-app` 的目录结构见 [README §1](./README.md) 与 synra-app 仓库的 `design-docs/08-ipc-and-frontend.md`。

### 4.2 `apps/synra-cli`（本仓库）

```text
synra devices                       # 列出发现的设备
synra status                        # 内核与链路状态
synra send --to <id> --text "..."   # 发剪切板/文本
synra send --to <id> --file a.txt   # 发文件
synra plugin list|install|enable|disable|remove
synra plugin dev --watch            # 本地插件开发
synra logs --follow --trace <id>
synra doctor                        # 环境诊断
synra db dump                       # 导出 SQLite（调试用，[17 §2.3](./17-storage-and-config.md)）
```

**`main.rs` ≤ 60 行**（逻辑在同 crate 的 lib 里，CI 守卫检查，§7.1）。

**CLI 是架构验收工具**：如果 CLI 不能在没有 Tauri 的情况下工作，说明内核边界破了。
它同时是 **M1 的验收手段**（两个 CLI 进程互相发现、握手、配对、发消息，见 [20 M1](./19-roadmap.md)）。

### 4.3 `tools/synra-plugin-cli`（本仓库）


| 命令            | 作用                                                       |
| --------------- | ---------------------------------------------------------- |
| `new`           | 脚手架（headless / ui / hybrid 三种模板）                    |
| `build`         | headless → `.wasm`；ui → `.zip`；hybrid → 两者              |
| `verify`        | manifest / WIT 兼容 / 体积 / 无禁用导入                      |
| `sign`          | ed25519 签名 → `.sig`                                       |
| `publish`       | 打包成 registry 可消费的形式                                 |
| `dev`           | 本地加载 + 热重载到运行中的宿主                               |

---

## 5. Feature Flags 设计

编译期形态由 feature 决定，避免"为了桌面不得不带上移动的依赖"。

### 5.1 `synra-net`

| feature        | 作用                                       | 默认 |
| -------------- | ------------------------------------------ | ---- |
| `discovery-udp`| UDP 广播发现                                | ✅   |
| `discovery-mdns`| mDNS 发现（额外依赖）                      | ❌   |
| `transport-tcp`| TCP 传输                                    | ✅   |
| `transport-quic`| QUIC 传输                                  | ❌   |
| `tls`          | 传输加密                                    | ❌   |

### 5.2 `synra-plugin-host`

| feature      | 作用                                                       | 默认 |
| ------------ | ---------------------------------------------------------- | ---- |
| `wasmtime`   | 完整 wasmtime（cranelift JIT，桌面）                        | ✅（桌面） |
| `wasmtime-aot`| 预编译 AOT（移动端；iOS 不允许 JIT）                       | ❌   |
| `wasmi`      | 轻量解释器后端（体积优先，如移动端或嵌入式）                 | ❌   |
| `signing`    | ed25519 签名校验                                            | ✅   |
| `metrics`    | 插件级指标采集                                              | ✅   |

> 后端通过 `trait WasmBackend` 抽象，`wasmtime` / `wasmi` 各一个实现（[13](./12-wasm-host.md) §2）。

### 5.3 `synra-core`

| feature        | 作用                                                    | 默认 |
| -------------- | ------------------------------------------------------- | ---- |
| `plugins`      | 启用插件子系统（CLI 的基础用法可能不需要）                | ✅   |
| `plugins-ui`   | 启用 UI 插件管理（需要 Tauri/WebView 才真正可用）          | ✅   |
| `cluster`      | 启用集群/中继/选主（M6+，见 [20](./19-roadmap.md)）       | ❌   |
| `kernel-remote`| 内核以独立进程运行时的传输层（unix socket / named pipe）   | ❌   |

### 5.4 `synra-store`

| feature   | 作用                                              | 默认 |
| --------- | ------------------------------------------------- | ---- |
| `sqlite`  | SQLite 后端（bundled）                             | ✅   |
| `keyring` | OS 密钥链集成（敏感数据）                           | ✅（桌面） |

---

## 6. Feature 组合矩阵（core 仓库的 CI 必须验证）

> ⚠️ **桌面/移动的构建不在本仓库**（[§4.1]）。
> core 只保证**内核本身**在所有 feature 组合下能编、能测；**移动端目标**的验证在 core 做
> （因为它验证的是"wasmtime/rusqlite 能否交叉编译"，这是内核的问题），而**应用的打包**在 app 仓库。

| 组合                        | 用途                    | 验证命令（示意）                                                        |
| --------------------------- | ----------------------- | ----------------------------------------------------------------------- |
| `--no-default-features`     | 最小内核                | `cargo check -p synra-core --no-default-features`                       |
| 默认（含插件）               | 常规开发/发布            | `cargo check --workspace`                                               |
| **移动目标（AOT，无 JIT）**   | 验证内核可交叉编译        | `cargo check -p synra-plugin-host --target aarch64-linux-android --features wasmtime-aot` |
| **移动目标（SQLite）**        | 验证持久化可交叉编译      | `cargo check -p synra-store --target aarch64-linux-android`             |
| 轻量（wasmi）                | 体积敏感                 | `cargo check -p synra-core --no-default-features --features plugins --features synra-plugin-host/wasmi` |
| CLI only                     | 无 UI 内核验收           | `cargo check -p synra-cli`                                              |
| 无插件                       | 最小可用                 | `cargo check -p synra-core --no-default-features --features transport-tcp` |

> **core 的 CI 必须跑满这个矩阵**，否则"内核可 headless / 可移动"会悄悄退化。
> **app 仓库的 CI** 另外跑：Tauri 桌面构建、Android 构建（M5 起 iOS）、前端测试、E2E。

---

## 7. 工作流与工具链

### 7.0 工具链：不写 `rust-toolchain.toml`

> ⚠️ **刻意的决定：不写 `rust-toolchain.toml`**（[ADR-014](./20-adr.md)）。
>
> `D:/Projects` 下的 `peon-burrow` / `bmux-cli` / `harbor` / `bigseek` / `crate-plugin-kit` / `w3wright` **全都不写**它；
> 唯一写的是 `clash-verge-rev`（`channel = "1.98.1"`）。
>
> `peon-burrow` 的理由（`implementation-order.md § 2`、`testing.md § 7`）：
> > **钉死 channel 会让 CI 的 MSRV job 失效。**
>
> MSRV 只写在 `Cargo.toml` 的 `rust-version`，由 CI 里一个**专门的 MSRV job** 装上那个版本真编一遍。
> 这与"Cargo.lock 锁定"并不冲突——锁的是依赖版本，不是编译器 channel。
>
> **代价（如实记录）**：不钉 channel 意味着"我本地能编、CI 不能编"这类问题要靠 MSRV job 提前发现；
> 且 rustfmt/clippy 的行为会随工具链升级而变 → **CI 里必须固定 `dtolnay/rust-toolchain` 的版本**，不能用 `@stable`。
>
> 我们另外需要的是 **targets**（移动端交叉编译），那放在 `.cargo/config.toml` 与 CI 里，不需要 toolchain 文件。
> **已知的坑**（来自 `peon-burrow § 8` 第 9 条）：依赖的 `rust-version` 可能高于 workspace 的 MSRV
> （实例：`sysinfo` 0.39 要求 1.95，而项目声明 1.88）→ **MSRV-aware resolver 只在"版本要求允许"时才挑低版本**。
> 应对：加依赖前先 `cargo info <crate>@<ver>` 看 `rust-version`；CI 里 MSRV job **跑在 checks job 之前**。

**关于 `[workspace.package] rust-version`**：定为 **`1.88`**，与 `peon-burrow` / `bmux-cli` / `crate-plugin-kit` 一致
（它能满足 `wasmtime` 48.x、edition 2024、`libloading` 0.9 等要求；**加依赖时要复核**）。

```toml
# .cargo/config.toml —— 目标平台与构建参数
[target.aarch64-linux-android]
# NDK linker 路径由 CI/本地环境提供，不写死绝对路径
rustflags = ["-C", "link-arg=-Wl,-z,max-page-size=16384"]   # Android 16 KB page

[target.aarch64-apple-ios]
rustflags = ["-C", "link-arg=-Wl,-no_compact_unwind"]

[alias]
# 常用组合，避免每次敲长命令
check-all = "clippy --workspace --all-targets --locked -- -D warnings"
```

```toml
# Cargo.toml（workspace 根）—— 关键片段
[workspace]
members = ["crates/*", "apps/*", "tools/*"]
resolver = "3"                       # peon-burrow / clash-verge 均用 3（edition 2024）

[workspace.package]
version      = "0.1.0"
edition      = "2024"
license      = "MIT"                 # 与 D:/Projects 生态一致（全部 MIT）
rust-version = "1.88"                # ★ MSRV 的唯一声明处；CI 有专门的 msrv job
repository   = "https://github.com/imba97/synra"

[workspace.dependencies]
# 内部 crate 同时写 path 与 version：发布时 path 被剥离，只剩 version
synra-protocol    = { path = "crates/synra-protocol",    version = "0.1.0" }
# ...

[profile.release]
# ★ panic = "unwind"：插件 panic 必须能被 catch_unwind 捕获（[12 §7.2](./12-wasm-host.md)）
# 这是 peon-burrow 与 clash-verge-rev 共同的选择，理由完全一致
panic         = "unwind"
lto           = "thin"
codegen-units = 1
opt-level     = 3
strip         = "symbols"
debug         = 1                    # 保留行号，崩溃报告才有用

[profile.release-fast]
inherits      = "release"
lto           = false
codegen-units = 64
opt-level     = 0
debug         = true
strip         = "none"

[workspace.lints.rust]
unsafe_code         = "forbid"       # ★ 唯一豁免是 synra-plugin-host 的 wasmtime 边界（用 #[allow] 局部打开）
missing_docs        = "warn"
unreachable_pub     = "warn"

[workspace.lints.clippy]
# 沿用 clash-verge-rev 的清单（已在其生产项目中验证），删掉与 wasm/异步不适配的几条
async_yields_async            = "deny"
await_holding_lock            = "deny"
cognitive_complexity          = "deny"
correctness                   = { level = "deny", priority = -1 }
suspicious                    = { level = "deny", priority = -1 }
dbg_macro                     = "deny"
todo                          = "deny"
unimplemented                 = "deny"
panic                         = "deny"    # ★ 内核不许 panic（trap 要返回结构化错误）
unwrap_used                   = "deny"    # ★ 内核不许 unwrap
expect_used                   = "deny"    # ★ 内核不许 expect
indexing_slicing              = "deny"    # ★ 帧解析里 t[i] 是越界 panic 的常见来源
arithmetic_side_effects       = "warn"    # 帧长度计算需要关注溢出
future_not_send               = "deny"
large_futures                 = "deny"
missing_const_for_fn          = "deny"
needless_pass_by_value        = "deny"
redundant_clone               = "deny"
significant_drop_tightening   = "deny"
use_self                      = "deny"
wildcard_imports              = "deny"
# ⚠️ 以下在 apps/*（Tauri 层）放宽：那里允许 unwrap/expect（启动期快速失败）
```

> **两档 lint**：`crates/*`（内核）用上面的**严格档**；`apps/*` 与 `tools/*` 放宽
> （`unwrap_used = "allow"`、`expect_used = "allow"`），因为它们本来就是"启动即失败"的边界层。
> 做法：在 `apps/*/Cargo.toml` 里 `[lints] workspace = false` + 自己的 `[lints]`。

### 7.1 CI 上的 5 条结构守卫（防漂移）

沿用 `peon-burrow` 的做法，把架构不变量做成**可执行的检查**而非文档约定：

```bash
# 守卫 1：内核不依赖 GUI（[05 §1.1](./05-system-architecture.md) 铁律 1）
cargo tree -p synra-core -p synra-net -p synra-protocol -p synra-plugin-host \
  | grep -E 'tauri|wry|tao' && exit 1 || exit 0

# 守卫 2：协议层不许有 IO 依赖（[06 §2.1](./06-workspace-layout.md)）
cargo tree -p synra-protocol | grep -E 'tokio|wasmtime|rusqlite|hyper' && exit 1 || exit 0

# 守卫 3：bin 只有薄壳（main.rs ≤ 60 行，逻辑在同 crate 的 lib）
test "$(wc -l < apps/synra-cli/src/main.rs)" -le 60
test "$(wc -l < apps/synra-daemon/src/main.rs)" -le 60
test "$(wc -l < tools/synra-plugin-cli/src/main.rs)" -le 60

# 守卫 4：paths 注入（不许直接调 directories，只能在 Paths::discover 里）
test "$(grep -rl 'directories::' crates/ --include='*.rs' | grep -v 'crates/synra-store/src/paths.rs' | wc -l)" -eq 0

# 守卫 5：host function 全部过闸门（[10 §3.1](./10-capability-and-sandbox.md)）
test "$(grep -rc 'ungated_host_fn!' crates/synra-plugin-host/src/host_funcs/ | grep -v 'env.rs' | grep -v ':0$' | wc -l)" -eq 0
```

> `peon-burrow` 的 `implementation-order.md § 0` 把这类守卫叫「铁律」，每条都配一条可执行的检查。
> 这个做法值得照抄：**架构不变量一旦只写在文档里，三个月后必然被违反。**

| 用途         | 选型                                                                 |
| ------------ | -------------------------------------------------------------------- |
| 测试         | `cargo nextest`（并行、更好的输出）；`cargo test --doc` 单独跑          |
| 属性测试     | `proptest`（capability 匹配、帧编解码、状态机）                        |
| 基准         | `criterion`（本地）+ `iai-callgrind`（CI，指令数更稳定）                |
| 覆盖率       | `cargo llvm-cov`                                                      |
| Lint         | `clippy`（`-D warnings`）+ `rustfmt`；自定义 lint 用 `dylint`（可选）   |
| 依赖审计     | `cargo-deny`（许可证、重复版本、安全公告、banned crates）               |
| 未使用依赖   | `cargo-machete`                                                       |
| 前端         | 保留 Vite + Vitest（`ui/` 目录），但**不再是业务宿主**                  |
| 端到端       | Tauri 的 WebDriver（`tauri-driver`）+ 自研 headless 内核测试            |
| **版本/tag** | ★ **`cargo-bumpp`**（本地工具，不进依赖）—— 生态统一做法                 |
| **发布编排** | ★ **`harbor`**（`cargo harbor`）—— 自动拓扑排序、可续跑、CI 预检         |
| 移动端构建   | `cargo tauri android/ios` + `just` 封装                               |

#### 7.1.1 版本与发布：用生态里已有的工具

**不自己写发布脚本**。`D:/Projects` 已有两件成熟的工具：

```bash
# 改版本 + 提交 + 打 tag + 推 tag（生态统一做法：Conventional Commits 决定级别）
cargo bumpp patch|minor|major|conventional

# 发布到 crates.io：harbor 自己算依赖顺序，不需要手工维护列表
cargo harbor plan                       # 只算计划，不联网
cargo harbor check v0.1.0 --plan        # tag 与 manifest 版本一致性 + token 预检
cargo harbor publish                    # 按顺序上传
cargo harbor publish --dry-run
```

`harbor` 的三个特性正好解决我们的痛点：

| `harbor` 特性        | 解决的问题                                                             |
| -------------------- | ---------------------------------------------------------------------- |
| 自动拓扑排序          | [06 §2](./06-workspace-layout.md) 有 11 个 crate，手工维护发布顺序必然出错 |
| 可续跑               | 中断的发布会跳过"已在 registry 上"的版本，重跑即可接着做完                 |
| 只重试该重试的        | 索引没跟上就等；token 不对就停（而不是无脑重试）                            |

CI 里：

```yaml
- run: cargo install harbor --locked
- name: Publish
  run: |
    if [ "$GITHUB_REF_TYPE" = "tag" ]; then cargo harbor check --plan "$GITHUB_REF_NAME"; fi
    cargo harbor publish
  env:
    CARGO_REGISTRY_TOKEN: ${{ secrets.CARGO_REGISTRY_TOKEN }}
```

> **不做 CHANGELOG 文件**——这是该生态的一贯做法（`peon-burrow § 7` 明确写了）：
> 提交信息用 Conventional Commits，Release notes 由 tag 与 PR 描述构成。
> 因此**不引入 changelog 生成器**。
>
> **不用 `cargo-dist`**：`peon-burrow § 7` 给了三条理由（生态风格不一致、管不了跨仓库取二进制、未关闭的 Windows 临时目录清理 bug）；
> 我们多一条：我们的产物里有**插件 artifact 与签名**，通用工具吐不出我们的格式。

#### 7.1.2 分类决策：几个 crate 发、几个不发

沿用 `peon-burrow` 的 `STABILITY.md` 做法，把 crate 分三层并在**文档里明确承诺等级**
（完整义务与检查见 [26 §2](./26-core-stability-and-perf.md)）：

| 层          | crate                                                                              | 承诺                          | 发布 | 公开 API 快照 |
| ----------- | ---------------------------------------------------------------------------------- | ----------------------------- | ---- | ------------- |
| **稳定层**   | `synra-protocol`、`synra-capability`、`synra-plugin-api`                              | 遵守 SemVer；破坏性变更需 major + ADR + 迁移说明 | ✅ | ★ **必须**（[26 §6.1](./26-core-stability-and-perf.md)） |
| **半稳定层** | `synra-net`、`synra-store`、`synra-core`、`synra-plugin-host`                          | 遵守 SemVer；允许 minor 里**扩展 trait**（带默认方法） | ✅ | ✅ 建议 |
| **内部层**   | `synra-observe`、`synra-platform`、`synra-plugin-ui`、`synra-testkit`                  | 无承诺                        | 部分 | ❌ |

**"半稳定"这个等级的由来**：`synra-core` 是编排层，业务规则一定会长。
但它要区分两件事 —— **接口**（`Kernel` 的公开方法）要稳，**内部结构**（聚合根怎么组合）可以动。
不写清这个区分，要么过度冻结（无法演进），要么随意改动（下游被反复打断）。

**稳定层的额外义务**（`STABILITY.md` 里要写明，[26 §2.2](./26-core-stability-and-perf.md)）：

- 公开签名里**不出现具体容器类型**（`HashMap`/`Vec`/`HashSet`）—— 否则换实现就是破坏性变更；
- 公开 struct 字段**全私有** + 构造函数 + 只读访问器；
- 面向第三方的 enum **必须** `#[non_exhaustive]`，内核内部的 enum **必须不要**（§[26 §2.3](./26-core-stability-and-perf.md)）；
- 错误类型对外只暴露**稳定错误码**，不暴露 `std::io::Error` / `serde_json::Error` 等第三方类型。


**为什么要发布**：`synra-protocol` 与 `synra-capability` 有独立价值（别人想写一个兼容的客户端/服务端，或复用 capability 匹配），
`synra-plugin-api` 是插件作者要依赖的契约 crate。其余发布是为了 `cargo install` 与依赖解析方便。

**发布顺序**（由 `harbor` 自动推导，此处仅记录预期）：

```
synra-protocol
  → synra-capability
      → synra-plugin-api
          → synra-plugin-host / synra-plugin-ui
synra-store
synra-observe
  → synra-net
      → synra-core
          → synra-platform
              → （apps，不发）
```

> ⚠️ **`synra-plugin-host` 里有唯一的 `unsafe` 岛**（wasmtime 边界 + `catch_unwind`）。
> 这与 `bmux` 的 `pmpx-plugin-abi` 的做法一致：**把 unsafe 收在一个 crate 里，其余 workspace 全 `forbid(unsafe_code)`**。
> 落点：`crates/synra-plugin-host/src/ffi.rs` 集中所有 `unsafe`，文件头写明为什么需要它，
> 且该文件**必须**有独立的测试覆盖（越界、null、对齐）。

### 7.2 推荐的 `justfile` 任务

```make
# 全量检查（每次提交前，对齐 peon-burrow 的 CI `checks` job）
check: fmt-check lint test test-doc doc

fmt:        ; cargo fmt --all
fmt-check:  ; cargo fmt --all -- --check
lint:       ; cargo clippy --workspace --all-targets --locked -- -D warnings
test:       ; cargo nextest run --workspace --locked
test-doc:   ; cargo test --workspace --doc
doc:        ; cargo doc --no-deps --workspace
bench:      ; cargo bench -p synra-protocol -p synra-capability -p synra-net

# 结构守卫（§7.1 的 5 条）
guards:     ; ./xtask/guards.sh

# feature 矩阵（与 §6 对应）+ MSRV
matrix:     ; ./xtask/check-matrix.sh
msrv:       ; ./xtask/check-msrv.sh          # 读 Cargo.toml 的 rust-version 真编一遍

# mobile-check: ; cargo check -p synra-net --target aarch64-linux-android
                cargo check -p synra-net --target aarch64-apple-ios
sqlite-check: ; cargo check -p synra-store --target aarch64-linux-android
                cargo check -p synra-store --target aarch64-apple-ios
wasm-check:   ; cargo check -p synra-plugin-host --target aarch64-linux-android \
                  --features synra-plugin-host/wasmtime-aot

# 协议一致性（[21 §3](./21-testing-strategy.md)）
wire:       ; cargo nextest run -p synra-protocol --test wire_fixtures
conformance: ; cargo nextest run -p synra-protocol --test conformance   # fixture + 双内核对称性（[21 §3](./21-testing-strategy.md)）

# 插件工具
plugin-new:  ; cargo run -p synra-plugin-cli -- new $(name)
plugin-dev:  ; cargo run -p synra-plugin-cli -- dev --watch

# 发布（[ADR-015](./20-adr.md)）
bump:        ; cargo bumpp conventional
publish-plan:; cargo harbor plan
publish:     ; cargo harbor publish
```

> **`justfile` 只服务本仓库（core）**。前端、桌面、移动的构建命令在 `synra-app` 仓库的
> `package.json` scripts（+ 必要时一个薄 `justfile`）里，见该仓库的 `design-docs/08-ipc-and-frontend.md`。
> `mobile-check` / `sqlite-check` / `wasm-check` 保留在 core，因为它们验证的是**内核能否交叉编译**，
> 而不是"应用能否打包"。

---


| 旧路径                                        | 新位置                                                    |
| --------------------------------------------- | --------------------------------------------------------- |
| `packages/protocol`                            | `crates/synra-protocol`                                   |
| `packages/envelope`                            | `crates/synra-protocol`（envelope 模块）                   |
| `packages/hooks/src/runtime/**`                 | `crates/synra-core`（聚合根）+ `crates/synra-net`（网络）  |
| `packages/hooks/src/file-transfer`              | `crates/synra-core/src/transfer.rs`                        |
| `packages/hooks/src/envelope/*`                 | `crates/synra-core/src/envelope.rs` + `synra-protocol`      |
| `packages/transport-core` / `transport-events`  | `crates/synra-net`                                        |
| `packages/plugin-system`                        | `crates/synra-plugin-api`（manifest）+ `synra-store`（store）|
| `packages/plugin-sdk`                           | `crates/synra-plugin-sdk`（作者侧）+ `wit/` + `tools/wit-codegen` |
| `apps/frontend/src/plugins/host/**`              | `crates/synra-plugin-host`（逻辑）+ `synra-plugin-ui`（artifact）|
| `packages/capacitor-*`                          | **删除**，能力并入 `crates/synra-net` + `synra-platform`    |
| `packages/bridge-schema` / `vite-plugin-synra-electron` | **删除**（不再需要跨语言桥 schema）                |
| `scripts/**`                                    | `xtask/**` + `justfile`                                    |
| **↓ 以下进 `synra-app` 仓库（不在本仓库）↓** |                                                          |
| `apps/frontend/src/pages|components|stores/**`  | `src/`（保留，改数据源）                                     |
| `apps/frontend/src/**`（其余）                   | `src/**`                                                   |
| `apps/electron/**`                              | `src-tauri/**`                                             |
| `apps/mobile/**`（Capacitor + 原生插件）          | `src-tauri/**` + `gen/android` + `gen/apple`                |

完整的逐条映射见 [19](./24-migration-map.md)（其中 §3.1 / §3.2 归属 app 仓库）。

---

## 9. 与 `D:/Projects` 生态的对齐结论

调研了目录下 13 个 Rust 项目后，本仓库**主动采用**以下既有约定（而不是自己发明）：

### 9.1 采用清单

| # | 约定                                              | 来源                                | 落点                          |
| - | ------------------------------------------------- | ----------------------------------- | ----------------------------- |
| 1 | **不写 `rust-toolchain.toml`**，MSRV 在 `Cargo.toml` | `peon-burrow`（6/7 项目如此）        | [ADR-014](./20-adr.md)         |
| 2 | **`panic = "unwind"`** 在 release（插件 panic 要可捕获） | `peon-burrow` + `clash-verge-rev`（两家都写了同样理由） | [ADR-013](./20-adr.md)、[12 §7.2](./12-wasm-host.md) |
| 3 | **`[workspace.lints.clippy]` 严格集**               | `clash-verge-rev`（生产验证过）       | §7 上方                       |
| 4 | **`unsafe` 收在一个岛**，其余 `forbid(unsafe_code)`  | `bmux/pmpx-plugin-abi`               | §7.1.2、[ADR-013](./20-adr.md) |
| 5 | **ABI/协议契约独立成 crate 且不依赖 IO**             | `peon-burrow-protocol`、`pmpx-plugin-abi` | [06 §2.1](./06-workspace-layout.md) |
| 6 | **`harbor` 发布**（拓扑排序 + 可续跑）               | `peon-burrow`、`harbor` 自身          | §7.1.1、[ADR-015](./20-adr.md) |
| 7 | **`cargo-bumpp` 版本 + 无 CHANGELOG**               | 全部项目                             | §7.1.1、[ADR-015](./20-adr.md) |
| 8 | **`[workspace.package]` 统一元数据 + `version.workspace = true`** | `peon-burrow`、`bmux-cli` | §7 上方 |
| 9 | **内部依赖同时写 `path` 与 `version`**               | `bmux-cli`（注释说明：发布时 path 被剥离） | §7 上方                  |
| 10 | **`publish = false` 给 testkit**                    | `peon-burrow`                       | §7.1.2                        |
| 11 | **CI：`-D warnings` + 独立 MSRV job + 测试不依赖外网** | `peon-burrow`                       | [21 §8](./21-testing-strategy.md) |
| 12 | **结构守卫做成可执行的 CI 检查**（不只写在文档里）    | `peon-burrow`                       | §7.1                          |
| 13 | **两级文档风格**：项目层 + `design/` + `decisions/`（ADR） | `peon-burrow`                     | 本目录结构                    |
| 14 | **"现状 / 目标" 必须分开写**                        | `peon-burrow` 明确踩过坑             | [README §6](./README.md)       |
| 15 | **归档命名 `{crate}-{target}.tar.gz`，不带版本号**（为了 `cargo-binstall`） | `peon-burrow` | 发布流程        |
| 16 | **跨语言测试向量文件锁定语义**                       | `bigseek`（`testdata/*-vectors.json`） | [21 §3](./21-testing-strategy.md) |
| 17 | **MIT 许可证**                                      | 全部项目                             | —                             |

### 9.2 我们**偏离**生态的地方（及理由）

| # | 生态做法                                  | 我们的做法                                  | 为什么                                                        |
| - | ----------------------------------------- | ------------------------------------------- | ------------------------------------------------------------- |
| 1 | 插件系统用 cdylib + `dlopen`（`crate-plugin-kit` / `pmpx`） | Wasm 组件模型                      | 插件来自不可信第三方 + 跨设备同步 + 移动端 iOS 不允许 dlopen → [ADR-013](./20-adr.md) |
| 2 | **core 与 GUI 拆两个仓库**（`peon-burrow` / `peon-hall`） | **同样拆**（`synra-core` / `synra-app`） | ★ **已从最初的"单仓库"方案改为对齐生态** → [ADR-016](./20-adr.md)、[README §1](./README.md) |
| 3 | 有的项目用 `edition 2021`（`crate-plugin-kit` / `bmux`） | **edition 2024**                | 我们是新项目；`peon-burrow` 已是 2024（`resolver = "3"`）        |
| 4 | 有的项目发到 crates.io 是核心目标            | 发布是**副产品**（主目标是应用）              | 但 `synra-protocol` / `synra-capability` / `synra-plugin-api` 确实有独立价值 → 仍发（§7.1.2） |
| 5 | `peon-burrow` 的 CLI 与 core **在同仓库** | 我们**一样**（`apps/synra-cli` + `synra-daemon` 在 core） | CLI/daemon 只是内核的薄宿主，且是"内核不依赖 UI"的验收工具 → §4.1 |
| 6 | `peon-burrow` 首发**不签名**（`adr-0011` 降级路径） | 我们**首发就强制签名** | 我们的插件来自不可信第三方；它的是自家扩展 → [ADR-008](./20-adr.md) |

### 9.3 生态的**空白**（我们需要自己趟）

| 空白                                             | 说明                                                                                     | 缓解 |
| ------------------------------------------------ | ---------------------------------------------------------------------------------------- | ---- |
| **没有项目用过 `wasmtime` / 组件模型**             | `bigseek` 的 wasm 是 `wasm-bindgen` 编译到浏览器，不是宿主嵌入运行时。**没有先例可抄**       | 做成 M0 spike（[19 §0.1](./19-roadmap.md)），**先验证再投入** → [23 R-01b](./22-risks.md) |
| **没有项目做过 Tauri 移动端**                     | `clash-verge-rev` 与 `peon-hall` 都是纯桌面                                                | M0 就做 Android spike（[19 §0.1](./19-roadmap.md) 1–4）；**iOS 因无 macOS 环境推迟** |
| **没有项目做过 UDP 广播 / 局域网发现**             | 全部是 CLI/服务/GUI，没有组网代码                                                          | 协议语义从旧 TS 端继承（[03](./04-wire-protocol.md)），实现自己趟 |
| **没有项目做过插件签名 + 信任链**                  | `peon-burrow` 的 `adr-0011` 在讨论 zipsign，但首发降级为"不签名"                              | 参考它的"签名上线路径"思路，但我们要**首发就强制** |
| **没有项目做过"内核 + 多宿主"的双仓库类型共享**      | `peon-burrow` 只共享 IPC 类型；我们还要共享**插件契约 + 生成的 TS 类型**                      | 用 `ts-rs` 生成 + npm 包（[README §1.3](./README.md)）；M0 spike 7 顺带验证 |
