# Unified Agent Platform — Summary Specification

> Aggregated synthesis of nine platform studies: **Anthropic** (Claude Managed Agents), **IBM Bob**, **Claude** (Agent SDK / Claude Code / Cowork), **Codex**, **Google Gemini** (Managed Agents / Interactions API), **IBM watsonx Orchestrate**, **Mistral** (Agents & Conversations), **OpenAI** (Agents SDK), and **Vibe** (Vibe Work + Vibe Code).

This document is the **union** of all capabilities and use cases found across the nine source files. It is organized as:

1. **A concept introduction** so the material is approachable to an end user.
2. **A cross-system terminology map** showing where different products use different names for the same concept.
3. **An exhaustive processing pipeline** — the ordered sequence of steps that a complete agent platform performs — where every capability from every system is placed, each step lists the **alternative approaches** different systems take, and a **super-API specification** is given for that stage.

The spec is written from the end user's perspective ("you"). Where a capability is unique to one system it is still included, marked with the originating system in parentheses. Field names are presented as a unified, vendor-neutral API surface; the mapping to each system's native names is given in the cross-reference tables.

---

## Table of Contents

- [Part I — Concepts & Terminology](#part-i--concepts--terminology)
  - [1. What an Agent Platform Is](#1-what-an-agent-platform-is)
  - [2. The Core Objects](#2-the-core-objects)
  - [3. Cross-System Terminology Map](#3-cross-system-terminology-map)
- [Part II — The Exhaustive Processing Pipeline](#part-ii--the-exhaustive-processing-pipeline)
  - [Stage 0 — Platform Onboarding & Authentication](#stage-0--platform-onboarding--authentication)
  - [Stage 1 — Agent Definition & Configuration](#stage-1--agent-definition--configuration)
  - [Stage 2 — Models, Providers & Reasoning](#stage-2--models-providers--reasoning)
  - [Stage 3 — Environments & Sandboxes](#stage-3--environments--sandboxes)
  - [Stage 4 — Sessions, Threads, Runs & Interactions](#stage-4--sessions-threads-runs--interactions)
  - [Stage 5 — The Agent Loop, Turns, Streaming & Events](#stage-5--the-agent-loop-turns-streaming--events)
  - [Stage 6 — Tools (Built-in, Custom & Function Calling)](#stage-6--tools-built-in-custom--function-calling)
  - [Stage 7 — MCP Connectors](#stage-7--mcp-connectors)
  - [Stage 8 — Skills](#stage-8--skills)
  - [Stage 9 — Permissions, Approvals & Human Review](#stage-9--permissions-approvals--human-review)
  - [Stage 10 — Hooks & Lifecycle Callbacks](#stage-10--hooks--lifecycle-callbacks)
  - [Stage 11 — Credentials, Secrets & Vaults](#stage-11--credentials-secrets--vaults)
  - [Stage 12 — Multi-Agent Orchestration](#stage-12--multi-agent-orchestration)
  - [Stage 13 — Memory & Knowledge (RAG)](#stage-13--memory--knowledge-rag)
  - [Stage 14 — Workflows, Scheduled Tasks & Automation](#stage-14--workflows-scheduled-tasks--automation)
  - [Stage 15 — Observability, Tracing & Evaluation](#stage-15--observability-tracing--evaluation)
  - [Stage 16 — Channels, Voice & Embedded Chat](#stage-16--channels-voice--embedded-chat)
  - [Stage 17 — Extensions, Plugins, Marketplaces & Interoperability](#stage-17--extensions-plugins-marketplaces--interoperability)
- [Part III — Quick Decision Guide](#part-iii--quick-decision-guide)

---

# Part I — Concepts & Terminology

## 1. What an Agent Platform Is

An **agent platform** is a system that lets you define an AI **agent**, hand it **tools** and **knowledge**, start it running on a **task**, have it loop autonomously (call a model, decide, act, observe, repeat), stream you its progress as **events**, pause for your **approval** before risky actions, and persist the conversation so you can resume, fork, schedule, or audit it later.

The nine systems studied fall on a spectrum:

- **Hosted, resource-oriented REST platforms** where the platform runs the agent loop on managed infrastructure: Anthropic Managed Agents, Google Gemini Interactions API, IBM watsonx Orchestrate, Mistral Agents & Conversations.
- **Local-first / embedded runtimes** where the agent loop runs in your own process or machine via an SDK, CLI, or IDE extension: Claude Agent SDK, Codex, IBM Bob, Vibe, OpenAI Agents SDK.

A complete platform combines **seven layers** that this spec covers in order:

1. **Definition** — describing the agent (model, instructions, tools, skills, permissions).
2. **Provisioning** — preparing the sandbox/environment where it will act.
3. **Execution** — starting a session/run and driving the loop.
4. **Streaming** — receiving progress as events/deltas in real time.
5. **Control** — approvals, interrupts, hooks, mid-run updates.
6. **Knowledge** — memory stores, RAG libraries, citations.
7. **Delivery** — channels, schedules, observability, evaluation.

> **Design principle.** Most systems separate **conversation state** (the message history) from **sandbox/filesystem state** (the environment). They can be mixed and matched independently — e.g. continue a conversation but get a fresh sandbox, or reuse a sandbox but reset the conversation. Google makes this explicit with two independent state dimensions; others achieve the same implicitly.

## 2. The Core Objects

These are the vendor-neutral nouns this spec uses. Each maps to one or more product-specific names (see [Section 3](#3-cross-system-terminology-map)).

| Object | One-line definition |
|---|---|
| **Agent** | A reusable, named configuration (model + instructions + tools + skills + permissions + optional collaborators). |
| **Agent Version / Release** | An immutable snapshot of an agent configuration. |
| **Environment** | The sandbox or execution location where a run's tools act (managed container, local dir, git worktree, or cloud container). |
| **Session** | The execution container — one running agent instance performing a task; holds the conversation state machine. |
| **Turn** | One round-trip of the agent loop (model call → tool execution → feedback). |
| **Event / Item** | A typed, persisted (or streamed) unit of progress within a turn (message, tool call, tool result, thinking, handoff). |
| **Tool** | A callable capability: built-in (server-side), custom function (your code), or MCP (external server). |
| **Skill** | A filesystem-based package of domain expertise loaded on demand (progressive disclosure). |
| **Connector / MCP Server** | An external tool/data source reachable via the Model Context Protocol. |
| **Permission / Approval** | A gate deciding whether a tool call runs automatically or waits for a human. |
| **Hook** | A user-registered callback fired at a lifecycle point (pre-tool, post-tool, pre-compact, etc.). |
| **Vault / Connection / Credential** | A reusable, referenceable secret (OAuth token, API key, env var). |
| **Memory Store / Library / Knowledge Base** | A persistent knowledge resource (semantic memory or RAG document index) attached to a session or agent. |
| **Subagent / Collaborator / Teammate** | Another agent that a coordinator delegates to. |
| **Workflow / Flow** | A deterministic, graph-based multi-step automation invoked as a tool. |
| **Scheduled Task / Deployment / Routine** | A recurring (cron) or one-off trigger that starts a session autonomously. |
| **Channel** | A delivery surface (web chat, Slack, Teams, SMS, phone, voice, embedded widget). |
| **Trace / Span** | A structured observability record of a run and its sub-operations. |

## 3. Cross-System Terminology Map

Different products use different names for the same underlying concepts. The table below is the authoritative mapping used throughout this spec.

| Generic concept (this spec) | Anthropic Managed | Bob | Claude Agent SDK | Codex | Google Gemini | IBM watsonx | Mistral | OpenAI | Vibe |
|---|---|---|---|---|---|---|---|---|---|
| Agent | Agent | (runtime; Mode) | (programmatic) | Custom agent | Managed Agent | Agent | Agent | Agent | Agent (3 senses) |
| Agent version/release | Agent version | — | — | — | — | Release / version_label | — | — | — |
| Execution container | Session | Session / subtask | Session | Thread | Interaction | Thread + Run | Conversation | Run + Session | Task / Conversation |
| Single round-trip | (implicit) | (step) | Turn | Turn | (step cycle) | Run | (turn) | Run | (step) |
| Progress unit | Event | (tool result) | Message / Item | Item | Step / Event | RunEvent | Entry | (stream event) | Entry / chunk |
| Sandbox | Environment | (workspace / `--sandbox`) | `sandbox` option | Sandbox (Local/Worktree/Cloud) | Environment | (Python tool container) | (code_interpreter container) | Sandbox | (remote sandbox / code_interpreter) |
| System prompt location | `system` field | Mode `roleDefinition` | `system_prompt` / CLAUDE.md | AGENTS.md / `developer_instructions` | `system_instruction` / AGENTS.md | `instructions` field | `instructions` field | `instructions` field | Agent bundle |
| Built-in tools | agent_toolset | (read/write/command tools) | built-in tools | (command/file/web) | `code_execution`/`google_search`/`url_context` | (catalog tools) | `web_search`/`code_interpreter`/… | hosted tools | `web_search`/`code_interpreter`/… |
| Custom tool | custom tool | (via MCP) | SDK MCP server `@tool` | dynamicToolCall | `function` | `client_side` binding / `function` | `function` | `tool()` / `@function_tool` | `function` |
| MCP | MCP server / mcp_toolset | mcpServers | `mcpServers` | `mcp_servers` / `codex mcp-server` | `mcp_server` tool | `mcp` binding / toolkit | Connector | `MCPServer*` | Connector (MCP family) |
| Skill | Skill (SKILL.md) | Skill (SKILL.md) | Skill (SKILL.md) | Skill (SKILL.md / SKILL.json) | Skill (SKILL.md) | Skill binding | — | — | Skill (SKILL.md) |
| Permission policy | permission_policy (`always_allow`/`always_ask`) | auto-approve toggles | permission_mode (`default`/`acceptEdits`/`plan`/`dontAsk`/`auto`/`bypassPermissions`) | sandbox_mode + approval_policy | — | `ToolPermission` | `requires_confirmation` | `needsApproval` / guardrails | agent mode (`default`/`plan`/`accept-edits`/`auto-approve`) |
| Approval pause | `requires_action` + `user.tool_confirmation` | interactive approve | `canUseTool` callback / interruption | `requestApproval` + decision | `requires_action` + `function_result` | `requires_input` status | `confirmation_status: pending` + `tool_confirmations` | `result.interruptions` + `state.approve/reject` | `Continue`/`Always allow`/`Decline` |
| Auto-review | — | — | `auto` mode | auto_review | — | — | — | — | — |
| Hook | (webhooks for vaults) | — | Hooks (PreToolUse, PostToolUse, …) | `hook/started`/`completed` | — | `plugins` / async callbacks | — | `RunHooks`/`AgentHooks` | — |
| Credential store | Vault + Credential | API Key | (env vars / MCP headers) | (env / secrets / `.worktreeinclude`) | network `transform` (egress proxy) | Connection | Connector `headers`/`auth_data` | (sandbox `environment`) | Connector auth |
| Knowledge / RAG | Memory Store | — | CLAUDE.md / auto-memory | (AGENTS.md / web search cache) | (sandbox files / google_search) | Knowledge base / Document collection / Vector index | Library | file search / `Memory()` | Library |
| Memory (semantic) | Memory Store | — | auto-memory | — | — | `memory_enabled` / `client.memory.*` | — | `Memory()` capability | (Chat Memories) |
| Subagent | roster entry / `self` | Subagent (`spawn_subagent`) / Subtask | Subagent (`Agent` tool) | Subagent (`collabToolCall`) | — | Collaborator | (handoff target) | (handoff target / agents-as-tools) | (handoff target) |
| Team / direct messaging | (threads) | — | Agent team (`SendMessage`) | (CSV fan-out workers) | — | (collaborators + flows) | — | — | — |
| Handoff | (delegation via threads) | — | (Agent tool delegation) | (collabToolCall) | — | (collaborator delegation) | Handoff (`handoffs[]`) | Handoff / agents-as-tools | Handoff (`handoffs[]`) |
| Workflow / flow | — | Workflow tool | Dynamic workflow (JS) | (Goal mode) | — | Flow (`@flow`) | — | (chained voice pipeline) | Workflow (Studio) |
| Scheduled task | Deployment (cron) | (external CI) | Routine / scheduled task | Scheduled task / Automations | — | `is_schedulable` (UI-scheduled) | — | — | Scheduled Task |
| Trace / observability | Span events | Bobalytics | OpenTelemetry / `ResultMessage.usage` | OpenTelemetry (`[otel]`) | `interaction.steps` + `usage` | `trace_id` / Langfuse / WXG | (`usage`) | Traces dashboard | (tool-call transparency) |
| Channel | — | (IDE/CLI only) | Surfaces (CLI/IDE/web/Slack/Chrome) | (GitHub/Linear/Slack triggers) | — | Channels (Slack/Teams/Twilio/phone/web) | — | Realtime API / ChatKit | (web/CLI/VS Code/mobile) |
| Voice | — | — | — | — | (TTS/audio models) | Voice configuration / RealtimeAgentSettings | — | RealtimeAgent / VoicePipeline | — |
| Plugin / marketplace | — | — | Plugin (bundled skills/agents/hooks/MCP) | Plugin / App / marketplace | — | Catalog | — | — | — |

---

# Part II — The Exhaustive Processing Pipeline

> **How to read each stage.** Every stage opens with a **purpose**, then lists the **capabilities** found across systems (the union), then the **alternative approaches** (where systems diverge), then a **unified API specification** covering the superset. Origin systems are cited in parentheses like `(Anthropic)`. Field names are vendor-neutral; see [Section 3](#3-cross-system-terminology-map) for native names.

---

## Stage 0 — Platform Onboarding & Authentication

**Purpose.** Establish identity and obtain credentials before you can define or run anything.

### Capabilities (union)
- **API-key auth** — bearer or `x-api-key` header on every call. (All REST systems.)
- **OAuth / login flow** — ChatGPT login (Codex), Mistral account OAuth (Vibe), IAM/MCSP/CPD bearer token from a login endpoint (IBM watsonx).
- **Scoped / ephemeral keys** — `CODEX_API_KEY` scoped to one invocation (Codex); Workspace Agent access tokens scoped to the trigger API (Codex); API keys scoped to Connector access only (Mistral).
- **Admin vs user key roles** — General vs Inference key types; admins manage keys for all users in an instance (Bob).
- **Beta / version headers** — `anthropic-version`, `managed-agents-2026-04-01`, `agent-memory-2026-07-22`, `skills-2025-10-02` (Anthropic); `Api-Revision: 2026-05-20` (Google); `Api-Revision` header gating.
- **Provider routing / gateways** — Anthropic reachable via Bedrock, AWS, GCP Agent Platform, Microsoft Foundry, or an LLM gateway (Claude); OpenAI-compatible model gateway passthrough (IBM watsonx).
- **WebSocket auth** — capability-token or signed-bearer-token with issuer/audience/clock-skew (Codex app-server).
- **License acceptance** — `--accept-license` required before first non-interactive run (Bob).

### Alternative approaches
- **Static API key** (most systems) vs **login-then-bearer-token** (IBM watsonx) vs **interactive OAuth** (Codex ChatGPT, Vibe).
- **Single provider** (Google, Mistral) vs **multi-provider gateway** (Claude via Bedrock/Foundry, IBM watsonx model gateway, OpenAI provider/adapter surface).

### Unified API specification

```
POST /v1/auth/login                 # Obtain a bearer token (login-flow systems)
  body: { identity, credential, scheme: "iam"|"mcsp"|"cpd" }
  → { access_token, expires_at }

# All subsequent calls carry one of:
#   Authorization: Bearer <token>
#   x-api-key: <key>
#   x-goog-api-key: <key>
# Plus versioning headers:
#   X-Platform-Version: <date>
#   X-Beta-Features: managed-agents, agent-memory, skills  (repeatable)

POST /v1/api_keys                   # Create a scoped key
  body: { type: "general"|"inference"|"connector_scoped"|"ephemeral",
          scope: ["agents","connectors","workspace_agents.trigger"], ttl_seconds? }
  → { key_id, secret }   # secret shown once

DELETE /v1/api_keys/{id}            # Revoke
GET /v1/api_keys                    # List (admin: all in instance; user: own)
```

---

## Stage 1 — Agent Definition & Configuration

**Purpose.** Describe the agent once, reuse it across many runs.

### Capabilities (union)
- **Reusable agent object** with a stable ID, created via API or file. (Anthropic, Google, IBM watsonx, Mistral, OpenAI; file-based: Claude `.claude/agents/`, Codex `~/.codex/agents/*.toml`, Bob `custom_modes.yaml`, Vibe `.vibe/`.)
- **Versioning / releases** — every save produces a new version; releases are immutable snapshots deployable per environment. (Anthropic `version` + optimistic concurrency; IBM watsonx `version_label`/`semantic_version`.)
- **Optimistic concurrency** — `version` field required on update; mismatch → 409 (Anthropic); `precondition.content_sha256` for memories (Anthropic).
- **Lifecycle ops** — create, retrieve, update, list, list versions, archive (read-only, one-way), delete. (Anthropic, IBM watsonx.)
- **System prompt location** — `system`/`instructions` field; or file-based `AGENTS.md`/`CLAUDE.md` auto-loaded; or per-run override. (All.)
- **AGENTS.md layered discovery** — global → project root → cwd, concatenated root-down, later overrides earlier; `AGENTS.override.md` precedence; fallback filenames; max-bytes cap. (Codex, Google; analogues: Claude CLAUDE.md, Bob AGENTS.md, Vibe `.vibe/`.)
- **Model selection per agent** — string ID or `{id, speed}` object (Anthropic); `provider/developer/model_id` (IBM watsonx); alias `opus`/`sonnet`/`haiku`/`fable`/`inherit` (Claude).
- **Default toolset** — a flag enabling all built-in tools (`agent_toolset_20260401`, Anthropic); or explicit `tools[]`; or mode-gated groups (Bob).
- **Structured output** — JSON Schema enforced on responses (OpenAI `outputType`, IBM watsonx `structured_output`, Anthropic `output_format`, Codex `--output-schema`).
- **Metadata / tags** — arbitrary key-value merged at key level (Anthropic `metadata`); `tags`, `hidden` (IBM watsonx).
- **Description as routing metadata** — the `description` drives supervisor/collaborator routing, not just human text (IBM watsonx, OpenAI `handoffDescription`, Codex `description`).
- **Agent styles / reasoning modes** — `default`/`react`/`planner`/`custom`/`react_intrinsic`/`code_act`/`experimental_customer_care` (IBM watsonx); built-in agent personas `default`/`plan`/`accept-edits`/`auto-approve` (Vibe Code); `default`/`worker`/`explorer` (Codex); `Explore`/`Plan`/`general-purpose` (Claude).
- **Context variables** — platform-provided runtime values (`wxo_email_id`, `wxo_user_name`) referenced in instructions as `{var}` (IBM watsonx).
- **Agent restrictions / editability** — `editable`/`non_editable`/`custom` controlling collaborator editability (IBM watsonx).
- **Timeout** — per-agent `timeout_seconds` min 120 max 900 (IBM watsonx).
- **Bundled / catalog agents** — derive from a base/catalog agent (`bundled_agent_id`, `base_agent`, `create-from-template`). (IBM watsonx, Google, Codex.)

### Alternative approaches
- **API-defined agents** (Anthropic, Google, IBM watsonx, Mistral, OpenAI) vs **file-defined agents** (Claude `.claude/agents/*.md`, Codex `*.toml`, Bob `custom_modes.yaml`, Vibe `.vibe/`) vs **code-defined agents** (OpenAI `new Agent({...})`, Claude `AgentDefinition`).
- **Versioned** (Anthropic, IBM watsonx) vs **mutable in place** (Mistral, OpenAI, Google).
- **Single system prompt** vs **layered AGENTS.md concatenation** (Codex/Google) vs **mode-based roleDefinition** (Bob).

### Unified API specification

```
POST /v1/agents
  body: {
    name: string,                       # required
    description: string,                # required; drives routing
    model: string | { id, speed? },     # required (unless inherited)
    system?: string,                    # system prompt; null to clear
    tools?: [ToolConfig],               # built-in + MCP + custom
    mcp_servers?: [MCPServerRef],       # ≤20
    skills?: [SkillRef],                # ≤20 per session across agents
    multiagent?: { type: "coordinator", agents: [...] },
    metadata?: object,
    tags?: [string],
    structured_output?: JSONSchema,
    style?: "default"|"react"|"planner"|"code_act"|...,
    collaborators?: [agent_name],
    knowledge_base?: [kb_name],
    timeout_seconds?: int,              # 120–900
    restrictions?: "editable"|"non_editable"|"custom",
    hidden?: bool,
    is_schedulable?: bool,
    context_access_enabled?: bool,
    context_variables?: [var_name],
    bundled_agent_id?: string           # derive from catalog/base
  }
  → { id, type, version, created_at, updated_at }

GET  /v1/agents/{id}                     # retrieve (specific version via ?version=n)
GET  /v1/agents/{id}/versions            # list versions (paginated)
POST /v1/agents/{id}                     # update (version required; new version on change)
     body: { ..., version: int }         # optimistic concurrency
POST /v1/agents/{id}/archive             # one-way read-only
DELETE /v1/agents/{id}
GET  /v1/agents                         # list (filters: agent_id, order, limit, cursor, include_hidden)

# File-based alternative (Codex/Claude/Bob/Vibe):
#   ~/.<platform>/agents/<name>.{md,toml,yaml}
#   .<platform>/agents/<name>.{md,toml}      # project-scoped
# Fields: name, description, developer_instructions/prompt, model,
#         model_reasoning_effort, sandbox_mode, mcp_servers, skills, tools,
#         nickname_candidates, allowedSubagents, groups, fileRegex, permissionMode

# AGENTS.md layered discovery (Codex/Google):
#   global: ~/.codex/AGENTS[.override].md
#   project: <git-root>/…/<cwd>/AGENTS[.override].md  (one per dir)
#   merge: concatenated root-down; later overrides earlier
#   config: project_doc_fallback_filenames[], project_doc_max_bytes (32 KiB)

# Releases / deployment (IBM watsonx):
POST /v1/agents/{id}/releases
  body: { version_label: int, semantic_version?, version_name?, version_description? }
POST /v1/agents/{id}/releases/{ver}/environment/{env_id}   # deploy
POST /v1/agents/{id}/releases/{ver}/undeploy
```

---

## Stage 2 — Models, Providers & Reasoning

**Purpose.** Choose and tune the model that powers the agent's reasoning.

### Capabilities (union)
- **Explicit model per agent** — highest priority. (All.)
- **Run-level / turn-level model override** — `RunConfig(model)` (OpenAI); `turn/start` `model` override (Codex); per-session `agent_with_overrides.model` (Anthropic); `RunOrchestrate.llm_params` (IBM watsonx); `conversations.start(model)` (Mistral).
- **Process-wide fallback** — `OPENAI_DEFAULT_MODEL` (OpenAI); `fallback_model` (Claude).
- **Reasoning effort / thinking config** — `model_reasoning_effort`: `minimal`/`none`/`low`/`medium`/`high`/`max`/`xhigh`/`ultra` (Codex); `effort`: `low`/`medium`/`high`/`xhigh`/`max` (Claude); `thinking` `ThinkingConfig` (Claude); `generation_config.thinking_summaries`/`thinking_level` (Google); `modelSettings` reasoning effort (OpenAI); `speed: "fast"` (Anthropic).
- **Thinking events** — `agent.thinking` events (Anthropic); `reasoning` items with `summary` + `content` (Codex); `thought` steps with `thought_summary` + `thought_signature` deltas (Google); `thinking chunk` (Vibe Workflows); `hide_reasoning` toggle (IBM watsonx).
- **Model rerouting / safety buffering** — `model/rerouted` `{fromModel, toModel, reason}` (Codex); `model/safetyBuffering/updated` with `fasterModel` (Codex).
- **Model catalog / list** — `model/list` with `supportedReasoningEfforts[]`, `inputModalities`, `supportsPersonality`, `isDefault`, `upgrade` (Codex); `/v1/models/list` + `/embeddings` (IBM watsonx).
- **Model policies / governance** — `/v1/model_policy` governance controls over which models may be used (IBM watsonx, public preview).
- **Provider capability bounds** — `modelProvider/capabilities/read` (Codex).
- **Generation params** — `temperature`, `top_p`, `max_tokens`, `n`, `seed`, `stop`, `frequency_penalty`, `presence_penalty`, `logit_bias` (IBM watsonx, Mistral `completion_args`).
- **Per-turn personality** — `personality` override on `turn/start` (Codex).
- **Multi-provider gateway** — OpenAI-compatible passthrough `/gateway/model/chat/completions`, `/embeddings` (IBM watsonx); Anthropic via Bedrock/Foundry/GCP (Claude).
- **Verification** — `model/verification` additional account verification (Codex).

### Alternative approaches
- **Single provider** (Google, Mistral, Anthropic) vs **gateway/multi-provider** (IBM watsonx, Claude via Bedrock, OpenAI adapter surface).
- **Reasoning effort as a discrete enum** (Codex, Claude) vs **thinking level + summaries config** (Google) vs **`speed` flag** (Anthropic) vs **no reasoning knob** (Mistral, Bob).
- **Model rerouting by platform** (Codex safety buffering) vs **explicit fallback model** (Claude) vs **layered resolution** (OpenAI per-agent → run → env).

### Unified API specification

```
GET /v1/models
  query: limit, include_hidden?, include_embeddings?
  → [{ id, model, displayName, hidden, defaultReasoningEffort,
       supportedReasoningEfforts: [{reasoningEffort, description}],
       inputModalities: ["text","image"], supportsPersonality,
       isDefault, upgrade?, upgradeInfo? }]

# Agent-level (see Stage 1):  model, model_reasoning_effort, model_settings
# Run-level override:
POST /v1/sessions/{id}/turns
  body: { model?, model_reasoning_effort?, personality?,
          generation_config?: { thinking_summaries, thinking_level, temperature, top_p, max_tokens, ... } }

# Fallbacks:
#   fallback_model (Claude)
#   OPENAI_DEFAULT_MODEL (OpenAI process-wide)

# Policy / governance:
POST /v1/model_policy
  body: { allowed_models: [...], constraints: {...} }

# Gateway passthrough (IBM watsonx-style):
POST /v1/gateway/model/chat/completions    # OpenAI-compatible, no agent
POST /v1/gateway/model/embeddings
```

---

## Stage 3 — Environments & Sandboxes

**Purpose.** Provision an isolated place where the agent's tools (shell, file ops, code) execute, with controlled resources, network, files, and secrets.

### Capabilities (union)
- **Managed cloud sandbox** — fresh Linux container per session from an environment config. (Anthropic `cloud`, Google `remote`, Codex `Cloud`, OpenAI `DockerSandboxClient`.)
- **Self-hosted / local sandbox** — agent runs on your machine/process; OS-level sandbox (Seatbelt/bubblewrap/seccomp/Windows sandbox). (Anthropic `self_hosted`, Codex `Local` + OS sandbox, Bob `--sandbox`, Claude `sandbox` option, Vibe CLI.)
- **Git worktree isolation** — each background session gets its own worktree; commits/pushes a branch; opens a draft PR. (Claude `.claude/worktrees/`, Codex `Worktree`.)
- **Pre-installed packages** — cached across sessions sharing an environment; multiple managers in alphabetical order (apt, cargo, gem, go, npm, pip) with optional version pins. (Anthropic.) Pre-installed runtimes (Google: Python 3.12, Node 22, common UNIX tools).
- **Declarative sources / mounts** — `repository` (git clone), `gcs` (Cloud Storage), `inline` (text), local dir, GitRepo, S3/GCS/R2/AzureBlob/Box mounts. (Google `sources`; OpenAI `Manifest` entries; Codex `.worktreeinclude`.)
- **Network policy** — `unrestricted` | `limited` with `allowed_hosts` (Anthropic); `network.allowlist` with `transform` header injection via egress proxy (Google); `network.domains` allow/deny + proxy + SOCKS5 + unix sockets (Codex Permission Profiles); `network: "disabled"` (Google); `network_proxy` feature (Codex).
- **Credentials injection** — vault env-var credentials substituted at egress (Anthropic); egress-proxy header `transform` never exposed inside sandbox (Google); cloud secrets decrypted only for setup scripts, removed before agent phase (Codex).
- **Lifecycle states** — Created → Active → Idle (auto-snapshot after 15 min) → Offline (retained 7 days) → Deleted (Google); archive/delete (Anthropic).
- **Resource limits** — CPU/memory (Google 4 cores/16 GB); agent `timeout_seconds` (IBM watsonx); `max_turns`/`max_budget_usd` (Claude).
- **Container cache** — up to 12h; clones default branch, runs setup, caches state; invalidated on script/env/secret change. (Codex Cloud.)
- **Setup scripts** — run on new worktree creation; platform-specific overrides; automatic (`npm`/`pip`/…) or manual custom script. (Codex, OpenAI Manifest `environment`.)
- **Filesystem download / snapshot** — `GET /files/environment-{id}:download` returns a tar archive of the full filesystem. (Google.) `snapshot()` saves workspace to seed a fresh sandbox (OpenAI). Checkpointing — git snapshot + conversation + tool-call record for rollback. (Bob, Claude `enable_file_checkpointing` + `rewind_files`.)
- **Permission Profiles** — named least-privilege policy combining filesystem rules (`read`/`write`/`deny`) + network rules + inheritance via `extends`. (Codex, beta.) Protected paths (`.git`, `.agents`, `.codex`).
- **Sandbox test command** — `codex sandbox macos|linux|windows` (Codex).
- **Background terminals / process spawn** — `thread/backgroundTerminals/*`, `process/spawn` outside the sandbox. (Codex.)
- **Antigravity IDE harness reuse** — managed agent uses the same harness as the IDE. (Google.)

### Alternative approaches
- **Managed container per session** (Anthropic, Google, Codex Cloud, OpenAI Docker) vs **OS-level sandbox on your machine** (Codex Local, Bob `--sandbox`, Claude) vs **no sandbox / your process** (Mistral code_interpreter is opaque, OpenAI local SDK).
- **Image + package config** (Anthropic, Google, OpenAI Manifest) vs **git-source environment** (Google `repository` source, Codex Cloud checks out repo) vs **`.worktreeinclude` file copy** (Codex).
- **Network allowlist with egress-proxy credential injection** (Google) vs **vault env-var injection at egress** (Anthropic) vs **TOML network domain rules + proxy** (Codex) vs **no network config** (Mistral, Bob).
- **Ephemeral mounts** (OpenAI — snapshots skip remote storage) vs **persistent sandbox reuse by ID** (Google `environment_id`, Anthropic sessions share environment).
- **Permission Profiles (least-privilege filesystem+network)** (Codex) vs **sandbox_mode enum** (Codex simpler) — the two do NOT compose.

### Unified API specification

```
POST /v1/environments
  body: {
    name: string,
    config: {
      type: "cloud"|"self_hosted"|"local"|"worktree",
      packages?: {                          # pre-install, cached per env
        apt?: ["ffmpeg"], cargo?: ["ripgrep@14.0.0"],
        gem?, go?, npm?, pip?
      },
      sources?: [                           # declarative mounts
        { type: "repository", source: "<git-url>", target: "/abs/path" },  # ≤500MB
        { type: "gcs", source: "gs://...", target: "/abs/path" },           # ≤2GB
        { type: "inline", content: "...", target: "/abs/path/.agents/AGENTS.md" },  # ≤1MB/file
        { type: "local_dir", path: "..." },
        { type: "git_repo", url, ref?, sha? },
        { type: "mount", provider: "s3"|"gcs"|"r2"|"azure_blob"|"box", ... }
      ],
      network?: {
        mode: "unrestricted"|"limited"|"disabled",
        allowed_hosts?: ["*.example.com"],          # no scheme/port/path
        allow_mcp_servers?: bool,                   # default false
        allow_package_managers?: bool,              # default false
        domains?: { "api.example.com": "allow", "*.evil.com": "deny" },
        unix_sockets?: { "/path": "allow"|"deny" },
        proxy_url?: "http://127.0.0.1:3128",
        enable_socks5?: bool,
        allow_local_binding?: bool
      },
      resources?: { cpu_cores: 4, memory_gb: 16 },
      setup_scripts?: [{ platform?: "...", script: "..." }],
      env?: { VAR: "value" },
      users?: [...], groups?: [...]
    }
  }
  → { id, ... }

GET    /v1/environments
GET    /v1/environments/{id}
POST   /v1/environments/{id}/archive
DELETE /v1/environments/{id}            # only if no sessions reference it

# Lifecycle (Google-style):
#   Created → Active → Idle (15min auto-snapshot) → Offline (7 days) → Deleted

# Permission Profiles (Codex-style):
[permissions.<name>]
extends = ":read-only" | ":workspace" | "<other-profile>"
workspace_roots = { "/path" = true }
[permissions.<name>.filesystem]
"/abs/path" = "read"|"write"|"deny"
glob_scan_max_depth = 3
[permissions.<name>.network]
enabled = false
domains = { "api.openai.com" = "allow" }
unix_sockets = { "/tmp/sock" = "allow" }

# Sandbox modes (Codex-style simpler alternative):
sandbox_mode = "read-only"|"workspace-write"|"danger-full-access"

# Filesystem snapshot / checkpoint:
GET /v1/files/environment-{env_id}:download    # → tar archive (Google)
POST /v1/sessions/{id}/snapshot                # save workspace (OpenAI)
POST /v1/sessions/{id}/rewind_files            # restore to user_message_id (Claude/Bob checkpoint)

# Credentials injection at egress (Google transform / Anthropic vault):
network.allowlist[].transform = { "Authorization": "Bearer <token>" }   # header injected by egress proxy
# OR vault environment_variable credential (Anthropic) substituted at egress
```

---

## Stage 4 — Sessions, Threads, Runs & Interactions

**Purpose.** Start the execution container that holds the conversation state machine, references the agent and environment, and can be resumed/forked/continued.

### Capabilities (union)
- **Creation** — `POST /v1/sessions` (Anthropic); `interactions.create` (Google); `POST /v1/threads` + `POST /v1/orchestrate/runs` (IBM watsonx); `conversations.start` (Mistral); `Runner.run()` (OpenAI); `query()` (Claude); `thread/start` + `turn/start` (Codex); `bob -p`/`bob -i` (Bob).
- **Two-step lifecycle** — create session (provisions sandbox) → send first message (starts work). (Anthropic.)
- **Status state machine** — `idle`/`running`/`rescheduling`/`terminated` (Anthropic); `in_progress`/`requires_action`/`completed`/`failed`/`cancelled` (Google); `pending`/`running`/`completed`/`async_wait`/`async_completed`/`failed`/`cancelled`/`requires_input`/`expired` (IBM watsonx); `success`/`error_max_turns`/`error_max_budget_usd`/`error_during_execution`/`error_max_structured_output_retries` (Claude Result subtypes); `turn/started` → `completed`/`interrupted`/`failed` (Codex).
- **Agent reference forms** — agent ID string (latest version); pinned `{id, version}`; `agent_with_overrides` (per-session model/system/tools/mcp/skills without versioning). (Anthropic.) `agent` vs `model` mutually exclusive (Google, Mistral). `agent_id` + optional `environment_id` + optional `version` pin (IBM watsonx).
- **Override rules** — omit → inherit; `null`/empty → clear (exceptions: `model` never clearable); value → full replacement. (Anthropic.)
- **Continue / resume / fork** — `continue_conversation` (find most recent in cwd, Claude); `resume: session_id` (specific, Claude/Codex/Anthropic); `fork_session` (new session, copied history, original unchanged, Claude/Codex `thread/fork` with `lastTurnId`); `previous_interaction_id` (Google server-side state); `previousResponseId` / `conversationId` (OpenAI); `append` with new conversation ID (Mistral, append-only immutable).
- **Multi-turn** — send new messages to an existing session; multiple `turn/start` append to a thread. (All.)
- **Mid-turn steering** — `turn/steer` appends user input to an active in-flight turn. (Codex.) `user.interrupt` + `user.message` redirects mid-execution (Anthropic).
- **Session storage / persistence** — JSONL on disk `~/.<platform>/projects/<encoded-cwd>/<session-id>.jsonl`; `SessionStore` adapter mirroring to S3/Redis/custom. (Claude.) SQLite rollouts `session-*.jsonl` (Codex).
- **Session utility functions** — `list_sessions`, `get_session_messages`, `get_session_info`, `rename_session`, `tag_session`. (Claude.) `thread/list` with filters (modelProviders, sourceKinds, archived, cwd, searchTerm, parentThreadId). (Codex.)
- **Thread operations** — name/set, goal/set, metadata/update, archive/unarchive/delete, unsubscribe, compact/start, shellCommand (outside sandbox), inject_items, rollback. (Codex.)
- **Two independent state dimensions** — conversation history (`previous_interaction_id`) and sandbox/files (`environment_id`) decoupled and mixable. (Google; implicit in others.)
- **Background / async mode** — `background: true` returns interaction ID immediately; poll/stream/cancel. (Google.) Background agents under a supervisor process (Claude `claude --bg`, Codex).
- **Goals** — long-running task target with progress tracking, pause/resume/edit/clear; keeps shared context across turns. (Codex `/goal`.)
- **Access scope** — public Conversations API can read/modify only conversations owned by the API key creator. (Mistral.)
- **Data retention** — Paid Tier 55 days (configurable 7/14/28/55), Free Tier 1 day; `store=false` opts out. (Google.) `store=False` (Mistral).
- **List/retrieve** — `GET /v1/sessions/{id}` includes status; `GET /v1/sessions` paginated with `limit`, `agent_id` filter, `order`, cursor pagination. (Anthropic.)

### Alternative approaches
- **Single unified execution container** (Anthropic Session, Google Interaction, Mistral Conversation, Bob session) vs **split Thread + Run** (IBM watsonx: Thread = history, Run = execution) vs **Run + Session + Conversation + Response** four strategies (OpenAI).
- **Server-managed conversation state** (Google `previous_interaction_id`, OpenAI `conversationId`/`previousResponseId`, Mistral append-only) vs **client-managed history** (Claude `result.history`, OpenAI `session`).
- **Fork as first-class** (Claude `fork_session`, Codex `thread/fork` with `lastTurnId`) vs **append-only new IDs** (Mistral) vs **resume only** (Anthropic, Google).
- **Background as a mode on the same endpoint** (Google `background: true`) vs **separate background surface** (Claude `claude --bg` + supervisor, Codex scheduled).

### Unified API specification

```
POST /v1/sessions
  body: {
    agent: string | { type: "agent", id, version? } | { type: "agent_with_overrides", id, model?, system?, tools?, mcp_servers?, skills? },
    environment_id: string,               # required
    vault_ids?: [string],
    resources?: [ { type: "memory_store", memory_store_id, access: "read_write"|"read_only", instructions? } ],
    title?: string,
    previous_session_id?: string,         # continue conversation history
    background?: bool,                    # async mode
    store?: bool,                         # default true; false opts out of persistence
    context?: object,                     # context variables (IBM watsonx)
    context_variables?: { wxo_email_id, wxo_user_name, ... },
    llm_params?: { temperature, top_p, max_tokens, ... },
    guardrails?: object
  }
  → { id, status, agent: {resolved snapshot}, environment_id, ... }

# Status state machine (union):
#   idle → running → idle | terminated | rescheduling
#   pending → running → completed | async_wait | async_completed | failed | cancelled | requires_input | expired
#   in_progress → requires_action → completed | failed | cancelled

# Continue / resume / fork:
POST /v1/sessions/{id}/events           # send user.message to continue (Anthropic)
POST /v1/sessions/{id}/resume           # resume by ID (Claude/Codex)
POST /v1/sessions/{id}/fork             # body: { last_turn_id? } → new session with copied history
POST /v1/interactions                   # with previous_interaction_id + environment_id (Google)

# Steering / interrupt:
POST /v1/sessions/{id}/steer            # append user input to active turn (Codex turn/steer)
POST /v1/sessions/{id}/interrupt        # cancel mid-execution (Anthropic user.interrupt)

# Goals:
POST /v1/sessions/{id}/goal             # set long-running target (Codex)
GET  /v1/sessions/{id}/goal
POST /v1/sessions/{id}/goal/clear

# Listing / management:
GET  /v1/sessions                       # filters: agent_id, order, limit, cursor, archived, model_providers, source_kinds, cwd, search_term, parent_thread_id
GET  /v1/sessions/{id}
POST /v1/sessions/{id}/archive
DELETE /v1/sessions/{id}
POST /v1/sessions/{id}/name             # set name
POST /v1/sessions/{id}/metadata         # patch metadata (gitInfo, tag, custom_title)

# Thread-level (multi-agent):
GET  /v1/sessions/{id}/threads
GET  /v1/sessions/{id}/threads/{tid}/stream
GET  /v1/sessions/{id}/threads/{tid}/events
POST /v1/sessions/{id}/threads/{tid}/archive

# Background:
GET  /v1/interactions/{id}              # poll status (Google)
GET  /v1/interactions/{id}?stream=true&last_event_id=...   # resumable stream
POST /v1/interactions/{id}/cancel
```

---

## Stage 5 — The Agent Loop, Turns, Streaming & Events

**Purpose.** Drive the autonomous loop (model → decide → tool → observe → repeat), stream progress in real time, and persist a replayable record.

### Capabilities (union)
- **Server-side managed loop** — platform runs: provision sandbox → stream events → Claude picks tools → executes → pauses on `always_ask` → resumes on confirmation → `status_idle` when done. (Anthropic, Google, Mistral, IBM watsonx.)
- **In-process loop** — SDK runs the loop in your process via a bundled native binary. (Claude, Codex, OpenAI, Bob.)
- **Loop lifecycle** — receive prompt → evaluate & respond → execute tools (read-only concurrent, state-modifying sequential) → repeat until no tool calls → return result. (Claude.)
- **Streaming via SSE** — `GET /v1/sessions/{id}/events/stream` (Anthropic); `stream: true` (Google, Mistral); `POST /runs?stream=true` (IBM watsonx); `async for message in query(...)` (Claude); `codex exec --json` JSONL (Codex).
- **Symmetric step model** — all content flows through `step.start` → `step.delta`(s) → `step.stop`. (Google.) Analogous: `item/started` → `item/completed` (Codex); `event_start` → `event_delta` (Anthropic).
- **Event type catalog** (union, `{domain}.{action}` convention):
  - **User events (you send):** `user.message`, `user.interrupt`, `user.custom_tool_result`, `user.tool_confirmation`, `user.define_outcome`, `user.tool_result` (self-hosted). (Anthropic.)
  - **Agent events (you receive):** `agent.message`, `agent.thinking`, `agent.tool_use`, `agent.tool_result`, `agent.mcp_tool_use`, `agent.mcp_tool_result`, `agent.custom_tool_use`, `agent.thread_context_compacted`, `agent.thread_message_received`, `agent.thread_message_sent`. (Anthropic.) `message.output`, `tool.execution`, `function.call`, `agent.handoff`. (Mistral.) `assistantMessage`, `userMessage`, `systemMessage`, `streamEvent`, `resultMessage`. (Claude.) `userMessage`, `agentMessage`, `plan`, `reasoning`, `commandExecution`, `fileChange`, `mcpToolCall`, `dynamicToolCall`, `collabToolCall`, `webSearch`, `imageView`, `contextCompaction`, `enteredReviewMode`/`exitedReviewMode`. (Codex.) `model_output`, `thought`, `function_call` + server-side tool steps. (Google.) `RunEvent` envelope `{id, event, data}`. (IBM watsonx.)
  - **Session events:** `session.status_running`, `session.status_idle`, `session.status_rescheduled`, `session.status_terminated`, `session.deleted`, `session.updated`, `session.error`, `session.thread_created`, `session.thread_status_*`. (Anthropic.) `turn/started`, `turn/completed`, `turn/diff/updated`, `turn/plan/updated`, `thread/tokenUsage/updated`. (Codex.) `interaction.created`, `interaction.status_update`, `interaction.completed`, `error`, `done`. (Google.)
  - **Span events (observability):** `span.model_request_start`/`_end` (with `model_usage`), `span.outcome_evaluation_start`/`_ongoing`/`_end`. (Anthropic.) `hook/started`/`hook/completed`. (Codex.) `model/rerouted`, `model/safetyBuffering/updated`, `model/verification`. (Codex.)
  - **System events (you send):** `system.message` (update system prompt between turns, Opus 4.8 only). (Anthropic.)
- **Deltas (stream-only, opt-in, never persisted)** — `event_start` + `event_delta` with `delta.type: "content_delta"` and `index`. (Anthropic.) `step.delta` types: `text`, `image`, `audio`, `thought_summary`, `thought_signature`, `arguments_delta`, `google_search_call`, `google_search_result`. (Google.) `item/agentMessage/delta`, `item/plan/delta`, `item/reasoning/summaryTextDelta`/`textDelta`, `item/commandExecution/outputDelta`. (Codex.) `StreamEvent` raw API streaming. (Claude.) `TextChunk`, `ToolReferenceChunk`, `ToolFileChunk`, `ReferenceChunk`. (Mistral/Vibe.)
- **Item lifecycle** — `item/started` (full item, `item.id` = delta `itemId`) → `item/completed` (authoritative final state). (Codex.) Reconcile per model request: `span.model_request_start` → `event_start` → `event_delta`s → buffered `agent.message` → `span.model_request_end`. (Anthropic.)
- **Stop reasons** — `requires_action` with blocking `event_ids`; `end_turn` for interrupted blocked threads. (Anthropic.) Result subtypes. (Claude.)
- **Errors** — `session.error` with typed `error` object + `retry_status`: `mcp_connection_failed_error`, `mcp_authentication_failed_error` (Anthropic); `environment_archived_error`, `agent_archived_error`, `session_rate_limited_error` (Anthropic deployments); `codexErrorInfo` variants `ContextWindowExceeded`, `UsageLimitExceeded`, `HttpConnectionFailed`, `BadRequest`, `Unauthorized`, `SandboxError`, `InternalServerError`, `Other` (Codex); `error` event with `message` + `code` e.g. `gateway_timeout` (Google).
- **Sending / listing events** — `POST /v1/sessions/{id}/events` with `{"events":[...]}`; multiple per request; `GET /v1/sessions/{id}/events` paginated with `types[]` filter. (Anthropic.) `thread/items/list`, `thread/turns/list` with `itemsView` omit/summarize/fully-load. (Codex.)
- **Compaction / context window** — `agent.thread_context_compacted` events (Anthropic); `SystemMessage(subtype="compact_boundary")` + `PreCompact` hook + `/compact` (Claude); `thread/compact/start` + `contextCompaction` item (Codex); native compaction at ~135k tokens (Google); `compaction_settings.context_compaction_threshold`/`compaction_sliding_window`/`large_message_threshold` (IBM watsonx); agent can override compaction prompt (Vibe Code).
- **Outcome definition & evaluation** — `user.define_outcome` + `span.outcome_evaluation_*` lifecycle. (Anthropic.)
- **Plan / todos streaming** — `turn/plan/updated` with steps `{step, status: pending/inProgress/completed}` (Codex); `update_todo_list` tool (Bob); `TaskCreate`/`TaskUpdate` tools (Claude); Todos panel (Vibe).
- **Token usage** — `thread/tokenUsage/updated` (Codex); `interaction.usage` with `total_tokens`, modality breakdowns, `total_cached_tokens`, `total_thought_tokens`, `total_tool_use_tokens` (Google); `ResultMessage.usage` + `total_cost_usd` + `num_turns` (Claude); `usage.connector_tokens` (Mistral).
- **Resumable streaming** — `last_event_id` cursor to resume after disconnect; each delta carries unique `event_id`. (Google.)
- **Handling unknown events** — new event/delta types may be added; handle gracefully (log and skip). (Google versioning policy.)

### Alternative approaches
- **Server-managed loop** (Anthropic, Google, Mistral, IBM watsonx) vs **in-process loop** (Claude, Codex, OpenAI, Bob).
- **Event-type naming** — `{domain}.{action}` dot convention (Anthropic, Codex) vs `step.*` (Google) vs typed `Entry` objects (Mistral) vs `Message` types (Claude) vs `RunEvent` envelope (IBM watsonx).
- **Deltas as opt-in preview** (Anthropic `event_deltas[]` query param) vs **symmetric step model always-streamed** (Google) vs **JSONL item stream** (Codex `--json`).
- **Compaction** — automatic server-side (Anthropic, Google ~135k) vs configurable threshold + sliding window (IBM watsonx) vs hook-customizable (Claude `PreCompact`) vs agent-overridable prompt (Vibe).
- **Plans/todos** — streamed `turn/plan/updated` events (Codex) vs tool-based `update_todo_list`/`TaskCreate` (Bob, Claude) vs UI Todos panel (Vibe).

### Unified API specification

```
# Streaming:
GET /v1/sessions/{id}/events/stream         # SSE (Anthropic)
  query: event_deltas[]=agent.message&event_deltas[]=agent.thinking   # opt-in deltas

# Event send:
POST /v1/sessions/{id}/events
  body: { events: [ { type: "user.message", content: [{type:"text", text:"..."}] }, ... ] }

# Event listing:
GET /v1/sessions/{id}/events
  query: types[]=agent.tool_use&types[]=agent.tool_result&limit&cursor

# Unified event schema:
{
  id: string,
  type: string,               # "agent.message" | "session.status_idle" | "step.start" | ...
  processed_at: timestamp?,   # null = queued
  data: {
    # For agent.message:
    content: [ { type: "text"|"image"|"audio"|"tool_reference"|"tool_file"|"reference", ... } ],
    # For agent.tool_use:
    tool: string, input: object,
    # For step.delta:
    delta: { type: "text"|"image"|"thought_summary"|"arguments_delta"|..., ... },
    index?: int,
    # For session.status_idle:
    stop_reason: { type: "requires_action"|"end_turn", event_ids?: [...] },
    # For errors:
    error: { type: string, message: string, retry_status: string, mcp_server_name? },
    # For spans:
    model_usage?: { input_tokens, output_tokens, ... }
  }
}

# Deltas (stream-only):
{ type: "event_start", event_type: "agent.message", event_id: "..." }
{ type: "event_delta", event_id: "...", delta: { type: "content_delta", index: 0, text: "..." } }

# Compaction:
POST /v1/sessions/{id}/compact              # trigger; progress via turn/* + item/*
# Emits: agent.thread_context_compacted | contextCompaction item | SystemMessage(compact_boundary)

# Token usage:
{ type: "thread/tokenUsage/updated", usage: { total_tokens, total_input_tokens, total_output_tokens, total_thought_tokens, total_tool_use_tokens, total_cached_tokens, modalities: {...} } }

# Resumable background stream:
GET /v1/interactions/{id}?stream=true&last_event_id=<cursor>
```

---

## Stage 6 — Tools (Built-in, Custom & Function Calling)

**Purpose.** Give the agent callable capabilities — server-side built-ins, your own functions, or MCP-backed tools.

### Capabilities (union)
- **Built-in / prebuilt tools** (union of all named):
  - **File ops:** `read`/`Read`/`read_file`, `write`/`Write`/`write_file`, `edit`/`Edit`/`apply_diff`/`insert_content`/`search_and_replace`. (Anthropic, Claude, Bob, Vibe.)
  - **Search:** `glob`/`Glob`, `grep`/`Grep`, `list_files`, `GetSymbolsOverview`, `FindSymbol`, `FindReferencingSymbols`. (Anthropic, Claude, Bob.)
  - **Shell:** `bash`/`Bash`/`execute_command`/`Shell`/`commandExecution`. (Anthropic, Claude, Codex, Bob, Vibe, OpenAI sandbox.)
  - **Web:** `web_search`/`WebSearch`/`google_search`/`web_search_premium`, `web_fetch`/`WebFetch`/`url_context`, `webSearch` items with `action: search`/`openPage`/`findInPage`. (Anthropic, Claude, Codex, Google, Mistral, Vibe.)
  - **Code execution:** `code_execution`/`code_interpreter` (isolated container). (Google, Mistral, Vibe.)
  - **Image generation:** `image_generation`. (Mistral, Vibe, Google `gemini-3-pro-image`.)
  - **Discovery:** `ToolSearch` (large tool sets). (Claude.)
  - **Orchestration:** `Agent`/`spawn_subagent`/`start_subtask`/`task`/`collabToolCall`. (Claude, Bob, Vibe, Codex.)
  - **Skill activation:** `Skill`/`use_skill`. (Claude, Bob.)
  - **Mode switching:** `switch_mode`. (Bob.)
  - **Workflow invocation:** `start_workflow`. (Bob.)
  - **Todo management:** `update_todo_list`/`TaskCreate`/`TaskUpdate`. (Bob, Claude.)
  - **Questions:** `ask_followup_question`/`AskUserQuestion`. (Bob, Claude.)
  - **Permission requests:** `request_permissions`. (Codex.)
  - **Monitoring:** `Monitor` (watch a background script). (Claude.)
  - **File search (RAG):** `file_search`. (OpenAI hosted.)
  - **Computer use:** `computer_use`. (OpenAI hosted, Google — not supported on Antigravity.)
  - **Document library (RAG):** `document_library` with `library_ids`. (Mistral, Vibe.)
- **Custom tool definition** — JSON-schema `input_schema`/`parameters` + `name` + `description` (Anthropic, Google, Mistral, IBM watsonx, Vibe); code-defined via `@tool`/`tool()` wrapping a function with typed args (Zod/Pydantic/type hints) exposed via an in-process SDK MCP server (Claude, OpenAI); client-executed `dynamicToolCall` (Codex).
- **Tool annotations (behavioral hints, metadata not enforcement)** — `readOnlyHint`, `destructiveHint`, `idempotentHint`, `openWorldHint` (Claude `ToolAnnotations`); `anthropic/maxResultSizeChars` per-tool output limit (Claude); destructive annotation always triggers approval (Codex).
- **Tool configuration (availability vs permission)** — `tools: ["Read","Grep"]` availability allowlist; `allowed_tools` permission allowlist (run without prompt); `disallowed_tools` deny (bare name removes from context; scoped `Bash(rm *)` denies matching); `enabled: false` per tool. (Claude, Anthropic, Vibe.) Connector `tool_configuration.include`/`exclude` (mutually exclusive) + `requires_confirmation`. (Mistral, Vibe.)
- **Scoped tool rules** — `allowed_tools=["Bash(npm *)"]` auto-approve matching; `mcp__github__*` wildcard per server; `mcp__*` removes every MCP tool. (Claude.)
- **Parallel execution** — read-only tools run concurrently; state-modifying tools sequentially; custom tools default sequential, set `readOnlyHint: true` to enable parallel. (Claude.) Subagents in parallel up to `max_threads`. (Codex.) Input guardrails parallel with main agent. (OpenAI.)
- **Tool result blocks** — `content` accepts `text`, `image` (base64 + mimeType), `audio`, `resource` (URI + inline), `resource_link`; `structuredContent` machine-readable JSON; `isError: true` signals failure. (Claude.) Large outputs >100k tokens auto-written to sandbox file with truncated preview + path. (Anthropic, Claude, Codex.)
- **Error handling** — denied tools return rejected result with `deny_message` (Anthropic); uncaught exceptions converted to error results (Claude); decline/cancel completes `mcpToolCall` with error (Codex); `is_error` flag (Google).
- **Function-calling two-turn flow** — agent emits `agent.custom_tool_use`/`function.call`/`function_call` step → session pauses (`requires_action`) → you send `user.custom_tool_result`/`FunctionResultEntry`/`function_result` input part → session resumes. (Anthropic, Mistral, Google, Vibe.) Chat Completions: `tool_calls` + `tool_choice: "auto"|"any"` + `tool` role message with `tool_call_id`. (Mistral, IBM watsonx.)
- **Tool search (large tool sets)** — `ENABLE_TOOL_SEARCH` env: unset (on by default), `true` (always), `auto` (when combined defs >10% context), `auto:N`, `false`; max 10,000 tools; returns 3–5 relevant per search. (Claude.)
- **Multimodal input** — `text` + `image` (inline base64 + `mime_type`). (Google, Anthropic.)
- **Async tools** — return immediately, post results to a callback URL with correlation ID; sub-correlation objects. (IBM watsonx.) Supported for OpenAPI, Python, Flow, MCP.
- **Direct tool calling (no model)** — `connectors.call_tool_async(connector_id, tool_name, arguments)` returns content blocks. (Mistral.)

### Alternative approaches
- **Built-in toolset as a flag** (Anthropic `agent_toolset_20260401`) vs **explicit tool list** (Mistral, Google, IBM watsonx) vs **mode-gated groups** (Bob `groups`) vs **code-defined** (OpenAI, Claude).
- **Custom tools as JSON schema** (Anthropic, Google, Mistral, IBM watsonx) vs **code-wrapped functions via in-process MCP server** (Claude `@tool`, OpenAI `tool()`) vs **client-executed dynamic tools** (Codex `dynamicToolCall`).
- **Availability + permission as separate axes** (Claude `tools` vs `allowed_tools` vs `disallowed_tools`) vs **single `tool_configuration.include`/`exclude`** (Mistral) vs **per-tool `permission: always|ask`** (Vibe) vs **per-tool `needsApproval: true`** (OpenAI).
- **Large output → sandbox file** (Anthropic, Claude, Codex auto) vs **inline only** (others).

### Unified API specification

```
# Built-in toolset (Anthropic-style):
{ type: "agent_toolset", default_config: { enabled: true, permission_policy: {type:"always_allow"} },
  configs: [ { name: "bash", enabled: true, permission_policy: {type:"always_ask"} } ] }

# Custom tool (JSON-schema style):
{ type: "custom"|"function",
  name: "get_weather",
  description: "Get current weather for a location",   # 3-4 sentences, very detailed
  input_schema: { type: "object", properties: {...}, required: [...] },
  annotations?: { readOnlyHint, destructiveHint, idempotentHint, openWorldHint, maxResultSizeChars? },
  needsApproval?: bool }

# Custom tool (code style — Claude/OpenAI):
@tool
def get_weather(location: str) -> str: ...   # type hints → schema; wrap in create_sdk_mcp_server

# MCP toolset (Anthropic):
{ type: "mcp_toolset", mcp_server_name: "weather",
  default_config: { enabled: true, permission_policy: {type:"always_ask"} },
  configs: [ { name: "<bare_tool_name>", enabled: true } ] }

# Connector (Mistral/Vibe):
{ type: "connector", connector_id: "<name|uuid>",
  tool_configuration: { include?: [...], exclude?: [...], requires_confirmation?: [...] } }

# Document library (Mistral/Vibe):
{ type: "document_library", library_ids: ["..."] }

# Scoped rules (Claude):
allowed_tools: ["Bash(npm *)", "mcp__github__list_issues"]
disallowed_tools: ["Bash(rm *)", "mcp__github"]   # bare removes server; scoped denies matching

# Tool search (Claude):
env: { ENABLE_TOOL_SEARCH: "auto"|"true"|"auto:N"|"false" }  # max 10000 tools, 3-5 per search

# Function-calling two-turn flow:
# 1) Session emits: { type: "agent.custom_tool_use", tool: "get_weather", input: {location:"Paris"} }
#    → session.status_idle, stop_reason.requires_action, event_ids=[<custom_tool_use_id>]
# 2) You send:
POST /v1/sessions/{id}/events
  body: { events: [ { type: "user.custom_tool_result", custom_tool_use_id: "...", result: "..." } ] }
# 3) Session resumes.

# Async tool callback (IBM watsonx):
POST /v1/tools/{tenant_id}/callback/{correlation_id}
  body: { result: ... }   # content-type application/json

# Tool result block schema:
{ content: [ { type: "text", text }, { type: "image", data: base64, mimeType },
             { type: "resource", uri, text? }, { type: "resource_link", ... } ],
  structuredContent?: object,
  isError?: bool }
```

---

## Stage 7 — MCP Connectors

**Purpose.** Connect the agent to external tools and data sources via the Model Context Protocol — as a client (consume tools) and optionally as a server (be orchestrated by others).

### Capabilities (union)
- **MCP client — declare servers** — `mcp_servers[]` with `type: "url"`, unique `name`, `url` (≤2048 chars); max 20 servers; every server referenced by a toolset and vice versa. (Anthropic.) `mcpServers` in `settings.json` (Bob, Claude `.mcp.json`). `[mcp_servers.<name>]` in `config.toml` (Codex). `mcp_server` tool in `tools[]` (Google). `mcp` binding / `MCPToolKitConfig` (IBM watsonx). Connector with `server` URL (Mistral, Vibe). `MCPServerStdio`/`MCPServerStreamableHttp` (OpenAI).
- **Transports** — **stdio** (local subprocess, `command`/`args`/`env`), **streamable HTTP** (`url`/`headers`), **SSE** (legacy deprecated), **SDK MCP server** (in-process custom tools), **MCP tunnels** (private servers). (All.)
- **Auth** — vault credentials at session creation matched by URL (Anthropic `mcp_oauth`/`static_bearer`); `env` (stdio) or `headers` (http) with `${VAR}` expansion (Claude, Bob); OAuth 2.1 — SDK doesn't run the flow, you complete it and pass bearer token (Claude); `mcpServer/oauth/login` returns auth URL (Codex); Connector `auth_data` (OAuth client_id/secret) + `get_auth_url` (Mistral, Vibe); `connections` object (IBM watsonx).
- **Allow/deny tools** — `default_config.enabled: false` + explicit enables (Anthropic); `alwaysAllow` array per server + `disabled` (Bob); `allowed_tools: ["mcp__github__*"]` wildcard (Claude); `tool_configuration.include`/`exclude` (Mistral); `tools[]` select which to import (`['*']` for all) (IBM watsonx); `allowed_tools` on `mcp_server` tool (Google).
- **Tool output handling** — outputs >100k tokens auto-written to sandbox file (Anthropic, Claude, Codex); `MAX_MCP_OUTPUT_TOKENS` with per-tool `anthropic/maxResultSizeChars` override (Claude).
- **Failures** — session creation does NOT validate connectivity; on failure session still starts, `session.error` emitted with `mcp_server_name` + `retry_status` (`mcp_connection_failed_error`, `mcp_authentication_failed_error`); retried on next idle→running. (Anthropic.) `init` message reports each server `status: connected|failed`; connection timeout 30s default via `MCP_TIMEOUT`. (Claude.) `mcpServer/startupStatus/updated` notification. (Codex.)
- **Runtime control** — `reconnect_mcp_server(name)`, `toggle_mcp_server(name, enabled)`, `get_mcp_status()` (Claude); `config/mcpServer/reload` (Codex); `Refresh tools` (Vibe).
- **MCP resources** — `mcpServer/resource/read` reads a single MCP resource through an initialized server (Codex).
- **MCP as server** — `codex mcp-server` exposes `codex` (start session with config overrides) + `codex-reply` (continue by threadId) tools to MCP clients (Codex). Codex-as-MCP-server for multi-agent orchestration with OpenAI Agents SDK.
- **Elicitation** — `mcpServer/elicitation/request`: `mode: "form"`/`"openai/form"` (`message` + `requestedSchema`) or `mode: "url"` (`message` + `url` + `elicitationId`); respond `accept`+`content` or `decline`/`cancel`. (Codex.) `AskUserQuestion` and MCP tools with `_meta["anthropic/requiresUserInteraction"]` always fall through to ask flow (Claude).
- **Listing tools without importing** — `POST /v1/toolkits/prepare/list-tools` (IBM watsonx); `connectors.list_tools_async` with `refresh`/`pretty` (Mistral).
- **Visibility / sharing** — `private`/`shared_workspace`/`shared_org` (Mistral).
- **System prompt injection** — Connector `system_prompt` injected when its tools are used (Mistral).
- **MCP workflows** — agentic workflows with MCP (IBM watsonx, public preview).

### Alternative approaches
- **Remote streamable HTTP only** (Google, Mistral) vs **stdio + HTTP + SSE + in-process SDK server** (Claude, Bob, Codex, OpenAI, IBM watsonx).
- **Auth via vault at session creation** (Anthropic) vs **env/headers with `${VAR}` expansion** (Claude, Bob) vs **OAuth login flow returning auth URL** (Codex, Mistral) vs **connections object** (IBM watsonx).
- **MCP client only** (most) vs **MCP client + MCP server** (Codex `codex mcp-server`).

### Unified API specification

```
# Declare MCP servers (agent-level):
mcp_servers: [
  { type: "url", name: "github", url: "https://...",   # streamable HTTP (Anthropic/Google)
    headers?: { Authorization: "Bearer ..." } },
  { type: "stdio", name: "filesystem",                  # local subprocess (Claude/Bob/Codex/OpenAI)
    command: "npx", args: ["@modelcontextprotocol/server-filesystem", "./sample_files"],
    env?: { GITHUB_TOKEN: "${GITHUB_TOKEN}" },          # ${VAR} expansion
    alwaysAllow?: ["list_files"], disabled?: bool },
  { type: "sdk", name: "my-tools", version: "1.0",      # in-process (Claude)
    tools: [...] }
]

# Connector (Mistral/Vibe managed MCP):
POST /v1/connectors
  body: { name, server: "<url>", visibility: "private"|"shared_workspace"|"shared_org",
          description?, icon_url?, headers?, auth_data?: { client_id, client_secret },
          system_prompt? }

# Auth:
POST /v1/mcp_servers/{name}/oauth/login     # → auth_url; emits oauthLogin/completed (Codex)
POST /v1/connectors/{id}/auth_url            # → { auth_url, ttl } (Mistral)
# Vault credential matched by mcp_server_url at session creation (Anthropic)

# Allow/deny:
tool_configuration: { include: ["list_issues"], exclude: ["delete_repo"], requires_confirmation: ["create_issue"] }
# OR default_config.enabled: false + configs[].enabled: true (Anthropic)
# OR alwaysAllow: ["list_files"] per server (Bob)

# Runtime control:
POST /v1/sessions/{id}/mcp/{name}/reconnect
POST /v1/sessions/{id}/mcp/{name}/toggle     # body: { enabled: bool }
GET  /v1/sessions/{id}/mcp/status
POST /v1/mcp/config/reload                   # reload from disk (Codex)

# Resources:
POST /v1/mcp/resource/read   body: { server, uri }

# Elicitation (server → client request):
{ type: "mcpServer/elicitation/request", mode: "form"|"openai/form"|"url",
  message: "...", requestedSchema?: {...}, url?: "...", elicitationId?: "..." }
# Respond: { action: "accept", content: {...} } | { action: "decline"|"cancel", content: null }

# MCP-as-server (Codex):
# Tool "codex":      { prompt, approval-policy, sandbox, model, cwd, ... } → start session
# Tool "codex-reply": { prompt, threadId } → continue session

# Failures (Anthropic):
# session.error with error.type: "mcp_connection_failed_error" | "mcp_authentication_failed_error"
#   + mcp_server_name + retry_status; retried on next idle→running
```

---

## Stage 8 — Skills

**Purpose.** Package reusable, filesystem-based domain expertise that loads on demand (progressive disclosure) so it only impacts the context window when relevant.

### Capabilities (union)
- **Skill concept** — `SKILL.md` + supporting files; loaded via progressive disclosure. (Anthropic, Claude, Codex, Google, Bob, Vibe.)
- **Progressive disclosure stages** — Discovery (name + description ~100 tokens) → Activation (full SKILL.md) → Execution (supporting files on demand). (Vibe, explicit; others implicit.)
- **File format / fields** — `SKILL.md` with YAML frontmatter: `name`/`display_title`, `description` (trigger text "Use when…"), `allowed-tools`, `disable-model-invocation`, `context: fork` (run in subagent). Plus optional `SKILL.json` with `interface` + `dependencies.tools[]` (`env_var`/`mcp`). (Claude, Codex, Vibe.) Body = markdown instructions/steps.
- **Pre-built skills** — `pptx`, `xlsx`, `docx`, `pdf` (Anthropic, Claude); `challenge-my-thinking`, `data-analysis`, `deep-research`, `doc-coauthoring`, `document-review`, `internal-comms`, `meeting-prep`, `research-synthesis`, `skill-creator`, `stakeholder-translator`, `structured-extraction`, `vibe-work-onboarding` (Vibe); Carbon Design, DITA, Jira, `create-plan` (Bob).
- **Custom skills** — authored by you; uploaded as zip or individual files; `POST /v1/skills` multipart upload returns `skill_*` ID + `latest_version`. (Anthropic.)
- **Creation routes** — from editor UI (Vibe); from a task ("turn this into a Skill") (Vibe); file-based (Bob, Codex, Claude, Google); API upload (Anthropic); Studio → `Publish to Vibe` (Vibe).
- **Activation modes** — auto-match (description triggers); slash command `/{skill-name}` (Vibe); natural reference by name (Vibe); `$<skill-name>` in user text (Codex); `skill` input item recommended (Codex injects full instructions); `use_skill` tool (Bob); `Skill` tool (Claude); `disable-model-invocation: true` for explicit-only. (Claude, Vibe.)
- **Scopes / admin controls** — Personal vs Workspace (Vibe); force-enabled by workspace admins (Vibe); org admins toggle per workspace (Vibe); `perCwdExtraUserRoots` / `skills/extraRoots/set` extra scan paths (Codex); max 20 skills per session across all agents (Anthropic).
- **Attach to agent/session** — `skills[]` on agent (Anthropic, Claude); `skills.config` on custom agent (Codex); `AgentDefinition.skills` preloads into subagent (Claude); implicit via environment filesystem `.agents/skills/<name>/SKILL.md` (Google); enabled per Work session (Vibe).
- **Versioning** — `version` pin or `latest` (Anthropic custom skills).
- **Skills vs Workflows distinction** — Skills = reusable behavior/method; Workflows = deterministic coded automation. (Vibe, Bob.)
- **Skill as tool binding** — `SkillToolBinding` calls a skillset/skill operation by id + path + method. (IBM watsonx.)
- **Skill dependencies** — `SKILL.json` `dependencies.tools[]` declaring required `env_var`/`mcp`. (Codex.)
- **Skill file watching** — `skills/changed` notification on local file changes → invalidation. (Codex.)
- **Plugin skills** — `plugin/skill/read` reads remote plugin skill Markdown on demand. (Codex.)
- **Skill create/update scheduled tasks** — invoke with `$skill-name` in desktop app. (Codex.)

### Alternative approaches
- **Skills as first-class agent attachment** (Anthropic, Claude, Codex) vs **skills auto-discovered from environment filesystem** (Google, Vibe `.agents/skills/`) vs **skill as a tool binding type** (IBM watsonx) vs **no skills abstraction** (Mistral, OpenAI).
- **Activation: model-invoked via description** (Anthropic, Claude, Vibe auto-match) vs **explicit slash command** (Vibe, Codex `$skill-name`) vs **tool-based `use_skill`** (Bob) vs **`skill` input item** (Codex).
- **Built-in skills shipped** (Anthropic/Claude office skills, Vibe 12 built-ins, Bob examples) vs **user-only** (others).

### Unified API specification

```
# File-based skill (universal):
<agent-project>/.agents/skills/<skill-name>/SKILL.md
---
name: slide-maker
description: "Use when generating slide decks from structured content."  # trigger text
allowed-tools: [read, write]
disable-model-invocation: false
context: fork          # run in a subagent
---
1. Read the source content.
2. Outline the deck.
...

# Optional SKILL.json (Codex):
{ interface: {...}, dependencies: { tools: [ {type:"env_var", value, description}, {type:"mcp", value, transport, url} ] } }

# API upload (Anthropic):
POST /v1/skills           # multipart: files
  → { skill_id: "skill_...", latest_version: 1 }

# Attach to agent:
skills: [ { type: "anthropic"|"custom", skill_id: "xlsx"|"skill_...", version?: "latest"|1 } ]

# Enable/disable:
POST /v1/skills/config/write   body: { path, enabled: bool }

# List:
POST /v1/skills/list   body: { cwds: [...], forceReload?: bool, perCwdExtraUserRoots?: [...] }

# Activation:
#  - auto-match: description triggers at session start (~100 tokens discovery)
#  - slash: /<skill-name>  or  $<skill-name> in user text  (Vibe/Codex)
#  - input item: { type: "skill", skill: "<name>" } on turn/start  (Codex — injects full instructions)
#  - tool: use_skill(skill_name) / Skill tool  (Bob/Claude)

# Limits: ≤20 skills per session across all agents (Anthropic)
```

---

## Stage 9 — Permissions, Approvals & Human Review

**Purpose.** Control which tool calls run automatically vs wait for a human, and provide review/auto-review mechanisms.

### Capabilities (union)
- **Permission policy types/modes** (union):
  - `always_allow` / `always_ask` (Anthropic).
  - `default` / `acceptEdits` / `plan` / `dontAsk` / `auto` / `bypassPermissions` (Claude).
  - `read-only` / `workspace-write` / `danger-full-access` sandbox_mode (Codex, Vibe).
  - `untrusted` / `on-request` / `never` approval_policy (Codex); granular `{sandbox_approval, rules, mcp_elicitations, request_permissions, skill_approval}`.
  - `read_only` / `write_only` / `read_write` / `admin` ToolPermission (IBM watsonx).
  - `default` / `plan` / `accept-edits` / `auto-approve` agent modes (Vibe Code).
  - `requires_confirmation` per tool (Mistral, Vibe).
  - `needsApproval: true` per tool (OpenAI).
- **Setting / changing** — at agent creation/update (Anthropic); mid-session via `set_permission_mode()` (Claude); `POST /v1/sessions/{id}` updates tools/mcp_servers without new agent version (Anthropic); CLI flags `--sandbox`, `--ask-for-approval`, `--yolo` (Codex, Bob); `/permissions` runtime overrides (Codex); `config.toml` `[apps.*]` (Codex); `turn/start` sandbox policy override (Codex).
- **Confirmation requests flow** (unified):
  1. Tool with `always_ask`/`requires_confirmation`/`needsApproval` invoked.
  2. Session pauses (`requires_action` / `confirmation_status: pending` / `result.interruptions`).
  3. You respond: `user.tool_confirmation` (`allow`/`deny` + optional `deny_message`) (Anthropic); `tool_confirmations: [{tool_call_id, confirmation: "allow"|"deny"}]` (Mistral); `state.approve(interruption)`/`state.reject(interruption)` (OpenAI); `Continue`/`Always allow`/`Decline` (Vibe Work); CLI keyboard `Enter`/`Y`/`N` (Vibe Code, Bob).
  4. Session resumes; denied tools return rejected result with `deny_message`.
- **Multiple confirmations per request** — batch approve/deny. (Anthropic, Mistral.)
- **Per-tool / per-app approval config** — `configs[].permission_policy` (Anthropic); `[apps.<id>.tools.<tool>]` with `approval_mode: auto|prompt|approve` (Codex); `alwaysAllow` per server (Bob); `--allowed-tools="git status"` (Bob Shell).
- **Auto-review** — a separate reviewer agent decides approvals in place of a human; only applies when approvals interactive; circuit breaker (3 consecutive denials or 10 in last 50); `/approve` override picker; reviewer sees compact transcript + exact request; denial returns rationale + stronger instruction. (Codex.) `auto` mode model classifier (Claude).
- **Granular approval decision payloads** — command execution: `accept`/`acceptForSession`/`decline`/`cancel`/`acceptWithExecpolicyAmendment`; file change: `accept`/`acceptForSession`/`decline`/`cancel`. (Codex.)
- **Permission requests (runtime escalation)** — `request_permissions` tool sends `item/permissions/requestApproval` with requested network/filesystem permissions; respond with granted subset; `scope: "session"` persists grant. (Codex.)
- **Network approval context** — concurrent network prompts grouped by destination (host + protocol + port). (Codex.)
- **MCP / app tool approvals** — `tool/requestUserInput` (Accept/Decline/Cancel); destructive annotations always trigger approval. (Codex.)
- **Guardrails** — input/output/tool guardrails; input guardrails run in parallel with main agent (`run_in_parallel`); output guardrails validate/redact final output; tool guardrails check args/results. (OpenAI.)
- **Sandbox/approval combinations** — `--sandbox workspace-write --ask-for-approval on-request` (auto preset); `read-only --never` (CI); `workspace-write --untrusted`; `workspace-write --on-request -c approvals_reviewer=auto_review` (auto-review); `--yolo` (full access). (Codex.)
- **Outside-CWD confirmation** — mandatory confirmation when a tool reads/writes/runs outside the current working directory, regardless of agent. (Vibe Code.)
- **Subagent inheritance** — when parent uses `bypassPermissions`/`acceptEdits`/`auto`, subagents inherit and cannot override. (Claude.) Subagents inherit parent turn's sandbox policy + live runtime overrides. (Codex.)
- **Evaluation order (Claude)** — Hooks → Deny rules → Ask rules → Permission mode → Allow rules → `canUseTool` callback. Precedence: `deny` > `defer` > `ask` > `allow`.
- **`canUseTool` callback** — final decision returning `PermissionResultAllow`/`Deny`; can `updated_input` to redirect paths, `interrupt: true`. (Claude.)
- **Trust gating** — load `.vibe/` config only from trusted folders; remembered in `~/.vibe/trusted_folders.toml`; temp via `--trust`. (Vibe Code.)
- **Admin restrictions** — `requirements.toml` disallows `approval_policy = "never"`, constrains sandbox modes. (Codex.)
- **Programmatic mode defaults** — `vibe --prompt` defaults to `auto-approve`, disables interactive tools; pass `--agent plan` for safe read-only. (Vibe.) `bob -p` defaults to non-destructive tools only. (Bob.)
- **`bypassPermissions` constraints** — cannot run as root on Unix; use only in isolated envs. (Claude.)

### Alternative approaches
- **Two-axis model: sandbox_mode (capabilities) + approval_policy (when to ask)** (Codex) vs **single permission_policy enum** (Anthropic `always_allow`/`always_ask`) vs **permission_mode + allow/deny rules + callback** (Claude) vs **per-tool `needsApproval`/`requires_confirmation` flag** (OpenAI, Mistral) vs **agent mode bundles** (Vibe Code) vs **ToolPermission enum** (IBM watsonx) vs **not covered** (Google).
- **Auto-review by separate agent** (Codex `auto_review`) vs **auto mode model classifier** (Claude) vs **no auto-review** (others).
- **Guardrails as a distinct validation layer** (OpenAI input/output/tool) vs **folded into permission flow** (others).

### Unified API specification

```
# Permission policy (agent/run level):
permission_mode: "default"|"acceptEdits"|"plan"|"dontAsk"|"auto"|"bypassPermissions"  # Claude
# OR
permission_policy: { type: "always_allow" } | { type: "always_ask" }                   # Anthropic
# OR (Codex two-axis):
sandbox_mode: "read-only"|"workspace-write"|"danger-full-access"
approval_policy: "untrusted"|"on-request"|"never" | { granular: { sandbox_approval, rules, mcp_elicitations, request_permissions, skill_approval } }

# Per-tool:
{ name: "bash", permission_policy: {type:"always_ask"} }                 # Anthropic
{ name: "bash", needsApproval: true }                                    # OpenAI
{ type: "connector", tool_configuration: { requires_confirmation: ["create_issue"] } }  # Mistral
[tools.bash] permission = "ask"; allow = ["git status"]; deny = ["rm -rf *"]             # Vibe Code
[apps.google_drive.tools."files/delete"] enabled=false; approval_mode="approve"          # Codex

# Confirmation flow:
# 1) Session emits pause: { stop_reason: { type: "requires_action", event_ids: [...] } }
# 2) You respond:
POST /v1/sessions/{id}/events
  body: { events: [ { type: "user.tool_confirmation", tool_use_id: "...",
                      result: "allow"|"deny", deny_message? } ] }
# OR (Mistral):
POST /v1/conversations/{id}  body: { tool_confirmations: [{tool_call_id, confirmation: "allow"|"deny"}] }
# OR (OpenAI):
state.approve(interruption) | state.reject(interruption)
run(agent, state)   # resume same run (not a new turn)

# Auto-review (Codex):
approvals_reviewer = "user" | "auto_review"
[auto_review] policy = "YOUR POLICY"
# Circuit breaker: 3 consecutive denials OR 10 in last 50 → interrupt turn
# Override: /approve → "Auto-review Denials" picker (one retry, still reviewed)

# Permission escalation (Codex):
# request_permissions tool → item/permissions/requestApproval
# Respond: { permissions: <granted subset>, scope: "session"|"turn" }

# Evaluation order (Claude):
# Hooks(PreToolUse allow/deny/ask/defer) → Deny rules → Ask rules → Permission mode → Allow rules → canUseTool
# Precedence: deny > defer > ask > allow

# Guardrails (OpenAI):
input_guardrails: [{ run_in_parallel: true|false, ... }]
output_guardrails: [...]
tool_guardrails: [{ tool: "...", ... }]

# Admin restrictions (Codex requirements.toml):
# disallow approval_policy = "never"; constrain sandbox_modes
```

---

## Stage 10 — Hooks & Lifecycle Callbacks

**Purpose.** Register your own code to run at specific lifecycle points — intercept/modify tool calls, observe results, react to session/subagent/compaction events.

### Capabilities (union)
- **Hook events available** (Claude, most complete): `PreToolUse`, `PostToolUse`, `PostToolUseFailure`, `PostToolBatch`, `UserPromptSubmit`, `MessageDisplay`, `Stop`, `SubagentStart`/`SubagentStop`, `PreCompact`, `PermissionRequest`, `SessionStart`/`SessionEnd`, `Notification`, `Setup`, `TeammateIdle`/`TaskCompleted`, `ConfigChange`, `WorktreeCreate`/`WorktreeRemove`. Plus `TaskCreated` (agent-team).
- **Codex hooks** — `hook/started` / `hook/completed` with `{threadId, turnId?, run}` boundaries.
- **Configuration** — `hooks` option: keys = event names; values = arrays of `{matcher, hooks, timeout}`. (Claude.) Shell-command hooks from `.claude/settings.json` loaded when `settingSources` includes `project`. (Claude.) External agent migration supports hooks via `externalAgentConfig/detect`/`import`. (Codex.)
- **Matchers** — exact string if `[a-z0-9_-|, ]`; regex if other chars; `*`/empty matches all; tool hooks match tool names only, filter by file path inside callback. (Claude.)
- **Callback signature / IO** — `async cb(input_data, tool_use_id, context)`; returns top-level `{systemMessage, continue}` + `hookSpecificOutput`:
  - `PreToolUse`: `permissionDecision` (`allow`/`deny`/`ask`/`defer`), `permissionDecisionReason`, `updatedInput` (must pair with allow/ask).
  - `PostToolUse`: `additionalContext`, `updatedToolOutput` (replace output before model sees it).
  - Async output: `{async: true, asyncTimeout}` — agent proceeds without waiting.
  - Return `{}` to allow without changes. (Claude.)
- **Multiple matching hooks** — run in parallel; most restrictive decision wins (`deny` > `defer` > `ask` > `allow`). (Claude.)
- **Quality-gate hooks (teams)** — `TeammateIdle` (exit 2 → feedback), `TaskCreated` (exit 2 → block creation), `TaskCompleted` (exit 2 → block completion). (Claude.)
- **Common patterns** — block `.env` writes; redirect Write paths to `/sandbox` via `updatedInput`; auto-approve read-only tools; forward `Notification` to Slack; track subagent completions; run webhooks after `PostToolUse`; enforce DB read-only. (Claude.)
- **Async tool callbacks** — tools post results to a callback URL with correlation ID; supported for OpenAPI, Python, Flow, MCP. (IBM watsonx.)
- **Flow in-graph control nodes** — error-handling / masking-sensitive-data nodes (flow-internal hooks). (IBM watsonx.)
- **OpenAI hooks note** — `RunHooks`/`AgentHooks` exist but are explicitly **not** for blocking/approval/execution-shaping — use guardrails/approvals/filters instead; hooks are for lifecycle side effects (logging, tracing). (OpenAI.)
- **Webhooks for vaults** — `vault.archived`, `vault.deleted`, `vault_credential.archived`/`deleted`/`refresh_failed`. (Anthropic.)
- **Span events as observability hooks** — `span.model_request_start`/`_end`, `span.outcome_evaluation_*`. (Anthropic.)

### Alternative approaches
- **Rich user-defined hook system** (Claude: 18+ events with matchers, permission decisions, input/output modification, async) vs **minimal hook boundaries** (Codex `hook/started`/`completed`) vs **async tool callbacks only** (IBM watsonx) vs **side-effect-only hooks explicitly not for blocking** (OpenAI `RunHooks`/`AgentHooks`) vs **no hooks** (Anthropic managed, Google, Mistral, Bob, Vibe).

### Unified API specification

```
# Hook registration (Claude-style):
hooks: {
  PreToolUse: [
    { matcher: "Write|Edit",                 # exact if [a-z0-9_-|, ]; regex otherwise; "*" = all
      hooks: [ async (input, tool_use_id, context) => ({
        systemMessage: "...",                 # shown to user
        continue: true,
        hookSpecificOutput: {
          permissionDecision: "allow"|"deny"|"ask"|"defer",
          permissionDecisionReason: "...",
          updatedInput: { ... }               # must pair with allow/ask
        }
      }) ],
      timeout: 30000 }
  ],
  PostToolUse: [ ... ],     # hookSpecificOutput: { additionalContext, updatedToolOutput }
  PreCompact: [ ... ],
  SubagentStart: [ ... ], SubagentStop: [ ... ],
  Notification: [ ... ],    # forward to Slack etc.
  SessionStart: [ ... ], SessionEnd: [ ... ],
  TeammateIdle: [ ... ], TaskCompleted: [ ... ],   # exit 2 → block/feedback
  WorktreeCreate: [ ... ], WorktreeRemove: [ ... ]
}

# Async hook output:
{ async: true, asyncTimeout: 5000 }   # agent proceeds without waiting; side-effects only

# Precedence on multiple matching hooks: deny > defer > ask > allow

# Codex-style hook boundaries:
# Events: hook/started { threadId, turnId?, run }, hook/completed { ... }

# Async tool callback (IBM watsonx):
POST /v1/tools/{tenant_id}/callback/{correlation_id}
  body: { result: ... }
```

---

## Stage 11 — Credentials, Secrets & Vaults

**Purpose.** Decouple secrets from agent definitions; register once, reference by ID; support rotation, outbound policies, and OAuth for connectors.

### Capabilities (union)
- **Vault concept** — workspace-scoped collection of credentials referenced by ID at session creation; decouples secrets from reusable agent definitions. (Anthropic.)
- **Credential types** — `mcp_oauth` (OAuth 2.0 with auto-refresh: `access_token`, `expires_at`, `refresh` block with `token_endpoint`/`client_id`/`scope`/`refresh_token`/`token_endpoint_auth.type`), `static_bearer` (fixed token + `mcp_server_url`), `environment_variable` (`secret_name` + `secret_value` stored as opaque placeholder, substituted at egress; `injection_location: header|body`; `networking.allowed_hosts` scopes hosts). (Anthropic.)
- **Keying / constraints** — MCP credentials keyed by `mcp_server_url`; env-var by `secret_name`; values write-only (never returned); unique key per vault (duplicate → 409); keys immutable; max 20 credentials per vault. (Anthropic.)
- **Referencing at session/run** — `vault_ids[]` on session creation; runtime matching by URL; multiple vaults → first match wins; in multi-agent, credentials apply to every thread. (Anthropic.) `connection_ids[]` on agent (IBM watsonx). `context_variables` per-run (IBM watsonx).
- **Rotation** — credentials re-resolved periodically; rotation/archival/deletion propagate to running sessions without restart; structural fields locked (archive + recreate); `mcp_oauth_validate` endpoint returns `valid`/`invalid`/`unknown`. (Anthropic.)
- **Outbound policies** — `networking.allowed_hosts` + `injection_location` scope env-var usage; `allow_mcp_servers`/`allow_package_managers` (Anthropic); `/v1/outbound-policies` URL allow/deny with `/check` (IBM watsonx); network domain rules (Codex); auto-review blocks sending secrets to untrusted destinations + credential/cookie probing (Codex).
- **Connections (IBM watsonx)** — credential binding (OAuth2/API key/basic auth) referenced by `connection_id`; OAuth callback flow via `/v1/connections/callback`; OpenAPI connections ≤1 `app_id`.
- **OAuth for connectors** — `mcp_oauth` with auto-refresh (Anthropic); `mcpServer/oauth/login` (Codex); `connectors.get_auth_url` (Mistral, Vibe); `connections/callback` (IBM watsonx); `claude mcp login` from shell (Claude).
- **Cloud secrets** — extra encryption; decrypted only for task execution; only available to setup scripts, removed before agent phase; container cache invalidated on secret change. (Codex.)
- **`.worktreeinclude`** — repo file listing ignored paths to copy into managed worktrees (e.g. `.env`); skips symlinks. (Codex.)
- **API keys** — General vs Inference types; admin manages all; revoke; secret shown once. (Bob.)
- **Env vars / headers for MCP** — stdio `env`, http `headers` with `${VAR}` expansion. (Claude, Bob.)
- **Egress proxy header transform** — credentials injected into HTTP headers by egress proxy; never exposed inside sandbox as env vars/files; refreshable per interaction. (Google.)
- **Data/training policy** — data accessed via Connectors never used to train/fine-tune models. (Vibe.)

### Alternative approaches
- **First-class vault resource with typed credentials + rotation + validation** (Anthropic) vs **connections CRUD with OAuth callback** (IBM watsonx) vs **egress-proxy header transform** (Google) vs **MCP env/headers with `${VAR}`** (Claude, Bob) vs **connector `headers`/`auth_data`** (Mistral, Vibe) vs **no vault** (OpenAI, Google, Mistral beyond connectors).

### Unified API specification

```
POST /v1/vaults
  body: { display_name, metadata?: { key: value } }   # maps to your user records
  → { id: "vlt_..." }

POST /v1/vaults/{id}/credentials
  body: {
    type: "mcp_oauth" | "static_bearer" | "environment_variable",
    # mcp_oauth:
    access_token, expires_at, refresh?: { token_endpoint, client_id, scope, refresh_token, token_endpoint_auth: {type:"none"|"client_secret_basic"|"client_secret_post"} },
    # static_bearer:
    mcp_server_url, token,
    # environment_variable:
    secret_name, secret_value, networking: { allowed_hosts, injection_location: "header"|"body" }
  }
  → { credential_id }

# Values write-only; unique key per vault; keys immutable; ≤20 credentials/vault
# Reference at session creation:
POST /v1/sessions  body: { vault_ids: ["vlt_..."], ... }

# Rotation (propagates to running sessions):
POST /v1/vaults/{id}/credentials/{cred_id}     # update secret values / injection_location
POST /v1/vaults/{id}/credentials/{cred_id}/mcp_oauth_validate   # → valid|invalid|unknown

# Outbound policies (IBM watsonx):
POST /v1/outbound-policies   body: { urls: { allow: [...], deny: [...] } }
POST /v1/outbound-policies/check   body: { urls: [...] }   # → allowed?

# Connections (IBM watsonx):
POST /v1/connections/applications   # create (validation, duplicate check, OAuth2)
GET  /v1/connections/callback       # OAuth callback

# Egress proxy transform (Google):
network.allowlist[].transform = { "Authorization": "Bearer <token>" }   # injected by proxy; never in sandbox

# Lifecycle webhooks (Anthropic):
# vault.archived, vault.deleted, vault_credential.archived/deleted/refresh_failed
```

---

## Stage 12 — Multi-Agent Orchestration

**Purpose.** Coordinate multiple agents — delegate subtasks, hand off conversation ownership, form teams with direct messaging, or fan out batch work.

### Capabilities (union)
- **Subagent** — separate agent instance spawned for a focused subtask in isolated context; only final message returns to parent. (Claude `Agent` tool; Codex `collabToolCall`; Bob `spawn_subagent`; Anthropic roster entries.)
- **Subagent definition methods** — programmatic `agents: {name: AgentDefinition}` (Claude); filesystem `.claude/agents/*.md` with YAML frontmatter (Claude); `~/.codex/agents/*.toml` (Codex); built-in `Explore`/`Plan`/`general-purpose` (Claude), `default`/`worker`/`explorer` (Codex), `explore`/`general` (Bob).
- **`AgentDefinition` fields** — `description` (drives auto-delegation), `prompt`, `tools`, `disallowedTools`, `model`, `skills`, `memory`, `mcpServers`, `initialPrompt`, `maxTurns`, `background`, `effort`, `permissionMode`, `isolation: worktree`, `color`. (Claude.) Codex custom agent: `name`, `description`, `developer_instructions`, `nickname_candidates`, `model`, `model_reasoning_effort`, `sandbox_mode`, `mcp_servers`, `skills.config`.
- **Inheritance** — receives own system prompt + Agent tool prompt + project CLAUDE.md + tool defs (inherited or subset); doesn't receive parent conversation history/system prompt/tool results. (Claude.) Subagents inherit parent turn's sandbox policy + live runtime overrides. (Codex.) `fork_context: true` passes parent conversation history into subagent. (Bob.)
- **Invocation** — automatic (via `description`), explicit (name in prompt, `@mention`, `claude --agent <name>`), dynamic (factory functions at query time). (Claude.) `spawn_subagent(type, description, fork_context?)` (Bob). `collabToolCall` items (Codex).
- **Foreground/background** — subagents run background by default; `run_in_background: false` when result needed before continuing. (Claude.) Parallel/background; `/agent` inspects/switches threads. (Codex.)
- **Nesting** — subagents can spawn their own subagents; depth 5 max can't spawn more (Claude); `agents.max_depth` default 1 prevents deeper descendants (Codex).
- **Resuming subagents** — Agent tool result includes `agentId`; resume by `resume: sessionId` + agent ID in prompt. (Claude.) `codex-reply` continues by `threadId` (Codex). Built-in `Explore`/`Plan` one-shot. (Claude.)
- **Restricting** — `tools: [Agent(worker, researcher)]` allowlist; `permissions.deny: ["Agent(Explore)"]`; omit `Agent` to block all. (Claude.)
- **Agent teams** — multiple coordinated instances (lead + teammates) with shared task list + direct inter-agent messaging (`SendMessage`); experimental; `in-process` or split panes (tmux/iTerm2). (Claude.)
- **Threads (multi-agent)** — coordinator delegates to roster; each agent runs in its own context-isolated thread sharing the same sandbox; max 25 concurrent threads; max 1 level of delegation; cross-posted blocking events with auto-routed responses. (Anthropic.)
- **Roster entry forms** — `{type:"agent", id}`, `{type:"agent", id, version}`, `{type:"self"}` (coordinator spawns copies). (Anthropic.)
- **Handoffs** — `handoffs[]` list of agent IDs (Mistral, OpenAI, Vibe); `handoff_execution: server` (autonomous) or `client` (returns to user) (Mistral, Vibe); `agent.handoff` entry (Mistral); `handoff()` wrapper with metadata/filtered history (OpenAI); `handoffDescription` drives routing (OpenAI).
- **Agents-as-tools** — `agent.asTool({name, description})` / `agent.as_tool()` — manager calls specialist as bounded tool, keeping reply ownership. (OpenAI.)
- **Collaborators** — `collaborators[]` agent names; supervisor routes based on `description`; native/external/assistant. (IBM watsonx.)
- **CSV batch fan-out** — `spawn_agents_on_csv`: one worker per row; `csv_path`, `instruction` with `{column}` placeholders, `id_column`, `output_schema`, `max_concurrency`, `max_runtime_seconds`; workers call `report_agent_job_result`; exports combined CSV. (Codex.)
- **Dynamic workflows** — JS script Claude writes orchestrating many subagents at scale; `agent()` spawns one, `pipeline()` one per list item; up to 16 concurrent / 1000 total; resumable within session. (Claude.)
- **Multi-agent via Codex MCP + Agents SDK** — run `codex mcp-server`, orchestrate with OpenAI Agents SDK; PM agent creates shared requirements, coordinates hand-offs; traces capture every prompt/tool/hand-off. (Codex.)
- **Foreground/background distinction** — subagents background by default (Claude); parallel/background (Codex); single foreground chain (Mistral handoffs).
- **`SendMessage` mailbox** — direct messaging between teammates. (Claude.)
- **Task list** — shared work items pending/in-progress/completed with dependencies; file-lock-based claiming. (Claude.)

### Alternative approaches
- **Subagents with isolated context returning summary** (Claude, Codex, Bob, Anthropic) vs **handoffs transferring conversation ownership** (Mistral, OpenAI, Vibe) vs **agents-as-tools keeping manager ownership** (OpenAI) vs **collaborators routed by description** (IBM watsonx) vs **teams with direct messaging + shared task list** (Claude) vs **threads sharing sandbox** (Anthropic) vs **not covered** (Google).
- **Nesting depth-limited** (Claude depth 5, Codex `max_depth` 1) vs **unlimited chained handoffs** (Mistral).
- **Foreground/background** (Claude, Codex) vs **single foreground** (Mistral) vs `handoff_execution: server|client` (Mistral, Vibe).
- **Batch fan-out** (Codex CSV, Claude `pipeline()`) vs **not covered** (others).

### Unified API specification

```
# Subagent definition (Claude/Codex):
# File: .claude/agents/<name>.md  or  ~/.codex/agents/<name>.toml
# Fields: name, description, prompt/developer_instructions, tools, model, skills,
#         mcp_servers, sandbox_mode, maxTurns, background, isolation, permissionMode

# Programmatic (Claude):
agents: { "researcher": AgentDefinition({ description, prompt, tools, model, skills, memory, background, isolation: "worktree" }) }

# Coordinator roster (Anthropic):
multiagent: { type: "coordinator", agents: [ {type:"agent", id, version?}, {type:"self"} ] }
# Max 20 unique agents, max 1 delegation level, max 25 concurrent threads

# Handoffs (Mistral/OpenAI/Vibe):
handoffs: ["agent_id_1", "agent_id_2"]
handoff_execution: "server" | "client"    # autonomous vs returns to user

# Agents-as-tools (OpenAI):
manager_agent.as_tool(name="researcher", description="...")   # manager keeps ownership

# Collaborators (IBM watsonx):
collaborators: ["agent_name_1", "agent_name_2"]   # supervisor routes by description

# Teams (Claude — experimental):
CLAUDE_CODE_EXPERIMENTAL_AGENT_TEAMS=1
# Lead + teammates, shared task list, SendMessage mailbox
# Display: in-process | split panes (tmux/iTerm2)

# CSV batch fan-out (Codex):
spawn_agents_on_csv(
  csv_path, instruction="Process {column_name}", id_column,
  output_schema={...}, output_csv_path, max_concurrency, max_runtime_seconds
)
# Each worker calls report_agent_job_result exactly once

# Dynamic workflow (Claude):
# JS script: agent() spawns one; pipeline() one per list item
# Limits: 16 concurrent, 1000 total per run; resumable within session

# Global controls (Codex):
[agents]
max_threads = 6
max_depth = 1
job_max_runtime_seconds = 1800
interrupt_message = true
```

---

## Stage 13 — Memory & Knowledge (RAG)

**Purpose.** Give the agent persistent knowledge — semantic memory across sessions, and RAG document libraries with citations.

### Capabilities (union)
- **Memory store** — workspace-scoped collection of text documents optimized for the model, mounted as a directory inside the sandbox; agent reads/writes with file tools; every change creates an immutable memory version. (Anthropic.)
- **Memory CRUD** — create store, seed memories (`path` + `content`, no overwrite), list (`path_prefix`, `depth`), read, update (content and/or path rename), delete; limits 100 kB/memory, 2,000 memories/store. (Anthropic.)
- **Memory versions (audit)** — immutable `memver_...` on every mutation; retained 30 days; list, retrieve, redact (scrubs content, preserves audit); cannot redact current head. (Anthropic.)
- **Safe edits (optimistic concurrency)** — `precondition: {type: "content_sha256", content_sha256}`; applies only if hash matches. (Anthropic.)
- **Attach to session** — `resources[]` at session creation only; `access: read_write|read_only`; `instructions` ≤4096 chars; max 8 memory stores per session; mounts under `/mnt/memory/{slug}/`. (Anthropic.)
- **Agentic memory** — `memory_enabled` on agent; `client.memory.add_messages/search/list/retrieve/delete`; endpoints `/memories`, `/memories/search`. (IBM watsonx.)
- **CLAUDE.md / auto-memory** — project context loaded at session start (re-injected, prompt-cached); auto-memory accumulates learnings across sessions; `AgentDefinition.memory: user|project|local`. (Claude.)
- **`Memory()` sandbox capability** — store reusable workflow lessons across runs. (OpenAI.)
- **Libraries (RAG)** — persistent knowledge base of uploaded documents/web pages; queried via `document_library` tool; cited answers with numbered footnotes + Sources button. (Mistral, Vibe.)
- **Library CRUD** — create (`name`, `description`), list (`nb_documents`), delete; documents upload/list/get/status(`Running`/`Completed`)/text_content/delete. (Mistral, Vibe.)
- **Library supported formats** — PDF, Word, PowerPoint, ODT, EPUB, RTF, Excel, CSV, ODS, Numbers, PNG, JPEG, WebP, GIF, TXT, Markdown, RST, LaTeX, JSON, JSONL, XML, YAML, code files, EML, MSG. (Vibe.)
- **Library limits** — up to 100 files/upload, 100 MB/file; monthly processing allowance; no limit on number of Libraries. (Vibe.)
- **Library sharing/access** — Collaborator/Viewer/Entire organization; `libraries.accesses.list` with `org_id`/`level`/`share_with_uuid`/`share_with_type`; owner-only sharing/deletion. (Mistral, Vibe.)
- **Vibe ↔ API bridge** — same Library IDs across product and API; share with Org for API access. (Vibe.)
- **Knowledge bases (IBM watsonx)** — built-in (managed Milvus) or external (own Milvus/Elasticsearch/custom_search); `embeddings_model_name`, `chunk_size`, `chunk_overlap`, `top_k`, `extraction_strategy`; representation `auto`|`tool`; `prioritize_built_in_index`. KB CRUD + `/{kb_id}/status`. Chat-with-docs per-thread transient KB.
- **Document collections / documents / vector indices** — lower-level CRUD; `CreateVectorIndex` with embeddings model + chunking + retrieval; `VectorIndexStatus: ready|not_ready|rebuilding|error|update_pending`; refresh/rebuild/retrieve. (IBM watsonx.)
- **File search (hosted RAG)** — OpenAI hosted tool. (OpenAI.) `file_search` NOT supported on Google Antigravity.
- **Citations / references structure** — `ToolReferenceChunk` (`tool`, `title`, `url`, `source`) interleaved with `TextChunk` (Mistral, Vibe); `ReferenceChunk` with `reference_ids` (Chat Completions); references with `url`/`title`/`snippets`/`description`/`date`/`source` provided to model via tool results (Mistral); numbered footnotes + Sources button (Vibe).
- **Sandbox files as knowledge** — `workspace/` directory holds initial data; packages/files persist across interactions sharing `environment_id`. (Google.)
- **Web search as indexed knowledge** — `web_search = "cached"` OpenAI-maintained index. (Codex.)
- **Context compaction as memory management** — native ~135k token compaction. (Google.)
- **Chat Memories** — legacy memory in Chat tab. (Vibe.)

### Alternative approaches
- **First-class memory store with versioning + redaction + optimistic concurrency** (Anthropic) vs **agentic memory with SDK CRUD** (IBM watsonx) vs **filesystem CLAUDE.md + auto-memory** (Claude) vs **`Memory()` sandbox capability** (OpenAI) vs **RAG Libraries with citations** (Mistral, Vibe) vs **knowledge bases + document collections + vector indices** (IBM watsonx) vs **hosted file_search** (OpenAI) vs **not covered** (Codex, Bob, Google beyond sandbox files).

### Unified API specification

```
# Memory store (Anthropic-style):
POST /v1/memory_stores   body: { name, description }   → { id: "memstore_..." }
POST /v1/memory_stores/{id}/memories   body: { path, content }   # no overwrite; ≤100kB, ≤2000/store
GET  /v1/memory_stores/{id}/memories   query: path_prefix, depth
GET  /v1/memory_stores/{id}/memories/{mem_id}
POST /v1/memory_stores/{id}/memories/{mem_id}   # update content/path; precondition: {content_sha256}
DELETE /v1/memory_stores/{id}/memories/{mem_id}

# Memory versions (audit):
GET  /v1/memory_stores/{id}/memory_versions   query: memory_id   # newest first
GET  /v1/memory_stores/{id}/memory_versions/{vid}
POST /v1/memory_stores/{id}/memory_versions/{vid}/redact   # cannot redact current head

# Attach to session:
resources: [ { type: "memory_store", memory_store_id, access: "read_write"|"read_only", instructions? } ]
# ≤8 stores/session; mounts at /mnt/memory/{slug}/

# Agentic memory (IBM watsonx):
agent.memory_enabled = true
client.memory.add_messages(...) | search(...) | list(...) | retrieve(...) | delete(...)

# Library / RAG (Mistral/Vibe):
POST /v1/libraries   body: { name, description? }   → { library_id, generated_name, generated_description }
POST /v1/libraries/{id}/documents   body: { file: { fileName, content } }
GET  /v1/libraries/{id}/documents   → [{ name, extension, number_of_pages, summary }]
GET  /v1/libraries/{id}/documents/{doc_id}/status   → { processing_status: "Running"|"Completed" }
GET  /v1/libraries/{id}/documents/{doc_id}/text_content   → { text, signed_url }
GET  /v1/libraries/accesses/{id}   → [{ org_id, level: "Viewer"|"Editor", share_with_uuid, share_with_type }]
# Attach: tools: [ { type: "document_library", library_ids: ["..."] } ]
# Citations: ToolReferenceChunk { tool, title, url, source } interleaved with TextChunk

# Knowledge base + vector index (IBM watsonx):
POST /v1/orchestrate/knowledge-bases   body: { knowledge_base, documents[], embeddings_model_name, chunk_size, ... }
POST /v1/vector-indices   body: { name, embeddings_model_name, chunk_size, chunk_overlap, top_k, extraction_strategy }
POST /v1/vector-indices/{id}/collections   # attach collections
POST /v1/vector-indices/{id}/refresh | /rebuild
GET  /v1/vector-indices/{id}/retrieve
# representation: "auto" | "tool"; prioritize_built_in_index: bool
```

---

## Stage 14 — Workflows, Scheduled Tasks & Automation

**Purpose.** Run deterministic multi-step automations, schedule recurring autonomous runs, fan out batch work, and handle unattended approvals.

### Capabilities (union)
- **Scheduled deployment (cron)** — `POST /v1/deployments` with agent + environment + initial `user.message` + `schedule: {type:"cron", expression, timezone}`; DST-aware (spring-forward skipped, fall-back triggers twice); up to 10s jitter; ≤1,000 deployments/org. (Anthropic.)
- **Deployment runs** — per-trigger record tracking success/failure independent of session; `GET /v1/deployment_runs` filter by `deployment_id`/`has_error`; error types `environment_archived_error`/`agent_archived_error`/`session_rate_limited_error`. (Anthropic.)
- **Deployment lifecycle** — pause (suppresses triggers, running sessions continue), unpause (missed not backfilled), archive (terminal), manual run (`trigger_context.type: "manual"`). (Anthropic.)
- **Failure behavior** — rate-limit → failed run (no retry, next occurrence retries); agent archived → auto-archive; subagent archived → failed run + auto-pause. (Anthropic.)
- **Routines / scheduled tasks** — Claude Code on Anthropic-managed cloud; triggers: cron, API call, GitHub event; creation from web/Desktop/`/schedule` CLI; surfaces: CLI, Desktop, web, Slack Claude Tag; Desktop scheduled tasks run on your machine with local file/tool access. (Claude.)
- **`/loop` (in-session)** — repeat a prompt within a CLI session for quick polling. (Claude.)
- **Codex scheduled tasks (Automations)** — recurring background tasks reviewed in Scheduled (active/paused/completed); standalone or in-chat; custom cadence via RFC 5545 RRULE; Git repos choose local project or dedicated worktree; same task can run on >1 project; skills create/update scheduled tasks. (Codex.)
- **Codex unattended run approvals** — `read-only` → modifying/network/app calls fail; `workspace-write` → outside-workspace calls fail; `full access` → elevated risk; admins restrict via `requirements.toml`; scheduled tasks use `approval_policy = "never"` when org policy allows. (Codex.)
- **Workspace Agents trigger API** — `POST https://api.chatgpt.com/v1/workspace_agents/{id}/trigger`; agent ID `agtch_XXX`; `input` + optional `conversation_key` + optional `Idempotency-Key`; `202 Accepted` no body; response not retrievable via API yet. (Codex.)
- **Goal mode** — long-running task target with outcome/constraints/verification; `/goal` set; progress row pause/resume/edit/clear; parallel goals keep separate context; "Prevent sleep while running"; system notifications for input needs. (Codex.)
- **Agentic workflows (Flows)** — directed graph of nodes acting as a tool; `@flow` decorator with `name`/`display_name`/`description`/`input_schema`/`output_schema`/`initiators`/`schedulable`/`llm_model`/`agent_conversation_memory_turns_limit`; run sync or async with `callbackUrl`. (IBM watsonx.)
- **Flow nodes** — Tool, Agent, Generative prompt, Branch (conditional), Parallel branch, Foreach (iterate), Loop, Decisions, Timer, User activity (pause for input, not in callback flows), Document classifier/field extractor/text extractor (preview), Data map, Error handling/masking sensitive data. (IBM watsonx.)
- **Flow callbacks** — fire-and-forget; OpenAPI/Python/Flow/MCP; OpenAPI recommended; Flow callbacks must not contain user-activity nodes. (IBM watsonx.)
- **Langflow / wxflows** — import visually-built Langflow flows as tools; integrate watsonx flows. (IBM watsonx.)
- **`is_schedulable`** — agent or `@flow` flag; schedules created via Chat UI natural language; new approach supersedes legacy. (IBM watsonx.)
- **Vibe Workflows** — coded deterministic Studio automations published as chat-compatible assistants; invoked explicitly via `+` > Workflows (not auto-triggered); `Workflow started`/output/`Workflow failed` events; gated by account tier; developer surface `Forms and confirmations`/`Progress tracking`/`Canvas`; must return `ChatAssistantWorkflowOutput` to `Publish in Vibe`. (Vibe.)
- **Vibe Scheduled Tasks** — once/daily/weekly/monthly/yearly; runs in Work mode using Workflows infra; pre-authorize sensitive actions with `Always allow` or read-only prompts; result in sidebar with unread dot; edit/pause/delete; concurrent limit by plan. (Vibe.)
- **Dynamic workflows (Claude)** — JS script Claude writes orchestrating subagents; `agent()`/`pipeline()`; up to 16 concurrent / 1000 total; `/workflows` view with pause/resume/stop/restart/save; size guidelines. (Claude.)
- **CSV batch fan-out** — `spawn_agents_on_csv` one worker per row; `max_concurrency`; exports combined CSV. (Codex.)
- **Bob workflows** — `start_workflow(workflow_name, args?)` curated multi-step processes (PR description, code review). (Bob.)

### Alternative approaches
- **Cron deployments with DST-aware semantics + run tracking** (Anthropic) vs **Routines triggered by cron/API/GitHub events on managed cloud** (Claude) vs **Scheduled tasks with RFC 5545 RRULE + unattended approval restrictions** (Codex) vs **`is_schedulable` flag + natural-language scheduling via Chat UI** (IBM watsonx) vs **Scheduled Tasks in Work mode via Workflows infra** (Vibe) vs **external CI calling `bob -p`** (Bob) vs **not covered** (Google, Mistral, OpenAI).
- **Graph-based flows with typed nodes** (IBM watsonx `@flow`, Vibe Studio Workflows) vs **Claude-written JS dynamic workflows** (Claude) vs **cron deployments** (Anthropic) vs **Goal mode long-running targets** (Codex).
- **Batch fan-out** (Codex CSV, Claude `pipeline()`) vs **flow `Foreach`/parallel-branch nodes** (IBM watsonx).

### Unified API specification

```
# Cron deployment (Anthropic):
POST /v1/deployments
  body: { name, agent, environment_id, initial_events: [{type:"user.message",...}],
          schedule: { type: "cron", expression: "0 20 * * 5", timezone: "America/New_York" },
          vault_ids?, files?, github_repos?, memory_stores? }
  → { schedule: { upcoming_runs_at: [...] } }
POST /v1/deployments/{id}/pause | /unpause | /archive | /run   # manual run
GET  /v1/deployment_runs   query: deployment_id, has_error
# DST-aware; ≤10s jitter; ≤1000 deployments/org

# Scheduled task (Codex/Vibe):
# Cadence: once | daily | weekly | monthly | yearly  (Vibe)
#   OR RFC 5545 RRULE (Codex): RRULE:FREQ=MONTHLY;BYMONTHDAY=1;BYHOUR=9;BYMINUTE=0
# Unattended: approval_policy = "never" when org allows; else fall back to permission mode
# Admin restrictions: requirements.toml disallows approval_policy="never", constrains sandbox_modes

# Trigger API (Codex Workspace Agents):
POST /v1/workspace_agents/{id}/trigger
  headers: Authorization, Idempotency-Key?
  body: { input: string, conversation_key?: string }
  → 202 Accepted (no body)

# Goal (Codex):
POST /v1/sessions/{id}/goal   body: { outcome, constraints, verification }

# Flow (IBM watsonx):
@flow(name, display_name, description, input_schema, output_schema, initiators, schedulable, llm_model)
# Nodes: tool, agent, generative_prompt, branch, parallel_branch, foreach, loop, decisions, timer, user_activity, document_classifier, data_map, error_handling
POST /v1/orchestrate/flows/{id}/run | /run/async   # async with callbackUrl

# Vibe Workflow:
# Studio-built, must return ChatAssistantWorkflowOutput, Publish in Vibe
# Invoked explicitly via + > Workflows; events: Workflow started / output / failed

# Dynamic workflow (Claude):
# JS script: agent() / pipeline(); ≤16 concurrent, ≤1000 total; /workflows view

# CSV batch fan-out (Codex):
spawn_agents_on_csv(csv_path, instruction, id_column, output_schema, output_csv_path, max_concurrency, max_runtime_seconds)
```

---

## Stage 15 — Observability, Tracing & Evaluation

**Purpose.** Record, inspect, score, and audit agent runs.

### Capabilities (union)
- **Span events** — `span.model_request_start`/`_end` (with `model_usage`), `span.outcome_evaluation_*`. (Anthropic.)
- **OpenTelemetry export** — traces, metrics, events to OTLP backend; `[otel]` config (`environment`, `exporter: none|otlp-http|otlp-grpc`, `log_user_prompt`); event categories `codex.conversation_starts`/`api_request`/`sse_event`/`websocket_*`/`user_prompt`/`tool_decision`/`tool_result`. (Claude, Codex.)
- **Traces dashboard** — structured record of every run; spans: model call, tool call, handoff, guardrail, custom; `trace(name, metadata)` + `gen_trace_id()`. (OpenAI.)
- **Usage / cost tracking** — `ResultMessage.usage` + `total_cost_usd` + `num_turns` (Claude); `interaction.usage` with modality breakdowns + cached/thought/tool tokens (Google); `thread/tokenUsage/updated` (Codex); `usage.connector_tokens` (Mistral); `AssistantRun.usage` (IBM watsonx).
- **Chronological steps array** — every interaction returns `steps[]` of thoughts/tool calls/results/function calls/output. (Google.)
- **Event persistence / replay** — events persisted, replayable, listable, filterable (`types[]`). (Anthropic.) Memory versions as audit trail. (Anthropic.) SQLite rollouts. (Codex.)
- **Session transcripts** — JSONL on disk; `SessionStore` adapter mirrors to S3/Redis/custom. (Claude.)
- **`trace_id` propagation** — through runs. (IBM watsonx.)
- **watsonx Governance (WXG) integration** — per-environment `enable_wxg_integration`; returns `wxg_metrics_url`; monitoring setup. (IBM watsonx.)
- **Langfuse / IBM telemetry** — Developer Edition observability backends; ADK flags `-l/--with-langfuse`, `-i/--with-ibm-telemetry`. (IBM watsonx.)
- **LLM analytics** — `/v1/llm-analytics` config CRUD. (IBM watsonx.)
- **Agentic Control Plane** — UI dashboards for alerts, incidents, insights. (IBM watsonx.)
- **Evaluation & testing** — CSV upload → run → export; `POST /test_case` upload, `POST /evaluate`, `GET /evaluations`, `POST /evaluations/export`; rubric evaluations, LLM agent vulnerability testing (adversarial/red-team). (IBM watsonx.)
- **Trace grading** — open Logs > Traces, inspect, create grader, run against traces; eval run config: model, date range, tool calls filter, test criteria, "Grade all". (OpenAI.)
- **Eval progression** — traces (debugging) → trace grading (scoring) → datasets & eval runs (repeatability). (OpenAI.)
- **Auto-review as runtime evaluation** — reviewer evaluates approval requests. (Codex.)
- **Auto-review transcripts** — retained at `~/.codex/sessions`. (Codex.)
- **Feedback upload** — `feedback/upload` with classification + reason/logs + conversation id + `extraLogFiles`. (Codex.)
- **Compliance** — `clientInfo.name` identifies client for OpenAI Compliance Logs Platform. (Codex.)
- **Bobalytics** — Enterprise analytics portal: adoption rate, Bob factor, Bobcoin spend; workspace/team/user views; avoids exposing individual user data to admins. (Bob.)
- **Stored interactions viewable** — Logs page in AI Studio; deletable. (Google.)
- **Data retention** — Paid 55 days (configurable 7/14/28/55), Free 1 day. (Google.)
- **Supervision surfaces (Vibe Work)** — Todos panel, reasoning summary, tool-call transparency (which tool, inputs, outputs, status pending/succeeded/failed), Stop button. (Vibe.)
- **Attestation** — `attestation/generate` opt-in via `requestAttestation`. (Codex.)

### Alternative approaches
- **Span events on the event stream** (Anthropic) vs **OpenTelemetry export** (Claude, Codex) vs **traces dashboard with grading** (OpenAI) vs **chronological steps array** (Google) vs **trace_id + WXG + Langfuse + analytics** (IBM watsonx) vs **Bobalytics aggregate portal** (Bob) vs **supervision UI surfaces** (Vibe) vs **not covered** (Mistral).
- **Evaluation harness with CSV datasets + rubrics + red-team** (IBM watsonx, OpenAI trace grading) vs **auto-review as runtime eval** (Codex) vs **not covered** (others).

### Unified API specification

```
# Span events (Anthropic, on the event stream):
{ type: "span.model_request_start" }
{ type: "span.model_request_end", model_usage: { input_tokens, output_tokens } }
{ type: "span.outcome_evaluation_start" } | _ongoing | _end

# OpenTelemetry (Claude/Codex):
[otel]
environment = "dev"|"staging"|"prod"
exporter = "none"|"otlp-http"|"otlp-grpc"
log_user_prompt = false   # redact unless policy allows
# Categories: conversation_starts, api_request, sse_event, websocket_*, user_prompt, tool_decision, tool_result

# Traces (OpenAI):
trace(name: "my-workflow", metadata: {...})
gen_trace_id()   # link traces
# Dashboard: Logs > Traces; spans: model_call, tool_call, handoff, guardrail, custom

# Usage (union):
{ usage: { total_tokens, total_input_tokens, total_output_tokens, total_thought_tokens,
           total_tool_use_tokens, total_cached_tokens, modalities: {...}, total_cost_usd?, num_turns? } }

# Evaluation (IBM watsonx):
GET  /v1/agent/test_case/templates                    # sample CSV
POST /v1/agent/{id}/test_case                         # upload CSV
POST /v1/agent/{id}/evaluate                          # → evaluation
GET  /v1/agent/{id}/evaluations
POST /v1/agent/{id}/evaluations/export                # body: { evaluation_ids: [...] }
# Rubric evaluations, LLM agent vulnerability testing (adversarial/red-team)

# Trace grading (OpenAI):
# 1) Logs > Traces → inspect representative trace
# 2) Create grader → run against selected traces
# 3) Eval run config: model, date_range, tool_calls filter, test_criteria, grade_all

# Feedback (Codex):
POST /v1/feedback/upload   body: { classification, reason, logs, conversation_id, extraLogFiles? }

# Attestation (Codex):
POST /v1/attestation/generate   # opt-in via requestAttestation capability
```

---

## Stage 16 — Channels, Voice & Embedded Chat

**Purpose.** Deliver the agent to end users through surfaces beyond the developer API — messaging apps, phone, voice, web embeds, IDE.

### Capabilities (union)
- **Channels (delivery surfaces bound to agent + environment)** — Slack, Microsoft Teams, Twilio SMS/WhatsApp, Facebook Messenger, Genesys Bot/Audio Connector, web chat/embedded chat, phone/voice. Per-channel CRUD (create/list/update/get/delete). (IBM watsonx.)
- **Phone channel** — `/v1/channels/phone` config CRUD + numbers management (add/list/patch/delete). (IBM watsonx.)
- **Channel integration / Slack app instances** — generic app registration + Slack app-instance CRUD. (IBM watsonx.)
- **Embedded chat config** — `PUT /v1/agents/{id}/embedded-chat-config` (`layout`, `is_live`); web chat SDK with events (`pre:send`, `pre:receive`, `chat:ready`, `feedback`, `view:change`) and instance methods (`send()`, `restartConversation()`, `loadThreadById()`, `updateAuthToken()`). (IBM watsonx.)
- **Embed settings** — `/v1/embed-settings/config` CRUD + `/generate-key-pair`. (IBM watsonx.)
- **Voice configurations** — `/v1/voice-configurations` CRUD; referenced from agents via `voice_configuration_id` and environments via `voice`; `AgentIdleHandler` (pre_hold_message, hold_message, typing_enabled, typing_duration_seconds, audio_clip_id); `RealtimeAgentSettings`/`RealtimeAgentSettingsIn`. (IBM watsonx.)
- **Chat starter settings** — per-agent `starter_prompts`/`welcome_content`/`icon`; `PUT /chat-starter-settings` with `setting_type: starter_prompts|welcome_content|icon|all`. (IBM watsonx.)
- **Voice agents — two architectures** (OpenAI): (1) speech-to-speech with live audio sessions (Realtime API, natural low-latency); (2) chained voice pipeline (STT → text agent → TTS, predictable).
- **Realtime API** — `POST /v1/realtime` (WebSocket upgrade), `POST /v1/realtime/calls` (WebRTC session), `POST /v1/realtime/client_secrets` (ephemeral credentials); connection methods: WebRTC (recommended for browser), WebSocket, SIP. (OpenAI.)
- **Voice classes** — TS: `RealtimeAgent` + `RealtimeSession`; Py: `VoicePipeline` + `SingleAgentVoiceWorkflow` + `AudioInput` + `TTSModelSettings` + `VoicePipelineConfig`; `pipeline.run(audio_input)` → `result.stream()` async iterator; event `voice_stream_event_audio`. (OpenAI.)
- **Voice models** — `gpt-realtime-2.1`, `gpt-realtime-translate`, `gpt-realtime-whisper`, `gpt-4o-mini-transcribe`, `gpt-4o-mini-tts`, `gpt-audio-mini`. (OpenAI.)
- **ChatKit** — browser UI component. (OpenAI.)
- **Surfaces (Claude)** — Terminal CLI, VS Code/Cursor, JetBrains, Desktop app (visual diffs, parallel sessions, Dispatch, computer use), Web (`claude.ai/code`, `--teleport`, `--cloud`), Mobile (iOS Remote Control), Slack (Claude Tag, admin-governed, sandboxed, channel memory), Chrome (Claude in Chrome), Office add-ins (Excel/PowerPoint/Word/Outlook). Cowork agentic workspace in Desktop; Dispatch long-running background agent routing coding→Code, knowledge→Cowork.
- **Codex embedded** — `codex app-server` JSON-RPC designed for deep embedding (auth, history, approvals, streamed events); WebSocket/Unix socket transports; remote terminal UI `codex --remote wss://...`. Cloud integrations: GitHub (PRs + issues), Linear (issues + comments), Slack (channels + threads) to start cloud containers. (Codex.)
- **Bob surfaces** — IDE extension (VS Code-style) + terminal CLI (`bob`); no REST API surface. (Bob.)
- **Vibe surfaces** — web (`chat.mistral.ai` Work/Code/Chat tabs), `vibe` CLI, VS Code extension, Vibe Code Web (remote sandbox), iOS/Android. (Vibe.)
- **Google multimodal output** — `response_format` can request `[text, image, audio]`; TTS/audio generation models (`gemini-3.1-flash-tts-preview`, `lyria-3-*`). (Google.)
- **MCP elicitation** — `tool/requestUserInput` for 1–3 short questions. (Codex.)

### Alternative approaches
- **Rich channel ecosystem with per-channel CRUD + phone + embedded SDK** (IBM watsonx) vs **surfaces as front-ends to the same engine** (Claude: CLI/IDE/desktop/web/mobile/Slack/Chrome/Office) vs **voice-first with Realtime API + two architectures** (OpenAI) vs **embedded app-server protocol** (Codex) vs **IDE + CLI only** (Bob) vs **web + CLI + VS Code + mobile** (Vibe) vs **not covered** (Anthropic managed, Google beyond multimodal, Mistral).

### Unified API specification

```
# Channels (IBM watsonx):
POST /v1/agents/{agent_id}/environments/{env_id}/channels   # bind channel to (agent, env)
# Per-channel CRUD: slack, teams, twilio_sms, twilio_whatsapp, facebook_messenger,
#   genesys_bot, genesys_audio, web_chat, embedded_chat
POST /v1/channels/phone   body: { ... }
POST /v1/channels/phone/{id}/numbers   # add numbers

# Embedded chat:
PUT  /v1/agents/{id}/embedded-chat-config   body: { layout: {...}, is_live: bool }
# Web chat SDK: events pre:send/pre:receive/chat:ready/feedback/view:change
#   methods send()/restartConversation()/loadThreadById()/updateAuthToken()

# Voice config:
POST /v1/voice-configurations   body: { ... }
# Agent: voice_configuration_id; Environment: voice
# AgentIdleHandler: pre_hold_message, hold_message, typing_enabled, typing_duration_seconds, audio_clip_id
# RealtimeAgentSettings in additional_properties

# Voice agents (OpenAI):
POST /v1/realtime                  # WebSocket upgrade
POST /v1/realtime/calls            # WebRTC session
POST /v1/realtime/client_secrets   # ephemeral credentials
# TS: RealtimeAgent + RealtimeSession
# Py: VoicePipeline + SingleAgentVoiceWorkflow
#   pipeline.run(AudioInput) → result.stream() → voice_stream_event_audio

# Chat starter settings (IBM watsonx):
PUT /v1/agents/{id}/chat-starter-settings
  body: { starter_prompts: [...], welcome_content: { welcome_message, description, is_default_voice_greeting }, icon: "<svg>" }
```

---

## Stage 17 — Extensions, Plugins, Marketplaces & Interoperability

**Purpose.** Extend the platform with packaged bundles, discover reusable agents/tools, and interoperate with external agent systems.

### Capabilities (union)
- **Plugins** — shareable packages bundling skills, agents, hooks, MCP servers. (Claude `SdkPluginConfig`.) Codex plugins bundle skills/apps/MCP servers from marketplaces; `source` union: `local`/`git`/`npm`/`remote`; `plugin/list`/`read`/`install`/`uninstall`; `plugin/skill/read` reads remote plugin skill Markdown on demand; `installPolicySource: null|WORKSPACE_SETTING|IMPLICIT_CANONICAL_APP`. (Codex.)
- **Marketplaces** — `marketplace/add`/`remove`/`upgrade`; remote plugin marketplaces persisted to user config; `plugin-hints`/`plugin-dependencies` (version constraints)/`plugin-relevance` (suggest when work matches). (Claude, Codex.)
- **Plugin subagent restrictions** — `hooks`/`mcpServers`/`permissionMode` frontmatter ignored for plugin-provided agents (security). (Claude.) Plugin subagents appear under scoped names (`my-plugin:review:security`). (Claude.)
- **Apps (connectors)** — `[apps.<id>]` with `enabled`/`destructive_enabled`/`open_world_enabled`/`approvals_reviewer`/`default_tools_approval_mode` + per-tool `[apps.<id>.tools.<tool>]`; `app/list` with `isAccessible`/`isEnabled`/branding/metadata/labels; invoke with `$<app-slug>` + `mention` input item (`path: "app://<id>"`). (Codex.)
- **Catalog** — governed library of pre-built agents and tools (HR, IT, procurement, sales, productivity) for discovery/reuse. (IBM watsonx.)
- **Templates** — `create-from-template` for agents and tools; `template-status`. (IBM watsonx.)
- **External agents / interoperability**:
  - **Agent Connect Framework (ACF)** — OpenAI-compatible `/chat/completions` for plugging in external agents as collaborators. (IBM watsonx.)
  - **A2A protocol** — `GET /v1/a2a/versions` (client/server role versions); A2A agents via `provider: external_chat/A2A/0.3.0`. (IBM watsonx.)
  - **External-chat agents** — `POST /v1/agents/external-chat` with `api_url` + `auth_scheme` (`BEARER_TOKEN|API_KEY|NONE`) + `auth_config`. (IBM watsonx.)
  - **watsonx/custom assistants** — registered assistant instances usable as collaborators; custom assistants support file uploads. (IBM watsonx.)
  - **Codex-as-MCP-server** for interop with OpenAI Agents SDK and other MCP clients. (Codex.)
  - **External agent migration** — `externalAgentConfig/detect`/`import` discovers & migrates artifacts from other agents (config, skills, AGENTS.md, plugins, MCP, subagents, hooks, commands, sessions). (Codex.)
- **Branding (partners)** — allowed: "Claude Agent", "Claude" (within Agents menu), "{YourAgentName} Powered by Claude"; not permitted: "Claude Code", "Claude Cowork", Claude Code visuals. (Anthropic, Claude.)
- **Dev containers** — secure dev container (`devcontainer.secure.json`, `Dockerfile.secure`, `init-firewall.sh`). (Codex.)
- **Model gateway passthrough** — OpenAI-compatible `/gateway/model/chat/completions`/`/embeddings` making Orchestrate a drop-in OpenAI replacement. (IBM watsonx.)
- **Structured output** — agent `structured_output` (JSON schema) enforces response structure; SDK supports structured outputs. (IBM watsonx, OpenAI `outputType`, Anthropic `output_format`, Codex `--output-schema`.)
- **`custom_join_tool`** — Python tool for custom synthesis of task results. (IBM watsonx.)
- **`sync_tool_flow_interactions`** — sync user interactions from a tool flow back to the agent. (IBM watsonx.)
- **`hide_reasoning`** — show/hide reasoning trace. (IBM watsonx.)
- **Web search control** — `web_search: cached|disabled|live|indexed`; `--yolo` defaults to live. (Codex.)
- **Tool search** — `ENABLE_TOOL_SEARCH` for large tool sets. (Claude.)
- **Experimental feature flags** — `experimentalFeature/list`/`enablement/set` with `stage: beta|underDevelopment|stable|deprecated|removed`; patch runtime settings (`apps`, `plugins`). (Codex.)
- **Filesystem API (app-server v2)** — `fs/*` (readFile/writeFile/createDirectory/getMetadata/readDirectory/remove/copy/watch) on absolute paths. (Codex.)
- **Process management** — `process/spawn` (+writeStdin/resizePty/kill) outside sandbox; `command/exec` under server sandbox. (Codex.)
- **Config management (app-server)** — `config/read`/`value/write`/`batchWrite`/`configRequirements/read`. (Codex.)
- **`CodexConfig(codex_bin=...)`** — Python SDK override pinned executable. (Codex.)
- **Context variables** — platform-provided runtime values referenced in instructions. (IBM watsonx.)
- **Custom-agent metadata** — `language`, `framework`, `tool count`, `tool names`, `connection requirements`. (IBM watsonx.)
- **AGENTS.md discovery** — layered (global → project → cwd). (Codex, Google; analogues Claude CLAUDE.md, Bob AGENTS.md, Vibe `.vibe/`.)
- **Trust gating** — load project config only from trusted directories; `~/.vibe/trusted_folders.toml`; `vibe --trust` temp. (Vibe.)
- **CLI surfaces** — `ant` (Anthropic), `bob`/`bob -p`/`bob -i` (Bob), `claude`/`claude --bg`/`claude --agent` (Claude), `codex exec`/`codex app-server`/`codex mcp-server`/`codex sandbox` (Codex), `orchestrate` (IBM watsonx), `vibe`/`vibe --prompt`/`vibe --trust` (Vibe).
- **Checkpointing & rollback** — auto git snapshot + conversation + tool-call record before file-modifying ops; `/restore`. (Bob.) `enable_file_checkpointing` + `rewind_files`. (Claude.)
- **Context Mentions (`@`)** — inject file/folder/git-changes/commit/problems/terminal content. (Bob.)
- **Canvas** — reviewable, editable document surface for structured outputs. (Vibe.)
- **Projects** — scoped work area grouping related conversations. (Vibe.)
- **Custom Instructions** — persistent behavior rules across all Work conversations. (Vibe.)
- **Files (single-chat context)** — upload for a single chat (vs Library for cross-session). (Vibe.)
- **Bobcoins** — unified billing metric abstracting per-model token costs; plans Free/Pro/Pro+/Ultra. (Bob.)

### Alternative approaches
- **Plugin/marketplace system** (Claude, Codex) vs **catalog + templates** (IBM watsonx) vs **config-driven extensibility only** (Bob, Vibe, Mistral) vs **not covered** (Anthropic managed, Google, OpenAI beyond ChatKit).
- **External agent interoperability** — ACF (OpenAI-compatible `/chat/completions`) + A2A protocol (IBM watsonx) vs Codex-as-MCP-server + external agent migration (Codex) vs MCP as the only interop surface (others).
- **AGENTS.md layered discovery** (Codex, Google) vs CLAUDE.md (Claude) vs `.vibe/` trust-gated (Vibe) vs Bob hierarchical rules.

### Unified API specification

```
# Plugins / marketplaces (Claude/Codex):
POST /v1/marketplaces/add    body: { source: { type: "local"|"git"|"npm"|"remote", path|url|package, ... } }
POST /v1/marketplaces/{name}/remove | /upgrade
GET  /v1/plugins             # list discovered + state (installPolicySource)
GET  /v1/plugins/{id}        # bundled skills/apps/MCP names, shareUrl?
POST /v1/plugins/install     body: { marketplacePath | remoteMarketplaceName }
POST /v1/plugins/{id}/uninstall
POST /v1/plugins/{id}/skill/read   # read remote plugin skill Markdown on demand
# Security: hooks/mcpServers/permissionMode frontmatter ignored for plugin-provided agents

# Apps / connectors (Codex):
[apps.<id>]
enabled = true
destructive_enabled = true
open_world_enabled = true
approvals_reviewer = "user"|"auto_review"
default_tools_approval_mode = "auto"|"prompt"|"approve"
[apps.<id>.tools.<tool>]
enabled = false
approval_mode = "approve"
# Invoke: $<app-slug> + mention input item { path: "app://<id>" }

# Catalog / templates (IBM watsonx):
POST /v1/agents/create-from-template   body: { template_id, ... }
POST /v1/tools/create-from-template
GET  /v1/agents/{id}/template-status
GET  /v1/catalog                       # pre-built agents + tools

# External agents (IBM watsonx):
POST /v1/agents/external-chat   body: { api_url, auth_scheme: "BEARER_TOKEN"|"API_KEY"|"NONE", auth_config }
GET  /v1/a2a/versions           # A2A protocol versions (client|server role)
# A2A agents: provider: "external_chat/A2A/0.3.0"

# Codex-as-MCP-server (interop):
# Tool "codex":      { prompt, approval-policy, sandbox, model, cwd, ... }
# Tool "codex-reply": { prompt, threadId }

# External agent migration (Codex):
POST /v1/externalAgentConfig/detect    # discover artifacts from other agents
POST /v1/externalAgentConfig/import    # migrate config, skills, AGENTS.md, plugins, MCP, subagents, hooks, commands, sessions

# Experimental features (Codex):
GET  /v1/experimentalFeatures          # [{ name, stage: "beta"|"underDevelopment"|"stable"|"deprecated"|"removed" }]
POST /v1/experimentalFeatures/{name}/enablement   body: { enabled: bool }

# Filesystem / process / config (app-server v2 — Codex):
fs/readFile | writeFile | createDirectory | getMetadata | readDirectory | remove | copy | watch
process/spawn | writeStdin | resizePty | kill
command/exec | write | resize | terminate
config/read | value/write | batchWrite | configRequirements/read
```

---

# Part III — Quick Decision Guide

> "If you want to…" → which stage and which systems offer it.

| You want to… | Stage | Systems that offer it |
|---|---|---|
| Define a reusable agent via API | 1 | Anthropic, Google, IBM watsonx, Mistral, OpenAI |
| Define a reusable agent via file/code | 1 | Claude, Codex, Bob, Vibe |
| Version and release agents | 1 | Anthropic, IBM watsonx |
| Run in a managed cloud sandbox | 3 | Anthropic, Google, Codex (Cloud), OpenAI (Docker) |
| Run locally with OS-level sandbox | 3 | Codex, Bob, Claude, Vibe |
| Isolate sessions in git worktrees | 3 | Claude, Codex |
| Inject secrets without exposing them in the sandbox | 3, 11 | Google (egress proxy), Anthropic (vault), Codex (cloud secrets) |
| Resume / fork a conversation | 4 | Claude, Codex, Anthropic, Google, Mistral, OpenAI |
| Stream progress as typed events | 5 | Anthropic, Google, Codex, Claude, IBM watsonx, Mistral |
| Get streaming text deltas | 5 | Anthropic, Google, Codex, Claude, Mistral |
| Auto-compact context | 5 | Anthropic, Google, Claude, IBM watsonx, Codex |
| Call built-in web search / code execution | 6 | All (varies) |
| Define custom tools via JSON schema | 6 | Anthropic, Google, Mistral, IBM watsonx, Vibe |
| Define custom tools as code functions | 6 | Claude, OpenAI |
| Search over a huge tool catalog | 6 | Claude (ToolSearch) |
| Connect external tools via MCP | 7 | All except Google-limited, Mistral |
| Expose the agent itself as an MCP server | 7 | Codex |
| Load domain expertise on demand (Skills) | 8 | Anthropic, Claude, Codex, Google, Bob, Vibe |
| Require human approval before risky actions | 9 | All (varies) |
| Auto-review approvals with a reviewer agent | 9 | Codex, Claude (auto mode) |
| Run guardrails (input/output/tool) | 9 | OpenAI |
| Intercept/modify tool calls via hooks | 10 | Claude (rich), Codex (boundaries), IBM watsonx (async callbacks) |
| Store secrets in a vault with rotation | 11 | Anthropic, IBM watsonx (connections) |
| Delegate to subagents in isolated context | 12 | Claude, Codex, Bob, Anthropic |
| Hand off conversation ownership | 12 | Mistral, OpenAI, Vibe |
| Form teams with direct messaging | 12 | Claude |
| Fan out batch work over CSV | 12, 14 | Codex |
| Give the agent persistent semantic memory | 13 | Anthropic, IBM watsonx, Claude |
| RAG over uploaded documents with citations | 13 | Mistral, Vibe, IBM watsonx, OpenAI (file search) |
| Schedule recurring autonomous runs | 14 | Anthropic, Claude, Codex, IBM watsonx, Vibe |
| Build deterministic graph workflows | 14 | IBM watsonx, Vibe, Claude (dynamic workflows) |
| Export traces to OpenTelemetry | 15 | Claude, Codex |
| Evaluate agents with CSV datasets + rubrics | 15 | IBM watsonx, OpenAI |
| Deliver via Slack/Teams/SMS/phone | 16 | IBM watsonx |
| Run a voice agent | 16 | OpenAI, IBM watsonx |
| Embed chat in a web page | 16 | IBM watsonx, OpenAI (ChatKit) |
| Install plugins from a marketplace | 17 | Claude, Codex |
| Interoperate with external agents (A2A/ACF) | 17 | IBM watsonx, Codex |

---

*End of specification. This document is the union of capabilities found across the nine source studies in `./platform-studies/agents/`. Each stage's API specification is a vendor-neutral superset; native field names per system are mapped in [Section 3 — Cross-System Terminology Map](#3-cross-system-terminology-map).*
