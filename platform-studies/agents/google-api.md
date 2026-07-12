# Google Gemini API Analysis — Agent Capabilities (Managed Agents / Interactions API)

> **Docs:** `https://ai.google.dev/gemini-api/docs/managed-agents-quickstart` (and linked sub-pages)
> **Base API:** `https://generativelanguage.googleapis.com/v1beta` | **Auth:** `x-goog-api-key: $GEMINI_API_KEY`
> **SDKs:** Python (`google-genai` ≥ 2.3.0), JavaScript (`@google/genai` ≥ 2.3.0) | REST headers: `Api-Revision: 2026-05-20` (background/streaming examples)
> **Status:** Interactions API GA (June 2026); Antigravity managed agent in preview; environment compute not billed during preview.
> **Description:** Google exposes agent capabilities through the **Interactions API** — a unified, resource-oriented REST surface (`/v1beta/interactions`, `/v1beta/agents`, `/v1beta/files`) that serves both standard Gemini models and specialized **managed agents** (the Antigravity agent, Deep Research agents, and custom managed agents you create). A single `interactions.create` call provisions a Google-hosted Linux sandbox, runs an autonomous server-side agent loop (reasoning → tool use → file management → web browsing → loop), and returns an `Interaction` resource holding the final output plus a chronological `steps` array of everything the agent did. The platform is organized around two first-class resources — **Interactions** and **Managed Agents** — plus the **Environment** (sandbox), **Steps**, **Tools**, **MCP servers**, and **Skills** (`AGENTS.md` / `SKILL.md`). State is tracked through two independent dimensions — conversation history (`previous_interaction_id`) and sandbox state (`environment`/`environment_id`) — that can be mixed and matched. Long-running work uses **background execution** with polling, streaming, cancellation, and multi-turn chaining.

---

## Table of Contents

