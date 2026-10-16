# 28 · App UI 规范（宿主与插件界面）

> 📍 **归属：`synra-app` + `synra-js`（`packages/theme`）两半**——宿主界面与插件界面共享同一套 token 与规则，
> 但"能被强制"的程度不同（§12、§13.3）。这份文档因此不是一篇整篇进单个仓库的文档。
>
> 状态：**规则 ✅ 已定**（§2–§12）；**设计细节 🟡 倾向**（§1.5 里那几个具体值，实现时定稿）；
> **未定 🔴** 集中列在 §16.2。
>
> 前置：[08 §6](./08-ipc-and-frontend.md)（前端构建与 WebView 矩阵）、[25](./25-js-monorepo.md)（`synra-js` 的包与守卫）、
> [13](./13-plugin-authoring.md)（插件写法）。相关：[27 §8](./27-website.md)（官网的样式立场）、[11](./11-design-principles.md)。

---

## 0. 一句话结论

**样式只有一份真相源，它在 `@synra/theme` 里；宿主与插件都从那里取 token，谁都不许再写第二个字面颜色；
断点、字体、动效、文案、无障碍下限都按本文的编号规则执行，每条规则要么有守卫要么在 §13.3 里标明守不住。**

本文与 [08 §6.2](./08-ipc-and-frontend.md) 的关系：08 说"用 UnoCSS"，本文说**具体怎么用、用什么、不许用什么**。

### 0.1 本次核对：现状与本文的差距

写下本文之前逐条看了源码，下面五条是真的（每条都指到了行），它们正是本文要改的东西：

| # | 现状 | 位置 | 本文的处置 |
| - | ---- | ---- | ---------- |
| 1 | **两套 token 已经漂移**：宿主 `--accent: #5b9dff`（蓝），官网 `--synra-accent: #22d3ee`（青） | `synra-app` 的 `index.html:26`；`docs` 的 `.vitepress/theme/shared/styles/tokens.css:13` | §6 R22：**两套都退役**，统一到新的暗紫，且主色要拆成两个 token |
| 2 | **`presetWind3` 没有 `targets` 选项**，[08 §6.1](./08-ipc-and-frontend.md) 写的是"UnoCSS + 显式 `targets`" | 08 的 §6.1 第 2 条 | 08 该条更正为 `build.cssTarget`；见 §4 R15 |
| 3 | **`index.html` 的 `font: 14px/1.5` 把 `1rem` 变成了 14px**，任何 rem 语义的间距/字号都静默缩到 87.5% | `synra-app` 的 `index.html:39` | §5 R19：根字号保持默认，14px 用 `text-sm` |
| 4 | **最小窗口 880 落在默认断点之间**，且根目录有 arm64 APK（手机宽度是真实目标） | `src-tauri/tauri.conf.json:15` | §4 R11：断点用标准五档，`minWidth` 对齐 `lg` |
| 5 | **全仓 0 处 UnoCSS**，只有 `pnpm-lock.yaml` 里 `@antfu/eslint-config` 的 optional peer 提到 `@unocss/eslint-plugin` | `synra-rust-app/`、`synra-js/` | §2 建 `@synra/theme`；§13 开 lint 侧守卫 |

> ★ 另有两处不属于"差距"而属于"笔误级"的问题，一并记在这里：
> `App.vue:75` 的 `.main { max-width: 900px }` 在 14px 正文字号下约合 100+ 字符行宽，超过 §5 R20 的 80 字符上限；
> `vite.config.ts:28` 只设了 `target: 'es2022'`，没有 `cssTarget`——见 §4 R15。

### 0.2 为什么这件事值得一篇文档

因为宿主界面与**第三方**插件界面必须看起来像同一个产品，而它们之间**没有运行时联系**：
插件页是一个独立文档、独立 origin、`connect-src 'none'`（[08 §7.3](./08-ipc-and-frontend.md)）。
宿主无法给插件页加样式，插件也无法从宿主继承变量。**唯一能共享的就是一份在各自构建期被读进去的 token。**

### 0.3 编号约定

**规则编号只增不改**：已经发出的编号不换含义、不复用（§12 R40 与 §13 都按编号引用）。
新增规则**追加到相关章节的末尾**，所以同一节里会出现编号不连续——
§2 的 R50、§6 的 R43–R46、§15 的 R47–R49 都是这么来的。**顺序上的不整齐是为了编号稳定。**

---

## 1. 三层与边界

### 1.1 三个消费者

| 消费者 | 它的样式从哪来 | 本文对它的强制力 |
| ------ | -------------- | ---------------- |
| **宿主界面**（`synra-app` 的 `src/`） | 构建期进产物；Tauri 主窗口 CSP `default-src 'self'` | **强**（同仓库，有守卫，§13.1） |
| **插件界面**（第三方 WebView 包） | 构建期进插件产物；插件页 CSP `style-src 'self' 'unsafe-inline'` | **弱**（对方自己构建，只能"提供"与文档，§12 R42） |
| **官网**（`docs`，VitePress） | VitePress 默认主题 + 我们注入的 **token 层**（**没有** reset） | **中**（跨仓库，但值不再是手抄副本——见 §15 R48） |
| **样式真相源**（`synra-js` 的 `@synra/theme`） | 它**就是** token 的家；发 npm，宿主、插件作者、官网都装 | **强**（同仓库上有守卫；跨仓库部分见 §15） |

### 1.2 官网也是消费者，但只吃 token 层

**这一节原先的结论是"官网不是消费者，只共享一个色值"。业主把它推翻了**：官网要和应用用同一套颜色，
最好直接共用那个库。

推翻的理由比原先的顾虑更硬——原先的做法留了一份**手抄的副本**，而副本唯一的维护机制是
"评审时有人记得"：那个色值当时就躺在 §13.3 的"守不住"清单里。**少一份副本，就少一条守不住的规则。**

但直接套用会砸掉东西：`presetSynra` 的 preflight 里有一段 **CSS reset**，而官网跑在 **VitePress 的默认主题**上，
一份 reset 会把 VitePress 自己的排版打散。所以官网只能吃 **token 层**（变量声明）——
于是 preset 的 preflight 必须**分成两段**（R47）。这是"官网成为消费者"的第一个技术后果。

**它恰好能成立的原因是两边的主题约定本来就一样**：VitePress 的暗色模式也是在 `<html>` 上加 `dark` 类，
而宿主 `index.html` 与本文的 preset 用的正是这个约定——所以 preset 发出的 `.dark` 覆盖
**无需改动就在官网上生效**。这不是巧合可以利用，而是"主题契约选对了"的表现（R40 之外的一条收益）。

**代价（写清楚）**：`docs` 的构建从此依赖一个**已发布**的 `@synra/theme` 版本。改一个 token，
应用先看到、官网要等一次版本升级。这是**有意的耦合**（业主要的正是"一起用"），
但它确实削弱了 [README §3.6](./README.md) 里"发布节奏解耦"那一条——所以那一条要跟着改，不能装作没发生。

### 1.3 `@synra/theme`：外部 UI 的适配层

**它在 `synra-js` monorepo 里**（`synra-js/packages/theme`），包名 `@synra/theme`。

**为什么不放 `synra-app`**：插件作者在仓库外，拿不到应用里的东西。

**为什么叫 `theme` 而不是 `ui`**：因为这个包里**没有组件，而且不许有**（R3 shortcuts 只放结构原语、
R4 不许假设框架）。而在前端生态里 `ui` 就等于"组件库"（`shadcn/ui`、`@chakra-ui/react`），
**组件是第三方插件作者看到 `ui` 后第一件会去找的东西**——一个名字做出的、这个包交付不了的承诺。
行业里的对照很现成：Chakra 的 token 与配置包叫 `@chakra-ui/theme`，组件包才叫 `@chakra-ui/react`。
**`theme` = token 与配置，`ui` = 组件**，这已经是默认读法。

> **会推翻它的唯一条件**：如果将来这个包要**出组件**（不只是绑定），`theme` 就低估了它，`ui` 才对。
> 但那要先改 R3/R4，而且会把组件变成我们必须负责的 API 面——**本文的立场是不出**。
> 所以这个命名问题的真身是"我们到底要不要出组件"。

**它封装的不止 UnoCSS**。这个包是这个项目里**"把外部 UI 系统对齐到同一套 token"的唯一去处**：
主色、间距、圆角、层级、模糊与发光、字体栈（含 CJK 回退）定义在**包根**（纯数据、零依赖），
各个外部系统通过**子路径**取各自的适配（R2）：

| 子路径 | 消费者 | 它给什么 |
| ------ | ------ | -------- |
| `@synra/theme` | 任何东西（零依赖） | `tokens` 纯数据、分段 CSS 文本、主题约定（`dark` 类） |
| `@synra/theme/unocss` | 宿主界面、插件界面 | `presetSynra()` / `synraConfig()`（theme + preflights + 结构性 shortcuts + blocklist） |
| `@synra/theme/vitepress` | 官网 | `--vp-c-brand-*` 的映射（亮/暗各一支紫，§15 R48） |
| `@synra/theme/css` | 内联反闪屏、任何只要变量的地方 | 分段 CSS 文本（reset 段 / tokens 段，§15 R47） |

