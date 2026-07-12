# Unified Text Generation & Conversation AI API Specification

> **Aggregated from:** OpenAI, Google Gemini, Anthropic Claude, Mistral, xAI Grok, OpenRouter, and Azure AI Language
> **Purpose:** A single, exhaustive specification that encompasses every text-generation, conversation, and NLP-analysis capability, concept, parameter, and processing step described across the seven platform studies in `./platform-studies/text/`.

---

## How to Read This Document

This document is written for end users — developers, product managers, and architects who want to understand the **full landscape** of text & conversation AI capabilities before choosing a provider or building a system. It is organized as follows:

1. **Part I — Concepts & Vocabulary** — An approachable introduction to every concept you will encounter, with plain-language explanations and a glossary that maps the many different names providers use for the *same* idea.
2. **Part II — The Exhaustive Processing Pipeline** — Every capability ordered into a single exhaustive end-to-end pipeline, from authentication through delivery. Each stage lists the alternative approaches available and the synonyms used by each platform.
3. **Part III — The Unified API Specification** — A detailed, provider-agnostic API reference that describes every endpoint, parameter, and data structure needed to implement a "super complete" text & conversation AI platform.

---

# Part I — Concepts & Vocabulary

## 1. What Is Text & Conversation AI?

Text and Conversation AI is the family of technologies that let machines **understand text**, **generate text**, **hold multi-turn conversations**, and **transform unstructured text into structured data**. A complete text & conversation AI platform touches eight broad domains:

| Domain | One-line description |
|--------|----------------------|
| **Text Generation** | Generate prose, code, JSON, or structured output from a prompt (single-turn or multi-turn). |
| **Conversation Management** | Maintain context across multiple turns with server-side state, manual replay, or persistent conversation objects. |
| **Reasoning / Thinking** | Models that generate internal reasoning tokens before producing visible output; control the depth of reasoning. |
| **Structured Outputs** | Force model output to conform to a JSON Schema, JSON object, or custom grammar (constrained decoding). |
| **Function / Tool Calling** | Let the model call external functions or built-in tools (web search, code execution, file search, MCP) during generation. |
| **Grounding & Citations** | Ground responses in provided documents or search results; return cited passages and references. |
| **Multimodal Input** | Combine text with images, PDFs, documents, video, and audio as model input. |
| **NLP Analysis (Classical)** | Pre-trained NLP tasks: language detection, NER, PII detection/redaction, sentiment, summarization, key phrase extraction, entity linking, classification, question answering, conversation understanding. |

A single product may span several domains. For example, a "chat → tool calls → structured output → grounded answer" pipeline chains conversation management, function calling, structured outputs, and grounding.

## 2. Core Concepts (Provider-Agnostic)

### 2.1 The Two Paradigms: Generative LLMs vs Classical NLP

The platforms in this study fall into two fundamentally different paradigms:

- **Generative LLM platforms** — OpenAI, Google Gemini, Anthropic Claude, Mistral, xAI Grok, OpenRouter. These expose large language models that generate text from prompts, support multi-turn conversations, reasoning, structured outputs, tool calling, and multimodal input. They are general-purpose: you steer behavior through prompts and schemas rather than choosing a fixed "feature."

- **Classical NLP platforms** — Azure AI Language. This exposes a suite of pre-trained, task-specific NLP models accessed through a small number of REST endpoints. Each task (NER, PII, sentiment, summarization, etc.) is selected by a `kind` discriminator. Some tasks support custom training (Custom NER, Custom Text Classification, CLU, CQA). These are specialized: you call a dedicated endpoint for each capability.

A unified platform may offer **both** paradigms: use generative LLMs for conversation, reasoning, and flexible extraction; use classical NLP for high-throughput, deterministic, task-specific pipelines (PII redaction, language detection, sentiment at scale).

### 2.2 Model

The **model** is the underlying AI. Providers expose multiple models tuned for different trade-offs:

- **Flagship vs fast/lightweight** — e.g. OpenAI `gpt-5.6` vs `gpt-4o-mini`, Anthropic `claude-opus-4-8` vs `claude-haiku-4-5`, Google `gemini-3.1-pro-preview` vs `gemini-2.5-flash-lite`, Mistral `mistral-large-2512` vs `ministral-3b-2512`, xAI `grok-4.5` (reasoning) vs standard Grok models.
- **Reasoning vs non-reasoning** — Some models generate internal reasoning tokens before output (OpenAI GPT-5.x/o-series, Google Gemini 3.x/2.5.x, xAI `grok-4.5`, Anthropic Claude with extended thinking, Mistral with `reasoning_effort`). Others do not.
- **Context window size** — 128k (OpenAI typical), 200k (Anthropic most models), 1M (Anthropic Opus 4.6+/Sonnet 5/Fable 5/Mythos 5; Google Gemini 1M+).
- **Versioning** — pinned dated snapshots (`gpt-5.5-2026-04-23`, `claude-opus-4-5-20251101`, `mistral-large-2512`) vs rolling aliases (`latest`, `~openai/gpt-latest`, `mistral-large-latest`).
- **Model variants (OpenRouter)** — suffix-based routing: `:nitro` (fastest), `:floor` (cheapest), `:thinking` (extended reasoning), `:free`, `:extended` (larger context), `:exacto` (best tool-calling accuracy).

### 2.3 API Surfaces — The Evolution from Chat Completions to Responses

Most generative LLM providers expose two API surfaces — a legacy one and a newer recommended one:

| Provider | Legacy API | Recommended API |
|----------|-----------|-----------------|
| OpenAI | Chat Completions (`/v1/chat/completions`) | Responses (`/v1/responses`) |
| Google Gemini | generateContent (`models.generate_content`) | Interactions (`interactions.create`) |
| xAI Grok | Chat Completions (`/v1/chat/completions`, deprecated) | Responses (`/v1/responses`) |
| Anthropic | — (single Messages API) | Messages (`/v1/messages`) |
| Mistral | — (single Chat Completions API) | Chat Completions (`/v1/chat/completions`) |
| OpenRouter | Chat Completions (primary) + Anthropic Messages | Responses (Beta) + Chat Completions |
| Azure AI Language | — | Shared `:analyze-text` / `jobs` endpoints |

**Key differences between legacy and modern surfaces:**
- **Input shape**: Legacy uses `messages[]` (flat array of role+content). Modern uses `input` (string, typed item array, or step array).
- **Output shape**: Legacy returns `choices[].message` (flat). Modern returns typed `output[]` / `steps[]` arrays (message items, reasoning items, function-call items, thought steps).
- **System guidance**: Legacy uses `role: "system"` in messages. Modern uses top-level `instructions` (OpenAI/xAI) or `system_instruction` (Google) parameter.
- **State management**: Legacy is stateless (replay history each turn). Modern supports `previous_response_id` / `previous_interaction_id` chaining and server-side storage.
- **Reasoning**: Modern surfaces have richer reasoning support (typed reasoning items, encrypted content, thought steps with signatures).

### 2.4 Message / Item / Step / Content Block

A unit of conversation context. The naming varies:

| Concept | OpenAI | Google | Anthropic | Mistral | xAI | OpenRouter | Azure |
|---------|--------|--------|-----------|---------|-----|------------|-------|
| Conversation unit | Message (Chat) / Item (Responses) | Content (generateContent) / Step (Interactions) | Message (with content blocks) | Message (with chunks) | Message (Chat) / Item (Responses) | Message (Chat/Responses/Anthropic) | Document / Conversation item |
| Typed sub-element | Content part (text, image, file) | Part (text, inline_data, file_data) | Content block (text, image, document, search_result, thinking, tool_use, tool_result) | Content chunk (TextChunk, ThinkChunk, ReferenceChunk, image_url) | Content part (input_text, input_file, output_text) | Content part (text, image_url) | — (feature-specific fields) |

### 2.5 Role / Instruction Hierarchy

Messages have differing levels of authority:

| Role | OpenAI | Google | Anthropic | Mistral | xAI | OpenRouter | Azure |
|------|--------|--------|-----------|---------|-----|------------|-------|
| System / developer | `developer` (Responses) / `system` (Chat) | `system_instruction` (top-level) | `system` (top-level param, not in messages) | `system` (first message) | `system` or `instructions` (top-level) | `system` / `developer` | — (N/A for classical NLP) |
| User | `user` | `user` | `user` | `user` | `user` | `user` | — (input documents) |
| Assistant / model | `assistant` | `model` | `assistant` | `assistant` | `assistant` | `assistant` | — |
| Tool | `tool` (Chat) / `function_call_output` (Responses) | `function_result` step | `tool_result` block | `tool` | — | `tool` | — |

**Priority order**: system/developer > user > assistant (OpenAI, OpenRouter). Anthropic: top-level `system` > mid-conversation `system` (Opus 4.8) > user > assistant. Google: `system_instruction` > user. Mistral: `system` is optional context, no strict hierarchy documented.

### 2.6 Prompt / Input

The text (and/or structured content) given to the model. May be:
- A plain string (simplest case).
- An array of messages/items (multi-turn conversation).
- An array of typed steps (stateless history replay, Google Interactions).
- A document with text + images + files (multimodal input).

**Prompt engineering** is the process of writing effective instructions. Best practices: pin model snapshots in production, build test/eval suites, store prompts in code, use typed schemas for dynamic values.

### 2.7 Response / Output

The object returned by the API:

| Provider | Non-streaming response | Text convenience helper |
|----------|----------------------|------------------------|
| OpenAI Responses | `output[]` (typed Items) | `response.output_text` |
| OpenAI Chat | `choices[].message` | `choices[0].message.content` |
| Google Interactions | `interaction.steps[]` | `interaction.output_text` |
| Google generateContent | `candidates[].content.parts[]` | `response.text` |
| Anthropic | `content[]` (typed blocks) | `content[0].text` (simple case) |
| Mistral | `choices[].message` (string or chunk list) | `choices[0].message.content` |
| xAI Responses | `output[]` (typed items) | `output_text` content |
| OpenRouter | `choices[].message` (normalized) | `choices[0].message.content` |
| Azure | `results.documents[]` (feature-specific) | — |

### 2.8 Context Window

The maximum tokens (input + output + reasoning) usable in a single request:

| Provider / model | Context window |
|-----------------|---------------|
| OpenAI `gpt-4o` | 128k |
| OpenAI GPT-5.x | varies (128k+) |
| Anthropic most models | 200k |
| Anthropic Opus 4.6+/Sonnet 5/Fable 5/Mythos 5/Mythos Preview | 1M |
| Google Gemini | 1M+ |
| Mistral | 32k–128k (varies by model) |
| xAI Grok | varies (131k for `grok-4`) |
| Azure AI Language | per-document limits, not token-windowed |

**Context rot**: accuracy/recall degrade as token count grows. Strategies: prompt caching, context compaction, context editing, RAG instead of stuffing everything.

### 2.9 Reasoning / Thinking / Extended Thinking

Models that generate internal **reasoning tokens** before producing visible output. Synonyms: *Reasoning* (OpenAI, xAI, OpenRouter), *Thinking* (Google, Anthropic), *Extended Thinking* (Anthropic), *Adjustable Thinking* (Mistral), *Test Time Computation* (Mistral).

| Aspect | OpenAI | Google | Anthropic | Mistral | xAI | OpenRouter |
|--------|--------|--------|-----------|---------|-----|------------|
| Control parameter | `reasoning.effort` (low/medium/high) | `thinking_level` (minimal/low/medium/high) | `output_config.effort` (low/medium/high/xhigh/max) + `thinking` (enabled/adaptive/disabled) | `reasoning_effort` (high/none) | `reasoning.effort` (low/medium/high) | `reasoning` object (effort or max_tokens) |
| Token budget | — (effort-based) | — (level-based) | `thinking.budget_tokens` (manual mode) or `output_config.task_budget` (loop) | — | — | `reasoning.max_tokens` (maps to Anthropic/Gemini) |
| Can disable? | Yes (`reasoning: none`) | Yes (Gemini 2.5-flash-lite: off; others: minimal) | Yes (`type: disabled`); mandatory on Fable 5/Mythos 5 | Yes (`reasoning_effort: none`) | No (`grok-4.5` cannot disable) | Depends on model (`mandatory` flag) |
| Encrypted reasoning | `include: ["reasoning.encrypted_content"]` | `signature` on thought steps | `signature` on thinking blocks / `redacted_thinking` | — | `include: ["reasoning.encrypted_content"]` | `reasoning_details` with encrypted type |
| Summary/trace visibility | `reasoning.summary` (auto/concise/detailed) | `thinking_summaries` (none/auto) | `display` (summarized/omitted) | ThinkChunk in content | `reasoning_content` (Chat) / summary deltas (Responses) | `reasoning` field / `reasoning_details` |

**Reasoning token billing**: reasoning tokens are counted and billed as output tokens in all providers. They count against `max_output_tokens` / `max_tokens` and the context window.

### 2.10 Structured Outputs

Force model output to conform to a JSON Schema or JSON object. Synonyms: *Structured Outputs* (OpenAI, xAI, Google), *JSON Mode* (OpenAI, Mistral, xAI), *Custom Structured Outputs* (Mistral), *Constrained Decoding* (Anthropic).

| Aspect | OpenAI | Google | Anthropic | Mistral | xAI | OpenRouter |
|--------|--------|--------|-----------|---------|-----|------------|
| Schema enforcement | `text.format` (Responses) / `response_format` (Chat) with `json_schema` + `strict: true` | `response_format` with `schema` (no `strict` flag) | `output_config.format` with `json_schema` | `client.chat.parse()` with Pydantic/Zod/JSON Schema | `text.format` (Responses) / `response_format` (Chat) with `json_schema` + `strict: true` | `response_format` (normalized across providers) |
| JSON mode (no schema) | `json_object` type | `response_mime_type: application/json` | — | `response_format: {type: json_object}` | `json_object` type | `json_object` type |
| Strict mode | `strict: true` (additionalProperties: false, all fields required) | — (no strict flag) | `strict: true` on tools; grammar compilation for format | Schema-enforced via `parse()` | `strict: true` | `strict: true` |
| Grammar/CFG constraint | — (custom tools with CFG) | — | Constrained decoding (compiled grammars) | — | — | `grammar` type (OpenAPI) |
| SDK helpers | `client.responses.parse()` / `pydantic_function_tool` / `zodFunction` | Pydantic/Zod via SDK | `client.messages.parse()` | `client.chat.parse()` | `chat.parse(PydanticModel)` | Pydantic/Zod |

### 2.11 Function Calling / Tool Calling

Let the model call external functions or built-in tools. Synonyms: *Function Calling* (OpenAI, Google, Mistral), *Tool Calling* (OpenAI, OpenRouter), *Tool Use* (Anthropic).

**Tool types:**
- **Function tools** — defined by JSON Schema; model emits structured arguments.
- **Custom tools** — free-form text input/output (OpenAI).
- **Built-in / server tools** — provider-hosted: web search, file search, code execution, image generation, URL context, remote MCP.

**Tool calling flow**: (1) define tools → (2) model emits tool call → (3) execute tool → (4) return result → (5) model produces final response (or more tool calls).

| Aspect | OpenAI | Google | Anthropic | Mistral | xAI | OpenRouter |
|--------|--------|--------|-----------|---------|-----|------------|
| Tool definition shape (modern) | Flat: `{type, name, description, parameters, strict}` | Flat: `{type, name, description, parameters}` | `{type, name, description, input_schema, strict}` | Externally tagged: `{type, function: {name, ...}}` | Flat (Responses) | Normalized |
| Tool definition shape (legacy) | Externally tagged: `{type, function: {...}}` | — | — | Externally tagged | Externally tagged (Chat) | Externally tagged |
| `tool_choice` | auto/required/none/specific/allowed_tools | auto/any/none/validated/allowed_tools | auto/any/tool/none (+ disable_parallel) | auto/any/none | auto/required/none | auto/none/specific |
| Parallel tool calls | `parallel_tool_calls: true` (default) | Yes (parallel + compositional) | `disable_parallel_tool_use` option | — | — | — |
| Built-in tools | web_search, file_search, code_interpreter, computer_use, image_generation, MCP | google_search, url_context, code_execution, file_search, mcp_server | — (client & server tools) | — (document_library on agents) | search, code exec, document search | web_search, web_fetch, bash, files, etc. |
| Tool namespaces | `{type: namespace, name, tools: [...]}` | — | — | — | — | — |
| Tool search (deferred loading) | `tool_search` (GPT-5.4+) | — | — | — | — | — |
| Strict mode for tools | `strict: true` | `validated` (preview) | `strict: true` | `strict: true` | Implicitly true | `strict: true` |

### 2.12 Grounding, Citations & RAG

Grounding responses in source documents or search results, with cited passages.

| Approach | Providers | Description |
|----------|-----------|-------------|
| **Document citations** | Anthropic (`citations: {enabled: true}` on document blocks) | Ground in provided PDFs/text; return `cited_text` with char/page/block locations |
| **Search result citations** | Anthropic (`search_result` blocks), Mistral (`ReferenceChunk` via tool calls) | RAG-style citations from search results |
| **Built-in web search** | OpenAI (`web_search`), Google (`google_search`), xAI (search), OpenRouter (`web_search`) | Model searches the web and grounds answers |
| **Managed RAG** | Mistral (Libraries + `document_library` tool), Azure (CQA knowledge base) | Upload documents → provider ingests/vectorizes/searches → grounded answers |
| **RAG from scratch** | Mistral (Embeddings + vector DB + chat), OpenAI (file_search), Google (context caching) | Build your own retrieval pipeline |
| **Context caching** | Google (context caching for large files), Anthropic (prompt caching), OpenRouter (prompt caching) | Cache large context to reduce cost of repeated queries |

### 2.13 Streaming

Incremental delivery of responses via Server-Sent Events (SSE). Set `stream: true`.

| Provider | Streaming format | Event types |
|----------|-----------------|-------------|
| OpenAI Chat | `delta` chunks | `delta.content`, `delta.tool_calls` |
| OpenAI Responses | Typed SSE events | `response.created`, `response.output_text.delta`, `response.completed`, `response.function_call_arguments.delta` |
| Google Interactions | Typed SSE events | `interaction.created`, `step.start`, `step.delta` (text/thought_summary/thought_signature/arguments), `step.stop`, `interaction.completed`, `done` |
| Google generateContent | `generate_content_stream` / `streamGenerateContent` | `GenerateContentResponse` chunks |
| Anthropic | SSE events | `message_start`, `content_block_start`, `content_block_delta` (text_delta/thinking_delta/signature_delta/citations_delta/input_json_delta/compaction_delta), `content_block_stop`, `message_delta`, `message_stop`, `ping`, `error` |
| Mistral | SSE events via `client.chat.stream` | `delta.content` (string or chunk list with ThinkChunk/TextChunk) |
| xAI Responses | Typed SSE events | `response.output_text.delta`, `response.reasoning_text.delta`, `response.reasoning_summary_text.delta` |
| xAI Chat | `delta` chunks | `delta.content`, `delta.reasoning_content` |
| OpenRouter Chat | `delta` chunks | `delta.content`, `delta.reasoning_details`; usage in final chunk |
| OpenRouter Responses | Typed SSE events | `response.created`, `response.content_part.delta`, `response.reasoning.delta`, `response.done` |

