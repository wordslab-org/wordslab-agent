# Nebius Token Factory — On-Demand LLM Inference API Analysis

> **Source:** Documentation pages reachable from [https://docs.tokenfactory.nebius.com/quickstart](https://docs.tokenfactory.nebius.com/quickstart)
>
> **Scope:** On-demand LLM inference services — both serverless API and dedicated (rented-GPU) endpoints. Sandboxes (code-execution environments) are covered as a complementary GPU-based capability.
>
> **Date of analysis:** 2026-07-14

---

## Table of Contents

1. [Platform Overview & Deployment Modes](#1-platform-overview--deployment-modes)
2. [Authentication & API Keys](#2-authentication--api-keys)
3. [Control Plane vs Data Plane](#3-control-plane-vs-data-plane)
4. [Serverless Inference — Chat Completions](#4-serverless-inference--chat-completions)
5. [Chat Completion Parameters](#5-chat-completion-parameters)
6. [Text Completions (Legacy)](#6-text-completions-legacy)
7. [Responses API](#7-responses-api)
8. [Embeddings](#8-embeddings)
9. [Rerank](#9-rerank)
10. [Image Generation](#10-image-generation)
11. [Function Calling & Tools](#11-function-calling--tools)
12. [Structured Output & JSON](#12-structured-output--json)
13. [Vision Inference](#13-vision-inference)
14. [Model Flavors & Inference Optimizations](#14-model-flavors--inference-optimizations)
15. [Rate Limits & Scaling](#15-rate-limits--scaling)
16. [Observability](#16-observability)
17. [Dedicated Endpoints (Reserved-Hardware Inference)](#17-dedicated-endpoints-reserved-hardware-inference)
18. [Dedicated Endpoint Lifecycle & Status](#18-dedicated-endpoint-lifecycle--status)
19. [Dedicated Endpoint Capacity, Billing & SLA](#19-dedicated-endpoint-capacity-billing--sla)
20. [Dedicated Endpoint Control-Plane API](#20-dedicated-endpoint-control-plane-api)
21. [Custom Model Weights](#21-custom-model-weights)
22. [Sandboxes (Cloud Code Execution with Branching)](#22-sandboxes-cloud-code-execution-with-branching)
23. [OpenAI Compatibility](#23-openai-compatibility)
24. [API Endpoint Reference Summary](#24-api-endpoint-reference-summary)

---

## 1. Platform Overview & Deployment Modes

**Source:** [AI Models Inference Overview](https://docs.tokenfactory.nebius.com/ai-models-inference/overview), [Dedicated Endpoints Overview](https://docs.tokenfactory.nebius.com/ai-models-inference/dedicated-endpoints/overview)

Nebius Token Factory is an OpenAI-compatible inference and fine-tuning platform running on Nebius GPU infrastructure. It offers two distinct deployment models for on-demand LLM inference:

| Mode | Description | Best for | Pricing |
|------|-------------|----------|---------|
| **Public serverless endpoints** | Shared multi-tenant capacity, per-token API, dynamic rate limits, platform-managed scaling | Prototyping, variable-traffic apps | Per token |
| **Dedicated endpoints** | Isolated GPU capacity reserved for your org, fixed region, user-controlled autoscaling, custom-weights support | Fine-tuned models, latency/cost control, data residency | Per GPU/hour (per-minute granularity) |

Beyond inference, the platform exposes **Sandboxes** — cloud code-execution environments with Git-like branching for AI agents (covered in §22) and a full **post-training** (fine-tuning) stack.

### Core API endpoint (shared across serverless + dedicated)

```
POST https://api.tokenfactory.nebius.com/v1/chat/completions
```

- **Auth:** `Authorization: Bearer $NEBIUS_API_KEY`
- **Content-Type:** `application/json`
- **OpenAI-compatible:** Drop-in replacement for OpenAI clients (change `api_key` + `base_url`).
- **Model identifier differs by mode:**
  - Serverless: `model="meta-llama/Meta-Llama-3.1-70B-Instruct"` (Hugging Face-style ID, optionally `-fast` suffix)
  - Dedicated: `model="<routing_key>"` (the routing key returned by the control plane at endpoint creation)

### Supported model types

- **Text-to-text** (chat/instruct LLMs)
- **Embedding** models
- **Vision** (multimodal) models
- The full catalog is browsable in the [Token Factory UI](https://tokenfactory.nebius.com/) and via the `GET /v1/models` endpoint.

---

## 2. Authentication & API Keys

**Source:** [API Reference Introduction](https://docs.tokenfactory.nebius.com/api-reference/introduction)

### Authentication method

Every request must include the API key in the `Authorization` header:

```
Authorization: Bearer $NEBIUS_API_KEY
Content-Type: application/json
```

### Key concepts

- Keys are created in the UI at [API keys settings](https://tokenfactory.nebius.com/project/api-keys) — no documented programmatic key CRUD API.
- A key is **shown only once** at creation and cannot be retrieved later; store it immediately in a secrets manager.
- Nebius can **automatically revoke** a compromised key.
- Keep keys server-side; never expose them in client-side code.

### Quickstart example

```python
from openai import OpenAI

client = OpenAI(
    base_url="https://api.tokenfactory.nebius.com/v1/",
    api_key=os.environ["NEBIUS_API_KEY"],
)
completion = client.chat.completions.create(
    model="meta-llama/Meta-Llama-3.1-70B-Instruct",
    messages=[{"role": "user", "content": "Hello!"}],
    temperature=0.6,
)
```

---

## 3. Control Plane vs Data Plane

**Source:** [Control & Data Plane](https://docs.tokenfactory.nebius.com/ai-models-inference/dedicated-endpoints/control-data-plane)

Dedicated endpoints split management and inference across two planes with different base URLs. This is a key architectural concept.

| Term | Description |
|------|-------------|
| **Template** | A deployable performance "blueprint" for a model — defines which `flavor_name`, `gpu_type`, and regions are supported. |
| **Flavor** | A template's sub-option (e.g. `base`, `fast`) with different performance/throughput/cost characteristics. |
| **Endpoint** | A dedicated deployment with API access. |
| `endpoint_id` | Identifier used for update/delete operations (control plane). |
| `routing_key` | The model identifier you pass to inference calls (data plane). Returned at endpoint creation. |
| **Control plane** | Sets up configuration & settings. Single common base URL. |
| **Data plane** | Processes inference requests. Regional base URLs. |

### Base URLs

| Plane | Base URL |
|-------|----------|
| Control plane (all endpoint management) | `https://api.tokenfactory.nebius.com` |
| Data plane — `eu-north1` | `https://api.tokenfactory.nebius.com` |
| Data plane — `eu-west1` | `https://api.tokenfactory.eu-west1.nebius.com` |
| Data plane — `us-central1` | `https://api.tokenfactory.us-central1.nebius.com` |

Using the region-appropriate data-plane URL avoids global routing and reduces latency. Region also impacts data locality and regulatory compliance.

---

## 4. Serverless Inference — Chat Completions

**Source:** [Create chat completion](https://docs.tokenfactory.nebius.com/api-reference/inference/create-chat-completion)

### Capability

Query chat/instruct models with single prompts, multi-turn conversations, system prompts, tools, and structured output. Conversations are **stateless** — pass the full `messages` array each call.

### API endpoint

```
POST https://api.tokenfactory.nebius.com/v1/chat/completions
```

SDK method: `client.chat.completions.create(...)`

### Common parameters (shared across all inference endpoints)

| Parameter | Type | Description |
|-----------|------|-------------|
| `ai_project_id` | string \| null | Query param — project ID scoping the request. |
| `service_tier` | enum | `auto` (default), `default`, `over-limit`, `flex`, `no-limit` — controls rate-limit/over-limit behavior. |

### `service_tier` semantics

| Value | Behavior |
|-------|----------|
| `auto` | Picks the best tier automatically (default). |
| `default` | Returns HTTP 429 when the rate limit is exceeded. |
| `over-limit` | Response-only tier (lower priority); set by the platform when over limit. |
| `flex` | Does not consume rate-limit credits but has lower priority. |
| `no-limit` | Uncapped. |

### Response structure (non-streaming)

```json
{
  "id": "chatcmpl-...",
  "object": "chat.completion",
  "created": 1717516032,
  "model": "meta-llama/Meta-Llama-3-70B-Instruct-cheap",
  "choices": [
    {
      "finish_reason": "stop",
      "index": 0,
      "message": { "role": "assistant", "content": "Hello!..." }
    }
  ],
  "usage": { "completion_tokens": 26, "prompt_tokens": 13, "total_tokens": 39 },
  "service_tier": "auto"
}
```

Access via `response.choices[0].message.content`.

### Streaming

Set `stream: true`. Returns Server-Sent Events with JSON-encoded chunks:
- Object type: `"chat.completion.chunk"`
- `delta.content` carries partial content
- `finish_reason` is `null` during streaming, set at the end
- Use `stream_options: {"include_usage": true}` to receive usage stats in the final chunk

---

## 5. Chat Completion Parameters

**Source:** [Create chat completion](https://docs.tokenfactory.nebius.com/api-reference/inference/create-chat-completion)

### Request parameters

| Parameter | Type | Description | Default / Constraint |
|-----------|------|-------------|----------------------|
| `model` | string | Model ID (e.g. `meta-llama/Meta-Llama-3.1-70B-Instruct`) | Required |
| `messages` | array | Conversation messages: `{content, role, name, tool_calls, tool_call_id, reasoning_content, reasoning}` | Required |
| `store` | boolean | Store output for model distillation | Default `false` |
| `max_tokens` | integer | Max generated tokens (prompt + max_tokens ≤ context length) | Default `100`, ≥ 0 |
| `max_completion_tokens` | integer | Upper bound incl. visible + reasoning tokens | Default `100`, ≥ 0 |
| `temperature` | number | Sampling temperature | Default `1`, `0–2` |
| `top_p` | number | Nucleus sampling | Default `1`, `0–1` |
| `tools` | array | Tool definitions `{function:{name,description,parameters,strict}, type:"function"}` | Optional |
| `tool_choice` | string / object | `"none"/"minimal"/"low"/"medium"/"high"/"xhigh"` or explicit tool selection | Optional |
| `n` | integer | Completions per prompt | Default `1`, `1–128` |
| `stream` | boolean | Stream partial deltas via SSE | Default `false` |
| `stream_options` | object \| null | `{include_usage: true}` to emit usage in the last chunk | Default `null` |
| `stop` | string / array \| null | Up to 4 stop sequences | Default `null` |
| `presence_penalty` | number | Penalize tokens already present | Default `0`, `-2..2` |
| `frequency_penalty` | number | Penalize frequent tokens | Default `0`, `-2..2` |
| `logit_bias` | object \| null | Map token IDs to bias `-100..100` | Default `null` |
| `logprobs` | boolean | Return log probabilities of output tokens | Default `false` |
| `top_logprobs` | integer \| null | N most likely tokens per position (requires `logprobs=true`) | Default `null`, ≥ 0 |
| `user` | string \| null | End-user identifier | Default `null` |
| `response_format` | object \| null | `{"type":"json_object"}` / `{"type":"json_schema",...}` / `{"type":"text"}` | Default `null` |
| `extra_body` | object \| null | Extra parameters (vLLM pass-through) | Default `null` |
| `service_tier` | enum | See §4 | Default `auto` |

### Message roles

| Role | Purpose |
|------|---------|
| `user` | Message from the end user |
| `assistant` | Model's prior response (carries `tool_calls`, `reasoning_content`, `reasoning`) |
| `system` | System/developer instructions |
| `tool` | Tool-call result (carries `tool_call_id`, `name`) |

### API vs Playground parameter coverage

- **Playground** supports the most commonly used parameters.
- **API** supports the **full set of vLLM parameters** (exposed via `extra_body`).

---

## 6. Text Completions (Legacy)

**Source:** [Create completion](https://docs.tokenfactory.nebius.com/api-reference/inference/create-completion)

### Capability

OpenAI-compatible legacy completions API — generates text from a raw prompt rather than a chat message array. Useful for non-chat models and code autocomplete.

### API endpoint

```
POST https://api.tokenfactory.nebius.com/v1/completions
```

### Request parameters (differences from chat completions)

| Parameter | Type | Description | Default / Constraint |
|-----------|------|-------------|----------------------|
| `model` | string | Model ID | Required |
| `prompt` | string / string[] / integer[] | Prompt(s) as string, array of strings, token IDs, or arrays of token IDs | Required |
| `echo` | boolean | Echo the prompt back with the completion | Default `false` |
| `logprobs` | integer \| null | N most likely tokens + the chosen token per position | Default `null` |
| `max_tokens` | integer \| null | Max completion token count | Example `100` |
| `temperature`, `top_p`, `n`, `stop`, `presence_penalty`, `frequency_penalty`, `logit_bias`, `user`, `stream`, `stream_options`, `extra_body`, `service_tier` | — | Same semantics as chat completions | — |

### Response structure

```json
{
  "id": "...",
  "object": "text_completion",
  "created": 1717516032,
  "model": "...",
  "choices": [{ "finish_reason": "stop", "index": 0, "text": "..." }],
  "usage": { "completion_tokens": 26, "prompt_tokens": 13, "total_tokens": 39 },
  "service_tier": "auto"
}
```

A dedicated **"Text generation for code autocomplete"** guide and example exist in the cookbook for FIM (fill-in-the-middle) usage.

---

## 7. Responses API

**Source:** [Create a response](https://docs.tokenfactory.nebius.com/api-reference/inference/create-a-response)

### Capability

The OpenAI **Responses API** — a richer, stateful, multi-modal endpoint supporting reasoning models, built-in tools (file search, code interpreter, web search, computer use, MCP), conversation chaining via `previous_response_id`, and background execution. This is the most feature-complete inference surface.

### API endpoint

```
POST https://api.tokenfactory.nebius.com/v1/responses
```

### Request parameters (key fields)

| Parameter | Type | Description | Default / Constraint |
|-----------|------|-------------|----------------------|
| `input` | string / object[] | Text / image / file inputs — a union of item types (messages, tool calls, reasoning, function outputs, MCP calls, etc.) | Required |
| `model` | string | Model used for the completion | Required |
| `background` | boolean \| null | Run the response in the background | Optional |
| `include` | enum[] \| null | Extra output data to return: `code_interpreter_call.outputs`, `computer_call_output.output.image_url`, `file_search_call.results`, `message.input_image.image_url`, `message.output_text.logprobs`, `reasoning.encrypted_content` | Optional |
| `instructions` | string \| null | System / developer message | Optional |
| `max_output_tokens` | integer \| null | Upper bound incl. visible + reasoning tokens | Optional |
| `max_tool_calls` | integer \| null | Max total built-in tool calls | Optional |
| `metadata` | object \| null | Up to 16 key-value pairs | Optional |
| `parallel_tool_calls` | boolean \| null | Allow parallel tool calls | Optional |
| `previous_response_id` | string \| null | Prior response ID — conversation chaining | Optional |
| `prompt` | object \| null | Prompt template ref `{id, variables, version}` | Optional |
| `reasoning` | object \| null | Reasoning model config (gpt-5/o-series style) | Optional |
| `store` | boolean \| null | Store the response for later retrieval | Optional |
| `stream` | boolean \| null | SSE streaming | Optional |
| `temperature` | number \| null | Sampling temperature | Default `1`, `0–2` |
| `text` | object \| null | Text response config `{format:{type}}` | Optional |
| `tool_choice` | enum / object | `none` / `auto` / `required` or a tool-choice object | Default `auto` |
| `tools` | array | Union of tool types: `FunctionTool`, `FileSearchTool`, `ComputerTool`, `WebSearchTool`, `Mcp`, `CodeInterpreter`, `ImageGeneration`, `LocalShell`, `FunctionShellTool`, `CustomTool`, `NamespaceTool`, `ToolSearchTool`, `WebSearchPreviewTool`, `ApplyPatchTool`. Function tool attrs: `{name, type, parameters, strict, defer_loading, description}` | Optional |
| `top_logprobs` | integer \| null | N most likely tokens per position | ≥ 0 |
| `top_p` | number \| null | Nucleus sampling | `0–1` |
| `truncation` | enum | `auto` / `disabled` | Default `disabled` |
| `user` | string \| null | End-user identifier | Optional |
| `prompt_cache_key` | string \| null | Cache key (replaces `user`) | Optional |
| `service_tier` | enum | See §4 | Default `auto` |

### Response structure

```json
{
  "id": "resp_...",
  "object": "response",
  "created_at": 1717516032,
  "model": "...",
  "status": "completed",            // completed/failed/in_progress/cancelled/queued/incomplete
  "output": [ /* ResponseOutputMessage, tool calls, reasoning items, MCP calls, image gen, code interpreter, shell/patch calls */ ],
  "error": null,
  "instructions": "...",
  "metadata": {},
  "max_output_tokens": 4096,
  "max_tool_calls": 20,
  "previous_response_id": null,
  "tools": [],
  "tool_choice": "auto",
  "parallel_tool_calls": true,
  "background": false,
  "reasoning": null,
  "text": { "format": { "type": "text" } },
  "top_logprobs": null,
  "top_p": 1.0,
  "temperature": 1.0,
  "truncation": "disabled",
  "service_tier": "auto",
  "user": null,
  "usage": {
    "input_tokens": 13,
    "input_tokens_details": { "cached_tokens": 0, "input_tokens_per_turn": 13, "cached_tokens_per_turn": 0 },
    "output_tokens": 26,
    "output_tokens_details": { "reasoning_tokens": 0, "tool_output_tokens": 0, "output_tokens_per_turn": 26, "tool_output_tokens_per_turn": 0 },
    "total_tokens": 39
  }
}
```

The `output` array contains `ResponseOutputMessage` items with `content[{annotations, text, type, logprobs}]`, plus tool-call, reasoning, image-generation, code-interpreter, shell, and patch items depending on the request.

---

## 8. Embeddings

**Source:** [Create embeddings](https://docs.tokenfactory.nebius.com/api-reference/inference/create-embeddings)

### Capability

Generate embedding vectors for text inputs, for use in retrieval, clustering, and similarity tasks.

### API endpoint

```
POST https://api.tokenfactory.nebius.com/v1/embeddings
```

### Request parameters

| Parameter | Type | Description | Default / Constraint |
|-----------|------|-------------|----------------------|
| `model` | string | Embedding model ID (e.g. `BAAI/bge-en-icl`) | Required |
| `input` | string / array | Text or tokens to embed | Required |
| `encoding_format` | string | `float` or `base64` | Default `float` |
| `dimensions` | integer | Embedding dimensions to use | Example `4096` |
| `user` | string | End-user identifier | Optional |
| `service_tier` | enum | See §4 | Default `auto` |

### Response structure

```json
{
  "object": "list",
  "model": "BAAI/bge-en-icl",
  "data": [
    { "object": "embedding", "embedding": [0.1, ...], "index": 0 }
  ],
  "usage": { "prompt_tokens": 5, "total_tokens": 5 },
  "service_tier": "auto"
}
```

---

## 9. Rerank

**Source:** [Rerank documents](https://docs.tokenfactory.nebius.com/api-reference/inference/rerank-documents)

### Capability

Rerank a list of documents by relevance to a query — a key building block for retrieval-augmented generation (RAG) pipelines.

### API endpoint

```
POST https://api.tokenfactory.nebius.com/v1/rerank
```

### Request parameters

| Parameter | Type | Description | Default / Constraint |
|-----------|------|-------------|----------------------|
| `model` | string | Reranker model ID (e.g. `Qwen/Qwen3-Reranker-8B`) | Required |
| `query` | string | Query to rerank documents against | Required |
| `documents` | string[] | Documents to rerank | Required |
| `user` | string \| null | End-user identifier | Optional |
| `service_tier` | enum | See §4 | Default `auto` |

### Response structure

```json
{
  "id": "rerank-...",
  "model": "Qwen/Qwen3-Reranker-8B",
  "results": [
    { "index": 2, "document": { "text": "Paris is the capital..." }, "relevance_score": 0.98 }
  ],
  "usage": { "prompt_tokens": 120, "total_tokens": 120 }
}
```

Results are sorted by descending `relevance_score`.

---

## 10. Image Generation

**Source:** [Generate](https://docs.tokenfactory.nebius.com/api-reference/inference/generate)

### Capability

Generate images from a text prompt using diffusion models (e.g. Black Forest Labs FLUX). Supports LoRA adapters, configurable dimensions, and multiple output formats.

### API endpoint

```
POST https://api.tokenfactory.nebius.com/v1/images/generations
```

### Request parameters

| Parameter | Type | Description | Default / Constraint |
|-----------|------|-------------|----------------------|
| `model` | string | Diffusion model ID (e.g. `black-forest-labs/flux-schnell`) | Required |
| `prompt` | string | Text description of the desired image(s) | Required, length `1–2000` |
| `loras` | object[] \| null | List of publicly accessible LoRAs, each `{scale, url}` | 1–10 elements |
| `width` | integer | Image width in pixels | Default `512`, `64–2048` |
| `height` | integer | Image height in pixels | Default `512`, `64–2048` |
| `num_inference_steps` | integer \| null | Denoising steps (higher = better quality, slower) | `1–80` |
| `seed` | integer | Random seed (`-1` = random) | Default `-1`, ≥ -1 |
| `guidance_scale` | number \| null | Guidance scale (higher = adheres more to prompt) | `0–100` |
| `negative_prompt` | string | Description of non-desired image(s) | Default `""`, max `2000` |
| `response_extension` | enum | `webp` / `jpg` / `png` | Default `webp` |
| `response_format` | enum | `url` or `b64_json` | Default `url` |

### Response structure

```json
{
  "data": [{ "url": "https://..." }],   // or { "b64_json": "..." }
  "id": "img-..."
}
```

---

## 11. Function Calling & Tools

**Source:** [Function calling & Tools](https://docs.tokenfactory.nebius.com/ai-models-inference/function-calling)

### Capability

Let a model select and call external functions (tools) to extend its capabilities. Define available tools, the model determines intent and emits a `tool_call`, you execute the tool and return the result, then iterate.

### Workflow

1. **Define available tools** — pass `tools` with JSON-schema `parameters` (Pydantic `model_json_schema()` or Zod `zodResponseFormat(...).json_schema.schema`).
2. **Model determines intent** — returns `message.tool_calls` with `id`, `function.name`, `function.arguments` (JSON string).
3. **Execute and iterate** — run the tool, append a `role: "tool"` message with `content`, `tool_call_id`, and `name`, then call again.

### `tool_choice` modes

| Mode | Behavior |
|------|----------|
| `"auto"` | Model decides whether to call a tool |
| `"none"` | Never call a tool |
| `{"type":"function","function":{"name":"<fn>"}}` | Force a specific function |
| `"minimal"/"low"/"medium"/"high"/"xhigh"` | Graduated tool-call eagerness |

### Example (Python)

```python
tools = [{
    "type": "function",
    "function": {
        "name": "get_current_weather",
        "description": "Get the current weather in a given location",
        "parameters": GetCurrentWeatherParams.model_json_schema()
    }
}]

chat = client.chat.completions.create(
    model="meta-llama/Meta-Llama-3.1-8B-Instruct-fast",
    messages=messages,
    tools=tools,
    tool_choice={"type": "function", "function": {"name": "get_current_weather"}}
)
```

The model responds with:

```json
{
  "role": "assistant",
  "tool_calls": [{
    "id": "chatcmpl-tool-...",
    "type": "function",
    "function": {
      "name": "get_current_weather",
      "arguments": "{ \"city\": \"Dallas\", \"state\": \"Texas\", \"unit\": \"fahrenheit\" }"
    }
  }]
}
```

### Limitations

Tool calling support is model-dependent; check model cards for compatibility. The Responses API (§7) exposes a much wider set of **built-in** tools (file search, code interpreter, web search, computer use, MCP) in addition to user-defined functions.

---

## 12. Structured Output & JSON

**Source:** [Structured output & JSON](https://docs.tokenfactory.nebius.com/ai-models-inference/json)

### Capability

Force the model to respond with JSON instead of natural language, for easier programmatic consumption.

### Two modes via `response_format`

| `response_format.type` | Behavior |
|------------------------|----------|
| `json_schema` | JSON **following a schema** you provide (JSON Schema Specification compliant). Best for strict structure. |
| `json_object` | Arbitrary valid JSON object — the model decides the schema. |

### Recommended workflow

1. **Choose response format type** — strict `json_schema` vs arbitrary `json_object`.
2. **Optionally provide a JSON schema** — JSON Schema Specification compliant.
3. **Add instructions to the system/user prompt** — best practice is to describe the schema in the prompt *and* pass it via the parameter.
4. **Test several models** — structured-output capability varies; look for the `JSON mode` tag on model cards.

### JSON-schema example

```python
completion = client.chat.completions.create(
    model="Qwen/Qwen3-235B-A22B",
    response_format={
        "type": "json_schema",
        "json_schema": Film.model_json_schema()   # Pydantic BaseModel
    },
    messages=[
        {"role": "system", "content": "Respond with film details in the provided structure."},
        {"role": "user", "content": "Jack Nicholson"}
    ],
)
output = completion.choices[0].message
if output.refusal:        # models may refuse (e.g. safety)
    print(output.refusal)
else:
    print(json.loads(output.content))
```

A schema can include `"strict": true` and `"additionalProperties": false` for the strictest enforcement.

### Arbitrary JSON object example

```python
completion = client.chat.completions.create(
    model="openai/gpt-oss-120b",
    messages=[{"role": "system", "content": "Respond with film details using JSON."}, ...],
    response_format={"type": "json_object"}
)
```

Use `json_schema` over `json_object` whenever a strict structure is required.

---

## 13. Vision Inference

**Source:** [Vision capabilities](https://docs.tokenfactory.nebius.com/api-reference/examples/vision-capabilities)

### Capability

Multimodal models accept image inputs alongside text. Vision is supported through the chat completions and Responses APIs by passing image content in the `messages`/`input` array — either as a URL or a base64-encoded image. The Responses API exposes a dedicated `include` value `message.input_image.image_url` for retrieving input-image data in responses.

---

## 14. Model Flavors & Inference Optimizations

**Source:** [AI Models Inference Overview](https://docs.tokenfactory.nebius.com/ai-models-inference/overview)

### Model flavors

Inference performance is determined by how the endpoint is configured — balancing batch size, GPU type (L40, H100, H200, B200), GPU count, and inference-time optimizations. Two flavors are offered:

| Flavor | Description | How to use |
|--------|-------------|------------|
| **Base** | Standard throughput/cost balance | Default model name |
| **Fast** | Lower latency via smaller batch sizes, more compute, and speculative decoding | Append `-fast` to the model name |

Both deliver **identical model outputs**; differences are token pricing, latency, and optimization level. For stricter latency/cost/throughput targets, endpoint configurations can be tailored via collaboration with the sales/solutions engineering team.

### Inference optimizations

| Technique | Description |
|-----------|-------------|
| **KV Cache** | Stores frequently accessed key-value pairs, reducing recomputation. |
| **Paged Attention** | Splits input sequences into chunks processed separately, reducing memory/computation. |
| **Flash Attention** | Modified attention mechanism reducing the number of computations for attention. |
| **Quantization** | Reduces precision of weights/activations, decreasing memory and computation. |
| **Continuous Batching** | Batches multiple input sequences together, increasing throughput. |
| **Context Caching** | Caches context (outputs of previous layers) per sequence. |
| **Speculative Decoding** | Auxiliary models pre-generate likely next tokens; the main model verifies, reducing forward passes. |

### Impact on model quality

Optimizations are designed to retain ~99% of the original model's quality. Each technique's impact is evaluated and monitored so the cumulative effect does not compromise quality.

---

## 15. Rate Limits & Scaling

**Source:** [Rate Limits & Scaling](https://docs.tokenfactory.nebius.com/ai-models-inference/rate-limits)

### How it works

Rate limits are **dynamic**. The platform auto-scales the ceiling as you consistently use capacity, and scales it back when traffic falls off. Defaults are visible in the UI at Rate Limits settings.

Evaluation uses rolling **15-minute buckets**:

| Rule | Condition | Effect on next window |
|------|-----------|------------------------|
| **Scale-up** | Average usage ≥ 80% of current limit | Limit increases × 1.2 |
| **Scale-down** | Average usage ≤ 50% | Limit decreases ÷ 1.5 |
| **Hard ceiling** | — | Limit can grow up to **20×** base allocation; beyond that requires Enterprise |

### Example scaling (sustained ≥ 80% utilization)

| Window | Scale factor | RPM limit | TPM limit |
|--------|-------------|-----------|-----------|
| Baseline | `1.00×` | 60 | 400,000 |
| +15 min | `1.20×` | 72 | 480,000 |
| +30 min | `1.44×` | 86 | 576,000 |
| +1 h | `2.07×` | 124 | 828,000 |
| +2 h | `4.30×` | 258 | 1,720,000 |

### Handling 429s

Exceeding the active limit returns **HTTP 429**. Track consumption against limits via response headers. Over-the-limit requests may still be processed when spare capacity exists (lower priority) — these include `x-ratelimit-over-limit: yes` as an early warning.

### Rate-limit response headers

| Header | Description |
|--------|-------------|
| `x-ratelimit-limit-requests` | Max requests per minute |
| `x-ratelimit-limit-tokens` | Max tokens per minute |
| `x-ratelimit-remaining-requests` | Remaining requests before limit |
| `x-ratelimit-remaining-tokens` | Remaining tokens (replenished continuously) |
| `x-ratelimit-reset-requests` | Seconds until request limit resets |
| `x-ratelimit-reset-tokens` | Seconds until token limit resets |
| `x-ratelimit-dynamic-scale-requests` | Current dynamic scale factor (requests) |
| `x-ratelimit-dynamic-scale-tokens` | Current dynamic scale factor (tokens) |
| `x-ratelimit-dynamic-period-remaining` | Seconds until next dynamic adjustment |
| `x-ratelimit-dynamic-period-usage-requests` | Average request usage this period (%) |
| `x-ratelimit-dynamic-period-usage-tokens` | Average token usage this period (%) |
| `x-ratelimit-over-limit` | `yes` if an over-limit request was processed |
| `Retry-After` | Seconds to wait before retrying |

**Enterprise tier** removes soft caps, provides dedicated capacity, and an SLA. The **Batch API** offers significantly higher limits for async workloads.

---

## 16. Observability

**Source:** [Inference Observability](https://docs.tokenfactory.nebius.com/ai-models-inference/observability)

### Capability

Real-time and historical metrics for inference workloads — latency, throughput, scaling, and error rates — for both public and dedicated endpoints. Accessed via **Navigation Bar → Inference → Observability**.

### Metrics available

**Traffic**

| Metric | Description |
|--------|-------------|
| Requests per minute | Requests sent to the API |
| Input tokens per minute | Incoming tokens processed |
| Output tokens per minute | Tokens generated |
| Input tokens per request | Distribution of prompt sizes |
| Output tokens per request | Distribution of response sizes |

**Latency** (percentiles `p50`, `p90`, `p99`)

| Metric | Description |
|--------|-------------|
| End-to-end latency | Request sent → full response received |
| TTFT (Time to First Token) | Time until first token generated |
| Output speed (TPS) | Tokens generated per second |

**Autoscaling & Capacity**

| Metric | Description |
|--------|-------------|
| Active replicas | Currently running replicas (capacity metrics may be hidden depending on pricing) |

**Errors & success rate**

| Metric | Description |
|--------|-------------|
| Error rate by status code | Failed requests grouped by HTTP status (4xx, 429, 5xx) |

### Filters & dimensions

Time range, model endpoint, project, API key, region, error code, prompt length, latency range — apply to all charts simultaneously.

### Exporting metrics & API access

- Preconfigured dashboards in the web UI.
- Metrics available via **Prometheus** and **Grafana®** integrations for custom filtering/visualization.

### Access control

Observability is project-nested and follows project permissions. Organization Admins and Project Admins/Members have view access; Billing Managers do not.

### Data characteristics

- Near-real-time updates (tens of seconds), percentile-based latency, rolling aggregation windows.
- Dashboards are for operational debugging, not billing reconciliation.
- Metrics are collected in the inference region but stored at `eu-north`.
- Observability data is deleted when its project is deleted.

---

## 17. Dedicated Endpoints (Reserved-Hardware Inference)

**Source:** [Dedicated Endpoints Overview](https://docs.tokenfactory.nebius.com/ai-models-inference/dedicated-endpoints/overview), [Quickstart: Deploy via API](https://docs.tokenfactory.nebius.com/ai-models-inference/dedicated-endpoints/deploy-api)

### Capability

Deploy a model on GPU capacity **reserved exclusively for your organization**. You control region, GPU type/count, scaling (min/max replicas), and can attach custom weights. Inference uses the same OpenAI-compatible data plane as serverless, but billed per GPU/hour with per-minute granularity rather than per token.

### Dedicated vs Public endpoints

| Feature | Dedicated Endpoints | Public Serverless Endpoints |
|---------|---------------------|------------------------------|
| Capacity | Isolated capacity reserved for your org | Shared multi-tenant |
| Rate limits | No standard rate limits; throughput depends on deployed capacity | Dynamic rate limits apply |
| Data residency | Fixed, user-selected region | Region may vary by capacity |
| Autoscaling | You control min/max replicas | Platform-managed with predefined limits |
| Custom weights | Supported for eligible models | Base models only |
| Pricing | Per GPU/hour, per-minute granularity | Per token |

### Deploy via API — three steps

1. **List available model templates** to find valid `model_name` + `flavor_name` + `gpu_type` + `region` combinations.
2. **Create a dedicated endpoint** — returns `endpoint_id` (for management) and `routing_key` (for inference).
3. **Send inference requests** to the OpenAI-compatible data plane, using the `routing_key` as the `model` value.

### Step 1 — List templates

```
GET https://api.tokenfactory.nebius.com/v0/dedicated_endpoints/templates
```

The template response is the **source of truth** for valid combinations of `model_name`, `flavor_name`, `gpu_type`, `gpu_count`, and `region`.

### Step 2 — Create endpoint

```
POST https://api.tokenfactory.nebius.com/v0/dedicated_endpoints
```

```json
{
  "name": "GPT-20B Endpoint",
  "description": "Dedicated GPT-20B for internal apps",
  "model_name": "openai/gpt-oss-20b",
  "flavor_name": "base",
  "gpu_type": "gpu-h100-sxm",
  "gpu_count": 1,
  "region": "eu-north1",
  "scaling": { "min_replicas": 1, "max_replicas": 2 }
}
```

The response includes `endpoint_id` and `routing_key`. Initial deployment can take several minutes; inference may fail (often `404`) until the endpoint is routable.

### Step 3 — Send inference

Use a region-appropriate data-plane base URL and the `routing_key`:

```python
client = OpenAI(
    base_url="https://api.tokenfactory.us-central1.nebius.com/v1",
    api_key=API_TOKEN,
)
response = client.chat.completions.create(
    model=routing_key,
    messages=[{"role": "user", "content": "Explain RAG vs fine-tuning."}],
)
```

OpenAI-compatible routes are exposed under `/v1`. Today, publicly available templates are primarily chat-capable.

---

## 18. Dedicated Endpoint Lifecycle & Status

**Source:** [Lifecycle & Readiness Status](https://docs.tokenfactory.nebius.com/ai-models-inference/dedicated-endpoints/lifecycle-and-status)

Dedicated endpoints expose **two separate status types**, separating deployment progress from actual availability:

### Lifecycle status — what stage the deployment is in

| Status | Meaning |
|--------|---------|
| `Starting` | Endpoint is being created/started; provisioning in progress; traffic may not be available. |
| `Updating` | Applying configuration changes; traffic may continue but capacity can degrade. |
| `Running` | Deployed and expected to operate normally. |
| `Warning` | A deployment issue requires attention; may still serve traffic. Contact support if unresolved > 3 h. |
| `Stopping` | Stop requested; shutdown in progress. |
| `Stopped` | Intentionally disabled. |

### Readiness status — whether the endpoint can currently serve traffic

| Status | Meaning |
|--------|---------|
| `Not ready` | Cannot reliably serve requests (expected during Starting/Stopping/Stopped). |
| `Partially ready` | Can serve traffic but below expected capacity (some replicas starting/unavailable). Contact support if unresolved > 3 h. |
| `Ready` | Fully provisioned and ready for expected traffic. |

### Common status combinations

| Lifecycle | Readiness | Meaning |
|-----------|-----------|---------|
| Starting | Not ready | Provisioning |
| Starting | Partially ready | Serving, still scaling |
| Updating | Ready | Serving normally during update |
| Running | Ready | Fully operational |
| Running | Partially ready | Degraded capacity |
| Running | Not ready | Unexpected outage |
| Error | Ready | Serving, but deployment issue exists |
| Error | Not ready | Deployment failure |
| Stopping | Partially ready | Shutting down |
| Stopped | Not ready | Fully stopped |

**Key principle:** Use **Readiness** to decide whether to send traffic; Lifecycle is deployment state.

---

## 19. Dedicated Endpoint Capacity, Billing & SLA

**Source:** [Capacity, availability, and service guarantees](https://docs.tokenfactory.nebius.com/ai-models-inference/dedicated-endpoints/capacity-and-scaling), [Billing Policy](https://docs.tokenfactory.nebius.com/ai-models-inference/dedicated-endpoints/billing-policy)

### Capacity availability

- Deployment depends on available regional capacity — not all listed GPU types may be available at all times.
- To guarantee capacity, contact Sales to **reserve dedicated GPU capacity**.

### Replica guarantees

| Replica type | Guarantee |
|--------------|----------|
| **Minimum replicas** | Reserved and **non-preemptible**; remain allocated to you while the endpoint is active. |
| **Maximum replicas (above min)** | Scaling depends on available burst capacity and is **not guaranteed indefinitely**; additional replicas may be reclaimed after scale-down. |

### SLA

Self-service dedicated endpoints do **not** include a formal SLA unless covered by contract. Historical average monthly request success rate has been ~99.9%.

### Billing policy

- An endpoint is **active and billable** when **at least one replica is running**.
- With **zero replicas running**, the endpoint is not accessible and **not billed**.
- Scaling above/below minimum replicas adjusts billing dynamically on a PAYG basis.
- As long as the endpoint is active, minimum replicas remain allocated; once released, capacity may not be immediately available again.

| Lifecycle step | Billing |
|----------------|---------|
| Capacity provisioning, `not ready`, `partially ready` | **Not billed** |
| Capacity provisioned, `ready` status | **Billed** |
| Replica restarts | Not billed |
| Graceful shutdown for rolling update / endpoint stop | Not billed |

---

## 20. Dedicated Endpoint Control-Plane API

**Source:** [Operating Dedicated Endpoint](https://docs.tokenfactory.nebius.com/ai-models-inference/dedicated-endpoints/operating), [API Reference: dedicated-endpoints](https://docs.tokenfactory.nebius.com/api-reference/dedicated-endpoints/list-dedicated-endpoint-templates)

All management operations use the common control-plane base URL `https://api.tokenfactory.nebius.com` and the `Authorization: Bearer <token>` header.

### Endpoint parameters (create/update)

| Field | Type | Required | Description |
|-------|------|----------|-------------|
| `name` | string | yes (create) | Display name for the endpoint. Length `1–100`. |
| `description` | string | no | Optional description. Max `500`. |
| `model_name` | string | yes (create) | Template model name (e.g. `openai/gpt-oss-120b`). |
| `flavor_name` | string | yes (create) | Template flavor (e.g. `base`, `fast`). |
| `gpu_type` | enum | yes (create) | `gpu-l40s-d`, `gpu-l40s-a`, `gpu-h100-sxm`, `gpu-h200-sxm`, `gpu-b200-sxm`, `gpu-b200-sxm-a`, `gpu-b300-sxm`. |
| `gpu_count` | integer | yes (create) | GPUs per replica. Total max GPUs = `gpu_count × scaling.max_replicas`. `≥ 0`. |
| `region` | enum | yes (create) | `eu-north1`, `us-central1`, `eu-west1`, `me-west1`, `uk-south1`, `tf-us1`, `tf-us2`, `tf-ca1`. **Cannot be updated after creation.** |
| `scaling.min_replicas` | integer | yes | Minimum replicas. |
| `scaling.max_replicas` | integer | yes | Maximum replicas. |
| `enabled` | boolean | no | Enable/disable the endpoint. Disabling frees up replicas; enabling starts the endpoint. |
| `custom_weights_id` | string \| null | no | Custom weights ID. Pass `"UNSET"` on update to clear. |

### `DedicatedEndpoint` response object

| Field | Type | Description |
|-------|------|-------------|
| `id` | string | Endpoint ID (control-plane identifier) |
| `name` | string | Display name |
| `description` | string | Description |
| `enabled` | boolean | Enabled flag |
| `routing_key` | string | Model identifier for data-plane inference |
| `model_name` | string | Template model name |
| `flavor_name` | string | Flavor |
| `region` | string | Region |
| `gpu_type` | enum | GPU type |
| `gpu_count` | integer | GPUs per replica |
| `custom_weights_id` | string \| null | Custom weights ID |
| `scaling` | `{min_replicas, max_replicas}` | Scaling config |
| `deployment` | `{ready_replicas}` | Deployment status |
| `created_at` | date-time | Creation timestamp |

### `Model` template object (from list templates)

| Field | Type | Description |
|-------|------|-------------|
| `name` | string | Template/model name |
| `type` | string | Template type |
| `metadata` | object | `vendor`, `context_window_k`, `size_b`, `huggingface_url`, `license{url,name}` |
| `flavors` | object (map) | Available flavors (flavor_name → flavor details) |

### Endpoints

| Method | Path | Description |
|--------|------|-------------|
| `GET` | `/v0/dedicated_endpoints/templates?ai_project_id=` | List available model templates (source of truth for valid combos) |
| `POST` | `/v0/dedicated_endpoints?ai_project_id=` | Create a dedicated endpoint (201 → `{endpoint: DedicatedEndpoint}`) |
| `GET` | `/v0/dedicated_endpoints?ai_project_id=` | List all dedicated endpoints (→ `{data: DedicatedEndpoint[], object:"list"}`) |
| `PATCH` | `/v0/dedicated_endpoints/{endpoint_id}` | Partially update an endpoint (region cannot be changed) (200 → `DedicatedEndpoint`) |
| `DELETE` | `/v0/dedicated_endpoints/{endpoint_id}` | Delete an endpoint permanently; releases `min_replicas` GPUs |

Scaling changes may trigger provisioning of additional replicas — plan for a short warm-up. Errors use the FastAPI-style `detail[].{loc, msg, type, input, ctx}` schema (HTTP 422).

---

## 21. Custom Model Weights

**Source:** [Custom model weights](https://docs.tokenfactory.nebius.com/ai-models-inference/dedicated-endpoints/custom-weights)

### Capability

Deploy or work with custom fine-tuned model weights on dedicated endpoints. This is **currently in beta and available on request** — contact Support to enable access and be guided through the current setup. Availability, supported configurations, and onboarding steps may vary during the beta period.

### Usage

On endpoint creation or update, pass a `custom_weights_id` (referencing previously uploaded weights) in the request body. On update, pass `"UNSET"` to clear custom weights. Weights are uploaded via the files API (`upload-custom-model-archive`). Custom weights support is available only on dedicated endpoints, not on public serverless endpoints.

---

## 22. Sandboxes (Cloud Code Execution with Branching)

**Source:** [Sandboxes Overview](https://docs.tokenfactory.nebius.com/sandboxes/overview)

> **Status:** Beta. Sandboxes is a complementary GPU/cloud capability rather than an LLM inference endpoint per se — it provides secure, isolated, Git-like-branchable code execution environments, primarily for AI agents (coding, research, CI/CD). Included here because it represents Nebius's on-demand compute model beyond direct model inference.

### Capability

A cloud-based sandbox API enabling secure code execution with **Git-like branching**. Built for AI agents that need to explore multiple execution paths, evaluate outcomes, and backtrack. Combines VM-level isolation with container efficiency for executing untrusted code.

### Key features

| Feature | Description |
|---------|-------------|
| **Secure isolation** | VM-level isolation; untrusted code cannot escape the sandbox or affect other workloads. |
| **Git-like branching** | Fork execution state at any checkpoint; explore multiple solution paths in parallel, score results, expand the best branches. |
| **Instant rollback** | Return to any previous state with a single API call — no rebuild/re-execute from scratch. |
| **OCI image support** | Import images from any OCI-compliant registry (Docker Hub, GHCR, etc.) as sandbox bases. |
| **Resource metrics** | Built-in tracking of CPU time, memory usage, and I/O operations per execution. |
| **Async operations** | All long-running operations (image imports, executions) are async with polling and cancellation. |

### Core concepts

| Concept | Description |
|---------|-------------|
| **Checkpoint / state** | The resulting state produced by a run; can be forked or rolled back to. |
| **Execution** | Running code inside a sandbox; long-running work is modeled as an operation. |
| **Operation** | Async long-running work (image imports, executions) with polling support and cancellation. |
| **Branch** | A fork from any useful checkpoint to try alternatives. |

### Quick-start workflow

1. **Choose a base state** — preloaded environment, prior checkpoint, or imported OCI image.
2. **Make inputs explicit** — attach files/runtime assumptions for replayability.
3. **Start an execution** — run code inside the sandbox; long-running work becomes an operation.
4. **Wait for the operation** — poll via CLI/SDK/generated client until terminal state.
5. **Inspect the resulting checkpoint** — read logs, metrics, files, artifacts.
6. **Branch when needed** — fork from any useful checkpoint to try alternatives.

### Access methods

| Method | Description |
|--------|-------------|
| **Contree SDK** (`/sandboxes/sdk`) | Python SDK for programmatic access to the Sandboxes API. |
| **Contree CLI** (`/sandboxes/cli`) | Terminal client for interactive and scripted sandbox workflows. |
| **Contree MCP** (`/sandboxes/mcp`) | Model Context Protocol server for AI assistants (exposes tools like `run`, `list_files`, `read_file`, `upload`, `download`, `import_image`, `list_operations`, `wait_operations`, `cancel_operation`). |
| **Sandboxes for SWE agents** (`/sandboxes/swe-agents`) | Preloaded environments and Hugging Face datasets for software-engineering agents. |

### Use cases

AI coding agents (safe code execution + branching to explore approaches); research/experimentation (isolated envs, fork to test variations); educational platforms (safe execution, automatic cleanup, resource limits); CI/CD pipelines (isolated build/test steps with resource tracking and artifact retrieval).

### Beta limitations

- Max **15 simultaneously running operations**.
- Checkpoint image retention: **180 days**; after that, untagged unreferenced images may be deleted.

---

## 23. OpenAI Compatibility

**Source:** [API Reference Introduction](https://docs.tokenfactory.nebius.com/api-reference/introduction)

Nebius Token Factory offers an **OpenAI-compatible API** for inference and fine-tuning. Any OpenAI client (Python `openai`, JavaScript/TypeScript `openai`, or raw HTTP) works by changing only the `base_url` and `api_key`:

| Setting | Value |
|---------|-------|
| `base_url` (serverless) | `https://api.tokenfactory.nebius.com/v1/` |
| `base_url` (dedicated) | Region-appropriate data-plane URL, e.g. `https://api.tokenfactory.us-central1.nebius.com/v1` |
| `api_key` | `NEBIUS_API_KEY` |

Supported OpenAI-compatible endpoints: `chat/completions`, `completions`, `embeddings`, `responses`, `images/generations`, `rerank`, `models`. The API additionally supports the **full set of vLLM parameters** via `extra_body`.

Third-party integrations (Cursor, Cline, Helicone, aiAdev, Google ADK, CrewAI, LlamaIndex, CamelAI) are documented under the Integrations section for use instead of the raw API.

---

## 24. API Endpoint Reference Summary

### Inference data plane (`/v1`, OpenAI-compatible)

| Method | Path | Description |
|--------|------|-------------|
| `GET` | `/v1/models` | List available models (`ai_project_id`, `verbose` query params) |
| `POST` | `/v1/chat/completions` | Chat completions (tools, JSON, streaming, logprobs) |
| `POST` | `/v1/completions` | Legacy text completions |
| `POST` | `/v1/responses` | Responses API (stateful, multi-modal, built-in tools, reasoning, background) |
| `POST` | `/v1/embeddings` | Text embeddings |
| `POST` | `/v1/rerank` | Rerank documents by query relevance |
| `POST` | `/v1/images/generations` | Image generation (diffusion, LoRAs) |

All inference endpoints accept the `ai_project_id` query param and `service_tier` body param; all return `service_tier` in the response. Errors use FastAPI-style `detail[].{loc,msg,type,input,ctx}` (HTTP 422).

### Dedicated endpoint control plane (`/v0`)

| Method | Path | Description |
|--------|------|-------------|
| `GET` | `/v0/dedicated_endpoints/templates` | List deployable model templates (flavor/gpu/region combos) |
| `POST` | `/v0/dedicated_endpoints` | Create a dedicated endpoint → returns `endpoint_id` + `routing_key` |
| `GET` | `/v0/dedicated_endpoints` | List all dedicated endpoints |
| `PATCH` | `/v0/dedicated_endpoints/{endpoint_id}` | Partially update an endpoint (region immutable) |
| `DELETE` | `/v0/dedicated_endpoints/{endpoint_id}` | Delete an endpoint permanently |

### Base URLs

| Plane / Region | Base URL |
|----------------|----------|
| Control plane (all endpoint management) | `https://api.tokenfactory.nebius.com` |
| Data plane — `eu-north1` | `https://api.tokenfactory.nebius.com` |
| Data plane — `eu-west1` | `https://api.tokenfactory.eu-west1.nebius.com` |
| Data plane — `us-central1` | `https://api.tokenfactory.us-central1.nebius.com` |

### Common request elements

- **Auth:** `Authorization: Bearer $NEBIUS_API_KEY` (required on all endpoints)
- **Content-Type:** `application/json`
- **Query:** `ai_project_id` (string \| null) — project scoping (all endpoints)
- **Body:** `service_tier` enum — `auto` / `default` / `over-limit` / `flex` / `no-limit`

### OpenAPI specification

A machine-readable OpenAPI spec is published at [`https://api.tokenfactory.nebius.com/openapi.json`](https://api.tokenfactory.nebius.com/openapi.json).