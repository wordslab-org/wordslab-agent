# Global Architecture v2 — Consolidated AI Platform Services Map

> **Derived from:** `architecture.md` (v1), itself derived from the nine `summary.md` studies in `./platform-studies/{text,images,voice,documents,agents,tools,admin,observability,gpu}`.
> **Purpose:** a single hierarchical, vendor-neutral architecture reference covering every service (independently callable API) and service capability found across the nine studies, organized into **Layers → Modules → Services**, with transversal conventions extracted into a dedicated **Platform Architecture Standards** chapter and fundamental technical services grouped into a new **Layer 0 (L0)**.

---

## How to read this document

The platform is described as **six layers**, ordered bottom-up by dependency. A lower layer is *implemented on top of* the layer above it (i.e. higher layers call lower layers), or is *supervised by* an observability/administration layer that sits across the stack.

```
┌─────────────────────────────────────────────────────────────────────────────┐
│ L5  OBSERVABILITY & ADMINISTRATION  (supervises all layers below)            │
│   Telemetry/Traces · Logs/Dashboards · Evals/Judges/Campaigns/Datasets       │
│   Moderation/Guardrails · Approvals (HITL) · Safety & Adversarial Testing     │
│   Model Lifecycle Admin · Analytics Portals · Telemetry Backend Integrations │
├─────────────────────────────────────────────────────────────────────────────┤
│ L4  AGENTIC ORCHESTRATION  (built on L2 inference + L3 modalities + L0)       │
│   Agent Definition · Sessions/Runs · Agent Loop/Events · Tools/MCP/Skills    │
│   Permissions/Hooks · Multi-Agent Orchestration · Memory/RAG · Workflows      │
│   Channels · Voice (agent channel) · Extensions/Plugins/Marketplace          │
├─────────────────────────────────────────────────────────────────────────────┤
│ L3  AI MODALITY PRODUCTS  (built on L2 intelligence APIs + L0)                │
│   Text & Conversation · Images & Video · Voice · Documents                    │
├─────────────────────────────────────────────────────────────────────────────┤
│ L2  MODEL INFERENCE & INTELLIGENCE APIs  (built on L1 compute + L0)           │
│   Model Catalog · Generation APIs · Reasoning · Structured Output             │
│   Function/Tool Calling · Streaming · Context Mgmt · Multimodal Input         │
│   Embeddings/Rerank · Batch · Grounding/Citations                             │
├─────────────────────────────────────────────────────────────────────────────┤
│ L1  COMPUTE & MODEL SERVING  (built on L0)                                    │
│   Hardware/Model Catalog · Deployment CRUD · Engine Configuration             │
│   Lifecycle/Autoscaling · Routing · Request Execution · Output Control        │
│   Reliability · Health Checks                                                  │
├─────────────────────────────────────────────────────────────────────────────┤
│ L0  FUNDAMENTAL TECHNICAL SERVICES                                            │
│   Identity/Keys · Tenancy/Workspaces · RBAC · Network · Storage · Files       │
│   Databases (vector/graph) · Environments/Sandboxes · Workflows/Scheduled     │
│   Jobs · Billing/Usage/Spend · Quotas/Rate Limits · Processing Tiers          │
│   Compliance/Privacy/Legal · Data Residency/Encryption · Webhooks             │
│   Developer SDKs/CLI · Caching/Token Counting · Routing/Gateway control       │
└─────────────────────────────────────────────────────────────────────────────┘
```

### Layer dependency rules

1. **L0 → all**: L0 provides fundamental technical services (identity, storage, network, billing, sandboxes, databases, workflows, quotas, compliance, SDKs). Every higher layer depends on L0.
2. **L0 → L1**: L1 provisions GPUs/engines/deployments and builds inference serving on top of L0 base compute/network/storage/identity/billing services.
3. **L1 → L2**: L2 inference endpoints are served *by* L1 deployments.
4. **L2 → L3**: L3 modality products (text/image/voice/document) call L2 generation/embedding/structured-output primitives and L0 files/storage.
5. **L2 + L3 + L0 → L4**: L4 agents call L2 model APIs (model call + tools) and consume L3 products (voice channels, file search, image generation tool, code execution container) and L0 primitives (sandboxes, vaults, scheduled jobs, webhooks).
6. **L5 → all**: L5 governs (telemetry, traces, evals, moderation, guardrails, approvals, model-lifecycle admin, analytics) and wraps every other layer; it is neither below nor above in the call graph but supervisory, built on top of L0 (identity, billing, quotas, compliance, files, webhooks).

### How services and capabilities are documented

For every **service** we record:
- **Name**: a unique, descriptive name.
- **Arguments**: the main arguments/parameters of the API.
- **Description**: a short statement of what the service does.

For every **service family** we record its **capabilities**: the options, variants, mechanisms, and configuration knobs that may be available inside the services of that family. Capabilities are *not* independently callable endpoints — they are things a service may or may not support, or selectable modes within a single service call.

**Modules** are groups of closely-related services that provide similar functions to the user. Modules are grouped by modality (in L3) or by functional area (in L0/L1/L2/L4/L5).

> **Note on providers.** This document is vendor-neutral. No provider names, provider capability matrices, cross-vendor comparison tables, or `*Providers*:` annotations appear. All such material from v1 has been removed; the underlying services and capabilities are documented in their canonical, vendor-neutral form.

---

# PLATFORM ARCHITECTURE STANDARDS (transversal)

This chapter records concepts, rules, best practices, and standards that are **transversal to all layers** — they enable consistency between every service in the platform. Items here are referenced from, but not duplicated in, the layer chapters.

## S.1 Authentication header conventions

Every API call carries one of:
- `Authorization: Bearer <token>` (de-facto standard; long-lived API key or short-lived bearer).
- `x-api-key: <key>` (alternative header).
- `x-goog-api-key: <key>` (alternative header).

Additional standard headers:
- `OpenAI-Organization: <org>` — org-scoped calls.
- `x-goog-user-project: <project>` — project-scoped calls.
- `OpenAI-Safety-Identifier: <id>` / `safety_identifier` — caller safety identifier.
- `x-goog-api-client: company-product/version` — mandatory partner integration header.

## S.2 API versioning & beta headers

- **Dated API versions**: path prefixes `/v1`, `/v0`; query parameter `api-version=YYYY-MM-DD`; request header `anthropic-version: 2023-06-01`.
- **Beta feature headers**: `anthropic-beta: <feature-tag-YYYY-MM-DD>` (repeatable); `X-Beta-Features: managed-agents, agent-memory, skills` (repeatable); SDK `betas=[...]`. Invalid or inaccessible beta feature → 400.
- **Api-Revision header** (`2026-05-20` style): avoids breaking changes within a major version.
- **Model versioning**: pinned dated snapshots vs rolling `-latest` aliases; lifecycle stages Experimental / Preview / Stable GA.
- **Endpoint deprecation lifecycle**: deprecated endpoints grouped under `/api/endpoint/deprecated/*`; newer equivalents under `/api/endpoint/beta/*`.
- **Monthly changelog**: dated entries with type tag (Feature / Update / Fix), affected models/endpoints, description.
- **Beta feature tags** (representative): `files-api-2025-04-14`, `compact-2026-01-12`, `fast-mode-2026-02-01`, `task-budgets-2026-03-13`, `context-management-2025-06-27`, `server-side-fallback-2026-06-01`, `computer-use-2025-11-24` / `-2025-01-24` / `-2024-10-22`, `advisor-tool-2026-03-01`, `fine-grained-tool-streaming-2025-05-14` (legacy, superseded by `eager_input_streaming`), `code-execution-2025-05-22` (legacy).

## S.3 Deprecation notices & replacement mappings

### Notice-period policy
- GA capabilities: ≥ 6 months notice.
- Specialized variants: ≥ 3 months notice.
- Preview capabilities: may retire with ~ 2 weeks notice (not recommended for production).
- Publicly released models with active deployments: ≥ 60 days notice + migration guide.
- Three-month sunset window before alias removal.
- Deprecation announcements on Release Notes; earliest shutdown dates on Deprecations page; Stable + Preview schedules tracked.

### Deprecated → replacement model mappings (selected, vendor-neutral)
- `gpt-5-2025-08-07` → `gpt-5.5`.
- `gpt-5-mini-2025-08-07` → `gpt-5.4-mini`.
- `o3-2025-04-16` → `gpt-5.5`.
- `dall-e-2` / `dall-e-3` → `gpt-image-2`.
- `claude-sonnet-4-20250514` → `claude-sonnet-4-6`.
- `claude-3-haiku-20240307` → `claude-haiku-4-5-20251001`.
- `gemini-2.5-pro` → `gemini-3.1-pro-preview`.
- `gemini-2.5-flash` → `gemini-3.5-flash`.
- `gemini-embedding-001` → `gemini-embedding-2`.
- Old aliases → `-latest` aliases.
- Native-reasoning models (e.g. `magistral-*`) → `reasoning_effort` on standard models.
- `:online` variant → `openrouter:web_search` server tool.
- `:thinking` variant → `reasoning` parameter.
- `mistral-moderation-2411` → `mistral-moderation-2603` (2026-03-31).
- Opus 4.7 Fast Mode (retired 2026-07-24) → Opus 4.8 Fast Mode.

### Deprecated endpoints / surfaces (selected, vendor-neutral)
- `POST /v1/prompts` (shutdown 2026-11-30) → migrate into app code.
- Assistants API (sunset 2026-08-26) → Responses / Conversations.
- Evals platform (read-only 2026-10-31, shutdown 2026-11-30) → external eval tooling.
- Agent Builder (shutdown 2026-11-30) → Agents SDK / Workspace Agents.
- Self-serve fine-tuning new job creation (shutdown 2027-01-06) → inference continues until base model deprecated.
- Video generation model (shuts down 2026-09-24); Image generation model (shut down 2026-08-17).
- Legacy NLP capabilities (LUIS, QnA Maker, Custom Vision, CLU, CQA, Orchestration, Sentiment, Key Phrase, Entity Linking, Custom Text Classification) retire 2025–2029 (Specific: LUIS retired 2026-03; QnA Maker retired 2025-10; Custom Vision retirement 2025-2028; legacy capabilities retire 2029-03-31).

### SDK versioning
- Major version semantic versioning; V1 → V2 unified-package migration.
- Judge versioning: `base_revision` / `up_revision` / `down_revision`.
- SDK language catalogs vary by surface (Node, Python, Go, Ruby, Java; Python/JS V1/V2; etc.) — capability of L0 Developer SDK module.

## S.4 Errors & conventions

### Streaming error recovery
- ≤ 4.5: partial response treated as start of new assistant message.
- 4.6+: user-message-instruct-continue.
- Tool-use and extended-thinking blocks cannot be partially recovered — resume from the most recent text block.

### Async job `errors[]` array
- Async jobs surface `errors[]` array in `status: failed` payloads.

### Error code enum examples
- `invalid_argument` / `permission_denied` / `failed_precondition` / `service_unavailable` / `internal_error`.
- Error hierarchy & codes (representative): `PROMPT_TOO_LONG`, `CONTENT_POLICY_VIOLATION`, `INDEX_OUT_OF_BOUNDS`, `MISSING_REQUIRED_PARAMETER`.

### HTTP error code taxonomy (unified)
- 400 `invalid_request_error`.
- 401 `authentication_error`.
- 403 `permission_error` (missing scopes).
- 404 `not_found_error`.
- 409 `conflict_error`.
- 413 `request_too_large`.
- 422 Unprocessable Entity.
- 429 `rate_limit_error` / `RESOURCE_EXHAUSTED` / `Too Many Requests`.
- 5xx `api_error`.

### Error shape
- Always JSON.
- `type: "error"`, `error: {type, message}`, `request_id`.
- Quote `request_id` when contacting support.

### Typed error objects (L4/L5 examples)
- `mcp_connection_failed_error`, `mcp_authentication_failed_error`, `environment_archived_error`, `agent_archived_error`, `session_rate_limited_error`, `gateway_timeout`, `ContextWindowExceeded`, `UsageLimitExceeded`, `HttpConnectionFailed`, `BadRequest`, `Unauthorized`, `SandboxError`, `InternalServerError`, `Other`, `too_many_requests`, `invalid_tool_input`, `max_uses_exceeded`, `query_too_long`, `request_too_large`, `unavailable`, `url_not_in_prior_context`, `url_too_long`, `url_not_allowed`, `url_not_accessible`, `unsupported_content_type`.

### Max request sizes (per endpoint family)
- Messages: 32 MB. Token Counting: 32 MB. Batch: 256 MB (variant up to 512 MB / 2 GB). Files: 500 MB – 2 GB.

### Retry guidance
- Exponential backoff with jitter for transient errors (429, 5xx).
- Failed requests still count against per-minute limits.
- Retryable / non-retryable code lists configurable via `retry_policy` (`max_attempts`, `initial_delay_ms`, `max_delay_ms`, `multiplier`, `retryable_codes`, `non_retryable_codes`).
- Hedge: `hedge: {hedge_delay, hedge_budget_pct}`.
- Circuit breaker: `circuit_breaker: {memory_threshold_pct, cooldown_seconds}`.
- Generic exponential backoff convention: 500 ms start × 1.5 cap 60 s, 15 min total.

### Reliability / rate-limit response headers
- `x-ratelimit-*`, `Retry-After`, `x-ratelimit-reset`.
- Per-prediction-attempt header (e.g. `X-BASETEN-MODEL-PREDICTION-ATTEMPTS`): retry-attempt count.
- `X-Scale-Up-Timeout`: hold request during scale-up.

### Performance / metric response headers
- prompt-tokens / cached-prompt-tokens / server-time-to-first-token / sampling-options.
- `perf_metrics_in_response`: streaming body performance metrics.

## S.5 Pagination conventions

- Cursor pagination: `after_id` / `before_id`, `has_more` / `first_id` / `last_id`; page-token `page` / `next_page`.
- Endpoint-family dependent; cursors are opaque (never parse; bound to sort key).
- List endpoints support `limit`, `cursor`, `order`, and family-specific filters.

## S.6 Role field convention

- Prefer plural `roles` / `role_names` over deprecated singular `role` / `role_name`.
- Mid-conversation `system` message placement: follow user/assistant turn, not first, not between `tool_use`/`tool_result`.

## S.7 Stop reasons / finish reasons (union)

- Responses `status: completed` / `incomplete` (`incomplete_details.reason: max_output_tokens` / `content_filter`).
- Chat `finish_reason: stop` / `length` / `content_filter` / `tool_calls`.
- `finishReason: STOP` / `MAX_TOKENS` / `SAFETY` / `RECITATION`.
- `stop_reason: end_turn` / `max_tokens` / `stop_sequence` / `tool_use` / `pause_turn` / `refusal` / `model_context_window_exceeded` / `compaction`.
- Normalized: `tool_calls` / `stop` / `length` / `content_filter` / `error` with raw value in `native_finish_reason`.
- `pause_turn`: server-side loop iteration limit; resend paused content to continue.
- Agent-loop `requires_action` (with `event_ids[]`), `end_turn` (interrupted blocked threads); Result subtypes `success` / `error_max_turns` / `error_max_budget_usd` / `error_during_execution` / `error_max_structured_output_retries`.

## S.8 Usage / token accounting (convention shape)

- `usage.prompt_tokens` / `completion_tokens` / `total_tokens` / `prompt_tokens_details.cached_tokens` / `completion_tokens_details.reasoning_tokens`.
- `usageMetadata.total_tokens` / `total_input_tokens` / `total_output_tokens` / `total_thought_tokens` / `tool_use_input_tokens` / `tool_use_input_tokens_details`.
- `usage.input_tokens` / `output_tokens` / `cache_creation_input_tokens` / `cache_read_input_tokens` / `cache_creation.ephemeral_5m_input_tokens` / `cache_creation.ephemeral_1h_input_tokens` / `output_tokens_details.thinking_tokens` / `server_tool_use` / `service_tier` / `iterations`.
- `usage.input_tokens` / `output_tokens` / `total_tokens` / `input_tokens_details.cached_tokens` / `output_tokens_details.reasoning_tokens` / `cost_in_usd_ticks` / `cost_in_nano_usd` / `num_sources_used` / `server_side_tool_usage_details`.
- `usage.prompt_tokens` / `completion_tokens` / `total_tokens` / `prompt_tokens_details.cached_tokens` / `prompt_tokens_details.cache_write_tokens` / `completion_tokens_details.reasoning_tokens` / `cost` / `is_byok` / `cost_details`; per-document results with `modelVersion`.
- Server-tool usage fields: `usage.server_tool_use.{web_search_requests, web_fetch_requests, code_execution_requests}`.
- Advisor usage: `usage.iterations[]` (`{type:"message"}` executor, `{type:"advisor_message", model}` advisor); top-level `usage` reflects executor only.
- Tool-def token cost: per-model tool-use system prompt + per-tool-def tokens.
- Reasoning tokens counted/billed as output tokens; count against `max_output_tokens` and context window.
- `n` / `best_of` cost rule: total generated tokens = `max_tokens × max(n, best_of)`; cost as function of (tokens × cost-per-token); reduce via model switch / shorter prompts / fine-tuning / caching.

## S.9 Refusal detection

- Chat `choices[0].message.refusal`.
- Responses content item `type === "refusal"`.
- `stop_reason: "refusal"` + `stop_details: {category, explanation, type: "refusal"}`.
- Moderation: `error.code: moderation_blocked`, `moderation_details.categories`.

## S.10 Streaming transport conventions

- SSE for HTTP streaming (`stream: true`); `stream_options.include_usage`.
- Error-before-tokens → JSON error. Error-after-tokens → HTTP 200 + SSE error event + `finish_reason: "error"`.
- SSE comments / keep-alive: `: <label> PROCESSING`; advisor `ping` keepalives (~30 s).
- Stream cancellation: abort connection; supported surfaces stop immediately; unsupported continue + billed.
- Streaming timeouts: reasoning models can take a long time to first token; override default client timeout (e.g. `timeout=3600`); some SDKs require streaming when `max_tokens` exceeds a threshold to avoid HTTP timeouts.
- SDK streaming helpers: typed-event iteration, `.stream()` context manager, `text_stream`, `.get_final_message()`, `stream=True`.
- Resumable streaming: `last_event_id` cursor; unique `event_id` per delta.
- Unknown-event handling: log and skip; new types may be added.

## S.11 Idempotency & naming conventions

- `external_id`: deterministic UUID5 slash-supported paths; idempotent re-upload overwrites.
- Idempotent webhook handlers by `generation_id` (or analogous correlation id).
- `Idempotency-Key` header for trigger APIs.
- Webhook Standard Webhooks spec: `webhook-id`, `webhook-timestamp`, `webhook-signature` (`v1,` HMAC-SHA256 or JWKS JWT); retry/backoff; delivery-not-guaranteed.
- Billing webhook body schema: `X-Signature` (HMAC-SHA256), `X-Request-ID`; body `{type:"API_BILLING_USAGE", data:{events:[{idempotencyKey, timestamp, requestId, modelSlug, externalEntityId, apiKeyPrefix, tokens:{input,output,cachedInput}}]}}`.

## S.12 File format conventions

- Image formats: PNG, JPEG, WebP, GIF, HEIC, AVIF, TIFF.
- Audio formats: MP3, AAC, OGG, OPUS, WAV, AIFF, PCM, μ-law, A-law.
- Video formats: MP4 (+ variants).
- Document formats union: PDF, DOCX, PPTX, ODT, EPUB, RTF, XLSX, CSV, ODS, Numbers, TXT, Markdown, RST, LaTeX, JSON, JSONL, XML, YAML, code files, EML, MSG, PNG, JPEG, WebP, GIF.
- Output formats (image): png / jpeg / webp / svg / transparent. (audio codec list). (video) MP4 + thumbnail/spritesheet variants. (document) md / html / json / chunks / doctags / doclang / docx / pdf / png.

## S.13 Coordinate systems standard

- Bounding box encodings: XYXY, XYWH, polygon (4-point or N-point), yxyx.
- Coordinate spaces: pixel (top-left origin) vs normalized 0-1 vs normalized 0-1000.
- Mask polarity conventions: white = edit / black = keep; alpha-embedded; polygon; COCO RLE.

## S.14 Response delivery & shape conventions

- Delivery modes: synchronous, streaming, asynchronous (poll), webhook delivery.
- SDK polling helpers: `createAndPoll`, `generate()` / `extend()`, `operations.get`.
- Response shape: image URL (ephemeral) vs base64 inline vs raw bytes via `Accept`; `revised_prompt`; `is_image_safe` / `content_violation` / `respect_moderation` flags; `file_output` block; usage / credit headers.
- Video job object fields: `id` / `object` / `created_at` / `status` / `model` / `progress` / `seconds` / `size` / `error`.

## S.15 Confidence / likelihood reporting standard

- Float 0-1 (`confidence` / `score` / `detectionConfidence`).
- Likelihood enum: UNKNOWN(0) → VERY_LIKELY(5).
- Threshold parameters convention.

## S.16 Async job state machine convention

- Canonical states: `queued` / `pending` / `in_progress` / `processing` / `completed` / `done` / `failed` / `expired` / `canceled`.
- Progress 0-100; terminal states; polling cadence ~10-20 s with exponential backoff; webhook-vs-poll choice.

## S.17 Webhook verification & idempotency standards

- Ed25519 signing; JWKS URL; signed-message template.
- `webhook_secret` shared-secret alternative.
- Idempotency keys; delivery-not-guaranteed semantics.

## S.18 Reasoning replay conventions

- Stateless + reasoning preservation rules: encrypted blobs; unmodified `thinking` / `reasoning_details` / `steps`; preservation-by-model rules; `include: ["reasoning.encrypted_content"]`.
- Encrypted / opaque replay: `encrypted_content` / `redacted_thinking` / `encrypted_index` opaque blobs must be passed back verbatim or 400.

## S.19 Strict-mode structured-output contract

- `additionalProperties: false`, all fields required, optional via union-null.
- Shared schema-complexity ceilings: max object properties (up to ~5000), nesting ≤ 10, enum ≤ 1000, strict tools ≤ 20, optional params ≤ 24, union ≤ 16, compilation timeout 180 s.
- JSON Schema keyword support (representative):
  - Common supported: `string`, `number`/`integer`, `boolean`, `object`, `array` (items / prefixItems / minItems / maxItems), `enum`, `anyOf`, `$ref`/`$defs` (recursive support varies; non-circular on some).
  - Gaps: `oneOf` (sometimes behaves like `anyOf`), `allOf` (best-effort single subschema only on some), `not`/`if-then-else`/`dependentRequired`/`dependentSchemas` (best-effort on some).
  - `pattern` regex (ECMAScript subset; semantic differences may apply for `.`, `^`-`$`, capturing groups).
  - `format` strings: date-time / date / time / email / uuid / ipv4 / ipv6 / uri.
  - Numeric/array/string constraints: `minimum`/`maximum`, `minLength`/`maxLength` (up to ~2048), `minItems`/`maxItems` (up to ~256), `additionalProperties` rules.

## S.20 JSONL batch file format convention

- `{custom_id, params/body}` line schema.
- Result correlation by `custom_id` (may not preserve input order).
- `custom_id` constraints: unique; 1-64 chars `^[a-zA-Z0-9_-]{1,64}$` (variant); or `metadata.key`.
- Max requests per batch (variable bound, up to ~100k). Max file size (variable bound, up to ~2 GB). Completion window (~24 h or queue-based). Results retention (~24 h download to 6 weeks).

## S.21 Tool concept names, content blocks, and event catalogs

### Tool concept names (canonical)
- Function tool definition shapes: modern flat `{type:"function", name, description, parameters, strict}`; `{type:"function", name, description, input_schema, strict}`; legacy externally tagged `{type:"function", function:{name, description, parameters, strict}}`; custom tools (free-form text I/O, no JSON Schema).
- Tool namespaces: `{type:"namespace", name, description, tools:[...]}`.
- `tool_choice` values: `auto` / `required` / `any` / `none` / specific function (`{type:"function", name}` / `{type:"tool", name}` / `{allowed_tools:{mode:"any", tools:[]}}`) / subset (`{type:"allowed_tools", mode:"auto", tools:[]}`) / `validated` / graduated eagerness (minimal/low/medium/high/xhigh) / force hosted tool (`{type:"image_generation"}`) / `max_tool_calls`.

### Content block types (union)
- Assistant / user / server-tool-result block types.
- Content block types: `text`, `image` (base64 + mimeType), `audio`, `resource` (URI + inline), `resource_link`, `structuredContent`, `isError`.

### Tool call block fields
- `type`, `id`, `name`, `input`/`arguments`, `caller?`, `confirmation_status?`, `safety_decision?`, `signature?`.

### Tool result block fields
- Canonical: `{content:[...], structuredContent?, isError?}`.
- Server tool result block types: `web_search_tool_result`, `web_fetch_tool_result`, `bash_code_execution_tool_result`, `code_execution_tool_result`, `advisor_tool_result`, `tool_search_tool_result`, `text_editor_code_execution_*_result`.

### Server tool error codes (representative)
- Web search / web fetch / code execution / advisor / tool search error code lists (per-tool).

### Versioned tool type strings
- Dated type strings for: `web_search`, `web_fetch`, `code_execution`, `computer`, `bash`, `text_editor`, `memory`, `advisor`, `tool_search`.

### Event type catalog (union)
- **User events**: `user.message`, `user.interrupt`, `user.custom_tool_result`, `user.tool_confirmation`, `user.define_outcome`, `user.tool_result`, `system.message`.
- **Agent events**: `agent.message`, `agent.thinking`, `agent.tool_use`, `agent.tool_result`, `agent.mcp_tool_use`, `agent.mcp_tool_result`, `agent.custom_tool_use`, `agent.thread_context_compacted`, `agent.handoff`, `message.output`, `tool.execution`, `function.call`.
- **Session events**: `session.status_running/idle/rescheduled/terminated/deleted/updated/error`, `turn/started/completed/diff/updated/plan/updated`, `thread/tokenUsage/updated`, `interaction.created/status_update/completed`.
- **Span / observability events**: `span.model_request_start/end`, `span.outcome_evaluation_*`, `hook/started/completed`, `model/rerouted`, `model/safetyBuffering/updated`, `model/verification`.
- **Item lifecycle**: `item/started`, `item/completed`, deltas.

### Delta type catalog
- `event_start` / `event_delta`, `content_delta`, `text` / `image` / `audio` / `thought_summary` / `thought_signature` / `arguments_delta` / `google_search_call` / `google_search_result`, `item/agentMessage/delta`, `item/plan/delta`, `item/reasoning/summaryTextDelta`/`textDelta`, `item/commandExecution/outputDelta`, `TextChunk` / `ToolReferenceChunk` / `ToolFileChunk` / `ReferenceChunk`.

### Symmetric step model
- `step.start` → `step.delta`(s) → `step.stop` (and equivalents).

### Tool result ordering & linkage rules
- `tool_result` immediately follows `tool_use`; all `tool_result`s first in user message then text; pending-programmatic-call message contains only `tool_result` blocks; missing `call_id` → 400.

### Mixed server + client turn rules
- `stop_reason:"tool_use"`; content has `server_tool_use` + client `tool_use`; user message of only `tool_result`s; keep same `tools`; pass `container` id.

### Tool annotations (behavioral hints)
- `readOnlyHint`, `destructiveHint`, `idempotentHint`, `openWorldHint`, `maxResultSizeChars`; destructive always triggers approval.

### AGENTS.md layered discovery
- Global → project → cwd; `AGENTS.override.md` precedence; ~32 KiB cap; fallback filenames; `project_doc_max_bytes`.

## S.22 Citation attribution contract

- Citations must be visible/clickable to end users (contractual).
- `cited_text` / `title` / `url` excluded from output-token billing.
- Annotation shapes: `url_citation` (start_index/end_index); `search_suggestions` widget HTML (must render); `sources` list incl. real-time feeds (sports/weather/finance); `tool_reference` chunks; `file_citation` (`file_id`/`file_name`/`filename`/`page_number`/`media_id`/`custom_metadata`); `container_file_citation` adds `container_id`; `place_citation` (`name`, `url`) — render as links; attribution + legal notices required.

## S.23 Tool pricing models (reference)

- Web search ($/1k searches; failed not billed; per-query vs per-prompt).
- Code execution (free w/ web search/fetch on newer versions, else execution-time billing: min 5 min/invocation, free hours/org/month, $/hour/container beyond; files in request billed even if tool not invoked; some platforms no extra charge — tokens only).
- Vector stores (1 GB free then $/GB/day; use `expires_after`; or free at query time, pay for embeddings + tokens).
- Computer use (token pricing at Vision rates + tool-def tokens).
- MCP / Connectors (tokens only, no per-call fee).

## S.24 Tool-specific rate limits (reference)

- Vector store files / file_batches: 300 RPM/store.
- MCP: 200 / 1000 / 2000 RPM by tier.
- File search: 100 / 500 / 1000 RPM by tier.
- Code interpreter: 100 RPM/org.

## S.25 Integration paths (convention)

- SDK (ecosystem frameworks / enterprise gateways) vs Direct REST/gRPC (edge platforms / aggregators) vs OpenAI-compat layer (text-only aggregators, feature ceiling — no native video, caching, etc.).
- Developer API vs Enterprise Agent Platform through one SDK (switch via config flags); model IDs identical; auth migrates API key → service accounts; AI Studio models must be retrained in Enterprise.

## S.26 Exhaustive processing pipeline stages (canonical)

- Stages 0–11: canonical end-to-end request pipeline (ingestion → preprocessing → model invocation → tool execution → output shaping → moderation → logging → billing → response delivery).

## S.27 Context management invalidation rules (canonical)

- Tool-def change → invalidates whole cache.
- Toggle web search / citations → invalidates system + messages.
- Change `tool_choice` / `disable_parallel` / images / thinking params → invalidates messages.
- `cache_control` + `defer_loading` mutually exclusive.
- Cache hierarchy: tools → system → messages; longer-TTL must appear before shorter.
- Automatic server-tool-result caching: 5-min TTL tracked in `cache_creation.ephemeral_5m_input_tokens`.
- Discovered tools appended at end of window to preserve cache.

## S.28 Open- vs fixed-vocabulary detection (note)

- Detection may be open-vocabulary (prompt), fixed vocabulary, or custom-trained.

## S.29 Prompt lifecycle conventions

- Prompt enhancement: `revised_prompt`, `prompt_upsampling` / `disable_pup`, `magic_prompt` / `/v1/images/magic-prompt`, `/v1/prompts/enhance`, auto-enhanced, `thinking_level`.
- Plain-text prompt length limits vary by model.
- Reference-image tagging: `<img>N</img>`, `<IMAGE_N>`, "person of image 1 wearing garments of image 2", character name verbatim.

## S.30 Realtime session & event references

- **Unified realtime session configuration object**: model, voice, instructions, modalities, audio formats, turn_detection, tools, thinking.
- **Unified realtime event reference**: client→server and server→client event union (see L3 Voice / L4 Voice for full per-event detail).
- Pronunciation dictionary locator convention: `pronunciation_dictionary_locators` / `pronunciation_dict_id` / `pronunciation_id`.

## S.31 Surcharge conventions

- Surcharges may apply per optional capability (e.g. entity detection +30%, keyterm +20%, speaker roles +10%, EU residency premium, bbox extraction surcharge).

## S.32 Unified data structures (canonical schemas)

The following canonical schemas are used across image/video services (defined once here, referenced from L3):

- **Image Input**: URL / base64 / file_id / multipart / hosted blob / project ref / cloud storage.
- **Mask**: alpha / B/W (black=edit / white=keep) / grayscale / polygon / COCO RLE.
- **Bounding Box**: XYXY / XYWH / polygon / yxyx; pixel or normalized (0-1 or 0-1000).
- **Style Specification**: curated names / presets / codes / reference images / color palette / structured description.
- **Controls**: colors, background_color, artistic_level, no_text, text_layout.
- **Layout** (region-based style): regions with `label`/`prompt`/`bbox` (normalized 0-1)/`image_index`/`parent`/`region_type` (coarse/medium/fine/text/hand/face); width/height multiples of 32.
- **LayoutCommand**: `op: add|shift|remove|place|keep|change`; `at`/`to` as `Bbox`/`Point`; `image_index`; `new_description`; `label`/`description`.
- **V4JsonPrompt** (structured-description style): `high_level_description` + `style_description` (aesthetics/art_style/lighting/medium/photo) + `compositional_deconstruction` (background + ordered elements with type obj/text, desc/text, optional bbox 0-1000).
- **Image Generation Request**: prompt / json_prompt / model / size / aspect_ratio / quality / n / output_format / background / seed / negative_prompt / style surface / controls / guidance / steps / moderation / safety_tolerance / character_reference_images / postprocessing / partial_images.
- **Image Editing Request**: prompt / image(s) / mask / strength / image_weight / edit operation variants.
- **Vision Request**: image(s) / prompt / detail / features enum.
- **Video Generation Request**: prompt / model / size / aspect_ratio / seconds / resolution / seed / personGeneration / input_reference / lastFrame / referenceImages / characters / storage_options.
- **Video Edit / Extension Request**: source video / prompt / seconds (extension portion).
- **Vision Analysis Response**: detected objects / labels / faces / OCR / segmentation masks / confidence.
- **Image Generation Response**: image URL / base64 / file_output / revised_prompt / is_image_safe / moderation_details / usage.
- **Async Response**: `id` / `object` / `created_at` / `status` / `progress` / `error`.
- **Video Response**: MP4 URL / thumbnail / spritesheet / persisted file / usage.

