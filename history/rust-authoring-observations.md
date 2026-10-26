# Rust 作者链路修复前的构建与纯度观察

归档日期：2026-10-26；原文中的目标、阻塞与测试状态不作为当前结论。
更正与实验证据见 [本次开发记录](./2026-10-26-rust-authoring.md)。

## 原构建流程

### 3.2 `build` 做什么

```rust
// tools/synra-plugin-cli/src/cmd/build.rs

pub async fn build(opts: BuildOptions) -> Result<BuildReport> {
    let manifest = Manifest::load(&opts.cwd)?;

    let mut report = BuildReport::default();

    // ── headless：cargo build --target wasm32-wasip2 ──
    if let Some(headless) = &manifest.runtime.headless {
        let status = Command::new("cargo")
            .args(["build", "--target", "wasm32-wasip2", "--release"])
            .env("RUSTFLAGS", "-C link-arg=-zstack-size=1048576")
            .status()?;
        // 产物：target/wasm32-wasip2/release/<crate>.wasm
        // → 复制到 dist/plugin.wasm
        // → 可选：wasm-opt / strip
        // → ★ 生成 component（wasm-tools component new）
        report.artifacts.push(headless_artifact);
    }

    // ── UI：vite build → zip ──
    if let Some(ui) = &manifest.runtime.ui {
        // 1. 在 ui/ 目录跑 vite build（输出到 dist/ui/）
        // 2. ★ 校验产物不含外部网络引用（connect-src 'none'）
        // 3. 校验不含内联 eval（CSP 兼容）
        // 4. 打包为 dist/ui.zip
        report.artifacts.push(ui_artifact);
    }

    // ── manifest 内联（规范化后）──
    write_normalized_manifest(&opts.cwd, &manifest)?;

    // ── 自检 ──
    let verify = verify(&opts.cwd, VerifyMode::Normal).await?;
    report.verify = verify;

    Ok(report)
}
```


## 原构建工具讨论

### 10-8 ★ 构建链的确切工具（wasm32-wasip2 的产出形态）

**问题**：Rust 的 `wasm32-wasip2` target 在不同版本下产出不同：
- 较新版本 `cargo build --target wasm32-wasip2` 直接产出**组件**；
- 较早/某些配置下产出**核心模块**，需要 `wasm-tools component new` + adapter。

**这是一个 spike 项**（[19 §0.1](../19-roadmap.md) spike 2/3 顺带验证），但**现在就要定文档口径**：

**倾向**：
- 文档写"`cargo build --target wasm32-wasip2 --release` 产出 Wasm 组件"；
- `synra-plugin build` **自动探测**：如果产出是核心模块，自动跑 `wasm-tools component new`；
- **锁定 `wasm-tools` / `wit-bindgen` 的版本**（写进 `synra-plugin-cli` 的 `Cargo.toml`，并在 `verify` 时检查插件是用哪个版本构建的）。

**需要确认**：是否允许插件作者用自己的 `wasm-tools` 版本（可能导致组件不兼容）。

---


## 原纯度调查

   > ★ **而「必须无副作用」这件事，一直没有人守。** `Work::allows_side_effects` 存在，
   > 它的注释写着"the host is what makes that true, by running it with a context whose action-shaped calls
   > are refused"——而 **那个 context 不存在**：这个函数从来没被调用过。
   > 于是 `can-handle` 可以做它能力允许的任何事——发消息、开浏览器、写存储、发请求——
   > 而它会在**每一次匹配**时这么做，对用户后来取消的每一个分享都做。
   > 现在：`Work::allows_side_effects()` → `StoreData::side_effects` → `HostCalls::pure()` →
   > `ACTION_SHAPED` 一组函数拒绝（`sideEffectNotAllowed`，**不是** `capabilityDenied`：能力是持有的）。
   > ★ **一处仍然没有测试覆盖**：`linker::run` 里读 `state.side_effects` 的那两行。
   > 把它删掉，现有测试全部依然绿（第二个对照实验证实了这一点），
   > 因为走通那条路需要一个真编译出来的 guest——受阻于 `wasm32-wasip2`。
   > 它被明写在 `host_calls_tests.rs` 那条测试的注释里，而不是留给读者去猜。
