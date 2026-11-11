# CodeForge-Go

**在浏览器里和 AI 编码智能体对话。它在你的项目目录里读写文件、跑命令、调模型。**

单文件静态二进制 · 零运行时依赖 · 25~36 MB 常驻 · Windows / Linux / Android Termux

[English](README_EN.md) | [中文详细版](README_ZH.md)

<!-- SCREENSHOT:PC -->
> **截图占位** —— 桌面端主界面（`test/cf-screenshot-chat.png`）

<!-- SCREENSHOT:MOBILE -->
> **截图占位** —— Android Termux WebView（`test/phone.png`）

<!-- GIF:APPROVAL -->
> **动图占位** —— 人工审批：策略判定 `Ask` 时操作挂起，等你决定

<!-- GIF:DIFF -->
> **动图占位** —— diff 预览：文件改动绿增红减，点击跳转改动位置

<!-- GIF:MOBILE -->
> **动图占位** —— 移动端：窄屏侧栏变抽屉，设置页改横向滚动

---

## 功能

### 对话与执行

- **浏览器即界面** —— 同一份二进制跑在桌面、无 GUI 服务器、安卓手机上，不需要图形环境
- **16 个内置工具** —— 文件读写改删、目录列举、搜索、`run_command`、任务清单、记忆、技能创建、网页抓取与搜索、子智能体委派、目标校验、工具结果回读
- **断点续跑** —— 打断后保留未完成轮次，可重试、编辑重发、连带回滚文件
- **运行中插话** —— 任务执行中途追加指令，在步骤边界并入当前轮

### 多模型接入

- **多供应商** —— OpenAI 兼容协议 + Anthropic 协议
- **模型库与供应商库** —— 界面上管理模型条目与连接信息，支持多模型并存
- **多模态** —— 图片与视频透传，能力三态声明（未知 / 支持 / 不支持），上游拒收时自动回滚历史
- **上下文窗口探测** —— 自动实测上游真正接受的上下文上限（1M 逐档降到 8K），不靠模型库手填
- **限流** —— 客户端 RPM 预约式滑动窗口（发请求前错开，从不撞墙）+ 上游 429 独立退避
- **思考强度** —— 按上游分级动态构建思考参数

### 多会话并发

- **每会话独立** —— 独立的运行槽、撤销栈、阅读登记，互不干扰
- **会话切换稳定** —— 切换会话自动跟随其绑定的工作区
- **项目分组** —— 侧栏按工作区分组展示全部会话

### 子智能体

- **并行委派** —— `delegate_subagents` 把任务拆给多个子智能体同时处理，进度实时回传
- **权限收窄** —— 可分别禁用写入、删除、记忆，能力上限三处同步（Agent / 调度器 / 工具 schema）
- **围栏独立且不可穿透** —— 子智能体的越界检查独立生效，人工审批也穿透不了
- **用量归并** —— 子智能体的 token 计入父会话

### 目标模式

- **强制校验** —— 完成前必须通过 `goal_verify`，不通过就继续做，不会静默放过

### 文件安全

- **工作区围栏** —— 路径必须落在工作区内，区内的软链指向区外会被逐级向上解析挡住
- **先读后写** —— 没读过的文件不允许凭记忆改写；读取后若被外部改动，指纹比对会发现并拒绝
- **CAS 写入** —— 「读当前 → 比对指纹 → 写」在同一把按路径的锁内完成，并发写不会静默互相覆盖
- **原子落盘** —— 同目录临时文件 + rename，进程写一半被杀不会留半截文件
- **原子级撤销** —— 写前快照，按会话分桶，可逐字节还原
- **检查点回滚** —— 按 (会话, 步骤, 路径) 精确回滚到任一步之前

### 审批与策略

- **三级策略** —— Allow / Ask / Deny，可运行期热切换
- **人工审批（HITL）** —— 命中 `Ask` 的操作挂起等决定，审批超时独立于工具执行超时
- **危险命令黑名单** —— `rm -rf /` 一类直接拒绝，优先级最高，规则表改不动
- **全量审计** —— 每次工具调用写审计日志（JSONL，超 1 MiB 轮转，保留 3 份）
- **能力可见面** —— 可隐藏部分工具不发给模型；隐藏与禁止刻意分离