## S.33 Design choices recap (cross-layer notes)

- **Separation of conversation state from sandbox/filesystem state**: two independent state dimensions (conversation history via `previous_interaction_id`; sandbox/files via `environment_id`).
- **Two paradigms for text**: modern stateful Responses/Interactions/Messages vs legacy Chat Completions.
- **Understanding vs generation split for images**: generative multimodal vision (chat-based) vs classical fixed-feature detection (batch annotation).
- **Server-managed vs in-process agent loop**: server-side managed loop vs SDK-driven in-process loop.
- **Bring-your-own OTLP backend vs hosted dashboard**: observability backend choice.
- **Sync vs streaming vs async vs batch**: cross-layer execution-mode convention.
- **Stateful server-side vs stateless replay**: conversation-state model.
- **Sandboxes appear in three places**: L1 (GPU-platform code-exec envs), L0 (sandbox provisioning infra), L4 (agent execution environments / code-execution tool surface).

---
# LAYER 0 — FUNDAMENTAL TECHNICAL SERVICES

> **Purpose.** The fundamental technical services that every higher layer builds on: identity, tenancy, RBAC, network, storage, files, databases, environments/sandboxes, workflows/scheduled jobs, billing, quotas, processing tiers, compliance/privacy, data residency, webhooks, SDKs, caching/token-counting, routing/gateway control plane. L1 builds inference serving on top of these; L4 builds agentic workflows and sandboxes on top of these primitives; L5 builds observability and administration on top of these services.

## Domain L0.A — Identity, Authentication & Keys

### Module: API Keys
- **Create API Key** — args: `name`, `scopes[]`, `expires_at?`, `workspace_id?` — `POST /v1/api_keys` / `POST /v1/organizations/api_keys` — long-lived static key CRUD.
- **List API Keys** — args: (prefix+name only) — `GET /v1/api_keys` / `GET /v1/organizations/api_keys`.
- **Revoke API Key** — args: `id_or_prefix` — `DELETE /v1/api_keys/{id_or_prefix}` / `DELETE /v1/organizations/api_keys/{id}`.
- **Register Externally-Owned Key** — args: Ed25519-signed key material — `POST /v1/api_keys/register` — register an externally-owned signed key.
- **Create Service Account** — args: `name`, `workspace_id?`, `scopes[]` — `POST /v1/organizations/service_accounts`.

### Module: Federation & SSO
- **Create Federation Issuer** — args: `issuer_url`, `audience`, `rules[]` — `POST/GET /v1/organizations/federation_issuers` — Workload Identity Federation issuer config.
- **Token Exchange / STS** — args: `grant_type=urn:ietf:params:oauth:grant-type:jwt-bearer`, `assertion`, `scope` — OAuth 2.0 jwt-bearer grant; exchange OIDC JWT for short-lived bearer token.
- **Domain Verification** — args: `domain` (DNS TXT) — verify domain before SSO.
- **SSO (SAML)** — args: `idp_metadata`, `org_id` — org-level single sign-on.
- **SCIM Provisioning** — args: `group_id`, `user_id`, `operation` — sync group membership from IdP.
- **Access Transparency Query** — args: `time_range`, `filters[]` — `GET /v1/compliance/access_events` — log of every access to your data by staff/systems.

### Module: Key Management & Scoping (capabilities of API Key family)
- Key types: static API key (header bearer); Admin API key; Service account; Workspace/personal key types (`PERSONAL` / `WORKSPACE_MANAGE_ALL` / `WORKSPACE_INVOKE` / `WORKSPACE_EXPORT_METRICS`).
- Fine-grained scoped tokens (`hf_...` style); per-repo / per-task permissions.
- Federated / resellable API keys (minted under a group, branded URL).
- Key expiration (`expires_at`); key immutability after creation.
- Key restriction by request origin (IP / referrer / application).
- Per-key scope to specific model IDs.
- Per-endpoint key permissions (`None` / `Restricted` / `Read-Write` / `Read-Only`).
- Hierarchical groups with inherited rate/usage limits (`INDEPENDENT` vs `CASCADING` enforcement).
- Organizations + Resource Groups (Enterprise); Cost centers (team/project/department).
- SSO + SCIM (Enterprise identity federation).
- Safety-identifier hashing (`safety_identifier`, `OpenAI-Safety-Identifier`).
- Workload Identity Federation (WIF); OAuth 2.0 / user credentials.
- Ephemeral tokens: Single Use Token; Ephemeral Client Secret (`ek_...`); Ephemeral Token / Access Token JWT / Temporary API Token JWT / Browser Token.
- Token configuration dimensions: `uses`, `expire_time` (max 20 h), `new_session_expire_time`, `field_mask`, `live_connect_constraints`, `safety_identifier`.
- Customer-Managed Encryption Keys (CMEK / EKM) — capability of broader Key Management.

## Domain L0.B — Tenancy, Workspaces & Projects

### Module: Organizations & Tenancy
- **Create Organization** — args: `name`, `parent_id?`, `description?` — `POST /v1/organizations` — top-level billing & identity container.
- **List/Get/Patch/Delete Organization** — `GET/POST/PATCH/DELETE /v1/organizations[/{id}]`.
- **Enterprise Account / Org Grouping** — args: `organization_id`, `linked_organization_ids[]` — grouping above organization for stronger isolation.
- **Create Org Invite** — args: `email`, `role` — `POST /v1/organization/invites`.

### Module: Workspaces
- **Create Workspace** — args: `name`, `organization_id`, `default_inference_geo?`, `allowed_inference_geos?` — `POST /v1/organizations/workspaces` — sub-tenant boundary scoping keys/files/limits/spend/residency.
- **List/Get/Patch/Archive/Delete Workspace** — `GET/PATCH /v1/organizations/workspaces[/{id}]`, archive/delete.
- **Workspace Members** — args: `workspace_id`, `user_ids[]`, `role` — `POST /api/admin/workspaces/{id}/add-users`, `PATCH .../users`, `DELETE .../users`; invites; Cloud IAM bindings.
- **Org / Project / Group CRUD** — args: org, project — `POST /v1/organizations/{org}/projects`, `POST /v1/gateway/groups` (hierarchical billing entity with inherited limits), `GET /v1/gateway/groups/{id}/usage`.
- **User CRUD** — args: `email`, `name`, `role` — `GET/POST/PATCH/DELETE /api/admin/users[/{id}]`, `POST .../users/invite`.
- **Tenant Management** — args: workspace_id, tenant — `POST /v1/workspaces/{id}/tenants` and lifecycle endpoints; tenant lifecycle ACTIVE→INACTIVE→OFFLOADED.

### Module: Workspace Capabilities
- Default Workspace (non-archivable).
- Workspace limits (max active workspaces/org; name uniqueness within org).
- Workspace `access_mode` private | public; budget controls; per-operation rate limiting.
- Workspace-scoped resources (Files, Batches, Skills, prompt caches, API keys).
- Workspace isolation (`shared` / `personal`).
- Org + workspace spending limits; project spend caps; billing-account tier spend caps.
- Cost attribution header `X-HF-Bill-To` / `bill_to` (org).

## Domain L0.C — Roles, RBAC & Groups

### Module: Roles & Permissions
- **List Roles** — `GET /api/admin/roles` — list/assign roles at org + workspace scope.
- **List User Groups** — args: `name`, `member_ids[]`, `idp_group_id?` — `GET /api/admin/user-groups` — collections assignable to roles, SCIM-syncable.
- **RBAC Permission Evaluation** — capability: union of direct + group roles, propagation delay ≤ 30 min.

### Module: RBAC Capabilities
- Preset vs custom roles.
- Permission catalogs: model/inference, files/vector stores, fine-tuning, admin/org, compliance (`read:compliance_*` / `delete:compliance_user_data`), tunnels (Read/Use/Manage), observability viewer.
- Workspace roles (User/Limited Developer/Developer/Admin/Billing).
- Key permissions intersect with user's role.
- Permission policy knob (`always_allow` / `always_ask`): server-executed tool permission gate (bridge to L5 approvals).
- `requirements.toml` admin restrictions: disallow `approval_policy = "never"`, constrain sandbox modes.

## Domain L0.D — Network & Private Connectivity

### Module: Private Connectivity
- **Private Link / PSC Healthcheck** — args: `region` — `GET /v2/privatelink_healthcheck`, `https://<region>.privatelink.../v1/*` — regional private endpoint health & API surface.
- **IP Egress Manifest** — published outbound IP ranges for allowlisting.
- **Tunnel Control Plane** — args: `tunnel_id`, `mcp_command`|`mcp_server_url`, `base_url`, `CONTROL_PLANE_API_KEY` — `HTTPS https://api...:443/v1/tunnel/*`, `mtls.api...:443` — outbound-only HTTPS tunnel to platform-hosted MCP endpoint.
- **Tunnel-Client Invocation** — CLI args: `--private-connection-resource-id`, `--tunnel-id`, `--mcp-command`|`--mcp-server-url`, env `CONTROL_PLANE_API_KEY`.
- **Network API** — `GET` network config.

### Module: Network & Data Security Capabilities
- TLS/SSL in transit.
- PrivateLink / VPC (private IP intra-region) — config params: AWS account id, VPC service name, sharing.
- `access_level: private | public | authenticated` — endpoint visibility control.
- Global private networking (Pod-to-Pod).
- Non-root containers.
- Model security (private repos, malware/pickle scans).
- Data privacy (no payload storage; logs 30 days).
- Data residency (fixed region, immutable).
- Container isolation (Secure Cloud T3/T4).
- Project-nested observability permissions.
- Environment networking controls: `unrestricted` default vs `limited` with `allowed_hosts[]`.
- mTLS mutual TLS (platform as MCP client or tunnel control plane).
- Secure MCP Tunnels (outbound-only).
- Tunnel RBAC permissions (Read/Use/Manage).
- Tunnel audit events (`tunnel.created/updated/deleted`).

### Module: Outbound Network Policies
- **Create Outbound Policy** — args: network domain rules, `allow_mcp_servers`, `allow_package_managers` — `POST /v1/outbound-policies` + `/check`.
- **Egress Proxy Header Transform** — request header transform via network `transform` (egress proxy).

## Domain L0.E — Files & Object Storage

### Module: Files API
- **Upload File** — args: `file`, `purpose` (`user_data` / `vision` / `batch` / `assistants`), `filename`, `metadata` — `POST /v1/files` — multipart / URL / base64; resumable upload → `uri` + `name` + `state`; processing state poll (PROCESSING → ACTIVE).
- **Get File** — args: `id` — `GET /v1/files/{id}` — metadata + status.
- **Download File Content** — args: `id` — `GET /v1/files/{id}/content`.
- **Delete File** — args: `id` — `DELETE /v1/files/{id}`.
- **Resumable Upload Sessions** — `/v1/files/uploads` + `/complete` + `/abort`.
- **Prechunked Upload** — `/v1/files/prechunked` (MXJSON).
- **List Files** — args: filters — `GET /v1/files` (filter).
- **Patch File Metadata** — args: `id`, metadata — `PATCH /v1/files/{id}`.
- **Bulk Delete Files** — `POST /v1/files/bulk-delete`.
- **Download File Chunks** — `GET /v1/files/{id}/chunks`.
- **Download File (original/rendered_pdf)** — `GET /v1/files/{id}/download`.
- **Download Tool-Generated File** — args: `file_id` — `client.files.download(file_id)`.

### Module: Files Capabilities
- Purpose variants (`user_data` / `vision` / `batch` / `assistants`).
- Resumable upload protocol.
- Processing state polling (PROCESSING → ACTIVE).
- Raw-file retention/expiration (e.g. raw 48 h / indefinite embeddings).
- File_id-based retrieval.
- File Object structure.
- Idempotent ingestion (`external_id` deterministic UUID5; re-upload overwrites).
- File auto-delete (`expires_after` / `expires_at`).
- Background store (~10 min for polling, ZDR-incompatible).
- External web access (offline/cache-only web search, BAA-eligible under ZDR).
- Storage limits (variable: 20 GB/project & 2 GB/file; 512 MB max file; 32 MB Messages / 256 MB Batch; 300 RPM vector store ingestion).
- Batch result-file retention (30 days / 24 h download / 6 weeks).
- Bidirectional Files API (input + output).
- Output persistence (`storage_options`, `file_output`, public URL).
- Ephemeral URL expiry rules (e.g. 1,000 active URLs/team).
- Input file methods: URL / base64 / file_id / multipart / hosted blob / project ref (`id:<uuid>` / `reference:@<name>`) / cloud storage loaders / local FS batch / in-memory stream / pre-chunked MXJSON / Docling JSON round-trip / `datalab://` URI / object insertion.
- `input_image_blob_path`.

## Domain L0.F — Storage & Databases (vector / graph)

### Module: Vector Store Management (file ingestion + indexing)
- **Create Vector Store** — args: `name`, `chunking_strategy`, `file_ids[]`, `metadata` — `POST /v1/vector_stores`.
- **List/Get/Delete Vector Store** — `GET/DELETE /v1/vector_stores[/{id}]`.
- **Attach File to Vector Store** — args: `vector_store_id`, `file_id` — `POST /v1/vector_stores/{id}/files`.
- **List Vector Store Files** — `GET /v1/vector_stores/{id}/files`.
- **Retrieve from Vector Store** — args: `query`, `rewrite_query`, `max_num_results` (≤50), `attribute_filter`, `ranking_options` (`ranker`, `score_threshold`, `hybrid_search` weights) — `POST /v1/vector_stores/{id}/search`.
- **Documents API** — `POST /v1/agents/documents` (up to 100 bulk), metadata filters, top_k.
- **Media Download** — `download_media(media_id)`; persistent media IDs.
- **Batch File Operations** — file_batches CRUD.

### Module: Vector Database Engine (capabilities)
- Self-hosted vector database: HNSW / Flat / Dynamic / HFresh; LSM-Tree; WAL + snapshots; lazy shards; async indexing.
- Vespa vector store (swappable schema).
- LanceDB local index (FTS / embedding / hybrid; no server).
- Native graph database (Neo4j; index-free adjacency; ACID; Bolt).
- File-based storage (CSV / JSON / Cypher / HTML).
- DPE database (persist until deleted).
- S3 integration; BYOB (Bring Your Own Bucket).
- Vector index types & quantization (PQ / BQ / SQ / RQ; re-scoring).
- Inverted index (roaring bitmaps; map index for BM25); per-property index flags.
- Consistency levels ALL / QUORUM / ONE.
- Replication + backups.
- Multi-tenancy (per-tenant sharding 50,000+ shards/node; tenant lifecycle).
- Lifecycle/cost control (`expires_after` / `expires_at`; TTL; result retention; raw file expiration).

## Domain L0.G — Environments & Sandboxes (Provisioning)

### Module: Environment CRUD
- **Create Environment** — args: `name`, `type` (managed cloud / self-hosted/local / git worktree / container cache), `sources`, `mounts`, `network_policy`, `credentials`, `resource_limits`, `setup_scripts`, `permission_profiles` — `POST /v1/environments`.
- **List/Get/Update Environment** — `GET /v1/environments[/{id}]`, `POST /v1/environments/{id}`, `POST /v1/environments/{id}/archive`, `DELETE /v1/environments/{id}`.

### Module: Sandbox Service (GPU-platform code-exec)
- **Create Sandbox** — args: Git-like branching, OCI images — `POST /v1/sandboxes`.
- **Run Code in Sandbox** — args: code, async operation — `POST /v1/sandboxes/{id}/executions`.
- **Branch Sandbox** — args: checkpoint id — `POST /v1/sandboxes/{id}/checkpoints/{cid}/branch` — fork.
- **Rollback Sandbox** — args: checkpoint id — `POST /v1/sandboxes/{id}/checkpoints/{cid}/rollback`.
- **Poll Sandbox Operations** — `GET /v1/sandboxes/{id}/operations`.

### Module: Environment & Sandbox Capabilities
- Sandbox types: managed cloud sandbox; self-hosted / local sandbox; git worktree isolation; declarative sources / mounts; container cache.
- Network policy (allowlist + `domain_secrets` placeholder injection); `unrestricted` vs `limited` with `allowed_hosts[]`.
- Credentials injection (vault references).
- Lifecycle states.
- Resource limits.
- Setup scripts.
- Filesystem download / snapshot.
- Permission profiles.
- Background terminals / process spawn.
- IDE harness reuse.
- Sandbox test command (validation command).
- SWE-agent preloaded environments.
- Contree SDK / CLI / MCP.

## Domain L0.H — Workflows & Scheduled Jobs (Cron)

### Module: Cron / Scheduled Task Service
- **Create Cron Deployment** — args: agent_ref, schedule (RFC 5545 RRULE or cadence `once`/`daily`/`weekly`/`monthly`/`yearly`), `timezone` (DST-aware), `max_surge`, `max_unavailable` — `POST /v1/deployments` — DST-aware; ≤10 s jitter; ≤1000 deployments/org.
- **Pause / Unpause / Archive / Run Deployment** — `POST /v1/deployments/{id}/{pause|unpause|archive|run}`.
- **List Deployment Runs** — `GET /v1/deployments/{id}/runs` — deployment run records.
- **Trigger Workspace Agent** — args: `Idempotency-Key` — `POST /v1/workspace_agents/trigger` — trigger API.
- **Scheduled Task CRUD** — args: cadence, unattended approval gating — `POST /v1/scheduled_tasks`.

### Module: Workflow / Pipeline Engine
- **Create Pipeline** — args: declarative YAML/Python config (draft → saved → published; immutable versioned snapshots) — `POST /v1/pipelines`.
- **Run Pipeline** — `POST /v1/pipelines/{id}/run`.
- **List Pipeline Executions** — `GET /v1/pipelines/{id}/executions`.
- **Optimize Pipeline (MOAR)** — `POST /v1/pipelines/{id}/optimize` — offline MCTS optimization; Pareto frontier.
- **Create Workflow** — `POST /v1/workflows` — Temporal-based workflows (long-running, fault-tolerant).
- **Declarative Map-Reduce Framework** — lazy immutable Frame; chained ops; terminal actions; Python ↔ YAML.
- **Pipeline Step Types Catalog** — convert / segment / extract / custom / fill + map / filter / reduce / resolve / cluster / rank / split / gather / unnest / code_map / code_reduce / code_filter / parallel_map / equijoin / link_resolve / kg_build.

### Module: Workflow / Cron Capabilities
- Failure behavior (retry / dead-letter).
- Unattended run approvals (`approval_policy = "never"` when org allows; else fall back to permission mode).
- Orchestration features: checkpointing; progress callbacks; parallelization; caching; retries/timeouts; cost & token tracking.
- Per-step intermediate results; per-step billing; eval integration.
- Automatic managed pipeline (no manual orchestration).
- MCP server exposure (`WS /v1/mcp` exposes operations as agent tools).

## Domain L0.I — Billing, Usage & Spend

### Module: Usage & Cost Reports
- **Usage Report** — args: `starting_at`, `ending_at`, `group_by[]` — `GET /v1/organizations/usage_report`, `GET /api/admin/usage` — usage totals per dimension.
- **Cost Report** — args: `starting_at`, `ending_at`, `group_by[]` — `GET /v1/organizations/cost_report` — cost report (excludes Priority Tier when distinct billing model).
- **Surface Analytics** — args: `time_range` — `GET /v1/organizations/{surface}/analytics` — per-product-surface analytics.
- **Generation Stats** — args: `id` (also in `X-Generation-Id` response header) — `GET /api/v1/generation?id={id}` — async token counts + cost lookup.

### Module: Spend Limits & Alerts
- **Create Spend Alert** — args: `threshold_amount` (cents), `currency`, `interval`, `notification_channel:{type:email, recipients, subject_prefix}` — `POST /v1/organization/projects/{id}/spend_alerts`.
- **Read Effective Spend Limits** — args: `scope`, `period` — `GET /v1/organizations/spend_limits/effective`.
- **Approve/Deny Spend Limit Increase** — `POST .../spend_limit_increase_requests/{id}/approve|deny`.
- **Spend Limit (admin)** — `POST/GET /api/admin/spend-limit`.
- **Billing History** — `GET /v1/billing/endpoints`, `GET /v1/billing/pods`, `GET /v1/gateway/groups/{id}/usage`.

### Module: Subscriptions & Plans
- **Manage Subscription / Billing Plan** — args: `plan_id`, `seats?`, `payment_method` — product-surface plans (prepay vs postpay, credits).
- **Priority Tier** — args: `commitment` — committed capacity tier with burndown rates.

### Module: Billing Models (capabilities)
- Per token (input/output/cached); per GPU-second; per minute (dedicated); per PTU; per-GPU hourly / reservation; per-replica resources; per successful response (batch 50% off); per megapixel (image) / per second of video / per second-char of audio; per compute time × hardware price/sec; per transaction / per document (custom features per text record).
- Cached input discount (automatic, 50% default).
- Spending tiers → higher rate-limit upper bounds.
- Prepaid credits / automatic recharge; savings plans (3/6 month upfront).
- Monetary values as strings in minor units (cents) to avoid FP errors.
- Report query params (`starting_at`/`ending_at` RFC3339, `group_by[]`, `period_to_date_spend`, `suppress_notification`).
- `retention_type` enum (`organization_default` / `zero_data_retention` / `modified_abuse_monitoring` / `none`) as request-level data-retention override tied to billing.
- Billing webhooks (per-request token counts, `externalEntityId`, idempotency key, HMAC-signed); billing webhook retries (exponential 1 s→5 s, 15 s max, dead-letter queue).
- Per-key cost analytics (`api_key_id` per-key tracking).

## Domain L0.J — Quotas, Rate Limits & Usage Tiers

### Module: Rate Limit Service
- **Read Rate Limits** — args: `scope`, `model?` — `GET /v1/fine_tuning/model_limits`, `GET /v1/organizations/rate_limits`, `GET /v1/organizations/workspaces/{id}/rate_limits`, `GET /api/admin/rate-limit`.

### Module: Rate Limit Capabilities
- **Dimensions**: RPM, RPD, TPM (input), OTPM (output), TPD, IPM (images/min), audio minutes/min; first metric reached = limit hit.
- **Scope**: per org / per project / per workspace / per model; org+project; workspace overrides inherit org value when absent.
- **Usage tiers**: auto-graduated by cumulative paid spend (Free → Tier 1-5; Start/Build/Scale; Free → Tier 1-3; Free/Scale).
- **Long-context limits**; shared TPM pools across model families.
- **Batch queue limits** (total input tokens queued per model; max batch requests in queue).
- **Vector store ingestion limit** (300 RPM/store).
- **Spend-based rolling 10-min window cap**; acceleration limits (spike rejection).
- **Priority tier rate-limit factor** (e.g. 0.3×); ramp downgrade (≥1 M TPM & >50% TPM increase in 15 min); drawn against both Priority and regular limits.
- **Cache-aware ITPM** (cache-read counted differently).
- **Per-surface concurrency limits** (images/min, requests/sec per user, in-flight counts).
- **Rate-limit response headers** (see S.4): `x-ratelimit-limit-requests`, `x-ratelimit-remaining-requests`, `x-ratelimit-reset-requests`, `x-ratelimit-limit-tokens`/`-project-tokens`, `anthropic-ratelimit-requests-limit/remaining/reset`, `anthropic-ratelimit-input-tokens-*`/`-output-tokens-*`, `anthropic-ratelimit-priority-input-tokens-*`, `retry-after`, `X-RateLimit-Remaining`, `x-gemini-service-tier` (priority/standard).
- **Static account-tier RPM/TPM** vs **Adaptive/dynamic** (15-min buckets, ×1.2 scale-up / ÷1.5 scale-down / 20× ceiling).
- **Per-group per-model with inheritance** (`INDEPENDENT` vs `CASCADING`).
- **Service tiers** (`auto`/`default`/`over-limit`/`flex`/`no-limit`, `priority`).
- **Token-based vs request-based** (TOKEN/REQUEST, SECOND/MINUTE/DAY).
- **Daily usage windows** (reset midnight UTC).

## Domain L0.K — Processing Tiers

### Module: Service Tier Selection
- **Service Tier Selector** — args: `service_tier` enum (`flex`/`priority`/`auto`/`default`/`standard`/`standard_only`) — request-level opt-in to a processing tier; response echoes actual tier.
- **Timeout (SDK)** — Flex ~900 s — client-side knob.

### Module: Processing Tier Capabilities
- Standard (1.0× price, high reliability, seconds-to-minutes latency).
- Batch (~0.5× price, async ≤24 h, separate rate-limit pool, 50% discount).
- Flex (~0.5×, best-effort sheddable, minutes latency, may be preempted).
- Priority (~1.8× / uplift, guaranteed throughput, non-sheddable, consistent latency).
- Ramp rate limit (Priority may downgrade to Standard on traffic ramp).
- Regional processing uplift (10% uplift for residency-eligible models released on/after 2026-03-05; 1.1× US-only inference on certain models).
- Tier eligibility (Flex/Priority on subset of models; Priority short-context only).

## Domain L0.L — Caching & Token Counting (governance / billing facet)

> The technical cache mechanism stays in L2.G; this L0 module records the **admin/governance** facet of caching and token counting.

### Module: Token Counting Service
- **Messages Count Tokens** — args: same as Messages; `context_management` — `POST /v1/messages/count_tokens` — returns `input_tokens` (post-edit) + `original_input_tokens`; free, RPM-limited.
- **Responses Input Tokens** — args: `model`, `input`, `tools?` — `POST /v1/responses/input_tokens` — model-exact token count handling images/files/tools.
- **Count Tokens (legacy)** — `GenerativeModel.count_tokens` — unbilled token count.

### Module: Cache Diagnostics (capabilities)
- Cache retention policies (`prompt_cache_retention`: `in_memory` / `24h`; `cache_control` `{type:ephemeral}` + `ttl:"5m"|"1h"`).
- Per-request fingerprint comparison / `cache_miss_reason` (beta, ZDR-eligible).
- Tokenizer tool (tiktoken); newer tokenizers yield ~30% more tokens than earlier models (cost planning).

## Domain L0.M — Compliance, Privacy, Data Retention & Legal

### Module: Compliance Activities
- **List Compliance Activities** — args: `after_id`, `order_by`, time-filter — `GET /v1/compliance/activities` — list compliance activities; supports eDiscovery, DLP, SIEM correlation, chain-of-custody exports. (Shared 600 RPM/parent-org across all `/v1/compliance/` endpoints.)
- **Compliance Chat Read** — args: `chat_id` — `GET /v1/compliance/apps/chats`, `GET .../chats/{chat_id}/messages`.
- **Compliance Chat Delete** — args: `chat_id` — `DELETE .../chats/{chat_id}`.
- **Compliance File Read/Delete** — args: `file_id` — `GET .../chats/files/{file_id}/content`, `GET .../files/{file_id}`, `DELETE .../files/{file_id}`.
- **Compliance Generated-Content** — args: `gen_file_id` / `artifact_version_id` — `GET .../generated_files/{gen_file_id}/content`, `GET .../artifacts/{artifact_version_id}/content`.
- **Compliance Project Read/Delete** — args: `project_id` — `GET /v1/compliance/apps/projects`, `DELETE .../projects/{project_id}`.
- **Org Directory Enumeration** — capability of Compliance API.

### Module: HIPAA / BAA
- **HIPAA / BAA Enablement** — self-serve: download BAA + Implementation Guide, accept as authorized legal rep; permanent, cannot be disabled; API auto-enforces feature restrictions.

### Module: GDPR & Privacy Controls
- **GDPR Rights** — access, rectification, erasure, data portability, object, restriction.
- **Privacy Controls** — model training opt-out, public chat sharing, user feedback, chat retention; API privacy: training opt-out, data retention settings, Labs model access.

### Module: Compliance Key Types & Scopes (capabilities)
- Compliance Access Key (`sk-ant-api01-...`, all scopes, immutable); Admin API key (`read:compliance_activities` only).
- Scopes: `read:compliance_activities`, `read:compliance_user_data`, `delete:compliance_user_data`, `read:compliance_org_data`.
- Read + delete key separation recommended (two keys).
- Compliance API errors: 400 (bad cursor / time-filter+sort-key pairing), 401, 403 (`Missing required scopes`), 409 (project delete with attached chats), 429 (shared 600/min/parent-org), 5xx.

### Module: Compliance / Privacy Capabilities
- ZDR-eligible features (Messages default for many features, Token Counting, Cache Diagnostics, `inference_geo: us`); per-endpoint ZDR eligibility tables.
- Non-ZDR / stateful resources (Managed Agents sessions persist until deleted; code-execution container data retained ≤30 days).
- Workspace-level 30-day data retention override (orgs with ZDR).
- CMEK legal retention exceptions (NCMEC reports under 18 U.S.C. § 2258A, exigent risk of serious harm, ToS violations).
- Supported regions / country access list; free-tier availability in all available regions.
- Data use by tier (Free tier content used to improve products; Paid/Enterprise NOT used).
- Licensing (content CC BY 4.0; code samples Apache 2.0).
- PHI guidance (PHI in message content/files/metadata; not in workspace names, user info, billing, support tickets; `strict:true` schemas cached separately — not PHI-protected).
- KYC registration/login requirements, linking existing accounts, credit card/ID (capability of Identity).
- Compliance certifications: SOC 2, HIPAA, GDPR/DPA/SCC, Enterprise SLA, SSO/SCIM.

## Domain L0.N — Data Residency, Encryption & Retention

### Module: Residency Policy
- **Set Data Residency / Retention Policy** — args: `project_id`, `retention_type` — `PATCH /v1/organization/projects/{id}/data_retention`, workspace privacy controls, admin-panel privacy controls.

### Module: Encryption
- **Encryption-at-Rest (CMEK/EKM)** — args: `kms_key_id`, `key_policy` — encrypt at rest with own KMS key (cross-account key policy / EKM KMS provider list).

### Module: Residency & ZDR Capabilities
- **Data residency**: project-level residency with regional host prefixes; workspace geo (`default_inference_geo`/`allowed_inference_geos`, per-request `inference_geo` `global`|`us`); Cloud project region; regional storage-only regions; regional processing uplift; EU data residency; `processing_location: us/eu`.
- **Zero Data Retention (ZDR)**: forces `store=false`; per-tool ZDR eligibility (client tools + basic web search/fetch + MCP eligible; code execution not — 30-day retention); `allowed_callers:["direct"]` web-search bypass; MCP compatible with ZDR.
- **Modified Abuse Monitoring (MAM)** — excludes customer content from abuse logs while preserving capabilities.
- **Application State** — data persisted by API features to fulfill a task.
- **`store` parameter** boolean (per-API defaults: `generateContent` defaults false, Interactions defaults true; toggling Interactions off disables history auto-storage).

## Domain L0.O — Webhooks & Event Delivery

### Module: Webhook Service
- **Register Webhook** — args: `url`, `events[]`, `secret` — webhook CRUD; static (HMAC) vs dynamic (JWKS RS256) webhooks.
- **Webhook Inbound HTTP** — Standard Webhooks spec (`webhook-id`/`timestamp`/`signature`).
- **Dynamic Webhook Configuration** — args: `uris`, `user_metadata`, `revocation_behavior`.
- **Rotate Webhook Secret** — secret rotation.

### Module: Webhook Capabilities
- Static (HMAC) vs dynamic (JWKS RS256) webhooks.
- Standard Webhooks HTTP headers.
- Retry / backoff policy; dead-letter queue.
- Webhook event catalog (see S.21 event type catalog).
- Webhook verification & idempotency (see S.17, S.11).

## Domain L0.P — Developer SDKs, CLI & Specs

### Module: SDK / CLI / Spec
- **OpenAI SDK drop-in** — OpenAI-compatible client points at platform base URL.
- **Native Python SDK** — per-platform native SDK.
- **Native CLI** — per-platform native CLI.
- **OpenAPI spec** — published.

### Module: SDK Capabilities
- SDK languages vary by surface (Node, Python, Go, Ruby, Java; Python, JS V1/V2).
- SDK streaming helpers (see S.10).
- SDK major version semantic versioning; V1 → V2 unified-package migration.
- Ecosystem integrations (frameworks / enterprise gateways / partner integrations / cloud integrations with issue trackers and chat platforms / storage / vector stores / app platforms).
- Self-hosted deployment (Docker / Kubernetes / SageMaker; on-prem).
- BYOB (Bring Your Own Bucket).

## Domain L0.Q — Routing & Gateway (control plane)

> The *data-plane* routing policies stay in L1; this L0 module records the **control-plane** routing/gateway services.