1. [Platform Overview & Core Concepts](#1-platform-overview--core-concepts)
2. [Interactions & the Agent Loop](#2-interactions--the-agent-loop)
3. [Environments & Sandboxes](#3-environments--sandboxes)
4. [Steps, Events & Streaming](#4-steps-events--streaming)
5. [Tools](#5-tools)
6. [Function Calling (Client-Side Custom Tools)](#6-function-calling-client-side-custom-tools)
7. [MCP Servers](#7-mcp-servers)
8. [Managed Agents (Custom Agents)](#8-managed-agents-custom-agents)
9. [Skills & File-Based Customization](#9-skills--file-based-customization)
10. [Background Execution & Lifecycle](#10-background-execution--lifecycle)
11. [Files API (Snapshots)](#11-files-api-snapshots)
12. [Capability Summary & Cross-Reference](#12-capability-summary--cross-reference)

---

## 1. Platform Overview & Core Concepts

### Main Concepts

The Gemini managed-agent platform is built around these core abstractions:

- **Interaction** — The central resource and unit of work. A single `interactions.create` call represents one turn (or one background task) that provisions/runs in a sandbox and returns a chronological record of execution steps plus the final output. Doubles as the server-side conversation-state record when using `previous_interaction_id`.
- **Managed Agent** — A saved, reusable agent configuration invoked by ID. Created via `agents.create` by extending the **Antigravity** base agent (or another managed agent) with instructions, tools, skills, and a `base_environment`.
- **Antigravity Agent** — The default general-purpose managed agent (`antigravity-preview-05-2026`), powered by Gemini 3.5 Flash and using the same harness as the Antigravity IDE. Reasons, executes code, manages files, and browses the web inside a Google-hosted Linux sandbox.
- **Environment** — A managed Linux sandbox where the agent executes code and persists files. Decoupled from conversation context, so it is reusable across many interactions or can be started fresh each time. Identified by `environment_id`.
- **Step** — One element of the `interaction.steps` chronological array: a thought, a server-side tool call/result, a client-side `function_call`, or the final `model_output`. In streaming mode each step is delivered as a `step.start` → `step.delta`(s) → `step.stop` cycle.
- **Tool** — A capability the agent can call. Four categories: (1) **server-side tools** executed automatically by the API (`code_execution`, `google_search`, `url_context`), (2) the **Filesystem** toolset (enabled implicitly by `environment`), (3) **client-side custom functions** (`function` type, you execute and return results), and (4) **remote MCP servers** (`mcp_server` type).
- **Sources** — Declarative mounts into a fresh sandbox via the `sources` array on an environment config: Git `repository`, Cloud Storage `gcs`, or `inline` text content, each written to a `target` path.
- **Network allowlist** — A `network` config restricting sandbox outbound traffic to listed domains, with optional header `transform` injection of credentials by an egress proxy (credentials never appear as env vars/files inside the sandbox).
- **AGENTS.md / SKILL.md** — File-based customization convention. `AGENTS.md` is auto-loaded as system instructions; skills live under `.agents/skills/<name>/SKILL.md` with YAML front-matter.
- **Background execution** — An async mode (`background: true`) where the API returns an interaction ID immediately and the task runs on the server until a terminal status; supports polling, streaming, cancellation.
- **Fork** — Creating a managed agent from (or invoking one against) an environment snapshots the base environment so every run starts clean.

### Agent Capabilities Map

| Capability | Description | Docs |
|------------|-------------|------|
| **Agents overview** | Top-level overview: Antigravity, security, pricing, limits | [agents](https://ai.google.dev/gemini-api/docs/agents) |
| **Quickstart** | First call, multi-turn, streaming, downloading files, saving a managed agent | [managed-agents-quickstart](https://ai.google.dev/gemini-api/docs/managed-agents-quickstart) |
| **Antigravity agent** | Default managed agent: tools, multimodal input, function calling, MCP, customization, background execution | [antigravity-agent](https://ai.google.dev/gemini-api/docs/antigravity-agent) |
| **Environments** | Sandbox config: sources, network/credentials, lifecycle, resources, snapshots | [agent-environment](https://ai.google.dev/gemini-api/docs/agent-environment) |
| **Building managed agents** | Inline customization, `agents.create`, fork-from-environment, invocation overrides, management | [custom-agents](https://ai.google.dev/gemini-api/docs/custom-agents) |
| **Streaming** | SSE event/step model, deltas, function-calling & thinking streams, image streams | [interactions/streaming](https://ai.google.dev/gemini-api/docs/interactions/streaming) |
| **Background execution** | Async runs, polling, streaming reconnect, cancellation/deletion, multi-turn chaining | [background-execution](https://ai.google.dev/gemini-api/docs/background-execution) |
| **Interactions API overview** | Core resource, server-side state, retention, supported models/agents | [interactions-overview](https://ai.google.dev/gemini-api/docs/interactions-overview) |

### Platform Architecture

```
Your application (Python google-genai / JS @google/genai / curl)
        │
        ▼
   interactions.create(agent="antigravity-preview-05-2026" | managed_agent_id,
                       input, environment, [previous_interaction_id], [tools], ...)
        │
        ▼
   ┌──────────────── Server-Side Agent Loop ─────────────────┐
   │  1. Provision Linux sandbox from `environment` config    │
   │  2. Run tool-use loop: plan → act (tool) → observe → loop │
   │  3. Server-side tools run automatically in sandbox       │
   │  4. Client functions pause with status "requires_action" │
   │  5. Native context compaction at ~135k tokens            │
   │  6. Return Interaction with output_text + steps[]        │
   └───────────────────────────────────────────────────────────┘
        │
        ├── Synchronous (default): returns when complete
        ├── Background (background=true): returns id immediately → poll/stream
        └── Streaming (stream=true): SSE step.start/delta/stop events
        │
        ▼
   Interaction resource
     ├── id, environment_id, status, output_text
     ├── steps[]  (thoughts, tool calls/results, function_call, model_output)
     └── usage (token counts)

   State (two independent dimensions, mix & match):
     previous_interaction_id  → conversation history (server-side state)
     environment_id            → sandbox/files (may span many interactions)
```

### Quickstart flow

The minimal end-to-end flow is: (1) `POST /v1beta/interactions` with `agent`, `input`, `environment="remote"` → returns `interaction.id` and `interaction.environment_id`; (2) to continue, `POST /v1beta/interactions` again with `previous_interaction_id` + `environment=<environment_id>` + new `input`. Files from turn 1 persist in turn 2 and the agent retains conversation context.

---

## 2. Interactions & the Agent Loop

An **Interaction** is the core resource. `interactions.create` either runs synchronously (default) or asynchronously (`background: true`). Each call to a managed agent provisions or reuses a sandbox and runs an autonomous server-side loop until the task is done.

### Interaction statuses

Background interactions transition through these states:

| Status | Description |
|--------|-------------|
| `in_progress` | Running on the server (synchronous calls also start here). |
| `requires_action` | Paused waiting for a client-side function result (function calling). |
| `completed` | Finished successfully; `output_text` available. |
| `failed` | Ended with an unrecoverable error. |
| `cancelled` | Cancelled via the cancel endpoint. |

### Creating an interaction

`interactions.create` — SDK: `client.interactions.create(...)` (Python) / `client.interactions.create({...}, { timeout })` (JS) | REST: `POST /v1beta/interactions`.

| Parameter | Type | Description |
|-----------|------|-------------|
| `agent` | string | Agent ID (`antigravity-preview-05-2026` or your managed agent ID). Mutually exclusive with `model`. |
| `model` | string | Standard Gemini model ID for model (non-agent) interactions. Mutually exclusive with `agent`. |
| `input` | string \| array of parts | The user prompt. String form, or array of parts (`{"type": "text", "text": ...}`, `{"type": "image", ...}`, `{"type": "function_result", ...}`). |
| `environment` | string \| object | `"remote"` (fresh sandbox), an `environment_id` (reuse), or a config object `{type, sources, network}`. Three forms. |
| `previous_interaction_id` | string | ID of a prior interaction to continue conversation history (server-side state). |
| `system_instruction` | string | System prompt defining behavior/persona. Override at invocation (interaction-scoped). |
| `tools` | array | Tools the agent can use (server-side tool types, custom `function`, `mcp_server`). Restricts/customizes the default set. |
| `background` | boolean | If `true`, run asynchronously; API returns immediately with an interaction ID. Requires `store=true` (default). |
| `stream` | boolean | If `true`, stream response via SSE. |
| `store` | boolean | Whether to store the interaction for later retrieval/state (default `true`). `false` is incompatible with background + `previous_interaction_id`. |
| `response_format` | array | Requested output modalities (e.g. `[{"type":"text"},{"type":"image"}]`) — model/agent interactions. |
| `generation_config` | object | Generation tuning: `thinking_summaries`, `thinking_level`, `temperature`, etc. (interaction-scoped). |
| `agent_config` | object | Agent-specific config (e.g. `{"type": "deep-research", "thinking_summaries": "auto"}`). |
| `last_event_id` | string | (On `interactions.get` streaming) resumes a stream from a given event after disconnect. |

### Interaction response fields

| Field | Description |
|-------|-------------|
| `interaction.id` | The interaction ID (store to continue). |
| `interaction.environment_id` | The sandbox ID (store to reuse the same sandbox). |
| `interaction.output_text` | The agent's final text output. |
| `interaction.steps` | Chronological array of execution steps (reasoning, tool calls, results). |
| `interaction.status` | Current status. |
| `interaction.usage` | Token counts (`total_tokens`, `total_input_tokens`, `total_output_tokens`, `total_thought_tokens`, `total_tool_use_tokens`, `total_cached_tokens`, modality breakdowns). |

### Two independent state dimensions

The API tracks conversation history and sandbox state **independently**:

| Dimension | Parameter | Carries |
|-----------|-----------|--------|
| Conversation history | `previous_interaction_id` | Prior inputs/outputs only. Interaction-scoped params (`tools`, `system_instruction`, `generation_config`) are **not** preserved — re-specify them each turn. |
| Sandbox / files | `environment` / `environment_id` | Installed packages, files, repo clones — persist across interactions sharing the environment. |

You can mix and match: pass `previous_interaction_id` with `environment="remote"` to keep conversation context but start a fresh sandbox, or reuse an `environment_id` without `previous_interaction_id` to resume a sandbox but reset the conversation.

### Automatic context compaction

In long multi-turn conversations, raw reasoning steps, tool calls, and large file contents grow toward the token limit. The Managed Agents API performs a **native context compaction step at around 135k tokens** automatically to prevent token-limit errors and maintain the agent's focus (preventing "context rot").

---

## 3. Environments & Sandboxes

An **Environment** is a managed Linux sandbox giving agents an isolated place to execute code and persist files. It is decoupled from interaction context, so reusable across interactions or fresh each time.

### The `environment` parameter — three forms

| Form | Example | When to use |
|------|---------|-------------|
| `"remote"` (string) | `environment="remote"` | Provision a fresh sandbox with default settings. |
| Environment ID (string) | `environment="env_abc123"` | Reuse an existing environment, preserving all files and state. |
| Config object | `environment={...}` | Provision a new sandbox with custom `sources` and/or `network` rules. |

### Environment config object

```json
{
  "type": "remote",
  "sources": [ ... ],
  "network": { "allowlist": [...] } | "disabled"
}
```

- `type`: `"remote"` (only value documented).
- `sources` (optional): declarative mounts.
- `network` (optional): allowlist object or `"disabled"`.

### Sources

Declarative mounts into the sandbox at creation:

| Source type | `type` value | Description | Limit |
|-------------|--------------|-------------|-------|
| Git repository | `repository` | Clones a repo from a URL into the sandbox at `target`. | 500 MB |
| Cloud Storage | `gcs` | Copies a file/directory from Cloud Storage (`gs://...`) into `target`. | 2 GB |
| Inline content | `inline` | Writes raw text content to a file at `target`. | 1 MB/file, 2 MB total |

Source object fields:
- `type` — `"repository"`, `"gcs"`, or `"inline"`.
- `source` (for `repository`/`gcs`) — git HTTPS URL or `gs://...` path.
- `content` (for `inline`) — raw text content.
- `target` — absolute path inside the sandbox. **Cannot be root (`/`)**; must be a sub-directory.

### Network config

Two forms:
1. Object: `{"allowlist": [ ...rules ]}` — only listed domains allowed (default is unrestricted).
2. String: `"disabled"` — blocks all outbound network access.

**Allowlist rule fields:**

| Field | Type | Description |
|-------|------|-------------|
| `domain` | string | Domain to match. Exact hostname, or `*` for all domains. |
| `transform` | object | Flat key-value pairs of headers to inject into matching requests (e.g. `{"Authorization": "Bearer ..."}`). |

**Wildcard semantics:** `{"domain": "*.example.com"}` matches subdomains but **not** the root `example.com` (add it separately). `{"domain": "*"}` is a catch-all permitting all other traffic without injected headers.

### Credentials injection

Credentials are injected via header transformations by an **egress proxy** into HTTP headers; they are **never exposed inside the sandbox** as environment variables or files. Headers can be unique per interaction and updated for the same environment (used to refresh/rotate tokens).

| Private source | Auth scheme |
|----------------|-------------|
| Private GitHub repo | `Basic` auth with a PAT, encoded `echo -n "x-oauth-basic:ghp_YourPATHere" \| base64` → `Authorization: Basic <BASE64>`. |
| Private GCS bucket | OAuth 2.0 Bearer token (e.g. `gcloud auth print-access-token`) → `Authorization: Bearer <TOKEN>`. |

### Environment lifecycle states

| State | Behavior |
|-------|----------|
| **Created** | Provisioned when an interaction specifies `environment: "remote"` or a config object. |
| **Active** | Running while an interaction is in progress. |
| **Idle** | Auto-snapshot and stopped after 15 minutes of inactivity. |
| **Offline** | Retained for 7 days since last active; resumable by passing its ID. |
| **Deleted** | Removed from the system (after 7 days of inactivity). |

VMs shut down after a brief period of inactivity to conserve resources; the next request restores state with a cold start.

### Resources & pre-installed software

| Resource | Value |
|----------|-------|
| CPU | 4 cores |
| Memory | 16 GB |

Pre-installed (Ubuntu base, Python 3.12, Node.js 22):

| Category | Packages |
|----------|----------|
| UNIX tools | `curl`, `wget`, `git`, `rsync`, `unzip`, `ripgrep`, `fd-find`, `gawk`, `bc`, `tree`, `which`, `lsof`, `htop`, `jq`, `iproute2`, `procps`, `gcloud CLI` |
| Python 3.12 | `numpy`, `pandas`, `requests`, `google-genai`, `beautifulsoup4`, `pyyaml`, `ast-grep-cli` |
| Node.js 22 | `create-next-app`, `create-vite`, `typescript` |

Packages installed during an interaction persist when you reuse the same `environment_id`; the agent can also install additional packages at runtime via `pip install` / `npm install`.

### Combining sources + iteration

You can declaratively mount known sources, then iterate with follow-up interactions to install packages or run setup scripts — all sharing the same `environment_id`.

### Pricing & limits

- Environment compute (CPU, memory, sandbox execution) is **not billed** during the preview.
- Token costs (model + tools) billed normally.
- **Environment Lifetime:** permanently deleted after 7 days of inactivity.
- **Max agents:** up to 1,000 managed agents.

### Antigravity interaction cost estimates

| Task category | Input tokens | Output tokens | Typical cost |
|---------------|-------------|---------------|--------------|
| Research & information synthesis | 100k–500k | 10k–40k | $0.30–$1.00 |
| Document & content generation | 100k–500k | 15k–50k | $0.30–$1.30 |
| Process & system design | 100k–400k | 10k–30k | $0.25–$0.80 |
| Data processing & analysis | 300k–3M | 30k–150k | $0.70–$3.25 |

50–70% of input tokens are typically cached. Complex workflows can accumulate 3–5M tokens (~$5) in a single interaction.

---

## 4. Steps, Events & Streaming

Set `stream: true` to incrementally stream the response using server-sent events (SSE). The Interactions API uses a **symmetric, step-based streaming model** — all content (text, tool calls, thinking) flows through a consistent `step.start` → `step.delta`(s) → `step.stop` cycle. When `stream: false`, the API returns a single `interaction` object whose `steps` array contains the fully assembled version of each cycle.

### Event flow

```
interaction.created
  step.start  → step.delta(s) → step.stop   (repeat per step)
interaction.status_update   (may appear between steps)
interaction.completed  (includes usage)
done  ([DONE])
error  (if an error occurs)
```

### Event types

| Event type | Description |
|------------|-------------|
| `interaction.created` | Sent when the interaction is first created. Contains interaction ID, model/agent, and initial `status`. |
| `interaction.status_update` | Signals an interaction-level status transition. May appear between steps. |
| `step.start` | Marks the beginning of a new step. Contains step `type` and `index` (and for function calls: `id`, `name`, empty `arguments`). |
| `step.delta` | Incremental data for the current step. `delta.type` determines its shape. |
| `step.stop` | Marks the end of a step. Contains the step `index`. |
| `interaction.completed` | Sent when the interaction is finished. Contains the final interaction object with `usage` stats. Non-streaming response omits `steps` here. |
| `error` | Sent on error. Contains an error object with `message` and `code` (e.g. `gateway_timeout`). |
| `done` | Terminal sentinel; data is `[DONE]`. |

### Step types (the `step.type` in `step.start`)

| Step type | Expected delta types | Description |
|-----------|----------------------|-------------|
| `model_output` | `text`, `image`, `audio` | The model's final response content. |
| `thought` | `thought_signature`, `thought_summary` | Chain-of-thought reasoning. `thought_summary` only present when `thinking_summaries` enabled. |
| `function_call` | `arguments_delta` | A request for the client to execute a function. Sets interaction status to `requires_action`. |
| Server-side tool steps | Varies by tool | e.g. `google_search_call`, `google_search_result`, `code_execution_call`, `code_execution_result`. |

### Delta types (the `delta.type` in `step.delta`)

| Delta type | Description |
|------------|-------------|
| `text` | Incremental text token from a `model_output` step. |
| `image` | Base64-encoded image data (`mime_type`, `data`) from a `model_output` step. |
| `thought_summary` | Thinking summary content (`{"content": {"type": "text", "text": ...}}`) from a `thought` step. |
| `thought_signature` | Encrypted representation of internal reasoning; sent as the last delta before `step.stop`. |
| `arguments_delta` | Partial JSON string for function-call arguments; must be accumulated across deltas. |
| `google_search_call` | Search query delta (`arguments.queries`) for a `google_search_call` step. |
| `google_search_result` | Result delta (`is_error`) for a `google_search_result` step. |

Each `step.delta` carries `index` to correlate it with its step, and a unique `event_id` (for resumable background streaming — see §10).

### Streaming with tools

Tool invocations appear as typed steps. For client-side `function_call`: `step.start` delivers the function `name`/`id`; `step.delta` events stream arguments as JSON strings (`arguments_delta`) which you must accumulate; the interaction ends with status `requires_action`. Server-side tools (Google Search, code execution) execute automatically, producing `google_search_call`/`google_search_result` and `code_execution_call`/`code_execution_result` steps inline.

### Streaming with thinking

`thought` steps emit `thought_summary` deltas (incremental summary text/image, only when `thinking_summaries` is enabled via `generation_config`) and a final `thought_signature` delta before `step.stop`.

### Streaming image generation

Request both `text` and `image` in `response_format` to receive interleaved text and generated images in the same stream (e.g. with `gemini-3.1-flash-image`).

### Handling unknown events

In accordance with the API's versioning policy, new event/delta types may be added over time. Handle unknown event types gracefully — log and skip rather than throwing.

---

## 5. Tools

Tools control what the agent can do. By default the Antigravity agent has `code_execution`, `google_search`, and `url_context`; **Filesystem** tools are enabled automatically when you specify `environment`. You only need to specify `tools` when customizing/restricting the defaults or adding custom functions / MCP servers.

### Available tools

| Tool | Type value | Description |
|------|-----------|-------------|
| Code Execution | `code_execution` | Run shell commands (bash, Python, Node) with stdout/stderr capture. (Server-side.) |
| Google Search | `google_search` | Search the public web. Supports `search_types` (e.g. `["web_search", "image_search"]`). (Server-side.) |
| URL Context | `url_context` | Fetch and read web pages. (Server-side.) |
| Filesystem | *(enabled via `environment`)* | Read, write, edit, search, and list files in the sandbox. No separate tool type; enabled automatically when `environment` is set. |
| Custom Functions | `function` | Define custom functions the agent requests; you execute and return results (see §6). |
| Remote MCP Server | `mcp_server` | Register an external MCP server over streamable HTTP as a tool (see §7). |

Default tools (when `tools` omitted): `code_execution`, `google_search`, `url_context`.

### Restricting tools

Pass only the tools you need to limit the agent (overrides the defaults):

```python
client.interactions.create(
  agent="antigravity-preview-05-2026",
  input="Summarize the latest AI reasoning papers.",
  environment="remote",
  tools=[{"type": "google_search"}, {"type": "url_context"}],
)
```

### Multimodal input

The Antigravity agent supports multimodal inputs: `text` and `image`. Images must be supplied as inline base64-encoded `data` (with `mime_type`):

```json
"input": [
  {"type": "text", "text": "Analyze this chart."},
  {"type": "image", "data": "<base64>", "mime_type": "image/png"}
]
```

### Antigravity limitations (tools)

The following are **not** supported with the Antigravity agent: `temperature`, `top_p`, `top_k`, `stop_sequences`, `max_output_tokens`, `file_search`, `computer_use`, `google_maps`. Passing a tool `name` that conflicts with a built-in returns `400 Bad Request`. `environment` requires `background=True` to be paired with `store=True`; `store=True` is also required for `previous_interaction_id`.

---

## 6. Function Calling (Client-Side Custom Tools)

Function calling connects the agent to external APIs/databases via custom tools the agent invokes and **you** execute. This is a multi-turn interaction: the agent requests a `function_call`, the interaction pauses with `status: "requires_action"`, you execute the function locally and send the result back in a follow-up `interactions.create` referencing `previous_interaction_id`.

### Defining a custom function

```json
{
  "type": "function",
  "name": "get_weather",
  "description": "Gets the current weather for a given location.",
  "parameters": {
    "type": "object",
    "properties": {
      "location": {"type": "string", "description": "The city and country, e.g. San Francisco, USA"}
    },
    "required": ["location"]
  }
}
```

Function tool fields: `type` (`"function"`), `name`, `description`, `parameters` (JSON schema object with `type`/`properties`/`required`).

### Two-turn function-calling flow

1. **Turn 1** — `interactions.create` with `tools=[{...defaults...}, get_weather_tool]` and the user prompt. If the agent decides to call the function, `interaction.status == "requires_action"`.
2. **Find pending calls** — iterate `interaction.steps`: collect `call_id`s from `function_result` steps (already executed), then `function_call` steps whose `id` is not in that set are pending. (Filesystem tools like `write_file` also appear as `function_call`s but execute automatically in the environment — exclude those by checking for matching `function_result`s.)
3. **Turn 2** — `interactions.create` with `previous_interaction_id=<interaction.id>`, `environment=<interaction.environment_id>`, and `input=[{"type": "function_result", "name": <fc_step.name>, "call_id": <fc_step.id>, "result": <your_result>}]`. The agent resumes and produces `output_text`.

### Pending-call detection (Python)

```python
if interaction.status == "requires_action":
    executed_calls = {s.call_id for s in interaction.steps if s.type == "function_result"}
    pending_calls = [s for s in interaction.steps
                     if s.type == "function_call" and s.id not in executed_calls]
    if pending_calls:
        fc_step = pending_calls[0]
        # execute locally, then send result back...
```

### Streaming with function calling

When streaming, accumulate `arguments_delta` deltas during `step.delta` events to reconstruct the full arguments. The `step.start` event for a `function_call` step delivers the function `name` and `id`. The stream ends with `interaction.completed` carrying `status: "requires_action"`; then send the result back via a new streaming `interactions.create` with `previous_interaction_id` and a `function_result` input part.

---

## 7. MCP Servers

Connect the Antigravity agent to external tools by registering **remote Model Context Protocol (MCP) servers** over streamable HTTP.

### MCP server tool fields (in the `tools` array)

| Field | Type | Required | Description |
|-------|------|----------|-------------|
| `type` | string | Yes | Must be `"mcp_server"`. |
| `name` | string | Yes | Unique identifier for the server. Must be strictly lowercase alphanumeric (`^[a-z0-9_-]+$`). |
| `url` | string | Yes | Endpoint URL of the remote MCP server. |
| `headers` | object | No | Custom headers (e.g. authentication) sent with requests. |
| `allowed_tools` | array | No | List of tool names allowed to be executed. If omitted, all tools are allowed. |

### Example

```python
client.interactions.create(
  agent="antigravity-preview-05-2026",
  input="What is the weather in Tokyo?",
  environment="remote",
  tools=[{
    "type": "mcp_server",
    "name": "weather",   # must be lowercase
    "url": "https://gemini-api-demos.uc.r.appspot.com/mcp"
  }]
)
```

### Limitation

**Gemini 3 does not support remote MCP** (coming soon); remote MCP is supported on the Antigravity agent / earlier models. No local (stdio) MCP servers are exposed via the Interactions API — only remote streamable HTTP.

---

## 8. Managed Agents (Custom Agents)

Managed agents let you extend the Antigravity agent with your own instructions, skills, and data. You can customize **inline** at interaction time (no registration), or save the configuration as a managed agent invoked by ID.

### Three extension mechanisms

1. **`system_instruction`** — system prompt defining behavior/persona, passed inline or via `AGENTS.md`.
2. **`AGENTS.md`** — auto-loaded from `.agents/AGENTS.md` (or `/.agents/AGENTS.md`) on startup; for long-form persona/guidelines, version-controlled.
3. **`SKILL.md` skills** — under `.agents/skills/<skill-name>/SKILL.md` (also `/.agents/skills/`), auto-discovered and registered; use YAML front-matter (`name:`, `description:`).

### Create a managed agent

`agents.create` — SDK: `client.agents.create(...)` | REST: `POST /v1beta/agents`.

**Agent definition reference:**

| Field | Type | Required | Description |
|-------|------|----------|-------------|
| `id` | string | Yes | Unique agent identifier within the Google Cloud project. Used to invoke the agent. Must not use reserved prefixes (see below). |
| `description` | string | No | Human-readable description of the agent. |
| `base_agent` | string | Yes | Base agent ID to extend (e.g. `antigravity-preview-05-2026`). |
| `system_instruction` | string | No | System prompt defining behavior and persona. |
| `tools` | array | No | Tools the agent can use. If omitted, defaults to `code_execution`, `google_search`, `url_context`. Supports `code_execution`, `google_search`, `url_context`, `mcp_server`, and custom `function` definitions. |
| `base_environment` | string \| object | No | `"remote"`, an `environment_id`, or a config object with `sources` and `network` (see §3). |

### Two ways to define `base_environment`

- **From sources** — define `sources` inline, or from GitHub / Cloud Storage. `agents.create` provisions a fresh sandbox with your files on every invocation.
- **From an existing environment (fork)** — iterate with the base agent until the environment is set up (packages installed, templates created), then `agents.create` with `base_environment=<interaction.environment_id>` to fork that environment into a managed agent.

### Reserved `id` prefixes (must NOT be used)

`antigravity-`, `veo-`, `omni-`, `lyria-`, `imagen-`, `gemma-`, `gemini-`, `google-`, `youtube-`, `android-`, `chrome-`, `pixel-`, `waze-`, `fitbit-`, `nest-`, `kaggle-`.

### Invoke a managed agent

Each invocation **forks the base environment**, so every run starts clean:

```python
client.interactions.create(
  agent="fibonacci-analyst",   # your managed agent ID
  input="Generate the first 50 prime numbers and save a PDF report.",
  environment="remote",
)
```

### Override configuration at invocation

You can override the agent's defaults per interaction without changing the stored definition:

| Override | Behavior |
|----------|----------|
| `system_instruction` | Replaces the agent's default system prompt for that run. |
| `tools` | Replaces the agent's default tool set (e.g. restrict to `code_execution` only). |
| `environment` (with `network`) | Passing an `environment` object with a new `network` config **fully replaces** previous network rules for that interaction; the base environment's sources (files, repositories) are **preserved**. Use to refresh expired tokens or rotate API keys. |

### Manage managed agents

| Operation | SDK / Endpoint | Behavior |
|-----------|----------------|----------|
| List | `client.agents.list()` · `GET /v1beta/agents` | Returns `.agents` array (`id`, `description`). |
| Get | `client.agents.get(id=...)` · `GET /v1beta/agents/{id}` | Retrieve a single agent. |
| Delete | `client.agents.delete(id=...)` · `DELETE /v1beta/agents/{id}` | Removes the configuration. **Existing environments and interactions created by the agent are NOT affected.** |

---

## 9. Skills & File-Based Customization

While you can pass configuration inline, Google recommends organizing agent files in a structured directory for management, version control, and mounting. The Antigravity runtime scans `.agents/` (and the root of the environment) for these files.

### Agent project directory structure

```
my-agent/
├── AGENTS.md          # long-form persona/guidelines (auto-loaded as system instructions)
├── skills/
│   └── <skill-name>/
│       └── SKILL.md   # skill definition (YAML front-matter + markdown body)
└── workspace/         # initial data/knowledge
```

### AGENTS.md

The agent automatically loads `.agents/AGENTS.md` (or `/.agents/AGENTS.md`) from the environment as system instructions on startup. Useful for version-controlled, long-form instructions instead of embedding in the `system_instruction` parameter.

### SKILL.md skills

Skills placed at `.agents/skills/<skill-name>/SKILL.md` (and `/.agents/skills/`) are both discovered and registered automatically by the harness. A `SKILL.md` uses YAML front-matter:

```markdown
---
name: slide-maker
description: Generates slide decks from structured content.
---
1. Read the source content.
2. Outline the deck.
3. Render each slide.
...
```

### Inline vs. file-based

- **Inline (no registration)** — pass `system_instruction` plus `environment.sources` with `inline` entries targeting `.agents/AGENTS.md` and `.agents/skills/<name>/SKILL.md`. Fastest way to build a custom agent.
- **File-based (recommended)** — mount the same files via sources; the runtime auto-discovers them on startup. Better for management and version control.

### Save and iterate workflow

Iterate inline until the configuration is right, then `agents.create` to save it as a managed agent (from sources, or by forking the iterated environment) and invoke by ID thereafter.

---

## 10. Background Execution & Lifecycle

For long-running tasks (deep research, complex reasoning, multi-step agent executions), standard HTTP requests can time out (~60s). The Interactions API provides **background execution** to run tasks asynchronously.

### Create a background interaction

Set `background: true` on `interactions.create`. The API immediately returns an interaction ID. Background execution **requires `store: true`** (the default) and is supported for standard Gemini models (e.g. `gemini-3.5-flash`, `gemini-3.1-pro-preview`) and managed agents (e.g. `antigravity-preview-05-2026`, `deep-research-*`).

### Retrieving results — polling or streaming

**Polling pattern (non-blocking):** periodically `interactions.get(id=...)` until status is terminal (`completed` / `failed` / `cancelled`):

```python
while interaction.status == "in_progress":
    time.sleep(5)
    interaction = client.interactions.get(id=interaction.id)
if interaction.status == "completed":
    print(interaction.output_text)
```

**Streaming pattern (resumable):** stream the background interaction with `interactions.get(id, stream=True)`. Each delta carries a unique `event_id`; pass it as `last_event_id` to resume from that event after a disconnect:

```python
stream = client.interactions.get(id=interaction_id, stream=True, last_event_id=last_event_id)
for event in stream:
    if event.event_id: last_event_id = event.event_id
    if event.event_type == "step.delta" and event.delta.type == "text":
        print(event.delta.text, end="", flush=True)
    if event.event_type == "interaction.completed": return
```

REST: `GET /v1beta/interactions/{id}?stream=true&last_event_id=...`.

### Cancellation & deletion

| Operation | SDK / Endpoint | Behavior |
|-----------|----------------|----------|
| Cancel | `client.interactions.cancel(id=...)` · `POST /v1beta/interactions/{id}:cancel` | Stops a running interaction; status → `cancelled`. |
| Delete | `client.interactions.delete(id=...)` · `DELETE /v1beta/interactions/{id}` | Removes interaction records from the server; subsequent GETs return `404 Not Found`. |

### Multi-turn with background execution

When a background interaction involves stateful tools (code execution in a sandbox), use the `environment_id` from the completed interaction to continue in the same environment so the agent picks up where it left off with all files/state intact. Chaining requires the prior interaction's status to be `completed` (chaining while `in_progress` returns `400 Bad Request`); then pass `previous_interaction_id` + `environment` on the next (also background) interaction.

### Constraints

- `previous_interaction_id` requires the prior interaction to be `completed` (not `in_progress`), else `400 Bad Request`.
- `background=True` requires `store=True` (default).
- `store=True` is required for `previous_interaction_id`.

### Data storage & retention

By default the API stores interactions (`store=true`) to enable server-side state, background execution, and observability.

| Tier | Retention |
|------|-----------|
| Paid Tier | 55 days (configurable in AI Studio: 7/14/28/55 days) |
| Free Tier | 1 day |

`store=false` is incompatible with background execution and prevents using `previous_interaction_id`. Stored interactions are viewable on the Logs page in Google AI Studio and deletable via `interactions.delete`.

---

## 11. Files API (Snapshots)

When the agent creates files inside the sandbox, download them using the Files API via a direct HTTP request (no SDK method yet). Returns a tar archive of the full environment snapshot.

### Download endpoint

`GET /v1beta/files/environment-{env_id}:download?alt=media` — header `x-goog-api-key`. Follow redirects (`allow_redirects=True` / `curl -L`).

```bash
curl -L -X GET "https://generativelanguage.googleapis.com/v1beta/files/environment-$ENV_ID:download?alt=media" \
  -H "x-goog-api-key: $GEMINI_API_KEY" \
  -o snapshot.tar
tar -xf snapshot.tar -C extracted_snapshot
```

The snapshot is a full tar of the environment filesystem; extract locally with `tarfile` (Python) or `tar -xf` (shell).

---

## 12. Capability Summary & Cross-Reference

| Capability | Primary resource(s) | Key endpoints / SDK calls | Core parameters |
|------------|---------------------|---------------------------|------------------|
| Interactions / agent loop | `Interaction` | `POST /v1beta/interactions` (`client.interactions.create`) | `agent`/`model`, `input`, `environment`, `previous_interaction_id`, `system_instruction`, `tools`, `background`, `stream`, `store`, `response_format`, `generation_config`, `agent_config` |
| Environments | `Environment` | inline `environment` param + `GET /v1beta/files/environment-{id}:download` | `type: "remote"`, `sources` (`repository`/`gcs`/`inline`, `source`/`content`, `target`), `network` (`allowlist`/`"disabled"`, rule `domain`/`transform`) |
| Steps & streaming | `Step` / SSE events | `stream: true` on create; `GET /v1beta/interactions/{id}?stream=true` | `event_type` (`interaction.created`/`status_update`/`step.start`/`step.delta`/`step.stop`/`interaction.completed`/`error`), `step.type`, `delta.type`, `last_event_id` |
| Tools | `tools[]` array | `interactions.create` `tools` param | `code_execution`, `google_search` (+`search_types`), `url_context`, `function`, `mcp_server` |
| Function calling | `function` tool + `function_result` input | two-turn `interactions.create` | tool `name`/`description`/`parameters`; result `type`/`name`/`call_id`/`result`; status `requires_action` |
| MCP servers | `mcp_server` tool | `tools` entry | `type`, `name` (`^[a-z0-9_-]+$`), `url`, `headers`, `allowed_tools` |
| Managed agents | `Agent` | `POST/GET/DELETE /v1beta/agents` (`client.agents.*`) | `id` (no reserved prefix), `base_agent`, `system_instruction`, `tools`, `base_environment`, `description` |
| Skills | `AGENTS.md`, `SKILL.md` | mounted via `sources` (`inline`/`repository`) or on `base_environment` | `.agents/AGENTS.md`, `.agents/skills/<name>/SKILL.md` (YAML `name`/`description`) |
| Background execution | `Interaction` (async) | `background: true`; `GET /v1beta/interactions/{id}`; `POST .../{id}:cancel`; `DELETE .../{id}` | `background`, `store`, `last_event_id`, statuses (`in_progress`/`requires_action`/`completed`/`failed`/`cancelled`) |
| Snapshots | Files API | `GET /v1beta/files/environment-{id}:download?alt=media` | `env_id`, `alt=media` |

### Supported models & agents (on the Interactions API)

| Type | Notable IDs |
|------|-------------|
| Agents | `antigravity-preview-05-2026`, `deep-research-preview-04-2026`, `deep-research-max-preview-04-2026` |
| Models | `gemini-3.5-flash`, `gemini-3.1-pro-preview`, `gemini-3.1-flash-lite`, `gemini-3-flash-preview`, `gemini-2.5-pro`, `gemini-2.5-flash`, `gemini-3-pro-image`, `gemini-3.1-flash-image`, `gemini-3.1-flash-tts-preview`, `gemma-4-31b-it`, `lyria-3-*` |

### Key design principles

1. **Single unified surface** — One `interactions.create` endpoint serves both standard Gemini models and specialized managed agents; same step/event model, same streaming, same background execution.
2. **Two independent state dimensions** — Conversation history (`previous_interaction_id`) and sandbox state (`environment`/`environment_id`) are decoupled and can be mixed and matched.
3. **Server-side agent loop** — The platform provisions the sandbox, runs server-side tools automatically, performs native context compaction (~135k tokens), and streams steps; your application owns client-side function execution and confirmations.
4. **Step-based observability** — Every interaction returns a chronological `steps` array (thoughts, tool calls/results, function calls, final output); streaming delivers the same as a symmetric `step.start`/`step.delta`/`step.stop` cycle.
5. **Credentials via egress proxy** — Secrets are injected into HTTP headers by an egress proxy and never exposed inside the sandbox as env vars or files; refreshable per interaction.
6. **Inline → saved workflow** — Customize inline at interaction time with no registration; iterate; then save as a managed agent invoked by ID, with each invocation forking a clean base environment.
7. **File-based expertise** — `AGENTS.md` (auto-loaded system instructions) and `.agents/skills/<name>/SKILL.md` (auto-discovered skills with YAML front-matter) provide a filesystem-native, version-controllable customization model.
8. **Async + resumable** — Background execution with polling, resumable streaming (`last_event_id`), cancellation, and deletion; long-running multi-turn chains via `previous_interaction_id` + `environment_id`.

### Interactions API limitations (vs. `generateContent`)

Not yet available on the Interactions API: video metadata, Batch API, automatic function calling (Python), explicit caching (implicit caching is available via `previous_interaction_id`), custom safety settings. Remote MCP not supported on Gemini 3 (coming soon). Mixing models in a multi-turn conversation requires subsequent models to support the prior models' output modalities as input.

### Rate limits / quotas

- Max **1,000 managed agents** per project.
- Environments deleted after **7 days of inactivity**; idle auto-snapshot after 15 minutes.
- Standard Gemini API quotas/tiers (free vs paid) apply to token usage; environment compute is not billed during preview.