### 2.14 Conversation State Management

How context is maintained across turns:

| Method | Providers | Description |
|--------|-----------|-------------|
| **Manual replay** | All generative LLMs | Replay full message/item/step history each turn (stateless API) |
| **`previous_response_id`** | OpenAI (Responses), xAI (Responses), Google (`previous_interaction_id`) | Server manages prior context; only send new turn |
| **Conversations API** | OpenAI (`/v1/conversations`) | Persistent conversation object across sessions/devices |
| **Conversations API (beta)** | Mistral (`/v1/conversations`) | Server-managed conversation state with tools |
| **Prompt caching** | Anthropic, Google, OpenAI, OpenRouter, xAI | Cache stable prefixes to reduce cost/latency of replaying history |
| **Context compaction** | Anthropic (`compact_20260112`), OpenAI (`/responses/compact`), xAI (`/v1/responses/compact`) | Server-side summarization of old context near window limit |
| **Context editing** | Anthropic (`clear_tool_uses`, `clear_thinking`) | Server-side clearing of tool results / thinking blocks |
| **Encrypted reasoning replay** | OpenAI, xAI, Anthropic, Google, OpenRouter | Pass encrypted reasoning blobs back for stateless + reasoning |
| **Mid-conversation system messages** | Anthropic (Opus 4.8) | Append `system` role messages mid-conversation without invalidating cache |

### 2.15 Prompt Caching

Cache stable prompt prefixes to reduce cost and latency.

| Provider | Mechanism | TTLs | Min tokens |
|----------|-----------|------|------------|
| OpenAI | Automatic (Responses API); `prompt_cache_breakpoint` (GPT-5.6+) | 30 min (explicit) | 1024 |
| Anthropic | `cache_control` (automatic top-level or explicit per-block, max 4 breakpoints) | 5 min (default), 1 hour | 512–4096 (varies by model) |
| Google | Context caching (cache files, pay per hour) | Per-hour storage | 1024 (Flash) / 4096 (Pro) |
| xAI | Automatic (`cached_tokens`, `prompt_cache_key` for sticky routing) | — | — |
| OpenRouter | Provider sticky routing + `cache_control`; automatic for most providers | 5 min / 1 hour (Anthropic) | Varies |
| Mistral | — (no explicit prompt caching documented) | — | — |

### 2.16 Multimodal Input

Combining text with images, PDFs, documents, video, and audio:

| Modality | OpenAI | Google | Anthropic | Mistral | xAI | OpenRouter |
|----------|--------|--------|-----------|---------|-----|------------|
| Images | `input_image` (Responses) / image content (Chat) | Image parts (inline_data / file_data / URL) | `image` content blocks (base64/URL/file_id) | `image_url` content parts (URL/base64) | `input_image` parts | `image_url` content parts |
| PDF / documents | `input_file` (Responses) / `file` (Chat) | Document parts (PDF, etc.) | `document` content blocks (base64/URL/file_id) | — (Document AI separate) | `input_file` parts (URL/file_id) | — |
| Video | — | Video parts (inline_data / file_data) | — | — | — | — |
| Audio | — | Audio parts | — | — | — | — |
| Files API | `/v1/files` (purpose: user_data) | Files API (upload → uri) | `/v1/files` (beta) | `/v1/files` (batch) | Files API (input + output storage) | — (provider-level) |

### 2.17 Moderation & Safety

| Provider | Mechanism |
|----------|-----------|
| OpenAI | `moderation` (auto/low); `moderation_details.categories`; refusal detection in structured outputs |
| Anthropic | Safety classifiers → `stop_reason: "refusal"` + `stop_details`; server-side `fallbacks` to alternate model |
| Mistral | `guardrails` array (inline input moderation, HTTP 403 on violation); Moderation API (`/v1/moderations`, 11 categories); `safe_prompt` (legacy) |
| Google | Safety filters; `personGeneration` controls (Veo); SynthID watermarking |
| xAI | `respect_moderation` flag |
| OpenRouter | `moderation` plugin; provider-level moderation |
| Azure | PII detection/redaction (text/conversation/document) |

### 2.18 Batch Processing

Asynchronous bulk processing at a discount:

| Provider | Endpoint | Discount | Limits |
|----------|----------|----------|--------|
| OpenAI | `/v1/batches` (targets various endpoints) | — | — |
| Anthropic | `/v1/messages/batches` | 50% | 100k requests or 256 MB; expire 24h |
| Mistral | `/v1/batch/jobs` | 50% | 1M requests (file) / 10k (inline) |
| xAI | `deferred: true` (Chat Completions) → poll | — | — |

### 2.19 Embeddings

Vector representations of text for retrieval, clustering, semantic search, RAG:

| Provider | Model(s) | Dimensions |
|----------|----------|------------|
| OpenAI | `text-embedding-3-small`, `text-embedding-3-large` | 1536 / 3072 |
| Anthropic | Voyage AI (third-party): `voyage-4-large`, `voyage-4`, `voyage-4-lite`, `voyage-4-nano` | 1024 (default) / 256 / 512 / 2048 |
| Mistral | `mistral-embed` | — |
| OpenRouter | Unified `/api/v1/embeddings` across providers | Varies by model |
| Google | — (separate Embeddings API) | — |
| xAI | — | — |
| Azure | — (separate service) | — |

### 2.20 Classical NLP Tasks (Azure AI Language)

Pre-trained, task-specific NLP capabilities accessed through shared REST endpoints:

| Task | Description | `kind` |
|------|-------------|--------|
| Language Detection | Detect primary language (100+ languages, ISO 639-1, script detection) | `LanguageDetection` |
| Named Entity Recognition (NER) | Extract & categorize entities (Person, Organization, Location, DateTime, etc.) | `EntityRecognition` |
| Custom NER | Train custom entity extraction on domain-specific labeled data | `CustomEntityRecognition` |
| PII Detection (text) | Detect & redact PII in text strings; configurable redaction policies | `PiiEntityRecognition` |
| PII Detection (conversation) | Detect & redact PII in chat transcripts / speech-to-text | `ConversationalPIITask` |
| PII Detection (document) | Detect & redact PII in native document files (PDF, DOCX) | `PiiEntityRecognition` |
| Text Analytics for Health | Biomedical NER + relation extraction + entity linking (UMLS) + assertion detection + FHIR output | `Healthcare` |
| Sentiment Analysis & Opinion Mining | Document/sentence sentiment + aspect-based opinion mining (targets + assessments) | `SentimentAnalysis` |
| Key Phrase Extraction | Extract main concepts/topics as key phrases | `KeyPhraseExtraction` |
| Entity Linking | Disambiguate entities by linking to Wikipedia | `EntityLinking` |
| Summarization (extractive) | Extract salient original sentences with rank scores | `ExtractiveSummarization` |
| Summarization (abstractive) | Generate concise coherent summary sentences | `AbstractiveSummarization` |
| Summarization (query-focused) | Summarize focused on a specific query | `AbstractiveSummarization` + `query` |
| Conversation summarization | Summarize conversations: issue, resolution, chapter titles, narrative, recap, action items | `ConversationalSummarizationTask` |
| Document summarization | Summarize native document files | `ExtractiveSummarization` / `AbstractiveSummarization` |
| Custom Text Classification | Train custom single-label or multi-label classification | `CustomSingleLabelClassification` / `CustomMultiLabelClassification` |
| Conversation Language Understanding (CLU) | Predict intents & extract entities from utterances | `Conversation` |
| Custom Question Answering (CQA) | Knowledge base of Q&A pairs; query for answers | — (query-knowledgebases) |
| Orchestration Workflow | Route utterances to CLU/CQA sub-projects | `Conversation` |
| LUIS (deprecated) | Intent prediction & entity extraction (retired March 2026) | — |
| QnA Maker (deprecated) | Knowledge base Q&A (retired October 2025) | — |

### 2.21 Synchronous vs Asynchronous

| Pattern | Providers | Description |
|---------|-----------|-------------|
| **Synchronous** | All generative LLMs (text gen), Azure (single-shot text analysis), Mistral (chat) | Request blocks until response returned |
| **Streaming (SSE)** | All generative LLMs | Incremental delivery via Server-Sent Events |
| **Asynchronous (LRO / poll)** | Azure (batch/custom/document/conversation), Anthropic (Batches), Mistral (Batch), xAI (deferred) | POST → job ID → poll until done |
| **Webhook delivery** | — (not prominently featured in text platforms; common in image/video) | — |

### 2.22 Token Counting

Pre-send input token estimation:

| Provider | Endpoint | Notes |
|----------|----------|-------|
| Anthropic | `POST /v1/messages/count_tokens` | Same params as Messages; supports context_management |
| OpenAI | Tokenizer tool (tiktoken) | API endpoints available |
| Others | — (use provider tokenizer libraries) | — |

### 2.23 Billing Units

| Provider | Unit | Notes |
|----------|------|-------|
| OpenAI | Per token (input/output) | Reasoning tokens billed as output; cache reads discounted |
| Anthropic | Per token (input/output) | Cache writes 1.25× (5m) / 2× (1h); cache reads 0.1×; batch 50% discount |
| Google | Per token (input/output/thinking) | Context caching ~4× less per request |
| Mistral | Per token | Batch 50% discount |
| xAI | Per token | `cost_in_usd_ticks`; reasoning tokens billed as output |
| OpenRouter | Per token (native tokenizer) | Cost in `usage.cost`; failed/fallback not billed |
| Azure | Per transaction / per document | Custom features billed per text record |

### 2.24 Privacy, Data Residency & ZDR

- **Zero Data Retention (ZDR)** — OpenAI (enforced for ZDR orgs), Anthropic (most features ZDR-eligible; Files API and Batches are not), xAI (`store: false`).
- **Data residency** — Anthropic (`inference_geo`), xAI (EU data residency).
- **Training opt-out** — xAI (not used for training), OpenRouter (`provider.data_collection: deny`).
- **Watermarking** — Google SynthID (invisible watermark).

### 2.25 Organization & Admin

| Provider | Mechanism |
|----------|-----------|
| Anthropic | Workspaces (per-org, API keys scoped, roles, spend limits) |
| OpenAI | Organizations, projects, Conversations API |
| OpenRouter | API keys, credits, BYOK |
| Azure | Azure resource, Entra ID / Managed Identity |
| xAI | Team access, service tiers |

## 3. Cross-Provider Synonym Glossary

The table below maps the **many different names** providers use for the **same concept**.

| Unified concept | OpenAI | Google Gemini | Anthropic | Mistral | xAI | OpenRouter | Azure AI Language |
|----------------|--------|---------------|-----------|---------|-----|------------|-------------------|
| API key header | `Authorization: Bearer` | `x-goog-api-key` | `x-api-key` | `Authorization: Bearer` | `Authorization: Bearer` | `Authorization: Bearer` | `Ocp-Apim-Subscription-Key` |
| Recommended API | Responses (`/v1/responses`) | Interactions (`interactions.create`) | Messages (`/v1/messages`) | Chat Completions (`/v1/chat/completions`) | Responses (`/v1/responses`) | Chat Completions (primary) | `:analyze-text` / `jobs` |
| Legacy API | Chat Completions (`/v1/chat/completions`) | generateContent (`models.generate_content`) | — | — | Chat Completions (`/v1/chat/completions`, deprecated) | Responses (Beta) | — |
| Input | `input` (string or Item[]) | `input` (string/part[]/step[]) | `messages[]` | `messages[]` | `input` (string or message[]) | `messages[]` / `input` | `analysisInput.documents[]` |
| System guidance | `instructions` (top-level) | `system_instruction` (top-level) | `system` (top-level param) | `system` role message | `instructions` (top-level) | `system` / `developer` role | — |
| Output | `output[]` (typed Items) | `steps[]` (typed steps) | `content[]` (typed blocks) | `choices[].message` | `output[]` (typed items) | `choices[].message` | `results.documents[]` |
| Text helper | `response.output_text` | `interaction.output_text` | `content[0].text` | `choices[0].message.content` | `output_text` content | `choices[0].message.content` | — |
| Reasoning control | `reasoning.effort` | `thinking_level` | `output_config.effort` + `thinking` | `reasoning_effort` | `reasoning.effort` | `reasoning.effort` / `reasoning.max_tokens` | — |
| Reasoning tokens field | `usage.output_tokens_details.reasoning_tokens` | `total_thought_tokens` | `output_tokens_details.thinking_tokens` | ThinkChunk | `usage.output_tokens_details.reasoning_tokens` | `completion_tokens_details.reasoning_tokens` | — |
| Encrypted reasoning | `include: ["reasoning.encrypted_content"]` | `signature` on thought steps | `signature` on thinking blocks / `redacted_thinking` | — | `include: ["reasoning.encrypted_content"]` | `reasoning_details` (encrypted type) | — |
| Structured outputs | `text.format` (Responses) / `response_format` (Chat) | `response_format` with `schema` | `output_config.format` | `response_format` / `client.chat.parse()` | `text.format` (Responses) / `response_format` (Chat) | `response_format` | — |
| JSON mode | `json_object` type | `response_mime_type: application/json` | — | `json_object` type | `json_object` type | `json_object` type | — |
| Function calling | `tools` (function type) | `tools` (function declarations) | `tools` (tool_use/tool_result) | `tools` (function type) | `tools` (function type) | `tools` (function type) | — |
| Tool result | `function_call_output` Item | `function_result` step | `tool_result` block | `tool` role message | — | `tool` role message | — |
| `tool_choice` | auto/required/none/specific/allowed_tools | auto/any/none/validated/allowed_tools | auto/any/tool/none | auto/any/none | auto/required/none | auto/none/specific | — |
| Server-side state | `previous_response_id` + `store: true` | `previous_interaction_id` + `store: true` | — (stateless; use prompt caching) | — (stateless; Conversations API beta) | `previous_response_id` + `store: true` | — (stateless) | — (stateless per request) |
| Persistent conversation | Conversations API (`/v1/conversations`) | — | — | Conversations API (`/v1/conversations`, beta) | — | — | — |
| Stateless + reasoning | `store: false` + encrypted reasoning | `store: false` + resend thought signatures | Resend thinking blocks with signatures | Resend ThinkChunks | `store: false` + encrypted reasoning | `reasoning_details` replay | — |
| Context compaction | `/responses/compact` | — | `context_management.edits` (`compact_20260112`) | — | `/v1/responses/compact` | — | — |
| Context editing | — | — | `context_management.edits` (`clear_tool_uses`, `clear_thinking`) | — | — | — | — |
| Prompt caching | Automatic (Responses); `prompt_cache_breakpoint` (GPT-5.6+) | Context caching | `cache_control` (automatic/explicit, max 4 breakpoints) | — | Automatic (`cached_tokens`, `prompt_cache_key`) | Provider sticky routing + `cache_control` | — |
| Streaming | `stream: true` (typed SSE / delta chunks) | `stream=True` (typed SSE) / `generate_content_stream` | `stream: true` (SSE events) | `client.chat.stream` (SSE) | `stream: true` (typed SSE / delta chunks) | `stream: true` (SSE) | — |
| Batch processing | `/v1/batches` | — | `/v1/messages/batches` (50% discount) | `/v1/batch/jobs` (50% discount) | `deferred: true` → poll | — | — (async LRO for all) |
| Token counting | Tokenizer (tiktoken) | — | `POST /v1/messages/count_tokens` | — | — | — | — |
| File inputs | `input_file` (Responses) / `file` (Chat) | File parts (inline_data / file_data) | `document` / `image` blocks (base64/URL/file_id) | `image_url` parts | `input_file` parts (URL/file_id) | `image_url` parts | Blob Storage (custom features) |
| Files API | `POST /v1/files` (purpose: user_data) | Files API (upload → uri) | `/v1/files` (beta) | `client.files.upload` | Files API (input + output) | — | — |
| Vision (images) | `input_image` | Image parts | `image` blocks | `image_url` parts | `input_image` | `image_url` parts | — |
| PDF / document input | `input_file` (PDF detail: auto/low/high) | Document parts | `document` blocks | — | `input_file` | — | Document PII / Document summarization |
| Citations | — (via web_search/file_search) | — (via google_search) | `citations: {enabled: true}` on documents; `search_result` blocks | `ReferenceChunk` via tool calls | — | — | — |
| Managed RAG | `file_search` tool | Context caching + file upload | — | Libraries + `document_library` tool | Document search (agentic) | — | CQA knowledge base |
| Embeddings | `text-embedding-3-*` | — (separate API) | Voyage AI (third-party) | `mistral-embed` | — | Unified `/api/v1/embeddings` | — |
| Moderation | `moderation` (auto/low) | Safety filters | `stop_reason: refusal` + `fallbacks` | `guardrails` array + Moderation API | `respect_moderation` | `moderation` plugin | PII detection/redaction |
| Stop reasons | `status: completed/incomplete` (Responses) / `finish_reason` (Chat) | `finishReason` (STOP/MAX_TOKENS/SAFETY/RECITATION) | `stop_reason` (end_turn/max_tokens/stop_sequence/tool_use/pause_turn/refusal/model_context_window_exceeded) | `finish_reason` (stop/length/tool_calls) | `status: completed` | `finish_reason` (normalized: tool_calls/stop/length/content_filter/error) | `status: succeeded/failed` |
| Refusal handling | `refusal` field / `type: refusal` content | Safety blocks | `stop_reason: refusal` + `stop_details` | — | — | — | — |
| Refusal fallback | — | — | `fallbacks` parameter (server-side) | — | — | `models[]` fallbacks | — |
| Fast mode | — | — | `speed: "fast"` (Opus 4.8/4.7, beta) | — | — | `:nitro` variant | — |
| Seed / determinism | — | — | — | `random_seed` | `seed` | `seed` | — |
| Temperature | `temperature` | `temperature` (keep defaults for Gemini 3.x) | `temperature` (deprecated post-Opus 4.6) | `temperature` | `temperature` (0–2) | `temperature` (0–2) | — |
| Top-p | `top_p` | `top_p` (keep defaults) | `top_p` (deprecated) | `top_p` | `top_p` | `top_p` (0–1) | — |
| Top-k | — | `top_k` (keep defaults) | `top_k` (deprecated) | — | `top_k` | `top_k` (not OpenAI) | — |
| Max tokens | — (Responses) / `max_tokens` (Chat) | `max_output_tokens` | `max_tokens` (required) | `max_tokens` | `max_output_tokens` (Responses) / `max_tokens` (Chat) | `max_tokens` / `max_completion_tokens` | — |
| Stop sequences | — (Responses) / `stop` (Chat, not reasoning models) | — | `stop_sequences` | `stop` | `stop` (not reasoning models) | `stop` (up to 4) | — |
| Multilingual | — (system prompt controls language) | — (native multilingual) | System prompt (no special params) | — (system message) | — | — | Language Detection (100+ languages); `countryHint` |
| Organization / admin | Orgs, projects | — | Workspaces (Admin API) | — | Team access | API keys, credits, BYOK | Azure resource, Entra ID |
| Model catalog | — | — | — | — | — | `GET /api/v1/models` (400+ models, metadata) | — |
| Model routing / fallbacks | — | — | `fallbacks` (safety refusal) | — | — | `models[]` fallbacks + `provider{}` routing + router slugs | Orchestration Workflow |
| Task budgets | — | — | `output_config.task_budget` (beta) | — | — | — | — |