**这个形状是整件事里最耐扩展的一点**：以后要接一个新的 UI 库（某组件库、某 CSS-in-JS），
**加一个子路径，不加一个包**（R50）——否则插件作者要面对 N 个包，而"同一套 token"会被拆成
N 份需要互相对齐的映射。

**代价与收益**：`synra-js` 因此多了一个**有运行时**的包（另一个是 `@synra/testkit`），
而它的版本线**不与那四个类型包相同**（R1）。收益是**没有第六个仓库、也没有第六个跨仓库接触点**——
上一版曾为此新增一个独立仓库，随这次改名一起取消——**仓库数仍是五个**
（[README §3.3](./README.md) 现在多了一个包，没有多一个仓库）。

规则的编号在 §2 R1。

### 1.4 设计方向速览（色 / 字 / 布局 / 原则）

这一节是"设计计划"，也是后面所有规则的来处。**具体数值标 🟡**：规则已定，值是倾向。

- **色**：产品是"两台设备之间的一条线"，所以面子是**一块屏幕**——这来自官网 `--synra-term-*` 那组 token
  的既有注释（`tokens.css:17-18`："这是一块设备屏幕，所以两种主题下都是暗的"）。本文把它从官网的一个演示面板
  提升为整个应用的面，并把色相定在**暗紫**：底色 `#0b0813`（偏紫的近黑，取替模板的 `#0f1115`），主色是一支紫。
  > ⚠️ **"暗紫"不能同时承担文字**——一支暗紫放在暗底上读不了。所以主色必须拆成两个 token：
  > `--synra-primary`（填充、发光、图形，`#7c5cff`）与 `--synra-primary-ink`（强调色**文字**，`#b9a5ff`）。
  > 这是本文从"主色是暗紫"这条指令里推出的第一个具体决定，数值与实测对比度见 §6 R22。
- **材质**：浮起的层用**毛玻璃**（`backdrop-filter` + 半透明面），活性与焦点用**发光**。
  两者都是"有代价的漂亮"，所以各有纪律：玻璃只给浮起的层（§6 R43），发光只是状态、不是皮肤（§6 R45）。
- **字**：一个超家族两个角色——**IBM Plex Sans**（正文/标题）+ **IBM Plex Mono**（只用于真正的数据与终端面）。
  一个超家族 = 两族明显不同但同源；工程味与这个产品的主题世界相符；**SIL OFL 1.1，免费可商用、允许再分发**
  （这是插件作者要把字体打进自己产物的前提，R18）。
  > ⚠️ **IBM Plex 没有中日韩字形**，而这个界面自称双语（`index.html:5`）。所以 `fontFamily.sans` 必须在 Plex 之后
  > 按平台列出 CJK 回退族——**但不打包任何 CJK 字体**（一套 CJK 字体是几十 MB，而它只需要出现在回退里，R18）。
- **布局**：这是一台**仪表**，不是一叠卡片。面与面之间靠层级和分隔，不靠"每块都套一张圆角卡"。
  Home 只有一件事——**那条线现在是什么状态**——其余都是指向别处的入口。
- **原则**：**胆量只花在一处。** 唯一值得做成记忆点的是首页那条"设备之间的线"（§7 R27）——
  而毛玻璃与发光**就是它的材质**，别处不用（§6 R43、R45）。

> ★ 自查：本文有意保留了"偏暗底色"这一条被列为常见默认手法的手法——因为**它在这里是有理由的**
> （设备屏幕，且官网已经写下了这条理由），而不是因为"暗色好看"。其余几条常见默认手法在 §0.1 里被点名退役：
> 全大写的字距标签、中点拼接的元信息与 `→`（§5 R21），每块一张同构圆角卡（§7 R26）、逐卡 hover（§8 R29）。
>
> ⚠️ **而"近黑底 + 一支亮色 + 发光"本身也是被点名的默认组合**，本文现在是朝那个方向走的——
> 因为这是**业主的指令**（"主色是暗色紫，配毛玻璃与发光"），而指令优先于默认值的顾虑。
> 那么"不撞脸"就必须靠**约束**而不是靠色相：① 中性色**只带一个色相**（全部偏紫，不许出现蓝灰——
> 一处蓝灰就把"一块材料"拆成两种材质）；② 玻璃只给浮起的层，发光只给状态（R43、R45），
> 于是"发光"在这个界面里是**信息**（这里活着）而不是皮肤；③ 全局只有一个模糊半径、一个发光颜色（R46）。
> 如果这三条被放开，这个设计会退回到"深色 SaaS 模板 + 霓虹"。**它们不是修饰性的建议，是这套方向成立的条件。**

---

## 2. `@synra/theme` 的形状

### R1 · 在 `synra-js/packages/theme`，独立版本线（MUST）

包在 `synra-js/packages/theme`，**独立版本线**，不与 `client` / `protocol` / `synra` / `testkit` 同版本。

理由：那四个包同版本的唯一理由是"它们互相 re-export，版本错位会产生'类型来自不同代次'的错误"
（[25 §4.1](./25-js-monorepo.md)）。`@synra/theme` 不 re-export 它们中的任何一个，也不被它们依赖——
**没有依赖边，就没有需要对齐的代次**。把两者绑在一起，会让"改一支紫的明度"触发一次协议包的版本号滚动。

代价（写清楚）：[25 守卫 6](./25-js-monorepo.md) 从"四个包同版本"改成**两组**——四个纯类型/替身包一组，
`@synra/theme` 自己一条线；并新增两条守卫：**它不依赖那三个包**，且"零运行时"名单里**没有**它
（它有运行时，而且必须有——它是构建期工具与映射函数）。

### R2 · 包根零依赖，适配器走子路径（MUST）

| 入口 | 内容 | 依赖 |
| ---- | ---- | ---- |
| `@synra/theme` | `tokens`（纯数据）、分段 CSS 文本、主题约定 | **零依赖**——不许 import `unocss` 或任何 UI 库 |
| `@synra/theme/unocss` | `presetSynra()` / `synraConfig()` | `unocss` 是 **optional peer** |
| `@synra/theme/vitepress` | `--vp-c-brand-*` 映射 | 零依赖（只返回对象与字符串） |
| `@synra/theme/css` | 分段 CSS 文本（reset 段 / tokens 段） | 零依赖 |

```ts
// 宿主与插件
import { defineConfig } from 'unocss'
import { synraConfig } from '@synra/theme/unocss'

export default defineConfig({ ...synraConfig() })
```

```ts
// 官网
import { vitepressBrand } from '@synra/theme/vitepress'
```

两条硬约束：

1. **包根零依赖。** 官网、内联反闪屏、以及任何"只要颜色不要工具类"的消费者都从包根或
   `./css` 取东西。**如果包根把 `unocss` 拉了进来，每个非 UnoCSS 消费者都要为 UnoCSS 付一次装包与体积。**
2. **`unocss` 必须是 optional peer，不能是 `dependencies`。** 技术原因：npm 与 pnpm 的 peer
   不能按子路径声明，它只能挂在包级；而写成 `dependencies` 就等于包根也有了依赖，第 1 条随即失效。
   `peerDependenciesMeta.unocss.optional = true` 是唯一同时满足两条的形状。

**理由（为什么必需 `synraConfig()` 而不只给一个 preset）**：`blocklist`、`shortcuts`、`presetIcons`
都是 **config 级**旋钮，一个 `Preset` 拿不到它们。只导出 preset 的后果是每个消费者各自抄一份禁用名单——
**那正是本文想消灭的那份会漂移的拷贝**。

**反例**：把 `blocklist` 写在 `synra-app/uno.config.ts` 里、插件模板再抄一份。两份名单会在第一次
"宿主发现某个类不许用"时分叉，而分叉的后果是插件里的字面颜色宿主看得见、宿主自己却看不见。

### R3 · shortcuts 只放结构原语（MUST）

**允许**：`screen` / `stack` / `row` / `panel` / `floating` / `focusable` 一类描述**结构与行为**的形状。
**不允许**：`card` / `btn-primary` / `badge` / `input` 一类**成品的视觉组件**。

理由：一旦 preset 里有 `card`，所有消费者都会长成同一张卡——那是把"统一"做成了"撞脸"。
统一该管的是"不许漂移"，不是"够不够独特"；独特性来自 §1.4 那少数一次性决策，不来自组件的复刻。

