# xAI (Grok) API Analysis — Text Generation & Conversation Capabilities

> **Base URL:** `https://api.x.ai/v1` | **Docs:** `https://docs.x.ai/developers/model-capabilities/text/generate-text` | **Auth:** Bearer token (`XAI_API_KEY`)
> **SDKs:** Official xAI SDK (Python, gRPC) | OpenAI SDK (Python / TypeScript — via `base_url` override) | Vercel AI SDK (`@ai-sdk/xai`) | Plain HTTP/cURL
> **Description:** xAI exposes text and conversation capabilities through two API surfaces — the legacy **Chat Completions API** (`/v1/chat/completions`, deprecated) and the recommended **Responses API** (`/v1/responses`). The Responses API supports stateful conversations (server-side storage for 30 days), reasoning models with encrypted/summarized reasoning content, structured JSON output, streaming via Server-Sent Events, file attachments, and context compaction. This study excludes agents, tools, skills, and MCP, which will be analyzed separately.

---

## Table of Contents

1. [Platform Overview & Core Concepts](#1-platform-overview--core-concepts)
2. [API Surfaces — Chat Completions vs Responses](#2-api-surfaces--chat-completions-vs-responses)
3. [Text Generation](#3-text-generation)
4. [Message Roles & Instruction Hierarchy](#4-message-roles--instruction-hierarchy)
5. [Conversation State Management & Storage](#5-conversation-state-management--storage)
6. [Reasoning (Thinking) Configuration](#6-reasoning-thinking-configuration)
7. [Encrypted Reasoning Content](#7-encrypted-reasoning-content)
8. [Summarized Reasoning Content](#8-summarized-reasoning-content)
9. [Structured Outputs (JSON Schema Enforcement)](#9-structured-outputs-json-schema-enforcement)
10. [JSON Schema Support Matrix](#10-json-schema-support-matrix)
11. [Streaming](#11-streaming)
12. [Sampling Parameters](#12-sampling-parameters)
13. [Context Compaction](#13-context-compaction)
14. [File Inputs (Chat with Files)](#14-file-inputs-chat-with-files)
15. [Deferred (Asynchronous) Completions](#15-deferred-asynchronous-completions)
16. [Usage, Billing & Token Accounting](#16-usage-billing--token-accounting)
17. [Capability Summary & Cross-Reference](#17-capability-summary--cross-reference)

---

## 1. Platform Overview & Core Concepts

### Main Concepts

xAI's text/conversation platform is organized around these core abstractions:

- **Model** — The Grok large language model. Two behavioural families: standard chat models and **reasoning models** (e.g. `grok-4.5`) that generate internal reasoning tokens before producing visible output. A special **multi-agent model** (`grok-4.20-multi-agent`) routes the request across multiple cooperating agents.
- **Prompt / Input** — The text (and/or structured content) given to the model. In the Responses API this is the `input` parameter, which may be a plain string or an array of typed message objects.
- **Message** — A unit of conversation context with a `role` (`system`, `user`, `assistant`) and `content`. The Responses API also accepts typed output items (e.g. `reasoning`, `message`, `compaction`) in `input` when chaining or replaying context.
- **Role** — `system` (high-level instructions), `user` (end-user input), `assistant` (model output). The Responses API additionally offers a top-level `instructions` parameter as an alternative to a `system` message.
- **Response** — The object returned by the API. Chat Completions returns `choices[].message`; the Responses API returns a typed `output` array containing `message` items (with `output_text` content), `reasoning` items, and (for tool use) tool-call items. Each response has a unique `id`.
- **Reasoning Tokens** — Internal tokens consumed by reasoning models while "thinking." Billed as part of total consumption and reported in `usage.output_tokens_details.reasoning_tokens`. Cannot be disabled on `grok-4.5`.
- **Encrypted Reasoning Content** — An opaque, server-encrypted blob representing the model's reasoning trace. Returned via `include: ["reasoning.encrypted_content"]` and re-injected into later requests to preserve chain-of-thought context without storing it in plaintext.
- **Server-side Storage** — Responses are stored on xAI servers for 30 days by default (`store: true`), enabling `previous_response_id` chaining and later retrieval/deletion. Set `store: false` for stateless/ZDR-style usage.
- **Compaction** — A mechanism to shrink a long conversation into a single opaque `compaction` item that preserves salient state (system prompt, prior reasoning, compacted turns) while dropping verbose content, reducing cost and latency on subsequent calls.
- **Context Window** — The maximum tokens (input + output + reasoning) usable in a single request. Compaction shrinks usage but cannot rescue an already-over-limit request.

### Text & Conversation Tasks

| Task | Description | API |
|------|-------------|-----|
| **Text generation** | Generate text (prose, code, JSON) from a prompt | Responses / Chat Completions |
| **Multi-turn conversation** | Maintain context across multiple messages/turns | Responses (stateful via `previous_response_id`) / Chat Completions (manual replay) |
| **Reasoning** | Model thinks step-by-step before answering; control via `reasoning.effort` | Responses (full, encrypted content) / Chat Completions (reasoning tokens reported, no encrypted content) |
| **Structured output** | Force output to conform to a JSON Schema | Responses (`text.format`) / Chat Completions (`response_format`) |
| **Streaming** | Receive generation progress incrementally via SSE | Responses / Chat Completions |
| **File-based input** | Attach PDFs/documents by URL or uploaded file ID | Responses (agentic document search) |
| **Context compaction** | Shrink long conversations into a single opaque item | `/v1/responses/compact` |
| **Deferred completion** | Submit a request asynchronously and poll for the result | Chat Completions (`deferred: true`) |

### Platform Architecture

```
Chat Completions API (/v1/chat/completions) — DEPRECATED:
  messages[] ──▶ Model ──▶ choices[].message (content / reasoning_content / tool_calls)
  Optional: deferred=true ──▶ request_id ──▶ GET /v1/chat/deferred-completion/{request_id}

Responses API (/v1/responses) — RECOMMENDED:
  input (string | message[]) + instructions ──▶ Model ──▶ output[] (typed items)
                                                     │
                ┌────────────────────────────────────┼────────────────────────────────────┐
                ▼                                    ▼                                    ▼
          message items                    reasoning items                      (tool-call items)
          (output_text)                    (summary / encrypted_content)        (excluded here)

State management:
  Manual:           append output items to input on each turn
  previous_response_id: chain stored responses (store: true, 30-day TTL)
  Encrypted reasoning: include: ["reasoning.encrypted_content"]  (replay blobs in input)
  Compaction:       POST /v1/responses/compact ──▶ compaction item ──▶ next request input

CRUD on stored responses:
  GET    /v1/responses/{id}   — retrieve
  DELETE /v1/responses/{id}   — delete
```

---

## 2. API Surfaces — Chat Completions vs Responses

The Responses API is xAI's recommended primitive; Chat Completions is deprecated and receives only limited updates. All new capabilities land on Responses first.

### Key Differences

| Concept | Chat Completions (Deprecated) | Responses (Recommended) |
|---------|-------------------------------|-------------------------|
| **Endpoint** | `POST /v1/chat/completions` | `POST /v1/responses` |
| **Input** | `messages[]` array | `input` (string or array of message objects) |
| **System guidance** | `messages` with `role: "system"` | Top-level `instructions` parameter **or** `system`-role message |
| **Output** | `choices[].message.content` | Typed `output` array (`message` items with `output_text` content) |
| **Max tokens param** | `max_tokens` (deprecated) / `max_completion_tokens` | `max_output_tokens` (includes reasoning tokens) |
| **Stateful conversations** | Stateless — resend full history each request | `previous_response_id` chains stored responses |
| **Server-side storage** | None — manage history yourself | Stored 30 days by default (`store: true`) |
| **Reasoning content** | `message.reasoning_content` (trace, no encrypted blob) | `reasoning` output items + encrypted content via `include` |
| **Billing optimization** | Full history billed each request | Automatic caching of conversation history |
| **Agentic tools** | Function calling only | Native server-side tools (search, code exec, etc.) |
| **Multiple candidates** | `n` parameter | Not emphasized; make separate requests |
| **Structured Outputs** | `response_format` | `text.format` |
| **Compaction** | Not available | `POST /v1/responses/compact` |
| **Future features** | Legacy, limited updates | All new capabilities first |

### Parameter Mapping

| Chat Completions | Responses API | Notes |
|------------------|---------------|-------|
| `messages` | `input` | Array of message objects (or plain string) |
| `max_tokens` / `max_completion_tokens` | `max_output_tokens` | Includes output + reasoning tokens |
| — | `previous_response_id` | Continue a stored conversation |
| — | `store` | Control server-side storage (default `true`) |
| — | `include` | Request additional data (e.g. `reasoning.encrypted_content`) |
| `reasoning_effort` (string) | `reasoning.effort` (object field) | Reasoning depth control |
| `response_format` | `text.format` | Structured outputs |

### Response Structure

**Chat Completions** returns content in `choices[0].message.content`:

```json
{
  "id": "chatcmpl-123",
  "choices": [{
    "message": { "role": "assistant", "content": "Hello!" }
  }]
}
```

**Responses API** returns content in an `output` array with typed items:

```json
{
  "id": "resp_123",
  "output": [{
    "type": "message",
    "role": "assistant",
    "content": [{ "type": "output_text", "text": "Hello!" }]
  }]
}
```

### Migration Path

1. Change endpoint: `/v1/chat/completions` → `/v1/responses`.
2. Rename `messages` → `input`.
3. Rename `max_tokens`/`max_completion_tokens` → `max_output_tokens`.
4. Move `reasoning_effort` string → `reasoning: { effort: "..." }` object.
5. Move `response_format` → `text: { format: ... }`.
6. (Optional) Adopt `previous_response_id`, `store`, `include`, and compaction for stateful/cost-optimized flows.

---

## 3. Text Generation

### Core Request — Responses API

`POST /v1/responses` accepts:

- `model` *(string, required)* — e.g. `grok-4.5`.
- `input` *(string | message array, required)* — the prompt content.
- `instructions` *(string, optional)* — alternative system prompt. Cannot be combined with `previous_response_id` (the previous system prompt is reused).
- `store` *(boolean, default `true`)* — whether to persist the response server-side for 30 days.
- `previous_response_id` *(string, optional)* — ID of a prior stored response to continue.
- `include` *(array, optional)* — extra data to return, e.g. `["reasoning.encrypted_content"]`.
- `stream` *(boolean, default `false`)* — enable SSE streaming.
- `max_output_tokens`, `temperature`, `top_p`, `top_k`, `min_p` — generation controls (see §12).

### Minimal Example

```bash
curl https://api.x.ai/v1/responses \
  -H "Content-Type: application/json" \
  -H "Authorization: Bearer $XAI_API_KEY" \
  -d '{
    "model": "grok-4.5",
    "input": [
      {"role": "system", "content": "You are Grok, a helpful AI."},
      {"role": "user", "content": "How big is the universe?"}
    ]
  }'
```

### Response Shape (Responses API)

```json
{
  "id": "resp_...",
  "object": "response",
  "model": "grok-4.5",
  "status": "completed",
  "store": true,
  "output": [
    {
      "type": "message",
      "role": "assistant",
      "id": "msg_...",
      "status": "completed",
      "content": [
        { "type": "output_text", "text": "...", "logprobs": null, "annotations": [] }
      ]
    }
  ],
  "usage": { ... }
}
```

The `response.id` is the handle used to retrieve, delete, or continue the conversation.

### xAI SDK (Python, gRPC)

The official xAI SDK exposes a chat-style API:

```python
chat = client.chat.create(model="grok-4.5")
chat.append(system("..."))
chat.append(user("..."))
response = chat.sample()        # non-streaming
# or
for response, chunk in chat.stream():   # streaming
    ...
```

`chat.parse(Model)` returns `(Response, parsed_pydantic_model)` for structured output (see §9).

### Timeout Consideration

Reasoning models can take a long time to first token. xAI recommends overriding the default client timeout (e.g. `timeout=3600` seconds / `-m 3600` for cURL) for reasoning-model requests.

---

## 4. Message Roles & Instruction Hierarchy

### Roles

| Role | Source | Purpose |
|------|--------|---------|
| `system` | App developer | High-level instructions (tone, persona, rules) |
| `user` | End-user | The question/instruction |
| `assistant` | Model | Prior model outputs (when replaying history manually) |

### `instructions` Parameter (Responses API)

A top-level alternative to a `system`-role message. Cannot be used alongside `previous_response_id` (the previous response's system prompt is reused automatically). Useful for setting developer-level guidance without mixing it into the `input` array.

### Priority

System/`instructions` guidance takes precedence over user content. There is no separate `developer` role distinct from `system` (unlike OpenAI's Responses API).

---

## 5. Conversation State Management & Storage

### Server-side Storage

- **Default:** `store: true` — input messages and model responses are persisted on xAI servers for **30 days**, after which they are permanently deleted.
- **Opt-out:** Set `store: false` to handle history entirely client-side (useful for ZDR-style compliance or sensitive data).
- **Retrieval:** `GET /v1/responses/{response_id}` returns a previously stored response (including its `output` items).
- **Deletion:** `DELETE /v1/responses/{response_id}` removes a stored response early.

### State Management Strategies

| Strategy | How | When to use |
|----------|-----|-------------|
| **`previous_response_id` chaining** | Pass the prior `response.id` + only the new user message(s) | Default for multi-turn chat; xAI rehydrates prior context server-side |
| **Manual replay** | Append prior `output` items (and/or encrypted reasoning) into `input` on each request | When `store: false`, or when you need to edit/trim history |
| **Encrypted reasoning replay** | Include `reasoning.encrypted_content` items in `input` | Preserve chain-of-thought across stateless turns (see §7) |
| **Compaction** | Call `/v1/responses/compact`, then pass the `compaction` item as the new head of `input` | Long conversations approaching cost/latency or context limits (see §13) |

### Chaining Example

```python
# First turn
response = client.responses.create(model="grok-4.5", input=[...])
# Continue — no need to resend history
second = client.responses.create(
    model="grok-4.5",
    previous_response_id=response.id,
    input=[{"role": "user", "content": "How do stars form?"}],
)
```

### CRUD Endpoints (Responses API)

| Method & Path | Purpose |
|---------------|---------|
| `POST /v1/responses` | Create a response (and store it by default) |
| `GET /v1/responses/{response_id}` | Retrieve a stored response |
| `DELETE /v1/responses/{response_id}` | Delete a stored response |
| `POST /v1/responses/compact` | Compact a conversation into a single opaque item |

---

## 6. Reasoning (Thinking) Configuration

### Key Features

- **Think before responding:** Reasoning models work through problems step-by-step before delivering an answer.
- **Math & quantitative strength:** Strong on numerical challenges, logic puzzles, complex analysis.
- **Reasoning trace:** `usage.output_tokens_details.reasoning_tokens` exposes token count; encrypted blobs available via `include`.

### `reasoning.effort` Parameter

Controls how much effort the model spends thinking. On `grok-4.5` reasoning **cannot be disabled**; if unspecified it defaults to `"high"`.

| Setting | Description | Best for |
|---------|-------------|----------|
| `"low"` | Some reasoning tokens, still fast | Latency-sensitive use, simple tool calling |
| `"medium"` | More thinking, less latency-sensitive | Complex data analysis, long-context reasoning |
| `"high"` (default) | More reasoning tokens, deeper thinking | Very challenging problems, complex math, multi-step logic, competition-level tasks |

### Incompatibilities

`presence_penalty`, `frequency_penalty`, and `stop` **cannot be used with reasoning models** — requests including them return an error.

### Request Shape

```json
{
  "model": "grok-4.5",
  "reasoning": { "effort": "high" },
  "input": [...]
}
```

In the Chat Completions API the equivalent is the top-level `reasoning_effort` string. In the Vercel AI SDK: `providerOptions: { xai: { reasoningEffort: "high" } }`.

### Multi-agent Model

For `grok-4.20-multi-agent`, `reasoning.effort` instead controls **how many agents** collaborate (`"low"`/`"medium"` → 4 agents; `"high"`/`"xhigh"` → 16 agents), not reasoning depth. (Multi-agent orchestration itself is out of scope for this text/conversation study.)

### Summary Table

| Model | `reasoning` parameter | Behavior |
|-------|----------------------|----------|
| `grok-4.5` | `reasoning.effort`: `"low"` / `"medium"` / `"high"` (default) | Controls reasoning depth (cannot be disabled) |
| `grok-4.20-multi-agent` | `reasoning.effort`: `"low"` / `"medium"` / `"high"` / `"xhigh"` | Controls agent count (4 or 16) |

---

## 7. Encrypted Reasoning Content

### Concept

The model's internal reasoning trace is encrypted by xAI and can be returned as an opaque blob. Re-injecting the blob into a subsequent request preserves the chain-of-thought context without exposing or persisting plaintext reasoning.

### Requesting Encrypted Content

- **Responses API (REST / OpenAI SDK):** `include: ["reasoning.encrypted_content"]` in the request body.
- **xAI SDK (gRPC):** `use_encrypted_content=True` on `client.chat.create(...)`.
- **Vercel AI SDK:** Included automatically as long as `store: false` is not set — no extra config needed.

> Reasoning models must be used when working with encrypted thinking content.

### Re-injecting into a New Request

Two equivalent patterns:

1. **Via `previous_response_id`** (stateful): the server rehydrates the encrypted reasoning automatically.
2. **Via manual replay** (stateless / `store: false`): spread the prior `response.output` (which now contains `reasoning` items with `encrypted_content`) into the new request's `input`, then append the new user message.

### Encrypted Reasoning Item Shape (in `output` / replayed `input`)

```json
{
  "id": "rs_...",
  "type": "reasoning",
  "status": "completed",
  "summary": [],
  "encrypted_content": "<opaque base64-ish blob>"
}
```

### Use Cases

- **ZDR-style compliance:** `store: false` + encrypted reasoning lets you keep reasoning context client-side without server persistence.
- **Long-running agentic loops:** carry reasoning forward across turns without ballooning plaintext context.
- **Conversation continuation after 30-day expiry:** store blobs locally and replay them in a fresh request.

---

## 8. Summarized Reasoning Content

For `grok-4.5`, xAI exposes **summarizations** of the model's internal reasoning (a human-readable trace), streamed alongside the final response.

### Streaming Reasoning Summaries

- **xAI SDK:** `chunk.reasoning_content` in the `chat.stream()` loop.
- **OpenAI SDK (Responses, streaming):** events of type `response.reasoning_text.delta` and `response.reasoning_summary_text.delta`.
- **Vercel AI SDK:** `part.type === "reasoning-delta"` in `result.fullStream`.

### Sample Output

A reasoning summary reads like a step-by-step worked solution (e.g. setting up conservation of energy, computing `v_f = sqrt(v0² + 2gh)`, noting angle-independence), terminated by the final answer.

### `reasoning.summary` Field (Compatibility)

The REST API exposes `reasoning.summary` with values `"auto"`, `"concise"`, `"detailed"` — included for compatibility; the model always returns `"detailed"`. `reasoning.generate_summary` is likewise compatibility-only.

### Billing

Reasoning tokens are billed as part of total consumption (reported in `usage.output_tokens_details.reasoning_tokens` and counted within `max_output_tokens`).

---

## 9. Structured Outputs (JSON Schema Enforcement)

Structured Outputs forces the model to return responses conforming to a JSON Schema you define — guaranteed when using supported schema features. Useful for document parsing, entity extraction, report generation.

### Two Methods

1. **`response_format` / `text.format`** (primary, most flexible):
   - `type: "json_schema"` + schema under `json_schema` → strict schema-conformant output.
   - `type: "json_object"` → any well-formed JSON (no specific structure).
   - `type: "text"` (default) → free-form text.
2. **Tool calling** (secondary): tool-call arguments always strictly conform to the tool's input JSON Schema (`strict` is implicitly `true`). *(Tool calling is out of scope for this study but noted for completeness.)*

Schemas can be authored with [Pydantic](https://pydantic.dev/) (Python) or [Zod](https://zod.dev/) (JS).

### Responses API — `text.format`

```json
{
  "model": "grok-4.5",
  "input": [...],
  "text": {
    "format": {
      "type": "json_schema",
      "name": "invoice",
      "schema": { ... },
      "strict": true
    }
  }
}
```

### Chat Completions — `response_format`

```json
{
  "model": "grok-4.5",
  "messages": [...],
  "response_format": {
    "type": "json_schema",
    "json_schema": { "name": "invoice", "schema": { ... }, "strict": true }
  }
}
```

### xAI SDK — `chat.parse(Model)` vs `response_format`

| Approach | Method | Returns | Parsing |
|----------|--------|---------|---------|
| `parse()` | `chat.parse(PydanticModel)` | Tuple `(Response, Model)` | Automatic — SDK parses for you |
| `response_format` | `chat.sample()` / `chat.stream()` with `response_format=PydanticModel` | `Response` with JSON string in `response.content` | Manual — `Model.model_validate_json(response.content)` |

Use `parse()` for convenience; use `response_format` + `sample()`/`stream()` for more control, raw-JSON handling, or **streaming** structured output (chunks progressively build the JSON string).

### Structured Output with Tools (Grok 4 family)

Structured outputs can be combined with tool calling (agentic server-side tools or client-side functions) to return strongly typed results from tool-augmented queries. *(Tool details are excluded from this study.)*

### Type-safe Output Example

```json
{
  "vendor_name": "Acme Corp",
  "vendor_address": { "street": "123 Main St", "city": "Springfield", "postal_code": "62704", "country": "IL" },
  "invoice_number": "INV-2025-001",
  "invoice_date": "2025-02-10",
  "line_items": [
    { "description": "Widget A", "quantity": 5, "unit_price": 10.0 },
    { "description": "Widget B", "quantity": 2, "unit_price": 15.0 }
  ],
  "total_amount": 80.0,
  "currency": "USD"
}
```

---

## 10. JSON Schema Support Matrix

Schemas authored against **Draft 2020-12** work best; **Draft-07** is also accepted. `additionalProperties` defaults to `false` and must be set to `true` explicitly. Fields not in `required` are optional. Nullable fields use a type array (`{"type": ["string", "null"]}`) or an `anyOf` including `null`.

### Supported Types

`string`, `number`, `integer`, `boolean`, `null`, `enum`, `const`, `array`, `object`, `anyOf`, `oneOf` (behaves like `anyOf`), `allOf` (single subschema only — see best-effort for multiple), `$ref` / `$defs` (non-circular only).

### Enforced String Formats

`date`, `time`, `date-time`, `email`, `uuid`, `ipv4`, `ipv6`, `uri`. Other `format` values are accepted but not enforced.

### Constraint Limits (guaranteed up to)

| Keyword | Guaranteed up to |
|---------|------------------|
| `minimum` / `maximum` / `exclusiveMinimum` / `exclusiveMaximum` | No limit |
| `minLength` / `maxLength` | 2,048 |
| `minItems` / `maxItems` | 256 |
| `minProperties` / `maxProperties` | 64 |

Schemas exceeding these limits are still accepted, but conformance relies on model behavior (not structurally enforced).

### Best-effort Keywords (accepted, not enforced)

`not`; `if`/`then`/`else`; `allOf` with more than one subschema; non-listed `format` values; constraints exceeding the limits above. Validate outputs yourself if strict conformance is required.

### Rejected Schemas (return `400`)

- `enum` or `anyOf` with zero variants.
- Properties with a schema of `true` or `false`.
- `maxContains` / `minContains`.
- `items` as an array (use `prefixItems` for tuple validation).

### Regex Support (`pattern`)

Practical subset of ECMAScript (ECMA-262):

**Supported:** literals & character classes (`[abc]`, `[a-z]`, `[^abc]`); `.` (matches any Unicode codepoint incl. newlines); alternation `|`; grouping `(...)`, non-capturing `(?:...)`; quantifiers `*`, `+`, `?`, `{n}`, `{n,}`, `{n,m}`; shorthand classes `\d`, `\w`, `\s` and negations; escapes `\n`, `\t`, `\r`, `\f`, `\xHH`, `\uHHHH`, `\u{HHHHHH}`.

**Not supported:** backreferences (`\1`, `\k<name>`); Unicode property escapes (`\p{L}`, `\P{Letter}`); word boundaries (`\b`, `\B`); lookahead/lookbehind; inline modifiers (`(?i)`); conditional expressions.

**Semantic differences vs standard JS RegExp:**

- `.` matches newlines.
- `^` and `$` are **implicit** — the pattern always matches the entire string.
- Capturing groups `(...)` have no semantic effect (behave like non-capturing).
- Evaluated with Unicode support.

---

## 11. Streaming

### Concept

Streaming outputs are **supported by all models with text output capability** (chat, image understanding). **Not supported** by image-generation models. Uses **Server-Sent Events (SSE)**; the server sends content deltas as event-stream chunks, terminated by `data: [DONE]`.

### Enabling

Set `"stream": true` in the request body (both APIs).

> **Caution:** When streaming with reasoning models, manually override the request timeout to avoid prematurely closing the connection.

### Chat Completions — Delta Chunks

Each chunk carries `choices[0].delta.content` plus running `usage`:

```json
data: {
  "id": "<completion_id>", "object": "chat.completion.chunk", "created": <ts>,
  "model": "grok-4.5",
  "choices": [{ "index": 0, "delta": { "content": "Ah", "role": "assistant" } }],
  "usage": { "prompt_tokens": 41, "completion_tokens": 1, "total_tokens": 42, ... }
}
data: [DONE]
```

`stream_options.include_usage` requests an additional usage chunk before `[DONE]`.

### Responses API — Typed SSE Events

Events are typed (e.g. `response.output_text.delta`, `response.reasoning_text.delta`, `response.reasoning_summary_text.delta`). SDKs expose these as event objects with `.type` and `.delta`.

### SDK Patterns

- **xAI SDK:** `for response, chunk in chat.stream():` — `chunk.content` is the delta; `response` auto-accumulates the full text.
- **OpenAI SDK (Chat):** `for chunk in stream: chunk.choices[0].delta.content`.
- **OpenAI SDK (Responses):** `for event in stream: if event.type == "response.output_text.delta": event.delta`.
- **Vercel AI SDK:** `for await (const chunk of result.textStream)` or `result.fullStream` for typed parts (incl. `reasoning-delta`).

### Streaming + Structured Outputs

Pass a Pydantic model to `response_format` and use `chat.stream()`; chunks progressively build the JSON string in `response.content`. Parse the complete string after the stream ends (`Model.model_validate_json(response.content)`).

### Streaming + Files

Supported; stream events surface tool calls (e.g. document search) and reasoning-token progress alongside the text deltas.

---

## 12. Sampling Parameters

### Responses API — Request Body (text-relevant subset)

| Parameter | Type | Default | Notes |
|-----------|------|---------|-------|
| `model` | string | — | Required. |
| `input` | string \| message[] | — | Required. |
| `instructions` | string | — | System prompt alternative; not with `previous_response_id`. |
| `previous_response_id` | string | — | Chain a stored conversation. |
| `store` | boolean | `true` | Server-side 30-day storage. |
| `include` | array | — | e.g. `["reasoning.encrypted_content"]`. |
| `stream` | boolean | `false` | SSE streaming. |
| `max_output_tokens` | integer | — | Max tokens (output **+ reasoning**). |
| `temperature` | number (0–2) | — | Higher = more random. |
| `top_p` | number | — | Nucleus sampling; alter this or `temperature`, not both. |
| `top_k` | integer | — | Top-k sampling; disabled when unset. |
| `min_p` | number | — | Min-p sampling; tokens below `min_p` × max-prob are excluded; disabled when unset. |
| `reasoning` | object | — | `{ effort: "low"|"medium"|"high" }`. |
| `text` | object | — | `{ format: ... }` for structured output. |
| `logprobs` | boolean | — | Return logprobs. **Not supported on `grok-4.20`+** (silently ignored). |
| `top_logprobs` | integer (0–8) | — | Requires `logprobs: true`. **Not supported on `grok-4.20`+**. |
| `seed` | integer | — | Best-effort deterministic sampling; monitor via `system_fingerprint`. |
| `service_tier` | `"default"` \| `"priority"` | — | Scheduling/billing tier. |
| `prompt_cache_key` | string | — | Stable cache key for sticky routing / prompt-cache hits (plumbed to `x-grok-conv-id`). |
| `user` | string | — | End-user identifier for abuse monitoring. |

### Chat Completions API — Additional/Legacy Parameters

| Parameter | Type | Notes |
|-----------|------|-------|
| `messages` | array | Input (instead of `input`). |
| `max_tokens` / `max_completion_tokens` | integer | `max_tokens` deprecated in favor of `max_completion_tokens`. |
| `n` | integer | Number of choices to generate (billed across all). Keep `1` to minimize cost. |
| `frequency_penalty` | number (−2..2) | **Not supported by reasoning models.** |
| `presence_penalty` | number (−2..2) | **Not supported by `grok-3` and reasoning models.** |
| `stop` | array (≤4) | **Not supported by reasoning models.** |
| `logit_bias` | object | Unsupported (accepted for compatibility). |
| `deferred` | boolean | If `true`, returns a `request_id` for async polling (see §15). |
| `response_format` | object | Structured output (instead of `text.format`). |
| `reasoning_effort` | string | `none`/`low`/`medium`/`high` (legacy; only `grok-4.3` supports `none`). |
| `search_parameters` / `web_search_options` | object | Live-search controls (search itself is out of scope, but the params exist on the text endpoints). |

### Notes on Reasoning-model Restrictions

- `presence_penalty`, `frequency_penalty`, `stop` → error if used with reasoning models.
- `logprobs` / `top_logprobs` silently ignored on `grok-4.20` and newer.

---

## 13. Context Compaction

### Concept

When a conversation grows past a few thousand tokens, every follow-up call resends all prior messages and pays input tokens for all of them. **Context compaction** shrinks those messages into a single opaque `compaction` item that preserves salient state — system prompts, attached files, prior reasoning, compacted turns — while dropping verbose tool output and back-and-forth. The compaction item is passed back verbatim into the next request.

### Benefits

- **Lower input cost** — next call only pays for the compacted context.
- **Lower latency** — smaller payloads, faster time-to-first-token.
- **Sharper responses** — tighter context keeps the model focused.
- **Longer conversations** — keep multi-hour loops well under the context window.

### When to Compact

Compact when **all** are true:

- The conversation is large enough that `input_tokens` per call is hurting cost/latency.
- You still want the model to remember prior turns (otherwise start a new conversation).
- The current window still fits within the model's context limit (compaction shrinks; it cannot rescue an over-limit request).

A typical pattern: compact every N turns inside an agent loop, or whenever rendered context crosses a chosen threshold.

### Compaction API — `POST /v1/responses/compact`

**Request:**

- `model` *(string, required)* — model to use for compaction summarization (a smaller/faster model is fine for frequent compaction).
- `input` *(string | message array, required)* — the conversation to compact.

**Response:**

```json
{
  "id": "cmp_01HZ9P0V8M2YQK3F7C4G6N5R2A",
  "object": "response.compaction",
  "created_at": 1748895600,
  "model": "grok-4.5",
  "output": [
    {
      "type": "compaction",
      "id": "cmp_01HZ9P0V8M2YQK3F7C4G6N5R2A",
      "encrypted_content": "<opaque blob>"
    }
  ],
  "usage": {
    "input_tokens": 12000,
    "input_tokens_details": { "cached_tokens": 0 },
    "output_tokens": 800,
    "output_tokens_details": { "reasoning_tokens": 240 },
    "total_tokens": 12800,
    "dropped_message_count": 45
  }
}
```

| Field | Description |
|-------|-------------|
| `id` | Stable compaction ID (`cmp_<uuid>`); echoed on the inner item. |
| `object` | Always `"response.compaction"`. |
| `output` | Array with a **single** compaction item; pass it verbatim into the next request. |
| `output[].type` | Always `"compaction"`. |
| `output[].encrypted_content` | Opaque blob containing the compacted conversation. |
| `usage.input_tokens` | Tokens in the pre-compaction conversation. |
| `usage.output_tokens` | Tokens generated for the compacted record. |
| `usage.dropped_message_count` | Number of input messages folded into the compaction. |

### Continuing After Compaction

Spread `compacted.output` into the next `/v1/responses` call's `input`, then append the new user turn **after** it:

```python
followup = client.responses.create(
    model="grok-4.5",
    input=[
        *compacted.output,   # use verbatim — do not modify
        {"role": "user", "content": "Based on our earlier conversation, what gives particles their mass?"},
    ],
)
```

### xAI SDK Convenience

- `client.chat.compact_context(model=..., messages=chat.messages)` → returns a `CompactContextResponse`.
- `chat.append(compact)` clears the in-memory message list and seeds it with just the compaction blob; subsequent `chat.sample()` calls run on top of the compacted context.
- `chat.compact()` (in-place) runs compaction against the chat's current messages and replaces them in-place with the compaction item — ideal for long-running loops.
- `AsyncClient` exposes `await client.chat.compact_context(...)` and `await chat.compact()`.

### Limits & Gotchas

- The conversation to compact **must already fit in context** — compaction shrinks, it does not rescue over-limit requests. If already past `context_length_exceeded`, prune or split first.
- **At most one compaction per call.**
- `encrypted_content` is **opaque** — do not parse, edit, or hand-merge multiple blobs. Always pass the full `output` array back verbatim.
- **Do not prune the compaction output.** Treat it as the new "start" of the conversation; append new user turns **after** it, never before. Reordering items inside the compacted output breaks the chain.
- **Re-compacting is fine** — compact an already-compacted conversation again later as it grows.
- The compaction call itself uses tokens (visible in `usage`). Pick a smaller/faster model for frequent compaction.

---

## 14. File Inputs (Chat with Files)

Files can be attached to chat conversations; the system automatically enables **document search**, turning the request into an agentic workflow. (The agentic/tooling internals are out of scope; this section covers the text-side attachment interface.)

### Attachment Methods

| Method | Content part type | Use case |
|--------|-------------------|----------|
| **Public URL** | `{"type": "input_file", "file_url": "https://..."}` | Any publicly accessible file; no upload step. |
| **Uploaded file ID** | `{"type": "input_file", "file_id": "file-abc123"}` | Private/sensitive documents uploaded via the Files API first. |

### Message Shape

A user message `content` is an array mixing `input_text` with one or more `input_file` parts:

```json
{
  "role": "user",
  "content": [
    { "type": "input_text", "text": "What was the total revenue in this report?" },
    { "type": "input_file", "file_url": "https://docs.x.ai/assets/api-examples/documents/sales-report.txt" }
  ]
}
```

### Capabilities

- **Single file** — ask questions about one document.
- **Multiple files** — query across several documents simultaneously.
- **Multi-turn** — maintain context across multiple questions about the same documents. Use `previous_response_id` (stateful) or `use_encrypted_content=True` (xAI SDK) to preserve file context efficiently across turns.
- **Combine with other modalities** — mix `input_file` with `input_image` parts in one message.
- **Combine with code execution** — attach data files (e.g. CSV) and enable the code-execution tool for analysis. *(Tool enabling excluded from this study.)*
- **Streaming** — supported; stream events surface document-search tool calls and reasoning-token progress alongside text deltas.

### xAI SDK Helpers

```python
from xai_sdk.chat import user, file
chat.append(user("Question?", file(url="https://...")))      # or file(file_id="...")
```

### Limitations & Considerations

- **No batch requests** — file attachments with document search are agentic; `n > 1` not supported.
- **Streaming recommended** for better observability of the document-search process.
- Highly unstructured or very long documents may require more processing; well-organized documents are easier to search.
- Large documents with many searches can result in higher token usage.
- **Recommended model:** `grok-4.5` for best document understanding.
- **Agentic requirement:** file attachments require agentic-capable models that support server-side tools.

---

## 15. Deferred (Asynchronous) Completions

### Concept (Chat Completions API)

For long-running requests, set `deferred: true` on `POST /v1/chat/completions`. The API immediately returns a `request_id` instead of blocking. Poll `GET /v1/chat/deferred-completion/{request_id}`:

- **`200 Success`** — request completed; body is the standard chat completion response.
- **`202 Accepted`** — request still pending.

This is useful for workloads where you don't want to hold a long-lived HTTP connection (e.g. reasoning-model requests that may take many minutes). The Responses API exposes a `background` field for OpenResponses compatibility but it is currently unsupported.

---

## 16. Usage, Billing & Token Accounting

### Responses API — `usage`

| Field | Description |
|-------|-------------|
| `input_tokens` | Number of input tokens used. |
| `input_tokens_details.cached_tokens` | Tokens served from xAI's prompt cache (reused from previous requests). |
| `output_tokens` | Number of output tokens used (visible + reasoning). |
| `output_tokens_details.reasoning_tokens` | Tokens generated by the model for reasoning. |
| `total_tokens` | `input_tokens + output_tokens`. |
| `num_sources_used` | Number of sources used (live search). |
| `num_server_side_tools_used` | Number of server-side tools used. |
| `context_details.input_tokens` / `context_details.output_tokens` | Prompt / (completion + reasoning) tokens in the latest context. |
| `cost_in_usd_ticks` | Accurate cost in USD ticks (10,000,000,000 ticks = 1 USD). |
| `cost_in_nano_usd` | Cost in nano-USD. |
| `server_side_tool_usage_details` | Per-tool call counts (code interpreter, document search, file search, image generation, MCP, web search, X search). |

### Chat Completions — `usage` (legacy shape)

Uses `prompt_tokens` / `completion_tokens` / `total_tokens` with `prompt_tokens_details` (text/audio/image/cached) and `completion_tokens_details` (reasoning/audio/accepted-prediction/rejected-prediction). Also reports `num_sources_used` and `cost_in_usd_ticks`.

### Billing Notes

- **Reasoning tokens are billed** as part of total consumption and count against `max_output_tokens`.
- **Prompt caching** is automatic on the Responses API; `cached_tokens` reflects reuse, and `prompt_cache_key` enables sticky routing across requests sharing a prefix.
- **Service tier** (`default` / `priority`) affects scheduling priority and billing.

---

## 17. Capability Summary & Cross-Reference

### Feature Matrix — Chat Completions vs Responses

| Feature | Chat Completions (Deprecated) | Responses (Recommended) | Section |
|---------|-------------------------------|-------------------------|---------|
| Endpoint | `POST /v1/chat/completions` | `POST /v1/responses` | §2 |
| Input | `messages[]` | `input` (string \| message[]) | §2, §3 |
| System guidance | `system`-role message | `instructions` parameter **or** `system` message | §4 |
| Output | `choices[].message.content` | typed `output` array (`output_text`) | §2 |
| Max tokens | `max_tokens` / `max_completion_tokens` | `max_output_tokens` (incl. reasoning) | §12 |
| Multiple candidates (`n`) | Yes | Not emphasized | §12 |
| Server-side storage | No | Yes (30-day TTL, `store: true` default) | §5 |
| Stateful chaining | No (manual replay) | `previous_response_id` | §5 |
| Retrieve stored response | No | `GET /v1/responses/{id}` | §5 |
| Delete stored response | No | `DELETE /v1/responses/{id}` | §5 |
| Reasoning models | Yes (tokens reported) | Yes (full, recommended) | §6 |
| `reasoning.effort` | `reasoning_effort` string | `reasoning: { effort }` object | §6 |
| Encrypted reasoning content | No | Yes (`include: ["reasoning.encrypted_content"]`) | §7 |
| Summarized reasoning content | `message.reasoning_content` | `reasoning` items + summary deltas | §8 |
| Structured Outputs | `response_format` | `text.format` | §9 |
| JSON mode (`json_object`) | Yes | Yes | §9 |
| Streaming | `delta` chunks | typed SSE events | §11 |
| Sampling: `temperature`/`top_p` | Yes | Yes | §12 |
| Sampling: `top_k` / `min_p` | — | Yes | §12 |
| `seed` (deterministic) | Yes | Yes | §12 |
| `logprobs` / `top_logprobs` | Yes (not on `grok-4.20`+) | Yes (not on `grok-4.20`+) | §12 |
| `presence_penalty` / `frequency_penalty` / `stop` | Yes (not on reasoning models) | Not supported | §12 |
| Context compaction | No | `POST /v1/responses/compact` | §13 |
| Prompt caching | Manual | Automatic (`cached_tokens`, `prompt_cache_key`) | §16 |
| File inputs (URL / file_id) | Limited | Yes (agentic document search) | §14 |
| Deferred (async) completion | Yes (`deferred: true`) | `background` field (unsupported) | §15 |
| `service_tier` | Yes | Yes | §12 |

### State Management Decision Matrix

| Scenario | Recommended Approach |
|----------|----------------------|
| Simple multi-turn chat | `previous_response_id` with `store: true` |
| Full control over context (trim/edit) | Manual replay of `output` items in `input` |
| Preserve reasoning across stateless turns | `store: false` + `include: ["reasoning.encrypted_content"]`, replay blobs |
| Conversation after 30-day storage expiry | Store encrypted blobs locally; replay in a fresh request |
| Long conversation approaching cost/latency or context limit | Compaction (`/v1/responses/compact`) |
| Multi-turn file chat | `previous_response_id` or `use_encrypted_content=True` (xAI SDK) |

### Structured Output Decision Matrix

| Need | Recommended Approach |
|------|----------------------|
| Schema-conformant response to user | `text.format` (Responses) / `response_format` (Chat) with `json_schema` + `strict: true` |
| Just valid JSON, no schema | `json_object` mode |
| Schema with optional fields | Omit from `required` (optional); nullable via `["string", "null"]` |
| Recursive schemas | `$defs` + `$ref` (non-circular only) |
| Streaming structured output | `response_format` + `stream()`; parse `response.content` after stream ends |
| Convenient typed parsing (xAI SDK) | `chat.parse(PydanticModel)` → `(Response, Model)` |
| Validate best-effort keywords | Validate outputs in code (not structurally enforced) |

### Sampling Decision Matrix

| Need | Recommended Parameter |
|------|----------------------|
| Balance randomness/determinism | `temperature` (0–2) |
| Nucleus sampling | `top_p` (not with `temperature`) |
| Top-k restriction | `top_k` (Responses only) |
| Min-prob cutoff | `min_p` (Responses only) |
| Reproducibility | `seed` + monitor `system_fingerprint` |
| Reasoning depth | `reasoning.effort` (`low`/`medium`/`high`) |
| Cost control on long contexts | Compaction + prompt caching (`prompt_cache_key`) |

---

### Sources

- Generate Text — `https://docs.x.ai/developers/model-capabilities/text/generate-text`
- Streaming — `https://docs.x.ai/developers/model-capabilities/text/streaming`
- Reasoning — `https://docs.x.ai/developers/model-capabilities/text/reasoning`
- Structured Outputs — `https://docs.x.ai/developers/model-capabilities/text/structured-outputs`
- Comparison (Chat Completions vs Responses) — `https://docs.x.ai/developers/model-capabilities/text/comparison`
- Chat with Files — `https://docs.x.ai/developers/model-capabilities/files/chat-with-files`
- Context Compaction — `https://docs.x.ai/developers/advanced-api-usage/context-compaction`
- REST API Reference (Chat / Responses / Compact) — `https://docs.x.ai/developers/rest-api-reference/inference/chat`