### Module: Gateway Endpoint Management
- **Create Gateway Endpoint** — args: slug, target (`provider`, `model_id`, `env`) — `POST /v1/gateway/endpoints` — map slug → target.
- **List Gateway Endpoints** — `GET /v1/gateway/endpoints`.
- **Update Gateway Endpoint** — args: re-point target — `PATCH /v1/gateway/endpoints/{id}`.
- **Delete Gateway Endpoint** — `DELETE /v1/gateway/endpoints/{id}`.
- **Mint Federated Key** — args: group id — `POST /v1/gateway/groups/{id}/api_keys`.
- **Register Existing Signed Key** — `POST /v1/gateway/groups/{id}/api_keys/register`.
- **Model Gateway Passthrough** — `POST /v1/gateway/model/chat/completions`, `/embeddings` — OpenAI-compatible gateway passthrough.

### Module: Routing Control Capabilities
- Model routing & fallbacks (model-level `models[]` fallback array; router slugs `auto`/`free`/`fusion`/Pareto Router; provider-level routing `provider.order`, `allow_fallbacks`, `require_parameters`, `data_collection:deny`, `zdr:true`, `sort`, performance floors, `max_price`, `quantizations`; session stickiness).
- Plugins (web, file-parser, response-healing, context-compression, moderation, web-fetch, fusion, auto-router, pareto-router).
- Presets (named server-side configuration referenced by slug).
- Whole-response caching (gateway): `X-Response-Cache:true` header, `X-Response-Cache-TTL` (1–86400 s), `X-Response-Cache-Clear:true`; no request coalescing.

## Domain L0.R — Secrets & Credentials (Vault)

### Module: Vault Service
- **Create Vault** — args: `name` — `POST /v1/vaults` — write-only storage; unique key per vault; keys immutable; ≤20 credentials/vault.
- **Vault CRUD** — `GET/DELETE /v1/vaults[/{id}]`.
- **Create Credential** — args: `type` (`mcp_oauth` / `static_bearer` / `environment_variable`), value — `POST /v1/vaults/{id}/credentials`.
- **Credential CRUD** — `GET/DELETE /v1/vaults/{id}/credentials/{cid}`.
- **Rotate Credential** — args: `credential_id` — propagates to running sessions.
- **Validate MCP OAuth** — `POST /v1/vaults/{id}/credentials/{cid}/mcp_oauth_validate`.

### Module: Connections (External App Auth)
- **Create Connection Application** — args: OAuth2 client_id/secret, API key, basic auth — `POST /v1/connections/applications`.
- **OAuth Callback** — `GET /v1/connections/callback`.
- **Bind by `connection_id`** — `connection_id` reference.

### Module: Vault Capabilities
- Credential types: `mcp_oauth`, `static_bearer`, `environment_variable`; API keys; env vars / headers for MCP.
- Referencing at session / run: vault IDs at session creation.
- Rotation (propagates to running sessions).
- Outbound policies (see L0.D).
- Cloud secrets lifecycle.
- `.worktreeinclude` (worktree-secret scoping).
- Egress proxy header transform (see L0.D).
- Data / training policy: data accessed via Connectors never used to train/fine-tune models.
- Vault lifecycle webhooks: `vault.archived`, `vault.deleted`, `vault_credential.archived`/`deleted`/`refresh_failed`.

## Domain L0.S — Process & Filesystem (host-side RPC)

### Module: Filesystem API (App-Server v2)
- **File ops** — `readFile` / `writeFile` / `createDirectory` / `getMetadata` / `readDirectory` / `remove` / `copy` / `watch`.
- **Process ops** — `process/spawn` + `writeStdin` / `resizePty` / `kill`.
- **Command exec** — `command/exec`.
- **Config ops** — `config/read` / `value` / `write` / `batchWrite` / `configRequirements/read`.

---
# LAYER 1 — COMPUTE & MODEL SERVING

> **Purpose.** Build inference serving on top of L0 base compute/network/storage services: package and deploy models onto GPU hardware, route inference requests to running replicas, and keep the fleet reliable. Every L2 inference call is ultimately served by an L1 deployment.
> **Depends on**: L0 (identity, billing, network, sandboxes, observability plumbing, compliance, SDKs).

## Domain L1.A — Hardware & Model Catalog

### Module: Hardware & Region Catalog
- **List Models** — args: filters (`provider?`, `task?`) — `GET /v1/models` — list models with pricing/latency/throughput/features.
- **Get Model** — args: `model_id` — `GET /v1/models/{model_id}` — one model with all variants.
- **Filter Models** — args: `provider`, `task` — `GET /v1/models?provider=&task=` — filtered model list.
- **List Deployable Templates** — `GET /v0/templates` — deployable model+flavor+gpu+region combos (dedicated).
- **List Hardware Options** — args: `model` — `GET /v1/hardware?model={id}` — hardware options for a model.
- **List Availability Zones** — `GET /v1/availability-zones`.

### Module: Catalog & Variant Capabilities
- Catalog metadata: `id`, `context_length`, `architecture` (modalities/tokenizer), `pricing`, `supported_parameters`, `reasoning` (efforts/mandatory), `benchmarks`, `expiration_date`.
- Catalog query filters: `output_modalities`, `supported_parameters`, sort by pricing/context/throughput/latency/popularity/newest.
- Model lifecycle stages: Experimental / Preview / Stable.
- Dated-snapshot vs rolling-alias variants (`-latest`, etc.).
- Model "warm" status tag — indicates model kept warm in shared fleet.
- Unified model id grammar: `<namespace>/<model>[:<flavor>][@<snapshot>][#<deployment>]` with optional `:provider`/`:policy` suffixes for routed serverless.
- Fast variant suffix (`-fast`, `:fastest`, `routers/...-fast`, `Turbo`).
- `service_tier` enum (`auto`/`default`/`over-limit`/`flex`/`no-limit`/`priority`) — also a routing/rate-limit capability.
- Deployment shape templates (Fast / Throughput / Minimal).
- Provider routing policy (`:fastest`/`:cheapest`/`:preferred`/`:<provider>`).
- Data-plane model id forms: `/Qwen/...` (deployment name as `model`); `accounts/<acct>/deployments/<id>`; `model#deployment` composite (multi-LoRA); `routing_key` (returned at endpoint creation); endpoint subdomain (`https://model-{id}.api...`, `https://{id}.{region}.{cloud}.endpoints...`).

## Domain L1.B — Model Packaging & Artifact Management

### Module: Weights Supply (capabilities)
- HF repo reference (`hf://org/repo@revision`, `MODEL_NAME`).
- 4-step signed upload (register → getUploadEndpoint → PUT → validateUpload).
- Files API archive upload (`upload-custom-model-archive`, `client.files`).
- `weights[].source` block with URI schemes (hf/s3/gs/azure/r2/bt/cw) + auth + `allow_patterns`.
- Custom container image (Docker Hub/ECR/ACR/GCR/GHCR/NGC).
- Custom handler (`handler.py` / `EndpointHandler` / `Sprocket` / Truss `Model`).
- Init hook (`Model.load` / `EndpointHandler.__init__` / `Sprocket.setup`) — distinct from predict entrypoint (`Model.predict` / `EndpointHandler.__call__` / `Sprocket.predict`); runs once at replica startup.

### Module: LoRA Adapters (capabilities)
- Live merge (one LoRA at deploy time, no runtime overhead).
- Multi-LoRA (base + addons loaded at request time; `model#deployment`, `lora_adapters`, `enable_lora`).
- Per-request image-gen LoRA (`loras:[{scale, url}]`).
- Merge single adapter into base weights (perf recommendation).

### Module: Quantization (capabilities)
- FP16/BF16 (`no_quant`).
- FP8/FP8_KV.
- FP4/FP4_KV/FP4_MLP_ONLY (Blackwell).
- INT8/SmoothQuant.
- AWQ/GPTQ pre-quantized checkpoints.
- GGUF (llama.cpp).
- bitsandbytes.

### Module: Cold-Start Mitigation (artifact side, capabilities)
- Multi-tier weight mirroring (blob→cluster→node).
- FlashBoot state retention / image pre-loading.
- Model cache on hosts.
- Compilation caching (`b10cache` for torch.compile / CUDA graphs).
- Network volumes shared across workers.

## Domain L1.C — Compute Provisioning & Deployment

### Module: Deployment CRUD
- **Create Deployment** — args: model, hardware, region, parallelism, engine args, autoscaling, mode — `POST /v1/deployments` — returns id/routing_key/state.
- **List Deployments** — `GET /v1/deployments`.
- **Get Deployment** — args: `id` — `GET /v1/deployments/{id}`.
- **Patch Deployment** — args: `id`, mutable fields (region immutable) — `PATCH /v1/deployments/{id}`.
- **Delete Deployment** — args: `id` — `DELETE /v1/deployments/{id}`.
- **Start/Stop/Wake Deployment** — args: `id`, action — `POST /v1/deployments/{id}/{start|stop|wake}`.
- **Activate/Deactivate/Wake** — `POST /v1/deployments/{id}/{activate|deactivate|wake}`.
- **Reset/Restart Deployment** — args: `id` — reset/restart running deployment.

### Module: Deployment Mode (capabilities)
- Serverless / shared public endpoints (per-token, no cold start).
- Dedicated endpoint / deployment (per-minute, reserved GPUs, autoscaling).
- Provisioned throughput PTU (SLA-backed reserved capacity, 1-month min).
- Dedicated containers (bring Docker image + job queue).
- GPU clusters (K8s/Slurm, full control).
- Pods (raw GPU instance, no autoscaling).

### Module: Hardware & Region Selection (capabilities)
- Named instance types (`L4:4x16`, `H100:8`).
- Accelerator enum + count.
- GPU type string IDs (~50 NVIDIA/AMD).
- GPU pools grouped by architecture+VRAM (`AMPERE_80`).
- Cloud provider + instance (AWS/Azure/GCP).
- Template-driven valid combos.
- Region enum at creation (immutable); cloud+region; per-endpoint subdomain per region; data center priority / country restriction.

### Module: Parallelism (capabilities)
- Tensor parallelism (`tensor_parallel_count/size`, `gpu_count`).
- Data parallelism (`Data Parallel Size`).
- Pipeline parallelism.
- Expert parallelism (MoE).

## Domain L1.D — Inference Engine Configuration

### Module: Engine Choice (capabilities)
- vLLM (PagedAttention, continuous batching).
- TGI (legacy, migrating to vLLM).
- SGLang (RadixAttention, structured output, multi-LoRA batching).
- llama.cpp (GGUF, CPU+GPU).
- TEI (embeddings/rerank).
- TensorRT-LLM (via Engine-Builder / BIS-LLM / BEI variants).
- BIS-LLM (MoE+dense, token autoscaling, KV-aware routing, disaggregated serving, spec decoding).
- BEI / BEI-Bert (embeddings, rerank, classification, NER).
- Inference Toolkit fallback.
- Custom container (must expose `/health`).
- Ollama / custom inside a Pod.

### Module: Performance Optimizations (capabilities)
- PagedAttention / paged KV cache.
- Flash Attention.
- Continuous batching.
- Chunked prefill.
- Context/prompt caching (automatic serverless; configurable dedicated).
- KV cache quantization (`fp8`, `fp8_e5m2`, `fp8_e4m3`).
- Speculative decoding (lookahead / Eagle / MTP / N-gram / draft model).
- Speculative-decoding tunables: `lookahead_ngram_size`, `lookahead_verification_set_size`, `lookahead_windows_size`, `enable_b10_lookahead`, `--no-speculative-decoding`, `speculator` block.
- Disaggregated serving (prefill/decode split).
- KV-aware routing (route to replica with cached prefix).

### Module: Engine Args (capabilities)
- `max_num_seqs`, `max_num_batched_tokens`, `tensor_parallel_size`, `data_parallel_size`, `kv_cache_dtype`, `gpu_memory_utilization`, `enable_chunked_prefill`, `enable_lora`, `block_size`, `swap_space`, `max_concurrency`, `served_model_name`.
- `max_num_tokens` / `max_batch_size` (max tokens per batch).
- `kv_cache_free_gpu_mem_fraction` (KV cache fraction).
- Runtime config — `batch_scheduler_policy`, `webserver_default_route`.

## Domain L1.E — Deployment Lifecycle & Traffic Management

### Module: Environment & Promotion Lifecycle
- **Promote to Environment** — args: `model_id`, `env` — `POST /v1/models/{id}/environments/{env}/promote`.
- **Patch Environment Config** — args: `model_id`, `env`, `max_surge`, `max_unavailable` — `PATCH /v1/models/{id}/environments/{env}` — rolling-deploy config.
- **Control Promotion** — args: `model_id`, `env`, action ∈ {pause, resume, cancel, force_cancel, force_roll_forward} — `POST /v1/models/{id}/environments/{env}/{action}_promotion`.

### Module: Autoscaling
- **Patch Autoscaling Settings** — args: `id`, `min_replica`, `max_replica`, `autoscaling_window`, `scale_down_delay`, `concurrency_target`, `target_utilization_percentage`, `target_in_flight_tokens`, `load_targets`, `scale_up_window`, `scale_down_window`, `scale_to_zero_window`, `idle_timeout`/`inactive_timeout`, `scale_down_half_life_seconds` — `PATCH /v1/deployments/{id}/autoscaling_settings`.

### Module: Autoscaling Capabilities (scaler signals)
- Concurrency per replica (`concurrency_target`, `target_utilization_percentage`, `MAX_CONCURRENCY`).
- Tokens in flight (`target_in_flight_tokens`).
- GPU utilization threshold (default 80%).
- Pending requests (per replica, default 1.5, 20 s window).
- Queue delay / request count scalers (`QUEUE_DELAY`, `REQUEST_COUNT`).
- Load targets (`tokens_generated_per_second`, `prompt_tokens_per_second`, `requests_per_second`, `concurrent_requests`, `default`); max replica count across all targets.
- K8s Cluster Autoscaler (scales nodes for GPU clusters).

### Module: Scale-to-Zero & Cold Starts (capabilities)
- `min_replica=0` / `workersMin=0` / `min_replicas=0`.
- Auto-delete after N days idle.
- Wake endpoint before use (`POST /wake`, `resume`, `enabled`, `start`).
- Pre-warming (raise min replicas ahead of spikes).
- `X-Scale-Up-Timeout` header (hold during scale-up).
- 503 DEPLOYMENT_SCALING_UP (immediate, retry).
- Parking (request waits at routing layer).

### Module: Health Checks
- **Health Endpoint** — `GET /health` — 200 ready / 503 loading.

### Module: Health Check Capabilities
- Startup probe (model finished initializing; 30-50 min timeout).
- Readiness probe.
- Liveness probe (restart on failure).
- Custom `is_healthy()` logic.

### Module: Environments & Promotion (capabilities)
- Environments (dev/staging/production) with stable endpoints.
- Rolling deployments (gradual shift, `max_surge`/`max_unavailable`, pause/resume/cancel/force_roll_forward).
- Development deployment with `truss push --watch` live-reload.
- Labels (key-value metadata on deployments).
- Frontier Gateway endpoint re-point (control plane in L0).
- Publishing / managing default deployments.
- Pause/resume (stop billing, keep config).

## Domain L1.F — Request Routing, Gateway & Rate Limiting (data plane)

### Module: Replica Selection (data-plane routing capabilities)
- Least-utilized replica (concurrency headroom).
- KV-aware routing.
- Session affinity / sticky routing (`x-session-affinity`, `x-multi-turn-session-id`, `user`).
- Load balancing (direct, no queue).
- Queue-based (built-in job queue + handler).
- Regional data-plane URL.

### Module: Rate Limiting (capabilities)
- Static account-tier RPM/TPM.
- Adaptive/dynamic (15-min buckets, ×1.2 up / ÷1.5 down / 20× ceiling).
- Per-group per-model with inheritance (`INDEPENDENT` vs `CASCADING`).
- Service tiers (`auto`/`default`/`over-limit`/`flex`/`no-limit`, `priority`).
- Token-based vs request-based (TOKEN/REQUEST, SECOND/MINUTE/DAY).
- Daily usage windows (reset midnight UTC).
- (Rate-limit response headers — see S.4.)

## Domain L1.G — Inference Request Execution

### Module: Inference — Chat / Completions
- **Chat Completions** — args: messages, model, sampling, tools, response_format, reasoning, stream — `POST /v1/chat/completions`.
- **Completions (legacy)** — args: prompt, model, sampling, stream — `POST /v1/completions`.
- **Responses API** — args: input/model, `previous_response_id`, tools, store, metadata, background — `POST /v1/responses` — stateful.
- **Messages (compat)** — args: messages, model, system, tools, thinking — `POST /v1/messages`.
- **Inference (compat alternate route)** — `POST /inference`.

### Module: Inference — Specialized Tasks
- **Embeddings** — args: input, model, encoding_format — `POST /v1/embeddings`.
- **Rerank** — args: query/documents, model, top_n — `POST /v1/rerank` (variant: `/rerank`).
- **Image Generations** — args: prompt, model, size, n, loras — `POST /v1/images/generations`.
- **Audio Transcription** — args: file/audio_url, model — `POST /v1/audio/transcriptions`.
- **Audio Translation** — args: file/audio_url, model — `POST /v1/audio/translations`.
- **Audio Speech** — args: input, model, voice — `POST /v1/audio/speech`.
- **Pipeline Tasks** — args: inputs — Summarization / classification / NER / QA / fill-mask / object detection / zero-shot / table QA / image-to-image / text-to-video (via `/models/{id}`).
- **BEI Predict / Predict Tokens** — args: inputs — `/predict`, `/predict_tokens` for embeddings/rerank/classification/NER.

### Module: Execution Modes (capabilities)
- Synchronous (block for result).
- Streaming (SSE, token-by-token, `stream:true`, `stream_options.include_usage`).
- Async (`async_predict`, `/run`+`/status`, queue API, `background:true`).
- Batch (JSONL, 50% off, 24 h window).
- RL rollout (session affinity, hot-load, MoE router replay).
- Responses API (stateful, `previous_response_id`, built-in tools, MCP).

### Module: Multimodal Input (capabilities)
- Image (`image_url` block, URL or base64; max 30 images on some surfaces).
- Video (`video_url` base64).
- Audio (`audio_url` base64).
- File inputs (base64, URL, `preprocess()`).

### Module: Async Inference Lifecycle
- **Submit Async Predict** — args: predict args, priority, retry_config, webhook — `POST /v1/async_predict` → `{request_id}`.
- **Get Async Request Status** — args: `id` — `GET /v1/async_request/{id}` — QUEUED/IN_PROGRESS/SUCCEEDED/FAILED/EXPIRED/CANCELED/WEBHOOK_FAILED.
- **Cancel Async Request** — args: `id` — `DELETE /v1/async_request/{id}`.
- **Async Queue Status** — `GET /v1/async_queue_status`.
- **Run/Status/Cancel** — args: input, id — `POST /run` + `GET /status/{id}` + `POST /cancel/{id}` — generic async execution primitives.
- **Dedicated-Containers Queue API** — args: job params — async queue.

### Module: Async Job Capabilities
- Async priority levels (0/1/2).
- Async retry config (`inference_retry_config`: `max_attempts`/`initial_delay`/`max_delay`).
- Webhook delivery (signed HMAC, 2 attempts ~2 s backoff).
- Async max queue time (72 h / configurable TTL).

### Module: Batch Inference Lifecycle
- **Submit Batch** — args: file (JSONL), model, params — `POST /v1/batches`.
- **Get/List/Cancel Batch** — args: `id` — `GET /v1/batches/{id}`, `GET /v1/batches`, `POST /v1/batches/{id}/cancel`.
- **Submit BatchInferenceJob** — args: file, model, params, window (12/24/48/72 h) — `POST /batchInferenceJobs`.
- **Get BatchInferenceJob** — args: `id` — `GET /batchInferenceJobs/{id}`.
- **Batch Files I/O** — args: file id — Files API for batch input/output download.

### Module: Batch Capabilities
- Batch discount (50% off serverless; per successful response).
- Batch window (12/24/48/72 h).

### Module: RL Rollout
- **Hot Load** — args: `reset_prompt_cache` ∈ {all, new_session, none}, headers `x-multi-turn-session-id`, `x-session-affinity`, `fireworks-model`, `fireworks-deployment`; body `include_routing_matrix`, `logprobs`, `echo` — `POST /hot_load/v1/models/hot_load` — RL rollout hot-load / MoE router replay.

### Module: Cold-Start Mitigation (control-plane knobs)
- **Wake Endpoint** — args: `id` — `POST /wake` / resume / enabled / start — wake a scaled-to-zero deployment.
- **Pre-warm** — args: `id`, min replicas — raise min replicas ahead of spikes.

## Domain L1.H — Output Control & Generation Parameters

### Module: Sampling & Penalties (capabilities)
- `temperature`, `top_p`, `top_k`, `min_p`, `typical_p`, `mirostat_target`/`mirostat_lr`, `seed`, `n`, `best_of`, `use_beam_search`, `length_penalty`, `ignore_eos`, `skip_special_tokens`, `watermark`.
- `frequency_penalty`, `presence_penalty`, `repetition_penalty` (exponential on some), `logit_bias`.
- `max_tokens` / `max_completion_tokens` / `max_output_tokens`, `stop` (up to 4), `truncate` (input).

### Module: Reasoning Control (capabilities)
- `reasoning:{enabled}` toggle (hybrid models).
- `reasoning_effort` (low/medium/high/xhigh/minimal/none).
- `thinking:{type:"enabled", budget_tokens}` (compat).
- `chat_template_kwargs` (thinking/enable_thinking/clear_thinking/medium_effort).
- `reasoning` object (Responses API, gpt-5/o-series).
- Preserved / turn-level / interleaved thinking.
- Reasoning output-field variants (`reasoning` / `reasoning_content` / `thought` tags).

### Module: Structured Output (capabilities)
- `response_format:{type:json_schema|json_object|text}` + strict.
- `response_format:{type:regex, pattern}`.
- `grammar:{type, value}` (TGI-style).
- `output_config:{format:{type:json_schema, schema}}` (compat).
- Pydantic via `client.beta.chat.completions.parse`.
- `text.format` / `text_format` (Pydantic, Responses API).

### Module: Function / Tool Calling (capabilities)
- `tools` array (function schema).
- `tool_choice` (auto/none/required/minimal/low/medium/high/xhigh/{function}).
- `max_tool_calls`, `parallel_tool_calls`.
- Tool-call parser env (`mistral`/`hermes`/`llama3_json`/`granite`).
- Built-in tools (file search, code interpreter, web search, computer use, MCP, image generation, local shell).
- MCP tools (`type:"mcp"`, `server_url`, `allowed_tools`, `require_approval`).
- SSE server-executed tools (`type:"sse"`, `server_url`).
- Function client-executed tools (`type:"function"`).

### Module: Inspection & Service Tier (capabilities)
- `logprobs`, `top_logprobs`, `echo`, `return_token_ids`, `raw_output`, `decoder_input_details`, `details`, `include_routing_matrix` (MoE expert selection).
- `service_tier`, `user`, `store`, `metadata`, `previous_response_id`, `background`, `prompt_cache_key`, `truncation`, `include`, `perf_metrics_in_response`.

## Domain L1.I — Observability Plumbing (runtime metrics/logs)

> Telemetry/metrics/traces plumbing is in L0/L5; this L1 module records the **runtime metrics/logs emitted by inference endpoints** that flow into L0/L5 backends.

### Module: Metrics Export
- **OpenMetrics API** — args: `id` — `GET /v1/endpoints/{id}/open-metrics` — Prometheus text format.
- **Prometheus + Grafana** — managed observability stack.
- **Prometheus-style metrics endpoint** (dedicated).
- **Analytics dashboard** (web UI).
- **Runtime Logs** — args: `id`, filters (timestamp/level/content/replica) — `GET /v1/endpoints/{id}/logs` — real-time, filterable; 30-day retention.

### Module: Metric Categories (capabilities)
- Traffic metrics (RPM, input/output tokens/min, tokens per request).
- Latency metrics (p50/p90/p95/p99, TTFT, end-to-end, TPS).
- Autoscaling/capacity metrics (active replicas, ready replicas).
- Error metrics (by status code, error rate).
- Engine-level metrics (`tps_per_request`, `speculation_rate`, `kv_cache_hit_rate`, cpu/gpu/memory usage).
- Cold start time.
- Execution/delay time percentiles.

## Domain L1.J — Reliability & Retries

### Module: Reliability & Retries (inference-specific capabilities)
> Generic retry policy / backoff convention is in S.4.

- Internal routing retries (502/503/504, exp backoff 500 ms ×1.5 cap 60 s, 15 min).
- Circuit breaker (disable retries under memory pressure).
- Hedge requests (duplicate after delay to reduce p99).
- Async retry config.
- Webhook delivery retries (2 attempts ~2 s backoff 10 s timeout).
- Billing webhook retries (exp 1 s→5 s, 15 s max, dead-letter queue).
- Parking (request waits for a replica).
- Load shedding (429 when queued payloads exceed memory).
- Request cancellation (client disconnect → 499).
- Rolling deployment rollback (cancel → traffic returns to previous).
- Reserved capacity (guarantee during scale-up).
- Reservations (guarantee for clusters).
- `hedge: {hedge_delay, hedge_budget_pct}`.
- `circuit_breaker: {memory_threshold_pct, cooldown_seconds}`.

---
# LAYER 2 — MODEL INFERENCE & INTELLIGENCE APIs

> **Purpose.** The intelligence primitives: model catalog, generation endpoints (legacy + modern), reasoning control, structured output, tool-calling primitive, streaming, context management, multimodal input, embeddings/rerank, batch, grounding/citations. These are the building blocks L3 modality products and L4 agents call.
> **Depends on**: L1 (deployments serve these endpoints); L0 (files, auth, billing, quotas, caching/token-counting, routing control).

## Domain L2.A — Model Catalog & Selection

### Module: Model Catalog
- **Model List** — `GET /v1/models` — list available models.
- **Model Get** — args: `id` — `GET /v1/models/{id}` — single model with all variants/providers.
- **Model Resolve Alias** — args: `author`, `slug` — `GET /api/v1/model/{author}/{slug}` — resolve alias/slug to canonical model id.
- **Model Policy / Governance** — `GET/POST /v1/model_policy` — read/apply model-selection governance policies.

### Module: Catalog & Selection Capabilities
- Catalog metadata fields: `id`, `context_length`, `architecture` (modalities/tokenizer), `pricing`, `supported_parameters`, `reasoning` (efforts/mandatory), `benchmarks`, `expiration_date`.
- Catalog query filters: `output_modalities`, `supported_parameters`, sort by pricing/context/throughput/latency/popularity/newest.
- Model lifecycle stages: Experimental / Preview / Stable.
- Dated-snapshot vs rolling-alias variants (`-latest`, etc.).
- Model variant suffixes: `:nitro` (fastest), `:floor` (cheapest), `:thinking` (extended reasoning), `:free`, `:extended` (larger context), `:exacto` (best tool-calling), `:online` (deprecated).
- Per-agent / per-run model selection: `model` param/path; run-level / turn-level override (`RunConfig(model)`, `turn/start model`, `agent_with_overrides.model`, `conversations.start(model)`); process-wide fallback (`OPENAI_DEFAULT_MODEL`, `fallback_model`).
- Provider capability bounds query (`modelProvider/capabilities/read`).
- Model rerouting / safety buffering events (`model/rerouted`, `model/safetyBuffering/updated`).
- Multi-agent model variant (`*-multi-agent`, `reasoning.effort` controls agent count 4/16).
- Routing fallback array (`models[]` with context-length/moderation/rate-limit/downtime triggers; safety-refusal fallback up to 3).
- Router slugs (`auto`, `free`, `fusion`, Pareto Router).
- Provider-level routing controls: `provider.order`, `allow_fallbacks`, `require_parameters`, `data_collection:deny`, `zdr:true`, `sort:price/throughput/latency`, `preferred_min_throughput`, `preferred_max_latency`, `max_price`, `quantizations`.
- Session stickiness (pin resolved model+provider per conversation).

## Domain L2.B — Generation API Surfaces

### Module: Modern Generation
- **Responses Create** — args: input/model, `instructions`, `previous_response_id`, tools, store, metadata, background, reasoning — `POST /v1/responses` — typed `output[]` Items, reasoning items, encrypted content.
- **Interactions Create** — args: input/model, `system_instruction`, `previous_interaction_id`, tools — `interactions.create` — typed `steps[]`, thought steps with signatures.
- **Messages Create** — args: messages, model, `system`, tools, thinking — `POST /v1/messages` — `content[]` typed blocks, thinking blocks with signatures.

### Module: Generation (legacy/compat)
- **Chat Completions Create** — args: messages, model, sampling, tools, stream — `POST /v1/chat/completions` — `messages[]`, `choices[].message` (primary or legacy surface).
- **Generate Content (legacy)** — args: contents, model — `models.generate_content` — `candidates[].content.parts[]`.
- **Analyze Text / Jobs (typed kind)** — args: `kind` discriminator — `:analyze-text` / `jobs` endpoints.
- **Gateway Passthrough** — `POST /v1/gateway/model/chat/completions`, `/embeddings` — OpenAI-compatible gateway passthrough (control plane in L0).

### Module: Generation Family Capabilities
- **System-instruction placement variants**: top-level `instructions`, `system_instruction`, `system` (string or text blocks), `system` first-message role, `system`/`developer` role in `messages[]`, mid-conversation `system` insertion (cache-preserving).
- **Sampling parameters**: `temperature`, `top_p`, `top_k`, `min_p`, `max_tokens`/`max_output_tokens`/`max_completion_tokens`, `stop`/`stop_sequences`, `frequency_penalty`, `presence_penalty`, `repetition_penalty`, `seed`/`random_seed`, `n` (multiple candidates), `logprobs`/`top_logprobs`, `logit_bias`, `verbosity`, `prediction` (predicted output), prefill / output-constraining (assistant-last-entry, `prefix:true`).
- **Reasoning-model parameter restrictions**: disallow penalties/`stop`; extended-thinking incompatible with temp/top_k modification.
- **Request-level search grounding controls**: `web_search_options` / `search_parameters`; `background` field for OpenResponses compatibility.
- **Deep Research mode**: extended agentic investigation; `background:true` for async; recommended effort `high`/`xhigh`; search-context cap (128k) even when model window larger.
- See L2.C (reasoning), L2.D (structured output), L2.E (tools), L2.F (streaming), L2.G (context mgmt), L2.H (multimodal input) for additional capabilities.

## Domain L2.C — Reasoning / Thinking Configuration

### Module: Reasoning Modes & Effort (capabilities)
- `reasoning.effort` (none/minimal/low/medium/high).
- `thinking_level` (minimal/low/medium/high).
- `output_config.effort` (low/medium/high/xhigh/max) + `thinking` (enabled/adaptive/disabled).
- `reasoning_effort` (high/none … minimal/none/low/medium/high/max/xhigh/ultra).
- `reasoning` object (effort or max_tokens).
- `reasoning.exclude` (hide but use).

### Module: Budget-Based & Adaptive (capabilities)
- `thinking.budget_tokens` (manual mode).
- `output_config.task_budget` (loop-level advisory total-work budget, min 20000).
- Adaptive thinking (`thinking:{type:"adaptive"}`, model decides).
- Mandatory-reasoning models (cannot disable).
- Cache pre-warming via `max_tokens:0` (zero-output warm request; unsupported in Batch).

### Module: Reasoning Content in Output (capabilities)
- `reasoning` Items in `output[]` with `summary` (auto/concise/detailed).
- `thought` steps in `steps[]` with `signature` + optional `summary` (text/image).
- `thinking` blocks in `content[]` (text + `signature`, `redacted_thinking` encrypted opaque).
- `ThinkChunk` in content chunk list.
- `reasoning` items with `summary` / `encrypted_content`; `reasoning_content` in Chat.
- `reasoning` field (string) + `reasoning_details` array (summary/encrypted/text types).

### Module: Thought Signatures & Multi-Turn Continuity (capabilities)
- Stateful (server manages via `previous_response_id`/`previous_interaction_id`).
- Stateless (resend all thought/thinking blocks exactly as received).
- Encrypted reasoning replay (`include:["reasoning.encrypted_content"]`).
- Reasoning context mode (`reasoning.context`: auto/all_turns/current_turn).
- Reasoning mode `reasoning.mode:"pro"` (deeper multi-pass).
- Fast mode: `speed:"fast"` + beta header (faster output tokens/sec; invalidates cache; not Batch/Priority).
- Reasoning-token billing rule (see S.8).

## Domain L2.D — Structured Outputs

### Module: Structured Output (capabilities)
- **JSON-schema enforcement**: `text.format` (Responses) / `response_format` (Chat) with `json_schema` + `strict:true`; `response_format` with `schema` (no strict flag); `output_config.format` with `json_schema` (grammar compilation + 24 h caching); `client.chat.parse()` / `client.responses.parse()` / `client.messages.parse()` with Pydantic/Zod/JSON Schema.
- **JSON mode (no schema)**: `json_object` type; `response_mime_type:"application/json"`.
- **Grammar / CFG constraints**: `{type:"grammar", syntax:"lark"|"regex", definition}`; constrained decoding (compiled grammars, 24 h cache); `response_format:{type:"grammar"}` (from OpenAPI); `grammar:{type:"json"|"regex"|"json_schema", value}`; `response_format:{type:regex, pattern}`.
- **Strict-mode contract & schema-complexity limits** (see S.19).
- **JSON Schema keyword support matrix** (see S.19).
- **Refusals in structured outputs**: `refusal` message / `type:"refusal"` content item / `stop_reason:"refusal"` + `stop_details`; output may not match schema.
- **Structured outputs + tools**: combine structured outputs with built-in tools + function calling in same request.

