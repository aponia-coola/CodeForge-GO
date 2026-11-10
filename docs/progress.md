# CodeForge 工作进度

> 跟踪 [CodeForge-Go-项目开发计划书.md](architecture/CodeForge-Go-项目开发计划书.md)（15 项功能路线图）的实施进度。
> 完成一项即在此标记 ✅；进行中标记 🔄；被打断/暂停标记 ⏸️。

## 🔄 实施顺序与状态

**批次 0：公共前置**
- [x] ctx 注入（`tools.WithSession`：sessionID/step；`tools.WithTaskSink`）
- [x] WithCheckpointSink（随 #4 做完：写工具执行**前**上报旧内容，Agent 侧按 (会话,步骤,路径) 去重落库）
- [ ] 补充：WithNoteSink（随 #5 后台任务一起做）

**第一批**
- [x] #1 联网 search/fetch（web_fetch / web_search + SSRF 防护 + Tavily/SearxNG，config: web.enabled 开启）
- [x] #2 任务清单 todo（todo_write 工具 + session_todos 表 + System Prompt 注入 + 前端折叠清单栏）

**第二批**
- [ ] #5 后台任务
- [ ] #6 记忆体系化
- [x] #4 检查点 / 回滚（含「编辑最近 3 条用户消息并重发」）

**第三批**
- [ ] #3 Hooks / 定时
- [ ] #11 Lint 反馈 / 编辑器上下文

**第四批**
- [ ] #7 确定性工作流
- [ ] #9 子智能体续聊
- [ ] #15 预算与成本

**第五批**
- [ ] #8 Worktree 隔离
- [ ] #12 多模态 / 富文档
- [ ] #13 斜杠命令 / 输出样式 / 技能生态
- [ ] #14 细粒度权限 / 沙箱

## 💾 检查点 / 回滚 + 编辑重发（2026-09-18，Plan.md #4）

**需求**：给最新 3 条用户消息加「编辑」按钮 → 点开把该条内容回填输入框并露出「发送/取消」→ 发送时从该条消息处拼接上下文、**丢弃其后全部对话**、**内部回退 context window 但不动左下角窗口统计**、重跑请求、新回复**覆盖**下方原有内容、并**回退该条消息之后产生的所有文件改动**。用户明确「要实现此功能必须先有回退点（检查点）」，「更改」范围按**步骤精确回滚**。

### 后端

- **公共前置**：`tools.WithCheckpointSink` / `CheckpointSinkFrom` + `CheckpointEvent{Path, Existed, OldContent}`（[session.go](../pkg/tools/session.go)）。负载必须带 `OldContent` 而非只给路径——上报发生在**写入之前**，实现方一旦延后读取，读到的已是覆盖后的内容，快照失去意义
- **写工具上报**：`FS.snapshot` 签名加 `ctx`，在写 undo 栈后上报 sink；`WriteFile` / `EditFile` / `DeleteFile` 三处调用点同步传 ctx（[file_ops.go](../pkg/tools/builtin/file_ops.go)）
- **落库**：`checkpoints` 表（`PRIMARY KEY(session_id, step, path)` + `ON DELETE CASCADE` + `idx_checkpoints_session`），`INSERT OR IGNORE` 让数据库承担去重——同一步骤内同一文件只留**最早**那份（回滚只关心「这一步之前文件长什么样」，中间态无意义）
- **Store CRUD**：[checkpoints.go](../pkg/store/checkpoints.go)——`InsertCheckpoint` / `ListCheckpoints` / `ListCheckpointSteps`（`GROUP BY step` 聚合）/ `CheckpointsFrom`（**含边界**、倒序供逐条回放）/ `DeleteCheckpointsFrom`
- **Agent 侧**：[checkpoints.go](../pkg/agent/checkpoints.go)——`recordCheckpoint`（失败只记日志，不打断工具执行）、`CheckpointSteps`、`RewindFiles`（逐条**倒序**恢复，同文件跨多步最终落到最早旧内容；全部成功才清理检查点，有失败项则保留供重试）、`restoreFile`（`Existed=true` → `MkdirAll`+`WriteFile`；`false` → `os.Remove` 并容忍 `IsNotExist`）、`RewindAfterEdit`
- **循环注入**：`runLoopWithLimit` 工具执行处 `toolCtx` 挂 sink；`persist=false` 的临时子智能体循环**不记录**（不落历史，回滚点无从对应）
- **`EditAndResend(ctx, sessionID, back, newText, emit)`**（[agent.go](../pkg/agent/agent.go)）：定位第 `back` 条纯文本用户消息 → 截断到该条（含）→ **复位压缩态**（`compressedUpTo`/`summaryText`）→ **刻意不动 `usageIn/usageHit/usageOut`** → 替换文本块（图片等其它块保留）→ 重跑循环
  - ⚠️ **「内部回退但不动左下角统计」的精确落点**：内部值 = `compressedUpTo` + `summaryText`；左下角统计 = 由 `usage*` 累计量算出。回退历史不等于这些 token 没花过，抹掉就是伪造账目 → 专门的测试 `TestEditResendKeepsUsageCounters` 钉住这条约束
- **辅助**：`nthLastPlainUserIndex(msgs, n)`（n 从 0 起，越界 -1）、`UserMessageRef{Index, Back, Text}`、`EditableUserMessages(sessionID, n)`——只认 `isPlainUserText`（**唯一安全的切点**，不会在 tool_use / tool_result 之间切开）
- **`RewindAfterEdit` 步骤映射推定**：检查点归属「ReAct 步骤」、消息归属顺序，两者无直接映射。用「提问 → (回复 + 工具调用/结果)* 」的同构关系，取保守口径 `from = maxStep - (dropped+1)/2`——**宁可多退**（用户能重跑）也不**少退**（留下与上下文矛盾的半成品）；无法定位目标消息时返回 `(nil, nil)`，只截断对话不动文件
- **WS**（[ws_handler.go](../pkg/server/ws_handler.go)）：上行新增 `back` / `to_step` / `rollback_files`；新增 `edit_user_message`（先 `RewindAfterEdit` → 下发 `rewind` → `EditAndResend` → 下发 `edit`）、`rewind`、`checkpoints` 三分支；新增 `checkpointEvent()` 一次性下发 `{steps, editables}`，在 `new_session` / `load_session` / 每轮 idle 后自动推送
- **REST**：`GET /api/sessions/rewind?id=` 列检查点与可编辑消息；`POST /api/sessions/rewind {id, to_step}` 执行回滚（缺 id → 400，其他方法 → 405）

### 前端

- 用户气泡 `dataset.rawText` 存**原文**——编辑按钮必须按原文与白名单对号，不能用 `textContent` 反推（`@路径` 在渲染时会被缩成文件名）
- **「哪几条能编辑」的唯一真相源在服务端**（`editables` 随 `checkpoints` 事件下发）：规则含「最近 N 条 + 必须纯文本」，两边各算一份必然漂移。`back` 用**相对距离**而非绝对下标，避免渲染顺序与 `Messages` 下标不一致时改错消息
- 编辑态状态机：`startEditMessage` / `exitEditMode` / `submitEdit` / `sendEditRequest` / `truncateAfterEditableMessage`；`submitMessage` 首行按 `isEditing()` 分流
- 即时反馈：`sendEditRequest` 在 `wsSend` **之前**先 `truncateAfterEditableMessage(back)` 清掉界面「该条之后」的内容，随后服务端 `edit` 事件再 `beginEditedView` 做权威重建
- `busy` 撤按钮 / `idle` 重挂；切会话（`replayHistory`）清空白名单；`describeRewind` 用人话区分「还原 N 个 / 删除 N 个 / M 个失败」，无改动时返回「没有需要回退的文件改动」

### 测试与验证

