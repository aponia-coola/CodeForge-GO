# CodeForge-Go

**Chat with an AI coding agent in your browser. It reads, writes and runs commands inside your project directory.**

Single static binary · Zero runtime dependencies · Windows / Linux / Android Termux

[中文](README.md) | [中文详细版](README_ZH.md)

<!-- SCREENSHOT:PC -->
> **Screenshot placeholder** — desktop UI (`test/cf-screenshot-chat.png`)

<!-- SCREENSHOT:MOBILE -->
> **Screenshot placeholder** — Android Termux WebView (`test/phone.png`)

<!-- GIF:APPROVAL -->
> **GIF placeholder** — human-in-the-loop approval: an `Ask` decision suspends the operation until you decide

<!-- GIF:DIFF -->
> **GIF placeholder** — diff preview: green/red file changes, click to jump to the change

<!-- GIF:MOBILE -->
> **GIF placeholder** — mobile: sidebar collapses into a drawer on narrow screens, settings scroll horizontally

---

## Features

### Conversation & execution

- **Browser is the UI** — the same binary runs on a desktop, a headless server, and an Android phone; no graphical environment needed
- **16 built-in tools** — file read/write/edit/delete, directory listing, search, `run_command`, todo list, memory, skill creation, web fetch & search, subagent delegation, goal verification, tool-result read-back
- **Resume after interruption** — an interrupted turn is preserved; retry, edit-and-resend, or roll back the files along with it
- **Steer mid-run** — inject an instruction while the agent is working; it merges at the next step boundary

### Model access

- **Multiple providers** — OpenAI-compatible and Anthropic protocols
- **Model & provider libraries** — manage model entries and connection info from the UI; multiple models coexist
- **Multimodal** — image and video pass-through with tri-state capability declaration (unknown / supported / unsupported); upstream rejection rolls back the history automatically
- **Context-window probing** — measures the upstream's real accepted limit (1M down to 8K), not a hand-filled value in the model library
- **Rate limiting** — client-side RPM sliding window (slots requests before sending, never hits the wall) plus separate backoff for upstream 429s
- **Thinking level** — reasoning parameters built dynamically per upstream tier

### Concurrent sessions

- **Isolated per session** — separate run slots, undo stacks and read registries; no interference
- **Stable session switching** — switching a session follows its bound workspace
- **Project grouping** — the sidebar groups all sessions by workspace

### Subagents

- **Parallel delegation** — `delegate_subagents` splits work across multiple subagents, progress streams back live
- **Narrowed permissions** — write, delete and memory can be disabled individually; caps enforced in three places (Agent / scheduler / tool schema)
- **Independent, non-bypassable fence** — subagent scope checks apply on their own; even a human approval cannot punch through
- **Usage merged** — subagent tokens roll up into the parent session

### Goal mode

- **Mandatory verification** — the run must pass `goal_verify` before it can finish; failing means it keeps working instead of silently passing

### File safety

- **Workspace fence** — paths must stay inside the workspace; a symlink pointing outside is caught by resolving upwards link by link
- **Read before write** — files you never read cannot be overwritten from memory; if the file changed externally after reading, the fingerprint check refuses the write
- **CAS writes** — "read current → compare fingerprint → write" happens inside one per-path lock, so concurrent writes never silently overwrite each other
- **Atomic landing** — temp file in the same directory plus rename; a process killed mid-write leaves no half file
- **Byte-level undo** — snapshot before write, bucketed per session, restorable byte for byte
- **Checkpoint rollback** — roll back to any step, keyed by (session, step, path)

### Approval & policy

- **Three tiers** — Allow / Ask / Deny, switchable at runtime
- **Human-in-the-loop** — an `Ask` decision suspends the operation; approval timeout is separate from tool-execution timeout
- **Dangerous-command blacklist** — `rm -rf /` and friends are denied outright; highest priority, not overridable by rules
- **Full audit trail** — every tool call is written to an audit log (JSONL, rotates past 1 MiB, keeps 3 copies)
- **Capability surface** — hide selected tools from the model; hiding and forbidding are deliberately separate concerns

### Context

- **Online calibration** — no tokenizer dependency; the coefficient is derived from the real usage the upstream reports (only scaled up, never down — better to compress early)
- **Compression state persisted** — summary cursor and calibration factor survive restarts, so the "first request always overflows" trap doesn't repeat
- **Compression never loses history** — the full conversation is always retained; compression only changes the view sent to the model
- **Large results offloaded** — tool output over 8 KiB spills to disk as a reference (2 KiB preview stays in context); the model reads it back in chunks via `tool_result_read`
- **Usage visualisation** — a progress bar in the bottom bar shows live occupancy, with a manual compress trigger

### Interface

- **Hand-written single-file frontend** — no npm, no build step, embedded into the binary via `go:embed`
- **Dark/light themes + custom background** — background image, blur and brightness adjustable
- **Liquid glass toggle** — available in settings
- **Mobile layout** — sidebar becomes a drawer on narrow screens, settings scroll horizontally, visible height measured with `visualViewport`
- **Built-in picker** — a browser-style directory picker when no native dialog exists (Android Termux)

### Extensibility

- **MCP plugins** — stdio / HTTP / SSE transports, adapted into standard tools
- **WASM plugins** — run WASM plugins through wazero
- **Skill system** — `<workspace>/.codeforge/skills/<name>/SKILL.md` with frontmatter

### Deployment

- **Single static binary** — `CGO_ENABLED=0`, 16 MB; copying one file is the entire deployment
- **Three-platform cross builds** — Windows x64 / Linux amd64 / Android Termux arm64
- **Zero runtime dependencies** — 4 direct dependencies: WebSocket · WASM · YAML · system-call wrappers
- **Pure-Go SQLite** — `modernc.org/sqlite`, no C compiler needed, so static builds are never blocked
- **Access token** — a random token is generated when listening on a non-loopback address, delivered via HttpOnly cookie, validated by both API and WebSocket

---

## Engineering scale

| | |
|---|---|
| Go code | 27.5k lines of production code + 29.3k lines of tests, 13 packages |
| Tests | 138 test files, 76%~87% coverage in core packages |
| Direct dependencies | 4 (`gorilla/websocket` · `wazero` · `yaml.v3` · `golang.org/x/sys`) |
| Frontend | Hand-written single-file JS, zero build step |

---

## Quick start

```bash
git clone https://github.com/raine-x/CodeForge-GO.git
cd CodeForge-GO

export CODEFORGE_API_KEY=sk-xxx      # read from the environment only; the program never reads key files
go build -o bin/codeforge ./cmd/agent
./bin/codeforge -no-open
```

Open <http://127.0.0.1:8420>, pick a workspace, start chatting.

Cross-compile for all three platforms:

```bash
CGO_ENABLED=0 GOOS=windows GOARCH=amd64   go build -trimpath -ldflags "-s -w" -o bin/codeforge-windows-amd64.exe ./cmd/agent
CGO_ENABLED=0 GOOS=linux   GOARCH=amd64   go build -trimpath -ldflags "-s -w" -o bin/codeforge-linux-amd64 ./cmd/agent
CGO_ENABLED=0 GOOS=android GOARCH=arm64   go build -trimpath -ldflags "-s -w" -o bin/codeforge-android-arm64 ./cmd/agent
```

Full configuration, key precedence and per-platform deployment: **[中文详细版](README_ZH.md)**.

---

## License

**GPL-3.0 with additional restrictions (no commercial use; commercial use requires written permission).**

Non-standard SPDX license — see section 2 and section 6 of [LICENSE](LICENSE).