## Domain L2.E — Function Calling & Tool Configuration (primitive)

### Module: Function Tool Definition (capabilities)
- Modern flat `{type:"function", name, description, parameters, strict}`.
- `{type:"function", name, description, input_schema, strict}`.
- Legacy externally tagged `{type:"function", function:{name, description, parameters, strict}}`.
- Custom tools (free-form text I/O, no JSON Schema).
- `input_examples` (validates against schema, ~20-50 tokens/example).
- `strict:true` (guaranteed conformance; not with recursive `$ref`).
- `cache_control` ephemeral on tools (cannot combine with `defer_loading`).
- `defer_loading` (withhold schema until tool search loads).
- `allowed_callers` (`direct`/`code_execution`/`programmatic`, not a security boundary).
- `eager_input_streaming` (fine-grained tool-input streaming, user-defined only).
- `output_schema` (programmatic; describes JSON in `function_call_output`).
- `requires_confirmation` (tool names needing approval).
- Tool namespaces `{type:"namespace", name, description, tools:[...]}` with `defer_loading` on inner functions; keep <10 functions/namespace.

### Module: tool_choice (capabilities — see also S.21)
- `auto`, `required`/`any`, `none`, specific function, subset, `validated` (schema-validated preview), graduated eagerness (minimal/low/medium/high/xhigh), force hosted tool, `max_tool_calls`.
- Parallel & compositional: `parallel_tool_calls:true` (default), parallel+compositional (chained), `disable_parallel_tool_use`.
- Multimodal function responses (images in `result`).
- Tool-definition token cost rule (see S.8).

## Domain L2.F — Streaming

### Module: Streaming Formats (capabilities — see also S.10)
- Delta-chunk SSE format (`stream:true`, `delta.content`/`delta.tool_calls`/`delta.reasoning_content`/`delta.reasoning_details`).
- Typed SSE event formats: `response.created`/`response.output_text.delta`/`response.completed`/`response.function_call_arguments.delta`; `interaction.created`/`step.start`/`step.delta` (text/thought_summary/thought_signature/arguments)/`step.stop`/`interaction.completed`/`done`; `response.reasoning_text.delta`/`response.reasoning_summary_text.delta`; `response.content_part.delta`/`response.reasoning.delta`/`response.done`.
- Content-block event format: `message_start` → `content_block_start` → `content_block_delta` (text_delta/thinking_delta/signature_delta/citations_delta/input_json_delta/compaction_delta) → `content_block_stop` → `message_delta` → `message_stop` → `ping` → `error`.
- Streaming-with-reasoning variants: `thought_summary`+`thought_signature` deltas; `thinking_delta`+`signature_delta` with `display:"omitted"`; Phase 1 ThinkChunk → transition → Phase 3 plain string; `delta.reasoning_details`/`response.reasoning.delta`.
- Streaming structured outputs (partial-JSON concatenation; parse after completion).
- Fine-grained tool-input streaming (`eager_input_streaming`; invalid-JSON → `tool_result` `is_error:true` `{"INVALID_JSON":"<raw>"}`; legacy beta header superseded).

### Module: WebSocket Mode
- **WebSocket Responses** — `WS wss://api.../v1/responses` — persistent connection for long-running, tool-call-heavy workflows (~40% faster for 20+ tool calls); send only incremental input items per turn plus `previous_response_id`; 60-min max duration, one in-flight response at a time.
- **WebSocket payload** — `response.create` with `model`, `input`, `tools`, `generate` fields.
- **Reconnect patterns** — (1) continue with `previous_response_id` if persisted; (2) start fresh with `previous_response_id: null` + full input; (3) use compacted window from `/responses/compact`.

## Domain L2.G — Context Management (Caching, Compaction, Editing)

### Module: Prompt Caching (capabilities)
- **Automatic caching**: automatic prefix-cache routing; single top-level `cache_control` (system-managed breakpoints); implicit caching; sticky-routing + `cache_control`; `cached_tokens`+`prompt_cache_key` sticky routing.
- **Explicit cache breakpoints**: `cache_control` on individual content blocks (max 4 breakpoints); `prompt_cache_breakpoint`; cache files (pay-per-hour).
- **Cache retention/TTL variants**: 5-min default TTL; 1-hour TTL (`cache_control:{type:ephemeral, ttl:"1h"}`, env-flag toggle); in-memory (5-10 min) vs extended 24h (`prompt_cache_retention`, GPU-local).
- **Cache pricing multipliers** (write 1.25×/2×; read 0.1×-0.5×; free for automatic).
- **Cache diagnostics**: `cache_miss_reason` (beta, ZDR-eligible).
- **Cache invalidation rules** (see S.27): byte-for-byte prefix identity; tool/system/messages hierarchy; longer-TTL-before-shorter; tool-def change invalidates all; tool_choice change invalidates messages; images/thinking-param change invalidates messages.

### Module: Context Compaction
- **Responses Compact (standalone)** — `POST /v1/responses/compact` — stateless, ZDR-friendly; returns opaque compaction item with `encrypted_content` to pass verbatim into next request; at most one per call; conversation must already fit.
- **Inline compaction capabilities**: `context_management.edits` with `compact_*` (`trigger`, min, `pause_after_compaction:true` → `stop_reason:"compaction"`, `usage.iterations`); `context_management.compact_threshold` on `/responses`; native compaction at ~135k tokens; configurable threshold + sliding window (`compaction_settings.context_compaction_threshold`/`compaction_sliding_window`/`large_message_threshold`); agent-overridable compaction prompt.

### Module: Context Editing (capabilities)
- `clear_tool_uses_*` with `trigger` (input_tokens or tool_uses, default 100000), `keep` (recent pairs, default 3), `clear_at_least`, `exclude_tools`, `clear_tool_inputs`.
- `clear_thinking_*` with `keep` (recent thinking_turns or "all"); must be listed first in `edits` array; invalidates cache when cleared.

### Module: Context Window Limits (capabilities)
- Everything counts (system prompt, messages, tool results, images, docs, output, extended thinking); overflow → 400.
- Context rot — accuracy degrades as token count grows even within the window.

### Module: Token Counting
> Token-counting services live in L0.L (governance/billing facet); the L2 facet notes that they accept the same params as the generation endpoints (including `context_management`) and return `input_tokens` (post-edit) + `original_input_tokens`.

### Module: Predicted Outputs (capabilities)
- `prediction:{type:"content", content}` to speed up generation when most output known; `accepted_prediction_tokens`/`rejected_prediction_tokens` usage.

### Module: Whole-Response Caching (gateway)
> Gateway-level capability — see L0.Q.

## Domain L2.H — Multimodal Input

### Module: Images (capabilities)
- `input_image` (Responses/Chat) / image content.
- Image parts (`inline_data` base64 / `file_data` URI / URL).
- `image` content blocks (base64/URL/file_id).
- `image_url` content parts (URL/base64).
- Image detail / resolution control: `detail: low` (512×512 fast) / `high` / `original` / `auto`; resolution tiers (high-resolution max long edge 2576 px / 4784 tokens; standard 1568 px / 1568 tokens; 28×28 patches); `media_resolution` (max tokens per image/video frame).
- Max images per request (variable bound, up to ~3600).

### Module: PDF & Documents (capabilities)
- `input_file` (Responses) / `file` (Chat) — PDF/DOCX/PPTX/TXT/code/spreadsheets; base64/file_id/URL; PDF detail auto/low/high; spreadsheets 1000 rows/sheet augmentation.
- Document parts (`inline_data`/`file_data`); page images + text.
- `document` content blocks (base64/URL/file_id); 600 pages (100 for 200k); 32 MB.
- `input_file` parts (URL/file_id); agentic document search.

### Module: Video & Audio (capabilities)
- Video parts (`inline_data`/`file_data`); video Q&A, memory, real-time transcription/translation.
- Audio parts (native audio understanding, no separate STT).

## Domain L2.I — Files API (shared resource)

> The Files API lives in **L0.E**; L2 documents the **shared resource** facet: many L2 consumers use it (vision input, batch input, document citations, agentic file outputs). See L0.E for the canonical service.

## Domain L2.J — Embeddings & Rerank (primitive)

### Module: Embeddings
- **Embeddings Create** — args: `model`, `input`, `dimensions` (MRL shortening), `encoding_format` (float/base64) — `POST /v1/embeddings`.
- **Embeddings (unified/gateway)** — `POST /api/v1/embeddings` — unified multi-provider embeddings.
- **Embed (vo-style)** — `vo.embed()` / `POST /v1/embeddings` — args: `model`, `inputs`, `input_type`.
- **Embeddings (SDK variant)** — `client.embeddings.create(model, inputs)`.
- **Contextualized Embed** — `contextualized_embed()` — chunk embeddings with surrounding context.

### Module: Embedding Capabilities
- MRL dimension shortening (`dimensions`).
- `encoding_format` (float/base64).
- `input_type` (`document`/`query`); contextualized chunk embeddings.
- Multimodal embeddings (text+images unified vector space).
- Vision/VLM embeddings over page images.
- Whole-document embeddings (required for audio/video).
- Named vectors (multi-model per collection); multi-vector (ColBERT/ColPali); self-provided vectors (no vectorizer).
- Auto-vectorization on insert (vectorizer modules).
- Per-property vectorization control (`source_properties`, `vectorize_property_name`, `skip_vectorization`).
- Distance metrics (COSINE/DOT/L2_SQUARED/HAMMING/MANHATTAN); index types (fts/embedding/hybrid).
- Embeddings as ML features (classification/clustering/regression/recommendations).
- Free-at-query-time / pay-at-index-time billing variant.
- Batch (array of strings) and multimodal (image) embedding inputs.

### Module: Rerank
- **Rerank** — args: `model`, `query`, `documents`, top-k — `POST /v1/rerank` — re-ranking retrieved documents.
- **Rerank variants** — `rerank-2.5` / `rerank-2.5-lite`.
- **Rerank (SDK variants)** — `Reranker` / `relevance_scoring` / `rerank`.

## Domain L2.K — Batch Processing

> Batch **job orchestration primitives** are shared with L0.H (Workflows & Scheduled Jobs). L2 records the **inference-specific** batch services and eligibility/billing rules.

### Module: Batch APIs
- **Batch Submit (multi-endpoint)** — args: JSONL input file, model, params — `POST /v1/batches` — targets `/v1/responses`, `/v1/chat/completions`, `/v1/embeddings`, `/v1/completions`, `/v1/moderations`, `/v1/images/*`, `/v1/videos`.
- **Messages Batch Submit** — args: JSONL, model — `POST /v1/messages/batches` — 50% discount; 100k requests or 256 MB; expire 24 h; `.jsonl` via `results_url`; not ZDR.
- **Batch Job Submit** — args: inline or file (1 M requests file / 10k inline), `output_file`+`error_file` — `POST /v1/batch/jobs` — supported endpoints: embeddings/chat/fim/moderations/chat-moderations/ocr/classifications/conversations/audio-transcriptions.
- **Deferred Completion (Chat)** — args: `deferred:true` — `POST /v1/chat/completions` → poll `GET /v1/chat/deferred-completion/{request_id}` (200 completed / 202 pending).
- **Batch Lifecycle** — list/get/cancel batch endpoints (state machine: validating/in_progress/finalizing/completed/expired/cancelled; variants).

### Module: Batch Capabilities
- Batch request structure (`{custom_id, params/body}`; correlation by `custom_id`; order not preserved) — see S.20.
- Batch unsupported parameters (no `stream:true`, no Fast mode `speed`, no `store`/`previous_thread_event_id`, no `max_tokens:0` cache pre-warm).
- Batch eligibility/billing rules (ZDR-ineligibility; data retained up to 29 days; non-billable on `invalid_request_error`/errored/canceled/expired; one model per batch).
- Batch + prompt caching stacking (best-effort 30-98% hits; 1-hour cache recommended for batch).
- Batch target endpoint sets (see services above).
- Deferred single-request async (`deferred:true` + poll).

## Domain L2.L — Grounding, Citations & RAG (primitive)

> The built-in web-search **tool** is catalogued in L4 Tools. Here we record the citation/annotation output it produces at the inference layer.

### Module: Citations / Grounding Annotations (capabilities — see also S.22)
- Document-citation block flag: `citations:{enabled:true}` on `document` blocks; returns `cited_text` with char/page/block locations; incompatible with structured outputs.
- `search_result` / `web_search_result_location` content blocks (source/title/cited_text/encrypted_index).
- `url_citation` annotations (start_index/end_index).
- `search_suggestions` widget HTML (must render).
- `sources` list incl. real-time feeds (sports/weather/finance).
- `tool_reference` chunks.
- `file_citation` (`file_id`/`file_name`/`filename`/`page_number`/`media_id`/`custom_metadata`); `container_file_citation` adds `container_id`.
- `place_citation` (`name`, `url`) — Maps; render as links; attribution + legal notices required.
- Citation visibility/clickability contract (see S.22).

---
# LAYER 3 — AI MODALITY PRODUCTS

> **Purpose.** End-user-facing products built on L2 primitives and L0 storage/files: text & conversation, images & video, voice, documents. Each domain packages L2 generation/embedding/structured-output APIs into shippable modality-specific capabilities.
> **Depends on**: L2 (and transitively L1); L0 (files, storage, async jobs, webhooks, billing, residency, SDKs).

## Domain L3.A — Text & Conversation

### Module: Text Generation
- **Single-Turn Generation** — args: `prompt`/`messages`, `model`, `max_tokens`, `temperature`, structured-output schema — single prompt → text response.
- **Multiple Candidates** — args: `n` (number of completions) — returns N completions per request; billed across all.

### Module: Conversation State
- **Multi-Turn Conversation (stateless replay)** — args: full `messages[]`/items/steps history each turn — replay full history per turn.
- **Stateful Conversation (server-side state)** — args: `previous_response_id` / `previous_interaction_id` / `client.chats.create()` contents — server rehydrates prior context; only send new turn.
- **Persistent Conversations API** — args: owner, conversation ID, append-only new turn — `POST /v1/conversations` — persistent conversation object across sessions/devices; CRUD on stored conversations.
- **Encrypted Reasoning Replay** — args: `store:false` + encrypted reasoning blobs / unmodified `thinking` blocks / `reasoning_details` — stateless but preserve reasoning across turns.
- **Mid-conversation System Messages** — args: `{"role":"system"}` placed inside `messages[]` at relevant point — inject system instruction mid-conversation preserving cache.
- **Stored Responses CRUD** — `GET /v1/responses/{id}` (retrieve); `DELETE /v1/responses/{id}` (delete); conversations API read/modify (owner-only).

### Module: Conversation State Capabilities
- See S.18 (reasoning replay conventions).
- Output-structure handling: multi-item `output[]`; iterate `steps` for interleaved text/image/tool; chunk-list vs plain-string `content`.
- Context-window overflow handling: input-only vs input+max_tokens; `stop_reason:"model_context_window_exceeded"`.
- Prompt caching / context compaction for long conversations.

### Module: Classical NLP Analysis
- **NLP Analysis Job** — args: `documents[]`/`conversations[]`/`documents` (Blob Storage), `tasks[]` with `kind` values, `stringIndexType`, `piiCategories`, `confidenceScoreThreshold`, `redactionSource`, `includeAudioRedaction` — `:analyze-text` (sync) / `:analyze-text-job` (async) — unified NLP analysis endpoint dispatching by task `kind`.
- **Custom Question Answering — query-knowledgebases** — args: deployed KB ID, `query`, metadata filters, multi-turn `dialog`/`prompts` — returns answer + confidence from deployed Q&A KB.
- **Custom Question Answering — query-text** — args: `query` (prebuilt, no project) — returns answer + `answerSpan`.
- **Orchestration Workflow** — args: `projectKind:Orchestration`, utterance — top-level dispatcher routing utterances to CLU/CQA sub-projects.
- **NLP via Generative LLMs** — args: prompt + structured-output schema — perform classification/NER/PII/summarization/sentiment/QA/intent/key-phrase via prompting.

### Module: NLP Task `kind` Catalog (capabilities of NLP Analysis Job)
- `LanguageDetection` (ISO 639-1, script, country hint, confidence).
- `EntityRecognition` (Person/Org/Location/DateTime; category/subcategory; `inclusionList`/`exclusionList`; sync + async batch).
- `CustomEntityRecognition` (trained on labeled data; authoring + runtime).
- `PiiEntityRecognition` (characterMask/noMask/entityMask/syntheticReplacement; `piiCategories`, `confidenceScoreThreshold`, `disableEntityValidation`, `excludeExtractionData`).
- `ConversationalPIITask` (text/transcript modality; `redactionSource`; `includeAudioRedaction`).
- Document PII (native PDF/DOCX; blur image redaction).
- `Healthcare` (biomedical NER + relation + UMLS linking + assertion + FHIR + SDOH).
- `SentimentAnalysis` (doc + sentence sentiment; aspect-based opinion mining targets/assessments; `isNegated`).
- `KeyPhraseExtraction`.
- `EntityLinking` (Wikipedia disambiguation).
- `ExtractiveSummarization` (`rankScore`, `sentenceCount` 1-20, `sortby` Offset/Rank).
- `AbstractiveSummarization` (`summaryLength` oneSentence/short/medium/long; query-focused).
- `ConversationalSummarizationTask` (issue/resolution/chapterTitle/narrative/recap/action items).
- Document summarization (native files via Blob Storage).
- `CustomSingleLabelClassification` / `CustomMultiLabelClassification`.
- `Conversation` (CLU — intents + entities; Standard English / Advanced multilingual; `multilingual` flag; `confidenceThreshold`).
- Legacy NLP input encoding (`stringIndexType` Utf16CodeUnit / TextElement_V8); conversation item fields (lexical/itn/maskedItn + `audioTimings[]`).
- (Some kinds have deprecation dates 2028-2029 — flagged.)

### Module: Custom NLP Training
- **Import Project** — args: project schema + labeled data (Blob Storage) — `POST :import` — create custom NLP project.
- **Train Model** — args: `modelLabel`, `trainingConfigVersion`, `evaluationOptions` split % — `POST :train`.
- **Deploy Model** — args: `trainedModelLabel` — `PUT deployments/{deploymentName}` — deploy trained model; swap deployments test↔prod.
- **Evaluate** — args: train job ID — view evaluation metrics included in training job.
- **Runtime Query** — args: `projectName`, `deploymentName`, input utterance — query deployed custom model.
- **Authoring API (analyze-text projects)** — `/language/authoring/analyze-text/projects/{name}/...` — CRUD for Custom NER, Custom Text Classification.
- **Authoring API (analyze-conversations projects)** — `/language/authoring/analyze-conversations/projects/{name}/...` — CRUD for CLU, Orchestration.
- **Authoring API (query-knowledgebases projects)** — `/language/authoring/query-knowledgebases/projects/{name}/...` — CRUD for CQA.

### Module: Custom NLP Training Capabilities
- Project lifecycle (import → train → evaluate → deploy → swap → runtime query).
- Authoring API split (analyze-text / analyze-conversations / query-knowledgebases).
- Deployment expiry (~18 months after training-config version).

## Domain L3.B — Images & Video

### Module: Image Generation
- **Image Generation** — args: `prompt`/`text_prompt`/`input`/`json_prompt`, `model`, `size`/`aspect_ratio`/`resolution`, `quality`/`image_size`/`rendering_speed`/`test_time_scaling`, `n`/`num_images`, `output_format`/`response_format`, `background`, `seed`/`random_seed`, `negative_prompt`, style surface (see Style & Asset Management), `custom_model_uri`, `character_reference_images`+`_mask`, `controls`, `guidance`, `steps`, `disable_pup`/`prompt_upsampling`, `moderation`/`safety_tolerance`, `enable_copyright_detection`, `finetune_id`+`finetune_strength`, `raw`, `image_prompt`+`image_prompt_strength`, `thinking_level`, `previous_response_id`/`image_generation_call` ids, `action:auto|generate|edit`, `input_fidelity`, `postprocessing`, `aspect_ratio:auto`, `storage_options`, `partial_images`/`stream` — `POST /v1/images/generations` / unified Interactions / `image_generation` chat tool / `POST /v1/image/create` — text→raster image (and conversational multi-turn variant).
- **Vector Image Generation** — args: `model: <vector-variant>`, vector styles — `/v1/images/generations/vector` — text→SVG; rejects raster.
- **Transparent-Background Image Generation** — args: `prompt`, `upscale_factor:X1/X2/X4`, `aspect_ratio` — `/v1/images/generate-transparent` — die-cut stickers/logos; transparent output.

### Module: Image Generation Capabilities
- Conversational / multi-turn (`previous_response_id`, `image_generation_call` ids, `action:auto|generate|edit`).
- Streaming / partial images (`partial_images` 0-3, +100 output-image tokens each).
- Interleaved text & image output (iterate `steps` → `content[]`).
- Transparent output via `background:transparent` / `removeBackground` postprocessing.
- Determinism (`seed`/`random_seed`, pinned model versions, `test_time_scaling` 1-15).
- `aspect_ratio: auto` (model selects).
- Native audio generation (video only — see Video).
- Prompt enhancement (`revised_prompt`, `magic_prompt`, `prompt_upsampling`, auto-enhanced, `thinking_level`).
- Negative prompt.
- Reference-image tagging (`<img>N</img>`, `<IMAGE_N>`, "person of image 1…", character name verbatim).
- Grounding / real-time info (search tool).
- Moderation & safety params (`moderation:auto|low`, `safety_tolerance` 0-5/0-6, `enable_copyright_detection`, `personGeneration`).
- Watermarking (invisible watermark).
- Access gating (Organization Verification; uploaded-video editing gating; human-likeness characters blocked by default).
- Content policy (under-18 only; no copyrighted chars/music; no real people; no input human faces).
- Generation parameter union (see S.32 Image Generation Request).

### Module: Image Editing
- **Image Edit** — args: `prompt`/`edit_instruction`, `image`/`input_image`/`images`, `mask`, `strength`, `image_weight`, plus all generation params; multi-image compositing (≤N refs); supports all edit operation kinds — `POST /v1/images/edits` / Interactions / `POST /v1/image/edit` / `/v1/images/imageToImage` / `/v1/edit` / `/v1/images/remix` — prompt-based image editing + compositing.

### Module: Image Edit Operation Variants (capabilities)
- Prompt-based editing (single image).
- Multi-image compositing (variable ref limits: ≤3 / ≤10 / ≤8 / 1-6 / ≤14 by surface).
- Inpainting (mask alpha / B/W black=edit / grayscale white=modify; `guidance` 1.5-100).
- Outpainting / border extension (`expand_left/right/top/bottom` 0-4096; `top/bottom/left/right` 0-2048; target `size`/canvas + `reference_offset_x/y` + `auto_crop`; `zoom_out_percentage` 0-100; reframe square→target aspect preserving focal point; outpainting mode `high|fast`).
- Background removal (`remove-background`, `removeBackground`, postprocessing `remove_background`).
- Background replace (auto-detect subject).
- Background generate (mask specifies fill regions).
- Object removal / erase (`erase-v1`/`eraseRegion`, `dilate_pixels` 0-25, content-aware fill).
- Deblur (no prompt, no mask, fixed LoRA).
- Virtual try-on (VTO) (`person`→`input_image`, `garment`→`input_image_2`).
- Restyle / relight (source + style/mood/lighting prompt).
- Remix / variate / explore (`/v1/image/remix` 1-6 refs + `<img>N</img>`; `/v1/images/remix` with `image_weight`; `variateImage` no-prompt; `explore` grid; `explore/similar` with `similarity` 1-5).
- Context-aware editing (Kontext-style).
- Typography / text-rendering specialization.
- Legacy image variations (promptless variant generation).

### Module: Layout Composition
- **Layout-Aware Create** — args: text/refs (typed `Layout` regions) — `create_layout` / `create` v2 — returns image + echoed `layout`.
- **Layout-Aware Edit** — args: layout + text + typed `LayoutCommand`s (`op:add|shift|remove|place|keep|change`, `at`/`to` Bbox/Point, `image_index`, `new_description`, `label`/`description`) — `edit_layout` / `edit` v2 — returns image + echoed layout.
- **Layout Render** — args: `Layout` — `render` — layout → image; echoes layout.
- **Image-to-Layout Extraction** — args: `image`, `include_bbox` — `image_to_layout` / `/v1/images/describe` — reverse-engineer `Layout`/`V4JsonPrompt` from image.

### Module: Layout Capabilities
- Structured (JSON) prompts — `V4JsonPrompt` (structured-description style), `Layout` (region-based style), JSON prompting (subject/background/lighting/style/camera_angle), `text_layout` (per-word 4-point polygon).
- Region types & hierarchy (`coarse_detail`/`medium_detail`/`fine_detail`/`text`/`hand`/`face`; parent/child).

### Module: Image Understanding (Vision / Analysis)
- **Image Understanding — Generative Multimodal Vision** — args: `input_image`, prompt — chat-based vision analysis (description, Q&A, structured describe).
- **Image Annotation Batch** — args: batch of `AnnotateImageRequest`s with `features` enum, `languageHints` — `images:annotate` / `imageanalysis:analyze` — returns labels+faces+OCR+objects+safe-search simultaneously.
- **Custom Vision Training (classification / object detection)** — args: project, labeled images, tags/regions, train, publish — train-your-own classifier/detector; export ONNX/TF/CoreML/Docker. [Deprecated 2025-2028.]

### Module: Image Understanding Feature Catalog (capabilities of Annotation Batch)
- `LABEL_DETECTION` / `Tags` / `<CAPTION>` / classification (text-prompted) / classification project / structured-output JSON.
- Object detection — open-vocabulary (Grounding DINO `query`/`box_threshold`/`text_threshold`, YOLO-World `class_names`/`score_thr`/`nms_thr`, OWL-ViT, Florence-2 `<OD>`), `OBJECT_LOCALIZATION` (NormalizedVertex 0-1), `Objects` (XYWH pixels), Custom Vision OD, structured detection (`[ymin,xmin,ymax,xmax]` 0-1000 + `mask` polygon + `label`).
- Semantic & instance segmentation (Semantic-Segment-Anything, Mask2Former ADE20k, SAM-2 points/boxes, Grounded SAM, polygon masks, specialized clothing/hair/salient; mask encodings PNG/COCO RLE/polygon; quality scores `predicted_iou`/`stability_score`).
- Image captioning & dense captioning (`<CAPTION>`/`<DETAILED_CAPTION>`/`<MORE_DETAILED_CAPTION>`, `Caption` gender-neutral, `denseCaptions` up to 10, `<DENSE_REGION_CAPTION>`, caption-to-phrase grounding `<CAPTION_TO_PHRASE_GROUNDING>`, structured describe round-trip `/v1/images/describe`→`V4JsonPrompt`, layout extraction `image_to_layout`).
- OCR (`TEXT_DETECTION`, `DOCUMENT_TEXT_DETECTION` hierarchical, `Read` sync, Document Intelligence Read async, `<OCR>`, `<OCR_WITH_REGION>` quad 8 coords, `/v1/images/layerize-text` text-layer).
- Face & pose detection (`FACE_DETECTION` ~30 landmarks 3D, rollAngle/panAngle/tiltAngle, emotion Likelihood; MediaPipe face mesh / pose; dedicated Face API).
- Object tracking in video (zero-shot SAMURAI SAM-2 + motion-aware memory COCO RLE per frame; YOLO-World video mode; SAM-2-video) — cross-listed under Video.
- Specialized detectors (`LANDMARK_DETECTION`, `LOGO_DETECTION`, `WEB_DETECTION`, `SAFE_SEARCH_DETECTION`, `IMAGE_PROPERTIES` dominantColors, `CROP_HINTS`, `PRODUCT_SEARCH`, NSFW ViT).
- Multi-feature batch annotation (one call runs batch; async `files:asyncBatchAnnotate` up to 2000 files to Cloud Storage).
- Confidence/likelihood reporting (see S.15).

### Module: Image Format Conversion
- **Image Vectorization** — args: `file`/`image_url` — `/v1/images/vectorize` — raster → SVG (deterministic, no model).
- **Text Layer Extraction** — args: `image`, optional `prompt` — `/v1/images/layerize-text` — returns `base_image_url` + `text_blocks[]` (role/text/geometry/font/color/alignment).
- **Layout Extraction (reverse)** — `image_to_layout`, `describe` (also in Layout Composition).

### Module: Image Postprocessing
- **Image Upscale — Crisp** — args: `file`/`image_url` — `/v1/images/crispUpscale` — interpolation sharpening preserving content.
- **Image Upscale — Creative** — args: `file`/`image_url`, optional `prompt`, `resemblance`/`detail`, `magic_prompt_option` — `/v1/images/creativeUpscale` / `/upscale` (guided) — regenerates finer details/faces.
- **Effect Apply** — args: `effect_name`, `effect_parameters {filterId:{uniformId:value}}`, `source` filter for list — postprocessing `effect` / `GET /v1/image/effect` — apply named visual filter.
- **Fit Image (scale-down)** — args: `max_dim`/`max_width`/`max_height` — postprocessing `fit_image` — free scale-down preserving aspect ratio.
- **Smart Cropping** — `CROP_HINTS`, `SmartCrops`.

### Module: Style & Asset Management
- **Custom Style Creation** — args: up to 5 reference images — `POST /v1/styles` — returns reusable UUID style.
- **Character Reference Asset Creation (Video)** — args: short MP4 + name — `POST /v1/videos/characters` — reusable non-human character asset; mention name in prompt.
- **Prompt Enhancement** — args: prompt text (≤2000 chars for `/enhance`) — `/v1/prompts/enhance` / `/v1/images/magic-prompt` — returns enhanced/structured prompt.

### Module: Style & Asset Capabilities
- Curated style names (`style`/`style_type`).
- Style presets (~60 named).
- Style codes (8-char hex, shareable).
- Style reference images.
- Custom style (`POST /v1/styles`, ≤5 refs, reusable UUID).
- Color palette (`controls.colors` / `color_palette` preset or hex+weight).
- Structured style description (aesthetics/art_style/lighting/medium/photo).
- LoRA finetune (`finetune_id`+`finetune_strength` 0-2).
- Custom model (`custom_model_uri` trained 15-100 assets).
- Custom Vision train-your-own (deprecated).
- Style exclusivity rules (`style_codes` not combinable; `color_palette` not mixable; `resolution` vs `aspect_ratio` mutual exclusion).
- Style compatibility (V4/V4.1 ignore style params).
- Character reference management (`character_reference_images` + `_mask`, character ref parts, reusable `characters` assets for video, locked character consistency).
- Prompt enhancement (`revised_prompt`, `prompt_upsampling`/`disable_pup`, `magic_prompt`/`/v1/images/magic-prompt`, `/v1/prompts/enhance`, auto-enhanced, `thinking_level`).
- Negative prompts.
- Plain-text prompt length limits.
- Reference-image tagging (`<img>N</img>`, `<IMAGE_N>`, role descriptions, character name verbatim).
- Grounding / real-time info (`google_search` tool, FLUX.2 [max] grounding).

### Module: Files & Async (image/video) — capabilities
> Files API and async job lifecycle are in L0; L3 documents the modality-specific facets.
- Input file methods (URL / base64 / file_id / multipart / hosted blob / project ref / cloud storage).
- Output persistence (`storage_options`, `file_output`, public URL).
- Ephemeral URL expiry.
- Async lifecycle (per-endpoint submit→poll/webhook state machines).
- Response delivery modes (sync / streaming / async / webhook) — see S.14.
- Response shape (image URL / base64 / raw bytes via `Accept`).
- Response headers & metadata (request-id, content-violation, credits-used/remaining, moderation_details).
- Input image formats & limits (union list — see S.12).
- Vision input detail / resolution control (`detail: low|high|original|auto`, `media_resolution`).
- Multi-image handling (per-surface limits).

### Module: Video Generation
- **Video Generation** — args: `prompt`, `model`, `size`/`aspect_ratio`/`aspectRatio`, `seconds`/`durationSeconds`/`duration`, `resolution`, `seed`, `personGeneration`, `input_reference`/`image`, `lastFrame`, `referenceImages`/`reference_images`+`<IMAGE_N>`, `characters`, `storage_options` — `POST /v1/videos` / `models.generate_videos` (predictLongRunning) / `POST /v1/videos/generations` — text→video.
- **Image-to-Video** — args: `input_reference`/`image` (first frame, must match `size`) — image → video.
- **Last-Frame Interpolation** — args: `image` + `lastFrame` — generate video transitioning between first and last frame.
- **Reference-to-Video** — args: `reference_images[]` + `<IMAGE_N>` placeholders / `referenceImages[]` with `reference_type` — multiple reference images guide video.
- **Batch Video Generation** — args: JSON-only batch, `input_reference` with `file_id`/`image_url`, `custom_id` — `POST /v1/batches` targeting `/v1/videos`.