- 新增 `pkg/store/checkpoints_test.go`（11 例：去重留最早、同路径跨步、聚合、`CheckpointsFrom` 含边界、会话隔离、级联删除）
- 新增 `pkg/agent/checkpoints_test.go`（约 24 例，含 Plan.md 验收「**改坏 3 个文件 → 回滚两步前全部还原**」、不清零 usage、非文本块保留、越界 back 拒绝、压缩态复位）
- 新增 `pkg/server/rewind_handler_test.go`（5 例 REST 全链路）
- **脚手架缺口修复**：`server_test.go` 的 `newTestDepsAtProvider` 原先漏了 `ag.SetMemoryStore(st)`（生产在 `cmd/agent/main.go:192` 有），导致回滚接口测出「存储未就绪」
- 前端断言 304 → **324**（新增「编辑并重发用户消息」14 项 +「检查点 / 回滚」6 项）
- `go build ./...` / `go vet ./pkg/...` 干净；**`go test ./...` 全部通过且零失败**（早先记录的 Windows `cat` 环境失败本轮未复现）；5 个产物已重编（windows-amd64 / linux-amd64 / linux-arm64 / android-arm64 / 本机入口）
- ⏸️ **前端 UI 未做自动截图验证**：agent-browser 不支持 Windows，靠 `web/test/` 断言 + 手动验证兜底
- ⬜ **未做**：Plan.md 提到的独立「回滚菜单 + 二次确认」入口（当前回滚经编辑重发联动触发）、可选的 `restore_checkpoint` 工具

## 🐛 三个前端故障修复（2026-09-18，用户反馈）

### 1. 自定义背景图未生效

**根因**：`#bg-layer` 是 `<body>` 的**子节点**、靠 `z-index:-1` 沉底，但 `html, body` 共用规则里挂了 `background: var(--bg)`（不透明）。CSS 绘制顺序是「元素自身背景 → 负 z 后代」，所以图层被 body 自己的不透明底色永久遮住。

**修复**：不透明底色**只挂 `html`**，`body` 改 `transparent`。`#bg-layer` 保持 `z-index: -1`。
⚠️ **两个条件缺一不可**，详见下方「z-index 反复」一节。

### 1b. ⚠️ 我在这一步引入了回归又修回来（2026-09-19）

第一版修复里我把 `#bg-layer` 的 `z-index` 从 `-1` 改成了 `0`，理由是「靠 DOM 顺序垫底」——
**这个理由是错的**，而且造成了新故障：**侧栏底色被背景图铺满**。

原因（CSS 绘制顺序）：
```
根背景 → 负 z 层 → 在流块背景 → 浮动 → 行内 → z-index:0/auto 定位元素 → 正 z
```
`#app` 不形成层叠上下文、`#sidebar` 是非定位的在流块，所以 `z-index:0` 的 `#bg-layer`
绘制顺序**晚于**侧栏背景 → 盖住它。

**最阴的地方**：`getComputedStyle(#sidebar).backgroundColor` 仍返回不透明的 `rgb(247,248,250)`，
**计算样式完全看不出被盖住**。我因此两次向你误报「侧栏是透明的（可能是液体玻璃皮肤）」——
其实那是我的回归。**只有看像素才能发现**（裁剪截图 + `getImageData` 采样）。

**最终正确组合**：`html` 不透明底色 + `body` 透明 + `#bg-layer { z-index: -1 }`
→ 层次为「html 底色 → 背景图 → 各面板底色 → 内容」。

**教训**：计算样式 ≠ 实际渲染。涉及层叠/遮挡的问题必须**看像素**。

### 2. 任务清单在界面上看不见（用户反馈「其他 AI 修过仍不可见」）

**根因**：`renderTodoBar` 用 `messagesEl.prepend(todoBar)` 把面板插进**滚动容器内部顶部**。会话一长、用户往上一滚，面板就滚出可视区 —— 表现成「模型报『已更新任务列表』但界面空白」。次因：列表为空时 `classList.add('hidden')` 直接隐藏，用户分不清「没有清单」与「面板消失」。

**修复**：
- DOM 落位改为 `#composer-wrap` 内、`#composer` **之前**（直接写在 index.html 里，不用 order 反转）。在 `#composer` 之外 → 不参与输入卡片的折叠/居中 transform，永远留在原地；在 `#composer-wrap` 之内 → 随卡片浮动、宽度对齐
- **默认关闭**：清单为空时整个面板 `hidden`（`display:none`，连 margin 一起收起，不在输入卡片上方留空隙）。清单非空 ⟺ AI 真的调用过任务清单 —— 服务端推来的 `todos` 只有两种情况：① AI 调了 `todo_write`；② 载入的会话里存着上次的清单，两种都该显示
- HTML 初始即带 `hidden`，避免 JS 执行前闪一下空壳面板
- **宽度 `100%`**：父级 `#composer-wrap` 已经是 `min(80%, --composer-max-w)`，子级**不能**再套一次 —— 会得到「80% 的 760px ≈ 608px」，比输入卡片窄 152px（由运行时验证抓出，见下）
- 默认展开，折叠状态存 `localStorage['cf_todo_collapsed']`（收起一次就不该被强行展开）
- 折叠只切类、不重建内容（箭头交给 CSS `::before`）
- `max-height: 38vh` + `.todo-body` 内部滚动，条数多也不把输入卡片顶出屏幕
- 显隐都会重算消息区底部留白（面板出现/消失会改变输入卡片高度）

**注**：曾短暂做过「空清单也显示『暂无任务』占位」的版本，按用户要求改回默认关闭。移除 `TODO_EMPTY_HINT` 常量与 `.todo-empty` 样式，不留死代码。

### 3. 主 agentloop 期间无法向上滑动查看历史

**三个叠加根因**：
1. `scrollBottom()` **无条件**滚底 —— 模型流式吐字时每来一小段就把视口拽回底部，用户刚滚上去就被拉回
2. `#messages` 的 `padding-bottom` 写死 `200px`，而输入卡片是**绝对定位浮在它上面**的、不占布局空间。卡片高度是变数（任务清单展开、输入框长高、底栏换行），常量必然对不上 → 留白不足时最后几条永远被盖住，滚到底也看不见
3. 折叠逻辑的反弹分支与内容增长叠加，加剧回拽

**修复**：
- 引入 **`followTail`**：`scrollBottom(force)` 只在跟随态滚底；scroll 回调里 `followTail = isNearBottom()`，用**是否贴近底部**判定而**不看滚动方向** —— 方向判定会被「内容增长把 scrollTop 顶大」误认成用户向下滚
- 发送消息 / 回放历史 / 编辑重建视图 / 清空视图（`syncComposerMode` 的 `empty` 分支，一处覆盖新建·归档·删除全部路径）均恢复 `followTail = true`
- 底部留白改**动态**：`syncComposerPadding()` 实测 `#composer-wrap.offsetHeight` + 7vh 写进 CSS 变量 `--composer-pad-bottom`（`#messages` padding 用它，兜底 200px）；`ResizeObserver` 跟随卡片高度，任务清单折叠后再 `requestAnimationFrame` 重算
- `#messages` 加 `scroll-behavior: auto`，避免 CSS 平滑滚动与 JS 跟随态滚底打架
- 右下角新增 `#to-bottom` 圆形箭头（`#main` 内绝对定位、`bottom:118px` 让开输入卡片），仅在不在底部时 `.show`；隐藏时 `pointer-events:none` 不挡内容

**验证**：前端断言 324 → **351**（新增「滚动：跟随态与回到底部按钮」9 项 +「自定义背景图可见性」6 项，todo 组扩为 12 项 + 底部留白 3 项）；`go test ./...` 全绿；5 产物 + 本机入口已重编，实测新二进制内嵌资源与 `/`、`/ui.js`、`/app.css` 响应均为新内容。

**运行时验证（新增，2026-09-19）**：单测只锁代码形状，证明不了「真的画出来了」。新增 `test/verify_ui_fixes.mjs`（CDP 驱动无头 Chrome，零依赖）在真实浏览器里量几何与计算样式，**23 项全过**：
- 背景图：body 背景 `rgba(0,0,0,0)`、html `rgb(255,255,255)`、图层 `z-index:0` + 是 body 首个子节点
- 任务清单：默认 `display:none` 且 0 行；派发 `todo` 事件后 `display:flex`、3 行、计数 `1/3`、**整体在 `#composer` 上方 8px**、**宽度 760px vs 760px**；清空后回到 `display:none`
- 滚动：滚到顶 `scrollTop=0` 不被拽回、内容继续增长仍停在原位、箭头出现→点击→回到底部 `gap=0px`→箭头自动收起

⚠️ **这次运行时验证抓出一个真 bug**：`.todo-bar` 曾写 `width: min(80%, --composer-max-w)`，而父级 `#composer-wrap` 已是该宽度 → 子级再乘 80% = 608px，比输入卡片窄 152px。已改为 `width: 100%`。
⚠️ **同时暴露了我自己写的比较基准错误**：判断「面板在输入卡片上方」时拿 `#composer-wrap` 当基准必然失败（面板就在 wrap 内部，wrap 的 top 天然在面板之上），必须比 `#composer`。**几何断言要挑对基准元素**，否则会把测试自身的错当成代码的 bug。

