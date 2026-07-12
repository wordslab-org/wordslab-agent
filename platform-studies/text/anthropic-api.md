# Anthropic API Analysis — Text Generation & Conversation Capabilities

> **Base URL:** `https://api.anthropic.com/v1` | **Docs:** `https://platform.claude.com/docs/en/build-with-claude` | **Auth:** `x-api-key: $ANTHROPIC_API_KEY` + `anthropic-version: 2023-06-01`
> **SDKs:** `anthropic` (Python / TypeScript / Java / Go / C# / Ruby / PHP) | **CLI:** `ant` (Anthropic CLI)
> **Description:** Anthropic exposes text and conversation capabilities through a single unified **Messages API** (`POST /v1/messages`). The platform covers text generation from prompts, multimodal vision and PDF inputs, extended thinking (reasoning), structured JSON outputs, citations grounded in provided documents, streaming, prompt caching, server-side context compaction/editing, batch processing, token counting, and fast-mode inference. Claude models are stateless — the full conversation history is resent each request — but prompt caching, context management, and mid-conversation system messages provide efficient state handling for long conversations.

---

## Table of Contents

1. [Platform Overview & Core Concepts](#1-platform-overview--core-concepts)
2. [The Messages API — Core Request/Response Schema](#2-the-messages-api--core-requestresponse-schema)
3. [Text Generation & Prompting](#3-text-generation--prompting)
4. [Message Roles & Instruction Hierarchy](#4-message-roles--instruction-hierarchy)
5. [Multi-turn Conversations](#5-multi-turn-conversations)
6. [Extended Thinking & Effort Control](#6-extended-thinking--effort-control)
7. [Streaming](#7-streaming)
8. [Vision (Image Inputs)](#8-vision-image-inputs)
9. [PDF & Document Inputs](#9-pdf--document-inputs)
10. [Files API](#10-files-api)
11. [Structured Outputs (JSON Schema Enforcement)](#11-structured-outputs-json-schema-enforcement)
12. [Citations & Search Results (Document Grounding)](#12-citations--search-results-document-grounding)
13. [Prompt Caching](#13-prompt-caching)
14. [Context Windows & Context Management](#14-context-windows--context-management)
15. [Task Budgets](#15-task-budgets)
16. [Batch Processing (Message Batches)](#16-batch-processing-message-batches)
17. [Token Counting](#17-token-counting)
18. [Stop Reasons, Refusals & Fallback](#18-stop-reasons-refusals--fallback)
19. [Fast Mode](#19-fast-mode)
20. [Multilingual Support](#20-multilingual-support)
21. [Workspaces (Organization & Admin)](#21-workspaces-organization--admin)
22. [Embeddings (Voyage AI)](#22-embeddings-voyage-ai)
23. [Capability Summary & Cross-Reference](#23-capability-summary--cross-reference)

---

## 1. Platform Overview & Core Concepts

### Main Concepts

Anthropic's text/conversation platform is organized around these core abstractions:

- **Model** — The underlying Claude large language model. Current families: **Claude Opus** (4.8, 4.7, 4.6, 4.5, 4.1), **Claude Sonnet** (5, 4.6, 4.5), **Claude Haiku** (4.5), **Claude Fable 5**, **Claude Mythos 5**, and **Claude Mythos Preview**. Models differ in context window size (200k or 1M tokens), output limits (64k–128k), thinking behavior, and feature support.
- **Message** — A unit of conversation context with a `role` (`user`, `assistant`, or `system`) and `content` (a plain string or an array of typed content blocks). The Messages API is **stateless**: you resend the full conversation history each request.
- **Content Block** — A typed element within a message's `content` array: `text`, `image`, `document`, `search_result`, `thinking`, `tool_use`, `tool_result`, `redacted_thinking`, `fallback`, `compaction`. Blocks enable multimodal, structured, and agentic interactions within a single message.
- **System Prompt** — Top-level `system` parameter (string or array of text blocks) for context, instructions, and role definition. Separate from `messages`; not a `system` *role* in the `messages` array (except for mid-conversation system messages on Opus 4.8).
- **Response** — The object returned by the API (`type: "message"`, `role: "assistant"`). Contains a `content` array of blocks, `stop_reason`, `stop_sequence`, `stop_details`, `model`, and `usage`.
- **Stop Reason** — Enum on every response indicating why generation stopped: `end_turn`, `max_tokens`, `stop_sequence`, `tool_use`, `pause_turn`, `refusal`, `model_context_window_exceeded`.
- **Usage** — Token accounting object: `input_tokens`, `output_tokens`, `cache_creation_input_tokens`, `cache_read_input_tokens`, `output_tokens_details.thinking_tokens`, `server_tool_use`, `service_tier`, and `iterations` (for compaction/fallback).
- **Context Window** — The maximum tokens (input + output + thinking) usable in a single request. 200k for most models; 1M for Opus 4.6+/Sonnet 5/Fable 5/Mythos 5/Mythos Preview.
- **Extended Thinking** — Internal reasoning via `thinking` content blocks, generated before `text` blocks. Controlled by `thinking` and `effort` parameters. Thinking tokens count toward `max_tokens` and the context window.
- **Prompt Caching** — Cache stable prompt prefixes to reduce latency and cost. Up to 4 cache breakpoints; 5-minute or 1-hour TTLs.
- **Context Management** — Server-side strategies for long conversations: compaction (summarization) and context editing (clearing tool results / thinking blocks), configured via `context_management.edits`.
- **Beta Headers** — Many features require `anthropic-beta` headers (e.g. `files-api-2025-04-14`, `compact-2026-01-12`, `fast-mode-2026-02-01`, `task-budgets-2026-03-13`, `context-management-2025-06-27`).

### Text & Conversation Tasks

| Task | Description | Mechanism |
|------|-------------|-----------|
| **Text generation** | Generate text from a prompt | `POST /v1/messages` |
| **Multi-turn conversation** | Maintain context across turns (stateless — resend history) | `messages[]` array |
| **Vision** | Analyze images (base64 / URL / file_id) | `image` content blocks |
| **PDF / document input** | Analyze PDFs and text documents | `document` content blocks |
| **Extended thinking** | Internal reasoning before output | `thinking` parameter |
| **Effort control** | Control reasoning thoroughness vs. token spend | `output_config.effort` |
| **Structured output** | Force JSON output conforming to a schema | `output_config.format` |
| **Citations** | Ground responses in provided documents with cited passages | `citations: {enabled: true}` on document blocks |
| **Search result citations** | RAG-style citations for search result blocks | `search_result` content blocks |
| **Streaming** | Incremental response via SSE | `stream: true` |
| **Prompt caching** | Cache stable prefixes for cost/latency reduction | `cache_control` field |
| **Context compaction** | Auto-summarize old context near window limit | `context_management.edits` (compact) |
| **Context editing** | Clear tool results / thinking blocks server-side | `context_management.edits` (clear) |
| **Batch processing** | Async bulk processing at 50% cost | `POST /v1/messages/batches` |
| **Token counting** | Estimate input tokens before sending | `POST /v1/messages/count_tokens` |
| **Fast mode** | Higher output tokens/sec on Opus models | `speed: "fast"` + beta header |
| **Refusal fallback** | Auto-fallback to another model on safety refusal | `fallbacks` parameter |

### Platform Architecture

```
Messages API (POST /v1/messages):
  system (string | TextBlock[]) + messages[] (role + content blocks)
    ──▶ Model ──▶ content[] (typed blocks) + stop_reason + usage

Content block types (input):                    Content block types (output):
  text, image, document, search_result,           text, thinking, redacted_thinking,
  tool_use, tool_result                           tool_use, fallback, compaction

State management (stateless API):
  Manual:           resend full messages[] each turn
  Prompt caching:   cache_control breakpoints (tools → system → messages hierarchy)
  Context mgmt:     server-side compaction + context editing (context_management.edits)
  Mid-conversation: system role messages (Opus 4.8 only)

Companion APIs:
  POST /v1/messages/batches       — async batch processing (50% discount)
  POST /v1/messages/count_tokens  — token estimation
  POST /v1/files ...              — file upload/storage (beta)
```

---

## 2. The Messages API — Core Request/Response Schema

### Main Concepts

`POST /v1/messages` is the single unified endpoint for all text and conversation capabilities. It accepts structured input messages (text and/or image/document content) and generates the next assistant message. The API is stateless — there is no server-side conversation state (unlike OpenAI's Responses API `previous_response_id` or Conversations API); the full history is resent each request.

### Standard Request Headers

| Header | Required | Description |
|--------|----------|-------------|
| `Content-Type` | Yes | `application/json` |
| `anthropic-version` | Yes | `2023-06-01` |
| `x-api-key` | Yes | `$ANTHROPIC_API_KEY` (or `Authorization: Bearer` on some platforms) |
| `anthropic-beta` | No | Beta feature flags (comma-separated), e.g. `files-api-2025-04-14`, `compact-2026-01-12`, `fast-mode-2026-02-01`, `task-budgets-2026-03-13`, `context-management-2025-06-27`, `server-side-fallback-2026-06-01` |
| `anthropic-user-profile-id` | No | User profile ID for attribution (requires `user-profiles` beta) |

### Request Body Parameters (Full Schema)

| Parameter | Type | Req/Opt | Description |
|-----------|------|---------|-------------|
| `model` | string (enum) | **Required** | Model ID. Enum: `claude-sonnet-5`, `claude-fable-5`, `claude-mythos-5`, `claude-opus-4-8`, `claude-opus-4-7`, `claude-mythos-preview`, `claude-opus-4-6`, `claude-sonnet-4-6`, `claude-haiku-4-5`, `claude-haiku-4-5-20251001`, `claude-opus-4-5`, `claude-opus-4-5-20251101`, `claude-sonnet-4-5`, `claude-sonnet-4-5-20250929`, `claude-opus-4-1`, `claude-opus-4-1-20250805` |
| `messages` | array of `{content, role}` | **Required** | Input messages. Each has `role` (`user`/`assistant`/`system` on Opus 4.8) + `content` (string or array of content blocks). Limit: 100,000 messages per request. |
| `max_tokens` | number | **Required** | Maximum tokens to generate. Models may stop earlier. Set `0` to populate prompt cache without generating. Minimum 0. |
| `system` | string \| array of `TextBlockParam` | Optional | System prompt (context/instructions/role). String or array of `{text, type: "text", cache_control?, citations?}`. |
| `stream` | boolean | Optional | Incrementally stream response via SSE. Default `false`. |
| `temperature` | number | Optional, **Deprecated** | Randomness. Models after Opus 4.6 don't support it; only `1.0` accepted (others → 400). Default 1.0, range 0.0–1.0. |
| `top_p` | number | Optional, **Deprecated** | Nucleus sampling. Models after Opus 4.6 only accept ≥0.99 (others → 400). Max 1, min 0. |
| `top_k` | number | Optional, **Deprecated** | Sample from top K options. Models after Opus 4.6 reject any value with 400. Minimum 0. |
| `stop_sequences` | array of string | Optional | Custom text sequences that cause stop → `stop_reason: "stop_sequence"`. |
| `metadata` | `{user_id}` | Optional | Request metadata. `user_id`: external opaque identifier (uuid/hash), maxLength 512, no PII. |
| `thinking` | `ThinkingConfigParam` | Optional | Extended thinking config. See §6. |
| `output_config` | `{effort, format, task_budget}` | Optional | Output configuration. See §6 (effort), §11 (format), §15 (task_budget). |
| `tools` | array of `ToolUnion` | Optional | Tool definitions (client & server tools). *(Tool-use mechanics excluded from this analysis.)* |
| `tool_choice` | `ToolChoice` | Optional | How model uses tools: `{type: "auto"}`, `{type: "any"}`, `{type: "tool", name}`, `{type: "none"}`, each with optional `disable_parallel_tool_use`. *(Tool-use mechanics excluded.)* |
| `cache_control` | `{type, ttl}` | Optional | Top-level cache control; auto-applies a cache marker to the last cacheable block. `type: "ephemeral"`, `ttl: "5m"` (default) or `"1h"`. See §13. |
| `context_management` | `{edits}` | Optional | Server-side context management strategies. See §14. |
| `speed` | `"fast"` \| `"standard"` | Optional | Fast mode. See §19. |
| `fallbacks` | array of `{model, max_tokens?, thinking?}` | Optional | Server-side refusal fallback. See §18. |
| `service_tier` | `"auto"` \| `"standard_only"` | Optional | Use priority capacity (if available) or standard. |
| `inference_geo` | string | Optional | Geographic region for inference. Defaults to workspace's `default_inference_geo`. |
| `container` | string | Optional | Container identifier for reuse across requests (code execution tool). |

### Input Content Block Types

| Block Type | Key Fields | Description |
|------------|------------|-------------|
| `text` | `text`, `cache_control?`, `citations?` | Text content |
| `image` | `source`, `cache_control?` | Image: `source` is `base64` `{data, media_type, type}` (jpeg/png/gif/webp) or `url` `{type, url}` or `file` `{type, file_id}` |
| `document` | `source`, `cache_control?`, `citations?`, `context?`, `title?` | PDF/text document: `source` is `url`, `base64`, or `file` |
| `search_result` | `source`, `title`, `content`, `citations?`, `cache_control?` | RAG search result block for citations |
| `tool_use` | `id`, `name`, `input` | Tool call from assistant *(excluded from analysis)* |
| `tool_result` | `tool_use_id`, `content` | Tool result from user *(excluded from analysis)* |

### Response Object (Full Schema)

| Field | Type | Description |
|-------|------|-------------|
| `id` | string | Unique object identifier (e.g. `msg_01XFDUDYJgACzvnptvVoYEL`) |
| `type` | `"message"` | Object type, always `"message"` |
| `role` | `"assistant"` | Always `"assistant"` |
| `content` | array of `ContentBlock` | Content generated by model. Blocks: `TextBlock {citations, text, type: "text"}`, `ThinkingBlock {thinking, signature, type}`, `ToolUseBlock`, `RedactedThinkingBlock {data, type}`, `FallbackBlock {from, to, type}`, `CompactionBlock {content, type}`. |
| `model` | string | Model that completed the prompt |
| `stop_reason` | `StopReason` enum | `"end_turn"` \| `"max_tokens"` \| `"stop_sequence"` \| `"tool_use"` \| `"pause_turn"` \| `"refusal"` \| `"model_context_window_exceeded"`. See §18. |
| `stop_sequence` | string \| null | Matched custom stop sequence (non-null if `stop_reason: "stop_sequence"`) |
| `stop_details` | `RefusalStopDetails` \| null | `{category, explanation, type: "refusal"}`. Non-null only for `refusal`. See §18. |
| `usage` | `Usage` | Token accounting (see below) |

### `usage` Object Fields

| Field | Type | Description |
|-------|------|-------------|
| `input_tokens` | number | Input tokens not from cache |
| `output_tokens` | number | Output tokens generated (inclusive authoritative billing total) |
| `cache_creation_input_tokens` | number | Tokens written to cache |
| `cache_read_input_tokens` | number | Tokens read from cache |
| `cache_creation` | `{ephemeral_5m_input_tokens, ephemeral_1h_input_tokens}` | Breakdown by TTL |
| `output_tokens_details` | `{thinking_tokens}` | Raw reasoning token count (≤ `output_tokens`) |
| `server_tool_use` | `{web_fetch_requests, web_search_requests}` | Server tool usage counts |
| `service_tier` | `"standard"` \| `"priority"` \| `"batch"` | Service tier used |
| `iterations` | array | Per-iteration breakdown for compaction/fallback (see §14, §18) |

**Total input tokens** = `input_tokens` + `cache_creation_input_tokens` + `cache_read_input_tokens`.

### Example

```bash
curl https://api.anthropic.com/v1/messages \
     --header "x-api-key: $ANTHROPIC_API_KEY" \
     --header "anthropic-version: 2023-06-01" \
     --header "content-type: application/json" \
     --data '{
         "model": "claude-opus-4-8",
         "max_tokens": 1024,
         "messages": [{"role": "user", "content": "Hello, Claude"}]
     }'
```

```json
{
  "id": "msg_01XFDUDYJgAACzvnptvVoYEL",
  "type": "message",
  "role": "assistant",
  "content": [{"type": "text", "text": "Hello!"}],
  "model": "claude-opus-4-8",
  "stop_reason": "end_turn",
  "stop_sequence": null,
  "usage": {"input_tokens": 12, "output_tokens": 6}
}
```

### Key Behaviors & Notes

- `max_tokens: 0` pre-warms the prompt cache with no response generation (rejected if combined with `stream: true`, `thinking.type: "enabled"`, `output_config.format`, or forced `tool_choice`).
- `temperature`, `top_p`, `top_k` are **deprecated** for models released after Opus 4.6 (only constrained values accepted; others → 400). Use prompting or `effort` to guide behavior instead.
- Content can be a plain string (shorthand for one `text` block) or an array of typed content blocks.
- An input ending with an `assistant` message causes the response to continue directly from that content (output constraining / prefill — see §3).
- No `"system"` role in `messages` (except mid-conversation system messages on Opus 4.8 — see §4); system prompt goes in the top-level `system` parameter.
- `output_tokens` is non-zero even for empty-string responses (token counts don't 1:1 match visible content).
- Limit: 100,000 messages per request.

---

## 3. Text Generation & Prompting

### Main Concepts

Text generation is the foundational capability — prompting a model to produce text. The model generates an array of content blocks in the `content` property. For simple text responses, `content[0]` is a `text` block, but the array can contain multiple blocks (thinking, text, tool_use, etc.).

### Output Constraining (Prefill)

You can pre-fill part of Claude's response by placing an `assistant` message as the last entry in `messages`. Claude continues from that content. This can shape the response format.

**Warning:** Prefilling is **not supported** on Claude Fable 5, Mythos 5, Mythos Preview, Opus 4.8, Opus 4.7, Opus 4.6, Sonnet 5, and Sonnet 4.6. Requests using prefill with these models return a 400 error. Use structured outputs (§11) or system prompt instructions instead.

### Example (Prefill — supported models only)

```json
{
    "model": "claude-sonnet-4-5",
    "max_tokens": 1,
    "messages": [
        {"role": "user", "content": "What is latin for Ant? (A) Apoidea, (B) Rhopalocera, (C) Formicidae"},
        {"role": "assistant", "content": "The answer is ("}
    ]
}
```

Response: `{"content": [{"type": "text", "text": "C"}], "stop_reason": "max_tokens"}`

### Key Behaviors & Notes

- Prefill is deprecated on newer models — use `output_config.format` (structured outputs) instead.
- Extended thinking is **not compatible** with prefill.
- `temperature`/`top_k` modifications are not compatible with extended thinking.

---

## 4. Message Roles & Instruction Hierarchy

### Main Concepts

Messages have differing levels of authority. Claude is trained on alternating `user`/`assistant` turns; consecutive same-role turns are combined.

| Role | Priority | Description |
|------|----------|-------------|
| `system` (top-level `system` param) | Highest | Operator-level instructions: context, role, behavior rules. Applies from the first turn. |
| `system` (mid-conversation, Opus 4.8) | Highest | Operator-level instruction appended mid-conversation. Same authority as top-level `system` but doesn't invalidate cached prefix. |
| `user` | Medium | End-user input, tool results. |
| `assistant` | — | Model-generated output, prefill. |

### Top-level `system` Parameter

String or array of text blocks. Used for whole-conversation instructions (persona, rules, context). Can include `cache_control` for prompt caching.

### Mid-conversation System Messages (Opus 4.8 only)

Append a `{"role": "system"}` message inside the `messages` array at the point a new instruction becomes relevant, instead of editing the top-level `system` field. This preserves prompt caching: the cached prefix stays byte-identical.

**Placement rules (violations return 400):**
- Cannot be the first entry in `messages` (use top-level `system` instead).
- Must immediately follow a `user` turn (including one carrying `tool_result` blocks) or an `assistant` turn ending in a server tool use.
- Must either be the last entry or be immediately followed by an `assistant` turn.
- **Cannot** sit between an `assistant` `tool_use` block and the `tool_result` that answers it.
- Consecutive system messages are not allowed.

**Availability:** Claude API, Claude Platform on AWS, Microsoft Foundry. **Not** Amazon Bedrock or Google Cloud. **Claude Opus 4.8 only.** ZDR-eligible. No beta header required.

### Example

```python
response = client.messages.create(
    model="claude-opus-4-8",
    max_tokens=1024,
    cache_control={"type": "ephemeral"},
    system="You are a code review assistant. Be concise.",
    messages=[
        {"role": "user", "content": "Review process() in utils.py..."},
        {"role": "assistant", "content": "The list comprehension is fine..."},
        {"role": "user", "content": "Now review the calling code..."},
        {"role": "system", "content": "From now on, every suggestion must include explicit type annotations."},
    ],
)
```

### Key Behaviors & Notes

- Later system messages take precedence over earlier ones; mid-conversation system messages take precedence over the top-level `system` field for subsequent turns.
- Avoid editing/removing a sent mid-conversation system message (invalidates cache); append new ones instead.
- Do **not** put untrusted content (tool output, retrieved documents, web content) in a system message — it grants operator-level authority. Keep that data in `tool_result` blocks.
- Phrase system content as context/facts, not commands overriding the user.

---

## 5. Multi-turn Conversations

### Main Concepts

The Messages API is **stateless** — you always send the full conversational history. Earlier turns don't need to originate from Claude; you can use synthetic `assistant` messages. There is no server-side conversation state, no `previous_response_id` chaining, and no persistent conversation object (unlike OpenAI's Responses API). State management relies on prompt caching (§13) and context management (§14) for efficiency.

### API Parameters

No special parameters — standard `messages` array with alternating `user`/`assistant` turns. Use `cache_control` on stable prefixes to reduce cost/latency.

### Example

```json
{
    "model": "claude-opus-4-8",
    "max_tokens": 1024,
    "messages": [
        {"role": "user", "content": "Hello, Claude"},
        {"role": "assistant", "content": "Hello!"},
        {"role": "user", "content": "Can you describe LLMs to me?"}
    ]
}
```

### Key Behaviors & Notes

- Each turn's user message + assistant response accumulate; previous turns are preserved completely.
- Context rot: accuracy/recall degrade as token count grows.
- For long conversations, use prompt caching (§13) and server-side compaction (§14).
- `usage.input_tokens` reports total input including all resent history (split across `input_tokens`, `cache_read_input_tokens`, `cache_creation_input_tokens` when caching is used).

---

## 6. Extended Thinking & Effort Control

### Main Concepts

Claude can perform **extended thinking** — internal reasoning via `thinking` content blocks generated before `text` blocks. Two thinking modes: **manual** (`type: "enabled"` + `budget_tokens`) and **adaptive** (`type: "adaptive"`, model decides). The `effort` parameter controls how eager Claude is to spend tokens (a behavioral signal, not a strict budget). For total-work budgets across an agentic loop, see **task budgets** (§15).

### Extended Thinking Parameters

`thinking` object variants:

| Variant | Fields | Description |
|---------|--------|-------------|
| `ThinkingConfigEnabled` | `type: "enabled"`, `budget_tokens` (number, ≥1024, < `max_tokens`), `display?` | Manual thinking with explicit budget. **Not supported** (400 error) on Fable 5, Mythos 5, Opus 4.8, Opus 4.7, Sonnet 5. **Deprecated** on Opus 4.6 & Sonnet 4.6. |
| `ThinkingConfigAdaptive` | `type: "adaptive"`, `display?` | Model decides thinking depth. Recommended on most current models. **Not disableable** on Mythos Preview, Fable 5, Mythos 5. |
| `ThinkingConfigDisabled` | `type: "disabled"` | Turns thinking off. |

`display` field (optional): `"summarized"` (summary of full thinking) or `"omitted"` (empty thinking field, signature carries encrypted full thinking). Defaults vary by model: `"summarized"` on Opus 4.6/Sonnet 4.6/earlier Claude 4; `"omitted"` on Fable 5/Mythos 5/Sonnet 5/Opus 4.8/4.7/Mythos Preview. Invalid with `type: "disabled"`.

### Effort Parameter

`output_config.effort` (string enum, no beta header required):

| Level | Description | Available On |
|-------|-------------|-------------|
| `max` | Absolute maximum capability, no token-spending constraints | Fable 5, Mythos 5, Opus 4.8, Mythos Preview, Opus 4.7, Opus 4.6, Sonnet 5, Sonnet 4.6 |
| `xhigh` | Extended capability for long-horizon work (30+ min agentic/coding, millions of tokens) | Fable 5, Mythos 5, Opus 4.8, Opus 4.7, Sonnet 5 |
| `high` | High capability. **Equivalent to not setting the parameter** (default). | All effort-supported models |
| `medium` | Balanced, moderate token savings | All effort-supported models |
| `low` | Most efficient, significant token savings, some capability reduction | All effort-supported models |

**Default:** `high`. Supported models: Fable 5, Mythos 5, Opus 4.8, Mythos Preview, Opus 4.7, Opus 4.6, Sonnet 5, Sonnet 4.6, Opus 4.5.

### Response Content Blocks

| Block Type | Fields | Description |
|------------|--------|-------------|
| `thinking` | `thinking` (string), `signature` (string), `type` | Internal reasoning. `thinking` empty when `display: "omitted"`. |
| `redacted_thinking` | `data` (string, encrypted), `type` | Safety-redacted thinking. Opaque. |
| `text` | `text`, `type` | Final response text. |

### Usage Fields

- `output_tokens`: inclusive authoritative total for billing.
- `output_tokens_details.thinking_tokens`: raw reasoning tokens generated (≤ `output_tokens`). In streaming, appears only on the final `message_delta` event.

### Special Headers

- `interleaved-thinking-2025-05-14` (beta): required for interleaved thinking on Opus 4.5, Sonnet 4.5, earlier Claude 4 models. Deprecated on Opus 4.6/4.8, Sonnet 5/4.6, Mythos Preview. Not needed on Fable 5/Mythos 5.
- `output-300k-2026-03-24` (beta, Message Batches): raises output limit to 300k for Opus 4.8/4.7/4.6, Sonnet 5/4.6.

### Key Behaviors & Notes

- `budget_tokens` must be < `max_tokens` (except with interleaved thinking where it can exceed `max_tokens`).
- Minimum budget 1,024 tokens; diminishing returns above 32k.
- SDKs require streaming when `max_tokens` > 21,333 (client-side validation).
- **Thinking block preservation by model:** Opus 4.5+ and Sonnet 4.6+ keep all thinking blocks across turns; earlier Opus/Sonnet and all Haiku keep only the last turn; Mythos Preview keeps all.
- When using thinking with tool use, you must pass complete unmodified `thinking` blocks (with signature) back; modifying them → 400 `invalid_request_error`.
- **Not compatible with:** `temperature`/`top_k` modifications, forced tool use (`tool_choice: {type: "any"}` or `{type: "tool"}`), response pre-filling. When thinking is enabled, `top_p` can be set between 0.95 and 1.
- Changes to thinking budget invalidate cached message prefixes.
- A specialized system prompt is auto-included when thinking is enabled.
- `signature` is opaque, longer on Claude 4 models, compatible across platforms.
- ZDR eligible.

### Example

```json
{
  "model": "claude-sonnet-4-6",
  "max_tokens": 16000,
  "thinking": {"type": "enabled", "budget_tokens": 10000, "display": "omitted"},
  "messages": [{"role": "user", "content": "What is 27 * 453?"}]
}
```

```json
{
  "content": [
    {"type": "thinking", "thinking": "", "signature": "EosnCkYICxIMMb3LzNrMu..."},
    {"type": "text", "text": "The answer is 12,231."}
  ],
  "usage": {"input_tokens": 25, "output_tokens": 348, "output_tokens_details": {"thinking_tokens": 312}}
}
```

---

## 7. Streaming

### Main Concepts

Incremental streaming of Messages API responses via **server-sent events (SSE)**. Set `stream: true` in the request body. Content blocks are streamed with `content_block_start` → one or more `content_block_delta` → `content_block_stop`, each with an `index`. SDKs provide streaming helpers (`.stream()` context manager, `text_stream`, `.get_final_message()`).

### SSE Event Types

| Event | Data Shape | Description |
|-------|-----------|-------------|
| `message_start` | `{type, message: {full Message object with empty content, usage: {input_tokens, output_tokens: 1}}}` | Contains a full `Message` object with empty `content` |
| `content_block_start` | `{type, index, content_block: {...}}` | Start of a content block |
| `content_block_delta` | `{type, index, delta: {...}}` | Incremental content. Delta variants: `text_delta {text}`, `input_json_delta {partial_json}`, `thinking_delta {thinking}`, `signature_delta {signature}`, `citations_delta {citation}`, `compaction_delta {text}` |
| `content_block_stop` | `{type, index}` | End of a content block |
| `message_delta` | `{type, delta: {stop_reason, stop_sequence}, usage: {...}}` | `usage` here is **cumulative**. `stop_reason` provided here. |
| `message_stop` | `{type}` | End of message |
| `ping` | `{type}` | May appear any number of times |
| `error` | `{type, error: {type, message}}` | e.g. `overloaded_error` (HTTP 529) |

### Event Flow

```
message_start
  → (per content block: content_block_start + content_block_delta(s) + content_block_stop)
  → one or more message_delta
  → message_stop
```

### Key Behaviors & Notes

- `usage` in `message_delta` is cumulative.
- New event types may be added per versioning policy; handle unknown types gracefully.
- `input_json_delta` deltas are partial JSON strings; final `tool_use.input` is always an object.
- For thinking: `signature_delta` is sent just before `content_block_stop`. With `display: "omitted"`, no `thinking_delta` events are sent.
- Server-side fallback: a `fallback` content block arrives as `content_block_start`/`content_block_stop` pair with no deltas.
- **Error recovery:**
  - Claude 4.5 and earlier: resume by placing the partial response as the start of a new assistant message.
  - Claude 4.6 and later: place a user message instructing the model to continue (e.g. "Your previous response was interrupted and ended with [...]. Continue from where you left off.").
  - Tool use and extended thinking blocks cannot be partially recovered; resume from the most recent text block.
- SDKs require streaming when `max_tokens` is large to avoid HTTP timeouts (> 21,333 with extended thinking).

### Example

```json
{
  "model": "claude-opus-4-8",
  "max_tokens": 256,
  "stream": true,
  "messages": [{"role": "user", "content": "Hello"}]
}
```

SSE response:
```
event: message_start
data: {"type":"message_start","message":{"id":"msg_...","role":"assistant","content":[],"usage":{"input_tokens":25,"output_tokens":1}}}
event: content_block_start
data: {"type":"content_block_start","index":0,"content_block":{"type":"text","text":""}}
event: content_block_delta
data: {"type":"content_block_delta","index":0,"delta":{"type":"text_delta","text":"Hello"}}
event: content_block_stop
data: {"type":"content_block_stop","index":0}
event: message_delta
data: {"type":"message_delta","delta":{"stop_reason":"end_turn"},"usage":{"output_tokens":15}}
event: message_stop
data: {"type":"message_stop"}
```

---

## 8. Vision (Image Inputs)

### Main Concepts

Multimodal image understanding via `image` content blocks. Three image source types: **base64-encoded**, **URL reference**, and **file_id** (Files API). Images are treated as visual tokens in 28×28-pixel patches: cost = `⌈width / 28⌉ × ⌈height / 28⌉` visual tokens. Resolution tiers (high-resolution vs standard) control max long edge and max visual tokens.

### API Parameters

`image` content block within a message's `content` array:

| Source Type | Fields | Description |
|------------|--------|-------------|
| `base64` | `type: "base64"`, `media_type`, `data` | `media_type`: `image/jpeg`, `image/png`, `image/gif`, `image/webp` |
| `url` | `type: "url"`, `url` | HTTPS URL |
| `file` | `type: "file"`, `file_id` | Files API reference (requires `anthropic-beta: files-api-2025-04-14`) |

### Key Behaviors & Notes

- **Max images per request:** 100 (200k-context models), 600 (other models). If >20 images, stricter per-image dimension limit applies (keep each dimension ≤2000px).
- **Max image size:** 10 MB (base64, Claude API/claude.ai), 5 MB on Bedrock/Google Cloud. Max dimensions 8000×8000 px.
- Animations unsupported — only the first frame is used.
- **High-resolution tier** (auto, no beta header): Fable 5, Mythos 5, Opus 4.8, Opus 4.7, Sonnet 5 — max long edge 2576 px, max 4784 visual tokens. **Standard tier** (all others): 1568 px / 1568 tokens.
- On Bedrock/Google Cloud, only base64 sources available.
- Best practice: place images before text in a turn; label each image (`Image 1:`, `Image 2:`).
- Limitations: cannot name people; may hallucinate on <200px/rotated/low-quality images; approximate spatial reasoning/counting; cannot detect AI-generated images; refuses explicit images.
- `usage.input_tokens` includes visual tokens.

### Example

```json
{
  "model": "claude-opus-4-8",
  "max_tokens": 1024,
  "messages": [{
    "role": "user",
    "content": [
      {"type": "image", "source": {"type": "url", "url": "https://example.com/ant.jpg"}},
      {"type": "text", "text": "What is in this image?"}
    ]
  }]
}
```

---

## 9. PDF & Document Inputs

### Main Concepts

`document` content blocks carry PDFs (text + images + charts + tables). Three source types like images: URL, base64, and `file_id`. PDFs are processed as page images, leveraging Claude's vision capabilities. Integrates with prompt caching and Message Batches API.

### API Parameters

`document` content block:

| Source Type | Fields | Description |
|------------|--------|-------------|
| `url` | `type: "url"`, `url` | HTTPS PDF URL |
| `base64` | `type: "base64"`, `media_type: "application/pdf"`, `data` | Base64-encoded PDF |
| `file` | `type: "file"`, `file_id` | Files API reference (requires `anthropic-beta: files-api-2025-04-14`) |

Optional fields: `cache_control` (prompt caching), `title`, `context`, `citations` (see §12).

### Key Behaviors & Notes

| Requirement | Limit |
|-------------|-------|
| Maximum request size | 32 MB (varies by platform) |
| Maximum pages per request | 600 (100 for 200k-context models) |
| Format | Standard PDF (no passwords/encryption) |

- Both limits apply to the entire request payload, not just the PDF.
- Dense PDFs can fill the context window before the page limit; split documents or downsample embedded images.
- Subject to the same vision limitations as images.
- Available on Claude API, Claude Platform on AWS, Amazon Bedrock, Google Cloud, Microsoft Foundry; all active models support it.
- On Bedrock/Google Cloud, only base64 sources available.
- Token cost = text extracted + page count (pages processed as images).
- ZDR eligible.

### Example

```json
{
  "model": "claude-opus-4-8",
  "max_tokens": 1024,
  "messages": [{
    "role": "user",
    "content": [
      {"type": "document", "source": {"type": "url", "url": "https://example.com/report.pdf"}},
      {"type": "text", "text": "What are the key findings in this document?"}
    ]
  }]
}
```

---

## 10. Files API

### Main Concepts

Upload-once / reference-many file storage keyed by `file_id`. Upload files once, then reference them by `file_id` in Messages requests (via `image`, `document`, or `container_upload` content blocks). In beta; **not** eligible for ZDR. Files are scoped to the workspace of the uploading API key.

### API Endpoints

| Endpoint | Method | Description |
|----------|--------|-------------|
| `/v1/files` | POST | Upload (multipart: `file=(filename, fileobj, mime_type)`) |
| `/v1/files` | GET | List (paginated: `limit` default 20, `before_id`, `after_id`) |
| `/v1/files/{file_id}` | GET | Retrieve metadata |
| `/v1/files/{file_id}` | DELETE | Delete |
| `/v1/files/{file_id}/download` | GET | Download (only `downloadable: true` files) |

### Special Header

`anthropic-beta: files-api-2025-04-14` (SDK adds automatically on `beta.files` methods; must pass via `betas` param for Messages referencing files).

### File Type → Block Mapping

| File Type | MIME Type | Content Block Type |
|-----------|-----------|-------------------|
| PDF | `application/pdf` | `document` |
| Plain text | `text/plain` | `document` |
| Images | `image/jpeg`, `image/png`, `image/gif`, `image/webp` | `image` |
| Datasets, others | Varies | `container_upload` |

### Upload Response Fields

`id` (e.g. `file_011...`), `type: "file"`, `filename`, `mime_type`, `size_bytes`, `created_at`, `downloadable` (false for user uploads).

### Key Behaviors & Notes

- **Storage limits:** max 500 MB per file; 500 GB total per organization.
- `downloadable: false` for user-uploaded files; only skill/code-execution-generated files can be downloaded (downloading an uploaded file returns 400).
- Files cannot be modified/renamed after upload; delete + re-upload to change content.
- Filename rules: 1–255 chars, forbidden chars `< > : " | ? * \ /` and Unicode 0–31.
- Free operations (upload/download/list/metadata/delete); file content in Messages priced as input tokens.
- Rate limits (beta): ~100 file-related requests/minute.
- Available on Claude API, Claude Platform on AWS, Microsoft Foundry (Hosted on Anthropic). **Not** on Bedrock or Google Cloud.
- For unsupported formats (.docx, .xlsx): convert to plain text or PDF.

### Example

```python
uploaded = client.beta.files.upload(
    file=("document.pdf", open("/path/to/document.pdf", "rb"), "application/pdf")
)
file_id = uploaded.id

client.beta.messages.create(
    model="claude-opus-4-8", max_tokens=1024,
    betas=["files-api-2025-04-14"],
    messages=[{"role": "user", "content": [
        {"type": "document", "source": {"type": "file", "file_id": file_id}},
        {"type": "text", "text": "Please summarize this document."}
    ]}]
)
```

---

## 11. Structured Outputs (JSON Schema Enforcement)

### Main Concepts

Constrain Claude's responses to follow a specific JSON schema via **constrained decoding** (compiled grammars). Two complementary features: **JSON outputs** (`output_config.format`) to control response format, and **strict tool use** (`strict: true` on tool definitions) to guarantee schema validation on tool inputs. Guarantees: always valid JSON, type-safe, reliable (no retries for schema violations). Grammar compilation + 24-hour caching of compiled grammars.

### API Parameters

`output_config` (top-level object):

| Field | Type | Description |
|-------|------|-------------|
| `format.type` | string | `"json_schema"` |
| `format.schema` | object | A JSON Schema definition (`type`, `properties`, `required`, `additionalProperties`, etc.) |

(Legacy/beta: `output_format` parameter — moved to `output_config.format`; Python SDK `client.messages.parse()` accepts `output_format` as convenience.)

`tools[].strict` (boolean, per tool): set `true` to enforce JSON Schema compliance on that tool's `input_schema`.

### Key Behaviors & Notes

- **GA model support (Claude API):** Fable 5, Mythos 5, Opus 4.8, Mythos Preview, Opus 4.7, Opus 4.6, Sonnet 5, Sonnet 4.6, Sonnet 4.5, Opus 4.5, Haiku 4.5.
- **Grammar compilation/caching:** First request with a given schema has extra latency to compile; compiled grammars cached 24 hours from last use; cache invalidated if JSON schema structure changes (changing only `name`/`description` does NOT invalidate).
- Claude auto-receives an additional system prompt explaining the output format (slightly higher input token count).
- **Property ordering:** required properties appear first (in schema order), then optional properties. Mark all properties required or account for reordering.
- **Invalid outputs:**
  - Refusals: `stop_reason: "refusal"`, 200 status, billed for tokens, output may not match schema.
  - Token limit: `stop_reason: "max_tokens"`, output may be incomplete; retry with higher `max_tokens`.
  - Enum casing: capitalization of `enum`/`const` values not guaranteed; compare case-insensitively.

### Schema Complexity Limits

| Limit | Value | Description |
|-------|-------|-------------|
| Strict tools per request | 20 | Max tools with `strict: true` |
| Optional parameters | 24 | Total optional params across all strict schemas and JSON output schemas |
| Parameters with union types | 16 | Total params using `anyOf` or type arrays (exponential compilation cost) |

Exceeding limits → 400 "Schema is too complex for compilation." Final stop-gap: 180-second compilation timeout.

### Compatibility

- **Works with:** Batch processing (50% discount), Token counting (no compilation), Streaming, combined JSON outputs + strict tool use.
- **Incompatible with:** Citations (returns 400 if citations enabled with `output_config.format`), Message Prefilling.
- Grammar applies only to Claude's direct output, not tool-use calls or thinking tags.
- ZDR eligible with limited technical retention; JSON schema cached up to 24h. HIPAA eligible but PHI must **not** be included in JSON schema definitions.

### Example

```json
{
  "model": "claude-opus-4-8",
  "max_tokens": 1024,
  "messages": [{"role": "user", "content": "Extract key info from this email: ..."}],
  "output_config": {
    "format": {
      "type": "json_schema",
      "schema": {
        "type": "object",
        "properties": {
          "name": {"type": "string"},
          "email": {"type": "string"},
          "plan_interest": {"type": "string"},
          "demo_requested": {"type": "boolean"}
        },
        "required": ["name", "email", "plan_interest", "demo_requested"],
        "additionalProperties": false
      }
    }
  }
}
```

Response: `{"content": [{"type": "text", "text": "{\"name\":\"John Smith\",\"email\":\"[email protected]\",...}"}]}`

---

## 12. Citations & Search Results (Document Grounding)

### 12a. Citations

**Source:** https://platform.claude.com/docs/en/build-with-claude/citations

#### Main Concepts

Ground Claude's responses in source documents you provide; the API returns exact passages (`cited_text`) supporting each claim. Three document types: **plain text**, **PDF**, and **custom content** documents. Each has a different chunking strategy and citation-location format. Citable content = text in a document's `source`; `title` and `context` are passed to the model but are **not** citable.

#### API Parameters

Enable on each `document` content block:
- `citations: {enabled: true}` — must be enabled on **all or none** of the documents in a request.

Document `source` variants:
- Plain text: `{type: "text", media_type: "text/plain", data: "..."}`
- PDF: base64 / URL / `file_id`
- Custom content: `{type: "content", content: [{type: "text", text: "..."}, ...]}` (blocks used as-is, no further chunking)

Optional: `title`, `context` (metadata, not citable), `cache_control`.

#### Response Citation Object Variants

| Citation Type | Location Fields | Use Case |
|--------------|-----------------|----------|
| `char_location` | `start_char_index`, `end_char_index` (0-indexed, exclusive) | Plain text documents |
| `page_location` | `start_page_number`, `end_page_number` (1-indexed, exclusive) | PDF documents |
| `content_block_location` | `start_block_index`, `end_block_index` (0-indexed, exclusive) | Custom content documents |

All citation objects also include: `cited_text`, `document_index` (0-indexed across all documents), `document_title`.

#### Key Behaviors & Notes

- All active models support citations **except Claude Haiku 3**.
- Only **text** citations supported; image citations not yet possible. PDFs that are scans without extractable text are not citable.
- Citations must be enabled on all or none of the documents in a request.
- **Incompatible with structured outputs** (`output_config.format`) — returns 400.
- `cited_text` does **not** count toward output tokens; when passed back in later turns, also does not count toward input tokens.
- Compatible with prompt caching, token counting, batch processing.
- Streaming: citations arrive as `citations_delta` delta type inside `content_block_delta` events.
- ZDR eligible.

#### Example

```json
{
  "model": "claude-opus-4-8",
  "max_tokens": 1024,
  "messages": [{
    "role": "user",
    "content": [
      {
        "type": "document",
        "source": {"type": "text", "media_type": "text/plain", "data": "The grass is green. The sky is blue."},
        "title": "My Document",
        "citations": {"enabled": true}
      },
      {"type": "text", "text": "What color is the grass and sky?"}
    ]
  }]
}
```

### 12b. Search Results

**Source:** https://platform.claude.com/docs/en/build-with-claude/search-results

#### Main Concepts

`search_result` content blocks give first-class, web-search-quality citations for RAG: each block carries a `source` (URL/id), `title`, and a `content` array of text blocks that Claude can cite. Two delivery methods: (1) returned from tool calls (dynamic RAG), (2) placed directly as top-level content in user messages (pre-fetched/cached). The text block is the minimal citable unit. Citations are **all-or-nothing**: either all `search_result` blocks have `citations.enabled=true` or all disabled.

#### API Parameters

`search_result` block schema:

| Field | Type | Required | Description |
|-------|------|----------|-------------|
| `type` | string | Yes | Must be `"search_result"` |
| `source` | string | Yes | Source URL or identifier |
| `title` | string | Yes | Descriptive title |
| `content` | array of text blocks | Yes | Text blocks containing the actual content (≥1 block) |
| `citations` | `{enabled: boolean}` | No | Citation configuration (disabled by default) |
| `cache_control` | object | No | Cache control settings |

#### Response Citation Object

`search_result_location` citation fields:

| Field | Type | Description |
|-------|------|-------------|
| `type` | string | Always `"search_result_location"` |
| `source` | string | Source from the original search result |
| `title` | string \| null | Title from the original search result |
| `cited_text` | string | Full text of cited block(s), concatenated. Not counted toward output tokens. |
| `search_result_index` | integer | 0-based index among all `search_result` blocks in the request |
| `start_block_index` | integer | 0-based index of first cited block in `content` array |
| `end_block_index` | integer | Exclusive end index (always > `start_block_index`) |

#### Key Behaviors & Notes

- Available models: Opus 4.8, 4.7, 4.6, Sonnet 5, 4.6, 4.5, Opus 4.5, Opus 4.1 (deprecated), Opus 4 (retired except Google Cloud), Sonnet 4 (retired except Bedrock/Google Cloud), Haiku 4.5, Haiku 3.5 (retired except Bedrock/Google Cloud).
- Available on Claude API, Amazon Bedrock, Google Cloud.
- Only text content supported (no images/media).
- Citations **disabled by default**; must explicitly set `citations.enabled=true`.
- ZDR eligible.

#### Example

```json
{
  "model": "claude-opus-4-8",
  "max_tokens": 1024,
  "messages": [{
    "role": "user",
    "content": [
      {
        "type": "search_result",
        "source": "https://docs.company.com/api-reference",
        "title": "API Reference - Authentication",
        "content": [{"type": "text", "text": "All API requests must include an API key..."}],
        "citations": {"enabled": true}
      },
      {"type": "text", "text": "How do I authenticate?"}
    ]
  }]
}
```

---

## 13. Prompt Caching

### Main Concepts

Cache prompt prefixes to reduce cost and latency for repetitive/consistent prompts. Two enabling modes: **automatic caching** (single top-level `cache_control` field; system applies breakpoint to the last cacheable block and moves it forward as conversations grow) and **explicit cache breakpoints** (place `cache_control` directly on individual content blocks). Cache hierarchy: `tools` → `system` → `messages` (each level builds on the previous). TTLs: 5-minute (default) and 1-hour (extended).

### API Parameters

| Parameter | Type | Description |
|-----------|------|-------------|
| `cache_control` (top-level) | `{type, ttl}` | Automatic caching. `type: "ephemeral"`, `ttl: "5m"` (default) or `"1h"`. Auto-applies a breakpoint to the last cacheable block. |
| `cache_control` (on content block) | `{type, ttl}` | Explicit cache breakpoint on a specific block. Same fields. |

Up to **4 cache breakpoints** per request; automatic caching consumes one slot.

### Response Usage Fields

| Field | Description |
|-------|-------------|
| `cache_creation_input_tokens` | Tokens written to cache |
| `cache_read_input_tokens` | Tokens retrieved from cache |
| `input_tokens` | Tokens NOT from cache (after last breakpoint) |
| `cache_creation` | `{ephemeral_5m_input_tokens, ephemeral_1h_input_tokens}` (1-hour breakdown) |

**Total input** = `cache_read_input_tokens` + `cache_creation_input_tokens` + `input_tokens`.

### Key Behaviors & Notes

- **Pricing multipliers:** 5m cache writes = 1.25× base input; 1h cache writes = 2× base input; cache reads = 0.1× base input. Stacks with Batch API discount.
- Supported on all active Claude models.
- **Minimum cacheable prompt length** varies by model: 512 tokens (Fable 5, Mythos 5); 1,024 tokens (Opus 4.8, Sonnet 5/4.6/4.5, Opus 4.1/4); 2,048 tokens (Mythos Preview, Opus 4.7); 4,096 tokens (Opus 4.6, 4.5, Haiku 4.5).
- Sub-minimum prompts are processed without caching (no error). Verify via usage fields.
- **What can be cached:** tools, system messages, text messages (user & assistant), images & documents (user turns), tool use and tool results.
- **What cannot be cached directly:** thinking blocks (but cached alongside other content in prior assistant turns), sub-content blocks like citations, empty text blocks.
- **Cache invalidation:** Tool definitions change → invalidates all levels. Tool choice change → invalidates messages only. Images/thinking parameters change → invalidates messages only.
- **Mixing TTLs:** longer TTL must appear before shorter (1h before 5m).
- Cache entry only available after first response begins; for parallel requests, wait for first response.
- Cache isolation: per organization; per workspace on Claude API, AWS, Microsoft Foundry (Bedrock & Google Cloud remain org-level).
- Exact matching required (100% identical including text and images up to and including the breakpoint).
- `max_tokens: 0` pre-warm is rejected if combined with `stream: true`, `thinking.type: "enabled"`, `output_config.format`, forced `tool_choice`, or inside Message Batches.
- ZDR eligible.

### Example

```json
{
  "model": "claude-opus-4-8",
  "max_tokens": 1024,
  "cache_control": {"type": "ephemeral"},
  "system": [
    {"type": "text", "text": "You are a helpful assistant.", "cache_control": {"type": "ephemeral", "ttl": "1h"}}
  ],
  "messages": [{"role": "user", "content": "What are the key terms?"}]
}
```

---

## 14. Context Windows & Context Management

### 14a. Context Windows

**Source:** https://platform.claude.com/docs/en/build-with-claude/context-windows

#### Main Concepts

The context window is all text the model can reference when generating a response (including the response itself) — the model's "working memory," distinct from training corpus. Progressive token accumulation: each turn accumulates. Context rot: accuracy/recall degrade as token count grows. Certain models track remaining token budget automatically (context awareness).

#### Context Window Sizes

| Size | Models |
|------|--------|
| 1M tokens | Opus 4.8, 4.7, 4.6; Sonnet 5, 4.6; Fable 5, Mythos 5, Mythos Preview |
| 200k tokens | Other models including Sonnet 4.5 |

#### Key Behaviors & Notes

- Everything counts: system prompt, all `messages` (incl. tool results, images, documents), tool definitions, and output Claude generates (including extended thinking).
- Extended thinking budget tokens are a subset of `max_tokens`, billed as output tokens, count toward rate limits.
- **Context awareness models:** Sonnet 5, 4.6, 4.5, Haiku 4.5 — automatic (API injects budget tags). Newer models (Opus 4.7+, Fable 5, Mythos 5) don't receive injected tags; use task budgets instead.
- Up to 600 images/PDF pages per request (100 for 200k-window models).
- **Overflow behavior:**
  - Input alone exceeds window → 400 `invalid_request_error` ("prompt is too long").
  - Input + `max_tokens` exceeds window: Claude 4.5+ accepts and stops with `stop_reason: "model_context_window_exceeded"`; earlier models return a validation error (opt in via `model-context-window-exceeded-2025-08-26` beta header).
- Cached prompt prefixes still occupy the context window (caching changes price, not count).
- Beta header `interleaved-thinking-2025-05-14` required for interleaved thinking on Opus 4.5, Sonnet 4.5, earlier Claude 4.

### 14b. Compaction (Server-side Context Compaction)

**Source:** https://platform.claude.com/docs/en/build-with-claude/compaction

#### Main Concepts

Server-side context compaction automatically summarizes older conversation context when approaching the context-window limit, extending effective context length. When the trigger threshold is hit, the API generates a summary and emits a `compaction` block at the start of the assistant response. On subsequent requests, all content blocks prior to the `compaction` block are dropped.

#### API Parameters

`context_management.edits` array containing a compaction edit object:

| Parameter | Type | Default | Description |
|-----------|------|---------|-------------|
| `type` | string | Required | Must be `"compact_20260112"` |
| `trigger` | `{type, value}` | `{type: "input_tokens", value: 150000}` | When to trigger. `input_tokens` is the only supported type. `value` must be ≥ 50,000. |
| `pause_after_compaction` | boolean | `false` | Whether to pause after generating the compaction summary |
| `instructions` | string | `null` | Custom summarization prompt (completely replaces default) |

**Beta header:** `anthropic-beta: compact-2026-01-12`.

#### Response

- Assistant `content` array begins with a `compaction` block: `{type: "compaction", content: "Summary..."}` followed by `text` blocks.
- When `pause_after_compaction: true`, response stops with `stop_reason: "compaction"` containing only the compaction block.
- `usage.iterations` array: each entry `{type, input_tokens, output_tokens}` with `type` `"compaction"` and/or `"message"`. Top-level `usage` = sum of non-compaction iterations only.
- Streaming: `compaction` block streams as `content_block_start`, then a single `content_block_delta` with `delta.type: "compaction_delta"`, then `content_block_stop`.

#### Key Behaviors & Notes

- **Supported models:** Fable 5, Mythos 5, Mythos Preview, Opus 4.8, 4.7, 4.6, Sonnet 5, 4.6.
- Must pass the `compaction` block back in subsequent requests (simplest: append full `response.content` to messages).
- Same model used for summarization — no option to use a cheaper model.
- Compaction may fail when `tools` are defined: the model may call a tool instead of writing a summary, yielding a `compaction` block with `content: null`. Mitigate with custom `instructions`.
- `trigger.value` minimum is 50,000 tokens.
- With server tools, compaction may occur multiple times in one request.
- Re-applying a previous `compaction` block incurs no additional compaction cost.
- Prompt caching: add `cache_control` on the compaction block; cache a system-prompt breakpoint separately.
- Token-counting endpoint applies existing `compaction` blocks but does not trigger new compactions; returns `context_management.original_input_tokens`.
- ZDR eligible.

#### Example

```json
{
  "model": "claude-opus-4-8",
  "max_tokens": 4096,
  "messages": [{"role": "user", "content": "Help me build a website"}],
  "context_management": {
    "edits": [{"type": "compact_20260112", "trigger": {"type": "input_tokens", "value": 150000}}]
  }
}
```

### 14c. Context Editing

**Source:** https://platform.claude.com/docs/en/build-with-claude/context-editing

#### Main Concepts

Selectively clear specific content from conversation history as it grows; applied **server-side before the prompt reaches Claude**. Client keeps the full unmodified history locally. Two server-side strategies: **tool result clearing** (`clear_tool_uses_20250919`) and **thinking block clearing** (`clear_thinking_20251015`).

#### API Parameters

`context_management.edits` array with strategy objects. **Beta header:** `anthropic-beta: context-management-2025-06-27`.

**Tool result clearing (`clear_tool_uses_20250919`):**

| Option | Default | Description |
|--------|---------|-------------|
| `trigger` | 100,000 input tokens | When strategy activates (`input_tokens` or `tool_uses`) |
| `keep` | 3 tool uses | How many recent tool use/result pairs to keep (oldest removed first) |
| `clear_at_least` | None | Minimum tokens cleared each time |
| `exclude_tools` | None | Tool names whose results should never be cleared |
| `clear_tool_inputs` | `false` | Whether tool call parameters are also cleared |

**Thinking block clearing (`clear_thinking_20251015`):**

| Option | Default | Description |
|--------|---------|-------------|
| `keep` | Model-specific | How many recent assistant turns with thinking to preserve (`{type: "thinking_turns", value: N}` or `"all"`). Opus 4.5+/Sonnet 4.6+: all turns. Earlier: last turn only. |

When combining both, `clear_thinking_20251015` **must be listed first** in the `edits` array.

#### Response

`context_management.applied_edits` array:
- Thinking: `{type: "clear_thinking_20251015", cleared_thinking_turns, cleared_input_tokens}`
- Tool: `{type: "clear_tool_uses_20250919", cleared_tool_uses, cleared_input_tokens}`

In streaming, included in the final `message_delta` event. Token-counting endpoint returns `input_tokens` (after editing) and `context_management.original_input_tokens` (before).

#### Key Behaviors & Notes

- Available on **all supported Claude models**.
- Tool result clearing **invalidates cached prompt prefixes** when content is cleared (use `clear_at_least` to make invalidation worthwhile).
- Thinking block clearing: when thinking blocks are **kept**, prompt cache is preserved; when **cleared**, cache is invalidated at that point.
- Client-side SDK compaction (`compaction_control`) is deprecated; use server-side `compact_20260112` instead.
- ZDR eligible.

#### Example

```json
{
  "model": "claude-opus-4-8",
  "max_tokens": 4096,
  "messages": [{"role": "user", "content": "Create a calculator app"}],
  "context_management": {
    "edits": [{
      "type": "clear_tool_uses_20250919",
      "trigger": {"type": "input_tokens", "value": 30000},
      "keep": {"type": "tool_uses", "value": 3},
      "clear_at_least": {"type": "input_tokens", "value": 5000},
      "exclude_tools": ["web_search"]
    }]
  }
}
```

---

## 15. Task Budgets

### Main Concepts

Advisory (soft) token budget for a full agentic loop — covering thinking, tool calls, tool results, and output — to help Claude self-regulate and finish gracefully. Claude sees a server-injected running countdown marker (visible only to the model, not in the API response). Complements the `effort` parameter: effort = per-step reasoning depth; task budgets = total work across the loop.

### API Parameters

Set via `output_config.task_budget` with beta header `anthropic-beta: task-budgets-2026-03-13`:

| Field | Type | Description |
|-------|------|-------------|
| `type` | string | Always `"tokens"` |
| `total` | number | Token budget for the full loop. **Minimum 20,000** (below → 400). |
| `remaining` | number (optional) | Remainder carried over from a prior request; defaults to `total` when omitted. Use to continue a budget across context compaction. |

`output_config.effort` is commonly set alongside.

### Key Behaviors & Notes

- **Advisory, not enforced.** Claude may exceed it if interrupting mid-action is worse. Hard cap remains `max_tokens` (`stop_reason: "max_tokens"`).
- Too-small budget → refusal-like behavior (Claude declines, scopes down, or stops early).
- Countdown visible only to model — no `task_budget` in response `usage`; track spend client-side by summing `usage.output_tokens` across requests.
- Mutating `task_budget.remaining` between requests invalidates cache prefixes containing it.
- **Model support (beta):** Fable 5, Mythos 5, Opus 4.8, Opus 4.7. **Not** supported on Sonnet 5, Opus 4.6, Sonnet 4.6, Haiku 4.5.
- At `xhigh`/`max` effort, set `max_tokens` to ≥64k per request.
- ZDR eligible.

### Example

```python
with client.beta.messages.stream(
  model="claude-opus-4-8",
  max_tokens=128000,
  output_config={
    "effort": "high",
    "task_budget": {"type": "tokens", "total": 64000},
  },
  messages=[{"role": "user", "content": "Review the codebase and propose a refactor plan."}],
  betas=["task-budgets-2026-03-13"],
) as stream:
  response = stream.get_final_message()
```

---

## 16. Batch Processing (Message Batches)

### Main Concepts

Asynchronous bulk processing of Messages requests at **50% of standard API prices**. Each batch = a list of `{custom_id, params}` requests processed independently/concurrently. Poll for batch status; retrieve `.jsonl` results via `results_url`. Four result types: `succeeded`, `errored`, `canceled`, `expired`. Supports vision, tool use, system messages, multi-turn, extended thinking, most beta features. **Not** eligible for ZDR; data retained up to 29 days.

### API Endpoints

| Endpoint | Method | Description |
|----------|--------|-------------|
| `/v1/messages/batches` | POST | Create batch (body: `requests: [{custom_id, params}]`) |
| `/v1/messages/batches/{batch_id}` | GET | Retrieve status |
| `/v1/messages/batches` | GET | List (paginated, `limit`) |
| `/v1/messages/batches/{batch_id}/cancel` | POST | Cancel |
| `/v1/messages/batches/{batch_id}/results` | GET | Stream results (`.jsonl`) |
| `/v1/messages/batches/{batch_id}` | DELETE | Delete after processing |

### Parameters

- `custom_id`: string, 1–64 chars, `^[a-zA-Z0-9_-]{1,64}$`
- `params`: standard Messages API params (model, max_tokens, messages, system, tools, thinking, etc.)
- **Beta header:** `anthropic-beta: output-300k-2026-03-24` (extended output; raises `max_tokens` cap to 300,000 for Opus 4.8/4.7/4.6, Sonnet 5/4.6)

### Unsupported Batch Parameters

| Parameter | Why |
|-----------|-----|
| `stream: true` | Results come back as a single file, not a stream |
| `speed` (Fast mode) | Tunes synchronous latency, doesn't apply to async batch |
| `store` / `previous_thread_event_id` (Threads) | Threads are stateful; batch requests are not |
| `max_tokens: 0` | Cache pre-warming not supported in a batch |

### Batch Response Fields

`id` (`msgbatch_...`), `type: "message_batch"`, `processing_status` (`in_progress` → `ended`; `canceling`), `request_counts: {processing, succeeded, errored, canceled, expired}`, `created_at`, `expires_at`, `ended_at`, `results_url`.

### Key Behaviors & Notes

- **Batch limits:** 100,000 requests OR 256 MB, whichever first. Most batches finish <1 hour. Expire if not done within 24 hours.
- Results available 29 days after `created_at`; after that, batch is viewable but results not downloadable.
- Batches scoped to a Workspace; rate limits apply to both HTTP calls and queued requests.
- Each request must have `max_tokens ≥ 1`.
- `invalid_request_error` results are not billable; errored/canceled/expired are not billed.
- Results may not match input order — match by `custom_id`.
- Prompt caching stacks with batch discount; cache hits best-effort (30–98% hit rates); consider 1-hour cache duration.
- Extended output (beta) available only on Batches API (Claude API + Claude Platform on AWS), not sync/Bedrock/Google Cloud/Foundry.
- 413 `request_too_large` if >256 MB.

### Example

```json
{
  "requests": [
    {"custom_id": "my-first-request", "params": {
      "model": "claude-opus-4-8", "max_tokens": 1024,
      "messages": [{"role": "user", "content": "Hello, world"}]
    }},
    {"custom_id": "my-second-request", "params": {
      "model": "claude-opus-4-8", "max_tokens": 1024,
      "messages": [{"role": "user", "content": "Hi again"}]
    }}
  ]
}
```

---

## 17. Token Counting

### Main Concepts

Pre-send input token estimation for the same structured inputs as Messages. Accepts system prompts, tools, images, PDFs, extended thinking blocks. Estimate only; actual usage may differ slightly. Eligible for ZDR.

### API Endpoint

`POST /v1/messages/count_tokens` (`client.messages.count_tokens`)

Accepts the same params as Messages create (non-streaming): `model`, `system`, `messages`, `tools`, `thinking`, image/document content blocks.

### Response

```json
{"input_tokens": 14}
```

### Key Behaviors & Notes

- All active models supported, including Sonnet 5.
- Newer tokenizer (Opus 4.7+, Fable 5, Mythos 5, Sonnet 5) yields ~30% more tokens than earlier models — don't reuse counts across tokenizer generations.
- Extended thinking rules: thinking blocks from **previous** assistant turns are ignored (do NOT count); **current** assistant turn thinking DOES count.
- Server tool token counts apply only to the first sampling call.
- PDF support has the same limitations as the Messages API.
- Free to use, but subject to RPM limits by usage tier: Start (2,000), Build (4,000), Scale (8,000). Token-counting and message-creation rate limits are separate.
- Supports `context_management` parameter; returns `input_tokens` (after editing) and `context_management.original_input_tokens` (before).

### Example

```json
{
  "model": "claude-opus-4-8",
  "system": "You are a scientist",
  "messages": [{"role": "user", "content": "Hello, Claude"}]
}
```

---

## 18. Stop Reasons, Refusals & Fallback

### 18a. Handling Stop Reasons

**Source:** https://platform.claude.com/docs/en/build-with-claude/handling-stop-reasons

#### Main Concepts

The `stop_reason` field appears on every successful Messages API response (HTTP 200) and tells you why Claude stopped generating. Stop reasons are distinct from errors (4xx/5xx).

#### All `stop_reason` Enum Values

| Value | Description | Action |
|-------|-------------|--------|
| `end_turn` | Claude finished its response naturally | Use the response |
| `max_tokens` | Response reached the `max_tokens` limit | Raise `max_tokens` or continue |
| `stop_sequence` | Claude emitted one of your `stop_sequences` | Read `stop_sequence` field |
| `tool_use` | Claude is calling a tool | Run the tool and return the result |
| `pause_turn` | A server-tool loop reached its iteration limit (default 10) | Send the assistant content back to continue |
| `refusal` | Claude declined to respond | Read `stop_details`, retry on a fallback model |
| `model_context_window_exceeded` | The response filled the model's context window | Treat as truncated |

#### Key Behaviors & Notes

- `pause_turn` is never returned when a client `tool_use` is waiting — that case uses `tool_use`.
- `model_context_window_exceeded` returned without a beta header on Sonnet 4.5+; for earlier models add `model-context-window-exceeded-2025-08-26` beta header.
- Streaming: `stop_reason` is `null` in `message_start`, provided in `message_delta`, absent from other events.
- `refusal` on Claude Fable 5 is a normal HTTP 200, not an error.

### 18b. Refusals and Fallback

**Source:** https://platform.claude.com/docs/en/build-with-claude/refusals-and-fallback

#### Main Concepts

Claude Fable 5 has safety classifiers that can decline a request; a refusal is a successful HTTP 200 with `stop_reason: "refusal"`. The `stop_details` object identifies the policy category. Three fallback approaches: server-side fallback (single API call), SDK middleware (client-side), or manual retry.

#### `stop_details` Object

```json
{"type": "refusal", "category": "cyber"|"bio"|"frontier_llm"|"reasoning_extraction"|null, "explanation": "string|null"}
```

| Category | Description |
|----------|-------------|
| `"cyber"` | Request could enable cyber harm (malware/exploit development) |
| `"bio"` | Request could enable biological harm |
| `"frontier_llm"` | Request could assist development of competing AI models |
| `"reasoning_extraction"` | Request asks model to reproduce internal reasoning; use adaptive thinking instead |

Both fields `null` when the refusal doesn't map to a named category.

#### Server-side Fallback Parameters

| Parameter | Type | Description |
|-----------|------|-------------|
| `fallbacks` | array of `{model, max_tokens?, thinking?}` | Up to 3 entries, each distinct from requested model and each other; tried in order; each must be a permitted target (`allowed_fallback_models` on model's Models API entry) |
| `betas` | array | Must include exactly `"server-side-fallback-2026-06-01"` |

SDK middleware sends `fallback-credit-2026-06-01` on every request it handles.

#### Response Fields

- `model` (top-level): reports the model that produced the returned message.
- `content`: may include a `fallback` block: `{type: "fallback", from: {model}, to: {model}}`.
- `usage.iterations`: array of `{type: "message"|"fallback_message", ...token counts}` — records every attempt including declining models.

#### Key Behaviors & Notes

- **Billing:** Not billed for a refusal before any output (`content` empty, tokens not charged, doesn't count against rate limits). A mid-stream refusal bills input + output-so-far at normal rates.
- Server-side fallback: only a safety-classifier decline triggers it; rate limits, overload, or server errors returned as-is. Rejected on Message Batches API, Bedrock, Google Cloud, Microsoft Foundry.
- SDK middleware: `BetaRefusalFallbackMiddleware` via client constructor; share one `BetaFallbackState` across a conversation to pin follow-ups to the accepting model. Do not combine with server-side `fallbacks` on the same request.
- Continuing after a mid-output fallback: keep the `fallback` block exactly where it appeared; keep `text` and blocks after the final `fallback`; drop `thinking`/`redacted_thinking`/`connector_text` and client-side `tool_use` before the final `fallback`.
- Common pitfalls: retry on a **different** model (same model usually re-refuses); configure fallback on every request path; instrument refusals as their own signal (HTTP 200, so error-rate monitoring misses them); branch on `stop_reason`, not `stop_details` or `content`.

#### Example

```python
response = client.beta.messages.create(
    model="claude-fable-5",
    max_tokens=1024,
    messages=[{"role": "user", "content": "Hello, Claude"}],
    fallbacks=[{"model": "claude-opus-4-8"}],
    betas=["server-side-fallback-2026-06-01"],
)
print(response.model)  # may be "claude-opus-4-8" if Fable 5 refused
```

---

## 19. Fast Mode

### Main Concepts

Up to **2.5× higher output tokens per second (OTPS)** from supported Claude Opus models via a faster inference configuration — same model weights/intelligence, not a different model. Opt in with `speed: "fast"` + beta header. Benefit is on output tokens/sec, not time-to-first-token (TTFT). Research preview.

### API Parameters

| Parameter | Type | Description |
|-----------|------|-------------|
| `speed` | `"fast"` \| `"standard"` | Added to request body. `client.beta.messages.create`. |
| `betas` | array | Must include `"fast-mode-2026-02-01"` |

### Response

`usage` object gains a `speed` field (`"fast"` or `"standard"`).

Dedicated rate-limit response headers:
- `anthropic-fast-input-tokens-limit` / `-remaining` / `-reset`
- `anthropic-fast-output-tokens-limit` / `-remaining` / `-reset`

### Key Behaviors & Notes

- **Supported models:** Claude Opus 4.8 (`claude-opus-4-8`) and Opus 4.7 (`claude-opus-4-7`).
- Opus 4.7 fast mode deprecated June 25, 2026; removed July 24, 2026 (no fallback to standard; model itself stays available).
- Opus 4.6: fast mode **not** available — `speed: "fast"` runs at standard speed, billed standard rates, returns `usage.speed: "standard"` (no error).
- Any other model sending `speed: "fast"` returns an error.
- **Pricing:** per-model multiplier on standard rates. Opus 4.8: $10/$50 per MTok (in/out); Opus 4.7: $30/$150 per MTok. Stacks with prompt-caching and data-residency multipliers.
- Dedicated rate limit separate from standard Opus limits; SDKs auto-retry `429` up to 2× by default.
- Switching fast↔standard speed **invalidates the prompt cache** (cached prefixes are not shared).
- Not available with Batch API, Priority Tier, or Claude Platform on AWS.
- ZDR eligible.

### Example

```python
response = client.beta.messages.create(
  model="claude-opus-4-8",
  max_tokens=4096,
  speed="fast",
  betas=["fast-mode-2026-02-01"],
  messages=[{"role": "user", "content": "Refactor this module..."}],
)
print(response.usage.speed)  # "fast" or "standard"
```

---

## 20. Multilingual Support

### Main Concepts

Claude maintains strong cross-lingual performance relative to English, including in zero-shot tasks, across widely-spoken and lower-resource languages. The recommended control is the **system prompt**: explicitly state the target response language for production reliability (stable across all turns). Use native scripts (not transliteration); state both input and output languages explicitly for best results.

### API Parameters

No special parameters or beta headers. The only relevant request parameter is `system` (string or array of text blocks): used to pin the response language.

### Key Behaviors & Notes

- Performance data (zero-shot MMLU, % of English = 100%): Spanish ~96–98%, French ~96–98%, Chinese ~94–97%, Japanese ~93–97%, Arabic ~92–97%, Hindi ~92–97%, Bengali ~90–96%, Swahili ~78–91%, Yoruba ~53–80% (lowest). With extended thinking.
- Processes input/generates output in most world languages using standard Unicode.
- Best practices: provide clear language context, use native scripts, consider cultural/regional context.
- For translation, name both languages: `"Translate the user's message from German to Korean. Respond with only the translation."`
- For runtime user-selected language, interpolate the choice into the system prompt rather than relying on inference.

### Example

```json
{
  "model": "claude-opus-4-8",
  "max_tokens": 1024,
  "system": "Always respond in French, regardless of the language the user writes in.",
  "messages": [{"role": "user", "content": "How do I reset my password?"}]
}
```

---

## 21. Workspaces (Organization & Admin)

### Main Concepts

Organize API usage within an organization: separate projects/environments/teams with centralized billing & admin. Every org has a non-deletable **Default Workspace** (no ID, not in list endpoints). Workspace IDs use `wrkspc_` prefix; max 100 workspaces per org. API keys are scoped to a single workspace. Resources scoped to workspaces: Files, Message Batches, Skills, and prompt caches (per-workspace on Claude API/AWS/Microsoft Foundry; per-org on Bedrock & Google Cloud).

### Admin API Endpoints

Require Admin API key (`sk-ant-admin...`), header `anthropic-version: 2023-06-01` + `x-api-key: $ANTHROPIC_ADMIN_KEY`:

| Endpoint | Method | Description |
|----------|--------|-------------|
| `/v1/organizations/workspaces` | POST | Create (`{"name": "Production"}`) |
| `/v1/organizations/workspaces` | GET | List (`limit`, `include_archived`) |
| `/v1/organizations/workspaces/{id}/archive` | POST | Archive |
| `/v1/organizations/workspaces/{id}/members` | POST | Add member (`{user_id, workspace_role}`) |
| `/v1/organizations/workspaces/{id}/members/{user_id}` | POST | Update role |
| `/v1/organizations/workspaces/{id}/members/{user_id}` | DELETE | Remove member |
| `/v1/organizations/usage_report/messages` | GET | Usage/cost tracking (`workspace_ids[]`, `group_by[]`, `bucket_width`) |

### Roles

- **Workspace User** — Workbench only
- **Workspace Limited Developer** — keys+API, no session tracing/file download
- **Workspace Developer** — keys+API
- **Workspace Admin** — full control
- **Workspace Billing** — inherited from org billing role (cannot be manually assigned)

### Key Behaviors & Notes

- Org admins auto-get Workspace Admin in all workspaces; org billing members auto-get Workspace Billing; org users/developers must be added explicitly.
- Only org admins can create workspaces.
- Archiving preserves historical data, deactivates workspace + all API keys (cannot be undone).
- Limits: set per-model-tier rate limits and spend notifications per workspace; can be ≤ but not > org limits; cannot set limits on Default Workspace.
- Special **Claude Code workspace**: auto-created on first Claude Code sign-in; per-user keys; per-user monthly spend limits.

---

## 22. Embeddings (Voyage AI)

### Main Concepts

Text embeddings = numerical vectors measuring semantic similarity (search, recommendations, anomaly detection, RAG). **Anthropic does not offer its own embedding model** — recommends **Voyage AI**. Embeddings are normalized to length 1 → dot product == cosine similarity. Use `input_type="document"` when indexing, `input_type="query"` when embedding queries.

### API Functions

**Voyage Python lib:** `vo.embed(texts, model="voyage-4", input_type="document")` → `.embeddings` (list of float vectors, default 1024-dim). Install: `pip install -U voyageai`; env var `VOYAGE_API_KEY`.

**Voyage HTTP API:** `POST https://api.voyageai.com/v1/embeddings`, headers `Content-Type: application/json` + `Authorization: Bearer $VOYAGE_API_KEY`, body `{"input": [...], "model": "voyage-4"}`.

Contextualized chunk embeddings: `contextualized_embed()`. Rerankers: `rerank()`.

### Voyage Models

| Category | Models | Context | Dimensions |
|----------|--------|---------|------------|
| Voyage 4 (latest) | `voyage-4-large`, `voyage-4`, `voyage-4-lite`, `voyage-4-nano` (open-weight) | 32,000 | 1024 (default)/256/512/2048 |
| Previous gen | `voyage-3-large`, `voyage-3.5`, `voyage-3.5-lite`, `voyage-code-3`, `voyage-finance-2`, `voyage-law-2` (16k ctx) | 32,000 | 1024/256/512/2048 |
| Multimodal | `voyage-multimodal-3.5` (text+images+video), `voyage-multimodal-3` (text+images) | 32,000 | 1024/256/512/2048 |
| Contextualized chunk | `voyage-context-4`, `voyage-context-3` | 120,000 | 1024/256/512/2048 |
| Rerankers | `rerank-2.5`, `rerank-2.5-lite` | 32,000 | — |

### Example

```python
import voyageai
vo = voyageai.Client()
doc_embds = vo.embed(documents, model="voyage-4", input_type="document").embeddings
query_embd = vo.embed([query], model="voyage-4", input_type="query").embeddings[0]
similarities = np.dot(doc_embds, query_embd)  # normalized → dot == cosine
```

---

## 23. Capability Summary & Cross-Reference

### Complete Capability Matrix

| Capability | Mechanism | Section |
|-----------|-----------|---------|
| Text generation | `POST /v1/messages` | §2, §3 |
| Message roles (user/assistant/system) | `messages[]` + top-level `system` | §4 |
| Mid-conversation system messages | `messages[]` with `role: "system"` (Opus 4.8) | §4 |
| Multi-turn conversation | Stateless — resend `messages[]` | §5 |
| Output constraining (prefill) | `assistant` message as last entry (deprecated on newer models) | §3 |
| Extended thinking | `thinking` parameter (`enabled`/`adaptive`/`disabled`) | §6 |
| Effort control | `output_config.effort` (`low`/`medium`/`high`/`xhigh`/`max`) | §6 |
| Task budgets | `output_config.task_budget` (beta) | §15 |
| Streaming | `stream: true` (SSE events) | §7 |
| Vision (images) | `image` content blocks (base64/URL/file_id) | §8 |
| PDF / document input | `document` content blocks (base64/URL/file_id) | §9 |
| Files API | `/v1/files` endpoints (beta) | §10 |
| Structured outputs (JSON Schema) | `output_config.format` (`json_schema`) | §11 |
| Citations (document grounding) | `citations: {enabled: true}` on `document` blocks | §12a |
| Search result citations (RAG) | `search_result` content blocks | §12b |
| Prompt caching | `cache_control` (automatic or explicit breakpoints) | §13 |
| Context window management | Implicit (200k or 1M depending on model) | §14a |
| Server-side compaction | `context_management.edits` (`compact_20260112`, beta) | §14b |
| Context editing (clear tool results/thinking) | `context_management.edits` (`clear_tool_uses`/`clear_thinking`, beta) | §14c |
| Batch processing | `POST /v1/messages/batches` (50% discount) | §16 |
| Token counting | `POST /v1/messages/count_tokens` | §17 |
| Stop reasons | `stop_reason` enum on every response | §18a |
| Refusal fallback | `fallbacks` parameter + beta header | §18b |
| Fast mode | `speed: "fast"` + beta header (Opus 4.8/4.7) | §19 |
| Multilingual support | System prompt (no special params) | §20 |
| Workspaces (org/admin) | Admin API endpoints | §21 |
| Embeddings | Voyage AI (third-party, not Anthropic) | §22 |

### State Management Decision Matrix

| Scenario | Recommended Approach |
|----------|---------------------|
| Simple multi-turn | Resend full `messages[]` each turn (stateless API) |
| Repetitive prompts / cost reduction | Prompt caching (`cache_control`, 5m or 1h TTL) |
| Long conversation approaching context limit | Server-side compaction (`compact_20260112`) |
| Long agentic loop with many tool results | Context editing (`clear_tool_uses_20250919`) |
| Long agentic loop with thinking blocks | Context editing (`clear_thinking_20251015`) |
| Mid-conversation instruction change (Opus 4.8) | Mid-conversation `system` role message |
| Total work budget across agentic loop | Task budgets (`output_config.task_budget`, beta) |
| ZDR compliance | Most features ZDR-eligible; Files API and Batches are not |

### Structured Output Decision Matrix

| Need | Recommended Approach |
|------|----------------------|
| JSON response conforming to schema | `output_config.format` with `json_schema` |
| Schema-validated tool inputs | `tools[].strict: true` |
| Prefill (constrain output start) | `assistant` message as last entry (deprecated on newer models — use structured outputs instead) |
| Pin response language | `system` prompt with explicit language instruction |

### Thinking & Effort Decision Matrix

| Model | Recommended Thinking Config | Recommended Effort |
|-------|---------------------------|-------------------|
| Fable 5, Mythos 5 | Adaptive (always on, not disableable) | Use `effort` as primary control |
| Opus 4.8, 4.7, Sonnet 5 | Adaptive with `effort` | `xhigh` for coding/agentic, `high` default |
| Opus 4.6, Sonnet 4.6 | Adaptive with `effort` (manual `budget_tokens` deprecated) | Set `effort` explicitly |
| Opus 4.5, Haiku 4.5, earlier Claude 4 | Manual (`enabled` + `budget_tokens`) | N/A (use `budget_tokens`) |

### Feature Compatibility Matrix

| Feature | Compatible With | Incompatible With |
|---------|----------------|-------------------|
| Structured outputs | Batch, token counting, streaming, strict tool use | Citations, prefill |
| Citations | Prompt caching, token counting, batch | Structured outputs |
| Extended thinking | Tool use (`tool_choice: auto`/`none`), interleaved thinking (beta) | `temperature`/`top_k` mods, forced tool use, prefill |
| Prompt caching | Vision, PDF, files, thinking (cached alongside), tool results | Mutating cache prefix content |
| Fast mode | Prompt caching (separate cache), data residency | Batch API, Priority Tier, switching speed mid-conversation |
| Prefill | (deprecated on newer models) | Extended thinking, structured outputs on newer models |
| `max_tokens: 0` (cache pre-warm) | Prompt caching | `stream: true`, `thinking.enabled`, `output_config.format`, forced `tool_choice`, batches |
| Server-side fallback | SDK middleware (not on same request) | Message Batches, Bedrock, Google Cloud, Foundry |