> ⚠️ **短名字不许以"变体名 + 连字符"开头。** 这一条是踩出来的：焦点环原来叫 `focus-ring`，
> 而 UnoCSS 把它解析成 **`focus` 变体 + `ring` 工具类**（`.focus-ring:focus{--un-ring-width:3px}`），
> 于是那个 shortcut 被静默遮蔽 —— 生成的是 `:focus`（鼠标点一下也亮环）、环宽也不对，
> **而名字看起来完全正常**。现在它叫 `focusable`。
>
> 判据不看名字（"哪些前缀危险"是一张永远不全的名单），看**展开结果**：一个 shortcut 的 body
> 展开出的每一条声明，都必须出现在"用 shortcut 名展开"的 CSS 里（§13.2 守卫 9）。

### R4 · 不许假设框架，不许绑 `#app`（MUST）

插件作者可能用 React / Svelte / 裸 DOM。因此 preset **不许**包含 Vue 专属的东西
（组件、指令、`v-` 相关），也**不许**写 `presetWind3({ important: '#app' })`——插件页不一定有 `#app`。
Vue 侧要的糖另开子入口（`@synra/theme/vue`），且只在宿主与官方模板里用。

### R5 · 不启用 `presetAttributify` 与 `transformerDirectives`（MUST）

| 不启用 | 理由 |
| ------ | ---- |
| `presetAttributify()` | Vue 的 attrs 与 props 在同一个位置，把 utility 写成属性会让"这个属性是组件的 prop 还是样式"变成一个要看 preset 才知道的问题；且它让标记里出现大量无来源的未知属性 |
| `transformerDirectives()` | 它带来 `@apply`，而 `@apply` 里的类**绕开 `unocss/blocklist` 与 `unocss/order`**——等于在整套强制执行上打一个洞 |

`transformerVariantGroup()` 可以启用（它只是 `hover:(a b)` 这种书写糖，不改变哪些类存在）。

**反例**：允许 `@apply` 的仓库里，"禁止字面颜色"这条规则会在第一个赶时间的人手里失效，
而失效现场（`@apply text-[#7c5cff]`）与合规现场（`class="text-primary-ink"`）在评审里看起来一样费解。

### R50 · 加一个外部 UI = 加一个子路径，不是加一个包（MUST）

要接一个新的外部 UI 系统（某组件库、某 CSS-in-JS、某设计工具、某编辑器主题），
**在 `@synra/theme` 里加一个子路径**。

理由：插件作者面对的是"我到底要装几个包"。每多一个包就多一次版本对齐、多一份依赖声明、
多一个"这个包和那个包谁说了算"的问题；而子路径的成本只有一行 import。
**这条规则是 R2 那个形状的存在理由**，也是包名为什么是 `theme` 而不是别的：它是一个**层**，
不是一个工具。

> ⚠️ 唯一例外：适配器需要**特定框架的运行时依赖**、且那个依赖大到不该让所有人付（例如某个组件库本身）。
> 那时仍先问一次"能不能只加子路径 + 把那个库声明成 optional peer"——**两个包只在子路径做不到时才允许出现**，
> 并且要在本文件里记下为什么做不到。

---

## 3. Token

### R6 · 单一真相源（MUST）

token 在 `@synra/theme` 里**声明一次**，同时喂给两处：

```
tokens (TS) ──┬──► theme.colors / theme.spacing / …   → 生成 utility（bg-panel、text-ink）
              └──► preflights 的 :root 块            → 生成 CSS 变量（var(--synra-panel)）
```

于是 `bg-panel` 与 `var(--synra-panel)` 是**同一条声明**，不是两份需要人工对齐的东西。
这直接回答了"把 CSS 变量都配进去"这件事：不是把变量抄进 preset，而是让 preset **产出**变量。

**禁止**在任何地方再写一个颜色字面值（`#` 开头的、`rgb(` / `hsl(` 的、命名色如 `red-500` 的）。
守卫见 §13.1（blocklist + 宿主扫查）。**例外只有一个**：§3 R8 的启动内联块。

### R7 · 命名空间 `--synra-*`（MUST）

现有的 `--bg` / `--panel` / `--border` / `--text` / `--muted` / `--accent` / `--warn` / `--bad`
全部改名为 `--synra-*`。

理由：宿主主窗口将来要托管第三方内容（[09 §5](./09-plugin-system.md) 的既定方向），
不带前缀的 `--border` 与任何第三方 CSS 变量都是必然碰撞；而这类碰撞的症状是"某个面板偶尔变透明"，
排查成本极高。官网那份 `tokens.css` 已经这么做了，宿主照做即可。

### R8 · 反闪屏内联块只留三件事（MUST）

`index.html` 内联的 `<style>` **只允许**三件事：`color-scheme`、页面底色、前景色。
它现在的七个变量是一个正在长成第二套设计系统的苗子，必须止住。

理由：内联这段存在的唯一理由是"首帧不能等异步资源"（`index.html:8-10` 的注释），
而只有这三件事是首帧真的需要的。token 一到齐，它就退化成第二份真相源。

**守卫**：那三个字面值必须与 preset 的对应 token 相等，由宿主守卫核对（§13.1 守卫 3）。

### R9 · 分层与预算（MUST）

- **z-index 用命名 token**（`--synra-z-base / -panel / -overlay / -modal / -toast / -drag`），**禁止裸值**。
  宿主 overlay 互相盖是唯一一类"看起来对、就是点不到"的 bug，而它只在特定交互顺序下出现。
- **token 有预算**：每个新 token 都要在 §1.4 那份设计计划里被指到用途，且**必须被某处使用**。
  这条不是新发明——官网 `tokens.css:8` 已经写着"Rule for this file: every token must be used by something
  in `theme/`. A token nobody reads is a token nobody maintains."，本文把这条规则升格为全产品的规则。
- **圆角分层**（控件 / 面 / 胶囊），禁止任意值 `rounded-[12px]`。现在的 8px / 10px / 999px 三个值肉眼不可分辨，
  正是"没有层级"的证据。

### R10 · 逻辑属性（MUST）

禁止 `ml-` / `mr-` / `pl-` / `pr-` / `text-left` / `text-right`，用 `ms-` / `me-` / `ps-` / `pe-` /
`text-start` / `text-end`。

理由：`index.html:5` 自己写着"the interface is bilingual"，而双语里迟早有 RTL。
这条现在做成本是零，等到第一个 RTL 语言进来再做成本是全量重排。

---

## 4. 断点与自适应

### R11 · 五档标准断点，每档写死"壳层发生什么"（MUST）

用 Tailwind / UnoCSS 的**标准数值**，不自己发明断点名与数值：

| 断点 | 值 | 壳层在这一档做什么 |
| ---- | --- | ------------------ |
| （base） | `< 640` | **手机竖屏**：导航在底部，单列，触控目标 ≥ 44px，内容避开 safe-area |
| `sm` | `≥ 640` | 大屏手机 / 横屏：仍底部导航，内容可以两列 |
| `md` | `≥ 768` | 平板 / 窄窗口：导航收成图标 rail |
| `lg` | `≥ 1024` | **桌面壳**：侧栏展开（与 `minWidth` 对齐，见 R12） |
| `xl` / `2xl` | `≥ 1280` / `≥ 1536` | 宽窗口：内容最大宽度生效，两个面板可并排 |

**为什么用标准数值**：插件作者是第三方，`md: 768` 是他们的肌肉记忆。**发明一套自定义断点会让每个人
在第一次写响应式时踩一次坑**，而这不是我们想要的独特性——独特性该花在 §1.4 那几处，不是花在让人猜数字上。
反过来，**每一档"做什么"必须写死在上表里**（而不是留给"到时候看"），这才是本规则真正要立的东西。

**为什么不是 880 那一档**：`minWidth` 是 880，落在 `md`(768) 与 `lg`(1024) 之间——也就是说用户把窗口
拖到最小宽度时，界面停在一个"没有设计过"的中间态。修法在 R12。

### R12 · `minWidth` 对齐 `lg`（MUST）

`src-tauri/tauri.conf.json` 的 `minWidth` 从 `880` 改为 `1024`，与 §4 R11 的 `lg` 相等。

理由：这比"为 880 单独加一档断点"更好——**它把问题从"多一个概念"变成"少一个中间态"**。
一个带侧栏的桌面应用把最小宽度定在 1024 是正常的；而让最小窗口等于断点，意味着"任何用户能拖到的尺寸"
都落在某个设计过的档位上。

**反例**：为 880 加一档 `window:` 断点。三个月后没人记得为什么 768 和 880 只差 112px，
而 880 这个数字的真实来源（一个配置文件里的常量）已经从记忆里消失。

### R13 · SFC 里不写 `@media`（MUST）

用 UnoCSS 的 variant：`md:` / `lt-md:` / `at-lg:`。**禁止**在组件的 `<style>` 里手写媒体查询。

理由与 R5 同源：手写的媒体查询是**守卫看不见**的样式——禁用名单、类顺序、token 强制对它全部失效。

### R14 · 容器查询暂缓（MUST）

宿主**不得**使用 `@container` 规则或容器查询 variant。[08 §6.1](./08-ipc-and-frontend.md) 的要求是
"Baseline Widely Available"，而 WebView 矩阵里最弱的一环是 Linux 的 WebKitGTK（[08 §6.1](./08-ipc-and-frontend.md) 的矩阵）。
插件 UI 可以**MAY**用，但必须带 `@supports` 回退。

