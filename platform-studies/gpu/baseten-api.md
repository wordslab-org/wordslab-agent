# Baseten — On-Demand LLM Inference API Analysis

> **Source:** Documentation pages reachable from [https://docs.baseten.co/overview](https://docs.baseten.co/overview)
>
> **Scope:** On-demand LLM inference services — both serverless API (Model APIs, Frontier Gateway) and GPU machine rental (Dedicated deployments via Truss).
>
> **Date of analysis:** 2026-07-14

Baseten offers on-demand LLM inference through two complementary surfaces:

- **Model APIs** — a serverless, shared-infrastructure endpoint (`https://inference.baseten.co/v1`) serving a fixed catalog of popular open models (DeepSeek, GLM, Kimi, etc.). OpenAI- and Anthropic-compatible. No deployment step, per-million-token pricing, automatic KV-cache discounts.
- **Dedicated deployments** — you package a model (open or custom) with **Truss** and deploy it on rented GPU instances you control (hardware, engine, scaling). Each deployment gets a dedicated subdomain (`https://model-{id}.api.baseten.co/...`). Billed per-minute per replica, with scale-to-zero support.
- **Frontier Gateway** — a managed API gateway layered *on top of* a Dedicated deployment, for AI labs who want to resell their hosted model under their own branded URL with federated API keys, hierarchical groups, inherited rate/usage limits, and billing webhooks. OpenAI-compatible inference endpoint (`https://inference.baseten.co/v1`).

Management is split across two API hosts:

- **Management API** (`https://api.baseten.co/v1/...`) — models, deployments, environments, autoscaling, API keys, lifecycle.
- **Frontier Gateway API** (`https://api.baseten.co/v1/gateway/...`) — endpoints, groups, federated keys, billing webhooks (onboarded workspaces only).
- **Inference endpoints** — per-model subdomains (`https://model-{id}.api.baseten.co/...`) for dedicated deployments, or the shared `https://inference.baseten.co/v1` for Model APIs / Frontier Gateway.

---

## Table of Contents

1. [Platform Overview & Deployment Modes](#1-platform-overview--deployment-modes)
2. [Authentication & API Keys](#2-authentication--api-keys)
3. [Model APIs — Serverless Inference](#3-model-apis--serverless-inference)
4. [Dedicated Deployments — GPU Machine Rental](#4-dedicated-deployments--gpu-machine-rental)
5. [Inference API — Predict Endpoints](#5-inference-api--predict-endpoints)
6. [Streaming](#6-streaming)
7. [Async Inference](#7-async-inference)
8. [Structured Outputs & JSON Mode](#8-structured-outputs--json-mode)
9. [Function Calling (Tool Calling)](#9-function-calling-tool-calling)
10. [Inference Engines](#10-inference-engines)
11. [Quantization](#11-quantization)
12. [LoRA Adapters](#12-lora-adapters)
13. [Speculative Decoding (Lookahead)](#13-speculative-decoding-lookahead)
14. [GPU Resources & Instance Types](#14-gpu-resources--instance-types)
15. [Deployment Lifecycle & Environments](#15-deployment-lifecycle--environments)
16. [Autoscaling & Scale-to-Zero](#16-autoscaling--scale-to-zero)
17. [Cold Starts & BDN (Baseten Delivery Network)](#17-cold-starts--bdn-baseten-delivery-network)
18. [Request Lifecycle, Queuing & Retries](#18-request-lifecycle-queuing--retries)
19. [Rolling Deployments](#19-rolling-deployments)
20. [Health Checks](#20-health-checks)
21. [Model Configuration (config.yaml)](#21-model-configuration-configyaml)
22. [Performance Client](#22-performance-client)
23. [Frontier Gateway — Managed API Gateway](#23-frontier-gateway--managed-api-gateway)
24. [File I/O & Integrations](#24-file-io--integrations)
25. [Errors & Status Codes](#25-errors--status-codes)
26. [API Endpoint Reference Summary](#26-api-endpoint-reference-summary)

---

## 1. Platform Overview & Deployment Modes

**Source:** [Inference Overview](https://docs.baseten.co/inference/overview), [Deployment Concepts](https://docs.baseten.co/deployment/concepts), [Frontier Gateway Overview](https://docs.baseten.co/frontier-gateway/overview)

Baseten inference reaches a model running on Baseten infrastructure. You never provision GPUs or build routing yourself — Baseten authenticates each request, matches it to a deployment environment, and runs it on a replica.

| Mode | Description | Best for | Compute | Billing |
|------|-------------|----------|---------|---------|
| **Model APIs** | Shared infrastructure serving a fixed catalog of popular open models via an OpenAI-compatible endpoint | App developers calling a Baseten-hosted open model | Shared Baseten infrastructure | Per million tokens (cached input discounted) |
| **Dedicated deployments** | You deploy your own model (open or custom) packaged with Truss, on a dedicated subdomain, controlling hardware/engine/scaling | Custom serving logic, fine-tuned weights, models outside the Model APIs catalog | Your Dedicated deployment (rented GPU instances) | Per-minute per replica (scale-to-zero supported) |
| **Frontier Gateway** | Managed API gateway on top of a Dedicated deployment, for AI labs reselling their hosted model under a branded URL | AI labs serving their own hosted model to downstream customers | Your Dedicated deployment | (Plus federated keys, hierarchical groups, billing webhooks) |

### Inference interfaces

When you deploy your own model, pick an interface matching your payloads:

- **Engine-Builder-LLM, BIS-LLM, BEI** expose `/v1/chat/completions` (or `/v1/embeddings` for BEI) on your deployment's own endpoint, with OpenAI-compatible parameters for structured outputs, tool calling, reasoning, and streaming.
- **Custom Truss code** can use `/predict` for arbitrary JSON when chat or embeddings are not a good fit.

### OpenAI compatibility

Model APIs and all engine-backed deployments are OpenAI-compatible. Switching from OpenAI requires only changing the `base_url` and the API key. The OpenAI SDK (Python/JS) works directly. Any OpenAI-protocol LLM gateway (LiteLLM, OpenRouter) can also route to a Baseten deployment.

### OpenAPI spec

- Inference API spec: `https://api.baseten.co/inference-spec`
- Frontier Gateway API spec: `https://api.baseten.co/v1/spec`

---

## 2. Authentication & API Keys

**Source:** [Manage deployments](https://docs.baseten.co/deployment/manage/overview), [Frontier Gateway API keys](https://docs.baseten.co/frontier-gateway/api-keys)

### Authentication headers

| Surface | Header | Key type |
|---------|--------|----------|
| Inference (Model APIs, dedicated deployments) | `Authorization: Bearer $BASETEN_API_KEY` | Workspace API key |
| Inference (legacy, still accepted) | `Authorization: Api-Key $BASETEN_API_KEY` | Workspace API key |
| Management API (`api.baseten.co/v1/...`) | `Authorization: Bearer $BASETEN_API_KEY` | Workspace API key (management scope) |
| Frontier Gateway API (`api.baseten.co/v1/gateway/...`) | `Authorization: Api-Key $BASETEN_API_KEY` | Workspace API key with management scope |
| Frontier Gateway inference (federated keys) | `Authorization: Api-Key YOUR_FEDERATED_KEY` | Federated API key minted under a group |

### API key types (Management API)

| Type | Purpose |
|------|---------|
| `PERSONAL` | Tied to your account/permissions; local dev/testing |
| `WORKSPACE_MANAGE_ALL` | Full workspace management |
| `WORKSPACE_INVOKE` | Invoke models; optionally scoped to specific model IDs |
| `WORKSPACE_EXPORT_METRICS` | Export metrics |

- Keys are created via the UI or `POST /v1/api_keys`.
- A key is shown in full only once at creation.
- Enterprise auth: SSO and SCIM supported (contact support to enable).

### Federated API keys (Frontier Gateway)

- Minted under a **group** via `POST /v1/gateway/groups/{group_id}/api_keys`.
- Inherit the group's effective model set and limits — no per-key overrides.
- Plaintext secret shown once; the **prefix** (substring before `.`) is used as the path parameter for fetch/revoke.
- Existing keys you control can be **registered** (`POST .../api_keys/register`) with an Ed25519 body signature; requires 32–128 chars, ≥3.0 bits Shannon entropy/char.

### API functions — API keys

| Function | Method & Endpoint | Purpose |
|----------|-------------------|---------|
| Create API key | `POST /v1/api_keys` | Create a workspace API key. Body: `{"type": "PERSONAL", "name": ""}` |
| List models | `GET /v1/models` | List deployed models (returns `id`, `name`, `deployments_count`, `production_deployment_id`) |
| List deployments of a model | `GET /v1/models/{model_id}/deployments` | List a model's deployments |

---

## 3. Model APIs — Serverless Inference

**Source:** [Model APIs Overview](https://docs.baseten.co/inference/model-apis/overview)

### Main concepts

- **Model APIs**: OpenAI-compatible endpoints giving instant access to high-performance LLMs. No deployment required — Baseten manages shared GPU clusters, model weights, and serving config.
- **Two compatible surfaces**: OpenAI Chat Completions API and Anthropic Messages API (beta). Point an existing OpenAI or Anthropic SDK at Baseten's inference endpoint.
- **Supported models**: a fixed catalog (e.g. DeepSeek, GLM, Kimi). Context and output limits reflect Baseten's live serving config (may differ from a model's native advertised max). The `/v1/models` endpoint reflects what's currently served.
- **Pricing**: per million tokens. **Cached input tokens** (served from KV cache) billed at a discounted rate. Caching is automatic — no opt-in.
- **Feature support**: all models support tool calling, structured outputs, and JSON mode. Reasoning and vision vary per model. GLM models, Nemotron Super, and Nemotron Ultra also support `top_p` and `top_k` sampling parameters.

### API functions

| Function | Method & Endpoint | Purpose |
|----------|-------------------|---------|
| Chat Completions (OpenAI) | `POST https://inference.baseten.co/v1/chat/completions` | OpenAI-compatible chat completions |
| Messages (Anthropic, beta) | `POST https://inference.baseten.co/v1/messages` | Anthropic-compatible messages (beta) |
| List available models | `GET https://inference.baseten.co/v1/models` | Current models with metadata (pricing, context windows, supported features) |

### Key parameters

| Parameter | Type | Description |
|-----------|------|-------------|
| `model` | string | Baseten model slug, e.g. `"deepseek-ai/DeepSeek-V4-Pro"` |
| `messages` | array | Chat messages with `role` (system/user/assistant) and `content` |
| `system` | string | System prompt (Anthropic SDK; separate from `messages`) |
| `max_tokens` | int | Required by the Anthropic Messages API (e.g. 4096) |
| `base_url` (client) | string | `https://inference.baseten.co/v1` (OpenAI SDK) or `https://inference.baseten.co` (Anthropic SDK) |
| `api_key` / `Authorization` | string | `Bearer $BASETEN_API_KEY` (Anthropic SDK must override `default_headers` to send `Authorization` instead of `x-api-key`) |
| `stream` | bool | Enable streaming (per-request flag) |

### Model APIs vs Frontier Gateway limits

- **Model APIs**: account-tier RPM/TPM ceilings applying to your workspace API key as a whole.
- **Frontier Gateway**: per-group, per-model slug, with an inheritance mode picked at the root (see §23).

---

## 4. Dedicated Deployments — GPU Machine Rental

**Source:** [Deployment Concepts](https://docs.baseten.co/deployment/concepts), [Deployments](https://docs.baseten.co/deployment/deployments), [Manage deployments](https://docs.baseten.co/deployment/manage/overview)

### Main concepts

- **Deployment**: a single version of your model running on specific hardware. Every `truss push` creates a new deployment. Multiple deployments of the same model can run simultaneously. Deployments can be deactivated (stop serving/stop cost) or deleted permanently.
- **Development deployment**: a mutable instance created with `truss push --watch` that live-reloads as you edit model code. Single replica, scales to zero when idle, no autoscaling or zero-downtime updates. Cannot be promoted directly to an environment; after promotion it becomes a persistent deployment.
- **Environments**: stable endpoints that persist across deployments (e.g. a `development` environment for testing, a `production` environment for live traffic). Each environment has its own autoscaling settings, metrics, and endpoint URL. Promoting a new deployment to an environment shifts traffic without changing the endpoint your app calls. Production exists by default; custom environments (e.g. `staging`) can be created.
- **Resources**: every deployment runs on a specific instance type defining GPU, CPU, and memory. Set in `config.yaml` before deployment or adjusted via the Baseten UI.
- **Deployment target**: part of the inference endpoint URL — an environment name (`production`), the development deployment (`development`), or a specific deployment ID.

### Deployment management methods

| Method | Notes |
|--------|-------|
| Console (app.baseten.co) | Dashboard UI |
| Baseten CLI | Every command supports `--output json` and `--jq` filtering |
| Management API | REST endpoints |
| CI/CD | GitHub Action `basetenlabs/action-truss-push@v0.1` |

### Deployment object fields

`id`, `name`, `model_id`, `is_production`, `is_development`, `status` (enum: `ACTIVE`, `INACTIVE`, `DEPLOYING`, `SCALED_TO_ZERO`, `WAKING_UP`), `active_replica_count`, `environment`, `instance_type_name`, `autoscaling_settings`, `created_at`, `labels`.

### Deployment names & labels

- Deployments default to sequential names `deployment-1`, `deployment-2`, etc. Name at deploy time with `truss push --deployment-name my-deployment`. Names are cosmetic; API paths use model and deployment IDs.
- **Labels**: JSON key-value metadata to organize/track a deployment. Set with `truss push --labels '{"team": "ml-platform", "env": "staging"}'`, the `labels` argument to `truss.push()`, or the `labels` input on the deploy GitHub Action.

---

## 5. Inference API — Predict Endpoints

**Source:** [Inference API Overview](https://docs.baseten.co/reference/inference-api/overview), [Call your model](https://docs.baseten.co/inference/calling-your-model)

### URL patterns

**Models:**
```
https://model-{model_id}.api.baseten.co/{deployment_type_or_id}/{endpoint}
```

**Chains:**
```
https://chain-{chain_id}.api.baseten.co/{deployment_type_or_id}/{endpoint}
```

**Regional environments** (env name embedded in hostname):
```
https://model-{model_id}-{env_name}.api.baseten.co/{endpoint}
https://chain-{chain_id}-{env_name}.api.baseten.co/{endpoint}
```

`deployment_type_or_id` is one of `development`, `production`, or a specific deployment's alphanumeric ID. `endpoint` is the API action (e.g. `predict`).

### Predict endpoints — Models

| Function | Method & Endpoint | Purpose |
|----------|-------------------|---------|
| Predict (production) | `POST /production/predict` | Call the production environment |
| Predict (named env) | `POST /environments/{env_name}/predict` | Call a named environment |
| Predict (development) | `POST /development/predict` | Call the development deployment |
| Predict (specific deployment) | `POST /deployment/{deployment_id}/predict` | Call a specific deployment |
| Async predict (production) | `POST /production/async_predict` | Async predict on production |
| Async predict (named env) | `POST /environments/{env_name}/async_predict` | Async predict on a named environment |
| Async predict (development) | `POST /development/async_predict` | Async predict on the development deployment |
| Async predict (specific deployment) | `POST /deployment/{deployment_id}/async_predict` | Async predict on a specific deployment |

### Predict endpoints — Chains (`run_remote`)

| Function | Method & Endpoint | Purpose |
|----------|-------------------|---------|
| run_remote (production) | `POST /production/run_remote` | Call the production environment |
| run_remote (named env) | `POST /environments/{env_name}/run_remote` | Call a named environment |
| run_remote (development) | `POST /development/run_remote` | Call the development deployment |
| run_remote (specific deployment) | `POST /deployment/{deployment_id}/run_remote` | Call a specific deployment |
| async_run_remote (production) | `POST /production/async_run_remote` | Async call on production |
| async_run_remote (named env) | `POST /environments/{env_name}/async_run_remote` | Async call on a named environment |
| async_run_remote (development) | `POST /development/async_run_remote` | Async call on the development deployment |
| async_run_remote (specific deployment) | `POST /deployment/{deployment_id}/async_run_remote` | Async call on a specific deployment |

### Regional endpoints (bare paths on regional hostnames)

| Function | Method & Endpoint |
|----------|-------------------|
| Sync predict | `POST /predict` |
| Sync chain call | `POST /run_remote` |
| Async predict | `POST /async_predict` |
| Async chain call | `POST /async_run_remote` |

### Sync endpoint (custom servers)

Custom servers support a `sync` endpoint to call different routes:

```
https://model-{model_id}.api.baseten.co/environments/production/sync/{route}
```

Example mappings: `/sync/health` → `/health`, `/sync/items/123` → `/items/123`.

### Status endpoints

| Function | Method & Endpoint | Purpose |
|----------|-------------------|---------|
| Get async request status | `GET /async_request/{request_id}` | Get the status of an async request |
| Cancel async request | `DELETE /async_request/{request_id}` | Cancel a queued async request |
| Queue status (production) | `GET /production/async_queue_status` | Queue status for production |
| Queue status (named env) | `GET /environments/{env_name}/async_queue_status` | Queue status for a named environment |
| Queue status (development) | `GET /development/async_queue_status` | Queue status for the development deployment |
| Queue status (specific deployment) | `GET /deployment/{deployment_id}/async_queue_status` | Queue status for a specific deployment |
| Queue status (regional) | `GET /async_queue_status` | Queue status (regional) |

### Wake endpoints

| Function | Method & Endpoint | Purpose |
|----------|-------------------|---------|
| Wake (production) | `POST /production/wake` | Wake the production environment |
| Wake (named env) | `POST /environments/{env_name}/wake` | Wake a named environment |
| Wake (development) | `POST /development/wake` | Wake the development deployment |
| Wake (specific deployment) | `POST /deployment/{deployment_id}/wake` | Wake a specific deployment |
| Wake (regional) | `POST /wake` | Wake (regional) |

### Timeouts (server-side, not user-configurable; exceeding returns 504)

| Surface | Default timeout |
|---------|-----------------|
| Sync predict (`/predict`, `/run_remote`) | 1200 seconds (20 minutes) |
| Async predict submit (`/async_predict`, `/async_run_remote`) | 3600 seconds (60 minutes) |
| Wake (`/wake`) | 600 seconds (10 minutes) |

### Request size

The ingress proxy caps each inbound request body at **100 MB**. Requests exceeding return `413 Request Entity Too Large` at the edge (before reaching the model). Applies to the full HTTP body (JSON envelope + base64-encoded media); not user-configurable. For large payloads, upload to object storage (S3/GCS/Azure Blob) and send a pre-signed URL in the request body.

### Alternative invocation methods

- **Baseten CLI**: `baseten model predict`
- **Model Dashboard**: "Playground" button in the Baseten UI

---

## 6. Streaming

**Source:** [Streaming](https://docs.baseten.co/inference/streaming)

### Main concepts

- **Streaming** returns a model's output incrementally, token by token, as it is generated, rather than holding the response until generation finishes. The first tokens arrive after the time-to-first-token (TTFT).
- Supported across Model APIs (OpenAI- and Anthropic-compatible endpoints), BIS-LLM, Engine-Builder-LLM, and dedicated Truss deployments. Custom Docker containers that expose an OpenAI-compatible API (vLLM, SGLang) stream the same way.
- Streaming is a **per-request flag** (`"stream": true`); only the base URL and model slug differ between surfaces.

### Enabling streaming

```python
# Model APIs (OpenAI-compatible)
client = OpenAI(base_url="https://inference.baseten.co/v1", api_key=os.environ["BASETEN_API_KEY"])
stream = client.chat.completions.create(
    model="deepseek-ai/DeepSeek-V4-Pro",
    messages=[{"role": "user", "content": "Write a haiku about the ocean."}],
    stream=True,
)
for chunk in stream:
    print(chunk.choices[0].delta.content or "", end="", flush=True)
```

```python
# Self-deployed Truss model
with requests.post(
    f"https://model-{model_id}.api.baseten.co/production/predict",
    headers={"Authorization": f"Bearer {os.environ['BASETEN_API_KEY']}"},
    json={"prompt": "Write a haiku about the ocean.", "stream": True},
    stream=True,
) as resp:
    for chunk in resp.iter_content():
        print(chunk.decode("utf-8"), end="", flush=True)
```

Streaming changes when the caller sees output, not how much the model produces. Both streaming and non-streaming finish together.

---

## 7. Async Inference

**Source:** [Async inference](https://docs.baseten.co/inference/async)

### Main concepts

- **Async inference**: a "fire and forget" pattern. You receive a `request_id` immediately while inference runs in the background; results are delivered to your webhook endpoint when complete.
- Works with any dedicated deployment (no code changes). Requests can queue up to 72 hours and run up to 1 hour.
- **Not compatible with streaming output. Not available on Model APIs.**
- Baseten does **not** store model outputs — if webhook delivery fails after all retries, data is lost.
- The async queue is decoupled from model scaling; requests queue even at zero replicas, triggering the autoscaler. If the model doesn't become ready within `max_time_in_queue_seconds`, the request expires with status `EXPIRED`.
- `predict_concurrency` (config.yaml) defines how many requests a model processes simultaneously per replica; sync and async share this pool (sync takes priority; queue processor backs off on 429s).
- Chains support async via `async_run_remote`; entrypoint requests queue but internal Chainlet-to-Chainlet calls run synchronously.

### API functions

| Function | Method & Endpoint | Purpose |
|----------|-------------------|---------|
| Submit async request | `POST /production/async_predict` | Submit an async inference request; returns `request_id` immediately |
| Check request status | `GET /async_request/{request_id}` | Get status of an async request (no environment segment in path) |
| Cancel request | `DELETE /async_request/{request_id}` | Cancel a queued request (only `QUEUED` requests can be canceled) |
| Webhook delivery (inbound) | `POST {your webhook URL}` | Baseten POSTs results to your webhook when inference completes |

### Key parameters (async_predict request body)

| Parameter | Type | Default | Description |
|-----------|------|---------|-------------|
| `model_input` | object | — | The model input, e.g. `{"prompt": "hello world!"}` |
| `webhook_endpoint` | string | — | HTTPS URL to receive results (localhost allowed for dev) |
| `priority` | int (0/1/2) | 0 | Lower value = higher priority; 0 processed before 1 or 2 |
| `max_time_in_queue_seconds` | int | 600 (10 min) | How long a request waits in queue before expiring (max 72 hours) |
| `inference_retry_config` | object | — | Automatic retry config for model calls |
| `inference_retry_config.max_attempts` | int (1–10) | 3 | Total inference attempts including original |
| `inference_retry_config.initial_delay_ms` | int (0–10000) | 1000 | Delay before first retry (ms) |
| `inference_retry_config.max_delay_ms` | int (0–60000) | 5000 | Max delay between retries (ms) |

Retries use exponential backoff with multiplier 2, capping at `max_delay_ms`. Only retryable codes (500, 502, 503, 504) are retried; non-retryable (422, 404) fail immediately.

### Webhook payload fields

| Field | Type | Description |
|-------|------|-------------|
| `request_id` | string | Matches the original `/async_predict` response |
| `model_id` | string | Which model ran the request |
| `deployment_id` | string | Which deployment ran the request |
| `type` | string | e.g. `"async_request_completed"` |
| `time` | string (ISO 8601 UTC) | e.g. `"2024-04-30T01:01:08.883423Z"` |
| `data` | object | Contains `output` field with model output; `null` if an error occurred |
| `errors` | array | Empty on success; error objects on failure (each with `code`, `message`) |

### Webhook headers (sent by Baseten)

| Header | Description |
|--------|-------------|
| `Content-Type: application/json` | — |
| `X-BASETEN-REQUEST-ID` | Correlates with original request |
| `X-BASETEN-SIGNATURE` | `v1=...` HMAC-SHA256 of raw request body (only if webhook secret configured) |

Webhooks must use HTTPS (except localhost); supports HTTP/2 and HTTP/1.1. Webhook secret created in Baseten settings; both old/new secrets valid for 24 hours during rotation.

### Request status values

| Status | Description |
|--------|-------------|
| `QUEUED` | Waiting in queue |
| `IN_PROGRESS` | Currently processing |
| `SUCCEEDED` | Completed successfully |
| `FAILED` | Failed after retries |
| `EXPIRED` | Exceeded `max_time_in_queue_seconds` |
| `CANCELED` | Canceled by user |
| `WEBHOOK_FAILED` | Inference succeeded but webhook delivery failed |

Status is available for 1 hour after completion. `request_id` is globally unique.

### Webhook delivery settings

- Total attempts: 2 (1 initial + 1 retry)
- Backoff: ~2 seconds before retry
- Timeout: 10 seconds per attempt
- Retryable codes: 500, 502, 503, 504

### Async error codes

| Code | HTTP | Description | Retried |
|------|------|-------------|---------|
| `MODEL_NOT_READY` | 400 | Model is loading or starting | Yes |
| `MODEL_DOES_NOT_EXIST` | 404 | Model or deployment not found | No |
| `MODEL_INVALID_INPUT` | 422 | Invalid input format | No |
| `MODEL_PREDICT_ERROR` | 500 | Exception in `model.predict()` | Yes |
| `MODEL_UNAVAILABLE` | 502/503 | Model crashed or scaling | Yes |
| `MODEL_PREDICT_TIMEOUT` | 504 | Inference exceeded timeout | Yes |
| `INTERNAL_SERVER_ERROR` | N/A | Baseten-side error | Yes |

### Rate limits

| Endpoint | Limit |
|----------|-------|
| `/async_predict` | 12,000 requests/minute (org-level) |
| Status polling | 100 requests/second |
| Cancel request | 100 requests/second |

---

## 8. Structured Outputs & JSON Mode

**Source:** [Structured outputs](https://docs.baseten.co/inference/structured-outputs), [JSON mode](https://docs.baseten.co/inference/json-mode)

### Structured outputs

Generate text conforming to specific JSON schemas for reliable data extraction and controlled text generation.

- Supported by Model APIs and self-deployed models using BIS-LLM, Engine-Builder-LLM, plus vLLM and SGLang.
- Requires a **Pydantic schema** (`BaseModel` subclass with typed fields) and an API call that enforces it via `client.beta.chat.completions.parse` (not regular `create`).
- The response includes a `parsed` attribute (the typed object) — no manual JSON parsing needed.
- **Engine support**: Engine-Builder-LLM (except when Lookahead speculative decoding configured); BIS-LLM (except some configs, e.g. overlap scheduler enabled).

| Function | Method & Endpoint | Purpose |
|----------|-------------------|---------|
| Parse (structured output) | `POST {base_url}/chat/completions` (via `client.beta.chat.completions.parse`) | Generate text conforming to a Pydantic/JSON schema |

Key parameters: `model`, `messages`, `response_format` (Pydantic class or JSON schema).

Response: `response.choices[0].message.parsed` — the typed object with fields populated per the schema.

### JSON mode

Forces a model to emit syntactically valid JSON by setting `response_format` to `{"type": "json_object"}`. The server only enforces well-formedness (valid JSON), not shape — you still describe fields in the prompt.

| Feature | JSON mode | Structured outputs |
|---------|-----------|-------------------|
| Output guarantee | Valid JSON | Valid JSON matching your schema |
| Schema enforcement | None | Strict (server rejects non-conforming generations) |
| Setup | `response_format: {"type": "json_object"}` | Provide JSON schema or Pydantic model |
| Best for | Lightweight extraction, ad hoc responses | Production data extraction, typed pipelines |

### Best practices

- **Schema design**: keep 2–3 levels nesting; use basic types (str, int, float, bool); set defaults for optional fields; descriptive names.
- **Prompt engineering**: low temperature (0.1–0.3); dump model schema and few-shot examples into context; provide background for complex schemas.

---

## 9. Function Calling (Tool Calling)

**Source:** [Function calling](https://docs.baseten.co/inference/function-calling)

### Main concepts

- **Function calling** lets a model choose a tool and produce its arguments from a user request. The model does **not** run the tool; your application runs it and may send the result back to the model for a final user-facing response.
- Supported engines: BIS-LLM, Engine-Builder-LLM, Model APIs, plus vLLM and SGLang.
- **Tool-calling loop**: (1) send user message + list of tools; (2) model returns text or one or more tool calls (name + JSON arguments); (3) execute the tool calls in your app; (4) send tool output back to the model; (5) receive a final response or additional tool calls.
- Tools can be anything (API calls, DB queries, internal scripts). Docstrings matter — models use them to decide which tool to call and how to fill parameters.
- Serialization: convert functions into JSON-schema tool definitions (OpenAI-compatible format).

### API functions

| Function | Method & Endpoint | Purpose |
|----------|-------------------|---------|
| Call Model API with tools | `POST https://inference.baseten.co/v1/chat/completions` | Run chat completions with a `tools` array |
| Call dedicated deployment with tools | `POST https://model-{MODEL_ID}.api.baseten.co/production/predict` | Run a custom deployed model's predict with tools |

### Key parameters

| Parameter | Type | Description |
|-----------|------|-------------|
| `model` | string | Model slug (Model APIs) |
| `messages` | array | Conversation messages with `role` (system/user) and `content` |
| `tools` | array | JSON-schema tool definitions (OpenAI-compatible) |
| `tool_choice` | enum/object (default `"auto"`) | Controls tool use |

`tool_choice` values:
- `"auto"` — model can respond with text or tool calls (default)
- `"required"` — model must return at least one tool call
- `"none"` — model returns plain text only
- `{"type": "function", "function": {"name": "<tool_name>"}}` — force a specific tool

### Tool call response fields

- `choices[0].message` — the assistant message
- `choices[0].message.tool_calls` (array, may be absent) — list of tool calls:
  - `tool_call.function.name` — the tool function name
  - `tool_call.function.arguments` — JSON string of arguments (parse with `json.loads`)
  - `tool_call.id` — identifier used to match the tool response via `tool_call_id`
- To continue the loop: append the assistant message (with `tool_calls`) and a `{"role": "tool", "tool_call_id": <id>, "content": <json result>}` message, then re-post.

### Practical tips

- Use low temperature (0.0–0.3) for reliable tool selection/argument values.
- Add `enum` and `required` constraints in the JSON schema to guide outputs.
- Consider parallel tool calls only if your model supports them.
- Always validate/sanitize inputs before calling real systems; treat model-provided arguments as untrusted input.

---

## 10. Inference Engines

**Source:** [Engines Overview](https://docs.baseten.co/engines), [BIS-LLM](https://docs.baseten.co/engines/bis-llm/overview), [Engine-Builder-LLM](https://docs.baseten.co/engines/engine-builder-llm/overview), [BEI](https://docs.baseten.co/engines/bei/overview)

Baseten engines optimize model inference for specific architectures using TensorRT-LLM. All engines mirror build artifacts to the Baseten Delivery Network (BDN) automatically.

### Engine comparison

| Feature | BIS-LLM | Engine-Builder-LLM | BEI | BEI-Bert |
|---------|---------|---------------------|-----|----------|
| Target | MoE & large dense LLMs | Dense text generation LLMs | Embeddings, reranking, classification (causal) | BERT-family encoders |
| Quantization | yes | yes | yes (FP8/FP4) | no (FP16/BF16 only) |
| KV quantization | yes | yes | partial | partial |
| Lookahead decoding | no (uses MTP/Eagle/N-gram) | yes (v1) | no | no |
| KV-routing | yes (Enterprise, locked) | no | no | no |
| Disaggregated serving | yes (Enterprise, locked) | no | no | no |
| Tool calling & structured output | yes | yes | no | no |
| Classification models | no | no | yes | yes |
| Embedding models | no | no | yes | yes |
| Mixture-of-experts | yes | partial (Qwen3MoE only) | no | no |
| MTP/Eagle/N-gram speculation | yes (v2) | no | no | no |
| Self-serviceable | requires Enterprise (locked) | yes | yes | yes |
| HTTP request cancellation | yes | partial (within first 10ms) | yes | yes |

### BEI (Baseten Embeddings Inference)

Production-grade embeddings, reranking, and classification. Sub-millisecond response times; up to 1,400 client embeddings per second on H100; 121K tokens/s on B200.

**Workflows:**

| Function | Method & Endpoint | Purpose |
|----------|-------------------|---------|
| Embeddings | `POST /v1/embeddings` | OpenAI-compatible embedding generation (causal embedders, `base_model: encoder`) |
| Reranking | `POST /rerank` | Rerank query-document pairs (`base_model: encoder`); response sorted by score desc |
| Classification/predict | `POST /predict` | Sequence classification (`base_model: encoder_bert`); needs `id2label` in HF config |
| Named entity recognition | `POST /predict_tokens` | Token-level entity classification (BEI-Bert only) |

Request/response formats:
- `/rerank` request body: `{"query": string, "texts": [string, ...]}`
- `/rerank` response: `[{"index": 0, "score": 0.92}, ...]`, sorted by score descending
- `/embeddings`: standard OpenAI client `embeddings.create(input=[...], model="not-required")`

Pooling strategy is read from the HF repo at build time (`modules.json`, `1_Pooling/config.json`); not set in config.yaml. SPLADE supported on BEI-Bert.

### BIS-LLM (Baseten Inference Stack v2)

Engine for Mixture of Experts (MoE) models and large dense LLMs. Targets MoE families: DeepSeek V3.x, Qwen3MoE, Kimi-K2, Llama 4, GLM-4.7, GPT-OSS 120B.

**Four production features:**
1. **Token-based autoscaling** — scales replicas on `target_in_flight_tokens` rather than request concurrency (mixed-length prompt workloads scale on real compute load).
2. **KV-aware routing** — routes requests to the worker most likely to serve them from KV cache; lower TTFT on prefix-overlapping traffic.
3. **Disaggregated serving** — splits prefill and decode onto independent worker groups that scale separately.
4. **Speculative decoding** — Eagle, MTP, and N-gram speculation; multiple tokens per forward pass.

API endpoints (OpenAI-compatible, served by BIS-LLM deployments):

| Function | Method & Endpoint | Purpose |
|----------|-------------------|---------|
| Chat completions | `POST /v1/chat/completions` | OpenAI-compatible text generation |
| Text completions | `POST /v1/completions` | OpenAI-compatible completion |
| Embeddings | `POST /v1/embeddings` (where applicable) | OpenAI-compatible embeddings |

Base URL pattern: `https://model-xxxxxx.api.baseten.co/environments/production/sync/v1`

**Engine-level metrics:**
- `tps_per_request`, `input_tokens`, `output_tokens`, `input_tokens_per_request`, `output_tokens_per_request`, `concurrent_requests`, `speculation_rate`, `cpu_usage`, `memory_usage`, `gpu_usage`, `gpu_memory_usage`, `replica_count_by_status`
- Metric prefix domains: `autoscaler_*`, `kv_cache_*` (includes `kv_cache_hit_rate`), engine-level.

### Engine-Builder-LLM (v1)

Optimizes dense text generation models with TensorRT-LLM; up to 4000 tokens/second for code generation with lookahead decoding.

Supported model families: Llama 3.3/3.2, Qwen 3/2.5, Mistral Small/7B, GPT-OSS-20B, Nemotron, Gemma 3 (27b/12b), Phi-4. (Llama 4, DeepSeek MoE, Kimi, GLM MoE → use BIS-LLM.)

API endpoints (OpenAI-compatible):

| Function | Method & Endpoint | Purpose |
|----------|-------------------|---------|
| Chat completions | `POST {base_url}/chat/completions` | Standard OpenAI chat generation |
| Structured outputs parse | `client.beta.chat.completions.parse` | Parse JSON-schema-validated response |

Base URL: `https://model-xxxxxx.api.baseten.co/environments/production/sync/v1`

**Architecture support** (auto-detected from HF `architectures` field):

| Hugging Face architecture | Backend | Example models |
|---------------------------|---------|----------------|
| LlamaForCausalLM, LLaMAForCausalLM | LLaMA | Llama 3.2, Llama 3.3 |
| MistralForCausalLM | LLaMA | Mistral 7B, Mistral Small |
| Qwen2ForCausalLM | Qwen | Qwen 2.5 dense |
| Qwen3ForCausalLM | Qwen3 | Qwen 3 dense |
| Qwen3MoeForCausalLM | Qwen3 | Qwen 3 MoE |
| Gemma2ForCausalLM, Gemma3ForCausalLM | Gemma | Gemma 2/3 (bf16 only) |
| DeciLMForCausalLM | Nemotron NAS | NVIDIA Nemotron NAS |

**Model size support:**

| Model size | Single GPU | Tensor parallel | Recommended GPU |
|------------|-----------|-----------------|-----------------|
| <8B | H100_40GB, H100, B200 | N/A | H100_40GB |
| 8B–30B | H100, B200 | TP1 | H100 |
| 30B–70B | H100 | TP2–TP4 | H100 (4 GPUs) |
| 70B+ | H100, B200 | TP4–TP8 | H100 (8 GPUs) or B200 (2–4 GPUs) |

---

## 11. Quantization

**Source:** [Quantization guide](https://docs.baseten.co/engines/performance-concepts/quantization-guide)

### Main concepts

Quantization trades precision for speed and memory efficiency. Format choice is bounded by which GPU families run it and how much weight memory it saves.

| Quantization | Min GPU | Recommended GPU | Memory reduction |
|---------------|---------|-----------------|------------------|
| FP16/BF16 (`no_quant`) | A100 | H100 | None |
| FP8 | L4 | H100 | ~50% |
| FP8_KV | L4 | H100 | ~60% |
| FP4 | B200 | B200 | ~75% |
| FP4_KV | B200 | B200 | ~80% |
| FP4_MLP_ONLY | B200 | B200 | Between FP8 and FP4 |

- `_KV` formats store compressed KV cache state. Not applicable to encoder models (BEI, BEI-Bert) since they have no decoder-style KV cache.
- `no_quant` skips post-training quantization; uses checkpoint's native precision. Used for unquantized FP16/BF16 checkpoints or pre-quantized NVIDIA ModelOpt checkpoints.
- Non-ModelOpt pre-quantized checkpoints (GPTQ, AWQ safetensors) are **not** supported.
- Re-quantizing a ModelOpt checkpoint with a different `quantization_type` causes a build error; use `no_quant`.

### Engine support

| Quantization | BIS-LLM | Engine-Builder-LLM | BEI |
|---------------|---------|---------------------|-----|
| FP8 | yes | yes | yes |
| FP8_KV | yes | yes | no |
| FP4 | yes | yes | no |
| FP4_KV | yes | yes | no |
| FP4_MLP_ONLY | yes | yes | yes |
| no_quant | yes | yes | yes |
| INT8 / SmoothQuant | no (v2 error) | yes (v1) | no |

### GPU support

| GPU type | FP8 | FP8_KV | FP4 | FP4_KV | FP4_MLP_ONLY |
|----------|-----|--------|-----|--------|--------------|
| L4 | yes | yes | no | no | no |
| H100 | yes | yes | no | no | no |
| H200 | yes | yes | no | no | no |
| B200 | yes | yes | yes | yes | yes |

### Model recommendations

- **Qwen2**: retains QKV projection bias; sensitive to symmetric KV cache quantization, so FP8_KV causes quality degradation. Use regular FP8; increase `calib_size` to 1024+.
- **Llama**: works well with FP8_KV and standard calibration (1024–1536). For B200, use FP4_MLP_ONLY.
- **BEI (embeddings)**: use FP8 for causal embedding models. Skip quantization for smaller models. BERT not supported.

### Calibration config (config.yaml)

| Parameter | Type | Description |
|-----------|------|-------------|
| `calib_size` | int | Number of calibration samples (e.g. 768, 1024). Increase for larger models. |
| `calib_dataset` | string | HuggingFace dataset name (default `abisee/cnn_dailymail`). Must have a `train` split with a `text` or `messages` column. |
| `calib_max_seq_length` | int | Max sequence length for calibration (e.g. 1024). |

`quantization_type` enum (in `build` section): `fp8`, `fp8_kv`, `fp4`, `fp4_kv`, `fp4_mlp_only`, `no_quant`, `int8`.

---

## 12. LoRA Adapters

**Source:** [Engine-Builder-LLM LoRA support](https://docs.baseten.co/engines/engine-builder-llm/lora-support)

### Main concepts

- Engine-Builder-LLM supports **multi-LoRA deployments** with runtime adapter switching. Share base model weights across fine-tuned variants; switch adapters without redeployment.
- The engine shares base model weights across all adapters for memory efficiency.
- The `model` parameter in OpenAI-format requests selects which adapter to use (base `served_model_name` or an adapter name).

### Limitations

- All adapters in one deployment must share the **same rank and target modules**.
- Adapter repos must be known ahead of time (build-time availability).
- If using only one LoRA adapter, merging it into base weights provides better performance.

### Adapter repository structure (standard HuggingFace)

- `adapter_config.json` — required fields: `base_model_name_or_path`, `target_modules` (array), `r` (int, rank)
- `adapter_model.safetensors` — the LoRA adapter weights
- The engine builder converts the adapter into internal format (`.npy`) server-side at build time.

### Build configuration (config.yaml, under `trt_llm.build`)

| Parameter | Type | Default | Description |
|-----------|------|---------|-------------|
| `lora_adapters` | dict | — | Dictionary of LoRA adapters to load. Names must match `^[a-zA-Z0-9_\-\.:]+$` |
| `lora_adapters.<name>.repo` | string | — | e.g. `"username/model-name"` |
| `lora_adapters.<name>.revision` | string | — | e.g. `"main"` or commit hash |
| `lora_adapters.<name>.source` | string | — | `"HF"`, `"GCS"`, `"S3"`, `"AZURE"` |
| `lora_configuration.max_lora_rank` | int | 64 | Maximum LoRA rank. Set to exactly the `r` value used. |
| `lora_configuration.lora_target_modules` | array | `[]` (auto-detect) | Target modules. Usually auto-detected from `adapter_config.json`. |

---

## 13. Speculative Decoding (Lookahead)

**Source:** [Engine-Builder-LLM lookahead decoding](https://docs.baseten.co/engines/engine-builder-llm/lookahead-decoding)

### Main concepts

- **Lookahead decoding** provides 2x–4x faster inference for suitable workloads by predicting future tokens using n-gram patterns. Output is identical to token-by-token decoding.
- B10 Lookahead implementation searches up to 10M past tokens for n-gram matches.
- During speculative decoding, sampling is disabled and temperature is set to 0.0.
- **Chunked prefill** is NOT supported with lookahead decoding (disabled automatically).
- **Structured output** with state-machine guarantees (enforced JSON via `response_format`) is NOT possible when lookahead is enabled.
- Lookahead isn't fully supported on Blackwell; check BIS-LLM for Blackwell support.

### When to use

- Code generation (predictable syntax: function signatures, variable names, idioms)
- Prompt lookup scenarios
- General low-latency use cases (trade slightly decreased throughput for faster individual responses)

### Configuration (`trt_llm.build.speculator` block)

| Parameter | Type | Description |
|-----------|------|-------------|
| `speculative_decoding_mode` | enum | `LOOKAHEAD_DECODING` to enable. (`DRAFT_TOKENS_EXTERNAL` deprecated.) |
| `lookahead_ngram_size` | int | N-gram pattern size. Min 1. Use 4 for simple patterns, 8 for general use (recommended), 16–32 for complex. |
| `lookahead_verification_set_size` | int | Verification buffer size. Min 1. Use 1 for high-confidence, 3 for general use (recommended), 5 for complex. |
| `lookahead_windows_size` | int | Speculation window size. Min 1. Pair with `lookahead_verification_set_size`. |
| `enable_b10_lookahead` | bool | Default false. Set true for Baseten's optimized B10 lookahead algorithm (recommended). |

### Recommended configurations

| Workload | window | ngram | verification |
|----------|--------|-------|-------------|
| General purpose | 3 | 8 | 3 |
| Coding agents | 1 | 8 | 3 |
| Highly predictable content | 1 | 32 | 1 |
| Code generation (highly predictable) | 7 | 5 | 7 |

### Performance

- Up to 2x faster for code/structured content; up to 10x for prompt-lookup workloads; reaching 4000 tokens/s per request on Qwen-3-8B with single H100.
- Optimal batch size < 32 (set `max_batch_size` to 32 or 64). Smaller batches (1–8) for maximum benefit.
- No additional GPU memory required.

For Eagle/MTP model-based speculation, use BIS-LLM speculative decoding instead.

---

## 14. GPU Resources & Instance Types

**Source:** [Resources](https://docs.baseten.co/deployment/resources)

### Main concepts

- **Instance**: the allocated hardware for inference.
- **Node**: the compute unit within an instance, comprising 8 GPUs with associated vCPU, RAM, and VRAM.
- **Instance type**: the full SKU name uniquely identifying a hardware configuration. When `instance_type` is specified, other resource fields are ignored.

### config.yaml `resources` fields

| Field | Type | Description |
|-------|------|-------------|
| `accelerator` | string | GPU family, e.g. `L4`. Baseten provisions the smallest instance meeting constraints. |
| `cpu` | string | vCPU count. `"3"`/`"4"` → 4-core; `"5"`–`"8"` → 8-core. |
| `memory` | string | RAM, e.g. `16Gi` (Gibibytes). |
| `instance_type` | string | Exact SKU, e.g. `"L4:4x16"`. Overrides other fields. |
| `use_gpu` | bool | Ignored when `instance_type` is set. |

### CPU-only instances

| Instance | $/min | vCPU | RAM |
|----------|-------|------|-----|
| `1x2` | 0.00058 | 1 | 2 GiB |
| `1x4` | 0.00086 | 1 | 4 GiB |
| `2x8` | 0.00173 | 2 | 8 GiB |
| `4x16` | 0.00346 | 4 | 16 GiB |
| `8x32` | 0.00691 | 8 | 32 GiB |
| `16x64` | 0.01382 | 16 | 64 GiB |

### GPU instances

| Instance | $/min | vCPU | RAM | GPU | VRAM |
|----------|-------|------|-----|-----|------|
| `T4x4x16` | 0.01052 | 4 | 16 GiB | 1 NVIDIA T4 | 16 GiB |
| `T4x8x32` | 0.01504 | 8 | 32 GiB | 1 NVIDIA T4 | 16 GiB |
| `T4:2x24x96` | 0.03912 | 24 | 96 GiB | 2 NVIDIA T4 | 32 GiB |
| `L4:4x16` | 0.01414 | 4 | 16 GiB | 1 NVIDIA L4 | 24 GiB |
| `L4:2x24x96` | 0.04002 | 24 | 96 GiB | 2 NVIDIA L4 | 48 GiB |
| `A10Gx4x16` | 0.02012 | 4 | 16 GiB | 1 NVIDIA A10G | 24 GiB |
| `A10G:2x24x96` | 0.05672 | 24 | 94 GiB | 2 NVIDIA A10G | 48 GiB |
| `A100:12x144` | 0.06667 | 12 | 144 GiB | 1 NVIDIA A100 | 80 GiB |
| `A100:2x24x288` | 0.13334 | 24 | 288 GiB | 2 NVIDIA A100 | 160 GiB |
| `A100:4x48x576` | 0.26668 | 48 | 576 GiB | 4 NVIDIA A100 | 320 GiB |
| `A100:8x96x1152` | 0.53333 | 96 | 1152 GiB | 8 NVIDIA A100 | 640 GiB |
| `H100` | 0.10833 | 16 | 118 GiB | 1 NVIDIA H100 | 80 GiB |
| `H100:2` | 0.21666 | 32 | 236 GiB | 2 NVIDIA H100 | 160 GiB |
| `H100:4` | 0.43332 | 64 | 472 GiB | 4 NVIDIA H100 | 320 GiB |
| `H100:8` | 0.86664 | 128 | 944 GiB | 8 NVIDIA H100 | 640 GiB |
| `H100MIG` | 0.06250 | 8 | 59 GiB | Fractional H100 | 40 GiB |
| `H200` | 0.12500 | 16 | 200 GiB | 1 NVIDIA H200 | 141 GiB |
| `H200:4` | 0.50000 | 64 | 800 GiB | 4 NVIDIA H200 | 564 GiB |
| `H200:8` | 1.00000 | 128 | 1600 GiB | 8 NVIDIA H200 | 1128 GiB |
| `B200` | 0.16633 | 16 | 224 GiB | 1 NVIDIA B200 | 180 GiB |
| `B200:4` | 0.66532 | 64 | 896 GiB | 4 NVIDIA B200 | 720 GiB |
| `B200:8` | 1.33064 | 128 | 1792 GiB | 8 NVIDIA B200 | 1440 GiB |
| `RTX-PRO-6000` | 0.06667 | 16 | 116 GiB | 1 RTX Pro 6000 | 96 GiB |
| `RTX-PRO-6000:8` | 0.53336 | 128 | 931 GiB | 8 RTX Pro 6000 | 768 GiB |

H200 and B200 instances available on request (contact support@baseten.co).

### instance_type naming conventions

- Single L4 or A100: `:x` format, e.g. `"L4:4x16"`
- Single T4 or A10G: `xx` format (no colon), e.g. `"T4x4x16"`, `"A10Gx8x32"`
- Multi-GPU: `:xx` format, e.g. `"A100:2x24x288"`
- H100/H200/B200/RTX-PRO-6000: `:N` format, e.g. `"H100:2"`
- Fractional H100: `"H100MIG"`

### GPU details

| GPU | Architecture | VRAM | Bandwidth | Best for |
|-----|-------------|------|-----------|----------|
| T4 | Turing | 16 GiB | — | Whisper, small LLMs (StableLM 3B) |
| L4 | Ada Lovelace | 24 GiB | 300 GiB/s | Stable Diffusion XL (not suitable for LLMs due to bandwidth) |
| A10G | Ampere | 24 GiB | 600 GiB/s | Mistral 7B, Whisper, SDXL |
| A100 | Ampere | 80 GiB | 1.94 TB/s | Mixtral, Llama 2 70B (2 A100s), Falcon 180B (5 A100s) |
| H100 | Hopper | 80 GiB | 3.35 TB/s | Mixtral 8x7B, Llama 2 70B (2xH100) |
| H100MIG | Hopper (fractional) | 40 GiB | 1.675 TB/s | Efficient LLM inference at lower cost than A100 |
| H200 | Hopper | 141 GiB | — | Large models needing more VRAM |
| B200 | Blackwell | 180 GiB | — | Largest models, FP4 quantization |
| RTX Pro 6000 | Blackwell | 96 GiB | — | Vision-language models, mid-size LLMs |

---

## 15. Deployment Lifecycle & Environments

**Source:** [Manage lifecycle](https://docs.baseten.co/deployment/manage/lifecycle), [Deployments](https://docs.baseten.co/deployment/deployments)

### Deployment statuses

| Status | Description |
|--------|-------------|
| `ACTIVE` | Serving traffic |
| `INACTIVE` | Deactivated; not serving |
| `DEPLOYING` | Building/starting |
| `SCALED_TO_ZERO` | No replicas running (min_replica=0, no traffic) |
| `WAKING_UP` | Transitioning from scaled-to-zero to active |

### API functions

| Function | Method & Endpoint | Purpose |
|----------|-------------------|---------|
| Promote to production | `POST /v1/models/{model_id}/deployments/{deployment_id}/promote` | Promote a deployment to the production environment |
| Promote to an environment | `POST /v1/models/{model_id}/environments/staging/promote` | Promote to a specific (custom) environment. Body: `{"deployment_id": "{deployment_id}"}` |
| Deactivate a deployment | `POST /v1/models/{model_id}/deployments/{deployment_id}/deactivate` | Stop compute spend. Returns `{"success": true}` |
| Activate a deployment | `POST /v1/models/{model_id}/deployments/{deployment_id}/activate` | Bring an inactive deployment back |
| Delete a deployment | `DELETE /v1/models/{model_id}/deployments/{deployment_id}` | Permanently delete. Returns `{"id", "deleted": true, "model_id"}` |
| Delete a model | `DELETE /v1/models/{model_id}` | Permanently delete a model (removes all deployments) |

- **Deactivate**: stops compute spend without deleting; requests fail with `400` until reactivated.
- **Delete**: irreversible; requests return `404`. A deployment associated with an environment, or the only deployment of a model, can't be deleted (promote a replacement first).
- **Activate**: re-deploys the model (passes through `DEPLOYING` before `ACTIVE`).

---

## 16. Autoscaling & Scale-to-Zero

**Source:** [Scale a deployment](https://docs.baseten.co/deployment/manage/scaling), [Autoscaling engines](https://docs.baseten.co/engines/performance-concepts/autoscaling-engines), [Traffic patterns](https://docs.baseten.co/deployment/autoscaling/traffic-patterns)

### Autoscaling settings

| Setting | Type | Description |
|---------|------|-------------|
| `min_replica` | int | Minimum replica count. Set to `0` for scale-to-zero. |
| `max_replica` | int | Maximum replica count. |
| `autoscaling_window` | int (seconds) | Lookback window for averaging traffic |
| `scale_down_delay` | int (seconds) | Delay after traffic drops before removing replicas. Default 900 (15 min). |
| `concurrency_target` | int | Concurrent requests per replica before adding another |
| `target_utilization_percentage` | int | Share of concurrency target at which scaling triggers |
| `max_scale_down_rate` | int | Largest percentage of replicas removed in a single scale-down step (console only) |

### API functions

| Function | Method & Endpoint | Purpose |
|----------|-------------------|---------|
| Update autoscaling settings | `PATCH /v1/models/{model_id}/deployments/{deployment_id}/autoscaling_settings` | Apply updates (any subset of fields; applies asynchronously). Returns `{"status": "ACCEPTED", ...}` |
| Get deployment | `GET /v1/models/{model_id}/deployments/{deployment_id}` | Fetch deployment incl. `autoscaling_settings` |
| Wake a deployment | `POST https://model-{model_id}.api.baseten.co/deployment/{deployment_id}/wake` | Wake a scaled-to-zero deployment. Returns HTTP 202. |

### Scale-to-zero

- Set `min_replica` to `0` so an idle deployment releases all replicas and stops billing for compute.
- Not recommended for production (first request after idle pays a cold start). Reserve for dev/staging or delay-tolerant workloads.
- **Wake** before you need it. A woken deployment with `min_replica: 0` scales back down after `scale_down_delay` if no requests arrive.

### Engine-specific autoscaling

**BEI** (recommendations):
- Target utilization: 25% (headroom for spikes)
- Concurrency target: 96+ (min ≥ 8) for max throughput
- Use the Performance Client for multi-payload routes (`/rerank`, `/v1/embeddings`)

**Engine-Builder-LLM** (recommendations):
- Target utilization: 40–50%
- Concurrency target: 32–256 (match or stay below `max_batch_size`)
- `concurrency_target` ≤ `max_batch_size` always (otherwise on-replica queueing)

**BIS-LLM** (token-based autoscaling):
- Scales on `target_in_flight_tokens` (not request count). The deployment API rejects `concurrency_target` and `target_utilization_percentage`.
- Scale-to-zero is **not** supported.
- In-flight token = prefill tokens (uncached input being processed) + decode tokens (full sequence length for every request decoding).
- `desired_replicas = avg_in_flight_tokens / target_in_flight_tokens`

BIS-LLM config (config.yaml):
```yaml
autoscaling_settings:
  min_replica: 1
  max_replica: 4
  autoscaling_window: 300
  scale_down_delay: 300
additional_autoscaling_config:
  metrics:
    - name: in_flight_tokens
      target: 40000
```

`scale_down_half_life_seconds` (exponential decay to prevent abrupt KV cache loss; default 900, recommended 600–1800) is configured in `b10_autoscaling_config` inside the `llm_config` block of the Management API (`POST /v1/llm_models`), not in config.yaml.

### Traffic pattern recommendations

| Pattern | Autoscaling window | Scale-down delay | Target util | Min replicas |
|---------|-------------------|------------------|-------------|--------------|
| Jittery (small frequent spikes) | 2–5 min | 300–600s | 70% | — |
| Bursty (sharp jumps, sustained) | 30–60s | 900s+ | 50–60% | ≥ 2 |
| Scheduled (long low/zero, predictable bursts) | 60–120s | moderate–high | 70% | 0 (if cold starts OK) or 1 (during job windows) |
| Steady (gradual rises/falls) | 60–120s | 300–600s | 70–80% | ≥ 2 |

---

## 17. Cold Starts & BDN (Baseten Delivery Network)

**Source:** [Cold starts](https://docs.baseten.co/deployment/autoscaling/cold-starts), [BDN](https://docs.baseten.co/development/model/bdn)

### Cold start concepts

- **Cold start**: the time a fresh replica spends starting up before it can accept traffic.
- **Two trigger types**: scale-from-zero (when `min_replica == 0`) and scaling events (adding replicas under load).
- **Contributing factors** (in order): container pull (rarely dominates; Baseten streams image in background), weight load (dominant for 70B+/large MoE), engine initialization (CUDA graph capture, `torch.compile`, KV cache profiling — dominates for small models).

### BDN (Baseten Delivery Network)

Reduces cold start times by mirroring model weights to Baseten's infrastructure and caching them close to replicas. Configure via the `weights` key in config.yaml.

- On `truss push`, BDN reads the weights config, mirrors files into Baseten's secure blob storage, writes a manifest of content hashes. Files keyed by hash (deduplication); each deployment mounts only its manifest's files.
- `truss push` returns immediately; mirroring runs in background; model deploys only after mirroring completes.
- Multi-tier caching: blob storage → in-cluster cache → node cache → mounted read-only.
- No upstream dependency at runtime (once mirrored, scale-ups never contact the original source).
- Used automatically for BEI, Engine-Builder-LLM, and BIS-LLM on every deploy (no `weights` block required).

### weights block (config.yaml)

| Field | Type | Description |
|-------|------|-------------|
| `source` | string (required) | URI: `hf://`, `bt://`, `s3://`, `gs://`, `r2://`, `cw://`, `azure://`. HF revision via `@revision` suffix. |
| `mount_location` | string (required) | Absolute path (must start with `/`). Unique per source. |
| `auth` | object | Auth for private/gated sources. `auth_method`: `CUSTOM_SECRET` \| `AWS_OIDC` \| `GCP_OIDC`; `auth_secret_name`. |
| `allow_patterns` | string[] | File patterns to include (Unix shell-style). Use `**/*.safetensors` for subdirs. |
| `ignore_patterns` | string[] | File patterns to exclude. |

Re-mirrors if `source`, `allow_patterns`, `ignore_patterns`, or `auth` change. Does NOT re-mirror on `mount_location` change.

### Reducing cold starts

- **Warm replicas**: keep `min_replica >= 1`.
- **Pre-warming**: raise `min_replica` ahead of predictable spikes (10–15 min before).
- **Scale-down delay**: longer delay keeps replicas warm through brief dips.
- **BDN**: automatic on engine-builder deployments; for other deployments, add a `weights` block.
- **Compilation caching** (b10cache): persists `torch.compile` and CUDA graph artifacts so a new replica loads them instead of recompiling (cuts compilation from minutes to ~5–20 seconds).

---

## 18. Request Lifecycle, Queuing & Retries

**Source:** [Request lifecycle](https://docs.baseten.co/deployment/autoscaling/request-lifecycle)

### Request flow

1. **Inference gateway**: authenticates the API key. Failure → `401 Unauthorized`.
2. **Routing layer**: decides which replica handles the request. Routes to the least-utilized replica based on how full it is relative to its concurrency target (prefers replicas with headroom).
3. **Replica**: runs inference (predict function). Response flows back through the same path.

### Parking

When no replica is available (scaled to zero, or all at capacity while autoscaler brings up new ones), Baseten **parks** the request at the routing layer and waits for a replica. Once ready, the parked request is forwarded normally. Makes scale-to-zero practical. If no replica becomes available before the predict timeout (1200s default), the parked request fails with `500`.

### Async request behavior

First async request parks and waits like sync. Subsequent async requests arriving while there's no capacity receive an immediate `429` with `CAPACITY_EXCEEDED` instead of `202 Accepted` (prevents clients polling for results still waiting for a replica). Async requests are **not** retried.

### Request queuing

When all replicas are at their concurrency target and the autoscaler hasn't finished adding new ones, requests queue at the routing layer (automatic, invisible to the client).

### Load shedding

Safety valve rejecting new requests with `429` if queued payloads exceed a memory threshold (rarely triggers under normal conditions).

### Internal retries

When a replica returns `502`/`503`/`504`, the routing layer retries automatically using exponential backoff starting at 500ms, growing by factor 1.5 up to 60s between attempts. For status code errors, retries continue until the request deadline or 15 minutes total elapsed. Connection-level failures capped at 16 attempts.

- Retries appear as added latency, not errors.
- Check `X-BASETEN-MODEL-PREDICTION-ATTEMPTS` response header (value > 1 = at least one retry).
- Under memory pressure (>80% utilization on routing layer), a circuit breaker disables retries entirely, resuming after 30-second cooldown.
- Sticky session `503` retries route to a different replica.

### Request cancellation

When a client disconnects before the response is written, the routing layer detects the closed connection and cancels in-flight work (logged as `499`). Works across engines including TRT-LLM and vLLM.

### Streaming timeouts

HTTP headers (including 200 status) are sent when the stream begins. If timeout expires mid-stream, the stream stops and the connection closes without an error code (most HTTP clients surface this as connection reset / incomplete response).

---

## 19. Rolling Deployments

**Source:** [Rolling deployments](https://docs.baseten.co/deployment/rolling-deployments)

### Main concepts

Gradually shift traffic to a new deployment with replica-based rolling deployments. Scale up the candidate, shift traffic proportionally, scale down the previous deployment in controlled steps. Disabled by default; enable per environment. Not supported for Chains.

**Three-step cycle (repeating):**
1. Scale up candidate replicas by the configured percentage.
2. Shift traffic proportionally to match the new replica ratio.
3. Scale down the previous deployment by the same percentage.

### Provisioning modes (mutually exclusive)

- `max_surge_percent`: scales up candidate BEFORE scaling down previous.
- `max_unavailable_percent`: scales down previous BEFORE scaling up candidate.

### Deployment statuses (in_progress_promotion field)

| Status | Description |
|--------|-------------|
| `RELEASING` | Candidate building & initializing replicas |
| `RAMPING_UP` | Scaling up candidate + shifting traffic |
| `PAUSED` | Paused at current traffic split |
| `RAMPING_DOWN` | Graceful cancel in progress |
| `SUCCEEDED` | Candidate now active |
| `FAILED` | Traffic remains on previous |
| `CANCELED` | Traffic returned to previous |

### API functions

| Function | Method & Endpoint | Purpose |
|----------|-------------------|---------|
| Enable rolling deployments | `PATCH /v1/models/{model_id}/environments/production` | Body: `{promotion_settings: {rolling_deploy: true, rolling_deploy_config: {...}}}` |
| Set cleanup strategy | `PATCH /v1/models/{model_id}/environments/production` | Body: `{promotion_settings: {promotion_cleanup_strategy: "DEACTIVATE"}}` |
| Pause | `POST /v1/models/{model_id}/environments/production/pause_promotion` | Pause at current traffic split |
| Resume | `POST /v1/models/{model_id}/environments/production/resume_promotion` | Resume a paused rollout |
| Cancel (graceful) | `POST /v1/models/{model_id}/environments/production/cancel_promotion` | Traffic ramps back to previous |
| Force cancel | `POST /v1/models/{model_id}/environments/production/force_cancel_promotion` | Immediate cancel (may cause brief disruption) |
| Force roll forward | `POST /v1/models/{model_id}/environments/production/force_roll_forward_promotion` | Immediately shift all traffic to candidate |

### rolling_deploy_config parameters

| Parameter | Type | Default | Range | Description |
|-----------|------|---------|-------|-------------|
| `max_surge_percent` | int | 10 | 0–50 | Additional replicas to provision per step. Set 0 for max_unavailable mode. |
| `max_unavailable_percent` | int | 0 | 0–50 | Replicas that can be unavailable per step. Set 0 for max_surge mode. |
| `stabilization_time_seconds` | int | 0 | 0–3600 | Wait between steps to monitor metrics |
| `replica_overhead_percent` | int | 0 | 0–500 | Pre-provision headroom on previous deployment (for non-autoscaling envs) |

`promotion_cleanup_strategy` enum: `SCALE_TO_ZERO` (default) | `KEEP` | `DEACTIVATE`.

---

## 20. Health Checks

**Source:** [Health checks](https://docs.baseten.co/development/model/health-checks)

### Main concepts

Baseten runs health checks every 10 seconds on each replica. Three Kubernetes probes: startup, readiness, liveness.

- **Startup probe**: confirms model finished initializing. For Truss models, complete when `load()` finishes and optional `is_healthy()` passes. Runs 30 min by default (extendable to 50 min). All readiness/liveness probes delayed until startup succeeds.
- **Readiness probe**: determines whether a replica can accept traffic. On failure, Kubernetes stops routing but does NOT restart.
- **Liveness probe**: determines whether a replica is still functioning. On failure, Kubernetes restarts the replica.
- **Custom health check logic**: define via `is_healthy()` method in Model class or Chainlet.

### config.yaml parameters (under `runtime.health_checks`)

| Parameter | Type | Default | Range | Description |
|-----------|------|---------|-------|-------------|
| `startup_threshold_seconds` | int | 1800 | 10–3000 | How long startup phase runs. Set to 2× worst observed cold start. |
| `stop_traffic_threshold_seconds` | int | 1800 | 10–3000 | How long checks must fail before stopping traffic. Recommend 60s. |
| `restart_threshold_seconds` | int | 1800 | 10–3000 | How long checks must fail before restarting. Recommend 1.5× stop_traffic. |

Custom `is_healthy()` only runs after `load()` completes. A 5xx-detection pattern: set `self._is_healthy = False` in `predict()` exception handler; `is_healthy()` returns `self._is_healthy`.

---

## 21. Model Configuration (config.yaml)

**Source:** [Configuration](https://docs.baseten.co/development/model/configuration), [Dependencies](https://docs.baseten.co/development/model/dependencies)

### Core config.yaml fields

| Field | Type | Description |
|-------|------|-------------|
| `model_name` | string | Model display name |
| `python_version` | string | e.g. `py310` |
| `environment_variables` | map | Env vars in the model serving environment |
| `requirements` | list | Inline Python packages (pin with `==`). Mutually exclusive with `requirements_file`. |
| `requirements_file` | string | Path to `requirements.txt`, `pyproject.toml`, or `uv.lock` |
| `system_packages` | list | apt-installable Debian packages |
| `resources` | object | Hardware resources (see §14) |
| `external_package_dirs` | list | Relative paths to external package dirs |
| `build_commands` | list | Shell commands run at build time (cached) |
| `base_image` | object | Custom Docker base image |
| `secrets` | map | Secret name → null (declares secrets referenced by config) |
| `weights` | list | Weight sources to mount (BDN) |

### Engine build configuration (trt_llm block)

| Field | Type | Description |
|-------|------|-------------|
| `trt_llm.inference_stack` | enum | `v1` (Engine-Builder-LLM/BEI) or `v2` (BIS-LLM) |
| `trt_llm.build.base_model` | enum | `decoder` (text gen), `encoder` (BEI causal), `encoder_bert` (BEI-Bert) |
| `trt_llm.build.checkpoint_repository.source` | string | `HF` |
| `trt_llm.build.checkpoint_repository.repo` | string | HF repo ID |
| `trt_llm.build.checkpoint_repository.revision` | string | Branch/tag/commit |
| `trt_llm.build.runtime_secret_name` | string | Secret name for HF token |
| `trt_llm.build.max_seq_len` | int | Max sequence length (e.g. 131072) |
| `trt_llm.build.max_batch_size` | int | Max batch size (e.g. 256) |
| `trt_llm.build.max_num_tokens` | int | Max tokens per batch (e.g. 8192) |
| `trt_llm.build.quantization_type` | enum | See §11 |
| `trt_llm.build.tensor_parallel_count` | int | v1 field (renamed to `tensor_parallel_size` in v2) |
| `trt_llm.build.num_builder_gpus` | int | GPUs for BF16 quantization loading |
| `trt_llm.build.plugin_configuration.paged_kv_cache` | bool | Enable paged KV cache |
| `trt_llm.build.plugin_configuration.use_paged_context_fmha` | bool | Enable paged context FMHA |
| `trt_llm.build.plugin_configuration.use_fp8_context_fmha` | bool | Enable FP8 context FMHA |
| `trt_llm.runtime.kv_cache_free_gpu_mem_fraction` | float | e.g. 0.9 |
| `trt_llm.runtime.enable_chunked_context` | bool | Enable chunked prefill |
| `trt_llm.runtime.batch_scheduler_policy` | string | e.g. `guaranteed_no_evict` |
| `trt_llm.runtime.served_model_name` | string | Model name selectable at runtime (base for LoRA) |
| `trt_llm.runtime.webserver_default_route` | enum | BEI: `/v1/embeddings`, `/rerank`, `/predict`, `/predict_tokens` |

### Private registry auth (base_image.docker_auth)

| Method | Provider | Fields |
|--------|----------|--------|
| `AWS_OIDC` (recommended) | AWS ECR | `aws_oidc_role_arn`, `aws_oidc_region`, `registry` |
| `AWS_IAM` | AWS ECR | secrets: `aws_access_key_id`, `aws_secret_access_key`; `registry` |
| `GCP_OIDC` (recommended) | GCR/Artifact Registry | `gcp_oidc_service_account`, `gcp_oidc_workload_id_provider`, `registry` |
| `GCP_SERVICE_ACCOUNT_JSON` | GCR | `secret_name` (JSON key), `registry` |

Other registries (Docker Hub, GHCR, NVIDIA NGC): base64-encoded `username:password` secret named `DOCKER_REGISTRY_<hostname>`.

---

## 22. Performance Client

**Source:** [Performance client](https://docs.baseten.co/inference/performance-client)

### Main concepts

- **Performance Client**: high-performance client library built in Rust, integrated with Python, Node.js, and native Rust. Releases the Python GIL while executing requests, enabling simultaneous sync and async usage.
- Benchmarks: 1200+ requests per second per client. Works with Baseten deployments or third-party providers (e.g. OpenAI) by replacing `base_url`.
- Compatible with BEI, BEI-Bert, Engine-Builder-LLM, text-embeddings-inference endpoints.
- Install: Python `uv pip install baseten_performance_client>=0.1.0`; JS `npm install @basetenlabs/performance-client`.
- **HttpClientWrapper**: enables connection pooling shared across multiple client instances.
- **CancellationToken**: cancel long-running operations; immediate cancellation, resource cleanup, Ctrl+C support, `is_cancelled()` status check.
- **Hedge delay**: sends duplicate requests after a specified delay to reduce p99 latency (clones and races request after delay).
- 429 and 5xx errors are always retried automatically.

### Client methods

| Function | Method | Purpose |
|----------|--------|---------|
| Embeddings | `client.embed(input, model, preference)` / `await client.async_embed(...)` | Generate embeddings with batching/concurrency. Returns `.model`, `.usage.total_tokens`, `.total_time`, `.numpy()` |
| Generic batch POST | `client.batch_post(url_path, payloads, preference, method)` / `await client.async_batch_post(...)` | Send HTTP requests to any URL with any JSON payload. Methods: GET, POST, PUT, PATCH, DELETE, HEAD, OPTIONS |
| Reranking | `client.rerank(query, texts, model, return_text, preference)` / `await client.async_rerank(...)` | Rerank documents by relevance. Response `.data` has `index`/`score` |
| Classification | `client.classify(inputs, model, preference)` / `await client.async_classify(...)` | Classify text inputs. Response `.data` is groups of results with `label`/`score` |

### PerformanceClient constructor

| Parameter | Type | Default | Description |
|-----------|------|---------|-------------|
| `base_url` | string | — | Endpoint URL |
| `api_key` | string | — | API key (also from `BASETEN_API_KEY` / `OPENAI_API_KEY`) |
| `http_version` | int | 1 | 1 = HTTP/1.1 (better for high concurrency); 2 = HTTP/2 (not recommended on Baseten) |
| `client_wrapper` | HttpClientWrapper | — | Shared connection pool |

### RequestProcessingPreference parameters

| Parameter (Python / JS) | Type | Default | Range | Description |
|--------------------------|------|---------|-------|-------------|
| `max_concurrent_requests` / `maxConcurrentRequests` | int | 128 | 1–1024 | Maximum parallel requests |
| `batch_size` / `batchSize` | int | 128 | 1–1024 | Items per batch |
| `timeout_s` / `timeoutS` | float | 3600.0 | 1.0–7200.0 | Per-request timeout (seconds) |
| `hedge_delay` / `hedgeDelay` | float | None | 0.2–30.0 | Hedge delay (seconds) |
| `hedge_budget_pct` / `hedgeBudgetPct` | float | 0.10 | 0.0–3.0 | Percentage of requests for hedging |
| `retry_budget_pct` / `retryBudgetPct` | float | 0.05 | 0.0–3.0 | Percentage of requests for retries |
| `max_retries` / `maxRetries` | int | 5 | 0–6 | Max HTTP status-code retries per request |
| `initial_backoff_ms` / `initialBackoffMs` | int | 125 | 50–45000 | Initial retry backoff (ms) |
| `non_retryable_status_codes` / `nonRetryableStatusCodes` | set | None | — | HTTP codes excluded from retry (e.g. `{529}`) |
| `total_timeout_s` / `totalTimeoutS` | float | None | ≥ timeout_s | Total operation timeout |
| `extra_headers` | dict | None | — | Custom headers |
| `max_chars_per_request` (Python) | int | — | — | e.g. 10000 for embed, 256000 for rerank/classify |

### Retry behavior

- Retries status codes 408, 409, 429, and 500–599 by default.
- Set `max_retries` to 0 to disable.
- Backoff starts at `initial_backoff_ms`, multiplies by 4 after each retry, caps at 45000 ms, adds up to 99 ms jitter.

### Automatic timeout headers (derived from `timeout_s`)

- `Request-Timeout-Ms`: relative timeout in milliseconds, rounded up
- `Request-Deadline-Ms`: absolute deadline as Unix timestamp in milliseconds

### Environment variables

| Variable | Description |
|----------|-------------|
| `BASETEN_API_KEY` | Baseten API key (also checks `OPENAI_API_KEY` as fallback) |
| `PERFORMANCE_CLIENT_LOG_LEVEL` | Logging level: trace, debug, info, warn (default), error |
| `PERFORMANCE_CLIENT_REQUEST_ID_PREFIX` | Custom prefix for request IDs (default `perfclient`) |

---

## 23. Frontier Gateway — Managed API Gateway

**Source:** [Frontier Gateway Overview](https://docs.baseten.co/frontier-gateway/overview), [Get started](https://docs.baseten.co/frontier-gateway/get-started), [Endpoints](https://docs.baseten.co/frontier-gateway/endpoints), [Groups & API keys](https://docs.baseten.co/frontier-gateway/api-keys), [Rate & usage limits](https://docs.baseten.co/frontier-gateway/rate-limits), [Billing webhooks](https://docs.baseten.co/frontier-gateway/billing-webhooks), [Gateway API Overview](https://docs.baseten.co/reference/gateway/overview)

### Main concepts

A managed API gateway for AI labs to serve hosted models under a branded URL with hierarchical groups, inherited rate and usage limits, and billing webhooks. Sits on top of an existing Dedicated deployment.

- **Endpoint**: a routing slug (e.g. `my-org/glm-5.2`) and the target it points to. Self-service via REST API.
- **Group**: one node in your organizational hierarchy. Owns an external identifier, model set (endpoint slugs allowed), rate/usage limits, and place in the hierarchy. Every API key belongs to a group. Groups can nest under a parent group; limits flow down the tree.
- **Federated API key**: credential bound to one group; inherits the group's effective model set and limits (no per-key overrides).
- **Effective models**: read-only block in the group response showing limits the runtime enforces after walking the hierarchy.
- **Two inheritance modes** (immutable after creation, fixed for whole hierarchy, capped at 5 levels):
  - **INDEPENDENT**: child inherits ancestor limits when omitted; child's usage metered separately; child can override thresholds up or down. Acts as a template.
  - **CASCADING**: child's usage counts against every ancestor simultaneously. Children can't declare a threshold higher than any ancestor's for the same (slug, type, unit). Acts as a shared pool.

### Base URLs

- Management: `https://api.baseten.co/v1/gateway/` (workspace API key with management scope; non-onboarded workspaces return `403`)
- Inference: `https://inference.baseten.co/v1` (OpenAI-compatible; becomes your branded domain once white-label routing is provisioned)
- OpenAPI spec: `https://api.baseten.co/v1/spec`

### Endpoints API

| Function | Method & Endpoint | Purpose |
|----------|-------------------|---------|
| Create an endpoint | `POST /v1/gateway/endpoints` | Map a slug to a target. Body: `slug`, `targets` (list with one target: `provider`, `model_id`, optional `environment_name`) |
| List endpoints | `GET /v1/gateway/endpoints` | Cursor-paginated (`limit`, `cursor` query params) |
| Get an endpoint | `GET /v1/gateway/endpoints/{endpoint_id}` | Retrieve by ID |
| Update (re-point) an endpoint | `PATCH /v1/gateway/endpoints/{endpoint_id}` | Replace target list (slug stays same; syncs every 60s) |
| Delete an endpoint | `DELETE /v1/gateway/endpoints/{endpoint_id}` | Take a slug out of service |

Endpoint target:
- `provider`: `BASETEN` (with `model_id` and optional `environment_name`) or `ANTHROPIC` / `OPENAI` (external provider)
- An endpoint holds exactly one target today.

### Groups API

| Function | Method & Endpoint | Purpose |
|----------|-------------------|---------|
| Create a group | `POST /v1/gateway/groups` | Create a group with `metadata`, `models`, `hierarchy` |
| List groups | `GET /v1/gateway/groups` | Cursor-paginated; supports `?external_entity_id=` lookup and `?cursor=` |
| Get a group | `GET /v1/gateway/groups/{group_id}` | Retrieve by ID |
| Get group usage | `GET /v1/gateway/groups/{group_id}/usage` | Inspect usage against configured usage_limits |
| Update a group | `PATCH /v1/gateway/groups/{group_id}` | Change `metadata.name` and/or `models` (hierarchy is immutable) |
| Delete a group | `DELETE /v1/gateway/groups/{group_id}` | Remove group, revoke all keys, recursively remove descendants |

Group body fields:
- `metadata.name` (string, optional): human-readable display name
- `metadata.external_entity_id` (string, required on create): stable unique identifier; appears as `externalEntityId` on billing webhook events
- `models[]` (array, required on create/non-empty): each entry has `slug`, `rate_limits[]`, `usage_limits[]`
- `hierarchy.limit_enforcement` (enum: `INDEPENDENT` | `CASCADING`, required): immutable
- `hierarchy.parent_group_id` (string|null, optional): immutable

### API keys API

| Function | Method & Endpoint | Purpose |
|----------|-------------------|---------|
| Mint an API key | `POST /v1/gateway/groups/{group_id}/api_keys` | Body: `{"name": "..."}`. Returns plaintext key once. |
| Register an existing key | `POST /v1/gateway/groups/{group_id}/api_keys/register` | Register a key you control (32–128 chars, ≥3.0 bits Shannon entropy/char). Requires Ed25519 `X-Baseten-Signature` header. |
| List a group's keys | `GET /v1/gateway/groups/{group_id}/api_keys` | Cursor-paginated; only `prefix` and `name` returned |
| Get an API key | `GET /v1/gateway/groups/{group_id}/api_keys/{api_key_prefix}` | Fetch by prefix |
| Revoke a key | `DELETE /v1/gateway/groups/{group_id}/api_keys/{api_key_prefix}` | Irreversibly cut off a credential |

### Rate & usage limits

**Rate limits** (`models[].rate_limits[]`):
- `type` (enum: `TOKEN` | `REQUEST`)
- `unit` (enum: `SECOND` | `MINUTE`)
- `threshold` (int ≥ 1)

**Usage limits** (`models[].usage_limits[]`):
- `type` (enum: `TOKEN` | `REQUEST`)
- `unit` (enum: `DAY` — only supported window)
- `threshold` (int ≥ 1)

Limits accept shorthand: `1k`, `60k`, `1M`, `1B`.

**Enforcement**: when a request exceeds any limit on the request's `effective_models` for the requested slug, the platform rejects with `429 Too Many Requests`. Daily usage windows reset at midnight UTC.

**Get group usage response:**
- `customer_id` (string): the external entity id
- `usage` (object): map keyed by model slug; each entry is an array of limit-consumption objects: `type`, `unit`, `threshold`, `current_usage`, `reset_at` (ISO 8601)
- Only models with `usage_limits` appear; rate-limit consumption not surfaced.

**Cascading mode validation** (fails with 400 "Child group exceeds parent group limit"):
- Creating a child whose threshold exceeds an ancestor's for the same (slug, type, unit)
- Raising a descendant past an ancestor via PATCH
- Lowering an ancestor below the highest existing descendant
- To raise a subtree: raise ancestor first, then descendants. To lower an ancestor: lower descendants first.

### Billing webhooks

For each inference request through Frontier Gateway, Baseten emits a signed webhook to your endpoint with token counts, the calling group's external identifier, the API key that made the request, and request metadata. Standard Baseten envelope: `type` (discriminator) + `data` (event-specific fields). Event type: `API_BILLING_USAGE`.

**Event fields (`data.events[]`):**

| Field | Type | Description |
|-------|------|-------------|
| `idempotencyKey` | string | Stable identifier for deduplication (at-least-once delivery) |
| `timestamp` | string (ISO 8601 UTC) | Timestamp of the inference request |
| `requestId` | string | Per-request identifier for correlation |
| `requestMetadata` | object \| null | Freeform JSON passed through from the inference request |
| `modelSlug` | string | The model slug invoked |
| `externalEntityId` | string | The `metadata.external_entity_id` of the owning group |
| `apiKeyPrefix` | string | Prefix of the federated API key |
| `tokens.inputTokens` | int | Prompt tokens |
| `tokens.outputTokens` | int | Generated tokens |
| `tokens.cachedInputTokens` | int | Prompt tokens served from cache (when applicable) |

**Headers (set by Baseten on every delivery):**
- `X-Baseten-Signature`: `v1=<hex>` HMAC-SHA256 of raw request body computed with the workspace's webhook signing secret. Verify against raw bytes (not re-serialized JSON); use constant-time comparison.
- `X-Baseten-Request-ID`: UUID per outbound delivery (correlation ID; same across retry attempts of one delivery; use `idempotencyKey` to dedupe).

**Delivery semantics:**
- Per-attempt timeout: 10 seconds
- Backoff: exponential, starting at 1s, capping at 5s
- Max elapsed time: 15s; after this, event routed to dead-letter queue (not redelivered automatically)
- 4xx responses are terminal (stop retries); only 5xx, network errors, and timeouts trigger retry

### Inference through the gateway

OpenAI-compatible. Use the OpenAI SDK with the gateway base URL and a federated key:

```python
client = OpenAI(base_url="https://inference.baseten.co/v1", api_key="YOUR_FEDERATED_KEY")
response = client.chat.completions.create(
    model="your-org/your-model",
    messages=[{"role": "user", "content": "Hello, world!"}],
)
```

---

## 24. File I/O & Integrations

**Source:** [Model I/O with files](https://docs.baseten.co/inference/output-format/files), [Integrations](https://docs.baseten.co/inference/integrations)

### File-based I/O

- **Truss CLI `-f` flag**: passes file input to `truss predict`. Only works for JSON-serializable content.
- **Base64 encoding**: for non-JSON-serializable files (e.g. audio WAV), base64-encode the content and include it in the JSON payload.
- **`preprocess()` function**: a Truss hook to load file content from a URL without blocking the GPU.
- File outputs (audio/image): response fields contain base64-encoded data that must be decoded.

### Integrations

Baseten works with external tools to call Model APIs and deployed models: Claude Code, Cline, Fused, HumanLayer, LangChain, Linkup, LiteLLM, LiveKit, LlamaIndex, NemoClaw, OpenClaw, Roo Code, Vercel (AI SDK v5).

---

## 25. Errors & Status Codes

**Source:** [Inference errors](https://docs.baseten.co/inference/errors)

Every failed response is JSON: `{"error": "..."}`. The HTTP status code indicates the broad category; the error body is a Baseten-generated string or the raw model response passed through.

### Key distinction

- **Model returned the error** (status code + body from your model server — handler exception, non-2xx, internal timeout): debug like an app bug.
- **Model unreachable** (failure in front of container — "Error making prediction" with 502/503): container down/restarting/killed/cold-starting.

### HTTP status codes

| Status | Meaning | When it occurs | What to do |
|--------|---------|-----------------|------------|
| 200 | Success | Normal predict response | None |
| 202 | Accepted | Async predict request queued successfully | Poll or wait for webhook |
| 400 | Bad Request | Malformed request or URL | Check valid JSON body, correct path |
| 401 / 403 | Unauthorized/Forbidden | Invalid or missing API key | Use valid Baseten API key in Authorization header |
| 402 | Payment Required | Unresolved billing/payment issue | Check billing or contact account owner |
| 404 | Not Found | Model/deployment ID doesn't exist or was deleted | Confirm ID and correct predict endpoint |
| 413 | Payload Too Large | Request body too large (100 MB edge cap, or 256 KiB async default) | Send large inputs by file or URL, not inline |
| 429 | Too Many Requests | Rate limit exceeded (Model APIs RPM/TPM, async endpoint limits, or capacity unavailable on arrival for async) | Back off and retry; persistent → increase max_replica or concurrency target |
| 499 | Client Closed Request | Client disconnected before response written | Review client-side timeout config |
| 500 | Internal Server Error | Sync request's parking timeout expired before a replica became available | Retry after brief wait; if persistent, increase max_replica or keep min_replica > 0 |
| 502 | Bad Gateway | Container crashed/restarted/killed (OOMKilled) mid-request, or Baseten-side error | Check model logs; if clean, retry with exponential backoff |
| 503 | Service Unavailable | Container not available yet (draining or routing) | Retry with exponential backoff (transient) |
| 504 | Gateway Timeout | Request exceeded server-side predict timeout (1200s sync) | Profile model, raise resources, or use async (up to 3600s) |

### 413 limits

- **Edge limit**: 100 MB (fixed, not configurable) — applies to all requests.
- **Async limit**: 256 KiB (262,144 bytes) by default per-org — raise via support. 100 MB edge is fixed.

### Streaming and async errors

- **Streaming**: starts with 200; failure partway through can't be sent as 5xx. Stream ends early — treat as failed request, check model logs, retry.
- **Async**: failure not reported on submit response. Errors reported later in the webhook payload's `errors` array (empty on success).

---

## 26. API Endpoint Reference Summary

### Model APIs (serverless) — `https://inference.baseten.co/v1`

Auth: `Authorization: Bearer $BASETEN_API_KEY`

| Concern | Method & Endpoint | Purpose |
|---------|-------------------|---------|
| Chat completions | `POST /v1/chat/completions` | OpenAI-compatible inference |
| Anthropic messages (beta) | `POST /v1/messages` | Anthropic-compatible inference |
| List models | `GET /v1/models` | Current catalog with metadata |

### Inference (dedicated deployments) — `https://model-{model_id}.api.baseten.co`

Auth: `Authorization: Bearer $BASETEN_API_KEY`

| Concern | Method & Endpoint | Purpose |
|---------|-------------------|---------|
| Predict | `POST /{production\|development\|environments/{env}\|deployment/{dep_id}}/predict` | Sync inference |
| Async predict | `POST .../async_predict` | Async inference |
| Chain call | `POST .../run_remote` | Sync chain call |
| Async chain call | `POST .../async_run_remote` | Async chain call |
| Sync (custom server routes) | `POST /environments/production/sync/{route}` | Call custom server routes |
| Async status | `GET /async_request/{request_id}` | Poll async status |
| Cancel async | `DELETE /async_request/{request_id}` | Cancel queued request |
| Queue status | `GET .../async_queue_status` | Queue depth |
| Wake | `POST .../wake` | Wake scaled-to-zero deployment |

### Management API — `https://api.baseten.co/v1`

Auth: `Authorization: Bearer $BASETEN_API_KEY`

| Concern | Method & Endpoint | Purpose |
|---------|-------------------|---------|
| List models | `GET /v1/models` | List deployed models |
| List deployments | `GET /v1/models/{model_id}/deployments` | List a model's deployments |
| Create API key | `POST /v1/api_keys` | Create workspace API key |
| Promote to production | `POST /v1/models/{model_id}/deployments/{dep_id}/promote` | Promote deployment |
| Promote to environment | `POST /v1/models/{model_id}/environments/{env}/promote` | Promote to custom env |
| Deactivate | `POST /v1/models/{model_id}/deployments/{dep_id}/deactivate` | Stop compute spend |
| Activate | `POST /v1/models/{model_id}/deployments/{dep_id}/activate` | Reactivate |
| Delete deployment | `DELETE /v1/models/{model_id}/deployments/{dep_id}` | Permanently delete |
| Delete model | `DELETE /v1/models/{model_id}` | Permanently delete model |
| Update autoscaling | `PATCH /v1/models/{model_id}/deployments/{dep_id}/autoscaling_settings` | Update scaling in place |
| Enable rolling deploys | `PATCH /v1/models/{model_id}/environments/production` | Configure promotion settings |
| Pause/resume/cancel rollout | `POST /v1/models/{model_id}/environments/production/{pause\|resume\|cancel\|force_cancel\|force_roll_forward}_promotion` | Control rolling deployment |

### Frontier Gateway API — `https://api.baseten.co/v1/gateway`

Auth: `Authorization: Api-Key $BASETEN_API_KEY` (onboarded workspaces only)

| Concern | Method & Endpoint | Purpose |
|---------|-------------------|---------|
| Create endpoint | `POST /v1/gateway/endpoints` | Map slug to target |
| List endpoints | `GET /v1/gateway/endpoints` | Cursor-paginated list |
| Get endpoint | `GET /v1/gateway/endpoints/{endpoint_id}` | Retrieve by ID |
| Update endpoint | `PATCH /v1/gateway/endpoints/{endpoint_id}` | Re-point target |
| Delete endpoint | `DELETE /v1/gateway/endpoints/{endpoint_id}` | Retire slug |
| Create group | `POST /v1/gateway/groups` | Create billable entity |
| List groups | `GET /v1/gateway/groups` | Cursor-paginated |
| Get group | `GET /v1/gateway/groups/{group_id}` | Retrieve by ID |
| Get group usage | `GET /v1/gateway/groups/{group_id}/usage` | Inspect usage |
| Update group | `PATCH /v1/gateway/groups/{group_id}` | Change name/models |
| Delete group | `DELETE /v1/gateway/groups/{group_id}` | Remove + revoke keys + descendants |
| Mint API key | `POST /v1/gateway/groups/{group_id}/api_keys` | Create federated key |
| Register API key | `POST /v1/gateway/groups/{group_id}/api_keys/register` | Register existing key |
| List keys | `GET /v1/gateway/groups/{group_id}/api_keys` | List group keys |
| Get key | `GET /v1/gateway/groups/{group_id}/api_keys/{prefix}` | Fetch by prefix |
| Revoke key | `DELETE /v1/gateway/groups/{group_id}/api_keys/{prefix}` | Revoke credential |