### Module: Video Generation Capabilities
- Character assets (reusable non-human, up to 2 per video, mention name).
- Native audio generation (always-on per model; silent-only per model; prompt audio cues).
- Video generation parameter union (size/aspect/seconds/duration/resolution/seed/personGeneration).
- Video output (MP4, thumbnail, spritesheet, persisted file).
- Video input preprocessing (first-frame match, image→first-frame default aspect, `lastFrame`, `referenceImages`, video tracking input).

### Module: Video Editing, Extension & Interpolation
- **Video Editing** — args: video {id}/{url}/{file_id}, `prompt` (one focused change) — `/v1/videos/edits` — edit existing video.
- **Video Extension** — args: source video, `prompt`, `seconds`/`duration` (extension portion) — `/v1/videos/extensions` / `generate_videos` with `video` param — extend video duration.
- **Video-to-Video / Audio-to-Video (multimodal director)** — args: prompt with `@Image`/`@Video`/`@Audio` tags, source assets; modes T2V/I2V/V2V/A2V — seamless scene extensions with auto audio-visual sync.

### Module: Video Editing Capabilities
- One focused change per edit; resolution/duration caps.
- Extension up to N seconds per extension; up to M total.

### Module: Image & Video Model Catalog & Versioning (capabilities)
- Image generation model tiers (flagship / default / fast).
- Video generation models (variable model catalog).
- Model versioning & deprecation (see S.2, S.3).

### Module: Moderation & Safety (image/video)
- See L5 Moderation / Safety Filters family for the engine; L3 documents the **attachment point** and image/video-specific tolerance/gating parameters.
- Safety tolerance (`safety_tolerance` 0-5/0-6), `moderation` (auto/low), `content_violation`, `respect_moderation`, `is_image_safe`, `enable_copyright_detection`, `personGeneration` (`allow_all`/`allow_adult`/`dont_allow`), `SAFE_SEARCH_DETECTION`, NSFW ViT, SynthID watermark.
- Access gating & content policy (under-18, no copyrighted, no real people, no human faces).

### Module: Billing Units (image/video)
> Billing is in L0.I; L3 documents the **modality-specific billing units**.
- Credits, per image, per output-image token, per patch/tile, per second, per request, surcharges (see S.31).

## Domain L3.C — Voice

### Module: Voice Asset Management
- **Voice Library Browse & Search** — args: filter by language/gender/country, `q` search, `expand[]=preview_file_url` — list/search voices.
- **Find Similar Voices** — args: source voice — `/voices/find-similar` — returns similar voices.
- **Instant Voice Cloning (IVC)** — args: short audio sample (+ consent recording for some) — returns `voice_id`.
- **Professional Voice Cloning (PVC)** — args: samples, train, verify captcha — multi-step clone with speaker separation; `/fine-tunes/create`.
- **Voice Design from Text** — args: 20-1000 char description — `/text-to-voice/design` — returns 3 preview voices; create from preview.
- **Voice Remixing** — args: existing voice + natural-language attribute transforms — `/text-to-voice/remix` — transform voice attributes maintaining recognizability.
- **Voice Localization** — args: source voice, target language/dialect — `/voices/localize` — adapt voice to new language.
- **Voice Metadata CRUD** — args: `voice_id`, name/description/gender/settings, share-to-library — manage voice assets.
- **Pronunciation Dictionary CRUD** — args: items (phonetic rules / `{text, pronunciation}` / language-specific rewrite rules), versioning, attachment — versioned text-to-pronunciation mappings.
- **Music Datasets & Fine-Tuning** — args: non-copyrighted tracks (5-10 min) — music model fine-tune capturing instrumentation/tempo/production.

### Module: Voice Asset Capabilities
- Prebuilt voice library (filter by language/gender/country; search; preview audio URLs).
- Pronunciation dictionary locator convention (see S.30).
- Voice metadata fields (settings: stability/similarity/style/speaker_boost/etc.).
- Voice sharing to library.

### Module: Audio Preprocessing
- **Voice Isolator** — args: audio (max 500 MB / 1 hour) — `/audio-isolation` + `/stream` — buffered + streamed noise removal.

### Module: Audio Preprocessing Capabilities
- Audio format detection & encoding (compressed/uncompressed/telephony/video/raw PCM).
- Sample rate configuration (8/16/22.05/24/32/44.1/48 kHz).
- Multichannel handling (independent channel transcription; `separate`/`combined`; channel count).
- Voice isolation / noise removal (`/audio-isolation` buffered + streamed; `remove_background_noise`; speaker separation; music stem separation).

### Module: Speech-to-Text / Audio Understanding
- **Batch Transcription (STT)** — args: audio file/URL, `language`/`language_code`, `diarize`, `keyterms`/`prompt`/`keyterm`/`keywords`, `smart_format`/`punctuate`/`paragraphs`/etc., `entity_detection`/`redact`, `timestamp_granularities`, `tag_audio_events`, `summarize`/`sentiment`/`topics`/`intents`, `webhook` — `/speech-to-text` / `/audio/transcriptions` / Audio Understanding / `/stt` / `/v1/listen` Pre-Recorded / `/post/speech/asr` — file-based transcription with full option suite.
- **Real-Time Streaming Transcription** — args: streaming audio, VAD/endpointing config, `is_final`/`speech_final`, KeepAlive, manual commit — WebSocket STT — realtime transcription.
- **Forced Alignment** — args: audio file + plain string transcript — `/forced-alignment` — word/phrase timestamps (29 languages).
- **Read API (STT intelligence on text)** — args: text content — `/v1/read` — apply intelligence features to text.

### Module: STT Capabilities
- Batch transcription (file-based; max sizes/durations; URL/video input).
- Real-time streaming (WebSocket; interim vs final; `is_final` vs `speech_final`; VAD events; KeepAlive; manual commit).
- Language detection & hinting (`language_probability`; ISO 639-1/BCP-47 hint; code-switching/multilingual; auto-detect with candidate list).
- Diarization (up to 32 speakers; `diarization_threshold`; `num_speakers` hint; `detect_speaker_roles` agent/customer; `use_speaker_library`; `known_speaker_names[]`; `known_speaker_references[]`; structured `speaker` field; `diarize_model` versions; `utt_split`).
- Vocabulary boosting / keyterm prompting (`keyterms` up to 1000/50; `prompt` 224 tokens; `keyterm` Nova-3/Flux; `keywords` legacy `term:boost`; `search`).
- Text normalization & formatting (`smart_format`, `punctuate`, `paragraphs`, `numerals`, `dictation`, `measurements`, `profanity_filter`, `filler_words`, `no_verbatim`, `apply_text_normalization`, `apply_language_text_normalization` Japanese, `rewrite_rules`, `search`+`replace`).
- Entity detection & redaction (`entity_detection` PII/PHI/PCI/offensive/names/orgs/dates; `detect_entities` NAME/PHONE/EMAIL/ORG/CARDINAL; redaction modes `redacted`/`entity_type`/`enumerated_entity_type`; `redact` pci/pii/numbers/ssn/aggressive_numbers).
- Timestamps (word/character/phoneme/segment; MM:SS prompt-based).
- Audio event tagging (`tag_audio_events`; laughter/applause/footsteps).
- Audio intelligence post-transcription (`summarize`, `sentiment` per segment + average, `topics` + `custom_topic` up to 100, `intents` + `custom_intent`, `detect_entities`, emotion via structured output, Read API on text).
- Turn detection (VAD) at STT layer — silence-based VAD (`server_vad`/automatic/`endpointing` with threshold/padding/silence), semantic VAD (`semantic_vad`/`step` messages with `horizon_s`/`inactivity_prob`; `eagerness` auto/low/medium/high), model-native turn detection (Flux v2/Ink-2 with StartOfTurn/Update/EagerEndOfTurn/TurnResumed/EndOfTurn lifecycle), manual/push-to-talk (`turn_detection:null`/manual/`finalize`/`flush`), adaptive delay (`delay_in_frames` 0-80; `delay` minimal/low/medium/high/xhigh), turn-detection config params.
- Export formats (SRT configurable, TXT, DOCX, HTML, PDF, segmented JSON, verbose_json, VTT, diarized_json, NDJSON).
- Domain-specific STT models (medical, meeting, finance, phonecall, voicemail, video, drivethru, automotive, conversationalai).
- Webhooks / async callbacks (`webhook=true` + `request_id`/`transcription_id`; `callback` URL; call-event webhooks; dubbing job-completion webhook; post-call analysis webhook).

### Module: Translation & Dubbing
- **File-Based Audio Translation** — args: audio (25 MB) — `/audio/translations` — output always English text (variant).
- **Translating Transcription (STT with translation output)** — args: `language`, `target_language` — `stt-translate` model — transcript in target language.
- **Batch Dubbing** — args: source (YouTube/X/TikTok/Vimeo/URL/upload), `cloning_strength`, `preserve_original_voices`, `keep_background_audio`, target language (90+) — `/dubbing` — translate audio/video preserving emotion/timing/tone/speaker.
- **Live Speech-to-Speech Translation** — args: streaming audio, `target_language`/`translationConfig`, `voice_id` — `gpt-realtime-translate` / `gemini-3.5-live-translate-preview` / `s2s-translate` (WebSocket) — real-time interpretation session.

### Module: Translation & Dubbing Capabilities
- Batch dubbing (90+ languages; speaker separation; sources YouTube/X/TikTok/Vimeo/URL/upload; up to 9 speakers; 5 concurrent self-serve / 100 Enterprise).
- Live S2S (continuous WebSocket; one session per target language; listen-along vs conversational patterns; `echoTargetLanguage`; ephemeral tokens).

### Module: Text-to-Speech Generation
- **Text-to-Speech (TTS)** — args: `text`/`transcript`/`input`, `model`, `voice`/`voice_id`, `language`, `output_format`, `voice_settings`/`generation_config`/`instructions`, pronunciation dictionary locators, `seed`, context stitching params, `optimize_streaming_latency`, `enable_logging`, `speed`, `stream`, `use_pvc_as_ivc` — single-speaker TTS.
- **Multi-Speaker TTS / Dialogue** — args: `inputs[]` with text + `voice_id` per turn / `speech_config` speaker+voice pairs — generate multi-speaker audio.
- **WebSocket TTS Control** — args: `flush:true`, `cancel:true` on context, `context_id` — force emission / cancel in-flight generation.

### Module: TTS Capabilities
- Voice behavior control (stability, similarity/cfg_coef, style, speaker boost, speed, volume, emotion, temperature/seed, instructions, accent, whispering).
- Inline audio directives (audio tags `[sad]`/`[laughing]`/`[whispering]`/etc.; SSML `<speed>`/`<volume>`/`<emotion>`/`<break>`/`<spell>`; in-text `<flush>`/`<break time>`).
- Context stitching & continuations (`previous_text`/`next_text` or `previous_request_ids`/`next_request_ids` up to 3; WebSocket `context_id` + `continue:true/false`; whitespace insertion).
- Buffering control (managed `max_buffer_delay_ms` default 3000; custom `max_buffer_delay_ms:0`).
- Multiplexing (multi-context WebSocket; only generation time counts; `close_ws_on_eos:false` + `client_req_id`).
- Timestamps in TTS output (character-level `with-timestamps`; word-level `add_timestamps` + phoneme + `use_normalized_timestamps`; `start_s`/`stop_s`; metadata headers).
- Determinism (`seed` 0-4294967295; `temp:0.0`; pinned model IDs).
- Free regenerations (up to 2 identical params).

### Module: Voice Transformation
- **Voice Changer (STS, no translation)** — args: `audio`, `voice_id`, `remove_background_noise`, `clip`, `output_format` — `/speech-to-speech/{voice_id}` + `/stream` / `/voice-changer/bytes` + `/sse` — transform input audio to target voice.
- **Audio Infill / Bridging** — args: `left_audio`, `transcript`, `right_audio` — `/infill/bytes` — bridge audio across a transcript.
- **Stem Separation** — args: song — `/music/separate-stems` — separate into instrument/vocal stems.

### Module: Sound Effects & Music Generation
- **Sound Effects Generation** — args: `text` description, `duration_seconds` 0.1-30s, `prompt_influence`, `loop` — `/text-to-sound-effect` — text→sound effect.
- **Music Composition** — args: text prompt, fine-tune, composition plan JSON — `/music/compose` (+ `/stream`, `/compose-detailed`, `/create-composition-plan`, `/video-to-music`, `/upload`) — text→music.

### Module: Sound & Music Capabilities
- Sound effects (`prompt_influence` high=literal/low=creative; `loop` seamless; output MP3/WAV 48 kHz).
- Music models (`music_v2` next-gen / `music_v1`).
- Music features (composition plan, inpainting edit/combine sections, stem separation, multilingual, vocals/instrumental, 3s-5min, MP3/WAV, commercial cleared).

### Module: Output Formatting & Delivery (voice)
> Files/async conventions in L0/Standards; L3 documents voice-specific output formats.
- Output formats (mp3/pcm/wav/opus/μ-law/a-law/aac/flac/raw; PCM f32le for Web Audio).
- Streaming delivery modes (buffered HTTP, HTTP chunked, WebSocket bidirectional, SSE, WebRTC).
- Concurrency management (HTTP per-request; WebSocket only generation time; TTS/STT separate limits; multiplexing; heuristic 5→~100 broadcasts).
- Cost-tracking headers (`character-cost`, `request-id`, `x-trace-id`, concurrency headers, `dg-*`).

### Module: Conversational Voice Agent Orchestration
- **Conversational Voice Agent Session** — args: unified realtime session config (model, voice, instructions, modalities, audio formats, turn_detection, tools, thinking) — `WS /agent/converse`, `WS /speech-engine`, `WS /realtime`, `WS /realtime/translations` — end-to-end / BYO-LLM / chained-pipeline voice agent.
- **Agent Configuration CRUD** — args: agent config (voice, LLM, tools, telephony) — `POST /agent/configs`, `GET /agent/configs/{id}`, `POST /agents` — persisted agent definitions.
- **Outbound Call / Call Batch** — args: phone number, agent config — `POST /agents/calls/create-outbound`, `POST /agents/call-batches/create-call-batch` — telephony outbound.
- **Agent Documents / Knowledge Base Upload** — args: documents/folders (up to 100 bulk), metadata filters, top_k — `POST /agents/documents` — knowledge base for RAG in agent.
- **Agent Webhooks CRUD** — args: webhook URL, event filters — `POST /agents/webhooks` — register call-event webhooks.
- **Realtime Call / Secrets Issuance** — args: call config — `POST /realtime/calls`, `POST /realtime/client_secrets`, `POST /realtime/translations/calls`, `POST /realtime/translations/client_secrets` — issue ephemeral tokens for browser/mobile realtime sessions.
- **Phone Number Provisioning / Import** — args: US number provisioning, Twilio import — telephony number management.
- **Call Management** — args: list/get/cancel/delete calls, call audio download, runtime logs — manage placed calls.

### Module: Conversational Voice Agent Capabilities
- Architecture choices (end-to-end / BYO-LLM / chained pipeline / managed platform).
- Connection methods (WebSocket / WebRTC / SIP).
- Session configuration (unified config fields; advanced config — server-stored prompt, context seeding, sessionResumption, contextWindowCompression, inputAudioTranscription/outputAudioTranscription, proactivity, enable_affective_dialog, historyConfig, mediaResolution; response-level overrides).
- Turn-taking & VAD in sessions (`create_response`, `interrupt_response`, VAD disabled, VAD without auto-response).
- Interruption / barge-in (WebRTC/SIP auto-truncate; WebSocket `conversation.item.truncate` + transcript caveat; `serverContent.interrupted`; `UserTurnStarted` + `cancel_filter`; `UserStartedSpeaking`).
- Function calling in sessions (synchronous flow; asynchronous `behavior:NON_BLOCKING` + `scheduling` INTERRUPT/WHEN_IDLE/SILENT; client-side execution; server-side execution; tool cancellation `toolCallCancellation`; Line tool types Loopback/Passthrough/Handoff; Line built-in tools `end_call`/`transfer_call`/`web_search`/`knowledge_base`/`send_dtmf`; HTTP server tools from JSON schemas; Google Search grounding).
- Image & video input (multimodal sessions) (`input_image` + `image_url`; `send_realtime_input(video=Blob)` max 1 FPS; turn coverage).
- Session management (session resumption with handle 2h; context window compression sliding window; `GoAway` message; `generationComplete` vs `turnComplete`; session duration limits 60min/15min/2min).
- Advanced session features (thinking control `reasoning.effort:low`/`thinkingLevel`/`reasoning_mode`; proactive audio; affective dialog; out-of-band responses `conversation:"none"`; server-stored prompts; latency metrics; pre-call handler — configure voice, language, pronunciation, or reject calls before agent starts; return `None` to reject with 403; multi-agent handoffs; knowledge base / RAG up to 100 bulk).
- Telephony integration (provision US numbers; import Twilio; outbound + batch; call management; SIP; telephony formats μ-law/A-law).
- LLM provider support (built-in / 100+ via LiteLLM / any LLM BYO; `LlmConfig` params).
- Per-platform event systems (Deep client→server / server→client events; Cart Line call events + event filters).
- Unified realtime event reference (client→server and server→client event union — see S.30).
- Unified realtime session configuration object (see S.30).

### Module: Multimodal Audio Chat
- **Multimodal Audio Chat (Chat Completions)** — args: audio input/output content parts in chat messages — audio I/O in standard chat.

### Module: Voice Management / Observability
> Management APIs/observability plumbing in L0/L5; L3 documents voice-specific facets.
- Credit monitoring endpoints (`/usage/credits`, `/usage/agents`, `/manage/projects/{id}/balances`, `/manage/projects/{id}/usage`).
- Management APIs (projects, members, invitations, scopes, billing, usage, requests, API keys, models, model metadata, self-hosted credentials; agent management, phone numbers, providers, webhooks, documents, folders, metrics, deployments; voice management, pronunciation dictionaries, models listing).
- Call logs, evaluations (LLM-as-a-Judge), deployments (roll back), metrics export CSV.
- Latency metrics `total_latency`/`tts_latency`/`ttt_latency`.
- Self-hosted deployment (Docker/Kubernetes/SageMaker; on-prem).
- Partner integrations / migration guides.

### Module: Voice Language Coverage
- Voice language coverage summary (variable; 90+ for dubbing, 20 for localization, 29 for forced alignment, etc.).

## Domain L3.D — Documents

### Module: Document Ingestion & Storage (index time)
- **Workspace / Container CRUD** — args: type, residency, embedding model, chunking config, expiration, access_mode — `POST /v1/workspaces`, list/get/put/delete, `GET /v1/workspaces/{id}/documents`, `GET/DELETE /v1/workspaces/{id}/documents/{doc}`. (Workspace CRUD is partially L0; the document-listing sub-endpoints stay in L3.)
- **File Upload & Ingestion** — args: `file`/`file_url`/`base64_string`/`http_sources`, `workspace_id`, `filename`, `metadata`/`attributes`/`tags`, `external_id` idempotent, `parser`/`parsing_strategy`, `chunking_config`, `processing_location`, `purpose` — `POST /v1/files` (multipart|URL|base64), `/v1/files/uploads` resumable, `/v1/files/prechunked` — ingest file → indexed chunks.
- **File Management** — `GET /v1/files` (filter), `GET /v1/files/{id}` (metadata+status), `GET /v1/files/{id}/download` (original/rendered_pdf), `GET /v1/files/{id}/chunks`, `PATCH /v1/files/{id}` (metadata), `DELETE /v1/files/{id}`, `POST /v1/files/bulk-delete`.
- **Document Segmentation** — args: `segmentation_schema` (segment names+descriptions), `segmentation_strategy:document_boundary`, page-structure header-based — output segments with name/page ranges/confidence.

### Module: Ingestion Capabilities
- Workspace / container management (type, residency, embedding model, chunking config, expiration, access_mode).
- File upload methods (direct multipart / URL / base64 / presigned-URL / resumable ~1 TB / cloud-storage loaders / local FS batch / in-memory stream / pre-chunked MXJSON / Docling JSON round-trip / `datalab://` URI / object insertion).
- Idempotent ingestion (`external_id` deterministic UUID5).
- Async processing & status lifecycles (per-platform state machines).
- Webhooks + batch ingestion (up to 500 files / bulk / `convert_all` / concurrent).

### Module: Document Understanding (parse + extract; checkpoint-reusable)
- **Document Convert / Parse** — args: `mode`/`processing_mode` fast/balanced/accurate, `output_format` md/html/json/chunks/doctags/doclang/docx/pdf/png, `do_ocr`/`force_ocr`/`ocr_lang`/`ocr_preset`, `table_mode`/`table_format`, image extraction flags, `include_blocks`/`add_block_ids`, `confidence_scores_granularity`, `bbox_annotation_format`, `paginate`/`page_range`/`max_pages`, `word_bboxes`/`table_cell_bboxes`, `token_efficient_markdown`, `keep_pageheader_in_output`/`keep_pagefooter_in_output`, `extract_header`/`extract_footer`/`extract_links`, `jsonOptions`, `media_resolution`, `extras[]` (track_changes/chart_understanding/infographic), `enrichments[]` (code/formula/picture_classification/picture_description), `save_checkpoint`, `skip_cache`, `processing_location`, `webhook_url`, `eval_rubric_id` — `POST /v1/convert` (returns `202` + `request_id`), poll `GET /v1/convert/{request_id}` — parse + OCR + layout + tables + enrichments.
- **Dedicated OCR API** — args: document — `ocr-latest` — returns 13 block types with bboxes in reading order + confidence + markdown/images/tables/blocks.
- **Data Extraction** — args: `file`/`page_schema`/`schema_id`/`schema_version`/`extraction_mode` turbo|fast|balanced/`mode`/`output_format`/`page_range`/`save_checkpoint`/`skip_cache`/`webhook_url`/`docClass`/`jsonOptions` — `POST /v1/extract`, poll `GET /v1/extract/{request_id}` — JSON-schema-driven LLM extraction with citations + per-field verification + confidence.
- **BBox / Document Annotation** — args: `bbox_annotation_format`/`document_annotation_format`/`include_image_base64` — `POST /v1/annotate` — per-image / document-level structured annotation.
- **Schema Auto-Generation** — args: `checkpoint_id` — `POST /v1/gen-schemas`, poll `GET /v1/gen-schemas/{request_id}` — returns simple/moderate/complex candidate schemas.
- **KVP Extraction with Ontology** — args: key+value with coordinates, confidence, KeyClass tagging, validators; output tiers basic/detailed/verbose.
- **Table / Line-Item Extraction** — args: recursive `ComplexKVPStructure` nested rows/cells — nested tables.
- **Semantic Normalization** — args: values (names/addresses) — cleans/standardizes with `OriginalValue` preservation.
- **Verbatim Text Extraction** — args: line_number or regex strategy — `Extract` — pull source text without synthesis.
- **Form Filling** — args: PDF/image form, `field_data {name:{value,description}}`, `context`, `page_range`, `output_format:pdf|png`, `confidence_threshold` — `POST /v1/fill` — AcroForm + visual + image field detection.
- **Classification / Facets CRUD** — args: `content_type_path`, `attribute_name`/`value`, `class_match` threshold — `GET /v1/content-types`, `POST /v1/content-types` (adopt|define_content_type|undefine_content_type|define_attribute|undefine_attribute), `GET /v1/content-types/templates`, `POST /v1/files/{id}/facets` (classify|unclassify|set_value|clear_value), `GET /v1/files/{id}/facets`, `GET /v1/tags`, `POST /v1/tags` — hierarchical taxonomy + flat tags.
- **Checkpoint Reuse** — args: `checkpoint_id` from prior `save_checkpoint:true` conversion — pass to `/convert`, `/extract`, `/segment`, `/gen-schemas` to skip re-parsing.

### Module: Document Understanding Capabilities
- Document parsing modes (multi-model pipeline DocLayNet+Tesseract/Surya/RapidOCR+TableFormer; single end-to-end VLM GraniteDocling; native multimodal vision LLM; managed automatic; dedicated OCR API; word-level OCR with font metadata; format-specific backends; audio/video ASR; legacy office conversion).
- Output representations (Markdown per-page/full-doc with tables/lists/headings; HTML `data-block-id`; JSON/DoclingDocument hierarchical; DocTags compact; DocLang XML `.dclx`; pre-segmented chunks; paragraph-level blocks with bboxes; per-word/cell/list-item/block bboxes + confidence; confidence scores page/word/0-5; extracted images; structured tables).
- Enrichments (code language detection; formula LaTeX; picture classification chart/diagram/logo/signature; picture description/captioning; chart understanding/infographics; link extraction; track changes/redline; headers/footers; reading order).
- Data extraction modes (JSON-schema-driven LLM turbo/fast/balanced; BBox annotation; document annotation; KVP with ontology; table/line-item recursive; semantic normalization; verbatim extraction; form filling; schema auto-generation; Pydantic-template extraction; dense extraction two-phase).
- Output quality signals (per-field citations; per-field verification PASS/FAIL_UNRESOLVABLE/FAIL_FIX/FAIL_CITATIONS/ITEMS_MISSING; per-field confidence 1-5 + reasoning; KeyConfidence/ValueConfidence/KeyClassConfidence; extraction score; parse quality 0-5).
- Classification & categorization (AI document class with ClassConfidence/ClassMatch/AlternateDocumentClass; hierarchical Facets taxonomy max depth 4 with typed inheritable attributes + seed templates; flat tags; custom processors; LLM map with enum + calibration; filter-based; picture classification; zero-shot via embeddings).
- Checkpoint reuse (`checkpoint_id` from `save_checkpoint:true` reusable by extract/segment/gen-schemas).

### Module: Chunking & Enrichment (prepare for indexing)
- **Chunking Operation** — args: `max_chunk_size_tokens` (100-4096), `chunk_overlap_tokens`, `chunk_size` chars/words, `tokenizer`, separator/hierarchical/markdown/header-aware/hybrid/line-based modes, `merge_list_items`/`repeat_table_header`/`keep_separator`/`strip_whitespace`, `num_splits_to_group` — produce chunks with metadata.
- **Chunk Enrichment / Contextualization** — args: `propagate_summary_to_chunks`, `with_metadata`, `with_file_context`, Gather context windows, generated metadata, custom `ChunkEnricher` — enrich chunks at index time.

### Module: Chunking Capabilities
- Static/token-count (`max_chunk_size_tokens` 100-4096 default 800, `chunk_overlap_tokens` default 400).
- Character-count (`chunk_size` default 1000).
- Separator/hierarchical (paragraph/sentence with fallback).
- Markdown/header-aware.
- Hierarchical/structure-pure (`HierarchicalChunker`).
- Hybrid tokenization-aware (`HybridChunker`; table-header repetition; production default).
- Line-based (`LineBasedTokenChunker`).
- Word-count with overlap.
- Structure-preserving Docling-based.
- Automatic/managed.
- Pre-chunked bypass (MXJSON/MXJSONL).
- Gather context enrichment (previous/next head/middle/tail + header hierarchy).
- Chunk types (`text`/`image_url`/`audio_url`/`video_url`/`content`/`image_annotation`/`summary`).
- Metadata preserved on chunks (page_number, filename, filepath, start_offset/end_offset, images, chunk_id, chunk_index, headings, token_count, text_hash, char_length, locator formats).

### Module: Embedding, Indexing & Graph (build the retrievable store)
- **Embedding Generation** — args: `input` string|string[], `model`, `dimensions` (MRL), `encoding_format` float|base64 — returns `data[{embedding, index}]` + `usage`.
- **Vector Index Build / Object Insertion** — args: collection, named vectors, distance metric, index type, per-property index flags, vectorizer config — insert objects/embeddings into vector store. (Backend storage is L0; the indexing pipeline service stays in L3.)
- **Knowledge Graph Build** — args: `source` file_id|checkpoint_id, `template` Pydantic class, `processing_mode` one-to-one|many-to-one, `extraction_contract` auto|direct|dense, `dense_config`, `backend` llm|vlm, `inference` local|remote, `use_chunking`, `chunk_max_tokens`, `provenance` off|standard|detailed, `gleaning_enabled`, `parallel_workers`, `export_format` — `POST /v1/knowledge-graph/build` — output graph_id + stats.
- **Knowledge Graph Get / Export / Visualize** — args: graph_id, format — `GET /v1/knowledge-graph/{id}`, `GET /v1/knowledge-graph/{id}/export?format=`, `GET /v1/knowledge-graph/{id}/visualize`, `POST /v1/visualize/embeddings`.
- **Entity Resolution** — args: `data`, `comparison_prompt`, `resolution_prompt`, `blocking_keys`, `blocking_threshold`, `blocking_target_recall` 0.95, `blocking_conditions`, `embedding_model`, `cascade` {proxy_model, guarantee:recall, target, delta} — `POST /v1/resolve` — blocking + pairwise LLM + union-find clustering.
- **Equijoin (fuzzy join)** — args: `left`, `right`, `comparison_prompt`, `limits`, `blocking_keys`, `blocking_threshold`, `cascade` — `POST /v1/equijoin` — LLM-evaluated semantic join of two datasets.
- **Clustering** — args: `data`, `embedding_keys`, `summary_prompt`, `summary_schema`, `embedding_model` — `POST /v1/cluster` — hierarchical agglomerative / KMeans / Louvain / value sampling.

### Module: Indexing & Graph Capabilities
- Managed vector store auto-indexed.
- Self-hosted vector database (HNSW/Flat/Dynamic/HFresh; LSM-Tree; WAL+snapshots; lazy shards; async indexing).
- Vespa vector store (swappable schema).
- LanceDB local index (FTS/embedding/hybrid; no server).
- Native graph database (Neo4j; Bolt; ACID).
- File-based storage (CSV/JSON/Cypher/HTML).
- DPE database (persist until deleted).
- S3 integration; BYOB.
- Vector index types & quantization (PQ/BQ/SQ/RQ; re-scoring).
- Inverted index (roaring bitmaps; map index for BM25); per-property index flags.
- Lifecycle/cost control (`expires_after`/`expires_at`; TTL; result retention; raw file expiration).
- Consistency levels ALL/QUORUM/ONE.
- Replication + backups.
- Multi-tenancy (per-tenant sharding 50,000+ shards/node; tenant lifecycle).

### Module: Knowledge Graph Capabilities
- Schema-validated Pydantic-driven pipeline (deterministic provenance ledger; stable cross-batch node IDs; NetworkX DiGraph; export CSV/Cypher/JSON/HTML; extraction contracts direct/dense/auto; backends LLM/VLM; gleaning; dense dedupe).
- Schema-free LLM triple extraction (SPO; entity standardization; relationship inference rule-based + LLM; PyVis HTML with Louvain).
- Native graph database storage (Cypher CRUD; GraphRAG; MCP server).
- Link resolve (one-sided; embedding blocking + LLM).
- Cross-references (directional links across collections).
- Graph export formats (CSV/Cypher/JSON/HTML/Docling).
- Graph statistics (node_count/edge_count/node_types/edge_types/avg_degree/density).
- Provenance levels off/standard/detailed (with char-offset spans).

### Module: Entity Resolution Capabilities
- Blocking + pairwise LLM + union-find clustering + resolution (auto-computed blocking threshold 95% recall; union-find DSU).
- Deterministic node ID registry (fingerprint hashing; cross-document graph merging).
- LLM-based entity standardization.
- Dense dedupe (off/standard/aggressive).
- Link resolve.
- Equijoin fuzzy join.
- Cross-references.
- BARGAIN cascade (proxy model + oracle-labeled threshold learning with statistical guarantees).
- Hierarchical agglomerative clustering (binary tree; cluster path; LLM summaries).
- KMeans.
- Louvain community detection.
- Value sampling cluster.
- t-SNE visualization.