**待办**：先在 WebKitGTK 上实测一次（§16.2 Q-3），有结论后再决定是否对宿主放开。

### R15 · CSS 兼容性靠 `build.cssTarget`，不靠 preset（MUST）

[08 §6.1](./08-ipc-and-frontend.md) 写的"UnoCSS + 显式 `targets`"**不存在这个旋钮**（见 §0.1 第 2 条）。
真正管 CSS 输出兼容性的是 Vite：`vite.config.ts` 要显式设 `build.cssTarget`（以及既有的 `build.target`），
值由 §4 R14 的实测结论与 [08 §6.1](./08-ipc-and-frontend.md) 的矩阵共同决定。

### R16 · 移动端四件事（MUST）

宿主有 Android 产物（根目录有 `synra-0.1.0-arm64-v8a-release.apk`），所以下面四条不是"以后再说"：

| # | 规则 | 不管会怎样 |
| - | ---- | ---------- |
| 1 | 贴边内容用 `env(safe-area-inset-*)` | 内容被刘海与手势条挡住 |
| 2 | 触控目标 ≥ 44×44px | 手机上"点不中" |
| 3 | hover 效果用 `@hover:` variant（UnoCSS 会输出 `@media (hover: hover) and (pointer: fine)`） | 触摸端出现"点一下卡住高亮"的粘滞 hover |
| 4 | 视口高度用 `dvh` 不用 `vh` | 软键盘弹出时底部被顶出屏幕 |

---

## 5. 字体与排版

### R17 · 字体用 `presetWebFonts`，且必须落在产物里（MUST）

用 `@unocss/preset-web-fonts`（`unocss` 包内含）。**不变量**：字体文件最终必须**在产物里**，
运行期一次网络请求都不许有。

理由：宿主主窗口 CSP 是 `default-src 'self'`；插件页是 `font-src 'self'` 且 `connect-src 'none'`
（[08 §7.3](./08-ipc-and-frontend.md)）。**任何指向 CDN 的 `@font-face` 在插件页里都是必然失败**，
而失败的样子是"字体悄悄变了"，不是报错。

两种机制，**推荐 A**：

> ★ **实现时发现机制 A 根本不需要 `presetWebFonts`。** 它要做的唯一一件事是生成 `font-family`
> 工具类，而那是 `theme.fontFamily`（同一份 token）本来就会生成的 —— 装它是一次空转。
> 所以现在 `presetSynra` 只带字族栈（含 CJK 回退），**而 `@font-face` 与字体文件那一半仍然是宿主的
> 责任**（那一半没做，见 §16.1）。机制 B 才需要那个 preset：它负责下载与改写 URL。
> **不变量不变**：运行期一次网络请求都不许有。

| | 机制 | 怎么做 | 代价 |
| - | ---- | ------ | ---- |
| **A（推荐）** | 字体文件入库 | `presetWebFonts({ provider: 'none', fonts: {…} })` 只生成 `font-family` 工具类；`@font-face` 写在一个由打包器处理的 CSS 里，字体放 `src/assets/fonts/`，`url()` 交给 Vite 改写 | 二进制文件进 git；升级字体是一次人工改动 |
| B（博客的做法） | 构建期下载 | `createLocalFontProcessor()`（`@unocss/preset-web-fonts/local`）下载到 `fontAssetsDir` 并改写 URL | **构建需要网络**；且见下面的 URL 坑 |

> ⚠️ **机制 B 的坑（实现时必读）**：processor 把 URL 写进**生成的 CSS**，而 **CSS 里的相对 URL 是相对
> 那个 CSS 文件解析的**——`vite.config.ts` 的 `base: './'` 只管文档级引用，救不了它。
> 宿主还开着 `base: './'`（Tauri 协议下路径必须相对），所以 `fontServeBaseUrl` 用相对值极易 404，
> 用绝对值又可能不被自定义协议解析。机制 B 还额外需要 `onDownload` + `emitFile` 才能在 CI 里稳定
> 把字体带进产物（官方文档给了这个补充写法）。
> **这正是推荐 A 的原因：A 里 URL 由打包器算，不存在这个歧义。**

### R18 · 两个家族、OFL、最小子集（MUST）

**家族已定**（✅ 业主授权按"合适且免费可商用"择定）：

| 角色 | 家族 | 字重 | 用在哪 |
| ---- | ---- | ---- | ------ |
| 文字 | **IBM Plex Sans** | 400 / 500 / 600 | 正文、标题、按钮、标签 |
| 数据 | **IBM Plex Mono** | 400 | 只用于设备 id、版本号、线协议日志（R21 禁止它当小标签装饰） |

**为什么是这一个超家族**：两族明显不同但同源（同一个设计语言，不是因为"两个看起来配"）；
工程/技术 Vernacular 与这个产品的主题世界相符；**SIL OFL 1.1**——免费商用、允许随包再分发，
这是 R17 的前提（插件作者的产物里会带着它）。OFL 的两条义务写在这里，免得实现时才发现：
① **许可文件必须随字体一起分发**；② 不许单独售卖字体本身（打包进插件不算）。

**CJK 回退（MUST）**：IBM Plex **没有中日韩字形**，而界面自称双语（`index.html:5`）。
所以 sans 栈必须在 Plex 之后按平台列出 CJK 族，**且不打包任何 CJK 字体**——一套 CJK 字体几十 MB，
而它只需要出现在回退里：

```ts
fontFamily: {
  sans: '"IBM Plex Sans", "PingFang SC", "Microsoft YaHei UI", "Noto Sans CJK SC", ui-sans-serif, system-ui, sans-serif',
  mono: '"IBM Plex Mono", ui-monospace, SFMono-Regular, Menlo, monospace',
}
```

**最小子集**：只带用到的字重与字形子集。字体是**每个插件产物都要下载一遍**的成本——
宿主多 100KB 是 100KB，插件多 100KB 是每个装了它的用户每次装插件都要多 100KB。

### R19 · 根字号保持默认（MUST）

`index.html` 的 `font: 14px/1.5` 删掉：根字号保持浏览器默认（16px），14px 正文用 `text-sm` 表达。

理由：把根字号改成 14px 之后，**所有 `rem` 语义的 utility 都静默地只有作者以为的 87.5%**
（§0.1 第 3 条）。而 rem 是这个系统里"跟随用户缩放与系统字号"的唯一手段——它正是为无障碍而存在的。

### R20 · 排版尺度（SHOULD）

- 一套显式字号阶梯，UI 的低端需要细步（12 / 13 / 14 / 16 / 20 / 26 / 34），每档配死行高，不许逐处微调。
- **正文行宽 < 80 字符**（衬线可以略长，并给更多行高）。`App.vue:75` 的 `max-width: 900px` 在 14px 下超标，要改。
- 字重只用 2–3 档（如 400 / 500 / 600）。理由同 R18 的最后一条：每个字重都是一份字体文件。

### R21 · 四种"模板化"标记禁用（MUST）

| 禁用 | 理由 |
| ---- | ---- |
| `uppercase` 当小标签 | 它已经成了"生成的页面"的标志手法之一，而它在这里不承担任何信息：`HomePage.vue:9-10` 的两个 `h2`（"In flight" / "Recent intents"）用大小写正常写出来同样清楚 |
| 为区分层级而加字距 | 字距是排版决策，不该被当成"让标签看起来重要"的默认手段 |
| 等宽字体当小号装饰 | 等宽只用于**真正的数据与终端面**（设备 id、版本号、线协议日志）。当"小标签字体"用是常见默认手法之一，也正是 `v{{ version }}` 最容易掉进去的地方（`App.vue:108`） |
| 中点拼接的元信息、缀在文案后的 `→` | `App.vue:108`（`platform · v…`）与 `HomePage.vue:93`（`kind → peer · state`）各是一例。它能编码的信息用分隔符或标签同样能表达，而它在这里是"不知道用什么连接符"的默认产物 |

---

## 6. 色彩、材质与状态

> ★ **规则编号只增不改**（约定见 §0.3）：所以本节里 R22–R24 是色彩与状态，R43–R46 是后加的材质规则。

### R22 · 主色是一支暗紫，因此必须拆成两个 token（MUST）

**两个都要改**：宿主的 `#5b9dff`（蓝）与官网的 `#22d3ee`（青）都退役，统一到暗紫。取值（🟡 倾向，实测比值）：

| token | 值 | 用途 | 实测对比度 |
| ----- | -- | ---- | ---------- |
| `--synra-primary` | `#7c5cff` | 填充、发光、图形（**不用于小字**） | 4.57:1（底）/ 4.00:1（浮起面）——过 3:1 的控件线，不过正文线 |
| `--synra-primary-ink` | `#b9a5ff` | 强调色**文字**、链接 | 9.34:1（底）/ 8.19:1（浮起面） |
| `--synra-primary-deep` | `#5b3fd6` | 按下态、渐变暗端 | —（不与文字直接配对） |

