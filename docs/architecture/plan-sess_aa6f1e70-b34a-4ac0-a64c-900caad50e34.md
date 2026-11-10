# 持久化存储方案：对话 / 记忆 / 技能

## 一、SQLite 存储层（新 pkg/store）

**驱动**：`modernc.org/sqlite`（纯 Go 零 CGO，不影响 Windows 交叉编译；项目已用 wazero 证明偏好零 CGO）。

**库文件**：`%USERPROFILE%/.codeforge/data.db`（用户级，跨工作区共享；会话用 workspace 字段隔离）。用 `os.UserHomeDir()` 解析，失败回退 configDir。

**Schema**：
```sql
sessions(id TEXT PK, workspace TEXT, title TEXT, created_at, updated_at)
messages(session_id, seq, role, content JSON, PRIMARY KEY(session_id, seq))
memories(id INTEGER PK, workspace TEXT, content TEXT, created_at, updated_at)
```
messages.content 存 `[]llm.ContentBlock` 的 JSON，读出解组——与现有 llm.Message 结构完全兼容。`database/sql` + `?` 占位符绑定，单写多读。

**旧数据迁移**：启动时扫描 `<旧DataDir>/*.json`（.codeforge 下每个会话文件），导入后改名 `.json.imported`（不删除，保留回滚余地）；workspace 字段回填为该 JSON 所在目录推导的工作区。一次性迁移，幂等。

## 二、会话持久化恢复（History 改造）

- `pkg/agent/history.go` 的 **对外 API 不变**（Create/Get/Save/List/Delete），内部实现改为调 pkg/store——agent/server 层几乎零改动。
- `List(workspace)` 按当前工作区过滤（对应你「不同工作区加载各自会话」的要求）。
- REST 新增：`GET /api/sessions?id=` 返回完整 messages；`POST /api/sessions/rename`。侧栏删除直接用现有 DELETE。
- WS 新增 `load_session` 消息类型：前端切换会话 → 服务端 `History().Get` → 下发 `history` 事件（含全部 messages）→ 前端用现有 addUser/appendText/addTool/addActions 原语重放渲染（审批条渲染为已决状态、无 spinner）。

## 三、记忆系统（pkg/agent/memory.go + 2 个工具）

- **用户记忆**：用户说「记住：X」→ AI 调用新工具 `save_memory`（content=「X」，存 SQLite memories 表，workspace 维度）；`/api/memory` REST 支持列表/增删（设置页可视化管理）。
- **注入**：每轮 runLoop 组装 System Prompt 时追加「## 用户记忆」段落（当前工作区全部记忆条目）。
- **项目级记忆**：组装时扫描工作区根的 `AGENTS.md`、`CLAUDE.md`（存在即全文注入「## 项目说明」段）。每次 runLoop 读文件（项目文件可能随时被用户改动），不做缓存。

## 四、技能系统（SKILL.md 目录约定）

- **位置**：`<workspace>/.codeforge/skills/<name>/SKILL.md`；frontmatter：`name` / `description` / `triggers`（逗号分隔触发词）。
- **发现**：runLoop 时扫描 skills 目录解析全部技能；**索引注入**（仅名称+描述+触发词，常驻约几十 token）；用户消息命中触发词（或显式 @技能名）时把该 SKILL.md 全文注入当轮 System Prompt 的「## 技能」段。
- **管理**：`/api/skills` REST 列表（设置页展示+启用开关用 frontmatter 的 enabled 字段）；文件型存储，用户可直接编辑 SKILL.md。

## 五、前端（ui.js + index.html + app.css）

1. **会话侧栏**：恢复旧版被删的侧栏列表（CSS `.session-list` 还在），处理 `ready`/`sessions` 事件（当前被忽略）；点击会话→`load_session`→重放；「新会话」按钮；每项带删除；刷新自动恢复该工作区最近会话。
2. **设置页**：新增「记忆」「技能」两个 nav-item（通用 handler 自动接管），列表交互复用模型管理页的 `.model-item` 模式；记忆支持编辑/删除，技能列表只读+提示用 SKILL.md 编辑。

## 六、文档与验证

- `docs/persistence.md`：schema、迁移规则、记忆注入规则、SKILL.md frontmatter 规范、workspace 隔离语义。
- 测试：store 层 CRUD + 迁移；memory 注入格式；skills 扫描/触发匹配；server 会话 REST。全量 `go test` 后停服务重编译二进制。

## 实施顺序（已建 Todo 表，逐项完成标记）

1. SQLite 存储层+迁移 → 2. 会话 API 改造 → 3. 记忆系统 → 4. AGENTS/CLAUDE 注入 → 5. 技能系统 → 6. 会话侧栏前端 → 7. 设置页记忆/技能 → 8. 文档 → 9. 全量测试+重编译