---

# Part II — The Exhaustive Processing Pipeline

This section orders **every** capability into a single end-to-end pipeline. The pipeline represents the full lifecycle of a text & conversation AI system, from setup through delivery. Each stage notes the **alternative approaches** available and the **synonyms** used across providers.

```
Stage 0:  Authentication & Access Control
Stage 1:  Model Selection & Catalog
Stage 2:  Input Preparation (Text, Multimodal, Files)
Stage 3:  System Instructions & Configuration
Stage 4:  Reasoning / Thinking Configuration
Stage 5:  Structured Output Configuration
Stage 6:  Function Calling & Tool Configuration
Stage 7:  Grounding, Citations & RAG Configuration
Stage 8:  Moderation & Safety Configuration
Stage 9:  Conversation State Management
Stage 10: Text Generation (Single-Turn & Multi-Turn)
Stage 11: Streaming
Stage 12: Classical NLP Analysis (Pre-trained Tasks)
Stage 13: Custom NLP Model Training & Deployment
Stage 14: Context Management (Compaction, Editing, Caching)
Stage 15: Batch Processing
Stage 16: Embeddings & Vector Operations
Stage 17: Output Processing & Delivery
Stage 18: Usage, Billing & Token Accounting
Stage 19: Organization, Admin & Asset Management
```

---

## Stage 0 — Authentication & Access Control

### 0.1 API Key Authentication
Every request is authenticated with an API key. The header name and format differ by provider (see glossary §3).

**Key management capabilities:**
- **Organizations & Workspaces** — keys scoped to projects/environments (Anthropic Workspaces, OpenAI Orgs/Projects, xAI teams, OpenRouter API keys/credits).
- **Subscription-tiered access** — model availability scales with plan tier.
- **Admin API** — Anthropic Admin API (`sk-ant-admin...`) for workspace management, usage reports.
- **Entra ID / Managed Identity** — Azure supports key auth or Entra ID bearer tokens.

### 0.2 API Versioning
- **Dated API versions** — Azure (`api-version=2022-05-01`, etc.), Anthropic (`anthropic-version: 2023-06-01`).
- **Beta headers** — Anthropic (`anthropic-beta: files-api-2025-04-14`, `compact-2026-01-12`, `fast-mode-2026-02-01`, `task-budgets-2026-03-13`, `context-management-2025-06-27`, `server-side-fallback-2026-06-01`).
- **Model versioning** — pinned dated snapshots vs rolling aliases (`latest`, `~openai/gpt-latest`).
- **Deprecation notices** — Azure legacy capabilities retire March 31, 2029; LUIS retired March 2026; QnA Maker retired October 2025; OpenAI Assistants API sunset August 2026; OpenAI prompts API shutdown November 2026.

### 0.3 Privacy, Data Residency & ZDR
- **Not used for training** — xAI, OpenRouter (`provider.data_collection: deny`).
- **Zero Data Retention** — OpenAI (enforced for ZDR orgs, `store: false`), Anthropic (most features ZDR-eligible; Files API and Batches are not), xAI (`store: false`), OpenRouter (`provider.zdr: true`).
- **Data residency** — Anthropic (`inference_geo`), xAI (EU data residency with SOC 2/HIPAA/GDPR).
- **Watermarking** — Google SynthID.

---

## Stage 1 — Model Selection & Catalog

### 1.1 Model Catalog & Selection

Providers expose multiple models with different trade-offs. Selection is via `model` parameter or URL path.

**Generative LLM models (by provider):**

| Provider | Flagship (reasoning) | Balanced | Fast / lightweight |
|----------|---------------------|----------|-------------------|
| OpenAI | `gpt-5.6` (reasoning) | `gpt-4o` | `gpt-4o-mini`, `o4-mini` |
| Google | `gemini-3.1-pro-preview` | `gemini-3.5-flash` | `gemini-2.5-flash-lite` |
| Anthropic | `claude-opus-4-8` | `claude-sonnet-5` | `claude-haiku-4-5` |
| Mistral | `mistral-large-2512` | `mistral-medium-3-5` | `ministral-3b-2512` |
| xAI | `grok-4.5` (reasoning) | `grok-4` | — |
| OpenRouter | 400+ models from 70+ providers | — | `:nitro` variant |

**Special model types:**
- **Multi-agent model** — xAI `grok-4.20-multi-agent` (routes across cooperating agents; `reasoning.effort` controls agent count: 4 or 16).
- **Moderation model** — Mistral `mistral-moderation-2603`.
- **Embedding models** — OpenAI `text-embedding-3-*`, Mistral `mistral-embed`, Voyage AI `voyage-4-*`.

### 1.2 Model Variants (OpenRouter)
Suffix-based routing shortcuts: `:nitro` (fastest), `:floor` (cheapest), `:thinking` (extended reasoning), `:online` (web search, deprecated), `:free`, `:extended` (larger context), `:exacto` (best tool-calling).

### 1.3 Latest Aliases
- OpenRouter: `~openai/gpt-latest`, `~anthropic/claude-sonnet-latest`.
- Mistral: `mistral-large-latest`, `mistral-small-latest`.
- Azure: `modelVersion: "latest"`.

### 1.4 Model Catalog API (OpenRouter)
`GET /api/v1/models` returns standardized metadata: `id`, `context_length`, `architecture` (input/output modalities, tokenizer), `pricing` (per-token), `supported_parameters`, `reasoning` (supported efforts, default, mandatory), `benchmarks`, `expiration_date`.

Query filters: `output_modalities`, `supported_parameters` (e.g. `tools`, `structured_outputs`, `reasoning`), `sort` (pricing, context, throughput, latency, popularity, newest).

---

## Stage 2 — Input Preparation (Text, Multimodal, Files)

### 2.1 Input Formats

| Format | Providers | Description |
|--------|-----------|-------------|
| Plain string | All generative LLMs | Simplest: `input: "Hello"` |
| Message array | All generative LLMs | `messages[]` or `input[]` with role + content |
| Step array (stateless replay) | Google Interactions | Array of typed steps (user_input, thought, function_call, etc.) |
| Document array | Azure | `analysisInput.documents[]` with `id`, `text`, `language` |
| Conversation array | Azure, Mistral | `conversations[]` with conversation items (text/transcript modality) |

### 2.2 Multimodal Input — Images

| Provider | Accepted formats | Source types | Max size |
|----------|-----------------|-------------|----------|
| OpenAI | PNG, JPEG, WebP, non-animated GIF | base64, file_id, URL (Responses) | 512 MB payload, 1500 images |
| Google | PNG, JPEG, WebP, HEIC, HEIF | inline_data (base64), file_data (URI), URL | inline <100MB; Files API 2GB |
| Anthropic | JPEG, PNG, GIF, WebP | base64, URL, file_id | 10 MB (Claude API), 5 MB (Bedrock); max 8000×8000 |
| Mistral | — (via image_url) | URL, base64 data URI | — |
| xAI | PNG, JPEG, WebP | URL, file_id | — |
| OpenRouter | — (via image_url) | URL, base64 | — |

**Image detail/resolution control:**
- OpenAI: `detail: low` (512×512, fast) / `high` / `original` / `auto`.
- Anthropic: Resolution tiers (high-resolution: max long edge 2576px, max 4784 visual tokens; standard: 1568px / 1568 tokens). Images treated as 28×28-pixel patches.
- Google: `media_resolution` controls max tokens per image/video frame.

**Max images per request:** OpenAI 1500, Anthropic 100 (200k) / 600 (other), Google 3600.

### 2.3 Multimodal Input — PDF & Documents

| Provider | Accepted formats | Source types | Processing |
|----------|-----------------|-------------|-----------|
| OpenAI | PDF, DOCX, PPTX, TXT, code, spreadsheets | base64, file_id, URL (Responses) | PDF: text + page images; spreadsheets: 1000 rows/sheet augmentation |
| Google | PDFs, documents | inline_data, file_data (URI) | Page images + text |
| Anthropic | PDF, plain text | base64, URL, file_id | Page images (vision); 600 pages (100 for 200k); 32 MB |
| xAI | PDFs, documents | URL, file_id | Agentic document search |
| Azure | PDF, DOCX, TXT (native document processing) | Blob Storage SAS URLs | Preserves layout, font, color |

**PDF detail control (OpenAI Responses):** `detail: auto` / `low` / `high`. Only affects page image processing; extracted text always included.

### 2.4 Multimodal Input — Video & Audio (Google)
Google Gemini supports video and audio as input parts: `inline_data` or `file_data` with appropriate MIME types. Native audio understanding (no separate speech-to-text pipeline needed). Video Q&A, video memory, real-time transcription/translation.

### 2.5 Files API

| Provider | Endpoint | Purpose | Notes |
|----------|----------|---------|-------|
| OpenAI | `POST /v1/files` (purpose: user_data) | Upload files for model input | <50 MB per file; combined <50 MB per request |
| Anthropic | `/v1/files` (beta) | Upload-once, reference by file_id | 500 MB per file; 500 GB per org; not ZDR |
| Google | Files API | Upload → uri + name; poll state PROCESSING→ACTIVE | Recommended for >100MB / reuse |
| xAI | Files API | Bidirectional: input substitution + output storage | `storage_options` for outputs |
| Mistral | `client.files.upload` | Batch file upload | purpose: "batch" |

### 2.6 Input for Classical NLP (Azure)

**Document** — the unit of input: `id`, `text`, `language` (optional or auto-detected).

**Conversation item** — for conversation features: `id`, `participantId`, `text`, and (for transcripts) `lexical`/`itn`/`maskedItn` speech-format variants plus `audioTimings[]`.

**Native document** — for document PII/summarization: source and target Blob Storage SAS URLs.

**`stringIndexType`** — defines offset/length encoding: `Utf16CodeUnit` (default for text analysis) or `TextElement_V8` (Unicode grapheme clusters, used by CLU).

---

## Stage 3 — System Instructions & Configuration

### 3.1 System Instructions / System Prompt

High-level guidance for model behavior (persona, tone, rules). Takes priority over user input.

| Provider | Parameter | Location |
|----------|-----------|----------|
| OpenAI Responses | `instructions` (top-level string) | Top-level param |
| OpenAI Chat | `system` role in messages | In `messages[]` |
| Google Interactions | `system_instruction` | Top-level param |
| Google generateContent | `system_instruction` | In `GenerateContentConfig` |
| Anthropic | `system` (top-level param, string or text blocks) | Top-level param |
| Mistral | `system` role message (first in array) | In `messages[]` |
| xAI Responses | `instructions` (top-level) | Top-level param |
| xAI Chat | `system` role in messages | In `messages[]` |
| OpenRouter | `system` / `developer` role | In `messages[]` |

**Mid-conversation system messages (Anthropic Opus 4.8 only):** Append `{"role": "system"}` inside `messages[]` at the point a new instruction becomes relevant. Preserves prompt caching. Placement rules apply (must follow user/assistant turn, cannot be first, cannot sit between tool_use and tool_result).

### 3.2 Generation Config / Sampling Parameters

| Parameter | OpenAI | Google | Anthropic | Mistral | xAI | OpenRouter |
|-----------|--------|--------|-----------|---------|-----|------------|
| `temperature` | Yes | Yes (keep defaults for Gemini 3.x) | Deprecated (post-Opus 4.6, only 1.0) | Yes | Yes (0–2) | Yes (0–2) |
| `top_p` | Yes | Yes (keep defaults) | Deprecated (≥0.99 only) | Yes | Yes | Yes (0–1) |
| `top_k` | — | Yes (keep defaults) | Deprecated (rejected) | — | Yes (Responses) | Yes (not OpenAI) |
| `min_p` | — | — | — | — | Yes (Responses) | Yes |
| `max_tokens` / `max_output_tokens` | `max_tokens` (Chat) | `max_output_tokens` | `max_tokens` (required) | `max_tokens` | `max_output_tokens` (Responses) / `max_tokens` (Chat) | `max_tokens` / `max_completion_tokens` |
| `stop` / `stop_sequences` | `stop` (Chat, not reasoning) | — | `stop_sequences` | `stop` | `stop` (not reasoning) | `stop` (up to 4) |
| `frequency_penalty` | Yes | — | — | — | Yes (not reasoning) | Yes (−2 to 2) |
| `presence_penalty` | Yes | — | — | — | Yes (not reasoning) | Yes (−2 to 2) |
| `repetition_penalty` | — | — | — | — | — | Yes |
| `seed` | — | — | — | `random_seed` | `seed` | `seed` |
| `n` (candidates) | Yes (Chat) | — | — | — | Yes (Chat) | — |
| `logprobs` / `top_logprobs` | Yes | — | — | — | Yes (not grok-4.20+) | Yes |
| `logit_bias` | — | — | — | — | — (accepted for compat) | Yes |
| `verbosity` | — | — | — | — | — | Yes (low/medium/high/xhigh/max) |
| `prediction` (predicted output) | — | — | — | — | — | Yes |
| `service_tier` | — | — | `service_tier` (auto/standard_only) | — | `service_tier` (default/priority) | Yes (auto/default/flex/priority/scale) |

**Critical Gemini 3.x note:** Google strongly recommends keeping `temperature`, `top_p`, `top_k` at defaults. Setting temperature <1.0 can cause looping/degraded performance.

**Reasoning model restrictions:** `presence_penalty`, `frequency_penalty`, `stop` cannot be used with xAI reasoning models. `temperature`/`top_k` modifications incompatible with Anthropic extended thinking.

### 3.3 Output Constraining / Prefill

Pre-fill part of the model's response by placing an `assistant` message as the last entry in the conversation:

| Provider | Mechanism | Status |
|----------|-----------|--------|
| Anthropic | `assistant` message as last entry | **Deprecated** on Fable 5, Mythos 5, Opus 4.8/4.7/4.6, Sonnet 5/4.6 (returns 400). Use structured outputs instead. |
| Mistral | `prefix: true` on assistant message | Supported |
| OpenRouter | Trailing `assistant` message | Supported |
| OpenAI | — | Not documented as a feature |

**Incompatibilities:** Prefill is not compatible with extended thinking (Anthropic) or structured outputs on newer models.

---

## Stage 4 — Reasoning / Thinking Configuration

### 4.1 Reasoning Modes

| Mode | Providers | Description |
|------|-----------|-------------|
| **Effort-based** | OpenAI, Google, xAI, OpenRouter | `reasoning.effort` / `thinking_level` controls depth (low/medium/high, etc.) |
| **Budget-based (manual)** | Anthropic (legacy), OpenRouter | `thinking.budget_tokens` / `reasoning.max_tokens` sets explicit token budget |
| **Adaptive** | Anthropic | `thinking: {type: "adaptive"}` — model decides thinking depth |
| **Mandatory (cannot disable)** | xAI (`grok-4.5`), Anthropic (Fable 5, Mythos 5), OpenRouter (`mandatory: true` models) | Reasoning always on |
| **Task budget (loop-level)** | Anthropic | `output_config.task_budget` — advisory total-work budget across an agentic loop |

### 4.2 Effort Levels

| Level | OpenAI | Google | Anthropic | xAI | OpenRouter |
|-------|--------|--------|-----------|-----|------------|
| Minimal / none | `minimal` / `none` | `minimal` | `type: disabled` | — (cannot disable) | `minimal` / `none` |
| Low | `low` | `low` | `low` | `low` | `low` |
| Medium | `medium` | `medium` | `medium` | `medium` | `medium` |
| High | `high` | `high` | `high` (default) | `high` (default) | `high` |
| Xhigh | — | — | `xhigh` | `xhigh` (multi-agent: 16 agents) | `xhigh` |
| Max | — | — | `max` | — | `max` |

### 4.3 Reasoning Content in Output

| Provider | How reasoning appears in output |
|----------|-------------------------------|
| OpenAI | `reasoning` Items in `output[]`; `summary` field (auto/concise/detailed); encrypted via `include` |
| Google | `thought` steps in `steps[]`; `signature` (always present) + optional `summary` (text/image) |
| Anthropic | `thinking` blocks in `content[]`; `thinking` (text) + `signature`; `redacted_thinking` (encrypted, opaque) |
| Mistral | `ThinkChunk` in content chunk list; `thinking` field is list of TextChunks |
| xAI | `reasoning` items with `summary` / `encrypted_content`; `reasoning_content` in Chat |
| OpenRouter | `reasoning` field (string) + `reasoning_details` array (summary/encrypted/text types) |

### 4.4 Thought Signatures & Multi-Turn Continuity

Thought signatures are **required** to maintain reasoning continuity across multi-turn interactions.

| Mode | Provider | Handling |
|------|----------|----------|
| Stateful | OpenAI, Google, xAI | Server manages automatically via `previous_response_id` / `previous_interaction_id` |
| Stateless | Google, Anthropic, OpenRouter | Must resend all thought/thinking blocks **exactly as received** — do not remove or modify |
| Encrypted | OpenAI, xAI, Anthropic, OpenRouter | Pass encrypted reasoning blobs back in input for stateless + reasoning |

### 4.5 Reasoning Context Mode (OpenAI GPT-5.6+)

| `reasoning.context` | Behavior |
|---------------------|----------|
| `auto` (default) | Model's default context mode |
| `all_turns` | Model references reasoning from all turns in input |
| `current_turn` | Only current-turn reasoning; prior reasoning ignored |

### 4.6 Reasoning Mode (OpenAI GPT-5.6+ via OpenRouter)

`reasoning.mode: "pro"` routes to the model's `*-pro` variant (deeper multi-pass reasoning). Independent of effort. Bills at same per-token rates but consumes more tokens.

### 4.7 Task Budgets (Anthropic)

Advisory (soft) token budget for a full agentic loop — covering thinking, tool calls, tool results, and output. `output_config.task_budget: {type: "tokens", total: 64000, remaining: ...}`. Minimum 20,000 tokens. Advisory, not enforced (hard cap remains `max_tokens`). Beta header required.

### 4.8 Fast Mode (Anthropic)

Up to 2.5× higher output tokens/sec on Opus models via `speed: "fast"` + beta header. Same model weights/intelligence. Benefit is on output tokens/sec, not time-to-first-token. Opus 4.8 and 4.7 only (4.7 deprecated June 2026). Switching fast↔standard invalidates prompt cache. Not available with Batch API or Priority Tier.

### 4.9 Reasoning Token Billing
Reasoning tokens are counted and billed as **output tokens** in all providers. They count against `max_output_tokens` / `max_tokens` and the context window.

---