### Module: Query Time (read path)
- **Search (unified)** — args: `search_type` semantic|keyword|hybrid|agentic|grep|list, `query`, `top_k`/`limit`, `hybrid_search {embedding_weight, text_weight}`, `rerank`, `rewrite_query`, `agentic {max_rounds, queries_per_round, instructions, strict_top_k, score_threshold}`, `relevance_scoring`, `filters`, `content_type[]`/`attribute[]`/`tag_id[]`/`file_ids`, `target_vector`, `distance`/`certainty`, `auto_limit`/`autocut`, `move_to`/`move_away`, `selection {type:mmr, balance}`, `boost`, `group_by`, `return_properties`/`return_references`/`return_metadata` — `POST /v1/search` — unified semantic/keyword/hybrid/agentic/list search.
- **Grep (regex)** — args: `store_identifiers[]`, `pattern` RE2, `content_groups`, `case_sensitive`, `file_ids[]`, `filters` — `POST /v1/grep` — regex over literal chunk text.
- **List Chunks** — args: `store_identifiers[]`, `top_k`, `file_ids[]`, `sort_by`, `filters` — `POST /v1/list-chunks` — metadata-only retrieval (no embeddings).
- **Metadata Facets** — args: `store_identifiers[]`, `query`, `top_k`, `filters`, `facets[]` — `POST /v1/metadata-facets` — aggregate chunk counts grouped by metadata.
- **Query Agent Ask / Search / Ask-Stream** — args: `query`, `messages`, `collections [{name, target_vector, view_properties, tenant, additional_filters}]`, `result_evaluation`, `timeout`, `limit`, `filtering` recall|precision, `diversity_weight`, `include_progress`, `include_final_state` — `POST /v1/query-agent/ask`, `/search`, `/ask-stream` — LLM translates NL → database operations across collections.
- **Retrieval API (vector store search)** — `POST /v1/vector_stores/{id}/search` (see L0.F).
- **Aggregate** — args: `workspace_id`, `query`, `total_count`, `return_metrics` {count, sum, max, min, mean, median, mode, top_occurrences, percentageTrue, percentageFalse, reference_count}, `group_by`, `filters`, `distance`, `object_limit` — `POST /v1/aggregate` — aggregate queries + grouped search.
- **Rank** — args: `data`, `prompt`, `input_keys`, `direction:asc|desc`, `initial_ordering_method:likert|embedding`, `call_budget`, `k` — `POST /v1/rank` — full sorting by latent attribute.
- **Rerank (standalone second-stage)** — args: `model`, `top_k`, `with_metadata`, `prop`, `query` — cross-encoder / listwise / LLM / RRF / sequential / boost reranking. (In many platforms rerank is a parameter on Search; treat as standalone service when independently invocable.)
- **Semantic Cache** — args: cosine threshold (0.99 strict / 0.95 balanced / 0.90 permissive), eviction (LRU/LFU/FIFO), TTL — skip retrieval on hit.

### Module: Query Time Capabilities
- Query rewriting (`rewrite_query`; `LLMQueryRewriter`; `LLMQueryExtension`; model-generated queries).
- Lexical/keyword BM25/BM25F (token frequency + IDF; multi-field; per-property tokenization WORD/LOWERCASE/WHITESPACE/FIELD/TRIGRAM; property boosting; `and`/`or` + `minimum_match`; accent folding; stopword presets).
- Grep regex (RE2; `content_groups`; case sensitivity).
- Vector/semantic (`near_text`/`near_vector`/`near_object`/`near_image`; `VectorRetriever`; semantic via embeddings; move params `move_to`/`move_away` + force + concepts; MMR diversity; Autocut/auto_limit).
- Hybrid vector+keyword fusion (weighted `alpha`; `RELATIVE_SCORE`/`RANKED` fusion; RRF `embedding_weight`/`text_weight`; `RRFRanker` with `rrf_k`; hybrid vector+keyword+vision score breakdown; hybrid web+internal virtual web store).
- Agentic search (multi-round; sub-queries; `max_rounds`/`queries_per_round`/`instructions`/`strict_top_k`/`score_threshold`; Query Agent Ask/Search/Suggest modes with streaming + progress; model-autonomous File Search).
- List chunks (metadata-only, no embeddings/similarity/rerank; sort by metadata).
- Multi-store/federated search; cross-collection Explore GraphQL; cross-document reasoning up to 1000 pages.
- Filtering, facets & metadata scoping (three metadata layers; comparison filters; compound filters; AIP-160; operators incl. `within_geo_range`/`length`; cross-reference filtering `Filter.by_ref`/`by_ref_count`; nested object property filtering; content-type/facet filters; tag filters; file ID scoping; facets aggregate; ACORN vs sweeping filter strategy; attributes 16 keys/256 chars).
- Reranking (cross-encoder; listwise instruction-steerable; LLM `LLMReRanker`; RRF fusion; relevance scoring modes none/scoring_only/scoring_and_filtering; sequential chaining; boost/soft ranking).
- Caching (semantic cache thresholds 0.99/0.95/0.90; LRU/LFU/FIFO; TTL; metrics; result caching `skip_cache`; LLM call caching; memoized terminal actions; persistent media IDs; HNSW snapshots).
- Aggregations & grouped search (counts/stats sum/max/min/mean/median/mode, `top_occurrences`, `percentageTrue`/`percentageFalse`, reference counts; `GroupByAggregate`; `GroupBy` with `objects_per_group`/`number_of_groups`; reduce operator with scratchpad + value sampling; facets; rank operator with "picky window" O(n) scaling).

### Module: Generation & Output
- **RAG Ask (retrieve + generate)** — args: `query`/`input`/`messages`, `model`, `stream`, `instructions`, `cite`, `multimodal`, `single_prompt`/`grouped_task`/`grouped_properties`, `generative_provider`, `qa_options` — `POST /v1/ask`, `POST /v1/generate` — RAG with citations (file_citation / `<cite i="n"/>` / `{field}_citations` / source chunks).
- **Hosted RAG (model-autonomous File Search)** — args: file store attached as tool — model decides when to search; `file_citation` annotations.
- **Managed RAG (one-call Question Answering)** — args: `query`, `instructions`, `cite`, `multimodal`, `stream` — retrieve-then-generate in one call with `<cite>` markers.
- **Document QnA** — args: `document_url` content block, multi-document — Chat Completions with document content.
- **GraphRAG** — args: vector search + graph traversal patterns (vector+graph, agentic retrieval, entity-centric RAG, ontology-driven RAG) — multi-hop reasoning.

### Module: Generation & Output Capabilities
- Generative search (Single Prompt + Grouped Task; `{property_name}` interpolation; multimodal prompts; query-time model override).
- Hosted RAG (model-autonomous; `file_citation` annotations with char offsets).
- Managed RAG one-call (`<cite i="n"/>` → `sources[n]`; multimodal; streaming; `instructions`).
- Two-stage retrieve + generate (search then Ask; SSE OpenAI-compatible; source attribution).
- Retrieval API + manual synthesis (`<sources>` XML pattern).
- Document QnA (Chat Completions with `document_url`; multi-document).
- RAG via retriever + map/reduce (`{{ retrieval_context }}`).
- GraphRAG (vector + graph; agentic retrieval; entity-centric; ontology-driven).
- Query Agent Ask mode (NL → answer + sources across collections).
- Structured grounded output (JSON Schema + file search).
- Managed RAG libraries (upload → ingest/vectorize/search → `document_library`/`file_search` tool on agents → grounded answers with `tool_reference` citations; context caching).
- RAG from scratch (load → split → embed → FAISS → retrieve → prompt → complete; HyDE; child/parent chunks; time-weighted; "lost in the middle" reordering; metadata filtering; BM25; few-shot).
- Citation mechanisms (`file_citation`; `<cite>`; `{field}_citations`; source chunks; char-offset annotations).
- Streaming (SSE OpenAI-compatible).

### Module: Document Transformation & Round-trip
- **DOCX Generation from Markdown** — args: markdown, track changes tags (`<ins>`/`<~~>`/`<comment>` with author/datetime) — `create-document` — native Word formatting.
- **Track Changes Extraction** — args: DOCX, `output_format:md|html|chunks`, `paginate`, `page_range`, `webhook_url` — `POST /v1/track-changes` — extract redlines + comments.
- **Thumbnail Generation** — args: `thumb_width`, `track_changes`, `page_range` — `GET /v1/thumbnails/{lookup_key}` — page thumbnails from prior conversion.
- **Synthetic Data Generation** — args: `output.n > 1` — multiply dataset.

### Module: Document Transformation Capabilities
- Form filling (`POST /v1/fill`; AcroForm + visual + image field detection; `confidence_threshold`; PDF/PNG).
- Document round-trip (DOCX → track-changes → md → create-document → DOCX with redlines).
- Markdown↔DOCX.
- Docling JSON round-trip.

### Module: Cross-Cutting Concerns (document)
> Orchestration/pipelines engine is in L0.H; L3 documents the document-specific orchestration facets.
- Orchestration, Pipelines & Workflows capabilities: declarative pipelines (YAML/Python; draft→saved→published; immutable versioned snapshots; per-step results; per-step billing); Temporal workflows; declarative map-reduce framework (lazy immutable Frame; chained ops; terminal actions); Pipeline + RoutedPipeline (ingestion orchestration with checkpointing + progress callbacks; multi-format routing); QueryEngine + CachedQueryEngine; Pipeline class (load→extract→chunk→enrich→embed→index); `run_pipeline(config)` (template loading → extraction → Docling export → graph conversion → export → statistics); datalab Workflow (Temporal-based); MCP server (expose operations as agent tools); automatic managed pipeline (no manual orchestration); orchestration features (checkpointing; progress callbacks; parallelization; caching; retries/timeouts; cost & token tracking); pipeline step types catalog (convert/segment/extract/custom/fill + map/filter/reduce/resolve/cluster/rank/split/gather/unnest/code_map/code_reduce/code_filter/parallel_map/equijoin/link_resolve/kg_build).
- Evaluation, Quality Assurance & Optimization capabilities: parse quality score 0-5; eval rubrics (block/page/document rules 0-5; `eval_rubric_id` on steps; from-feedback generation); Forge Evals (max 10 docs × 5 configs × 3 iterations; visual diffs; multi-model comparison); custom processor eval definitions (`eval_definition` per processor; `run_eval`); per-field verification (PASS/FAIL statuses); per-field confidence scoring 1-5 + reasoning; extraction score average; class confidence + alternate candidates; KVP validation (`POST /v1/validate`; `ValidatorResult` Pass/Fail + `ValidatorFailures`); gleaning (LLM iterative validation/refinement); validate (Python-expression-based with retries); calibration (reference anchors); plan rewrites (selection_pushdown, limit_pushdown); BARGAIN model cascades (statistical guarantees); MOAR (offline MCTS optimization; Pareto frontier; `.best()`/`.cheapest()`/`.frontier`); operation-level `optimize` flag; semantic cache metrics (hit_rate, avg_hit_similarity, retrieval time).
- Provenance, Citations & Source Tracking capabilities: block IDs (`data-block-id`; `{field}_citations` arrays); `file_citation` annotations (file_name/page_number/media_id/custom_metadata; char offsets); `<cite i="n"/>` markers; ProvenanceLedger (deterministic; document→page→batch→chunk→char-offset; `SourceAnchor` kinds observed/verbatim/derived/reconciled; `bind_stats`); source chunks alongside answer; chunk provenance + locator (`char:`/`page:`/`summary:`); bounding boxes in search results (UI highlighting); media download (`download_media(media_id)`; persistent IDs); per-page extraction results; unified Citation Object.
- Visualization capabilities: interactive HTML via Cytoscape (zoom/pan, search, image export); PyVis/Vis.js HTML (Louvain communities; centrality-sized nodes; dashed inferred edges; light/dark; physics toggle); t-SNE 2D visualization; Web UI playground; DocWrangler IDE (spreadsheet interface; automatic visualizations; in-situ feedback; prompt refinement with diffs; version control); graph HTML export / Docling output visualization.

### Module: Custom Processors
- **Custom Processor CRUD** — args: processor definition, settings — `POST /v1/custom-processors`, `GET`, `GET /{id}`, `POST /{id}/iterate`, `POST /{id}/describe`, `POST /{id}/execute`, `GET /{id}/pipelines` — AI-generated custom processor lifecycle.

### Module: Custom Processor Capabilities
- AI-generated custom processor (create/list/get/iterate/describe/execute).
- Custom processor as pipeline step (`type:"custom"` + `custom_processor_id` + `settings`).

### Module: Schema Management
- **Schema CRUD** — args: schema definition — `POST /v1/schemas`, list/get/`PUT /{id}` (with `create_new_version`), `DELETE` (soft) — extraction schema management.

### Module: Schema Capabilities
- Extraction schema CRUD (soft delete; `create_new_version`).
- Schema auto-generation (simple/moderate/complex from checkpoint).
- Schema Object (`schema_id` stable vs inline `page_schema`; `version` pin via `schema_version`; `version_history`; `archived`).

### Module: MCP Server Tools (document)
- **MCP Server Tools** — `WS /v1/mcp` — exposes search/ask/convert/extract/graph_query as agent tools. (MCP transport is L0; the tool catalog is L3.)

### Module: Billing Units (document)
> Billing in L0.I; L3 documents document-specific billing units.
- Per page, per token, per request, storage, surcharges (see S.31).

### Module: Multi-tenancy, Security, Residency & Administration (document)
> Mostly L0; L3 residual capabilities:
- Workspace isolation (`shared`/`personal`).
- Tenant sharding lifecycle.
- Organization context.
- `processing_location:us/eu` + EU premium; BYOB; consistency levels; replication + backups.
- `access_mode` private|public; budget controls (org monthly cap → 429 `budget_capped`; percentage email alerts); per-operation rate limiting.
- Tenancy endpoints.
- Deployment modes (managed SaaS / self-hosted container / on-prem / open-source local / BYOB).
- SDKs / CLI.
- Integrations (ecosystem frameworks; cloud storage; app platforms; document sources).

---
# LAYER 4 — AGENTIC ORCHESTRATION

> **Purpose.** Build agentic workflows and sandboxes on top of L0 primitives (sandboxes, vaults, scheduled jobs, webhooks) and L2/L3 intelligence APIs: agent definition, sessions, agent loop/events, tools/MCP/skills, permissions/hooks, multi-agent orchestration, memory/RAG, workflows, channels, voice (agent channel), extensions/plugins/marketplace.
> **Depends on**: L2 (model APIs + tools); L3 (voice channels, file search, image generation tool, code execution container); L0 (sandboxes, vaults, scheduled jobs, webhooks, files, vector stores).

## Domain L4.A — Agent Definition & Configuration

### Module: Agent Object & Versioning
- **Create Agent** — args: `name`, `description`, `model`, `system`, `tools`, `mcp_servers`, `skills`, `multiagent`, `metadata`, `tags`, `structured_output`, `style`, `collaborators`, `knowledge_base`, `timeout_seconds`, `restrictions`, `hidden`, `is_schedulable`, `context_access_enabled`, `context_variables`, `bundled_agent_id` — `POST /v1/agents` — returns `{id, version, created_at, updated_at}`. Persist a reusable, versioned agent definition.
- **Retrieve Agent** — args: `id`, `?version=n` — `GET /v1/agents/{id}` — fetch agent (optionally pinned version).
- **List Agents** — args: `agent_id`, `order`, `limit`, `cursor`, `include_hidden` — `GET /v1/agents` — paginated listing.
- **Update Agent** — args: `id`, `version` (required), fields to replace — `POST /v1/agents/{id}` — produces a new version on change.
- **Archive Agent** — `POST /v1/agents/{id}/archive` — one-way read-only archival.
- **Delete Agent** — `DELETE /v1/agents/{id}`.
- **List Agent Versions** — `GET /v1/agents/{id}/versions` (paginated).
- **Create Agent Release** — args: `version_label`, `semantic_version`, `version_name`, `version_description` — `POST /v1/agents/{id}/releases` — immutable release snapshot.
- **Deploy Agent Release to Environment** — `POST /v1/agents/{id}/releases/{ver}/environment/{env_id}`.
- **Undeploy Agent Release** — `POST /v1/agents/{id}/releases/{ver}/undeploy`.

### Module: Agent Definition Capabilities
- **Agent reference forms**: agent ID (latest), pinned `{id, version}`, `agent_with_overrides` (per-session model/system/tools/mcp/skills).
- **Override rules**: omit→inherit; null/empty→clear (model never clearable); value→full replacement.
- **Agent styles / reasoning modes**: `default`/`react`/`planner`/`custom`/`react_intrinsic`/`code_act`/`experimental_customer_care`; personas `default`/`plan`/`accept-edits`/`auto-approve`; `default`/`worker`/`explorer`; `Explore`/`Plan`/`general-purpose`.
- **Model selection**: string ID, `{id, speed}`, `provider/developer/model_id`, aliases (`opus`/`sonnet`/`haiku`/`fable`/`inherit`).
- **Default toolset**: flag enabling all built-in tools; explicit `tools[]`; mode-gated groups.
- **Structured output**: JSON Schema enforced (`outputType`, `structured_output`, `output_format`, `--output-schema`).
- **Metadata / tags**: arbitrary key-value; `tags`, `hidden`.
- **Description as routing metadata**: drives supervisor/collaborator routing.
- **Context variables**: platform runtime values referenced as `{var}`.
- **Agent restrictions / editability**: `editable`/`non_editable`/`custom`.
- **Timeout**: per-agent `timeout_seconds` (120-900).
- **Bundled / catalog agents**: derive via `bundled_agent_id`, `base_agent`, `create-from-template`.
- **File-based agent**: `~/.<platform>/agents/<name>.{md,toml,yaml}`.
- **Code-defined agent**: `new Agent({...})`, `AgentDefinition`.
- **Versioning / releases**: every save → new version; releases immutable; optimistic concurrency `version`; `precondition.content_sha256` for memories; `version_label`/`semantic_version`.
- **AGENTS.md layered discovery** (see S.21): global → project → cwd; `AGENTS.override.md` precedence; ~32 KiB cap; fallback filenames; `project_doc_max_bytes`.

### Module: Skills
- **Upload Skill (Custom)** — args: multipart files — `POST /v1/skills` — returns `{skill_id, latest_version}`.
- **Skills Config Write** — args: `path`, `enabled` — `POST /v1/skills/config/write` — enable/disable a skill.
- **List Skills** — args: `cwds`, `forceReload`, `perCwdExtraUserRoots` — `POST /v1/skills/list`.

### Module: Skills Capabilities
- **Skill concept & progressive disclosure**: SKILL.md + supporting files; Discovery (name + description ~100 tokens) → Activation (full SKILL.md) → Execution (supporting files on demand).
- **File format / fields**: YAML frontmatter `name`/`display_title`/`description`/`allowed-tools`/`disable-model-invocation`/`context: fork`; optional `SKILL.json` with `interface` + `dependencies.tools[]` (`env_var`/`mcp`); body = markdown instructions.
- **Pre-built skills**: pptx/xlsx/docx/pdf; challenge-my-thinking/data-analysis/deep-research/doc-coauthoring/document-review/internal-comms/meeting-prep/research-synthesis/skill-creator/stakeholder-translator/structured-extraction/onboarding; Carbon Design/DITA/Jira/create-plan.
- **Custom skills & creation routes**: authored; zip or individual files; from editor UI; "turn this into a Skill"; file-based; API upload; Studio publish.
- **Activation modes**: auto-match; slash `/{skill-name}`; natural reference; `$<skill-name>`; `skill` input item (injects full instructions); `use_skill`/`Skill` tool; `disable-model-invocation:true` for explicit-only.
- **Scopes / admin controls**: Personal vs Workspace; force-enabled by admins; per-workspace toggle; `perCwdExtraUserRoots`/`skills/extraRoots/set`; max 20 skills per session across all agents.
- **Attach to agent / session**: `skills[]` on agent; `skills.config` on custom agent; `AgentDefinition.skills`; implicit via environment filesystem `.agents/skills/<name>/SKILL.md`; per Work session.
- **Versioning**: `version` pin or `latest`.
- **Skills vs workflows distinction**: skills = reusable behavior; workflows = deterministic coded automation.
- **Skill as tool binding**: `SkillToolBinding` calls skillset/skill operation by id + path + method.
- **Skill dependencies**: `SKILL.json` `dependencies.tools[]` declaring required `env_var`/`mcp`.
- **Skill attachment to shell environments**: hosted `skill_reference` (skill_id[,version]) on `container_auto`/`container_reference`; local skill `{name, description, path}`; inline skill `{type:"inline", name, description, source:{type:"base64", media_type:"application/zip", data}}` uploaded to `/v1/containers`; privileged instructions (review as untrusted).
- **Skill file watching**: `skills/changed` notification → invalidation.
- **Plugin skills**: `plugin/skill/read` reads remote plugin skill Markdown on demand.
- **Skill create/update scheduled tasks**: invoke with `$skill-name` in desktop app.

## Domain L4.B — Models & Reasoning (agent-level)

### Module: Models (agent-level)
- **List Models** — `/v1/models/list` + `/embeddings`; `model/list` (`supportedReasoningEfforts`, `inputModalities`, `supportsPersonality`, `isDefault`, `upgrade`).
- **Model Policy CRUD** — `/v1/model_policy`.
- **Provider Capability Read** — `modelProvider/capabilities/read`.

### Module: Models (agent-level) Capabilities
- **Per-turn / run override**: `RunConfig(model)`, `turn/start` model override, `agent_with_overrides.model`, `RunOrchestrate.llm_params`, `conversations.start(model)`.
- **Reasoning effort / thinking config**: `model_reasoning_effort` (minimal/none/low/medium/high/max/xhigh/ultra); `effort` (low/medium/high/xhigh/max); `thinking` ThinkingConfig; `generation_config.thinking_summaries`/`thinking_level`; `modelSettings` reasoning effort; `speed:"fast"`.
- **Thinking events**: `agent.thinking`; `reasoning` items with summary+content; `thought` steps with `thought_summary`+`thought_signature` deltas; `thinking chunk`; `hide_reasoning` toggle.
- **Model rerouting / safety buffering**: `model/rerouted {fromModel, toModel, reason}`; `model/safetyBuffering/updated` with `fasterModel`.
- **Generation params**: `temperature`, `top_p`, `max_tokens`, `n`, `seed`, `stop`, `frequency_penalty`, `presence_penalty`, `logit_bias`, `completion_args`.
- **Per-turn personality**: `personality` override on `turn/start`.
- **Multi-provider gateway**: OpenAI-compatible passthrough (control plane in L0).
- **Verification**: `model/verification` additional account verification.

## Domain L4.C — Sessions, Threads, Runs & Interactions

### Module: Sessions
- **Create Session** — args: agent ref | `agent_with_overrides`, `environment_id`, `vault_ids`, `resources`/`memory_stores`, `title`, `previous_session_id`, `background`, `store`, `context`, `context_variables`, `llm_params`, `guardrails` — `POST /v1/sessions` — returns `{id, status, agent snapshot, environment_id}`. Provisions sandbox + starts conversation history.
- **Retrieve Session** — `GET /v1/sessions/{id}`.
- **List Sessions** — args: `limit`, `agent_id`, `order`, `cursor` — `GET /v1/sessions`.
- **Send Session Events** — args: events array (`user.message` / `user.tool_confirmation` / `user.custom_tool_result` / `system.message`) — `POST /v1/sessions/{id}/events` — continue/steer conversation.
- **List Session Events** — args: `types[]`, `limit`, `cursor` — `GET /v1/sessions/{id}/events`.
- **Stream Session Events (SSE)** — args: `event_deltas[]` opt-in — `GET /v1/sessions/{id}/events/stream`.
- **Resume Session** — `POST /v1/sessions/{id}/resume`.
- **Fork Session** — args: `last_turn_id` — `POST /v1/sessions/{id}/fork` — new session with copied history.
- **Steer Session** — `POST /v1/sessions/{id}/steer` — append user input to in-flight turn.
- **Interrupt Session** — `POST /v1/sessions/{id}/interrupt` — cancel mid-execution.
- **Session Goal CRUD** — `POST /v1/sessions/{id}/goal`, `GET .../goal`, `POST .../goal/clear` — long-running target with progress.
- **Rename Session** — `POST /v1/sessions/{id}/name`.
- **Update Session Metadata** — args: `gitInfo`/`tag`/`custom_title` patch — `POST /v1/sessions/{id}/metadata`.
- **Archive Session** — `POST /v1/sessions/{id}/archive`.
- **Delete Session** — `DELETE /v1/sessions/{id}`.
- **List Session Threads** — `GET /v1/sessions/{id}/threads`.
- **Stream Thread** — `GET /v1/sessions/{id}/threads/{tid}/stream`.
- **List Thread Events** — `GET /v1/sessions/{id}/threads/{tid}/events`.
- **Archive Thread** — `POST /v1/sessions/{id}/threads/{tid}/archive`.
- **Compaction Trigger** — `POST /v1/sessions/{id}/compact`.
- **Background Interaction Poll** — `GET /v1/interactions/{id}`.
- **Background Interaction Stream** — args: `stream=true`, `last_event_id=` — `GET /v1/interactions/{id}`.
- **Cancel Background Interaction** — `POST /v1/interactions/{id}/cancel`.

### Module: Session Capabilities
- **Status state machine**: `idle`/`running`/`rescheduling`/`terminated`; `in_progress`/`requires_action`/`completed`/`failed`/`cancelled`; `pending`/`running`/`completed`/`async_wait`/`async_completed`/`failed`/`cancelled`/`requires_input`/`expired`; Result subtypes (`success`/`error_max_turns`/`error_max_budget_usd`/`error_during_execution`/`error_max_structured_output_retries`).
- **Two-step lifecycle**: create provisions sandbox → first message starts work.
- **Continue / resume / fork**: `continue_conversation`, `resume: session_id`, `fork_session` (`thread/fork` with `lastTurnId`), `previous_interaction_id`, `previousResponseId`/`conversationId`, append-only immutable.
- **Multi-turn**: multiple `turn/start` append to thread.
- **Mid-turn steering**: `turn/steer`, `user.interrupt` + `user.message`.
- **Session storage / persistence**: JSONL on disk; `SessionStore` adapter (S3/Redis/custom); SQLite rollouts.
- **Session utility functions**: `list_sessions`, `get_session_messages`, `get_session_info`, `rename_session`, `tag_session`; `thread/list` filters.
- **Thread operations**: name/goal/metadata set, archive/unarchive/delete, unsubscribe, compact/start, shellCommand, inject_items, rollback.
- **Two independent state dimensions** (see S.33): conversation history (`previous_interaction_id`) and sandbox/files (`environment_id`) decoupled.
- **Background / async mode**: `background: true` returns interaction ID; poll/stream/cancel.
- **Goals**: long-running task target with progress; pause/resume/edit/clear.
- **Access scope**: API key creator only.
- **Data retention**: configurable days; `store=false` opt-out.

## Domain L4.D — The Agent Loop, Turns, Streaming & Events

### Module: Loop Execution
> Loop execution modes are capabilities of the Session service; documented here as a distinct module because they shape how events are emitted.

### Module: Loop Execution Capabilities
- **Server-Side Managed Loop**: platform runs the loop.
- **In-Process Loop**: SDK + bundled binary runs the loop.
- **Loop Lifecycle**: prompt → respond → tools → repeat.
- **Streaming via SSE**: `GET .../events/stream`, `stream: true`, `POST /runs?stream=true`, JSONL.
- **Symmetric step model** (see S.21): `step.start` → `step.delta`(s) → `step.stop`.
- **Item lifecycle**: `item/started` → `item/completed`; `span.model_request_start` → `event_start` → deltas → `agent.message` → `span.model_request_end`.
- **Compaction / context window**: `agent.thread_context_compacted`; `SystemMessage(subtype="compact_boundary")` + `PreCompact` hook + `/compact`; `thread/compact/start` + `contextCompaction` item; `compaction_settings.context_compaction_threshold`/`compaction_sliding_window`/`large_message_threshold`.
- **Outcome definition & evaluation**: `user.define_outcome` + `span.outcome_evaluation_*`.
- **Plan / todos streaming**: `turn/plan/updated` with steps; `update_todo_list`/`TaskCreate`/`TaskUpdate` tools; Todos panel.
- **Token usage events**: `thread/tokenUsage/updated`, `interaction.usage`, `ResultMessage.usage`, `usage.connector_tokens`.
- **Resumable streaming** (see S.10): `last_event_id` cursor.
- **Unknown event handling policy** (see S.10).
- **Event type catalog** (see S.21).
- **Delta type catalog** (see S.21).
- **Stop reasons / finish reasons** (see S.7).

## Domain L4.E — Tools (Built-in, Custom & Function Calling)

### Module: Built-in Tools Catalog
> Tools are configured on agents/sessions; here we document the built-in tool catalog as a module.

### Module: Built-in Tool Catalog (capabilities — see also S.21)
- **File ops tools**: read/write/edit/apply_diff/insert_content/search_and_replace.
- **Search tools**: glob/grep/list_files/GetSymbolsOverview/FindSymbol/FindReferencingSymbols.
- **Shell tools**: bash/execute_command/Shell/commandExecution.
- **Web tools**: web_search/google_search/web_search_premium, web_fetch/url_context.
- **Code execution tools**.
- **Image generation tool**.
- **Discovery tools**: ToolSearch.
- **Orchestration tools**: Agent/spawn_subagent/start_subtask/task/collabToolCall; Skill/use_skill; switch_mode; start_workflow; update_todo_list; ask_followup_question; request_permissions; Monitor.
- **Retrieval / RAG tools**: `file_search` (hosted RAG), `computer_use`, `document_library` with `library_ids`.

### Module: Built-in Tool Variant Capabilities
- **Web search variants**: dynamic filtering (model writes/runs code to filter); `max_uses`; `allowed_domains`/`blocked_domains`; `user_location`; `response_inclusion` (`full`/`excluded`); `allowed_callers` (`direct`/code-execution caller); errors; `search_context_size` low/medium/high; `filters`; `search_content_types` (image/text); `image_settings`; `return_token_budget` (default/unlimited); `external_web_access`; real-time feeds; `search_suggestions` widget; premium news-provider verification.
- **Web fetch / URL context variants**: `max_uses`, `citations.enabled`, `max_content_tokens`, `use_cache`, security (URL must be in prior context; no JS-rendered; text/HTML/PDF only), errors; two-step retrieval (index cache → live fetch); `url_context_result.status` `"unsafe"`; `open_page`/`find_in_page` actions.
- **Code execution variants**: Bash + file ops; REPL state persistence; programmatic tool calling; 90s/cell wall-clock; Python 3.11; 5 GiB RAM/disk; 1 CPU; networking disabled; no package install; pre-installed libs; container reuse via `container` id; idle ~5 min; 30-day cap; Files API integration via `container_upload`; bundled matplotlib (inline images); image code execution; server-side isolated container; `auto` mode (`{type:"auto",memory_limit,file_ids}`) or explicit container id; container expires 20 min idle; 100 RPM/org.
- **Shell variants**: long-lived bash process (cwd/env persists); schema-less; `restart:true`; no interactive/GUI; `local_shell` legacy; hosted `container_auto`/`container_reference`; Local environment; `network_policy` allowlist + `domain_secrets` (placeholder injection); Debian 12 `/mnt/data`; preinstalled runtimes.
- **Text/code file editing variants**: `view`/`str_replace`/`create`/`insert`; `apply_patch` (create_file/update_file/delete_file + V4A diff); patch harness helpers.
- **Computer use variants**: `screenshot`, clicks (left/right/middle/double/triple), `type`, `key`, `mouse_move`, `scroll`, `left_click_drag`, `hold_key`, `wait`, `zoom`; normalized coordinates 0-999; `safety_decision.require_confirmation`; browser/mobile/desktop environments; batched `actions[]`; key names; custom-tool/code-execution harness shapes.
- **Maps / places tool**: grounds answers in places; `place_citation` annotations.
- **Tool annotations** (see S.21): `readOnlyHint`, `destructiveHint`, `idempotentHint`, `openWorldHint`, `maxResultSizeChars`; destructive always triggers approval.
- **Tool configuration (availability vs permission)**: `tools[]` allowlist; `allowed_tools`/`disallowed_tools`; `enabled:false`; scoped rules (`Bash(npm *)`, `mcp__github__*`); Connector `tool_configuration.include/exclude` + `requires_confirmation`.
- **Parallel execution rules**: read-only concurrent; state-modifying sequential; `readOnlyHint:true` for parallel; subagents parallel up to `max_threads`; input guardrails parallel.
- **Tool result blocks** (see S.21): `text`/`image`/`audio`/`resource`/`resource_link`; `structuredContent`; `isError`; large outputs >100k tokens → sandbox file + truncated preview.
- **Tool error handling**: `deny_message`; uncaught exceptions → error results; decline/cancel completes with error; `is_error` flag.
- **Function-calling two-turn flow** (see S.21): `agent.custom_tool_use` → `requires_action` → `user.custom_tool_result`/`FunctionResultEntry`/`function_result` → resume; Chat Completions `tool_calls` + `tool_choice: auto|any` + tool-role message.
- **Multimodal tool input**: text + image (inline base64 + mime_type).
- **Async tools**: return immediately → callback URL with correlation ID; sub-correlation objects.
- **Tool result ordering & linkage rules** (see S.21).
- **Mixed server + client turns** (see S.21).
- **Server tool result block types** (see S.21).

### Module: Tool Search / Deferred Loading (capabilities)
- `tool_search_tool_regex_*`, `tool_search_tool_bm25_*`; `defer_loading:true`; ≥1 tool non-deferred; never defer search tool; 3-5 frequent tools non-deferred; max 10,000 deferred; ≤5 results/search; `tool_search_call` + `tool_search_output`; `additional_tools` input item; `parallel_tool_calls=False`; namespaces group tools; client vs hosted mode.

