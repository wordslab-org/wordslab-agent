# Anthropic API Analysis — Agent Capabilities (Claude Managed Agents)

> **Docs:** `https://platform.claude.com/docs/en/managed-agents/quickstart`
> **Base API:** `https://api.anthropic.com/v1` | **Auth:** `x-api-key: $ANTHROPIC_API_KEY` + `anthropic-version: 2023-06-01`
> **Beta header:** `anthropic-beta: managed-agents-2026-04-01` (SDK sets automatically); memory store endpoints use `agent-memory-2026-07-22` instead
> **SDKs:** Python (`anthropic`), TypeScript (`@anthropic-ai/sdk`), Go, Java, C#, Ruby, PHP | **CLI:** `ant`
> **Description:** Anthropic exposes agent capabilities through **Claude Managed Agents** — a hosted, resource-oriented API where autonomous agents run inside Anthropic-managed (or self-hosted) sandboxes. Unlike a raw chat completions loop, the platform provisions the sandbox, executes pre-built and custom tools server-side, streams an event-based conversation, and manages the agent loop for you. The platform is organized around four first-class resources — **Agents**, **Environments**, **Sessions**, and **Events** — plus auxiliary resources for **Vaults** (credentials), **Memory Stores**, **Skills**, **Scheduled Deployments**, and **Multi-agent threads**. A REST surface (`/v1/agents`, `/v1/environments`, `/v1/sessions`, `/v1/vaults`, `/v1/memory_stores`, `/v1/deployments`) is complemented by official SDKs and the `ant` CLI.

---

## Table of Contents