## Stage 5 — Structured Output Configuration

### 5.1 Structured Outputs (JSON Schema Enforcement)

Force model output to conform to a JSON Schema.

| Provider | Parameter | Strict mode |
|----------|-----------|-------------|
| OpenAI Responses | `text.format: {type: "json_schema", name, strict, schema}` | `strict: true` (additionalProperties: false, all fields required) |
| OpenAI Chat | `response_format: {type: "json_schema", json_schema: {name, strict, schema}}` | Same |
| Google Interactions | `response_format: {type: "text", mime_type: "application/json", schema}` | No `strict` flag |
| Google generateContent | `response_mime_type: "application/json"` + `response_schema` | — |
| Anthropic | `output_config.format: {type: "json_schema", schema}` | Grammar compilation + 24h caching; no `strict` flag on format, but `strict: true` on tools |
| Mistral | `client.chat.parse(response_format=Pydantic/Zod/JSONSchema)` | Schema-enforced |
| xAI Responses | `text.format: {type: "json_schema", name, schema, strict}` | `strict: true` |
| xAI Chat | `response_format: {type: "json_schema", json_schema: {name, schema, strict}}` | Same |
| OpenRouter | `response_format` (normalized) | `strict: true` |

### 5.2 JSON Mode (No Schema)

Guarantees valid JSON but not schema adherence.

| Provider | Parameter |
|----------|-----------|
| OpenAI | `text.format: {type: "json_object"}` / `response_format: {type: "json_object"}` |
| Google | `response_mime_type: "application/json"` (without `response_schema`) |
| Mistral | `response_format: {type: "json_object"}` |
| xAI | `text.format: {type: "json_object"}` / `response_format: {type: "json_object"}` |
| OpenRouter | `response_format: {type: "json_object"}` |

**Rule:** You must instruct the model to produce JSON in the conversation. The API may throw an error if "JSON" doesn't appear in the context (OpenAI).

### 5.3 Supported JSON Schema Subset

**Common supported types:** `string`, `number`, `integer`, `boolean`, `object`, `array`, `null` (via type array), `enum`, `anyOf`.

**Common supported constraints:**
- String: `pattern` (regex), `format` (date-time, date, time, email, uuid, ipv4, ipv6, uri, hostname, duration)
- Number: `minimum`, `maximum`, `exclusiveMinimum`, `exclusiveMaximum`, `multipleOf`
- Array: `items`, `minItems`, `maxItems`, `prefixItems` (tuple)
- Object: `properties`, `required`, `additionalProperties`

**Common supported features:** `$defs` / `$ref` (reusable subschemas), recursive schemas, `anyOf`.

**Common unsupported keywords:** `allOf` (multiple subschemas), `not`, `if`/`then`/`else`, `dependentRequired`, `dependentSchemas`.

### 5.4 Strict Mode Requirements (OpenAI, xAI, OpenRouter)

When `strict: true`:
- `additionalProperties: false` mandatory on every object.
- All fields must be in `required`.
- Optional fields via union with `null`: `"type": ["string", "null"]`.
- Root object must be an object (cannot be `anyOf` at top level).

### 5.5 Schema Complexity Limits

| Limit | OpenAI | Anthropic | xAI |
|-------|--------|-----------|-----|
| Max object properties | 5,000 | — | 64 (maxProperties) |
| Max nesting levels | 10 | — | — |
| Max enum values | 1,000 | — | — |
| Max strict tools per request | — | 20 | — |
| Max optional params (Anthropic) | — | 24 | — |
| Max union type params (Anthropic) | — | 16 | — |
| Compilation timeout (Anthropic) | — | 180s | — |

### 5.6 Grammar / CFG Constraints

| Provider | Mechanism |
|----------|-----------|
| OpenAI | Custom tools with `format: {type: "grammar", syntax: "lark"|"regex", definition: "..."}` |
| Anthropic | Constrained decoding (compiled grammars, 24h cache) |
| OpenRouter | `response_format: {type: "grammar"}` (from OpenAPI) |

### 5.7 Refusals in Structured Outputs

Safety-based refusals are programmatically detectable:
- OpenAI Chat: `choices[0].message.refusal`
- OpenAI Responses: message content item `type === "refusal"`
- Anthropic: `stop_reason: "refusal"`, output may not match schema

### 5.8 Streaming Structured Outputs

Streamed chunks are valid partial JSON strings that concatenate to form the final JSON object. Parse after stream completes.

### 5.9 Structured Outputs + Tools (Gemini 3, xAI Grok 4)
Gemini 3 models can combine Structured Outputs with built-in tools (Google Search, URL Context, Code Execution, File Search) and Function Calling in the same request. xAI Grok 4 can combine structured outputs with tool calling.

---

## Stage 6 — Function Calling & Tool Configuration

### 6.1 Tool Types

| Type | Description | Providers |
|------|-------------|-----------|
| **Function tools** | Defined by JSON Schema; model emits structured arguments | All generative LLMs |
| **Custom tools** | Free-form text input/output (no JSON Schema) | OpenAI |
| **Built-in / server tools** | Provider-hosted tools | OpenAI, Google, xAI, OpenRouter |
| **Remote MCP** | Connect to external MCP servers | OpenAI, Google |

### 6.2 Function Definition Shape

| Provider | Modern API shape | Legacy API shape |
|----------|-----------------|-----------------|
| OpenAI Responses | Flat: `{type: "function", name, description, parameters, strict}` | — |
| OpenAI Chat | — | Externally tagged: `{type: "function", function: {name, description, parameters, strict}}` |
| Google | Flat: `{type: "function", name, description, parameters}` | — |
| Anthropic | `{type: "function", name, description, input_schema, strict}` | — |
| Mistral | — | Externally tagged: `{type: "function", function: {name, description, parameters, strict}}` |
| xAI Responses | Flat | — |
| xAI Chat | — | Externally tagged |
| OpenRouter | Normalized | Externally tagged |

### 6.3 Tool Calling Flow

1. Define tools and send with request.
2. Model emits tool call (function call item / step / block).
3. Execute tool on application side.
4. Return tool result (function_call_output / function_result / tool_result).
5. Model produces final response (or more tool calls).

The modern APIs (OpenAI Responses, Google Interactions, xAI Responses) can continue this flow for **as many tool calls as the task requires within a single agentic-loop request**.

### 6.4 `tool_choice` Parameter

| Value | OpenAI | Google | Anthropic | Mistral | xAI |
|-------|--------|--------|-----------|---------|-----|
| Auto (default) | `auto` | `auto` | `auto` | `auto` | `auto` |
| Required | `required` | `any` | `any` | `any` | `required` |
| None | `none` | `none` | `none` | `none` | `none` |
| Specific function | `{type: "function", name: "..."}` | `{allowed_tools: {mode: "any", tools: ["..."]}}` | `{type: "tool", name: "..."}` | — | — |
| Subset | `{type: "allowed_tools", mode: "auto", tools: [...]}` | `{allowed_tools: {mode: "any", tools: [...]}}` | — | — | — |
| Validated | — | `validated` (preview) | — | — | — |

### 6.5 Parallel & Compositional Function Calling
- **Parallel** — multiple independent functions in one turn (OpenAI `parallel_tool_calls: true` default; Google parallel; Anthropic `disable_parallel_tool_use` option).
- **Compositional** — chained function calls where model uses result of one to decide next call (Google automatic within single interaction).

### 6.6 Built-in / Server Tools

| Tool | OpenAI | Google | xAI | OpenRouter |
|------|--------|--------|-----|------------|
| Web search | `web_search` | `google_search` | search | `openrouter:web_search` |
| URL content fetch | — | `url_context` | — | `openrouter:web_fetch` |
| Code execution | `code_interpreter` | `code_execution` | code exec | `bash` |
| File search / retrieval | `file_search` | `file_search` | document search | `files` |
| Image generation | `image_generation` | — | — | `image-generation` |
| Computer use | `computer_use` | — | — | — |
| Remote MCP | MCP | `mcp_server` | — | — |
| Local shell | `local_shell` | — | — | `shell` |
| Apply patch | `apply_patch` | — | — | `apply-patch` |

### 6.7 Tool Namespaces (OpenAI)
Group related tools by domain: `{type: "namespace", name: "crm", description: "...", tools: [...]}`. Individual functions can set `defer_loading: true`.

### 6.8 Tool Search / Deferred Tool Loading (OpenAI GPT-5.4+)
`tool_search` tool lets the model search for relevant tools, add them to context, then use them. Flow: request with tool_search + deferred tools → model calls tool_search → relevant tools loaded → model calls function tools → outputs returned.

### 6.9 Multimodal Function Responses (Google Gemini 3)
Function responses can include multimodal content (images) in the `result` field.

### 6.10 Custom Tools with Context-Free Grammars (OpenAI)
`type: "custom"` tools accept free-form text input. Optional `format: {type: "grammar", syntax: "lark"|"regex", definition: "..."}` constrains the model's text input via CFG.

### 6.11 Tool Calling Token Usage
Functions are injected into the system message in a trained syntax, so they count against the context limit and are billed as input tokens. Limit functions, shorten descriptions, or use tool search/fine-tuning to reduce tokens.

---

## Stage 7 — Grounding, Citations & RAG Configuration

### 7.1 Document Citations (Anthropic)

Ground responses in source documents you provide; the API returns exact passages (`cited_text`) supporting each claim.

- Enable: `citations: {enabled: true}` on each `document` content block (all or none).
- Document types: plain text (`char_location`), PDF (`page_location`), custom content (`content_block_location`).
- Citation fields: `cited_text`, `document_index`, `document_title`, + location-specific fields.
- **Incompatible with structured outputs** (returns 400).
- `cited_text` does not count toward output tokens.

### 7.2 Search Result Citations (Anthropic)

`search_result` content blocks for RAG-style citations. Each block carries `source` (URL/id), `title`, `content` array of text blocks. Citations are all-or-nothing. `search_result_location` citation fields include `source`, `title`, `cited_text`, `search_result_index`, `start_block_index`, `end_block_index`.

### 7.3 Citations via Tool Calls (Mistral)

Mistral's citation flow: (1) define a tool returning references as JSON → (2) model emits tool_call → (3) execute tool, return JSON map of references → (4) model produces answer with `TextChunk`(s) and `ReferenceChunk`(s) interleaved → (5) `ReferenceChunk.reference_ids` index into references map.

### 7.4 Built-in Web Search Grounding

| Provider | Tool | Citations |
|----------|------|-----------|
| OpenAI | `web_search` built-in tool | URL citations in annotations |
| Google | `google_search` tool | `google_search_call`/`google_search_result` steps; `search_suggestions` HTML to render |
| xAI | Search (server tool) | `num_sources_used` in usage |
| OpenRouter | `openrouter:web_search` server tool / `web` plugin (deprecated) | — |

### 7.5 Managed RAG

| Provider | Mechanism |
|----------|-----------|
| Mistral | Libraries (upload docs → ingest/vectorize/search → `document_library` tool on agents → grounded answers with `tool_reference` citations) |
| Azure | Custom Question Answering (KB of Q&A pairs → layered ranking: Azure AI Search → NLP re-ranking → confidence score) |
| OpenAI | `file_search` built-in tool |
| Google | Context caching (cache large files, query against cached context) |

### 7.6 RAG from Scratch (Mistral guide)

1. Get data → load text document.
2. Split into chunks (by character, tokens, sentences, paragraphs, HTML headers, or AST for code).
3. Create embeddings (`client.embeddings.create`).
4. Load into vector DB (e.g. FAISS).
5. Embed question (same model).
6. Retrieve similar chunks (`index.search`).
7. Combine context + question in prompt → `client.chat.complete`.

**RAG techniques:** HyDE (hypothetical answer embedding), child/parent chunks, time-weighted retrieval, "lost in the middle" reordering, metadata filtering, BM25, few-shot prompting.

---

## Stage 8 — Moderation & Safety Configuration

### 8.1 Input Moderation

| Provider | Mechanism | On violation |
|----------|-----------|-------------|
| Mistral | `guardrails` array (inline, `moderation_llm_v2` config with `custom_category_thresholds`, `ignore_other_categories`, `action: "block"`, `block_on_error`) | HTTP 403 |
| Mistral (legacy) | `safe_prompt: true` | — |
| OpenAI | `moderation` (auto/low); `moderation_details.categories` | `moderation_blocked` error |
| OpenRouter | `moderation` plugin | — |
| Anthropic | Safety classifiers → `stop_reason: "refusal"` | HTTP 200 with refusal (not an error) |

### 8.2 Moderation Categories (Mistral)

Sexual, Hate and Discrimination, Violence and Threats, Dangerous, Criminal, Self-Harm, Health, Financial, Law, PII, Jailbreaking. Custom thresholds (0–1) per category; set to 1 to disable. `ignore_other_categories: true` to evaluate only listed categories.

### 8.3 Moderation API (Mistral)

Dedicated classification API: `POST /v1/moderations` (raw text) and `POST /v1/chat/moderations` (conversational). Returns scores per category. Backed by `mistral-moderation-2603` model.

### 8.4 Refusal Handling & Fallback (Anthropic)

Claude Fable 5 safety classifiers can decline a request; refusal is **HTTP 200** with `stop_reason: "refusal"`. `stop_details` identifies the policy category (`cyber`, `bio`, `frontier_llm`, `reasoning_extraction`, or null).

**Server-side fallback:** `fallbacks` parameter (up to 3 entries, each `{model, max_tokens?, thinking?}`). Tried in order. `usage.iterations` records every attempt. Only safety-classifier decline triggers fallback; rate limits/overload returned as-is.

**SDK middleware fallback:** `BetaRefusalFallbackMiddleware` with shared `BetaFallbackState` to pin follow-ups to the accepting model.

### 8.5 PII Detection & Redaction (Azure)

| Variant | Input | Modality | Redaction policies |
|---------|-------|----------|-------------------|
| Text PII | Raw text strings | — | CharacterMask (default), NoMask, EntityMask, SyntheticReplacement |
| Conversation PII | Chat transcripts / speech-to-text | text / transcript | characterMask, entityMask, noMask; `redactionSource` (text/lexical/itn/maskedItn); `includeAudioRedaction` |
| Document PII | Native document files (PDF, DOCX) | — | noMask, characterMask, entityMask, syntheticReplacement; blur-based image redaction (GA) |

**Configurable parameters:** `piiCategories` (filter entity types), `confidenceScoreThreshold` (with overrides), `disableEntityValidation`, `excludeExtractionData` (document: only redacted doc stored).

### 8.6 Content Moderation in Structured Outputs
Safety-based refusals are programmatically detectable even in structured output mode (OpenAI: `refusal` field; Anthropic: `stop_reason: refusal`, output may not match schema).

### 8.7 Guardrail Attachment Points (Mistral)
- Chat Completions: `guardrails` field on `POST /v1/chat/completions`.
- Conversations API: `guardrails` field on `POST /v1/conversations` (override agent).
- Agent-level: `guardrails` at agent creation; inherited by conversations; overridable per-request.

---

## Stage 9 — Conversation State Management

### 9.1 State Management Strategies

| Strategy | Providers | Description |
|----------|-----------|-------------|
| **Manual replay** | All generative LLMs | Replay full message/item/step history each turn (stateless) |
| **`previous_response_id`** | OpenAI (Responses), xAI (Responses) | Server rehydrates prior context; only send new turn |
| **`previous_interaction_id`** | Google (Interactions) | Server manages all state including thought signatures |
| **Conversations API** | OpenAI (`/v1/conversations`), Mistral (`/v1/conversations`, beta) | Persistent conversation object across sessions/devices |
| **Encrypted reasoning replay** | OpenAI, xAI, Anthropic, Google, OpenRouter | `store: false` + pass encrypted reasoning blobs back |
| **Chat SDK** | Google (generateContent) | `client.chats.create()` manages `contents` client-side |

### 9.2 Server-Side Storage

| Provider | Default | TTL | Opt-out |
|----------|---------|-----|---------|
| OpenAI | `store: true` | 30 days | `store: false` |
| xAI | `store: true` | 30 days | `store: false` |
| Google | `store: true` | — | `store: false` |
| Anthropic | — (stateless) | — | — (always resend history) |
| Mistral | — (stateless) | — | — |
| OpenRouter | — (stateless) | — | — |

### 9.3 CRUD on Stored Responses

| Provider | Endpoints |
|----------|-----------|
| OpenAI | `GET /v1/responses/{id}` (retrieve); Conversations API |
| xAI | `GET /v1/responses/{id}` (retrieve), `DELETE /v1/responses/{id}` (delete) |
| Anthropic | Batches: `GET /v1/messages/batches/{id}`, `DELETE`, cancel |

### 9.4 Critical Rules for Stateless + Reasoning

- **Anthropic:** When using thinking with tool calls, pass complete unmodified `thinking` blocks (with signature) back; modifying → 400 error. Thinking block preservation by model: Opus 4.5+ and Sonnet 4.6+ keep all; earlier keep only last turn.
- **Google:** Must resend all thought blocks exactly as received. Built-in tool result signatures must also be resent.
- **Mistral:** Always replay the full assistant message including `ThinkChunk` back. Dropping reasoning trace across turns degrades performance.
- **OpenAI:** Preserve every Item in `response.output` and use `include: ["reasoning.encrypted_content"]`.
- **OpenRouter:** Pass back `reasoning_details` unmodified; works across OpenAI and Anthropic reasoning models.

### 9.5 Context Window Management

The context window is the maximum tokens (input + output + reasoning) per request. Token types counted:
- **Input tokens** — inputs in messages/input array
- **Output tokens** — tokens generated in response
- **Reasoning/thinking tokens** — used by reasoning models to plan (counted toward context window and billed as output)

**Overflow behavior (Anthropic):**
- Input alone exceeds → 400 `invalid_request_error`.
- Input + `max_tokens` exceeds: Claude 4.5+ accepts and stops with `stop_reason: "model_context_window_exceeded"`; earlier models return validation error.

---

## Stage 10 — Text Generation (Single-Turn & Multi-Turn)

### 10.1 Single-Turn Text Generation

The foundational capability — prompting a model to produce text.

**API patterns:**

| Pattern | Providers | Description |
|---------|-----------|-------------|
| Responses API | OpenAI, xAI | `input` (string or Item[]) + `instructions` → `output[]` typed items |
| Interactions API | Google | `input` (string/part[]/step[]) + `system_instruction` → `steps[]` typed steps |
| Messages API | Anthropic | `system` + `messages[]` → `content[]` typed blocks |
| Chat Completions | OpenAI (legacy), Mistral, xAI (deprecated), OpenRouter | `messages[]` → `choices[].message` |
| Analyze-text | Azure | `kind` + `parameters` + `analysisInput` → `results` |

### 10.2 Output Structure

**Critical (OpenAI Responses):** The `output` array often has more than one item. It is **not safe** to assume text is at `output[0].content[0].text`. Use the `output_text` SDK helper.

**Critical (Google Interactions):** `output_text` does **not** include text blocks separated by non-text content (thoughts, images, tool calls). Manually iterate over `steps` for interleaved responses.

