# 插件生命周期历史观察

> 归档日期：2026-10-25。保留早期调查与后续更正，不能作为当前实现状态或接口规范。
> 当前契约见 [14](../14-plugin-lifecycle.md)，后续工作见 [19 §0.2](../19-roadmap.md)。

## 已替代的旧约定

以下两句保留用于追溯，不再作为当前契约；任务失败与实例故障已分开，guard 也不提供数据回滚保证。

> **关键**：`Failed` 是一个**真实状态**，不是"回到 Available 并记一条日志"。

> Rust 的 `Drop` + `?` 运算符天然提供这个保证。

## 早期调查与更正

> ★ **2026-10-19 更正：下面这段"生产里没有入口"的诊断已经过期，保留它是为了让下一次调查知道当时缺在哪。**
>
> 今天这条链的两端都通了：`synra_core::install` 被两个宿主共用（守护进程的 `plugin install` 与壳的
> `plugins_install` 都走它），per-capability 同意由 `synra_core::consent` 与 `plugin_grants` 落在库里，
> `synra_core::plugin::load_installed` 在**两个宿主启动时**装载已启用且已同意的插件，壳还会在
> `enable` / `consent` / `update` 之后就地把一个插件装进集合、在 `disable` / `uninstall` 时把它拿出去；
> 守护进程的 `enable` / `disable` / `remove` 也走同一条装载 / 停用路径（[19 §8.142](../19-roadmap.md)）。
> 安全模式也因此第一次对窗口生效（[18 §10](../18-observability.md)）。
> **装载现在也真的会 `activate`**（2026-10-19，见 [19 §8.141](../19-roadmap.md)）：`load_plugin` 在编译之后调用
> 插件的 `lifecycle.activate(config)`（配置里是宿主的语言与主题，见 [19 §8.147](../19-roadmap.md)），
> 失败就把它从集合里拿掉并记录原因；`deactivate` 在停用 / 卸载前调用。
> **仍然没有**的是 `PluginSlot` 那套类型状态机本身：生产路径直接调契约的两个函数，
> 没有经过 `Installed → Activating → Active` 的槽位记录，所以"Faulted"仍只是卡片上的一个原因，不是一个可查询的状态。
>
> 下面这段是 2026-10-18 的原始诊断，逐条证据在那一天都是真的：