### 上下文

- **在线校准** —— 无 tokenizer 依赖，用上游返回的真实 usage 反推估算系数（只放大不缩小，宁可早压）
- **压缩状态落盘** —— 摘要游标与校准系数持久化，重启后复用，不重复踩「第一次就炸上下文」
- **压缩永不丢历史** —— 完整对话始终保留，压缩只改送模视图
- **大结果外置** —— 工具输出超 8 KiB 落盘留引用（上下文里留 2 KiB 预览），模型用 `tool_result_read` 分段取回
- **占用可视化** —— 底部栏进度条实时显示占用，可手动触发压缩

### 界面

- **手写单文件前端** —— 无 npm、无构建步骤，`go:embed` 打进二进制
- **深浅色主题 + 自定义背景图** —— 背景图、模糊度、亮度可调
- **液态玻璃开关** —— 设置页可开
- **移动端适配** —— 窄屏侧栏变抽屉、设置页改横向滚动、用 `visualViewport` 实测可见高度
- **内置选择器** —— 无原生对话框时（安卓 Termux）用浏览器式目录选择器

### 扩展

- **MCP 插件** —— stdio / HTTP / SSE 三种传输，适配成标准工具接入
- **WASM 插件** —— 用 wazero 跑 WASM 插件
- **技能系统** —— `<workspace>/.codeforge/skills/<name>/SKILL.md`，带 frontmatter

### 部署

- **单文件静态二进制** —— `CGO_ENABLED=0`，产物 16 MB，拷贝一个文件就是完整部署
- **三平台交叉编译** —— Windows x64 / Linux amd64 / Android Termux arm64
- **零运行时依赖** —— 4 个直接依赖：WebSocket · WASM · YAML · 系统调用封装
- **纯 Go SQLite** —— `modernc.org/sqlite`，不需要 C 编译器，所以静态编译不受阻
- **访问令牌** —— 监听非回环地址时生成随机令牌，经 HttpOnly Cookie 下发，API 与 WebSocket 都校验

---

## 工程规模

| | |
|---|---|
| Go 代码 | 27.5k 行生产代码 + 29.3k 行测试，13 个包 |
| 测试 | 138 个测试文件，核心包覆盖率 76%~87% |
| 直接依赖 | 4 个（`gorilla/websocket` · `wazero` · `yaml.v3` · `golang.org/x/sys`） |
| 前端 | 手写单文件 JS，零构建步骤 |

---

## 快速开始

```bash
git clone https://github.com/raine-x/CodeForge-GO.git
cd CodeForge-GO

export CODEFORGE_API_KEY=sk-xxx      # 只从环境变量读，程序不读密钥文件
go build -o bin/codeforge ./cmd/agent
./bin/codeforge -no-open
```

浏览器打开 `http://127.0.0.1:8420`，选择工作区，开始对话。

交叉编译三平台：

```bash
CGO_ENABLED=0 GOOS=windows GOARCH=amd64   go build -trimpath -ldflags "-s -w" -o bin/codeforge-windows-amd64.exe ./cmd/agent
CGO_ENABLED=0 GOOS=linux   GOARCH=amd64   go build -trimpath -ldflags "-s -w" -o bin/codeforge-linux-amd64 ./cmd/agent
CGO_ENABLED=0 GOOS=android GOARCH=arm64   go build -trimpath -ldflags "-s -w" -o bin/codeforge-android-arm64 ./cmd/agent
```

完整配置、密钥优先级、各平台部署见 **[中文详细版](README_ZH.md)**。

---

## 许可

**GPL-3.0 + 附加限制（禁止商业使用，商业使用须书面授权）。**

非标准 SPDX 许可，详见 [LICENSE](LICENSE) 第 2 条与第 6 条。