⚠️ 旧断言按新意图改写两次：「空清单自动隐藏」→（先改）「显示暂无任务占位」→（用户要求默认关闭，再改回）「空清单整个面板隐藏」；「居中态不参与滚动折叠」→ 拆成「折叠仍提前返回」+「但跟随态/箭头判定必须在提前返回**之前**」；「面板宽度 min(80%…)」→「width:100% 且不得再套 min(80%)」。

## 🐛 选完背景图「啥也没有」——根因与修复（2026-09-19，用户二次反馈）

**现象**：点「选择图片」→ 系统对话框正常 → 选好图 → **界面毫无反应，也不报错**。

**根因**：`handleAppearancePick` 的 **Windows 分支只把路径回给前端，没有写 `cfg.Appearance.BackgroundPath`**：
```go
path, err := runPSDialog(...)   // 弹出对话框、拿到用户选的路径
s.finishPick(w, path, err, "背景图")   // ← finishPick 只回 {ok, path}，不落库
```
而 `/api/appearance/background` 读的正是这个字段：
```go
p := s.cfg.Appearance.BackgroundPath
if p == "" { w.WriteHeader(http.StatusNotFound); return }   // 永远 404
```
于是前端 `applyBgImage(true)` 把图层 `.on` 了、URL 也挂上了，但服务端 404 → **无图可画、无错可报**。
Termux 分支当年顺手写了 `cfg`，Windows 分支漏了，所以故障只在 Windows 暴露。
前端注释「Windows/安卓已由服务端落库」也是错的。

**为什么之前没发现**：上一轮我用 `POST /api/appearance` 直接设路径来验证渲染，那条路**本来就落库**，
绕过了有问题的选图分支 —— 测的是「渲染对不对」，而坏的是「选完有没有存」。**验证路径必须覆盖用户真实操作链**。

**修复**：
- 新增 `finishBackgroundPick`：校验 → **落库（写 cfg + local.yaml）** → 响应 `{ok, path, background:true}`；
  Windows 与 Termux 两个分支统一走它，杜绝「某平台忘了写」
- 抽出 `validatePick` 供 `finishPick` / `finishBackgroundPick` 共用（对话框失败 / 非法 UTF-8 的判定单一来源）
- `saveLocalYAML` 改为返回 `bool`，失败时调用方立即 return（原先会二次写 body，产出拼接的坏 JSON）
- **前端加兜底**：服务端返回 `background:true` 才直接生效；否则自己补一次 `POST /api/appearance`，
  且**必须等保存完成再 `applyBgImage`**（否则请求赶在落库前发出，照样 404）

**回归防线**：
- `pkg/server/appearance_test.go`（新增 8 例）：核心是 `TestFinishBackgroundPickPersists` ——
  同时断言「cfg 被改写」+「local.yaml 真的写盘」+「GET 报 background_set」+「背景图接口 200 且内容正确」。
  **已实测把修复注释掉后该测试会失败**（否则它就没有价值）
- 另有 `TestFinishPickDoesNotPersist` 钉住 `finishPick` 的语义边界（只回报不落库），
  防止有人把落库塞进通用收尾函数
- 前端断言 +4（`saveBg` 返回 Promise、`background:true` 直通、兜底先保存后应用、内置选择器分支同规则）
- 运行时验证 `test/verify_bg_pick.mjs`（新增 8 项）：拦截 `/api/appearance/pick` 返回**未落库**的响应，
  复刻老服务端行为 → 前端必须自己补保存。实测 fetch 序列为
  `POST /pick → POST /api/appearance（兜底保存）→ GET /background → 200`，截图确认图片显示

**教训**：⚠️ **验证要覆盖用户的真实操作路径**。我上一轮验证的是「路径已在配置里时渲染是否正确」，
而用户的路径是「通过选图对话框设置」。两者共用渲染代码，但**设置入口完全不同**，
所以我的验证给了虚假的安全感。

## 🐛 外观页全量审计：又抓出一个真 bug（2026-09-19）

背景图一个功能连爆两个 bug，于是把同一页其余控件逐个走一遍（`test/verify_appearance.mjs`，19 项）：

| 控件 | 结果 |
|---|---|
| 皮肤分段选择 | ✅ 写 localStorage + 选中态 + 滑条按皮肤重建 |
| 液态玻璃开关 | ✅ body 类 + localStorage |
| 对话框宽/高滑条 | ✅ CSS 变量 + localStorage |
| 背景模糊/亮度 | ❌ **模糊值存不进服务端** → 已修 |
| 背景图显隐 | ✅（含 z-index 回归修复，见上） |

**新发现的 bug：模糊与亮度共用了一个防抖定时器，且各自只提交自己的字段。**
```js
// 修复前：两个 handler 各自 setTimeout，后者把前者**尚未发出**的保存取消掉
blurEl.oninput   → clearTimeout(bgSaveTimer); bgSaveTimer = setTimeout(() => saveBg({ blur }), 400);
brightEl.oninput → clearTimeout(bgSaveTimer); bgSaveTimer = setTimeout(() => saveBg({ brightness }), 400);
```
后果：**拖完模糊紧接着拖亮度 → 模糊值永远存不进服务端**，刷新后被打回原值；亮度正常。
只在特定操作顺序下复现，所以极难自查。

**修复**：抽出 `scheduleBgSave()`，共用同一个防抖，且每次都提交**完整状态** `{blur, brightness}`。
既不会互相取消，最后一次请求也必然带着全部值。

**防线**：前端断言 +4（统一入口存在 / 提交完整状态 / 全页只有一个防抖 setTimeout / 不再有单字段保存）；
运行时断言把「模糊值落库」也钉住了（修复前 `srv=19`，修复后 `srv=40`）。

**同时给验证脚本加了像素采样能力**（截图 → data URL → 页面内 canvas `getImageData`）：
计算样式会说谎，涉及遮挡/层叠的问题只有像素能揭穿。

### 1c. 侧栏也要透（2026-09-19，用户明确要求）

**用户反馈**：设了背景图后主区透出图了，但**侧栏还是实心的**，要求「这半边也要透」。

**修正**：`body.has-bg #sidebar { background: transparent; }` —— 左右两块面板统一透明，
让图铺满整个窗口。

**⚠️ 关键区分**：上一轮我把它「修」成不透明，方向搞反了。而且实现方式很要紧 ——
**必须是面板自己变透明**，而不是让 `#bg-layer` 去盖住面板（把 z-index 从 -1 改成 0）。
后者看起来效果一样，但语义完全不同：
- 让面板透明：面板内文字仍在**正常层级**上绘制，可读性可控
- 让图层去盖：图层在绘制顺序上晚于在流块背景，会连带盖掉面板内的文字，
  且以后任何新增面板都会莫名被覆盖

所以 `#bg-layer` 必须保持 `z-index: -1`，透明靠 `#sidebar/#main` 自己的 `background: transparent` 实现。

**可读性代价**：侧栏文字（`--text` / `--text-dim` 深灰）直接压在图上了，
对比度取决于图片本身，靠「背景亮度」滑条压暗来保证。在纯色测试图下对比度偏低。
若觉得影响可读性，可改成毛玻璃（半透 + `backdrop-filter: blur()`），一行的事 —— **待用户定夺**。

**验证**：运行时像素采样三处（侧栏中部 / 侧栏右缘 / 主区空白）现在**全部**为 `rgb(224,0,224)`（图透出来），
侧栏计算样式为 `rgba(0,0,0,0)`，与主区一致；`verify_appearance.mjs` 19 项全过。

⚠️ 过程中又踩了一次残留实例：旧进程占着 8420，新实例绑不上，浏览器测到的仍是旧 CSS
（侧栏仍不透明），报出 3 项假失败。**已在验证脚本流程里加了前置校验**：
先 `curl /app.css | grep` 确认服务端资源确实含本次改动，否则直接中止 ——
不然测到旧实例会被误判成代码 bug。

## 🎨 深浅色自适应：文字永远清楚（2026-09-19，用户要求）

**需求**：外观要又美又清晰，**优先保证文字清晰**；亮度调低导致页面变暗时，文字应转为白色。