> ★ **先说一件最重要的实情，它决定了本文剩下部分的读法：这套生命周期在生产里没有入口。**
>
> 本次逐条核对源码后确认：**一个插件无法在生产中被安装，因此也永远不会被装载。**
> 证据链是闭合的，每一环都单独查过：
>
> 1. `plugins` 表**唯一的写入方**是 `synra-store/src/sqlite/plugins.rs:53` 的 `insert_plugin`，
>    而它的调用者只有 `tests.rs`（两处）。**没有任何生产代码插入一个插件行。**
> 2. 迁移脚本从不种子化一行 —— `INSERT INTO plugins` 在全仓只出现在 `insert_plugin` 里。
> 3. `set_plugin_trusted`（`:118`，用户"我信任它"的那一步）**同样只有测试调用**；
>    它本身还是一条 `UPDATE ... WHERE plugin_id = ?1`，即**只能改已存在的行**。
> 4. app 的 `plugins_enable` → `set_plugin_enabled`（UPDATE），没有行时返回 `PluginNotInstalled`。
>    也就是说界面上的"启用"按钮对不存在的行无能为力，这是对的；**问题是没有东西负责创建那行**。
> 5. 本文 §1 那张 13 状态机的实现 `PluginSlot`（`plugin-host/src/lifecycle/pipeline.rs`）
>    连同它的测试都在，**但没有任何生产驱动者**。
> 6. 于是守护进程 `load_installed`（它要求行满足 `enabled && trusted`）读到的 `db.plugins()`
>    **只能是空的** —— 所以"插件宿主"在端到端意义上是不可达的。
>
> ★ 这解释了此前几轮分别记下的几件事，它们不是各自独立的问题，而是同一个洞的不同侧面：
> `plugin_grants` 没有写入方（没有同意流程就没有决定可记；★ 2026-10-18 起 daemon 侧有了 `plugin consent`）、
> 装载期 verify 管线没有调用者（★ 2026-10-18 起有了：守护进程的 `plugin install` 走它，见 §2.5 末尾）、`session: None` 导致闸门从未运行、
> 审计环没有生产者、以及 app 侧 26 个未注册命令里那一整组 `plugins_*`。
> 也就是说：**插件子系统的每一块都被很好地单独测试了，而它们之间没有一条路。**
>
> ★ 为什么本轮没有补上完整安装路径：它意味着实现一条完整的纵向功能
> （来源解析 → 下载 → 校验 → 同意 → 落盘 → 写库 → 启用 → 装载），
> 其中"下载"还需要一个**全仓都不存在的 HTTP 客户端**（见 [10](../10-capability-and-sandbox.md) §4.4）。
> 记在这里，是因为**本文其余部分描述的是一个尚未接线、但已逐块实现的设计**，读者需要知道这一点。
>
> ★ **但是第一步本轮补上了，而且它是纯增量的：`record_discovered_plugins`。**
> 守护进程启动时扫描 `plugins/`，把**数据库还不认识的**插件目录记成"已安装、未启用、未信任"。
> 这一条通路此前完全不存在 —— `insert_plugin`（`plugins` 表唯一的写入方）的调用者**只有测试**，
> 于是 `db.plugins()` 永远是空，于是 `load_installed` 永远什么都不装载，
> **无论磁盘上放了什么**。
>
> 几个刻意的决定：
> * **只记录、不信任。** 插件目录对以该用户身份运行的任何东西都是可写的，
>   所以"磁盘上有这个目录"不能等于"用户批准了它" —— 那会把文件系统变成权限边界。
>   `should_run` 是 `enabled && trusted`，两个都不写，所以它出现在列表里但不会运行。
> * **一层目录，名字是安装器写的那个。** `@acme/hello` 住在 `plugins/acme+hello/`，
>   目录名来自 `synra_store::sanitise_component` —— 与 `Paths::plugin_dir`、卸载、启动清扫、数据沙箱
>   和 `synra_core::install` 是同一个函数。
>   ★ **2026-10-19 更正：这一条原来写的是 `plugins/@acme/hello/`（两层），而安装器写的是
>   `acme+hello`（一层）。** 装载器与目录扫描都按旧的两层规则找文件，于是**两个宿主都装得上、都列得出来、
>   都加载不了**；测试之所以全绿，是因为夹具用手写的两层路径而不是安装器的输出。现在扫描从 manifest 的
>   `id` 取标识，并要求目录名等于 `sanitise_component(id)`，否则报出"它应该叫哪个名字"而不是记一行永远加载不到的行。
> * **幂等。** 每次启动都会跑；重复插入会撞主键，而每次启动都打印"记录失败"会淹没真正的问题。
> * **纯增量。** 用户已经启用并信任的插件重启后原样保留 ——
>   一个会重插或重置标志的协调逻辑会**每次开机把所有插件静默关掉**，
>   而这种 bug 表现为"我的插件老是自己关掉"。
>   测试 `discovering_does_not_overwrite_a_decision_the_user_has_made` 守这一条。
> * **坏目录要说出来。** 一个既没有 manifest 也没有子目录的目录是"误放的"或"拷了一半的"，
>   用户需要知道自己属于哪一种，所以原因是带回的而不是丢掉的。
>
> 负控制：把插入时的 `false, false` 改成 `true, true`，测试立刻红（"a discovered directory must not be enabled"）。
>
> ★ 仍然缺的下一步：**信任**。`set_plugin_trusted` 依旧只有测试调用，
> 也就是说插件现在能被看见、能被启用，但**没有任何路径能让它变成 `trusted`**，
> 因此仍然不会装载。缺的是本文 §2.4 的 `AwaitingConsent` 那一步：
> 展示它要的能力、记录每一条决定（写 `plugin_grants`）、再置 `trusted`。
> 那一步同时会给 `plugin_grants` 补上它一直缺的写入方。
>
> ★ **那一步本轮补上了：`plugins_consent_respond`（app 侧）。**
> 两处代码：
> * **store 的写入方** `SynraDb::record_grants`（`synra-store/src/sqlite/plugins.rs`）。
>   `plugin_grants` 此前有 schema、有索引、有行类型、有读者，**就是没有写入方** ——
>   全仓每一句 `INSERT INTO plugin_grants` 都在测试里。
>   `decided_ms` 与 `manifest_digest` 是**每条决定**的而不是每插件的，因为两者回答的都是
>   "用户同意的是哪一份文档"：一个升级后多声明了能力的插件，其 digest 不同，
>   于是旧行明显早于磁盘上现在的 manifest —— 这就是让"能力悄悄变多"可被发现而不是隐形的东西。
>   每行 upsert，所以改主意是覆盖而不是堆积历史（历史在审计里，这张表是当前状态）。
> * **那个动作** `plugins_consent_respond(plugin_id, granted)`。
>   规则：**manifest 声明的每一条都被授予，才算 trusted**；拒绝任何一条就不算。
>   ★ 这比设计文档暗示的更严，而且是刻意的 —— 见 [15](../15-plugin-sdk.md) 里对
>   "支持可选性"那一条的更正：`CapabilityDeclaration` 只有 `{ cap, reason }`，
>   **没有 `optional`**，所以没有"作者说少了它也能用"这回事，
>   任何猜测哪条可以不要的规则都是凭空发明。
>   一个缺了声明能力的插件跑起来，正是同意机制存在的意义所在。
> * 无法读取 manifest 时**拒绝**而不是当作"什么都没声明"：
>   后者会记下一组空决定，与"用户全部拒绝"无法区分，还会把插件标成已信任 ——
>   而没有任何东西被展示给任何人。
>
> 负控制两条，都对：把记录改成写空数组 → `granting_every_capability…` 红；
> 把 `trusted` 改成恒为 true → `declining_a_capability…` 红（`right: "awaiting_consent"`）。
>
> ★ **整条链现在通了**：目录被发现（上一轮）→ 出现在列表里且状态是 `awaiting_consent`
> → 用户回应 → 决定写入 `plugin_grants`、`trusted` 置位 → `should_run` 为真 → 装载。
> 路径上**每一个环节此前都是断的**，而现在只剩最后一跳没验证：
> 让一个真的 wasm 插件走完它（受夹具与环境限制，见 [12](../12-wasm-host.md)）。
>
> ★ 一个仍然存在的不一致（记下来而不是藏着）：装载器 `load_one` 仍然用
> `grants_of(&manifest)`，**没有读 `plugin_grants`**。
> 也就是说用户现在可以拒绝某条能力，而装载路径照样按 manifest 授权。
> 这不是可以顺手改的一行：一旦取交集，任何**没有**做过同意的插件（例如手工放进目录、
> 从未打开过设置界面的）授权集就是空的，会全部停止工作。
> 正确的下一步是在装载时**检查**是否存在与当前 `manifest_digest` 匹配的决定，
> 不匹配就拒绝装载并说明原因（"需要重新确认"），而不是静默取交集。
> 那需要一个"拒绝并让用户去确认"的交互闭环，比一行交集大。
>
> ★ **2026-10-19 更正：这一条已经补上，而形状正是上面说的"检查而不是取交集"。**
> `synra_core::plugin::load_plugin` 读 `plugin_grants`：没有任何决定 → 拒绝并说"还没有被用户确认"；
> 任何一条决定的 `manifest_digest` 与磁盘上的字节不一致 → 拒绝并说"需要重新确认"；
> 只有全部一致时才用**用户授予的**那几条建能力集。守护进程与壳现在调的是同一个函数
> （这一段的历史见 [19 §8.136](../19-roadmap.md)）。

