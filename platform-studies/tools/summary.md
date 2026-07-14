# Unified Agent Tooling Specification

This document aggregates and unifies the agent/tool capabilities described in the four source studies in this directory:

- `anthropic-api.md` — Claude Messages API (`POST /v1/messages`)
- `google-api.md` — Gemini Interactions API (`POST /v1beta/interactions`)
- `mistral-api.md` — Mistral Agents & Conversations APIs (`/v1/conversations`, `/v1/agents`)
- `openai-api.md` — OpenAI Responses API (`POST /v1/responses`) and Chat Completions API

It performs the **union** of every capability and use case found across the four systems, orders them into a single **exhaustive processing pipeline**, calls out where different vendors use different names for the same concept, and notes where **alternative approaches** exist for the same step. It then presents a **synthetic, super-complete API specification** written for the end user, preceded by an approachable introduction to all the concepts.

> The spec below is a conceptual union — no single vendor implements every field. Where a concept only exists in one system it is still included so the union is complete. Vendor attributions (`Anthropic`, `Google`, `Mistral`, `OpenAI`) mark provenance.

---

## Table of Contents

1. [Introduction & Core Concepts](#1-introduction--core-concepts)
2. [Cross-System Naming Reference](#2-cross-system-naming-reference)
3. [The Exhaustive Processing Pipeline](#3-the-exhaustive-processing-pipeline)
4. [Unified API Specification](#4-unified-api-specification)
   - 4.1 [Resources & Setup Endpoints](#41-resources--setup-endpoints)
   - 4.2 [Conversation / Request Lifecycle](#42-conversation--request-lifecycle)
   - 4.3 [Tool Declarations](#43-tool-declarations)
   - 4.4 [Built-in Tools Catalog](#44-built-in-tools-catalog)
   - 4.5 [Tool Choice & Routing](#45-tool-choice--routing)
   - 4.6 [Generation & Streaming](#46-generation--streaming)
   - 4.7 [Tool Execution & Results](#47-tool-execution--results)
   - 4.8 [Programmatic Tool Calling](#48-programmatic-tool-calling)
   - 4.9 [Context Management](#49-context-management)
   - 4.10 [Safety, Approvals & Human-in-the-Loop](#410-safety-approvals--human-in-the-loop)
   - 4.11 [Output Assembly: Citations, Files, Structured Output](#411-output-assembly-citations-files-structured-output)
   - 4.12 [Usage, Pricing & Zero Data Retention](#412-usage-pricing--zero-data-retention)
5. [Appendix: Stop Reasons, Block Types, Error Codes](#5-appendix-stop-reasons-block-types-error-codes)

---

## 1. Introduction & Core Concepts

Modern AI chatbot/agent platforms let a language model do more than produce prose: the model can **call tools** that fetch fresh data, run code, control a computer, edit files, query external services, or hand work to another agent. This document describes the full surface of those capabilities, unioned across four major vendors.

### 1.1 What is a tool?

A **tool** is a callable capability the model may invoke during generation. Every platform lets you declare tools alongside a conversation and the model decides — per turn — whether to call one (or several) and with what arguments. Tools split into three execution buckets:

- **Client tools (user-defined / local).** You write the schema and run the code on your own infrastructure, then return the result to the model. Examples: a `get_weather` function, a database query, the Anthropic `bash`/`text_editor`/`computer`/`memory` tools (Anthropic publishes their schema but *you* execute them).
- **Server tools (hosted / built-in).** The vendor runs the operation on its infrastructure and returns the result inline; you never build a result block. Examples: web search, web fetch, code execution, file search, image generation, the Anthropic `advisor` and `tool_search` tools.
- **Connector / MCP tools.** Tools exposed by an external **Model Context Protocol** server (or a vendor-maintained wrapper around one). The vendor proxies the call to the external server; discovery and execution are typically server-side. (Anthropic MCP connector, Google `mcp_server` tool entry, Mistral Connectors, OpenAI `mcp` tool + Connectors.)

### 1.2 The agentic loop

Tool use is iterative. A single turn may produce one or more tool calls; the application executes client tools and returns results; the model produces a new turn that may call more tools or finish. The loop is keyed on a **stop reason / status** that tells the application: "the model wants tools executed" vs. "the model is done." Long-running server-side loops may **pause** and require the application to resend the conversation to continue.

### 1.3 Conversations, state and persistence

A **conversation** is an ordered history of entries (messages, tool calls, tool results, handoffs, thoughts). Platforms differ in where state lives:

- **Stateful** (server-side history): Google `previous_interaction_id`, OpenAI `previous_response_id`, Mistral `conversation_id` (cloud-stored by default). You send only new input; the server appends to stored history.
- **Stateless**: you pass the full history in every request (Google `store=false`, OpenAI/Mistral `store=false`, Anthropic messages are always stateless unless you replay). Some items (e.g. Google **thought signatures**, Anthropic **`tool_reference` expansions**) must be replayed verbatim to preserve model reasoning context.

### 1.4 Agents

Some platforms let you preconfigure an **Agent**: a named bundle of `model + instructions + tools + handoffs + sampler args` that every conversation can reuse (Mistral `agents`, OpenAI skills/containers, Anthropic Managed Agents). An agent is a convenience over re-declaring the same config each turn.

### 1.5 Citations & grounding

When a tool returns external content (web search, web fetch, maps, file search), the model's answer is **grounded** in that content and the platform attaches **citation annotations** to the text spans that reference each source. Displaying these to the end user is usually contractually required.

### 1.6 Context management

Long tool-driven conversations bloat the context window. Four complementary techniques exist (Anthropic names them explicitly, others implement subsets):

- **Tool search / deferred loading** — withhold most tool definitions until needed (suits 20+ tools).
- **Programmatic tool calling** — let the model run code that calls many tools in one execution, keeping intermediate results out of context.
- **Prompt caching** — cache stable prefixes (tool definitions, system prompt, history) across turns to cut token cost.
- **Context editing / compaction** — summarize or remove old `tool_result` blocks from history.

### 1.7 Safety & human-in-the-loop (HITL)

Tools that take side effects (send email, modify data, accept terms) can require **approval** before execution. Platforms surface a pending call and resume only after the user/system allows or denies it. Computer-use tools add **safety policies** that flag risky actions for confirmation.

### 1.8 Zero Data Retention (ZDR) & data residency

Some enterprise deployments require that no prompt content be stored by the vendor. Each tool declares whether it is **ZDR-eligible** (e.g. client tools where your app controls storage, basic web search, MCP). Code execution containers generally are *not* ZDR-eligible because data is retained up to 30 days.

---

## 2. Cross-System Naming Reference

The same concept is named differently across vendors. Use this table to map between them.

| Concept | Anthropic | Google | Mistral | OpenAI |
|---|---|---|---|---|
| Primary request endpoint | `POST /v1/messages` | `POST /v1beta/interactions` | `POST /v1/conversations` | `POST /v1/responses` |
| Low-level chat endpoint | (messages) | `generateContent` (legacy) | `POST /v1/chat/completions` | `POST /v1/chat/completions` |
| Tool array | `tools` | `tools` | `tools` | `tools` |
| Custom function tool | user-defined tool w/ `input_schema` | `{"type":"function","parameters":...}` | `{"type":"function","function":{...}}` | `{"type":"function",...}` / `function` |
| Tool call emitted by model | `tool_use` block | `function_call` step | `function.call` entry | `function_call` item |
| Tool result returned by app | `tool_result` block | `function_result` step | `FunctionResultEntry` | `function_call_output` item |
| Tool result id linkage | `tool_use_id` | `call_id` | `tool_call_id` | `call_id` |
| Server tool call | `server_tool_use` block | built-in tool step (e.g. `google_search_call`) | `tool.execution` entry | hosted tool call (e.g. `web_search_call`) |
| Stop reason / status | `stop_reason` | step types / interaction status | entry types | `status` on output items |
| Tool must call / optional / forbidden | `tool_choice` `auto`/`any`/`tool`/`none` | `tool_choice` `auto`/`any`/`none`/`validated` | (function calling) | `tool_choice` `auto`/`required`/specific |
| Disable parallel calls | `disable_parallel_tool_use` | — | — | `parallel_tool_calls=false` |
| Web search | `web_search` (server) | `google_search` (server) | `web_search` / `web_search_premium` (server) | `web_search` / `web_search_preview` (server) |
| URL/page fetch | `web_fetch` (server) | `url_context` (server) | — | `open_page`/`find_in_page` actions |
| Code execution (server sandbox) | `code_execution` (server) | `code_execution` (server) | `code_interpreter` (server) | `code_interpreter` (server, w/ container) |
| Shell execution | `bash` (client) | — | — | `shell` (hosted + local) / `local_shell` (legacy) |
| Computer / UI automation | `computer` (client) | `computer_use` (client) | — | `computer` (client) |
| Text / code file editing | `text_editor` (client) | — | — | `apply_patch` (client harness) |
| File search / RAG | (memory; no vector store) | `file_search` + FileSearchStore (server) | `document_library` (server) | `file_search` + vector stores (server); `retrieval` search API |
| Image generation | — | — | `image_generation` (server) | `image_generation` (server) |
| Maps / places | — | `google_maps` (server) | — | — |
| Persistent memory | `memory` (client, `/memories`) | — | — | (skills as files) |
| Advisor (model consulting model) | `advisor` (server) | — | — | — |
| Tool search / deferred loading | `tool_search_tool_regex`/`_bm25` + `defer_loading` | — | — | `tool_search` + `defer_loading` |
| Programmatic tool calling | programmatic tool calling (Python in container) | — | — | `programmatic_tool_calling` (JS V8) |
| MCP / Connectors | MCP connector | `mcp_server` tool entry | `connector` (Connectors API) | `mcp` tool + Connectors |
| Human-in-the-loop approval | (client tools only, manual) | `safety_decision: require_confirmation` + `safety_acknowledgement` | `requires_confirmation` + `tool_confirmations` | `require_approval` + `mcp_approval_request`/`response` |
| Citations (web) | `web_search_result_location` / citations | `url_citation` annotation | `tool_reference` chunk | `url_citation` annotation + `sources` |
| Citations (files) | — | `file_citation` annotation | (document_library refs) | `file_citation` / `container_file_citation` |
| Citations (maps) | — | `place_citation` annotation | — | — |
| Prompt caching | `cache_control: {"type":"ephemeral"}` | — | — | (cache preservation via tool search ordering) |
| Containers (execution sandbox) | `container` (code execution) | — | — | `/v1/containers` (shell + code interpreter) |
| Files API | Files API (code execution) | Files API + FileSearchStore | Files endpoint (download) | `/v1/files` + vector stores |
| Persistent conversation id | (stateless by default) | `previous_interaction_id` | `conversation_id` | `previous_response_id` |
| Stateless opt-out | (always) | `store=false` | `store=false` | `store=false` |
| Agent bundle | Managed Agents | — | `agents` | (skills / containers) |
| Agent-to-agent handoff | — | — | `handoffs` + `agent.handoff` | — |
| Skills (reusable instruction bundles) | — | — | — | `skills` (SKILL.md) |
| Extended thinking / reasoning | extended thinking | `thought` steps + `signature` | — | `reasoning.effort` (`low`/`high`/`xhigh`), `encrypted_content` |
| Strict / structured output | `strict: true` | `validated` tool_choice / `response_format` | `completion_args.response_format` (json_schema) | structured output / `response_format` |
| Domain filtering | `allowed_domains`/`blocked_domains` | — | — | `filters.allowed_domains`/`blocked_domains` |
| User location | `user_location` (search) | `latitude`/`longitude` (maps) | — | `user_location` (search) |
| Streaming | SSE `stream:true` | `stream=true` (alt=sse) | `.stream` | streaming (`stream:true`) |
| ZDR | per-tool ZDR-eligibility | — | — | ZDR / data residency |

---

## 3. The Exhaustive Processing Pipeline

The end-to-end flow of an agentic request, with every step any vendor performs. Stages compose; not every request exercises every stage.

### Stage 0 — Resource setup (before any conversation)
Provision long-lived resources the conversation will reference.
- **Files**: upload documents for code execution, file search, or image generation (Anthropic Files API; Google Files API + FileSearchStore; Mistral Files endpoint; OpenAI `/v1/files` + vector stores).
- **Vector stores / FileSearchStores**: create semantic-search containers, chunk + embed + index files, set embedding model, chunking strategy, attributes/metadata (Google, OpenAI; Mistral document_library).
- **Containers**: pre-create execution sandboxes with memory limits and expiry (Anthropic code-execution container; OpenAI `/v1/containers`).
- **Connectors / MCP servers**: register external MCP servers, complete OAuth flows, list/refresh discovered tools (Mistral Connectors lifecycle; OpenAI connectors; Anthropic MCP connector; Google `mcp_server`).
- **Skills**: upload + version reusable instruction bundles (OpenAI skills).
- **Agents**: create named bundles of model + instructions + tools + handoffs + completion args (Mistral agents; Anthropic Managed Agents; OpenAI skills-as-config).

### Stage 1 — Conversation / request construction
- Choose **driver**: an Agent (`agent_id`) or a bare `model` (mutually exclusive on Mistral; Google/OpenAI support both stored and inline).
- Decide **state model**: stateful (`previous_interaction_id`/`previous_response_id`/`conversation_id`) vs stateless (`store=false`, full history replayed).
- Build **input**: a string prompt, or an array of typed entries/parts (text, image, audio, document, prior tool calls/results, thought signatures, handoffs).
- Attach **tools** array (see Stage 2).
- Set **generation/sampler args**: `max_tokens`, `temperature`, `top_p`, `response_format`/structured output, reasoning effort, thinking config.
- Optional **system prompt / instructions** (note: Google `google_search` cannot combine with `system_instruction`; Anthropic `system` is separate; OpenAI `instructions`; Mistral agent `instructions`).
- Optional **guardrails** override (Mistral), **safety policies** override (Google `disabled_safety_policies`).

### Stage 2 — Tool declaration & context optimization
- Declare each tool's `name`, `description`, and argument schema (JSON Schema). Add `input_examples` (Anthropic), `strict` (Anthropic/OpenAI), `output_schema` (OpenAI programmatic).
- For large toolsets: mark tools **deferred** (`defer_loading:true` — Anthropic, OpenAI) and add a **tool search** tool so definitions load on demand. Group deferred tools in **namespaces** (OpenAI) or as `tool_reference` blocks (Anthropic).
- Place **prompt-cache breakpoints** on stable prefixes (Anthropic `cache_control` ephemeral on last tool / system / message; OpenAI preserves cache by appending discovered tools at end of window).
- Configure per-tool **allowed callers** (`allowed_callers`: `direct` / code-execution / `programmatic` — Anthropic, OpenAI) to control direct vs programmatic invocation.
- Configure per-tool **approval** requirements (`require_approval`, `requires_confirmation`, `requires_confirmation`) — see Stage 10.

### Stage 3 — Tool choice & routing
- Set `tool_choice` / `tool_choice` mode: `auto` (model decides), `any`/`required` (must call some tool), a **specific tool** (must call that one), `none` (must not call), `validated` (Google, schema-validated).
- Restrict allowed tools via `allowed_tools`/`allowed_function_names`/`include`/`exclude` lists.
- Toggle **parallel tool use** (default on for capable models; disable with `disable_parallel_tool_use` / `parallel_tool_calls=false`).
- Force a hosted tool with a typed `tool_choice` (OpenAI `{"type":"image_generation"}`).

### Stage 4 — Generation & streaming
- Model reasons (**extended thinking** / `thought` steps / `reasoning.effort`), possibly emitting **thought signatures** that must be preserved across turns (Google, OpenAI encrypted reasoning).
- If streaming: receive incremental events — text deltas, tool-input deltas (Anthropic fine-grained `input_json_delta`), `step.start`/`step.delta` (Google), partial images (OpenAI image generation).
- Model emits one or more **tool calls** (client `tool_use`/`function_call`/`function.call`/`function_call`) and/or **server tool calls** (`server_tool_use`/built-in steps/`tool.execution`/hosted tool calls). Parallel calls allowed when enabled.

### Stage 5 — Tool execution
- **Client tools**: your application executes them. For shell/computer/editor, run in an isolated sandbox, enforce allowlists, resource limits, logging. Return results in the next user message.
- **Server tools**: the vendor executes. Some run an internal **server-side agentic loop** with an iteration limit; exceeding it yields `pause_turn` (Anthropic) — resend the paused content to continue.
- **Connector/MCP tools**: vendor proxies to the external server; tool list is cached (`mcp_list_tools`) to avoid re-fetching each turn.
- **Programmatic tool calling**: the model writes code (Python in an Anthropic container, or JS in an OpenAI V8 runtime) that calls tools as functions; execution **pauses** on each tool call, the app returns the result, and code resumes — intermediate results never enter the model context.

### Stage 6 — Result return
- Format the result block: `tool_result` (Anthropic, `content` string/array + `is_error`), `function_result`/`function_call_output` (Google/OpenAI), `FunctionResultEntry` (Mistral). Link by id (`tool_use_id`/`call_id`/`tool_call_id`).
- Preserve ordering: Anthropic requires `tool_result` blocks immediately after their `tool_use` and before any text; Mistral/OpenAI require `call_id`. Google requires echoing `id` + `signature`.
- Handle errors: set `is_error:true` / send an error `output`; the model recovers. For invalid streamed JSON, return `{"INVALID_JSON": "<raw>"}`.
- For server-tool mixed turns (server + client calls in one response), send only `tool_result` blocks and keep the same `tools`; pass back the `container` id when a programmatic call is in flight (Anthropic).

### Stage 7 — Loop control
- Continue while stop reason indicates tool use (`"tool_use"` / pending `function_call` / `function.call` / `status:"in_progress"`).
- **Stop / exit reasons**: `end_turn`, `max_tokens`, `stop_sequence`, `refusal`, `pause_turn` (Anthropic); interaction completed (Google); final `message.output` (Mistral/OpenAI).
- **Handoffs**: an agent delegates the conversation to another agent (Mistral `agent.handoff`), executed server-side (`handoff_execution:"server"`) or returned to the caller (`"client"`). Unlimited chaining.
- **Pause/resume**: resend paused assistant content unchanged (with same `tools` and any pending `container` id) to continue a long server loop.

### Stage 8 — Context management (cross-cutting)
- **Prompt caching**: reuse cached tool definitions + system + stable history; automatic server-tool-result caching (Anthropic, 5-min TTL). Invalidate on tool-definition changes, `tool_choice` changes, image toggling, thinking param changes.
- **Context editing / compaction**: summarize or drop old `tool_result` blocks (Anthropic server-side context editing; SDK client-side compaction). Pairs with **memory** tool for cross-conversation state.
- **Tool search** keeps the catalog small; discovered tools are expanded inline and persist across turns without re-searching.

### Stage 9 — Output assembly
- Final assistant **message** with text (and possibly inline images, files).
- **Citations** attached as annotations on text spans: `url_citation` (web), `file_citation`/`container_file_citation` (files), `place_citation` (maps). Some also expose raw `sources` lists or `search_suggestions` widgets.
- **Files** produced by tools (generated images, code-execution outputs, fetched documents) referenced by `file_id`; download via Files API.
- **Structured output**: enforce a JSON schema via `strict`/`validated`/`response_format` so the answer (or a sub-agent's answer) conforms.

### Stage 10 — Safety, approvals & HITL (cross-cutting)
- **Tool approval gating**: pending call returned with `confirmation_status:"pending"`; resume with `allow`/`deny` per call id (Mistral `tool_confirmations`), `mcp_approval_request`/`mcp_approval_response` (OpenAI), or `safety_acknowledgement:true` (Google computer use).
- **Safety policies** (Google computer use): categories like `FINANCIAL_TRANSACTIONS`, `SENSITIVE_DATA_MODIFICATION`, `DATA_MODIFICATION`, `LEGAL_TERMS_AND_AGREEMENTS`; risky actions surface `require_confirmation`.
- **Prompt-injection detection** (Google `enable_prompt_injection_detection`); **domain filtering** to limit web access (Anthropic/OpenAI `allowed_domains`/`blocked_domains`); **URL validation** restricting fetch to URLs already in context (Anthropic `web_fetch`).
- **Path traversal protection** for memory/file tools; **allowlists** for shell; **secret injection** with placeholder names (OpenAI `domain_secrets`).
- Treat all tool/web/page content as **untrusted input**; only direct user instructions count as permission.

### Stage 11 — Observability, usage, pricing, ZDR
- **Usage accounting**: input/output tokens, cache read/write, server-tool requests (`web_search_requests`, `web_fetch_requests`, `code_execution_requests`), per-tool metering, advisor sub-inference iterations.
- **Pricing models**: per search (Google Gemini 3, Anthropic $10/1k searches, OpenAI per-query), per execution hour (Anthropic/OpenAI containers), per GB/day storage (OpenAI vector stores), free tier allowances.
- **ZDR eligibility** per tool; data-residency constraints; data retention windows (Anthropic code execution up to 30 days).
- **Rate limits** per model tier and per resource (vector store files 300 RPM, MCP tiers, code interpreter 100 RPM).

---

## 4. Unified API Specification

The following is the synthetic specification of a super-complete agent-tooling API. Field names follow a normalized convention; the **Aliases** row under each field maps to each vendor's native name. Required/optional and constraints are unioned — a real implementation picks the subset it supports.

### 4.1 Resources & Setup Endpoints

These endpoints exist outside the conversation request and provision resources referenced by `tools`.

#### 4.1.1 Files

`POST /v1/files` — upload a file. Fields: `file` (stream), `purpose` (`"assistants"`/`code_interpreter`/`file_search`/etc.), optional `filename`. Returns `{file_id, filename, ...}`.

`GET /v1/files/{id}` / `GET /v1/files/{id}/content` — retrieve metadata / bytes. `DELETE /v1/files/{id}` — delete.

- **Google**: raw `File` objects are deleted after 48 h; embeddings in FileSearchStore persist until deleted.
- **Mistral**: `client.files.download(file_id=...)` retrieves tool-generated files (images, code outputs).
- **OpenAI**: `purpose:"assistants"` for file search; container files auto-uploaded from input.

#### 4.1.2 Vector Stores / FileSearchStores

A searchable container that auto-chunks, embeds, and indexes files.

`POST /v1/vector_stores` (OpenAI) / `POST /v1beta/fileSearchStores` (Google) — create.

| Field | Type | Notes |
|---|---|---|
| `name` / `display_name` | string | Human-readable. Google store names are globally scoped. |
| `embedding_model` | string | Google: `models/gemini-embedding-001` (text) or `models/gemini-embedding-2` (multimodal). |
| `file_ids` | array | (OpenAI) initial files. |
| `expires_after` | object | (OpenAI) `{anchor, days}`. |

Operations: `GET` (list/get), `POST` (update `name`/`expires_after`), `DELETE` (with `force`).

**Add files**: OpenAI `POST /v1/vector_stores/{id}/files` (`file_id`, optional `attributes`, `chunking_strategy`) or upload stream; Google `uploadToFileSearchStore` (resumable) or `importFile` (existing file). Both return long-running operations.

**Chunking strategy** (`static`): `max_chunk_size_tokens` (OpenAI 100–4096, default 800), `chunk_overlap_tokens` (≥0, ≤ half max, default 400); Google `whiteSpaceConfig.maxTokensPerChunk` / `maxOverlapTokens`.

**Attributes / custom_metadata**: per-file key/value map (OpenAI max 16 keys, 256 chars each; Google `{key, string_value|numeric_value}` array) used for filtering.

**Documents API** (Google): `GET/DELETE /fileSearchStores/{store}/documents[/{doc}]`.

**Media download**: `GET /v1/fileSearchStores/{store}/media/{BlobId}` (Google) for cited media chunks.

**Batch operations** (OpenAI): `POST /v1/vector_stores/{id}/file_batches` (up to 500 files, shared or per-file attributes/chunking), `GET`, `cancel`.

#### 4.1.3 Containers

An execution sandbox reused across turns.

`POST /v1/containers` — create.

| Field | Type | Notes |
|---|---|---|
| `name` | string | Container name. |
| `memory_limit` | string | OpenAI `"1g"`/`"4g"`/`"16g"`/`"64g"`; Anthropic fixed 5 GiB. |
| `expires_after` | object | OpenAI `{anchor:"last_active_at", minutes}`; OpenAI code-interpreter containers expire after 20 min idle; Anthropic idle ~5 min, hard 30-day cap. |
| `skills` | array | (OpenAI) skill references/inline. |

`GET /v1/containers`, `DELETE /v1/containers/{id}`. Any operation refreshes `last_active_at`.

**Container files** (OpenAI code interpreter): create (multipart or `file_id`), list, retrieve content.

#### 4.1.4 Connectors / MCP Servers

Register an external MCP server as a reusable tool source.

`POST /v1/connectors` (Mistral) — create.

| Field | Type | Notes |
|---|---|---|
| `name` | string | Unique within workspace (Mistral ≤64 chars alnum/`_`/`-`). |
| `description` | string | optional. |
| `server` / `server_url` | string | MCP server URL (Mistral `server`; OpenAI `server_url`; Google inline `url`; Anthropic MCP connector). |
| `visibility` | enum | Mistral `private`/`shared_workspace`/`shared_org`. |
| `icon_url` | string | optional. |
| `headers` | object | Static HTTP headers (API keys). |
| `auth_data` | object | OAuth2 `{client_id, client_secret}`. |
| `system_prompt` | string | Injected when connector tools are used (Mistral). |
| `connector_id` | string | (OpenAI) one of `connector_dropbox`, `connector_gmail`, `connector_googlecalendar`, `connector_googledrive`, `connector_microsoftteams`, `connector_outlookcalendar`, `connector_outlookemail`, `connector_sharepoint`. |

Operations: `get` (by id or name), `list` (paginated, filterable), `list_tools` (paginated, `refresh` to bypass cache, `pretty` for compact schema), `update` (requires UUID), `delete` (permanent; agents referencing it lose tools).

**OAuth**: `get_auth_url` (Mistral) returns `{auth_url, ttl}`; user grants in Studio UI; tokens not programmatically passable. Configure OAuth redirect URI in your app.

**MCP approvals**: see §4.10.

**Connectors Debugger** (Mistral, preview): pre-flight diagnostic against an MCP server URL (transport detection, tool discovery, auth) with per-step report (`Likely cause`, `Suggested fix`, `Raw response`, `Copy as curl`). Non-persistent credentials.

**Secure MCP Tunnel** (OpenAI): `openai/tunnel-client` exposes private/on-prem firewalled servers without public exposure.

#### 4.1.5 Skills

`POST /v1/skills` (OpenAI) — create. Multipart: multiple `files[]` (paths under one top-level folder) **or** a single zip part. Bundle must contain exactly one `SKILL.md` (case-insensitive). Front matter: `name`, `description` (per agentskills.io).

`POST /v1/skills/{skill_id}/versions` — new version (zip upload). `POST /v1/skills/{skill_id}` — set `default_version`. Delete rules: cannot delete default version; deleting last version deletes skill; cascade.

Limits: zip ≤50 MB, ≤500 files/version, uncompressed file ≤25 MB.

#### 4.1.6 Agents

`POST /v1/agents` (Mistral `client.beta.agents.create`).

| Field | Type | Notes |
|---|---|---|
| `model` | string | Underlying chat model. |
| `name` | string | Agent name. |
| `description` | string | Short use-case description. |
| `instructions` | string | System prompt. |
| `tools` | array | Tool descriptors (built-in, `function`, `connector`). |
| `handoffs` | array | Other agent ids this agent may delegate to. |
| `completion_args` | object | Sampler args (`temperature`, `top_p`, `response_format`). |

Operations: `update`, `get`, `list`, `delete`. Guardrails set on an agent apply to all its conversations.

---

### 4.2 Conversation / Request Lifecycle

`POST /v1/responses` (OpenAI) / `POST /v1/messages` (Anthropic) / `POST /v1beta/interactions` (Google) / `POST /v1/conversations` (Mistral) — start or continue a conversation.

#### Top-level request fields

| Field | Type | Required | Notes / Aliases |
|---|---|---|---|
| `model` | string | yes* | Model id. *Mutually exclusive with `agent_id` (Mistral). |
| `agent_id` | string | no | Drive with a preconfigured Agent (Mistral). |
| `input` / `messages` / `inputs` | string \| array | yes | First user message or array of typed entries. Anthropic `messages`; Google `input`; Mistral `inputs`; OpenAI `input`. |
| `tools` | array | no | Tool declarations (§4.3, §4.4). |
| `tool_choice` | object \| string | no | §4.5. |
| `system` / `instructions` / `system_instruction` | string \| array | no | Anthropic `system`; OpenAI `instructions`; Google `system_instruction` (cannot combine with `google_search`); Mistral agent `instructions`. |
| `max_tokens` | integer | yes (Anthropic) | Output cap. |
| `temperature`, `top_p` | number | no | Sampler args (Mistral `completion_args`). |
| `response_format` | object | no | Structured output / JSON schema (Mistral/OpenAI). |
| `reasoning` / thinking | object | no | OpenAI `reasoning.effort` `low`/`high`/`xhigh`; Anthropic extended thinking; Google thinking levels; cannot combine `tool_choice:tool`+advisor with extended thinking. |
| `stream` | boolean | no | SSE streaming. Google REST `?alt=sse`. |
| `store` | boolean | no | `false` for stateless (Google/Mistral/OpenAI). |
| `previous_interaction_id` / `previous_response_id` / `conversation_id` | string | no | Stateful continuation (Google/OpenAI/Mistral). |
| `handoff_execution` | enum | no | Mistral `"server"` (default) / `"client"`. |
| `guardrails` | object | no | Mistral override. |
| `include` | array | no | OpenAI extra fields to surface (`web_search_call.results`, `file_search_call.results`, `reasoning.encrypted_content`). |
| `background` | boolean | no | OpenAI long-running async research. |
| `parallel_tool_calls` | boolean | no | OpenAI toggle. |
| `container` | string | no | Anthropic code-execution container id (required while a programmatic call waits). |

#### Entry / content block types (union)

- **User content**: `text`, `image` (`data` base64 + `mime_type`), `audio`, `document` (text/base64 source), `tool_result`/`function_call_output`/`function_result`/`FunctionResultEntry`, `tool_reference`, `search_result`, `tool_confirmations`, `mcp_approval_response`, `safety_acknowledgement`, `additional_tools`, `container_upload`, `image_generation_call` (reference for multi-turn edit).
- **Assistant content**: `text`, `tool_use`/`function_call`/`function.call`/`function_call`, `server_tool_use`/built-in tool steps/`tool.execution`/hosted tool calls, `thought` (with `signature`), `program`, `program_output`, `agent.handoff`, `apply_patch_call`, `shell_call`, `computer_call`, `image_generation_call`, `mcp_approval_request`.
- **Result/continuation items**: `tool_result`, `function_result`, `function_call_output`, `shell_call_output`, `computer_call_output`, `apply_patch_call_output`, `tool_search_output`, `program_output`, `local_shell_call_output`, `tool_confirmations`.

---

### 4.3 Tool Declarations

Each item in `tools` is one of:

#### 4.3.1 Custom function tool

| Field | Type | Required | Notes / Aliases |
|---|---|---|---|
| `type` | string | yes | `"function"` (Google/Mistral/OpenAI); Anthropic omits and uses `input_schema`. |
| `name` / `function.name` | string | yes | Anthropic regex `^[a-zA-Z0-9_-]{1,64}$`; namespace recommended (`github_list_prs`). |
| `description` | string | yes | 3–4 sentences; when to use, caveats. |
| `parameters` / `input_schema` | JSON Schema | yes | Google/Mistral/OpenAI `parameters`; Anthropic `input_schema`. `type:"object"`, `properties`, `required`, `enum`, `items`. |
| `input_examples` | array | no | Anthropic; validates against schema; invalid → 400; ~20–50 tokens/example. |
| `strict` | boolean | no | Anthropic/OpenAI; guarantees schema conformance; not supported with programmatic calling (Anthropic) or recursive `$ref`. |
| `cache_control` | object | no | Anthropic `{"type":"ephemeral"}`; cannot combine with `defer_loading`. |
| `defer_loading` | boolean | no | Anthropic/OpenAI; withhold full schema until tool search loads it. |
| `allowed_callers` | array | no | Anthropic `["direct"]`/`["code_execution_20260120"]`; OpenAI `["direct"]`/`["programmatic"]`. Not a security boundary. |
| `eager_input_streaming` | boolean | no | Anthropic; fine-grained tool-input streaming (user-defined tools only). |
| `output_schema` | JSON Schema | no | OpenAI programmatic; describes the JSON in `function_call_output.output`. |
| `requires_confirmation` | array | no | Mistral tool names needing approval. |

#### 4.3.2 Namespace (OpenAI)

`{"type":"namespace","name","description","tools":[<function defs>]}`. `defer_loading` applies to inner functions. Keep <10 functions/namespace.

#### 4.3.3 Built-in / server tool

`{"type":"<built_in_type>", ...}` — see §4.4. Each vendor has versioned `type` strings (Anthropic `web_search_20260209`, etc.).

#### 4.3.4 Connector / MCP tool

| Field | Type | Notes |
|---|---|---|
| `type` | string | `"mcp"` (OpenAI) / `"connector"` (Mistral) / `"mcp_server"` (Google) / MCP connector (Anthropic). |
| `connector_id` / `server_url` / `server` | string | OpenAI connector id or remote URL; Mistral connector name/UUID; Google `url`. |
| `server_label` | string | OpenAI label for the server in this request. |
| `server_description` | string | optional. |
| `headers` | object | Static headers. |
| `allowed_tools` | array | Allowlist of tool names to import (OpenAI/Google). |
| `require_approval` | string\|object | OpenAI `"always"`/`"never"`/`{"never":{"tool_names":[...]}}`. |
| `authorization` | string | OpenAI OAuth token; not stored; not echoed. |
| `defer_loading` | boolean | OpenAI; load defs only when needed. |
| `tool_configuration` | object | Mistral `{include\|exclude, requires_confirmation}`. |

---

### 4.4 Built-in Tools Catalog

Each tool below is the union of vendor implementations. "Side" = client (you execute) or server (vendor executes).

#### 4.4.1 Web Search  *(Anthropic server, Google server, Mistral server, OpenAI server)*

Grounds answers in real-time web content with citations.

**Tool identifiers**: Anthropic `web_search_YYYYMMDD` (`name:"web_search"`); Google `{"type":"google_search"}`; Mistral `{"type":"web_search"}` / `"web_search_premium"` (premium adds news-provider verification); OpenAI `{"type":"web_search"}` (GA) / `"web_search_preview"` (legacy, ignores new controls).

| Parameter | Type | Notes |
|---|---|---|
| `search_context_size` | enum `low`/`medium`/`high` | OpenAI; default `medium`. |
| `max_uses` | integer | Anthropic limit searches/request (factual 1–3, research 15–20 or omit); exceeding → `max_uses_exceeded`. |
| `filters` / `allowed_domains` / `blocked_domains` | array | OpenAI `filters.{allowed,blocked}_domains` (≤100 each); Anthropic bare domains w/ optional path, subdomains auto-included, wildcards only in path; mutually exclusive pair. |
| `user_location` | object | Anthropic/OpenAI `{type:"approximate", city, region, country (ISO), timezone (IANA)}`; not for deep research (OpenAI). |
| `search_content_types` | array enum `image`/`text` | OpenAI. |
| `image_settings` | object | OpenAI `{max_results, caption}`. |
| `return_token_budget` | enum `default`/`unlimited` | OpenAI GPT-5+ reasoning only; `unlimited` removes cap for long research. |
| `external_web_access` | boolean | OpenAI; `false` for offline/cache-only; ignored by preview. |
| `response_inclusion` | enum `full`/`excluded` | Anthropic `20260318`; drop consumed nested pairs. |
| `allowed_callers` | array | Anthropic; basic ZDR-eligible, `_20260209+` default code-execution caller. |
| `requires_confirmation` | array | Mistral (premium). |

**Dynamic filtering** (Anthropic `_20260209+`): Claude writes/runs code to filter results; uses internal code execution (not ZDR-eligible by default; set `allowed_callers:["direct"]` to bypass).

**Response**: search call items (`web_search_call` / `google_search_call` / `tool.execution`), results with `url`/`title`/`encrypted_content` (Anthropic — pass back verbatim or 400), `page_age`; `url_citation` annotations (Google/OpenAI) with `start_index`/`end_index`; `search_suggestions` widget HTML (Google); `sources` list incl. real-time feeds `oai-sports`/`oai-weather`/`oai-finance` (OpenAI); `tool_reference` chunks (Mistral).

**Pricing**: Anthropic $10/1k searches (failed not billed); Google per query (Gemini 3) or per prompt (≤2.5); OpenAI search action incurs tool-call cost, `open_page`/`find_in_page` do not.

**Errors**: `too_many_requests`, `invalid_tool_input`, `max_uses_exceeded`, `query_too_long`, `request_too_large`, `unavailable` (Anthropic); HTTP 200 returned on tool errors.

#### 4.4.2 URL / Page Fetch  *(Anthropic server, Google server, OpenAI actions)*

Retrieve full content from specific URLs into context.

- Anthropic `web_fetch_YYYYMMDD` (`name:"web_fetch"`): versions add dynamic filtering (`20260209`), `use_cache` bypass (`20260309`), `response_inclusion` (`20260318`). Params: `max_uses`, `allowed_domains`/`blocked_domains`, `citations:{enabled:true}` (disabled by default), `max_content_tokens`, `use_cache`. **Security**: can only fetch URLs already in conversation context; no JS-rendered sites; text/HTML/PDF only. Errors: `invalid_tool_input`, `url_too_long` (>250), `url_not_allowed`, `url_not_in_prior_context`, `url_not_accessible`, `unsupported_content_type`. No extra charge beyond tokens.
- Google `url_context`: two-step retrieval (index cache → live fetch); `url_citation` annotations; `url_context_result.status` (`"unsafe"` if moderation fails); retrieved content counts as `tool_use_input_tokens`.
- OpenAI: `open_page`/`find_in_page` actions inside `web_search_call` (reasoning models only; no extra cost).

#### 4.4.3 Code Execution  *(Anthropic server, Google server, Mistral server, OpenAI server)*

Run model-generated code in a sandboxed container; iterate on results.

- **Anthropic** `code_execution_YYYYMMDD` (`name:"code_execution"`): Bash + file ops; `20260120` adds REPL state persistence + programmatic tool calling; `20260521` adds 90s per-cell wall-clock limit. Auto-grants `bash_code_execution` + `text_editor_code_execution` sub-tools. Python 3.11, Linux x86_64, 5 GiB RAM/disk, 1 CPU, networking disabled, no package install. Pre-installed: pandas, numpy, scipy, scikit-learn, statsmodels, matplotlib, seaborn, pyarrow, openpyxl, pillow, python-pptx/-docx, pypdf, pdfplumber, sympy, mpmath, tqdm, joblib; CLI unzip/unrar/7zip/bc/rg/fd/sqlite. Container reuse via top-level `container` id; idle ~5 min, 30-day cap. Files API integration via `container_upload` blocks (beta header). Not ZDR (30-day retention). Free with web search/fetch `_20260209+`; otherwise billed by execution time (min 5 min/invocation, 1,550 free hours/org/month, $0.05/hour/container beyond).
- **Google** `{"type":"code_execution"}`: Python only; iterative `code_execution_call`/`code_execution_result` steps; bundled matplotlib (inline graph images); image code execution (Gemini 3, requires thinking). No custom library install. No extra charge (tokens only).
- **Mistral** `{"type":"code_interpreter"}`: server-side isolated container; `tool.execution.info` carries `code` + `code_output`. Conversations/Agents API only. No container config exposed.
- **OpenAI** `{"type":"code_interpreter","container":...}`: container `auto` mode (`{type:"auto", memory_limit:"1g"|"4g"|"16g"|"64g", file_ids}`) or explicit container id. `code_interpreter_call` reveals `container_id`; `container_file_citation` annotations point to generated files. Container expires after 20 min idle (data discarded). Container file endpoints (create/list/retrieve). Model knows it as "the python tool." 100 RPM/org.

#### 4.4.4 Shell  *(Anthropic client, OpenAI hosted+client)*

- **Anthropic** `bash_20250124` (`name:"bash"`, client): one long-lived bash process persists cwd/env/files across calls. Input `command` (or `restart:true`). Schema-less. No interactive/GUI apps; truncate oversized outputs yourself; no streaming. Tool def adds 244–325 input tokens. Run isolated, least-privileged, allowlist (not blocklist), `ulimit`, log/redact creds. ZDR-eligible.
- **OpenAI** `{"type":"shell","environment":{...}}`:
  - Hosted `container_auto` (fresh) or `container_reference` (reuse `container_id`).
  - Local `environment.type:"local"` — you execute `shell_call` actions, return `shell_call_output` (`stdout`/`stderr`/`outcome` `{type:"exit",exit_code}`|`{type:"timeout"}`).
  - `network_policy:{type:"allowlist", allowed_domains, domain_secrets:[{domain,name,value}]}` — secrets injected at runtime; model sees placeholder `name`. Org allow list is the superset; request further restricts.
  - Hosted runtime: Debian 12, `/mnt/data`, no TTY/sudo; preinstalled Python 3.11, Node 22.16, Java 17, PHP 8.2, Ruby 3.1, Go 1.23.
- **OpenAI legacy** `local_shell` (`codex-mini-latest` only): `local_shell_call` (`command`/`working_directory`/`env`/`timeout_ms`) + `local_shell_call_output`. Prefer `shell` with GPT-5.1.

#### 4.4.5 Computer Use  *(Anthropic client, Google client, OpenAI client)*

Model inspects screenshots and emits UI actions your harness executes.

- **Anthropic** `computer_20250124` / `computer_20251124` (`name:"computer"`, client, beta headers required): params `display_width_px`/`display_height_px`/`display_number`/`enable_zoom`. Actions: `screenshot`, `left_click`/`right_click`/`middle_click`/`double_click`/`triple_click` (`[x,y]`), `type`, `key`, `mouse_move`, `scroll`, `left_click_drag`, `left_mouse_down`/`up`, `hold_key`, `wait`; `20251124` adds `zoom` (`region [x1,y1,x2,y2]`). Schema-less. Adds 466–499 system-prompt tokens + 735 tool-def tokens + Vision screenshot pricing. macOS Retina: downscale 2× or halve coordinates. ZDR-eligible.
- **Google** `{"type":"computer_use","environment":"browser"|"mobile"|"desktop"}`: normalized coordinates 0–999 (you scale to pixels). `excluded_predefined_functions`, `enable_prompt_injection_detection`, `disabled_safety_policies`. Every action includes `intent`; risky actions carry `safety_decision:{explanation, decision:"require_confirmation"}`. Browser actions: `click`/`double_click`/`triple_click`/`middle_click`/`right_click`/`mouse_down`/`mouse_up`/`move`/`type`/`drag_and_drop`/`wait`/`press_key`/`key_down`/`key_up`/`hotkey`/`take_screenshot`/`scroll`/`go_back`/`navigate`/`go_forward`. Mobile adds `open_app`/`list_apps`/`long_press`. Desktop = Browser minus navigation. Resume with `function_result` containing text (JSON of url+action_result, `safety_acknowledgement:true` if confirmed) + image. Legacy Gemini 2.5 action set differs.
- **OpenAI** `{"type":"computer"}` (GA): emits `computer_call` with batched `actions[]`; you return `computer_call_output` with a `computer_screenshot` (`detail:"original"`, ≤10.24M px). Actions: `screenshot`, `click`/`double_click` (`button`,`x`,`y`,`keys`), `scroll` (`scrollX`,`scrollY`), `type` (`text`), `keypress` (`keys[]`), `drag` (`path` of ≥2 points), `move`, `wait`. Key names: `ENTER`,`ESC`,`TAB`,`SPACE`,`BACKSPACE`,`DELETE`,`HOME`,`END`,`PAGEUP`/`PAGEDOWN`,`UP`/`DOWN`/`LEFT`/`RIGHT`,`CTRL`,`SHIFT`,`OPTION`/`ALT`,`META`/`CMD`. Also supports custom-tool and code-execution harness shapes; `gpt-5.4` trained for code-execution harness. Migration from `computer-use-preview`: single `action` → batched `actions[]`; `truncation:"auto"` no longer needed.

#### 4.4.6 Text / Code File Editing  *(Anthropic client, OpenAI client)*

- **Anthropic** `text_editor_20250728` (`name:"str_replace_based_edit_tool"`, client, schema-less): commands `view` (`view_range`), `str_replace` (`old_str`/`new_str`), `create` (`file_text`), `insert` (`insert_line`/`insert_text`). Pairs with `bash` as the canonical coding loop.
- **OpenAI** `{"type":"apply_patch"}`: model emits `apply_patch_call` with `operation` (`create_file`/`update_file`/`delete_file`, `path`, V4A-diff `diff`). App returns `apply_patch_call_output` (`status:"completed"|"failed"`, optional `output`). You implement the patch harness (Python/TS Agents SDK helpers). Pairs with `shell` for file discovery. Validate paths, prevent traversal, return `failed` with informative `output` on errors.

#### 4.4.7 File Search / RAG  *(Google server, Mistral server, OpenAI server)*

Semantic search over an indexed knowledge base; retrieved chunks ground the answer with file citations.

- **Google** `{"type":"file_search","file_search_store_names":[...],"metadata_filter":"<AIP-160 expr>"}`: vector embeddings (`gemini-embedding-001` text / `-2` multimodal); `file_citation` annotations (`file_name`,`source`,`page_number`,`media_id`,`custom_metadata`); media download. Free at query time; pay for embeddings + tokens.
- **Mistral** `{"type":"document_library"}`: RAG over uploaded Libraries; reference-emitting (citations guide). Conversations/Agents only.
- **OpenAI** `{"type":"file_search","vector_store_ids":[...],"max_num_results":N,"filters":{...}}`: hosted; `file_citation` annotations (`file_id`,`filename`); include raw results via `include:["file_search_call.results"]`. Retrieval API (`POST /v1/vector_stores/{id}/search`): `query`, `rewrite_query`, `max_num_results` (≤50), `attribute_filter` (comparison `eq/ne/gt/gte/lt/lte/in/nin` or compound `and/or`), `ranking_options` (`ranker` auto/`default-2024-08-21`, `score_threshold`, `hybrid_search` `{embedding_weight,text_weight}`/`{rrf_embedding_weight,rrf_text_weight}`). Pricing 1 GB free then $0.10/GB/day.

#### 4.4.8 Image Generation  *(Mistral server, OpenAI server)*

- **Mistral** `{"type":"image_generation"}`: model decides; `tool_file` chunk (`file_id`/`file_name`/`file_type`); download via Files endpoint. Works on Conversations, Agents, **and** Chat Completions. No size/quality params exposed.
- **OpenAI** `{"type":"image_generation"[, partial_images:1-3]}`: options `size`, `quality` (low/medium/high/auto), `background` (transparent/opaque/auto), `format`, `compression` (0–100%), `action` (auto/generate/edit). Uses GPT Image models internally; mainline model revises prompt (`revised_prompt`). `image_generation_call` returns base64 `result`; streaming `partial_image` events; multi-turn edit via prior call id. Force with `tool_choice={"type":"image_generation"}`.

#### 4.4.9 Maps / Places  *(Google server only)*

`{"type":"google_maps","latitude":..,"longitude":..}`: grounds answers in 250M+ Google Maps places; `place_citation` annotations (`name`,`url`) must be rendered as links; attribution + legal notices required.

#### 4.4.10 Memory  *(Anthropic client only)*

`memory_20250818` (`name:"memory"`, client, schema-less): store/retrieve info across conversations in `/memories` (your handler maps to real storage). Commands `view`/`create`/`str_replace`/`insert`/`delete`/`rename` (paths within `/memories`). Auto system-prompt "MEMORY PROTOCOL" (view dir first). Pairs with context editing. Validate path traversal (`../`, `%2e%2e%2f`); cap sizes; delete stale files. ZDR-eligible.

#### 4.4.11 Advisor (model consulting model)  *(Anthropic server only, beta)*

`advisor_20260301` (`name:"advisor"`, server, beta header `advisor-tool-2026-03-01`): a faster executor model consults a higher-intelligence advisor mid-generation. Params: `model` (advisor id), `max_uses`, `max_tokens` (min 1024), `caching:{type:"ephemeral",ttl:"5m"|"1h"}`. `server_tool_use` (empty input) → `advisor_tool_result` (`advisor_result` text, or `advisor_redacted_result` encrypted). Billed at advisor rates (`usage.iterations[]`). Advisor must be ≥ Sonnet 4.6 and ≥ executor. Cannot combine with extended thinking; forcing via `tool_choice:tool` works. Resume `pause_turn` by resending paused content. Claude API + AWS only.

#### 4.4.12 Tool Search  *(Anthropic server, OpenAI server)*

Discover/load tool definitions on demand for large catalogs (>20 tools; >10k tokens of defs).

- **Anthropic**: `tool_search_tool_regex_20251119` (Claude builds Python regex, `pattern` ≤200) / `tool_search_tool_bm25_20251119` (NL `query` ≤500, BM25 ranking). `defer_loading:true` on tools to defer; ≥1 tool (usually the search tool) must stay non-deferred; never defer the search tool; keep 3–5 frequent tools non-deferred; max 10,000 deferred tools; ≤5 results/search. `server_tool_use` → `tool_search_tool_result` with `tool_reference` blocks the API auto-expands. Never return a `tool_result` for the search id. Custom client-side impl: return `tool_reference` blocks from a custom tool's `tool_result`. Not separately metered.
- **OpenAI** `{"type":"tool_search"[, execution:"client", description, parameters]}`: `defer_loading:true` on functions/MCP tools; namespaces group tools; hosted mode returns `tool_search_call` (`{paths:[...]}`) + `tool_search_output` (`tools[]`); client mode emits a `tool_search_call` you fulfill. Discovered tools appended at end of context to preserve cache. `additional_tools` input item injects tools mid-conversation. Only GPT-5.4+. Use `parallel_tool_calls=False`.

#### 4.4.13 Programmatic Tool Calling  *(Anthropic server, OpenAI server)*

Model writes code that calls tools as functions within a single execution, reducing round-trips and keeping intermediate results out of context.

- **Anthropic**: requires `code_execution_20260120`+. `allowed_callers` per tool (`["direct"]`/`["code_execution_20260120"]`/both). `caller` field on `tool_use` (`{type:"direct"}` or `{type:"code_execution_20260120", tool_id}`). Code pauses on each tool call; you return `tool_result` (string/`text` blocks only when pending); pass `container` id. Pending call times out ~4 min; 90s/cell (20260521). Constraints: no `strict:true` tools, can't force programmatic call of a specific tool, `disable_parallel_tool_use` unsupported, recursive `$ref` → 400, MCP tools can't be called programmatically. Tool results from programmatic calls don't count toward token usage. Not ZDR.
- **OpenAI** `{"type":"programmatic_tool_calling"}`: `allowed_callers` omitted/`["direct"]`/`["programmatic"]`/both. Fresh V8 runtime per program (JS + top-level await; no Node/packages/network/fs/subprocess/console/persistent JS state). `program` item (`code`,`fingerprint`), `function_call` with `caller:{type:"program",caller_id}`, `program_output` (`result`,`status`). Supports `function`/`custom`/`mcp`/`apply_patch`/shell/code_interpreter. `function_call_output` must copy `caller` unchanged. ZDR supported (no persistent container). With `store:false` replay full sequence; for stored use `previous_response_id`. Make calls idempotent; require app-level approval for high-impact actions regardless of caller.

#### 4.4.14 Handoffs (agent-to-agent)  *(Mistral only)*

Agent's `handoffs:[other_agent_id,...]` (no chaining limit). `agent.handoff` entry records it. `handoff_execution:"server"` (default) or `"client"` (returned to caller for HITL/custom orchestration). Sub-agents may use `response_format` for structured output. Independent of Connectors.

#### 4.4.15 Skills  *(OpenAI only)*

Attach reusable SKILL.md bundles to shell environments. Hosted: `{"type":"skill_reference","skill_id"[,"version"]}` (container_auto/reference). Local: `{name,description,path}` (no skill_reference). Inline: `{type:"inline",name,description,source:{type:"base64",media_type:"application/zip",data}}` to `/v1/containers`. Platform injects `name`/`description`/`path` into prompt; model decides to invoke. Skills are privileged instructions — review as untrusted input.

---

### 4.5 Tool Choice & Routing

| Mode | Behavior | Vendor |
|---|---|---|
| `auto` | Model decides whether to call any tool | all |
| `any` / `required` | Must call some tool (not a specific one) | Anthropic `any`; Google `any`; OpenAI `required` |
| `tool` / specific | Must call a named tool (`name`/typed `tool_choice`) | Anthropic `tool`; OpenAI `{"type":"<tool>"}` |
| `none` | Must not call any tool | Anthropic/Google |
| `validated` | Schema-validated function call | Google (preview) |

- **Allowed-tools restriction**: Google `allowed_tools:{mode,tools:[...]}`; OpenAI `allowed_tools`/`allowed_function_names`; Mistral `tool_configuration.include`/`exclude` (mutually exclusive).
- **Parallel toggle**: Anthropic `disable_parallel_tool_use` (inside `tool_choice`); OpenAI `parallel_tool_calls:false`. Anthropic: with `auto` → ≤1 tool; with `any`/`tool` → exactly one. Not supported with programmatic calling (Anthropic).
- **Extended thinking** supports only `auto`/`none` (Anthropic).
- **Forcing a hosted tool**: OpenAI `tool_choice={"type":"image_generation"}`; Anthropic `tool_choice:{"type":"tool","name":"advisor"}`.

---

### 4.6 Generation & Streaming

- **Reasoning/thinking**: Anthropic extended thinking; OpenAI `reasoning.effort` `low`/`high`/`xhigh` (deep research uses `high`/`xhigh`); Google `thought` steps with `signature` (must replay `id`+`signature` via `previous_interaction_id`); OpenAI stateless reasoning needs `reasoning.encrypted_content` replay.
- **Streaming events**:
  - Anthropic: `content_block_start` (`tool_use` → `input:{}`), `content_block_delta` (`input_json_delta` `partial_json`), `content_block_stop`; advisor `ping` keepalives ~30s; result blocks arrive in single `content_block_start`.
  - Google: `step.start` (`step.id`,`step.name`, partial `arguments`), `step.delta` (`arguments` `partial_arguments` or `text`), `interaction.completed`; aggregate `partial_arguments` per `event.index`, `JSON.parse` at completion.
  - OpenAI: text deltas, `response.image_generation_call.partial_image` (`partial_image_index`,`partial_image_b64`), `response.completed`.
- **Fine-grained tool-input streaming** (Anthropic `eager_input_streaming`): stream a tool's input as generated without buffering/validation; on invalid JSON return `tool_result` `is_error:true` `{"INVALID_JSON":"<raw>"}`. Legacy beta header `fine-grained-tool-streaming-2025-05-14`.
- **Deep research** (OpenAI): extended agentic investigation; `background:true` for async; best with `gpt-5.5` at `high`/`xhigh`. Responses-API search context capped at 128k tokens even when model window is larger.

---

### 4.7 Tool Execution & Results

#### Client tool flow
1. Model emits call blocks (`tool_use`/`function_call`/`function.call`/`function_call`).
2. You execute (sandboxed, allowlisted, logged).
3. Return result blocks in the next user message:
   - Anthropic `tool_result` (`tool_use_id`, `content` string|array of `text`/`image`/`document`/`search_result`/`tool_reference`, `is_error`).
   - Google `function_result` (`name`,`call_id`,`result` array of `text`/`image` blocks).
   - Mistral `FunctionResultEntry` (`tool_call_id`,`result` JSON string).
   - OpenAI `function_call_output` (`call_id`,`output` string); `shell_call_output`/`computer_call_output`/`apply_patch_call_output`/`local_shell_call_output` typed variants.

#### Ordering & linkage rules
- Anthropic: `tool_result` immediately follows its `tool_use`; in the user message, all `tool_result`s first, then any text; splitting results into separate user messages breaks parallelism.
- When pending programmatic calls wait (Anthropic), user message must contain **only** `tool_result` blocks; `content` must be string/`text` (no image/document).
- A turn calling a server tool with no result yet: user message must contain only `tool_result` blocks (text after results ends the turn → 400 naming the unresolved server tool).
- Google: echo `id` + `signature` on every step when circulating via `previous_interaction_id`.
- Missing `call_id` (OpenAI local_shell) → 400.

#### Server tool flow
- Vendor executes; result blocks (`web_search_tool_result`/`web_fetch_tool_result`/`bash_code_execution_tool_result`/`code_execution_tool_result`/`advisor_tool_result`/`tool_search_tool_result`) follow the `server_tool_use` paired by `tool_use_id`. You do **not** return a `tool_result` for server tools (except tool_search — never return one).
- Server-side agentic loop has an iteration limit; exceeding → `stop_reason:"pause_turn"` (Anthropic). Resend paused assistant content unchanged + same `tools` (+ `container` id if programmatic call in flight) to continue.

#### Mixed server + client turns (Anthropic)
- `stop_reason:"tool_use"`; content has a `server_tool_use` (no result yet) + a client `tool_use`. Run client tools; send a user message of only `tool_result`s; keep same `tools`; pass back `container` id for programmatic. `mcp_tool_use` behaves the same.

#### Errors
- `is_error:true` / error `output` string; model recovers. Server tool errors return HTTP 200 with error result blocks (`error_code`).
- Code-injection risk: sanitize tool outputs that will be interpreted/executed as code.

---

### 4.8 Programmatic Tool Calling

See §4.4.13. Summary of the union:
- Anthropic: Python in code-execution container; `allowed_callers` w/ `code_execution_20260120`; `caller` on `tool_use`; `container` id required while pending; ~4 min pending timeout; 90s/cell (20260521); results don't count toward tokens; not ZDR.
- OpenAI: JS in V8 runtime; `allowed_callers` w/ `programmatic`; `caller:{type:"program",caller_id}`; `program`/`program_output` items; `fingerprint` for replay/resume; ZDR supported; `function_call_output` copies `caller`; idempotent calls; app-level approval for high-impact actions.
- Both: intermediate results kept out of context; strong fit for fan-out/parallel, large-result filtering, agentic search; weak fit for sequential workflows or first-turn few small calls.

---

### 4.9 Context Management

- **Prompt caching** (Anthropic): `cache_control:{type:"ephemeral"}` on last tool in `tools` (caches whole tool prefix), on `system`, on messages. Hierarchy: `tools` → `system` → `messages`. Cache-write markup 25% over base; breaks even 2nd hit. Automatic server-tool-result caching (5-min TTL) when ≥1 `cache_control` marker present; tracked in `cache_creation.ephemeral_5m_input_tokens`. `cache_control`+`defer_loading` mutually exclusive. Invalidation table: tool-def change → whole cache; toggle web search/citations → system+messages; change `tool_choice`/`disable_parallel_tool_use`/images/thinking → messages cache.
- **Tool search / deferred loading** (Anthropic, OpenAI): keeps catalog small; discovered tools expanded inline and persist across turns (no re-search). OpenAI appends at end of window to preserve cache; removing a tool from `tool_search_output` breaks cache.
- **Context editing / compaction** (Anthropic): server-side context editing; SDK client-side compaction (deprecated). Pairs with `memory`.
- **Stateful vs stateless replay**: replay full history (incl. `thought`/`signature`, `tool_reference` expansions, reasoning `encrypted_content`) for stateless; use `previous_interaction_id`/`previous_response_id`/`conversation_id` for stateful.

---

### 4.10 Safety, Approvals & Human-in-the-Loop

| Mechanism | Vendor | How |
|---|---|---|
| Tool approval gating | Mistral | `requires_confirmation:[tool_names]` on `tool_configuration`; pending `function.call` w/ `confirmation_status:"pending"`; resume with `tool_confirmations:[{tool_call_id, confirmation:"allow"\|"deny"}]`; batch OK. Python SDK `RunContext`+`DeferredToolCallsException` (`dc.confirm()`/`dc.reject()`); stateless via `deferred.to_dict()`. Works for Connectors, built-ins, local functions (Python SDK). |
| MCP approval | OpenAI | `require_approval:"always"\|"never"\|{never:{tool_names}}`; `mcp_approval_request` item → respond with `mcp_approval_response:{approve,approval_request_id}` via new Response w/ `previous_response_id`. |
| Computer-use safety | Google | `safety_decision:{explanation,decision:"require_confirmation"}`; confirm via `safety_acknowledgement:true` in action result. Policies: `FINANCIAL_TRANSACTIONS`,`SENSITIVE_DATA_MODIFICATION`,`COMMUNICATION_TOOL`,`ACCOUNT_CREATION`,`DATA_MODIFICATION`,`USER_CONSENT_MANAGEMENT`,`LEGAL_TERMS_AND_AGREEMENTS`; disable via `disabled_safety_policies`. |
| Prompt-injection detection | Google | `enable_prompt_injection_detection` scans screenshot pixels. |
| Domain filtering | Anthropic/OpenAI | `allowed_domains`/`blocked_domains` (mutually exclusive; bare domains w/ optional path; subdomains auto-included; wildcards in path only; ASCII-only; ≤100 each OpenAI); request-level must be subset of org list. |
| URL validation | Anthropic | `web_fetch` only fetches URLs already in context. |
| Path traversal | Anthropic | `memory` validates paths within `/memories`; reject `../`,`..\`,`%2e%2e%2f`; canonicalize. |
| Shell sandboxing | Anthropic/OpenAI | isolated container/VM, least-privileged user, allowlist (not blocklist), `ulimit`, log/redact creds, no interactive TTY, no sudo (OpenAI hosted). |
| Secret injection | OpenAI | `domain_secrets` inject values at runtime; model sees placeholder `name`. |
| ZDR eligibility | Anthropic/OpenAI | client tools + basic web search/fetch + MCP are ZDR-eligible; code execution not (30-day retention); `allowed_callers:["direct"]` to bypass dynamic-filtering code execution. |
| Trust model | all | treat all tool/web/page content as untrusted input; only direct user instructions count as permission; keep human in the loop for high-impact actions. |

---

### 4.11 Output Assembly: Citations, Files, Structured Output

#### Citations (annotations on text spans)
- `url_citation` (`url`,`title`,`start_index`,`end_index`) — Google/OpenAI web + url_context; Anthropic `web_search_result_location` (`url`,`title`,`encrypted_index`,`cited_text` ≤150 chars; `cited_text`/`title`/`url` don't count toward tokens).
- `file_citation` (`file_id`/`file_name`,`filename`,`page_number`,`media_id`,`custom_metadata`) — Google/OpenAI; OpenAI `container_file_citation` adds `container_id`.
- `place_citation` (`name`,`url`) — Google Maps.
- `tool_reference` chunk (`tool`,`title`,`url`,`source`) — Mistral web search/document_library.
- `sources` list (may exceed citations) + real-time feeds `oai-sports`/`oai-weather`/`oai-finance` — OpenAI.
- `search_suggestions` widget HTML/CSS — Google.
- Citations must be visible/clickable to end users (contractual).

#### Files produced by tools
- Generated images (`image_generation_call.result` base64 / `tool_file` chunk → Files download).
- Code-execution outputs (`bash_code_execution_tool_result.content[].file_id` / `container_file_citation`).
- Fetched documents (text/base64 PDF).
- Download via Files API / container file endpoints / `client.files.download`.

#### Structured output
- Anthropic `strict:true` (+ `tool_choice:any` guarantees call + validation).
- Google `validated` tool_choice; `response_format`.
- Mistral `completion_args.response_format` (e.g. `json_schema` from pydantic `model_json_schema()`), set per-agent so sub-agents return structured output.
- OpenAI `response_format`/structured output; `output_schema` for programmatic tools.

---

### 4.12 Usage, Pricing & Zero Data Retention

#### Usage fields (union)
- `input_tokens`/`output_tokens`/`total_tokens`; `cache_read_input_tokens`/`cache_creation_input_tokens`/`cache_creation.ephemeral_5m_input_tokens` (Anthropic); `thoughts_tokens`/`tool_use_input_tokens`/`tool_use_input_tokens_details` (Google).
- `usage.server_tool_use.{web_search_requests, web_fetch_requests, code_execution_requests}` (Anthropic).
- `usage.iterations[]` (`{type:"message"}` executor, `{type:"advisor_message",model}` advisor) (Anthropic advisor). Top-level `usage` reflects executor only.
- Tool def token cost: Anthropic per-model tool-use system prompt (e.g. Opus 4.8: 290 tokens auto/none, 410 any/tool) + per-tool def tokens (bash 244–325, computer 735, computer-use beta 466–499).

#### Pricing models (union)
- Web search: Anthropic $10/1k searches (failed not billed); Google per query (Gemini 3) or per prompt (≤2.5); OpenAI search action billed, `open_page`/`find_in_page` not; OpenAI `web_search` `search` actions incur tool-call cost.
- Code execution: Anthropic free w/ web search/fetch `_20260209+`, else execution-time billing (min 5 min/invocation, 1,550 free hours/org/month, $0.05/hour/container beyond); files in request billed even if tool not invoked; Google/OpenAI/Mistral no extra charge (tokens only).
- Vector stores: OpenAI 1 GB free then $0.10/GB/day (use `expires_after`); Google free at query time (pay for embeddings + tokens).
- Computer use: token pricing (screenshots at Vision rates) + tool-def tokens (Anthropic).
- MCP/Connectors: pay for tokens only (no per-call fee).

#### ZDR / data residency
- Per-tool ZDR eligibility (Anthropic); MCP compatible w/ ZDR (OpenAI) but data governed by third party; code execution not ZDR (≤30-day retention).
- `store:false` alone does **not** enable ZDR (OpenAI programmatic — must be enabled for org/project).
- Secure MCP Tunnel for private servers (OpenAI).
- Data-residency constraints vary by platform/region.

#### Rate limits (examples)
- OpenAI vector store files/file_batches: 300 RPM per store; MCP: T1 200 / T2-3 1000 / T4-5 2000 RPM; file search: T1 100 / T2-3 500 / T4-5 1000 RPM; code interpreter 100 RPM/org.
- Model rate limits follow tiered limits.

---

## 5. Appendix: Stop Reasons, Block Types, Error Codes

### Stop reasons / statuses (union)
| Reason | Meaning | Vendor |
|---|---|---|
| `tool_use` / pending `function_call` / `function.call` / `status:"in_progress"` | Model wants tools executed | Anthropic/Google/Mistral/OpenAI |
| `end_turn` / interaction completed / final `message.output` | Final answer | Anthropic/Google/Mistral/OpenAI |
| `max_tokens` | Output cap hit | Anthropic |
| `stop_sequence` | Stop sequence hit | Anthropic |
| `refusal` | Refusal | Anthropic |
| `pause_turn` | Server-side loop iteration limit; resend paused content to continue | Anthropic |

### Content block types (union)
- **Assistant**: `text`, `tool_use`, `server_tool_use`, `function_call`, `function.call`, `thought` (+`signature`), `program`, `program_output`, `agent.handoff`, `apply_patch_call`, `shell_call`, `computer_call`, `image_generation_call`, `mcp_approval_request`, hosted tool calls (`web_search_call`,`google_search_call`,`code_execution_call`,`file_search_call`,`code_interpreter_call`).
- **User**: `text`, `image`, `audio`, `document`, `tool_result`, `function_result`, `function_call_output`, `FunctionResultEntry`, `tool_reference`, `search_result`, `tool_confirmations`, `mcp_approval_response`, `safety_acknowledgement`, `additional_tools`, `container_upload`, `shell_call_output`, `computer_call_output`, `apply_patch_call_output`, `local_shell_call_output`, `tool_search_output`, `image_generation_call` (ref for edit).
- **Server tool results**: `web_search_tool_result`, `web_fetch_tool_result`, `bash_code_execution_tool_result`, `text_editor_code_execution_*_result`, `code_execution_tool_result`, `advisor_tool_result`, `tool_search_tool_result`.

### `tool_use`/call block fields (union)
`type`, `id` (Anthropic `toolu_...` client / `srvtoolu_...` server; Google/OpenAI `call_...`; Mistral `tool_call_id`), `name`, `input`/`arguments` (object or JSON string), `caller?` (`{type:"direct"}` or `{type:"code_execution_20260120",tool_id}` / `{type:"program",caller_id}`), `confirmation_status?` (`"pending"`), `safety_decision?`, `signature?`.

### `tool_result`/output block fields (union)
`type`, `tool_use_id`/`call_id`/`tool_call_id`, `content`/`output`/`result` (string or array), `is_error?`, `status?` (`"completed"|"failed"`), `outcome?` (`{type:"exit",exit_code}`|`{type:"timeout"}`), `safety_acknowledgement?`.

### Common error codes (server tools)
`too_many_requests`, `invalid_tool_input`, `max_uses_exceeded`, `query_too_long`, `request_too_large`, `unavailable` (web search); `invalid_tool_input`, `url_too_long`, `url_not_allowed`, `url_not_in_prior_context`, `url_not_accessible`, `unsupported_content_type` (web fetch); `unavailable`, `execution_time_exceeded`, `too_many_requests`, `output_file_too_large`, `file_not_found` (code execution/text editor); `max_uses_exceeded`, `too_many_requests`, `overloaded`, `prompt_too_long`, `execution_time_exceeded`, `unavailable` (advisor); `invalid_tool_input`, `unavailable`, `too_many_requests`, `execution_time_exceeded` (tool search). HTTP 200 returned on tool errors.

### Versioned tool type strings (Anthropic)
`web_search_20250305`/`_20260209`/`_20260318`; `web_fetch_20250910`/`_20260209`/`_20260309`/`_20260318`; `code_execution_20250825`/`_20260120`/`_20260521` (legacy `_20250522`); `computer_20250124`/`_20251124`; `bash_20250124` (legacy `_20241022`); `text_editor_20250728`; `memory_20250818`; `advisor_20260301`; `tool_search_tool_regex_20251119`/`tool_search_tool_bm25_20251119`.

### Beta headers (Anthropic)
`computer-use-2025-11-24` / `-2025-01-24` / `-2024-10-22`; `advisor-tool-2026-03-01`; `fine-grained-tool-streaming-2025-05-14` (legacy, superseded by `eager_input_streaming`); `code-execution-2025-05-22` (legacy); `files-api-2025-04-14` (Files API w/ code execution).

---

*End of unified specification. This document is a conceptual union for design reference; consult each vendor's source study (`anthropic-api.md`, `google-api.md`, `mistral-api.md`, `openai-api.md`) for exact, current behavior.*