# 27 · 官网与文档站

> 本文定义仓库 **`synra-rs/docs`**：官网首页 + 用户文档 + 插件作者文档 + **JSON Schema 托管**。
>
> 域名 **`synra-app.dev`**，部署 **Netlify**，技术栈 **VitePress**（参考实现：本机的 `pmpx-docs`）。
>
> 前置：[README §1.1](./README.md)（与 `design-docs` 的区别）、[15 §2](./15-plugin-sdk.md)（manifest 规范，schema 的来源）。
> 本文同时是**做这个站点的规格书**和**往站点里写内容的规范**。

---

## 1. 它是什么，不是什么

| | 是 | 不是 |
| --- | --- | --- |
| **对象** | 用户、插件作者 | 我们自己（实现者看 [design-docs](./README.md)） |
| **回答** | 怎么装、怎么用、这个参数是什么、插件怎么写 | 为什么这样设计、有哪些取舍、哪里还没定 |
| **内容** | 首页、指南、参考、插件作者手册 | 目标架构、ADR、风险登记、路线图 |
| **语气** | 事实与操作步骤 | 结论 → 理由 → 反例 |
| **对未定项** | ❌ 一律不提（只写**已定**的行为） | ✅ 显式标 🔴 |

> **核心约束**：站点上的每一句话都必须是**已经确定的**行为。
> 设计文档里标 🔴 的东西**不许**出现在站点上 —— 否则用户会把"我们在考虑"当成承诺。

---

## 2. 为什么官网、用户文档、schema 在同一个仓库

三个东西的读者不同（访客 / 用户 / 插件作者），但**必须一起改**：

| 关系                              | 例子                                                              |
| --------------------------------- | ----------------------------------------------------------------- |
| 首页的"30 秒上手"必须与安装指南一致   | 首页写的命令 = `guide/installation.md` 里的命令                       |
| 插件作者手册必须与 manifest schema 一致 | 手册里的字段表 = `public/schema/manifest/v1.json` 的属性             |
| schema 的版本必须与 SDK 版本对应      | `$schema: …/schema/manifest/v1.json` ↔ `synra-plugin-cli verify` 能接受的形状 |
| 站内搜索要能搜到 schema 的字段        | schema 里的 `description` 就是站点的文案                             |

**放两个仓库会必然漂移**：schema 改了但手册忘了改，作者就会按错的手册写、被 `verify` 拒绝，
然后来报 bug —— 而"手册与 schema 不一致"这个 bug 我们自己在 CI 里能抓到（§7）。

**因此**：`docs` 仓库同时是站点、用户文档、schema 的**唯一来源**。
`schema/*.json` 是**手写**的（不是从 Rust 生成），因为站点要给它加 `description` 文案；
CI 反向检查它与 Rust 类型的字段集合一致（§7 守卫 4）。

---

## 3. 设计文档**不上站点**

`design-docs` 的内容**不发布**。三个理由：

| 理由                    | 说明                                                            |
| ----------------------- | --------------------------------------------------------------- |
| **会把内部权衡当承诺**    | "M0 只做 Android、iOS 推迟" 对用户是"iOS 不支持"，但那是排期不是产品定位 |
| **会把未定项当事实**      | 大量 🔴 项；用户读到会以为功能已经存在或永远不会做                    |
| **会暴露安全设计细节**    | 沙箱逃逸面分析、能力闸门的绕过历史 —— 等于给攻击者一份指引            |

**但设计文档与用户文档要能互查**（我们内部需要这个）：

| 方向                 | 做法                                                                     |
| -------------------- | ------------------------------------------------------------------------ |
| 用户文档 → 设计文档   | ❌ 不链（用户不该也没必要读）                                                |
| 设计文档 → 用户文档   | ✅ 链 GitHub URL：`synra-rs/docs` 的 `guide/installation.md`                |

> **为什么不链回去**：链接是双向的路标。用户文档里出现"设计文档"会诱使读者去读，
> 而其中的 🔴 与取舍会造成困惑。想读的人从仓库首页能找到。