---


## 各阶段的原始观察

> ★ **2026-10-18：这一步现在有写入方了（daemon 侧）。**
>
> `Request::PluginConsent { plugin_id, granted }` → `Kernel::plugin_consent`，CLI 是
> `synra plugin consent <id> --allow <cap>...`。它做四件事，每件都有理由：
>
> * **读磁盘上的 manifest** —— 决定是针对**一份文档**做的，所以摘要从当前字节算，而不是从数据库里抄；
>   更新后摘要变了，那些 row 会显得比 manifest 旧，装载时就会要求重新确认（见 §2.6）。
> * **granted ⊆ 声明** —— 点名了一个插件没要的能力就拒（`InvalidArgument`）。通常是笔误，而若不是笔误，
>   就是客户端在替用户扩大决定。
> * **每个声明写一行**（允许/拒绝），不是只写允许的：缺一行等于"没有决定"，而装载器要的是每一条都有决定。
> * **trusted = 至少允许一条** —— 全拒不是批准。`should_run` 是 `enabled && trusted`，所以全拒之后即使再来一次
>   `enable` 也不会运行，这正是设计里"完全拒绝"应有的样子。
>
> 仍然缺的：**升级差异**（`ConsentDiff`）没有实现 —— 现在重新确认是全量重来，不是只问新增的那几条。
>
> ★ **2026-10-18（第二轮）：规则搬进内核了，因为它有两个答案。**
>
> 上一轮之后发现：app 的 `plugins_consent_respond` 规定"**每一条**声明都授予了才算 trusted"，而 daemon 是
> "**至少一条**"。装载器两边都照已授予的行建能力集，于是**同一个用户决定**在一个宿主里得到一个跑不起来的插件、
> 在另一个里得到一个带着子集运行的插件 —— 而用户看到的只有一个开关。
>
> 现在规则在 `synra_core::consent::decide`（纯函数，无 IO）：每个声明一行、granted ⊆ 声明、摘要取当前字节、
> **trusted = 至少允许一条**。选"部分同意可以运行"是因为它正是 `Consent::is_partial()` 与能力闸门存在的原因：
> 授予集合是权威，没授予的能力在闸门被拒，而不是让整个插件不跑 —— 后者会让用户看到自己"同意了两条"的插件
> 永远不工作，且没有任何设置解释原因。
>
> 两个宿主现在都只问这个函数，`plugin_consent` / `plugins_consent_respond` 各自只剩"把决定写进库"。
> 断言在两处：`partial_consent_loads_the_plugin_with_only_what_was_allowed`（daemon，检查装载器拿到的
> 能力集里**没有**被拒的那条）与 `declining_one_capability_records_the_denial_and_still_trusts`（app）。


