# Claude API Analysis — Agent Capabilities (Agent SDK, Claude Code & Cowork)

> **Docs:** `https://code.claude.com/docs/en/overview` (Claude Code / Agent SDK) · `https://claude.com/docs/cowork/overview` (Cowork)
> **Base API:** `https://api.anthropic.com/v1` (via `ANTHROPIC_API_KEY`); also Amazon Bedrock, Claude Platform on AWS, Google Cloud's Agent Platform, Microsoft Foundry, or an LLM gateway
> **SDKs:** Python (`claude-agent-sdk`), TypeScript (`@anthropic-ai/claude-agent-sdk`) | **CLI:** `claude` (native binary, bundled with the TS SDK)
> **Description:** Anthropic exposes agent capabilities primarily through the **Claude Agent SDK** — a Python/TypeScript library that embeds the same agentic loop, tools, context management, and hooks that power the **Claude Code** CLI. Unlike a hosted REST agent API, the SDK runs the agent loop *inside your own process*, communicating with an embedded native Claude Code binary over a transport. The core primitive is the `query()` function, an async iterator yielding a typed message stream (`SystemMessage` → `AssistantMessage`/`UserMessage` per turn → `ResultMessage`). The same engine is surfaced interactively across multiple surfaces — terminal CLI, VS Code/JetBrains, desktop app, web, and mobile — and is packaged for end users as **Cowork** (an agentic workspace in Claude Desktop) and **Claude Tag** (Claude in Slack channels). The platform is organized around these first-class concepts: **the agent loop**, **tools** (built-in, MCP, custom), **permissions**, **hooks**, **sessions**, **subagents**, **agent teams**, **dynamic workflows**, **background agents (agent view)**, **skills**, **CLAUDE.md memory**, **checkpoints**, and **routines/scheduled tasks**.

---

## Table of Contents