**Critical (Mistral):** `message.content` can be a plain string OR a list of chunks (when reasoning, citations, or vision are involved).

### 10.3 Multi-Turn Conversations

Each turn's user message + assistant response accumulate. For long conversations, use prompt caching, context compaction, or context editing (see Stage 14).

### 10.4 Multiple Candidates (`n`)

| Provider | Support |
|----------|---------|
| OpenAI Chat | Yes (`n` parameter) |
| OpenAI Responses | No (make separate requests) |
| xAI Chat | Yes (`n`, billed across all) |
| OpenRouter | — |
| Others | — |

### 10.5 Prompt Engineering Best Practices

- Pin model snapshots in production for consistent behavior.
- Build test/eval suites to measure prompt performance.
- Store prompts in code, not as reusable prompt objects (OpenAI deprecating `v1/prompts`).
- Use typed function arguments or schemas for dynamic values.
- For long context (Google), place the query at the **end** of the prompt.
- For multilingual, explicitly state the target response language in the system prompt.

---

## Stage 11 — Streaming

### 11.1 Enabling Streaming
Set `stream: true` in the request body (all providers). Returns Server-Sent Events (SSE).

### 11.2 Streaming Formats

**Delta chunks (Chat Completions style):** OpenAI Chat, xAI Chat, OpenRouter Chat, Mistral. Each chunk carries `delta.content` (and `delta.tool_calls`, `delta.reasoning_content`, `delta.reasoning_details` as applicable).

**Typed SSE events (Responses/Interactions style):** OpenAI Responses, Google Interactions, xAI Responses, OpenRouter Responses. Consumers must branch on each event's `type`.

**Content block events (Anthropic):** `message_start` → `content_block_start` → `content_block_delta`(s) → `content_block_stop` → `message_delta` → `message_stop`. Delta variants: `text_delta`, `input_json_delta`, `thinking_delta`, `signature_delta`, `citations_delta`, `compaction_delta`.

### 11.3 Streaming with Reasoning

| Provider | Reasoning in stream |
|----------|-------------------|
| Google | `thought_summary` deltas + `thought_signature` delta (last before step.stop) |
| Anthropic | `thinking_delta` events; `signature_delta` just before `content_block_stop`. With `display: "omitted"`, no `thinking_delta` events. |
| Mistral | Phase 1: `delta.content` is list with `ThinkChunk`. Phase 2: transition (both ThinkChunk + TextChunk). Phase 3: `delta.content` is plain string. |
| xAI | `response.reasoning_text.delta`, `response.reasoning_summary_text.delta` |
| OpenRouter | `delta.reasoning_details`; `response.reasoning.delta` |

### 11.4 Streaming with Structured Outputs
Streamed chunks are valid partial JSON strings. Concatenate to form the final JSON object. Parse after stream completes.

### 11.5 SSE Comments / Keep-Alive
OpenRouter sends SSE comments (`: OPENROUTER PROCESSING`) to prevent timeouts. These are spec-compliant and can be safely ignored.

### 11.6 Stream Cancellation (OpenRouter)
Abort the connection to cancel. For supported providers, immediately stops processing and billing. For unsupported providers, model continues and you are billed for the complete response.

### 11.7 Error Handling During Streaming

| When error occurs | Behavior (OpenRouter) |
|-------------------|----------------------|
| Before any tokens sent | Standard JSON error response with HTTP status |
| After tokens sent (mid-stream) | Error sent as SSE event with `error` at top level + `finish_reason: "error"`. HTTP status stays 200. |

### 11.8 SDK Streaming Helpers
- Anthropic: `.stream()` context manager, `text_stream`, `.get_final_message()`.
- OpenAI: typed event iteration.
- xAI: `chat.stream()` — `chunk.content` is delta; `response` auto-accumulates.
- Google: `generate_content_stream()` / `stream=True` with typed events.

### 11.9 Timeout Considerations
Reasoning models can take a long time to first token. xAI recommends overriding default client timeout (e.g. `timeout=3600` seconds). Anthropic SDKs require streaming when `max_tokens` > 21,333 to avoid HTTP timeouts.

---

## Stage 12 — Classical NLP Analysis (Pre-trained Tasks)

This stage encompasses all pre-trained NLP capabilities that derive meaning from text without generative LLM conversation. Primarily **Azure AI Language**, though generative LLMs can also perform many of these tasks via prompting or structured outputs.

### 12.1 Language Detection (Azure)
Detects primary language across 100+ languages. Returns ISO 639-1 code, human-readable name, confidence score. For select languages: script name + ISO 15924 script code. Optional country/region hint. Synchronous. `kind: LanguageDetection`.

### 12.2 Named Entity Recognition (Azure NER)
Identifies and categorizes entities: Person, PersonType, Location, Organization, events, products, quantities, DateTime, addresses, emails, URLs, phone numbers. GA: `category`/`subcategory`. Preview: `type`/`tags`/`metadata`. Filter via `inclusionList`/`exclusionList`. Sync and async batch. `kind: EntityRecognition`.

### 12.3 Custom NER (Azure)
Trains ML models to extract domain-specific entities from your labeled data. Full lifecycle: define schema → label data → train → evaluate → deploy → extract. Requires Blob Storage. Authoring API + Runtime API. `kind: CustomEntityRecognition`.

### 12.4 PII Detection & Redaction (Azure)
Three variants (text, conversation, document) — see Stage 8.5. Detects PII/PHI: person names, organizations, addresses, emails, phone numbers, SSN, passport, financial data. Returns detected entities + `redactedText`. Configurable redaction policies.

### 12.5 Text Analytics for Health (Azure)
Prebuilt biomedical NLP: NER (diagnosis, medication, symptom, dosage, age, etc.) + relation extraction + entity linking (UMLS Metathesaurus) + assertion detection (certainty, conditionality, association, temporality) + FHIR output. Languages: English, German, French, Italian, Spanish, Portuguese, Hebrew. `kind: Healthcare`.

### 12.6 Sentiment Analysis & Opinion Mining (Azure)
Document-level + sentence-level sentiment (`positive`/`neutral`/`negative`/`mixed`) with confidence scores. Opinion mining: aspect-based sentiment with **targets** (nouns/aspects) and **assessments** (adjectives), linked via relations. Synchronous. `kind: SentimentAnalysis`. Legacy — retiring March 2029.

### 12.7 Key Phrase Extraction (Azure)
Extracts main concepts/topics as key phrases. Prebuilt, no customization. Synchronous. `kind: KeyPhraseExtraction`. Legacy — retiring March 2029.

### 12.8 Entity Linking (Azure)
Disambiguates entity identity by linking to Wikipedia. Returns `name`, `url`, `bingId`, `dataSource`, `matches[]`. Synchronous. `kind: EntityLinking`. Legacy — retiring September 2028.

### 12.9 Summarization (Azure)
Three genres:
- **Extractive** — extracts salient original sentences with `rankScore`; `sentenceCount` (1–20), `sortby` (Offset/Rank).
- **Abstractive** — generates concise coherent sentences; `summaryLength` (oneSentence/short/medium/long); query-focused with `query` field.
- **Conversation** — structured conversational input; aspects: issue, resolution, chapterTitle, narrative, recap, follow-up tasks.
- **Document** — native document files via Blob Storage.

Legacy — retiring March 2029.

### 12.10 Custom Text Classification (Azure)
Train custom single-label or multi-label classification. Full lifecycle: define schema → label data → train → evaluate → deploy → classify. Requires Blob Storage. `kind: CustomMultiLabelClassification` / `CustomSingleLabelClassification`. Legacy — retiring March 2029.

### 12.11 Conversation Language Understanding / CLU (Azure)
Predicts user intents and extracts entities from utterances. Project: schema (intents + entity categories) + labeled utterances. Training modes: Standard (English only, faster) and Advanced (multilingual). `multilingual` flag. `settings.confidenceThreshold`. Runtime: `kind: Conversation` → `topIntent`, `intents[]`, `entities[]`.

### 12.12 Custom Question Answering / CQA (Azure)
Knowledge base of Q&A pairs. Imports from URLs, PDFs, FAQs, manuals. Layered ranking: Azure AI Search → NLP re-ranking → confidence score. Two query modes: `query-knowledgebases` (deployed KB) and `query-text` (prebuilt, no project). Multi-turn follow-up prompts. Metadata filtering. Active learning. Legacy — retiring March 2029.

### 12.13 Orchestration Workflow (Azure)
Routes utterances to connected CLU/CQA sub-projects. Top-level dispatcher/router model. `projectKind: Orchestration`. Legacy — retiring March 2029.

### 12.14 Confidence / Likelihood Reporting
- **Float 0–1** — Azure (`confidenceScore`), all providers.
- **Threshold parameters** — Azure PII (`confidenceScoreThreshold` with `default` and `overrides[]`).

### 12.15 NLP Tasks via Generative LLMs
All classical NLP tasks can also be performed by generative LLMs via prompting or structured outputs:
- **Classification** → structured output with enum schema.
- **NER** → structured output with entity array schema.
- **PII detection** → prompt instruction + structured output.
- **Summarization** → prompt instruction (abstractive) or structured output (extractive).
- **Sentiment** → structured output with sentiment enum.
- **Question answering** → RAG pipeline or direct prompt.
- **Intent detection** → structured output with intent enum.
- **Key phrase extraction** → structured output with string array.

**Trade-off:** Classical NLP is deterministic, high-throughput, and task-specific. Generative LLMs are flexible but non-deterministic, slower, and more expensive per request.

---

## Stage 13 — Custom NLP Model Training & Deployment (Azure)

### 13.1 Project Lifecycle
All custom features (Custom NER, Custom Text Classification, CLU, CQA, Orchestration) share a common lifecycle:

1. **Import project** — `POST :import` (schema + labeled data from Blob Storage) → 202 + operation-location.
2. **Train model** — `POST :train` (modelLabel, trainingConfigVersion, evaluationOptions with split percentages) → 202 + operation-location.
3. **Evaluate** — train job includes evaluation; view metrics.
4. **Deploy model** — `PUT deployments/{deploymentName}` (trainedModelLabel) → 202 + operation-location.
5. **Swap deployments** — swap test ↔ production.
6. **Query runtime** — use runtime endpoints with `projectName` + `deploymentName`.

### 13.2 Requirements
- **Azure Blob Storage** — for training data (Custom NER, Custom Text Classification, CLU).
- **Azure AI Search** — for CQA knowledge base.
- **Language resource** — linked to storage/search.

### 13.3 Custom Model Deployment Expiry
Custom model deployments expire ~18 months after their training-config version is deployed.

### 13.4 Authoring API Split
- **Text analysis projects** (Custom NER, Custom Text Classification): `/language/authoring/analyze-text/projects/{name}/...`
- **Conversation projects** (CLU, Orchestration): `/language/authoring/analyze-conversations/projects/{name}/...`
- **Question answering projects** (CQA): `/language/authoring/query-knowledgebases/projects/{name}/...`

---

## Stage 14 — Context Management (Compaction, Editing, Caching)

### 14.1 Context Compaction

Server-side summarization of old context when approaching the context-window limit.

| Provider | Mechanism | Trigger |
|----------|-----------|---------|
| Anthropic | `context_management.edits` with `compact_20260112` | `trigger: {type: "input_tokens", value: 150000}` (min 50,000) |
| OpenAI | `context_management` with `compact_threshold` on `/responses`; standalone `POST /responses/compact` | Threshold-based |
| xAI | `POST /v1/responses/compact` | Explicit call |

**Anthropic compaction:** Emits `compaction` block at start of assistant response. On subsequent requests, content blocks prior to compaction are dropped. `pause_after_compaction: true` → `stop_reason: "compaction"`. `usage.iterations` array records compaction + message iterations. Same model used for summarization. Must pass compaction block back in subsequent requests.

**xAI compaction:** Returns opaque `compaction` item with `encrypted_content`. Pass verbatim into next request's `input` (append new user turns **after** it). At most one compaction per call. Re-compacting is fine. Conversation must already fit in context (compaction shrinks, doesn't rescue over-limit).

### 14.2 Context Editing (Anthropic)

Server-side clearing of specific content from conversation history. Client keeps full unmodified history locally.

**Tool result clearing (`clear_tool_uses_20250919`):**
- `trigger` (input_tokens or tool_uses, default 100,000)
- `keep` (number of recent tool use/result pairs, default 3)
- `clear_at_least` (minimum tokens cleared)
- `exclude_tools` (tool names whose results should never be cleared)
- `clear_tool_inputs` (whether to also clear tool call parameters)

**Thinking block clearing (`clear_thinking_20251015`):**
- `keep` (number of recent assistant turns with thinking: `{type: "thinking_turns", value: N}` or `"all"`)

When combining both, `clear_thinking_20251015` **must be listed first** in the `edits` array. Tool result clearing invalidates cached prompt prefixes. Thinking block clearing preserves cache when blocks are kept, invalidates when cleared.

### 14.3 Prompt Caching

| Provider | Mechanism | TTLs | Min tokens | Cache write cost | Cache read cost |
|----------|-----------|------|------------|-----------------|----------------|
| OpenAI | Automatic (Responses API); `prompt_cache_breakpoint` (GPT-5.6+) | 30 min (explicit) | 1024 | Free | 0.25–0.50× input |
| Anthropic | `cache_control` (automatic top-level or explicit per-block, max 4 breakpoints) | 5 min (default), 1 hour | 512–4096 | 1.25× (5m) / 2× (1h) | 0.1× input |
| Google | Context caching (cache files, pay per hour) | Per-hour storage | 1024 (Flash) / 4096 (Pro) | Input + storage | 0.25× input |
| xAI | Automatic (`cached_tokens`, `prompt_cache_key`) | — | — | Free | 0.25× input |
| OpenRouter | Provider sticky routing + `cache_control` | 5 min / 1 hour (Anthropic) | Varies | Varies | Varies |
| DeepSeek (via OpenRouter) | Automatic | — | — | Same as input | 0.1× input |

**Cache hierarchy (Anthropic):** `tools` → `system` → `messages` (each level builds on previous). Mixing TTLs: longer TTL must appear before shorter (1h before 5m). Cache invalidation: tool definitions change → invalidates all levels; tool choice change → invalidates messages only; images/thinking parameters change → invalidates messages only.

**Sticky routing (OpenRouter):** Tracked at account level, per model, per conversation. `session_id` overrides hash as sticky key. Only used when cache-read pricing is cheaper than regular prompt pricing.

### 14.4 Response Caching (OpenRouter)

Distinct from prompt caching: OpenRouter caches **entire identical API responses** at the gateway level. Cache hits are free (all usage counters zeroed).

- Enable: `X-OpenRouter-Cache: true` header.
- Custom TTL: `X-OpenRouter-Cache-TTL` (1–86400 seconds, default 300).
- Force refresh: `X-OpenRouter-Cache-Clear: true`.
- No request coalescing: concurrent identical requests both MISS.

---

## Stage 15 — Batch Processing

### 15.1 Batch APIs

| Provider | Endpoint | Discount | Limits | Result format |
|----------|----------|----------|--------|---------------|
| Anthropic | `POST /v1/messages/batches` | 50% | 100k requests or 256 MB; expire 24h | `.jsonl` via `results_url` |
| Mistral | `POST /v1/batch/jobs` | 50% | 1M requests (file) / 10k (inline) | `output_file` + `error_file` |
| OpenAI | `POST /v1/batches` | — | — | — |
| xAI | `deferred: true` (Chat) → poll `GET /v1/chat/deferred-completion/{request_id}` | — | — | Standard chat completion (200) or pending (202) |

### 15.2 Batch Request Structure
Each item: `{custom_id, params/body}`. `custom_id` maps results to internal IDs. Results may not match input order — match by `custom_id`.

### 15.3 Batch Job Status
- Anthropic: `processing_status` (in_progress → ended; canceling). `request_counts: {processing, succeeded, errored, canceled, expired}`.
- Mistral: `QUEUED`, `RUNNING`, `SUCCESS`, `FAILED`, `TIMEOUT_EXCEEDED`, `CANCELLATION_REQUESTED`, `CANCELLED`.
- xAI: `200 Success` (completed) / `202 Accepted` (pending).

### 15.4 Unsupported Batch Parameters (Anthropic)
`stream: true`, `speed` (Fast mode), `store` / `previous_thread_event_id`, `max_tokens: 0` (cache pre-warming).

### 15.5 Batch + Prompt Caching
Prompt caching stacks with batch discount (Anthropic). Cache hits best-effort (30–98% hit rates). Consider 1-hour cache duration for batch.

### 15.6 Batch Eligibility
- Anthropic: Not eligible for ZDR; data retained up to 29 days. `invalid_request_error` results not billable; errored/canceled/expired not billed.
- Mistral: One model per batch; run multiple batches on same files to compare models.

### 15.7 Mistral Batch Supported Endpoints
`/v1/embeddings`, `/v1/chat/completions`, `/v1/fim/completions`, `/v1/moderations`, `/v1/chat/moderations`, `/v1/ocr`, `/v1/classifications`, `/v1/conversations`, `/v1/audio/transcriptions`.

---

## Stage 16 — Embeddings & Vector Operations

### 16.1 Embedding APIs

| Provider | Endpoint / SDK | Model(s) | Dimensions |
|----------|---------------|----------|------------|
| OpenAI | `/v1/embeddings` | `text-embedding-3-small`, `text-embedding-3-large` | 1536 / 3072 |
| Anthropic (Voyage AI) | `vo.embed()` / `POST https://api.voyageai.com/v1/embeddings` | `voyage-4-large`, `voyage-4`, `voyage-4-lite`, `voyage-4-nano` | 1024 (default) / 256 / 512 / 2048 |
| Mistral | `client.embeddings.create(model, inputs)` | `mistral-embed` | — |
| OpenRouter | `POST /api/v1/embeddings` | Unified across providers | Varies by model |

### 16.2 Input Types
- `input_type="document"` when indexing documents.
- `input_type="query"` when embedding queries.
- Batch: pass array of strings for multiple embeddings in one request.
- Multimodal: some models accept image inputs (OpenRouter `nvidia/llama-nemotron-embed-vl-1b-v2`).
- Contextualized chunk embeddings: `contextualized_embed()` (Voyage AI).

### 16.3 Rerankers (Voyage AI)
`rerank-2.5`, `rerank-2.5-lite` for re-ranking retrieved documents.

### 16.4 Embedding Use Cases
RAG retrieval, clustering, document classification, semantic search, duplicate detection, anomaly detection, code search.

### 16.5 Limitations
- No streaming — embeddings return as complete responses.
- Deterministic — same input always yields identical vectors.
- Token limits per model; long texts may need chunking/truncation.

---

## Stage 17 — Output Processing & Delivery

### 17.1 Stop Reasons / Finish Reasons

