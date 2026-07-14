# Fireworks AI API Analysis — On-Demand LLM Inference Services

> **Inference base URL:** `https://api.fireworks.ai/inference/v1` (OpenAI-compatible) | `https://api.fireworks.ai/inference` (Anthropic-compatible)
> **Control-plane base URL:** `https://api.fireworks.ai/v1/accounts/{ACCOUNT_ID}/...`
> **Docs:** `https://docs.fireworks.ai/getting-started/introduction` | **Auth:** `Authorization: Bearer $FIREWORKS_API_KEY` (keys at `https://app.fireworks.ai/settings/users/api-keys` or `firectl api-key create`)
> **SDKs:** Fireworks Python SDK (`pip install --pre fireworks-ai`, alpha), OpenAI Python/JS SDK (drop-in via `base_url="https://api.fireworks.ai/inference/v1"`), Anthropic Python/JS SDK (via `base_url="https://api.fireworks.ai/inference"`), `firectl` CLI (deployment/account management)
> **Description:** Fireworks AI is a platform for inference and fine-tuning of 100+ open-source models (text, vision, audio, embeddings). It exposes two complementary on-demand LLM inference services: **Serverless** (multi-tenant, pay-per-token, no GPUs to manage) and **On-Demand Deployments** (dedicated GPUs, billed per GPU-second, autoscaling, broader model selection, supports custom/LoRA models). Both are queried through the same OpenAI-compatible inference endpoints, distinguished only by the `model` identifier string. This study covers the capabilities offered by both inference surfaces — text generation, vision-language models, embeddings & reranking, the Responses API, batch inference, video/audio inputs, and RL rollout inference — plus the deployment/autoscaling/hardware controls that govern on-demand capacity. Fine-tuning (training), the Admin/Quota API, and MCP/tool-calling are referenced where they intersect inference but are not the primary focus.

---

## Table of Contents