> ⚠️ **"暗色紫作为主色"这条指令的直接后果就是这张表有两行紫。**
> 一支暗紫在暗底上读不了正文——`#7c5cff` 在浮起面上只有 4.00:1。
> 所以"主色"在实现里是两个角色：**能发光的**（填充）与**能读的**（文字）。
> 把它们混成一个 token 的后果是"链接要么太暗、要么发光得像个按钮"。

理由：① 中性的暗紫不与"通用亮蓝/通用青"撞脸，而这两支正是"没有人做过选择"的颜色；
② 它给 §1.4 的毛玻璃与发光提供了一支**可以在暗底上发光而不刺眼**的紫——`#7c5cff` 的亮度刚好落在
3:1 与 4.5:1 之间：做光晕够亮，做文字不够。这正是它该待的位置。

**这次是 0.x 期的改值，代价为零**；一旦有插件发出去，同样的改动就要走 R40 的规矩（改值 = 破坏性视觉变更）。

### R23 · 状态色是独立 token，且状态不靠色相单独表达（MUST）

`--synra-ok` / `--synra-warn` / `--synra-danger` / `--synra-offline` 与主色**分开声明**，
且设备状态（在线 / 离线 / 配对中）必须同时有**文字或形状**的冗余表达。

取值（🟡）：`--synra-ok: #5fd6a8`（11.0:1）、`--synra-warn: #e8b055`（10.2:1）、`--synra-danger: #ff7676`（7.7:1）。

理由：这个界面主要显示的就是状态，而"只有颜色不同的两个点"对色觉障碍用户等于没有区别；
另外主色一旦被拿去当"在线"用，它就不能再用于"可点"——一个 token 承担两个语义是漂移的起点。
**紫色主色还多一层风险**：它对红绿色盲用户与 `danger` 的区分依赖亮度而不是色相，
所以状态上的"文字或形状"冗余在这套配色里不是加分项，是必需品。

### R24 · 对比度是计算出来的守卫（MUST）

preset 里成对声明前景/背景，测试**算 WCAG 比值**：正文 < 4.5:1 或大字/控件 < 3:1 即失败。
§1.4 与 R22/R23 里的每一个比值都是这条守卫在实现时的预期输出，不是设计稿上的标注。

**它已经能抓到东西**——按上一版（青）的取值算，官网的 `--synra-term-dim: #6b7d8a` 作正文色是 **4.47:1**，
差 0.03 被守卫判红。新配色里 `--synra-ink-dim: #a89dc6` 是 7.85:1（底）/ 6.88:1（浮起面），过了；
但这条守卫的第二个战果**已经预定**：`--synra-line`（`#2e2547`，1.22:1）**不够当控件边界**——
暗底上要 3:1 的边框必须相当亮（`#8b7ab8`，4.62:1），所以"分隔线"与"控件边界"必须是两个 token：

| token | 值 | 用途 | 对比度 |
| ----- | -- | ---- | ------ |
| `--synra-line` | `#2e2547` | 分隔线（纯装饰） | 1.22:1 |
| `--synra-line-strong` | `#8b7ab8` | 输入框边缘、聚焦目标等**承担语义的**边界 | 4.62:1 |

### R43 · 毛玻璃是浮起层的材质，不是基底（MUST）

`backdrop-blur-*`（以及任何 `backdrop-filter`）**只允许出现在浮起的层**：对话框、命令面板、
以及首页那条"线"所在的浮层。**静止的页面与面板必须是不透明 token。**

理由有三条，第一条是硬的：

1. [08 §6.1](./08-ipc-and-frontend.md) 的 WebView 矩阵里，`backdrop-filter` **正是被点名的差异项**
   （WKWebView 部分 CSS 差异；WebKitGTK 最弱）。所以玻璃**必须有回退**，而回退等于"不透明底色"——
   这只在玻璃叠在不透明基底之上时才成立。
2. 玻璃让对比度变成"取决于后面是什么"，而 R24 的守卫只能算静态对（见 R44）。
3. 屏幕上同时只能有一层可见的玻璃：叠两层模糊在 Android WebView 与 WebKitGTK 上是真的掉帧，
   而且两层玻璃互相模糊之后"浮起"这个语义就没有了——**浮起之所以能被看见，是因为它下面那层不动。**

### R44 · 玻璃的对比度按它的不透明基底算（MUST）

每个玻璃面必须**成对**声明：`--synra-glass`（半透明面 + 模糊）与 `--synra-glass-base`（不透明基底色）。
R24 的守卫用**基底色**算对比度，不用模糊后的结果；玻璃自身透明度不得低于约 60%——
再低就等于"没有基底"，那时任何对比度结论都不成立。

取值（🟡）：`--synra-glass: rgb(28 22 48 / 62%)` + `backdrop-filter: blur(14px) saturate(140%)`，
基底 `--synra-glass-base: #1c1630`；发光色 `--synra-glow: rgb(124 92 255 / 45%)`；
模糊半径 `--synra-blur: 14px`（全局唯一，见 R46）。

### R45 · 发光是状态，不是皮肤（MUST）

glow 只允许出现在**三处**：

1. `focus-visible` 的环（R35）；
2. 那条"线"的**活动**状态（连接中 / 已连接 / 传输中，§7 R27）；
3. 主按钮的按下/激活态。

**禁止**把发光当静止阴影用（卡片、面板、图标底、选中项的静态外观一律不行）。

理由：① 两处发光等于没有重点，暗色界面上的光晕会互相抵消；
② 大半径的彩色 `box-shadow` 在 Android WebView / WebKitGTK 上是真的掉帧；
③ 这是本文"胆量只花一处"（§1.4）在材质上的具体形式——**发光在这里的含义是"这里活着"**，
一旦它变成装饰，这条信息就没了。

### R46 · 一个模糊半径，一个发光颜色（MUST）

`--synra-blur` 与 `--synra-glow` 各只有一个值，并计入 §3 R9 的 token 预算。

理由：玻璃的"厚度"与发光的"强度"是**材质**而不是**可调参数**——两个模糊半径会让同一屏出现两种玻璃，
而那读起来像两种材质而不是两种层级。要区分层级用面色的明度差（`screen` / `surface` / `raised`），
不要用模糊半径。

---

## 7. 布局与结构

### R25 · 结构装饰必须编码信息（MUST）

边框、分隔线、编号只在它们**说明结构**时出现：

- ✅ `App.vue` 的 `border-right`（侧栏与内容的分界）与 `border-top`（身份块与导航分开）是信息。
- ❌ 编号标记（01 / 02 / 03）**仅当内容真的是一个序列**（步骤、时间线）才允许。
- ❌ 不为了"看起来有层次"加边框或分割。

### R26 · 面不是卡（MUST）

界面由**面板与层级**构成，不由"每块内容一张同构圆角卡"构成。具体到现在的代码：
`HomePage.vue:15-19` 那三张等长文字、同构的 tile（Devices / Plugins / Settings）是这条规则要治的形态。

**允许**卡片式的容器，但**同屏不许出现三张以上完全同构的卡**；入口式的链接应当用更轻的形态
（一行文字 + 一个方向），而不是三个占满宽度的方框。

### R27 · 一个记忆点：设备之间那条线（SHOULD）

首页唯一值得花胆量的东西是**这个产品特有的东西**：两台设备之间那条线现在处于什么状态。
它是 §1.4 里"仪表"这个布局原则的具体落点，也是官网 `--synra-term-*` 那组 token 该被用到的地方。

**其余全部保持克制**：设置页、插件列表、错误面、空状态都不许争抢注意力。

**约束**：这条线展示的必须是**真状态**（配对中 / 已连接 / 正在传输 / 无连接），不是装饰性动画。
假内核下它应当诚实地画成"还没有连接"，并且那个空状态就是一个动作（R28）。

### R28 · 空状态是邀请，不是陈述（MUST）

不要把"什么都没有"写成一句话就完。空状态要给**下一步动作**。

现例：`HomePage.vue:88-90` 的 "Nothing is being transferred." 与 `:98-100` 的
"Nothing has been shared to this device yet." 都是陈述句；它们应当指向"去配对一台设备"这类可点的下一步。

---

## 8. 动效

### R29 · 非用户触发的动效，每屏一个（MUST）

一次入场或一次揭示，就够了。**禁止**每段都 fade-and-slide-up、**禁止**每张卡都有 hover 过渡
（现例：`HomePage.vue:33` 的 `.tile:hover` 与 `App.vue:45` 的 `.link:hover`）。

**用户动作引发的动效不受此限**（展开、确认、发送成功、状态变化），因为它回答的是"什么变了"。

### R30 · 时长与缓动来自 token，且尊重 reduced motion（MUST）