---

## 4. 站点结构

```
docs/                                    # 仓库根 = VitePress 源目录
├─ .vitepress/
│  ├─ config.mts                         # 站点配置（nav / sidebar / 双语言 / SEO）
│  ├─ writing-style.md                   # ★ 写作规范（不发布，见 §8）
│  ├─ theme/
│  │  ├─ index.ts                        # 默认主题 + 首页自定义 Layout
│  │  └─ shared/{components,composables,styles}/
│  └─ plugins/                           # 构建期插件（版本号等）
├─ index.md                              # 首页（英文）
├─ zh/index.md                           # 首页（简中）
├─ guide/                                # 英文：指南
│  ├─ introduction.md                    #   Synra 是什么、两种模型
│  ├─ installation.md                    #   三平台安装
│  ├─ quick-start.md                     #   手机 → 电脑 跑通第一条
│  ├─ devices-and-pairing.md             #   发现、配对、在线状态
│  ├─ transfers.md                       #   剪切板与文件
│  ├─ plugins.md                         #   装插件、授权、跨设备同步
│  └─ troubleshooting.md                 #   连不上 / 没反应 / 怎么导出诊断
├─ reference/                            # 英文：参考
│  ├─ capabilities.md                    #   ★ capability 语法（作者最常查）
│  ├─ events.md                          #   ★ 内置事件目录（来自 [04 §6.3](./04-wire-protocol.md)）
│  ├─ errors.md                          #   错误码 + 建议动作
│  ├─ paths.md                           #   数据目录、标准位置
│  └─ platform-matrix.md                 #   各平台支持哪些能力
├─ plugins/                              # 英文：插件作者
│  ├─ getting-started.md                 #   第一个插件（30 分钟跑通）
│  ├─ manifest.md                        #   ★ manifest 字段详解（与 schema 同步）
│  ├─ structure.md                       #   目录结构、构建产物
│  ├─ authoring.md                       #   headless 代码（on_event / execute / can_handle）
│  ├─ ui.md                              #   UI 部分 + @synra/client
│  ├─ testing.md                         #   @synra/testkit 写测试
│  ├─ publishing.md                      #   签名、发布到 registry
│  └─ faq.md
├─ zh/                                   # ★ 与英文结构镜像（§6 有 CI 强制）
│  ├─ guide/ · reference/ · plugins/
│  └─ faq.md
├─ public/
│  ├─ schema/                            # ★ 发布为 https://synra-app.dev/schema/…
│  │  ├─ manifest/v1.json
│  │  ├─ plugin-events/v1.json           #   插件自定义事件的 payload schema 语法
│  │  └─ index.json                      #   可用 schema 的索引（便于工具发现）
│  ├─ brand-mark.svg · favicon.svg · og-image.png
│  ├─ robots.txt
│  └─ CNAME                              #   synra-app.dev
├─ scripts/
│  └─ check-structure.mjs                # ★ 双语言结构一致性校验（§6）
├─ netlify.toml                          # 构建 + 重定向 + 缓存头（§5）
└─ package.json · tsconfig.json · tsconfig.node.json
```

### 4.1 为什么英文在根、简中在 `/zh`

| 方案                        | 评价 |
| --------------------------- | ---- |
| **英文 `/`，简中 `/zh`** ✅   | 站点只有一个"默认语言"的概念（根路径），VitePress 原生支持；SEO 上根路径就是英文 |
| 两者都在 `/en` `/zh`         | 根路径要重定向，多一跳；VitePress 的 `locales.root` 概念被绕过 |
| 只做中文                     | 插件作者社区以英文为主，等于放弃一半读者 |

### 4.2 内容优先级（先写哪些）

站点不必一次写全。**按"读者会在什么时候需要它"排序**：