### Module: Programmatic Tool Calling (capabilities)
- Model writes code (Python/JS) that calls tools within single execution; pauses per tool call; intermediate results never enter model context; `allowed_callers:["programmatic"]`; V8 runtime (JS + top-level await; no Node/packages/network/fs); `program` item (`code`,`fingerprint`); `function_call` with `caller:{type:"program",caller_id}`; `program_output`; `function_call_output` must copy `caller`; supports function/custom/mcp/apply_patch/shell/code_interpreter; ZDR supported; pending call timeout ~4 min; 90s/cell; no `strict:true` tools; can't force programmatic of specific tool; `disable_parallel_tool_use` unsupported; recursive `$ref` → 400; MCP tools can't be called programmatically; results don't count to token usage.

### Module: Advisor Tool (capabilities)
- Faster executor consults higher-intelligence advisor mid-generation; params `model`/`max_uses`/`max_tokens`/`caching:{type:"ephemeral",ttl}`; `server_tool_use` → `advisor_tool_result` (`advisor_result` or `advisor_redacted_result` encrypted); advisor ≥ Sonnet 4.6 and ≥ executor; cannot combine with extended thinking; `tool_choice:tool` forcing; resume `pause_turn` by resending paused content.

### Module: Memory Tool (capabilities)
- `memory_*` (client, schema-less); store/retrieve across conversations; commands `view`/`create`/`str_replace`/`insert`/`delete`/`rename`; paths within `/memories`; auto "MEMORY PROTOCOL" system prompt; path-traversal validation; size caps; ZDR-eligible.

### Module: Container Management
- **Create Container** — args: `name`, `memory_limit`, `expires_after`, `skills` — `POST /v1/containers` — returns `{id}`.
- **List Containers** — `GET /v1/containers`.
- **Delete Container** — `DELETE /v1/containers/{id}`.
- **Container File Create** — multipart or `file_id`.
- **Container File List** — list generated files.
- **Container File Retrieve** — retrieve content.

### Module: Containers Capabilities
- Container reuse: top-level `container` id; pass back when programmatic call in flight.
- Container expires 20 min idle; 100 RPM/org.

## Domain L4.F — Connectors / MCP Servers

### Module: Connectors / MCP
- **Create Connector / MCP Server Registration** — args: `name` (≤64 alnum/_/-), `description`, `server`/`server_url`, `visibility`, `icon_url`, `headers` (static HTTP), `auth_data` (OAuth2 client_id/secret), `system_prompt`, `connector_id` (one of predefined catalog) — `POST /v1/connectors`.
- **Get Connector** — `GET /v1/connectors/{id|name}`.
- **List Connectors** — `GET /v1/connectors` (paginated, filterable).
- **List Connector Tools** — args: `refresh`, `pretty` — `GET /v1/connectors/{id}/tools`.
- **Update Connector** — args: UUID — `POST /v1/connectors/{id}`.
- **Delete Connector** — `DELETE /v1/connectors/{id}`.
- **Connector OAuth URL** — `POST /v1/connectors/{id}/auth_url` → `{auth_url, ttl}`.
- **Connectors Debugger** — pre-flight diagnostic against MCP URL.
- **Secure MCP Tunnel** — exposes private servers.
- **MCP OAuth Login** — `POST /v1/mcp_servers/{name}/oauth/login` → auth_url.
- **MCP Runtime Reconnect** — `POST /v1/sessions/{id}/mcp/{name}/reconnect`.
- **MCP Runtime Toggle** — args: `enabled` — `POST /v1/sessions/{id}/mcp/{name}/toggle`.
- **MCP Status** — `GET /v1/sessions/{id}/mcp/status`.
- **MCP Config Reload** — `POST /v1/mcp/config/reload`.
- **MCP Resource Read** — args: `server`, `uri` — `POST /v1/mcp/resource/read`.
- **Prepare List Tools** — `POST /v1/toolkits/prepare/list-tools`.
- **Async Tool Callback** — args: `tenant_id`, `correlation_id`, `result` — `POST /v1/tools/{tenant_id}/callback/{correlation_id}`.
- **Direct Tool Call Async** — `connectors.call_tool_async(connector_id, tool_name, arguments)`.

