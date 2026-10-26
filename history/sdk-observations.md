# SDK 早期实现观察

原文来自 15 插件 SDK 的实现清单与第 180 轮记录；保留原编号、判断和调查过程。
其中未实现项、行号和安全阻塞推断仅表示当时观察，当前规范见 [15](../15-plugin-sdk.md)，
后续修正见 [Rust 作者链路](./2026-10-26-rust-authoring.md) 与 [SDK 资源](./2026-10-26-sdk-resources.md)。

## 8. 实现清单

> 以下清单及第 180 轮记录是早期评估，保留用于追溯；其中模板、命令和测试数量不能作为当前状态。
> 当前契约以正文为准，本轮设计修订的待开发项见 [19 §0.2](../19-roadmap.md)。

> ★ 这一节在实现之前写就，长期整节未勾选。逐条核对源码后：**合同、生成器、类型库是真货**；
> 薄封装、属性宏、打包、发布、模板都没有。**"verify 通过"与"部署会被拒"之间目前有一道缝**（见 755）。

- [x] `wit/synra-plugin.wit` 完整定义（host + lifecycle + share + actions）—— **A**：
      `wit/synra-plugin.wit:88` host，以及 lifecycle、share、actions，`:352` `world plugin`。
- [x] `tools/wit-codegen`：生成 Rust 宿主绑定与 SDK 绑定 —— **A**：
      `tools/wit-codegen/`（`parse/{contract,gate,signature,table}.rs`、`emit.rs`）；
      `crates/synra-plugin-host/build.rs` 每次构建重新生成 `capability_table.rs` 与 `host_trait.rs`。
      ★ SDK 侧**另走一条路**：`synra-plugin-sdk/src/lib.rs:67` 用 `wit_bindgen::generate!`，
      所以"生成器负责 SDK 绑定"只说对了一半。
- [ ] `synra-plugin-sdk`：9 个 namespace 薄封装 + `Error` —— **未做**。
      `synra-plugin-sdk/src/lib.rs` **只有** `wit_bindgen` 生成的绑定 + `synra-plugin-api` 三个类型的 re-export；
      没有 9 个 namespace 薄封装，也没有 `Error` 类型。
      ★ 文档点名的 **`fs` namespace 在合同里根本不存在** —— 合同没有文件系统接口；
      真实分层是"裸绑定 + `synra-plugin-api` 边界类型"。
      ★ **第 126 轮核实：这一条是准确的，而且这个 crate 比它读起来更完整。** 实测
      `cargo test -p synra-plugin-sdk` 是 **6 + 1 全过**：6 条宿主目标单测（每个导入名与合同一致、
      名字互不相同、合同版本、边界类型就是 `synra-plugin-api` 的而不是副本、接口与世界名、
      以及"合同声明的每个宿主函数都有绑定"）加 1 条 guest 世界的集成测试。
      绑定由 `wit_bindgen::generate!` 在**编译期**从 `wit/synra-plugin.wit` 生成，所以"改了合同忘改 SDK"这一类
      漂移在结构上不可能；上面那句"没有薄封装"说的是**便利层**，不是绑定层。
      ★ **一条窄的注意点**：那条"每个函数都有绑定"的测试读的是**签入的**
      `crates/synra-plugin-host/src/generated/capability_table.rs`，而它由**宿主**的 `build.rs` 重新生成。
      所以单跑 `-p synra-plugin-sdk` 时，比较的一方可能是**上一次宿主构建**留下的产物；
      在工作区整体构建（CI 的做法）里宿主先生成，这条守卫才是在比较当前合同。
      ★ **而这个注意点第 127 轮修掉了**：那条测试现在读的是**合同本身** ——
      把 `wit-codegen`（宿主 `build.rs` 用的同一个解析器）作为 **dev-dependency**，直接 `Contract::parse`
      读 `wit/synra-plugin.wit`。它不是 `synra-plugin-host`，所以不是原注释担心的那个循环；
      作为 dev-dependency，它也不会进插件下载的内容里。
      ★ **而证据正是第 126 轮做不到的那件事**：往合同里加一个函数，**只跑
      `cargo test -p synra-plugin-sdk`**（不重建宿主），现在会失败 ——
      `assertion left == right failed: the contract's own count: 36 / left: 36 / right: 35`。
      第 126 轮同样的探针是**通过**的，那正是这条守卫读错了东西的证据。
      ★ 以及第 126 轮想给它做"能不能失败"的探针时，先撞上的是**另一条更早的守卫**：
      往合同里加一个没有 `@ungated` / `@capability` 的函数，宿主 `build.rs` 直接报
      `'probe-added' has neither @ungated nor @capability…` 并让构建失败 ——
      也就是说"未声明的能力"根本走不到绑定这一步。
- [ ] `#[plugin]` 属性宏（含编译期检查）—— **未做**：全仓 `Cargo.toml` grep `proc-macro` **0 命中**，
      工作区没有任何 proc-macro crate。
      ★ 而且这里有一个**已知的硬冲突**，不是"还没写"那么简单：
      `synra-plugin-sdk/tests/guest/world_fixture.rs:17-20` 自己写明 `export!` 需要 `unsafe`，
      而工作区 `unsafe_code = "forbid"` **不能**用局部 `allow` 放开。
      要落地要么把宏放进一个独立 crate（像 `synra-testkit-alloc` 那样把例外集中到一处），
      要么改变对这个 fixture 的处理方式 —— 这是一次设计选择，先得做。
