# Anthropic API — Agent & Tool Capabilities

Analysis of the agent-related capabilities offered by the Anthropic Messages API (`POST /v1/messages`), based on the official Claude Platform Docs under `platform.claude.com/docs/en/agents-and-tools/tool-use/`. Each capability is broken down into main concepts, API surface (parameters, request fields, content blocks, response shapes), and notes/constraints.

Tools are declared in the top-level `tools` array on the **Messages API**. Claude decides when to call a tool based on the request and the tool's description, then either returns a structured `tool_use` block your application executes (**client tools**) or runs the operation on Anthropic's infrastructure and returns the result inline (**server tools**). The examples below reference `claude-opus-4-8`.

---

## Table of Contents

1. [Tool Use Overview & Mechanics](#1-tool-use-overview--mechanics)
2. [Define Tools](#2-define-tools)
3. [Handle Tool Calls](#3-handle-tool-calls)
4. [How Tool Use Works (the agentic loop)](#4-how-tool-use-works-the-agentic-loop)
5. [Parallel Tool Use](#5-parallel-tool-use)
6. [Tool Use with Prompt Caching](#6-tool-use-with-prompt-caching)
7. [Fine-Grained Tool Streaming](#7-fine-grained-tool-streaming)
8. [Manage Tool Context](#8-manage-tool-context)
9. [Server Tools (shared mechanics)](#9-server-tools-shared-mechanics)
10. [Web Search Tool](#10-web-search-tool)
11. [Web Fetch Tool](#11-web-fetch-tool)
12. [Code Execution Tool](#12-code-execution-tool)
13. [Computer Use Tool](#13-computer-use-tool)
14. [Bash Tool](#14-bash-tool)
15. [Text Editor Tool](#15-text-editor-tool)
16. [Memory Tool](#16-memory-tool)
17. [Advisor Tool](#17-advisor-tool)
18. [Tool Search Tool](#18-tool-search-tool)
19. [Tool Runner (SDK)](#19-tool-runner-sdk)
20. [Programmatic Tool Calling](#20-programmatic-tool-calling)
21. [Tool Combinations](#21-tool-combinations)
22. [Cross-Cutting Reference](#22-cross-cutting-reference)

---

## 1. Tool Use Overview & Mechanics

**Summary** — Tool use lets Claude call functions you define or that Anthropic provides, returning structured calls (client tools) or executing them server-side (server tools).

### Main concepts
- **Three execution buckets:**
  - *User-defined tools (client)* — you write the schema, run the code, return results.
  - *Anthropic-schema tools (client)* — Anthropic publishes & trains on the schema; you run the code, return results (`memory`, `bash`, `text_editor`, `computer`).
  - *Server tools* — Anthropic runs the code; you never build a `tool_result` (`web_search`, `web_fetch`, `code_execution`, `tool_search`).
- **When Claude uses tools:** With default `tool_choice: {"type": "auto"}`, Claude decides per turn whether to call a tool or respond directly. Steerable via system prompt (e.g. `"Always call a tool first before responding."`).
- **Trigger boundary:** each server tool's page documents its own.
- **Pricing:** based on (1) total input tokens incl. the `tools` parameter, (2) output tokens, (3) for server tools, additional usage-based charges. Token cost added by tool-use system prompt per model (e.g. Claude Opus 4.8: 290 tokens for `auto`/`none`, 410 for `any`/`tool`).

### API surface
Top-level request parameters: `model`, `max_tokens`, `messages`, `tools`, `tool_choice`, `system`, `stream`.

Minimal server-tool request:
```
POST /v1/messages
tools: [{"type": "web_search_20260209", "name": "web_search"}]
messages: [{"role": "user", "content": "..."}]
```

### Notes
- For MCP integration, see the MCP connector. For Managed Agents, see its own toolset.

---

## 2. Define Tools

**Summary** — Specify tool schemas, write descriptions, and control when Claude calls tools.

### Main concepts
- Tools are declared in the top-level `tools` array. The API constructs a special tool-use system prompt from tool definitions + the user's system prompt.
- Effective descriptions are 3–4+ sentences; consolidate related operations into one tool with an `action` parameter; namespace names (`github_list_prs`).

### API surface — tool definition fields

| Field | Type | Required | Notes |
|---|---|---|---|
| `name` | string | yes | Regex `^[a-zA-Z0-9_-]{1,64}$` |
| `description` | string | yes | Plaintext; what the tool does, when to use, caveats |
| `input_schema` | JSON Schema object | yes | Defines expected parameters (`type`, `properties`, `required`, `enum`, etc.) |
| `input_examples` | array | no | Example input objects validating against `input_schema`; invalid → 400. User-defined & Anthropic-schema client tools only. ~20–50 tokens/example. |
| `cache_control` | object | no | `{"type": "ephemeral"}` for prompt caching |
| `strict` | boolean | no | `true` guarantees schema conformance |
| `defer_loading` | boolean | no | Exclude from system-prompt prefix (tool search) |
| `allowed_callers` | array | no | `["direct"]` and/or `["code_execution_20260120"]` |
| `eager_input_streaming` | boolean | no | Fine-grained streaming (user-defined tools only) |

### `tool_choice` options

| `tool_choice.type` | Behavior | Default? |
|---|---|---|
| `auto` | Claude decides whether to call any tool | default when `tools` provided |
| `any` | Must call one of the provided tools (not a particular one) | no |
| `tool` | Must call a specific tool; requires `name` | no |
| `none` | Must not call any tools | default when no `tools` |

- `disable_parallel_tool_use` (boolean) nests inside `tool_choice`.
- `tool`/`any` prefills the assistant message to force tool use; no natural-language text precedes `tool_use`.

### `tool_use` response block
```
{"type": "tool_use", "id": "toolu_...", "name": "get_weather", "input": {...}}
```

### Notes
- Extended thinking supports only `auto` and `none` (not `any`/`tool`).
- "Claude Mythos Preview" model does not support forced tool use.
- `strict: true` + `tool_choice: {"type": "any"}` guarantees both a tool call and schema validation.

---

## 3. Handle Tool Calls

**Summary** — Parse `tool_use` blocks, format `tool_result` blocks, and handle errors.

### Main concepts
- Client tools: `stop_reason: "tool_use"`, response `content` contains `tool_use` blocks. You execute and return a `tool_result` in the next `user` message.
- Server tools produce `server_tool_use` blocks; results follow inline paired by `tool_use_id`.
- Tool Runner (SDK) automates this loop.

### `tool_result` content block

| Field | Type | Required | Notes |
|---|---|---|---|
| `type` | string | yes | `"tool_result"` |
| `tool_use_id` | string | yes | matches a `tool_use` `id` |
| `content` | string \| array | no | string, or nested blocks: `text`, `image`, `document`, `search_result` |
| `is_error` | boolean | no | `true` signals an execution error |

### Notes
- `tool_result` blocks must immediately follow their corresponding `tool_use` blocks in message history.
- In the user message containing results, `tool_result` blocks must come **first**; any text after all results.
- A turn calling a server tool with no result block yet: user message must contain only `tool_result` blocks (text after results ends the turn; yields a 400 naming the unresolved server tool).
- Error on bad ordering: `"tool_use ids were found without tool_result blocks immediately after"`.
- Security: keep untrusted content (web pages, uploads) inside `tool_result`, not `system` or plain user text, to limit indirect prompt injection.

---

## 4. How Tool Use Works (the agentic loop)

**Summary** — The conceptual model of the tool-use contract, where tools execute, and when to use tools vs. prose.

### Main concepts
- **Agentic loop (client tools):** a `while` loop keyed on `stop_reason`:
  1. Send request with `tools` + user message.
  2. Claude responds with `stop_reason: "tool_use"` and `tool_use` blocks.
  3. Execute each tool; format `tool_result` blocks.
  4. Send a new request with original messages + assistant response + user message of `tool_result`s.
  5. Repeat while `stop_reason == "tool_use"`.
- **Loop exit stop reasons:** `"end_turn"`, `"max_tokens"`, `"stop_sequence"`, `"refusal"`.
- **Server-side loop:** has an iteration limit; exceeding it returns `stop_reason: "pause_turn"`. Resend the conversation (incl. the paused response) to continue.

### Notes
- Server loop also returns `stop_reason: "tool_use"` with a `server_tool_use` lacking a result block when Claude calls a server tool + client tool in the same parallel group — the server tool runs after you return client results.
- Tool use fits: actions with side effects, fresh/external data, guaranteed-shape structured outputs, calling existing systems.
- Heuristic: if you'd write a regex to extract a decision from model output, that decision should have been a tool call.

---

## 5. Parallel Tool Use

**Summary** — By default Claude may call multiple tools in a single response.

### Main concepts
- One assistant turn can contain multiple `tool_use` blocks; each matched by a `tool_result` keyed on `tool_use_id`.
- Execution order is your choice (concurrent, sequential, mixed). Independent read-only ops → parallel; side-effect/ordering-sensitive → sequential.
- Claude 4 models make parallel calls by default when beneficial.

### API surface
- `disable_parallel_tool_use` (boolean, nested in `tool_choice`):
  - with `auto` → at most one tool per response (may still answer in prose).
  - with `any`/`tool` → exactly one tool.
- Skipped/failed call: return a `tool_result` with `is_error: true` and a brief reason.

### Notes
- All `tool_result` blocks must be returned together in one user message, all before any text.
- Splitting results into separate user messages "teaches" Claude to avoid parallel calls (breaks parallelism).

---

## 6. Tool Use with Prompt Caching

**Summary** — Caching tool definitions across turns: where to place `cache_control` breakpoints, `defer_loading` cache preservation, and automatic server-tool-result caching.

### Main concepts
- Place `cache_control: {"type": "ephemeral"}` on the last tool in the `tools` array to cache the whole tool-definitions prefix.
- Deferred tools (`defer_loading: true`) are excluded from the prefix; discovered tools load as `tool_reference` blocks appended inline, preserving the cache.
- Cache follows a prefix hierarchy: `tools` → `system` → `messages`.

### Cache invalidation hierarchy

| Change | Invalidates |
|---|---|
| Modifying tool definitions | Entire cache (tools, system, messages) |
| Toggling web search or citations | System + messages caches |
| Changing `tool_choice` | Messages cache |
| Changing `disable_parallel_tool_use` | Messages cache |
| Toggling images present/absent | Messages cache |
| Changing thinking parameters | Messages cache |

### Automatic server-tool-result caching
- When prompt caching is enabled and Claude uses a server tool (web search/fetch/code execution), the API auto-places a cache breakpoint on the server tool result before the next iteration.
- Always uses the default **5-minute TTL**, regardless of your own markers.
- Appears in `usage` under `cache_creation.ephemeral_5m_input_tokens`.
- Only applies when ≥1 `cache_control` marker exists in the request.

### Notes
- `cache_control` + `defer_loading` cannot both be set on the same tool (returns 400). Put cache breakpoints on non-deferred tools.

---

## 7. Fine-Grained Tool Streaming

**Summary** — Delivers a tool's input as Claude generates it, without server-side buffering/JSON validation, reducing time-to-first-fragment for large parameters. ZDR-eligible.

### API surface
- `eager_input_streaming` (boolean, optional, per user-defined tool):
  - `true` → fine-grained streaming enabled.
  - omitted → standard buffered streaming (buffer + validate each parameter before streaming).
  - `false` → buffered even when the legacy beta header is sent.
- Legacy beta header: `anthropic-beta: fine-grained-tool-streaming-2025-05-14` (turns fine-grained on for tools with `eager_input_streaming` unset).

### Streaming events
- `content_block_start` with `type: "tool_use"` → `input: {}` (empty placeholder).
- `content_block_delta` with `type: "input_json_delta"` → `partial_json` (string fragment).
- `content_block_stop` → parse accumulated string.

### Accumulation contract (manual)
1. On `content_block_start` (`type: "tool_use"`), init `input_json = ""`.
2. Append `event.delta.partial_json` per `input_json_delta`.
3. On `content_block_stop`, parse the accumulated string (guard the parse).

### Handling invalid JSON
- On invalid/incomplete input, return a `tool_result` with `is_error: true` and `content` `{"INVALID_JSON": "<raw invalid input>"}` (build via a JSON library).

### Notes
- Requires streaming enabled on the request.
- `eager_input_streaming` is for user-defined tools only.
- Accumulated string may be invalid JSON; guard the parse. A `max_tokens` stop can cut a parameter mid-stream.
- All models support it on Claude API, AWS, Bedrock, Vertex AI, Foundry.

---

## 8. Manage Tool Context

**Summary** — Four approaches to context bloat at different pipeline stages: tool search, programmatic tool calling, prompt caching, context editing.

### Approaches

| Approach | What it reduces | When it fits |
|---|---|---|
| Tool search | Tool definitions loaded upfront | 20+ tools, most not needed each turn |
| Programmatic tool calling | `tool_result` roundtrips | Chains executable as a single script |
| Prompt caching | Token cost of repeated tool definitions | Stable toolsets across many requests |
| Context editing | Old `tool_result` blocks in history | Long conversations, early results irrelevant |

### Notes
- Approaches compose; no conflict using them together.
- Prompt-caching cache-write markup: 25% over base input pricing; breaks even on the 2nd cache-hit request.
- Tool search threshold: ~20+ tools.

---

## 9. Server Tools (shared mechanics)

**Summary** — Common mechanics for tools Anthropic executes server-side: `server_tool_use` blocks, `pause_turn`, mixed server+client turns, ZDR via `allowed_callers`, and domain filtering.

### Main concepts
- Server tools are executed by the API; the client does **not** return a `tool_result`. The result block (e.g. `web_search_tool_result`) follows the `server_tool_use` block paired by `tool_use_id`.
- The API runs server tools in a server-side agentic loop; long turns may pause with `stop_reason: "pause_turn"`. To continue, resend the paused assistant content unchanged + same `tools`.
- A `pause_turn` response never leaves a client `tool_use` waiting on you.

### `server_tool_use` block
```
{"type": "server_tool_use", "id": "srvtoolu_...", "name": "<tool>", "input": {...}}
```
- `id` uses the `srvtoolu_` prefix (distinct from client `toolu_`).

### Mixing server + client tools in one turn
- `stop_reason` is `"tool_use"` (NOT `"pause_turn"`).
- `content` has both a `server_tool_use` (no result yet) and a client `tool_use`.
- Detect by finding a `server_tool_use` whose `id` has no matching result block.
- To continue: run client tools; send a user message containing **only** `tool_result` blocks; keep the same `tools`. With programmatic tool calling, also pass back the `container` ID.
- `mcp_tool_use` blocks behave the same way.

### `allowed_callers` & ZDR
- Controls how a tool can be invoked:
  - `"direct"` — Claude calls directly.
  - `"code_execution_20260120"` — invoked from inside a code execution container.
  - Both can be listed.
- Basic `web_search_20250305` and `web_fetch_20250910` are ZDR-eligible. `_20260209`+ versions with dynamic filtering are **not** ZDR-eligible by default (they rely on internal code execution). Set `allowed_callers: ["direct"]` to bypass.
- `_20260209`+ web tools default to the code-execution caller only; earlier versions default to `["direct"]`.
- On models without programmatic tool calling, `_20260209`+ versions require `allowed_callers: ["direct"]` or a validation error.

### Domain filtering (`allowed_domains` / `blocked_domains`)
- Use one or the other, never both.
- Bare domains with optional path, no scheme (`example.com`, `example.com/blog`).
- Subdomains of `example.com` auto-included; a specific subdomain restricts to it.
- Web search supports subpaths; web fetch matches on domain only.
- Wildcards `*` allowed only in the path, not the domain. Invalid formats → 400 at request time.
- Request-level `allowed_domains` must be a subset of any org-level allowed list (org-blocked domains removed rather than erroring).
- ASCII-only domains (homograph attack warning).

### Dynamic filtering with code execution
- `_20260209`+ web search/fetch use code execution internally to apply dynamic filters; the API provisions it automatically. If you include a `code_execution` tool, use `code_execution_20260120`+ (older versions rejected alongside these web tools).
- Both tools share a single execution container when dynamic filtering runs.

### Notes
- `server_tool_use` blocks stream like client `tool_use`; result blocks arrive complete in a single `content_block_start`.
- All server tools support batch processing; hitting the per-turn iteration limit ends the turn with `stop_reason: "pause_turn"`.

---

## 10. Web Search Tool

**Summary** — Gives Claude real-time web access with cited sources. `_20260209`+ adds dynamic filtering (Claude writes/runs code to filter results before they reach context).

**Client/Server:** Server tool. ZDR-eligible (basic version).

### Tool versions (`type` string)
- `web_search_20250305` — basic.
- `web_search_20260209` — adds dynamic filtering.
- `web_search_20260318` — adds `response_inclusion` control.

### Tool definition parameters

| Parameter | Type | Required | Notes |
|---|---|---|---|
| `type` | string | yes | version string above |
| `name` | string | yes | must be `"web_search"` |
| `max_uses` | integer | no | limit searches per request; exceeding → error `max_uses_exceeded`. Factual: 1–3; research: 15–20 or omit. |
| `allowed_domains` | array | no | only these domains |
| `blocked_domains` | array | no | never these (cannot combine with `allowed_domains`) |
| `user_location` | object | no | localize. `type: "approximate"`, plus at least one of `city`, `region`, `country` (ISO 3166-1 alpha-2), `timezone` (IANA). |
| `allowed_callers` | array | no | defaults `["direct"]` on `20250305`; `["code_execution_20260120"]` on `20260209`+. |
| `response_inclusion` | string | no | requires `20260318`. `"full"` (default) or `"excluded"` (drop nested `server_tool_use`+result pairs consumed by completed code execution). |

### Content block shapes
`server_tool_use` input:
```
{"type": "server_tool_use", "id": "srvtoolu_...", "name": "web_search", "input": {"query": "..."}}
```
`web_search_tool_result` (success):
```
{"type": "web_search_tool_result", "tool_use_id": "srvtoolu_...", "content": [
  {"type": "web_search_result", "url": "...", "title": "...", "encrypted_content": "...", "page_age": "..."}
]}
```
- `encrypted_content` must be passed back verbatim in multi-turn conversations (missing/modified → 400).
Error (note: `content` is a single object, not a list, on error):
```
{"type": "web_search_tool_result", "tool_use_id": "...", "content": {"type": "web_search_tool_result_error", "error_code": "..."}}
```

### Citations
- Always enabled. Each `web_search_result_location` citation: `url`, `title`, `encrypted_index`, `cited_text` (≤150 chars). `cited_text`, `title`, `url` do NOT count toward token usage.

### Error codes
`too_many_requests`, `invalid_tool_input`, `max_uses_exceeded`, `query_too_long`, `request_too_large` (long domain filter list), `unavailable`. HTTP 200 still returned on tool errors.

### Pricing/usage
- **$10 per 1,000 searches** plus standard token costs for search-generated content.
- Each web search = one use regardless of result count. Failed searches not billed.
- Usage tracked under `usage.server_tool_use.web_search_requests`.

### Notes
- Citations required when displaying outputs to end users.
- Dynamic filtering available on Claude Fable 5, Opus 4.8, Mythos 5, Opus 4.7/4.6, Sonnet 5/4.6. Not on Amazon Bedrock; Google Cloud gets basic only; Microsoft Foundry requires Hosted on Anthropic.
- Web search can be disabled per-org in Claude Console (request fails 400 if disabled).

---

## 11. Web Fetch Tool

**Summary** — Retrieve full content from specified web pages and PDFs and insert it into the conversation. `_20260209`+ adds dynamic filtering.

**Client/Server:** Server tool. ZDR-eligible (basic version).

### Tool versions
- `web_fetch_20250910` — basic fetch.
- `web_fetch_20260209` — adds dynamic filtering.
- `web_fetch_20260309` — adds `use_cache` (cache bypass).
- `web_fetch_20260318` — adds `response_inclusion`.

### Tool definition parameters

| Parameter | Type | Required | Notes |
|---|---|---|---|
| `type` | string | yes | version string |
| `name` | string | yes | must be `"web_fetch"` |
| `max_uses` | integer | no | limit fetches per request; failed fetches count against the limit |
| `allowed_domains` | array | no | only fetch from these |
| `blocked_domains` | array | no | never fetch from these (cannot combine with `allowed_domains`) |
| `citations` | object | no | `{"enabled": true}` — citations disabled by default (optional for web fetch) |
| `max_content_tokens` | integer | no | max content length in tokens; truncated if exceeded (text only; approximate) |
| `use_cache` | boolean | no | requires `20260309`+. Default `true`; `false` bypasses cache (fresh fetch, more latency) |
| `response_inclusion` | string | no | requires `20260318`. `"full"` or `"excluded"`. |
| `allowed_callers` | array | no | same semantics as web search |

### Content block shapes
`server_tool_use` input:
```
{"type": "server_tool_use", "id": "srvtoolu_...", "name": "web_fetch", "input": {"url": "..."}}
```
`web_fetch_tool_result` (text/HTML):
```
{"type": "web_fetch_tool_result", "tool_use_id": "...", "content": {
  "type": "web_fetch_result", "url": "...", "content": {
    "type": "document", "source": {"type": "text", "media_type": "text/plain", "data": "..."}, "title": "...", "citations": {"enabled": true}
  }, "retrieved_at": "2025-08-25T10:30:00Z"
}}
```
- PDF result: `source.type` is `"base64"`, `media_type` `"application/pdf"`, `data` base64-encoded.

### Error codes
`invalid_tool_input` (malformed URL / non-HTTP(S)), `url_too_long` (>250 chars), `url_not_allowed` (blocked by domain filtering / private addresses / robots.txt), `url_not_in_prior_context`, `url_not_accessible` (HTTP error), `too_many_requests`, `unsupported_content_type` (only text/HTML/PDF), `max_uses_exceeded`, `unavailable`.

### URL validation (security)
- Can only fetch URLs that previously appeared in conversation context (user messages, client tool results, prior search/fetch results). Cannot fetch arbitrary URLs Claude generates, or URLs from container-based server tools.

### Pricing/usage
- **No additional charges** beyond standard token costs for fetched content.
- Usage tracked under `usage.server_tool_use.web_fetch_requests`. Example costs: ~2,500 tokens (10 kB page), ~25,000 (100 kB doc), ~125,000 (500 kB PDF).

### Notes
- Does not support JavaScript-rendered sites.
- Results cached; use `use_cache: false` for fresh content.
- Claude API, Claude Platform on AWS, Microsoft Foundry (Hosted on Anthropic). Not Bedrock/Google Cloud.

---

## 12. Code Execution Tool

**Summary** — Run Bash commands and manipulate files (incl. writing Python) in a secure sandboxed container. Also powers dynamic filtering in web search/fetch. `_20260120`+ adds REPL state persistence and programmatic tool calling.

**Client/Server:** Server tool. **Not ZDR-eligible** (data retained up to 30 days).

### Tool versions
- `code_execution_20250825` — Bash + file operations.
- `code_execution_20260120` — adds REPL state persistence + programmatic tool calling. (Haiku 4.5 accepts this version but the features aren't available — behaves like `20250825`.)
- `code_execution_20260521` — same runtime; tool description tells Claude about the 90-second per-Python-cell wall-clock limit (programmatic tool calling). A cell exceeding the limit returns a normal result with non-zero `return_code` and a `detection_timeout` status message.
- Legacy `code_execution_20250522` (Python only) — migrate. Beta header `code-execution-2025-05-22`.

All current versions are GA (no `anthropic-beta` header required).

### Tool definition
```
{"type": "code_execution_20250825", "name": "code_execution"}
```
- Both fields fixed; `name` must be `"code_execution"`. No additional parameters.
- When provided, Claude gains two sub-tools automatically:
  - `bash_code_execution` — run shell commands. Input: `{"command": "..."}`.
  - `text_editor_code_execution` — view/create/edit files. Input varies: `view` (`command`, `path`), `create` (`command`, `path`, `file_text`), `str_replace` (`command`, `path`, `old_str`, `new_str`).

### Content block shapes
`bash_code_execution_tool_result`:
```
{"type": "bash_code_execution_tool_result", "tool_use_id": "srvtoolu_...", "content": {
  "type": "bash_code_execution_result", "stdout": "...", "stderr": "...", "return_code": 0, "content": []
}}
```
- `content[]`: one entry per file the command created, each with a `file_id` for Files API retrieval.
- File op results: `view` (`file_type`, `content`, `num_lines`, `start_line`, `total_lines`), `create` (`is_file_update`), `str_replace` (`old_start`, `old_lines`, `new_start`, `new_lines`, `lines`).

### Error codes
`unavailable`, `execution_time_exceeded`, `invalid_tool_input`, `too_many_requests`; bash also `output_file_too_large`; text_editor also `file_not_found`.

### Containers
- Response includes top-level `container` with `id` and `expires_at`.
- Reuse: pass `container: <id>` (top-level) in the next request. Files persist; with `20260120`+, Python interpreter state persists.
- Idle containers reclaimed ~5 min; sending the ID within 30 days restores it. Expired containers error (retry without the `container` param). Containers scoped to the API key's workspace.

### Files API integration
- Requires beta header `anthropic-beta: files-api-2025-04-14`.
- Upload via Files API; reference with a `container_upload` block `{"type": "container_upload", "file_id": ...}`.
- Supported: CSV, Excel, JSON, XML, images (JPEG/PNG/GIF/WebP), text files. Files Claude creates appear in result `content[].file_id`.

### Runtime / limits
- Python 3.11; Linux x86_64 container. 5 GiB RAM, 5 GiB disk, 1 CPU.
- Networking fully disabled; file access limited to workspace dir.
- Max per-invocation execution time (else `execution_time_exceeded`); programmatic cells: 90s wall-clock.
- Containers expire 30 days after creation; data retained up to 30 days.
- No runtime package installation. Pre-installed: pandas, numpy, scipy, scikit-learn, statsmodels, matplotlib, seaborn, pyarrow, openpyxl, pillow, python-pptx, python-docx, pypdf, pdfplumber, sympy, mpmath, tqdm, joblib; CLI tools unzip, unrar, 7zip, bc, rg, fd, sqlite.

### Pricing
- **Free when used with web search/fetch** `_20260209`+ (covers dynamic filtering + Claude's direct code).
- Otherwise: billed by execution time. Minimum 5 min per invocation; **1,550 free hours/month per org**; **$0.05/hour/container** beyond. If files included in the request, execution time billed even if the tool isn't invoked.
- Usage tracked under `usage.server_tool_use.code_execution_requests`.

### Notes
- Platform: Claude API, AWS, Foundry (Hosted on Anthropic). Not Bedrock/Google Cloud.
- Migration from `20250522`: response types `code_execution_result` → `bash_code_execution_result`, `text_editor_code_execution_*_result`.

---

## 13. Computer Use Tool

**Summary** — Beta feature: screenshot capture + mouse/keyboard control of a desktop environment for autonomous desktop interaction. Claude emits requests; **your application** executes them against a sandboxed X11 environment.

**Client/Server:** **CLIENT tool.** ZDR-eligible (your app controls storage). Anthropic processes screenshots in real time.

### Beta headers (required)
- `anthropic-beta: computer-use-2025-11-24` — Claude Sonnet 5, Opus 4.8/4.7/4.6, Sonnet 4.6, Opus 4.5.
- `anthropic-beta: computer-use-2025-01-24` — Claude Sonnet 4.5, Haiku 4.5, Opus 4.1, Sonnet 4, Opus 4.
- Only required for the computer use tool itself.

### Tool versions
- `computer_20250124`
- `computer_20251124`

### Tool definition parameters

| Parameter | Type | Required | Notes |
|---|---|---|---|
| `type` | string | yes | version string |
| `name` | string | yes | must be `"computer"` |
| `display_width_px` | integer | yes | display width in pixels |
| `display_height_px` | integer | yes | display height in pixels |
| `display_number` | integer | no | display number for X11 |
| `enable_zoom` | boolean | no | `20251124` only; default `false`. Enables the `zoom` action. |

- Schema-less: no `input_schema`; built into the model.

### Available actions
- Basic (all versions): `screenshot`, `left_click` (`[x,y]`), `type` (text), `key` (e.g. `ctrl+s`), `mouse_move`.
- Enhanced (`20250124`+): `scroll` (direction + amount), `left_click_drag`, `right_click`, `middle_click`, `double_click`, `triple_click`, `left_mouse_down`, `left_mouse_up`, `hold_key` (key + duration seconds), `wait`.
- `20251124` only: `zoom` — view a region at full resolution (requires `enable_zoom: true`); takes `region` `[x1,y1,x2,y2]`.

### Content block shapes
Standard client tool flow:
```
tool_use: {"type": "tool_use", "id": "toolu_...", "name": "computer", "input": {"action": "...", "coordinate": [x,y], ...}}
tool_result: {"type": "tool_result", "tool_use_id": "toolu_...", "content": "..."}
```

### Pricing / token usage
- Computer use beta adds **466–499 tokens** to the system prompt.
- Tool definition: **735 input tokens** for Claude 4.x models.
- Plus screenshot images (Vision pricing) and tool execution results.

### Notes
- Slow vs. human actions; use where speed isn't critical.
- Claude may hallucinate coordinates; extended thinking helps.
- Screenshot long-edge limits: Sonnet 5/Opus 4.8/4.7 up to 2576 px; earlier models up to 1568 px (~1.15 megapixels). Images >8000 px on a side are rejected, not downscaled.
- macOS Retina: device pixel ratio 2; downscale screenshot by 2x or halve returned coordinates.
- Place instruction text before the screenshot image in a user turn's `content` array.
- Reference implementation: `github.com/anthropics/anthropic-quickstarts/computer-use-demo`.

---

## 14. Bash Tool

**Summary** — A client tool: Claude requests shell commands; your app runs them in a persistent bash session it owns, returning output as tool results. One long-lived bash process persists state (cwd, env vars, files) across calls.

**Client/Server:** **CLIENT tool.** ZDR-eligible.

### Tool versions
- `bash_20250124` — current; no beta header. Accepted by every model from Sonnet 3.7 onward.
- `bash_20241022` — original; part of computer use beta. Only the retired Oct-2024 Sonnet 3.5 accepts it. Requires `anthropic-beta: computer-use-2024-10-22`. SDKs expose it only in beta namespaces. New integrations should use `20250124`.

### Tool definition
```
{"type": "bash_20250124", "name": "bash"}
```
- Schema-less (built into the model). `name` must be `"bash"`.

### Input fields Claude sets

| Parameter | Required | Notes |
|---|---|---|
| `command` | yes* | The bash command (*unless using `restart`) |
| `restart` | no | `true` to restart the bash session (clean cwd/env/processes) |

### Content block shapes
Standard client tool flow:
```
tool_use: {"type": "tool_use", "id": "toolu_...", "name": "bash", "input": {"command": "ls *.py"}}
tool_result: {"type": "tool_result", "tool_use_id": "toolu_...", "content": "..."}
```
- Error handling: `is_error: true` on `tool_result` with the message in `content`.
- Multiple `tool_use` blocks per response: run in order in the same session; return all results in one user message.

### Pricing / token usage
Tool definition adds input tokens on top of per-model tool-use system prompt:
| Model | Additional input tokens |
|---|---|
| Opus 4.7 / 4.8 | 325 |
| Opus 4.6, Sonnet 4.6, and earlier | 244 |

### Notes
- No interactive commands (vim, less, password prompts). No GUI apps; CLI only.
- API doesn't truncate tool results (oversized requests rejected); truncate large outputs in your app.
- No streaming: output reaches Claude only via the next `tool_result`.
- Security: run in an isolated container/VM as least-privileged user; use an allowlist (not blocklist); set resource limits (ulimit); log/redact credentials.

---

## 15. Text Editor Tool

**Summary** — An Anthropic-schema client tool for viewing and modifying text files to debug, fix, and improve code.

**Client/Server:** **CLIENT tool.** ZDR-eligible.

### Tool versions
- `text_editor_20250728` — current; no beta header. Conventional `name`: `str_replace_based_edit_tool`.

### Tool definition
```
{"type": "text_editor_20250728", "name": "str_replace_based_edit_tool"}
```
- Schema-less (built into the model). `name` is conventionally `str_replace_based_edit_tool`.

### Commands (input dispatched on `command`)

| Command | Parameters | Notes |
|---|---|---|
| `view` | `command`, `path`, `view_range`? | Examine file or list directory. `view_range`: `[start_line, end_line]`, 1-indexed; `-1` end means to end of file. Applies to files only. |
| `str_replace` | `command`, `path`, `old_str`, `new_str` | Replace a specific string (must match exactly incl. whitespace). |
| `create` | `command`, `path`, `file_text` | Create a new file with specified content. |
| `insert` | `command`, `path`, `insert_line`, `insert_text` | Insert text at a specific line. |

### Content block shapes
Standard client tool flow:
```
tool_use: {"type": "tool_use", "id": "toolu_...", "name": "str_replace_based_edit_tool", "input": {"command": "str_replace", "path": "primes.py", "old_str": "...", "new_str": "..."}}
tool_result: {"type": "tool_result", "tool_use_id": "toolu_...", "content": "Successfully replaced text at exactly one location."}
```

### Notes
- Pairs with the Bash tool as the canonical coding dev loop (inspect → edit → run tests → repeat).
- When used alongside Code execution, two separate execution environments exist (state not shared) — prompt Claude to distinguish them.

---

## 16. Memory Tool

**Summary** — Lets Claude store and retrieve information across conversations in a directory of memory files under `/memories`. Supports just-in-time context retrieval. Claude only requests file operations; your app executes them against storage you control.

**Client/Server:** **CLIENT tool.** ZDR-eligible.

### Tool definition
```
{"type": "memory_20250818", "name": "memory"}
```
- Generally available; no beta header required. `name` must be `"memory"`. Schema-less.
- Available on all Claude 4 and later models. (SDK helpers live in beta namespaces despite GA status.)

### Commands (input dispatched on `command`)
All paths must be within `/memories` (your handler validates this).

| Command | Parameters | Returns / Errors |
|---|---|---|
| `view` | `command`, `path`, `view_range`? | Directory listing (up to 2 levels, excludes hidden/`node_modules`) or file contents with line numbers. Files >999,999 lines error. Displays `.jpg/.jpeg/.png`; truncates text >16,000 chars. |
| `create` | `command`, `path`, `file_text` | `"File created successfully at: {path}"`. Error if exists: `"Error: File {path} already exists"`. |
| `str_replace` | `command`, `path`, `old_str`, `new_str`? | `"The memory file has been edited."` + snippet. Errors: not found, not found, duplicate occurrence. `new_str` optional (omit = delete). |
| `insert` | `command`, `path`, `insert_line`, `insert_text` | `"The file {path} has been edited."`. `insert_line` 0..n_lines. Errors: not found, invalid line. |
| `delete` | `command`, `path` | `"Successfully deleted {path}"`. Reject delete of `/memories` root. |
| `rename` | `command`, `old_path`, `new_path` | `"Successfully renamed {old_path} to {new_path}"`. Errors: source missing, destination exists (no overwrite). Reject rename of `/memories` root. |

### Content block shapes
Standard client tool flow:
```
tool_use: {"type": "tool_use", "id": "toolu_...", "name": "memory", "input": {"command": "view", "path": "/memories"}}
tool_result: {"type": "tool_result", "tool_use_id": "toolu_...", "content": "..."}
```

### Auto system prompt
When the memory tool is present, the API auto-adds a MEMORY PROTOCOL instruction (always view the memory directory first; assume context may be reset at any moment).

### Security considerations
- Your app executes every file operation; safeguards are your responsibility:
  - Validate sensitive info; strip where needed.
  - Cap memory file sizes and `view` return size; page with `view_range`.
  - Periodically delete stale memory files.
  - **Path traversal protection**: validate every path starts with `/memories`; resolve to canonical form; reject `../`, `..\`, URL-encoded `%2e%2e%2f`.

### Notes
- `/memories` is a prefix your handler maps to real storage (per-user dir, DB keys, cloud storage, encrypted files).
- Claude automatically checks its memory directory before starting a task.
- Pairs with context editing/compaction for long-running conversations.
- Not separately metered (standard tool use; tool definition counts as input tokens).

---

## 17. Advisor Tool

**Summary** — Beta: lets a faster, lower-cost **executor model** consult a higher-intelligence **advisor model** mid-generation for strategic guidance. Close to advisor-solo quality while bulk generation happens at executor-model rates. Fits long-horizon agentic workloads.

**Client/Server:** **Server tool.** ZDR-eligible. The advisor runs a separate server-side inference pass (no tools, no context management; thinking blocks dropped before result returns) within a single `/v1/messages` request.

### Beta header
`anthropic-beta: advisor-tool-2026-03-01` (via `betas=[...]` on the beta client).

### Tool version
`advisor_20260301` (the `type` string).

### Tool definition parameters

| Parameter | Type | Default | Notes |
|---|---|---|---|
| `type` | string | *required* | `"advisor_20260301"` |
| `name` | string | *required* | `"advisor"` |
| `model` | string | *required* | Advisor model ID (e.g. `claude-fable-5`); billed at this model's rates |
| `max_uses` | integer | unlimited | Max advisor calls per request; exceeding → `max_uses_exceeded` |
| `max_tokens` | integer | advisor output cap | Caps advisor total output (thinking + text) per call. Minimum 1024; above the advisor's own cap → 400. Recommended start: 2048. |
| `caching` | object | `null` | `{"type": "ephemeral", "ttl": "5m" \| "1h"}` — enables advisor-side prompt caching across calls |

Also accepts `cache_control`, `allowed_callers`, `defer_loading`, `strict`.

### Content block shapes
`server_tool_use` (input always empty):
```
{"type": "server_tool_use", "id": "srvtoolu_...", "name": "advisor", "input": {}}
```
`advisor_tool_result` (success) — discriminated union on `content.type`:
| Variant | Fields | Returned when |
|---|---|---|
| `advisor_result` | `text`, `stop_reason`? | Advisor returns plaintext (e.g. Opus 4.8) |
| `advisor_redacted_result` | `encrypted_content`, `stop_reason`? | Advisor returns encrypted output (Fable 5, Mythos 5) |

- `stop_reason` present only when you set `max_tokens`; typically `"end_turn"` or `"max_tokens"` when cap hit. When cap hit, API appends `[Advisor output truncated at max_tokens=2048.]`.

Error result:
```
{"type": "advisor_tool_result", "tool_use_id": "...", "content": {"type": "advisor_tool_result_error", "error_code": "..."}}
```
Error codes: `max_uses_exceeded`, `too_many_requests` (advisor rate-limited), `overloaded`, `prompt_too_long`, `execution_time_exceeded`, `unavailable`. Executor sees the error and continues; the request itself does not fail.

### Streaming
- Advisor sub-inference does not stream. `server_tool_use` signals start; pause begins at `content_block_stop`. SSE `ping` keepalives ~every 30s. `advisor_tool_result` arrives fully formed in a single `content_block_start` (no deltas).

### Usage and billing
- Advisor calls run as a separate sub-inference billed at the advisor model's rates. Reported in `usage.iterations[]`:
  - `{"type": "message", ...}` — executor iterations.
  - `{"type": "advisor_message", "model": "...", ...}` — advisor iterations.
- Top-level `usage` reflects executor tokens only (advisor tokens NOT rolled in). `output_tokens` = sum of executor iterations; `input_tokens`/`cache_read_input_tokens` = first executor iteration only.
- Top-level `max_tokens` bounds executor only; advisor draws from no task budget.
- Typical advisor output: 400–700 text tokens, or 1,400–1,800 total incl. thinking.

### Advisor prompt caching (two layers)
1. **Executor-side:** `advisor_tool_result` is cacheable like any content block; `cache_control` breakpoint after it.
2. **Advisor-side:** set `caching` on the tool definition. Breaks even at ~3 calls/conversation. `clear_thinking` with `keep != "all"` shifts the advisor's quoted transcript each turn (cache misses, cost only). Default on earlier Opus/Sonnet & all Haiku: `keep: {type: "thinking_turns", value: 1}`; on Opus 4.5+/Sonnet 4.6+ default keeps all turns. Set `keep: "all"` to preserve cache stability.

### Model compatibility (executor ↔ advisor)
Advisor must be Sonnet 4.6 or more capable and at least as capable as the executor.

| Executor | Allowed advisors |
|---|---|
| Haiku 4.5 / Sonnet 4.6 | Fable 5, Mythos 5, Opus 4.8/4.7/4.6, Sonnet 4.6 |
| Sonnet 5 / Opus 4.7 | Fable 5, Mythos 5, Opus 4.8/4.7 |
| Opus 4.6 | Fable 5, Mythos 5, Opus 4.8/4.7/4.6 |
| Opus 4.8 | Fable 5, Mythos 5, Opus 4.8/4.7 |
| Fable 5 | Fable 5 only |
| Mythos 5 | Mythos 5 only |

Invalid pair → 400 `invalid_request_error`.

### Notes
- `tool_choice: {"type": "tool", "name": "advisor"}` forces a consult. Cannot combine with extended thinking (400).
- Omitting the advisor tool while history still has `advisor_tool_result` blocks → 400.
- `pause_turn` resume: append paused assistant message unchanged (keep `server_tool_use`); resend with same tool + beta header; no user message or `tool_result` needed.
- Platform: Claude API and Claude Platform on AWS only (beta).

---

## 18. Tool Search Tool

**Summary** — Lets Claude work with hundreds/thousands of tools by discovering and loading them on demand, rather than loading all definitions up front. Searches the catalog (names, descriptions, argument names/descriptions) and loads only what it needs. Reduces context by >85% and improves selection accuracy past ~30–50 tools.

**Client/Server:** **Server tool** (built-in variants); custom client-side implementations also possible. ZDR-eligible.

### Tool versions

| `type` string | `name` | Query mechanism | Max length |
|---|---|---|---|
| `tool_search_tool_regex_20251119` | `tool_search_tool_regex` | Claude constructs Python regex (case-insensitive) | 200 chars (`pattern`) |
| `tool_search_tool_bm25_20251119` | `tool_search_tool_bm25` | Natural-language queries (BM25 ranking) | 500 chars (`query`) |

- No beta header (GA). Available on Fable 5, Mythos 5, Opus 4.8/4.7/4.6, Sonnet 4.6, Opus 4.5, Sonnet 4.5, Haiku 4.5. Not on Opus 4.1 or earlier.

### Tool definition
```
{"type": "tool_search_tool_regex_20251119", "name": "tool_search_tool_regex"}
```
No additional parameters on the tool search tool itself.

### Deferred tool loading (`defer_loading`)
Add `"defer_loading": true` to tools that shouldn't load up front:
```
{"name": "get_weather", "description": "...", "input_schema": {...}, "defer_loading": true}
```
- You still send every tool's full definition in `tools` on every request (server needs them to run search and expand `tool_reference` blocks).
- At least one tool (normally the tool search tool) must stay non-deferred.
- Never set `defer_loading: true` on the tool search tool itself.
- Keep 3–5 most frequently used tools non-deferred.
- A tool with `defer_loading: true` cannot also carry `cache_control` (returns 400). `defer_loading` + strict mode compose without grammar recompilation.
- For MCP: set `defer_loading` on the `mcp_toolset`'s `default_config` (whole server) or per tool in `configs`.

### Content block shapes
`server_tool_use` input (regex uses `pattern`; BM25 uses `query`):
```
{"type": "server_tool_use", "id": "srvtoolu_...", "name": "tool_search_tool_regex", "input": {"pattern": "weather"}}
```
`tool_search_tool_result` (success):
```
{"type": "tool_search_tool_result", "tool_use_id": "srvtoolu_...", "content": {
  "type": "tool_search_tool_search_result", "tool_references": [{"type": "tool_reference", "tool_name": "get_weather"}]
}}
```
- API **automatically expands** `tool_reference` blocks into full tool definitions before showing them to Claude. Empty `tool_references` (no match) is not an error.
- **Never** return a `tool_result` for the `srvtoolu_...` ID.

Error result (HTTP 200):
```
{"type": "tool_search_tool_result", "tool_use_id": "...", "content": {
  "type": "tool_search_tool_result_error", "error_code": "invalid_tool_input", "error_message": "..."
}}
```
Error codes: `invalid_tool_input` (malformed regex or over length limit), `unavailable`, `too_many_requests`, `execution_time_exceeded`.

### Custom client-side implementation
Return `tool_reference` blocks from a custom tool's `tool_result`:
```
{"type": "tool_result", "tool_use_id": "toolu_...", "content": [{"type": "tool_reference", "tool_name": "discovered_tool_name"}]}
```
- Each referenced tool must have a definition in `tools` (normally `defer_loading: true`). Enables embedding-based retrieval.

### Continuing the conversation
- Pass assistant content back unchanged (incl. `server_tool_use` + `tool_search_tool_result`).
- Add your `tool_result` for the discovered tool in a user message; send the same `tools` array.
- API expands `tool_reference` blocks throughout history, so discovered tools reuse on later turns without re-searching.

### Limits / pricing
- Max deferred tools: 10,000. Search results: up to 5 per search. `input_examples` work with tool search.
- Not metered as a separate server tool (`usage.server_tool_use` has no tool-search field); loaded definitions count as input tokens.
- Bedrock: server-side tool search via InvokeModel API only (not Converse). Claude Platform on AWS: works like the Claude API.

### When to use
10+ tools; >10k tokens of definitions; selection accuracy drops; 200+ MCP tools; growing library. Not worth it for <10 tools or <100 tokens of definitions.

---

## 19. Tool Runner (SDK)

**Summary** — An SDK helper that handles the agentic loop, error wrapping, and type safety automatically, so you don't manually manage tool calls/results/state. Not itself a tool; an SDK convenience layer over client tool calling.

### Availability
Beta, in the Python, TypeScript, C#, Go, Java, PHP, and Ruby SDKs.

### Basic usage (Python)
- Define tools with the `@beta_tool` decorator (or `@beta_async_tool`); the decorator inspects args + docstring to derive JSON schema.
- Construct a runner:
  ```python
  runner = client.beta.messages.tool_runner(
      model="claude-opus-4-8", max_tokens=1024, tools=[...], messages=[...]
  )
  ```
- Iterable: yields messages; on each iteration it checks for tool calls, runs the tool, sends the result back, yields the next message. `runner.until_done()` returns the final message.

### Request parameters

| Parameter | Type | Required | Notes |
|---|---|---|---|
| `model` | string | yes | Model ID |
| `max_tokens` | integer | yes | Output token cap |
| `tools` | array | yes | Tool definitions / SDK tool objects |
| `messages` | array | yes | Initial conversation |
| `max_iterations` | integer | no | Loop cap (all 7 SDKs) |
| `stream` | boolean | no | `true` → returns a `BetaMessageStream` per iteration |

### Key methods
| Method | Purpose |
|---|---|
| `generate_tool_call_response()` / `generateToolResponse()` | Inspect/compute the tool result before sending; intercept errors, add `cache_control`, etc. |
| `append_messages(*messages)` | Flags state modified; runner skips auto-append for this iteration. |
| `set_messages_params(lambda)` | Change request params (e.g. `max_tokens`) without taking over history. |

### Tool error handling
- On throw, the runner returns the error to Claude as a `tool_result` with `is_error: true` carrying the exception's message (not full stack trace).
- Python SDK logs the full exception via `logging`; Python/TypeScript/Java read `ANTHROPIC_LOG` env var (`info`/`debug`). Go/Ruby/C#/PHP do not — catch/log inside the tool function.

### Automatic context management
- Python, TypeScript, Ruby runners support client-side compaction (deprecated in favor of server-side context editing).
- Go, Java, C#, PHP runners do not. Server-side context editing available in every SDK.

### Streaming
Set `stream=True`; iterate events; `get_final_message()` for the accumulated message.

### Notes
- Add `cache_control: {"type": "ephemeral"}` to cache tool results (use the append pattern to prevent auto-append of the original).
- Not separately metered; standard Messages API / tool-use pricing applies.

---

## 20. Programmatic Tool Calling

**Summary** — Lets Claude write code that calls your tools programmatically **within a code execution container**, rather than requiring a model round-trip per tool invocation. Reduces latency and cuts tokens (intermediate results never enter Claude's context). Reported: +11% avg performance with 24% fewer input tokens on BrowseComp/DeepSearchQA; ~38% fewer billed tokens on a 75-tool benchmark.

**Client/Server:** **Server-side** execution inside the code execution container. Requires code execution enabled. **Not ZDR-eligible** (data retained up to 30 days).

### Model compatibility
Requires `code_execution_20260120`+. Supported on Fable 5, Mythos 5, Opus 4.8/4.7/4.6, Sonnet 5/4.6, Opus 4.5, Sonnet 4.5. Platform: Claude API, AWS, Foundry (Hosted on Anthropic). Not Bedrock/Google Cloud.

### `allowed_callers` field (per tool definition)

| Value | Meaning |
|---|---|
| `["direct"]` | Claude calls directly (default if omitted) |
| `["code_execution_20260120"]` | Claude calls only from within code execution |
| `["direct", "code_execution_20260120"]` | Either |

- `code_execution_20260120` and `code_execution_20260521` both accepted (interchangeable); response blocks always tag the caller as `code_execution_20260120`.
- `allowed_callers` is **NOT** a hard security boundary — it controls presentation, validated against `tool_choice`. Do not rely on it for security.

### `caller` field in `tool_use` blocks
Direct invocation:
```
{"type": "tool_use", "id": "toolu_abc123", "name": "query_database", "input": {...}, "caller": {"type": "direct"}}
```
Programmatic invocation:
```
{"type": "tool_use", "id": "toolu_xyz789", "name": "query_database", "input": {...},
 "caller": {"type": "code_execution_20260120", "tool_id": "srvtoolu_abc123"}}
```
- `caller.tool_id`: the `id` of the code execution `server_tool_use` block that made the call (use to match a programmatic `tool_use` to its code execution run).

### Content block shapes
Initial response (Claude writes code, a programmatic call pauses):
```
[
  {"type": "text", "text": "I'll query..."},
  {"type": "server_tool_use", "id": "srvtoolu_abc123", "name": "code_execution", "input": {"code": "rows = json.loads(await query_database({'sql': ''}))\n..."}},
  {"type": "tool_use", "id": "toolu_def456", "name": "query_database", "input": {"sql": ""},
   "caller": {"type": "code_execution_20260120", "tool_id": "srvtoolu_abc123"}}
]
```
- Response includes top-level `container` with `id` and `expires_at`. `stop_reason: "tool_use"`.

`tool_result` you send back (content must be a string or `text` blocks):
```
{"type": "tool_result", "tool_use_id": "toolu_def456", "content": "[{\"customer_id\": \"C1\", ...}]"}
```
- When pending programmatic tool calls exist, the user message must contain **only** `tool_result` blocks (no text, even after results).

Code execution completion:
```
{"type": "code_execution_tool_result", "tool_use_id": "srvtoolu_abc123", "content": {
  "type": "code_execution_result", "stdout": "...", "stderr": "", "return_code": 0, "content": []
}}
```
Timeout error inside code execution result:
```
{"type": "code_execution_result", "stdout": "", "stderr": "TimeoutError: Calling tool ['query_database'] timed out (no response after 270s).", "return_code": 0, "content": []}
```

### How it works
1. Claude writes Python code that invokes tools as async functions (possibly multiple calls + pre/post-processing).
2. Claude runs the code in a sandboxed container via code execution.
3. When a tool function is called, code execution **pauses**; the API returns a `tool_use` block (with `caller` naming the run).
4. You provide the tool result; code execution continues (intermediate results NOT loaded into Claude's context).
5. Once all code completes, Claude receives the final output and continues.
- Tools allowing a code execution caller are exposed as async Python functions (parallelizable via `asyncio.gather`); each takes a single dict, returns a string.

### Container lifecycle
- New container per request unless reused. Reuse: pass the container ID back. **While a programmatic tool call waits, the container ID is REQUIRED** on the continuation request (API rejects without it).
- Idle containers reclaimed ~5 min; none reused >30 days after creation.
- A pending programmatic call **times out after ~4 minutes** (raises `TimeoutError` inside the code).
- With `code_execution_20260521`, each REPL cell has a **90-second wall-clock limit** (else `detection_timeout` status, non-zero `return_code`).

### Constraints
| Constraint | Detail |
|---|---|
| Structured outputs | `strict: true` tools not supported |
| `tool_choice` | Cannot force programmatic calling of a specific tool; naming a tool whose `allowed_callers` omits `"direct"` → 400 |
| Parallel tool use | `disable_parallel_tool_use: true` not supported |
| Recursive `$ref` | Custom tools with a recursive `$ref` cycle → 400 `Circular $ref detected`. Workarounds: keep direct-only, unroll to fixed depth, or replace with `{"type": "object"}`. |
| MCP connector tools | Cannot be called programmatically |
| Message formatting | When pending programmatic calls wait, user message must contain only `tool_result` blocks; each `content` must be string/`text` blocks (image/document rejected). Regular client tool calls can include text after results. |
| Tool result content | Beware code injection if output will be interpreted/executed as code. |

### `stop_reason` values
- `"tool_use"` — programmatic (or mixed direct) call pending; return `tool_result`(s) + container ID.
- `"end_turn"` — code execution completed; final text delivered.
- `"pause_turn"` — long-running server-side loop paused; resend paused assistant content unchanged (with container ID if a programmatic call is in flight).

### Pricing / token efficiency
- Same as code execution: free with web search/fetch `_20260209`+; otherwise billed by execution time (min 5 min/invocation, 1,550 free hours/org/month, $0.05/hour/container beyond).
- **Tool results from programmatic calls do NOT count toward input/output token usage** — only the final code execution result and Claude's response count.
- Three efficiency mechanisms: programmatic results not added to context; intermediate processing in code (no model tokens); multiple calls in one execution reduce overhead.
- Strong fit: fan-out/parallel across items; large results to filter/aggregate; agentic search/retrieval. Weak fit: strictly sequential workflows; few small tool calls (esp. first turn); tools needing immediate user feedback between calls.

### Notes
- Beta headers: none specific (underlying code execution versions are GA), except `files-api-2025-04-14` when using Files API with code execution.

---

## 21. Tool Combinations

**Summary** — Common Anthropic-provided tool pairings. Each tool entry requires `type` (versioned string) and `name`. Snippets show only the `tools` array.

### Research agent: web_search + code_execution
Search → execute (server-side) → optionally search again.
```
"tools": [
  {"type": "web_search_20260209", "name": "web_search"},
  {"type": "code_execution_20260521", "name": "code_execution"}
]
```

### Coding agent: text_editor + bash
Canonical dev loop: inspect → edit → run tests. Both client-executed; constrain the working directory and command allowlist for untrusted code.
```
"tools": [
  {"type": "text_editor_20250728", "name": "str_replace_based_edit_tool"},
  {"type": "bash_20250124", "name": "bash"}
]
```

### Cite-then-fetch: web_search + web_fetch
Search surfaces URLs; fetch retrieves full content for relevant ones.
```
"tools": [
  {"type": "web_search_20260209", "name": "web_search"},
  {"type": "web_fetch_20260209", "name": "web_fetch"}
]
```

### Long-running agent: memory + any toolset
Memory persists state across conversations; orthogonal to other tools.
```
"tools": [{"type": "memory_20250818", "name": "memory"}]
```

### All-in-one: computer_use
Subsumes most tools by operating a full desktop; slowest (every action requires a screenshot roundtrip). Prefer narrower tools when they suffice.
```
"tools": [{
  "type": "computer_20250124", "name": "computer",
  "display_width_px": 1280, "display_height_px": 800
}]
```

---

## 22. Cross-Cutting Reference

### Content block types
- **Assistant content:** `text`, `tool_use`, `server_tool_use`
- **User content:** `text`, `image`, `tool_result`, `document`, `search_result`, `tool_reference`

### `tool_use` block fields
`type`, `id` (format `toolu_...`), `name`, `input` (object), `caller`? (programmatic tool calling).

### `tool_result` block fields
`type`, `tool_use_id`, `content` (string | array), `is_error`? (boolean).

### `server_tool_use` block fields
`type`, `id` (format `srvtoolu_...`), `name`, `input` (object). Result blocks pair by `tool_use_id`.

### `tool_choice` types
`auto` (default w/ tools), `any`, `tool` (requires `name`), `none` (default w/o tools). Plus `disable_parallel_tool_use` (boolean) nested inside.

### Stop reasons
- `"tool_use"` — run client tools.
- `"end_turn"` — final answer.
- `"max_tokens"` — output cap hit.
- `"stop_sequence"` — stop sequence hit.
- `"refusal"` — refusal.
- `"pause_turn"` — server-side loop iteration limit; resend the conversation (incl. paused response) to continue.

### Tool definition optional properties
`cache_control` (`{"type": "ephemeral"}`), `strict`, `defer_loading`, `allowed_callers`, `input_examples`, `eager_input_streaming`.

### Versioned tool type strings
- `web_search_20250305`, `web_search_20260209`, `web_search_20260318`
- `web_fetch_20250910`, `web_fetch_20260209`, `web_fetch_20260309`, `web_fetch_20260318`
- `code_execution_20250825`, `code_execution_20260120`, `code_execution_20260521` (legacy `code_execution_20250522`)
- `computer_20250124`, `computer_20251124`
- `bash_20250124` (legacy `bash_20241022`)
- `text_editor_20250728`
- `memory_20250818`
- `advisor_20260301`
- `tool_search_tool_regex_20251119`, `tool_search_tool_bm25_20251119`

### Beta headers
- `anthropic-beta: computer-use-2025-11-24` / `computer-use-2025-01-24` / `computer-use-2024-10-22`
- `anthropic-beta: advisor-tool-2026-03-01`
- `anthropic-beta: fine-grained-tool-streaming-2025-05-14` (legacy; superseded by per-tool `eager_input_streaming`)
- `anthropic-beta: code-execution-2025-05-22` (legacy)
- `anthropic-beta: files-api-2025-04-14` (Files API with code execution)

### Usage fields
- `usage.server_tool_use.web_search_requests`
- `usage.server_tool_use.web_fetch_requests`
- `usage.server_tool_use.code_execution_requests`
- `cache_creation.ephemeral_5m_input_tokens` (automatic server-tool-result cache writes, 5-min TTL)
- `usage.iterations[]` (`{"type": "message"}` executor; `{"type": "advisor_message", "model": ...}` advisor)

### Model referenced in examples
`claude-opus-4-8` (latest Claude Opus 4.8).