| Provider | Values |
|----------|--------|
| OpenAI Responses | `status: completed` / `status: incomplete` (`incomplete_details.reason: max_output_tokens` / `content_filter`) |
| OpenAI Chat | `finish_reason: stop` / `length` / `content_filter` / `tool_calls` |
| Google | `finishReason: STOP` / `MAX_TOKENS` / `SAFETY` / `RECITATION` |
| Anthropic | `stop_reason: end_turn` / `max_tokens` / `stop_sequence` / `tool_use` / `pause_turn` / `refusal` / `model_context_window_exceeded` / `compaction` |
| Mistral | `finish_reason: stop` / `length` / `tool_calls` |
| xAI | `status: completed` |
| OpenRouter | Normalized: `tool_calls` / `stop` / `length` / `content_filter` / `error`; raw in `native_finish_reason` |

### 17.2 Usage / Token Accounting

| Provider | Key fields |
|----------|-----------|
| OpenAI | `usage.prompt_tokens`, `completion_tokens`, `total_tokens`, `prompt_tokens_details.cached_tokens`, `completion_tokens_details.reasoning_tokens` |
| Google | `usageMetadata.total_tokens`, `total_input_tokens`, `total_output_tokens`, `total_thought_tokens` |
| Anthropic | `usage.input_tokens`, `output_tokens`, `cache_creation_input_tokens`, `cache_read_input_tokens`, `cache_creation.ephemeral_5m/1h_input_tokens`, `output_tokens_details.thinking_tokens`, `server_tool_use`, `service_tier`, `iterations` |
| Mistral | `usage` (standard) |
| xAI | `usage.input_tokens`, `output_tokens`, `total_tokens`, `input_tokens_details.cached_tokens`, `output_tokens_details.reasoning_tokens`, `cost_in_usd_ticks`, `cost_in_nano_usd`, `num_sources_used`, `server_side_tool_usage_details` |
| OpenRouter | `usage.prompt_tokens`, `completion_tokens`, `total_tokens`, `prompt_tokens_details.cached_tokens/cache_write_tokens`, `completion_tokens_details.reasoning_tokens`, `cost`, `is_byok`, `cost_details` |
| Azure | Per-document results with `modelVersion` |

### 17.3 Refusal Detection

| Provider | Signal |
|----------|--------|
| OpenAI Chat | `choices[0].message.refusal` |
| OpenAI Responses | Message content item `type === "refusal"` |
| Anthropic | `stop_reason: "refusal"` + `stop_details: {category, explanation, type: "refusal"}` |
| OpenAI moderation | `error.code: moderation_blocked`, `moderation_details.categories` |

### 17.4 Error Handling

**Anthropic streaming error recovery:**
- Claude 4.5 and earlier: resume by placing partial response as start of new assistant message.
- Claude 4.6 and later: place a user message instructing model to continue.
- Tool use and extended thinking blocks cannot be partially recovered; resume from most recent text block.

**xAI video error codes:** `invalid_argument`, `permission_denied`, `failed_precondition`, `service_unavailable`, `internal_error`.

**Azure job errors:** `errors[]` array in results; `status: failed` for async jobs.

### 17.5 Generation Stats (OpenRouter)
`GET /api/v1/generation?id={id}` returns token counts and cost asynchronously. Generation ID also in `X-Generation-Id` response header.

### 17.6 Finish Reason Normalization (OpenRouter)
OpenRouter normalizes `finish_reason` across all providers: `tool_calls`, `stop`, `length`, `content_filter`, `error`. Raw provider value in `native_finish_reason`.

---

## Stage 18 — Usage, Billing & Token Accounting

### 18.1 Billing Models

| Provider | Unit | Notes |
|----------|------|-------|
| OpenAI | Per token (input/output) | Reasoning tokens billed as output; cache reads 0.25–0.50× input; cache writes free (automatic) or 1.25× (explicit) |
| Anthropic | Per token (input/output) | Cache writes 1.25× (5m) / 2× (1h); cache reads 0.1×; batch 50% discount; fast mode per-model multiplier |
| Google | Per token (input/output/thinking) | Context caching ~4× less per request; thinking tokens billed at output rate |
| Mistral | Per token | Batch 50% discount |
| xAI | Per token | `cost_in_usd_ticks` (10B ticks = $1); `cost_in_nano_usd`; reasoning tokens billed as output |
| OpenRouter | Per token (native tokenizer) | `usage.cost` in credits; failed/fallback not billed; zero-completion insurance |
| Azure | Per transaction / per text record | Custom features billed per text record; deployments expire ~18 months |

### 18.2 Token Counting API (Anthropic)
`POST /v1/messages/count_tokens` — pre-send input token estimation. Accepts same params as Messages (non-streaming). Supports `context_management` (returns `input_tokens` after editing + `original_input_tokens` before). Free but RPM-limited by tier. Newer tokenizer (Opus 4.7+, Fable 5, Mythos 5, Sonnet 5) yields ~30% more tokens than earlier models.

### 18.3 Service Tiers

| Provider | Tiers |
|----------|-------|
| Anthropic | `service_tier: auto` / `standard_only`; Priority Tier |
| xAI | `service_tier: default` / `priority` |
| OpenRouter | `service_tier: auto` / `default` / `flex` / `priority` / `scale` |

### 18.4 Rate Limits & Concurrency
- Anthropic: Token-counting and message-creation rate limits are separate; RPM by usage tier (Start 2000, Build 4000, Scale 8000).
- OpenRouter: Failed/fallback attempts not billed.

---

## Stage 19 — Organization, Admin & Asset Management

### 19.1 Workspaces (Anthropic)
Organize API usage within an organization: separate projects/environments/teams with centralized billing & admin. Non-deletable Default Workspace. Max 100 workspaces per org. API keys scoped to single workspace. Resources scoped: Files, Message Batches, Skills, prompt caches.

**Admin API endpoints:** Create/list/archive workspaces, add/update/remove members, usage reports.

**Roles:** Workspace User, Workspace Limited Developer, Workspace Developer, Workspace Admin, Workspace Billing.

### 19.2 Model Routing & Fallbacks (OpenRouter)

**Model-level routing:**
- `models[]` fallback array — if primary errors, try next. Triggers on: context-length, moderation, rate-limit, downtime.
- Router slugs: `openrouter/auto` (meta-model routes to best model), `openrouter/free`, `openrouter/fusion` (multi-model deliberation), Pareto Router (coding quality).
- Session stickiness: pins resolved model + provider per conversation.

**Provider-level routing:**
- `provider.order` — try providers in order (disables load balancing).
- `provider.allow_fallbacks` — allow backup providers.
- `provider.require_parameters` — only use providers supporting all params.
- `provider.data_collection: "deny"` — only providers that don't store/train on data.
- `provider.zdr: true` — restrict to Zero Data Retention endpoints.
- `provider.sort: "price"` / `"throughput"` / `"latency"` — sort providers.
- `provider.preferred_min_throughput` / `preferred_max_latency` — performance floors.
- `provider.max_price` — max pricing (prevents request from running if unavailable).
- `provider.quantizations` — filter by quantization.

### 19.3 Plugins (OpenRouter)
Request/response transforms that always run once when enabled:

| Plugin | Description |
|--------|-------------|
| `web` | Web search (deprecated — use `openrouter:web_search`) |
| `file-parser` | PDF parsing/extraction |
| `response-healing` | Auto-repair malformed JSON responses |
| `context-compression` | Middle-out truncation when prompts exceed context window |
| `moderation` | Content moderation |
| `web-fetch` | URL content fetching with domain filtering |
| `fusion` | Multi-model deliberation |
| `auto-router` | Auto-router config |
| `pareto-router` | Coding quality tier config |

### 19.4 Model Catalog API (OpenRouter)
`GET /api/v1/models` — 400+ models with standardized metadata. Query by `output_modalities`, `supported_parameters`, `sort`. Single model lookup: `GET /api/v1/model/{author}/{slug}` (resolves aliases).

### 19.5 Asset Management

| Provider | Assets |
|----------|--------|
| OpenAI | Videos (list, delete); Files (upload, list, delete) |
| Anthropic | Files (upload, list, metadata, delete, download); Batches (list, cancel, delete); Workspaces |
| xAI | Files (upload, get, delete, public-url create/revoke); stored responses (get, delete) |
| OpenRouter | Models (catalog); Generation stats (query) |
| Azure | Projects (import, train, deploy, swap, delete); Deployments |

### 19.6 Presets (OpenRouter)
Named, server-side configuration (model, fallbacks, provider rules, parameters, system prompt) referenced by slug — routing lives in one place without redeploying.

---

# Part III — The Unified API Specification

This part defines a **provider-agnostic API** that encompasses all features. It is written as a specification for a hypothetical "super complete" text & conversation AI platform.

## API Surface Overview

```
Authentication & Files:
  POST /files                          — Upload file (image/PDF/document) → file_id/uri
  GET  /files                          — List files (paginated)
  GET  /files/{id}                     — Get file metadata/state
  DELETE /files/{id}                    — Delete file
  GET  /files/{id}/download             — Download file content

Text Generation & Conversation:
  POST /chat/completions               — Chat completion (messages[] → choices[])
  POST /responses                      — Response (input → typed output[])
  POST /interactions                   — Interaction (input → typed steps[])
  POST /messages                       — Message (system + messages[] → content[])
  GET  /responses/{id}                 — Retrieve stored response
  DELETE /responses/{id}                — Delete stored response
  POST /conversations                  — Create persistent conversation
  POST /responses/compact              — Compact conversation into opaque item

Reasoning & Configuration (embedded in generation requests):
  reasoning: {effort, max_tokens, exclude, summary, context, mode}
  thinking: {type, budget_tokens, display}

Structured Outputs (embedded in generation requests):
  text.format / response_format: {type: json_schema|json_object|grammar, ...}

Tool Calling (embedded in generation requests):
  tools: [{type: function|custom|namespace|built-in, ...}]
  tool_choice: auto|required|none|specific|allowed_tools

Grounding & Citations:
  (Anthropic) citations: {enabled: true} on document blocks
  (Mistral) tool-call flow with ReferenceChunk
  (Built-in) web_search, google_search, file_search tools

Moderation & Safety:
  POST /moderations                    — Classify text (moderation model)
  POST /chat/moderations               — Classify conversational content
  guardrails: [{moderation_llm_v2: {...}}]  (inline on chat/conversations)
  fallbacks: [{model, max_tokens?, thinking?}]  (refusal fallback)

Batch Processing:
  POST /batches                        — Create batch (async, 50% discount)
  GET  /batches/{id}                   — Get batch status
  GET  /batches/{id}/results           — Stream results (.jsonl)
  POST /batches/{id}/cancel            — Cancel batch
  DELETE /batches/{id}                  — Delete batch

Token Counting:
  POST /messages/count_tokens          — Estimate input tokens

Embeddings:
  POST /embeddings                     — Generate embeddings (text/image)

Classical NLP Analysis:
  POST /language/:analyze-text         — Synchronous single-task NLP
  POST /language/analyze-text/jobs     — Async batch/custom NLP (LRO)
  GET  /language/analyze-text/jobs/{jobId}  — Poll NLP job
  POST /language/:analyze-conversations    — Sync conversation analysis (CLU/Orchestration)
  POST /language/analyze-conversations/jobs — Async conversation analysis (PII/Summarization)
  POST /language/analyze-documents/jobs    — Async document analysis (PII/Summarization)
  POST /language/:query-knowledgebases    — Query QA knowledge base
  POST /language/:query-text               — Query text without project

Custom NLP Authoring:
  POST /language/authoring/.../projects/{name}/:import    — Import project
  POST /language/authoring/.../projects/{name}/:train     — Train model
  PUT  /language/authoring/.../projects/{name}/deployments/{name}  — Deploy model
  POST /language/authoring/.../projects/{name}/swap-deployments    — Swap deployments

Model Catalog & Routing:
  GET  /models                         — List models with metadata
  GET  /models/{author}/{slug}         — Get single model details
  (OpenRouter) models[] fallbacks, provider{} routing, router slugs

Management:
  GET  /users/me                       — Get user info / credit balance
  GET  /organizations/workspaces       — List workspaces (Anthropic)
  POST /organizations/workspaces       — Create workspace
  GET  /organizations/usage_report     — Usage/cost tracking
  GET  /generation?id={id}             — Query generation stats (OpenRouter)
```

---

## Unified Data Structures

### Message / Input Item (Unified)

```json
{
  "role": "system | developer | user | assistant | tool",
  "content": "string | ContentPart[]",
  "name": "optional name",
  "tool_calls": [{"id", "type": "function", "function": {"name", "arguments"}}],
  "tool_call_id": "call_id (tool role)",
  "prefix": false,
  "reasoning": "plaintext reasoning (OpenRouter)",
  "reasoning_details": [{"type": "reasoning.summary|reasoning.encrypted|reasoning.text", ...}]
}
```

### Content Part (Unified)

```json
// Text
{"type": "text", "text": "..."}

// Image
{"type": "image_url", "image_url": {"url": "https://... or data:image/...;base64,...", "detail": "auto|low|high|original"}}
// or
{"type": "input_image", "image_url": "...", "detail": "auto"}

// File / Document
{"type": "input_file", "file_id": "file-...", "detail": "auto|low|high"}
// or
{"type": "input_file", "file_url": "https://...", "detail": "auto|low|high"}
// or
{"type": "input_file", "file_data": "data:application/pdf;base64,...", "filename": "doc.pdf"}

// Anthropic-specific content blocks
{"type": "document", "source": {"type": "url|base64|file", ...}, "citations": {"enabled": true}, "title": "...", "context": "..."}
{"type": "search_result", "source": "...", "title": "...", "content": [{"type": "text", "text": "..."}], "citations": {"enabled": true}}
{"type": "thinking", "thinking": "...", "signature": "..."}
{"type": "redacted_thinking", "data": "encrypted..."}
{"type": "tool_use", "id": "...", "name": "...", "input": {...}}
{"type": "tool_result", "tool_use_id": "...", "content": "..."}
{"type": "compaction", "content": "Summary..."}

// Mistral-specific chunks
{"type": "thinking", "thinking": [{"type": "text", "text": "..."}]}
{"type": "reference", "reference_ids": ["0", "1"]}
```

### Reasoning Configuration (Unified)

```json
{
  "reasoning": {
    "effort": "none | minimal | low | medium | high | xhigh | max",
    "max_tokens": 10000,
    "exclude": false,
    "enabled": true,
    "summary": "auto | concise | detailed",
    "context": "auto | all_turns | current_turn",
    "mode": "standard | pro"
  },
  // Anthropic alternative:
  "thinking": {
    "type": "enabled | adaptive | disabled",
    "budget_tokens": 10000,
    "display": "summarized | omitted"
  },
  "output_config": {
    "effort": "low | medium | high | xhigh | max",
    "task_budget": {"type": "tokens", "total": 64000, "remaining": 50000}
  }
}
```

### Structured Output Configuration (Unified)

```json
{
  "text": {
    "format": {
      "type": "json_schema | json_object | text | grammar",
      "name": "schema_name",
      "strict": true,
      "schema": {
        "type": "object",
        "properties": {...},
        "required": [...],
        "additionalProperties": false
      }
    }
  },
  // Chat Completions equivalent:
  "response_format": {
    "type": "json_schema",
    "json_schema": {"name": "...", "strict": true, "schema": {...}}
  },
  // Google equivalent:
  "response_format": {"type": "text", "mime_type": "application/json", "schema": {...}},
  // Anthropic equivalent:
  "output_config": {"format": {"type": "json_schema", "schema": {...}}}
}
```

### Tool Definition (Unified)

```json
// Function tool (modern / flat shape)
{
  "type": "function",
  "name": "get_weather",
  "description": "Retrieves current weather.",
  "parameters": {"type": "object", "properties": {...}, "required": [...], "additionalProperties": false},
  "strict": true,
  "defer_loading": false
}

// Function tool (legacy / externally tagged)
{
  "type": "function",
  "function": {"name": "get_weather", "description": "...", "parameters": {...}, "strict": true}
}

// Custom tool (free-form text I/O)
{
  "type": "custom",
  "name": "code_exec",
  "description": "Execute code.",
  "format": {"type": "grammar", "syntax": "lark|regex", "definition": "..."}
}

// Namespace
{
  "type": "namespace",
  "name": "crm",
  "description": "CRM tools.",
  "tools": [...]
}

// Built-in tool
{"type": "web_search"}
{"type": "google_search"}
{"type": "file_search"}
{"type": "code_execution"}
{"type": "url_context"}
{"type": "computer_use"}
{"type": "image_generation"}
{"type": "mcp_server", "name": "...", "url": "...", "headers": {...}, "allowed_tools": [...]}
{"type": "tool_search"}
```

### Tool Call & Result (Unified)

```json
// Tool call in output
{
  "type": "function_call",
  "id": "call_...",
  "call_id": "call_...",
  "name": "get_weather",
  "arguments": "{\"location\": \"London\"}"
}
// or (Chat Completions)
{"id": "call_...", "type": "function", "function": {"name": "get_weather", "arguments": "..."}}
// or (Anthropic)
{"type": "tool_use", "id": "...", "name": "get_weather", "input": {"location": "London"}}
// or (Google step)
{"type": "function_call", "id": "...", "name": "get_weather", "arguments": {...}}

// Tool result returned to model
{"type": "function_call_output", "call_id": "call_...", "output": "{\"temp\": 15}"}
// or (Chat Completions)
{"role": "tool", "tool_call_id": "call_...", "content": "{\"temp\": 15}"}
// or (Anthropic)
{"type": "tool_result", "tool_use_id": "...", "content": "..."}
// or (Google step)
{"type": "function_result", "name": "get_weather", "call_id": "...", "result": [{"type": "text", "text": "..."}]}
```

### Sampling Parameters (Unified)

```json
{
  "temperature": 1.0,
  "top_p": 1.0,
  "top_k": 0,
  "min_p": 0.0,
  "max_tokens": 4096,
  "max_output_tokens": 4096,
  "max_completion_tokens": 4096,
  "stop": ["..."],
  "stop_sequences": ["..."],
  "frequency_penalty": 0.0,
  "presence_penalty": 0.0,
  "repetition_penalty": 1.0,
  "seed": 42,
  "random_seed": 42,
  "n": 1,
  "logprobs": false,
  "top_logprobs": 0,
  "logit_bias": {},
  "verbosity": "medium",
  "prediction": {"type": "content", "content": "predicted output..."},
  "service_tier": "auto | default | flex | priority | scale",
  "speed": "standard | fast"
}
```

### Prompt Caching Configuration (Unified)

```json
{
  // Automatic (top-level)
  "cache_control": {"type": "ephemeral", "ttl": "5m | 1h"},
  // Explicit (per content block)
  "cache_control": {"type": "ephemeral", "ttl": "5m"},
  // OpenAI explicit (GPT-5.6+)
  "prompt_cache_breakpoint": {...},
  "prompt_cache_key": "stable-key",
  // OpenRouter
  "session_id": "sticky-routing-key"
}
```

### Context Management Configuration (Unified, Anthropic-style)