**判据不能只看滑条值**。亮图调到 60% 可能仍然偏亮，暗图即使 100% 也已经够暗。所以：
```
有效亮度 = 图片平均亮度 × (亮度滑条 / 100)
有效亮度 < 0.4  →  加 body.bg-dark（整套配色翻深色，文字转白）
```
- 图片平均亮度：缩到 32×32 画进 canvas 再读像素，按感知权重 `0.2126R+0.7152G+0.0722B` 求均值；
  **按图片 URL 缓存**，拖亮度滑条不会重复解码
- 图片走同源接口，canvas 不会被跨域污染；量不出来（异常/加载失败）就退回浅色主题，不让主题卡住
- 没有背景图时清掉 `bg-dark`，不误伤默认浅色主题

**样式做法：覆盖同一组 CSS 变量**（`--bg/--bg-elev/--bg-elev-2/--border/--text/--text-dim/--accent/--accent-dim` + 语义色提亮），
而不是逐个组件写 dark 规则 —— 漏一个组件就会白底白字。
另加：用户气泡翻成低饱和深粉（否则亮粉块在暗主题里刺眼）、`color-scheme: dark`（原生控件/滚动条跟着转）。

⚠️ **深色规则不得改动「透明」**：`body.has-bg #app/#sidebar/#main` 仍是 `transparent`，
背景图照样透出来 —— 这一条专门写了断言锁住。

**验证**（`test/verify_bg_theme.mjs`，13 项）：不靠肉眼，直接算 **WCAG 对比度** ——
取文字颜色与它背后真正渲染出的像素，`(L1+0.05)/(L2+0.05)`，要求正文 ≥ 4.5（AA）、次要文字 ≥ 3.0。

| 场景 | bg-dark | 文字色 | 正文对比度 |
|---|---|---|---|
| 深色图 + 亮度 100 | ✓ | rgb(238,240,246) | **14.97** |
| 浅色图 + 亮度 100 | ✗ | rgb(31,36,48) | **12.73** |
| **浅色图 + 亮度 20（用户核心诉求）** | ✓ | rgb(238,240,246) | **11.89** |
| 深色图 + 亮度 20 | ✓ | rgb(238,240,246) | **17.86** |

全部远高于 AA 阈值。截图 `cf-ui-dark-theme.png` / `cf-ui-light-theme.png` 已肉眼确认。

**顺带修的**：`verify_ui_fixes.mjs` 与 `verify_bg_image.mjs` 里还留着「图层 z-index=0」的旧断言
（那是上一轮的错误方向），已改为断言 `-1` 并把原因写进注释 —— 否则这两个套件会一直假失败。

## 🔴 「图片设置下次启动就没了」——根因是启动时的工作目录（2026-09-19）

**用户反馈**：图片的设置以及状态要保存，不要下次启动就没了。

**排查**：后端持久化本身是好的 —— 设图后 `local.yaml` 写入 `background_path`，
重启服务后 `GET /api/appearance` 仍返回 `background_set:true`，全新浏览器（无 localStorage）
打开页面图层也正确恢复（`blur(7px) brightness(0.55)` 一并还原）。**先证明没坏，再找真因。**

**真因**：`-config` 是相对**当前工作目录**解析的。用户若直接运行 `bin/` 下的可执行文件
（双击、快捷方式），默认值 `config` 会解析到 `bin/config`（不存在）→
读不到 `config/local.yaml` → 外观、模型、密钥**全部落回内置默认**，而程序照常启动。
实测复现：从 `bin/` 启动时 `background_set:false, blur:0, brightness:100`。
程序其实**打了警告**，但只在控制台，双击启动的人看不到。

**修复**：新增 `resolveConfigDir` —— CWD 相对路径下没有 `default.yaml` 时，
回退到 `<exe目录>/../<dir>`（对应仓库约定 `<root>/bin/exe` + `<root>/config`），
并打印提示。**保守回退**：绝对路径不猜、两处都没有就原样交给告警。
`stop`/`restart`/`start` 共用同一份解析（否则错误 CWD 下 `stop` 找不到运行信息文件）。

实测修复后从 `bin/` 启动：自动改用 `<root>/config`，`.env` 也一并加载，
`background_set:true`、模糊/亮度全部还原 ✓

**回归防线**：`cmd/agent/main_test.go` 的 `TestResolveConfigDirAt`（5 个子用例）：
回退生效 / 两处都没有则原样 / exeDir 为空则原样 / 绝对路径绝不改判 / CWD 可用时行为不变。

## 🎨 全 UI 对比度审计：又扫出 2 处真问题（2026-09-19）

写了 `test/verify_contrast.mjs` —— 遍历所有可见元素，逐个算「文字色 vs 它背后真正生效的背景色」
的 WCAG 对比度（背景从自身向上逐层合成；一路透明则取**元素框内众数色**作为背景图区域）。
覆盖主界面 + 设置面板的外观页/模型页。

**第一版报出 47 处，其中绝大多数是假失败**，两个原因：
1. **`<html>` 的背景被当成真实背景**：开了背景图后 html 的白底会被 `#bg-layer` 整个盖住，
   走到 `body` 就该停，否则所有压在图上的文字都被误判成「白底」。
2. **单点采样采到字形上**：得出「字色 == 底色、对比度 1.0」的荒谬结果。
   改成在元素框内打网格、量化后取**众数色**才可靠。

修好采样后：**深色模式 0 处问题** ✓。顺带修掉 2 处真问题：

| 问题 | 原因 | 修复 |
|---|---|---|
| 「选择图片」等 10 个实心主按钮白字对比度 **2.49** | 深色主题把 `--accent` 提亮到 `#7aa2ff`，而按钮填充与「accent 当文字」对亮度的需求**正好相反** | 拆出 `--accent-solid`（白字对比度 ≈5.2 / 深色 ≈4.8）与 `--on-accent`，10 处实心按钮全部改用 |
| 思考强度指示器 `#model-level` 对比度 **3.14** | 颜色写死 `#d6336c`，压在深色输入卡片上太暗 | 提为 `--level-accent`，深色下用 `#f06e9c`（≈5.1） |

**浅色模式仍有 45 处 3.6–4.3**（`--text-dim`/`--accent`/`--danger` 等**原有配色**，
略低于 AA 4.5）。这是应用既有的默认外观，**改动会影响用户熟悉的样子，故未擅自改**，留待用户定夺。

## 🎨 浅色模式文字色校准 + 图上文字光晕（2026-09-19）

全 UI 对比度审计在浅色模式扫出 **45 处 3.6–4.3**，全是「当文字用」的颜色压在浅底上不达 AA。

**① 配色校准**（同色相轻微压暗，肉眼几乎无感）：

| 变量 | 原值 | 新值 | 浅底上对比度 |
|---|---|---|---|
| `--text-dim` | #6b7280 | #636a78 | 4.2 → 4.7 |
| `--accent` | #4373f5 | #3a63e0 | 3.6 → 4.6 |
| `--accent-dim` | #e8eefc | #eef3fe | 激活胶囊 4.47 → 达标 |
| `--ok` | #16a34a | #127237 | **2.9** → 5.2 |
| `--warn` | #d97706 | #a35208 | 3.9 → 4.9 |
| `--danger` | #dc2626 | #d01f1f | 4.2 → 4.7 |
| `--level-accent` | #d6336c | #c02a5f | 4.05 → 4.9 |

设置面板：**0 处问题** ✓（原 45 处）

**② 图上文字：光晕 + 次要文字推向两端**

剩下的全部是「文字直接压在**中间调**背景图上」。这里有个硬性数学上限：