1. [Platform Overview & Core Concepts](#1-platform-overview--core-concepts)
2. [The Agent Loop & Messages](#2-the-agent-loop--messages)
3. [Tools (Built-in, MCP, Custom)](#3-tools-built-in-mcp-custom)
4. [MCP Connector](#4-mcp-connector)
5. [Custom Tools (In-Process MCP Server)](#5-custom-tools-in-process-mcp-server)
6. [Tool Search (Large Tool Sets)](#6-tool-search-large-tool-sets)
7. [Permissions](#7-permissions)
8. [Hooks](#8-hooks)
9. [Sessions, Resume & Fork](#9-sessions-resume--fork)
10. [Subagents](#10-subagents)
11. [Agent Teams](#11-agent-teams)
12. [Dynamic Workflows](#12-dynamic-workflows)
13. [Background Agents & Agent View](#13-background-agents--agent-view)
14. [Skills, CLAUDE.md & Plugins](#14-skills-claudemd--plugins)
15. [Structured Outputs](#15-structured-outputs)
16. [File Checkpointing](#16-file-checkpointing)
17. [Hosting, Observability & Session Storage](#17-hosting-observability--session-storage)
18. [Cowork & Dispatch](#18-cowork--dispatch)
19. [Routines & Scheduled Tasks](#19-routines--scheduled-tasks)
20. [Capability Summary & Cross-Reference](#20-capability-summary--cross-reference)

---

## 1. Platform Overview & Core Concepts

### Main Concepts

The Claude agent platform (as exposed by code.claude.com and claude.com/docs) is built around these core abstractions:

- **Agent loop** — The execution cycle Claude runs autonomously: receive prompt → evaluate → request tool calls → execute tools → feed results back → repeat until a no-tool-call response. One round trip is a *turn*; the loop yields a typed message stream. The SDK ships this loop; you consume the stream.
- **`query()`** — The primary entry point. An async generator (`async for` in Python / `for await` in TypeScript) yielding messages. Each call starts a fresh session by default; resume/continue/fork are opt-in.
- **`ClaudeSDKClient` (Python)** — A stateful client maintaining one session across multiple `query()` calls, supporting interrupts, dynamic permission/model changes, and MCP reconnection. TypeScript uses `continue: true` instead of a client object.
- **Message types** — Five core types: `SystemMessage` (lifecycle: `init`, `compact_boundary`, `informational`, `worker_shutting_down`), `AssistantMessage` (text + tool_use blocks per turn), `UserMessage` (tool results / streamed input), `StreamEvent` (raw API deltas, opt-in), `ResultMessage` (terminal: final text, usage, cost, `session_id`, `subtype`).
- **Tool** — A capability the agent can call. Three categories: (1) **built-in tools** (Read, Write, Edit, Bash, Glob, Grep, WebSearch, WebFetch, Monitor, ToolSearch, Agent, Skill, AskUserQuestion, TaskCreate/Update), (2) **MCP tools** from connected servers (stdio/http/sse or in-process SDK servers), (3) **custom tools** defined in-process via `@tool`/`tool()`.
- **Permission** — Controls whether a tool call runs. Layered evaluation: hooks → deny rules → ask rules → permission mode → allow rules → `canUseTool` callback. Modes: `default`, `acceptEdits`, `dontAsk`, `auto`, `bypassPermissions`, `plan`.
- **Hook** — A callback fired at execution points (`PreToolUse`, `PostToolUse`, `Stop`, `SubagentStart/Stop`, `PreCompact`, `Notification`, …). Runs in your process (not the model context), can block/modify/inject. Mirrors the Claude Code shell-command hook system.
- **Session** — Persisted conversation history (JSONL on disk by default). Supports `resume` (by ID), `continue` (most recent in cwd), and `fork` (copy history into a new session). Session state is decoupled from the filesystem (use checkpointing for file rewind).
- **Subagent** — A separate agent instance spawned via the `Agent` tool to run a focused subtask in its own fresh context; only its final message returns to the parent. Defined programmatically (`agents` option) or as markdown files in `.claude/agents/`.
- **Agent team** — Multiple coordinated Claude Code instances (lead + teammates) sharing a task list and direct inter-agent messaging. Experimental, opt-in.
- **Dynamic workflow** — A JavaScript script Claude writes (and the runtime executes outside the conversation) that orchestrates many subagents at scale, with intermediate results in script variables. Resumable within a session.
- **Background agent / agent view** — Full Claude Code sessions running detached (supervisor process), monitored from a single screen (`claude agents`). Auto-isolated in git worktrees.
- **Skill** — Filesystem-based reusable expertise (`SKILL.md` + files) loaded on demand via progressive disclosure. Pre-built (Anthropic) or custom; project/user/plugin scopes.
- **CLAUDE.md / memory** — Project context and instructions loaded at session start from `.claude/CLAUDE.md` / `CLAUDE.md`; "auto memory" accumulates learnings across sessions.
- **Checkpoint** — File-change snapshot enabling rewind to any prior state during a session.
- **Routine / scheduled task** — A session run on a recurring schedule (cron), API call, or GitHub event, hosted on Anthropic-managed cloud.
- **Cowork** — Claude's agentic workspace in the Desktop app, powered by the same engine as Claude Code. **Dispatch** is its long-running background-agent feature.
- **Surface** — A front-end to the same engine: Terminal, VS Code, JetBrains, Desktop, Web, mobile, Slack (Claude Tag), Chrome.

### Agent Capabilities Map

| Capability | Description | Docs |
|------------|-------------|------|
| **Agent loop / `query()`** | Autonomous server-side loop, typed message stream, turns, budget | [agent-loop](https://code.claude.com/docs/en/agent-sdk/agent-loop), [overview](https://code.claude.com/docs/en/agent-sdk/overview) |
| **Tools** | Built-in toolset, custom tools, MCP, tool annotations | [custom-tools](https://code.claude.com/docs/en/agent-sdk/custom-tools) |
| **MCP connector** | stdio / http / sse / in-process SDK MCP servers | [mcp](https://code.claude.com/docs/en/agent-sdk/mcp) |
| **Tool search** | Defer tool definitions, discover on demand (up to 10k tools) | [tool-search](https://code.claude.com/docs/en/agent-sdk/tool-search) |
| **Permissions** | Modes, allow/deny/ask rules, `canUseTool` callback | [permissions](https://code.claude.com/docs/en/agent-sdk/permissions) |
| **Hooks** | Lifecycle callbacks (block/modify/inject/audit) | [hooks](https://code.claude.com/docs/en/agent-sdk/hooks) |
| **Sessions** | Resume, continue, fork; JSONL persistence | [sessions](https://code.claude.com/docs/en/agent-sdk/sessions) |
| **Subagents** | Programmatic or file-based delegated workers | [subagents](https://code.claude.com/docs/en/agent-sdk/subagents), [sub-agents](https://code.claude.com/docs/en/sub-agents) |
| **Agent teams** | Lead + teammates, shared task list, direct messaging | [agent-teams](https://code.claude.com/docs/en/agent-teams) |
| **Dynamic workflows** | Claude-written JS scripts orchestrating many subagents | [workflows](https://code.claude.com/docs/en/workflows) |
| **Background agents / agent view** | Detached sessions, supervisor, one-screen monitor | [agent-view](https://code.claude.com/docs/en/agent-view) |
| **Skills** | `SKILL.md` progressive disclosure, pre-built & custom | [skills](https://code.claude.com/docs/en/agent-sdk/skills), [skills (CLI)](https://code.claude.com/docs/en/skills) |
| **CLAUDE.md / memory** | Project context, auto memory | [memory](https://code.claude.com/docs/en/memory) |
| **Plugins** | Bundle skills, agents, hooks, MCP servers | [plugins](https://code.claude.com/docs/en/agent-sdk/plugins) |
| **Structured outputs** | JSON Schema / Zod / Pydantic validated results | [structured-outputs](https://code.claude.com/docs/en/agent-sdk/structured-outputs) |
| **File checkpointing** | Snapshot & rewind file changes | [file-checkpointing](https://code.claude.com/docs/en/agent-sdk/file-checkpointing) |
| **Hosting** | Docker/K8s, subprocess arch, multi-tenant, scaling | [hosting](https://code.claude.com/docs/en/agent-sdk/hosting) |
| **Observability** | OpenTelemetry traces, metrics, events | [observability](https://code.claude.com/docs/en/agent-sdk/observability) |
| **Session storage** | Mirror transcripts to S3/Redis/custom backends | [session-storage](https://code.claude.com/docs/en/agent-sdk/session-storage) |
| **Cowork / Dispatch** | Agentic workspace; long-running background Dispatch agent | [cowork/overview](https://claude.com/docs/cowork/overview), [dispatch](https://claude.com/docs/cowork/guide/dispatch) |
| **Routines** | Cron-scheduled sessions on Anthropic cloud | [routines](https://code.claude.com/docs/en/routines) |

### Platform Architecture

```
Your application (Python claude-agent-sdk / TypeScript @anthropic-ai/claude-agent-sdk / claude CLI)
        │
        ▼
   query(prompt, options)  ── async generator yielding SDKMessage
        │
        ▼
   Embedded native Claude Code binary (bundled; runs the agent loop)
        │  (transport: subprocess stdin/stdout JSON, or custom Transport)
        ▼
   ┌──────────── Agent Loop (in-process / server-side) ────────────┐
   │  1. SystemMessage(init) → session_id, MCP status               │
   │  2. Evaluate prompt → AssistantMessage (text + tool_use)      │
   │  3. Execute tools (built-in / MCP / custom)                  │
   │     • hooks (PreToolUse) can block/modify                     │
   │     • permission evaluation (hooks→deny→ask→mode→allow→cb)   │
   │  4. UserMessage(tool_result) feeds back                       │
   │  5. Repeat until no tool_use; final AssistantMessage          │
   │  6. ResultMessage (result, usage, cost, session_id, subtype) │
   │  • auto-compaction near context limit (compact_boundary)      │
   └────────────────────────────────────────────────────────────────┘
        │
        ▼   Optional extensions on the same loop:
   Subagents ─── Agent tool spawns fresh-context workers (parallel, background)
   Agent teams ─ lead + teammates, shared task list, SendMessage
   Workflows ─── JS script orchestrates dozens-hundreds of subagents
   Background ── claude --bg / claude agents (supervisor, worktree isolation)
```

### Quickstart flow

Minimal: install SDK → set `ANTHROPIC_API_KEY` → `async for message in query(prompt="...", options=ClaudeAgentOptions(allowed_tools=[...])):` → read `ResultMessage.result`. The four-step loop (init → assistant turns → result) is handled by the SDK; you only consume the stream.

### How the Agent SDK relates to other Claude tools

| | Agent SDK | Managed Agents (REST) | Client SDK | Claude Code CLI |
|---|---|---|---|---|
| **Runs in** | Your process | Anthropic infra | Your process | Terminal/IDE |
| **Interface** | Python/TS library | REST API | Python/TS library | CLI |
| **Agent works on** | Your filesystem & services | Managed sandbox | Your tool impls | Your filesystem |
| **Session state** | JSONL on your disk | Anthropic event log | Your state | JSONL on disk |
| **Custom tools** | In-process functions | Claude triggers, you execute | You implement the whole loop | Custom tools / hooks |
| **Best for** | Local prototyping, agents on your infra | Prod without operating sandbox/infra | Full custom loop control | Interactive dev |

A common path: prototype with the Agent SDK locally, move to Managed Agents for production (see the companion `anthropic-api.md` study for the REST/managed surface).

---

## 2. The Agent Loop & Messages

### Loop lifecycle

1. **Receive prompt** — SDK sends prompt + system prompt + tool definitions + history; yields `SystemMessage(subtype="init")` with session metadata (`session_id`, MCP server status).
2. **Evaluate & respond** — Claude produces text and/or tool calls; `AssistantMessage` yielded.
3. **Execute tools** — SDK runs each requested tool, collects results. Read-only tools run concurrently; state-modifying tools run sequentially. `PreToolUse` hooks may intercept.
4. **Repeat** — steps 2–3 cycle (one cycle = one turn) until a response with no tool calls.
5. **Return result** — final `AssistantMessage` (text-only) then `ResultMessage` (text, `usage`, `total_cost_usd`, `num_turns`, `session_id`, `subtype`, `stop_reason`).

### Turns & budget controls

| Option | Controls | Default |
|--------|----------|---------|
| `max_turns` / `maxTurns` | Max tool-use round trips | No limit |
| `max_budget_usd` / `maxBudgetUsd` | Max cost before stopping | No limit |
| `effort` | Reasoning depth: `low`/`medium`/`high`/`xhigh`/`max` | Model default |
| `model` | Model alias or full ID | Claude Code default |
| `fallback_model` | Used if primary fails | — |

### Message types

| Type | When | Key fields |
|------|------|------------|
| `SystemMessage` | Lifecycle events | `subtype` (`init`, `compact_boundary`, `informational`, `worker_shutting_down`); `session_id` (TS direct / Py in `data`) |
| `AssistantMessage` | After each Claude response | `content` blocks (text, tool_use); Py: `message.content`; TS: `message.message.content` |
| `UserMessage` | After tool execution / streamed input | tool result content; `parent_tool_use_id` when inside a subagent |
| `StreamEvent` | Only with partial messages enabled | raw API streaming events (text deltas, tool input chunks) |
| `ResultMessage` | End of loop | `result` (only on success), `subtype`, `usage`, `total_cost_usd`, `num_turns`, `session_id`, `stop_reason` |

### Result subtypes

| `subtype` | Meaning | `result` present? |
|-----------|---------|-------------------|
| `success` | Finished normally | Yes |
| `error_max_turns` | Hit `maxTurns` | No |
| `error_max_budget_usd` | Hit budget | No |
| `error_during_execution` | API failure / cancellation | No |
| `error_max_structured_output_retries` | Structured output validation exhausted | No |

> A single-shot `query()` raises after yielding an error result; wrap in try/except. Streaming-input sessions stay alive on error.

### Context window & compaction

Context accumulates across turns (system prompt, tool defs, history, tool I/O). Stable prefixes are prompt-cached automatically. When near the limit, the SDK **auto-compacts** (summarizes older history), emitting `SystemMessage(subtype="compact_boundary")` (TS: `SDKCompactBoundaryMessage`). Customize via CLAUDE.md summarization instructions, a `PreCompact` hook, or manual `/compact` prompt. Use subagents and scoped tools to keep context lean.

---

## 3. Tools (Built-in, MCP, Custom)

### Built-in tools

| Category | Tools |
|----------|-------|
| File operations | `Read`, `Write`, `Edit` |
| Search | `Glob`, `Grep` |
| Execution | `Bash` |
| Web | `WebSearch`, `WebFetch` |
| Discovery | `ToolSearch` |
| Orchestration | `Agent`, `Skill`, `AskUserQuestion`, `TaskCreate`, `TaskUpdate` |
| Monitoring | `Monitor` (watch a background script line-by-line) |

### Tool configuration (availability vs. permission)

Two distinct layers:

| Option | Layer | Effect |
|--------|-------|--------|
| `tools: ["Read", "Grep"]` | Availability | Only listed built-ins appear in context; unlisted removed. MCP tools unaffected. |
| `tools: []` | Availability | All built-ins removed; only MCP/custom tools usable. |
| `allowed_tools` / `allowedTools` | Permission | Listed tools run without a prompt; unlisted still available but fall through to permission flow. |
| `disallowed_tools` / `disallowedTools` | Both | Bare name (`"Bash"`) removes from context; scoped (`"Bash(rm *)"`) keeps visible but denies matching calls. |

### Scoped tool rules

- `allowed_tools=["Bash(npm *)"]` — auto-approve only matching Bash calls.
- `disallowed_tools=["mcp__github"]` or `["mcp__github__*"]` — remove all tools from a server; `mcp__*` removes every MCP tool.
- Allow-rule globs only after a literal `mcp__<server>__` prefix (`mcp__github__get_*`); bare `*` / `mcp__*` in allow rules are ignored with a warning.

### Parallel tool execution

Read-only tools (`Read`, `Glob`, `Grep`, MCP tools marked read-only) run concurrently; state-modifying tools (`Edit`, `Write`, `Bash`) run sequentially. Custom tools default to sequential; set `readOnlyHint: true` in annotations to enable parallel execution.

---

## 4. MCP Connector

Connect [Model Context Protocol](https://modelcontextprotocol.io) servers for external tools/data. MCP tools are named `mcp__{server}__{tool}`.

### Transport types

| Type | Use | Config |
|------|-----|--------|
| **stdio** | Local process (e.g. `npx @modelcontextprotocol/server-github`) | `{command, args, env}` |
| **http** (streamable HTTP) | Cloud-hosted server | `{type:"http", url, headers}` |
| **sse** | Remote SSE server | `{type:"sse", url, headers}` |
| **SDK MCP server** | In-process custom tools | `create_sdk_mcp_server(name, version, tools=[...])` |

MCP servers can be passed in code (`mcpServers` option) or loaded from `.mcp.json` via `settingSources: ["project"]`.

### Authentication

- **stdio**: pass `env: { GITHUB_TOKEN: ... }` (the `${VAR}` syntax expands in `.mcp.json`).
- **http/sse**: pass `headers: { Authorization: "Bearer ..." }`.
- **OAuth 2.1**: SDK doesn't run the flow; complete it in your app, pass the resulting bearer token via headers. `claude mcp login` authenticates servers from the shell.

### Allowing MCP tools

`allowedTools: ["mcp__github__*"]` (wildcard per server) or `["mcp__github__list_issues"]` (per tool). Prefer `allowedTools` over `bypassPermissions` for scoped MCP access. Discover available tools from the `init` `SystemMessage` (`mcp_servers` field with `status`).

### Error handling & limits

- The `init` message reports each server's `status` (`connected`/`failed`) before the agent works.
- Connection timeout 30s default; raise via `MCP_TIMEOUT` env var (ms).
- Tool outputs > `MAX_MCP_OUTPUT_TOKENS` are persisted to a sandbox file with a truncated preview + path returned to the model; per-tool override via `anthropic/maxResultSizeChars` annotation.
- Runtime control: `ClaudeSDKClient.reconnect_mcp_server(name)`, `toggle_mcp_server(name, enabled)`, `get_mcp_status()`.

---

## 5. Custom Tools (In-Process MCP Server)

Custom tools are user-defined functions exposed via an in-process MCP server. Define with `@tool` (Python) / `tool()` (TS), wrap in `create_sdk_mcp_server` / `createSdkMcpServer`, pass to `mcpServers`.

### Tool definition parts

1. **Name** — unique identifier.
2. **Description** — what it does; Claude reads this to decide when to call it (write detailed, 3–4 sentence descriptions).
3. **Input schema** — TS: Zod schema (args typed automatically); Py: dict mapping names to types (`{"lat": float}`) or full JSON Schema dict (for enums/ranges/nested).
4. **Handler** — async function receiving validated args; returns `{content: [...blocks], structuredContent?, isError?}`.

### Tool annotations (`ToolAnnotations`)

Optional behavioral hints (metadata, not enforcement):

| Field | Default | Meaning |
|-------|---------|---------|
| `readOnlyHint` | `false` | No env modification → enables parallel calls |
| `destructiveHint` | `true` | May perform destructive updates |
| `idempotentHint` | `false` | Repeated calls have no extra effect |
| `openWorldHint` | `true` | Reaches external systems |

### Result blocks

`content` accepts `text`, `image` (base64 `data` + `mimeType`), `audio` (TS only → saved to disk), `resource` (URI + inline `text`/`blob`), `resource_link`. `structuredContent` returns machine-readable JSON (TS only for in-process servers; Py needs a standalone MCP server). `isError: true` signals a tool failure Claude can react to (compose the message vs. surfacing the raw exception).

### Error handling

Uncaught exceptions are converted to error results carrying the raw message; the loop continues. Catch and return `isError: true` to control the message Claude reads.

---

## 6. Tool Search (Large Tool Sets)

Tool search defers tool definitions from context and loads only the 3–5 most relevant on demand, solving context bloat and selection-accuracy degradation at scale.

### Configuration (`ENABLE_TOOL_SEARCH` env var)

| Value | Behavior |
|-------|----------|
| (unset) | On by default; falls back to upfront on Google Cloud Agent Platform or non-first-party `ANTHROPIC_BASE_URL` |
| `true` | Always on (sends beta header; may fail on unsupported providers/proxies) |
| `auto` | Activates when combined tool defs exceed 10% of context window |
| `auto:N` | Like `auto` with custom percentage threshold |
| `false` | Off; all defs loaded every turn (faster for <~10 tools) |

### Limits

- Max **10,000 tools** in catalog.
- Returns **3–5** most relevant per search.
- Every Claude model **except Haiku**.
- Applies to remote MCP servers and custom SDK MCP servers alike.
- Set via the `env` option on `query()`.

Optimize discovery with descriptive tool names (`search_slack_messages` > `query_slack`) and a system-prompt section listing available tool categories.

---

## 7. Permissions

Permissions gate whether a requested tool call runs. Evaluation order (each step can resolve the call):

1. **Hooks** — `PreToolUse` may `allow`/`deny`/`ask`/`defer`. A hook `allow` does *not* skip later deny/ask rules.
2. **Deny rules** (`disallowed_tools` + settings.json) — matched → blocked (even in `bypassPermissions`). Bare names remove from context before this step; scoped rules checked here.
3. **Ask rules** (settings.json) — matched → fall through to `canUseTool` (even in `bypassPermissions`). `AskUserQuestion` and MCP tools with `_meta["anthropic/requiresUserInteraction"]` always fall through.
4. **Permission mode** — `bypassPermissions` approves all; `acceptEdits` approves file ops; `plan` routes file/shell writes to callback; others fall through.
5. **Allow rules** (`allowed_tools` + settings.json) — matched → approved.
6. **`canUseTool` callback** — final decision. Skipped in `dontAsk` (deny).

> Precedence on conflicting hook decisions: `deny` > `defer` > `ask` > `allow`.

### Permission modes

| Mode | Behavior |
|------|----------|
| `default` | No auto-approvals; unmatched → `canUseTool` (no callback = deny) |
| `acceptEdits` | Auto-approves file edits + filesystem commands (`mkdir`, `touch`, `rm`, `mv`, `cp`, `sed`) inside cwd/addDirs; other Bash follows default |
| `plan` | Read-only exploration; file edits never auto-approved → `canUseTool` |
| `dontAsk` | Anything not pre-approved by rules is denied; `canUseTool` never called |
| `auto` | A model classifier approves/denies each call |
| `bypassPermissions` | Runs all allowed tools without prompting (unless an `ask` rule matches). Cannot run as root on Unix. Use only in isolated envs. |

### Setting & changing mode

- At query time: `permission_mode` / `permissionMode` option.
- Mid-session: `ClaudeSDKClient.set_permission_mode(mode)` / `query.setPermissionMode(mode)`.
- Custom approval: `can_use_tool` / `canUseTool` callback returns `PermissionResultAllow`/`Deny` (can `updated_input` to redirect paths, `interrupt: true`).

### Subagent inheritance

When the parent uses `bypassPermissions`, `acceptEdits`, or `auto`, subagents inherit that mode and cannot override. `plan` mode subagents get `ExitPlanMode` only if their own `permissionMode` is `plan`.

---

## 8. Hooks

Hooks are callbacks fired at execution points, running in your process (not consuming model context). They can block, modify input, inject context, audit, or run async side effects.

### Available hook events

| Event | Py | TS | Triggers |
|-------|----|----|---------|
| `PreToolUse` | ✅ | ✅ | Tool call request (block/modify/approve) |
| `PostToolUse` | ✅ | ✅ | Tool result returned |
| `PostToolUseFailure` | ✅ | ✅ | Tool execution failure |
| `PostToolBatch` | ❌ | ✅ | A full batch resolves before next model call |
| `UserPromptSubmit` | ✅ | ✅ | User prompt submission |
| `MessageDisplay` | ❌ | ✅ | Assistant text message completes (redact/reformat display) |
| `Stop` | ✅ | ✅ | Agent execution stop |
| `SubagentStart` / `SubagentStop` | ✅ | ✅ | Subagent lifecycle |
| `PreCompact` | ✅ | ✅ | Before context compaction |
| `PermissionRequest` | ✅ | ✅ | Permission dialog would show |
| `SessionStart` / `SessionEnd` | ❌ | ✅ | Session lifecycle |
| `Notification` | ✅ | ✅ | Agent status messages |
| `Setup` | ❌ | ✅ | Session setup/maintenance |
| `TeammateIdle` / `TaskCompleted` | ❌ | ✅ | Agent-team events |
| `ConfigChange` | ❌ | ✅ | Config file changes |
| `WorktreeCreate` / `WorktreeRemove` | ❌ | ✅ | Git worktree lifecycle |

### Configuration

```python
options = ClaudeAgentOptions(
    hooks={"PreToolUse": [HookMatcher(matcher="Write|Edit", hooks=[callback])]}
)
```

- **Keys**: event names; **Values**: arrays of `{matcher, hooks, timeout}`.
- **Matchers**: exact string if only `[a-z0-9_-|, ]` (alternatives via `|`/`,`); regex if other chars; `*`/empty/omitted matches all. Hyphen-exact matching requires v2.1.195+.
- Tool hooks match tool *names* only; filter by file path inside the callback via `tool_input.file_path`.

### Callback signature & I/O

`async def cb(input_data, tool_use_id, context)` / `HookCallback` receives input + tool use ID + context (TS: `{signal: AbortSignal}`). Returns an output object:

- **Top-level**: `systemMessage` (shown to user), `continue`/`continue_` (keep running).
- **`hookSpecificOutput`**: event-specific. For `PreToolUse`: `permissionDecision` (`allow`/`deny`/`ask`/`defer`), `permissionDecisionReason`, `updatedInput` (must pair with `allow`/`ask`). For `PostToolUse`: `additionalContext`, `updatedToolOutput` (replace output before Claude sees it).
- **Async output**: `{async: true, asyncTimeout}` — agent proceeds without waiting (side effects only; can't block/modify).
- Return `{}` to allow without changes.

> Multiple matching hooks run in parallel; most restrictive decision wins (`deny` > `defer` > `ask` > `allow`).

### Common patterns

Block `.env` writes, redirect Write paths to `/sandbox` (via `updatedInput`), auto-approve read-only tools, forward `Notification` events to Slack, track subagent completions, run webhooks after `PostToolUse`, enforce DB read-only with a `PreToolUse` validation script.

Shell-command hooks (from `.claude/settings.json`) also load when `settingSources` includes `project`; SDK callback hooks mirror their JSON I/O format.

---

## 9. Sessions, Resume & Fork

A session is persisted conversation history (JSONL on disk by default at `~/.claude/projects/<encoded-cwd>/<session-id>.jsonl`; `CLAUDE_CONFIG_DIR` overrides root). Sessions persist the *conversation*, not the filesystem.

### Choosing an approach

| Use case | Mechanism |
|----------|-----------|
| One-shot task | Single `query()` |
| Multi-turn in one process | `ClaudeSDKClient` (Py) / `continue: true` (TS) |
| Pick up after restart | `continue_conversation=True` (Py) / `continue: true` (TS) — most recent in cwd |
| Resume a specific session | `resume: session_id` |
| Try alternatives safely | Fork |
| Stateless (no disk) | `persistSession: false` (TS only) |

### Continue vs. resume vs. fork

- **Continue** — find most recent session in cwd; no ID tracking.
- **Resume** — take a specific `session_id` (capture from `ResultMessage.session_id` or the `init` `SystemMessage`).
- **Fork** — `resume` + `fork_session=True`/`forkSession: true` → new session with copied history; original unchanged.

> Resume requires matching `cwd` (encoded path). For cross-host, mirror transcripts via a `SessionStore` adapter or pass needed state into a fresh prompt.

### Session utility functions

| Function | Purpose |
|----------|---------|
| `list_sessions(directory?, limit?, include_worktrees?)` | List past sessions (`SDKSessionInfo`) |
| `get_session_messages(session_id, directory?, limit?, offset)` | Retrieve messages |
| `get_session_info(session_id, directory?)` | Single-session metadata |
| `rename_session(session_id, title)` | Rename (appends custom title) |
| `tag_session(session_id, tag\|None)` | Tag / clear tag |

`SDKSessionInfo` fields: `session_id`, `summary`, `last_modified`, `file_size`, `custom_title`, `first_prompt`, `git_branch`, `cwd`, `tag`, `created_at`.

### `ClaudeSDKClient` (Python) methods

`connect`, `query(prompt, session_id)`, `receive_messages()`, `receive_response()` (until ResultMessage), `interrupt()` (streaming only; drains buffer after), `set_permission_mode`, `set_model`, `rewind_files(user_message_id)` (requires `enable_file_checkpointing`), `get_mcp_status`, `reconnect_mcp_server`, `toggle_mcp_server`, `stop_task(task_id)`, `get_server_info`, `disconnect`. Usable as an async context manager.

---

## 10. Subagents

Subagents are separate agent instances spawned via the `Agent` tool to handle focused subtasks in isolated context. Only the final message returns to the parent (as the Agent tool result), keeping the parent's context lean.

### Definition methods

1. **Programmatic** (recommended for SDK) — `agents: { "name": AgentDefinition(...) }` option; include `Agent` in `allowed_tools` to auto-approve invocations.
2. **Filesystem** — markdown files with YAML frontmatter in `.claude/agents/` (project) or `~/.claude/agents/` (user); `--agents` flag (CLI JSON); plugin `agents/`; managed settings.
3. **Built-in** — `Explore` (read-only, inherits model capped at Opus on API), `Plan` (plan-mode research), `general-purpose` (full tools, complex multi-step). Disable via `permissions.deny: ["Agent(Explore)"]`, `CLAUDE_CODE_DISABLE_EXPLORE_PLAN_AGENTS=1`, or `CLAUDE_AGENT_SDK_DISABLE_BUILTIN_AGENTS=1` (SDK/headless).

### `AgentDefinition` fields

| Field | Required | Description |
|-------|----------|-------------|
| `description` | Yes | When to use this agent (drives auto-delegation) |
| `prompt` | Yes | System prompt |
| `tools` | No | Allowed tools (inherits all if omitted) |
| `disallowedTools` | No | Tools to deny (supports `mcp__server`, `mcp__server__*`, `mcp__*`) |
| `model` | No | Alias (`opus`/`sonnet`/`haiku`/`fable`/`inherit`) or full ID |
| `skills` | No | Skill names to preload into context at startup |
| `memory` | No | `user` / `project` / `local` persistent memory scope |
| `mcpServers` | No | MCP servers (inline defs or name refs) |
| `initialPrompt` | No | First user turn when run as main-thread agent |
| `maxTurns` | No | Max agentic turns |
| `background` | No | Force background execution |
| `effort` | No | Reasoning effort override |
| `permissionMode` | No | Permission mode for this agent |
| `isolation` | No | `worktree` → isolated git worktree copy |
| `color` | No | Display color |

### What subagents inherit (and don't)

- **Receives**: own system prompt + Agent tool prompt; project CLAUDE.md (via settingSources); tool defs (inherited or `tools` subset).
- **Doesn't receive**: parent conversation history, parent system prompt, tool results, unlisted skill content.
- The parent receives the subagent's **final message verbatim** as the Agent tool result (may summarize in its own response).
- Messages inside a subagent carry `parent_tool_use_id`.

### Invocation

- **Automatic** — Claude delegates based on `description` (write clear, specific descriptions; use "use proactively" phrasing).
- **Explicit** — name in prompt ("Use the code-reviewer agent to..."); `@mention` the agent (CLI); `claude --agent <name>` to run the whole session as that subagent.
- **Dynamic** — factory functions creating `AgentDefinition` at query time (e.g. stricter model for high-stakes).

### Foreground / background

Since v2.1.198, subagents run in the **background by default**; Claude sets `run_in_background: false` when it needs the result before continuing. Set `background: true` to force background. Foreground subagents block the main conversation; background ones run concurrently (permission prompts surface in the main session).

### Nested subagents

Since v2.1.172, subagents can spawn their own subagents; a subagent 5 levels deep can't spawn more. Prevent nesting by omitting `Agent` from `tools` or adding to `disallowedTools`.

### Resuming subagents

On completion, the Agent tool result includes `agentId: <id>` in a text block. Resume by passing `resume: sessionId` (same session) and including the agent ID in the prompt. Built-in `Explore`/`Plan` are one-shot (no `agentId`). Subagent transcripts persist independently of main-conversation compaction.

### Restricting which subagents spawn

`tools: [Agent(worker, researcher), ...]` — allowlist of subagent types (only for main-thread `--agent` runs). Omitting `Agent` blocks all subagents. Disable specific ones via `permissions.deny: ["Agent(Explore)"]` or `--disallowedTools "Agent(Explore)"`.

---

## 11. Agent Teams

Agent teams coordinate multiple Claude Code instances (a **lead** + **teammates**) with a shared task list and direct inter-agent messaging. Experimental; disabled by default (`CLAUDE_CODE_EXPERIMENTAL_AGENT_TEAMS=1`).

### vs. subagents

| | Subagents | Agent teams |
|---|---|---|
| Context | Own window; result returns to caller | Own window; fully independent |
| Communication | Report back to main agent only | Teammates message each other directly |
| Coordination | Main agent manages | Shared task list, self-coordination |
| Token cost | Lower (summarized back) | Higher (each is a separate instance) |

### Architecture

| Component | Role |
|-----------|------|
| Team lead | Main session; spawns & coordinates |
| Teammates | Separate Claude Code instances |
| Task list | Shared work items (pending/in-progress/completed) with dependencies; file-lock-based claiming |
| Mailbox | Messaging system (`SendMessage`) |

- Team config: `~/.claude/teams/{team-name}/config.json` (runtime state, auto-generated; removed at session end).
- Task list: `~/.claude/tasks/{team-name}/` (persists for resumed sessions).
- Team name: `session-` + first 8 chars of session ID.

### Display modes

`in-process` (default; one terminal, agent panel) or split panes (`auto`/`tmux`/`iterm2`; requires tmux or iTerm2 + `it2` CLI). Set via `teammateMode` setting or `--teammate-mode`.

### Behavior

- Each teammate has its own context, loads project context (CLAUDE.md, MCP, skills) + spawn prompt; no lead history.
- Messages auto-delivered; idle teammates notify lead; API-error idle includes error text (v2.1.198+).
- Permission prompts bubble to the lead; teammates can't approve for each other; auto-mode treats relayed approvals as untrusted.
- Teammates inherit the lead's `bypassPermissions`/`acceptEdits`/`auto` mode and effort.
- Reference a subagent definition as a teammate (honors `tools`/`model`; body appended to teammate system prompt; `skills`/`mcpServers` frontmatter ignored on teammates).

### Hooks for quality gates

`TeammateIdle` (exit 2 → feedback, keep working), `TaskCreated` (exit 2 → block creation), `TaskCompleted` (exit 2 → block completion).

### Limitations

No session resumption with in-process teammates; task status can lag; shutdown can be slow; one team per session; no nested teams; no background subagents from in-process teammates; lead is fixed; permissions set at spawn; split panes need tmux/iTerm2.

---

## 12. Dynamic Workflows

A dynamic workflow is a JavaScript script Claude writes (and a runtime executes outside the conversation) that orchestrates many subagents at scale. Intermediate results live in script variables, not Claude's context.

### When to use

| | Subagents | Skills | Agent teams | Workflows |
|---|---|---|---|---|
| Plan holder | Claude turn-by-turn | Claude following prompt | Lead agent turn-by-turn | The script |
| Intermediate results | Claude's context | Claude's context | Shared task list | Script variables |
| Repeatable | Worker definition | Instructions | Team definition | Orchestration itself |
| Scale | Few per turn | Few per turn | Handful of peers | Dozens–hundreds per run |
| Interruption | Restarts turn | Restarts turn | Teammates keep running | Resumable in session |

### Triggers

- Include `ultracode` keyword in a prompt (or ask "use a workflow" in natural language).
- `/effort ultracode` — `xhigh` effort + automatic workflow orchestration for every substantive task.
- Run a saved/bundled workflow command (e.g. `/deep-research`).

### Script shape

```javascript
export const meta = { name: 'audit-routes', description: '...' }
const found = await agent('List every .ts file under src/routes/.', { schema: {...} })
const audits = await pipeline(found.files, file => agent(`Audit ${file}...`, { label: file }))
return audits.filter(Boolean)
```

Top-level `await`; `agent()` spawns one subagent; `pipeline()` runs one per list item. Saved to `.claude/workflows/` (project) or `~/.claude/workflows/` (user) as `/<name>` commands; accepts `args` for input.

### Approval by permission mode

| Mode | When prompted |
|------|---------------|
| Default / acceptEdits | Every run (unless "don't ask again" for this workflow in this project) |
| Auto | First launch only |
| Bypass / `claude -p` / Agent SDK | Never |

Workflow subagents always run in `acceptEdits` and inherit your tool allowlist regardless of session mode.

### Runtime limits

- No mid-run user input (only agent permission prompts can pause).
- No direct filesystem/shell access from the script (agents do that).
- Up to **16 concurrent agents** (fewer on limited cores).
- **1,000 agents total** per run.
- Resumable within the same session (cached results reused).
- Large-run warning at >25 agents or >1.5M projected tokens (v2.1.203+; configurable via size guideline `unrestricted`/`small`(<5)/`medium`(<15)/`large`(<50)).

### Management

`/workflows` view: per-phase agent counts, tokens, elapsed time; drill into agents; pause/resume (`p`), stop (`x`), restart (`r`), save (`s`). Disable via `disableWorkflows: true` / `CLAUDE_CODE_DISABLE_WORKFLOWS=1`.

---

## 13. Background Agents & Agent View

Agent view (`claude agents`) is one screen to dispatch and monitor full Claude Code sessions running detached in the background. A **supervisor process** hosts them, so they keep running with no terminal attached.

### Dispatching

- From agent view: type a prompt + Enter (each starts a new session).
- From inside a session: `/background` (`/bg`).
- From shell: `claude --bg "<prompt>"` (prompt is positional, not `-p`), `--name`, `--agent`, `--model`, `--permission-mode`, `--exec` (shell job).
- First-word subagent name or `@name` runs that subagent as the session's main agent.

### File-edit isolation

Every background session auto-moves into an isolated git worktree under `.claude/worktrees/` before editing (parallel sessions read the same checkout but write separately). Skip when already in a worktree, non-git dir w/o `WorktreeCreate` hook, or writes outside cwd. Disable via `worktree.bgIsolation: "none"`. Since v2.1.198, an isolated session also commits, pushes its own branch, and opens a draft PR without asking (never to `main`/`master`, never force-push/merge). Subagents inherit the session's worktree; set `isolation: worktree` for a separate one.

### Session states (row icons)

Working (animated), Needs input (yellow), Idle (dimmed), Completed (green), Failed (red), Stopped (grey). Shape: `✻`/`✽` alive, `∙` exited (peek/reply/attach still work; restarts from where left off), `✢` `/loop` sleeping. PR labels `#N` colored by PR status (yellow/green/purple/grey).

### Interaction

- `Space` — peek panel (status sentence, question, PRs; reply without attaching).
- `Enter`/`→` — attach (full session; posts a recap on attach).
- `←` (empty prompt) / `/exit` / `Ctrl+Z` — detach (session keeps running).
- `Ctrl+X` — stop; twice within 2s deletes (removes Claude-created worktree too).
- Filters: `a:<name>`, `s:<state>`, `#<number>`/PR URL, any URL.
- `Ctrl+S` toggle grouping (state/dir), `Ctrl+T` pin, `Ctrl+R` rename, `Shift+↑/↓` reorder.

### Shell management

| Command | Purpose |
|---------|---------|
| `claude agents` | Open agent view |
| `claude --bg "<prompt>"` | Start background session |
| `claude --bg --exec '<cmd>'` | Start background shell job |
| `claude attach <id>` | Attach in this terminal |
| `claude logs <id>` | Recent output |
| `claude stop <id>` | Stop a session |
| `claude rm <id>` | Delete (keeps worktree with uncommitted changes) |

Sessions persist on disk across auto-updates and supervisor restarts; sleep/wake resume supported. Row summaries generated by a Haiku-class model (15s updates between model rewrites; falls back to session model on non-Anthropic providers).

---

## 14. Skills, CLAUDE.md & Plugins

### Skills

Filesystem-based reusable expertise (`SKILL.md` + supporting files) loaded on demand — only impacting context when relevant (progressive disclosure).

- **Pre-built Anthropic skills**: `pptx`, `xlsx`, `docx`, `pdf`, and SDK/CLI built-ins.
- **Custom skills**: authored by you; uploaded as files/zip.
- **Scopes**: project `.claude/skills/`, user `~/.claude/skills/`, plugin, managed.
- **In the SDK**: `skills` option (list of names or `"all"`); `AgentDefinition.skills` preloads into subagent context. Invoke via `Skill` tool; `disable-model-invocation: true` prevents model-initiated loading.
- **Skill features**: `context: fork` runs a skill in a subagent; frontmatter `description`, `allowed-tools`, `disable-model-invocation`, etc.

### CLAUDE.md / memory

- `CLAUDE.md` / `.claude/CLAUDE.md` — project context & instructions loaded at session start (re-injected every request; prompt-cached). Nested files in monorepos.
- **Auto memory** — learnings accumulated across sessions automatically (build commands, debugging insights).
- Control loading via `settingSources` (Python) / `settingSources` (TS): `"project"`, `"user"`, `"local"`.
- In subagents: `Explore`/`Plan` skip CLAUDE.md; others load it.

### Plugins

Bundle skills, agents, hooks, and MCP servers into a shareable package.

- **Scopes**: `.claude/` (project), `~/.claude/` (user), plugin marketplace, managed.
- **SDK**: `plugins` option (`SdkPluginConfig`); loaded programmatically. Plugin subagents appear under scoped names (`my-plugin:review:security`).
- **Plugin subagent restrictions**: `hooks`, `mcpServers`, `permissionMode` frontmatter ignored for plugin-provided agents (security).
- **Marketplaces**: discover/install; `plugin-hints` (CLI emits install markers), `plugin-dependencies` (version constraints), `plugin-relevance` (suggest when work matches).

---

## 15. Structured Outputs

Return validated JSON from agent workflows (after multi-turn tool use) using JSON Schema, Zod (TS), or Pydantic (Py).

### Configuration

`outputFormat` (TS) / `output_format` (Py): `{ type: "json_schema", schema: {...} }`. On success, `ResultMessage.structured_output` holds validated data matching the schema.

### Type-safe schemas

- **TypeScript**: `z.toJSONSchema(MyZodSchema)` → pass to `outputFormat`; parse back with `MyZodSchema.safeParse(message.structured_output)` for full typing.
- **Python**: `MyPydanticModel.model_json_schema()` → pass; parse back with `MyPydanticModel.model_validate(...)`.

### Supported JSON Schema features

Basic types, `enum`, `const`, `required`, nested objects, `$ref`. `format` accepted as annotation but not enforced. Invalid schema fails at startup (v2.1.205+; earlier silently ignored). Limited to [API JSON Schema limitations](https://platform.claude.com/docs/en/build-with-claude/structured-outputs#json-schema-limitations).

### Error handling

| `subtype` | Meaning |
|-----------|---------|
| `success` | Generated & validated |
| `error_max_structured_output_retries` | No valid output after retries (validation failures, or model-fallback retraction with no retry) |

Check `errors` field to distinguish validation failures from fallback retractions. Tips: keep schemas focused, make uncertain fields optional, use clear prompts.

---

## 16. File Checkpointing

Track file changes during agent sessions and restore files to any previous state. Decoupled from session (conversation) resume.

### Enable

`ClaudeAgentOptions(enable_file_checkpointing=True)` (Py) / `enableFileCheckpointing: true` (TS).

### Rewind

`ClaudeSDKClient.rewind_files(user_message_id)` — restore files to their state at the specified user message. Checkpoints are per-session.

> Sessions persist the *conversation*; checkpointing persists the *filesystem*. Use both together for full rewind.

---

## 17. Hosting, Observability & Session Storage

### Hosting (production)

The Agent SDK runs the agent loop in your process via a bundled native Claude Code binary. Production deployment guidance:

- **Subprocess architecture** — your app drives the binary; the binary drives the model.
- **Session persistence** — JSONL by default; mirror to external storage for multi-host.
- **Scaling** — one binary per concurrent session; resource considerations.
- **Multi-tenant isolation** — Docker, Kubernetes, or sandbox-provider isolation patterns.
- **Secure deployment** — isolation, credential management, network controls.

### Observability (OpenTelemetry)

Export traces, metrics, and events to an OTLP backend. Configure via the SDK/CLI; spans cover model requests, tool calls, hooks. The `agent-loop` emits `span.model_request_start`/`_end` markers; cost/usage tracked per turn on `ResultMessage`.

### Session storage adapters

Mirror session transcripts to S3, Redis, or a custom backend so any host can resume them. `SessionStore` adapter interface; `session_store_flush` mode (`"batched"` default). Enables resume across CI workers, ephemeral containers, serverless.

### `ClaudeAgentOptions` (key fields beyond those above)

| Field | Description |
|-------|-------------|
| `cwd` | Working directory |
| `add_dirs` | Additional accessible directories |
| `env` | Env vars merged over process env |
| `setting_sources` | Which config sources load (`project`/`user`/`local`) |
| `system_prompt` | String, `{"type":"preset","preset":"claude_code"}` (+ `"append"`), or `{"type":"file","path":"..."}` |
| `betas` | Beta features (`SdkBeta`) |
| `thinking` | `ThinkingConfig` (extended thinking) |
| `effort` | Reasoning effort level |
| `sandbox` | `SandboxSettings` |
| `strict_mcp_config` | Only use passed `mcp_servers` (maps to `--strict-mcp-config`) |
| `include_partial_messages` | Enable `StreamEvent` streaming |
| `include_hook_events` | Surface hook output in the stream |
| `can_use_tool` | `CanUseTool` callback |
| `cli_path` | Custom path to the CLI binary |
| `transport` | Custom `Transport` (remote connection) |

---

## 18. Cowork & Dispatch

### Cowork

Cowork is Anthropic's agentic workspace, accessible in **Claude Desktop** without the terminal. It uses the **same agentic architecture that powers Claude Code** — multi-step autonomous work producing documents, spreadsheets, presentations, organized files, synthesized research.

**Key capabilities:**
- Works directly on your computer (reads/writes local files, no manual uploads).
- **Claude in Chrome** pairing for web automation.
- **Sub-agent coordination** — parallel workstreams.
- Professional outputs (Excel with formulas, PowerPoint, formatted docs).

**Extensibility (same across Claude products):**
- **Connectors** — connect to tools/data via MCP.
- **Skills** — reusable workflow instructions.
- **Plugins** — bundle skills, connectors, and more into shareable packages.
- **Monitoring** — OpenTelemetry-based usage/activity tracking across the org.

### Dispatch

Dispatch is Cowork's long-running background agent. You describe an outcome in one conversation; the Dispatch agent breaks it into tasks, runs each as a separate Cowork or Code session, and surfaces results in the sidebar.

- **Routing**: coding work → Code (against a workspace you've set up); knowledge work → Cowork (in a specified project). Tell it which to use or let it choose.
- **Child tasks** run as separate sessions; child tasks don't spawn further children.
- **States**: Running, Awaiting input, Awaiting answer, Completed, Error, Archived.
- **Permission forwarding**: prompts forwarded to you; auto-denied after 10 min of no response.
- **Mobile**: your desktop registers as a Dispatch host; start tasks from the Claude mobile app, review on either device.
- **Prerequisites**: Pro/Max plan, latest Claude Desktop on macOS/Windows.

### Related Cowork surfaces

- **Projects** — group folders, instructions, and context for consistent session setup.
- **Plugins** — install packaged skills/connectors/agents from marketplace or file.
- **Office agents** — Claude for Excel/PowerPoint/Word/Outlook add-ins (Pro/Max/Team/Enterprise).

---

## 19. Routines & Scheduled Tasks

### Routines

Routines put Claude Code on autopilot on Anthropic-managed cloud infrastructure — they keep running even when your computer is off.

- **Triggers**: cron schedule, API call, or GitHub event.
- **Creation**: from the web, the Desktop app, or `/schedule` in the CLI.
- **Surfaces**: CLI, Desktop app, web (Claude Code on the web), and Claude Tag (Slack) routines.

### Desktop scheduled tasks

Run on your machine (not cloud) with direct access to local files/tools.

### `/loop` (in-session)

Repeat a prompt within a CLI session for quick polling; runs as a `/loop` row in agent view (`✢` icon, run count, countdown).

### CLI scheduling flags

- `claude --bg` + scheduling — background sessions on a schedule.
- `/schedule` — create a routine from the CLI.
- Scheduled sessions count toward plan usage and rate limits.

---

## 20. Capability Summary & Cross-Reference

| Capability | Primary primitive(s) | Key functions / options | Core parameters |
|------------|----------------------|------------------------|-----------------|
| **Agent loop** | `query()` | `query(prompt, options, transport)` → `AsyncIterator[Message]` | `prompt`, `ClaudeAgentOptions` |
| **Loop control** | Options | `max_turns`, `max_budget_usd`, `effort`, `model`, `fallback_model` | turn/budget caps, effort level |
| **Built-in tools** | `tools` / `allowed_tools` | `Read`, `Write`, `Edit`, `Bash`, `Glob`, `Grep`, `WebSearch`, `WebFetch`, `Agent`, `Skill`, `Monitor`, `ToolSearch` | availability + permission lists |
| **MCP** | `mcp_servers` | stdio (`command`/`args`/`env`), http/sse (`url`/`headers`), SDK servers | `mcp__{server}__{tool}` names, `strict_mcp_config` |
| **Custom tools** | `@tool` / `tool()` + `create_sdk_mcp_server` | name, description, input_schema, handler, annotations | `content[]`, `structuredContent`, `isError`, `readOnlyHint` |
| **Tool search** | `ENABLE_TOOL_SEARCH` env | `true`/`auto`/`auto:N`/`false` | context % threshold, 10k tool cap |
| **Permissions** | `permission_mode` + rules | `default`/`acceptEdits`/`plan`/`dontAsk`/`auto`/`bypassPermissions` | `allowed_tools`, `disallowed_tools`, `can_use_tool`, ask rules |
| **Hooks** | `hooks` option | `PreToolUse`, `PostToolUse`, `Stop`, `SubagentStart/Stop`, `PreCompact`, `Notification`, … | `matcher`, `hooks[]`, `timeout`, `hookSpecificOutput` (`permissionDecision`, `updatedInput`, `updatedToolOutput`), `async` |
| **Sessions** | `resume`/`continue`/`fork_session` | `query()` options, `ClaudeSDKClient` | `session_id`, `cwd`, `SessionStore` |
| **Session utils** | Functions | `list_sessions`, `get_session_messages`, `get_session_info`, `rename_session`, `tag_session` | `directory`, `limit`, `tag` |
| **Subagents** | `agents` option / `.claude/agents/` | `AgentDefinition` | `description`, `prompt`, `tools`, `model`, `skills`, `memory`, `mcpServers`, `background`, `effort`, `isolation` |
| **Agent teams** | `CLAUDE_CODE_EXPERIMENTAL_AGENT_TEAMS` | lead + teammates + task list + `SendMessage` | `teammateMode`, spawn prompts, `TeammateIdle`/`TaskCreated`/`TaskCompleted` hooks |
| **Dynamic workflows** | `Workflow` tool / `/ultracode` | JS script (`agent()`, `pipeline()`), `meta` | `args`, size guideline, 16 concurrent / 1000 total caps |
| **Background agents** | `claude --bg` / `claude agents` | supervisor, worktree isolation, peek/attach | `--name`, `--agent`, `--model`, `--permission-mode`, `--exec` |
| **Skills** | `skills` option / `.claude/skills/` | `SKILL.md` + files | `type` (anthropic/custom), `skill_id`, `version`, `disable-model-invocation`, `context: fork` |
| **CLAUDE.md / memory** | `setting_sources` | `CLAUDE.md`, `.claude/CLAUDE.md`, auto memory | `"project"`/`"user"`/`"local"` sources |
| **Plugins** | `plugins` option | `SdkPluginConfig` | skills, agents, hooks, MCP servers (scoped names) |
| **Structured outputs** | `output_format` | `{type:"json_schema", schema}` | JSON Schema / Zod / Pydantic; `structured_output`, `error_max_structured_output_retries` |
| **File checkpointing** | `enable_file_checkpointing` | `rewind_files(user_message_id)` | per-session snapshots |
| **Hosting** | Subprocess + SDK | Docker/K8s, multi-tenant isolation | `transport`, `sandbox`, `cli_path` |
| **Observability** | OpenTelemetry | traces/metrics/events | OTLP backend |
| **Session storage** | `session_store` | `SessionStore` adapter, `session_store_flush` | S3/Redis/custom |
| **Cowork / Dispatch** | Desktop app | Dispatch agent → child tasks (Code/Cowork) | routing, 10-min permission timeout, mobile |
| **Routines** | `/schedule`, cron | schedule/API/GitHub triggers | cron expression, timezone |

### Key design principles

1. **Loop-in-process** — The Agent SDK runs Claude Code's agent loop inside your process via an embedded binary; you consume a typed message stream rather than implementing the tool loop yourself (contrast with the Client SDK).
2. **Typed message stream** — `SystemMessage` / `AssistantMessage` / `UserMessage` / `StreamEvent` / `ResultMessage`; the `ResultMessage.subtype` is the primary termination signal.
3. **Layered permissions** — hooks → deny → ask → mode → allow → callback; auto-approved tools never reach `canUseTool`; hooks run before every other step and can block even in `bypassPermissions`.
4. **Context isolation via subagents** — Each subagent runs in a fresh context; only its final message returns, keeping the parent lean. Background-by-default since v2.1.198; nested since v2.1.172.
5. **Multi-agent orchestration spectrum** — Subagents (turn-by-turn delegation) → agent teams (peers with shared task list + direct messaging) → dynamic workflows (scripted orchestration at scale) → background agents (detached, supervised sessions).
6. **Filesystem-based configuration** — CLAUDE.md, `.claude/agents/`, `.claude/skills/`, `.claude/settings.json`, `.mcp.json`, `.claude/workflows/` load via `settingSources`; programmatic definitions override filesystem ones.
7. **Hooks as cross-cutting control** — Same JSON I/O as Claude Code shell-command hooks; SDK callbacks run in-process, can block/modify/inject/audit at every execution point without consuming model context.
8. **Sessions decoupled from filesystem** — Conversation history (JSONL, resumable/forkable) is separate from file state (checkpointing); mirror transcripts via `SessionStore` for cross-host.
9. **Same engine, many surfaces** — Terminal, IDE, Desktop, web, mobile, Slack (Claude Tag), Chrome all drive the same Claude Code engine; Cowork packages it for non-terminal users with Dispatch for long-running background work.
10. **Tool scaling** — Tool search defers definitions and loads 3–5 on demand (up to 10k tools), keeping context lean for large MCP/custom tool catalogs.

### Surface comparison (where the agent engine runs)

| Surface | Entry point | Notes |
|---------|-------------|-------|
| Terminal CLI | `claude` | Full-featured; `--bg`, `--agent`, `--print` |
| VS Code / Cursor | Extension | Inline diffs, @-mentions, plan review |
| JetBrains | Plugin (needs CLI) | IntelliJ/PyCharm/WebStorm |
| Desktop app | Claude Desktop | Visual diffs, parallel sessions, Dispatch, computer use |
| Web | claude.ai/code | Browser/phone; `--teleport` to terminal; `--cloud` |
| Mobile | Claude iOS app | Start/continue tasks; Remote Control |
| Slack | Claude Tag | Admin-governed, sandboxed, channel memory |
| Chrome | Claude in Chrome | Web automation paired with Cowork |
| Agent SDK | `query()` | Python/TS library; loop in your process |
| Agent view | `claude agents` | Background session monitor/dispatch |

### Branding (partners)

Allowed: "Claude Agent", "Claude" (within an Agents menu), "{YourAgentName} Powered by Claude". Not permitted: "Claude Code", "Claude Cowork", Claude Code-branded visuals.