### Module: Connector / MCP Capabilities
- **MCP transports**: stdio (local subprocess `command`/`args`/`env`); streamable HTTP (`url`/`headers`); SSE (legacy deprecated); SDK MCP server (in-process custom tools); MCP tunnels (private servers).
- **MCP tool in request**: `type:"mcp"`/`"connector"`/`"mcp_server"`; `connector_id`/`server_url`/`server`; `server_label`; `server_description`; `headers`; `allowed_tools`; `require_approval` (always/never/`{never:{tool_names:[...]}}`); `authorization` OAuth not stored/echoed; `defer_loading`; `tool_configuration` include/exclude + requires_confirmation.
- **MCP approvals**: `mcp_approval_request`/`mcp_approval_response`; `requires_confirmation` + `tool_confirmations`; `safety_decision:require_confirmation` + `safety_acknowledgement:true`; `readOnlyHint` annotation (absence → write requires confirmation).
- **MCP tool-list caching**: cached per turn (`mcp_list_tools`); `refresh` to bypass.
- **MCP auth**: vault credentials matched by URL (`mcp_oauth`/`static_bearer`); `env` (stdio) or `headers` (http) with `${VAR}` expansion; OAuth 2.1 (SDK doesn't run flow); `mcpServer/oauth/login` returns auth URL; Connector `auth_data`; `connections` object.
- **MCP tool output handling**: >100k tokens → sandbox file; `MAX_MCP_OUTPUT_TOKENS` + per-tool `maxResultSizeChars`.
- **MCP failures**: session creation does NOT validate connectivity; `session.error` with `mcp_server_name` + `retry_status` (`mcp_connection_failed_error`/`mcp_authentication_failed_error`); retried on next idle→running; `init` message per-server `status: connected|failed`; connection timeout 30s default (`MCP_TIMEOUT`); `mcpServer/startupStatus/updated` notification.
- **MCP runtime control**: reconnect/toggle/status; `config/mcpServer/reload`; Refresh tools.
- **MCP resources**: `mcpServer/resource/read`.
- **MCP as server**: a platform CLI exposes `codex` + `codex-reply` style tools to MCP clients.
- **MCP elicitation**: `mcpServer/elicitation/request` `mode: "form"` (or `"openai/form"` variant) (`message`+`requestedSchema`) or `mode:"url"` (`message`+`url`+`elicitationId`); respond `accept`+`content` or `decline`/`cancel`; `AskUserQuestion` and tools with `_meta` requiring user interaction fall through to ask flow.
- **MCP listing / visibility / system prompt**: visibility `private`/`shared_workspace`/`shared_org`; connector `system_prompt` injected when its tools used.
- **MCP workflows**: agentic workflows with MCP.
- **Connectors Debugger**: pre-flight diagnostic.
- **Secure MCP Tunnel**: exposes private/on-prem firewalled servers without public exposure.

## Domain L4.G — Permissions, Approvals & Human Review

> Approval policy knob (`always_allow`/`always_ask`) lives in L0.C RBAC; the **approval flow itself** (runtime pause/resume, HITL) lives here in L4 and is mirrored in L5 (admin/observability facet).

### Module: Permission Policies (capabilities — see also S.21)
- **Permission policy types / modes**: `always_allow`/`always_ask`; `default`/`acceptEdits`/`plan`/`dontAsk`/`auto`/`bypassPermissions`; sandbox_mode `read-only`/`workspace-write`/`danger-full-access`; approval_policy `untrusted`/`on-request`/`never`; granular `{sandbox_approval, rules, mcp_elicitations, request_permissions, skill_approval}`; ToolPermission `read_only`/`write_only`/`read_write`/`admin`; agent modes `default`/`plan`/`accept-edits`/`auto-approve`; `requires_confirmation` per tool; `needsApproval:true` per tool.
- **Setting / changing permissions**: at agent creation/update; mid-session `set_permission_mode()`; `POST /v1/sessions/{id}` updates tools/mcp_servers without new agent version; CLI flags `--sandbox`/`--ask-for-approval`/`--yolo`; `/permissions` runtime overrides; `config.toml [apps.*]`; `turn/start` sandbox policy override.
- **Confirmation requests flow**: tool with `always_ask`/`requires_confirmation`/`needsApproval` invoked → session pauses → respond (`user.tool_confirmation` allow/deny + `deny_message`; `tool_confirmations: [{tool_call_id, confirmation}]`; `state.approve/reject`; `Continue`/`Always allow`/`Decline`; CLI keyboard) → resume; denied tools return rejected result with `deny_message`.
- **Multiple confirmations per request**: batch approve/deny.
- **Per-tool / per-app approval config**: `configs[].permission_policy`; `[apps.<id>.tools.<tool>]` with `approval_mode: auto|prompt|approve`; `alwaysAllow` per server; `--allowed-tools`.
- **Auto-review**: separate reviewer agent decides approvals; circuit breaker (3 consecutive denials or 10 in last 50); `/approve` override picker; denial returns rationale + stronger instruction; `auto` mode model classifier.
- **Granular approval decision payloads**: command execution `accept`/`acceptForSession`/`decline`/`cancel`/`acceptWithExecpolicyAmendment`; file change `accept`/`acceptForSession`/`decline`/`cancel`.
- **Permission requests (runtime escalation)**: `request_permissions` tool sends `item/permissions/requestApproval` with requested network/filesystem permissions; respond with granted subset; `scope:"session"` persists grant.
- **Network approval context**: concurrent network prompts grouped by destination (host + protocol + port).
- **MCP / app tool approvals**: `tool/requestUserInput` (Accept/Decline/Cancel); destructive annotations always trigger approval.
- **Guardrails**: input/output/tool guardrails; input guardrails `run_in_parallel`; output guardrails validate/redact; tool guardrails check args/results.
- **Sandbox / approval combinations**: e.g. `workspace-write --ask-for-approval on-request`; `read-only --never` (CI); `workspace-write --untrusted`; auto-review preset; `--yolo`.
- **Outside-CWD confirmation**: mandatory when tool reads/writes/runs outside cwd.
- **Subagent inheritance**: subagents inherit parent `bypassPermissions`/`acceptEdits`/`auto` and cannot override; inherit parent turn's sandbox policy + live runtime overrides.
- **Evaluation order**: Hooks → Deny rules → Ask rules → Permission mode → Allow rules → `canUseTool` callback; precedence `deny > defer > ask > allow`.
- **`canUseTool` callback**: returns `PermissionResultAllow`/`Deny`; can `updated_input` to redirect paths; `interrupt:true`.
- **Trust gating**: load project config only from trusted folders; trusted-folders config file; `--trust` temp.
- **Admin restrictions** (see L0.C): `requirements.toml` disallows `approval_policy = "never"`, constrains sandbox modes.
- **Programmatic mode defaults**: `--prompt` defaults to `auto-approve`, disables interactive tools; non-destructive tools only by default.
- **`bypassPermissions` constraints**: cannot run as root on Unix; isolated envs only.

### Module: Approval Pause & Resume (capabilities)
- `requires_action` + `user.tool_confirmation`; `canUseTool`/interruption; `requestApproval` + decision; `requires_input` status; `confirmation_status:pending` + `tool_confirmations[]` / `DeferredToolCallsException`; `result.interruptions` + `state.approve/reject`; resumable `result.state`; serialize `state` and resume later; works across handoffs and nested `agent.asTool()`.
- **Permission events**: `tool_decision` log events; `blocked_on_user` spans; `permission_mode_changed` events; hook spans capture lifecycle.
- **MCP require_approval**: `tool_config.require_approval` policy e.g. `"never"`.

## Domain L4.H — Hooks & Lifecycle Callbacks

### Module: Hooks
> Hooks are configured on agents/sessions; this module documents the hook event catalog and capabilities.

### Module: Hook Capabilities
- **Hook events catalog**: `PreToolUse`, `PostToolUse`, `PostToolUseFailure`, `PostToolBatch`, `UserPromptSubmit`, `MessageDisplay`, `Stop`, `SubagentStart`/`SubagentStop`, `PreCompact`, `PermissionRequest`, `SessionStart`/`SessionEnd`, `Notification`, `Setup`, `TeammateIdle`/`TaskCompleted`, `ConfigChange`, `WorktreeCreate`/`WorktreeRemove`, `TaskCreated` (agent-team).
- **Hook registration**: `hooks` option: keys = event names; values = arrays of `{matcher, hooks, timeout}`; shell-command hooks from settings; external agent migration supports hooks via `externalAgentConfig/detect`/`import`.
- **Hook boundaries**: `hook/started` / `hook/completed` with `{threadId, turnId?, run}`.
- **Hook attributes**: `hook_event`, `hook_name`, `num_hooks`, `hook_definitions` (gated), `duration_ms`, `num_success`, `num_blocking`, `num_non_blocking_error`, `num_cancelled`.
- **Matchers**: exact string if `[a-z0-9_-|, ]`; regex otherwise; `*`/empty matches all; tool hooks match tool names only.
- **Callback signature / IO**: `async cb(input_data, tool_use_id, context)`; returns `{systemMessage, continue}` + `hookSpecificOutput`:
  - PreToolUse: `permissionDecision` (allow/deny/ask/defer), `permissionDecisionReason`, `updatedInput` (must pair with allow/ask).
  - PostToolUse: `additionalContext`, `updatedToolOutput` (replace output before model sees it).
  - Async output: `{async: true, asyncTimeout}` — agent proceeds without waiting.
  - Return `{}` to allow without changes.
- **Multiple matching hooks**: run in parallel; most restrictive wins (`deny > defer > ask > allow`).
- **Quality-gate hooks (teams)**: `TeammateIdle` (exit 2 → feedback), `TaskCreated` (exit 2 → block creation), `TaskCompleted` (exit 2 → block completion).
- **Common patterns**: block `.env` writes; redirect Write paths via `updatedInput`; auto-approve read-only; forward `Notification` to Slack; track subagent completions; run webhooks after `PostToolUse`; enforce DB read-only.
- **Async tool callbacks**: tools post results to callback URL with correlation ID (OpenAPI/Python/Flow/MCP).
- **Flow in-graph control nodes**: error-handling / masking-sensitive-data nodes (flow-internal hooks).
- **Vault lifecycle webhooks**: `vault.archived`, `vault.deleted`, `vault_credential.archived`/`deleted`/`refresh_failed`.
- **Span events as observability hooks**: `span.model_request_start`/`_end`, `span.outcome_evaluation_*`.
- **Hook scope (one platform variant)**: hooks for lifecycle side effects only (logging, tracing); NOT for blocking/approval/execution-shaping (use guardrails/approvals/filters).

## Domain L4.I — Multi-Agent Orchestration

### Module: Subagents / Collaborators / Teams
- **Agent Roster/Coordinator config** — args: `multiagent: {type:"coordinator", agents:[{type:"agent",id,version?},{type:"self"}]}` (on agent definition).
- **CSV Batch Fan-Out** — args: `csv_path`, `instruction` with `{column}` placeholders, `id_column`, `output_schema`, `output_csv_path`, `max_concurrency`, `max_runtime_seconds` — `spawn_agents_on_csv` — one worker per row; workers call `report_agent_job_result`; combined CSV export.
- **Subagent / Agent tool invocation** — args: `name`, `prompt` — `Agent(name, prompt)` / `spawn_subagent(type, description, fork_context?)` / `collabToolCall`.
- **Agent Teams (experimental)** — shared task list + `SendMessage` mailbox; env flag; in-process or split panes.
- **Handoff config** — args: `handoffs:[agent_ids]`, `handoff_execution: server|client`.
- **Agents-as-Tools** — args: `name`, `description` — `manager.as_tool(name, description)` / `agent.as_tool()` — manager keeps reply ownership.
- **Collaborators config** — args: `collaborators:[agent_names]` (native/external/assistant).
- **Resume Subagent by threadId** — args: `prompt`, `threadId` — resume-by-thread tool.
- **Dynamic Workflow JS** — args: `agent()`/`pipeline()` (≤16 concurrent / 1000 total) — model writes JS orchestrating subagents at scale; resumable within session.

### Module: Multi-Agent Capabilities
- **Subagent definition methods**: programmatic `agents: {name: AgentDefinition}`; filesystem `.<platform>/agents/*.md` (YAML frontmatter); `.<platform>/agents/*.toml`; built-in `Explore`/`Plan`/`general-purpose`/`default`/`worker`/`explorer`/`explore`/`general`.
- **`AgentDefinition` fields**: `description` (drives auto-delegation), `prompt`, `tools`, `disallowedTools`, `model`, `skills`, `memory`, `mcpServers`, `initialPrompt`, `maxTurns`, `background`, `effort`, `permissionMode`, `isolation: worktree`, `color`; custom agent fields `name`/`description`/`developer_instructions`/`nickname_candidates`/`model`/`model_reasoning_effort`/`sandbox_mode`/`mcp_servers`/`skills.config`.
- **Inheritance**: own system prompt + Agent tool prompt + project context doc + tool defs (inherited or subset); no parent conversation history/system prompt/tool results; subagents inherit parent turn's sandbox policy + live runtime overrides; `fork_context: true` passes parent history.
- **Invocation**: automatic (via `description`); explicit (name in prompt, `@mention`, `--agent <name>`); dynamic (factory functions at query time); `spawn_subagent(type, description, fork_context?)`; `collabToolCall` items.
- **Foreground / background**: background by default; `run_in_background: false` when result needed; parallel/background; `/agent` inspects/switches threads.
- **Nesting**: subagents can spawn own subagents; depth 5 max; `agents.max_depth` default 1.
- **Resuming subagents**: Agent tool result includes `agentId`; resume by `resume: sessionId` + agent ID in prompt; resume-by-thread tool by `threadId`; built-in `Explore`/`Plan` one-shot.
- **Restricting**: `tools: [Agent(worker, researcher)]` allowlist; `permissions.deny: ["Agent(Explore)"]`; omit `Agent` to block all.
- **Agent teams**: lead + teammates; shared task list + `SendMessage` direct messaging; experimental env flag; in-process or split panes.
- **Threads (multi-agent)**: coordinator delegates to roster; each agent in context-isolated thread sharing same sandbox; max 25 concurrent threads; max 1 levels of delegation; cross-posted blocking events with auto-routed responses.
- **Roster entry forms**: `{type:"agent", id}`, `{type:"agent", id, version}`, `{type:"self"}`.
- **Handoffs**: `handoffs[]` list of agent IDs; `handoff_execution: server|client`; `agent.handoff` entry; `handoff()` wrapper with metadata/filtered history; `handoffDescription` drives routing; unlimited chaining.
- **Agents-as-tools**: `agent.asTool({name, description})` / `agent.as_tool()` — manager keeps reply ownership.
- **Collaborators**: `collaborators[]` agent names; supervisor routes by `description`; native/external/assistant.
- **CSV batch fan-out**: `spawn_agents_on_csv` one worker per row; fields `csv_path`/`instruction` with `{column}` placeholders/`id_column`/`output_schema`/`max_concurrency`/`max_runtime_seconds`; workers call `report_agent_job_result`; combined CSV export.
- **Dynamic workflows (JS)**: JS orchestrating subagents at scale; `agent()`/`pipeline()`; ≤16 concurrent / 1000 total; resumable within session.
- **Multi-agent via MCP + Agents SDK**: run platform CLI as MCP server, orchestrate with Agents SDK; traces capture every prompt/tool/hand-off.
- **Task list**: shared work items pending/in-progress/completed with dependencies; file-lock-based claiming.
- **Team / direct messaging**: threads; Agent team `SendMessage`; CSV fan-out workers; collaborators + flows.
- **Delegation**: via threads / Agent tool / `collabToolCall` / collaborator / Handoff `handoffs[]` / agents-as-tools.
- **Global controls**: `max_threads`, `max_depth`, `job_max_runtime_seconds`, `interrupt_message`.

## Domain L4.J — Memory & Knowledge (RAG)

### Module: Memory Store / Library / Knowledge Base
- **Create Memory Store** — args: `name`, `description` — `POST /v1/memory_stores` — returns `{id}`.
- **Seed Memory** — args: `path`, `content` (no overwrite) — `POST /v1/memory_stores/{id}/memories`.
- **List Memories** — args: `path_prefix`, `depth` — `GET /v1/memory_stores/{id}/memories`.
- **Get Memory** — `GET /v1/memory_stores/{id}/memories/{mem_id}`.
- **Update Memory** — args: `content`/`path`, `precondition.content_sha256` — `POST /v1/memory_stores/{id}/memories/{mem_id}`.
- **Delete Memory** — `DELETE /v1/memory_stores/{id}/memories/{mem_id}`.
- **List Memory Versions** — args: `memory_id` — `GET /v1/memory_stores/{id}/memory_versions`.
- **Get Memory Version** — `GET /v1/memory_stores/{id}/memory_versions/{vid}`.
- **Redact Memory Version** — `POST /v1/memory_stores/{id}/memory_versions/{vid}/redact`.
- **Agentic Memory add/search/list/retrieve/delete** — `client.memory.*` / `/memories`, `/memories/search`.
- **Create Library** — args: `name`, `description` — `POST /v1/libraries` — returns `{library_id, generated_name, generated_description}`.
- **Upload Library Document** — args: `file` — `POST /v1/libraries/{id}/documents`.
- **List Library Documents** — `GET /v1/libraries/{id}/documents`.
- **Library Document Status** — `GET /v1/libraries/{id}/documents/{doc_id}/status`.
- **Library Document Text Content** — `GET /v1/libraries/{id}/documents/{doc_id}/text_content`.
- **List Library Accesses** — `GET /v1/libraries/accesses/{id}`.
- **Create Knowledge Base** — args: `knowledge_base`, `documents`, `embeddings_model_name`, `chunk_size` — `POST /v1/orchestrate/knowledge-bases`.
- **Knowledge Base Status** — `GET /v1/orchestrate/knowledge-bases/{kb_id}/status`.
- **Create Vector Index** — args: `name`, `embeddings_model_name`, `chunk_size`, `chunk_overlap`, `top_k`, `extraction_strategy` — `POST /v1/vector-indices`.
- **Attach Collections to Vector Index** — `POST /v1/vector-indices/{id}/collections`.
- **Refresh / Rebuild Vector Index** — `POST /v1/vector-indices/{id}/refresh | /rebuild`.
- **Retrieve from Vector Index** — `GET /v1/vector-indices/{id}/retrieve`.

### Module: Memory / RAG Capabilities
- **Memory store concept**: workspace-scoped collection of text documents optimized for the model; mounted as directory inside sandbox; agent reads/writes with file tools; every change → immutable memory version.
- **Memory CRUD**: create store, seed memories (`path`+`content`, no overwrite), list (`path_prefix`, `depth`), read, update (content and/or path rename), delete; limits 100 kB/memory, 2,000 memories/store.
- **Memory versions (audit)**: immutable `memver_...` on every mutation; retained 30 days; list/retrieve/redact (scrubs content, preserves audit); cannot redact current head.
- **Safe edits (optimistic concurrency)**: `precondition: {type: "content_sha256", content_sha256}`.
- **Attach to session**: `resources[]` at session creation only; `access: read_write|read_only`; `instructions` ≤4096 chars; max 8 memory stores per session; mounts under `/mnt/memory/{slug}/`.
- **Agentic memory**: `memory_enabled` on agent; `client.memory.add_messages/search/list/retrieve/delete`; endpoints `/memories`, `/memories/search`.
- **Project context doc / auto-memory**: project context loaded at session start (re-injected, prompt-cached); auto-memory accumulates learnings; `AgentDefinition.memory: user|project|local`.
- **`Memory()` sandbox capability**: store reusable workflow lessons across runs.
- **Libraries (RAG)**: persistent knowledge base of uploaded documents/web pages; queried via `document_library` tool; cited answers with numbered footnotes + Sources button.
- **Library CRUD**: create (name, description), list (nb_documents), delete; documents upload/list/get/status (Running/Completed)/text_content/delete.
- **Library supported formats**: PDF, Word, PowerPoint, ODT, EPUB, RTF, Excel, CSV, ODS, Numbers, PNG, JPEG, WebP, GIF, TXT, Markdown, RST, LaTeX, JSON, JSONL, XML, YAML, code files, EML, MSG.
- **Library limits**: up to 100 files/upload, 100 MB/file; monthly processing allowance; no limit on number of Libraries.
- **Library sharing / access**: Collaborator/Viewer/Entire organization; `libraries.accesses.list` with `org_id`/`level`/`share_with_uuid`/`share_with_type`; owner-only sharing/deletion; Viewer/Editor roles, User/Workspace/Org scope.
- **Product ↔ API bridge**: same Library IDs across product and API; share with Org for API access.
- **Knowledge bases**: built-in (managed Milvus) or external (own Milvus/Elasticsearch/custom_search); `embeddings_model_name`, `chunk_size`, `chunk_overlap`, `top_k`, `extraction_strategy`; representation `auto|tool`; `prioritize_built_in_index`; chat-with-docs per-thread transient KB.
- **Document collections / vector indices**: lower-level CRUD; `CreateVectorIndex` with embeddings model + chunking + retrieval; `VectorIndexStatus: ready|not_ready|rebuilding|error|update_pending`; refresh/rebuild/retrieve.
- **File search (hosted RAG)**: hosted tool.
- **Citations / references structure**: `ToolReferenceChunk` (`tool`, `title`, `url`, `source`) interleaved with `TextChunk`; `ReferenceChunk` with `reference_ids`; references with `url`/`title`/`snippets`/`description`/`date`/`source`; numbered footnotes + Sources button.
- **Sandbox files as knowledge**: `workspace/` directory; packages/files persist across interactions sharing `environment_id`.
- **Web search as indexed knowledge**: `web_search = "cached"` index.
- **Data / training policy**: data accessed via Connectors never used to train/fine-tune models.

## Domain L4.K — Workflows, Scheduled Tasks & Automation

> Cron scheduling mechanics live in L0.H; this L4 module documents the **agentic flow / workflow graph**.

### Module: Workflow / Flow
- **Run Flow** — args: flow id, input — `POST /v1/orchestrate/flows/{id}/run` (sync).
- **Run Flow Async** — args: flow id, input, `callbackUrl` — `POST /v1/orchestrate/flows/{id}/run/async`.
- **Start Workflow (curated)** — args: `workflow_name`, `args?` — `start_workflow(workflow_name, args?)`.
- **Studio Workflow publish** — Studio-built workflow returning `ChatAssistantWorkflowOutput`, publish to chat.

### Module: Workflow / Flow Capabilities
- **Flow `@flow` decorator**: `name`/`display_name`/`description`/`input_schema`/`output_schema`/`initiators`/`schedulable`/`llm_model`/`agent_conversation_memory_turns_limit`; run sync or async with `callbackUrl`.
- **Flow nodes**: Tool, Agent, Generative prompt, Branch (conditional), Parallel branch, Foreach (iterate), Loop, Decisions, Timer, User activity (pause for input, not in callback flows), Document classifier/field extractor/text extractor (preview), Data map, Error handling/masking sensitive data.
- **Flow callbacks**: fire-and-forget; OpenAPI/Python/Flow/MCP; OpenAPI recommended; Flow callbacks must not contain user-activity nodes.
- **Langflow / flows import**: import visually-built Langflow flows as tools; integrate external flow platforms.
- **`is_schedulable`**: agent or `@flow` flag; schedules created via Chat UI natural language.
- **Studio workflows**: coded deterministic Studio automations published as chat-compatible assistants; invoked explicitly via `+` > Workflows; `Workflow started`/output/`Workflow failed` events; gated by account tier; developer surfaces (Forms and confirmations / Progress tracking / Canvas); must return `ChatAssistantWorkflowOutput` to publish.
- **Dynamic workflows (JS)**: JS script orchestrating subagents; `agent()`/`pipeline()`; ≤16 concurrent / 1000 total; `/workflows` view with pause/resume/stop/restart/save.
- **Curated workflows**: `start_workflow(workflow_name, args?)` curated multi-step processes.
- **Goal mode**: long-running task target with outcome/constraints/verification; `/goal` set; progress row pause/resume/edit/clear; parallel goals keep separate context; "Prevent sleep while running"; system notifications for input needs.
- **`/loop` (in-session)**: repeat a prompt within a CLI session for quick polling.
- **`custom_join_tool`**: Python tool for custom synthesis of task results.
- **`sync_tool_flow_interactions`**: sync user interactions from a tool flow back to the agent.

## Domain L4.L — Channels, Voice & Embedded Chat

### Module: Channels
- **Bind Channel to (Agent, Environment)** — args: `agent_id`, `env_id`, channel config — `POST /v1/agents/{agent_id}/environments/{env_id}/channels`.
- **Channel CRUD (per channel type)** — Slack, Teams, Twilio SMS/WhatsApp, Facebook Messenger, Genesys Bot/Audio, web_chat, embedded_chat.
- **Phone Channel CRUD + Numbers Management** — `POST /v1/channels/phone`, `POST /v1/channels/phone/{id}/numbers` (add/list/patch/delete).
- **Slack App Instance CRUD** — generic app registration + Slack app instances.
- **Embedded Chat Config** — args: `layout`, `is_live` — `PUT /v1/agents/{id}/embedded-chat-config` + web chat SDK.
- **Embed Settings** — `/v1/embed-settings/config` CRUD + `/generate-key-pair`.
- **Chat Starter Settings** — args: `starter_prompts`/`welcome_content`/`icon` — `PUT /v1/agents/{id}/chat-starter-settings`.

### Module: Channels Capabilities
- **Delivery surfaces**: IDE/CLI; web; Slack; Chrome; GitHub/Linear/Slack triggers; phone/voice; mobile; Office add-ins; Cowork workspace; Dispatch.
- **Channel types**: Slack, Microsoft Teams, Twilio SMS/WhatsApp, Facebook Messenger, Genesys Bot/Audio Connector, web chat/embedded chat, phone/voice.
- **Embedded chat config**: `layout`, `is_live`; web chat SDK events (`pre:send`, `pre:receive`, `chat:ready`, `feedback`, `view:change`) and methods (`send()`, `restartConversation()`, `loadThreadById()`, `updateAuthToken()`).
- **App-server embedded mode**: JSON-RPC for deep embedding (auth, history, approvals, streamed events); WebSocket/Unix socket transports; remote terminal UI; cloud integrations (issue trackers and chat platforms: GitHub PRs/issues, Linear issues/comments, Slack channels/threads) to start cloud containers.
- **MCP elicitation (tool/requestUserInput)**: 1-3 short questions.
- **Multimodal output**: `response_format` can request `[text, image, audio]`; TTS/audio generation models.

### Module: Voice (agent channel)
- **Voice Configuration CRUD** — args: `AgentIdleHandler`, `RealtimeAgentSettings` — `POST /v1/voice-configurations`; referenced by agent `voice_configuration_id` + environment `voice`.
- **Realtime API WebSocket upgrade** — `POST /v1/realtime`.
- **Realtime WebRTC Call** — `POST /v1/realtime/calls`.
- **Realtime Ephemeral Client Secret** — `POST /v1/realtime/client_secrets`.

### Module: Voice (agent channel) Capabilities
- **Voice configuration**: referenced from agents via `voice_configuration_id` and environments via `voice`; `AgentIdleHandler` (pre_hold_message, hold_message, typing_enabled, typing_duration_seconds, audio_clip_id); `RealtimeAgentSettings`/`RealtimeAgentSettingsIn`.
- **Two voice architectures**: (1) speech-to-speech with live audio sessions (natural low-latency); (2) chained voice pipeline (STT → text agent → TTS, predictable).
- **Realtime API connection methods**: WebRTC (recommended for browser), WebSocket, SIP.
- **Voice classes**: TS `RealtimeAgent` + `RealtimeSession`; Py `VoicePipeline` + `SingleAgentVoiceWorkflow` + `AudioInput` + `TTSModelSettings` + `VoicePipelineConfig`; `pipeline.run(audio_input)` → `result.stream()` async iterator; event `voice_stream_event_audio`.
- **Voice models**: realtime/transcribe/tts/audio models.
- **ChatKit**: browser UI component.

## Domain L4.M — Extensions, Plugins, Marketplaces & Interoperability

### Module: Plugin / Marketplace
- **Add Marketplace** — args: `source: local|git|npm|remote` — `POST /v1/marketplaces/add`.
- **Remove/Upgrade Marketplace** — `POST /v1/marketplaces/{name}/remove | /upgrade`.
- **List Plugins** — `GET /v1/plugins`.
- **Get Plugin** — `GET /v1/plugins/{id}`.
- **Install Plugin** — args: `marketplacePath` | `remoteMarketplaceName` — `POST /v1/plugins/install`.
- **Uninstall Plugin** — `POST /v1/plugins/{id}/uninstall`.
- **Read Plugin Skill** — `POST /v1/plugins/{id}/skill/read`.
- **App Registration / Apps config** — `[apps.<id>]` + per-tool `[apps.<id>.tools.<tool>]`; `app/list`.
- **Create Agent from Template** — args: `template_id` — `POST /v1/agents/create-from-template`.
- **Create Tool from Template** — `POST /v1/tools/create-from-template`.
- **Template Status** — `GET /v1/agents/{id}/template-status`.
- **Browse Catalog** — `GET /v1/catalog`.

### Module: Extensions / Plugins Capabilities
- **Plugins**: bundle skills/agents/hooks/MCP servers; `SdkPluginConfig`.
- **Plugin sources**: `local`/`git`/`npm`/`remote`; `plugin/list`/`read`/`install`/`uninstall`; `plugin/skill/read` reads remote plugin skill Markdown on demand; `installPolicySource: null|WORKSPACE_SETTING|IMPLICIT_CANONICAL_APP`.
- **Marketplaces**: `marketplace/add`/`remove`/`upgrade`; remote plugin marketplaces persisted to user config; `plugin-hints`/`plugin-dependencies` (version constraints)/`plugin-relevance` (suggest when work matches).
- **Plugin subagent restrictions**: `hooks`/`mcpServers`/`permissionMode` frontmatter ignored for plugin-provided agents (security); plugin subagents appear under scoped names (`my-plugin:review:security`).
- **Apps (connectors)**: `[apps.<id>]` with `enabled`/`destructive_enabled`/`open_world_enabled`/`approvals_reviewer`/`default_tools_approval_mode` + per-tool `[apps.<id>.tools.<tool>]`; `app/list` with `isAccessible`/`isEnabled`/branding/metadata/labels; invoke with `$<app-slug>` + `mention` input item (`path: "app://<id>"`).
- **Catalog**: governed library of pre-built agents and tools (HR, IT, procurement, sales, productivity) for discovery/reuse.
- **Templates**: `create-from-template` for agents and tools; `template-status`.
- **External Agent Migration**: `externalAgentConfig/detect`/`import` discovers & migrates artifacts (config, skills, AGENTS.md, plugins, MCP, subagents, hooks, commands, sessions).
- **Branding (partners)**: allowed names; not permitted names/visuals.
- **Dev Containers**: `devcontainer.secure.json`, `Dockerfile.secure`, `init-firewall.sh`.
- **Experimental feature flags**: `experimentalFeature/list`/`enablement/set` with `stage: beta|underDevelopment|stable|deprecated|removed`; patch runtime settings (`apps`, `plugins`).
- **Custom-agent metadata**: `language`, `framework`, `tool count`, `tool names`, `connection requirements`.
- **Checkpointing & rollback**: auto git snapshot + conversation + tool-call record before file-modifying ops; `/restore`; `enable_file_checkpointing` + `rewind_files`.
- **Context mentions (`@`)**: inject file/folder/git-changes/commit/problems/terminal content.
- **Canvas**: reviewable, editable document surface for structured outputs.
- **Projects**: scoped work area grouping related conversations.
- **Custom instructions**: persistent behavior rules across all Work conversations.
- **Files (single-chat context)**: upload for a single chat (vs Library for cross-session).
- **CLI surfaces**: per-platform CLI binaries/flags.
- **Web search control**: `web_search: cached|disabled|live|indexed`; `--yolo` defaults to live.
- **`hide_reasoning`**: show/hide reasoning trace.
- **Developer Mode MCP Client**: full MCP client support for all tools, both read and write; OAuth/No Auth/Mixed Auth; tool call review/confirmation respects `readOnlyHint`.
- **Default manifest / capabilities**: `defaultManifest`/`default_manifest` default workspace contract for fresh sessions; `capabilities` array replaces defaults (filesystem, shell, compaction).

### Module: External Agents / Interoperability
- **Create External-Chat Agent** — args: `api_url`, `auth_scheme: BEARER_TOKEN|API_KEY|NONE`, `auth_config` — `POST /v1/agents/external-chat`.
- **A2A Protocol Versions** — args: client/server role — `GET /v1/a2a/versions`.
- **External Agent Config Detect** — `POST /v1/externalAgentConfig/detect`.
- **External Agent Config Import** — `POST /v1/externalAgentConfig/import`.
- **CLI-as-MCP-Server** — platform CLI exposes `codex` + `codex-reply` style tools.
- **Model Gateway Passthrough** — `/gateway/model/chat/completions`, `/embeddings` (control plane in L0).

## Domain L4.N — Webhooks, Event Delivery & ChatGPT Developer Mode

> Webhook registration/delivery infrastructure is in L0.O; L4 documents the agent/event-delivery facet.

### Module: Webhook Registration & Delivery (agent facet)
- **Webhook Registration** — args: webhook URL, event filters — agent-scoped event subscription.
- **Webhook Inbound HTTP** — Standard Webhooks spec (see S.11).
- **Webhook Event Catalog** — see S.21 event type catalog.

### Module: Sandbox Defaults & Manifests
- **Default Manifest** — `defaultManifest`/`default_manifest` default workspace contract for fresh sessions (see Extensions capabilities).

---
# LAYER 5 — OBSERVABILITY & ADMINISTRATION

> **Purpose.** Build observability and administration features on top of L0 services (identity, billing, quotas, compliance, files, webhooks): telemetry/traces, logs/dashboards, evals/judges/campaigns/datasets, moderation/guardrails, approvals (HITL), safety & adversarial testing, model-lifecycle admin, analytics portals, telemetry backend integrations.
> **Depends on**: L0 (identity, billing, quotas, compliance, files, webhooks); wraps every other layer.

## Domain L5.A — Telemetry & Observability Backend Wiring

### Module: Observability Export
- **Telemetry Configuration Service** — args: master `enable_telemetry` flag; per-signal exporters (`metrics_exporter`/`logs_exporter`/`traces_exporter`: `console`/`otlp`/`prometheus`/`none`); `enable_enhanced_telemetry_beta` for span tracing; tracing scope (SDK-level / per-run reduction) — configures telemetry export.

### Module: Observability Export Capabilities
- OTLP transport config (env vars `OTEL_EXPORTER_OTLP_PROTOCOL` grpc/http/json/http/protobuf, `OTEL_EXPORTER_OTLP_ENDPOINT` :4317 gRPC / :4318 + /v1/{metrics,logs,traces}, `OTEL_EXPORTER_OTLP_HEADERS`, per-signal overrides, `OTEL_METRIC_EXPORT_TEMPORALITY_PREFERENCE`).
- Export interval tuning (`OTEL_METRIC_EXPORT_INTERVAL` 60000ms, `OTEL_LOGS_EXPORT_INTERVAL` 5000ms, `OTEL_TRACES_EXPORT_INTERVAL` 5000ms; set ~1000ms for short-lived `query()`).
- mTLS & dynamic rotating-token headers (client cert/key/passphrase per protocol; dynamic rotating-token headers via external script with configurable debounce default 29 min).
- SDK/CLI telemetry mechanics (SDK forwards config to CLI child; `console` exporter not usable through SDK; admin central config via managed settings `env` block MDM-distributed high precedence not user-overridable; export intervals & flush on clean exit bounded timeout, spans dropped if collector slow, buffered data lost on kill).
- Hosted dashboards (built-in enabled by default / Enterprise always-on / paid tier billing-enabled).

### Module: Hosted Observability UI
- **Hosted Dashboard Service** — args: workspace/project filter — Traces dashboard / Explorer / AI Studio Logs — built-in hosted observability UI.

## Domain L5.B — Identity, Resource Attribution & Tenancy Configuration

### Module: Observability Attribution
- **Resource Attributes Service** — configures attributes stamped on telemetry: `service.name`, `resource_attributes` env, `enduser.id`/`tenant.id`/`user.account_uuid`/`user.id`/`user.email`/`user.groups`/`identity.source`/`app.version`/`app.entrypoint`/`terminal.type`; `metadata.user_id` opaque no-PII; `metadata` custom KV per-request filterable in Explorer; `api_agent_id`; `correlation_id`; `organization.id`/`prompt.id`/`workflow.run_id`/`workflow.name`/`workspace.host_paths`; bool controls `include_resource_attributes`/`include_session_id`/`include_account_uuid`/`include_version`/`include_entrypoint`.

### Module: Attribution Capabilities
- Gateway OIDC identity stamping (when agent calls through IdP gateway, `user.*`/`identity.*` stamped from OIDC session, override env-supplied).
- Attribute behavior & formatting (built-in standard attrs always win on collision; `OTEL_RESOURCE_ATTRIBUTES` formatting rules; per-call `env` injection with percent-encoded values).
- Cardinality discipline (exclude custom attrs from datapoint labels via `OTEL_METRICS_INCLUDE_RESOURCE_ATTRIBUTES=false`).

## Domain L5.C — Observability Signals Emitted During the Run

### Module: Observability Signals
- **Metrics Emission** — emits `session.count`, `lines_of_code.count`, `pull_request.count`, `commit.count`, `cost.usage` (USD), `token.usage`, `code_edit_tool.decision`, `active_time.total`; attributes (`start_type` fresh/resume/continue/agents_view; lines added/removed). Equivalent counts via response metadata on other platforms (`usageMetadata`, event fields `input_tokens`/`output_tokens`/`total_time_elapsed`).
- **Log Events Emission** — per-prompt / per-API-request+error / per-tool (`tool_result`, `tool_decision`) / MCP (`mcp_server_connection`) / `permission_mode_changed` / `assistant_response` events; content-bearing events gated by flags; security-relevant events form per-user audit trail forwardable to SIEM.
- **Traces / Spans Emission** — span hierarchy `interaction` → `llm_request`/`hook`/`tool` → `tool.blocked_on_user`/`tool.execution`/subagent spans; full attribute sets per span type; span status `ERROR` on failure; retry as `gen_ai.request.attempt` span event; default trace contents (overall run/workflow, each model call, tool calls & outputs, handoffs & guardrails, custom spans via `withTrace`/`trace`).
- **In-process Cost & Usage Aggregator** — `message_id` dedup; per-step token counts; `total_cost_usd` cumulative across steps; `modelUsage`/`model_usage` map; constraints (estimates drift, don't bill end users, each `query()` returns own total, read cost from result message, on `output_tokens` discrepancy use highest & prefer result message's accumulated estimate).
- **W3C Trace-Context Propagation** — auto-inject `TRACEPARENT`/`TRACESTATE` into CLI subprocess & Bash/PowerShell children; direct API calls carry `traceparent` header with `traceresponse` recorded as span link; skip if user sets `TRACEPARENT` explicitly; interactive CLI ignores inbound (only Agent SDK and `-p` honor it); env flag to enable through custom proxy.

## Domain L5.D — Storage, Logging & Retention

### Module: Observability Storage
- **Observability Store / Log Toggle Service** — args: per-signal enable, `store` boolean per request/project, always-captured for workspace Enterprise, tracing-enabled-by-default scope.

### Module: Observability Retention & Gating Capabilities
- Retention windows (7/14/28/55-day configurable, datasets don't expire, hosted Enterprise/dashboard, abuse-monitoring ≤30 days, flagged data 55 days, Compliance Activity Feed 6 years, Access Transparency + preservation events).
- Sensitive-content gating flags: `OTEL_LOG_USER_PROMPTS=1`, `OTEL_LOG_ASSISTANT_RESPONSES=1` (falls back to user-prompts flag), `OTEL_LOG_TOOL_DETAILS=1`, `OTEL_LOG_TOOL_CONTENT=1` (truncated 60 KB), `OTEL_LOG_RAW_API_BODIES` (`1` inline truncated 60 KB or `file:<dir>` untruncated on disk with `body_ref`; implies consent to all three above; extended-thinking content redacted); leave all unset unless pipeline approved.
- Data-use policy opt-in sharing; Enterprise gating & beta allowlist (entire Observability suite Enterprise-only — Explorer/Judges/Campaigns/Datasets, SDK under `beta.observability.*`; Moderation API & Custom Guardrails available to standard users; logs storage requires paid tier; safety settings/feedback available to standard users; tracing beta may require org allowlist in interactive CLI; built-in tracing enabled by default; legacy Evals platform deprecated read-only 2026-10-31 shutdown 2026-11-30).
- Data-use & sharing policy (default prompts/responses not used for product improvement; sharing dataset with platform opts into "Unpaid Services" terms incl. human review; account/key/project disconnected before reviewers see/annotate; license extends to prompts incl. system instructions/cached content/files and generated responses).

## Domain L5.E — Search, Filter & Inspect Production Traffic

### Module: Traffic Inspection
- **Traffic Search Service** — hosted dashboard search/filter; log filtering by timestamp/level/content/replica.
- **Observability Logs Search Service** — args: structured filter condition `{field, op, value}` with `AND`/`OR` trees, operators (`=`/`eq`, `!=`/`ne`, `contains`, `includes`, `excludes`, `>`/`gt`, `<`/`lt`, `>=`/`gte`, `<=`/`lte`, `isnull`, `length_equals`, `starts_with`, `ends_with`, `matches` regex); filterable fields (`timestamp`, `model_name`, `last_user_message_preview`, `response_messages_preview`, `invoked_tools`, `total_time_elapsed`, `input_tokens`, `output_tokens`, `api_agent_id`, `event_id`, `correlation_id`, `first_system_message`, `metadata`); `extra_fields[]`, `page_size` — `POST /v1/observability/logs/search`.
- **Observability Traces Search Service** — `POST /v1/observability/traces/search`, `GET /v1/observability/traces/{id}`, `GET /v1/observability/traces/{id}/spans/{id}`.
- **Workflow Events Service** — args: cursor pagination — `GET /v1/workflows/events` — list workflow events.

### Module: Traffic Inspection Capabilities
- Explorer filter language (structured filter condition object with operator set listed above).
- SDK search params.
- Query design best practices (start broad time-range → add one business condition → one technical condition → scan before exporting; treat exports as snapshots with descriptive names).
- Explorer access control (restricted to Workspace administrators (Enterprise); seed Judge/Campaign from filter; export filtered slices to Datasets).
- AI Studio Logs page (filter bar, reverse-chronological, click entry for payload preview incl. full prompt/response/prior-turn context; Interactions entries link via `previous_interaction_id`).
- Traces dashboard (Logs > Traces, inspect representative workflow trace).
- OTel backend queries (filter on `session.id`).

## Domain L5.F — Evaluation, Scoring & Datasets

### Module: Judges & Evaluators
- **Judge Service** — args: `name`, `description`, `model_name`, `instructions`, `output` (`CLASSIFICATION {type, options:[{value, description}]}` or `REGRESSION {type, min, max, min_description, max_description}`), `tools` (Web Search, Code Interpreter, or `[]`) — `POST /v1/observability/judges` — auto-injects conversation history / user message / assistant response / available tools; Jinja2 vars (`{{ conversation_history }}`, `{{ user_message }}`, `{{ assistant_message }}`, `{{ system_prompt }}`, `{{ available_tools }}`, `{{ answer_type_definition }}`, `{{ properties.* }}`); versioning via `base_revision`/`up_revision`/`down_revision`.
- **Trace Grader Service** — LLM-as-a-judge / model grader with rubrics, created from Logs > Traces; scores traces with structured criteria.

### Module: Eval Runs / Campaigns
- **Campaign Service** — args: `name`, `description`, `judge_id`, `search_params`, `max_nb_events` (100-10,000) — `POST /v1/observability/campaigns` — background async; annotations written back into Explorer linked to original events; `fetch_status()`, `list_events()`; filter cannot change after start; deleting a Campaign does not necessarily lose annotations.
- **Eval Run Service** — run graders over a dataset; config: model, date_range, tool_calls filter, test_criteria, `grade_all`.
- **Batch Re-run Service** — re-run curated dataset via Batch API.
- **Test Case Upload Service** — args: agent_id, CSV body — `POST /v1/agent/{id}/test_case`.
- **Evaluate Service** — args: agent_id, evaluation config — `POST /v1/agent/{id}/evaluate` — run evaluation (rubric evaluations, LLM agent vulnerability testing incl. adversarial/red-team).
- **Evaluations List Service** — `GET /v1/agent/{id}/evaluations`.
- **Evaluations Export Service** — args: `evaluation_ids:[...]` — `POST /v1/agent/{id}/evaluations/export`.

### Module: Datasets
- **Dataset Service** — args: `name`, `description`; record = Conversation + Properties + Source; sources `EXPLORER`/`UPLOADED_FILE`/`DIRECT_INPUT`/`PLAYGROUND`; editable (fix messages, add expected outputs, remove duplicates) — `POST /v1/observability/datasets` — `add records`, `import_from_explorer()`, `list records`, export to JSONL; Google Dataset (curated logs, ≤1000/project, export to CSV/JSONL/Sheets); golden-set datasets with labels/expected outputs.
- **Dataset Import / Ingest Service** — from production traffic/logs; manual entry; JSONL file upload (one record per line `{"messages":[...],"properties":{"expected_output":"...","category":"..."}}`); from Playground/Campaign; `campaigns.list_events()` → `datasets.import_from_explorer()`; Import Tasks status check.
- **Dataset Export Service** — export to CSV/JSONL/Sheets/JSONL.
- **Test Case Template Service** — `GET /v1/agent/test_case/templates` — returns sample CSV.

### Module: Evaluator & Dataset Capabilities
- Evaluator types: LLM-as-a-judge subtypes (pairwise comparison, single-answer grading, reference-guided grading; watch position bias & verbosity bias; prefer pairwise or pass/fail; use most capable model; add CoT before scoring); architecture-aware evals (single-turn → instruction following + functional correctness; workflows → per-step + end-to-end; single-agent → add tool selection + data precision; multi-agent → add agent handoff accuracy); metric-based (exact match, ROUGE/BLEU, function-call accuracy, executable evals); human labelers (rank/grade 1-5, consensus votes, "show rather than tell"); classification cookbook; safety classifier; LLM-as-judge in guidance; no strategy perfect — combine types, expert human ground-truth expensive/slow.
- Maturity progression (traces → trace grading → datasets & eval runs; Explorer → Judges → Campaigns → Explorer → Datasets → re-run/improve).
- Judge best practices (single output type, classification options each need `value`+`description`, be specific in instructions, never assume Judge understands context, use boundary examples); validate Judge on 10-20 real records before scaling; validate on single record before launching full Campaign.
- Record metadata/properties (`expected_output`, `category`, `grading_guidance`, `difficulty`); referenced by Judges via `{{ properties.* }}`.
- Continuous evaluation (run evals on every change, monitor for new nondeterminism, grow eval set over time).
- Trace-grading workflow (Logs > Traces → inspect → create grader → run against traces → refine prompts/tools/routing/guardrails).
- Dataset best practices (explicit names with scope/date, track sources/curation, version baselines, don't mix unrelated tasks, check class balance, freeze baseline between uses).

## Domain L5.G — Feedback & Improvement

### Module: Feedback & Improvement
- **Feedback Upload Service** — args: `classification`, `reason`, `logs`, `conversation_id`, `extraLogFiles?` — `POST /v1/feedback/upload` — upload user feedback.
- **Attestation Service** — args: opt-in via `requestAttestation` capability — `POST /v1/attestation/generate` — generate attestation.
- **Auto-Review Service** — Reviewer agent evaluates approval requests; transcripts retained at platform sessions directory.

### Module: Feedback Capabilities
- Client identification (`clientInfo.name` identifies client).
- Auto-review transcripts retained.
- Attestation opt-in via `requestAttestation`.

## Domain L5.H — Production Monitoring & Improvement Loop

### Module: Improvement Loop
- **Improvement Loop Service** — orchestration pattern: Observe → moderate → approve → record → score → curate datasets → re-run → improve prompts/routing/fine-tuning; iterative cycle: understand risks → adjust/test → monitor; Explorer → Judges → Campaigns → Explorer filter by annotations → Datasets → re-run/improve.
- **Moderation Analysis Service** — analyze moderated content to identify trends; precision/recall tracking; iteratively refine prompts/keywords/criteria.

### Module: Safety & Adversarial Testing
- **Safety Benchmarking Service** — design safety metrics, test against eval datasets, define minimum acceptable levels.
- **Adversarial Testing / Red-Team Service** — proactively try to break the app; diverse test data; automated red-team LLM to find inputs eliciting harmful outputs.

### Module: Production Monitoring
- **Production Monitoring Service** — monitored user-feedback channel thumbs up/down; user studies with diverse users; especially when usage patterns differ from expectations.

### Module: Improvement & Monitoring Capabilities
- Mitigation techniques toolkit: blocklists; trained classifiers labeling prompts for harms/adversarial signals; unique user IDs + per-user volume limits; prompt-injection safeguards; scope-narrowing; adjust safety settings; grounding with web search for factuality with verifiable citations — disable for creative non-information-seeking use cases.
- Eval-driven development guidance (evaluate early/often; log everything to mine for eval cases; automate scoring; calibrate with human feedback; treat evaluation as continuous; data flywheel — feed eval data into reinforcement fine-tuning).

## Domain L5.I — Safety, Guardrails, Moderation & Approvals

### Module: Moderation
- **Moderation Service** — args: `input`/`messages`, `model` (e.g. `moderation-latest`), `safety_settings:[{category, threshold}]` — `POST /v1/moderations`, `POST /v1/chat/moderations` — reduce unsafe content; returns per-category scores.
- **Inline Moderation in Generation** — capability: `moderation` auto/low, `moderation_details.categories`, `moderation_stage: input|output|unknown`.
- **Moderation-via-Prompting Service** — no dedicated moderation model; prompt model on Messages API with content + category list, instruct strict JSON verdict, parse result; supports binary/multi-level risk 0-3/category definitions/batch processing; recommended small model, `max_tokens=200` single / `2048` batch, `temperature=0`, `output_config.format` json_schema; applies to both input and output moderation.

### Module: Moderation Capabilities
- **Moderation categories (capability group)**:
  - 11 fixed: Sexual, Hate and Discrimination, Violence and Threats, Dangerous, Criminal, Self-Harm, Health, Financial, Law, PII, **Jailbreaking**.
  - 4 adjustable + built-in: Harassment, Hate speech, Sexually explicit, Dangerous; built-in non-adjustable child safety, prohibited content.
  - Customizable: Child Exploitation, Conspiracy Theories, Hate, Indiscriminate Weapons, Intellectual Property, Non-Violent Crimes, Privacy, Self-Harm, Sex Crimes, Sexual Content, Specialized Advice, Violent Crimes (append e.g. "Underage Posting").
  - Fully custom (define categories in guardrail agent).
- **Decision granularity**: binary `violation:bool`/`violated`; multi-level integer risk 0-3 (0=No risk, 1=Low, 2=Medium, 3=High); probability enum + numeric probability/severity scores (`SafetyRating` `category`/`probability` HIGH/MEDIUM/LOW/NEGLIGIBLE/`probabilityScore`/`severity`/`severityScore`/`blocked`); classification label set.
- **Guardrail response shapes**: guardrail success response with per-category `score`/`violated`; blocked response HTTP 403 with `decisions` per category (threshold, score, violated); `promptFeedback` `blockReason` (`BLOCK_REASON_UNSPECIFIED`/`SAFETY`/`OTHER`/`BLOCKLIST`/`PROHIBITED_CONTENT`/`IMAGE_SAFETY`) + `safetyRatings[]`; built-in AUP harmlessness training may flag content regardless of prompt.
- Logging limitations (not loggable on some surfaces: image generation, video generation, embedding, Robotics, inputs with videos/GIFs/PDFs, Public Preview Agents).

### Module: Guardrails
- **Guardrail Engine — Input Guardrails** — fast validation step before expensive/side-effecting part; separate guardrail agent classifies input with structured output schema returning `tripwire_triggered`; raises `InputGuardrailTripwireTriggered`; `Agent.input_guardrails` / `@input_guardrail` decorator.
- **Guardrail Engine — Output Guardrails** — validates/redacts final output before leaves system; runs only for agent producing final output; same `GuardrailFunctionOutput`/tripwire contract; `Agent.output_guardrails`.
- **Guardrail Engine — Tool Guardrails** — checks arguments or results around a specific function tool call; attached to tool not agent.
- **Custom Guardrails Service** — declarative `guardrails` array on request each with `moderation_llm_v2` config: `custom_category_thresholds` 0-1 per category (`1` disables), `ignore_other_categories`, `action:"block"`, `model_name` override, `block_on_error`; input-only; triggered blocks HTTP 403; multiple guardrail objects; blocked if any triggers; attachable inline on `chat.complete`/`conversations.start`/agent-level `agents.create` inherited overridable per-request.
- **Safety Settings Service** — per-request per-category adjustable filters gate prompts; blocking based on probability of content being unsafe; built-in non-adjustable protections child safety `PROHIBITED_CONTENT` always block; default threshold for newer models `Off`.

### Module: Guardrail Capabilities
- **Guardrail attachment points**: input guardrail (before expensive/side-effecting part); output guardrail (final-output agent only); tool guardrail (attached to tool not agent); `runInParallel` false=blocking sequential / true=parallel lower-latency speculative; tripwire boolean (`tripwireTriggered`/`tripwire_triggered`); guardrail workflow boundaries (input guardrails run only for first agent in chain; output guardrails only for final-output agent; tool guardrails only on attached tool — for per-tool validation in manager-style workflows attach validation to tool not agent); decision to use multi-agent architecture should be driven by evals (starting multi-agent adds unnecessary complexity).

### Module: PII & Redaction
- **PII Detection & Redaction Service** — Text PII / Conversation PII / Document PII; configurable redaction policies `characterMask`/`noMask`/`entityMask`/`syntheticReplacement`; `piiCategories`, `confidenceScoreThreshold` with `default` + `overrides[]`, `disableEntityValidation`, `excludeExtractionData`.

### Module: Safety Filters (media)
- **Safety Filters & Watermarking Service** — image/video safety filters; `personGeneration` `allow_all`/`allow_adult`/`dont_allow`; invisible watermarking on all generated images and video outputs.

### Module: Refusal Handling
- **Refusal Fallback Service** — server-side `fallbacks` parameter up to 3 entries each `{model, max_tokens?, thinking?}`; tried in order; `usage.iterations` records every attempt; only safety-classifier decline triggers fallback; rate limits/overload returned as-is; SDK middleware `BetaRefusalFallbackMiddleware` with shared `BetaFallbackState` to pin follow-ups to accepting model.

### Module: Refusal Handling Capabilities
- Refusal HTTP 200 with `stop_reason:"refusal"` + `stop_details` (`cyber`/`bio`/`frontier_llm`/`reasoning_extraction`/null).

### Module: Approvals (HITL)
> Approval policy knob lives in L0.C RBAC; this L5 module documents the **admin/observability facet** of approvals (mirrors L4.G).

- **Human-in-the-Loop Approval Service** — resumable serialized state `state.approve()` + resume `run(agent, state)`; guardrails (automatic) and approvals (human) share same resumable `state` model; permission events + hooks `tool_decision`/`permission_mode_changed`/`blocked_on_user`; human reviewers alter/block content; `confirmation_status:pending` + `tool_confirmations[]` / `DeferredToolCallsException`; `interaction.requires_action`.
- **Tool-Approval SDK Mechanics** — capability: `RunContext` + `DeferredToolCallsException`; `dc.confirm()`/`dc.reject()` per pending tool call; stateless via `deferred.to_dict()`; works for Connectors, built-ins, local functions; `tool_confirmations:[{tool_call_id, confirmation:"allow"|"deny"}]` resume with batch OK.
- **Computer-Use Safety Policies Service** — risky actions surface `safety_decision:{explanation, decision:"require_confirmation"}`; confirm via `safety_acknowledgement:true` in action result; policy categories `FINANCIAL_TRANSACTIONS`/`SENSITIVE_DATA_MODIFICATION`/`COMMUNICATION_TOOL`/`ACCOUNT_CREATION`/`DATA_MODIFICATION`/`USER_CONSENT_MANAGEMENT`/`LEGAL_TERMS_AND_AGREEMENTS`; disable via `disabled_safety_policies`.

### Module: Prompt Injection Defense
- **Prompt Injection Defense Service** — capability group: developer-message precedence; structured outputs for data flow; tool approvals; guardrails for user inputs; trace graders/evals; `enable_prompt_injection_detection`; dedicated Jailbreaking category; domain filtering `allowed_domains`/`blocked_domains`; URL validation restricting fetch to URLs already in context; path-traversal protection for memory/file tools; shell allowlists; secret injection with placeholder names (`domain_secrets`); treat all tool/web/page content as untrusted — only direct user instructions count as permission.

## Domain L5.J — Model Lifecycle Admin

### Module: Model Lifecycle Admin
- **Model Allowlist/Denylist Service** — args: `mode:"allow_list"|"deny_list"`, `model_ids[]` — `PATCH /v1/organization/projects/{id}/model_permissions` — project-level model permissions.

### Module: Model Lifecycle Admin Capabilities
- Model lifecycle stages (Experimental/Preview/Stable GA, dated snapshots + `-latest` aliases, deprecation windows).
- Reasoning / Verbosity / Phase admin: `reasoning.effort` none/low/medium/high/xhigh; `text.verbosity` low/medium/high; `phase` labels `commentary`/`final_answer`; `tool_search`/`defer_loading` deferred tool loading; `reasoning.encrypted_content` stateless handoff of reasoning items for ZDR.

## Domain L5.K — Caching Diagnostics (admin)

### Module: Caching Diagnostics
- **Cache Diagnostics Service** — per-request fingerprint comparison reveals `cache_miss_reason`; beta, ZDR-eligible.

## Domain L5.L — Enterprise Analytics Portals / Supervision Surfaces

### Module: Analytics Portals
- **Enterprise Analytics Portal** — adoption rate, factor metrics, coin spend; workspace/team/user views; avoids exposing individual user data to admins.
- **Supervision Surfaces Service** — Todos panel, reasoning summary, tool-call transparency — which tool, inputs, outputs, status pending/succeeded/failed, Stop button.
- **Stored Interactions Viewer** — Logs page in studio; deletable.

## Domain L5.M — Telemetry Backend Integrations

### Module: Telemetry Backend Integrations
- **External Governance (WXG) Integration Service** — args: per-environment `enable_wxg_integration` — returns `wxg_metrics_url`; monitoring setup.
- **External Telemetry Backend Integration** — Developer Edition observability backends; ADK flags for langfuse/IBM-telemetry integration.
- **LLM Analytics Service** — `/v1/llm-analytics` config CRUD.
- **Agentic Control Plane** — UI dashboards for alerts, incidents, insights.

## Domain L5.N — Cost & Usage Lookup

### Module: Cost & Usage Lookup
- **Generation Stats Service** — args: `id` (also in `X-Generation-Id` response header) — `GET /api/v1/generation?id={id}` — async token counts + cost lookup.

---

# APPENDIX — Cross-Layer Notes

## A.1 Layer dependency summary (v2)

- L0 → all (fundamental technical services).
- L0 → L1 (inference serving built on L0 compute/network/storage/identity/billing).
- L1 → L2 (L2 endpoints served by L1 deployments).
- L2 → L3 (L3 modality products call L2 primitives + L0 files).
- L2 + L3 + L0 → L4 (agents call L2 + L3 + L0 primitives).
- L5 → all (governance built on L0).

## A.2 Design choices recap

See S.33 in the Platform Architecture Standards chapter for cross-layer design notes (state separation, two text paradigms, understanding vs generation, server-managed vs in-process loop, BYO OTLP vs hosted dashboard, sync/streaming/async/batch, stateful vs stateless replay, sandboxes in three places).

## A.3 Quick capability → layer mapping

(Regenerated vendor-neutral from v1's vendor-attributed Quick Decision Guide.)

| If you want to… | Layer · Module |
|---|---|
| Provision GPUs / deploy a model | L1 · Deployment CRUD |
| Serve a chat / completion / responses call | L2 · Generation API Surfaces |
| Generate / edit images | L3 · Image Generation / Image Editing |
| Transcribe / synthesize voice | L3 · Speech-to-Text / Text-to-Speech |
| Parse / extract / search documents | L3 · Document Understanding / Query Time |
| Build an agent with tools / sessions / hooks | L4 · Agent Definition / Sessions / Tools / Hooks |
| Connect external tools via MCP | L4 · Connectors / MCP |
| Orchestrate multiple agents | L4 · Multi-Agent Orchestration |
| Schedule an agent / cron a workflow | L0 · Workflows & Scheduled Jobs (L4.K for the agentic flow) |
| Manage billing / quotas / rate limits | L0 · Billing, Usage & Spend / Quotas & Rate Limits |
| Manage identity / keys / RBAC / workspaces | L0 · Identity, Auth & Keys / Tenancy & Workspaces / RBAC |
| Store / retrieve files | L0 · Files & Object Storage |
| Build a vector / graph store | L0 · Storage & Databases |
| Provision a sandbox / environment | L0 · Environments & Sandboxes |
| Manage secrets / vaults / connections | L0 · Secrets & Credentials (Vault) |
| Emit / search telemetry / traces / logs | L5 · Observability Export / Signals / Traffic Inspection |
| Run evals / judges / campaigns / datasets | L5 · Evaluation, Scoring & Datasets |
| Moderate / guardrail / approve | L5 · Moderation / Guardrails / Approvals (HITL) |
| Administer model lifecycle / analytics | L5 · Model Lifecycle Admin / Analytics Portals |

---

> **End of architecture_v2.md.** This document is vendor-neutral and reorganized into L0 (fundamental technical services), L1-L5 (built on L0), with transversal conventions extracted into the Platform Architecture Standards chapter. Real services are documented with name, arguments, and description; capabilities are grouped under service families; modules group closely-related services. No provider information, vendor capability matrices, or cross-vendor comparison tables are present.