- [ ] `testing` feature：`MockHost` + audit 断言 —— **未做**：
      `synra-plugin-sdk/Cargo.toml` 连 `[features]` 段都没有；无 `MockHost`。
      `host_calls_tests.rs:49` 的 `Stub` 是 crate 内 `#[cfg(test)]`，**插件作者拿不到**。
- [ ] `tools/synra-plugin-cli`：new / build / verify / sign / pack / publish / dev / test ——
      **4/9**：`args.rs:35-44` 只有 `New`/`Build`/`Verify`/`Sign`，缺 `pack`/`publish`/`dev`/`test`。
- [ ] `verify` 的完整检查项（§2.4 + §3.1）—— **一半，而且有一处静默失效**。
      CLI 有：manifest 校验（`verify/check.rs:75`，`ValidationLevel::Schema`）+ 体积/文件数 + import 名单。
      ★ `verify/target.rs:38` 的 `wasm_imports` **恒为空向量**（诚实降级为 warn），
      所以 **import 检查在生产中从不运行**；
      ★ 更要紧的是：**CLI 里没有"import ⊆ 已声明的 capability"这一 §3.1 的核心检查** ——
      只有宿主侧 `plugin-host/src/verify/mod.rs:64` 的 `Stage::ImportsWithinCapabilities` 有。
      后果是作者本地 `verify` 得到的"通过"，并不包含部署时会被拒的那一条 ——
      一个本地全绿的插件可能在装载期被拒，而 CLI 没有告诉他为什么。
- [ ] Rust headless 模板 + Rust hybrid 模板；纯 UI 模板不属于目标支持范围。以下为早期模板观察，并非当前实现状态：
      `template.rs:36-43` 的 `TemplateKind{Headless,Ui,Hybrid}` 三套确实存在并被测试（`validate_manifest`）。
      ★ **headless 模板不是组件**：`template.rs:386-403` 产出的是 `cdylib` +
      `#[no_mangle] extern "C" fn synra_plugin_activate()`，与 WIT 要求的
      `export!("synra:plugin/lifecycle@1.0.0")` **不匹配** —— 生成的插件装上去，
      `compile` 会因缺导出被拒。模板能过它自己的测试，因为它测的是 manifest。
      ★ UI 模板是单个裸 `index.html`，无 TS/Vue/`package.json`/`tsconfig.json`。
- [x] `@synra/client` 类型库（`.d.ts` only）—— **A**：
      `synra-js/packages/client/package.json:10-14` 的 `exports` **只有**
      `"types": "./dist/index.d.ts"`，无 `main`/`module`；`src/global.ts:27` 声明全局。
- [ ] 打包格式 `.synra` + registry 发布流程 —— **未做**：无 `.synra` 格式、无 `publish` 命令；
      `sign` 只写 `signature.json`（`run.rs:248`）。
- [ ] 迁移文档 + 一个真实的迁移示例（把旧 chat 插件迁过来）—— **未做，且前提不成立**：
      见 §7 的说明，仓库里没有旧插件可以迁。

## 第 180 轮：这 16 条在 51 轮之后仍然准确

★ 第 127 轮判断这 16 条「准确且被挡住」。这一轮逐条核实 —— **16 条全部仍然成立**，
包括每一处行号引用（`tools/synra-plugin-cli` 确在，`args.rs` 确实只有 4 个子命令，
全仓 `proc-macro` 仍然 **0 命中**）。

★ **而其中两条值得单独点出，因为它们是同一件事的两个实例 —— 一个检查能通过而事实不成立：**

1. **`verify` 里有一条静默失效的检查。** `verify/target.rs` 的 `wasm_imports` **恒为空向量**
   （诚实降级为 warn），所以 import 检查在生产中**从不运行**；而更要紧的是
   **CLI 里没有「import ⊆ 已声明的 capability」这一 §3.1 的核心检查** —— 只有宿主侧有。
   后果是作者本地 `verify` 全绿，而插件在装载期被拒，**而 CLI 没有告诉他为什么**。

2. **headless 模板产出的不是一个组件。** 它生成 `cdylib` + `#[no_mangle] extern "C" fn synra_plugin_activate()`，
   而 WIT 要的是 `export!("synra:plugin/lifecycle@1.0.0")` —— 生成的插件装上去会因缺导出被拒。
   **而模板能过它自己的测试，因为它测的是 manifest。** 那正是「一个检查瞄准了错的分支」的形状。

★ **而它们都修不了，原因是同一个硬冲突**：`export!` 需要 `unsafe`，
而工作区 `unsafe_code = "forbid"` **不允许**局部 `allow`。
要落地要么把宏放进一个独立 crate（像 `synra-testkit-alloc` 那样把例外集中到一处），要么改变那个 fixture 的处理方式。
**那是一次设计选择，不是一行改动** —— 与 `AckPolicy`、`synra-net`、`ReceiverCheck` 属于同一类：
**这个仓库剩下的工作，几乎全部是决定，而不是实现。**
