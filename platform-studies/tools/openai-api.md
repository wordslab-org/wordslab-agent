# OpenAI API — Agent Tools Capabilities

Analysis of the agent-related capabilities offered by the OpenAI API, based on the official developer guides. Each capability is broken down into main concepts, API surface (functions, parameters, response fields), and notes/constraints.

All hosted tools are enabled via the `tools` array on the **Responses API** (`POST /v1/responses`) unless otherwise noted. Several tools also work on the **Chat Completions API** (`POST /v1/chat/completions`) with reduced parameter support.

---

## Table of Contents

1. [Web Search](#1-web-search)
2. [MCP and Connectors](#2-mcp-and-connectors)
3. [Skills](#3-skills)
4. [Shell](#4-shell)
5. [Computer Use](#5-computer-use)
6. [File Search](#6-file-search)
7. [Retrieval (Vector Stores Search)](#7-retrieval-vector-stores-search)
8. [Tool Search](#8-tool-search)
9. [Programmatic Tool Calling](#9-programmatic-tool-calling)
10. [Apply Patch](#10-apply-patch)
11. [Local Shell](#11-local-shell)
12. [Image Generation](#12-image-generation)
13. [Code Interpreter](#13-code-interpreter)

---

## 1. Web Search

**Summary** — Allows models to search the internet for up-to-date information before generating a response, returning sourced citations.

### Main concepts
- **Non-reasoning web search**: Fast lookup; the model passes the query to the web search tool and relays top results.
- **Agentic search (reasoning models)**: The model actively manages the search process within its chain of thought, analyzes results, and may keep searching.
- **Deep research**: A specialized agent-driven method for extended investigations (often hundreds of sources, several minutes); best with background mode and `gpt-5.5` at `high`/`xhigh` reasoning.
- **`web_search` vs `web_search_preview`**: The GA `web_search` tool supports newer controls (`filters`, `external_web_access`, `return_token_budget`); `web_search_preview` is legacy and ignores those.
- **Citations & sources**: Inline `url_citation` annotations (URL, title, start/end index) appear in the model text; the `sources` field lists all consulted URLs (often more than citations), including real-time feeds labeled `oai-sports`, `oai-weather`, `oai-finance`.

### API functions & parameters

**Responses API** — `POST /v1/responses`. Enable via the `tools` array. Tool object `type: "web_search"`:

| Parameter | Type | Description |
|---|---|---|
| `type` | string | `"web_search"` (fixed) |
| `search_context_size` | enum: `"low"` \| `"medium"` \| `"high"` | How much context from web search results is passed to the model. Does not set an exact token count. Default `medium`. |
| `filters` | object | Domain filtering. Contains `allowed_domains` (array, up to 100) and `blocked_domains` (array, up to 100). Omit HTTP/HTTPS prefix; includes subdomains. Responses API only. |
| `user_location` | object | Approximate location. `type: "approximate"`, plus `country` (ISO 3166-1 alpha-2), `city` (free text), `region` (free text), `timezone` (IANA). Not supported for deep research. |
| `search_content_types` | array of enum `"image"` \| `"text"` | Whether to return image results and/or text results. |
| `image_settings` | object | `{ max_results: number, caption: boolean }` — controls image-specific behavior. |
| `return_token_budget` | enum: `"default"` \| `"unlimited"` | Controls how much web search result content the tool may return (GPT-5+ reasoning only). `unlimited` removes the cap for long research runs. |
| `external_web_access` | boolean | Default `true` (live). Set `false` for offline/cache-only mode. Ignored by `web_search_preview`. |

Request-level parameters used with web search:
- `reasoning`: `{ "effort": "low" | "high" | "xhigh" }` (for deeper research).
- `tool_choice`: `"auto"` (search optional), `"required"`, or a specific tool choice to force search.
- `include`: array such as `["web_search_call.action.sources"]`, `["web_search_call.results"]` to surface sources / raw image results in the response.
- `background`: `true` for long-running async research.

**Chat Completions API** — `POST /v1/chat/completions` (always searches before responding):
- `model`: `"gpt-5-search-api"` (200k context). (`gpt-4o-search-preview`, `gpt-4o-mini-search-preview` are deprecated, shutdown 2026-07-23.)
- `web_search_options`: object. Supports `user_location` (with nested `approximate` object) and `search_context_size`. Does NOT support domain filters, complete source lists, live-access control, or returned-token budget.

**Important response fields (Responses API)**:
- `output[]` contains:
  - `web_search_call` item: `id` (e.g. `ws_...`), `status`, `action` (`type: "search"` | `"open_page"` | `"find_in_page"`, with `query`/`queries`), optional `results[]` (incl. `image_result` items with `image_url`, `source_website_url`, `thumbnail_url`, `caption`), and `sources[]` (if included).
  - `message` item: `content[0].text` and `content[0].annotations[]` of type `url_citation` (`url`, `title`, `start_index`, `end_index`).

**Chat Completions response fields**:
- `choices[0].message.content` (text with inline citations).
- `choices[0].message.annotations[]` of type `url_citation` (`url`, `title`, `start_index`, `end_index`).

### Notes
- Responses API search context window is limited to **128k tokens** even when the model context window is larger (e.g. `gpt-4.1` 1M).
- Web search does not support `gpt-5` with `minimal` reasoning; `gpt-5.4` with `none` effort may yield lower quality.
- `open_page` and `find_in_page` actions are supported only in reasoning models.
- `search` actions incur a tool-call cost; `open_page`/`find_in_page` do not.
- Inline citations must be made clearly visible and clickable to end users.
- Domain filtering is Responses API `web_search` only (up to 100 each of `allowed_domains`/`blocked_domains`).
- Rate limits follow the underlying model's tiered limits.
- `return_token_budget` applies only to hosted Responses `web_search` with GPT-5+ reasoning web search; rejected values include `null`, numbers, other strings.

---

## 2. MCP and Connectors

**Summary** — Give models access to external services via OpenAI-maintained **Connectors** or arbitrary **remote MCP servers** (Model Context Protocol), with optional per-tool approval flows.

### Main concepts
- **Connectors**: OpenAI-maintained MCP wrappers for popular services (e.g. Google Workspace, Dropbox). Identified by a `connector_id`.
- **Remote MCP servers**: Any public server implementing the remote MCP spec. Identified by a `server_url`.
- **Secure MCP Tunnel**: For private/on-prem/firewalled servers, use the `openai/tunnel-client` to expose them without public exposure.
- **Tool lifecycle**: The API first lists tools from the server (`mcp_list_tools`), then the model may call tools (`mcp_call`). Failed calls populate `error` (MCP protocol errors, execution errors, connectivity errors).
- **Approvals**: By default OpenAI requests approval before data is shared. `require_approval` controls this; `mcp_approval_request`/`mcp_approval_response` items implement the flow.
- **`authorization`**: OAuth access token; not stored by the API (must be sent on every request; not echoed back in the Response).
- **`defer_loading`**: When used with tool search, defers loading MCP function definitions until needed (reduces token usage for large servers).

### API functions & parameters

**Responses API** — `POST /v1/responses`. Tool object `type: "mcp"`:

Common parameters:

| Parameter | Type | Description |
|---|---|---|
| `type` | string | `"mcp"` (fixed) |
| `server_label` | string | A label/name for the server in this request. |
| `server_description` | string | (Optional) Human-readable description of the remote server. |
| `require_approval` | string \| object | `"always"` (default behavior), `"never"`, or `{ "never": { "tool_names": [...] } }` to skip approvals only for named tools. |
| `allowed_tools` | array of strings | Only import these tool names (filters the server's tool list). |
| `authorization` | string | OAuth access token for the server/connector. |
| `defer_loading` | boolean | If `true`, function definitions load only when the model needs them (used with tool search). |

Remote MCP server-specific:

| Parameter | Type | Description |
|---|---|---|
| `server_url` | string | URL of the remote MCP server. Supports Streamable HTTP or HTTP/SSE transports. |

Connector-specific:

| Parameter | Type | Description |
|---|---|---|
| `connector_id` | string | One of the supported connector IDs (e.g. `connector_dropbox`). |

Available connector IDs: `connector_dropbox`, `connector_gmail`, `connector_googlecalendar`, `connector_googledrive`, `connector_microsoftteams`, `connector_outlookcalendar`, `connector_outlookemail`, `connector_sharepoint`.

**Approvals** — To respond to an `mcp_approval_request`, create a new Response with `previous_response_id` and an input item:
```
{ "type": "mcp_approval_response", "approve": boolean, "approval_request_id": "mcpr_..." }
```

**Important response fields**:
- `mcp_list_tools` output item: `id` (`mcpl_...`), `server_label`, `tools[]` (each with `name`, `description`, `input_schema`, `annotations`).
- `mcp_call` output item: `id` (`mcp_...`), `type`, `approval_request_id`, `arguments` (JSON string), `error`, `name`, `output` (string), `server_label`.
- `mcp_approval_request` output item: `id` (`mcpr_...`), `type`, `arguments`, `name`, `server_label`.

### Notes
- You only pay for tokens used when importing tool definitions or making tool calls — no per-tool-call fee beyond tokens.
- Keep `mcp_list_tools` in conversation context to avoid re-fetching the tool list every turn (latency optimization).
- Available connectors and their per-tool OAuth scopes are documented (e.g. Dropbox `search`/`fetch`/`search_files`/`fetch_file`/`list_recent_files`/`get_profile`; Gmail `get_profile`/`search_emails`/`search_email_ids`/`get_recent_emails`/`read_email`/`batch_read_email`; etc.).
- MCP tool is compatible with Zero Data Retention / Data Residency, but data sent to an MCP server is governed by the third party's policies.
- Malicious servers can exfiltrate data via prompt injection; only connect to trusted/official servers; log and review shared data.
- Rate limits: Tier 1 200 RPM, Tier 2/3 1000 RPM, Tier 4/5 2000 RPM.

---

## 3. Skills

**Summary** — Upload, version, and attach reusable bundles of files (with a `SKILL.md` manifest) to hosted shell or local shell environments so the model can follow codified instructions and workflows.

### Main concepts
- **Skill**: A versioned bundle of files plus a `SKILL.md` manifest (YAML front matter + instructions). Compatible with the open [Agent Skills standard](https://agentskills.io/).
- **`SKILL.md` front matter**: `name`, `description` (follows the agent skills specification).
- **Form factors**: Local execution (via shell tool's local mode) and hosted container-based execution (via shell tool's `container_auto`/`container_reference`).
- **Versioning**: `default_version`, `latest_version`, and `skill_reference.version` (integer or `"latest"`).
- **Curated skills**: OpenAI-maintained first-party skills referenced by id (e.g. `openai-spreadsheets`).
- **Inline skills**: Inline a base64 zip bundle in the environment's `skills` array (no hosted skill needed).
- **Prompting behavior**: The platform injects each skill's `name`, `description`, `path` into user prompt context; the model decides whether to invoke (reads `SKILL.md` from `path`). Skill instructions are treated as user-prompt input priority.

### API functions & parameters

**Skills management endpoints** (separate from Responses):

- `POST https://api.openai.com/v1/skills` — Create a skill.
  - Multipart form: either multiple `files[]` parts (each with `filename` path inside a single top-level folder) or a single `files` zip part (`type: application/zip`).
- `POST https://api.openai.com/v1/skills/{skill_id}/versions` — Create a new version (multipart zip upload).
- `POST https://api.openai.com/v1/skills/{skill_id}` — Set `default_version` (JSON body `{ "default_version": 2 }`).
- (Delete rules: cannot delete the default version; deleting the last version deletes the skill; deleting a skill cascades.)

**Using skills in the Responses API** (via the shell tool). Tool object `type: "shell"` with `environment`:

Hosted shell (`environment.type: "container_auto"` or `"container_reference"`):
```json
"skills": [
  { "type": "skill_reference", "skill_id": "<id>" },
  { "type": "skill_reference", "skill_id": "<id>", "version": 2 }
]
```

Local shell (`environment.type: "local"`):
```json
"skills": [
  { "name": "csv-insights", "description": "...", "path": "<local path>" }
]
```
(Local shell does NOT support `skill_reference`; provide local file paths.)

**Inline skills** (for containers): `{ "type": "inline", "name", "description", "source": { "type": "base64", "media_type": "application/zip", "data": "<base64>" } }` passed to `POST /v1/containers`.

### Notes
- `SKILL.md` matching is case-insensitive; exactly one `skill.md`/`SKILL.md` allowed per bundle.
- Limits: max zip upload **50 MB**; max **500** files per skill version; max uncompressed file size **25 MB**.
- Hosted skills follow the hosted shell container lifecycle (mounted while active, discarded on expiry/deletion). For fully self-managed execution, use local shell mode.
- Security: Skills are privileged instructions — review as untrusted input; don't expose an open Skills repository to end users; gate high-impact actions behind approval.

---

## 4. Shell

**Summary** — Lets models run shell commands either in OpenAI-managed hosted containers or in a local runtime you host and execute yourself (Responses API only, not Chat Completions).

### Main concepts
- **Hosted shell**: OpenAI provisions/manages a container. Two environment modes:
  - `container_auto`: OpenAI creates a container for the request.
  - `container_reference`: Reuse a previously created container by `container_id`.
- **Local shell** (`environment.type: "local"`): You execute `shell_call` actions in your own runtime and return `shell_call_output` to the model.
- **Containers API** (`/v1/containers`): Create, list, delete reusable long-running containers with memory limits and expiry.
- **Network access**: Off by default; enable via org allow list + request-level `network_policy`.
- **Domain secrets**: Inject secret values for approved domains at runtime; the model sees only placeholder names (e.g. `$API_KEY`).
- **Output items**: `shell_call` (commands the model wants to run) and `shell_call_output` (your captured stdout/stderr/outcome).

### API functions & parameters

**Responses API** — `POST /v1/responses`. Tool object `type: "shell"`:
```json
{ "type": "shell", "environment": { ... } }
```

`environment` variants:

| `type` | Fields | Description |
|---|---|---|
| `container_auto` | `network_policy` (optional), `skills` (optional) | OpenAI provisions a fresh container. |
| `container_reference` | `container_id` (string, e.g. `cntr_...`) | Reuse an existing container. |
| `local` | (none / `skills` for local) | You execute commands locally. |

`network_policy` object (when outbound network enabled):

| Field | Type | Description |
|---|---|---|
| `type` | `"allowlist"` | Policy type. |
| `allowed_domains` | array of strings | Domains the container may reach (must be within org allow list). |
| `domain_secrets` | array | `{ "domain", "name", "value" }` — secret injected only for that domain; model sees `name` placeholder. |

**Containers API** — `POST /v1/containers`:

| Field | Type | Description |
|---|---|---|
| `name` | string | Container name. |
| `memory_limit` | string | e.g. `"1g"`. |
| `expires_after` | object | `{ "anchor": "last_active_at", "minutes": <int> }`. |
| `skills` | array | Skill references or inline skills attached to the container. |

- `DELETE /v1/containers/{container_id}` — proactively delete a container.

**`shell_call` output item** (model → you):
```json
{ "type": "shell_call", "call_id": "call_...", "action": { "commands": ["ls -l"], "timeout_ms": 120000, "max_output_length": 4096 }, "status": "in_progress" }
```

**`shell_call_output`** (you → model, in the next request's `input`):
```json
{ "type": "shell_call_output", "call_id": "...", "max_output_length": 4096,
  "output": [ { "stdout": "...", "stderr": "...", "outcome": { "type": "exit", "exit_code": 0 } },
               { ..., "outcome": { "type": "timeout" } } ] }
```
- `outcome.type`: `"exit"` (with `exit_code`) or `"timeout"`.
- Include `max_output_length` from the `shell_call` if present.

Multi-turn: reuse the container and pass `previous_response_id` to continue work.

### Notes
- Hosted runtime: Debian 12; default working dir `/mnt/data` (always present, supported path for downloadable artifacts); no interactive TTY; no `sudo`.
- Preinstalled languages: Python 3.11, Node.js 22.16, Java 17.0, PHP 8.2, Ruby 3.1, Go 1.23.
- Network policy precedence: org allow list defines the full set; request-level `network_policy` further restricts; requests fail if `allowed_domains` exceeds the org list.
- Container data is deleted on expiry or explicit delete (ephemeral block storage).
- For local shell, in Agents SDK you pass a custom `Shell` executor implementation to `shellTool({ shell, needs_approval, on_approval })`.
- Errors: return a timeout outcome with partial output on timeout; preserve non-zero exit outputs for model reasoning; execution should be non-interactive.

---

## 5. Computer Use

**Summary** — Lets a model operate software through the UI: it inspects screenshots and returns structured UI actions (clicks, typing, scrolling) for your harness to execute, or drives a custom/code-execution harness.

### Main concepts
- **Three harness shapes**:
  1. **Built-in Computer use loop** — `computer` tool returns batched `actions[]`; you execute them and send back `computer_call_output` with a screenshot.
  2. **Custom tool/harness** — Expose an existing Playwright/Selenium/VNC/MCP harness as a normal tool.
  3. **Code-execution harness** — Model writes and runs short scripts in a runtime (REPL) to mix visual and programmatic UI interaction. `gpt-5.4` is explicitly trained for this.
- **Loop**: send task → receive `computer_call` with `actions[]` → run actions in order → capture screenshot → send back as `computer_call_output` → repeat until no more `computer_call`.
- **Screenshot detail**: Use `detail: "original"` on screenshot inputs (up to 10.24M pixels) for best click accuracy; avoid `high`/`low`. Good downscale resolutions: 1440x900, 1600x900.
- **Modifier keys**: Mouse actions (`click`, `double_click`, `drag`, `move`, `scroll`) accept an optional `keys` array (e.g. `["SHIFT"]`) for modifier-assisted actions.
- **`keypress`** is for standalone keyboard input; for held modifiers during mouse actions, use the mouse action's `keys` array instead.

### API functions & parameters

**Responses API** — `POST /v1/responses`. Tool object:
```json
{ "type": "computer" }
```
(No additional parameters needed for the GA `computer` tool.)

**`computer_call` output item** (model → you):
```json
{ "type": "computer_call", "call_id": "call_001", "actions": [ ... ], "status": "completed" }
```

Action types in `actions[]`:

| Action type | Fields |
|---|---|
| `screenshot` | (none) |
| `click` | `button` (`"left"`\|`"middle"`\|`"right"`, default `"left"`), `x` (int), `y` (int), `keys` (array, optional) |
| `double_click` | same as `click` |
| `scroll` | `x`, `y`, `scrollX`, `scrollY`, optional `keys` |
| `type` | `text` (string) |
| `keypress` | `keys` (array of key names) |
| `drag` | `path` (array of `[x,y]` or `{x,y}`; at least 2 points), optional `keys` |
| `move` | `x`, `y`, optional `keys` |
| `wait` | (none; ~2s pause) |

Model-emitted key names include: `ENTER`/`RETURN`, `ESC`/`ESCAPE`, `TAB`, `SPACE`, `BACKSPACE`, `DELETE`/`DEL`, `HOME`, `END`, `PAGEUP`, `PAGEDOWN`, `UP`/`ARROWUP`, `DOWN`/`ARROWDOWN`, `LEFT`/`ARROWLEFT`, `RIGHT`/`ARROWRIGHT`, `CTRL`/`CONTROL`, `SHIFT`, `OPTION`/`ALT`, `META`/`CMD`/`COMMAND`.

**`computer_call_output` input item** (you → model, in next request's `input`, with `previous_response_id`):
```json
{ "type": "computer_call_output", "call_id": "<call_id>",
  "output": { "type": "computer_screenshot", "image_url": "data:image/png;base64,...", "detail": "original" } }
```

### Notes
- Run in an isolated browser or VM; keep a human in the loop for high-impact actions; treat page content as untrusted input (only direct user instructions count as permission).
- GA migration from `computer-use-preview`:
  - Model: `computer-use-preview` → `gpt-5.5`.
  - Tool name: `computer_use_preview` → `computer`.
  - Actions: single `action` → batched `actions[]` array.
  - `truncation: "auto"` no longer necessary.
  - Preview tool params (`display_width`, `display_height`, `environment: "browser"`) are replaced by the simpler `{ type: "computer" }`.
- Confirmation policy: delay until the exact risky action; group imminent well-defined risky actions; confirm before typing/submitting sensitive data (typing counts as transmission); hand-off required for password changes and bypassing browser/website safety barriers.
- A CUA sample app (GitHub `openai/openai-cua-sample-app`) demonstrates multi-environment integration.

---

## 6. File Search

**Summary** — A hosted Responses API tool that lets models retrieve information from a knowledge base of uploaded files (in vector stores) via semantic + keyword search, returning cited file references.

### Main concepts
- **Vector store**: A container of searchable files; files are auto-chunked, embedded, and indexed when added. Backed by `file` objects wrapped in `vector_store.file` objects.
- **Hosted tool**: OpenAI manages execution; when the model decides to use it, it calls the tool, retrieves info, and returns output. You don't implement execution.
- **Citations**: `file_citation` annotations (`file_id`, `filename`) appear in the output text; raw search results are not returned by default (use `include`).

### API functions & parameters

**Setup endpoints** (separate from Responses):
- `POST /v1/files` — Upload a file (`purpose: "assistants"`).
- `POST /v1/vector_stores` — Create a vector store (`name`).
- `POST /v1/vector_stores/{vector_store_id}/files` — Add a file (`file_id`). Check status via `GET .../files` (list) until `status: "completed"`.

**Responses API** — `POST /v1/responses`. Tool object `type: "file_search"`:

| Parameter | Type | Description |
|---|---|---|
| `type` | string | `"file_search"` (fixed) |
| `vector_store_ids` | array of strings | Vector stores to search (required). |
| `max_num_results` | integer | Limit number of results retrieved (reduces tokens/latency, may reduce quality). |
| `filters` | object | Metadata filter (e.g. `{ "type": "in", "key": "category", "value": ["blog","announcement"] }`). See Retrieval guide for filter grammar. |

Request-level:
- `include`: e.g. `["file_search_call.results"]` to include raw search results in the response.

**Important response fields**:
- `output[]` contains:
  - `file_search_call` item: `id` (`fs_...`), `status`, `queries[]`, `search_results` (null unless `include`d).
  - `message` item with `content[0].text` and `content[0].annotations[]` of type `file_citation` (`index`, `file_id`, `filename`).

### Notes
- **Supported file formats** (text/ MIME must be utf-8/utf-16/ascii): `.c`, `.cpp`, `.cs`, `.css`, `.doc`, `.docx`, `.go`, `.html`, `.java`, `.js`, `.json`, `.md`, `.pdf`, `.php`, `.pptx`, `.py`, `.rb`, `.sh`, `.tex`, `.ts`, `.txt`.
- For metadata filtering and full vector store semantics, see the Retrieval guide.
- Rate limits: Tier 1 100 RPM, Tier 2/3 500 RPM, Tier 4/5 1000 RPM.

---

## 7. Retrieval (Vector Stores Search)

**Summary** — The Retrieval API performs semantic search over your data using vector stores, returning semantically similar chunks (even with few/no keyword overlap), with similarity scores and file-of-origin metadata.

### Main concepts
- **Semantic search**: Uses vector embeddings (`text-embedding-3-small` cosine similarity) to surface relevant results beyond keyword matches.
- **Vector stores**: Containers that auto-chunk, embed, and index added files. Power both the Retrieval API and the file search tool.
- **Object types**: `file` (uploaded content), `vector_store` (container), `vector_store.file` (chunked/embedded wrapper with `attributes`).
- **Query rewriting**: `rewrite_query=true` rewrites queries for optimal performance; rewritten query in result's `search_query` field.
- **Attribute filtering**: Compare a file's `attributes` key with comparison operators, or combine with `and`/`or` compound filters.
- **Ranking**: `ranking_options` with `ranker` (`auto` or `default-2024-08-21`), `score_threshold` (0.0–1.0), and `hybrid_search` tuning (`embedding_weight`/`rrf_embedding_weight`, `text_weight`/`rrf_text_weight` — at least one > 0).
- **Chunking**: Configurable via `chunking_strategy` (`static` type with `max_chunk_size_tokens` and `chunk_overlap_tokens`).
- **Attributes**: Per-file dictionary (max 16 keys, 256 chars each) used for filtering.
- **Expiration**: `expires_after` policy on vector stores (`anchor`, `days`).

### API functions & parameters

**Search** — `client.vector_stores.search` (underlying endpoint `POST /v1/vector_stores/{vector_store_id}/search`):

| Parameter | Type | Description |
|---|---|---|
| `vector_store_id` | string | The vector store to search. |
| `query` | string | Natural-language search query. |
| `rewrite_query` | boolean | Auto-rewrite the query for better results. Rewritten query returned in `search_query`. |
| `max_num_results` | integer | Up to 50 (default 10). |
| `attribute_filter` | object | Filter (comparison or compound) applied before search. |
| `ranking_options` | object | `{ ranker, score_threshold, hybrid_search: { embedding_weight, text_weight } }`. |

**Comparison filter**:
```json
{ "type": "eq" | "ne" | "gt" | "gte" | "lt" | "lte" | "in" | "nin",
  "key": "attributes_key", "value": "target_value" }
```
(Note: filename filters use `"property": "filename"` instead of `"key"`.)

**Compound filter**: `{ "type": "and" | "or", "filters": [...] }`.

**Search response** (`vector_store.search_results.page`):
- `search_query` (string; rewritten if `rewrite_query=true`).
- `data[]`: each result has `file_id`, `filename`, `score` (float), `attributes`, `content[]` (each `{ "type": "text", "text": "..." }`).
- `has_more` (boolean), `next_page` (pagination).

**Vector store operations**:
- `POST /v1/vector_stores` — Create (`name`, optional `file_ids`).
- `GET /v1/vector_stores/{id}` — Retrieve.
- `POST /v1/vector_stores/{id}` (update) — `name`, `expires_after`.
- `DELETE /v1/vector_stores/{id}` — Delete.
- `GET /v1/vector_stores` — List.

**Vector store file operations**:
- `POST /v1/vector_stores/{vs_id}/files` — Create (`file_id`, optional `attributes`, `chunking_strategy`). Async; use `create_and_poll`/`upload_and_poll` helpers.
- `POST .../files` (upload variant) — `file` (stream).
- `GET /v1/vector_stores/{vs_id}/files/{file_id}` — Retrieve.
- `POST /v1/vector_stores/{vs_id}/files/{file_id}` (update) — `attributes`.
- `DELETE /v1/vector_stores/{vs_id}/files/{file_id}` — Delete (eventually consistent).
- `GET /v1/vector_stores/{vs_id}/files` — List.

**Batch operations** (`/v1/vector_stores/{vs_id}/file_batches`):
- `POST` — Create batch with `file_ids` (+ shared `attributes`/`chunking_strategy`) OR `files[]` array (per-file `{ file_id, attributes, chunking_strategy }`). Up to 500 files per batch. Use `create_and_poll`.
- `GET` — Retrieve batch (`batch_id`).
- `POST .../cancel` — Cancel.
- `GET` — List.

**Chunking strategy** (`static`): `max_chunk_size_tokens` (100–4096 inclusive), `chunk_overlap_tokens` (≥0, ≤ `max_chunk_size_tokens/2`). Defaults: 800 / 400.

### Notes
- **Pricing**: Up to 1 GB across all stores is free; beyond 1 GB it's $0.10/GB/day. Use `expires_after` to minimize costs.
- **Limits**: Max file size 512 MB; max 5,000,000 tokens per file (auto-computed). Attributes: max 16 keys, 256 chars each.
- **Rate limits**: `/vector_stores/{vector_store_id}/files` and `/file_batches` share a per-vector-store limit of **300 requests/minute**.
- Removing files is eventually consistent; search may still include removed content briefly.
- **Synthesizing responses**: Use `vector_stores.search` to get results, format them, then call `chat.completions.create` (e.g. `gpt-5.5`) with a developer message instructing grounded answering, and a user message containing sources + original query.
- **Supported file formats** (same as file search): `.c`, `.cpp`, `.cs`, `.css`, `.doc`, `.docx`, `.go`, `.html`, `.java`, `.js`, `.json`, `.md`, `.pdf`, `.php`, `.pptx`, `.py`, `.rb`, `.sh`, `.tex`, `.ts`, `.txt` (text/ MIME must be utf-8/utf-16/ascii).

---

## 8. Tool Search

**Summary** — Allows the model to dynamically search for and load tool definitions into its context only as needed, reducing upfront token usage and cost while preserving the model cache.

### Main concepts
- **Deferred tools / `defer_loading`**: A flag (`true`) on functions, namespace functions, or MCP server tool definitions that withholds their full parameter schema from the model until tool search loads them. The model still sees the tool/namespace/server name and description at request start; tool search mostly defers the parameter schema.
- **Namespaces**: Grouped tool containers (`{"type": "namespace", "name": ..., "tools": [...]}`) that the model has been primarily trained to search. `defer_loading` applies to functions inside a namespace, not the namespace object itself.
- **Hosted tool search**: OpenAI searches across the deferred tools declared in the request and returns the loaded subset in the same response (`execution: "server"`).
- **Client-executed tool search**: The model emits a `tool_search_call`, the application performs the lookup, and returns a `tool_search_output` (`execution: "client"`).
- **Cache preservation**: Newly discovered tools are injected at the end of the context window so the model cache is preserved across requests.

### API functions & parameters

**Tool declaration** — add to the `tools` array:
- `{"type": "tool_search"}` (hosted mode, default).
- Client mode: `{"type": "tool_search", "execution": "client", "description": <string>, "parameters": <JSON schema>}` — the `parameters` schema defines the search arguments (e.g. `goal: {type: string}`).

**Function/namespace definition flags:**
- `defer_loading` (boolean) — on a function or MCP tool, marks it as deferred.
- Namespace: `{"type": "namespace", "name": <string>, "description": <string>, "tools": [<function defs>]}`.

**Response output items:**
- `tool_search_call`: `type`, `execution` (`"server"` | `"client"`), `call_id` (`null` in hosted mode; defined in client mode), `status` (`"completed"`), `arguments`:
  - hosted: `{"paths": ["crm"]}` — namespace/server paths to load.
  - client: matches the configured `parameters` schema (e.g. `{"goal": "..."}`).
- `tool_search_output`: `type`, `execution`, `call_id` (echo the `tool_search_call`'s `call_id` in client mode), `status`, `tools` (array of loaded tool definitions).
- Subsequent `function_call`: `type: "function_call"`, `name`, `namespace`, `call_id`, `arguments` (JSON string).

**Advanced input item:**
- `additional_tools`: `{"type": "additional_tools", "role": "developer", "tools": [...]}` — injects tools at a specific point in the conversation; tools become available only after that item appears in the input.

### Notes
- Only `gpt-5.4` and later models support `tool_search`.
- Activation requires (1) adding `{"type": "tool_search"}` to `tools` and (2) marking tools to defer with `defer_loading: true`. Examples use `parallel_tool_calls=False`.
- Recommended to use namespaces or MCP servers for greater token savings; keep each namespace to fewer than 10 functions for better efficiency and model performance.
- Namespaces may mix deferred and non-deferred tools; non-deferred tools are callable immediately.
- In hosted mode, the model can load multiple namespaces/MCP servers in a single `tool_search_call`.
- In client mode, you do not need to reload the same tool across turns (loaded tools remain available); removing a tool from `tool_search_output` breaks the cache from that point forward.
- Client-executed tool search supports returning tools not present in the original request (advanced workflow; validate returned schemas carefully).
- Best practice: clear, concise namespace descriptions; put richer detail inside deferred function descriptions.

---

## 9. Programmatic Tool Calling

**Summary** — Lets a model write and run hosted JavaScript that orchestrates (composes, loops over, parallelizes, and reduces intermediate results from) tool calls in a Responses API request.

### Main concepts
- **`programmatic_tool_calling` hosted tool**: Added to the `tools` array to enable the feature.
- **`allowed_callers`**: Per-tool flag controlling how the model may invoke a tool — omitted/`["direct"]` (direct calls only), `["programmatic"]` (only from a program), or `["direct", "programmatic"]` (either).
- **V8 runtime**: Fresh, isolated runtime per generated program. Supports JavaScript with top-level `await`; no Node.js, package install, network, filesystem, subprocess, console, or persistent JS state between executions. Programs interact with external systems only through enabled tools and emit output via `text(...)` or `image(...)`.
- **`output_schema`**: Describes the JSON object encoded in a `function_call_output.output` string, so generated JavaScript can use returned fields reliably.
- **Zero Data Retention (ZDR)**: Supported without a persistent code-execution container; must be enabled for the org/project (`store: false` alone does not enable ZDR).
- **Program lifecycle**: A program can pause multiple times while client-owned function calls execute; the application returns each function result with the original `call_id` and `caller`, then continues until a final assistant `message` arrives.

### API functions & parameters

**Tool declarations:**
- Hosted tool: `{"type": "programmatic_tool_calling"}`.
- Eligible function tool: `{"type": "function", "name", "description", "parameters", "output_schema": {<JSON schema>}, "allowed_callers": ["programmatic"]}`.

**`allowed_callers` values:**

| Value | Behavior |
| --- | --- |
| Omitted or `["direct"]` | Model can call the tool directly. |
| `["programmatic"]` | Only code in a `program` item can call the tool. |
| `["direct", "programmatic"]` | Model can call directly or from a program. |

**Supported tool types for `allowed_callers: ["programmatic"]`:** `function` and `custom`, `mcp`, `apply_patch`, local and hosted `shell`, `code_interpreter`.

**Response output items (all standard Responses API items; no separate envelope):**
- `program`: `type: "program"`, `id` (e.g. `prog_123`), `call_id` (e.g. `call_prog_123`), `code` (generated JavaScript), `fingerprint` (opaque replay/resume state).
- `function_call` (made by the program): `type: "function_call"`, `id`, `call_id`, `name`, `arguments` (JSON string), `caller: {"type": "program", "caller_id": <program call_id>}`.
- `program_output`: `type: "program_output"`, `id` (e.g. `prog_out_123`), `call_id` (matches program's `call_id`), `result` (JSON string following the program result shape), `status` (`"completed"` | `"incomplete"`).

**Function result return:** `function_call_output` — copy `caller` unchanged from the function call; the service uses it to resume the correct program.

### Notes
- Use Programmatic Tool Calling for predictable control flow where code can return a smaller structured result; use direct tool calling for single lookups, adaptive/semantic decisions, approval-sensitive writes, or final citation/artifact validation.
- Tool search runs as a top-level tool, not from inside generated JavaScript; an already-running program cannot invoke tool search, so deferred tools must be loaded before starting a program that needs them.
- For MCP tools, the tool's `require_approval` policy can pause the program until approval.
- With `store: false`, replay the complete sequence (every `program`, reasoning, function-call, function-call-output, `program_output` item); for stored responses, use `previous_response_id`.
- For stateless reasoning-model requests, include `reasoning.encrypted_content` and replay reasoning items.
- Make function calls idempotent (retries/replays should not repeat unsafe side effects); check arguments and permissions in the application even for calls from a hosted program; require application-level approval before high-impact actions regardless of caller.
- A final `message` can arrive with or after the `program_output`; continue until you receive that message.

---

## 10. Apply Patch

**Summary** — Lets GPT-5.1 (and other supported models) create, update, and delete files in a codebase by emitting structured V4A-diff patch operations that the integration applies and reports back on, enabling iterative multi-step code editing.

### Main concepts
- **`apply_patch` tool**: `{"type": "apply_patch"}` — no input schema is provided by the developer; the model knows how to construct `operation` objects.
- **`apply_patch_call`**: Output item emitted by the model describing a single file operation.
- **`apply_patch_call_output`**: The item the application returns per `call_id` with a `status` (`"completed"` | `"failed"`) and optional `output` string.
- **V4A diff format**: The diff format used inside `operation.diff` for `create_file`/`update_file`; the developer's patch harness interprets it.
- **Patch harness**: The developer-implemented code that interprets diffs, applies them to a working directory/repo, and records success/logs. Reference implementations exist in the Python Agents SDK (`agents/apply_diff.py`) and TypeScript Agents SDK (`utils/applyDiff.ts`).

### API functions & parameters

**Tool declaration:** `tools=[{"type": "apply_patch"}]`.

**`apply_patch_call` object fields:**
- `id` (e.g. `apc_...`)
- `type`: `"apply_patch_call"`
- `status`: `"completed"`
- `call_id` (e.g. `call_...`)
- `operation`: object with:
  - `type`: one of `"create_file"`, `"update_file"`, `"delete_file"`
  - `path`: target file path (e.g. `"lib/fib.py"`)
  - `diff`: V4A diff string (present for `create_file` and `update_file`; absent for `delete_file`)

**`apply_patch_call_output` fields:**
- `type`: `"apply_patch_call_output"`
- `call_id`: the matching call's `call_id`
- `status`: `"completed"` | `"failed"`
- `output`: optional string (success log or error message)

**Operation types table:**

| Operation Type | Purpose | Payload |
| --- | --- | --- |
| `create_file` | Create a new file at `path`. | `diff` is a V4A diff representing the full file contents. |
| `update_file` | Modify an existing file at `path`. | `diff` is a V4A diff with additions, deletions, or replacements. |
| `delete_file` | Remove a file at `path`. | No `diff`; delete the file entirely. |

**Agents SDK helpers:** `applyDiff(currentContent, operation.diff, "create")` (TypeScript) / `apply_diff("", operation.diff, create=True)` (Python); `applyPatchTool` (TS) / `ApplyPatchTool` (Python) and an `Editor` interface with `createFile`/`updateFile`/`deleteFile` methods returning `{"status", "output"}`.

### Notes
- API availability: Chat Completions; supported models GPT-5.5, GPT-5.4, GPT-5.2, GPT-5.1.
- Provide file context inline in `input` or equip the model with exploring tools (e.g. the `shell` tool); apply_patch pairs well with `shell` for agentic file discovery.
- Encourage small, focused diffs via system instructions; after patches, run tests/linters and feed failures back into the next `input`.
- Safety/robustness: validate paths (prevent directory traversal, restrict to allowed dirs), consider backups/scratch copies, always return `failed` with an informative `output` string on errors, and decide on all-or-nothing vs per-file atomicity.
- Continue the loop by calling the Responses API again with `previous_response_id` (or by passing conversation items into `input`), keeping `tools=[{"type": "apply_patch"}]` so the model can continue editing.
- Common error examples: file not found (`"Error: File not found at path '...'"`), patch conflict (`"Error: Invalid Context:\n@@ ..."`).

---

## 11. Local Shell

**Summary** — A tool allowing agents to run shell commands locally on a machine you/user provide; the API only returns command instructions, your code executes them and returns output back to the model. (Outdated for new use cases — use the `shell` tool with GPT-5.1 instead.)

### Main concepts
- **`local_shell` tool**: `{"type": "local_shell"}` — designed to work with Codex CLI and `codex-mini-latest`.
- **`local_shell_call`**: Output item containing an action (e.g. `exec`) with a command to execute; may also surface as a `tool_call` with `tool_name == "local_shell"`.
- **`local_shell_call_output`**: Item returned by the application with command output and metadata (status code).
- **Local execution**: Commands run in the developer's own runtime, not on OpenAI infrastructure; the developer is fully in control of which commands run.

### API functions & parameters

**Tool declaration:** `tools=[{"type": "local_shell"}]`.

**`local_shell_call` action/arguments fields (accessed via `call.action` or `call.arguments`):**
- `command`: string or pre-split argv tokens to execute.
- `working_directory`: directory to run the command in (falls back to current dir).
- `env`: environment variables dict merged with the base environment.
- `timeout_ms`: hint for execution timeout (the model provides this; the app should enforce its own limits).

**`local_shell_call_output` fields:**
- `type`: `"local_shell_call_output"`
- `call_id`: the matching call's `call_id` (missing `call_id` causes a `400` validation error)
- `output`: string combining stdout + stderr (or an error message on failure)

### Notes
- Available only via the Responses API with `codex-mini-latest`; not available on other models or via the Chat Completions API.
- Outdated for new use cases — prefer the `shell` tool with GPT-5.1.
- Running arbitrary shell commands is dangerous: always sandbox/containerize execution (Docker, firejail, jailed user), impose resource limits (time, memory, network), filter high-risk commands (`rm`, `curl`, network utilities), and log every command and output.
- `timeout_ms` is only a hint — enforce your own limits.
- On failure (non-zero exit, timeout), still send a `local_shell_call_output` with the error in `output`; the model can recover or try a different command.
- Malformed data (e.g. missing `call_id`) returns a standard `400` validation error.

---

## 12. Image Generation

**Summary** — Allows a text-capable mainline model to generate or edit images using GPT Image models, with optional image inputs, multi-turn editing, and streaming partial images.

### Main concepts
- **`image_generation` hosted tool**: Added to `tools`; the model decides when/how to generate images from the prompt and any provided image inputs.
- **`image_generation_call`**: Output item containing a base64-encoded image (`result`).
- **GPT Image models**: `gpt-image-2`, `gpt-image-1.5`, `gpt-image-1`, `gpt-image-1-mini` — used internally for generation; these are not valid values for the Responses API `model` field. Use a mainline text model (e.g. `gpt-5.5`) with the hosted tool.
- **Revised prompt**: The mainline model automatically revises the prompt for improved performance; accessible via the `revised_prompt` field.
- **Multi-turn editing**: Refine images across turns using `previous_response_id` or by referencing prior image call IDs.
- **Partial images (streaming)**: Configurable count of intermediate images streamed during generation.

### API functions & parameters

**Tool declaration:** `tools=[{"type": "image_generation"}]` — or with options: `{"type": "image_generation", "partial_images": <1-3>}`.

**Force the tool call:** `tool_choice = {"type": "image_generation"}`.

**Tool options (parameters on the tool object):**
- `size`: image dimensions (e.g. 1024×1024 or 1024×1536); supports `auto`. `gpt-image-2` supports flexible sizes meeting its resolution constraints.
- `quality`: rendering quality — `low`, `medium`, `high`, or `auto`.
- `background`: `transparent` or `opaque`, or `auto`. (`gpt-image-2` doesn't support transparent backgrounds — such requests fail.)
- `format`: file output format.
- `compression`: compression level (0–100%) for JPEG and WebP formats.
- `action`: `"auto"` (default — model chooses generate vs edit), `"generate"`, or `"edit"`.
- `partial_images`: integer 1–3 (streaming partial image count).

**Input images:** provide via file IDs or base64 data.

**`image_generation_call` response object fields:**
- `id` (e.g. `ig_123`)
- `type`: `"image_generation_call"`
- `status`: `"completed"`
- `revised_prompt`: string (the model's revised prompt)
- `result`: base64-encoded image string

**Streaming events:**
- `response.image_generation_call.partial_image`: fields `partial_image_index` (int), `partial_image_b64` (string).
- `response.completed`: final response with `image_generation_call` items in `event.response.output`.

**Multi-turn via image ID:** pass `{"type": "image_generation_call", "id": <previous call id>}` in the follow-up `input` array.

### Notes
- Supported mainline models: `gpt-5.5`, `gpt-5.4-mini`, `gpt-5.4-nano`, `gpt-5.2`, `gpt-5`, `gpt-5-nano`, `o3`, `gpt-4.1`, `gpt-4.1-mini`, `gpt-4.1-nano`, `gpt-4o`, `gpt-4o-mini`.
- Prompting tips: use terms like `draw` or `edit`; for combining images say "edit the first image by adding this element from the second image" rather than "combine"/"merge".
- The model can choose whether to generate a new image or edit one already in the conversation; `action` controls this (default `auto`).
- `gpt-image-2` supports flexible `size` values but not transparent backgrounds.

---

## 13. Code Interpreter

**Summary** — Allows models to write and run Python in a sandboxed container VM to solve complex problems (data analysis, coding, math, file/image processing, iterative code execution). The model knows this as the "python tool."

### Main concepts
- **`code_interpreter` tool**: `{"type": "code_interpreter", "container": ...}` — requires a container object.
- **Container**: A fully sandboxed virtual machine the model runs Python in; can hold uploaded or generated files. Two creation modes:
  - **Auto mode**: `"container": {"type": "auto", "memory_limit": "4g", "file_ids": ["file-1", "file-2"]}` — automatically creates or reuses an active container from a previous `code_interpreter_call` in context.
  - **Explicit mode**: create a container via `v1/containers` endpoint, then pass its `id` as the `container` value (string) in the tool config.
- **`code_interpreter_call`**: Output item; its `container_id` reveals the container generated/used (auto mode).
- **`container_file_citation`**: Annotation on the assistant's message pointing to files the model created in the container (`container_id`, `file_id`, `filename`).
- **Memory limits**: `1g` (default), `4g`, `16g`, `64g`; higher tiers billed at built-in tools rates; the tier applies for the container's entire life.

### API functions & parameters

**Tool declaration (auto mode):**
```json
{"type": "code_interpreter", "container": {"type": "auto", "memory_limit": "4g", "file_ids": ["file-1", "file-2"]}}
```
- `container.type`: `"auto"`
- `container.memory_limit`: `"1g"` | `"4g"` | `"16g"` | `"64g"` (omit to keep default 1 GB)
- `container.file_ids`: array of file IDs to upload to the container

**Explicit container creation — `POST /v1/containers`:**
- `name` (string, e.g. `"My Container"`)
- `memory_limit` (string, e.g. `"4g"`)

Then use the returned container `id` as the tool's `container` value:
```json
{"type": "code_interpreter", "container": "cntr_abc123"}
```

**Other Responses API params used in examples:** `model` (e.g. `"gpt-5.6"`), `instructions`, `input`, `tool_choice` (e.g. `"required"` to force tool use).

**Output:**
- `code_interpreter_call` item (contains `container_id`).
- Assistant `message` with `annotations` of type `container_file_citation` (fields: `file_id`, `container_id`, `filename`, `index`, `end_index`, `start_index`, `type`).

**Container file endpoints:**
- `Create container file` (`/v1/containers/{id}/files`) — multipart upload or JSON body with `file_id`.
- `List container files`.
- `Retrieve container file content` (download bytes).
- Files in the model input are automatically uploaded to the container.

### Notes
- The model knows this as the "python tool"; the most explicit way to invoke it is to ask for "the python tool" in prompts.
- API availability: Chat Completions; rate limit 100 RPM per org; see Pricing and ZDR/data residency docs.
- **Expiration**: a container expires if unused for 20 minutes; expired containers cannot be used in `v1/responses` (the call fails), data is discarded and unrecoverable, and you cannot move a container from expired to active (create a new one and re-upload files). Any container operation (retrieve, add/delete files) refreshes `last_active_at`. Download needed files while the container is active.
- Treat containers as ephemeral; store all needed data on your own systems.
- Auto-mode-created containers are also accessible via the `/v1/containers` endpoint.
- Supported file formats include `.c`, `.cs`, `.cpp`, `.csv`, `.doc`, `.docx`, `.html`, `.java`, `.json`, `.md`, `.pdf`, `.php`, `.pptx`, `.py`, `.rb`, `.tex`, `.txt`, `.css`, `.js`, `.sh`, `.ts`, `.jpeg`, `.jpg`, `.gif`, `.pkl`, `.png`, `.tar`, `.xlsx`, `.xml`, `.zip` (with corresponding MIME types).
- Use cases: processing files with diverse data/formatting; generating files with data and graph images; iterative code writing/running until success; boosting visual intelligence (crop, zoom, rotate, transform images) in reasoning models like `o3` and `o4-mini`.