1. [Platform Overview & Core Concepts](#1-platform-overview--core-concepts)
2. [Agent Configuration](#2-agent-configuration)
3. [Environments & Sandboxes](#3-environments--sandboxes)
4. [Sessions](#4-sessions)
5. [Session Event Stream & Events](#5-session-event-stream--events)
6. [Tools](#6-tools)
7. [MCP Connector](#7-mcp-connector)
8. [Skills](#8-skills)
9. [Permission Policies & Tool Confirmation](#9-permission-policies--tool-confirmation)
10. [Vaults & Credentials](#10-vaults--credentials)
11. [Multi-Agent Sessions](#11-multi-agent-sessions)
12. [Memory Stores](#12-memory-stores)
13. [Scheduled Deployments](#13-scheduled-deployments)
14. [Session Operations & Lifecycle](#14-session-operations--lifecycle)
15. [Capability Summary & Cross-Reference](#15-capability-summary--cross-reference)

---

## 1. Platform Overview & Core Concepts

### Main Concepts

Claude Managed Agents is built around these core abstractions:

- **Agent** — A reusable, *versioned* configuration that bundles the model, system prompt, tools, MCP servers, skills, and optional multi-agent roster. Created once and referenced by ID from every session.
- **Environment** — Configuration for *where* sessions run: an Anthropic-managed cloud sandbox or a self-hosted sandbox on your own infrastructure. Each session gets its own isolated Linux container even when sharing an environment.
- **Session** — A running agent instance within an environment, performing a specific task. Sessions are state machines (`idle` → `running` → `idle`/`terminated`) that maintain conversation history across multiple interactions.
- **Events** — The bidirectional, persisted messages exchanged between your application and the agent: user turns, tool results, status updates, agent messages, and span observability markers. Communication is fundamentally event-based over an SSE stream.
- **Thread** — In multi-agent sessions, each agent runs in its own context-isolated event stream (a session thread). The primary thread is the session-level stream; subagent threads are spawned at runtime.
- **Vault** — A workspace-scoped collection of credentials (OAuth, bearer, env-var) registered once and referenced by ID at session creation. Decouples secrets from reusable agent definitions.
- **Memory Store** — A workspace-scoped collection of text documents mounted as a directory inside the sandbox, giving the agent persistent memory that survives across sessions. Every change creates an immutable memory version.
- **Skill** — A reusable, filesystem-based resource (a `SKILL.md` plus supporting files) that supplies domain-specific expertise on demand via progressive disclosure. Pre-built (`xlsx`, `docx`, `pdf`…) or custom.
- **Deployment** — A scheduled, recurring trigger that autonomously starts sessions on a cron schedule.
- **Tool** — A capability the agent can call. Three categories: (1) pre-built **agent toolset** tools (bash, file ops, web search/fetch), (2) **MCP toolset** tools from connected MCP servers, (3) **custom tools** executed by your application.

### Agent Capabilities Map

| Capability | Description | Docs |
|------------|-------------|------|
| **Agent configuration** | Reusable, versioned definitions (model, system, tools, MCP, skills, multi-agent) | [agent-setup](https://platform.claude.com/docs/en/managed-agents/agent-setup) |
| **Environments** | Cloud or self-hosted sandboxes with packages, networking, pre-installed runtimes | [environments](https://platform.claude.com/docs/en/managed-agents/environments) |
| **Sessions** | Agent instances within environments, version pinning, per-session overrides | [sessions](https://platform.claude.com/docs/en/managed-agents/sessions) |
| **Event stream** | Bidirectional events, SSE streaming, interrupts, event deltas, history listing | [events-and-streaming](https://platform.claude.com/docs/en/managed-agents/events-and-streaming) |
| **Tools** | Pre-built agent toolset, custom tools, tool configuration & output handling | [tools](https://platform.claude.com/docs/en/managed-agents/tools) |
| **MCP connector** | Connect remote MCP servers; agent-scoped servers + session-scoped auth | [mcp-connector](https://platform.claude.com/docs/en/managed-agents/mcp-connector) |
| **Skills** | Attach pre-built or custom filesystem-based expertise to agents | [skills](https://platform.claude.com/docs/en/managed-agents/skills) |
| **Permission policies** | `always_allow` / `always_ask` controls for server-executed tools | [permission-policies](https://platform.claude.com/docs/en/managed-agents/permission-policies) |
| **Vaults** | Register per-user credentials once, reference by ID at session creation | [vaults](https://platform.claude.com/docs/en/managed-agents/vaults) |
| **Multi-agent** | Coordinator delegates to a roster of agents; isolated threads, shared sandbox | [multi-agent](https://platform.claude.com/docs/en/managed-agents/multi-agent) |
| **Memory stores** | Persistent cross-session memory mounted in the sandbox, with versioned audit trail | [memory](https://platform.claude.com/docs/en/managed-agents/memory) |
| **Scheduled deployments** | Run an agent on a recurring cron schedule; pause/unpause/archive, manual runs | [scheduled-deployments](https://platform.claude.com/docs/en/managed-agents/scheduled-deployments) |
| **Session operations** | Retrieve, list (paginated), update (mid-session tools/MCP), archive, delete sessions | [session-operations](https://platform.claude.com/docs/en/managed-agents/session-operations) |

### Platform Architecture

```
Your application (Python / TS / Go / Java / C# / Ruby / PHP / curl / ant CLI)
        │
        ▼
   Create Agent  ── model, system, tools[], mcp_servers[], skills[], multiagent?
        │          (versioned; update → new version; archive → read-only)
        ▼
   Create Environment ── config: cloud | self_hosted
        │                 packages[], networking: unrestricted | limited
        ▼
   Create Session ── agent (id | pinned | with_overrides), environment_id,
        │            vault_ids[], resources[] (memory stores), title
        ▼
   ┌──────────────── Agent Loop (server-side) ─────────────────┐
   │  1. Provision sandbox from environment config             │
   │  2. Stream opens (SSE) → send user.message events          │
   │  3. Claude picks tools → executes in sandbox              │
   │  4. Emit agent.* / span.* events in real time             │
   │  5. Pause on always_ask → user.tool_confirmation          │
   │  6. Custom tool → agent.custom_tool_use → app runs it     │
   │  7. session.status_idle when nothing more to do            │
   └────────────────────────────────────────────────────────────┘
        │
        ▼
   Event Stream (persisted, replayable, listable, filterable)
     ├── user.* events (you send)
     ├── agent.* events (messages, thinking, tool_use, tool_result)
     ├── session.* events (status_running, status_idle, errors, threads)
     ├── span.* events (model_request_start/end, outcome_evaluation)
     └── event_start / event_delta (opt-in streaming previews)
```

### Quickstart flow

The minimal end-to-end flow is four steps: (1) `POST /v1/agents` → `agent.id`; (2) `POST /v1/environments` → `environment.id`; (3) `POST /v1/sessions` with both IDs → `session.id`; (4) open `GET /v1/sessions/{id}/stream` (SSE) then `POST /v1/sessions/{id}/events` with a `user.message` event and process streamed `agent.*` events until `session.status_idle`.

---

## 2. Agent Configuration

An **agent** is a reusable, versioned resource. You create it once and reference it by ID each time you start a session. Agents are the primary unit of configuration reuse across many sessions.

### Agent configuration fields

| Field | Required | Description |
|-------|----------|-------------|
| `name` | Yes | Human-readable name. |
| `model` | Yes | Claude model ID string (e.g. `"claude-opus-4-8"`) or object `{"id": "...", "speed": "fast"}`. All Claude 4.5-family and later supported. |
| `system` | No | System prompt defining behavior/persona. Distinct from user messages (which describe the work). Can be cleared with `null`. |
| `tools` | No | Array combining pre-built agent tools, MCP toolsets, and custom tools. |
| `mcp_servers` | No | MCP servers (type `url`, unique `name`, `url`) providing standardized third-party capabilities. Max 20. |
| `skills` | No | Skills (pre-built `anthropic` or `custom` with `skill_id` + optional `version`). Max 20 per session across all agents. |
| `multiagent` | No | Coordinator declaration listing agents this agent can delegate to. |
| `description` | No | Description of what the agent does. Clearable with `null`. |
| `metadata` | No | Arbitrary key-value pairs for your own tracking. Merged at key level on update. |

### Create an agent

`POST /v1/agents` — SDK: `client.beta.agents.create(...)`.

```json
{
  "name": "Coding Assistant",
  "model": "claude-opus-4-8",
  "system": "You are a helpful coding agent.",
  "tools": [{"type": "agent_toolset_20260401"}]
}
```

Response echoes configuration plus `id`, `type`, `version` (starts at 1), `created_at`, `updated_at`, `archived_at`. The toolset's `default_config` shows the default permission policy (`always_allow`).

### Update an agent — versioned, optimistic concurrency

`POST /v1/agents/{id}` — requires `version` field matching the agent's current version; mismatch returns 409.

**Update semantics:**
- Omitted fields are preserved (partial update).
- **Scalar fields** (`model`, `system`, `name`, `description`): replaced. `system`/`description` clearable with `null`; `model`/`name` mandatory.
- **Array fields** (`tools`, `mcp_servers`, `skills`): fully replaced. Clear with `null` or empty array.
- **`multiagent`**: replaced wholesale (including `agents` roster); `null` clears it.
- **`metadata`**: merged at key level; set a key to `null` to delete it.
- **No-op detection**: if no change, existing version returned (no new version).
- **Coordinator rosters are not auto-updated**: pinned versions persist; update the coordinator to delegate to a new subagent version.

### Agent lifecycle

| Operation | Endpoint | Behavior |
|-----------|----------|----------|
| Create | `POST /v1/agents` | Returns version 1. |
| Retrieve | `GET /v1/agents/{id}` | Current configuration. |
| Update | `POST /v1/agents/{id}` | New version on change; `version` required. |
| List versions | `GET /v1/agents/{id}/versions` | Paginated full history. |
| Archive | `POST /v1/agents/{id}/archive` | Read-only; new sessions can't reference it; existing sessions continue. One-way. |

---

## 3. Environments & Sandboxes

An **environment** defines the sandbox where an agent runs. Created once, referenced by ID at session creation. Each session gets its own isolated sandbox (fresh Linux container) even when sharing an environment — sessions do not share filesystem state. Environments are **not versioned**.

### Create an environment

`POST /v1/environments` — `config` has `type: "cloud"` (this page) or `"self_hosted"` (see Self-hosted sandboxes).

```json
{
  "name": "python-dev",
  "config": {
    "type": "cloud",
    "networking": {"type": "unrestricted"}
  }
}
```

### Configuration options

**Packages** — pre-install packages into the sandbox before the agent starts; cached across sessions sharing the environment. Multiple managers run in alphabetical order (apt, cargo, gem, go, npm, pip). Versions optionally pinned.

| Field | Package manager | Example |
|-------|-----------------|---------|
| `apt` | System (apt-get) | `"ffmpeg"` |
| `cargo` | Rust | `"ripgrep@14.0.0"` |
| `gem` | Ruby | `"rails:7.1.0"` |
| `go` | Go modules | `"golang.org/x/tools/cmd/goimports@latest"` |
| `npm` | Node.js | `"express@4.18.0"` |
| `pip` | Python | `"pandas==2.2.0"` |

**Networking** — controls outbound sandbox access (does not affect `web_search`/`web_fetch` allowed domains):

| Mode | Description |
|------|-------------|
| `unrestricted` | Full outbound except a safety blocklist. Default. |
| `limited` | Restricts to `allowed_hosts`; set `allow_mcp_servers` / `allow_package_managers` to allow additional access. |

`limited` networking fields:
- `allowed_hosts` — bare hostnames or wildcard patterns (e.g. `*.example.com`); no scheme/port/path.
- `allow_mcp_servers` (default `false`) — outbound to MCP server endpoints beyond `allowed_hosts`.
- `allow_package_managers` (default `false`) — outbound to public registries (PyPI, npm) beyond `allowed_hosts`.

> Production guidance: use `limited` with an explicit `allowed_hosts` list (least privilege).

### Lifecycle & management

| Operation | Endpoint | Behavior |
|-----------|----------|----------|
| List | `GET /v1/environments` | All environments. |
| Retrieve | `GET /v1/environments/{id}` | Single environment. |
| Archive | `POST /v1/environments/{id}/archive` | Read-only; existing sessions continue. |
| Delete | `DELETE /v1/environments/{id}` | Only if no sessions reference it. |

Cloud sandboxes include common runtimes out of the box (languages, databases, utilities). Environments persist until explicitly archived or deleted.

---

## 4. Sessions

A **session** is an agent instance within an environment. It requires an `agent` ID and an `environment` ID. Sessions follow a two-step lifecycle: create (provisions the sandbox) → send a user event (starts work). The session acts as a state machine tracking progress while events drive execution.

### Session statuses

| Status | Description |
|--------|-------------|
| `idle` | Waiting for input (messages or tool confirmations). Sessions start here. |
| `running` | Actively executing. |
| `rescheduling` | Transient error; retrying automatically. |
| `terminated` | Ended due to unrecoverable error. |

### Creating a session — three agent reference forms

`POST /v1/sessions`:

1. **Agent ID string** — `"agent": "$AGENT_ID"` → uses latest agent version.
2. **Pinned version** — `"agent": {"type": "agent", "id": "...", "version": 1}` → exact version control for staged rollouts.
3. **Overrides** — `"agent": {"type": "agent_with_overrides", "id": "...", "model": {...}, "system": null, ...}` → change model/system/tools/mcp_servers/skills for one session without versioning the agent.

**Override rules per field:**
- Omit → inherit from referenced agent version.
- `null` (or empty array for list fields) → clear for this session. Exceptions: `model` is never clearable (`null` → 400 `agent_model_required`); `tools: null` → 400 only when effective `skills` is non-empty (skills require the `read` tool).
- Set to a value → full replacement (overrides never merge).

Overrides are session-local; they don't modify the agent resource or create a new version. The response's `agent` object reflects the resolved snapshot (with `id`/`version` still identifying the base agent).

### Other session-creation parameters

| Parameter | Description |
|-----------|-------------|
| `environment_id` | Required. The sandbox configuration. |
| `vault_ids` | Array of vault IDs supplying MCP/env-var credentials for this session. |
| `resources` | Array of resources attached at creation: `memory_store` entries (with `access: read_write|read_only`, optional `instructions`). |
| `title` | Optional human-readable title. |

### Starting work

Creating a session provisions the sandbox but does no work. Send events via `POST /v1/sessions/{id}/events` (a `user.message` event kicks off the agent loop). Memory stores attach only at creation time (no add/remove on running sessions).

---

## 5. Session Event Stream & Events

Communication is **event-based**. You send user/system events to the agent and receive session/span/agent events back for observability and control. Event type strings follow a `{domain}.{action}` convention (stream-only delta previews are the exception). Every persisted event includes a `processed_at` timestamp (null = queued).

### Event type catalog

**User events** (you send):

| Type | Description |
|------|-------------|
| `user.message` | User message with text content. Starts/continues work. |
| `user.interrupt` | Stop the agent mid-execution; redirect with a following `user.message`. |
| `user.custom_tool_result` | Response to a custom tool call. |
| `user.tool_confirmation` | Approve/deny an agent or MCP tool call requiring confirmation. |
| `user.define_outcome` | Define an outcome for the agent to work toward. |
| `user.tool_result` | Self-hosted environments only: provide `agent_toolset` results (SDK/CLI do this automatically). |

**Agent events** (you receive):

| Type | Description |
|------|-------------|
| `agent.message` | Agent response with text content blocks. |
| `agent.thinking` | Agent thinking content (separate from messages). |
| `agent.tool_use` | Agent invokes a pre-built agent tool (bash, file ops…). |
| `agent.tool_result` | Result of a pre-built agent tool. |
| `agent.mcp_tool_use` | Agent invokes an MCP server tool. |
| `agent.mcp_tool_result` | Result of an MCP tool. |
| `agent.custom_tool_use` | Agent invokes a custom tool → respond with `user.custom_tool_result`. |
| `agent.thread_context_compacted` | History compacted to fit context window. |
| `agent.thread_message_received` | Multi-agent: agent delivered result to coordinator. |
| `agent.thread_message_sent` | Multi-agent: coordinator sent follow-up to another agent. |

**Session events** (you receive):

| Type | Description |
|------|-------------|
| `session.status_running` | Agent actively processing. |
| `session.status_idle` | Finished, waiting for input; includes `stop_reason`. |
| `session.status_rescheduled` | Transient error; retrying. |
| `session.status_terminated` | Unrecoverable error. |
| `session.deleted` | Session deleted; stream terminates. |
| `session.updated` | Update changed ≥1 field (applies next turn). |
| `session.error` | Error with typed `error` object + `retry_status`. |
| `session.thread_created` / `session.thread_status_running` / `session.thread_status_idle` / `session.thread_status_rescheduled` / `session.thread_status_terminated` | Multi-agent thread lifecycle. |

**Span events** (observability markers):

| Type | Description |
|------|-------------|
| `span.model_request_start` / `span.model_request_end` | Model inference call boundaries; end includes `model_usage` token counts. |
| `span.outcome_evaluation_start` / `_ongoing` / `_end` | Outcome evaluation lifecycle. |

**System events** (you send):

| Type | Description |
|------|-------------|
| `system.message` | Update the agent's system prompt between turns. Only Claude Opus 4.8. |

**Event deltas** (stream-only, opt-in, never persisted):

| Type | Description |
|------|-------------|
| `event_start` | A previewed event started generating; carries upcoming event's `type` and `id`. |
| `event_delta` | Incremental content for a previewed event, identified by `event_id`. |

### Sending events

`POST /v1/sessions/{id}/events` with `{"events": [...]}`. Multiple events per request supported. The `user.interrupt` + `user.message` pattern redirects mid-execution.

### Streaming events

`GET /v1/sessions/{id}/events/stream` (SSE). Open the stream *before* sending events to avoid a race (only events emitted after stream open are delivered). Reconnect pattern: open stream → list event history to seed seen IDs → tail live stream skipping seen IDs.

**Event deltas (previews):** opt in per stream connection with `event_deltas[]=agent.message` (repeatable query param; accepted values `agent.message` and `agent.thinking`; others → 400). A previewed `agent.message` gets a single `event_start` then `event_delta` events (`delta.type: "content_delta"`, with `index`). Previews are best-effort; the buffered `agent.message` is always authoritative. Reconcile per model request: `span.model_request_start` → `event_start` → `event_delta`s → buffered `agent.message` → `span.model_request_end`. SDK accumulator helpers exist (Python/TS/Go).

### Listing past events

`GET /v1/sessions/{id}/events` — paginated, supports a `types[]` filter (e.g. `agent.tool_use`, `agent.tool_result`). Each event has `id`, `type`, `processed_at`.

---

## 6. Tools

Tools control what the agent can do within a session. Three categories: pre-built agent tools, MCP tools, and custom tools. You control availability in the agent configuration.

### Available pre-built tools

All enabled by default when the toolset is included. Outputs exceeding 100,000 tokens are auto-written to a sandbox file; the model gets a truncated preview plus the file path.

| Tool | Name | Description |
|------|------|-------------|
| Bash | `bash` | Execute bash commands in a shell session |
| Read | `read` | Read a file from the sandbox filesystem |
| Write | `write` | Write a file to the sandbox filesystem |
| Edit | `edit` | String replacement in a file |
| Glob | `glob` | Fast file pattern matching |
| Grep | `grep` | Text search with regex |
| Web fetch | `web_fetch` | Fetch content from a URL |
| Web search | `web_search` | Search the web |

### Configuring the toolset

Include `{"type": "agent_toolset_20260401"}` in `tools`. Use `default_config` (baseline for all tools) and `configs` (per-tool overrides) to disable tools or set `permission_policy`:

```json
{
  "type": "agent_toolset_20260401",
  "default_config": {"enabled": false},
  "configs": [
    {"name": "bash", "enabled": true},
    {"name": "read", "enabled": true}
  ]
}
```

- **Disable specific tools**: `"enabled": false` per tool config.
- **Enable only specific tools**: set `default_config.enabled: false`, then enable per tool.
- **Permission policy**: `default_config.permission_policy` or per-tool `configs[].permission_policy`.

### Custom tools

Custom tools are user-defined; your application executes them and returns results. Analogous to client-executed tools in the Messages API. The model emits a structured request; your code runs it; the result flows back.

```json
{
  "type": "custom",
  "name": "get_weather",
  "description": "Get current weather for a location",
  "input_schema": {
    "type": "object",
    "properties": {"location": {"type": "string", "description": "City name"}},
    "required": ["location"]
  }
}
```

**Best practices:**
- Extremely detailed descriptions (3–4 sentences; explain when to use/not use, parameter meaning, caveats).
- Consolidate related operations into fewer tools with an `action` parameter.
- Use meaningful namespacing (`db_query`, `storage_read`).
- Return only high-signal information (semantic IDs, only needed fields).

Custom tools are governed by you, not permission policies. Self-hosted sandboxes can serve custom tools (including MCP-server wrappers inside your network).

### Handling custom tool calls

When the agent invokes a custom tool: it emits `agent.custom_tool_use` → session pauses (`session.status_idle`, `stop_reason.type: requires_action`, blocking event IDs in `stop_reason.event_ids`) → you send `user.custom_tool_result` with `custom_tool_use_id` → session resumes.

---

## 7. MCP Connector

Connect [Model Context Protocol](https://modelcontextprotocol.io) servers to agents for external tools, data, and services. MCP configuration is split across two steps: (1) **agent creation** declares servers by name/URL; (2) **session creation** supplies auth via vaults. This keeps secrets out of reusable agent definitions.

### Declare MCP servers on the agent

`mcp_servers` array (each: `type: "url"`, unique `name` (1–255 chars), `url` ≤2048 chars). Each server needs a matching `mcp_toolset` entry in `tools` with `mcp_server_name` = server `name`.

```json
{
  "mcp_servers": [{"type": "url", "name": "github", "url": "https://api.githubcopilot.com/mcp/"}],
  "tools": [
    {"type": "agent_toolset_20260401"},
    {"type": "mcp_toolset", "mcp_server_name": "github"}
  ]
}
```

**Constraints:** max 20 MCP servers; names unique; every server referenced by a toolset and vice versa (dangling → rejected). MCP toolset defaults to `always_ask` permission policy. Remote servers must support MCP streamable HTTP transport; private servers via MCP tunnels.

### Configure which MCP tools are available

Same `default_config`/`configs` shape as the agent toolset, applied to tools the MCP server exposes. `name` = bare tool name from the server. Default: all exposed tools enabled. Use `default_config.enabled: false` + explicit enables for an allowlist.

### MCP tool output handling

Outputs >100,000 tokens auto-written to a sandbox file; model gets truncated preview + file path.

### Provide auth at session creation

Pass `vault_ids` at session creation. Credentials matched by URL — the vault must contain a credential whose `mcp_server_url` exactly matches the declared `url`. No match → unauthenticated attempt. In multi-agent sessions, vault credentials apply to every thread.

### Connection & auth failures

Session creation does **not** validate MCP connectivity/credentials. On failure, the session still starts; a `session.error` event is emitted with `mcp_server_name` and `retry_status`:

| Error type | Meaning |
|-----------|---------|
| `mcp_connection_failed_error` | Server unreachable (network/timeout/non-auth HTTP failure). |
| `mcp_authentication_failed_error` | Server reached but rejected the vault credential. |

Retried on the next `idle` → `running` transition. You decide whether to block, rotate credentials, or continue without the server.

---

## 8. Skills

**Skills** are reusable, filesystem-based resources giving agents domain-specific expertise (workflows, context, best practices) loaded on demand — only impacting the context window when relevant (progressive disclosure). Two types:

- **Pre-built Anthropic skills** (`anthropic`): common document tasks — `pptx`, `xlsx`, `docx`, `pdf`.
- **Custom skills** (`custom`): authored by you (a directory with `SKILL.md` + supporting files) and uploaded as a zip or individual files.

### Create a custom skill

`POST /v1/skills` (beta header `skills-2025-10-02`; SDK sets automatically). Multipart upload of files. Returns `skill_*` ID and `latest_version`. `display_title` is optional (derived from `SKILL.md`; must be unique among workspace custom skills).

### Attach skills to an agent

`skills` array on agent creation. Max **20 skills per session** across all agents (multi-agent).

| Field | Description |
|-------|-------------|
| `type` | `anthropic` (pre-built) or `custom`. |
| `skill_id` | Short name for Anthropic skills (e.g. `xlsx`); `skill_*` ID for custom skills. |
| `version` | Custom only. Pin a version or `latest`. Defaults to `latest`. |

```json
"skills": [
  {"type": "anthropic", "skill_id": "xlsx"},
  {"type": "custom", "skill_id": "skill_abc123", "version": "latest"}
]
```

---

## 9. Permission Policies & Tool Confirmation

Permission policies control whether **server-executed tools** (agent toolset + MCP toolset) run automatically or wait for approval. Custom tools are executed by your application and are **not** governed by permission policies.

### Policy types

| Policy | Behavior |
|--------|----------|
| `always_allow` | Tool executes automatically, no confirmation. |
| `always_ask` | Session pauses, waits for your approval before executing. |

Defaults: agent toolset → `always_allow`; MCP toolset → `always_ask`.

### Setting a policy

In the agent's `tools` config at creation (or update). `default_config.permission_policy` applies to all tools in a toolset; per-tool `configs[].permission_policy` overrides.

```json
{
  "type": "agent_toolset_20260401",
  "default_config": {"permission_policy": {"type": "always_allow"}},
  "configs": [{"name": "bash", "permission_policy": {"type": "always_ask"}}]
}
```

Running sessions keep the config they were created with; updates apply to sessions created afterward.

### Respond to confirmation requests

When a tool with `always_ask` is invoked:
1. Session emits `agent.tool_use` or `agent.mcp_tool_use`.
2. Session pauses (`session.status_idle`, `stop_reason.type: requires_action`, blocking IDs in `stop_reason.event_ids`). Waits indefinitely.
3. You send `user.tool_confirmation` for each blocking event — `tool_use_id`, `result: "allow"|"deny"`, optional `deny_message`. Multiple confirmations per request allowed.
4. Once all resolved → `running`. Allowed tools execute; denied tools return a rejected tool result (with your `deny_message`) to the agent.

In multi-agent sessions, confirmation requests are cross-posted to the primary thread with `session_thread_id`; responses are auto-routed.

---

## 10. Vaults & Credentials

**Vaults** and credentials are authentication primitives: register per-user credentials once, reference by ID at session creation. Avoids running your own secret store, transmitting tokens on every call, and losing track of which end user an agent acted on behalf of. Vaults are workspace-scoped (anyone with a workspace API key can reference them).

### Create a vault

`POST /v1/vaults` — `display_name` + optional `metadata` (key-value, maps to your user records). Returns `vlt_...` ID.

### Add a credential

Three credential categories:

- **`mcp_oauth`** — OAuth 2.0 for MCP servers. `access_token`, `expires_at`, optional `refresh` block (`token_endpoint`, `client_id`, `scope`, `refresh_token`, `token_endpoint_auth.type`: `none` | `client_secret_basic` | `client_secret_post`). Anthropic refreshes tokens automatically.
- **`static_bearer`** — Fixed bearer token (API key/PAT). `mcp_server_url` + `token`.
- **`environment_variable`** — Authenticates via an env var in the sandbox. `secret_name` + `secret_value`; stored as an opaque placeholder; substituted at egress (agent never sees the value). `networking.allowed_hosts` scopes which hosts use the secret; `injection_location` (`header`, `body`) scopes which request parts. Not supported with self-hosted sandboxes.

MCP credentials keyed by `mcp_server_url`; env-var credentials keyed by `secret_name`. Credential values are write-only (never returned). Constraints: unique key per vault (duplicate → 409); keys immutable; max 20 credentials per vault.

### Reference the vault at session creation

`vault_ids` array on `POST /v1/sessions`. Runtime: no matching MCP credential → unauthenticated attempt; multiple vaults with a match → first wins; in multi-agent sessions, credentials apply to every thread.

### Credential lifecycle & rotation

Credentials re-resolved periodically (rotation/archival/deletion propagate to running sessions without restart). Updateable: secret values, `display_name`, `injection_location` (merge per field). Structural fields (`mcp_server_url`, `secret_name`, `token_endpoint`, `client_id`) locked — archive and recreate to change.

**Webhook events:** `vault.archived`, `vault.deleted`, `vault_credential.archived`, `vault_credential.deleted`, `vault_credential.refresh_failed`.

**Diagnose OAuth refresh failure:** `POST /v1/vaults/{vault_id}/credentials/{credential_id}/mcp_oauth_validate` → `status: "valid" | "invalid" | "unknown"` plus `mcp_probe` and `refresh` details.

### Other operations

List (paginated, `include_archived`), archive vault (cascades to credentials; purges secrets, retains records), archive credential (purges secret, frees the key), hard delete (no retention).

---

## 11. Multi-Agent Sessions

One agent (the **coordinator**) delegates to others to complete complex work. Agents act in parallel with isolated context but share the same sandbox, filesystem, and vault credentials. Each agent runs in its own **session thread** (context-isolated event stream). The coordinator reports activity in the **primary thread** (the session-level stream); additional threads spawn at runtime on delegation. Threads are persistent — follow-ups retain prior turns.

Each agent uses its own configuration (model, system, tools, MCP servers, skills). Session-level agent config overrides apply to the coordinator and its `self` copies. Tools/MCP/context are not shared.

### What to delegate

- **Parallelization** — fan out independent subtasks, synthesize results.
- **Specialization** — route to domain-focused agents (security, docs) rather than overloading one agent.
- **Escalation** — consult a more capable agent/model for complex subtasks.

### Configure the coordinator

`multiagent` field on agent creation:

```json
"multiagent": {
  "type": "coordinator",
  "agents": [
    {"type": "agent", "id": "$REVIEWER_AGENT_ID"},
    {"type": "agent", "id": "$TEST_WRITER_AGENT_ID", "version": 2},
    {"type": "self"}
  ]
}
```

Roster entry forms:
- `{"type": "agent", "id": ...}` — reference by ID; pinned to latest version at coordinator creation time (if no `version`).
- `{"type": "agent", "id": ..., "version": n}` — pin a specific version.
- `{"type": "self"}` — coordinator spawns copies of itself (overrides apply to copies; ID-referenced entries unaffected).

The roster is snapshotted at coordinator creation/update; referenced agents don't auto-pick up later updates (update the coordinator to delegate to newer versions). Max 20 unique agents in roster; max 1 level of delegation (depth > 1 ignored); multiple copies of each agent allowed.

### MCP servers in multi-agent

MCP servers are agent-scoped (each agent declares its own); vault credentials are session-scoped (`vault_ids` apply to every thread). Include a vault credential for every MCP server across all agents; to limit access, declare only needed servers per agent.

### Threads

Max **25 concurrent threads**. The session-level stream is the **primary thread** (`parent_thread_id` null); it surfaces a condensed view (thread start/end, blocking events). Session `status` aggregates all threads (running if any thread is running).

| Operation | Endpoint |
|-----------|----------|
| List threads | `GET /v1/sessions/{id}/threads` |
| Interrupt a thread | `user.interrupt` with `session_thread_id` (omit → primary) |
| Archive a thread | `POST /v1/sessions/{id}/threads/{thread_id}/archive` (only if `idle`; interrupt first if running/blocked) |
| Stream thread events | `GET /v1/sessions/{id}/threads/{thread_id}/stream` |
| List thread events | `GET /v1/sessions/{id}/threads/{thread_id}/events` |

**Primary thread events:** `session.thread_created`, `session.thread_status_running`, `session.thread_status_idle`, `session.thread_status_terminated`, `agent.thread_message_received`, `agent.thread_message_sent`.

Tool permission requests and custom tool calls from subagents are cross-posted to the primary thread with `session_thread_id`; you post `user.tool_confirmation`/`user.custom_tool_result` and the server routes to the correct thread.

Interrupting a child thread blocked on `requires_action` marks pending tool calls denied and re-emits `session.thread_status_idle` with `stop_reason: end_turn` (no model sampling).

---

## 12. Memory Stores

Each session starts with fresh context by default. **Memory stores** let the agent carry information across sessions (user preferences, project conventions, prior mistakes, domain context). A memory store is a workspace-scoped collection of text documents optimized for Claude, mounted as a directory inside the session sandbox. The agent reads/writes with the same file tools; a note describing each mount is auto-added to the system prompt. Requires the agent toolset.

Each **memory** is addressed by a path and is read/editable via API or Console. Every change creates an immutable **memory version** (audit trail + point-in-time recovery). Memory store endpoints use the `agent-memory-2026-07-22` beta header (do **not** combine with `managed-agents-2026-04-01` on these endpoints).

### Create a memory store

`POST /v1/memory_stores` — `name` + `description` (passed to the agent). Returns `memstore_...` ID. Optionally seed with memories before any session runs (`POST /v1/memory_stores/{id}/memories` with `path` + `content`). Limits: 100 kB (~25k tokens) per memory; 2,000 memories per store.

### Attach to a session

In the session's `resources[]` array at **session creation only** (no add/remove on running sessions):

```json
"resources": [{
  "type": "memory_store",
  "memory_store_id": "$store_id",
  "access": "read_write",
  "instructions": "User preferences and project context. Check before starting any task."
}]
```

- `access`: `read_write` (default) or `read_only`.
- `instructions`: session-specific guidance, ≤4,096 chars.
- Max **8 memory stores per session**.

> Security: `read_write` + untrusted input risks prompt injection writing malicious memory later read as trusted. Use `read_only` for reference/shared material.

### How the agent accesses memory

Each store mounts under `/mnt/memory/{slug}/` (slug = sanitized display name; exact path in `mount_path` on the session resource). Writes under the mount path persist to the store and sync across sessions sharing it; writes elsewhere under `/mnt/memory/` are container-local scratch (lost on session end). `access` enforced at filesystem level. Reads/writes appear as normal `agent.tool_use`/`agent.tool_result` events.

### Memory CRUD

| Operation | Endpoint | Notes |
|-----------|----------|-------|
| List memories | `GET /v1/memory_stores/{id}/memories` | `path_prefix` (ends `/`, whole-segment match), `depth` (0/1 only). Stable server-defined order. |
| Read a memory | `GET /v1/memory_stores/{id}/memories/{mem_id}` | Full content. |
| Create a memory | `POST /v1/memory_stores/{id}/memories` | `path` + `content`. Does not overwrite. |
| Update a memory | `POST /v1/memory_stores/{id}/memories/{mem_id}` | Change `content` and/or `path` (rename). |
| Delete a memory | `DELETE /v1/memory_stores/{id}/memories/{mem_id}` | |

**Safe edits (optimistic concurrency):** pass `precondition: {"type": "content_sha256", "content_sha256": ...}`; update applies only if stored hash matches. On mismatch, re-read and retry.

### Memory versions (audit)

Every mutation creates an immutable memory version (`memver_...`). Versions belong to the store and survive parent memory deletion. Retained 30 days (recent versions always kept). No dedicated restore — retrieve a version's `content` and write it back.

| Operation | Endpoint | Notes |
|-----------|----------|-------|
| List versions | `GET /v1/memory_stores/{id}/memory_versions` | Newest first; `memory_id` filter. |
| Retrieve version | `GET /v1/memory_stores/{id}/memory_versions/{version_id}` | Full content. |
| Redact version | `POST /v1/memory_stores/{id}/memory_versions/{version_id}/redact` | Scrubs content, preserves audit trail. Cannot redact current head — write new version first. |

### Manage stores

`create`, `retrieve`, `update`, `list` (paginated, `include_archived`), `archive` (one-way, read-only, no new attachments), `delete` (permanent, removes all memories/versions).

### Best practices

Use focused stores (per user/team/project); condense/prune before filling; attach fresh stores and keep originals `read_only`; limit write access. A full store rejects new writes (existing memories stay readable/editable). A "dreaming session" can consolidate fragmented content into a new output store.

---

## 13. Scheduled Deployments

A **scheduled deployment** lets an agent start sessions autonomously on a recurring cron schedule. Managed via the Deployments API (part of the Claude API).

### Create a deployment

`POST /v1/deployments` — requires agent config, environment config, an initial `user.message` event (starts the session's work), and a `schedule`. Optionally accepts files, GitHub repos, memory stores, and vaults.

```json
{
  "name": "Weekly compliance scan",
  "agent": "$AGENT_ID",
  "environment_id": "$ENVIRONMENT_ID",
  "initial_events": [
    {"type": "user.message", "content": [{"type": "text", "text": "Run the weekly compliance scan."}]}
  ],
  "schedule": {"type": "cron", "expression": "0 20 * * 5", "timezone": "America/New_York"}
}
```

Response includes `schedule.upcoming_runs_at` (next fire times). Up to **1,000 scheduled deployments per organization**.

**Cron/timezone semantics:** standard POSIX cron (minute hour day-of-month month day-of-week); IANA timezone; literal wall-clock matching (DST-aware — spring-forward times don't trigger, fall-back times trigger twice). Up to 10s jitter for load distribution.

### Deployment runs

Each trigger attempt generates a **deployment run** record (success/failure tracked independent of session lifecycle). Successful runs include `session_id`; failed runs include `error.type` (e.g. `environment_archived_error`, `agent_archived_error`, `session_rate_limited_error`).

`GET /v1/deployment_runs` — list, filter by `deployment_id` and `has_error`. `GET /v1/deployment_runs/{id}` — retrieve single. Webhook events carry run IDs.

### Lifecycle management

| Operation | Endpoint | Behavior |
|-----------|----------|----------|
| Pause | `POST /v1/deployments/{id}/pause` | Suppresses future triggers; running sessions continue; manual runs still allowed. `paused_reason: {"type": "manual"}`. |
| Unpause | `POST /v1/deployments/{id}/unpause` | Resumes from next occurrence; missed triggers not backfilled. |
| Archive | `POST /v1/deployments/{id}/archive` | Terminal; schedule terminates, cannot modify. |
| Manual run | `POST /v1/deployments/{id}/run` | Creates a session immediately; run with `trigger_context.type: "manual"`. Useful for testing. |

**Failure behavior:** rate-limit → `session_rate_limited_error` run (no retry, next occurrence retries); agent archived/deleted → deployment auto-archived; subagent archived → failed run + auto-pause (update agent and resume); other unrecoverable errors (archived env/vault) → failed run + auto-pause. `paused_reason.error.type` mirrors the failed run's `error.type`.

---

## 14. Session Operations & Lifecycle

Once a session exists, use these operations to read, update, archive, or delete it.

### Updating the agent configuration mid-session

`POST /v1/sessions/{id}` — update `agent.tools` and `agent.mcp_servers` (including permission policies) mid-session without a new agent version. Session-local; doesn't propagate to the agent resource. Only `tools`/`mcp_servers` can change after creation (use overrides at creation for `model`/`system`/`skills`; `system` is fixed for the session — but `system.message` events can replace the effective system prompt between turns on supported models). Update is full-replacement (GET → modify → POST to preserve). Session must be `idle` (interrupt first if running).

### Retrieving & listing sessions

`GET /v1/sessions/{id}` — retrieve (includes `status`). `GET /v1/sessions` — paginated list with `limit`, `agent_id` filter, `order` (`asc`/`desc`, default `desc`), and cursor pagination (`prev_page`/`next_page`; cursors encode `order` — reusing with a different order returns 400).

### Archiving & deleting

| Operation | Endpoint | Behavior |
|-----------|----------|----------|
| Archive | `POST /v1/sessions/{id}/archive` | Prevents new events, preserves history. Cannot archive `running` (interrupt first). |
| Delete | `DELETE /v1/sessions/{id}` | Permanently removes record, events, sandbox. Cannot delete `running` (interrupt first). Independent resources (files, memory stores, vaults, skills, environments, agents) unaffected. |

---

## 15. Capability Summary & Cross-Reference

| Capability | Primary resource(s) | Key endpoints / SDK calls | Core parameters |
|------------|---------------------|--------------------------|-----------------|
| Agent configuration | Agent (versioned) | `POST/GET /v1/agents`, `.../versions`, `.../archive` | `name`, `model`, `system`, `tools`, `mcp_servers`, `skills`, `multiagent`, `description`, `metadata` |
| Environments | Environment (not versioned) | `POST /v1/environments`, `.../archive`, `DELETE` | `config.type` (cloud/self_hosted), `packages`, `networking` (unrestricted/limited, `allowed_hosts`, `allow_mcp_servers`, `allow_package_managers`) |
| Sessions | Session | `POST /v1/sessions`, `GET .../sessions/{id}`, `.../archive`, `DELETE` | `agent` (string/pinned/overrides), `environment_id`, `vault_ids`, `resources`, `title` |
| Event stream | Event | `POST /v1/sessions/{id}/events`, `GET .../events/stream`, `GET .../events` | `events[]` (user.* / system.*), `types[]` filter, `event_deltas[]` |
| Tools | Agent `tools[]` | Agent create/update | `agent_toolset_20260401` (with `default_config`/`configs`), `mcp_toolset`, `custom` (name/description/input_schema) |
| MCP connector | Agent `mcp_servers` + `tools` | Agent create/update | `type: url`, `name`, `url`, matching `mcp_toolset.mcp_server_name` |
| Skills | Skill + agent `skills[]` | `POST /v1/skills` (upload), agent create | `type` (anthropic/custom), `skill_id`, `version` |
| Permission policies | Agent `tools` config | Agent create/update | `permission_policy: {type: always_allow \| always_ask}`, `default_config`, per-tool `configs` |
| Vaults & credentials | Vault + credential | `POST /v1/vaults`, `.../credentials`, `.../archive`, `mcp_oauth_validate` | `display_name`, `metadata`, `auth` (mcp_oauth/static_bearer/environment_variable) |
| Multi-agent | Agent `multiagent` + session threads | Agent create, `GET .../threads`, `.../threads/{id}/stream` | `multiagent.type: coordinator`, `agents[]` (agent/self), `session_thread_id` |
| Memory stores | Memory store + memory + version | `/v1/memory_stores`, `.../memories`, `.../memory_versions`, `.../redact` | `name`, `description`, `path`, `content`, `access`, `instructions`, `precondition`, `path_prefix`, `depth` |
| Scheduled deployments | Deployment + deployment run | `POST /v1/deployments`, `.../pause`, `.../unpause`, `.../archive`, `.../run`, `/v1/deployment_runs` | `name`, `agent`, `environment_id`, `initial_events`, `schedule` (cron/expression/timezone) |

### Key design principles

1. **Resource-oriented & versioned** — Agents and memories are versioned; environments and sessions are not. Optimistic concurrency via `version`/`content_sha256`.
2. **Event-based communication** — Everything flows through the persisted event stream; bidirectional, replayable, filterable, with opt-in streaming previews.
3. **Server-executed agent loop** — The platform provisions the sandbox, runs the pre-built tools, manages context compaction, and streams events; your application owns user events, custom tool execution, and confirmations.
4. **Secrets decoupled from definitions** — Vaults hold credentials; agents declare MCP servers by URL; sessions bind them via `vault_ids`.
5. **Granular permission control** — `always_allow`/`always_ask` per toolset and per tool, with mid-execution pauses and `user.tool_confirmation` responses.
6. **Cross-session persistence** — Memory stores (mounted filesystems with versioned audit trails) give agents durable, reviewable memory.
7. **Autonomous scheduling** — Cron deployments with run history, manual triggers, pause/unpause, and auto-pause on unrecoverable failures.
8. **Multi-agent orchestration** — Coordinator + roster (by ID or `self`), isolated threads sharing a sandbox, cross-posted blocking events with auto-routed responses.

### Rate limits

| Operation | Limit |
|-----------|-------|
| Create endpoints (agents, sessions, environments) | 300 req/min per org |
| Read endpoints (retrieve, list, stream) | 1,200 req/min per org |

Organization-level spend limits and usage-tier rate limits also apply.

### Branding

Partners may use "Claude Agent", "Claude" (within an Agents menu), or "{YourAgentName} Powered by Claude". Not permitted: "Claude Code", "Claude Cowork", or Claude Code-branded visuals.