```json
{
  "context_management": {
    "edits": [
      {
        "type": "compact_20260112",
        "trigger": {"type": "input_tokens", "value": 150000},
        "pause_after_compaction": false,
        "instructions": "custom summarization prompt"
      },
      {
        "type": "clear_thinking_20251015",
        "keep": {"type": "thinking_turns", "value": 3}
      },
      {
        "type": "clear_tool_uses_20250919",
        "trigger": {"type": "input_tokens", "value": 100000},
        "keep": {"type": "tool_uses", "value": 3},
        "clear_at_least": {"type": "input_tokens", "value": 5000},
        "exclude_tools": ["web_search"],
        "clear_tool_inputs": false
      }
    ]
  }
}
```

### Moderation / Guardrails Configuration (Unified, Mistral-style)

```json
{
  "guardrails": [
    {
      "block_on_error": true,
      "moderation_llm_v2": {
        "custom_category_thresholds": {"sexual": 0.1, "selfharm": 0.1},
        "ignore_other_categories": false,
        "action": "block"
      }
    }
  ],
  "safe_prompt": false,
  "moderation": "auto | low",
  "fallbacks": [
    {"model": "fallback-model-id", "max_tokens": 1024, "thinking": {"type": "adaptive"}}
  ]
}
```

### PII Redaction Configuration (Unified, Azure-style)

```json
{
  "kind": "PiiEntityRecognition",
  "parameters": {
    "modelVersion": "latest",
    "piiCategories": ["Person", "Organization", "PhoneNumber"],
    "redactionPolicies": [
      {"policyKind": "CharacterMask", "redactionCharacter": "*", "defaultRedactionPolicy": true}
    ],
    "confidenceScoreThreshold": {"default": 0.5, "overrides": [{"category": "Person", "threshold": 0.8}]},
    "disableEntityValidation": false
  }
}
```

### Azure NLP Document Input (Unified)

```json
{
  "kind": "EntityRecognition",
  "parameters": {"modelVersion": "latest", "stringIndexType": "Utf16CodeUnit"},
  "analysisInput": {
    "documents": [
      {"id": "1", "language": "en", "text": "Microsoft was founded by Bill Gates.", "countryHint": "us"}
    ]
  }
}
```

### Azure NLP Async Job Input (Unified)

```json
{
  "displayName": "Job name",
  "analysisInput": {"documents": [{"id": "1", "language": "en", "text": "..."}]},
  "tasks": [
    {"kind": "EntityRecognition", "taskName": "NER", "parameters": {"modelVersion": "latest"}},
    {"kind": "ExtractiveSummarization", "taskName": "Summary", "parameters": {"sentenceCount": 3, "sortby": "Offset"}}
  ]
}
```

### Generation Request (Unified — full union of all parameters)

```json
{
  "model": "string | null",
  "input": "string | Item[]",
  "messages": "Message[] (legacy)",
  "instructions": "string (system guidance, top-level)",
  "system": "string | TextBlock[] (Anthropic)",
  "system_instruction": "string (Google)",

  "store": true,
  "previous_response_id": "resp_...",
  "previous_interaction_id": "interaction_...",
  "conversation": "conv_...",
  "include": ["reasoning.encrypted_content"],

  "reasoning": {"effort": "high", "exclude": false, "summary": "auto", "context": "all_turns", "mode": "pro"},
  "thinking": {"type": "adaptive", "display": "summarized"},
  "output_config": {"effort": "high", "task_budget": {"type": "tokens", "total": 64000}},
  "speed": "standard",

  "text": {"format": {"type": "json_schema", "name": "...", "strict": true, "schema": {...}}},
  "response_format": {"type": "json_schema", "json_schema": {...}},

  "tools": [{"type": "function", "name": "...", "parameters": {...}, "strict": true}],
  "tool_choice": "auto",
  "parallel_tool_calls": true,

  "temperature": 1.0, "top_p": 1.0, "top_k": 0, "min_p": 0.0,
  "max_output_tokens": 4096, "max_tokens": 4096,
  "stop": ["..."], "stop_sequences": ["..."],
  "frequency_penalty": 0.0, "presence_penalty": 0.0, "repetition_penalty": 1.0,
  "seed": 42, "random_seed": 42, "n": 1,
  "logprobs": false, "top_logprobs": 0, "logit_bias": {},
  "verbosity": "medium",
  "prediction": {"type": "content", "content": "..."},
  "service_tier": "auto",

  "stream": true,

  "cache_control": {"type": "ephemeral", "ttl": "1h"},
  "prompt_cache_key": "stable-key",
  "session_id": "sticky-key",

  "context_management": {"edits": [{"type": "compact_20260112", "trigger": {"type": "input_tokens", "value": 150000}}]},

  "guardrails": [{"moderation_llm_v2": {"custom_category_thresholds": {"sexual": 0.1}, "action": "block"}}],
  "safe_prompt": false,
  "moderation": "auto",
  "fallbacks": [{"model": "fallback-model"}],

  "metadata": {"user_id": "..."},
  "user": "end-user-id",
  "container": "container-id",
  "inference_geo": "region",
  "plugins": [{"id": "response-healing"}],
  "models": ["fallback-model-1", "fallback-model-2"],
  "provider": {"order": ["anthropic"], "data_collection": "deny", "zdr": true, "sort": "price"},
  "debug": {"echo_upstream_body": true}
}
```

### Generation Response (Unified)

```json
{
  "id": "resp_...",
  "object": "response",
  "model": "model-that-served",
  "status": "completed | incomplete",
  "store": true,
  "created_at": 1756315696,

  "output": [
    {
      "type": "reasoning",
      "id": "rs_...",
      "summary": [{"type": "summary", "text": "..."}],
      "encrypted_content": "opaque-blob"
    },
    {
      "type": "message",
      "id": "msg_...",
      "role": "assistant",
      "status": "completed",
      "content": [
        {"type": "output_text", "text": "Final answer...", "annotations": [], "logprobs": []}
      ]
    }
  ],

  "steps": [
    {"type": "thought", "signature": "...", "summary": [{"type": "text", "text": "..."}]},
    {"type": "model_output", "content": [{"type": "text", "text": "..."}]}
  ],

  "choices": [
    {
      "index": 0,
      "finish_reason": "stop",
      "native_finish_reason": "stop",
      "message": {"role": "assistant", "content": "..." }
    }
  ],

  "content": [
    {"type": "thinking", "thinking": "...", "signature": "..."},
    {"type": "text", "text": "...", "citations": [...]}
  ],

  "output_text": "convenience: all text joined",

  "usage": {
    "input_tokens": 100,
    "output_tokens": 50,
    "total_tokens": 150,
    "prompt_tokens": 100,
    "completion_tokens": 50,
    "prompt_tokens_details": {"cached_tokens": 80, "cache_write_tokens": 20},
    "completion_tokens_details": {"reasoning_tokens": 30},
    "cache_creation_input_tokens": 20,
    "cache_read_input_tokens": 80,
    "cache_creation": {"ephemeral_5m_input_tokens": 10, "ephemeral_1h_input_tokens": 10},
    "output_tokens_details": {"thinking_tokens": 30},
    "total_thought_tokens": 30,
    "cost": 0.0015,
    "cost_in_usd_ticks": 15000000,
    "server_tool_use": {"web_search_requests": 1},
    "service_tier": "standard",
    "iterations": [{"type": "message", "input_tokens": 100, "output_tokens": 50}]
  },

  "stop_reason": "end_turn",
  "stop_sequence": null,
  "stop_details": null,
  "finish_reason": "stop",

  "guardrails": [{"moderation_llm_v2": {"action": "pass", "categories": {"sexual": {"score": 0.03, "violated": false}}}}]
}
```

### Streaming Event (Unified)

```
event: response.created / interaction.created / message_start
event: step.start / content_block_start
event: step.delta / content_block_delta / response.output_text.delta
  delta types: text, thought_summary, thought_signature, arguments, thinking_delta, signature_delta, citations_delta, compaction_delta
event: step.stop / content_block_stop
event: message_delta (stop_reason, cumulative usage)
event: response.completed / interaction.completed / message_stop
event: done / [DONE]
event: error
```

### Azure NLP Analysis Response (Unified)

```json
{
  "kind": "EntityRecognitionResults",
  "results": {
    "documents": [
      {
        "id": "1",
        "entities": [
          {
            "text": "Microsoft",
            "category": "Organization",
            "subcategory": null,
            "type": "Organization",
            "tags": [{"name": "Organization", "confidenceScore": 0.96}],
            "metadata": {"metadataKind": "EntityMetadata"},
            "offset": 0,
            "length": 9,
            "confidenceScore": 0.96
          }
        ],
        "warnings": []
      }
    ],
    "errors": [],
    "modelVersion": "2023-09-01"
  }
}
```

### Azure NLP Async Job Response (Unified)

```json
{
  "jobId": "...",
  "displayName": "...",
  "status": "succeeded",
  "createdDateTime": "...",
  "expirationDateTime": "...",
  "tasks": {
    "completed": 2, "failed": 0, "inProgress": 0, "total": 2,
    "items": [
      {
        "kind": "EntityRecognitionResults",
        "taskName": "NER",
        "results": {"documents": [...], "errors": [], "modelVersion": "..."}
      },
      {
        "kind": "ExtractiveSummarizationLROResults",
        "taskName": "Summary",
        "results": {"documents": [{"id": "1", "sentences": [{"text": "...", "rankScore": 0.95, "offset": 0, "length": 100}], "warnings": []}], "errors": [], "modelVersion": "..."}
      }
    ]
  }
}
```

### Batch Request (Unified)

```json
{
  "requests": [
    {"custom_id": "req-1", "params": {"model": "...", "max_tokens": 1024, "messages": [{"role": "user", "content": "Hello"}]}},
    {"custom_id": "req-2", "params": {"model": "...", "max_tokens": 1024, "messages": [{"role": "user", "content": "Hi"}]}}
  ]
}
```

### Batch Response (Unified)

```json
{
  "id": "batch_...",
  "type": "message_batch",
  "processing_status": "in_progress | ended | canceling",
  "request_counts": {"processing": 0, "succeeded": 2, "errored": 0, "canceled": 0, "expired": 0},
  "created_at": "...",
  "expires_at": "...",
  "ended_at": "...",
  "results_url": "https://..."
}
```

### Embeddings Request / Response (Unified)

```json
// Request
{
  "model": "mistral-embed | text-embedding-3-large | voyage-4",
  "input": "text or [text1, text2] or [{content: [image_part]}]",
  "encoding_format": "float",
  "input_type": "document | query"
}

// Response
{
  "data": [{"embedding": [0.1, -0.2, ...], "index": 0}],
  "usage": {"prompt_tokens": 10, "total_tokens": 10}
}
```

---

## Capability Decision Matrix

The definitive guide to choosing the right capability for any need:

### Text Generation & Conversation

| If you need... | Use this capability | Key parameters |
|----------------|---------------------|---------------|
| Generate text from a prompt | Text generation | `model`, `input`/`messages`, `instructions`/`system` |
| Multi-turn conversation (server-managed state) | `previous_response_id` / `previous_interaction_id` | `store: true`, `previous_response_id` |
| Multi-turn conversation (client-managed state) | Manual replay | Resend full history each turn |
| Persistent conversation across sessions | Conversations API | `conversation: "conv_..."` (OpenAI), `/v1/conversations` (Mistral) |
| Stateless + reasoning preservation | Encrypted reasoning replay | `store: false`, `include: ["reasoning.encrypted_content"]` |
| Guide model to start response with specific text | Prefill / prefix | Trailing `assistant` message (Mistral `prefix: true`; Anthropic deprecated on newer models) |
| Multiple candidate responses | `n` parameter | `n: 3` (OpenAI Chat, xAI Chat) |
| Pin response language | System prompt | `"Always respond in French."` in system/instructions |

### Reasoning

| If you need... | Use this capability | Key parameters |
|----------------|---------------------|---------------|
| Control reasoning depth | Effort/thinking level | `reasoning.effort` / `thinking_level` / `output_config.effort` |
| Explicit reasoning token budget | Budget-based thinking | `thinking.budget_tokens` (Anthropic) / `reasoning.max_tokens` (OpenRouter) |
| Model decides reasoning depth | Adaptive thinking | `thinking: {type: "adaptive"}` (Anthropic) |
| Total work budget across agentic loop | Task budgets | `output_config.task_budget` (Anthropic, beta) |
| Preserve reasoning across stateless turns | Encrypted reasoning | `include: ["reasoning.encrypted_content"]`, pass blobs back |
| View reasoning summary | Thought summaries | `thinking_summaries: "auto"` (Google) / `display: "summarized"` (Anthropic) / `reasoning.summary` (OpenAI) |
| Hide reasoning from output but still use it | Exclude reasoning | `reasoning.exclude: true` (OpenRouter) |
| Faster output tokens | Fast mode | `speed: "fast"` (Anthropic Opus 4.8/4.7) / `:nitro` (OpenRouter) |
| Deeper multi-pass reasoning | Pro mode | `reasoning.mode: "pro"` (GPT-5.6+ via OpenRouter) |

### Structured Outputs

| If you need... | Use this capability | Key parameters |
|----------------|---------------------|---------------|
| JSON conforming to a schema | Structured Outputs | `text.format` / `response_format` with `json_schema` + `strict: true` |
| Just valid JSON, no schema | JSON Mode | `json_object` type |
| Schema with optional fields | Union with null | `"type": ["string", "null"]`, all fields in `required` |
| Recursive schemas | `$defs` + `$ref` | `$ref: "#/$defs/..."` |
| Constrain free-form text via grammar | CFG constraint | Custom tool + `format: {type: "grammar"}` (OpenAI) |
| Auto-repair malformed JSON | Response healing | `response-healing` plugin (OpenRouter) |
| Structured output + tools in same request | Combined (Gemini 3, Grok 4) | `response_format` + `tools` in same request |
| Typed parsing in SDK | SDK parse helper | `client.responses.parse()` / `client.chat.parse()` / `chat.parse(Model)` |

### Function Calling & Tools

| If you need... | Use this capability | Key parameters |
|----------------|---------------------|---------------|
| Structured arguments to external function | Function tool | `type: "function"`, `parameters` (JSON Schema), `strict: true` |
| Free-form text input to tool | Custom tool | `type: "custom"` (OpenAI) |
| Web search grounding | Built-in web search | `web_search` (OpenAI) / `google_search` (Google) |
| File retrieval | Built-in file search | `file_search` (OpenAI / Google) |
| Code execution | Built-in code interpreter | `code_interpreter` (OpenAI) / `code_execution` (Google) |
| Remote MCP server | MCP tool | `mcp_server` (Google) / MCP (OpenAI) |
| Large tool ecosystem (>20 tools) | Tool search | `tool_search` (OpenAI GPT-5.4+) with `defer_loading: true` |
| Group tools by domain | Namespaces | `type: "namespace"` (OpenAI) |
| Force a specific tool | Forced tool choice | `tool_choice: {type: "function", name: "..."}` / `{type: "tool", name: "..."}` |
| Restrict to subset of tools | Allowed tools | `tool_choice: {type: "allowed_tools", tools: [...]}` |
| Multiple tool calls per turn | Parallel tool calling | `parallel_tool_calls: true` (default) |
| Tool calls always conform to schema | Strict mode | `strict: true` on tool definition |

### Grounding & RAG

| If you need... | Use this capability | Key parameters |
|----------------|---------------------|---------------|
| Ground in provided documents with citations | Document citations | `citations: {enabled: true}` on document blocks (Anthropic) |
| RAG-style citations from search results | Search result citations | `search_result` blocks (Anthropic) / `ReferenceChunk` (Mistral) |
| Managed RAG (no infrastructure) | Libraries / CQA | Mistral Libraries + `document_library` tool; Azure CQA |
| Full control over chunking/retrieval | RAG from scratch | Embeddings API + vector DB + chat |
| Cache large context for repeated queries | Context caching | Google context caching; Anthropic prompt caching |
| Ground answers in real-time web info | Web search tool | `web_search` / `google_search` built-in tool |

### Moderation & Safety

| If you need... | Use this capability | Key parameters |
|----------------|---------------------|---------------|
| Inline input moderation | Guardrails | `guardrails` array with `moderation_llm_v2` (Mistral) |
| Classify text for policy categories | Moderation API | `POST /moderations` (Mistral) |
| Auto-fallback on safety refusal | Server-side fallback | `fallbacks` parameter (Anthropic) |
| Detect & redact PII in text | PII detection (text) | `kind: PiiEntityRecognition` (Azure) |
| Detect & redact PII in conversations | PII detection (conversation) | `kind: ConversationalPIITask` (Azure) |
| Detect & redact PII in documents | PII detection (document) | `kind: PiiEntityRecognition` on documents (Azure) |
| Custom redaction style | Redaction policies | `CharacterMask` / `NoMask` / `EntityMask` / `SyntheticReplacement` |
| Detect refusals programmatically | Refusal detection | `stop_reason: "refusal"` / `message.refusal` / `type: "refusal"` |

### Context Management

| If you need... | Use this capability | Key parameters |
|----------------|---------------------|---------------|
| Long conversation approaching context limit | Compaction | `context_management.edits` (Anthropic) / `/responses/compact` (OpenAI, xAI) |
| Clear old tool results server-side | Context editing | `clear_tool_uses_20250919` (Anthropic) |
| Clear old thinking blocks server-side | Context editing | `clear_thinking_20251015` (Anthropic) |
| Reduce cost of repetitive prompts | Prompt caching | `cache_control` (Anthropic) / automatic (OpenAI, Google) |
| Cache identical full responses | Response caching | `X-OpenRouter-Cache: true` (OpenRouter) |
| Keep cache warm across turns | Sticky routing | `session_id` (OpenRouter) / `prompt_cache_key` (xAI) |

### Batch & Async

| If you need... | Use this capability | Key parameters |
|----------------|---------------------|---------------|
| Bulk async processing at 50% discount | Batch API | `POST /batches` (Anthropic, Mistral) |
| Async single request (long-running) | Deferred completion | `deferred: true` (xAI Chat) |
| Pre-send token estimation | Token counting | `POST /messages/count_tokens` (Anthropic) |

### Classical NLP

| If you need... | Use this capability | Key parameters |
|----------------|---------------------|---------------|
| Detect language of text | Language Detection | `kind: LanguageDetection` (Azure) |
| Extract named entities | NER | `kind: EntityRecognition` (Azure) |
| Extract domain-specific entities | Custom NER | Train custom model → `kind: CustomEntityRecognition` |
| Detect sentiment + opinions | Sentiment & Opinion Mining | `kind: SentimentAnalysis` with `opinionMining: "True"` |
| Extract key phrases | Key Phrase Extraction | `kind: KeyPhraseExtraction` |
| Link entities to Wikipedia | Entity Linking | `kind: EntityLinking` |
| Summarize text (extractive) | Extractive Summarization | `kind: ExtractiveSummarization` with `sentenceCount`, `sortby` |
| Summarize text (abstractive) | Abstractive Summarization | `kind: AbstractiveSummarization` with `summaryLength` |
| Summarize conversations | Conversation Summarization | `kind: ConversationalSummarizationTask` with `summaryAspects` |
| Biomedical NLP | Text Analytics for Health | `kind: Healthcare` with optional `fhirVersion` |
| Classify text (custom) | Custom Text Classification | Train → `kind: CustomMultiLabelClassification` / `CustomSingleLabelClassification` |
| Predict intents from utterances | CLU | Train → `kind: Conversation` (Azure) |
| Question answering from KB | CQA | `query-knowledgebases` / `query-text` (Azure) |
| Route utterances to sub-projects | Orchestration Workflow | `projectKind: Orchestration` (Azure) |