> ★ **2026-10-18：上面第 1–5 步有了第一个实现，而它的几处与伪代码不同。**
>
> 实现是 `synra-daemon/src/runtime/install.rs`（`from_directory`），入口是控制通道的
> `Request::PluginInstall` → `Kernel::plugin_install` —— 也就是 `synra plugin install <目录>` 第一次真的做事。
> 差异每一条都是刻意的：
>
> * **没有 `<version>` 这一层**：现存布局是 `plugins/<plugin-id>/`（`load_one` 与 `record_discovered_plugins` 都按
>   这条规则解析目录），换成带版本号的目录是另一件事，不该被"接上安装路径"顺手改掉。
> * **staging 在叶子旁边**：`plugins/@acme/.hello.installing-<ts>` → rename 成 `plugins/@acme/hello`。临时名只能用
>   **叶子**（`hello`）而不是 id（`@acme/hello`）—— 后者会拼出两级路径，第一版就是错的，测试当场报 `os error 3`。
> * **先读字节再写**：每个 artifact 只读一次，哈希与落盘用同一份字节。读两次（一次算摘要、一次复制）会留下一个
>   窗口，写下去的是没人校验过的字节。
> * **没有 `source.json` / `.installed` 标记**：来源与时间在 `plugins` 行上（时间戳由 `insert_plugin` 写），
>   再写一份文件就是同一事实的第二个作者。
> * **失败即回滚**：目录已就位而写库失败时删掉目录。反过来会让一行指向从未写下的文件，
>   `load_installed` 会把它报成"插件坏了"，而不是"安装失败了"。

> ★ **2026-10-19：卸载也有 CLI 入口了。** `Request::PluginRemove { plugin_id, remove_data }` →
> `Kernel::plugin_remove`：守护进程先用 `PluginFiles::size_of` 量字节（artifact，加 `--data` 时的 sandbox），删行，
> 再删目录，答案带 `bytesFreed` 与 `dataKept`。CLI `synra plugin remove <id> [--data]` 因此从"解析了但什么都没做"变成
> 真的卸载；prose 输出一句话而不是 JSON 字段名。默认保留数据这一点没变 —— 与窗口的对话框是同一个默认值。
> * **落点是 `enabled=false, trusted=false`**：第 5 步写的是"安装"，不是"批准"。同意仍然是 `plugins_consent_respond`
>   （app 侧）与 `plugin enable` 那两步。
>
> 仍然没有的：`PluginSlot` 仍然没有驱动者（状态机与数据库行是同一件事的两份描述，尚未合一）、
> registry 安装（没有 HTTP 客户端）、以及 app 侧的 `plugins_install` 命令。
>
> ★ **2026-10-18（第二轮）：第二条门也查签名了。**
>
> `plugins/` 有两条入口：`plugin install`（跑完 11 步）与**手工放进去一个目录**（启动时的
> `record_discovered_plugins`，作者开发时就是这么做的）。后者原来"有 manifest 就记成已安装"，于是
> **一个被换掉 `plugin.wasm`、签名是真的但覆盖的是别的字节的包，会变成"已安装插件"** —— 用户随后看到的同意提示，
> 是关于一份没人签过的文档。现在扫描时若目录里有 `signature.json`，就验签 + 比对 subject 的摘要；
> 没有签名的目录仍然照记（那是作者的本地构建），只是它照样是 untrusted，同意仍是闸门。
>
> 代价是一次扫描每个新目录多几个哈希、不编译；收益是"签了名的包不能声称自己是另一个包"。