- 禁止字面 `300ms` / `ease-in-out`；用 `--synra-duration-*` / `--synra-ease-*`，并 blocklist 掉 `duration-[…]`。
- `prefers-reduced-motion: reduce` 下必须**真的减少**（不是把动画调快）——它对应 UnoCSS 的 `motion-reduce:` variant。

---

## 9. 文案

### R31 · 组件里没有裸的用户可见字符串（MUST）

所有用户可见文字走一个 messages 模块，**不许**在模板里直接写。

理由：两条合一条——一是双语（`index.html:5`），二是"文案不许各写各的"。现在的 `loading…` /
"Nothing is being transferred." / "The kernel refused the snapshot:" 都是散落在组件里的字面量。

### R32 · 一份词汇表，一个动作全程同名（MUST）

- **词汇表**：一个东西只有一个词。现在的四种说法（`peerName`、device、"nearby"、"shared to this device"）
  要收敛成"设备"一个词，并在词汇表里登记。
- **同名贯穿**：命令名 → 按钮 → 结果提示用同一个词。按钮说"发布"，提示就必须说"已发布"。

### R33 · 错误必须渲染 `actions`，且不道歉、不含糊（MUST）

**这条不是新要求，是既有契约的一半没接上。** `IpcError` 已经带了 `message` + `actions: Vec<SuggestedAction>`
（[08 §2.3](./08-ipc-and-frontend.md)），而那份设计的原话就是"前端需要可编程的错误（switch code、渲染按钮）"。
现在前端只读了 `message`：`HomePage.vue:148` 把 `refusal.error.message` 直接塞进界面，`actions` 一个都没渲染。

- 每个错误面**必须**处理 `actions`（按钮或指向）。
- 不许道歉、不许模糊：说清**发生了什么**、**能做什么**。
- 保留原始码（`code`）给诊断页用，但主界面不展示裸错误串。

### R34 · 界面里不出现内部词（MUST）

`kernel` / `snapshot` / `refused` / `capability` / `intent`(内部语义) 这类只能出现在诊断页。
现例：`HomePage.vue:69` 的 "The kernel refused the snapshot" 是给实现者看的句子，不是给用户看的。

---

## 10. 无障碍质量下限

### R35 · 可见的键盘焦点（MUST）

`focus-visible` 的环是一个 token（`--synra-focus-ring`），**始终存在**：
不许 `outline: none` 而没有替代，不许只在 hover 时可见。

### R36 · 其余质量下限（MUST）

不需要声明，但一条都不许省：键盘可达全部交互、对比度（R24）、reduced motion（R30）、
缩放与系统字号（R19）、触控目标（R16）、状态不靠色相单独表达（R23）。

---

## 11. 宿主专属规则

### R37 · `<style>` 是一个需要理由的逃生舱（MUST）

宿主允许 `<style>` 块**仅当** UnoCSS 表达不了：复杂选择器、`:deep()` 透传 `v-html`、`grid-template-areas`。

使用条件（守卫扫描，§13.1 守卫 4）：

1. 块内必须有**一句理由注释**；
2. 块内**只读 `--synra-*` 变量**，不许出现任何颜色/尺寸字面值；
3. 不许出现 `@media`（R13）。

### R38 · 第三方界面只进独立 WebView（MUST）

宿主主文档**不得**嵌入第三方 UI（同文档的 iframe / 组件都不行）。

理由：这是整个 token 体系的前提。独立文档 ⇒ 各带各的 CSS ⇒ preflight 与全局选择器互不影响；
一旦同文档，插件页的 reset 会打到宿主界面上，症状是"某些宿主样式偶尔消失"。
反向也成立：宿主不得依赖"给插件页加样式"这种能力，因为插件页在另一个 origin。

---

## 12. 插件 UI 专属规则

### R39 · preset 必须自包含（MUST）

`presetSynra` 产出的东西必须**不依赖宿主任何东西**：token 由 preflight 自己发，`@font-face` 自己带，
没有一个 shortcut 读"宿主会提供的变量"。

理由：插件页是另一个文档、另一个 origin（R38），宿主的 `:root` 到不了那里。

### R40 · 版本偏移是不对称的（MUST）

UnoCSS 是**构建期**的，插件的 CSS 编译进它自己的产物。于是：

| 改动 | 对已发布插件的影响 |
| ---- | ------------------ |
| token **改名** | **安全**。旧插件的产物里还是旧名字 + 旧声明，照常跑 |
| token **改值** | **静默改掉所有旧插件的外观** |
| shortcut 改定义 | 同"改值"（它被编译进了插件产物） |

**规则**：改值视为**破坏性视觉变更**；宁可新增一个 token，也不要改旧 token 的值。
这一条不受"反正是 0.x"影响——第三方已经装出去的产物不会跟着升级。

### R41 · 图标必须构建期（MUST）

**不设 `presetIcons` 的 `cdn` 选项** —— 这就是"不取 CDN"，+ `@iconify-json/*` 作为 devDependency。

理由：插件页 `connect-src 'none'`，运行期取图标**必然失败**，而失败的样子是一排空方块——
它看起来像图标库坏了，而不是像网络被拒。**注意**：这一条与 §1.4 引用的参考工程相反
（那边写的是 `cdn: 'https://esm.sh/'`），那个写法在 Synra 的两个 CSP 下都不能用。

> ★ **本节原来写的是 `cdn: false`，而那个写法在 `@unocss/preset-icons@66` 里是类型错误。**
> 那个版本的选项是 `cdn?: string` —— 只有一个 URL，**没有"关掉"的布尔值**（`false` 是更早版本的配方）。
> 实现时被 `tsc` 纠正过来：**"不取 CDN"的表达方式是不设它**，所以现在的守卫断言的是
> "preset 自己的 options 里没有 `cdn` 这个键"（§13.2 守卫 10）。
> 这也是为什么"照抄一份能跑的配置"不够 —— 选项的形状会变，而抄来的那份不会跟着变。

### R42 · 诚实条款：第三方守不住（MUST 写明）

插件用自己的构建，我们**无法**强制它使用 preset。所以：

- preset 是**提供**，不是**强制**；
- 落实点在三个地方：**官方模板**（生成出来的工程默认带 `uno.config.ts`）、**文档**
  （`synra-rs/docs` 的 `plugins/ui.md` 的 Styling 一节）、**我们自己写的示例插件**；
- **不做**"校验插件产物里是否含 `--synra-` 前缀"这类拒绝式检查——它会把"没用我们的样式"变成一个
  安装失败，而理由对作者不显然。

**这条规则的价值就是它写明了自己守不住。** 一条守不住的规则如果写得像能守住，会烂成"没人知道它还算不算数"。

---

## 13. 强制执行

### 13.1 有守卫的（`synra-app` 仓库）

| # | 守卫 | 保证哪条 | 怎么实现 |
| - | ---- | -------- | -------- |
| 1 | lint 的类检查 | R6、R10、R21、R30、R45、R46 | `@antfu/eslint-config` 的 `unocss: true` + `@unocss/eslint-plugin`（`pnpm-lock.yaml` 里已是 optional peer，一行启用）→ `unocss/order` + `unocss/blocklist`；**禁用名单来自 `synraConfig()`**，宿主与插件模板共用一份 |
| 2 | 没有字面颜色 | R6 | 同上 blocklist + `scripts/check-shapes.ts` 新增扫查（`.vue` / `.css` 里出现 `#hex`、`rgb(`、`hsl(` 即失败；白名单就是 preset 自己声明的那组 token，此外为空） |
| 3 | 反闪屏与 token 一致 | R8 | `scripts/check-shapes.ts`：读 `index.html` 内联块里的三个值与 `@synra/theme` 的 token 比对 |
| 4 | `<style>` 逃生舱 | R37 | `scripts/check-shapes.ts`：每个 `<style>` 块必须有理由注释、至少一个 `--synra-` 引用、零字面值、零 `@media` |
| 5 | token 与变量同源 | R6 | `@synra/theme` 自己的测试：preflight 发出的变量名集合 **==** theme 的颜色键集合 |
| 6 | 对比度 | R24 | `@synra/theme` 自己的测试：成对算 WCAG 比值，不达标即失败 |
| 7 | 禁用名单真的会红 | R6、R10、R21 | 负控制：加一个 `uppercase` / `text-[#fff]` 到测试夹具 → 守卫必须报错 |
| 8 | 第三方不进主文档 | R38 | 结构守卫：宿主 `src/` 下不出现 `iframe` / `srcdoc` / 外部 UI 装载 |
| 9 | preset 自包含 | R39 | `@synra/theme` 的测试：不依赖 `@synra/client` / `@synra/protocol` |
| 10 | 接入方式 | R1、R2 | 宿主 `pnpm-workspace.yaml` 里有 `'@synra/theme': link:../synra-js/packages/theme`，且 `uno.config.ts` 从 `@synra/theme/unocss` 取 `synraConfig()`（而不是手抄一份配置） |
| 11 | 玻璃与发光的用法 | R43、R45 | `scripts/check-shapes.ts`：`backdrop-` 只允许出现在白名单里的几个**浮起层组件**（对话框 / 命令面板 / 那条线所在的浮层）；带 `--synra-glow` 的用法只允许出现在 focus-ring、线的活动态、主按钮按下态。另加一条 `@supports (backdrop-filter: …)` 的回退测试（回退必须落到 `--synra-glass-base`） |