### Multimodal

| If you need... | Use this capability | Key parameters |
|----------------|---------------------|---------------|
| Analyze images alongside text | Vision input | `image` / `image_url` / `input_image` content parts |
| Analyze PDFs / documents | Document input | `input_file` / `document` blocks; `detail: auto/low/high` (OpenAI PDFs) |
| Analyze video | Video input | Video parts (Google only) |
| Analyze audio | Audio input | Audio parts (Google only) |
| Upload files for reuse | Files API | `POST /files` → `file_id` / `uri` |

### Model Selection & Routing

| If you need... | Use this capability | Key parameters |
|----------------|---------------------|---------------|
| Automatic model selection | Auto Router | `model: "openrouter/auto"` (OpenRouter) |
| Fallback across models | Model fallbacks | `models: ["primary", "fallback1", "fallback2"]` (OpenRouter) |
| Cheapest serving | Floor variant | `:floor` suffix or `provider.sort: "price"` (OpenRouter) |
| Fastest serving | Nitro variant | `:nitro` suffix or `provider.sort: "throughput"` (OpenRouter) |
| Best tool-calling accuracy | Exacto variant | `:exacto` suffix (OpenRouter) |
| Data privacy | Privacy routing | `provider.data_collection: "deny"` / `provider.zdr: true` (OpenRouter) |
| Specific provider/region | Provider ordering | `provider.order: ["google-vertex/us-east5"]` (OpenRouter) |
| Always newest model version | Latest alias | `~openai/gpt-latest` (OpenRouter) / `mistral-large-latest` |
| Browse available models | Model catalog | `GET /models` (OpenRouter) |

---

## Sync vs Async Reference

| Provider / surface | Pattern |
|--------------------|---------|
| OpenAI Responses / Chat | Synchronous (with optional streaming) |
| OpenAI Batch | Asynchronous (poll or webhook) |
| Google Interactions / generateContent | Synchronous (with optional streaming) |
| Google Live API | WebSocket bi-directional streaming |
| Anthropic Messages | Synchronous (with optional streaming) |
| Anthropic Batches | Asynchronous (poll) |
| Mistral Chat Completions | Synchronous (with optional streaming) |
| Mistral Batch | Asynchronous (poll) |
| xAI Responses | Synchronous (with optional streaming) |
| xAI Deferred | Asynchronous (poll) |
| OpenRouter (all surfaces) | Synchronous (with optional streaming) |
| Azure `:analyze-text` | Synchronous |
| Azure `analyze-text/jobs` | Asynchronous (LRO poll) |
| Azure `:analyze-conversations` | Synchronous |
| Azure `analyze-conversations/jobs` | Asynchronous (LRO poll) |
| Azure `analyze-documents/jobs` | Asynchronous (LRO poll) |
| Azure `:query-knowledgebases` / `:query-text` | Synchronous |
| Azure Authoring (`:import`, `:train`, deployments) | Asynchronous (LRO poll) |

---

## State Management Reference

| Provider | Stateless | Server-side state | Persistent conversation | Encrypted reasoning |
|----------|-----------|-------------------|------------------------|-------------------|
| OpenAI | Manual replay | `previous_response_id` + `store: true` | Conversations API | `store: false` + `include` |
| Google | `store: false` + step replay | `previous_interaction_id` + `store: true` | — | Resend thought signatures |
| Anthropic | Always (resend messages) | — (no server-side state) | — | Resend thinking blocks with signatures |
| Mistral | Manual replay | — | Conversations API (beta) | Resend ThinkChunks |
| xAI | Manual replay | `previous_response_id` + `store: true` (30-day TTL) | — | `store: false` + encrypted reasoning |
| OpenRouter | Always (stateless) | — | — | `reasoning_details` replay |

---

## Reasoning Effort Mapping Reference

| Unified effort | OpenAI | Google | Anthropic | Mistral | xAI | OpenRouter |
|---------------|--------|--------|-----------|---------|-----|------------|
| None / disabled | `none` | (off / `minimal`) | `type: disabled` | `reasoning_effort: none` | — (cannot disable) | `none` |
| Minimal | `minimal` | `minimal` | — | — | — | `minimal` |
| Low | `low` | `low` | `low` | — | `low` | `low` |
| Medium | `medium` | `medium` | `medium` | — | `medium` | `medium` |
| High (default) | `high` | `high` | `high` | `reasoning_effort: high` | `high` | `high` |
| Xhigh | — | — | `xhigh` | — | `xhigh` (multi-agent: 16) | `xhigh` |
| Max | — | — | `max` | — | — | `max` |

---

## Structured Output Schema Support Reference

| Feature | OpenAI | Google | Anthropic | xAI | OpenRouter |
|---------|--------|--------|-----------|-----|------------|
| `string` | ✔ | ✔ | ✔ | ✔ | ✔ |
| `number` / `integer` | ✔ | ✔ | ✔ | ✔ | ✔ |
| `boolean` | ✔ | ✔ | ✔ | ✔ | ✔ |
| `object` | ✔ | ✔ | ✔ | ✔ | ✔ |
| `array` | ✔ | ✔ (items, prefixItems, minItems, maxItems) | ✔ | ✔ | ✔ |
| `enum` | ✔ | ✔ | ✔ | ✔ | ✔ |
| `anyOf` | ✔ | ✔ | ✔ | ✔ (behaves like anyOf) | ✔ |
| `oneOf` | — | — | — | ✔ (behaves like anyOf) | — |
| `allOf` | — | — | — | Best-effort (single subschema only) | — |
| `not` | — | — | — | Best-effort | — |
| `$ref` / `$defs` | ✔ (recursive) | ✔ (recursive) | ✔ | ✔ (non-circular only) | ✔ |
| `pattern` (regex) | ✔ | — | — | ✔ (ECMAScript subset) | ✔ |
| `format` (string) | ✔ (date-time, email, uuid, etc.) | ✔ (date-time, date, time) | — | ✔ (date, time, date-time, email, uuid, ipv4, ipv6, uri) | ✔ |
| `minimum` / `maximum` | ✔ | ✔ | — | ✔ (no limit) | ✔ |
| `minLength` / `maxLength` | ✔ | — | — | ✔ (up to 2048) | ✔ |
| `minItems` / `maxItems` | ✔ | ✔ | — | ✔ (up to 256) | ✔ |
| `additionalProperties: false` | Required (strict) | ✔ (boolean or schema) | ✔ | Defaults to false | Required (strict) |
| `if`/`then`/`else` | — | — | — | Best-effort | — |
| `dependentRequired` | — | — | — | — | — |

---

## Summary of Unique Provider Capabilities

Each provider contributes unique features that no other provider offers:

| Provider | Unique capabilities not found elsewhere |
|----------|----------------------------------------|
| **OpenAI** | Conversations API (`/v1/conversations`) for persistent cross-session state; tool namespaces (`type: "namespace"`); tool search / deferred tool loading (GPT-5.4+); custom tools with context-free grammars (`lark`/`regex`); reasoning context mode (`all_turns`/`current_turn`, GPT-5.6+); reasoning mode `pro` (deeper multi-pass); `n` parameter (multiple candidates, Chat); spreadsheet augmentation (1000 rows/sheet); PDF `detail` control (`auto`/`low`/`high`); built-in `computer_use` tool; `apply_patch` / `local_shell` tools |
| **Google Gemini** | Interactions API with first-class `steps` model (chronologically ordered typed steps); thought steps with `signature` + optional `summary` (text/image); `thinking_summaries` control; `thinking_level` (minimal/low/medium/high); video and audio as native input modalities; Live API (WebSocket bi-directional streaming); context caching (~4× less per request); `url_context` built-in tool; `mcp_server` tool type (Streamable HTTP only); `validated` tool_choice mode; multimodal function responses (images in result); Google Search grounding with `search_suggestions` HTML; `media_resolution` control; 1M+ token context windows; `personGeneration` controls |
| **Anthropic Claude** | Mid-conversation `system` role messages (Opus 4.8, preserves prompt caching); `thinking` modes: `enabled` (manual budget) / `adaptive` (model decides) / `disabled`; `display: summarized|omitted` for thinking; `redacted_thinking` blocks (safety-redacted, opaque); `effort` levels including `xhigh` (30+ min agentic/coding) and `max` (absolute maximum); `output_config.task_budget` (advisory loop-level token budget, beta); server-side context compaction (`compact_20260112`, beta) with `pause_after_compaction` and custom `instructions`; context editing (`clear_tool_uses_20250919` with `exclude_tools`/`clear_tool_inputs`, `clear_thinking_20251015`); document citations with char/page/content_block locations; `search_result` content blocks for RAG citations; server-side refusal `fallbacks` (up to 3 models, `usage.iterations`); `stop_details` with policy categories (`cyber`/`bio`/`frontier_llm`/`reasoning_extraction`); Fast mode (`speed: "fast"`, 2.5× OTPS on Opus 4.8/4.7, beta); `output_config.format` structured outputs via constrained decoding (compiled grammars, 24h cache, 180s timeout); strict tool complexity limits (20 strict tools, 24 optional params, 16 union-type params); `pause_turn` stop reason (server-tool loop iteration limit); `model_context_window_exceeded` stop reason; `inference_geo` (geographic region for inference); Workspaces Admin API (roles, spend limits, usage reports); `max_tokens: 0` cache pre-warming; Voyage AI multimodal embeddings (text+images+video); `container` for code execution reuse; `anthropic-user-profile-id` header |
| **Mistral** | `guardrails` array with `moderation_llm_v2` (inline input moderation, per-category thresholds, `ignore_other_categories`, `block_on_error`, HTTP 403); dedicated Moderation API with 11 policy categories (sexual, hate, discrimination, violence, dangerous, criminal, self-harm, health, financial, law, PII, jailbreaking); `mistral-moderation-2603` model; `prefix: true` on assistant messages (prefill); `safe_prompt` (legacy moderation flag); citations via tool-call flow with `ReferenceChunk` + references JSON map; `ThinkChunk` in content (reasoning trace as list of TextChunks); Libraries (managed RAG: upload docs → ingest/vectorize/search → `document_library` tool → grounded answers with `tool_reference` citations); Libraries access control (Viewer/Editor, User/Workspace/Org); Conversations API (beta) with agent-level guardrails; batch supports 9 endpoints including `/v1/ocr` and `/v1/audio/transcriptions`; inline batching (up to 10k requests, no file upload); `reasoning_effort` as simple `high`/`none` toggle |
| **xAI Grok** | `grok-4.20-multi-agent` model (routes across cooperating agents; `reasoning.effort` controls agent count: 4 or 16); encrypted reasoning content via `include: ["reasoning.encrypted_content"]` (opaque blob, re-injectable); summarized reasoning content streamed (`response.reasoning_text.delta`, `response.reasoning_summary_text.delta`); context compaction (`POST /v1/responses/compact` → opaque `compaction` item with `encrypted_content`, `dropped_message_count`); `prompt_cache_key` for sticky routing; `cost_in_usd_ticks` / `cost_in_nano_usd` billing; `server_side_tool_usage_details` (per-tool call counts); `deferred: true` for async completions (Chat); `min_p` sampling parameter; `seed` with `system_fingerprint` monitoring; `logprobs`/`top_logprobs` (not on grok-4.20+); JSON Schema `oneOf` support; regex `pattern` with ECMAScript subset (with semantic differences: `.` matches newlines, `^`/`$` implicit, capturing groups non-capturing); `background` field for OpenResponses compatibility; agentic document search on file attachments; `web_search_options` / `search_parameters` on text endpoints; `grok-4.5` reasoning cannot be disabled |
| **OpenRouter** | Unified gateway + router across 400+ models from 70+ providers; model variants (`:nitro`, `:floor`, `:thinking`, `:online`, `:free`, `:extended`, `:exacto`); latest aliases (`~openai/gpt-latest`); model fallbacks (`models[]` array); provider routing (`provider{}` with `order`, `allow_fallbacks`, `require_parameters`, `data_collection`, `zdr`, `enforce_distillable_text`, `only`, `ignore`, `quantizations`, `sort`, `preferred_min_throughput`, `preferred_max_latency`, `max_price`); router slugs (`openrouter/auto`, `openrouter/free`, `openrouter/fusion`, Pareto Router); session stickiness (`session_id` / `x-session-id`); response caching at gateway level (`X-OpenRouter-Cache` headers, custom TTL); plugins (`web`, `file-parser`, `response-healing`, `context-compression`, `moderation`, `web-fetch`, `fusion`, `auto-router`, `pareto-router`); unified `reasoning` parameter (normalizes OpenAI effort, Anthropic budget_tokens, Google thinkingLevel, Alibaba thinking_budget); `reasoning_details` with three types (summary/encrypted/text) and `format` field (openai-responses-v1, anthropic-claude-v1, google-gemini-v1, etc.); `verbosity` parameter (low/medium/high/xhigh/max); `prediction` (predicted output for latency reduction); `modalities` (text/image/audio output); `repetition_penalty` and `top_a` sampling params; `logit_bias` (token ID → -100..100); `top_logprobs` up to 20; `debug.echo_upstream_body`; `metadata` (max 16 key-value pairs); Auto Exacto (tool-call success rate ordering); `sort.partition` (`model`/`none`); Anthropic Messages API surface (native protocol pass-through); model catalog API with `supported_parameters`, `reasoning` metadata, `benchmarks`, `expiration_date`; preset system (named server-side config); stream cancellation with immediate billing stop (supported providers); SSE comments for keep-alive; generation stats query (`GET /api/v1/generation`); `is_byok` and `cost_details` (upstream cost breakdown); zero-completion insurance; failed/fallback not billed |
| **Azure AI Language** | Shared `:analyze-text` / `jobs` endpoints with `kind` discriminator (minimal API surface); Language Detection (100+ languages, ISO 639-1, script name + ISO 15924 script code, country/region hint); prebuilt NER with `category`/`subcategory` (GA) vs `type`/`tags`/`metadata` (preview) structure; `inclusionList`/`exclusionList` entity filtering; Custom NER (train domain-specific entity extraction, Blob Storage); three PII detection modalities (text, conversation, document) with configurable redaction policies (`CharacterMask`/`NoMask`/`EntityMask`/`SyntheticReplacement`), `confidenceScoreThreshold` with `overrides[]`, `redactionSource` for transcripts, `includeAudioRedaction` with `audioTimings`, blur-based image redaction (document); Text Analytics for Health (biomedical NER + relation extraction + UMLS entity linking + assertion detection with certainty/conditionality/association/temporality + FHIR bundle output + SDOH extraction); Sentiment Analysis & Opinion Mining (document + sentence level, aspect-based with targets/assessments linked via relations, `isNegated`); Entity Linking to Wikipedia (`bingId`, `url`); three summarization genres (extractive with `rankScore`/`sentenceCount`/`sortby`, abstractive with `summaryLength`/`query`, conversation with `summaryAspects`: issue/resolution/chapterTitle/narrative/recap/follow-up tasks); Custom Text Classification (single-label and multi-label); Conversation Language Understanding (CLU: intent prediction + entity extraction, Standard/Advanced training modes, `multilingual` flag, `confidenceThreshold`); Custom Question Answering (CQA: KB of Q&A pairs, layered ranking with Azure AI Search, `query-text` prebuilt mode with `answerSpan`, multi-turn `dialog`/`prompts`, metadata filtering, active learning); Orchestration Workflow (routes utterances to CLU/CQA sub-projects); `stringIndexType` (`Utf16CodeUnit` / `TextElement_V8`); LRO pattern (202 + `operation-location` header → poll until `succeeded`); custom model deployment expiry (~18 months); Docker container deployment (some features); custom project authoring API (import/train/deploy/swap); preview vs GA model versioning; `multilingual` project setting (query in any language regardless of training language); retirement dates (March 2029 for most legacy, Sep 2028 for Entity Linking) |

---

## Language & Multilingual Notes

- **Anthropic Claude** — Strong cross-lingual performance; system prompt is the recommended control for pinning response language. Performance data (zero-shot MMLU, % of English): Spanish ~96–98%, French ~96–98%, Chinese ~94–97%, Japanese ~93–97%, Arabic ~92–97%, Hindi ~92–97%, Bengali ~90–96%, Swahili ~78–91%, Yoruba ~53–80%.
- **Google Gemini** — Native multilingual; supports text, image, video, audio inputs in most world languages.
- **Azure AI Language** — Language Detection supports 100+ languages with ISO 639-1 codes and script detection. Text Analytics for Health supports English, German, French, Italian, Spanish, Portuguese, Hebrew. CLU supports `multilingual` flag for cross-language querying. Conversation PII GA model supports English only (preview adds multi-language).
- **OpenAI** — Vision reduced performance for Japanese/Korean; small text should be enlarged.
- **Mistral** — System message sets language; no special parameters needed.

---

## Deprecation & Migration Reference

| Service / Feature | Provider | Retirement date | Successor |
|-------------------|----------|-----------------|-----------|
| LUIS | Azure | March 31, 2026 | CLU |
| QnA Maker | Azure | October 31, 2025 | CQA |
| Sentiment & Opinion Mining | Azure | March 31, 2029 | Microsoft Foundry generative models |
| Key Phrase Extraction | Azure | March 31, 2029 | Microsoft Foundry generative models |
| Entity Linking | Azure | September 1, 2028 | NER or Foundry models |
| Summarization (all genres) | Azure | March 31, 2029 | Microsoft Foundry generative models |
| Custom Text Classification | Azure | March 31, 2029 | Microsoft Foundry generative models |
| CQA | Azure | March 31, 2029 | Microsoft Foundry generative models |
| Orchestration Workflow | Azure | March 31, 2029 | Microsoft Foundry generative models |
| Assistants API | OpenAI | August 26, 2026 | Responses API |
| Prompts API (`v1/prompts`) | OpenAI | November 30, 2026 | Store prompts in code |
| Mistral native reasoning models (`magistral-*`) | Mistral | Deprecated | `reasoning_effort` on standard models |
| `mistral-moderation-2411` | Mistral | March 31, 2026 | `mistral-moderation-2603` |
| Anthropic `:thinking` variant (OpenRouter) | OpenRouter | Deprecated | `reasoning` parameter |
| Anthropic Opus 4.7 Fast Mode | Anthropic | July 24, 2026 | Opus 4.8 Fast Mode |
| OpenRouter `:online` variant | OpenRouter | Deprecated | `openrouter:web_search` server tool |

---
