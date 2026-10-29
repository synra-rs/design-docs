# 2026-10-29 · 插件定位与权限/注册表流程

应用：`5581374`，`fix: make plugin management flows reliable`，GPG签名、42字符、无body，已推送main。
依据：[专业软件完成度](../ui-design/00-product-quality-bar.md)、[插件管理设计](../ui-design/07-plugins.md)。

## 1. 本批行为

通知的 `?plugin=` 原来从 `window.location.search` 读取，遗漏生产hash路由内的查询；清除筛选还直接改写history。
现在通过Vue Router定位完整ID，与关键词筛选分开；支持同页连续跳转、对象已移除和返回列表，保留其他参数、筛选草稿和焦点。
管理区域仍就地展开，不为布局模板引入Drawer。

权限待处理入口依据内核状态，不根据声明数量或范围集合推断；新增disabled、awaiting_consent、unreadable本地化。
表单仅比较精确编辑值是否改变，不代替内核的能力范围判定；已有决定不重复显示保存按钮，外部更新刷新干净草稿，用户未保存修改保留。
权限提交按pluginId隔离，等待期间锁定对应选择；修正勾选区居中和保存按钮占满一行的问题。

注册表读取状态提取到 `plugins/catalog.ts`：修改搜索词/排序只使旧结果失效，不按键请求；每个页面最多一个在途读取。
分页按服务端返回行数推进offset，显示去重不改变游标；失败保留已读行，重试绑定失败的搜索/分页操作。
区分有结果、无匹配、读取失败；错误放在查找区域，不再在有结果时追加“没有匹配”。

预览请求绑定页面、来源、输入与区域打开状态；输入变化或离开后旧响应不再弹窗，取消不会安装。
实际安装时锁定来源/输入，避免成功结果清空新的草稿；输入法确认Enter不提交搜索或安装。
预览仍只是发布者元数据声明，下载字节后才验签；未新增锁定注册表版本的IPC，也没有声称预览就是已验签。
发布者决定失败保留提示与完整指纹，成功才关闭；没有隐式信任或授予权限。

## 2. 验证

- 先运行类型检查、受影响文件lint与定向测试；新增竞态测试使用真实KernelClient入口和可控bridge响应。
- 真实hash history挂载验证精确ID、同页查询变更、缺失对象、保留参数、返回与焦点；原先只用普通URL参数的测试被替换。
- 组件测试覆盖预览在输入/来源/关闭/卸载后的失效、发布者拒绝重试、权限草稿更新与保留、输入法Enter。
- Catalog测试覆盖晚到结果/拒绝、单请求上限、重复结果游标、分页失败与首屏重试、页面释放。
- 阶段门禁：**43文件、392项前端测试通过**，全量ESLint、生产构建、8项shapes通过；47命令、67调用点、58Native wrapper未变化。
- `CARGO_BUILD_JOBS=1 cargo build --locked -j1 --manifest-path src-tauri/Cargo.toml` 成功，27.38秒；更新开发EXE，未改Rust源码、未重复未变的Rust单测。

日志：工作区 `.scratch2/ui-phase-b-{all-tests,lint,build,shapes,native}.log`。
已有生产块大小警告仍在，主JS840.97kB/gzip262.35kB；Cargo仍报告bin/lib PDB同名警告，没有修改门禁阈值。
构建后即清理两个无用staticlib路径，保留EXE/PDB/增量依赖；实际回收约2.12GiB，不重复计算硬链接，D盘可用约4.92GiB。

## 3. 实际界面与边界

使用本地假内核页面、真实Ant Design Vue组件和browser-skill检查通知定位、管理自动展开、缺失对象返回、权限对齐与重复授权按钮。
390px下实际主区clientWidth/scrollWidth均为390，没有横向溢出；桌面视口1904×855。浏览器移动模拟已恢复，会话已关闭。
实测发现目标缺失时AList还会补一个默认空状态，已改为有对象才渲染列表；最终图不包含这一重复提示。

- [桌面定位页](./2026-10-29-plugin-management-flow/plugin-target-desktop.png)
- [390px定位页](./2026-10-29-plugin-management-flow/plugin-target-mobile.png)

浏览器未执行真实安装、权限变更或信任决策；没有声称原生窗口真实交互、多语言全状态或完整无障碍验收完成。
当前审计读取/清除仍需隔离与恢复处理，其他按对象管理任务、设备配对和设置组织继续见[当前进展](../ui-design/11-current-progress.md)。