> **R5 不在这张表里，因为它由结构保证**：`presetAttributify()` 与 `transformerDirectives()` **不在
> `synraConfig()` 默认集里**，要用它们得显式改配置。一条"默认不存在"比一条 lint 规则更难绕过——
> 这也与 [25 §7 的取舍](./25-js-monorepo.md)同一个思路：少一步就是少一个会忘记跑的地方。

### 13.2 有守卫的（`synra-js` 仓库的 `packages/theme`）

它**在这个仓库里**（R1），所以守卫落在两处：包自己的测试（构建期行为），
与 [25 §6](./25-js-monorepo.md) 的 `scripts/check-shapes.ts`（清单、依赖关系与"哪一层能碰到什么"）。

| # | 守卫 | 落在哪 | 保证哪条 | 实现 |
| - | ---- | ------ | -------- | ---- |
| 1 | preflight 发出的变量名集合 **==** `colors` 的键集合（加一个颜色就自动多一个变量） | 包测试 | R6 | ✅ `tokens.test.ts` |
| 2 | 成对前景/背景的 WCAG 比值（正文 4.5、大字与控件 3），玻璃面按**基底**算 | 包测试 | R24、R44 | ✅ `tokens.test.ts` |
| 3 | 玻璃必须成对声明 `--synra-glass` 与 `--synra-glass-base`，且基底是一个真的不透明色 | 包测试 | R44 | ✅ `tokens.test.ts` |
| 4 | `--synra-blur` / `--synra-glow` 各只有一个值 | 包测试 | R46 | ✅ `tokens.test.ts` |
| 5 | 禁用名单**正向与负向**都有：`uppercase` / `ml-4` / `text-[#fff]` / `z-[99]` 被拦，而 `ms-4` / `bg-surface` / `focusable` 不被拦；每条都带 message | 包测试 | R6、R10、R21、R30 | ✅ `unocss.test.ts` |
| 6 | **包根零依赖**：manifest 里没有 `dependencies`，`unocss` 是 optional peer，入口不 import 它 | `check-shapes.ts` | R2 | ✅ |
| 7 | **只有 `./unocss` 那层能碰 `unocss`**：除 `src/unocss/**` 外不许有文件 import 它（三种形态：`from`、副作用 `import 'unocss'`、动态 `import()`） | `check-shapes.ts` | R2、R50 | ✅ |
| 8 | token 变量名集合与 `./css` 输出的一致（两个入口不能各有一套） | 包测试 | R2、R6 | ✅ `tokens.test.ts` |
| 9 | **shortcut 没有被遮蔽**：body 展开出的每一条声明都出现在"用名字展开"的 CSS 里 | 包测试 | R3 | ✅ `unocss.test.ts` |
| 10 | 图标 preset 的 options 里没有 `cdn`（也没有任何 `http`） | 包测试 | R41 | ✅ `unocss.test.ts` |
| 11 | preflight 三档：`app` 有 reset、`tokens` **无元素选择器**、`none` 是空串 | 包测试 | R47 | ✅ `unocss.test.ts` |
| 12 | 真跑一次 UnoCSS 生成器：工具类读的是变量、而 **Tailwind 的 reset 没有漏进来**（`border-width: 0` 不出现） | 包测试 | R6、R47 | ✅ `unocss.test.ts` |
| 13 | 官网映射：亮/暗各用能读的那一支紫，并在**各自的底色**上过 4.5:1 | 包测试 | R48 | ✅ `vitepress.test.ts` |
| 14 | token 的**预算**：变量名集合与 shortcut 名字集合都是写死的列表，加一个要改测试 | 包测试 | R9 | ✅ 两个测试文件 |

> ★ **守卫 9 的判据是踩出来的**：第一版写的是"哪些前缀危险"的名单，而 `focus-ring` 这个名字
> （`focus` 变体 + `ring` 工具类）证明了那种名单永远不全。改成看**展开结果**之后，它同时抓得住
> 变体遮蔽、内建规则遮蔽与错拼（负控制实测：把 `focus-ring` 加回去，它报
> "少了 `outline:2px solid transparent`"）。
>
> ★ **守卫 7 的第一版漏了一种形态**：模式写的是 `from 'unocss'`，于是**副作用导入** `import 'unocss'`
> 从它下面走过去了 —— 而那个形态恰恰是最容易被顺手写下的。负控制抓出来的，现在三种都覆盖。

### 13.3 守不住的（写明，不假装）

| 规则 | 为什么守不住 | 靠什么 |
| ---- | ------------ | ------ |
| R42 第三方插件用 preset | 对方自己构建（§12 R42） | 官方模板 + 文档 + 示例插件 |
| R25–R28 的结构与文案质量 | 这是判断，不是断言 | 评审 + §14 的流程 |
| 视觉回归 | 没有 golden screenshot 之前，`pnpm run check` 看不见"设计还在不在" | **目前的纪律**：改 UI 必须在真机/真宽度上看过（§14） |

> ★ **这张表少了一行，而"少一行"正是官网成为消费者的收益。**
> 上一版的第一行是"R22 主色在官网一致 | 三个独立仓库 | **评审项**"——它守不住的根因是**值有两份**。
> 现在官网吃 token 层（§15 R48），副本消失，**这条规则连同它的"守不住"一起消失了**。
> 一条规则从"靠评审"变成"不需要规则"，比给它加一条守卫更值得记下来。

---

## 14. 流程

1. **先有一份四行的设计计划**（色 / 字 / 布局 / 原则），并说清**哪一处不是默认值**。落在本文 §1.4 或改动说明里。
2. **看完再算完**。宿主有假内核（`synra-app` 的 `src/kernel.ts`），`pnpm dev` 在普通浏览器里就能跑，
   所以"看一眼"的成本很低：至少要在 `lg` 与手机两个宽度下看过。
3. **自检那几条常见默认手法**：全大写标签、中点拼接的元信息、逐卡 hover、三张同构圆角卡、
   等宽字体当小标签、`→` 缀在文案后，以及**"近黑底 + 一支亮色 + 发光"这个组合本身**。
   前六条出现任何一条，先问"这是选择还是默认"；最后一条不必问——它是有意的（§1.4），
   但要检查那三条约束（单一色相的中性、玻璃只给浮起层、发光只给状态）**都还在**。
4. **胆量只花一处**：改完之后去掉一件装饰（R27）。对这套材质尤其要问一句：**这块玻璃/这处发光，删掉会不会少一个信息？**
   会 → 留着；不会 → 删掉。

---

## 15. 官网（第三个消费者）

官网从"只共享一个色值"变成**真正的消费者**（§1.2），所以它有自己的规则。三条，都是 MUST。

### R47 · preflight 分两段：reset 与 token 声明是两个开关

`presetSynra({ preflight })`（在 `@synra/theme/unocss`）与 `@synra/theme/css` 的分段函数接受三档：

| 值 | 发出什么 | 谁用 |
| - | -------- | ---- |
| `'app'`（默认） | CSS reset + `:root` / `.dark` 的 token 变量 | 宿主界面、插件页面——**各自是独立文档，reset 是安全的**（R38、R39） |
| `'tokens'` | 只有 token 变量，**没有 reset** | **官网**（跑在 VitePress 默认主题上，reset 会打散它的排版） |
| `'none'` | 都不发 | 测试，以及"只要 utility、不要变量"的场合 |

理由：reset 是**文档级**的决定，而 token 变量是**跨仓库共享**的东西。把它们绑在同一个开关上，
"想共享颜色"就必然"接受一份 reset"——而官网正是**不能**接受 reset 的那个消费者。
**把两个决定拆开，才让第三个消费者存在。**

**守卫**：`preflight: 'tokens'` 时输出的 CSS 里不得出现任何元素选择器（`html` / `body` / `*` / `:where(*)`）。

### R48 · 官网的映射：亮色与暗色各用哪一支紫

`.vitepress/theme/shared/styles/chrome.css` 把 VitePress 的 `--vp-c-brand-*` 映射到我们的 token，
**而亮色与暗色用不同的那一支紫**——因为"能读的紫"在两种底色上是两个值。全部是实测：

| VitePress 变量 | 亮色主题 | 暗色主题 |
| -------------- | -------- | -------- |
| `--vp-c-brand-1`（链接与主色文字） | `--synra-primary-deep` `#5b3fd6`（白底 **6.72:1**，`#f6f6f7` 上 6.22:1） | `--synra-primary-ink` `#b9a5ff`（VitePress 暗底 `#1b1b1f` 上 **8.08:1**） |
| `--vp-c-brand-2` / `--vp-c-brand-3`（hover、实心） | `--synra-primary` `#7c5cff` | `--synra-primary` |
| `--vp-c-brand-soft` | 主色的浅底 | 主色的深底 |

