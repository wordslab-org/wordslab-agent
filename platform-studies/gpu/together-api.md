# Together AI — On-Demand LLM Inference API Analysis

> **Source:** Documentation pages reachable from [https://docs.together.ai/docs/inference/overview](https://docs.together.ai/docs/inference/overview)
>
> **Scope:** On-demand LLM inference services — both serverless API and GPU machine rental.
>
> **Date of analysis:** 2026-07-14

---

## Table of Contents

1. [Platform Overview & Deployment Modes](#1-platform-overview--deployment-modes)
2. [Authentication & API Keys](#2-authentication--api-keys)
3. [Serverless Inference — Chat Completions](#3-serverless-inference--chat-completions)
4. [Chat Completion Parameters](#4-chat-completion-parameters)
5. [Reasoning Models](#5-reasoning-models)
6. [Structured Outputs](#6-structured-outputs)
7. [Log Probabilities (Logprobs)](#7-log-probabilities-logprobs)
8. [Vision Inference](#8-vision-inference)
9. [Embeddings](#9-embeddings)
10. [Rerank](#10-rerank)
11. [Batch Processing](#11-batch-processing)
12. [OpenAI Compatibility](#12-openai-compatibility)
13. [Dedicated Endpoints (Reserved-Hardware Inference)](#13-dedicated-endpoints-reserved-hardware-inference)
14. [Provisioned Throughput](#14-provisioned-throughput)
15. [GPU Clusters (Raw GPU Rental)](#15-gpu-clusters-raw-gpu-rental)
16. [Dedicated Containers (Managed Container Inference)](#16-dedicated-containers-managed-container-inference)
17. [Serverless Model Catalog](#17-serverless-model-catalog)
18. [Pricing & Billing](#18-pricing--billing)
19. [Rate Limits](#19-rate-limits)
20. [Recommended Models](#20-recommended-models)
21. [API Endpoint Reference Summary](#21-api-endpoint-reference-summary)

---

## 1. Platform Overview & Deployment Modes

**Source:** [Inference Overview](https://docs.together.ai/docs/inference/overview)

Together AI runs inference on 200+ open-source models. All deployment modes share the **same inference API** — you only change the `model` parameter to switch between them.

| Mode | Description | Best for |
|------|-------------|----------|
| **Serverless models** | Shared fleet of popular open models, per-token API, no GPU management | Prototyping, variable-traffic apps |
| **Provisioned throughput** | Reserved capacity for a stock model with SLA (committed throughput/reliability) | Production with stronger guarantees |
| **Dedicated endpoints** | Single model on GPUs reserved for you, billed per minute by hardware | Fine-tuned models, direct hardware control |
| **GPU clusters** | Full Kubernetes/Slurm cluster of GPU nodes you fully control | Training, fine-tuning, custom stacks |
| **Dedicated containers** | Managed deployment platform — you bring a Docker image, Together runs it on GPU infra | Custom model inference without infra management |

### Core API endpoint (shared across serverless, provisioned, and dedicated)

```
POST https://api.together.ai/v1/chat/completions
```

- **Auth:** `Authorization: Bearer $TOGETHER_API_KEY`
- **Content-Type:** `application/json`
- **OpenAI-compatible:** Drop-in replacement for OpenAI clients (change `api_key` + `base_url`).
- **SDKs:** Python (`together`), TypeScript (`together-ai`).

### Model parameter conventions

- Serverless: `model="moonshotai/Kimi-K2.6"` (namespace/name)
- Dedicated endpoint: `model="/Qwen/Qwen3.5-9B-FP8-bb04c904"` (leading slash + endpoint name)
- GPU cluster: not applicable (you run containers directly)

---

## 2. Authentication & API Keys

**Source:** [API Keys & Authentication](https://docs.together.ai/docs/api-keys-authentication)

### Authentication method

Every request must include the API key in the `Authorization` header:

```
Authorization: Bearer $TOGETHER_API_KEY
Content-Type: application/json
```

### Key concepts

- **Project-scoped keys:** A key only accesses resources within its project. Multi-project key scoping is in early access.
- **No per-key usage limits:** Spend limits apply at the organization level, not per-key.
- **Keys are created via the UI** at [API keys settings](https://api.together.ai/settings/projects/~current/api-keys) — no documented programmatic key CRUD API.
- **Key shown only once** at creation — must be saved to a secrets manager immediately.

### Key attributes

| Attribute | Description |
|-----------|-------------|
| Name | Descriptive label (e.g. `prod-inference`, `ci-pipeline`) |
| Expiration date | Optional; set via three-dot menu → "Set expiration" |

### Best practices

- Name keys descriptively for their purpose.
- Set expiration dates for temporary/testing keys.
- Rotate keys regularly; revoke unused ones.
- Never commit keys to source control — use env vars or secrets manager.
- Legacy keys (deprecated) are org-scoped, cannot be scoped to a project, cannot be revoked (only regenerated). Avoid in production.

### Usage tracking

The `api_key_id` field is supported for inference and code interpreter requests, enabling per-key spend tracking in [project cost analytics](https://api.together.ai/settings/projects/~current/cost-analytics).

---

## 3. Serverless Inference — Chat Completions

**Source:** [Chat Overview](https://docs.together.ai/docs/inference/chat/overview)

### Capability

Query chat models with single prompts, multi-turn conversations, and system prompts. Queries are **stateless** — pass full conversation history in `messages` each call.

### API endpoint

```
POST https://api.together.ai/v1/chat/completions
```

SDK method: `client.chat.completions.create(...)`

### Request parameters (core)

| Parameter | Type | Description | Default/Required |
|-----------|------|-------------|-----------------|
| `model` | string | Model name (e.g. `"Qwen/Qwen3.5-9B"`) | Required |
| `messages` | array | Array of message objects, each with `role` and `content` | Required |
| `reasoning` | object | `{"enabled": true/false}` — toggle reasoning for hybrid models | Optional |
| `stream` | boolean | Return response as server-sent events | Default: `false` |

### Message roles

| Role | Purpose |
|------|---------|
| `user` | Message from the end user |
| `assistant` | Model's prior response (carries historical context for multi-turn) |
| `system` | System prompt giving the model context/instructions |
| `developer` | Used by some models (e.g. GPT-OSS) instead of `system` |

### Response structure (non-streaming)

```json
{
  "choices": [
    {
      "message": {
        "role": "assistant",
        "content": "9.9 is bigger than 9.11.",
        "reasoning": "Let me compare 9.11 and 9.9..."
      }
    }
  ]
}
```

Access via `response.choices[0].message.content`.

### Response structure (streaming)

Stream of Server-Sent Events with JSON-encoded chunks:

```json
{
  "choices": [{"index": 0, "delta": {"content": " A"}}],
  "id": "85ffbb8a6d2c4340-EWR",
  "token": {"id": 330, "text": " A", "logprob": 1, "special": false},
  "finish_reason": null,
  "created": 1709700707,
  "object": "chat.completion.chunk"
}
```

- Object type: `"chat.completion.chunk"`
- `delta.content` contains partial content
- `finish_reason` is `null` during streaming, set at end
- Each chunk includes a `token` object with `id`, `text`, `logprob`, `special`
- Guard with `if chunk.choices` since the final chunk may have empty `choices`

### Notable features

- **Streaming:** Set `stream=True`; iterate chunks, access `chunk.choices[0].delta.content`.
- **Async parallel requests (Python):** Use `AsyncTogether` with `asyncio.gather()` to run independent calls in parallel.
- **Reasoning field:** For reasoning models, `reasoning` appears alongside `content` (see [Reasoning Models](#5-reasoning-models)).

---

## 4. Chat Completion Parameters

**Source:** [Chat Parameters](https://docs.together.ai/docs/inference/chat/parameters)

### Quick reference (problem → parameter)

| Problem | Parameter |
|---------|-----------|
| Output cuts off mid-sentence | Increase `max_tokens` |
| Need exactly one token (yes/no, class label) | Set `max_tokens` to `1` |
| Responses feel generic or repetitive | Increase `temperature`, or set `frequency_penalty` to small positive value |
| Output loops on same phrase | Set `repetition_penalty` to ~`1.1` |
| Need same answer every run | Set `seed` and use low `temperature` |
| Need machine-parseable output | Use `response_format` with JSON schema |
| Need token-level confidence scores | Set `logprobs` |

### Length and stopping parameters

| Parameter | Type | Description | Default |
|-----------|------|-------------|---------|
| `max_tokens` | integer | Maximum tokens the model can generate. Shorter = faster but risks truncation. Set to `1` for single-token outputs. | unset (generates until stop condition or context limit) |
| `stop` | string or array | String(s) that tell the model to stop generating when produced. Useful for structured outputs with known boundaries. | unset |

### Sampling parameters

| Parameter | Type | Description | Default |
|-----------|------|-------------|---------|
| `temperature` | decimal | Controls randomness. `0` = deterministic (highest-probability token). Higher = more variety. >1 usually too noisy. Lower for extraction/classification; raise for creative. | Model-specific (often `0.7`) |
| `top_p` | decimal | Nucleus sampling — model samples from smallest set of tokens whose cumulative probability exceeds `top_p`. `0.9` = top 90% probability mass. Softer alternative to `temperature`. | `1.0` (no truncation) |
| `top_k` | integer | Limits sampling to `k` most likely next tokens. `top_k=1` = greedy decoding. Hard cap on candidate set. | `0` or unset (no cap) |
| `repetition_penalty` | decimal | Reduces probability of tokens already in prompt/response. >1.0 discourages repetition; <1.0 encourages it. Use ~`1.1` for loops. | `1.0` |
| `frequency_penalty` | decimal | Penalizes tokens proportionally to frequency of appearance. Range: `-2.0` to `2.0`. Finer-grained than `repetition_penalty`. | `0` |
| `presence_penalty` | decimal | Penalizes tokens that appeared at all, regardless of count. Range: `-2.0` to `2.0`. Pushes toward new topics/vocabulary. | `0` |
| `seed` | integer | Makes sampling deterministic (same seed + prompt + model + params = same response). Best-effort; may not hold across model/backend updates. | unset |

### Response shape parameter

| Parameter | Type | Description | Default |
|-----------|------|-------------|---------|
| `n` | integer | Number of independent completions to generate. Each appears as separate entry in `choices`. Higher values cost more. Use for ranking/self-consistency. | `1` |

### Capability-specific parameters

| Parameter | Description | Reference |
|-----------|-------------|-----------|
| `response_format` | Constrain output to JSON or regex pattern | [Structured Outputs](#6-structured-outputs) |
| `tools` and `tool_choice` | Let the model call functions you define, with control over whether/which tool it picks | Function calling |
| `logprobs` | Return per-token log probabilities | [Logprobs](#7-log-probabilities-logprobs) |
| `stream` | Receive response as server-sent events | [Chat Overview](#3-serverless-inference--chat-completions) |

### Notable behaviors

- Tune **one sampling parameter at a time**; don't stack `temperature`/`top_p`/`top_k` together, or `repetition_penalty`/`frequency_penalty`/`presence_penalty` together.
- `seed` determinism is best-effort, may not hold across model/backend updates.
- Model defaults are published in `generation_config.json` on Hugging Face; if a parameter isn't defined there, the inference engine's own default applies.

---

## 5. Reasoning Models

**Source:** [Reasoning](https://docs.together.ai/docs/inference/chat/reasoning)

### Capability

Reasoning models think step-by-step before answering. They produce a chain of thought (visible as tokens in the `reasoning` output field), then output a final answer in `content`.

### API endpoint

Same as chat completions: `POST https://api.together.ai/v1/chat/completions` (model-dependent behavior).

### Supported models and types

| Model | API string | Type | Context length |
|-------|-----------|------|----------------|
| MiniMax M2.7 | `MiniMaxAI/MiniMax-M2.7` | Reasoning only | 202K |
| DeepSeek-V4-Pro | `deepseek-ai/DeepSeek-V4-Pro` | Hybrid (on by default) | 512K |
| GLM-5 | `zai-org/GLM-5` | Hybrid (on by default) | 200K |
| Kimi K2.6 | `moonshotai/Kimi-K2.6` | Hybrid (on by default) | 262K |
| Qwen3.6 Plus | `Qwen/Qwen3.6-Plus` | Hybrid (on by default) | 1M |
| Qwen3.5 9B | `Qwen/Qwen3.5-9B` | Hybrid (on by default) | 262K |
| Cogito v2.1 671B | `deepcogito/cogito-v2-1-671b` | Hybrid (on by default) | 164K |
| Nemotron 3 Ultra 550B A55B | `nvidia/nemotron-3-ultra-550b-a55b` | Hybrid (on by default) | 512K |
| GPT-OSS 120B | `openai/gpt-oss-120b` | Adjustable effort | 128K |
| GPT-OSS 20B | `openai/gpt-oss-20b` | Adjustable effort | 128K |

### Reasoning behavioral types

| Type | Description |
|------|-------------|
| **Reasoning only** | Always produces reasoning tokens, cannot be toggled off |
| **Hybrid** | Supports both reasoning and non-reasoning modes via `reasoning={"enabled": True/False}` |
| **Adjustable effort** | Supports `reasoning_effort` parameter to control reasoning depth |

### Request parameters (reasoning-specific)

| Parameter | Type | Description | Default |
|-----------|------|-------------|---------|
| `reasoning` | object | `{"enabled": true/false}` — toggle reasoning on/off for hybrid models. Disable for simple tasks to save cost/latency. | Model-dependent (on by default for hybrid) |
| `reasoning_effort` | string | `"low"`, `"medium"`, or `"high"` — controls reasoning depth (GPT-OSS models). | `"medium"` (recommended) |
| `chat_template_kwargs` | object | Alternative way to enable/disable: `{"thinking": true}` or `{"enable_thinking": true}`. Also `{"clear_thinking": false}` for preserved thinking, `{"medium_effort": true}` for Nemotron. | — |

#### Reasoning effort by model

| Model | Accepted values | Notes |
|-------|-----------------|-------|
| GPT-OSS | `"low"`, `"medium"`, `"high"` | `"high"`: set `max_tokens` ~30,000 |
| DeepSeek-V4-Pro | `"high"`, `"max"` only | `"low"`/`"medium"` → `"high"`; `"high"`/`"xhigh"` → `"max"` |
| Nemotron 3 Ultra | defaults to high | Use `chat_template_kwargs={"medium_effort": True}` for medium |

### Response structure

**Separate `reasoning` field** (most models — Kimi K2.6, GLM-5, DeepSeek-V4-Pro, GPT-OSS):

```json
{
  "choices": [
    {
      "message": {
        "role": "assistant",
        "content": "9.9 is bigger than 9.11.",
        "reasoning": "Let me compare 9.11 and 9.9..."
      }
    }
  ]
}
```

- In streaming: access via `chunk.choices[0].delta.reasoning` and `chunk.choices[0].delta.content`.
- The `reasoning` key is used for both input (passing back prior turns) and output.
- Older `reasoning_content` key still accepted on input for backward compatibility.

**`<think>` tags in content** (DeepSeek-R1):

```xml
<think>Let me compare 9.11 and 9.9 by looking at their decimal parts...</think>

**Answer:** 9.9 is bigger.
```

- Reasoning embedded in `content` field using `<think>` tags.
- Extract with regex: `re.search(r"<think>(.*?)</think>", content, re.DOTALL)`.

### Thinking modes (GLM-5)

| Mode | Description | How to enable |
|------|-------------|---------------|
| **Interleaved thinking** (default) | Model reasons between tool calls and after receiving tool results, interpreting each output before deciding next step. | Default |
| **Preserved thinking** | Model retains reasoning content from previous turns, improving continuity and cache hit rates. Ideal for coding agents/multi-turn agentic workflows. | `chat_template_kwargs={"clear_thinking": false}`. Must include unmodified `reasoning` from previous turns back in conversation. |
| **Turn-level thinking** | Enable/disable reasoning per-turn within same session. Enable for hard turns (planning, debugging), disable for simple ones. | Toggle `reasoning={"enabled": ...}` per request |

### Reasoning token billing

Reasoning tokens are billed as **completion tokens**, reported under `usage.completion_tokens_details.reasoning_tokens`.

### Recommended temperatures by model

| Model | Recommended temperature |
|-------|--------------------------|
| DeepSeek-R1 | 0.6 |
| Kimi K2.6 (thinking) / GLM-5 | 1.0 |
| GPT-OSS | 1.0 |
| Kimi K2.6 (instant) | 0.6 |

### System prompt guidance by model

| Model | System prompt guidance |
|-------|----------------------|
| DeepSeek-R1 | Omit system prompts |
| Kimi | Use `"You are Kimi, an AI assistant created by Moonshot AI."` |
| GPT-OSS | Use `developer` role instead of `system` |

### Prompting best practices

| Tip | Details |
|-----|---------|
| Use the right temperature | Model-specific (see above) |
| System prompts vary by model | See table above |
| Don't add chain-of-thought instructions | These models already reason; "think step by step" can hurt |
| Avoid few-shot examples | Can degrade performance; describe task and output format instead |
| Think in goals, not steps | Provide high-level objectives, let model determine methodology |
| Structure your prompt | Use XML tags, markdown, labeled sections |
| Set generous `max_tokens` | Reasoning tokens can be tens of thousands |

### When NOT to use reasoning

- Latency is critical (real-time voice, instant chatbots)
- Tasks are straightforward (classification, basic generation, factual lookups)
- Cost is priority (high-volume simple queries — reasoning tokens increase per-query costs)

### Cost/latency management strategies

- Count reasoning tokens (`usage.completion_tokens_details.reasoning_tokens`)
- Use `max_tokens` to cap total output
- Toggle reasoning on hybrid models for simple queries
- Use reasoning effort levels (GPT-OSS: `low` for routine, `high` for critical)
- Use turn-level thinking (GLM-5)
- Prompt for shorter reasoning
- Stream responses (reasoning models produce longer outputs)

---

## 6. Structured Outputs

**Source:** [Structured Outputs](https://docs.together.ai/docs/inference/chat/structured-outputs)

### Capability

Use JSON mode (and regex mode) to get structured outputs from supported chat models. Models return JSON conforming to a schema you supply, so you can parse output directly without retries or fragile parsing.

### API endpoint

```
POST https://api.together.ai/v1/chat/completions
```

### Request parameter: `response_format`

The `response_format` key accepts two types:

#### JSON Schema mode (`type: "json_schema"`)

```json
{
  "response_format": {
    "type": "json_schema",
    "json_schema": {
      "name": "voice_note",
      "schema": {
        "properties": {
          "title": {"description": "...", "title": "Title", "type": "string"},
          "summary": {"description": "...", "title": "Summary", "type": "string"},
          "actionItems": {"description": "...", "items": {"type": "string"}, "title": "Actionitems", "type": "array"}
        },
        "required": ["title", "summary", "actionItems"],
        "title": "VoiceNote",
        "type": "object"
      }
    }
  }
}
```

| Field | Type | Description |
|-------|------|-------------|
| `response_format.type` | string | `"json_schema"` or `"regex"` |
| `response_format.json_schema.name` | string | Schema name |
| `response_format.json_schema.schema` | object | JSON Schema object (properties, required, type, etc.) |

#### Regex mode (`type: "regex"`)

```json
{
  "response_format": {
    "type": "regex",
    "pattern": "(positive|neutral|negative)"
  }
}
```

| Field | Type | Description |
|-------|------|-------------|
| `response_format.type` | string | `"regex"` |
| `response_format.pattern` | string | Regex pattern to constrain output (e.g. `"(positive|neutral|negative)"`, `"\\w+@\\w+\\.com\\n"`) |

### Response structure

The model responds with output matching the schema in `response.choices[0].message.content` (JSON string to parse):

```json
{
  "title": "Morning Routine",
  "summary": "Starting the day with a quick breakfast and checking emails",
  "actionItems": ["Cook scrambled eggs and toast", "Brew a cup of coffee", "Check emails for urgent messages"]
}
```

### Notable features and behaviors

- **Always instruct the model in the prompt** to respond "only in JSON" and include a plain-text copy of the schema. Send this instruction *in addition to* passing `response_format`. The combination of explicit direction + schema text + `response_format` produces consistent valid JSON.
- **Helper libraries:** Pydantic (Python) for schema definition; Zod (TypeScript) with `z.toJSONSchema()`.
- **Regex mode:** Every model that supports JSON mode also supports regex mode. Useful for classification to one of N labels.
- **Works with reasoning models:** Model produces chain of thought in `reasoning`, then writes structured answer to `content`.
- **Works with vision models:** Extract typed data from images.
- **Schema in prompt:** Include the schema string in the system prompt or user message.

### Troubleshooting

| Issue | Cause | Fix |
|-------|-------|-----|
| JSON cut off / incomplete | Token limits — model runs out mid-structure | Raise `max_tokens` or simplify schema; watch for `finish_reason: "length"` |
| Stray characters / parse errors | Malformed example JSON in prompt — model copies example exactly | Validate any JSON embedded in prompts before using |

---

## 7. Log Probabilities (Logprobs)

**Source:** [Logprobs](https://docs.together.ai/docs/inference/chat/logprobs)

### Capability

Return per-token log probabilities to measure model confidence, route low-confidence outputs to a stronger model, and compare alternatives the model considered. Applications: classification, autocomplete ranking, retrieval evaluation, content moderation.

### API endpoint

```
POST https://api.together.ai/v1/chat/completions
```

### Request parameter

| Parameter | Type | Description | Default |
|-----------|------|-------------|---------|
| `logprobs` | integer | Set to `1` to enable logprobs in response. | unset (disabled) |

### Response structure

The response includes a `logprobs` object on each choice. Its `content` field is a list with one entry per output token:

```json
{
  "content": [
    {
      "token": "New",
      "bytes": [78, 101, 119],
      "logprob": -0.39648438,
      "top_logprobs": [
        {"token": "New", "bytes": [78, 101, 119], "logprob": -0.39648438}
      ]
    },
    {
      "token": " York",
      "bytes": [32, 89, 111, 114, 107],
      "logprob": -2.026558e-6,
      "top_logprobs": [
        {"token": " York", "bytes": [32, 89, 111, 114, 107], "logprob": -2.026558e-6}
      ]
    }
  ]
}
```

| Field | Type | Description |
|-------|------|-------------|
| `logprobs.content` | array | One entry per output token |
| `.token` | string | The chosen token text |
| `.bytes` | array of integers | Raw bytes of the token |
| `.logprob` | float | Natural log of probability (negative; closer to 0 = higher confidence) |
| `.top_logprobs` | array | Alternatives the model considered, each with `token`, `bytes`, `logprob` |

### Converting logprobs to probabilities

```python
import math
probability = math.exp(logprob)
# e.g. math.exp(-0.39648438) = 0.6726 → 67% confident
```

- Logprobs are negative (natural logs of probabilities between 0 and 1).
- Closer to 0 = higher confidence; more negative = lower confidence.

### Confidence-based routing pattern

Run a fast/cheap model first, check confidence via `completion.choices[0].logprobs.content[0]["logprob"]`, and escalate to a larger model if confidence falls below a threshold (e.g. 0.85).

### When NOT to use logprobs

- **Open-ended generation:** Logprobs measure token-level certainty, not correctness. A confident wrong answer is still wrong.
- **Long outputs:** First few tokens dominate classification/routing decisions; logprobs deeper in long responses are noisier and less actionable.
- **Cross-model comparison:** Logprob magnitudes aren't directly comparable across model families.

---

## 8. Vision Inference

**Source:** [Vision Overview](https://docs.together.ai/docs/inference/vision/overview), [Vision Inputs](https://docs.together.ai/docs/inference/vision/inputs)

### Capability

Vision-language models (VLMs) accept images alongside text and reply in natural language, structured JSON, or tool calls. Vision models bill images as **input tokens**.

### API endpoint

```
POST https://api.together.ai/v1/chat/completions
```

### Request structure

A `messages` array where the user `content` is a **list of content blocks** (multimodal prompt):

```json
{
  "model": "moonshotai/Kimi-K2.6",
  "max_tokens": 2048,
  "messages": [
    {
      "role": "user",
      "content": [
        {"type": "text", "text": "<prompt>"},
        {"type": "image_url", "image_url": {"url": "https://.../trello-board.png"}}
      ]
    }
  ],
  "stream": true
}
```

### Content block types

| Block type | Field | Value |
|------------|-------|-------|
| `text` | `text` | string prompt |
| `image_url` | `image_url.url` | remote URL **or** base64 data URI (`data:image/jpeg;base64,<base64_data>`) |
| `video_url` | `video_url.url` | video URL (**dedicated endpoints only**) |

### Main request parameters

| Parameter | Type | Required | Description |
|-----------|------|----------|-------------|
| `model` | string | Yes | Vision-capable model ID (e.g. `moonshotai/Kimi-K2.6`) |
| `messages` | array | Yes | Array of message objects; user message `content` is an array of content blocks |
| `messages[].content[]` | array | Yes | List of content blocks: `{"type":"text","text":...}` and `{"type":"image_url","image_url":{"url":...}}` |
| `max_tokens` | integer | No | Max output tokens |
| `temperature` | number | No | Sampling temperature |
| `stream` | boolean | No (default false) | Stream token-by-token via SSE |

### Image token pricing model

Images bill as **input tokens**, broken into a tile grid capped at 2×2 of 560-pixel tiles, **1,601 tokens per tile**:

| Image size (W × H) | Tile grid | Image tokens |
|---------------------|-----------|--------------|
| Up to 559 × 559 | 1 × 1 | 1,601 |
| Up to 559 tall, wider than 560 | 1 × 2 | 3,202 |
| Taller than 560, up to 559 wide | 2 × 1 | 3,202 |
| Wider than 560 **and** taller than 560 | 2 × 2 | 6,404 |

Token formula:

```
image_tokens = min(2, max(width // 560, 1)) * min(2, max(height // 560, 1)) * 1601
```

Image tokens are added to the prompt's text tokens; output tokens billed separately at the model's standard rate.

### Input modes

| Mode | Description | Availability |
|------|-------------|--------------|
| **Remote URL** | Pass image URL in `image_url.url` | All vision models |
| **Local file (base64)** | Encode to base64, pass as data URI `data:image/jpeg;base64,<base64_data>` | All vision models |
| **Video input** | Pass `video_url` content block | Select VLMs on **dedicated endpoints only** |
| **Multiple images** | Pass multiple `image_url` content blocks in a single user message | All vision models |

### Response structure

- Streaming: `choices[].delta.content` chunks (guard with `if chunk.choices` since the final chunk may have empty `choices`).
- Non-streaming: `choices[0].message.content`.

---

## 9. Embeddings

**Source:** [Embeddings](https://docs.together.ai/docs/inference/embeddings/embeddings)

### Capability

Turn text into vector embeddings for search, classification, recommendations, and retrieval-augmented generation (RAG). Compare two vectors to measure how closely related source texts are. Store embeddings in a vector database for long-term retrieval.

### API endpoint

```
POST https://api.together.ai/v1/embeddings
```

SDK method: `client.embeddings.create(...)`

### Request parameters

| Parameter | Type | Description | Required |
|-----------|------|-------------|----------|
| `model` | string | Embedding model name (e.g. `"intfloat/multilingual-e5-large-instruct"`) | Required |
| `input` | string or array of strings | Text to embed. Single string or array of strings for multiple embeddings in one call. | Required |

### Response structure (single input)

```json
{
  "model": "intfloat/multilingual-e5-large-instruct",
  "object": "list",
  "data": [
    {
      "index": 0,
      "object": "embedding",
      "embedding": [0.2633975, 0.13856208, 0.04331574]
    }
  ]
}
```

### Response structure (multiple inputs)

`response.data` contains one object per input, each with matching `index`:

```json
{
  "model": "intfloat/multilingual-e5-large-instruct",
  "object": "list",
  "data": [
    {"index": 0, "object": "embedding", "embedding": [0.2633975, 0.13856208, 0.04331574]},
    {"index": 1, "object": "embedding", "embedding": [-0.14496337, 0.21044481, -0.16187587]}
  ]
}
```

| Field | Type | Description |
|-------|------|-------------|
| `model` | string | Model used |
| `object` | string | `"list"` |
| `data` | array | One object per input |
| `data[].index` | integer | Index matching input position |
| `data[].object` | string | `"embedding"` |
| `data[].embedding` | array of floats | The embedding vector |

### Notable features

- Multiple embeddings in one call: pass array of strings to `input`.
- Available on serverless and dedicated endpoints.
- Works with Vercel AI SDK.

---

## 10. Rerank

**Source:** [Rerank](https://docs.together.ai/docs/inference/embeddings/rerank)

### Capability

Reorder retrieved documents by relevance to a query for sharper search and RAG results. A reranker takes a `query` and a set of text inputs (`documents`) and returns a relevancy score for each document. In RAG pipelines, reranking sits between initial retrieval and final generation as a quality filter.

### Key features

- Long **8K context per document**.
- Low latency for fast search queries.
- Returns relevancy score and ordering index for each document.
- Can filter to top `n` most relevant documents.

### Deployment note

Rerank models (e.g. `mxbai-rerank-large-v2`) are **only available on dedicated endpoints** (not serverless).

### API endpoint

```
POST https://api.together.ai/v1/rerank
```

SDK method: `client.rerank.create(...)`

### Request parameters

| Parameter | Type | Description | Default/Required |
|-----------|------|-------------|------------------|
| `model` | string | Rerank model name (e.g. `"mixedbread-ai/mxbai-rerank-large-v2"`, `"Salesforce/Llama-Rank-V1"`) | Required |
| `query` | string | The query to rank documents against | Required |
| `documents` | array of strings OR array of objects | Documents to rank. Strings for most models; JSON objects for `Salesforce/Llama-Rank-V1`. | Required |
| `top_n` | integer | Filter response to top `n` most relevant documents | Optional |
| `return_documents` | boolean | Include each document in the response alongside rankings | Optional (default: false) |
| `rank_fields` | array of strings | (Dedicated endpoints / `Salesforce/Llama-Rank-V1` only) Which JSON object keys to rank over and the order to consider them in. | Defaults to `["text"]` |

### Two document formats

**Text format** (all rerank endpoints):
- `documents` is a list of strings.
- Works with `mixedbread-ai/mxbai-rerank-large-v2` and others.

**JSON data format** (dedicated endpoints + `Salesforce/Llama-Rank-V1` only):
- `documents` is a list of objects with keys like `from`, `to`, `date`, `subject`, `text`.
- `rank_fields` names which keys to rank over and the order (e.g. `["from", "to", "date", "subject", "text"]`).
- If `rank_fields` not passed, defaults to `text` key.
- Set `return_documents=True` to include documents in response.

### Response structure

```json
{
  "model": "Salesforce/Llama-Rank-V1",
  "choices": [
    {
      "index": 0,
      "document": {
        "text": "{\"from\":\"Paul Doe...\", \"text\":\"We are happy to give you the following pricing...\"}"
      },
      "relevance_score": 0.606349439153678
    },
    {
      "index": 5,
      "document": {
        "text": "{\"from\":\"Paul Doe...\", \"subject\":\"Price Adjustment\", ...}"
      },
      "relevance_score": 0.5059948716207964
    }
  ]
}
```

| Field | Type | Description |
|-------|------|-------------|
| `model` | string | Model used |
| `choices` | array | Ranked results (sorted by relevance, most relevant first) |
| `choices[].index` | integer | Original index of the document in the input `documents` array |
| `choices[].document` | object | (Only if `return_documents=true`) The document content, with `text` field |
| `choices[].relevance_score` | float | Relevance score (higher = more relevant) |

### Notable behaviors

- Results are sorted by relevance score (most relevant first).
- `index` refers to the original position in the input `documents` array.
- `top_n` limits number of returned results.
- The `Salesforce/Llama-Rank-V1` model is the only one supporting the JSON object format with `rank_fields`; all other rerank endpoints accept documents as a list of strings only.

---

## 11. Batch Processing

**Source:** [Batch Overview](https://docs.together.ai/docs/inference/batch/overview), [Batch Manage](https://docs.together.ai/docs/inference/batch/manage)

### Capability

Run asynchronous batch workloads from a single uploaded **JSONL** file for up to **50% lower cost** vs. serverless rates, with a **separate rate-limit pool**, in exchange for a job-shaped (not request-shaped) workflow.

### When to use

When latency is not the primary concern — classifying large datasets, evaluations, synthetic data generation, offline summarizations. 24-hour completion window is a **maximum**, not typical wait. Small batches (<1,000 requests) typically finish in minutes.

### Limits

| Limit | Value |
|-------|-------|
| Requests per batch | up to 50,000 |
| Input file size | up to 100 MB |
| Tokens enqueued per model | up to 30B at any time |
| Completion window | defaults to `24h`, cannot be changed; best-effort target |

### Supported endpoints within batches

| Endpoint | Use |
|----------|-----|
| `/v1/chat/completions` | Most serverless models |
| `/v1/audio/transcriptions` | Audio models like `openai/whisper-large-v3` |
| `/v1/audio/translations` | Audio translation models |
| Dedicated endpoints | Supported, but discount does **not** apply |

### Batch management API endpoints

| Operation | HTTP Method & Path | SDK Method |
|-----------|-------------------|------------|
| Check batch status | `GET /v1/batches/{batch_id}` | `client.batches.retrieve(id)` |
| Cancel a batch | `POST /v1/batches/{batch_id}/cancel` | `client.batches.cancel(id)` |
| List batches | `GET /v1/batches` | `client.batches.list()` |
| Download output/error file | Files API | `client.files.content(id)` |

### Batch status values

| Status | Description |
|--------|-------------|
| `VALIDATING` | Input file being validated before batch begins |
| `IN_PROGRESS` | Requests being processed |
| `COMPLETED` | All requests processed; results available |
| `FAILED` | Processing failed |
| `EXPIRED` | Job exceeded its time limit |
| `CANCELLED` | Job was cancelled |

### Batch object fields

| Field | Description |
|-------|-------------|
| `id` | batch ID (e.g. `batch-xyz789`) |
| `status` | one of the statuses above |
| `progress` | progress indicator |
| `output_file_id` | file ID for successful results (JSONL) |
| `error_file_id` | file ID for per-request failures (JSONL) |

### Billing

- Billed per **successful response** in the output file. Failed requests in the error file aren't billed.
- Cancelling doesn't refund successful responses generated before the cancel landed.
- Discounted models (50% off batch rates) include select Llama, Qwen, Mixtral, GLM, and Whisper models.
- Some models are NOT available for batch (DeepSeek-R1, DeepSeek-V3.1, DeepSeek-V4-Pro, MiniMax-M2.7, Kimi-K2.5, Kimi-K2.6).

### Best practices

- Aim for **1,000 to 10,000 requests per batch** (smaller wastes per-job overhead; larger risks the 50,000 cap).
- Keep `custom_id` values stable and meaningful (treat as join key between input and output).
- Results returned in **arbitrary order** — use `custom_id` to reconcile.
- A single uploaded file can back **multiple batch jobs** without re-uploading.
- For classification/labeling: set `max_tokens` to 4 and `temperature` to 0.
- Validate JSONL locally before uploading (malformed input fails the entire batch in `VALIDATING`).
- Poll every 30 to 60 seconds (tighter loops hit rate limits).
- Always inspect the error file — per-request errors don't change overall batch status even when `COMPLETED`.

### Error file format

Each line pairs `custom_id` with the failure reason:

```json
{"custom_id": "req-1", "error": {"message": "Invalid model specified", "code": "invalid_model"}}
{"custom_id": "req-5", "error": {"message": "Request timeout", "code": "timeout"}}
```

### Error codes

| Code | Description | Solution |
|------|-------------|----------|
| 400 | Invalid request format | Check JSONL syntax and required fields |
| 401 | Authentication failed | Verify your API key |
| 404 | Batch not found | Check the batch ID |
| 429 | Rate limit exceeded | Reduce request frequency |
| 500 | Server error | Retry with exponential backoff |

---

## 12. OpenAI Compatibility

**Source:** [OpenAI Compatibility](https://docs.together.ai/docs/inference/openai-compatibility)

### Capability

Together's API is compatible with the OpenAI REST API and SDKs. Point an existing OpenAI client at Together with **two changes**: the API key and the base URL.

### Drop-in client setup

```python
# Python
openai.OpenAI(api_key=os.environ["TOGETHER_API_KEY"], base_url="https://api.together.ai/v1")
```

```typescript
// TypeScript
new OpenAI({ apiKey: process.env.TOGETHER_API_KEY, baseURL: "https://api.together.ai/v1" })
```

### Endpoint compatibility matrix

| OpenAI SDK call | Together endpoint | Status |
|----------------|-------------------|--------|
| `chat.completions.create` | `POST /v1/chat/completions` | Supported |
| `chat.completions.create` (vision input) | `POST /v1/chat/completions` | Supported |
| `chat.completions.create` (tools) | `POST /v1/chat/completions` | Supported |
| `chat.completions.create` (`response_format`) | `POST /v1/chat/completions` | Supported |
| `completions.create` | `POST /v1/completions` | Supported (legacy text completions) |
| `embeddings.create` | `POST /v1/embeddings` | Supported |
| `images.generate` | `POST /v1/images/generations` | Supported |
| `audio.speech.create` | `POST /v1/audio/speech` | Supported |
| `audio.transcriptions.create` | `POST /v1/audio/transcriptions` | Supported |
| `audio.translations.create` | `POST /v1/audio/translations` | Supported |
| `models.list`, `models.retrieve` | `GET /v1/models` | Supported |
| `assistants.*`, `threads.*`, `runs.*` | n/a | **Not supported** |
| `fine_tuning.jobs.*` (OpenAI shape) | n/a | **Not supported** (use Together-native fine-tuning) |
| `files.*` (OpenAI shape) | n/a | **Partial** (Together has its own Files API) |
| `batches.*` (OpenAI shape) | n/a | **Not supported** (use Together-native Batch API) |
| `moderations.create` | n/a | **Not supported** (use Llama Guard via chat completions) |

### Together-native endpoints NOT exposed by OpenAI SDKs

- Video generation
- Image edits and inpainting beyond `images.generate`
- Reasoning controls and `reasoning_content`
- Logprobs surface (returns Together's own richer shape)
- Batch API and Files API (use Together-native equivalents)

### Known incompatibilities and quirks

| Aspect | Details |
|--------|---------|
| **Model identifiers** | Together model IDs are namespaced (e.g. `openai/gpt-oss-20b`). OpenAI strings like `gpt-4o` return 404. |
| **`logprobs`** | Returns Together's own richer shape, not OpenAI's. |
| **`seed`** | Best-effort; determinism not guaranteed across replicas/versions. |
| **`n`** | Supported on most chat models but not every model — loop client-side if rejected. |
| **`logit_bias`** | Not supported on most models. |
| **`service_tier`, `store`, `metadata`, `prediction`** | Accepted but ignored. |
| **`reasoning_effort`** | Works on GPT-OSS models. Together's `reasoning={"enabled":...}` toggle is not part of OpenAI's API surface. |
| **`detail`** (vision) | Accepted but ignored. |

### Response shape differences

- `usage` always includes `prompt_tokens`, `completion_tokens`, `total_tokens`.
- Extra token counts vary in **location** by model — read both shapes defensively:
  - Reasoning models nest OpenAI-style: `usage.prompt_tokens_details.cached_tokens`, `usage.completion_tokens_details.reasoning_tokens`.
  - Some non-reasoning models return `cached_tokens` flat at top level of `usage`.
  - Defensive fallback: `(usage.prompt_tokens_details or {}).get("cached_tokens") or usage.get("cached_tokens", 0)`.
- Reasoning models return chain of thought in `reasoning` field on the assistant message (not OpenAI's nested `reasoning` object). Pass it back under same `reasoning` key for preserved thinking/multi-turn.
- `id` and `system_fingerprint` are present but use Together's formats.
- Errors: Together returns OpenAI-shaped error objects `{"error": {"message", "type", "code"}}`, but `type`/`code` values are Together's. Match on **HTTP status** for portable handling.

---

## 13. Dedicated Endpoints (Reserved-Hardware Inference)

**Sources:** [Overview](https://docs.together.ai/docs/dedicated-endpoints/overview), [Scaling](https://docs.together.ai/docs/dedicated-endpoints/scaling), [Settings](https://docs.together.ai/docs/dedicated-endpoints/settings), [Manage](https://docs.together.ai/docs/dedicated-endpoints/manage)

### Capability

A dedicated endpoint serves a **single model** on hardware reserved only for you, offering **predictable latency** and **no shared-fleet rate limits**. Highly configurable: upload custom fine-tuned models, configure autoscaling and decoding optimizations. Dedicated endpoints use the **same inference APIs** as serverless — swap only the `model` parameter.

### Key concepts

- **Per-minute billing by hardware** while the endpoint is running, regardless of model or request volume.
- **Autoscaling**: configure min/max replica count; platform autoscales between bounds based on demand.
- **Multi-GPU per replica**: deploy 2, 4, or 8 GPUs per replica for higher throughput / lower latency.
- Billing starts when the endpoint is running. Provisioning, queuing, and failed deployments are **not** billed.

### Hardware pricing (single-GPU cost/hour)

| GPU | Hardware ID | Cost/hour |
|-----|-------------|-----------|
| H100 80GB SXM | `1x_nvidia_h100_80gb_sxm` | $5.40 |
| H200 140GB SXM | `1x_nvidia_h200_140gb_sxm` | $6.60 |
| B200 180GB SXM | `1x_nvidia_b200_180gb_sxm` | $9.00 |

### Multi-GPU hardware ID format

```
<count>x_nvidia_<model>_<memory>_<form_factor>
```

e.g., `4x_nvidia_h100_80gb_sxm` = four H100s. Cost scales with GPU count.

### Endpoint management API/CLI/SDK operations

| Operation | CLI Command | Python SDK | TypeScript SDK |
|-----------|-------------|------------|---------------|
| List hardware options | `together endpoints hardware --model <MODEL_ID>` | `client.endpoints.list_hardware(model=...)` | `client.endpoints.listHardware({model: ...})` |
| List availability zones | `together endpoints availability-zones` | `client.endpoints.list_avzones()` | `client.endpoints.listAvzones()` |
| Create endpoint | `together endpoints create ...` | `client.endpoints.create(...)` | `client.endpoints.create(...)` |
| Inspect endpoint | `together endpoints retrieve <id>` | `client.endpoints.retrieve("id")` | `client.endpoints.retrieve("id")` |
| List endpoints | `together endpoints list` | `client.endpoints.list(mine=True)` | `client.endpoints.list({mine: true})` |
| Start | `together endpoints start <id>` | `client.endpoints.update("id", state="STARTED")` | `client.endpoints.update("id", {state: "STARTED"})` |
| Stop | `together endpoints stop <id>` | `client.endpoints.update("id", state="STOPPED")` | `client.endpoints.update("id", {state: "STOPPED"})` |
| Update settings | `together endpoints update ... <id>` | `client.endpoints.update("id", ...)` | `client.endpoints.update("id", ...)` |
| Delete | `together endpoints delete <id>` | `client.endpoints.delete("id")` | `client.endpoints.delete("id")` |

### Create endpoint — request parameters

| Parameter (CLI flag) | SDK field | Type | Description | Default/Required |
|----------------------|-----------|------|-------------|-----------------|
| `--model` | `model` | string | Model ID to deploy (e.g., `Qwen/Qwen3.5-9B-FP8`) | Required |
| `--hardware` | `hardware` | string | Hardware configuration ID (e.g., `1x_nvidia_h100_80gb_sxm`) | Required |
| `--display-name` | `display_name` | string | Human-readable endpoint name | Required |
| `--min-replicas` | `autoscaling.min_replicas` | int | Minimum replicas kept running | Optional |
| `--max-replicas` | `autoscaling.max_replicas` | int | Maximum replicas to scale up to (cost ceiling) | Optional |
| `--availability-zone` | `availability_zone` | string | Target a specific availability zone | Optional |
| `--inactive-timeout` | — | int (minutes) | Auto-shutdown threshold; `0` disables | Default: 60 |
| `--no-speculative-decoding` | — | bool flag | Disable speculative decoding (enabled by default) | Optional |
| `--wait` | — | bool flag | Wait for endpoint to be ready | Optional |

### Create endpoint — response structure

```json
{
  "object": "endpoint",
  "id": "endpoint-d23901de-ef8f-44bf-b3e7-de9c1ca8f2d7",
  "name": "devuser/Qwen/Qwen3.5-9B-FP8-a32b82a1",
  "display_name": "My endpoint",
  "model": "Qwen/Qwen3.5-9B-FP8",
  "hardware": "1x_nvidia_h100_80gb_sxm",
  "type": "dedicated",
  "owner": "devuser",
  "state": "PENDING",
  "autoscaling": { "min_replicas": 1, "max_replicas": 1 },
  "created_at": "2026-05-04T10:43:55.405Z"
}
```

| Field | Purpose |
|-------|---------|
| `id` | Unique endpoint ID; used for all management operations |
| `name` | Model identifier passed as `model` parameter to inference APIs (username + base model + unique suffix) |
| `state` | Lifecycle state: `PENDING` → `STARTED` (ready for inference) |

### Scaling (vertical vs. horizontal)

| Axis | What it controls | How to set | When to use |
|------|-----------------|------------|-------------|
| **Vertical** | GPUs per replica | Multi-GPU hardware SKU at create time | Compute-bound, memory-intensive, low-latency needs |
| **Horizontal** | Number of replicas | `min_replicas`/`max_replicas` (dynamic, autoscaling) | High concurrency, I/O-bound, fault tolerance |

- Hardware can't be changed on a running endpoint — redeploy for a different SKU.
- Billing is proportional to the number of active replicas.

### Settings

| Setting | Default | Notes |
|---------|---------|-------|
| `--min-replicas` / `--max-replicas` | — | Must be specified together when updating |
| `--inactive-timeout` | 60 minutes | Auto-shutdown after inactivity; `0` disables. Stopped endpoints are not deleted. |
| Speculative decoding | Enabled | `--no-speculative-decoding` to disable |
| Prompt caching | **Always enabled** | Cannot be disabled; `--no-prompt-cache` deprecated and ignored |
| Availability zone | Any | Restricting limits hardware availability |

### Endpoint lifecycle

```
PENDING → STARTED (ready for inference) → STOPPED (paused, not billed) → deleted
```

- An auto-stopped endpoint is **not deleted**; configuration preserved, restart with `together endpoints start`.
- Stopping pauses billing immediately.
- Deletion is permanent.

### List parameters

| Parameter | Values | Description |
|-----------|--------|-------------|
| `mine` | bool | Only endpoints owned by you |
| `type` | `dedicated` (and others) | Filter by endpoint type |
| `usage_type` | `on-demand` (and others) | Filter by usage type |

### Hardware list output fields

| Field | Example |
|-------|---------|
| Hardware ID | `1x_nvidia_h100_80gb_sxm` |
| GPU | `h100` |
| Memory | `80GB` |
| Count | `1` |
| Price (per minute) | `$0.06` |
| Availability | `✓ available` |

### Troubleshooting

| Issue | Cause | Fix |
|-------|-------|-----|
| Low availability | Hardware available but only for partial replica count | Lower the min replica count |
| Hardware unavailable error | Hardware fully claimed | Try different hardware or retry later |
| Model not supported | Not every model is available for dedicated deployment | Fine-tuned models deploy only if base model is supported |

---

## 14. Provisioned Throughput

**Source:** [Provisioned Throughput](https://docs.together.ai/docs/inference/provisioned-throughput)

### Capability

Reserved inference capacity for production workloads. You commit to a number of **provisioned throughput units (PTUs)** for a fixed term of one month or more; Together commits to throughput and reliability targets. Uses the **same inference API** surface as serverless/dedicated.

### Availability

Currently unavailable for self-service — contact sales to request a quote.

### Supported models

| Model | Model string |
|-------|-------------|
| MiniMax M3 | `MiniMaxAI/MiniMax-M3` |
| GLM-5.2 | `zai-org/GLM-5.2` |

### How PTUs work

- A PTU is a fixed slice of guaranteed throughput for a model/family, priced at **$0.05 per PTU per minute**.
- Input tokens, output tokens, and cached reads consume PTUs at different rates (output more expensive than input; cached reads cheaper than fresh inputs; exact ratios model-specific, defined in contract).
- Traffic converts to a single normalized rate. You don't need to forecast a precise traffic mix.
- Output-heavy/cache-light traffic consumes PTUs faster; cache-heavy consumes more slowly.
- Traffic shape changes consumption rate, not the SLA.

### SLA

| Dimension | Commitment |
|-----------|------------|
| **Throughput** | Serve eligible traffic up to max throughput your purchased PTUs entitle you to (tokens per minute) |
| **Reliability** | At least 99% of eligible requests complete successfully each month |

### Overage behavior

- Traffic above purchased PTU capacity may fall back to shared serverless pool **best-effort** when capacity is available, **billed at standard serverless rates**.
- If serverless capacity isn't available, that traffic may be rate-limited or not served.
- Overage traffic is **not covered** by the SLA.
- Unused PTUs don't roll over and aren't refunded/credited.

### Comparison: Serverless vs Provisioned vs Dedicated

| Dimension | Serverless | Provisioned throughput | Dedicated endpoints |
|-----------|------------|------------------------|---------------------|
| What you reserve | Nothing; shared fleet | Committed PT capacity for a selected model/family | GPUs reserved only for you |
| Billing unit | Per token/second/unit of output | Per PTU, fixed term (1-month min) | Per minute of reserved hardware |
| SLA | Best-effort with dynamic rate limits | Defined throughput & reliability targets | No shared-fleet rate limits; shaped by your hardware |
| Best for | Prototyping, variable traffic | Production workloads on stock models | Fine-tuned models, custom hardware, fine-grained control |
| How to access | Self-serve | Contact sales | Self-serve |

---

## 15. GPU Clusters (Raw GPU Rental)

**Sources:** [Overview](https://docs.together.ai/docs/gpu-clusters-overview), [Quickstart](https://docs.together.ai/docs/gpu-clusters-quickstart), [API & Integrations](https://docs.together.ai/docs/gpu-clusters-api), [Management](https://docs.together.ai/docs/gpu-clusters-management)

### Capability

Together GPU Clusters provide **on-demand access to high-performance GPU infrastructure** for training, fine-tuning, and large-scale AI workloads. Clusters provision in minutes and support real-time scaling, persistent storage, and two workload managers: **Kubernetes** or **Slurm-on-Kubernetes**.

### Architecture

- **Control Plane** — Manages cluster state, scheduling, API access
- **Worker Nodes** — GPU-equipped nodes running workloads
- **Networking** — High-speed **InfiniBand** for multi-node communication
- **Storage Layer** — Persistent volumes, local NVMe, shared storage

### Slurm on Kubernetes via Slinky

Together runs Slurm on top of Kubernetes using **Slinky**, an integration layer:
- **Slurm Controller** — Runs as Kubernetes pods, manages job queues/scheduling
- **Login Nodes** — SSH-accessible entry points for job submission
- **Compute Nodes** — GPU workers registered with both Kubernetes and Slurm

### Available hardware

| GPU | Notes |
|-----|-------|
| NVIDIA HGX B200 | Latest generation, maximum performance |
| NVIDIA HGX H200 | Enhanced memory for large models |
| NVIDIA HGX H100 SXM | High-bandwidth training and inference |

All nodes feature InfiniBand for multi-node training, **except inference-optimized variants**.

### Storage tiers

| Tier | Persistence | Description |
|------|-------------|-------------|
| Shared volumes | **Persistent** | High-throughput filesystem; survives pod restarts, node reboots, cluster deletion |
| Local NVMe | **Ephemeral** | Fast local disks per node; data lost during reboots/migrations/recreations |
| `/home` directory | **Persistent on Slurm** (NFS-backed) / **Ephemeral on Kubernetes** (node-local) | — |

Shared volumes are dynamically resizable.

### Capacity types (billing modes)

| Type | Commitment | Payment | Pricing | Use when |
|------|------------|---------|---------|-----------|
| **Reserved** | 1–90 days | Upfront (credits charged at provisioning) | Discounted vs on-demand | Predictable workloads, multi-day training runs, cost optimization |
| **On-demand** | None (terminate anytime) | Hourly | Standard (higher per-hour) | Unpredictable needs, short experiments, exploratory testing |

**Mixing capacity types:** Reserved + on-demand can be combined in the same cluster. Usage beyond reserved capacity is automatically billed at on-demand rates.

**Special note:** Shared volumes attached to a reserved cluster are decoupled from cluster lifecycle — when a reserved cluster is decommissioned, attached shared volumes are not deleted; they auto-move to on-demand pricing and persist.

### Cluster management CLI commands

| Command | Purpose |
|---------|---------|
| `tg beta clusters create ...` | Create a cluster |
| `tg beta clusters delete [CLUSTER_ID]` | Delete a cluster |
| `tg beta clusters list` | List clusters |
| `tg beta clusters update [CLUSTER_ID] --num-gpus N` | Scale a cluster |
| `tg beta clusters get-credentials [CLUSTER_ID] --set-default-context` | Download kubeconfig |

### CLI flags (clusters create)

| Flag | Example | Purpose |
|------|---------|---------|
| `--name` | `my-cluster` | Cluster name |
| `--num-gpus` | `8` | Number of GPUs |
| `--gpu-type` | `H100_SXM` | GPU type identifier |
| `--region` | `us-central-8` | Datacenter region |
| `--billing-type` | `ON_DEMAND` / `RESERVED` | Billing mode |
| `--duration-days` | `30` | Reservation length (Reserved only) |
| `--cluster-type` | `KUBERNETES` / `SLURM` | Workload manager |
| `--non-interactive` | — | Skip prompts for scripts/CI |
| `--json` | — | JSON output (skip prompts) |

### Cluster creation (console configuration options)

| Field | Description | Notes |
|-------|-------------|-------|
| Capacity type | Reserved or On-demand | Drives billing mode |
| Cluster size | Number + type of GPUs (e.g., `8xH100`) | Options: H100, H200, B200 |
| Cluster name | Descriptive identifier | — |
| Cluster type | `Kubernetes` or `Slurm` | Workload manager choice |
| Region | Default **Any region** (Together assigns region with most capacity). Specific region optional. | Changing GPU type resets region to "Any" and clears selected shared volume |
| Duration | 1–90 days | Reserved only |
| Shared volume | Create + name persistent storage | Min size 1 TiB; resizable later |
| NVIDIA driver version | Optional | — |
| CUDA version | Optional | — |

### Kubernetes cluster usage

1. Download kubeconfig: `tg beta clusters get-credentials [CLUSTER_ID] --set-default-context`
2. Verify connectivity: `kubectl get nodes`
3. Deploy workloads via `kubectl apply` or K8s Dashboard

Storage via PersistentVolumeClaim (PVC) → Pod with `volumeMounts`. GPU access via `resources.limits.nvidia.com/gpu: N` in pod spec.

### Slurm cluster usage

1. Add SSH key at api.together.ai/settings/ssh-key (before cluster creation)
2. SSH to login node (always `slurm-login`)
3. Use `sinfo`, `squeue`, `srun`, `sbatch`, `scancel`

### Cluster autoscaling (Kubernetes Cluster Autoscaler)

- **Scales up** when pods pending due to insufficient resources
- **Scales down** when nodes underutilized for extended period
- **Respects constraints** (min/max node counts, resource limits)
- Works with reserved and on-demand capacity
- Enable via UI toggle "Enable Autoscaling" + configure maximum GPUs

### Targeted scale-down

1. **Cordon nodes** to prevent new workloads: `kubectl cordon` (K8s) or `sudo scontrol update NodeName=<name> State=drain` (Slurm)
2. **Trigger scale-down** via UI/CLI/API

### SSH access matrix

| Access method | Kubernetes (SSH to worker node) | Slurm (SSH to compute node) |
|---------------|--------------------------------|------------------------------|
| GPU visibility | All GPUs via `nvidia-smi` | All GPUs via `nvidia-smi` |
| Run GPU workloads | ✗ Not available | ✓ Available |
| Access PersistentVolumes | ✗ Not available | ✓ Available via `/home` |
| Best for | Node-level monitoring/debugging | Monitoring, debugging, direct Slurm workflows |

### Project-based permissions

| Role | Capabilities |
|------|-------------|
| **Admin** | Create/delete clusters, modify config, manage users, use clusters |
| **Editor** | Use clusters (SSH, kubectl, Slurm) but cannot create/delete/modify infrastructure |

### SkyPilot integration

```bash
uv pip install skypilot[kubernetes]
# Download kubeconfig, then:
sky check k8s                    # Verify
sky show-gpus --infra k8s        # Check available GPUs
sky launch -c my-job task.yaml   # Launch job
```

### Automation patterns

- **CI/CD (GitHub Actions):** install Together CLI, `tg beta clusters create ... --non-interactive`, `kubectl apply`, cleanup with `tg beta clusters delete`
- **Scheduled (cron):** daily batch creation with same create command
- **Auto-scaling script:** scale up/down based on job queue length

---

## 16. Dedicated Containers (Managed Container Inference)

**Sources:** [Dedicated Container Inference](https://docs.together.ai/docs/dedicated-container-inference), [Containers Quickstart](https://docs.together.ai/docs/containers-quickstart)

### Capability

Dedicated Containers let you **run your own Dockerized inference workloads** on Together's managed GPU infrastructure. You bring the container; Together handles compute provisioning, autoscaling, networking, and observability.

Available through sales team (contact sales to enable for your org).

### Architecture

1. Build and push a Docker image using the **Jig CLI**
2. Inside your container, the **Sprocket SDK** connects your inference code to Together's managed **job queue**
3. Once deployed, workers receive requests

### Key components

| Component | Description |
|-----------|-------------|
| **Jig CLI** | Build, deploy, secrets, volumes management |
| **Sprocket SDK** | Inference workers with `setup()` and `predict()` methods |
| **Queue API** | Async jobs with priority and progress |
| **REST API** | Deployments, secrets, storage, and queue management |

### Sprocket SDK — worker pattern

```python
import os
import sprocket

class HelloWorld(sprocket.Sprocket):
    def setup(self) -> None:
        self.greeting = "Hello"

    def predict(self, args: dict) -> dict:
        name = args.get("name", "world")
        return {"message": f"{self.greeting}, {name}!"}

if __name__ == "__main__":
    queue_name = os.environ.get("TOGETHER_DEPLOYMENT_NAME", "hello-world")
    sprocket.run(HelloWorld(), queue_name)
```

Key concepts: subclass `sprocket.Sprocket`, implement `setup()` (one-time init) and `predict(args)` (per-request inference), then `sprocket.run(instance, queue_name)`.

### Deployment configuration (`pyproject.toml`)

Deployment name **must be globally unique**.

#### `[tool.jig.image]` — image build config

| Field | Example | Description |
|-------|---------|-------------|
| `python_version` | `"3.11"` | Python version in image |
| `cmd` | `"python3 hello_world.py --queue"` | Container entrypoint command |
| `copy` | `["hello_world.py"]` | Files to copy into image |

#### `[tool.jig.deploy]` — deployment/resource config

| Field | Example | Type/Unit | Description |
|-------|---------|-----------|-------------|
| `gpu_type` | `"none"` | string | GPU type (`"none"` for CPU-only) |
| `gpu_count` | `0` | int | Number of GPUs per replica |
| `cpu` | `1` | int | CPUs per replica |
| `memory` | `2` | int (GB) | Memory per replica |
| `storage` | `10` | int (GB) | Storage per replica |
| `port` | `8000` | int | Container port exposed |
| `min_replicas` | `1` | int | Minimum replicas (autoscaling floor) |
| `max_replicas` | `1` | int | Maximum replicas (autoscaling ceiling) |

### Jig CLI commands

| Command | Purpose |
|---------|---------|
| `tg beta jig deploy` | Build + push image, create deployment |
| `tg beta jig status` | View deployment status |
| `tg beta jig logs --follow` | Stream logs |
| `tg beta jig destroy` | Delete deployment |

### REST API endpoints (Dedicated Containers)

| Method | Endpoint | Purpose |
|--------|----------|---------|
| `GET` | `/v1/deployment-request/{model}/health` | Health check for a deployment |
| `POST` | `/v1/queue/submit` | Submit async inference job to queue |
| `GET` | `/v1/queue/status?model={model}&request_id={request_id}` | Poll job status/result |

### Queue submit — request parameters

```json
{
  "model": "hello-world",
  "payload": {"name": "Together"},
  "priority": 1
}
```

| Field | Type | Required | Description |
|-------|------|----------|-------------|
| `model` | string | Yes | Deployment name (globally unique) |
| `payload` | object | Yes | Input args dict passed to `predict(args)` |
| `priority` | int | Yes | Job priority (higher = processed first) |

**Response:**

```json
{ "request_id": "req_abc123" }
```

Real request IDs use **UUIDv7** format.

### Queue status — query parameters

| Param | Type | Required | Description |
|-------|------|----------|-------------|
| `model` | string | Yes | Deployment name |
| `request_id` | string | Yes | Job ID returned by `/queue/submit` |

**Response (when complete):**

```json
{
  "request_id": "req_abc123",
  "model": "hello-world",
  "status": "done",
  "outputs": {"message": "Hello, Together!"}
}
```

### Health check

```bash
curl https://api.together.ai/v1/deployment-request/hello-world/health \
  -H "Authorization: Bearer $TOGETHER_API_KEY"
```

**Response:**

```json
{"status": "healthy"}
```

---

## 17. Serverless Model Catalog

**Source:** [Serverless Models](https://docs.together.ai/docs/serverless/models)

### Chat models (selected — full catalog in docs)

| Organization | Model name | API model string | Context length | Input $/1M | Cached input $/1M | Output $/1M | Quant. | Func call | Struct out |
|---|---|---|---|---|---|---|---|---|---|
| Minimax | Minimax M3 | `MiniMaxAI/MiniMax-M3` | 524K | $0.30 | $0.06 | $1.20 | FP4 | Yes | Yes |
| Qwen | Qwen3.7 Max | `Qwen/Qwen3.7-Max` | — | $1.25 | — | $3.75 | — | — | — |
| Z.ai | GLM-5.2 | `zai-org/GLM-5.2` | 262K | $1.40 | $0.26 | $4.40 | FP4 | Yes | Yes |
| OpenAI | GPT-OSS 120B | `openai/gpt-oss-120b` | 128K | $0.15 | — | $0.60 | MXFP4 | Yes | Yes |
| OpenAI | GPT-OSS 20B | `openai/gpt-oss-20b` | 128K | $0.05 | — | $0.20 | MXFP4 | Yes | Yes |
| DeepSeek | DeepSeek-V4-Pro | `deepseek-ai/DeepSeek-V4-Pro` | 512K | $1.74 | $0.20 | $3.48 | FP4 | Yes | Yes |
| Meta | Llama 3.3 70B Turbo | `meta-llama/Llama-3.3-70B-Instruct-Turbo` | 131K | $1.04 | — | $1.04 | FP8 | Yes | Yes |
| Qwen | Qwen 2.5 7B Turbo | `Qwen/Qwen2.5-7B-Instruct-Turbo` | 32K | $0.30 | — | $0.30 | FP8 | Yes | Yes |
| Google | Gemma 4 31B IT | `google/gemma-4-31B-it` | 262K | $0.39 | — | $0.97 | FP8 | Yes | Yes |

### Image models (selected — price per megapixel)

| Organization | Model name | Model string | Price/MP | Default steps |
|---|---|---|---|---|
| Google | Flash Image 2.5 | `google/flash-image-2.5` | $0.039 | — |
| Google | Gemini 3 Pro Image | `google/gemini-3-pro-image` | $0.134 | — |
| Black Forest Labs | FLUX.1 schnell (Turbo) | `black-forest-labs/FLUX.1-schnell` | $0.0027 | 4 |
| Black Forest Labs | FLUX.2 [pro] | `black-forest-labs/FLUX.2-pro` | $0.03 | — |
| ByteDance | Seedream 3.0 | `ByteDance-Seed/Seedream-3.0` | $0.018 | — |
| OpenAI | GPT Image 1.5 | `openai/gpt-image-1.5` | $0.034 | — |

**FLUX pricing formula** (excludes pro models):

```
Cost = MP × Price per MP × (Steps ÷ Default Steps)
```

If you use *more* steps than default, cost increases proportionally. If you use *fewer* steps, cost does **not** decrease.

### Vision models (selected)

| Organization | Model name | API model string | Context length | Input $/1M | Output $/1M |
|---|---|---|---|---|---|
| Qwen | Qwen3.5 9B | `Qwen/Qwen3.5-9B` | 262K | $0.17 | $0.25 |
| Google | Gemma 4 31B IT | `google/gemma-4-31B-it` | 262K | $0.39 | $0.97 |
| Minimax | Minimax M3 | `MiniMaxAI/MiniMax-M3` | 524K | $0.30 | $1.20 |
| Moonshot | Kimi K2.6 | `moonshotai/Kimi-K2.6` | 262K | $1.20 | $4.50 |

### Video models (selected — price per video)

| Organization | Model name | Model string | Price/video | Res/duration |
|---|---|---|---|---|
| Google | Veo 3.0 | `google/veo-3.0` | $1.60 | 720p / 8s |
| Google | Veo 3.0 Fast | `google/veo-3.0-fast` | $0.80 | 1080p / 8s |
| ByteDance | Seedance 1.0 Pro | `ByteDance/Seedance-1.0-pro` | $0.57 | 1080p / 5s |
| OpenAI | Sora 2 Pro | `openai/sora-2-pro` | $2.40 | 1080p / 8s |

### Audio models (selected)

| Organization | Modality | Model name | Model string | Pricing |
|---|---|---|---|---|
| Cartesia | TTS | Sonic 3 | `cartesia/sonic-3` | $65.00 per 1M chars |
| Canopy Labs | TTS | Orpheus 3B | `canopylabs/orpheus-3b-0.1-ft` | $15.00 per 1M chars |
| OpenAI | STT | Whisper Large v3 | `openai/whisper-large-v3` | $0.0015 per audio min |

### Embedding models

| Model name | Model string | Model size | Embedding dim | Context window | Pricing $/1M tok |
|---|---|---|---|---|---|
| Multilingual-e5-large-instruct | `intfloat/multilingual-e5-large-instruct` | 560M | 1024 | 514 | $0.02 |

### Rerank models

**None** offered via serverless. Rerank models like `mixedbread-ai/mxbai-rerank-large-v2` are **only available as dedicated endpoints**.

### Moderation models

**None** currently offered via serverless. Use Llama Guard via chat completions.

---

## 18. Pricing & Billing

**Source:** [Pricing](https://docs.together.ai/docs/inference/pricing)

### Billing by deployment mode

| Mode | Billing model |
|------|---------------|
| **Serverless models** | Per usage (token/megapixel/second/char); no minimums, no provisioning cost |
| **Provisioned throughput** | Per PTU on a reserved term (1-month minimum); $0.05/PTU/minute |
| **Dedicated endpoints** | Per-minute by reserved hardware while running; per active replica |
| **GPU clusters** | Per-GPU hourly (on-demand) or upfront reservation (1–90 days), mixable |
| **Dedicated containers** | Per-replica resource allocation (cpu/memory/storage/gpu), min/max replicas |
| **Batch processing** | Per successful response; 50% discount on select serverless models |

### Serverless billing units

| Model type | Billing unit |
|------------|--------------|
| Chat, language, embedding, rerank | Per input and output **token** |
| Image generation | Per **megapixel** of output |
| Video generation | Per **second** of output |
| Speech-to-text and text-to-speech | Per **second of audio** |
| Text-to-speech (some models) | Per 1M **characters** |

### Cached input discounts (serverless chat models)

- **Automatic**: no header, parameter, or account toggle needed. Send the same prompt prefix again; any portion still warm in shared cache is billed at cached rate.
- **Prefix-based**: only the **longest matching prefix** of input counts as cached. Tokens after the first difference are billed at standard input rate.
- **Best-effort and short-lived**: shared cache across the fleet, entries evicted as traffic shifts; cache hits not guaranteed. For predictable caching, use a **dedicated endpoint** (prompt caching enabled by default and scoped to your own replicas).
- **Limited to supported models**: only models with a "Cached input pricing" column value support it.

### Dedicated endpoint billing

- Pay per minute for the hardware you reserve, regardless of model or request volume.
- Billing starts when the endpoint is running.
- Provisioning, queuing, and failed deployments are **not** billed.
- Per active replica; scales with GPU count.
- Stopping pauses billing immediately.

### Batch billing

- Billed per **successful response** in the output file. Failed requests aren't billed.
- Cancelling doesn't refund successful responses generated before the cancel landed.
- Dedicated endpoints can run batch jobs but at **full price** (no discount).

---

## 19. Rate Limits

**Source:** [Rate Limits](https://docs.together.ai/docs/serverless/rate-limits)

### Dynamic per-model rate limiting

Together AI applies **dynamic per-model rate limits** that scale with your sustained traffic on serverless inference. Limits are set **per organization** and **per model**, and adjust based on recent usage.

### Key concepts

- **Dynamic (not fixed) thresholds**: Each organization has a dynamic rate per model that adjusts based on the model's live capacity and your recent successful usage.
- **Sustained, successful traffic** raises your dynamic rate over time.
- **Sudden spikes** far above recent usage may be throttled.
- The platform **buffers sudden traffic spikes** with best-effort smoothing before applying any limiting.

### Error codes

| Condition | HTTP Status | Error type | Attribution |
|-----------|-------------|------------|-------------|
| Request at or below your dynamic rate fails | `503` | — | Platform capacity (model overloaded) — not your usage |
| Request above dynamic rate (request-based) | `429` | `dynamic_request_limited` | Your usage exceeded limit |
| Request above dynamic rate (token-based) | `429` | `dynamic_token_limited` | Your token usage exceeded limit |

### Rate limit headers

Every serverless inference API response includes the `x-ratelimit-reset` response header, reporting the **suggested retry interval (seconds)** for the model. On a `429`, the value reports how many seconds to wait before retrying.

### How to inspect current rate limit

```python
response = client.chat.completions.with_raw_response.create(
    model="meta-llama/Llama-3.3-70B-Instruct-Turbo",
    messages=[{"role": "user", "content": "ping"}],
)
print("x-ratelimit-reset:", response.http_response.headers.get("x-ratelimit-reset"))
```

### Best practices

- Stay within your rate limit.
- Send **steady, consistent** traffic — avoid bursts.
- Use **exponential backoff** for retries.
- Track spend/usage via project cost analytics page.

### Alternatives for high-volume or bursty workloads

| Option | Use case |
|--------|----------|
| **Batch inference** | High request/token volumes when latency isn't critical; pay-per-use with discounts |
| **Provisioned throughput** | Production workloads on stock models needing a defined SLA |
| **Dedicated inference** | Predictable, reserved capacity you control |
| **GPU clusters** | Full control, custom training pipelines, multi-node distributed training |

### Rate limits comparison

| Aspect | Serverless | Dedicated |
|--------|------------|-----------|
| Limit type | Dynamic per org per model | No shared-fleet rate limits |
| Behavior on exceed | `429` or `503` | N/A (reserved hardware) |
| Inspection | `x-ratelimit-reset` header | N/A |
| Adjustment | Scales with sustained successful traffic | N/A |

---

## 20. Recommended Models

**Source:** [Recommended Models](https://docs.together.ai/docs/inference/recommended-models)

### Chat & text

| Use case | Recommended model | Model string | Alternatives |
|----------|-------------------|--------------|---------------|
| Chat | Kimi K2.6 (instant mode) | `moonshotai/Kimi-K2.6` | `MiniMaxAI/MiniMax-M3` |
| Reasoning | Kimi K2.6 (reasoning mode) | `moonshotai/Kimi-K2.6` | `zai-org/GLM-5.2` |
| Coding agents | Kimi K2.7 Code | `moonshotai/Kimi-K2.7-Code` | `zai-org/GLM-5.2` |
| Small and fast | Gemma 4 31B IT | `google/gemma-4-31B-it` | `openai/gpt-oss-20b`, `Qwen/Qwen3.5-9B` |
| Mid-size general purpose | MiniMax M3 | `MiniMaxAI/MiniMax-M3` | `MiniMaxAI/MiniMax-M2.7`, `meta-llama/Llama-3.3-70B-Instruct-Turbo` |
| Function calling | GLM-5.2 | `zai-org/GLM-5.2` | `moonshotai/Kimi-K2.6` |

### Vision

| Use case | Recommended model | Model string | Alternatives |
|----------|-------------------|--------------|---------------|
| Vision | Kimi K2.6 | `moonshotai/Kimi-K2.6` | `google/gemma-4-31B-it`, `MiniMaxAI/MiniMax-M3` |

### Image generation

| Use case | Recommended model | Model string | Alternatives |
|----------|-------------------|--------------|---------------|
| Text-to-image | Flash Image 2.5 | `google/flash-image-2.5` | `black-forest-labs/FLUX.2-pro`, `ByteDance-Seed/Seedream-4.0` |
| Image-to-image | Flash Image 2.5 | `google/flash-image-2.5` | `black-forest-labs/FLUX.1-kontext-max`, `google/gemini-3-pro-image` |

### Video generation

| Use case | Recommended model | Model string | Alternatives |
|----------|-------------------|--------------|---------------|
| Text-to-video | Sora 2 Pro | `openai/sora-2-pro` | `google/veo-3.0`, `ByteDance/Seedance-1.0-pro` |
| Image-to-video | Veo 3.0 | `google/veo-3.0` | `ByteDance/Seedance-1.0-pro`, `kwaivgI/kling-2.1-master` |

### Audio

| Use case | Recommended model | Model string | Alternatives |
|----------|-------------------|--------------|---------------|
| Text-to-speech | Cartesia Sonic 3 | `cartesia/sonic-3` | `canopylabs/orpheus-3b-0.1-ft`, `hexgrad/Kokoro-82M` |
| Speech-to-text | Whisper Large v3 | `openai/whisper-large-v3` | `nvidia/parakeet-tdt-0.6b-v3`, `deepgram/nova-3-en` |

### Embeddings, rerank, and moderation

| Use case | Recommended model | Model string | Notes |
|----------|-------------------|--------------|-------|
| Embeddings | Multilingual E5 Large | `intfloat/multilingual-e5-large-instruct` | — |
| Rerank | MixedBread Rerank Large | `mixedbread-ai/Mxbai-Rerank-Large-V2` | Only on dedicated endpoints |
| Moderation | Llama Guard 4 12B | `meta-llama/Llama-Guard-4-12B` | — |

---

## 21. API Endpoint Reference Summary

### Inference endpoints (shared across serverless, provisioned, and dedicated)

| Capability | HTTP Method | Path | SDK Method |
|-----------|-------------|------|------------|
| Chat completions | POST | `/v1/chat/completions` | `client.chat.completions.create()` |
| Legacy text completions | POST | `/v1/completions` | `client.completions.create()` |
| Embeddings | POST | `/v1/embeddings` | `client.embeddings.create()` |
| Rerank | POST | `/v1/rerank` | `client.rerank.create()` |
| Image generation | POST | `/v1/images/generations` | `client.images.generate()` |
| Text-to-speech | POST | `/v1/audio/speech` | `client.audio.speech.create()` |
| Speech-to-text | POST | `/v1/audio/transcriptions` | `client.audio.transcriptions.create()` |
| Translation | POST | `/v1/audio/translations` | `client.audio.translations.create()` |
| Model list | GET | `/v1/models` | `client.models.list()` / `client.models.retrieve()` |

### Batch API endpoints

| Operation | HTTP Method | Path | SDK Method |
|-----------|-------------|------|------------|
| Check batch status | GET | `/v1/batches/{batch_id}` | `client.batches.retrieve(id)` |
| Cancel a batch | POST | `/v1/batches/{batch_id}/cancel` | `client.batches.cancel(id)` |
| List batches | GET | `/v1/batches` | `client.batches.list()` |
| Download output/error file | — | Files API | `client.files.content(id)` |

### Dedicated endpoint management

| Operation | CLI Command | Python SDK | TypeScript SDK |
|-----------|-------------|------------|---------------|
| List hardware | `together endpoints hardware --model <id>` | `client.endpoints.list_hardware(model=...)` | `client.endpoints.listHardware(...)` |
| List zones | `together endpoints availability-zones` | `client.endpoints.list_avzones()` | `client.endpoints.listAvzones()` |
| Create | `together endpoints create ...` | `client.endpoints.create(...)` | `client.endpoints.create(...)` |
| Inspect | `together endpoints retrieve <id>` | `client.endpoints.retrieve("id")` | `client.endpoints.retrieve("id")` |
| List | `together endpoints list` | `client.endpoints.list(mine=True)` | `client.endpoints.list({mine: true})` |
| Start/Stop | `together endpoints start/stop <id>` | `client.endpoints.update("id", state=...)` | `client.endpoints.update("id", {state: ...})` |
| Update | `together endpoints update ... <id>` | `client.endpoints.update("id", ...)` | `client.endpoints.update("id", ...)` |
| Delete | `together endpoints delete <id>` | `client.endpoints.delete("id")` | `client.endpoints.delete("id")` |

### GPU cluster management

| Operation | CLI Command |
|-----------|-------------|
| Create cluster | `tg beta clusters create ...` |
| Delete cluster | `tg beta clusters delete [CLUSTER_ID]` |
| List clusters | `tg beta clusters list` |
| Scale cluster | `tg beta clusters update [CLUSTER_ID] --num-gpus N` |
| Get kubeconfig | `tg beta clusters get-credentials [CLUSTER_ID] --set-default-context` |

### Dedicated container endpoints

| Method | Endpoint | Purpose |
|--------|----------|---------|
| `GET` | `/v1/deployment-request/{model}/health` | Health check |
| `POST` | `/v1/queue/submit` | Submit async job to queue |
| `GET` | `/v1/queue/status?model={m}&request_id={id}` | Poll job status |

### Common configuration

- **Base URL:** `https://api.together.ai`
- **OpenAI-compatible base URL:** `https://api.together.ai/v1`
- **Auth header:** `Authorization: Bearer $TOGETHER_API_KEY`
- **Content-Type:** `application/json`
- **Full OpenAPI spec:** `https://docs.together.ai/openapi.yaml`

### Deployment mode comparison summary

| Dimension | Serverless | Provisioned Throughput | Dedicated Endpoints | GPU Clusters | Dedicated Containers |
|-----------|------------|------------------------|---------------------|--------------|----------------------|
| Abstraction level | Hosted models | Reserved model capacity | Reserved hardware for one model | Raw infra (K8s/Slurm) | Managed deployment platform |
| What you bring | Nothing (use existing models) | Nothing (commit to capacity) | Model selection + config | Workload + scheduling | Docker image + inference code |
| Control | None | None | Container + config | Full (nodes, storage, network) | Container + config only |
| Access | Chat/completions API | Same API | Same API + endpoint management | kubectl, SSH, Slurm, REST/CLI | REST API, Jig CLI, Sprocket SDK |
| Scaling | Together-managed (dynamic) | Committed capacity | min/max replicas autoscaling | Manual or K8s autoscaler | min/max replicas autoscaling |
| Billing | Per token | Per PTU ($0.05/min), 1-month min | Per minute by hardware | Per-GPU hourly or reservation (1-90 days) | Per-replica resources |
| Rate limits | Dynamic per org per model | SLA-backed | None (reserved hardware) | N/A | N/A |
| Best for | Prototyping, variable traffic | Production stock models | Fine-tuned models, hardware control | Training, fine-tuning, custom stacks | Custom model inference, quick deploy |
| Setup time | Instant | Contact sales | Minutes | Minutes | ~20 minutes |
| Access gating | Self-serve | Contact sales | Self-serve | Self-serve | Contact sales |