| 优先级 | 页面                                          | 为什么先写 |
| ------ | --------------------------------------------- | ---------- |
| P0     | 首页、`guide/installation`、`guide/quick-start`  | 没有这三个站点就没有意义 |
| P0     | `plugins/manifest`、`plugins/getting-started`    | 插件作者是核心受众；manifest 是最高频查询 |
| P1     | `reference/capabilities`、`reference/events`     | 作者写代码时反复查 |
| P1     | `plugins/authoring`、`plugins/ui`                | 完整走通一个插件 |
| P2     | `guide/devices-and-pairing`、`guide/transfers`    | 功能说明 |
| P2     | `reference/errors`、`guide/troubleshooting`      | 出问题时才看 |
| P3     | `plugins/testing`、`plugins/publishing`          | 进阶 |
| P3     | `reference/paths`、`reference/platform-matrix`   | 参考 |

---

## 5. 部署（Netlify）

### 5.1 配置

```toml
# netlify.toml
[build]
  command = "pnpm build"
  publish = "dist"

[build.environment]
  NODE_VERSION = "22"

# ── ★ schema 的缓存策略：长缓存 + 不可变，因为 URL 里有版本号（v1）──
[[headers]]
  for = "/schema/*"
  [headers.values]
    Cache-Control = "public, max-age=31536000, immutable"
    Access-Control-Allow-Origin = "*"        # ★ 工具与 IDE 需要跨域拉取
    Content-Type = "application/schema+json; charset=utf-8"

# ── 静态资源长缓存（VitePress 带 hash）──
[[headers]]
  for = "/assets/*"
  [headers.values]
    Cache-Control = "public, max-age=31536000, immutable"

# ── HTML 不缓存（部署后立刻可见）──
[[headers]]
  for = "/*.html"
  [headers.values]
    Cache-Control = "public, max-age=0, must-revalidate"

# ── 安全头 ──
[[headers]]
  for = "/*"
  [headers.values]
    X-Content-Type-Options = "nosniff"
    Referrer-Policy = "strict-origin-when-cross-origin"
    X-Frame-Options = "DENY"
    Permissions-Policy = "geolocation=(), microphone=(), camera=()"

# ── ★ 让 schema 的短路径也稳定：/schema/manifest/v1 → …/v1.json ──
[[redirects]]
  from = "/schema/manifest/v1"
  to = "/schema/manifest/v1.json"
  status = 200

# ── 旧路径兜底（将来若调整结构）──
[[redirects]]
  from = "/docs/*"
  to = "/:splat"
  status = 301
```

### 5.2 域名与 HTTPS

| 项              | 值                                                  |
| --------------- | --------------------------------------------------- |
| 主域名           | `synra-app.dev`                                       |
| `www` 子域       | 301 → 主域（Netlify 的 primary domain 设置）            |
| `CNAME` 文件     | `public/CNAME` 内容为 `synra-app.dev`（防止 Netlify 每次重新分配子域） |
| 证书             | Netlify 自动（Let's Encrypt）                          |
| `_redirects` 冲突 | 只用 `netlify.toml`（两处都有时优先级不同，容易搞混）        |

### 5.3 schema URL 是**公开契约**

`https://synra-app.dev/schema/manifest/v1.json` 被**每个插件的 manifest** 引用（[15 §2](./15-plugin-sdk.md)）。
所以它：

| 规则                                       | 理由 |
| ------------------------------------------ | ---- |
| **URL 里带版本号**（`v1`），且**永不改内容**   | 已经发布的 manifest 会指向它；改了等于改历史 |
| `v2` 是**新文件**，不是 `v1` 的修改            | 旧插件继续用 `v1` 校验                      |
| 长缓存 + `immutable`                        | URL 不可变，缓存可以无限久                    |
| 允许跨域（`Access-Control-Allow-Origin: *`）  | 编辑器插件要在浏览器里拉取                    |
| `index.json` 列出可用版本                     | 工具能自动发现最新版                         |
| **`v1.json` 一旦发布就不删**                   | 删掉会让所有旧插件的 `$schema` 变成 404        |