**这也给 `--synra-primary-deep` 找到了第二个用途**（R9 要求每个 token 都得有人用）：
它同时是"按下态"与"亮色主题里的主色文字"。一个 token 有两个正当用途，比加第三个紫好。

**接入方式**：`.vitepress/config.mts` 加 `UnoCSS()` 插件 + 一份 `uno.config.ts`
（`synraConfig({ preflight: 'tokens' })`，来自 `@synra/theme/unocss`），
而 `--vp-c-brand-*` 的映射来自 `@synra/theme/vitepress`；本地开发用
`overrides: link:../synra-js/packages/theme`，CI 用版本依赖。
**官网的 token 值从此不再是副本**，所以 §13.3 那条"守不住"被删掉了。

### R49 · 官网的 `--synra-term-*` 退役

官网那份 `tokens.css` 有一组"演示面板"专用色（`--synra-term-bg/fg/dim/line`，含蓝黑 `#0b0f14`），
注释写着"这是一块设备屏幕，所以两种主题下都是暗的"。

**现在这句话字面成立了**——应用的底色**就是**那块屏幕。所以这组的取值改为**引用应用 token**
（`--synra-screen` / `--synra-surface` / `--synra-ink` / `--synra-ink-dim` / `--synra-line`），自己不再带色相。

理由：一处蓝灰就足够把"一块材料"拆成两种材质（§1.4）——`#0b0f14` 是**偏蓝**的近黑，而应用是偏紫的。
而"两种主题下都是暗的"这条要求在引用之后**自然满足**，因为应用 token 本身就是暗的。
**它是官网上唯一可以在亮色主题里出现暗面的地方**，而这正是它原本的意图（设备屏幕不随页面变白）。

---

## 16. 实现清单与待讨论

### 16.1 实现清单

> ★ **这一节不再全部是空的**：`@synra/theme` 本身已经落地（四个入口、14 条守卫），
> 而**宿主与官网的接入仍未开始**。下面按"已做 / 未做"分开，因为混在一起会让"还差多少"变成一个
> 需要逐行读的问题。

**已完成**：

- [x] `packages/theme`：包骨架 + 四个入口（`.` / `./css` / `./unocss` / `./vitepress`），
      `unocss` 走 `peerDependenciesMeta.optional`（R2）
- [x] token 真相源：14 个颜色 + 材质（玻璃/发光/模糊/焦点环）+ 层级 + 圆角 + 时长 + 字族（含 CJK 回退）+ 断点
- [x] 分段 CSS：`css('app' | 'tokens' | 'none')`，而 `tokens` 档**没有元素选择器**（R47）
- [x] `presetSynra()` / `synraConfig()` / `synraBlocklist` / 结构性 shortcuts（R3、R5、R6、R10、R21、R30）
- [x] `vitepressBrand`：亮/暗各一支紫，各自的底色上都过 4.5:1（R48）
- [x] §13.2 的 **14 条守卫**，每条都做过负控制（含两条因此被改强的：shortcut 遮蔽、副作用的 unocss import）
- [x] `check-shapes.ts` 的两条层守卫 + 版本线拆两组 + 发布物的构建缓存（R1、R2）
- [x] `@synra/theme` 的发布物实测：37 项、四个入口都能被 Node ESM 直接 import

**未完成**（宿主与官网）：

- [ ] `synra-app`：装 `unocss` + `@synra/theme`，写 `uno.config.ts`（`synraConfig()` 来自 `@synra/theme/unocss`）
- [ ] `synra-app`：`pnpm-workspace.yaml` 的 `overrides` 加一行 `'@synra/theme': link:../synra-js/packages/theme`
- [ ] `synra-app`：`index.html`——七变量退役为三件事；删掉 `font: 14px/1.5`
- [ ] `synra-app`：`App.vue` / 四个 page 的 scoped CSS 迁到 utility，`<style>` 只留 R37 允许的那些
- [ ] `synra-app`：`tauri.conf.json` 的 `minWidth` 改 1024（R12）
- [ ] `synra-app`：`vite.config.ts` 显式设 `build.cssTarget`（R15）
- [ ] `synra-app`：`eslint.config.ts` 开 `unocss: true`，装 `@unocss/eslint-plugin`
- [ ] `synra-app/scripts/check-shapes.ts`：新增守卫 2 / 3 / 4 / 8 / 11（各配负控制）
- [ ] 材质落地到组件：`floating` 的 `@supports` 回退、发光只出现在允许的三处（R43、R45）
- [ ] 字体：字族栈已定（IBM Plex Sans + Mono，R18），**而字体文件与 `@font-face` 那一半没做**
      （R17：宿主负责把它放进产物；许可文件随产物）
- [ ] 文案：messages 模块 + 词汇表（R31、R32）
- [ ] 错误面渲染 `actions`（R33）
- [x] `synra-rs/docs` 的 `plugins/ui.md` 的 Styling 一节：已按 §12 里**今天就已经成立**的部分重写
      （页面可以加载什么、主题、文字方向；中英两侧，`pnpm check` 三条守卫通过）
- [ ] 同上的 **install 说明在 `@synra/theme` 发布之后**才补进那一页——
      现在不能写：`docs` 的守卫不允许出现"计划中"的东西（`check-public-content.ts`），
      而一个还没发布的包写进公开文档就是那句守卫要拦的话
- [ ] 插件官方模板：生成出来的工程默认带一个用 `synraConfig()` 的 `uno.config.ts`
- [ ] **官网（§15）**：`docs` 装 `unocss` + `@synra/theme`；`.vitepress/config.mts` 加 `UnoCSS()` 插件
      + `uno.config.ts` 用 `synraConfig({ preflight: 'tokens' })`（R47）
      ★ **这一组必须排在 `@synra/theme` 首次发布之后**：CI 装不到一个还不存在的包
      （本地可以先 `link:` 联调，但那不是"能发布的状态"）
- [ ] **官网**：`chrome.css` 的 `--vp-c-brand-*` 映射按 R48——亮色用 `--synra-primary-deep`，
      暗色用 `--synra-primary-ink`（实测 6.72:1 / 8.08:1）
- [ ] **官网**：`tokens.css` 的 `--synra-term-*` 退役为引用应用 token（R49）
- [ ] **官网**：`overrides: link:../synra-js/packages/theme`（本地）；CI 走版本依赖
- [ ] **`27-website.md` 的 §8.2** 要加一条更正：那条"别加自定义样式"现在有了一个例外（R47），
      而它**不是被违反，是被收窄**——见 §15

### 16.2 待讨论 🔴

**本轮已定（业主答复）**，落点不在这一节：

| 问题 | 结论 | 落点 |
| ---- | ---- | ---- |
| Q-1 字体家族 | **IBM Plex Sans + IBM Plex Mono**（"合适且免费可商用"→ SIL OFL 1.1，允许随包再分发） | §1.4、R18 |
| （新）包名与位置 | **`@synra/theme`**，回到 `synra-js/packages/theme`；**不叫 `ui`**（`ui` 承诺组件，而这个包不许有组件），也不做第六个仓库；外部 UI 一律走子路径 | §1.3、§2 R1/R2/R50 |
| Q-7 官网跟不跟 | **跟，而且直接共用这个库**；`--synra-term-*` 退役、preflight 分两段、亮/暗各一支紫 | §1.2、§15 R47–R49 |

仍是 🔴 的：

- **Q-2 字号阶梯的具体值**：§5 R20 给的是倾向（12/13/14/16/20/26/34）。家族已定，这组值可以跟着定稿，
  但仍需要有人在真机上把正文 14 与 13 看一遍。
- **Q-3 容器查询**：需要一次 WebKitGTK 实测（R14）。在此之前宿主禁用、插件需回退。
- **Q-4 字体机制 A 还是 B**（R17）。推荐 A（入库、无网络依赖、URL 由打包器算）。
- **Q-5 亮色主题**：本文按"**v1 只做深色**"写，并把 `color-scheme` 从 `light dark` 改为 `dark`（诚实：
  现在只有深色代码，写着 `light dark` 是自相矛盾）。若将来要做亮色，它需要**一次独立的设计**，
  不是把深色取反——token 名已按"可加第二套值"的形状设计，加值是加一段 `.light` 覆盖，不改名。
  > ⚠️ **而官网**（VitePress）**是亮色默认的**——所以"只做深色"这句话**只对应用成立**。
  > 官网亮色主题下用的是 `--synra-primary-deep` 那一支（R48），而**不是**应用的那套暗面 token。
  > 这是"三个消费者"里最容易混的一处，所以写在这里。
- **Q-6 已消失**：包回到 monorepo（§1.3、R1），所以"独立仓库的目录名"与"什么时候建仓"这两个问题
  **都不存在了**。这是这次改动最直接的收益——**一个问题在正确的形状下不是被回答，而是不再存在。**