> ★ **2026-10-18：卸载的阻塞点不是 UI，是删除职责；而上面第 6 步的路径写错了。**
>
> **路径。** 实现的布局是 `plugins/<id>/`（artifact）与 **`plugin-data/<sanitised-id>/`**（用户数据）**两个目录**，
> 不是"`<data>/plugins/<id>/` 里面还装着用户数据"。`Paths::plugins_dir` 与 `Paths::plugin_data_dir` 是两个成员，
> `plugin_data_dir_for` 会对 id 做 sanitise。所以"删插件、留数据"在布局上成立 —— 这正是第 6 步要的默认值。
>
> **删除职责。** 结构性守卫 `only the sandbox may delete a directory tree` 只允许
> `crates/synra-store/src/sandbox.rs` 调 `remove_dir_all`，而 `plugins/<id>/` 不属于沙箱（沙箱管的是 plugin-data）。
> 于是卸载卡在**所有权**上而不是缺一个函数：要删插件目录，必须先有一个明确命名的所有者。
>
> **决定。** 在 `synra-store` 里新增 `PluginFiles`，与 `Sandbox` 并列，只做一件事：删掉某个插件 id 的 artifact
> 目录，以及（可选）它的 data 目录。守卫的允许名单加这一条，理由写在同一行。三条约束：
>
> * id 先 `sanitise_component`，再断言结果仍在 `plugins_dir` 之下（`is_within` 已存在）；
> * 只认这两个目录，不接受任意路径；
> * data 目录**只在 `remove_data` 为真时**删，默认保留 —— 第 6 步的"询问"就是它的默认值。
>
> 之后三步（store 的 `remove_plugin`、命令 `plugins_uninstall`、页面按钮）按这个形状写。
>
> **四步均已落地**（2026-10-18）：`crates/synra-store/src/plugin_files.rs`（含三条测试）、
> `SynraDb::remove_plugin`（事务，含一条断言两张表都干净）、`commands::plugins::plugins_uninstall`
> （先记录后目录，含两条）、页面上的 `Remove` + `APopconfirm`。
> ★ **2026-10-19：那一行确认换成了一个能问的对话框。** 页面不再用 `APopconfirm`（它的确认句是“数据留在磁盘上”，
> 所以 `remove_data` 恒为 `false`），改成 `AModal` + “Also delete its data” 勾选：不勾是保留（默认、可逆），勾了才把
> `remove_data: true` 传给命令。对话框用 `get-container="false"` 留在组件里，测试才看得见那个勾选框；每次打开都把勾选
> 重置为“不删”，记住上次的破坏性选择等于第二次访问默认删数据。
> ★ **对话框显示数据目录路径**：`PluginSnapshot` 增加了 `data_path`，壳用 `Paths::plugin_sandbox_dir` 构造它 ——
> 与卸载本身同一个函数，否则“数据在哪”会有两个答案。
> ★ **2026-10-19：`RemoveReport` 回来了。** 契约里这条命令一直是 `One`，而壳返回 `()`、界面按 `null` 读 —— 声明与实现
> 差了一个形状，而且没有任何守卫检查返回值。现在 `plugins_uninstall` 返回 `RemoveReport{bytes_freed, data_kept,
> leftovers}`：字节数在删除**之前**量（artifact 目录 + `remove_data` 为真时的 sandbox），`data_kept` 是
> `!remove_data`，`leftovers` 恒空 —— `PluginFiles::remove` 要么整棵删掉要么拒绝，没有“删了一半”的状态。
> 页面把 `bytes_freed` 格式化后放进报告句（`removedKept`/`removedDeleted`），`formatBytes` 同时从 HomePage 抽到
> `src/format.ts`，两个页面共用一个实现。
> ★ 这一节的两项（删除前询问、删除后报告）都已闭环。
> ★ **同日更晚：`leftovers` 不再恒空（19 §8.107）。** `PluginFiles::remove` 改为**尽力删除**：一个目录删不掉
> （Windows 上被进程打开、权限拒绝）不再让整个卸载失败 —— 数据库行已经删了，拒绝只会让文件留在磁盘上而且没人知道。
> 它返回删不掉的路径，两个宿主都把它放进 `RemoveReport.leftovers`；CLI 的句子列出路径，插件页报告条数。
> ★ **同日更晚：`bytes_freed` 也只算真的走了的（19 §8.108）。** 删除前量的字节数会把留下的目录算成“已释放”；现在
> `PluginFiles::remove` 返回 `Removal { bytes_freed, leftovers }`，字节数是删除前减去 leftovers 当前大小。锁定文件的
> 用例断言 `bytes_freed == 0`，关掉句柄后第二次删除才变成 4。
> ★ **同日更晚：留下的目录在下一次启动被清扫（19 §8.109）。** `PluginFiles::sweep_orphans(installed)` 遍历 artifact 根，
> 删掉任何不属于已安装插件的目录；sandbox 与散落文件不动。两个宿主在启动时、加载插件之前调用它。