> ⚠️ **这是一个需要进 CI 的约束**：如果某次改动让 `public/schema/manifest/v1.json` 的内容变了，
> 那就是**破坏性变更**。CI 要拦：对已发布版本只允许改 `description`（文案），不允许改结构。
> 用一份 `schema/.frozen/v1.sha256` 记录结构部分的哈希（§7 守卫 5）。

---

## 6. 双语策略

沿用 `pmpx-docs` 的做法（已被验证有效）：

| 规则                                    | 检查方式 |
| --------------------------------------- | -------- |
| `zh/` 与根目录的页面**一一对应**           | `check-structure.mjs`（缺页直接报错） |
| 标题层级、顺序**完全一致**                 | 同上 |
| 容器（`::: tip` 等）类型与顺序一致，标题翻译 | 同上 |
| 表格行数一致                             | 同上 |
| **代码块内容逐字节一致**（含注释与输出）      | 同上 |
| 中文侧链接带 `/zh` 前缀，**不带锚点**         | 同上 |

**一个 VitePress 1.6.4 的已知限制**（`pmpx-docs` 踩过）：
`locales.zh` 里**不能**为容器标题做本地化（`LocaleSpecificConfig` 没有 `markdown` 字段）。
所以：**每个容器都必须写显式标题**（`::: tip 想先试试？` 而不是裸 `::: tip`），
否则中文页面会渲染出英文的 `TIP`。

**任务清单（`- [ ]`）也不内置**：需要 `markdown-it-task-lists` + 自定义 CSS。

---

## 7. CI 守卫

| # | 守卫                                            | 实现 |
| - | ----------------------------------------------- | ---- |
| 1 | **双语言结构一致**                                | `node scripts/check-structure.mjs`（§6） |
| 2 | **死链检查**                                     | VitePress 构建时的 dead-link 检测（默认开启，失败了要让构建失败） |
| 3 | **schema 是合法 JSON Schema**                     | `ajv` 编译每个 `public/schema/**/*.json`；失败即退出 |
| 4 | **schema 字段集合与 Rust 类型一致**                 | 从 `synra-core` 的 `ts-rs`/schema 导出比对（跨仓库 job，nightly 或 tag 触发） |
| 5 | **已发布 schema 版本的结构未被修改**                | `public/schema/.frozen/v1.sha256`（只对结构部分取哈希，忽略 `description`） |
| 6 | **站点不出现 🔴 / 未定表述**                        | grep 守卫：`🔴`、`尚未拍板`、`待定` 出现在 `guide/` `reference/` `plugins/` 下即失败 |
| 7 | **站点不出现内部设计文档链接**                       | grep 守卫：站内内容不得出现 `design-docs` / `synra-rs/design-docs` |
| 8 | **每个页面的 frontmatter 有效**                     | 构建期校验（`title` 必填） |
| 9 | **代码块里的命令可执行**（可选，P2）                  | 从 `guide/installation` 抽出命令做冒烟（只在 CI 有对应平台时） |
| 10 | **Lighthouse 门禁**（可选，P2）                    | 性能 / 可访问性 / SEO 分数下限 |

### 7.1 守卫 6 为什么重要

站点是**承诺**。一个 🔴 混进去，就变成"官方说要做这个功能"。
所以 CI 直接禁止未定表述出现在面向用户的目录里 —— 这比靠 review 可靠。

```bash
# scripts/ 或 justfile 里
if grep -rn '🔴\|尚未拍板\|待定\|TODO' guide/ reference/ plugins/ zh/; then
  echo "面向用户的页面上出现了未定表述。要么定下来，要么别写。"
  exit 1
fi
```

### 7.2 守卫 4：schema 与 Rust 的一致性

```
synra-core/crates/synra-plugin-api/   （manifest 的 Rust 类型 = 唯一真相源）
        │  导出字段集合（schema 或 ts-rs）
        ▼
docs/public/schema/manifest/v1.json    （手写的 JSON Schema，带 description 文案）
        │  CI 比对：字段名集合、必填集合、类型
        ▼
不一致 → 站点 CI 失败，指出哪个字段多了/少了/类型变了
```