> 在 rgb(120,120,134) 这类中调底上，深字 (#1f2430) ≈ 3.2、浅字 (#eef0f6) ≈ 3.7
> —— **换任何颜色都到不了 4.5**。

所以换配色无解，必须靠「字形周围的局部反差」：
- `body.has-bg` 下给文字加**紧贴字形**的光晕（深字配浅晕、浅字配深晕）
- `--text-dim` 在 has-bg 下**推向两端**（浅色主题压到 #2b3140、深色提到 #d6dbe6）——
  原本的中间灰压在同样中调的图上，连光晕都救不回来（灰字配白晕 = 糊成一团白）
- **不用整块遮罩**，以保留用户明确要的「背景图完整透出来」

⚠️ 迭代了三轮才成形：第一版光晕太散（2px/50%）反而更糊，收紧到 1px/90% + 0.6 的 2px 才清楚。
**这类改动必须看截图**，审计量不到光晕（见下）。

⚠️ **`verify_contrast.mjs` 的已知局限**：它算的是「颜色 vs 背景色」，**看不见 text-shadow**。
所以中间调区域仍会报 < 4.5 —— 那是数学上限而非缺陷，已在脚本头部写明。
判断这类残留要**看 `shot_full_bg.mjs` 出的侧栏 2x 近景**，不要盲改颜色。

## ✅ 已完成（本轮会话，先于 Plan.md）

- [x] 新建项目改为弹窗（选工作区 / 项目名 / 默认权限）
- [x] 删除输入框左上角的工作区标签（入口并入新建项目弹窗）
- [x] 上下文统计弹窗改造：模型名称 / 上下文长度 / 已使用总 tokens / 缓存命中 / 缓存未命中（含 LLM 用量上报链路）
- [x] Termux：工具安装建议弹窗（安装 / 跳过 / 不再提示）
- [x] Termux BUG 修复：选择目录列表为空（/api/tree?picker=1 + 起始目录挂 ~）
- [x] web_fetch 联网增强：JS 动态页面无头浏览器渲染兜底 + 系统代理读取（Windows 注册表按平台拆分）+ 代理失败自动重试一次 → 回落无代理直连 → 渲染兜底（实测经代理抓取 GitHub 783ms 成功）
- [x] P0 工具描述清理（防止工具链泄露）：todo/web/file 工具描述剔除内部实现细节（无头浏览器、安全层、快照撤销、○/◐/✓/✕ 渲染符号等），保留「能做什么/何时调用/输入输出」；「整表替换」保留为输入契约（模型需按完整清单提交）

## 🌐 远程 MCP（Streamable HTTP）— 已落地

- [x] 新增 `mcp-http` 驱动（`pkg/tools/plugins/driver_mcphttp.go`）：POST 单发单收的 Streamable HTTP（MCP 2025-03-26），响应**兼容一次性 JSON 与 SSE** 两种；`initialize` → `tools/list` → `tools/call`；工具以 `插件名.工具名` 注册（`qualify`）
- [x] 默认启用 **Parallel Search**（`https://search.parallel.ai/mcp`，免费无 Key）：`web_search` 返压缩摘录 + `web_fetch` 返 Markdown，成为默认联网来源；`exa_search`（`https://mcp.exa.ai/mcp`，语义搜索）默认关，供切换
- [x] 内置 `web_fetch`/`web_search` 缺省保持关闭（`web.enabled` 为 nil 即 false）→ 默认联网来源就是 Parallel MCP，无重叠
- [x] `/api/plugins`：GET 回传 `endpoint`/`path`（前端展示远程端点）；POST 支持 `type`（mcp / mcp-http）并按类型校验（stdio 要启动命令、mcp-http 要 http(s) 绝对 URL）
- [x] 设置页「MCP 服务」新增表单：加「类型」下拉（本地 stdio / 远程 Streamable HTTP）+ 端点 URL 字段，按类型切换字段与校验（前后端同源规则）
- [x] 侧栏 MCP 入口与设置页列表同时展示 `type=mcp` 与 `mcp-http`，远程条目显示端点而非启动命令
- [x] 前端工具文案：插件限定名（`parallel_search.web_search`）按去前缀本名匹配文案，并认 Parallel 的入参形状（`objective` / `urls[]`）→ 工作流显示「搜索了 …」而不是生硬的限定名
- [x] 测试：`driver_mcphttp_test.go`（握手顺序 / 真实 RemoteTool.Execute / SSE 解析；联网冒烟改为 `CODEFORGE_E2E=1` 才跑）、`pkg/server/plugins_handler_test.go`（远程添加全链路 + 按类型校验 6 例）；前端断言 249 项
- [x] **真实 LLM 端到端验证**（`pkg/server/e2e_remote_mcp_test.go`，本地假远程 MCP + 真实模型）：模型调用 `demo_search.web_search` → 假服务收到 `[initialize tools/list tools/call]` → 回复中复述响应里的标记，证明内容确实经 HTTP 回填（非模型编造）
- [x] **修复 SSE 路径未解包 JSON-RPC 信封**（2026-09-16 补，TRAE 遗留未验的 Exa 端点暴露）：`request()` 的 SSE 分支原样返回整个信封而非 `result`，与一次性 JSON 分支语义不一致 → Exa（走 SSE）表现为「握手成功、零报错、**0 个工具**、调用返回空内容」。修复 = SSE 分支也过 `parseJSONResponse` 统一解包 + 校验 id + 抛 error
- [x] 联网冒烟改为**覆盖两个预置端点**（`TestMCPHTTPLiveEndpoints`：Parallel 一次性 JSON / Exa SSE），并加强 SSE 单测到「断言真的解析出工具与调用结果」—— 旧测试用空 tools 数组且只断言 `err == nil`，正是它放过了这个 bug

## 🔧 请求重试设置（2026-09-16，设置 → 常规 可调）

**需求**：上游限流（如「该模型当前访问量过大」）时自动重试，但次数与间隔希望可自定义。

**新增配置**（`config/config.go` LLMConfig）：
- `max_attempts`：总尝试次数（含首次），默认 5，范围 1~15
- `retry_mode`：`fixed`（每次等同样时长）/ `backoff`（1×→2×→3××6× 常用退避序列，之后封顶 6×），默认 backoff
- `retry_backoff_ms`：基础间隔，默认 1000（backoff → 1s/2s/3s/6s；fixed → 每次 1s）

**后端**：
- `pkg/llm/sse.go`：`RetryPolicy` 加 `Mode` + `attemptDelay(attempt)`（fixed 直返 Backoff；backoff 按 [1,2,3,6] 乘子，封顶 6×）；`normalize()` 对非法/大小写脏值统一回退 backoff；postJSON 的 `delay *= 2` 旧逻辑改为按次计算，上游 `Retry-After` 仍优先生效
- `pkg/llm/provider.go`：`retryPolicyFromConfig` 透传 RetryMode
- `pkg/server/api_handlers.go`：GET `/api/config` 暴露 `retry_max_attempts/retry_mode/retry_interval_sec`；POST 支持三个字段（空值=不改），非法模式 / 越界次数 / 越界间隔直接 400，保存后 `rebuildProvider()` 热生效 + 写回 local.yaml
- `config/default.yaml`：更新注释（原 1500ms 纯指数 → 1000ms 的 1/2/3/6 序列）

**前端**（`web/dist/index.html` + `ui.js`）：常规页新增「请求重试」卡片（次数 / 间隔模式 / 基础间隔秒），打开设置时从 `/api/config` 回填，保存走 POST + 「已保存」反馈；次数上限与后端同步为 15。

**测试**：
- `pkg/llm/retry_policy_test.go`：退避序列 [1,2,3,6,6]s、fixed 恒定、模式归一化（含 `""`/`exponential`/大小写脏值）
- `pkg/server/retry_settings_test.go`：合法保存后 GET 读回 + 热生效验证；非法模式 / 越界次数被拒绝且不污染已存配置
- 全量 `go test ./...` 仅剩既有 Windows `cat` 环境失败；5 产物已重编；服务已重启，`/api/config` 实测返回三个新字段

## 🔁 双报错修复（2026-09-16，402 余额不足显示两遍）

**故障**：上游返回 402 时，界面上同一条错误显示两遍。

**链路定位**：
1. `pkg/agent/agent.go` runLoop：`provider.Stream()` 失败时 emit 一次 EventError，随后 `return err`
2. `pkg/server/ws_handler.go:227` `run()`：agentFn 返回错误时又 `c.send({type:"error"})` 一次
3. 前端 `ui.js:2709` 每个 error 事件 `addError` 追加一条气泡 → 双显示

**改动**（[agent.go](../pkg/agent/agent.go)）：
- runLoop 的 `Stream` / `consumeStream` 两处致命错误**不再 emit**，只 `return err` —— 错误统一由 WS 层（`ws_handler.run`）下发一次；调用方还有子智能体路径，其错误经 `result.Error` → failed 事件呈现，不依赖内部 emit
- `consumeStream` 循环内的 `EventError` emit 移到函数末尾，且**只在「部分内容被截断」时发**（致命错误不 emit、交给调用方；有文本/工具调用时保留内容并补一条截断提示）

**回归测试**（`pkg/agent/error_emit_test.go`）：
- `TestRunLoopFatalErrorEmitsOnce`：Stream 直接失败 → 循环 emit 0 次 + 错误原样返回
- `TestRunLoopPartialErrorEmitsOnce`：流到一半出错 → 只 emit 1 次 + 不返回错误（本轮照常收尾）

全量 `go test ./...` 仅剩既有 Windows `cat` 环境失败（`TestAutoModeOutsideRequiresApproval`）；5 个跨平台产物已重编；服务已重启（parallel_search 14 工具正常）。

## 📦 插件/MCP 服务内嵌二进制（2026-09-16，支持裸 exe 分发）

**背景**：用户希望只发 exe 给别人就能直接跑。原先插件配置只存在于仓库 `config/plugins.yaml`，二进制里只有前端 `web/dist`（go:embed），裸 exe 启动后无 Parallel 搜索。

**改动**：
- `config/plugins.yaml` 通过 `//go:embed plugins.yaml` 打进 config 包（[config.go](../config/config.go)）；`DefaultPlugins()` 解析内嵌列表
- `Load()` 合并语义：内建默认 ⊕ 用户本地 `plugins.yaml` —— 同名条目以用户为准（可停用内建、改端点），内建独有的保留，用户新增的追加（`mergePlugins`，名称匹配不区分大小写，与 API 判重一致）
- `LoadPlugins` 与 `DefaultPlugins` 共用 `parsePlugins`；磁盘坏 YAML 仍报错，内嵌解析错误忽略（编译期已定）

**行为**：
- 裸 exe（空目录启动）实测：`[plugins] 已加载插件 parallel_search（mcp-http），工具数 2`，13 个工具（11 内置 + Parallel 的 2），仅几条非致命的「缺 default.yaml」提示
- config 目录与 plugins.yaml 仍可选：缺则用内嵌默认，有则按名覆盖；设置页增删插件照旧写回 `config/plugins.yaml`
- 仍然需要的外部依赖：LLM API Key（环境变量或设置页）、工作区需在界面选择

**测试**：`TestDefaultPluginsBuiltin`（内嵌含 parallel_search 且默认启用）、`TestLoadMergesBuiltinPlugins`（停用内建 + 追加新插件 + 保留未覆盖项）、`TestLoadWithoutPluginsYamlUsesBuiltin`（裸目录 = 全部内建）；`TestPluginsAddValidation` 断言改为「校验失败不留下用户插件（排除内建条目）」；config + server 包全过，全量 `go test ./...` 仅剩既有 Windows `cat` 环境失败；5 个跨平台产物已重编。

## 🐛 限定名泄露给远端（2026-09-16，Parallel Search 实测故障定位）

**现象**：DeepSeek-Flash 测试「搜索最近新闻」时，`parallel_search.web_search` 连续十几次返回 `Unknown tool: parallel_search.web_search`，模型陷入无效重试，最后靠 web_fetch 抓新闻站首页拼出未核验的结果。GPT 分析列为「调用名称链路不一致」。

**静态定位过程**（关键证据 = 审计日志 `work_dir/.codeforge/audit.jsonl`）：
1. 健康检查 `/api/health` 显示 14 个工具 = 11 内置 + web_fetch + Parallel 的 2 个 → 插件已加载，注册成功；
2. 审计日志里执行器收到的名字是 `parallel_search.web_search`（注册名），错误却是**英文** `Unknown tool: ...` —— executor 的「未知工具」是中文，说明错误来自**远端服务器**；
3. 结论：名称链路全程正确（wire 名 → ResolveWire → 注册名 → executor Get 成功），唯独最后一步 `RemoteTool.Execute` 把**本地限定名**当作工具名发给了 Parallel.ai，远端只认发现时的原始名 `web_search`。

**根因**：`manager.loadLocked` 用 `qualify` 把 `rt.name` 改成「插件名.工具名」用于本地路由，但 `RemoteTool.invokeFn` 的闭包/`Execute` 用的是 `t.name`（改写后的限定名）。`driver_mcp.go`（stdio）、`driver_mcphttp.go`、`driver_http.go`、`driver_wasmplugin.go` **四个驱动全部中招**——stdio 版一直没暴露只是因为之前没有启用的 stdio MCP 插件被真实调用过。

**修复**：`RemoteTool` 新增 `remoteName`（发现时的原始名），`Execute` 用它调远端，限定名只用于本地路由/权限/审计。四驱动构造处一并设置。

**回归防线**：
- `TestRemoteToolUsesRemoteNameAfterQualify`：fake 服务端模拟限定名改写后调用，断言 tools/call 收到的是原始名（离线、常跑）；
- `TestMCPHTTPLiveManagerQualifiedCall`（`CODEFORGE_E2E=1`）：走 `NewManager.LoadAll` → 注册表按 `parallel_search.web_search` 取出 → Execute 打真实 Parallel.ai，返回搜索结果 JSON（复刻故障现场的全链路）；
- 验证结果：限定名路径真实调用 2.34s 返回 `search_id` + results，`go test ./pkg/tools/plugins/` 全过；全量 `go test ./...` 仅剩既有 Windows `cat` 环境失败；前端 265 项断言全过；5 个跨平台产物已重编，服务重启后日志 `[plugins] 已加载插件 parallel_search（mcp-http），工具数 2`。

**GPT 分析中仍然成立的点（后续可选做，本轮未动）**：确定性「未知工具」错误的止损机制（同一失败循环十几次才停）；网页提取质量校验（JS 脚本当成功）；最终回答的证据约束（标题缺链接/时间）。

## ⏳ 重试提示「具体化」（2026-09-16）

- [x] 后端补透「第几次 / 共几次」：`RetryHook` 签名加 `maxAttempts`（原先 `attempt` 被 `agent.go` 直接丢弃），`Event` 新增 `attempt` / `max_attempts` 字段 → WS 直接下发
- [x] 前端 `retryReasonBrief()`：把原始错误串压成一句人话 —— `429→上游限流`、`503→上游过载`、`502/504→上游网关异常`、`5xx→上游服务异常`、`4xx→请求被上游拒绝`、网络层→`网络连接异常/请求超时`；并尽力带出上游错误体里的 `message`（如「该模型当前访问量过大，请您稍后再试」）
- [x] 提示从「请求失败，正在重试…（原因要点一次才看得到）」改为**主文案直接带摘要**：`请求失败，正在重试…（第 2/3 次 · 上游限流：该模型当前访问量过大，请您稍后再试）`；完整原始错误仍可点击展开，并补了可见的「详情/收起」小标签（原先只有悬停才猜得到可点）
- [x] 测试：前端断言 249 → **260**（11 条覆盖分类/降级/透传/样式契约）；新增 `pkg/server/retry_event_test.go` —— 假 503 网关逼出重试，断言 WS 事件里 `attempt` 递增、`max_attempts` 正确、原因含状态码与上游原话，**全程不依赖外网**

## 🔒 工具名不外泄（2026-09-16，用户反馈模型在正文里说「我该调用 xxx」）

- [x] **提示词纪律从「两种场景」扩到全覆盖**：原纪律只管「描述动作」与「被问能力」，没管住模型在**计划 / 步骤说明 / 思考小结 / 收尾总结**里复述工具名。新增一条：凡用户能看到的文字都不得出现工具 ID、函数名、调用代号或其片段，也不要预告「我该调用某工具」，直接说做什么（「我先搜索一下」）
- [x] **前端兜底不再吐工具 ID**：`toolLabel` 的 `default` 原为 `'调用了 ' + name`（直接把 ID 打给用户），改为中文兜底「调用了外部能力」；`toolPhrase` 同样改为「外部能力」。**Exa 的 `web_search_exa` / `web_fetch_exa` 之前会落到兜底泄露 ID**，改为同族前缀匹配（`web_search*` / `web_fetch*`）
- [x] 回归防线：`TestSystemPromptLeaksNoToolIDs` 增断言钉住「凡用户能看到的文字都不得出现工具 ID」这句不被精简掉；前端新增 4 条断言（含**全量兜底扫描**：对已注册的全部工具名跑一遍 `toolLabel`，断言输出不含 `.`、也不是 `调用了 <英文标识>`）；前端断言 260 → **265**


## 🔒 工具泄露防护路线（GPT 建议，已确认采纳）

- [x] P0 Description 清理/分离：剥离内部实现细节，保留能力描述与时机规则
- [x] P1 Exposure 层（极简版）：Registry.DefinitionsFor(pick) 过滤 + Agent.SetExposure + config.agent.hidden_tools 隐藏工具（按真实工具名，不发给 LLM，注册/执行/权限/审计链路不变），无 AI 动态判断
- [x] P0.5 System Prompt 工具对照清理：删除 prompt.go 的「中文名=工具ID」对照表与工具纪律中的裸工具 ID，改为能力导向（"使用长期记忆能力"而非 "用 save_memory"）；新增防泄露测试 TestSystemPromptLeaksNoToolIDs（提示词出现任何内部工具 ID 即失败）；隐藏 ≠ 禁止执行的边界已在各处注释澄清（禁止执行需 Executor/Policy 层）
- [x] 真实 A/B 泄露验证（DeepSeek-Flash）：todo_write 参数/状态泄露 6/6 → 0/6；tool_names 探针 隐藏后说不出被隐藏工具的内部 ID（4/7 → 2/7）；save_memory 残余仅剩「记住/偏好」等能力语义词（复述能力边界，非工具标识符），模型明确"不编造工具定义"；测试文件 exposure_ab_test.go 保留，CODEFORGE_E2E=1 可复跑
- [x] 收尾安全审计（全项目工具 ID 扫描）：确认第二泄露通道 = 内置插件注入段（builtin_plugins.go 的 Instructions 曾写 create_skill/delegate_subagents），已改为中文能力描述；审计 memory/project_memory/skills/subagent/工具结果渲染/WS 广播均无 ID 泄露（代码内白名单/注册表引用属合法）；新增 TestBuiltinPluginSectionLeaksNoToolIDs 回归防线；web.go 工具描述互指改为中文功能名
- [x] todo.go 丢失问题处理：文件曾从磁盘消失且未被 git 跟踪，已按早先内容重建并 git add 纳入版本控制（防再次丢失）
- [x] 「能力询问」泄露修复（用户问"你可以调用计划功能吗"模型曾答"底层是 todo_write"）：根因 = 工具 ID 必然在 Tool Definition 中（模型要调用就必须知道名字），P0.5 删 Prompt 对照后模型改从 Schema 学名；修复 = prompt.go 新增披露纪律「被问能力时只用中文能力名，不附英文工具代号」；真实 LLM 验证 capability_ask 探针 A/B 均 0/5（不再泄露 ID），B 实例明确回复「本轮工具清单中不存在计划类工具」；回归防线由 TestSystemPromptLeaksNoToolIDs + exposure_ab_test.go 的 capability_ask 探针共同锁定
- [x] Plan 计划模式（内置插件，从技能迁移）：builtin_plugins.go 新增 BuiltinPlan（ID=plan，设置页可开关，config: builtin_plugins.plan，缺省开）；常驻 System Prompt 注入（只读规划 + 计划书结构），不依赖工作区技能文件；已删除 .codeforge/skills/plan/SKILL.md；真实 LLM 验证 @plan → 模型"使用插件 Plan（计划模式）"→ 探索 → 输出计划书
- [x] @ 提及面板规范化（技能 + 内置插件，不含 MCP）：技能新增可选 display_name（SKILL.md frontmatter / create_skill 支持，@ 面板与 +菜单显示显示名、插入用 @slug 确保后端命中）；内置插件（Plan 等）与技能并组展示（名称 + 简要介绍）；@ 面板不再拉取 MCP 插件；空态提示文案更新；前后端测试同步更新（246 项前端断言全过）
- [ ] P2 确定性 Tool Group / Mode（coding / web / restricted 等）
- [ ] P3 动态工具暴露（按 session / mode / task 裁剪）
- [ ] P4 复合能力（research 等，非 Coding 原子工具链）

## 📋 测试状态（P0 → P1 期间）

- 核心/全量 Go 测试执行（`go test ./...`），除下述环境遗留问题外无新增失败；
- 环境遗留问题：Windows 环境缺少 `cat` 命令，导致 `fence_scope_test.go` 的越界命令用例无法执行（`'cat' is not recognized...`），此问题早于本次改动已存在，与 P0/P1 无关。

## 🎯 已产生的教训

- Go 默认只认环境变量代理；代理软件开「系统代理」写注册表时，Go 直连被墙站点会超时，需读注册表（Windows-only 代码必须拆 `_windows.go`，否则 Linux/Android 交叉编译报 build constraints）
- 代理隧道有偶发 EOF：一次失败不代表代理不可用，先重试代理再回落直连（直连对墙站要等满超时，成本更高）

## 🧭 任务工作流提示词增强（2026-09-23，参考 Claude Code / Cursor 提示词语料）

- [x] 「任务工作流」段（[prompt.go](../pkg/agent/prompt.go)）吸收可迁移纪律，只取行为判据、不带产品专属机制：
  - **探索**：找同类功能作参照、沿关键调用路径走一遍（Plan 子代理）；委派子智能体时任务说明要**自含**（它看不到本会话上下文）并指明**搜索广度**（快速/适中/彻底多角度，Explore thoroughness 分级）
  - **规划**：任务清单加「依赖先后」并预判风险点（Plan：dependencies and sequencing / anticipate challenges）
  - **编写**：命令要新建目录或文件时先确认父目录位置正确（Cursor Shell 的 Directory Verification）；不镀金、也不留半成品（general-purpose：don't gold-plate, don't leave it half-done）
- [x] 同步压缩重复表述保持体积预算：删除与「搜索与探索」段重复的「多换关键词与命名习惯」等，净增长控制在 4096 字符上限内（3997 → 4093）
- [x] 回归防线：`TestDefaultSystemPromptHasTaskWorkflow` 新增 6 条断言钉住新纪律（自含任务说明 / 搜索广度 / 同类参照 / 依赖先后 / 父目录确认 / 不留半成品），防日后精简被删；`go test ./...` 全绿；按 AGENTS.md 完成全平台编译（win-amd64 / linux-amd64 / android-arm64 + 本机 bin/codeforge.exe）

## 🐛 四处用户反馈修复（2026-09-27）

一次收到四条反馈，逐条定位后都找到了**具体根因**（不是症状），改动落在服务端语义上而不是打补丁。

### 1. 子智能体：轮次太少 + 上下文超限

四条根因，都在 `newSubagentIn`（[subagent.go](../pkg/agent/subagent.go)）：

| 症状 | 根因 | 修 |
|---|---|---|
| 工具调用轮次太少 | 写死 `if MaxSteps > 8 \|\| <= 0 { = 8 }`，与主 loop 的 `max_steps` 毫无关系 | 新增独立配置项 `subagents.max_steps`，**0 = 继承主 loop**（硬上限 `config.SubagentStepCap = 60`）；设置 → 子智能体有「单任务步数上限」输入框，0 显示为「跟随主循环（N 步）」 |
| **可能上下文超限** | `contextWindow` 是 Agent 上的 atomic，`New()` **不填** → 子智能体窗口 = 0 → `compressBudget()` 回退到写死的 `context_token_budget` **120000**。模型真实窗口若更小（32k / 64k 端点），子智能体会堆到上游 400 才压缩 → 收缩两次仍失败 → 子任务直接挂 | `child.contextWindow.Store(int64(a.ContextWindow()))` |
| 提示词与工具不一致 | `builtinOn` 被继承 → System Prompt 注入「你可以创建技能 / 委派子智能体」，而白名单里**没有** `create_skill` / `delegate_subagents` → 模型反复调不存在的工具直到步数耗尽 | `child.builtinOn = map[string]bool{}`（刻意不继承） |
| 隐藏的工具又出现 | `exposure` 没继承 | `child.SetExposure(a.exposureFn())` |

顺带把 `a.SubagentPolicy()` 从两次调用收成一次（`sp`）：白名单与步数必须来自同一个版本，
否则设置页刚好在那一瞬间改配置，子智能体会拿到「按旧策略挑工具、按新步数跑」的组合。

> ⚠️ **踩过的坑（写测试时才发现）**：`New()` 根本不填 `contextWindow` 这件事，
> 单测里如果不显式 `a.SetContextWindow(...)`，父子窗口都是 0，测试会「通过」而什么也没验证到。
> `TestSubagentInheritsContextWindow` 因此显式设 32768 再断言子智能体拿到 32768。

### 2. Termux：装过 termux-tools 也换不了背景图

三个叠加问题（[appearance.go](../pkg/server/appearance.go) / [termux.go](../pkg/server/termux.go)）：

- **探测只看进程 PATH 里的 `termux-open-url`**。从 Termux 外部拉起 CodeForge（Termux:API、
  桌面快捷方式、某些启动器）时 `PATH` 里没有 `$PREFIX/bin` → 明明装过也判「未安装」→ 反复弹安装建议。
  改为：显式把 `$PREFIX/bin` 并入查找，并探测三个代表命令（`termux-open-url` /
  `termux-storage-get` / `termux-setup-storage`），缺哪个逐项报。
  `$PREFIX/bin` 那一段直接 `stat` 而不再 `exec.LookPath` —— LookPath 在 Windows 上只认带扩展名的
  可执行文件（裸文件永远判「找不到」），而 Termux 是 POSIX 语义；顺带让这段逻辑可测。
- **失败一律写成「需要 termux-tools」**，真实原因（存储未授权 / SAF 要前台）全被掩盖 ——
  用户明明装过包也被这句话带偏。改为 `termuxToolsHint()` 区分「命令确实不在」与「命令在但调用失败」，
  后者把上游原始报错带出来并指向 `termux-setup-storage`。
- **Termux 没有兜底路径**：`builtin` 选择器分支只给「非 Windows 非 Termux」，
  `termux-storage-get` 一挂就彻底换不了图。改为失败时回落内置选择器
  （`start_path` 优先已授权的 `~/storage/shared`，否则退回 `~`），响应带 `degraded_from` +
  `warning`，前端 `addInfo` 如实告知为什么换了选图方式。
- 落盘路径 `filepath.Join(cfg.DataDir, "background_user")` 用的是**相对** `DataDir`，
  靠进程 CWD 才落得下去 → 工作区切换 / 从别处启动就写错位置。改为 `bgPickDest()`：
  相对则挂到 `agent.WorkDir()` 下；`DataDir` 或工作区为空时**报错**而不是照做
  （`filepath.Join("", "x")` = `"x"`，等于把图片写进进程工作目录 —— AGENTS.md 记过同类坑）。

> ⚠️ **验证时踩的坑**：在 Windows 产物里 `findstr` 搜 `degraded_from` 搜不到，一度以为代码没编进去。
> 实际是 `platform.IsTermux()` 在 `sys_windows.go` 里是 `return false` 常量，编译器把整个
> `case platform.IsTermux():` 分支常量折叠掉了（该分支在 Windows 上本就不可达）。
> 在 android / linux 产物里能搜到。**「Windows 二进制搜不到」不等于「代码没生效」**。

### 3. 主 loop：「继续」按钮直接发送「继续」

- **根因**：`sendContinueRequest()` 走 `sendAsUserText('继续')` —— 往历史里塞一条**用户从未说过**的
  假提问。后果有三：历史回放时它仍在（用户会问「我什么时候说过继续」）；模型分不清
  「被打断后接着跑」与「用户新提了一个要求」；界面多一条无意义气泡。
- **修**（选方案 A：服务端语义）：
  - 新增 `Agent.ContinueTurn()`（[agent.go](../pkg/agent/agent.go)）：
    **不追加任何用户消息**，只补齐悬空 `tool_result`（上游对消息序列有硬约束，残缺序列会被 400 拒绝），
    再把「本轮是继续上一轮」作为**一次性系统提示尾巴**挂上，跑完即清（`defer sess.SetContinueHint("")`）。
  - 提示挂在 `Session.continueHint`（不是 Agent 字段 —— 并发会话会互相覆盖），
    由 `systemPromptFor` 接在任务清单之后，**不落库、不进 `Messages`**。
  - WS 新增 `continue_turn`；`checkpointEvent.resume_back` 语义不变（判据仍在服务端）。
- 与 Regenerate / RerunFrom 的分工写进注释：**不截断历史、不回退文件**；
  破坏性的重来是那两个显式入口的职责。

### 4. 主 loop：编辑之前的消息时，被编辑的消息直接就没了

- **根因**：前端**先破坏视图、再等服务端确认**。`sendEditRequest` 先
  `truncateAfterEditableMessage(back)` 把被编辑那条**连同其后全部**删掉，才 `addUser(text)` 补回。
  只要中途 `wsSend` 失败、或服务端在 emit `edit` 之前就 `return error`
  （`RerunFrom` 里 `back` 定位不到消息 → `agent.go` 直接返回），那条消息就**永远回不来**。
- **修**：
  1. 改成 `applyEditInPlace()`：**只删目标 `nextSibling` 之后的节点，目标那条就地改写**
     （`dataset.rawText` + `renderUserText`）。任何失败路径下它都还在。
     找不到目标行时退回旧的「全删 + 重建」，宁可位置算错也不让消息消失。
  2. 加 `pendingEdit` 标志：提交后置位，收到 `history` / `idle` 清除；
     收到 `error` 且仍置位 → 主动 `load_session` 拉权威快照，把视图拉回真相
     （否则界面停在乐观改动后的状态，与库里实际历史对不上）。
  3. **倒计数排除 steer**：`.msg-steer` 视觉上也是一条用户消息（带 `.msg-user` 类），
     但插话不是「提问」。服务端算 `back` 用 `isUserQuestion`（排除 `OriginSteer`），
     前端把 steer 算进去会数错条数、截错位置。改用 `.msg-user:not(.msg-steer)` 定位。

### 测试与验证

- **Go**（`go test ./...` 全绿，`go vet` 干净）：
  - `pkg/agent/max_steps_test.go`：`TestMaxStepsSubagentCap` → `TestMaxStepsSubagentInheritsParent`
    （旧用例钉的是 `{100,8}`/`{0,8}`，即写死的 8，必须一起改）+ 新增
    `TestMaxStepsSubagentPolicyOverride` / `TestSubagentInheritsContextWindow` / `TestSubagentDropsBuiltinPlugins`
  - `pkg/agent/resume_test.go`：`TestContinueAfterMaxStepsOnlyGrowsHistory` 改走 `ContinueTurn`；
    新增 `TestContinueTurnAppendsNoUserMessage` / `TestContinueTurnHintIsTransient`
  - `pkg/server/continue_turn_ws_test.go`（新文件）：`TestWSContinueTurnRunsWithoutFakeUserMessage`
    —— 协议层钉「`continue_turn` 不追加用户消息」，并断言系统提示带「本轮是继续上一轮」
  - `pkg/server/appearance_termux_test.go`（新文件）：`$PREFIX/bin` 命中、逐项报缺哪个、
    已安装时的提示**不含**「未安装/需要 termux-tools」且带 `termux-setup-storage`、
    `bgPickDest` 绝对路径 / 空 `DataDir` 报错 / 无工作区报错、`termuxStartDir` 回落
  - `pkg/server/subagents_handler_test.go`：`TestSubagentMaxStepsRoundTrip`（默认继承 →
    显式 6 → 负数归零 → 越界收敛 → 落盘重载）
  - `config/subagents_test.go`：`TestSubagentMaxStepsInheritsWhenUnset`、
    `TestStateOmitsUnsetSubagentMaxSteps`（未配置时 `state.yaml` 不出现该键）
- **前端**（`web/test/render_md.test.js` 512 项全过，原 488 项）：新增三组断言
  「继续上一轮」「子智能体：步数上限与上下文窗口继承」「Termux 换背景图」。
  加了 `goCode()` / `uiCode()` 辅助函数**剥掉行注释**再断言「代码里没有 X」——
  不剥掉的话，注释里正好在解释那个被删掉的旧写法（「早先是 sendAsUserText('继续')」），
  会被当成残留代码，测试恒红。
- **编译**（按 AGENTS.md）：`bin/codeforge.exe` + `codeforge-windows-amd64.exe` +
  `codeforge-linux-amd64` + `codeforge-android-arm64` 全部重编（`CGO_ENABLED=0 -trimpath -ldflags "-s -w"`）。
  `bin/codeforge-linux-arm64` 按约定**不在默认范围**，仍是 09-23 的旧产物。
- ⏸️ **未做端到端验证**：本机是 Windows + 无浏览器自动化环境，
  Termux 相关的降级路径（`termux-storage-get` 失败 → 内置选择器）只有单元测试覆盖，
  未在真机上跑过；「继续」与「编辑重发」的前端交互同理（契约断言齐了，真实点击未跑）。
  `gofmt -l` 仍列出 `cmd/agent/main.go` / `pkg/agent/context.go` / `pkg/agent/concurrency_test.go`
  三个**本次未改动**的既有文件，未擅自重排以免混入无关 diff。



