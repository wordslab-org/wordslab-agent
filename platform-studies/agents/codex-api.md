# Codex API Analysis — Agent Capabilities (Codex)

> **Docs:** `https://learn.chatgpt.com/codex` | **Product:** `https://openai.com/codex/for-work/`
> **Auth:** ChatGPT login (CLI/desktop), `OPENAI_API_KEY` / `CODEX_API_KEY` (`codex exec`), Workspace Agent access tokens (`api.chatgpt.com`)
> **Packages:** `@openai/codex-sdk` (npm) · `openai-codex` (pip) | **CLI:** `codex` (Rust binary; `codex mcp-server`, `codex app-server`, `codex exec`)
> **Open source:** [openai/codex](https://github.com/openai/codex) (`codex-rs/app-server`)
> **Description:** Codex exposes agent capabilities through a **local-first agent runtime** — an open-source Rust binary (`codex`) that manages the agent loop (model calls, sandboxed command/file execution, approvals, streaming events) on the developer's machine or in OpenAI-managed cloud containers. Unlike the hosted Agents SDK, Codex is a *productized coding agent*: configuration lives in `config.toml` + `AGENTS.md`, execution is sandboxed at the OS level (Seatbelt/bubblewrap/Windows sandbox), and a JSON-RPC **app-server** protocol lets you embed the full Codex experience into your own product. Four integration surfaces of increasing depth — `codex exec` (non-interactive CLI), the **Codex SDK** (TypeScript/Python library), `codex app-server` (JSON-RPC protocol), and `codex mcp-server` (Codex-as-MCP-tool for multi-agent orchestration) — cover automation, embedding, deep integration, and orchestration. The platform is organized around eleven capability areas: execution surfaces, agent configuration, the agent loop (threads/turns/items), sandboxing & permissions, approvals & human review, models & reasoning, skills & plugins/apps, MCP integration, subagents & multi-agent, environments (local/worktree/cloud), and long-running/scheduled work plus the hosted Workspace Agents trigger API.

---

## Table of Contents

1. [Platform Overview & Core Concepts](#1-platform-overview--core-concepts)
2. [Execution Surfaces & Integration Paths](#2-execution-surfaces--integration-paths)
3. [Agent Configuration — AGENTS.md, config.toml, Custom Agents](#3-agent-configuration--agentsmd-configtoml-custom-agents)
4. [The Agent Loop — Threads, Turns, Items & Events](#4-the-agent-loop--threads-turns-items--events)
5. [Sandboxing & Permission Profiles](#5-sandboxing--permission-profiles)
6. [Approvals & Human Review (incl. Auto-review)](#6-approvals--human-review-incl-auto-review)
7. [Models & Reasoning Effort](#7-models--reasoning-effort)
8. [Skills, Plugins & Apps (Connectors)](#8-skills-plugins--apps-connectors)
9. [MCP Integration](#9-mcp-integration)
10. [Subagents & Multi-Agent Orchestration](#10-subagents--multi-agent-orchestration)
11. [Environments — Local, Worktree, Cloud](#11-environments--local-worktree-cloud)
12. [Long-running Work, Scheduled Tasks & Workspace Agents](#12-long-running-work-scheduled-tasks--workspace-agents)
13. [Capability Summary & Cross-Reference](#13-capability-summary--cross-reference)

---

## 1. Platform Overview & Core Concepts

### Main Concepts

Codex's agent platform is built around these core abstractions:

- **Thread** — A conversation between a user and the Codex agent. Threads contain turns, persist as SQLite-backed rollouts (`session-*.jsonl`), and can be resumed, forked, archived, named, and goal-tagged.
- **Turn** — A single user request plus the agent work that follows. A turn streams incremental updates (`item/started`, `item/completed`, deltas) and ends with `turn/completed` carrying a `status` of `completed`, `interrupted`, or `failed`.
- **Item** — A unit of input or output within a turn: `userMessage`, `agentMessage`, `reasoning`, `commandExecution`, `fileChange`, `mcpToolCall`, `dynamicToolCall`, `webSearch`, `plan`, `contextCompaction`, review-mode markers. Tagged union (`ThreadItem`).
- **Sandbox** — An OS-level boundary (macOS Seatbelt, Linux bubblewrap+seccomp/Landlock, Windows sandbox/WSL2) constraining *spawned commands* (git, package managers, test runners), not just built-in file ops. Reduces approval fatigue with a clear trust model.
- **Approval** — A server-initiated JSON-RPC request that pauses a turn before a side effect (command exec, file change, network request, MCP/app tool, permission escalation). The client responds with a decision (`accept`/`acceptForSession`/`decline`/`cancel`, or an execpolicy amendment).
- **Auto-review** — A separate *reviewer agent* that decides approval requests in place of a human when `approvals_reviewer = "auto_review"`. Swaps the reviewer, never grants broader permissions.
- **Permission Profile** — A named least-privilege policy combining filesystem rules (`read`/`write`/`deny` per path) and network rules (domain allow/deny + proxy). Beta replacement for the older `sandbox_mode` settings.
- **Skill** — A filesystem-based resource (`SKILL.md` + supporting files, optional `SKILL.json` with `interface`/`dependencies`) supplying domain expertise on demand via progressive disclosure. Pre-built or custom; invoked with `$<skill-name>`.
- **Plugin / App** — A packaged extension connecting Codex to team tools/data (Slack, Google Drive, GitHub…). Plugins ship bundled skills/apps/MCP servers; apps are connectors with per-tool approval controls.
- **Subagent** — A delegated agent spawned in parallel by the main thread for bounded work; runs in its own *agent thread*, returns a summary. Controlled by `agents.max_threads` (default 6) and `agents.max_depth` (default 1).
- **Environment** — Where a task runs: **Local** (current project dir on your machine), **Worktree** (isolated Git worktree on your machine), or **Cloud** (OpenAI-managed container with repo checkout, setup script, two-phase network).
- **Goal** — A long-running task target set with `/goal`; Codex tracks progress, supports pause/resume/edit, and keeps shared context in one thread.
- **`AGENTS.md`** — Layered instruction files (global `~/.codex/AGENTS.md` → project root → cwd) that Codex concatenates into a system prompt at run start. `AGENTS.override.md` takes precedence; fallback filenames configurable.

### Agent Capabilities Map

| Capability | Description | Docs |
|------------|-------------|------|
| **Execution surfaces** | `codex exec`, Codex SDK, app-server JSON-RPC, MCP server — four integration depths | [developers](https://learn.chatgpt.com/codex/developers) |
| **Agent configuration** | `config.toml` + `AGENTS.md` layering + custom agent TOML files | [config](https://learn.chatgpt.com/codex/config-file/config-reference) · [agents-md](https://learn.chatgpt.com/codex/agent-configuration/agents-md) |
| **Agent loop** | Threads → turns → items; streaming events; fork/resume/archive; compaction | [app-server](https://learn.chatgpt.com/codex/app-server) |
| **Sandboxing** | OS-level sandbox modes (`read-only`/`workspace-write`/`danger-full-access`) + permission profiles | [sandboxing](https://learn.chatgpt.com/codex/sandboxing) · [permissions](https://learn.chatgpt.com/codex/permissions) |
| **Approvals** | Server-initiated approval requests; `untrusted`/`on-request`/`never` policies; auto-review | [approvals](https://learn.chatgpt.com/codex/agent-approvals-security) · [auto-review](https://learn.chatgpt.com/codex/sandboxing/auto-review) |
| **Models & reasoning** | Per-turn/per-agent model + `model_reasoning_effort`; model/list discovery | [models](https://learn.chatgpt.com/codex/models) |
| **Skills, plugins & apps** | Filesystem skills, plugin marketplaces, connector apps with per-tool approval | [skills-and-plugins](https://learn.chatgpt.com/codex/skills-and-plugins) |
| **MCP** | Local/remote MCP servers as tools; Codex-as-MCP-server for orchestration | [mcp](https://learn.chatgpt.com/codex/extend/mcp) · [mcp-server](https://learn.chatgpt.com/codex/mcp-server) |
| **Subagents** | Parallel delegated agents; custom agent files; CSV batch fan-out | [subagents](https://learn.chatgpt.com/codex/agent-configuration/subagents) |
| **Environments** | Local, Git worktree, Cloud container execution | [environments](https://learn.chatgpt.com/codex/environments/modes) |
| **Long-running & scheduled** | Goal mode, scheduled (RRULE) tasks, Workspace Agents trigger API | [long-running](https://learn.chatgpt.com/codex/long-running-work) · [automations](https://learn.chatgpt.com/codex/automations) |

### Platform Architecture

```
Developer / CI / embedded product
        │
        ▼
   codex binary (Rust, open source)
        │
        ├── codex exec ............... non-interactive CLI (stdout final msg / --json JSONL)
        ├── Codex SDK ................ @openai/codex-sdk / openai-codex (wraps app-server JSON-RPC)
        ├── codex app-server ......... JSON-RPC 2.0 (stdio | ws | unix), embeddable protocol
        └── codex mcp-server ........ exposes `codex` + `codex-reply` tools to MCP clients
        │
        ▼
   ┌──────────────── Agent Loop ────────────────┐
   │  thread/start → turn/start                  │
   │  model call → inspect → tool calls          │
   │  commandExecution / fileChange / mcpToolCall│
   │  approvals pause → client decision → resume │
   │  stream item/* + deltas → turn/completed   │
   └─────────────────────────────────────────────┘
        │  sandboxed at OS level (Seatbelt/bwrap/Win)
        ▼
   Thread rollout (SQLite, session-*.jsonl)
     ├── items, turns, diffs, plans, token usage
     ├── resume / fork / archive / name / goal
     └── subagent threads (max_threads=6, max_depth=1)
```

---

## 2. Execution Surfaces & Integration Paths

Codex offers four integration surfaces of increasing depth. Choose by how much control you need versus how much of Codex you want to reuse.

### Choosing a surface

| Surface | Best for | Loop ownership | Transport |
|---------|----------|---------------|----------|
| **`codex exec`** | Scripts, CI, pipelines — one-shot or resumable runs | Codex owns the loop; you get final output / JSONL stream | Process (stdout/stderr/stdin) |
| **Codex SDK** | Programmatic control from an app (CI/CD, internal tools, your own agent) | Library wraps the app-server; you drive threads/turns | In-process library (Node 18+ / Py 3.10+) |
| **`codex app-server`** | Deep embedding into a product (auth, history, approvals, streamed events) | You own the client; full JSON-RPC protocol | stdio (JSONL), WebSocket, Unix socket |
| **`codex mcp-server`** | Codex as one specialist inside a broader orchestrated workflow | An MCP client (e.g. Agents SDK) orchestrates | stdio MCP |

### `codex exec` — non-interactive mode

**Docs:** [non-interactive-mode](https://learn.chatgpt.com/codex/non-interactive-mode) · **Command ref:** `codex exec`

Streams progress to `stderr`, prints final agent message to `stdout`. Supports stdin piping (prompt-plus-stdin, or `codex exec -` to read prompt from stdin).

| Flag / behavior | Description |
|----------------|-------------|
| `codex exec "<prompt>"` | Run a task; final message to stdout |
| `--json` | stdout becomes JSONL event stream (`thread.started`, `turn.started`, `item.*`, `turn.completed`, `turn.failed`, `error`) |
| `--output-schema ./schema.json` | Constrain final response to a JSON Schema (structured output) |
| `-o` / `--output-last-message <path>` | Write final message to file (still printed to stdout) |
| `--sandbox <mode>` | `read-only` (default) · `workspace-write` · `danger-full-access` |
| `--ephemeral` | Don't persist session rollout files |
| `--skip-git-repo-check` | Run outside a Git repo (Codex normally requires one to prevent destructive changes) |
| `--ignore-user-config` / `--ignore-rules` | Skip `$CODEX_HOME/config.toml` / `.rules` execpolicy files for controlled automation |
| `codex exec resume --last` / `resume <SESSION_ID>` | Continue a previous run (two-stage pipelines) |
| `CODEX_API_KEY` env | API key scoped to a single invocation (only supported in `codex exec`) |

Auth guidance: use the [Codex GitHub Action](https://learn.chatgpt.com/codex/github-action) for GitHub Actions (starts a secure Responses API proxy) rather than passing keys via job-level env vars. Don't set `OPENAI_API_KEY`/`CODEX_API_KEY` as job-level env in workflows running repo-controlled code.

### Codex SDK — TypeScript & Python libraries

**Docs:** [codex-sdk](https://learn.chatgpt.com/codex/codex-sdk)

The SDK controls the local Codex app-server over JSON-RPC. More comprehensive than `codex exec`. Use server-side.

| Package | Install | Runtime |
|---------|---------|---------|
| `@openai/codex-sdk` | `npm install @openai/codex-sdk` | Node.js 18+ |
| `openai-codex` | `pip install openai-codex` (beta: `--pre` for prereleases) | Python 3.10+ (pinned Codex CLI runtime included) |

Core API (TypeScript):

| Method | Description |
|--------|-------------|
| `new Codex()` | Create a Codex client |
| `codex.startThread()` | Start a new thread |
| `codex.resumeThread(threadId)` | Resume a past thread by ID |
| `thread.run(prompt, opts?)` | Run a turn; returns result with `finalResponse` |
| `Sandbox.read_only` / `workspace_write` / `full_access` | Sandbox presets (passed to `thread_start` or `run`) |

Core API (Python):

| Method | Description |
|--------|-------------|
| `Codex()` / `AsyncCodex()` (context manager) | Sync / async client; auto-starts & stops the app-server |
| `codex.thread_start(model=..., sandbox=...)` | Start a thread |
| `thread.run(prompt, sandbox=?)` | Run a turn; `result.final_response` |
| `CodexConfig(codex_bin=...)` | Override the pinned Codex executable |

Sandbox passed to `run()`/`turn()` applies to that turn and later turns on the thread. Omitting `sandbox=` uses the app-server's configured default.

### `codex app-server` — JSON-RPC protocol

**Docs:** [app-server](https://learn.chatgpt.com/codex/app-server) · **Source:** [openai/codex/codex-rs/app-server](https://github.com/openai/codex/tree/main/codex-rs/app-server)

The interface that powers rich clients (e.g. the VS Code extension). Like MCP, uses JSON-RPC 2.0 (the `"jsonrpc":"2.0"` header omitted on the wire). Generate version-specific schemas with `codex app-server generate-ts --out ./schemas` / `generate-json-schema --out ./schemas`.

| Transport | `--listen` | Notes |
|-----------|-----------|-------|
| stdio (default) | `stdio://` | Newline-delimited JSON (JSONL) |
| WebSocket | `ws://IP:PORT` | One message per text frame; experimental/unsupported; bounded queues reject with code `-32001` when full |
| Unix socket | `unix://` / `unix://PATH` | WebSocket over the default control socket or a custom path |
| off | `off` | No local transport |

Remote terminal UI: `codex app-server --listen ws://127.0.0.1:4500` then `codex --remote wss://...`. Health probes: `GET /readyz`, `GET /healthz` (rejects `Origin` headers with 403). WebSocket auth flags: `--ws-auth capability-token --ws-token-file <path>` / `--ws-token-sha256 HEX` / `--ws-auth signed-bearer-token --ws-shared-secret-file <path>` (plus `--ws-issuer`/`--ws-audience`/`--ws-max-clock-skew-seconds`).

Connection lifecycle: send `initialize` (with `clientInfo`) → emit `initialized` notification → call `thread/start` → drive `turn/start` → read notifications. Requests before init return `Not initialized`; repeated `initialize` returns `Already initialized`.

`initialize.params.capabilities`:
- `experimentalApi` (bool) — gate experimental methods/fields
- `optOutNotificationMethods` — exact notification names to suppress (no wildcards)
- `requestAttestation` — opt into server-initiated `attestation/generate`
- `mcpServerOpenaiFormElicitation` — allow OpenAI extended-form MCP elicitation

> **Compliance:** `clientInfo.name` identifies your client for the OpenAI Compliance Logs Platform. New enterprise integrations should contact OpenAI to be added to the known clients list.

### `codex mcp-server` — Codex as an MCP tool

**Docs:** [mcp-server](https://learn.chatgpt.com/codex/mcp-server)

Run with `codex mcp-server`; expose two tools to MCP clients (e.g. the OpenAI Agents SDK). Lets Codex be one specialist in a broader orchestrated workflow.

| Tool | Required params | Description |
|------|-----------------|-------------|
| `codex` | `prompt` | Start a Codex session with config overrides (`approval-policy`, `base-instructions`, `compact-prompt`, `config`, `cwd`, `developer-instructions`, `model`, `sandbox`) |
| `codex-reply` | `prompt`, `threadId` | Continue a session by thread ID (returns `structuredContent.threadId` + `content`) |

`approval-policy` values: `untrusted` · `on-request` · `never`. `sandbox` values: `read-only` · `workspace-write` · `danger-full-access`. Approval prompts (exec/patch) include `threadId` in `params`. Response carries both `structuredContent` (modern clients) and `content` (legacy clients).

---

## 3. Agent Configuration — AGENTS.md, config.toml, Custom Agents

Codex configuration is layered: a global/home `config.toml`, project `.codex/config.toml`, `AGENTS.md` instruction files, and standalone custom agent files. See [Configuration](https://learn.chatgpt.com/codex/configuration) and [AGENTS.md](https://learn.chatgpt.com/codex/agent-configuration/agents-md).

### `AGENTS.md` discovery

Codex builds an instruction chain once per run (per TUI session). Precedence:

1. **Global scope** — `~/.codex/` (or `$CODEX_HOME`): reads `AGENTS.override.md` if present, else `AGENTS.md`. First non-empty file only.
2. **Project scope** — from project root (Git root) walking down to cwd; in each directory checks `AGENTS.override.md`, then `AGENTS.md`, then `project_doc_fallback_filenames`. At most one file per directory.
3. **Merge** — concatenated root-down with blank lines; later (closer to cwd) overrides earlier.

| Config key | Default | Description |
|------------|---------|-------------|
| `project_doc_fallback_filenames` | `[]` | Extra filenames treated as instruction files (e.g. `TEAM_GUIDE.md`) |
| `project_doc_max_bytes` | `32 KiB` | Combined instruction size cap before truncation |
| `CODEX_HOME` env | `~/.codex` | Codex home directory (profiles, config, sessions) |

### Custom agents (subagent definitions)

**Docs:** [subagents](https://learn.chatgpt.com/codex/agent-configuration/subagents)

Standalone TOML files under `~/.codex/agents/` (personal) or `.codex/agents/` (project-scoped). Each file defines one custom agent as a configuration layer over a normal session config.

| Field | Required | Description |
|-------|----------|-------------|
| `name` | Yes | Agent name used when spawning/referring |
| `description` | Yes | Human-facing guidance for when to use this agent |
| `developer_instructions` | Yes | Core behavior instructions |
| `nickname_candidates` | No | Pool of display nicknames (ASCII letters/digits/space/-/_) |
| `model`, `model_reasoning_effort`, `sandbox_mode`, `mcp_servers`, `skills.config` | No | Inherit from parent session if omitted |

Built-in agents: `default` (general fallback), `worker` (execution/fixes), `explorer` (read-heavy exploration). A custom agent with a built-in name takes precedence.

### Global subagent settings (`[agents]` in `config.toml`)

| Field | Type | Default | Purpose |
|-------|------|---------|---------|
| `agents.max_threads` | number | `6` | Concurrent open agent thread cap |
| `agents.max_depth` | number | `1` | Spawned agent nesting depth (root = 0); default prevents deeper descendants |
| `agents.job_max_runtime_seconds` | number | `1800` (per-call fallback) | Default per-worker timeout for `spawn_agents_on_csv` |
| `agents.interrupt_message` | boolean | `true` | Record model-visible message when an agent turn is interrupted |

### Other key config surfaces

| Area | Keys | Docs |
|------|------|------|
| Sandbox/approval | `sandbox_mode`, `approval_policy`, `approvals_reviewer`, `sandbox_workspace_write.*`, `[permissions.*]` | §5, §6 |
| Models | `model`, `model_reasoning_effort` | §7 |
| MCP | `[mcp_servers.<name>]` (`url`/`command`/`args`) | §9 |
| Skills | `[[skills.config]]` (`path`, `enabled`) | §8 |
| Apps | `[apps.<id>]` (`enabled`, `destructive_enabled`, `approvals_reviewer`, `default_tools_approval_mode`, `[apps.<id>.tools.<tool>]`) | §8 |
| Profiles | `~/.codex/<name>.config.toml`, selected via `codex --profile <name>` | §6 |
| Telemetry | `[otel]` (`environment`, `exporter`, `log_user_prompt`) | opt-in, off by default |

---

## 4. The Agent Loop — Threads, Turns, Items & Events

The app-server protocol is the canonical description of the Codex loop. Core primitives: **Thread** (conversation) → **Turn** (user request + agent work) → **Item** (input/output unit). Drive with thread APIs, run with turn APIs, stream progress via notifications.

### Thread methods

| Method | Description |
|--------|-------------|
| `thread/start` | Create a new thread; emits `thread/started`; auto-subscribes to turn/item events |
| `thread/resume` | Reopen an existing thread by ID so later `turn/start` calls append |
| `thread/fork` | Copy stored history into a new thread ID (`lastTurnId` copies through that turn); emits `thread/started`; result includes `forkedFromId` |
| `thread/read` | Read a stored thread by ID without resuming; `includeTurns` for full history; includes runtime `status` |
| `thread/list` | Cursor-paginate stored threads; filters: `modelProviders`, `sourceKinds`, `archived`, `cwd`, `useStateDbOnly`, `searchTerm`, experimental `parentThreadId`/`ancestorThreadId` |
| `thread/turns/list` (exp) | Page through a thread's turn history; `itemsView`: omit/summarize/fully load |
| `thread/items/list` (exp) | Page through persisted thread items, optional `turnId` filter |
| `thread/loaded/list` | Thread IDs currently loaded in memory |
| `thread/name/set` | Set/update user-facing name; emits `thread/name/updated` |
| `thread/goal/set` · `goal/get` · `goal/clear` | Goal management; emits `thread/goal/updated` / `goal/cleared` |
| `thread/metadata/update` | Patch SQLite-backed metadata (currently `gitInfo`) |
| `thread/archive` · `unarchive` · `delete` | Archive (move to archived dir, cascade descendants) / restore / permanent delete; emit `thread/archived`/`unarchived`/`deleted` |
| `thread/unsubscribe` | Unsubscribe; if last subscriber, unloads after grace period → `thread/closed` |
| `thread/compact/start` | Trigger history compaction; returns `{}` immediately, progress via `turn/*` + `item/*` |
| `thread/shellCommand` | Run a user-initiated shell command (outside sandbox, full access, no thread sandbox inheritance) |
| `thread/backgroundTerminals/*` (exp) | `clean` / `list` / `terminate` background terminals |
| `thread/inject_items` | Append raw Responses API items to history without starting a user turn |
| `thread/rollback` (deprecated) | Drop last N turns from in-memory context; persist rollback marker |

### Turn methods

| Method | Description |
|--------|-------------|
| `turn/start` | Add user input to a thread and begin generation; responds with initial `turn`, streams events. Optional fields override model, personality, `cwd`, sandbox policy, `collaborationMode`, `dynamicTools`. For `collaborationMode`, `settings.developer_instructions: null` = use built-in instructions for the selected mode. |
| `turn/steer` | Append user input to the active in-flight turn (no new turn); returns accepted `turnId` |
| `turn/interrupt` | Request cancellation; success `{}`; turn ends `status: "interrupted"` |
| `review/start` | Kick off the Codex reviewer; emits `enteredReviewMode` + `exitedReviewMode` items |

`turn/start` input items: `text`, `image`, `localImage`, `skill` (recommended — injects full skill instructions), `mention` (for apps, path `app://<id>`).

### Turn/streaming events (notifications)

| Event | Payload |
|-------|---------|
| `turn/started` | `{ turn }` (id, empty `items`, `status: "inProgress"`) |
| `turn/completed` | `{ turn }` (`status: completed\|interrupted\|failed`; failures carry `error.message`/`codexErrorInfo`/`additionalDetails`) |
| `turn/diff/updated` | `{ threadId, turnId, diff }` aggregated unified diff across the turn's file changes |
| `turn/plan/updated` | `{ turnId, explanation?, plan }` — each entry `{ step, status }` (`pending`/`inProgress`/`completed`) |
| `thread/tokenUsage/updated` | Usage updates for the active thread |
| `hook/started` · `hook/completed` | `{ threadId, turnId?, run }` lifecycle hook boundaries |
| `model/safetyBuffering/updated` | Transient safety buffering (`model`, `useCases`, `reasons`, `showBufferingUi`, `fasterModel`) |
| `model/rerouted` | `{ fromModel, toModel, reason }` |
| `model/verification` | Additional account verification required |

### Item lifecycle & deltas

All items emit `item/started` (full item, `item.id` = delta `itemId`) and `item/completed` (authoritative final state).

| Item type | Key fields |
|-----------|-----------|
| `userMessage` | `id`, `content` (list of `text`/`image`/`localImage`) |
| `agentMessage` | `id`, `text`, `phase?` (`commentary`/`final_answer` Responses API wire values) |
| `plan` | `id`, `text` (plan mode; final `item/completed` authoritative) |
| `reasoning` | `id`, `summary` (streamed summaries), `content` (raw reasoning blocks) |
| `commandExecution` | `id`, `command`, `cwd`, `status`, `commandActions`, `aggregatedOutput?`, `exitCode?`, `durationMs?` |
| `fileChange` | `id`, `changes` (`{path, kind, diff}`), `status` |
| `mcpToolCall` | `id`, `server`, `tool`, `status`, `arguments`, `appContext?` (`connectorId`/`linkId`/`resourceUri`/`appName`/`templateId`/`actionName`), `pluginId?`, `result?`, `error?` |
| `dynamicToolCall` | `id`, `tool`, `arguments`, `status`, `contentItems?`, `success?`, `durationMs?` (client-executed) |
| `collabToolCall` | `id`, `tool`, `status`, `senderThreadId`, `receiverThreadId?`, `newThreadId?`, `prompt?`, `agentStatus?` (subagent delegation) |
| `webSearch` | `id`, `query`, `action?` (`search`/`openPage`/`findInPage`) |
| `imageView` | `id`, `path` |
| `enteredReviewMode` / `exitedReviewMode` | `id`, `review` |
| `contextCompaction` | `id` (history compacted; replaces deprecated `thread/compacted`) |

Deltas: `item/agentMessage/delta`, `item/plan/delta`, `item/reasoning/summaryTextDelta` (+`summaryPartAdded`), `item/reasoning/textDelta`, `item/commandExecution/outputDelta`. (`item/fileChange/outputDelta` deprecated; use `fileChange` items + `turn/diff/updated`.)

### Errors

A failed turn emits an `error` event (`{ error: { message, codexErrorInfo?, additionalDetails? } }`) then `turn/completed` with `status: "failed"`. `codexErrorInfo` variants include `httpStatusCode` when upstream status is available: `ContextWindowExceeded`, `UsageLimitExceeded`, `HttpConnectionFailed`, `ResponseStreamConnectionFailed`, `ResponseStreamDisconnected`, `ResponseTooManyFailedAttempts`, `BadRequest`, `Unauthorized`, `SandboxError`, `InternalServerError`, `Other`.

### Other app-server methods (selected)

| Method | Description |
|--------|-------------|
| `command/exec` (+ `write`/`resize`/`terminate`, `outputDelta` notify) | Run a single command under the server sandbox without a thread/turn |
| `process/spawn` (+ `writeStdin`/`resizePty`/`kill`, `outputDelta`/`exited` notify) (exp) | Explicit process session outside Codex's sandbox |
| `model/list` | List available models + effort options / `inputModalities` / `upgrade` / `supportsPersonality` / `isDefault`; `includeHidden` to show hidden |
| `modelProvider/capabilities/read` | Provider capability bounds for model/provider combos |
| `experimentalFeature/list` · `enablement/set` | Feature flags with `stage` (`beta`/`underDevelopment`/`stable`/`deprecated`/`removed`); patch runtime settings (`apps`, `plugins`) |
| `permissionProfile/list` | Beta permission profiles + whether effective requirements allow them |
| `collaborationMode/list` (exp) | Collaboration mode presets |
| `environment/info` (exp) | Connect to a configured execution environment; return shell + default cwd |
| `config/read` · `config/value/write` · `config/batchWrite` | Effective config / single key / atomic batch edits to `config.toml` |
| `configRequirements/read` | `requirements.toml`/MDM allow-lists, pinned features, residency/network requirements |
| `externalAgentConfig/detect` · `import` | Discover & migrate artifacts from other agents (config, skills, `AGENTS.md`, plugins, MCP, subagents, hooks, commands, sessions) |
| `fs/*` (readFile/writeFile/createDirectory/getMetadata/readDirectory/remove/copy/watch) | App-server v2 filesystem API on absolute paths |
| `feedback/upload` | Submit feedback (classification + reason/logs + conversation id + optional `extraLogFiles`) |
| `windowsSandbox/setupStart` (+ `setupCompleted` notify) | Windows sandbox setup for `elevated`/`unelevated` mode |

---

## 5. Sandboxing & Permission Profiles

**Docs:** [sandboxing](https://learn.chatgpt.com/codex/sandboxing) · [permissions](https://learn.chatgpt.com/codex/permissions)

Codex sandboxes *spawned commands* (git, package managers, test runners), not just built-in file ops, using platform-native enforcement. This reduces approval fatigue and provides a clear trust model.

### Sandbox modes (`sandbox_mode`)

| Mode | Behavior |
|------|----------|
| `read-only` | Inspect files only; no edits/commands without approval |
| `workspace-write` | Read, edit within workspace + configured writable roots, run routine local commands (default low-friction) |
| `danger-full-access` | No sandbox restrictions (alias `--dangerously-bypass-approvals-and-sandbox` / `--yolo`) |

### Approval policies (`approval_policy`)

| Policy | Behavior |
|--------|----------|
| `untrusted` | Asks before non-trusted-set commands |
| `on-request` | Works in sandbox; asks to go beyond |
| `never` | No approval stops |

Granular policy (TOML): `approval_policy = { granular = { sandbox_approval, rules, mcp_elicitations, request_permissions, skill_approval } }`.

### Platform enforcement

| Platform | Mechanism |
|----------|-----------|
| macOS | Seatbelt (`sandbox-exec -p <profile>`) — works out of the box |
| Linux / WSL2 | `bubblewrap` + `seccomp` (Landlock fallback); requires `bubblewrap` package; Ubuntu 24.04 AppArmor fix |
| Native Windows | Windows sandbox (`elevated` strongest, `unelevated` fallback) or WSL2 Linux sandbox (WSL1 unsupported in 0.115+) |

Protected paths (recursive read-only inside writable roots): `<root>/.git` (or resolved gitdir), `<root>/.agents`, `<root>/.codex`.

Test locally: `codex sandbox macos|linux|windows [--permissions-profile <name>] [--log-denials] [COMMAND]...` (alias `codex debug`; `codex sandbox seatbelt`, `codex sandbox landlock`).

### Permission Profiles (beta)

A **profile** is a named least-privilege policy combining filesystem rules (`read`/`write`/`deny`) and network rules (domain allow/deny + proxy). Does **not** compose with older `sandbox_mode`/`sandbox_workspace_write` settings — use one or the other (managed `allowed_permission_profiles` is the exception forcing profiles).

| Config key | Description |
|------------|-------------|
| `default_permissions` | Name of the active profile |
| `[permissions.<name>]` | Define a named profile |
| `permissions.<name>.extends` | Inherit from `:read-only`/`:workspace`/named profile (rejects `:danger-full-access`, unknown parents, cycles) |
| `[permissions.<name>.workspace_roots]` | Profile-defined workspace roots (boolean per path) |
| `[permissions.<name>.filesystem]` | Paths → `read`/`write`/`deny`; `glob_scan_max_depth` limits deny-read glob expansion (≥1) |
| `[permissions.<name>.network]` | Network sandbox proxy + policy |
| `network.enabled` (default `false`) | Enable network for sandboxed commands |
| `network.domains."<pattern>"` | `allow`/`deny`; exact hosts, `*.example.com` (subdomains), `**.example.com` (apex+sub), `*` (allow-only wildcard); deny > allow |
| `network.unix_sockets."<path>"` | `allow`/`deny` absolute Unix socket paths |
| `network.proxy_url` (default `http://127.0.0.1:3128`) | HTTP proxy listener (HTTP_PROXY/HTTPS_PROXY/websocket/tool proxy) |
| `network.enable_socks5` (default `true`) | SOCKS5 listener (`socks_url` default `http://127.0.0.1:8081`) |
| `network.enable_socks5_udp` (default `true`) | UDP over SOCKS5 |
| `network.allow_upstream_proxy` (default `true`) | Honor upstream HTTP(S)_PROXY/ALL_PROXY |
| `network.allow_local_binding` (default `false`) | Disable local/private-network guard |
| `network.dangerously_allow_non_loopback_proxy` (default `false`) | Allow non-loopback proxy binding |
| `network.dangerously_allow_all_unix_sockets` (default `false`) | Bypass Unix socket allowlist |

**Filesystem access values:** `read` (read/list), `write` (read+modify+create+rename+delete), `deny`. Precedence: deny > write > read; more specific paths override broader; can reopen a narrower subtree inside a broader deny.

**Special filesystem paths:** `:root`, `:minimal` (platform/runtime common tool paths), `:workspace_roots` (session + profile roots, supports scoped subpaths), `:tmpdir`, `:slash_tmp`, `/absolute/path`, `~/path`. Nested subpaths must stay inside a workspace root; `..` rejected.

**Local/private destinations:** `allow_local_binding = false` (default) blocks loopback/link-local/private; add exact `localhost`/IP literal allow for exceptions (wildcards don't count). Best-effort DNS rebinding protection (failed/non-public lookups blocked).

Built-in profiles: `:read-only`, `:workspace`, `:danger-full-access`.

### Network proxy feature

```toml
[features.network_proxy]
enabled = true
domains = { "api.openai.com" = "allow", "example.com" = "deny" }
```

`network_proxy` enforces allowed network; it doesn't *grant* network. `sandbox_workspace_write.network_access` decides whether commands have network at all. CLI shorthand: `-c 'features.network_proxy=true'`. Admin `experimental_network` requirements are separate.

### Web search control

```toml
web_search = "cached"   # default (OpenAI-maintained index)
# web_search = "disabled"
# web_search = "live"     # same as --search
# web_search = "indexed"  # gated by search index
```

`--yolo`/full access defaults web search to live.

---

## 6. Approvals & Human Review (incl. Auto-review)

**Docs:** [agent-approvals-security](https://learn.chatgpt.com/codex/agent-approvals-security) · [auto-review](https://learn.chatgpt.com/codex/sandboxing/auto-review)

Two security layers: **sandbox mode** (technical capabilities) + **approval policy** (when to ask). Depending on settings, command execution and file changes may require approval via a server-initiated JSON-RPC request; the client responds with a decision.

### Approval decision payloads

| Decision type | Options |
|---------------|---------|
| Command execution | `accept` · `acceptForSession` · `decline` · `cancel` · `{ "acceptWithExecpolicyAmendment": { "execpolicy_amendment": ["cmd", "..."] } }` |
| File change | `accept` · `acceptForSession` · `decline` · `cancel` |

Requests include `threadId` and `turnId` to scope UI state. The server resumes or declines and ends the item with `item/completed`.

### Command execution approvals

1. `item/started` — pending `commandExecution` (`command`, `cwd`, …)
2. `item/commandExecution/requestApproval` — `itemId`, `threadId`, `turnId`, optional `reason`/`command`/`cwd`/`commandActions`/`proposedExecpolicyAmendment`/`networkApprovalContext`/`availableDecisions`; with `experimentalApi=true`, optional `additionalPermissions` (absolute paths) for per-command sandbox access
3. Client responds with a decision
4. `serverRequest/resolved` confirms the pending request is answered/cleared
5. `item/completed` — final `commandExecution` (`status: completed | failed | declined`)

`networkApprovalContext` (target `host` + `protocol`) means managed network access, not a general shell approval — render a network-specific prompt. Codex groups concurrent network prompts by destination (`host` + protocol + port); different ports on the same host are separate.

### File change approvals

1. `item/started` — `fileChange` with proposed `changes`, `status: "inProgress"`
2. `item/fileChange/requestApproval` — `itemId`, `threadId`, `turnId`, optional `reason`/`grantRoot`
3. Client responds → 4. `serverRequest/resolved` → 5. `item/completed` (`status: completed | failed | declined`)

### Permission requests (`request_permissions` tool)

The built-in `request_permissions` tool sends `item/permissions/requestApproval` with `threadId`, `turnId`, `itemId`, `environmentId`, `cwd`, optional `reason`, and requested network/filesystem permissions. Respond with `permissions` containing only the granted subset. `scope: "session"` persists the grant for later turns; omit or `"turn"` for turn-scoped. Unrequested permissions are ignored.

### MCP elicitation & app/tool approvals

- `mcpServer/elicitation/request` (server request) — interrupt a turn; `mode: "form"`/`"openai/form"` (`message` + `requestedSchema`) or `mode: "url"` (`message` + `url` + `elicitationId`). Respond `action: "accept"` + `content`, or `"decline"`/`"cancel"` + `content: null`. Opt into `openai/form` via `initialize.params.capabilities.mcpServerOpenaiFormElicitation`.
- MCP **app/connector** tool calls can require approval via `tool/requestUserInput` (Accept/Decline/Cancel). Destructive tool annotations always trigger approval even with less-privileged hints. Decline/cancel completes the `mcpToolCall` with an error.
- `tool/requestUserInput` — 1–3 short questions for a tool call (questions can set `isOther`); params include `autoResolutionMs` (ms timeout or `null`).

### Dynamic tool calls (experimental)

`dynamicTools` on `thread/start` + `item/tool/call` flow. Names must follow Responses API naming constraints; avoid reserved built-in namespaces. Flow: `item/started` (`dynamicToolCall`, inProgress) → `item/tool/call` (server request) → client returns content items → `item/completed` (final status, `contentItems`/`success`).

### Auto-review

Replaces manual approval at the sandbox boundary with a **reviewer agent**. The main agent still runs under the same sandbox/approval/network limits — only the reviewer changes. Only applies when approvals are interactive (`on-request` or granular); nothing to review with `never`.

| Aspect | Behavior |
|--------|----------|
| Trigger | Shell/exec requesting escalation; network blocked by sandbox/policy; file edits outside writable roots; MCP/app tool calls requiring approval; Computer Use new domain (Computer Use app approvals still surface to user directly) |
| What it blocks | Sending secrets to untrusted destinations; credential/cookie probing; broad/persistent security weakening; destructive irreversible actions |
| Reviewer visibility | Compact transcript + exact approval request (user messages, assistant updates, tool calls/outputs, proposed action); rare read-only checks; hidden assistant reasoning NOT included |
| Denial | Returns rationale + stronger instruction (no workarounds/circumvention; find materially safer alternative or stop and ask user) |
| Circuit breaker | Interrupts turn after **3 consecutive denials** or **10 denials** in the rolling window of last **50 reviews** in the same turn; any non-denial resets the consecutive counter |
| Override | `/approve` in TUI opens "Auto-review Denials" picker → retry one recent denied action (records up to **10 recent denials per task**); narrow (exact action, one retry, still goes through auto-review) |

Config:

```toml
approval_policy = "on-request"
approvals_reviewer = "auto_review"
[auto_review]
policy = "YOUR POLICY"   # managed guardian_policy_config takes precedence
```

Default policy at `codex-rs/core/src/guardian/policy.md` (open source). Reviewer statuses: Reviewing, Approved, Denied, Aborted, Timed out. Transcripts retained at `~/.codex/sessions`.

### Common sandbox/approval combinations

| Intent | Flags/config | Effect |
|--------|--------------|--------|
| Auto (preset) | `--sandbox workspace-write --ask-for-approval on-request` | Read/edit/run in workspace; approval outside/network |
| Safe read-only | `--sandbox read-only --ask-for-approval on-request` | Read + answer; approval for edits/commands/network |
| Read-only CI | `--sandbox read-only --ask-for-approval never` | Read only; never asks |
| Auto-edit + untrusted | `--sandbox workspace-write --ask-for-approval untrusted` | Read/edit; approval before untrusted commands |
| Auto-review | `--sandbox workspace-write --ask-for-approval on-request -c approvals_reviewer=auto_review` | Same boundary; reviewer handles approvals |
| Full access | `--dangerously-bypass-approvals-and-sandbox` (`--yolo`) | No sandbox; no approvals (not recommended) |

Profiles persist as `~/.codex/<name>.config.toml`, selected via `codex --profile <name>`. Auto-detects version control → `Auto` for VC folders, `read-only` for non-VC.

### Dev Containers & Telemetry

- Secure dev container: `github.com/openai/codex/tree/main/.devcontainer` (`devcontainer.secure.json`, `Dockerfile.secure`, `init-firewall.sh`).
- OpenTelemetry (opt-in, off by default): `[otel]` with `environment` (dev/staging/prod), `exporter` (`none`/`otlp-http`/`otlp-grpc`), `log_user_prompt` (redact unless policy allows). Event categories: `codex.conversation_starts`, `codex.api_request`, `codex.sse_event`, `codex.websocket_request/event`, `codex.user_prompt`, `codex.tool_decision`, `codex.tool_result`.

---

## 7. Models & Reasoning Effort

**Docs:** [models](https://learn.chatgpt.com/codex/models) · **App-server:** `model/list`

### Model choice

| Model | Use |
|-------|-----|
| `gpt-5.6` | Start here for demanding agents — ambiguous, multi-step work needing planning, tool use, validation, follow-through across larger context |
| `gpt-5.4` | Pinned-to-5.4 workflows; strong coding/reasoning/tool use/broader workflows |
| `gpt-5.6-terra` | Speed/efficiency over depth — exploration, read-heavy scans, large-file review, parallel workers returning distilled results |
| `gpt-5.3-codex-spark` | ChatGPT Pro research preview; near-instant text-only iteration when latency matters |

### Reasoning effort (`model_reasoning_effort`)

| Effort | Use |
|--------|-----|
| `ultra` | Deepest reasoning (supported models) |
| `max` / `xhigh` | Especially demanding reasoning (supported models) |
| `high` | Complex logic, assumption-checking, edge cases (reviewer/security agents) |
| `medium` | Balanced default |
| `low` | Straightforward tasks where speed matters |
| `minimal` / `none` | Little/no reasoning, lower latency (supported models) |

Higher effort increases response time and token usage but can improve quality for complex work.

### `model/list` response shape

```json
{ "method": "model/list", "id": 6, "params": { "limit": 20, "includeHidden": false } }
```

Each entry: `id`, `model`, `displayName`, `hidden`, `defaultReasoningEffort`, `supportedReasoningEfforts[]` (`{reasoningEffort, description}`), `inputModalities` (default `["text","image"]` if missing), `supportsPersonality`, `isDefault`, optional `upgrade`/`upgradeInfo`. Set `includeHidden: true` for the full list.

If you don't pin a model or effort, Codex chooses a setup balancing intelligence, speed, and price (may favor `gpt-5.6-terra` for fast scans or higher-effort `gpt-5.6` for demanding reasoning). Pin via prompt, `model`, or `model_reasoning_effort`.

---

## 8. Skills, Plugins & Apps (Connectors)

**Docs:** [skills-and-plugins](https://learn.chatgpt.com/codex/skills-and-plugins) · **App-server:** `skills/list`, `app/list`, `plugin/*`, `marketplace/*`

### Skills

Filesystem-based resources (`SKILL.md` + supporting files, optional `SKILL.json` with `interface`/`dependencies`) giving domain expertise on demand (progressive disclosure). Pre-built or custom. Invoke with `$<skill-name>` in user text; add a `skill` input item (recommended) so the server injects full instructions instead of relying on the model to resolve the name.

| Method | Description |
|--------|-------------|
| `skills/list` | List skills for one or more `cwds`; `forceReload` to refresh; `perCwdExtraUserRoots` to scan extra paths as `user` scope |
| `skills/config/write` | Enable/disable a skill by `path` |
| `skills/changed` (notify) | Watched local skill files changed → invalidation signal |
| `skills/extraRoots/set` | Replace process-level extra roots for standalone skill discovery (not persisted) |

`SKILL.json` `dependencies.tools[]`: `env_var` (`value`, `description`) and `mcp` (`value`, `transport`, `url`).

### Apps (connectors)

Connect Codex to team tools/data (Slack, Google Drive, GitHub…). Invoke with `$<app-slug>` + a `mention` input item (`path: "app://<id>"`).

| Method | Description |
|--------|-------------|
| `app/list` | Paginate available apps; entries include `isAccessible` + `isEnabled` (+ `branding`/`appMetadata`/`labels`); `threadId` for feature gating; `forceRefetch` to bypass cache |
| `app/list/updated` (notify) | Merged app list after accessible/directory apps load |

### Plugins & marketplaces

| Method | Description |
|--------|-------------|
| `marketplace/add` · `remove` · `upgrade` | Add/remove/refresh a remote plugin marketplace (persisted to user config) |
| `plugin/list` (WIP) | List discovered marketplaces + plugin state (install/auth policy, icons, `installPolicySource`: `null`/`WORKSPACE_SETTING`/`IMPLICIT_CANONICAL_APP`) |
| `plugin/read` (WIP) | Read one plugin (bundled skills/apps/MCP server names, optional `shareUrl`) |
| `plugin/install` · `uninstall` (WIP) | Install/uninstall from a marketplace path or remote marketplace name |
| `plugin/skill/read` | Read remote plugin skill Markdown on demand |

Plugin `source` union: `{type:"local",path}`, `{type:"git",url,path,refName,sha}`, `{type:"npm",package,version,registry}`, `{type:"remote"}`. For remote-only entries, `PluginMarketplaceEntry.path` can be `null` — pass `remoteMarketplaceName` instead of `marketplacePath`.

### App approval config (`config.toml`)

```toml
[apps._default]
enabled = true
destructive_enabled = true
open_world_enabled = true
approvals_reviewer = "user"            # or "auto_review"
default_tools_approval_mode = "auto"  # fallback for tools without per-app/per-tool override

[apps.google_drive]
enabled = true
destructive_enabled = false
approvals_reviewer = "auto_review"
default_tools_approval_mode = "prompt"
[apps.google_drive.tools."files/delete"]
enabled = false
approval_mode = "approve"
```

`apps._default.approvals_reviewer` sets the reviewer for all apps unless a per-app value overrides; if both omitted, inherits the top-level `approvals_reviewer`. Managed approval-mode requirements override tool approval-mode settings. Update via `config/read` / `config/value/write` (`keyPath`, `value`, `mergeStrategy: replace|upsert`) / `config/batchWrite` (`edits[]`).

---

## 9. MCP Integration

**Docs:** [mcp](https://learn.chatgpt.com/codex/extend/mcp) · [mcp-server](https://learn.chatgpt.com/codex/mcp-server)

Two directions: Codex as an **MCP client** (connect external tools/data) and Codex as an **MCP server** (be orchestrated by other agents).

### Codex as MCP client

| Method | Description |
|--------|-------------|
| `mcpServer/oauth/login` | Start OAuth login for a configured MCP server; returns auth URL; emits `mcpServer/oauthLogin/completed` |
| `mcpServerStatus/list` | List MCP servers, tools, resources, auth status; `detail: "full"` or `"toolsAndAuthOnly"` |
| `mcpServer/resource/read` | Read a single MCP resource through an initialized server |
| `mcpServer/tool/call` | Call a tool on a thread's configured MCP server |
| `mcpServer/startupStatus/updated` (notify) | A configured MCP server's startup status changed for a loaded thread |
| `config/mcpServer/reload` | Reload MCP server config from disk; queue a refresh for loaded threads |

MCP servers are configured in `config.toml` (`[mcp_servers.<name>]` with `url` for streamable HTTP, or `command`/`args` for stdio). An enabled server with `required = true` that fails to init makes `codex exec` exit with an error. MCP tool calls appear as `mcpToolCall` items; outputs >100k tokens auto-write to a sandbox file. MCP tool-call approvals can route through auto-review or `tool/requestUserInput`.

### Codex as MCP server (`codex mcp-server`)

Exposes `codex` (start a session) and `codex-reply` (continue by `threadId`) tools to MCP clients — the recommended path when Codex is one specialist in a broader orchestrated workflow (e.g. OpenAI Agents SDK `MCPServerStdio` with `command: "codex", args: ["mcp-server"]`). See §2 and §10.

---

## 10. Subagents & Multi-Agent Orchestration

**Docs:** [subagents](https://learn.chatgpt.com/codex/agent-configuration/subagents) · **MCP guide:** [mcp-server multi-agent](https://learn.chatgpt.com/codex/mcp-server)

### Subagent workflows

Codex/ChatGPT Work spawn specialized agents in parallel and collect results into one response. Helps with **context pollution** and **context rot** by moving noisy work (exploration, tests, logs) off the main thread and returning summaries.

| Surface | Trigger |
|---------|---------|
| ChatGPT Work | Ask to delegate; **Ultra** can proactively delegate suitable independent work |
| Codex app task | Ask to delegate independent parts; or `AGENTS.md`/skill instructions request it |
| Codex CLI | Ask for subagents/parallel work; `/agent` to inspect/switch between agent threads |
| IDE extension | Ask to delegate; background-agent panel shows status, stop-all, open-thread |

Subagents inherit the parent turn's sandbox policy and live runtime overrides (e.g. `/permissions` changes, `--yolo`) even if a custom agent file sets different defaults. Approval requests can surface from inactive agent threads (overlay shows source thread label; press `o` to open). In non-interactive flows, an action needing fresh approval fails and the error surfaces to the parent workflow.

### Global controls

`agents.max_threads` (default 6) caps concurrent open threads; `agents.max_depth` (default 1) lets root spawn direct children but prevents deeper descendants — raising it risks fan-out (more tokens, latency, resources). `agents.interrupt_message` (default true) records a model-visible interruption message.

### CSV batch fan-out (`spawn_agents_on_csv`, experimental)

One worker subagent per CSV row; Codex waits for the batch and exports combined results to CSV. Good for repeated audits (one file/package/service per row).

| Parameter | Description |
|-----------|-------------|
| `csv_path` | Source CSV |
| `instruction` | Worker prompt template with `{column_name}` placeholders |
| `id_column` | Stable item IDs from a specific column |
| `output_schema` | Each worker returns a fixed-shape JSON object |
| `output_csv_path`, `max_concurrency`, `max_runtime_seconds` | Job control (per-call `max_runtime_seconds` overrides `agents.job_max_runtime_seconds` default 1800s) |

Each worker must call `report_agent_job_result` exactly once; workers that exit without reporting are marked with an error. Exported CSV adds `job_id`, `item_id`, `status`, `last_error`, `result_json`. State stored in SQLite (`sqlite_home`).

### Multi-agent via Codex MCP + Agents SDK

Run `codex mcp-server` and orchestrate it with the OpenAI Agents SDK (`MCPServerStdio`). Build deterministic, reviewable pipelines: a Project Manager agent creates shared requirements (`REQUIREMENTS.md`, `TEST.md`, `AGENT_TASKS.md`), coordinates hand-offs to Designer/Frontend/Backend/Tester agents, each calling the `codex` MCP tool with `{"approval-policy":"never","sandbox":"workspace-write"}` to write scoped artifacts. Traces capture every prompt, tool call, and hand-off for audit.

### Custom agent examples (patterns)

| Pattern | Agents |
|---------|--------|
| PR review | `pr_explorer` (read-only, spark) + `reviewer` (read-only, high effort) + `docs_researcher` (docs MCP, read-only) |
| Frontend integration debugging | `code_mapper` (read-only) + `browser_debugger` (workspace-write, chrome_devtools MCP) + `ui_fixer` (spark) |
| Game builder (single-agent) | `Game Designer` (handoff → `Game Developer` which calls Codex MCP) |

---

## 11. Environments — Local, Worktree, Cloud

**Docs:** [environments/modes](https://learn.chatgpt.com/codex/environments/modes) · [local](https://learn.chatgpt.com/codex/environments/local-environment) · [cloud](https://learn.chatgpt.com/codex/environments/cloud-environment) · [worktrees](https://learn.chatgpt.com/codex/environments/git-worktrees)

Three execution modes (selectable in the ChatGPT desktop app composer). Local and Worktree run on your machine; Cloud runs remotely.

### Local

Desktop-app only. Configured via `codex://settings` (`.codex` folder at project root, checkable into Git). Features: **setup scripts** (run on new worktree creation; platform-specific overrides), **actions** (common tasks in the top bar running in the integrated terminal), built-in Git tools (diff pane, stage/revert, commit, push, create PR).

### Git Worktrees

Desktop-app only; requires a Git repo. Uses Git worktrees (own file copy, shared `.git` metadata).

| Aspect | Behavior |
|--------|----------|
| Codex-managed worktrees | Lightweight/disposable, one per task; created in `$CODEX_HOME/worktrees`; detached HEAD by default; uncommitted local changes applied; `AGENTS.override.md` auto-copied in |
| Permanent worktrees | Long-lived, from sidebar three-dot menu; not auto-deleted; multiple tasks can start from the same worktree |
| `.worktreeinclude` | Repo-root file listing ignored paths/patterns to copy into managed worktrees (e.g. `.env`); skips source symlinks; won't overwrite existing files |
| Handoff | Move a task between Worktree and Local ("Create branch here" or "Hand off") |
| Retention | Default keeps most recent **15** Codex-managed worktrees (configurable/disableable); not auto-deleted if pinned conversation / in-progress task / permanent; snapshot saved before deletion (restore option) |

Git allows a branch checked out in only one place at a time (`fatal: 'feature/a' is already used by worktree at '<path>'`).

### Cloud

OpenAI-managed isolated containers; start from web, GitHub, Linear, or Slack. Three pillars: run work in parallel, reproduce the environment, review before merge.

| Stage | Behavior |
|-------|----------|
| Container creation | Repo checked out at branch/commit SHA |
| Setup | Setup script + optional maintenance script run (with internet) |
| Network | Internet available during setup; agent phase **off by default** (configurable to limited/unrestricted); all traffic via HTTP/HTTPS proxy |
| Agent phase | Runs terminal commands in a loop; uses `AGENTS.md` for lint/test commands; offline by default |
| Result | Summary + diff; follow-up changes or open PR |

| Config | Description |
|--------|-------------|
| Default image | `universal` (pre-installed languages/packages; pin versions; `openai/codex-universal` Dockerfile) |
| Environment variables | Set for full task duration (setup + agent) |
| Secrets | Extra encryption; decrypted only for task execution; **only available to setup scripts, removed before agent phase** |
| Automatic setup | `npm`/`yarn`/`pnpm`/`pip`/`pipenv`/`poetry` |
| Manual setup | Custom script in a separate Bash session (exports don't persist; use `~/.bashrc` or env settings) |
| Container cache | Up to **12 hours**; clones default branch, runs setup, caches state; on resume checks out task branch + runs maintenance; invalidated on script/env/secret change; shared across Business/Enterprise users |

Integrations: GitHub (PRs + issues), Linear (issues + comments), Slack (channels + threads). App-server method `environment/info` (experimental) connects to a configured environment and returns shell + default cwd.

---

## 12. Long-running Work, Scheduled Tasks & Workspace Agents

### Long-running work & Goal mode

**Docs:** [long-running-work](https://learn.chatgpt.com/codex/long-running-work)

Multi-step work with a clear outcome, constraints, and a definition of done. Keep related work in the same task/conversation for shared context. Enter `/goal` (desktop app, CLI, IDE); goal text = first prompt + completion criteria. `/plan` to start with planning when the outcome is unclear (interview, identify constraints, turn into a measurable goal).

| Element | What to include |
|---------|-----------------|
| Outcome | Describe the result wanted, not just activity |
| Constraints | Required tools, boundaries, compatibility, approaches to avoid |
| Verification | Tests/measurements/review criteria proving completion |

Goal progress row (desktop): pause/resume/edit/clear; follow-up messages add context/adjust; side conversation for status recap without interrupting. Starting a goal does **not** grant broader access — same sandbox/approval policy; pauses for decisions; auto-review can evaluate eligible requests. Parallel goals keep separate context (avoid two tasks changing the same files; use worktrees). "Prevent sleep while running" setting; Pets/system notifications for input needs.

### Scheduled tasks (Automations)

**Docs:** [automations](https://learn.chatgpt.com/codex/automations)

Recurring background tasks reviewed in **Scheduled** (active/paused/completed). Combine with skills for complex work.

| Surface | Capability |
|---------|-----------|
| Desktop app | Works with local projects (project dir or isolated worktree); computer on + app running |
| Web (ChatGPT Work) | Uploaded context + connected tools; can't work in a local folder |
| CLI / IDE extension | No Scheduled management UI (use web/desktop); can prepare/test prompts/skills |

| Type | Behavior |
|------|----------|
| Standalone scheduled task | New task per run, independent; reports in Scheduled |
| Scheduled task in a chat | Returns to the same conversation/context |

Custom cadence via RFC 5545 RRULE (e.g. `RRULE:FREQ=MONTHLY;BYMONTHDAY=1;BYHOUR=9;BYMINUTE=0`). Git repos: choose local project or dedicated background worktree; non-VC runs in the project dir. Same scheduled task can run on >1 project. Skills create/update scheduled tasks (invoke with `$skill-name` in the desktop app).

Permissions: run unattended with default sandbox settings. `read-only` → modifying/network/app tool calls fail. `workspace-write` → outside-workspace/network/app calls fail (use rules to allowlist). `full access` → elevated risk. Admins restrict via `requirements.toml` (disallow `approval_policy = "never"`, constrain sandbox modes). Scheduled tasks use `approval_policy = "never"` when org policy allows; else fall back to the selected permission mode's approval behavior.

### Workspace Agents trigger API

**Docs:** [workspace-agents/trigger-runs](https://learn.chatgpt.com/workspace-agents/trigger-runs) · [authentication](https://learn.chatgpt.com/workspace-agents/authentication)

Programmatically trigger a published ChatGPT workspace agent via API — for external systems needing to trigger agents outside the ChatGPT UI/third-party triggers.

| Aspect | Detail |
|--------|--------|
| Endpoint | `POST https://api.chatgpt.com/v1/workspace_agents/{id}/trigger` |
| Agent ID | Stable public API trigger identifier, format `agtch_XXX` |
| Auth | `Authorization: Bearer $AGENT_ACCESS_TOKEN` (Workspace Agents scope; provisioned from ChatGPT admin access-token flow) |
| Body | `input` (string, required) — message text passed to the agent; `conversation_key` (string, optional) — caller-defined stable identifier to continue the same agent conversation across trigger events |
| Idempotency | Optional `Idempotency-Key` header; same key returns the original accepted outcome instead of enqueuing a second event |
| Response | `202 Accepted`, no body; no public run ID returned; agent response **cannot currently be retrieved via API** (coming soon) |
| Errors | 401 (bearer missing/expired/revoked/invalid) · 403 (no permission for agent) · 404 (`id` not visible to caller's workspace) · 409 (channel/agent not runnable) |

Tokens are scoped to Workspace Agents API operations only.

---

## 13. Capability Summary & Cross-Reference

### Capability → Key API surfaces

| Capability | Key API surfaces |
|------------|-----------------|
| Execution surfaces | `codex exec` (`--json`, `--output-schema`, `--sandbox`, `--ephemeral`, `resume`), `@openai/codex-sdk` / `openai-codex` (`Codex`, `startThread`/`resumeThread`, `thread.run`, `Sandbox.*`), `codex app-server` (JSON-RPC 2.0; stdio/ws/unix), `codex mcp-server` (`codex`/`codex-reply` tools) |
| Agent configuration | `config.toml`, `AGENTS.md`/`AGENTS.override.md` (`project_doc_fallback_filenames`, `project_doc_max_bytes`=32KiB), `~/.codex/agents/*.toml` (custom agents: `name`/`description`/`developer_instructions`/`nickname_candidates`), `[agents]` (`max_threads`=6, `max_depth`=1, `job_max_runtime_seconds`=1800, `interrupt_message`) |
| Agent loop | `thread/start`/`resume`/`fork`/`read`/`list`/`name/set`/`goal/*`/`archive`/`unarchive`/`delete`/`compact/start`/`inject_items`, `turn/start`/`steer`/`interrupt`, `review/start`, `ThreadItem` union, `item/started`/`completed` + deltas, `turn/*` events, `codexErrorInfo` |
| Sandboxing | `sandbox_mode` (`read-only`/`workspace-write`/`danger-full-access`), `[permissions.<name>]` (filesystem `read`/`write`/`deny`, network `domains`/`unix_sockets`/`proxy_url`/`socks_url`/`enable_socks5`), `[features.network_proxy]`, `web_search`, `codex sandbox <os>` |
| Approvals | `approval_policy` (`untrusted`/`on-request`/`never`/granular), `approvals_reviewer` (`user`/`auto_review`), `item/commandExecution/requestApproval`, `item/fileChange/requestApproval`, `item/permissions/requestApproval` (`scope: session|turn`), `mcpServer/elicitation/request`, `tool/requestUserInput`, auto-review circuit breaker (3 consecutive / 10 of 50), `/approve` override (10 denials/task) |
| Models & reasoning | `model` (`gpt-5.6`/`gpt-5.4`/`gpt-5.6-terra`/`gpt-5.3-codex-spark`), `model_reasoning_effort` (`ultra`/`max`/`xhigh`/`high`/`medium`/`low`/`minimal`/`none`), `model/list` (`includeHidden`, `supportedReasoningEfforts`, `inputModalities`), `modelProvider/capabilities/read` |
| Skills, plugins & apps | `skills/list`/`config/write`/`extraRoots/set`, `app/list`, `marketplace/add`/`remove`/`upgrade`, `plugin/list`/`read`/`install`/`uninstall`/`skill/read`, `[apps.*]` (`enabled`/`destructive_enabled`/`approvals_reviewer`/`default_tools_approval_mode`/`tools.<tool>.approval_mode`), `SKILL.json` (`interface`/`dependencies`) |
| MCP | `[mcp_servers.<name>]`, `mcpServer/oauth/login`, `mcpServerStatus/list`, `mcpServer/resource/read`, `mcpServer/tool/call`, `config/mcpServer/reload`, `codex mcp-server` |
| Subagents | `collabToolCall` items, `/agent` (CLI), `spawn_agents_on_csv` (`csv_path`/`instruction`/`id_column`/`output_schema`/`output_csv_path`/`max_concurrency`/`max_runtime_seconds`), `report_agent_job_result`, Agents SDK + `MCPServerStdio` orchestration |
| Environments | Local (`.codex/`, setup scripts, actions), Worktree (`$CODEX_HOME/worktrees`, `.worktreeinclude`, 15-worktree retention, handoff), Cloud (`universal` image, 12h cache, two-phase network, secrets-in-setup-only), `environment/info` (exp) |
| Long-running & scheduled | `/goal` + `/plan`, goal progress row, RRULE scheduled tasks, Workspace Agents `POST /v1/workspace_agents/{id}/trigger` (`input`, `conversation_key`, `Idempotency-Key`) |

### Cross-capability concepts

| Concept | Where it appears |
|---------|------------------|
| **Thread / Turn / Item** | Agent loop (core); all surfaces drive it; subagents extend it with `collabToolCall` |
| **Sandbox + approval** | Sandboxing (modes/profiles), Approvals (policies/decisions/auto-review), Environments (cloud two-phase), Subagents (inherit parent overrides) |
| **Configuration layering** | `config.toml` (home/project/profile) + `AGENTS.md` (global/project/nested) + custom agent files |
| **Streaming** | App-server notifications (`item/*` deltas, `turn/*`, `thread/tokenUsage/updated`), `codex exec --json` JSONL |
| **State / resumption** | `thread/resume`/`fork`/`archive`, `codex exec resume`, SQLite rollouts, goal pause/resume |
| **Skills / plugins / apps / MCP** | Progressive disclosure skills, connector apps with per-tool approval, MCP client+server, plugin marketplaces |
| **OS-level isolation** | Seatbelt/bubblewrap/Windows sandbox, network proxy + DNS-rebinding guard, protected paths (`.git`/`.agents`/`.codex`) |
| **Multi-agent** | Subagents (parallel, max_threads=6/max_depth=1), CSV fan-out, Codex-as-MCP-server orchestrated by Agents SDK |

### Decision guide — "If you want to…"

| If you want to | Start here |
|----------------|------------|
| Run Codex in a script/CI pipeline | `codex exec` (§2) — `--json` for events, `--output-schema` for structured output |
| Control Codex from an application | Codex SDK (§2) — `@openai/codex-sdk` / `openai-codex` |
| Embed Codex deeply into a product | `codex app-server` JSON-RPC (§2, §4) |
| Use Codex as one specialist in a larger workflow | `codex mcp-server` + Agents SDK (§2, §10) |
| Give Codex persistent project instructions | `AGENTS.md` layering (§3) |
| Define custom parallel agents | `~/.codex/agents/*.toml` (§3, §10) |
| Constrain what commands Codex can run | Sandbox modes / permission profiles (§5) |
| Reduce approval fatigue without losing safety | `approvals_reviewer = "auto_review"` (§6) |
| Pick a model / reasoning depth | Models & reasoning effort (§7) |
| Add domain expertise on demand | Skills (`$<skill-name>`) (§8) |
| Connect team tools (Slack, Drive, GitHub) | Apps/connectors + plugins (§8) |
| Connect external MCP tools | `[mcp_servers.*]` (§9) |
| Run parallel bounded work | Subagents / `spawn_agents_on_csv` (§10) |
| Isolate changes from your working dir | Git worktrees (§11) |
| Run remotely / reproducibly | Codex cloud environments (§11) |
| Run a long multi-step task | Goal mode `/goal` (§12) |
| Schedule recurring work | Scheduled tasks (RRULE) (§12) |
| Trigger a published workspace agent from outside ChatGPT | Workspace Agents trigger API (§12) |

### Key limits & defaults

| Limit | Value |
|-------|-------|
| `project_doc_max_bytes` | 32 KiB |
| `agents.max_threads` | 6 |
| `agents.max_depth` | 1 |
| `agents.job_max_runtime_seconds` | 1800 (CSV fan-out per-worker) |
| Auto-review circuit breaker | 3 consecutive / 10 of last 50 denials per turn |
| Auto-review override history | 10 recent denials per task |
| Worktree retention | 15 Codex-managed worktrees |
| Cloud container cache TTL | 12 hours |
| Network proxy defaults | `http://127.0.0.1:3128` (HTTP), `http://127.0.0.1:8081` (SOCKS5) |