> **为什么手写而不是从 Rust 生成**：站点要给每个字段写**面向作者的文案**
> （"这个字段什么时候用"、"填错了会怎样"）。生成器给不出这些。
> 但**结构**必须与 Rust 一致 —— 所以用手写 + CI 比对，而不是生成。

---

## 8. 写作规范

`docs` 仓库里有一份 `.vitepress/writing-style.md`（**不发布**），写给往站点里写内容的人。
要点（借鉴 `pmpx-docs` 那份，它归纳得很准）：

### 8.1 结构优先于排版

> 大多数"文字太密"的抱怨是**结构问题**，不是行距问题。

| 规则                                                | 判断标准 |
| --------------------------------------------------- | -------- |
| **并列内容拆成 `###`**                                | 如果一节里有四点，每点都以加粗短语开头，那它们是四个小节，不是四段 |
| **一段一个意思**，2–4 句                              | 超过 5 句通常是两段或一个列表 |
| **列举用列表**                                        | 写"第一个是…第二个是…"时，改成列表 |
| **并列事实用表格**                                    | 每项都有同样两三个属性时，属性就是列 |
| **源码手动折行 ~100 列**                              | 排版宽度由 CSS 决定，源码宽度是为了编辑 |

### 8.2 用 VitePress 内置模块，别加自定义样式

| 模块                | 什么时候用 |
| ------------------- | ---------- |
| `::: code-group`     | 两种以上可互换的做法（按平台、按 shell）—— **不是**用来展示顺序步骤 |
| `::: tip`            | 主要捷径，或值得记住的规则。每节最多一个 |
| `::: info`           | 有帮助但非必需的背景 |
| `::: warning`        | 会咬人的东西：数据丢失风险、静默错误行为、有后果的平台差异 |
| `::: danger`         | 破坏性且不可逆 |
| `::: details`        | 打断主线的深层材料：线上格式、完整表格、决策的长推理 |
| `- [ ]` / `- [x]`     | 读者自己确认的清单（本站是非交互的，它是清单不是表单） |

**容器标题必填**（原因见 §6）。

### 8.3 不要动的东西

重排版式时不能改含义：

- 命令、参数、文件名、路径、类型名、数字；
- 代码块里的控制台输出；
- 技术论断；
- 已有的内部链接与目标；
- frontmatter（除非这一页确实需要页面级设置）。

> 如果重排时发现某句话不清楚，**保留原措辞**并在 PR 里提出来，不要猜着改。

### 8.4 版本号不要手写

站点里的当前版本号从 `synra-core` 的 `Cargo.toml` 或发布 API 取，构建期注入：

| 位置                    | 写法 |
| ----------------------- | ---- |
| Markdown（含代码块内）    | `{{synra.version}}` |
| Vue 组件                | `import { version } from 'virtual:synra-version'` |

**代码块内的版本不能靠虚拟模块**（进不去代码块），所以需要两种机制 —— 这是 `pmpx-docs` 的结论，照用。

> ⚠️ **别替换语义不同的版本号**：`synra:plugin@1.0.0`（WIT 版本）、
> `sdk.min: 1.0`（SDK 下限）是**固定事实**，不是当前版本，替换会让它们变错。

---

## 9. 首页要讲清什么

首页是唯一一个"必须让人在 20 秒内明白这是什么"的页面。三件事，按顺序：

| # | 讲什么                        | 怎么讲 |
| - | ----------------------------- | ------ |
| 1 | **它是什么**                   | 一句话：跨设备动作编排框架；两端都是宿主，具体做什么由插件定义 |
| 2 | **它长什么样**（两种模型）       | 分享/接收 + 动作编排（面板）；**配一段可读的示意，而不是截图** |
| 3 | **怎么开始**                   | 一行安装命令 + 一个"30 秒跑通"的三步 |