1. [Platform Overview & Core Concepts](#1-platform-overview--core-concepts)
2. [API Surfaces — Serverless vs On-Demand, OpenAI vs Anthropic](#2-api-surfaces--serverless-vs-on-demand-openai-vs-anthropic)
3. [Authentication & Request Conventions](#3-authentication--request-conventions)
4. [Text Generation (Chat Completions)](#4-text-generation-chat-completions)
5. [Sampling & Generation Parameters](#5-sampling--generation-parameters)
6. [Reasoning (Thinking) Models](#6-reasoning-thinking-models)
7. [Structured Outputs & Function Calling](#7-structured-outputs--function-calling)
8. [Streaming & Performance Metrics](#8-streaming--performance-metrics)
9. [Vision-Language Models (VLMs)](#9-vision-language-models-vlms)
10. [Video & Audio Inputs (Omni Models)](#10-video--audio-inputs-omni-models)
11. [Embeddings](#11-embeddings)
12. [Reranking](#12-reranking)
13. [Responses API (Stateful Conversations & Tools)](#13-responses-api-stateful-conversations--tools)
14. [Batch Inference](#14-batch-inference)
15. [RL Rollout Inference (Session Affinity, Hot-Load, MoE Router Replay)](#15-rl-rollout-inference-session-affinity-hot-load-moe-router-replay)
16. [On-Demand Deployments — Creation, Shapes, Hardware](#16-on-demand-deployments--creation-shapes-hardware)
17. [Autoscaling Configuration](#17-autoscaling-configuration)
18. [LoRA Deployment (Live Merge & Multi-LoRA)](#18-lora-deployment-live-merge--multi-lora)
19. [Custom Model Upload (REST API)](#19-custom-model-upload-rest-api)
20. [Serving Paths — Standard, Priority, Fast](#20-serving-paths--standard-priority-fast)
21. [Pricing & Billing](#21-pricing--billing)
22. [Rate Limits & Quotas](#22-rate-limits--quotas)
23. [Error Codes](#23-error-codes)
24. [Lifecycle, Compliance & Known Limitations](#24-lifecycle-compliance--known-limitations)
25. [Capability Summary & Cross-Reference](#25-capability-summary--cross-reference)

---

## 1. Platform Overview & Core Concepts

### Main concepts

Fireworks AI organizes its on-demand inference offering around a small number of abstractions shared across the Serverless and On-Demand surfaces:

- **Model** — A specific open-source AI model identified by a slug of the form `accounts/fireworks/models/<model-id>` (e.g. `accounts/fireworks/models/deepseek-v3p1`, `accounts/fireworks/models/gpt-oss-120b`). Fast variants use a router path: `accounts/fireworks/routers/<model-id>-fast`. Custom/uploaded models use `accounts/<ACCOUNT_ID>/models/<model-id>`. Browse the catalog at `https://fireworks.ai/models`; each model has a per-model page at `https://app.fireworks.ai/models/fireworks/<model-id>`.
- **Deployment** — A dedicated GPU allocation running a model, identified by `accounts/<ACCOUNT_ID>/deployments/<DEPLOYMENT_ID>`. Created via `firectl deployment create` or the control-plane API. Querying a deployment uses the same inference endpoints as Serverless, with the deployment `Name` substituted for the `model` field. Deployments have a lifecycle: `CREATING` → `READY` → (`UPDATING`/`DELETING`) → `DELETED`/`FAILED`.
- **Serverless** — Multi-tenant inference for popular models running on Fireworks-managed infrastructure. Pay per token. No GPUs to size, no autoscaler to tune, no cold starts. Models eligible for Serverless carry the **Serverless** tag in the model library. At least 2 weeks advance notice before any model removal (longer for popular models).
- **On-Demand Deployment** — Dedicated GPUs giving better performance, **no hard rate limits** (only capacity-bound), fast autoscaling, minimal cold starts, and a wider model selection (including custom/LoRA models). Billed per **GPU-second**.
- **Deployment Shape** — A pre-configured template bundling hardware type, quantization, and performance factors. Three categories: **Fast** (low latency), **Throughput** (cost-per-token at scale), **Minimal** (lowest cost for testing). Referenced by shorthand (`fast`, `throughput`, `cost`) or full ID (`accounts/fireworks/deploymentShapes/<model>-<shape>`).
- **Accelerator** — The GPU type backing a deployment: `NVIDIA_A100_80GB`, `NVIDIA_H100_80GB`, `NVIDIA_H200_141GB` (and MI300X mentioned in hardware guidance). Availability varies by region.
- **Region** — Deployment placement: `GLOBAL` (recommended default), `US`, `EUROPE`, `APAC`. Set at creation time; cannot be changed in place.
- **Serving Path** — One of three routing/pricing tiers on the Serverless framework: **Standard** (default), **Priority** (higher reliability during peak, opt in via `service_tier: "priority"`), **Fast** (high-speed variant, opt in via different model ID).
- **Prompt Cache** — On by default for every Serverless model. Cached input tokens billed at a discounted rate (default 50% of input on text/vision). Caching is replica-local; maximize hit rate via `x-session-affinity` header or the OpenAI `user` field.
- **Datum** (Training API only) — Unit of training data for the separate Training API (private preview); not part of inference but referenced for checkpoint/weight-sync interplay with rollout inference.

### Two inference services at a glance

| Dimension | Serverless | On-Demand Deployment |
|-----------|-----------|----------------------|
| **Pricing unit** | per token (input / cached input / output) | per GPU-second |
| **Capacity** | multi-tenant, adaptive rate limits | dedicated GPUs, no hard rate limits (only capacity) |
| **Cold starts** | none | scale-from-zero returns 503 `DEPLOYMENT_SCALING_UP` |
| **Model selection** | popular base models (Serverless-tagged) | broader: any deployable base model + custom + LoRA |
| **Autoscaling** | Fireworks-managed | user-tuned (`--min/--max-replica-count`, `--load-targets`) |
| **LoRA support** | not supported | supported (live merge or multi-LoRA) |
| **Custom models** | not supported | supported (upload from HuggingFace) |
| **Hardware/region control** | none | full (`--accelerator-type`, `--region`) |
| **Stability** | models may be updated/deprecated (≥2 weeks notice) | full control over versions/updates |
| **Same API?** | yes — `/v1/chat/completions` etc. | yes — same endpoints, deployment `Name` as `model` |

### Platform architecture

```
Client ──▶ Fireworks Inference Gateway (https://api.fireworks.ai/inference/v1)
            │  Authorization: Bearer <key>, Content-Type: application/json
            │  Optional: x-session-affinity, x-multi-turn-session-id
            ▼
         Routing layer (model string selects path)
            │
   ┌────────┴─────────────────────────────────────┐
   ▼                                              ▼
Serverless (multi-tenant)                On-Demand Deployment (dedicated GPUs)
   model = accounts/fireworks/models/...     model = accounts/<acct>/deployments/<id>
   Standard | Priority | Fast                 deployment-shape (fast/throughput/minimal)
   adaptive rate limits (TPM)                 autoscaling (replicas, load-targets)
   prompt caching (replica-local)             scale-to-zero (503 DEPLOYMENT_SCALING_UP)
   prompt-cache discount (default 50%)        supports LoRA (live merge | multi-LoRA)
   batch = 50% off                            supports custom uploaded models
                                              GPU-second billing
            │
            ▼
         Endpoints:
           /v1/chat/completions   (text, vision, video/audio, rollout, MoE replay)
           /v1/completions        (legacy completions, VLM via <image> token, rollout)
           /v1/embeddings         (embeddings + rerank-via-logits)
           /v1/rerank             (document reranking)
           /v1/responses          (Responses API — stateful, MCP/SSE/function tools)

Control plane (https://api.fireworks.ai/v1/accounts/{ACCOUNT_ID}/...):
   /datasets, /datasets/{id}:upload, /datasets/{id}:getDownloadEndpoint
   /batchInferenceJobs[?batchInferenceJobId=], /batchInferenceJobs/{id}
   quota endpoints (/api-reference/list-quotas, get-quota, update-quota)
   /models (create/upload/validateUpload custom models)

Hot-load control (https://api.fireworks.ai/hot_load/v1/models/hot_load):
   per-snapshot reset_prompt_cache config
```

### Compliance

SOC 2, HIPAA; audit reports available.

---

## 2. API Surfaces — Serverless vs On-Demand, OpenAI vs Anthropic

Fireworks exposes inference through two compatibility layers and two serving models. The key insight: **Serverless and On-Demand use the exact same inference endpoints** — only the `model` string differs.

### Endpoint catalog

| Endpoint | Full URL | Method | Used by |
|----------|----------|--------|---------|
| Chat Completions | `https://api.fireworks.ai/inference/v1/chat/completions` | POST | Text, vision, video/audio, rollout, MoE replay |
| Completions (legacy) | `https://api.fireworks.ai/inference/v1/completions` | POST | Legacy completions, VLM via `<image>` token + `extra_body.images`, rollout |
| Embeddings | `https://api.fireworks.ai/inference/v1/embeddings` | POST | Text embeddings + rerank-via-logits |
| Rerank | `https://api.fireworks.ai/inference/v1/rerank` | POST | Document reranking |
| Responses API | `https://api.fireworks.ai/inference/v1/responses` | POST | Stateful conversations, MCP/SSE/function tools |
| Delete stored response | `https://api.fireworks.ai/inference/v1/responses/{response_id}` | DELETE | Responses API storage cleanup |

### Compatibility layers

| Layer | Base URL | Notes |
|-------|----------|-------|
| OpenAI-compatible | `https://api.fireworks.ai/inference/v1` | Drop-in for OpenAI Python/JS SDK via `base_url` override. Chat Completions, Completions, Embeddings, Responses. |
| Anthropic-compatible | `https://api.fireworks.ai/inference` | Drop-in for Anthropic Python/JS SDK. Uses `messages` API with `max_tokens` (required), `thinking` for reasoning, `output_config.format` for structured output, `tool_use`/`thinking`/`text` content blocks. |

### Model identifier formats

| Type | Format | Example |
|------|--------|---------|
| Serverless model | `accounts/fireworks/models/<model-id>` | `accounts/fireworks/models/deepseek-v3p1` |
| Fast variant | `accounts/fireworks/routers/<model-id>-fast` | `accounts/fireworks/routers/glm-5p2-fast` |
| On-demand deployment | `accounts/<ACCOUNT_ID>/deployments/<DEPLOYMENT_ID>` | `accounts/alice/deployments/12345678` |
| Deployment model string (Fireworks SDK) | `<base-model>#<deployment>` | `accounts/fireworks/models/gpt-oss-120b#accounts/alice/deployments/12345678` |
| Deployment model string (OpenAI/curl) | `<DEPLOYMENT_NAME>` | `accounts/alice/deployments/12345678` |
| Custom model | `accounts/<ACCOUNT_ID>/models/<model-id>` | `accounts/alice/models/my-custom-model` |
| Embeddings shorthand | `fireworks/<model-id>` | `fireworks/qwen3-embedding-8b` |
| Hot-load snapshot (streaming `model` field) | `accounts/<acct>/models/<model>@<snapshot>` | `accounts/alice/models/my-model@version_002` |

### Querying a deployment

Same request shape as Serverless; substitute the deployment `Name` for `model`. Examples from the quickstart work with either Serverless or a deployment by replacing only the `model` string.

```python
# OpenAI SDK pointing at a deployment
from openai import OpenAI
client = OpenAI(api_key=os.environ["FIREWORKS_API_KEY"],
                base_url="https://api.fireworks.ai/inference/v1")
response = client.chat.completions.create(
    model="accounts/<acct>/deployments/<id>",
    messages=[{"role": "user", "content": "Explain quantum computing"}]
)
```

---

## 3. Authentication & Request Conventions

### Authentication

- **Header:** `Authorization: Bearer $FIREWORKS_API_KEY` on every request.
- **Env var:** `FIREWORKS_API_KEY`.
- **Create keys:** dashboard at `https://app.fireworks.ai/settings/users/api-keys`, or `firectl api-key create`.
- **CLI auth:** `firectl signin`.
- **Additional header for stored-response deletion:** `x-fireworks-account-id: <account-id>` (on `DELETE /v1/responses/{id}`).

### Standard request headers

| Header | Value | Required |
|--------|-------|----------|
| `Authorization` | `Bearer <API_KEY>` | Yes |
| `Content-Type` | `application/json` | Yes |
| `x-session-affinity` | sticky-routing key (maximize prompt-cache hits) | Optional |
| `x-multi-turn-session-id` | RL trajectory ID (preferred over `x-session-affinity` for RL) | Optional |
| `fireworks-model` | `accounts/<acct>/models/<model>` (rollout) | Optional |
| `fireworks-deployment` | `accounts/<acct>/deployments/<dep>` (rollout) | Optional |

### Standard response headers (Serverless inference)

| Header | Meaning |
|--------|---------|
| `fireworks-prompt-tokens` | Input tokens for the request |
| `fireworks-cached-prompt-tokens` | Cached portion of the input |
| `fireworks-server-time-to-first-token` | TTFT |
| `fireworks-sampling-options` | JSON of actual sampling params used (incl. HuggingFace `generation_config.json` defaults) |
| `X-Ratelimit-Limit-Tokens-Prompt` | Current Total Prompt Tokens/sec limit |
| `X-Ratelimit-Limit-Tokens-Cache-Adjusted-Prompt` | Current Total Uncached Prompt Tokens/sec limit |
| `X-Ratelimit-Limit-Tokens-Generated` | Current Total Generated Tokens/sec limit |

Streaming responses do not carry per-request performance headers; set `perf_metrics_in_response: true` on the request body to receive metrics in the streamed body instead.

---

## 4. Text Generation (Chat Completions)

**Source pages:** `getting-started/quickstart`, `guides/querying-text-models`

### Main concepts

- OpenAI-compatible Chat Completions for 100+ open-source text models.
- Fireworks automatically applies each model's HuggingFace `generation_config.json` recommended sampling params when not specified.
- Most models auto-format `messages` with the correct chat template. Verify the exact prompt with the `echo` parameter.
- Three query APIs: **Chat Completions** (recommended), **Completions** (legacy `/v1/completions`), **Responses API** (`/v1/responses`, see §13).

### API function

| Function | Method & Endpoint | Purpose |
|----------|------------------|---------|
| Chat Completions | `POST https://api.fireworks.ai/inference/v1/chat/completions` | Generate text from a message history |
| Completions | `POST https://api.fireworks.ai/inference/v1/completions` | Legacy single-prompt completion (also VLM via `<image>` token) |

### Request parameters (Chat Completions)

| Parameter | Type | Default / values | Notes |
|-----------|------|------------------|-------|
| `model` | string | — | Serverless model id or deployment id |
| `messages` | array | — | Each `{role, content}`; roles: `system`, `user`, `assistant` |
| `stream` | bool | `false` | SSE streaming |
| `temperature` | float | from `generation_config.json` | 0 = deterministic, higher = more creative |
| `max_tokens` | int | **2048** | Limit reached → `finish_reason: "length"`; most models support full context window |
| `top_p` | float | from config | Nucleus sampling |
| `top_k` | int | from config | Top-k restriction |
| `min_p` | float | from config | Exclude tokens below threshold |
| `typical_p` | float | from config | Typical sampling |
| `frequency_penalty` | float | — | OpenAI-compatible; penalize frequent tokens |
| `presence_penalty` | float | — | OpenAI-compatible; penalize any repeated token |
| `repetition_penalty` | float | — | Exponential penalty from prompt + output |
| `n` | int | 1 | Multiple generations |
| `logprobs` | bool | `false` | Return token logprobs |
| `top_logprobs` | int | — | Number of top logprobs per token (requires `logprobs: true`) |
| `echo` | bool | `false` | Return prompt + generation (prompt inspection) |
| `return_token_ids` | bool | `false` | Return prompt/completion token IDs |
| `raw_output` | bool | `false` | Experimental; raw completion output |
| `ignore_eos` | bool | `false` | Experimental; always generate `max_tokens` (quality may degrade) |
| `logit_bias` | object | — | `{token_id: bias}`; bias -50..10 |
| `mirostat_target` | float | — | Target perplexity (Mirostat sampling) |
| `mirostat_lr` | float | — | Learning rate for Mirostat adjustments |
| `perf_metrics_in_response` | bool | `false` | Include perf metrics in streaming body |
| `service_tier` | string | `"priority"` | Opt into Priority tier (Serverless) |
| `reasoning_effort` | string | e.g. `"medium"` | Reasoning control (passed via `extra_body` with OpenAI SDK) |
| `response_format` | object | — | Structured outputs (see §7) |
| `tools` | array | — | Function calling (see §7) |
| `user` | string | — | OpenAI field; also usable for session affinity |

### Response fields

- `choices[].message.content` (non-streaming) / `choices[].delta.content` (streaming)
- `choices[].message.tool_calls`, `choices[].message.reasoning_content`
- `choices[].finish_reason` — e.g. `"length"`, `"stop"`
- `usage.prompt_tokens`, `usage.completion_tokens`, `usage.total_tokens` (auto-included in final streaming chunk — a Fireworks extension)
- `choices[].logprobs.content[].token`, `.logprob`, `.routing_matrix` (MoE, see §15)
- `prompt_token_ids`, `choices[].token_ids` (with `return_token_ids`)
- `choices[].raw_output.completion` (with `raw_output`)

### Multi-turn conversations

Include full history each request (stateless by default; use Responses API for server-side state). To omit a system prompt, set the first message content to an empty string.

```python
messages = [
    {"role": "system", "content": "You are a helpful assistant."},
    {"role": "user", "content": "What's the capital of France?"},
    {"role": "assistant", "content": "The capital of France is Paris."},
    {"role": "user", "content": "What's its population?"}
]
```

---

## 5. Sampling & Generation Parameters

**Source page:** `guides/querying-text-models`

Fireworks pulls defaults from each model's HuggingFace `generation_config.json` when parameters are omitted. The actual values used are returned in the `fireworks-sampling-options` response header (JSON).

### Sampling parameter families

| Family | Parameters | Notes |
|--------|-----------|-------|
| **Temperature/nucleus** | `temperature`, `top_p` | Standard OpenAI semantics |
| **Top-k / min-p / typical-p** | `top_k`, `min_p`, `typical_p` | Pulled from `generation_config.json` if unspecified |
| **Penalties** | `frequency_penalty`, `presence_penalty` (OpenAI-compatible), `repetition_penalty` (exponential) | OpenAI pair + Fireworks-native exponential |
| **Multiple generations** | `n` | Returns `choices[]` array |
| **Logprobs** | `logprobs`, `top_logprobs` | Per-token logprobs |
| **Prompt inspection** | `echo`, `return_token_ids`, `raw_output` | Debugging/verification |
| **EOS control** | `ignore_eos` | Experimental; forces `max_tokens` generation |
| **Logit bias** | `logit_bias` | `{token_id: bias}`; bias -50..10 |
| **Mirostat** | `mirostat_target`, `mirostat_lr` | Perplexity-targeted adaptive sampling |

### Reading effective defaults

```python
response = client.chat.completions.with_raw_response.create(
    model="accounts/fireworks/models/deepseek-v3p1",
    messages=[{"role": "user", "content": "Hello"}]
)
sampling_options = response.headers.get('fireworks-sampling-options')
# e.g. '{"temperature": 0.7, "top_p": 0.9}'
```

### Async requests

Use `AsyncFireworks` / `AsyncOpenAI` for async/await patterns (identical parameters).

---

## 6. Reasoning (Thinking) Models

**Source page:** `getting-started/quickstart`

### Main concepts

- Some models (e.g. `accounts/fireworks/models/glm-5p2`) generate internal **reasoning tokens** before visible output.
- Reasoning content returned in a separate `reasoning_content` field on the message (distinct from `content`).
- Control via `reasoning_effort` (string, e.g. `"medium"`) — passed through `extra_body` when using the OpenAI SDK.

### OpenAI-compatible reasoning control

| Parameter | Type | Values | Notes |
|-----------|------|--------|-------|
| `reasoning_effort` | string | `"medium"` (example) | OpenAI SDK: `extra_body={"reasoning_effort": "medium"}` |

### Anthropic-compatible reasoning control

| Parameter | Type | Example | Notes |
|-----------|------|---------|-------|
| `thinking` | object | `{type: "enabled", budget_tokens: 4096}` | Anthropic SDK; response returns `thinking` and `text` blocks |

### Response

- Chat Completions: `choices[0].message.reasoning_content` (separate from `content`).
- Anthropic: `response.content` contains blocks with `type == "thinking"` (`block.thinking`) and `type == "text"` (`block.text`).

---

## 7. Structured Outputs & Function Calling

**Source pages:** `getting-started/quickstart`, `guides/querying-text-models`

### Structured outputs (JSON Schema enforcement)

Force output to conform to a JSON Schema. Detailed in `/structured-responses/structured-response-formatting`.

| API | Parameter | Shape |
|-----|-----------|-------|
| OpenAI/Fireworks | `response_format` | `{type: "json_schema", json_schema: {name, schema: {type: "object", properties, required}}}` |
| Anthropic | `output_config` | `{format: {type: "json_schema", schema: {...}}}` |

### Function / tool calling

Type-safe tool definitions. Detailed in `/guides/function-calling`.

| API | `tools` element shape |
|-----|----------------------|
| OpenAI/Fireworks | `{type: "function", function: {name, description, parameters: {type: "object", properties, required}}}` |
| Anthropic | `{name, description, input_schema: {type, properties, required}}` |

### Response

- Chat Completions: `choices[0].message.tool_calls`
- Anthropic: content blocks with `block.type == "tool_use"`, `block.name`, `block.input`

---

## 8. Streaming & Performance Metrics

**Source page:** `guides/querying-text-models`

### Streaming

Set `stream: true`. Receive `choices[].delta.content` chunks. Usage is automatically included in the final chunk (the chunk with `finish_reason` set) — a Fireworks extension (the OpenAI SDK does not return usage for streaming by default).

### Aborting streams

`stream.close()` / `break` — stops generation and avoids billing ungenerated tokens.

### Performance metrics in streaming

Non-streaming responses carry perf metrics in headers. For streaming, set `perf_metrics_in_response: true` to receive metrics in the response body:

```python
stream = client.chat.completions.create(
    model="accounts/fireworks/models/deepseek-v3p1",
    messages=[{"role": "user", "content": "Hello, world!"}],
    stream=True,
    extra_body={"perf_metrics_in_response": True}
)
for chunk in stream:
    if chunk.choices[0].delta.content:
        print(chunk.choices[0].delta.content, end="")
    if chunk.choices[0].finish_reason:
        if chunk.usage:
            print(f"\n\nTokens: {chunk.usage.total_tokens}")
        if hasattr(chunk, 'perf_metrics'):
            print(f"Performance: {chunk.perf_metrics}")
```

### Usage & analytics dashboards

- Analytics/usage dashboard: `https://app.fireworks.ai/account/usage` — measures server-acknowledged requests (not client-observed outcomes; does not capture connection timeouts, client retries, or network-path failures).
- Dedicated deployments: Prometheus-style metrics (`/deployments/exporting-metrics`).

---

## 9. Vision-Language Models (VLMs)

**Source page:** `guides/querying-vision-language-models`

### Main concepts

- VLMs process both text and images in a single request via the Chat Completions API.
- Use cases: image captioning, VQA, document analysis, chart interpretation, OCR, content moderation.
- Request structure is **identical to OpenAI's vision API**. Browse at `https://app.fireworks.ai/models?filter=Vision`.
- Available via serverless or dedicated deployments.
- VLMs support prompt caching (text and image portions) — can reduce TTFT by up to 80%.

### Message structure

`content` is an array of typed content parts:
- `{"type": "text", "text": "..."}`
- `{"type": "image_url", "image_url": {"url": "<URL or data URI>"}}`

### Image input formats

| Method | Format | Notes |
|--------|--------|-------|
| URL | `{"image_url": {"url": "https://..."}}` | Preferred for long conversations (lower latency) |
| Base64 | `{"image_url": {"url": "data:image/jpeg;base64,<data>"}}` | For local files; prefix with MIME type |

### Example (image via URL)

```python
response = client.chat.completions.create(
    model="accounts/fireworks/models/kimi-k2p5",
    messages=[{
        "role": "user",
        "content": [
            {"type": "text", "text": "Can you describe this image?"},
            {"type": "image_url", "image_url": {"url": "https://images.unsplash.com/..."}}
        ]
    }]
)
```

### Completions API alternative (VLM)

Insert the `<image>` token in the prompt and supply images as an ordered list via `extra_body.images`:

```python
response = client.completions.create(
    model="accounts/fireworks/models/kimi-k2p5",
    prompt="SYSTEM: Hello\n\nUSER:\n<image> tell me about the image\n\nASSISTANT:",
    extra_body={"images": ["https://images.unsplash.com/..."]}
)
```

### PDFs

VLMs do **not** natively accept PDF files. Convert each page to an image (e.g. PyMuPDF `fitz`, or JS `pdf-to-img`) and pass via base64. Remember the 30-image limit per request; batch pages for long documents.

### Known limitations (exact values)

| Limit | Value |
|-------|-------|
| Maximum images per request | **30** |
| Base64 total size | **< 10 MB** |
| Image URL size | **< 5 MB** |
| Image URL download timeout | **1.5 seconds** |
| Supported formats | `.png`, `.jpg`, `.jpeg`, `.gif`, `.bmp`, `.tiff`, `.ppm` |
| Llama 3.2 Vision ordering | Pass images **before** text in content (temporary limitation) |

### Performance tips

- Use URLs for long conversations (reduces latency vs base64).
- Downsize images (fewer tokens, faster).
- Structure prompts for caching (static instructions first, variable content last).

---

## 10. Video & Audio Inputs (Omni Models)

**Source page:** `guides/video-audio-inputs`

### Main concepts

- Some multimodal models process audio and/or video directly — video captioning, scene analysis, multimodal QA.
- Flagship: **Qwen3 Omni** (`qwen3-omni-30b-a3b-instruct`) — video, audio, and text in a single request.
- **Dedicated deployments required** for production; video models are NOT available on serverless.

### Available models

| Model | Input support | Deployment requirement |
|-------|---------------|------------------------|
| Qwen3 Omni 30B A3B Instruct | Video, audio, text | Dedicated deployment required |
| Molmo2-4B | Video, text | Dedicated deployment required (omit `audio_url`) |
| Molmo2-8B | Video, text | Dedicated deployment required (omit `audio_url`) |

Molmo2 models are video-only — they cannot understand audio from videos.

### Deployment creation

Must use the predefined `qwen3-omni-30b-a3b-instruct-minimal` deployment shape:

```
firectl deployment create qwen3-omni-30b-a3b-instruct \
  --account-id <YOUR_ACCOUNT_ID> \
  --min-replica-count 1 \
  --max-replica-count 1 \
  --deployment-shape qwen3-omni-30b-a3b-instruct-minimal
```

### Request structure

Endpoint: `POST https://api.fireworks.ai/inference/v1/chat/completions`. `model` uses the `model#deployment` form. Content types accepted in `messages[].content[]`: `video_url`, `audio_url`, `text`. Video/audio passed as **base64 data URLs**:

```json
{
  "model": "accounts/<acct>/models/qwen3-omni-30b-a3b-instruct#accounts/<acct>/deployments/<dep>",
  "messages": [{
    "role": "user",
    "content": [
      {"type": "video_url", "video_url": {"url": "data:video/mp4;base64,{video_b64}"}},
      {"type": "audio_url", "audio_url": {"url": "data:audio/ogg;base64,{audio_b64}"}},
      {"type": "text", "text": "Describe this video."}
    ]
  }],
  "max_tokens": 1000,
  "temperature": 0.3
}
```

### Preprocessing (ffmpeg)

**Video** (1 FPS, 360p, ≤60s):
```
ffmpeg -y -i input_video.mp4 -t 60 -vf "fps=1,scale=-1:360" -c:v libx264 -preset fast -an processed_video.mp4
```
| Flag | Effect |
|------|--------|
| `-t 60` | Limit to first 60 seconds |
| `fps=1` | 1 frame per second |
| `scale=-1:360` | Downscale to 360p, maintain aspect ratio |
| `-an` | Remove audio track (extract separately) |

**Audio** (Opus, 16 kHz mono, 24 kbps):
```
ffmpeg -y -i input_video.mp4 -t 60 -vn -c:a libopus -b:a 24k -ar 16000 -ac 1 audio.ogg
```
| Flag | Effect |
|------|--------|
| `-vn` | Remove video track |
| `-c:a libopus` | Opus codec |
| `-b:a 24k` | 24 kbps bitrate |
| `-ar 16000` | 16 kHz sample rate |
| `-ac 1` | Mono |

### Known limitations

| Limit | Value |
|-------|-------|
| Video duration | ≤ 60 seconds recommended |
| Supported video format | `.mp4` |
| Supported audio format | `.ogg` (Opus) |
| Base64 payload total | < 10 MB |
| Deployment | Required (not serverless) |

---

## 11. Embeddings

**Source page:** `guides/querying-embeddings-models`

### Main concepts

- Embeddings models take **text** as input and output a vector of floats (for similarity/search).
- OpenAI-compatible. Billed **only on input tokens** (no output dimension).
- Input is **text only** — no image input mentioned for embeddings.

### API function

| Function | Method & Endpoint | Purpose |
|----------|-------------------|---------|
| Embeddings | `POST https://api.fireworks.ai/inference/v1/embeddings` | Generate text embeddings / rerank via logits |

### Request parameters

| Parameter | Type | Notes |
|-----------|------|-------|
| `input` | string or array of strings | Text input |
| `model` | string | Embeddings model id (e.g. `fireworks/qwen3-embedding-8b`) |
| `dimensions` | int | Optional; variable-length embeddings (e.g. `128`) |
| `return_logits` | array of token IDs | Used for reranking via the embeddings endpoint |
| `normalize` | bool | Applies softmax to selected logits (used with `return_logits`) |

> Note: `encoding_format` is not documented on the embeddings page despite being an OpenAI-standard param.

### Example

```python
payload = {
    "input": "The quick brown fox jumped over the lazy dog",
    "model": "fireworks/qwen3-embedding-8b",
}
```

### Model availability

**Purpose-built embeddings (Qwen3 family — SOTA):**
- `fireworks/qwen3-embedding-8b` (serverless)
- `fireworks/qwen3-embedding-4b`
- `fireworks/qwen3-embedding-0p6b`

**Any LLM as an embeddings model:** `fireworks/glm-4p5`, `fireworks/gpt-oss-20b`, `fireworks/kimi-k2-instruct-0905`, `fireworks/deepseek-r1-0528`.

**Bring your own model** via custom upload (`/models/uploading-custom-models`).

**BERT-based (legacy, serverless only):** `nomic-ai/nomic-embed-text-v1.5`, `nomic-ai/nomic-embed-text-v1`, `WhereIsAI/UAE-Large-V1`, `thenlper/gte-large`, `thenlper/gte-base`, `BAAI/bge-base-en-v1.5`, `BAAI/bge-small-en-v1.5`, `mixedbread-ai/mxbai-embed-large-v1`, `sentence-transformers/all-MiniLM-L6-v2`, `sentence-transformers/paraphrase-multilingual-MiniLM-L12-v2`.

### Deployment

Qwen3 Embedding 8b and Reranker 8b are available on serverless; can also be deployed on-demand.

---

## 12. Reranking

**Source page:** `guides/querying-embeddings-models`

### Main concepts

- Reranking models rerank a list of documents based on a query. Qwen3 Reranker family: `fireworks/qwen3-reranker-8b`, `fireworks/qwen3-reranker-4b`, `fireworks/qwen3-reranker-0p6b` (8b on serverless).

### API functions

| Function | Method & Endpoint | Notes |
|----------|-------------------|-------|
| Rerank | `POST https://api.fireworks.ai/inference/v1/rerank` | Does not yet support all models/parallelism |
| Rerank via embeddings | `POST .../v1/embeddings` with `return_logits` + `normalize` | More flexible; recommended for large sets |

### `/rerank` parameters

| Parameter | Type | Notes |
|-----------|------|-------|
| `model` | string | Reranker model id |
| `query` | string | Search query |
| `documents` | array of strings | Candidate documents |
| `top_n` | int | Number of top results |
| `return_documents` | bool | Include documents in response |

### Rerank via `/embeddings` (return_logits)

Format prompts as query-document pairs using the **Qwen3 Reranker format**: `<instruct>\n{instruction}\n<query>\n{query}\n<doc>\n{doc}`. Token IDs: `token_false_id = 2753` ("no"), `token_true_id = 9454` ("yes").

```python
payload = {
    "model": "fireworks/qwen3-reranker-8b",
    "input": prompts,  # Qwen3-formatted query-doc pairs
    "return_logits": [token_false_id, token_true_id],
    "normalize": True  # softmax → probabilities sum to 1
}
# response["data"][i]["embedding"] = [no_prob, yes_prob]
# relevance_score = probs[1]  # "yes" probability
```

### Parallel reranking

For large document sets, async minibatches of 100 using `aiohttp`/`asyncio` are recommended (see cookbook).

---

## 13. Responses API (Stateful Conversations & Tools)

**Source page:** `guides/response-api`

### Main concepts

- **Responses API** = Fireworks' API for stateful, multi-turn, conversational workflows. Two differentiators vs Chat Completions: stateful continuation via `previous_response_id`, and a different **data retention policy** (`/guides/security_compliance/data_handling#response-api-data-retention`).
- Fully OpenAI-SDK compatible (`client.responses.create`).
- Designed for: continuing conversations without resending history, external tools (MCP/SSE server-executed, function client-executed), streaming, tool-usage control via `max_tool_calls`, data-retention management.

### API functions

| Function | Method & Endpoint | Purpose |
|----------|-------------------|---------|
| Create response | `POST https://api.fireworks.ai/inference/v1/responses` | Stateful generation with optional tools |
| Delete stored response | `DELETE https://api.fireworks.ai/inference/v1/responses/{response_id}` | Permanent removal (requires `x-fireworks-account-id` header) |

### Request parameters

| Parameter | Type | Default | Notes |
|-----------|------|---------|-------|
| `model` | string | — | e.g. `accounts/fireworks/models/qwen3-235b-a22b` |
| `input` | string | — | Prompt text |
| `previous_response_id` | string | — | ID of prior response to continue; fails if prior had `store=False` |
| `store` | bool | `true` | When `false`, cannot later use `previous_response_id` |
| `stream` | bool | `false` | Streaming |
| `tools` | array | — | Tool descriptors (see below) |
| `tool_choice` | string | `"auto"` | Tool selection policy |
| `max_tool_calls` | int | `null` | Max number of tool calls allowed |

### Tool types

| Type | Example | Execution |
|------|---------|-----------|
| `sse` | `{"type": "sse", "server_url": "https://gitmcp.io/docs"}` | Server-executed (MCP/SSE) |
| `mcp` | `{"type": "mcp", "server_url": "https://mcp.deepwiki.com/mcp"}` | Server-executed (MCP) |
| `function` | `{"type": "function", "name": "get_weather", "description": "...", "parameters": {...}}` | Client-executed |

### Response structure

```json
{
  "id": "resp_abc123...",
  "created_at": 1735000000,
  "status": "completed",
  "model": "accounts/fireworks/models/qwen3-235b-a22b",
  "output": [
    {"id": "msg_xyz789...", "role": "user", "content": [{"type": "input_text", "text": "What is 2+2?"}], "status": "completed"},
    {"id": "msg_def456...", "role": "assistant", "content": [{"type": "output_text", "text": "2 + 2 equals 4."}], "status": "completed"}
  ],
  "usage": {"prompt_tokens": 15, "completion_tokens": 8, "total_tokens": 23,
             "prompt_tokens_details": {"cached_tokens": 0}},
  "previous_response_id": null,
  "store": true,
  "max_tool_calls": null
}
```

### Key behaviors

- `store=True` (default) → responses stored and referenceable; deletable via `DELETE /v1/responses/{id}` (permanent, unrecoverable).
- `store=False` → cannot use `previous_response_id` to continue; attempting it raises an exception.
- Content types: `input_text`, `output_text`.
- `usage.prompt_tokens_details.cached_tokens` reports prompt-cache hits.

> Note: The Responses API page does not document `instructions`, `reasoning`, or `text.format` parameters — those are absent from the source. Reasoning/structured-output streaming events are not described on this page.

---

## 14. Batch Inference

**Source page:** `guides/batch-inference`

### Main concepts

- **Batch API** = process large volumes of async requests at **50% off** serverless per-token prices.
- Prompt caching automatically applied for additional ~50% savings on cached tokens (place static content first).
- Use cases: data labeling, synthetic data generation, distillation, large-scale evals/benchmarking, document processing.
- Model compatibility: any model supporting On-Demand Deployments in the Model Library; custom models on a batch-compatible base.
- This is the **Fireworks-native Batch API** (account-scoped `/batchInferenceJobs`), NOT an OpenAI-style `/v1/batches` endpoint.

### API functions (account-scoped; base `https://api.fireworks.ai/v1`)

| Function | Method & Endpoint | Purpose |
|----------|-------------------|---------|
| Create dataset | `POST /v1/accounts/{ACCOUNT_ID}/datasets` | Register input/output dataset |
| Upload dataset file | `POST /v1/accounts/{ACCOUNT_ID}/datasets/{datasetId}:upload` | Upload JSONL (multipart `file=@./...jsonl`) |
| Create batch job | `POST /v1/accounts/{ACCOUNT_ID}/batchInferenceJobs?batchInferenceJobId={job-id}` | Submit async batch job |
| Get job | `GET /v1/accounts/{ACCOUNT_ID}/batchInferenceJobs/{job-id}` | Check status |
| List jobs | `GET /v1/accounts/{ACCOUNT_ID}/batchInferenceJobs` | Enumerate all jobs |
| Get download endpoint | `GET /v1/accounts/{ACCOUNT_ID}/datasets/{output-dataset-id}:getDownloadEndpoint` | Get signed URLs for output files (body `{}`) |

### `firectl` CLI commands

| Command | Purpose |
|---------|---------|
| `firectl dataset create <name> ./batch_input_data.jsonl` | Create dataset |
| `firectl batch-inference-job create --model ... --input-dataset-id ...` | Create job (minimal) |
| `firectl batch-inference-job create --job-id ... --model ... --input-dataset-id ... --output-dataset-id ... --max-tokens 1024 --temperature 0.7 --top-p 0.9` | Create job (full) |
| `firectl batch-inference-job get <job-id>` | Get job status |
| `firectl batch-inference-job list` | List jobs |
| `firectl dataset download <output-dataset-id>` | Download output |
| `firectl dataset download <id> --download-lineage` | Download all datasets in continuation chain |
| `firectl batch-inference-job create --continue-from <original-job-id> --model ... --output-dataset-id <new>` | Resume expired job (processes only unfinished/failed requests) |

### Input dataset format

- **Format:** JSONL — one valid JSON object per line.
- **Size limit:** under **1 GB**.
- **Required fields per line:** `custom_id` (unique) and `body` (request parameters).
- `body` contains normal chat-completion-style params (`messages`, `max_tokens`, `temperature`, etc.).

```jsonl
{"custom_id": "request-1", "body": {"messages": [{"role": "system", "content": "You are a helpful assistant."}, {"role": "user", "content": "What is the capital of France?"}], "max_tokens": 100}}
{"custom_id": "request-2", "body": {"messages": [{"role": "user", "content": "Explain quantum computing"}], "temperature": 0.7}}
```

### Create batch job — request body fields

| Field | Notes |
|-------|-------|
| `model` | e.g. `accounts/fireworks/models/llama-v3p1-8b-instruct` |
| `inputDatasetId` | `accounts/{ACCOUNT_ID}/datasets/batch-input-dataset` |
| `outputDatasetId` | `accounts/{ACCOUNT_ID}/datasets/batch-output-dataset` |
| `inferenceParameters` | Object with `maxTokens`, `temperature`, `topP` (camelCase) |
| Query param `batchInferenceJobId` | Custom job ID |

### Job states

| State | Description |
|-------|-------------|
| `VALIDATING` | Dataset is being validated for format requirements |
| `PENDING` | Job is queued and waiting for resources |
| `RUNNING` | Actively processing requests |
| `COMPLETED` | All requests successfully processed |
| `FAILED` | Unrecoverable error (check status message) |
| `EXPIRED` | Exceeded chosen time limit; completed requests are saved |

### Output / results retrieval

Output dataset contains **two files**:
- **results file** — successful responses (JSONL).
- **error file** — failed requests with debugging info.

Download via signed URLs: response JSON has `filenameToSignedUrls` map; extract with jq and `curl -L -o "$fname" "$signed_url"`.

### Limits & constraints

| Constraint | Value |
|-----------|-------|
| Per-request limits | Same as Chat Completion API |
| Input dataset max | 1 GB |
| Output dataset max | 8 GB (job may expire early if reached) |
| Job expiration windows | 12, 24, 48, or 72 hours (selectable; default 24h) |

### Troubleshooting

1. If validation failed, check JSONL — each line must be complete valid JSON.
2. Jobs wait in "pending" during the chosen window — may not run immediately.
3. If "creating" a deployment for >30 minutes, contact support with job ID; confirm model supports batch; check account quota.
4. Progress may pause while waiting on capacity; resumes automatically.
5. If a model doesn't support batch, submitting a job may NOT error immediately — the job may sit "pending" and never schedule. Always verify compatibility.

---

## 15. RL Rollout Inference (Session Affinity, Hot-Load, MoE Router Replay)

**Source page:** `guides/rollout-inference`

### Main concepts

- **Rollout inference** = features on the regular `/v1/completions` and `/v1/chat/completions` endpoints tailored to multi-turn, stateful **RL rollout traffic**.
- Features: **session affinity**, **KV-cache behavior**, **weight-swap behavior**, **MoE Router Replay (R3)**.
- Compatible with OpenAI SDKs — all features exposed via request headers or optional body fields; no SDK upgrade required.
- Works whether or not the underlying deployment is a hot-load deployment.

### Session affinity (headers)

| Header | Purpose |
|--------|---------|
| `x-multi-turn-session-id` | Identifies the agent trajectory; set once per trajectory, keep constant across turns. **Preferred** when both are present. |
| `x-session-affinity` | Fallback sticky-routing key when `x-multi-turn-session-id` is absent. |
| `user` | General prompt-caching flows; RL traffic should use the headers above. |

```
-H "x-multi-turn-session-id: traj-42f1"
-H "x-session-affinity: traj-42f1"
-H "fireworks-model: accounts/<acct>/models/<model>"
-H "fireworks-deployment: accounts/<acct>/deployments/<dep>"
```

### KV-cache behavior — three-layer model

| Layer | Scope | What it controls | What it does not control |
|-------|-------|------------------|--------------------------|
| **Single request stream** | One in-flight HTTP request | Active in-flight KV/state for that stream | Future prompt-prefix reuse after the stream ends |
| **Session ID** | Later requests with the same trajectory key | Sticky routing to same replica + `new_session` namespace behavior | A cache hit by itself, or active-stream recompute |
| **`reset_prompt_cache`** | Requests admitted after a checkpoint swap | Which reusable prompt-prefix KV namespace later requests can use | The active in-flight KV for a request already decoding |

#### Active request stream

- **Async transition** (recommended for RL): stream pauses, weights swap, same HTTP stream resumes with existing active KV/state. `reset_prompt_cache` does NOT flush that active KV.
- **Sync transition**: server waits for in-flight requests to finish on old weights before swapping. New requests arriving during swap may receive **HTTP `425 Too Early`** and should retry.

#### Session ID behavior

- Sticky routing to same replica → can see that replica's local prompt-prefix KV.
- With `reset_prompt_cache=new_session`, later requests with existing session ID stay pinned to previous prompt-cache namespace after a checkpoint swap.
- A reusable hit requires: request reaches a replica that owns the cached prefix, prompt tokens match, prompt-cache isolation key matches.

#### `reset_prompt_cache` (per snapshot via `POST /hot_load/v1/models/hot_load`)

```json
{ "identity": "version_002", "reset_prompt_cache": "new_session" }
```

| `reset_prompt_cache` | Active in-flight request crossing the swap | Later request with same `x-multi-turn-session-id` | Later request with new session ID |
|----------------------|---------------------------------------------|---------------------------------------------------|-----------------------------------|
| `all` (default) | Not recomputed by this setting | Recomputes prompt-prefix KV under new namespace | Recomputes prompt-prefix KV under new namespace |
| `new_session` | Not recomputed by this setting | Can reuse eligible prompt-prefix KV for that existing session | Recomputes prompt-prefix KV under new namespace |
| `none` | Not recomputed by this setting | Can reuse eligible prompt-prefix KV | Can reuse eligible prompt-prefix KV |

RL guidance:
- `new_session` — episode may continue across a weight sync; later turns keep reuse while new episodes use latest snapshot namespace.
- `all` — next request should recompute prompt-prefix KV even with same session ID.
- `none` — both existing and new sessions keep using previous namespace after swap.

### MoE Router Replay (R3)

- For Mixture-of-Experts models, aligning training/inference router choices (top-K experts per token) eliminates divergence. Paper: https://arxiv.org/abs/2510.11370.
- Request fields: pass `include_routing_matrix: true` together with `logprobs: true` (top-level body fields).

```bash
curl https://api.fireworks.ai/inference/v1/chat/completions \
  -H "Authorization: Bearer <KEY>" -H "Content-Type: application/json" \
  -d '{
    "model": "accounts/<acct>/models/<model>",
    "messages": [{"role": "user", "content": "..."}],
    "include_routing_matrix": true,
    "logprobs": true
  }'
```

- Selected expert indices returned alongside logprobs:
  - `/v1/chat/completions`: `choices[i].logprobs.content[j].routing_matrix`
  - `/v1/completions`: analogous structure.
- Each value = **flattened, base64-encoded `uint8` array** of shape `[num_layers_with_moe, num_active_experts]`.

Example (DeepSeek V3): 58 MoE layers, 8 active experts/token → 464 bytes per token.
```python
import base64, numpy as np
raw_bytes = base64.b64decode(choice["logprobs"]["content"][0]["routing_matrix"])
routing_matrix = np.frombuffer(raw_bytes, dtype=np.uint8).reshape(58, 8)
```

Other modes:
- **Streaming (`stream: true`):** `routing_matrix` on each streamed token chunk's `logprobs.content`.
- **Prompt tokens (`echo: true`):** returns expert selection for prompt tokens too. Combine with `echo_last: N` for last-N-only.

### Policy version in responses (hot-load)

- **Streaming:** each chunk's `model` field = `accounts/<acct>/models/<model>@<snapshot>`; parse suffix after `@` as policy version. If weight swap mid-stream (async), later chunks may reflect the new snapshot.
- **Non-streaming:** same `model@snapshot_identity` convention being added.

### Endpoints referenced

| Endpoint | Purpose |
|----------|---------|
| `POST /v1/completions` | Completions with rollout features |
| `POST /v1/chat/completions` | Chat with rollout features |
| `POST /hot_load/v1/models/hot_load` | Per-snapshot `reset_prompt_cache` config |

---

## 16. On-Demand Deployments — Creation, Shapes, Hardware

**Source pages:** `getting-started/ondemand-quickstart`, `guides/ondemand-deployments`, `faq/deployment/ondemand/hardware-options`

### Main concepts

- On-demand deployments give dedicated GPUs: better performance, **no hard rate limits** (only deployment capacity), fast autoscaling, minimal cold starts, broader model selection, custom/LoRA support.
- Billed per **GPU-second**. Pricing: https://fireworks.ai/pricing.
- By default, deployments scale to zero if unused for 1 hour. Deployments with min replicas = 0 are auto-deleted after 7 days of no traffic.
- When scaled to zero, requests return a `503` with `DEPLOYMENT_SCALING_UP` immediately while scaling up — implement retry logic.

### `firectl` CLI commands

| Command | Purpose |
|---------|---------|
| `firectl deployment create <model-ref> --wait` | Create deployment; block until ready |
| `firectl deployment get <DEPLOYMENT_ID>` | Check status/region placement |
| `firectl deployment list` | List all deployments |
| `firectl deployment delete <DEPLOYMENT_ID>` | Delete deployment |
| `firectl deployment-shape-version list --base-model <model-id>` | List available shapes for a model |
| `firectl deployment-shape-version get <full-shape-version-id>` | Inspect a shape |
| `firectl load-lora <FINE_TUNED_MODEL_ID> --deployment <DEPLOYMENT_ID>` | Load a LoRA adapter (multi-LoRA) |

### `firectl deployment create` flags

| Flag | Type | Default | Values / notes |
|------|------|---------|----------------|
| `--wait` | flag | — | Block until creation completes |
| `--region` | enum | (single datacenter) | `GLOBAL` (recommended), `US`, `EUROPE`, `APAC` — set at creation, cannot change in place |
| `--deployment-shape` | string | — | Shorthand (`fast`, `throughput`, `cost`) or full ID (`accounts/fireworks/deploymentShapes/<model>-<shape>`) |
| `--accelerator-type` | enum | model default | `NVIDIA_A100_80GB`, `NVIDIA_H100_80GB`, `NVIDIA_H200_141GB` |
| `--accelerator-count` | int | 1 | Multiple GPUs per replica; scaling is **sub-linear** (2x GPUs ≠ 2x performance) |
| `--min-replica-count` | int | 0 | 0 = scale-to-zero |
| `--max-replica-count` | int | 1 | Maximum replicas |
| `--scale-up-window` | duration | 30s | Wait before scaling up |
| `--scale-down-window` | duration | 10m | Wait before scaling down |
| `--scale-to-zero-window` | duration | 1h (min 5m) | Idle time before scaling to zero |
| `--load-targets` | key-value | `default=0.8` | Scaling thresholds (see §17) |
| `--enable-addons` | flag | — | Enable multi-LoRA addon support (BF16 only; not with FP8/FP4) |

### Deployment shapes

Pre-configured templates optimized for speed/cost/efficiency (hardware + quantization + performance factors):

| Shape | Optimized for |
|-------|---------------|
| **Fast** | Low latency for interactive workloads |
| **Throughput** | Cost-per-token at scale for high-volume workloads |
| **Minimal** | Lowest cost for testing or light workloads |

```
firectl deployment-shape-version list --base-model accounts/fireworks/models/llama-v3p3-70b-instruct
firectl deployment create accounts/fireworks/models/deepseek-v3 --deployment-shape throughput
firectl deployment create accounts/fireworks/models/llama-v3p3-70b-instruct \
  --deployment-shape accounts/fireworks/deploymentShapes/llama-v3p3-70b-instruct-fast
```

### Deployment states

| State | Meaning |
|-------|---------|
| `CREATING` | Still being created |
| `READY` | Ready to be used |
| `UPDATING` | In-progress updates |
| `DELETING` | Being deleted |
| `DELETED` | Soft-deleted |
| `FAILED` | Creation failed (see status) |

UI-only display labels (computed, not backend enums):
- `Inactive`: `state == READY && max_replica_count == 0 && ready_replica_count == 0`
- `Scaled to 0`: `state == READY && min_replica_count == 0 && max_replica_count > 0 && desired_replica_count == 0 && ready_replica_count == 0`

### Deployment response fields (from `firectl deployment create`)

`Name: accounts/<ACCOUNT_ID>/deployments/<DEPLOYMENT_ID>`, `Create Time`, `Expire Time`, `Created By`, `State`, `Status`, `Min Replica Count`, `Max Replica Count`, `Desired Replica Count`, `Replica Count`, `Autoscaling Policy` (`Scale Up Window`, `Scale Down Window`, `Scale To Zero Window`), `Base Model`.

### GPU hardware

| Accelerator | Notes |
|-------------|-------|
| `NVIDIA_A100_80GB` | Smaller, less expensive; cost-effective for low-volume use cases |
| `NVIDIA_H100_80GB` | Blazing fast; often highest throughput, especially for high-volume |
| `NVIDIA_H200_141GB` | Recommended for large models (DeepSeek V3/R1); min config 8 H200s for DeepSeek V3/R1 |
| MI300X (FAQ guidance) | Highest memory; unquantized Llama 3.1 70B fits on 1 MI300X; FP8 Llama 405B fits on 4 MI300Xs; more affordable than H100 |

GPU availability varies by region (`/deployments/regions`). Multiple GPUs per replica via `--accelerator-count N` (sub-linear scaling).

### Advanced topics (linked)

- Speculative decoding (`/deployments/speculative-decoding`) — draft models or n-gram speculation.
- Quantization (`/models/quantization`) — FP16→FP8 reduces costs 30-50%.
- Managing default deployments (`/deployments/managing-default-deployments`) — control which deployment handles model-name-only queries.
- Publishing deployments (`/deployments/publishing-deployments`) — make accessible to other Fireworks users.
- Reservations (`/deployments/reservations`) — guarantee availability during scale-up.

### Benchmarking

Open-source tool: https://github.com/fw-ai/benchmark
```
python benchmark.py \
  --model "accounts/fireworks/models/llama-v3p1-8b-instruct" \
  --deployment "your-deployment-id" \
  --num-requests 1000 \
  --concurrency 10
```
Key metrics: throughput (RPS), latency (TTFT, end-to-end), token generation rate, error rate. Best practices: warm up, test realistic scenarios, gradually increase load, monitor errors, compare configurations.

---

## 17. Autoscaling Configuration

**Source page:** `deployments/autoscaling`

### Configuration options

| Flag | Type | Default | Description |
|------|------|---------|-------------|
| `--min-replica-count` | Integer | 0 | Minimum replicas; 0 = scale-to-zero |
| `--max-replica-count` | Integer | 1 | Maximum replicas |
| `--scale-up-window` | Duration | 30s | Wait time before scaling up |
| `--scale-down-window` | Duration | 10m | Wait time before scaling down |
| `--scale-to-zero-window` | Duration | 1h (min 5m) | Idle time before scaling to zero |
| `--load-targets` | Key-value | `default=0.8` | Scaling thresholds |

### Load target options

Format: `--load-targets <key>=<value>[,<key>=<value>...]`

| Key | Description |
|-----|-------------|
| `default=<val>` | General load target (0–1) |
| `tokens_generated_per_second=<val>` | Desired tokens/sec per replica |
| `prompt_tokens_per_second=<val>` | Desired prompt tokens/sec per replica |
| `requests_per_second=<val>` | Desired requests/sec per replica |
| `concurrent_requests=<val>` | Desired concurrent requests per replica |

When multiple targets specified, the **maximum replica count across all** is used.

### Common patterns

| Pattern | Flags | Best for |
|---------|-------|----------|
| **Cost optimization** (scale to zero when idle) | `--min-replica-count 0 --max-replica-count 3 --scale-to-zero-window 1h` | Development, testing, intermittent production |
| **Performance-focused** (keep warm) | `--min-replica-count 2 --max-replica-count 10 --scale-up-window 15s --load-targets concurrent_requests=5` | Low-latency, avoiding cold starts, high-traffic |
| **Predictable traffic** | `--min-replica-count 3 --max-replica-count 5 --scale-down-window 30m --load-targets tokens_generated_per_second=150` | Steady workloads with known load ranges |

### Scaling from zero behavior

When a scaled-to-zero deployment receives a request, the system immediately returns a `503` with error code `DEPLOYMENT_SCALING_UP`:
```json
{
  "error": {
    "message": "Deployment is currently scaled to zero and is scaling up. Please retry your request in a few minutes.",
    "code": "DEPLOYMENT_SCALING_UP",
    "type": "error"
  }
}
```

- Requests are **NOT queued** — application must implement retry logic.
- Cold start times vary by model size.
- For instant responses, set `--min-replica-count 1` or higher.
- Deployments with min replicas = 0 auto-deleted after 7 days of no traffic.
- Reserved capacity guarantees availability during scale-up.

### Retry logic (scale-from-zero)

Exponential backoff: initial 5s, multiply by 1.5, cap 60s, max 30 retries. Pattern (Python):
```python
def query_deployment_with_retry(url, payload, max_retries=30, initial_delay=5):
    delay = initial_delay
    for attempt in range(max_retries):
        response = requests.post(url, json=payload, headers=headers)
        if response.status_code == 503:
            error_code = response.json().get("error", {}).get("code")
            if error_code == "DEPLOYMENT_SCALING_UP":
                time.sleep(delay)
                delay = min(delay * 1.5, 60)
                continue
        response.raise_for_status()
        return response.json()
    raise Exception("Deployment did not scale up in time")
```

---

## 18. LoRA Deployment (Live Merge & Multi-LoRA)

**Source page:** `fine-tuning/deploying-loras`

### Main concepts

- Fine-tuned LoRA models (created on Fireworks or imported) can **only** be deployed to **on-demand (dedicated) deployments** — Serverless is not supported.
- Two deployment methods: **live merge** and **multi-LoRA**.

| Method | Live merge | Multi-LoRA |
|--------|-----------|------------|
| How it works | LoRA weights merged into base model at deployment time | Base model deployed with addon support; adapters loaded dynamically at request time |
| Number of LoRAs | One per deployment | Multiple per deployment |
| Inference performance | Matches base model (no overhead) | Some overhead per request (dynamic adapter application) |
| Throughput | Same as base model | Lower max throughput under high concurrency |
| Cost efficiency | One deployment per fine-tune | Share a single deployment across many fine-tunes |
| Best for | Production requiring max performance | Experimentation, A/B testing, many variants |

### LoRA addon shape compatibility

**FP8 and FP4 quantized shapes do NOT support `--enable-addons`.**

| Precision | `--enable-addons` supported? |
|-----------|-------------------------------|
| BF16 | Yes |
| FP8 | No |
| FP4 | No |

For FP8/FP4 models needing LoRA addons: use a BF16 deployment shape, or merge the LoRA into a standalone model (live merge).

### Deploy with live merge

```
firectl deployment create "accounts/<acct>/models/<fine-tuned-model-id>"
```
Query directly: `model: "accounts/<acct>/models/<fine-tuned-model-id>"`.

### Deploy with multi-LoRA

1. Create base model deployment with addon support:
```
firectl deployment create "accounts/fireworks/models/<base-model-id>" --enable-addons
```
2. Load LoRA adapters:
```
firectl load-lora <FINE_TUNED_MODEL_ID> --deployment <DEPLOYMENT_ID>
```
3. Route requests to a specific adapter using the `#` separator:
   - `model: "accounts/<acct>/models/<ft-model-id>#accounts/<acct>/deployments/<dep-id>"`
   - The deprecated `deployedModel` request key is being phased out; use `model` with `#`.

### Performance considerations

- **TTFT:** increases ~10–30% (multi-LoRA) due to adapter loading + prompt processing overhead.
- **Generation speed:** overhead grows with higher request concurrency.
- **Maximum throughput:** lower than live-merge under sustained load.

---

## 19. Custom Model Upload (REST API)

**Source page:** `models/uploading-custom-models-api`

### Upload pipeline (four steps)

1. **Create the model** — register a model object referencing the files.
2. **Get upload URLs** — request a signed URL per file.
3. **Upload the files** — PUT each file to its signed URL.
4. **Validate the upload** — finalize; move model to `READY`. **Required** — until `validateUpload` is called, the model stays `UPLOADING` and cannot be deployed.

### API functions (control plane; base `https://api.fireworks.ai/v1`)

| Function | Method & Endpoint | Purpose |
|----------|-------------------|---------|
| Create model | `POST /v1/accounts/{ACCOUNT_ID}/models` | Register model object |
| Get upload endpoint | `POST /v1/accounts/{ACCOUNT_ID}/models/{MODEL_ID}:getUploadEndpoint` | Get signed URLs (body: `{filenameToSize, enableResumableUpload}`) |
| Upload file | `PUT <signed-url>` | Upload file directly (headers: `Content-Type: application/octet-stream`, `x-goog-content-length-range: {size},{size}`) |
| Validate upload | `GET /v1/accounts/{ACCOUNT_ID}/models/{MODEL_ID}:validateUpload` | Poll until `200` (model `READY`); `FAILED_PRECONDITION` = files still landing |

### Create model body

```json
{
  "modelId": "<MODEL_ID>",
  "model": {
    "kind": "HF_BASE_MODEL",
    "huggingFaceUrl": "https://huggingface.co/<org>/<repo>",
    "baseModelDetails": {
      "checkpointFormat": "HUGGINGFACE",
      "worldSize": 1,
      "huggingfaceFiles": ["<file1>", "<file2>", ...]
    }
  }
}
```

### Key behaviors

- `validateUpload` returns `FAILED_PRECONDITION` while files are still landing — poll until `200`.
- Every step is safe to retry. `create` and `getUploadEndpoint` no-op if the model already exists (`ALREADY_EXISTS`).
- Set `huggingFaceUrl` if the uploaded base model should be considered for Fireworks managed fine-tuning (tunability check runs async ~every 30 min).

---

## 20. Serving Paths — Standard, Priority, Fast

**Source pages:** `serverless/overview`, `serverless/serving-paths`

### Three serving paths (Serverless)

| Path | Opt-in mechanism | Use case | Pricing |
|------|-------------------|----------|---------|
| **Standard** | Default; no `service_tier` needed | General use | Base per-token |
| **Priority** | `service_tier: "priority"` on chat completions / Anthropic messages | Higher reliability during peak; less likely to be load-shed (503) | Premium |
| **Fast** | Different `model` ID (`accounts/fireworks/routers/<model-id>-fast`) | Latency-sensitive; targets 100+ tokens/sec generated | Higher price point |

### Key distinctions

- **Priority tier**: opt in via request **parameter** `service_tier`. Prioritized above Standard traffic. Available on select models (pricing page indicates availability). Standard + Priority share the **same rate limits** for a given model.
- **Fast**: opt in via **different model ID** (a router path). Quality remains the same. Fast and regular variants have **separate** rate limits.

### Fast model IDs

| Model | `model` ID |
|-------|------------|
| Kimi K2.6 Fast | `accounts/fireworks/routers/kimi-k2p6-fast` |
| GLM 5.2 Fast | `accounts/fireworks/routers/glm-5p2-fast` |
| GLM 5.1 Fast | `accounts/fireworks/routers/glm-5p1-fast` |

### Priority tier example

```bash
curl https://api.fireworks.ai/inference/v1/chat/completions \
  -H "Content-Type: application/json" \
  -H "Authorization: Bearer $FIREWORKS_API_KEY" \
  -d '{
    "model": "accounts/fireworks/models/glm-5p2",
    "service_tier": "priority",
    "messages": [{"role": "user", "content": "Hello"}]
  }'
```

---

## 21. Pricing & Billing

**Source page:** `serverless/pricing`

### Serverless pricing (per 1M tokens, USD)

- Text/vision requests billed across three dimensions: **input tokens**, **cached input tokens**, **generated tokens**.
- **Cached input default discount: 50% of input** on text and vision models (unless a model lists a different cached rate).
- **Batch inference: 50% of serverless pricing** on both input and output.
- **Embeddings: billed only on input tokens.**

### Text and vision models (Standard | Priority)

Format per cell: **input / cached input / output**. `—` = Priority not available for that model.

| Model | Standard | Priority |
|-------|----------|----------|
| Kimi K2.7 Code | $0.95 / $0.19 / $4.00 | $1.425 / $0.285 / $6.00 |
| Kimi K2.7 Code Fast | $1.90 / $0.38 / $8.00 | — |
| Kimi K2.6 | $0.95 / $0.16 / $4.00 | $1.50 / $0.22 / $6.00 |
| Kimi K2.6 Fast | $2.00 / $0.30 / $8.00 | — |
| DeepSeek V4 Pro | $1.74 / $0.145 / $3.48 | $2.61 / $0.218 / $5.22 |
| DeepSeek V4 Flash | $0.14 / $0.028 / $0.28 | $0.21 / $0.042 / $0.42 |
| GLM 5.2 | $1.40 / $0.14 / $4.40 | $1.75 / $0.18 / $5.50 |
| GLM 5.2 Fast | $2.10 / $0.21 / $6.60 | — |
| GLM 5.1 | $1.40 / $0.26 / $4.40 | $2.10 / $0.39 / $6.60 |
| GLM 5.1 Fast | $2.80 / $0.52 / $8.80 | — |
| Qwen 3.7 Plus | $0.40 / $0.08 / $1.60 | — |
| MiniMax M3 | $0.30 / $0.06 / $1.20 | $0.45 / $0.09 / $1.80 |
| MiniMax M2.7 | $0.30 / $0.06 / $1.20 | $0.45 / $0.09 / $1.80 |
| OpenAI GPT OSS 120B | $0.15 / $0.015 / $0.60 | $0.18 / $0.018 / $0.72 |
| OpenAI GPT OSS 20B | $0.07 / $0.035 / $0.30 | — |
| NVIDIA Nemotron 3 Ultra (Preview) | $0.60 / $0.12 / $2.40 | — |

### Other base models (size-based, per 1M tokens; input = output, no separate cached rate)

| Model size | $ / 1M tokens |
|------------|---------------|
| < 4B parameters | $0.10 |
| 4B – 16B parameters | $0.20 |
| > 16B parameters | $0.90 |
| MoE up to 56B (e.g. Mixtral 8x7B) | $0.50 |
| MoE 56.1B – 176B (e.g. DBRX, Mixtral 8x22B) | $1.20 |

### Embeddings (per 1M input tokens)

| Base model parameter count | $ / 1M input tokens |
|----------------------------|---------------------|
| up to 150M | $0.008 |
| 150M – 350M | $0.016 |
| Qwen3 8B | $0.10 |

### On-demand deployment pricing

- Billed per **GPU-second** (not per token). See https://fireworks.ai/pricing.
- Account-level controls (spend tiers, monthly budget, on-demand GPU quotas): `/guides/quotas_usage/account-quotas`.
- Quantization (FP16→FP8) reduces costs 30-50%.

### Spending tiers

Higher **Spending Tier** correlates with higher upper-bound rate limits and on-demand GPU quotas. Enterprise accounts get higher upper bounds automatically. Refs: `/guides/quotas_usage/account-quotas#spending-tiers`.

---

## 22. Rate Limits & Quotas

**Source page:** `serverless/rate-limits`

### Error behavior

- `429 Too Many Requests` and `503 Service Overloaded` possible.
- Stay below adaptive rate limits to avoid 429s.
- Upgrade to Priority tier to reduce likelihood of 503s.

### Three rate-limit metrics (per account, Serverless)

| Metric | Default ceiling | Approx TPS |
|--------|-----------------|------------|
| Total Prompt TPM (cached + uncached input tokens/min) | **21.6M** | ~360k |
| Uncached Prompt TPM | **5.4M** | ~90k |
| Generated TPM (output tokens/min) | **216k** | ~3.6k |

**Enforcement uses TPM, not TPS.**

### Reading current effective limits (response headers, TPS)

- `X-Ratelimit-Limit-Tokens-Prompt`
- `X-Ratelimit-Limit-Tokens-Cache-Adjusted-Prompt`
- `X-Ratelimit-Limit-Tokens-Generated`

### Adaptive behavior

- Adaptive rate limits grow and shrink with usage.
- If traffic ramps up too quickly → 429s.
- Adaptive limits have upper and lower bounds.
- Higher Spending Tier → higher upper bounds; Enterprise accounts get higher upper bounds automatically.

### Scoping

- Rate limits scoped **per account AND per model**.
- **Fast** and **regular** variants have **separate** limits.
- **Priority tier** and **regular** requests **share the same** rate limits for a given model.

### On-demand deployments

**No account-level rate limits.** A 429 indicates the deployment's processing capacity is saturated (queued + active requests exceed what the deployment's GPUs can handle) — a **capacity signal, not quota enforcement**. Resolutions: reduce burst concurrency, scale up the deployment, optimize request patterns (shorter prompts, fewer max tokens, batch). Contact Fireworks to increase capacity if consistent.

### Quota API (account management)

Account-scoped quota endpoints: `list-quotas`, `get-quota`, `update-quota` (under `/api-reference/`).

### Getting higher limits

Contact inquiries@fireworks.ai if:
- Need higher than defaults from day one (launch traffic exceeds starting limit).
- Already at highest Spending Tier and adaptive limits aren't growing.

---

## 23. Error Codes

**Source page:** `guides/inference-error-codes`

### HTTP error codes

| Code | Error Name | Possible Issue | How to Resolve |
|------|-----------|----------------|----------------|
| `400` | Bad Request | Invalid input or malformed request | Review request parameters and format |
| `401` | Unauthorized | Invalid API key or insufficient permissions | Verify API key and permissions |
| `402` | Payment Required | Account not on paid plan or exceeded usage limits | Check billing; upgrade plan |
| `403` | Forbidden | Authentication issues | Verify correct API key |
| `404` | Not Found | Endpoint path, model, deployment, or permissions | Verify URL path; check model exists/available; ensure permissions |
| `405` | Method Not Allowed | Unsupported HTTP method (e.g. GET instead of POST) | Check docs for correct method |
| `408` | Request Timeout | Request took too long (server overload/network) | Retry after brief wait; consider increasing timeout |
| `412` | Precondition Failed | Account suspended or LoRA model failed to load | Check account status/billing; for LoRA ensure correct upload/compatibility |
| `413` | Payload Too Large | Input data exceeds size limit | Reduce input payload size |
| `429` | Too Many Requests | Rate limited (serverless) or deployment capacity exceeded (on-demand) | See below |
| `500` | Internal Server Error | Server-side bug unlikely to self-resolve | Contact support immediately |
| `502` | Bad Gateway | Invalid response from upstream | Wait and retry; may indicate outage |
| `503` | Service Unavailable | Service down for maintenance or issues | Retry; check https://status.fireworks.ai |
| `504` | Gateway Timeout | No response in time from upstream | Wait briefly and retry; consider shorter prompt |
| `520` | Unknown Error | Unexpected error, no clear explanation | Retry; contact support if persistent |
| `425` | Too Early | (Rollout only) Sync weight swap in progress; new requests rejected | Retry with backoff, keep same session-affinity key |

### Special error code: `DEPLOYMENT_SCALING_UP`

Returned as a `503` body `error.code` when a scaled-to-zero deployment is scaling up. Requests are NOT queued — implement retry logic.

### 429 semantics

- **Serverless:** account exceeded current serverless request or TPM limit. Standard, Priority, and Fast all use the same public rate-limit policy combining request-rate limits with adaptive TPM limits. Resolutions: exponential backoff, smooth bursts, check rate-limit headers, review `/serverless/rate-limits`, or use on-demand deployment.
- **Dedicated/on-demand:** no account-level rate limits; 429 = deployment processing capacity saturated. Resolutions: reduce burst concurrency, scale up deployment, optimize request patterns.

### Status page & support

- Status: https://status.fireworks.ai
- Support: support@fireworks.ai
- Discord: https://discord.gg/fireworks-ai

---

## 24. Lifecycle, Compliance & Known Limitations

### Model lifecycle

- **Serverless models:** managed by Fireworks; may be updated/deprecated. At least **2 weeks advance notice** before removing any model (longer for popular models based on usage). For long-term stability, use on-demand deployments.
- **On-demand deployments:** full control over model versions/updates. Default scale-to-zero after 1 hour idle; auto-deleted after 7 days of no traffic (when min replicas = 0).

### Compliance

SOC 2, HIPAA; audit reports available.

### Known limitations summary

| Capability | Limit |
|-----------|-------|
| VLM images per request | 30 |
| VLM base64 total | < 10 MB |
| VLM image URL size | < 5 MB |
| VLM image URL download timeout | 1.5 s |
| VLM supported formats | .png, .jpg, .jpeg, .gif, .bmp, .tiff, .ppm |
| Llama 3.2 Vision | Images before text in content (temporary) |
| Video duration | ≤ 60 s recommended |
| Video format | .mp4 |
| Audio format | .ogg (Opus) |
| Video/audio payload total | < 10 MB |
| Video models | Dedicated deployment required (not serverless) |
| Batch input dataset | < 1 GB |
| Batch output dataset | < 8 GB |
| Batch job window | 12/24/48/72 hours |
| LoRA on Serverless | Not supported (on-demand only) |
| LoRA addons on FP8/FP4 | Not supported (BF16 only) |
| Embeddings input | Text only (no image input) |
| `/rerank` | Does not yet support all models/parallelism options |
| Responses API | `store=False` prevents `previous_response_id` continuation |

---

## 25. Capability Summary & Cross-Reference

### Endpoint ↔ capability matrix

| Capability | Endpoint | Serverless | On-Demand | Key params |
|-----------|----------|:----------:|:---------:|------------|
| Text generation | `POST /v1/chat/completions` | ✅ | ✅ | `model`, `messages`, `stream`, `max_tokens`, `temperature`, `top_p`, `top_k`, `n`, `logprobs`, `response_format`, `tools`, `service_tier`, `reasoning_effort` |
| Legacy completions | `POST /v1/completions` | ✅ | ✅ | `model`, `prompt`, `stream`, sampling, `include_routing_matrix`, `logprobs`, `echo` |
| Vision (images) | `POST /v1/chat/completions` | ✅ | ✅ | `content[]` with `image_url` parts; URL or base64; max 30 images |
| Video + audio (Omni) | `POST /v1/chat/completions` | ❌ | ✅ (required) | `content[]` with `video_url`/`audio_url`/`text`; base64 data URLs; deployment-required shape |
| Embeddings | `POST /v1/embeddings` | ✅ | ✅ | `input`, `model`, `dimensions`, `return_logits`, `normalize` |
| Reranking | `POST /v1/rerank` | ✅ | ✅ | `model`, `query`, `documents`, `top_n`, `return_documents` |
| Rerank via logits | `POST /v1/embeddings` | ✅ | ✅ | `return_logits=[no_id, yes_id]`, `normalize=true` |
| Responses API | `POST /v1/responses` | ✅ | ✅ | `model`, `input`, `previous_response_id`, `store`, `stream`, `tools`, `tool_choice`, `max_tool_calls` |
| Batch inference | `POST /v1/accounts/{acct}/batchInferenceJobs` | ✅ (50% off) | — | `model`, `inputDatasetId`, `outputDatasetId`, `inferenceParameters` |
| RL rollout | `POST /v1/chat/completions` (+headers) | — | ✅ | `x-multi-turn-session-id`, `x-session-affinity`, `include_routing_matrix`, `logprobs` |
| MoE Router Replay | `POST /v1/chat/completions` | — | ✅ | `include_routing_matrix: true`, `logprobs: true` |
| LoRA (live merge) | via deployment `model` | ❌ | ✅ | `model: accounts/<acct>/models/<ft-id>` |
| LoRA (multi-LoRA) | via `model#deployment` | ❌ | ✅ | `model: .../<ft-id>#.../<dep-id>`; `--enable-addons` (BF16 only) |

### Deployment-shape decision matrix

| Need | Recommended shape | Why |
|------|-------------------|-----|
| Lowest latency (interactive) | `fast` | Optimized for speed |
| High-volume, cost-per-token | `throughput` | Trades latency for scale efficiency |
| Testing / light workloads | `minimal` / `cost` | Lowest cost; for prototyping |
| Large models (DeepSeek V3/R1) | `NVIDIA_H200_141GB` (min 8 GPUs) | Recommended hardware for large models |
| LoRA multi-adapter | BF16 shape + `--enable-addons` | FP8/FP4 don't support addons |
| Scale-to-zero cost savings | `--min-replica-count 0 --scale-to-zero-window 1h` | Pay nothing when idle (auto-deleted after 7d no traffic) |
| Low-latency, no cold starts | `--min-replica-count >= 1` | Always warm; no 503 DEPLOYMENT_SCALING_UP |

### Serving-path decision matrix

| Need | Recommended path | Opt-in |
|------|-----------------|--------|
| General use, pay-per-token | Standard | Default (no `service_tier`) |
| Higher reliability during peak | Priority | `service_tier: "priority"` |
| Latency-sensitive (100+ tok/s) | Fast | Different `model` ID (`routers/...-fast`) |
| Dedicated capacity, no rate limits | On-Demand Deployment | `firectl deployment create` |
| Custom/LoRA models | On-Demand Deployment | Serverless doesn't support LoRA/custom |
| Async bulk, 50% off | Batch Inference | `batchInferenceJob` |

### Session/state decision matrix

| Scenario | Recommended approach |
|----------|---------------------|
| Simple multi-turn chat | Responses API `previous_response_id` with `store: true` |
| Stateless / data-retention-sensitive | Responses API `store: false` (no continuation) |
| RL rollout trajectory | `x-multi-turn-session-id` + `x-session-affinity` headers |
| Maximize prompt-cache hits | `x-session-affinity` or OpenAI `user` field |
| Weight swap preserving session KV | `reset_prompt_cache: "new_session"` per snapshot |
| Weight swap recomputing all KV | `reset_prompt_cache: "all"` (default) |
| Weight swap keeping previous namespace | `reset_prompt_cache: "none"` |

### SDK compatibility matrix

| SDK | Base URL override | Notes |
|-----|--------------------|-------|
| Fireworks Python (`pip install --pre fireworks-ai`, alpha) | (built-in) | Native; supports `echo`, `return_token_ids`, `raw_output`, `perf_metrics_in_response` |
| OpenAI Python / JS | `https://api.fireworks.ai/inference/v1` | Drop-in; pass Fireworks-specific params via `extra_body` |
| Anthropic Python / JS | `https://api.fireworks.ai/inference` | Drop-in; `max_tokens` required; `thinking`, `output_config` for reasoning/structured |
| `firectl` CLI | (control plane) | Deployment/dataset/batch/LoRA/model management |

---

### Sources

- Introduction — `https://docs.fireworks.ai/getting-started/introduction`
- Serverless Quickstart — `https://docs.fireworks.ai/getting-started/quickstart`
- On-Demand Quickstart — `https://docs.fireworks.ai/getting-started/ondemand-quickstart`
- Serverless Overview — `https://docs.fireworks.ai/serverless/overview`
- Serving Paths — `https://docs.fireworks.ai/serverless/serving-paths`
- Serverless Pricing — `https://docs.fireworks.ai/serverless/pricing`
- Rate Limits — `https://docs.fireworks.ai/serverless/rate-limits`
- On-Demand Deployments — `https://docs.fireworks.ai/guides/ondemand-deployments`
- Hardware Options — `https://docs.fireworks.ai/faq/deployment/ondemand/hardware-options`
- Autoscaling — `https://docs.fireworks.ai/deployments/autoscaling`
- Benchmarking — `https://docs.fireworks.ai/deployments/benchmarking`
- Querying Text Models — `https://docs.fireworks.ai/guides/querying-text-models`
- Querying Embeddings Models — `https://docs.fireworks.ai/guides/querying-embeddings-models`
- Querying Vision-Language Models — `https://docs.fireworks.ai/guides/querying-vision-language-models`
- Responses API — `https://docs.fireworks.ai/guides/response-api`
- Batch Inference — `https://docs.fireworks.ai/guides/batch-inference`
- Video & Audio Inputs — `https://docs.fireworks.ai/guides/video-audio-inputs`
- Inference Error Codes — `https://docs.fireworks.ai/guides/inference-error-codes`
- Rollout Inference — `https://docs.fireworks.ai/guides/rollout-inference`
- API Reference Introduction — `https://docs.fireworks.ai/api-reference/introduction`
- Deploying LoRAs — `https://docs.fireworks.ai/fine-tuning/deploying-loras`
- Upload via REST API — `https://docs.fireworks.ai/models/uploading-custom-models-api`
- Training API Introduction — `https://docs.fireworks.ai/fine-tuning/training-api/introduction`