**参考 `pmpx-docs` 首页的做法**：hero 里放一个**动画终端**，展示"一个命令进去、三个生态出来"。
我们的对应物是：**手机上的一次分享 → 电脑上的浏览器打开**（这正是 [02 §3](./02-domain-and-flows.md) 的主旅程）。

> **首页上的每个技术陈述都必须能在 `guide/` 里找到对应页面。**
> 首页是入口不是内容；读者应该从首页"跳进"文档，而不是在首页读完全部。

---

## 10. 实现清单

仓库骨架：

- [ ] `package.json`（`vitepress` / `vue` / `vue-tsc` / `markdown-it-task-lists` / `@antfu/eslint-config`）
- [ ] `.vitepress/config.mts`（title / description / `cleanUrls` / `lastUpdated` / `outDir` / `srcExclude`）
- [ ] `locales.root` + `locales.zh`，两套 nav / sidebar
- [ ] `transformHead`：canonical + `hreflang` alternates（`en` / `zh-CN` / `x-default`）
- [ ] `sitemap.hostname = https://synra-app.dev`
- [ ] `.vitepress/writing-style.md`（§8）
- [ ] `tsconfig.json`（`strict` / `verbatimModuleSyntax` / `noUnusedLocals`）
- [ ] `public/{CNAME,favicon.svg,brand-mark.svg,robots.txt}`
- [ ] `netlify.toml`（§5.1）
- [ ] `.gitignore`（`node_modules/` `dist/` `.vitepress/cache/` `.vitepress/.temp/`）
- [ ] `scripts/check-structure.mjs`（§6）

内容（按 §4.2 的优先级）：

- [ ] P0：首页 ×2、`guide/installation`、`guide/quick-start`、`plugins/manifest`、`plugins/getting-started`
- [ ] P1：`reference/capabilities`、`reference/events`、`plugins/authoring`、`plugins/ui`
- [ ] P2：`guide/devices-and-pairing`、`guide/transfers`、`reference/errors`、`guide/troubleshooting`
- [ ] P3：`plugins/testing`、`plugins/publishing`、`reference/paths`、`reference/platform-matrix`

schema：

- [ ] `public/schema/manifest/v1.json`（与 [15 §2](./15-plugin-sdk.md) 的字段表逐项对应）
- [ ] `public/schema/plugin-events/v1.json`
- [ ] `public/schema/index.json`
- [ ] `public/schema/.frozen/v1.sha256`

CI：

- [ ] 10 条守卫（§7），至少 1–3、6、7 在 M6 前必须有
- [ ] `netlify.toml` 与 Netlify 站点设置一致的复核（build command / publish dir / 域名）

---

## 11. 待确认（🔴）

| # | 问题                                                        | 倾向 |
| - | ----------------------------------------------------------- | ---- |
| 1 | 是否真的做**双语**（中文优先？还是英文优先、中文后续补）         | 🟡 英文在根、中文 `/zh`；**但先只发布英文**，中文页面结构就位、内容逐步补。若先做中文，`check-structure.mjs` 会因缺页失败 —— 需要在脚本里支持"允许缺失"的过渡模式 |
| 2 | 首页的动画终端用什么实现（`pmpx-docs` 是 Vue 组件 + 打字机）     | 🟡 照搬它的做法（`Home.vue` + `useLocale` + 打字机 composable） |
| 3 | API 参考（capabilities / events / errors）是手写还是从 Rust 生成 | 🔴 未定。手写能加文案但会漂移；生成可保证一致但文案弱。**倾向：字段表手写 + CI 比对**（同 §7.2 的 schema 做法），**错误码表生成**（它没有文案需求，只有 code → 建议动作的映射） |
| 4 | 是否要展示插件目录（marketplace 浏览页）                        | 🔴 未定。需要 registry 有查询 API；倾向 M6+ 再说 |
| 5 | `docs` 仓库与 `design-docs` 的**版本对应关系**                 | 🟡 站点内容跟随**已发布版本**，`main` 分支代表"下一个版本"。发版时打 tag，站点从 tag 构建 |
