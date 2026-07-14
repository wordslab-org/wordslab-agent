# RunPod API — On-Demand LLM Inference Analysis

> Source: documentation reachable from https://docs.runpod.io/overview
> Scope: on-demand LLM inference services only — both serverless APIs and rented GPU machines. Other RunPod capabilities (fine-tuning, image/video/audio generation endpoints unrelated to LLMs, Instant Clusters for distributed training, Hub publishing, cost-centers/team admin) are intentionally excluded, except where they directly support LLM inference.
> Generated: 2026-07-14

This document analyzes the on-demand LLM inference capabilities exposed by the RunPod platform. For each capability it lists the **main concepts**, the **API functions/endpoints**, and the **parameters** that govern behavior.

RunPod exposes four distinct API surfaces relevant to on-demand LLM inference, plus two supporting management APIs:

- **Public Endpoints** (`https://api.runpod.ai/v2/{model-id}/...`) — pre-deployed, RunPod-hosted LLM models (e.g. Qwen3 32B AWQ, GPT-OSS 120B). No deployment required; pay per token. OpenAI-compatible.
- **Serverless** (`https://api.runpod.ai/v2/{endpoint-id}/...` for inference; `https://rest.runpod.io/v1/...` for management) — self-deployed containerized workers that autoscale per request. Two endpoint sub-types: **queue-based** (built-in job queue + handler) and **load balancing** (direct HTTP routing). vLLM is the primary LLM worker.
- **Flash** (local Python SDK `runpod_flash`, deploys to Serverless) — define `@Endpoint`-decorated functions locally; Flash provisions GPU workers and runs them remotely. An abstraction over Serverless.
- **Pods** (`https://rest.runpod.io/v1/pods/...` REST; GraphQL API at `api.runpod.io/graphql`) — dedicated, persistent GPU/CPU instances billed by the second/minute. Run any container (vLLM, Ollama, TGI, custom). No queue, no autoscaling — you manage the server inside the Pod.
- **Management REST API** (`https://rest.runpod.io/v1/...`) — OpenAPI 3.0 spec for managing Pods, Serverless endpoints, network volumes, templates, container registry auths, and billing.
- **GraphQL API** (`https://api.runpod.io/graphql`) — legacy programmatic management of Pods, templates, and Serverless endpoints.

---

## Table of Contents

1. [Public Endpoints — Pre-Deployed LLM Models](#1-public-endpoints--pre-deployed-llm-models)
2. [Serverless — Self-Deployed LLM Inference (vLLM)](#2-serverless--self-deployed-llm-inference-vllm)
3. [Serverless — Queue-Based Endpoint Operations](#3-serverless--queue-based-endpoint-operations)
4. [Serverless — Handler Functions & Workers](#4-serverless--handler-functions--workers)
5. [Serverless — Load Balancing Endpoints](#5-serverless--load-balancing-endpoints)
6. [Serverless — Endpoint Configuration & Scaling](#6-serverless--endpoint-configuration--scaling)
7. [Flash — Local-Code Remote-GPU SDK](#7-flash--local-code-remote-gpu-sdk)
8. [Pods — Dedicated GPU Instance Rental](#8-pods--dedicated-gpu-instance-rental)
9. [Storage — Network Volumes & S3 API](#9-storage--network-volumes--s3-api)
10. [Templates & Container Registry Auth](#10-templates--container-registry-auth)
11. [Management REST API (OpenAPI)](#11-management-rest-api-openapi)
12. [GraphQL Management API](#12-graphql-management-api)
13. [Authentication & API Keys](#13-authentication--api-keys)
14. [Billing, Pricing & Limits](#14-billing-pricing--limits)
15. [GPU Types & Selection](#15-gpu-types--selection)
16. [Known Limitations](#16-known-limitations)

---

## 1. Public Endpoints — Pre-Deployed LLM Models

**Source pages:** `public-endpoints/overview`, `public-endpoints/quickstart`, `public-endpoints/reference` (models list), `public-endpoints/models/qwen3-32b`, `public-endpoints/requests`, `public-endpoints/ai-coding-tools`, `public-endpoints/ai-sdk`

### Main concepts

- **Public Endpoint**: an AI model API hosted by RunPod, pre-deployed on optimized GPU infrastructure. No deployment or infrastructure required — create an API key and make a request. Models span image, video, audio, and **text generation** categories.
- **Model ID / endpoint slug**: each model has a fixed slug used in the URL path, e.g. `qwen3-32b-awq`, `gpt-oss-120b`. The endpoint base is `https://api.runpod.ai/v2/{model-slug}/...`.
- **Two request modes**:
  - **Synchronous** (`/runsync`): wait for the result in the response. Best for quick generations. Max payload 20 MB; results retained 1 minute (5 min max with `?wait=`).
  - **Asynchronous** (`/run`): receive a job ID immediately, poll `/status/{job_id}` for results. Best for longer generations/batch. Max payload 10 MB; results retained 30 minutes.
- **OpenAI API compatibility**: text-generation Public Endpoints (Qwen3 32B AWQ, GPT-OSS 120B) expose an OpenAI-compatible base URL `https://api.runpod.ai/v2/{model-slug}/openai/v1`. Usable with the OpenAI Python/JS client and any OpenAI-compatible tool (Cursor, Cline, OpenCode).
- **Vercel AI SDK**: the `@runpod/ai-sdk-provider` npm package integrates Public Endpoints with the Vercel AI SDK for TypeScript streaming/image generation.
- **Playground**: browser-based UI at `console.runpod.io/hub/playground/...` for testing models and generating code snippets.
- **Pricing**: per-token for text models (e.g. Qwen3 32B AWQ = $10.00 / 1M tokens); per-request or per-second for media models.

### Available LLM (text-generation) Public Endpoints

| Model | Base URL | Model ID (OpenAI) | Context window | Pricing |
| --- | --- | --- | --- | --- |
| Qwen3 32B AWQ | `https://api.runpod.ai/v2/qwen3-32b-awq` | `Qwen/Qwen3-32B-AWQ` | 32,768 tokens | $10.00 / 1M tokens |
| GPT OSS 120B | `https://api.runpod.ai/v2/gpt-oss-120b` | `openai/gpt-oss-120b` | 131,072 tokens | (see console) |
| IBM Granite 4.0 | `https://api.runpod.ai/v2/granite-4` | — | long-context | (see console) |

### API functions (Public Endpoints)

| Function | Method & Endpoint | Purpose |
| --- | --- | --- |
| Synchronous run | `POST /v2/{model}/runsync` | Run model, wait for result |
| Asynchronous run | `POST /v2/{model}/run` | Submit job, get job ID |
| Job status | `GET /v2/{model}/status/{job_id}` | Poll async job result |
| Cancel job | `POST /v2/{model}/cancel/{job_id}` | Cancel in-progress/queued job |
| Stream results | `GET /v2/{model}/stream/{job_id}` | Incremental output (handler must support streaming) |
| OpenAI chat completions | `POST /v2/{model}/openai/v1/chat/completions` | OpenAI-compatible chat |
| OpenAI completions | `POST /v2/{model}/openai/v1/completions` | OpenAI-compatible text completion |
| OpenAI models list | `GET /v2/{model}/openai/v1/models` | List served models |

### Key parameters (request `input` object — text generation)

| Parameter | Type | Default | Description |
| --- | --- | --- | --- |
| `prompt` | string | — | Prompt for text generation (completion-style) |
| `messages` | array[{role, content}] | — | Chat-style messages (instruction-tuned models; worker applies chat template) |
| `max_tokens` | integer | 512 | Maximum tokens to output |
| `temperature` | float | 0.7 | Sampling randomness, 0.0–1.0 |
| `top_p` | float | — | Nucleus sampling threshold |
| `top_k` | integer | — | Top-K sampling |
| `stop` | string | — | Stop sequence |
| `sampling_params` | object | — | vLLM-style nested sampling params (`temperature`, `max_tokens`, etc.) |
| `stream` | bool | false | Enable token streaming (native API) |
| `guidance` | float | — | Guidance scale (image models) |
| `num_inference_steps` | int | — | Diffusion steps (image models) |
| `width`, `height` | int | — | Output dimensions (image models) |

### Response fields (text generation)

`id`, `status` (`COMPLETED`/`FAILED`), `delayTime` (ms queued), `executionTime` (ms), `workerId`, `output` (object/array). For Qwen3: `output[].choices[].tokens` (array of token strings), `output[].cost` (USD), `output[].usage.{input,output}` (token counts).

### OpenAI-compatibility differences from OpenAI

- **Token counting** may differ (different tokenizers).
- **Rate limits** follow RunPod policies, not OpenAI's.
- **Function/tool calling** depends on model + vLLM support (set `ENABLE_AUTO_TOOL_CHOICE`, `TOOL_CALL_PARSER` env vars on self-deployed vLLM).
- **Vision/multimodal** depends on underlying model support.
- Set `RAW_OPENAI_OUTPUT=1` for raw OpenAI SSE format if response format mismatches.

---

## 2. Serverless — Self-Deployed LLM Inference (vLLM)

**Source pages:** `serverless/vllm/overview`, `serverless/vllm/get-started`, `serverless/vllm/configuration`, `serverless/vllm/environment-variables`, `serverless/vllm/vllm-requests`, `serverless/vllm/openai-compatibility`, `serverless/overview`, `serverless/endpoints/model-caching`

### Main concepts

- **Serverless endpoint**: a deployable compute unit with a URL (`https://api.runpod.ai/v2/{endpoint_id}/...`) that autoscales workers based on demand. Pay-per-second from worker start to stop.
- **vLLM**: open-source LLM inference engine using PagedAttention + continuous batching for high throughput. The primary worker for LLM serving on RunPod Serverless. Deploy from the Runpod Hub `runpod-workers/worker-vllm` repo or the Hub template.
- **Worker**: a container instance running the handler (vLLM server). Scales 0→`workersMax`. Each worker loads the model into GPU VRAM.
- **Model caching (FlashBoot)**: RunPod caches Hugging Face models on hosts so workers start faster and avoid re-downloading. Specify model in endpoint config (e.g. `Qwen/qwen3-32b-awq`); provide HF token for gated models. Shared across workers on the same host.
- **OpenAI compatibility**: vLLM workers expose `https://api.runpod.ai/v2/{endpoint_id}/openai/v1/...` — usable with the OpenAI client by setting `base_url` and `api_key` (RunPod API key).
- **VRAM estimation**: FP16/BF16 = 2 bytes/param; INT8 = 1 byte/param; INT4 (AWQ/GPTQ) = 0.5 bytes/param; vLLM reserves 10–30% remaining VRAM for KV cache.

### vLLM environment variables (LLM settings)

| Variable | Default | Description |
| --- | --- | --- |
| `MODEL_NAME` | — | Hugging Face model ID to serve (e.g. `meta-llama/Llama-3.2-3B-Instruct`) |
| `MAX_MODEL_LEN` | model default | Maximum context length (e.g. `16384`) |
| `GPU_MEMORY_UTILIZATION` | `0.95` | Fraction of VRAM vLLM may use for KV cache/runtime. Lower for OOM. |
| `MAX_PARALLEL_LOADING_WORKERS` | None | Load model in batches to avoid RAM OOM with tensor parallelism |
| `BLOCK_SIZE` | `16` | Token block size (`8`/`16`/`32`) |
| `SWAP_SPACE` | `4` | CPU swap space (GiB) per GPU |
| `TENSOR_PARALLEL_SIZE` | None | Number of GPUs for tensor parallelism (multi-GPU) |
| `MAX_CONCURRENCY` | — | Max concurrent requests per worker |
| `CONFIG_FORMAT` | — | Model config format (e.g. `mistral`) |
| `LOAD_FORMAT` | — | Load format override |

### vLLM environment variables (OpenAI compatibility)

| Variable | Default | Description |
| --- | --- | --- |
| `RAW_OPENAI_OUTPUT` | `1` | Enable raw OpenAI SSE format for streaming |
| `OPENAI_SERVED_MODEL_NAME_OVERRIDE` | None | Custom model ID exposed via `/v1/models` and accepted as `model` field |
| `OPENAI_RESPONSE_ROLE` | `assistant` | Role in chat completion responses |
| `ENABLE_AUTO_TOOL_CHOICE` | `false` | Enable vLLM auto tool selection (only for tool-capable models) |
| `TOOL_CALL_PARSER` | None | Tool-call parser: `mistral`, `hermes`, `llama3_json`, `llama4_json`, `llama4_pythonic`, `granite`, `granite-20b-fc`, ... |

### vLLM sampling parameters (in request `sampling_params`)

| Parameter | Type | Default | Description |
| --- | --- | --- | --- |
| `temperature` | float | 1.0 | Sampling temperature |
| `top_p` | float | 1.0 | Nucleus sampling |
| `top_k` | int | -1 | Top-K sampling |
| `max_tokens` | int | — | Max output tokens |
| `n` | int | 1 | Number of sequences |
| `presence_penalty` | float | 0.0 | Presence penalty |
| `frequency_penalty` | float | 0.0 | Frequency penalty |
| `stop` | array[str] | — | Stop strings |
| `use_beam_search` | bool | false | Beam search instead of sampling |
| `length_penalty` | float | 1.0 | Length penalty for beam search |
| `ignore_eos` | bool | false | Continue after EOS |
| `skip_special_tokens` | bool | true | Omit special tokens |
| `echo` | bool | false | Include prompt in output |

### Input formats (vLLM native API)

- **Prompt (completion models)**: `{"input": {"prompt": "...", "sampling_params": {...}}}`
- **Messages (chat models)**: `{"input": {"messages": [{"role": "system", "content": "..."}, {"role": "user", "content": "..."}], "sampling_params": {...}}}` — worker auto-applies chat template.
- **Streaming**: `{"input": {"prompt": "...", "sampling_params": {...}, "stream": true}}` — then poll `/stream/{job_id}`.

### Supported model families (examples)

Llama 3, Mistral/Ministral, Qwen3, OpenChat, Gemma, DeepSeek-R1, Phi-3, IBM Granite. Gated models require a Hugging Face access token.

---

## 3. Serverless — Queue-Based Endpoint Operations

**Source pages:** `serverless/endpoints/operation-reference`, `serverless/endpoints/send-requests`, `serverless/endpoints/job-states`

### Main concepts

- **Job**: a unit of work containing input data, packaged for processing by a worker's handler. If no worker is available, the job is queued.
- **Sync vs async**: `/runsync` blocks and returns the result; `/run` returns a job ID immediately for polling.
- **Job states**: `IN_QUEUE` → `IN_PROGRESS` → `RUNNING` → `COMPLETED` | `FAILED` | `CANCELLED` | `TIMED_OUT`.
- **Retention**: sync results 1 min (5 min max via `?wait=`); async results 30 min. Job TTL default 24h (configurable 10s–7 days).
- **Endpoint metrics** (Metrics tab): requests (total/completed/failed/retried), execution time P70/P90/P98, delay time P70/P90/P98, cold start time.

### API functions (queue-based endpoint operations)

Base URL: `https://api.runpod.ai/v2/{endpoint_id}`. Auth: `Authorization: Bearer {RUNPOD_API_KEY}` (or bare key per older docs — prefer Bearer).

| Function | Method & Endpoint | Purpose | Limits |
| --- | --- | --- | --- |
| Run sync | `POST /runsync?wait={ms}` | Synchronous job, blocks for result | payload ≤ 20 MB; result 1 min; wait 1000–300000 ms (default 90s) |
| Run async | `POST /run` | Async job, returns job ID | payload ≤ 10 MB; result 30 min |
| Status | `GET /status/{job_id}?ttl={ms}` | Poll job state + output | — |
| Stream | `GET /stream/{job_id}` | Incremental results (handler must support) | chunk ≤ 1 MB |
| Cancel | `POST /cancel/{job_id}` | Cancel queued/in-progress job | — |
| Retry | `POST /retry/{job_id}` | Requeue FAILED/TIMED_OUT job (same ID) | only before result expiry |
| Purge queue | `POST /purge-queue` | Remove all pending jobs | in-progress jobs continue |
| Health | `GET /health` | Worker availability + queue status | — |

### Request structure

```json
{
  "input": { "prompt": "...", "max_tokens": 200 },
  "webhook": "https://...",          // optional callback URL
  "webhook_events": ["IN_PROGRESS", "COMPLETED"],  // optional
  "policy": {                        // optional per-request overrides
    "executionTimeout": 600000,      // ms
    "ttl": 86400000,                  // ms, job lifespan
    "queueTime": 60000                // ms, max time in queue
  }
}
```

### Response (sync)

```json
{
  "delayTime": 824,
  "executionTime": 3391,
  "id": "sync-...",
  "output": { "...": "..." },
  "status": "COMPLETED"
}
```

### Response (async `/run`)

```json
{ "id": "...", "status": "IN_QUEUE" }
```

### Response (`/health`)

```json
{
  "jobs": { "completed": 1, "failed": 5, "inProgress": 0, "inQueue": 2, "retried": 0 },
  "workers": { "idle": 0, "running": 0 }
}
```

### SDKs

| Language | Package | Install |
| --- | --- | --- |
| Python | `runpod` | `pip install runpod` (Python ≥ 3.10) |
| JavaScript | `runpod-sdk` | `npm install runpod-sdk` |
| Go | `github.com/runpod/go-sdk` | `go get github.com/runpod/go-sdk` |

Python SDK usage:
```python
import runpod, os
runpod.api_key = os.getenv("RUNPOD_API_KEY")
endpoint = runpod.Endpoint(os.getenv("ENDPOINT_ID"))
run_request = endpoint.run({"prompt": "Hello, World!"})
status = run_request.status()
output = run_request.output(timeout=60)
run_request.cancel()
```

---

## 4. Serverless — Handler Functions & Workers

**Source pages:** `serverless/workers/handler-functions`, `serverless/workers/concurrent-handler`, `serverless/workers/create-dockerfile`, `serverless/workers/deploy`, `serverless/workers/github-integration`, `serverless/development/*`

### Main concepts

- **Handler function**: a Python function `def handler(job)` that receives a job dict (`job["input"]`) and returns a result. Launched with `runpod.serverless.start({"handler": handler})`. The core abstraction for queue-based workers.
- **Streaming handler**: a generator function that `yield`s partial outputs; consumed via `/stream/{job_id}`.
- **Concurrent handler**: an `async def handler(job)` using `asyncio` for non-blocking I/O; multiple requests per worker via `concurrency_modifier`. Enables higher throughput per worker.
- **Progress updates**: `runpod.serverless.progress_update(job, "message")` — visible when polling `/status`.
- **Worker refresh**: optionally clear logs/wipe worker state after a job for clean state.
- **Local testing**: place `test_input.json` next to handler; run `python handler.py` — SDK runs the handler locally with that input.
- **Dockerfile**: package handler + dependencies into a Docker image; deploy from Docker Hub or GitHub.

### Handler implementation

```python
import runpod

def handler(job):
    job_input = job["input"]
    # process input
    return {"result": "..."}

runpod.serverless.start({"handler": handler})
```

### Concurrent handler

```python
import runpod, asyncio

async def process_request(job):
    job_input = job["input"]
    await asyncio.sleep(job_input.get("delay", 0.1))
    return f"Processed: {job_input}"

def adjust_concurrency(current_concurrency):
    return 50  # max concurrent requests per worker

runpod.serverless.start({
    "handler": process_request,
    "concurrency_modifier": adjust_concurrency
})
```

### Handler parameters (config dict for `runpod.serverless.start`)

| Key | Type | Description |
| --- | --- | --- |
| `handler` | callable (required) | The handler function (sync or async) |
| `concurrency_modifier` | callable(int)->int | Dynamic max concurrency per worker |

---

## 5. Serverless — Load Balancing Endpoints

**Source pages:** `serverless/load-balancing/overview`, `serverless/load-balancing/build-a-worker`, `serverless/load-balancing/vllm-worker`

### Main concepts

- **Load balancing endpoint**: routes traffic directly to available workers **without a queue**. Lower latency, no guaranteed execution, no built-in retries. Ideal for real-time/streaming.
- **Custom HTTP server**: you define your own API with any framework (FastAPI, Flask). No handler function required. Must listen on `0.0.0.0` port 8000, implement `/ping` health check.
- **URL**: `https://{endpoint_id}-proxy.runpod.net/...` — your custom paths.
- **Comparison**: queue-based = guaranteed processing like TCP (retries, queue); load balancing = direct routing like UDP (low latency, no backlog).

### Load balancing worker (FastAPI example)

```python
from fastapi import FastAPI
import uvicorn

app = FastAPI()

@app.get("/ping")
async def health_check():
    return {"status": "healthy"}

@app.post("/generate")
async def generate(request: dict):
    return {"generated_text": f"Generated text for: {request['prompt']}"}

if __name__ == "__main__":
    uvicorn.run(app, host="0.0.0.0", port=8000)
```

### vLLM with load balancing

Deploy a custom vLLM server (not the standard handler-based worker) to a load balancing endpoint for direct OpenAI-compatible API access with lower latency. The worker runs `vllm serve` directly.

---

## 6. Serverless — Endpoint Configuration & Scaling

**Source pages:** `serverless/endpoints/endpoint-configurations`, `serverless/pricing`, `serverless/endpoints/model-caching`

### Main concepts

- **Endpoint types**: **Queue-based** (`QB`, default) — built-in queue, handler pattern, `/run`+`/runsync`. **Load balancing** (`LB`) — direct routing, custom HTTP server.
- **Active workers (`workersMin`)**: always-on workers (eliminate cold starts); charged even when idle, at a lower rate than autoscaling workers.
- **Max workers (`workersMax`)**: maximum concurrent workers (default 3).
- **GPUs per worker (`gpuCount`)**: GPU count per worker instance (default 1; use for multi-GPU / tensor parallelism).
- **Idle timeout**: seconds a worker stays active after completing a request before scaling down (default 5s; range 1–3600).
- **Execution timeout**: max duration for a single job (default 600s/10 min; range 5s–7 days). Overridable per-request via `policy.executionTimeout`.
- **Job TTL**: total lifespan of a job in the system (default 24h; range 10s–7 days). When TTL expires, job data is deleted regardless of state.
- **FlashBoot**: faster cold starts via state retention (enabled by default).
- **Scaler type**: `QUEUE_DELAY` (scale based on how long requests wait in queue) or `REQUEST_COUNT` (scale based on number of queued requests). Default `QUEUE_DELAY`.
- **Scaler value**: for `QUEUE_DELAY`, seconds a request can wait before scaling up (default 4); for `REQUEST_COUNT`, workers = queue depth / scalerValue.
- **Auto-scale-down**: if an endpoint receives no requests for a while, RunPod may reduce `workersMax` to save cost. To re-enable, raise `workersMax` in the console.

### Endpoint settings (quick reference)

| Setting | Default | Description |
| --- | --- | --- |
| Active workers (`workersMin`) | 0 | Always-on workers (no cold starts) |
| Max workers (`workersMax`) | 3 | Maximum concurrent workers |
| GPUs per worker (`gpuCount`) | 1 | GPU count per worker |
| Idle timeout (`idleTimeout`) | 5s | Time before idle worker shuts down (1–3600) |
| Execution timeout (`executionTimeoutMs`) | 600000 ms | Max job duration (5s–7 days) |
| Job TTL | 24h | Total job lifespan (10s–7 days) |
| FlashBoot (`flashboot`) | enabled | Faster cold starts via state retention |
| Scaler type (`scalerType`) | `QUEUE_DELAY` | Scaling algorithm |
| Scaler value (`scalerValue`) | 4 | Scaling threshold |

### Billing phases (worker lifecycle)

1. **Start time**: initializing container + loading model into VRAM (minimize with FlashBoot / model caching).
2. **Execution time**: processing requests.
3. **Idle timeout duration**: worker stays active waiting for more requests before scaling down.

---

## 7. Flash — Local-Code Remote-GPU SDK

**Source pages:** `flash/overview`, `flash/quickstart`, `flash/create-endpoints`, `flash/execution-model`, `flash/pricing`, `flash/configuration/parameters`, `flash/configuration/gpu-types`, `flash/configuration/cpu-types`, `flash/custom-docker-images`, `flash/apps/requests`

### Main concepts

- **Flash**: a Python SDK (`runpod_flash`) for building distributed GPU apps using local Python scripts. Write `@Endpoint`-decorated functions locally; Flash automatically provisions GPU/CPU workers on RunPod Serverless and runs them remotely.
- **`@Endpoint` decorator**: marks a function for remote execution. Configures hardware, scaling, dependencies, storage in one place. Each unique `name` creates one Serverless endpoint.
- **What runs where**: only the decorated function runs on the remote worker; everything else (imports, `print`, control flow) runs locally. Packages must be imported **inside** the function body.
- **Execution flow**: Flash checks if endpoint exists (by `name`) → updates config if changed, else creates endpoint + initializes worker + installs dependencies → sends code to GPU worker → worker executes → result returned to local machine as a Python dict.
- **Endpoint naming**: same name + same config = reuse; same name + different config = update automatically; new name = create new endpoint.
- **Endpoint types** (Flash supports four patterns):
  1. **Queue-based** (`@Endpoint(...)`) — batch/async, `/run` + `/runsync`.
  2. **Load-balanced** (`@Endpoint(...).get/post/...`) — custom HTTP routes.
  3. **CPU** (`cpu=...`) — CPU-only workers.
  4. **Custom Docker image** (`image="..."`) — deploy pre-built images (e.g. vLLM).
- **Flash apps**: `flash dev` (local FastAPI + remote endpoints) → `flash deploy` (all endpoints on Runpod). `flash init` scaffolds a project; `flash build` creates a tarball.
- **`EndpointJob` object**: returned by `.run()` for `id=`/`image=` endpoints — async status/wait/cancel.
- **Configuration change behavior**: changing GPU/CPU/image/storage/datacenter/CUDA/Python **recreates workers** (30–90s downtime); changing workers/timeouts/scaler/env/name **updates settings only** (no downtime).

### `@Endpoint` parameters (complete reference)

| Parameter | Type | Default | Description |
| --- | --- | --- | --- |
| `name` | str | required (unless `id=`) | Endpoint name (identifies endpoint) |
| `id` | str | `None` | Connect to existing endpoint by ID |
| `gpu` | GpuGroup / GpuType / list | `GpuGroup.ANY` | GPU type(s); list = fallback order |
| `cpu` | str / CpuInstanceType | `None` | CPU instance (mutually exclusive with `gpu`) |
| `workers` | int or (min, max) | `(0, 1)` | Worker scaling; `N` = (0,N); tuple = (min,max) |
| `idle_timeout` | int | 60 | Seconds before scaling down idle workers (1–3600) |
| `dependencies` | list[str] | `None` | Python pip packages to install on worker |
| `system_dependencies` | list[str] | `None` | apt packages to install |
| `accelerate_downloads` | bool | True | Faster downloads for deps/models/files |
| `volume` | NetworkVolume or list | `None` | Network volume(s); mounted at `/runpod-volume/` |
| `datacenter` | DataCenter / list / str / None | `None` (all DCs) | Datacenter(s) for deployment |
| `env` | dict[str, str] | `None` | Environment variables for workers |
| `gpu_count` | int | 1 | GPUs per worker (multi-GPU) |
| `execution_timeout_ms` | int | 0 (no limit) | Max execution time per job (ms) |
| `flashboot` | bool | True | Fast cold starts via image pre-loading |
| `image` | str | `None` | Custom Docker image (bypasses managed workers) |
| `scaler_type` | ServerlessScalerType | auto | `QUEUE_DELAY` (queue-based) / `REQUEST_COUNT` (LB) |
| `scaler_value` | int | 4 | Scaling threshold |
| `template` | PodTemplate | `None` | Advanced pod config overrides |
| `min_cuda_version` | str / CudaVersion | `"12.8"` (GPU) | Min CUDA driver on host |
| `python_version` | str | local Python | Worker Python version (`3.10`–`3.13`) |

### `PodTemplate` sub-parameters

| Parameter | Type | Default | Description |
| --- | --- | --- | --- |
| `containerDiskInGb` | int | 64 | Container disk size in GB |
| `env` | list[{key, value}] | `None` | Env vars as list of dicts |

### GPU selection (Flash)

- **GPU pools** (`GpuGroup`): `AMPERE_16`, `AMPERE_24`, `ADA_24`, `AMPERE_48`, `ADA_48_PRO`, `AMPERE_80`, `ADA_80_PRO`, `HOPPER_141`, `ANY` (fastest provisioning).
- **Specific GPU types** (`GpuType`): e.g. `NVIDIA_A100_80GB_PCIe`, `NVIDIA_GEFORCE_RTX_4090`, `NVIDIA_H200`, `NVIDIA_B200`, etc.
- **Fallback**: pass a list `[GpuType.A, GpuType.B]` to try A first, fall back to B.

### CPU instance types (Flash)

| Shorthand | vCPU | RAM | Notes |
| --- | --- | --- | --- |
| `cpu3c-2-8` | 2 | 8 GB | Light |
| `cpu3g-2-8` | 2 | 8 GB | More RAM/vCPU |
| `cpu5c-4-8` | 4 | 8 GB | API/webhook |
| `cpu5c-8-16` | 8 | 16 GB | Heavy processing |
| `cpu3g-8-32` | 8 | 32 GB | Memory-intensive |

### Flash request methods

| Method | Use | Format |
| --- | --- | --- |
| `await endpoint.run({"input": {...}})` | Queue-based async | returns `EndpointJob` |
| `await endpoint.runsync(...)` | Queue-based sync | blocks for result |
| `await endpoint.post("/path", {...})` | Load-balanced HTTP | custom routes |
| `await endpoint.get("/path")` | Load-balanced HTTP | custom routes |

### `EndpointJob` methods

```python
job = await ep.run({"prompt": "hello"})
status = await job.status()   # "IN_PROGRESS", "COMPLETED", ...
await job.wait(timeout=60)    # wait for completion
print(job.id, job.output, job.error, job.done)
await job.cancel()
```

---

## 8. Pods — Dedicated GPU Instance Rental

**Source pages:** `pods/overview`, `pods/choose-a-pod`, `pods/connect-to-a-pod`, `pods/manage-pods`, `pods/pricing`, `pods/networking`, `pods/templates/*`, `api-reference/pods/*`

### Main concepts

- **Pod**: a dedicated GPU or CPU instance for containerized AI/ML workloads — training, inference, rendering. Full control over the environment. Billed by the second/minute. No queue, no autoscaling; you run the server (vLLM, Ollama, TGI, custom) inside the Pod.
- **Pod types (cloud options)**:
  - **Secure Cloud**: T3/T4 data centers, high reliability/security for enterprise/production.
  - **Community Cloud**: vetted peer-to-peer compute providers, competitive pricing.
- **On-demand vs savings plan**: on-demand = pay-as-you-go, dedicated resources, need ≥1h credits to deploy. Savings plan = 3 or 6 month upfront commitment for discounts (covers GPU compute only, not storage).
- **Storage tiers**: container disk (temporary, erased on stop), volume disk (persistent across restarts), network volume (permanent, portable between Pods).
- **Connection options**: web terminal, SSH, JupyterLab (port 8888), VS Code/Cursor (remote SSH), HTTP/TCP port forwarding.
- **Global networking**: secure private network connecting all Pods in an account; Pod-to-Pod via `POD_ID.runpod.internal`. NVIDIA GPU Pods only, select Secure Cloud data centers.
- **Templates**: bundle a container image + hardware specs + network settings + env vars for reuse.
- **Limitations**: no Docker Compose (RunPod runs Docker for you), no UDP (TCP/HTTP only), no Windows.

### Workload → GPU guidance

| Workload | Recommended GPU | Min VRAM |
| --- | --- | --- |
| LLM inference 7B–13B | RTX 4090, L4 | 24 GB |
| LLM inference 30B–70B | A100, H100 | 48–80 GB (may need multi-GPU) |
| LLM training/fine-tuning | A100, H100 | 40–80 GB |

### API functions (REST API — Pods)

Base URL: `https://rest.runpod.io/v1`. Auth: `Authorization: Bearer {RUNPOD_API_KEY}`.

| Function | Method & Endpoint | Purpose |
| --- | --- | --- |
| Create Pod | `POST /pods` | Create and optionally deploy a Pod |
| List Pods | `GET /pods` | Returns list of Pods |
| Get Pod | `GET /pods/{podId}` | Retrieve a single Pod |
| Update Pod | `PATCH /pods/{podId}` | Update (may trigger reset) |
| Start/resume Pod | `POST /pods/{podId}/start` | Start or resume a stopped Pod |
| Stop Pod | `POST /pods/{podId}/stop` | Stop a Pod (volume preserved) |
| Restart Pod | `POST /pods/{podId}/restart` | Restart a Pod |
| Reset Pod | `POST /pods/{podId}/reset` | Reset a Pod |
| Delete Pod | `DELETE /pods/{podId}` | Delete a Pod |

### `PodCreateInput` parameters (REST API)

| Parameter | Type | Default | Description |
| --- | --- | --- | --- |
| `name` | string | `my pod` | User-defined name (need not be unique; max 191) |
| `computeType` | enum | `GPU` | `GPU` or `CPU` |
| `cloudType` | enum | `SECURE` | `SECURE` or `COMMUNITY` |
| `gpuTypeIds` | array[string] | — | GPU types in priority order (full enum of ~50 NVIDIA/AMD IDs) |
| `gpuCount` | int | 1 | Number of GPUs |
| `gpuTypePriority` | enum | `availability` | `availability` (respond to stock) or `custom` (strict order) |
| `imageName` | string | — | Docker image tag |
| `containerDiskInGb` | int | 50 | Container disk GB (wiped on restart) |
| `volumeInGb` | int | 20 | Pod volume GB (persistent across restarts) |
| `volumeMountPath` | string | `/workspace` | Mount path for volume |
| `networkVolumeId` | string | — | Attach a network volume (replaces Pod volume) |
| `env` | object | `{}` | Environment variables |
| `ports` | array[string] | `["8888/http","22/tcp"]` | Exposed ports (`port/proto`) |
| `dockerEntrypoint` | array | `[]` | Override image ENTRYPOINT |
| `dockerStartCmd` | array | `[]` | Override image start CMD |
| `templateId` | string | — | Create from template |
| `containerRegistryAuthId` | string | — | Private registry credentials |
| `interruptible` | bool | false | Spot Pod (lower cost, can be stopped anytime) |
| `locked` | bool | false | Lock Pod (disable stop/reset) |
| `dataCenterIds` | array[string] | (all DCs) | Data center IDs in priority order |
| `dataCenterPriority` | enum | `availability` | `availability` or `custom` |
| `countryCodes` | array[string] | — | Restrict to countries |
| `minRAMPerGPU` | int | 8 | Min RAM (GB) per GPU |
| `minVCPUPerGPU` | int | 2 | Min vCPUs per GPU |
| `minDownloadMbps` | number | — | Min download speed (Mbps) |
| `minUploadMbps` | number | — | Min upload speed (Mbps) |
| `minDiskBandwidthMBps` | number | — | Min disk bandwidth (MBps) |
| `vcpuCount` | int | 2 | vCPUs (CPU Pods) |
| `cpuFlavorIds` | array[enum] | — | CPU flavors: `cpu3c`, `cpu3g`, `cpu3m`, `cpu5c`, `cpu5g`, `cpu5m` |
| `cpuFlavorPriority` | enum | `availability` | `availability` or `custom` |
| `allowedCudaVersions` | array[enum] | — | Acceptable CUDA versions (`11.8`–`13.0`) |
| `supportPublicIp` | bool | true | Public IP (Community Cloud) |
| `globalNetworking` | bool | false | Enable global networking |

### Pod response fields

`id`, `name`, `image`, `gpu.{id,count,displayName,securePrice,communityPrice,...}`, `machine.{...}`, `machineId`, `memoryInGb`, `vcpuCount`, `ports`, `portMappings`, `publicIp`, `desiredStatus` (`RUNNING`/`EXITED`/`TERMINATED`), `lastStartedAt`, `lastStatusChange`, `costPerHr`, `adjustedCostPerHr`, `interruptible`, `locked`, `env`, `containerDiskInGb`, `volumeInGb`, `volumeMountPath`, `networkVolume.{id,name,size,dataCenterId}`, `savingsPlans[]`, `templateId`, `endpointId`, `slsVersion`, `containerRegistryAuthId`.

### Storage pricing (Pods)

| Storage type | Running Pod | Stopped Pod |
| --- | --- | --- |
| Container disk | $0.10/GB/mo | Not charged |
| Volume disk | $0.10/GB/mo | $0.20/GB/mo |
| Network volume | $0.07/GB/mo (<1TB) / $0.05/GB/mo (>1TB) | Same |

---

## 9. Storage — Network Volumes & S3 API

**Source pages:** `storage/network-volumes`, `storage/s3-api`, `storage/high-performance-storage`, `serverless/storage/overview`

### Main concepts

- **Network volume**: persistent, portable storage attachable to multiple Pods/workers. Survives Pod stops. Mounted at `/runpod-volume` (Serverless) or `/workspace` (Pods). Size can be increased but not decreased; >4TB contact support.
- **High-performance storage**: premium tier with up to 3x throughput and 4x IOPS, available in select data centers.
- **S3-compatible API**: interact with network volumes using standard S3 tools (AWS CLI `s3`/`s3api`, Boto3). Path mapping: network volume files are exposed as S3 objects.
- **Serverless use**: attach network volumes to endpoints to share models/data across workers — reduces cold starts (no re-download), lowers cost, centralizes data. Concurrent writes from multiple workers may corrupt data — handle in app logic.
- **Balance safety**: if balance reaches $0, Pods with network volumes are stopped (data preserved); Pods without volumes are terminated (data lost). Storage charges continue on stopped Pods.

### API functions (REST API — Network Volumes)

Base URL: `https://rest.runpod.io/v1`.

| Function | Method & Endpoint | Purpose |
| --- | --- | --- |
| Create volume | `POST /networkvolumes` | Create a network volume |
| List volumes | `GET /networkvolumes` | List network volumes |
| Get volume | `GET /networkvolumes/{networkVolumeId}` | Retrieve a volume |
| Update volume | `PATCH /networkvolumes/{networkVolumeId}` | Update a volume |
| Delete volume | `DELETE /networkvolumes/{networkVolumeId}` | Delete a volume |

### Create volume parameters

| Parameter | Type | Description |
| --- | --- | --- |
| `name` | string | Volume name |
| `size` | int | Size in GB |
| `dataCenterId` | string | Data center ID (e.g. `US-KS-2`) |

### S3-compatible API

- Use AWS `s3`/`s3api` CLI, Boto3, or any S3 client.
- Standard operations (`ls`, `cp`, `mv`, `rm`) work; `sync` may have issues with >10,000 files or complex dirs.
- Network volume path mapping: Serverless `/runpod-volume`, Pods `/workspace`.

---

## 10. Templates & Container Registry Auth

**Source pages:** `api-reference/templates/*`, `api-reference/container-registry-auths/*`, `pods/templates/*`

### Main concepts

- **Template**: saves and reuses Pod/Serverless endpoint configurations — bundles a container image with hardware specs, network settings, env vars, and disk sizes. Can be public or private. Official RunPod templates flagged `isRunpod`.
- **Container registry auth**: credentials for private Docker registries (Docker Hub, private registries), attached to Pods/workers that need private images.

### API functions (REST API — Templates)

| Function | Method & Endpoint | Purpose |
| --- | --- | --- |
| Create template | `POST /templates` | Create a new template |
| List templates | `GET /templates` | List templates |
| Get template | `GET /templates/{templateId}` | Retrieve a template |
| Update template | `PATCH /templates/{templateId}` | Update a template |
| Delete template | `DELETE /templates/{templateId}` | Delete a template |

### Template fields

`id`, `name` (unique), `imageName`, `containerDiskInGb`, `volumeInGb`, `volumeMountPath`, `env`, `ports` (e.g. `["8888/http","22/tcp"]`), `dockerEntrypoint`, `dockerStartCmd`, `containerRegistryAuthId`, `isPublic`, `isRunpod`, `isServerless` (true = Serverless worker, false = Pod), `category` (`NVIDIA`/`AMD`/`CPU`), `readme` (markdown), `earned` (revenue-sharing credits).

### API functions (REST API — Container Registry Auths)

| Function | Method & Endpoint | Purpose |
| --- | --- | --- |
| Create auth | `POST /containerregistryauth` | Add registry credentials |
| List auths | `GET /containerregistryauth` | List registry auths |
| Get auth | `GET /containerregistryauth/{id}` | Retrieve an auth |
| Delete auth | `DELETE /containerregistryauth/{id}` | Delete an auth |

---

## 11. Management REST API (OpenAPI)

**Source pages:** `api-reference/overview`, `api-reference/openapi.json`, all `api-reference/*` sub-pages

### Main concepts

- **REST API**: OpenAPI 3.0.3 spec at `https://rest.runpod.io/v1`. Programmatic management of all compute resources. Standard HTTP methods, JSON responses.
- **OpenAPI schema**: `GET https://rest.runpod.io/v1/openapi.json` (requires auth) — use for client generation, request validation, tooling.
- **Resources managed**: Pods, Serverless endpoints, network volumes, templates, container registry auths, billing.
- **Authentication**: all requests require `Authorization: Bearer {RUNPOD_API_KEY}`.

### API functions (REST API — Serverless Endpoints)

| Function | Method & Endpoint | Purpose |
| --- | --- | --- |
| Create endpoint | `POST /endpoints` | Create a new Serverless endpoint |
| List endpoints | `GET /endpoints?includeTemplate=&includeWorkers=` | List endpoints |
| Get endpoint | `GET /endpoints/{endpointId}` | Retrieve an endpoint |
| Update endpoint | `PATCH /endpoints/{endpointId}` | Update an endpoint |
| Delete endpoint | `DELETE /endpoints/{endpointId}` | Delete an endpoint |

### `EndpointCreateInput` parameters (REST API)

| Parameter | Type | Default | Description |
| --- | --- | --- | --- |
| `templateId` | string | required | Template ID for the endpoint |
| `name` | string | — | Endpoint name (max 191) |
| `computeType` | enum | `GPU` | `GPU` or `CPU` |
| `gpuTypeIds` | array[enum] | — | GPU types in priority order (~50 NVIDIA/AMD IDs) |
| `gpuCount` | int | 1 | GPUs per worker |
| `cpuFlavorIds` | array[enum] | — | CPU flavors (`cpu3c`, `cpu3g`, `cpu5c`, `cpu5g`) |
| `vcpuCount` | int | 2 | vCPUs (CPU endpoints) |
| `workersMax` | int | 3 | Max concurrent workers (≥0) |
| `workersMin` | int | 0 | Min always-on workers (≥0, charged at lower rate) |
| `idleTimeout` | int | 5 | Seconds before idle worker scales down (1–3600) |
| `executionTimeoutMs` | int | 600000 | Max job duration in ms |
| `flashboot` | bool | true | FlashBoot fast startup |
| `scalerType` | enum | `QUEUE_DELAY` | `QUEUE_DELAY` or `REQUEST_COUNT` |
| `scalerValue` | int | 4 | Scaling threshold |
| `networkVolumeId` | string | — | Attach a network volume |
| `networkVolumeIds` | array[string] | — | Multiple volumes (multi-region) |
| `dataCenterIds` | array[enum] | (all DCs) | Data center IDs (26 options) |
| `allowedCudaVersions` | array[enum] | — | Acceptable CUDA versions (`11.8`–`13.0`) |
| `minCudaVersion` | enum | — | Min acceptable CUDA version |
| `env` | object | `{}` | Environment variables |

### Endpoint response fields

`id`, `name`, `computeType` (`GPU`/`CPU`), `gpuTypeIds`, `gpuCount`, `workersMin`, `workersMax`, `idleTimeout`, `executionTimeoutMs`, `flashboot`, `scalerType`, `scalerValue`, `dataCenterIds`, `networkVolumeId`, `networkVolumeIds`, `env`, `templateId`, `template`, `userId`, `createdAt`, `version` (bumped on template/env changes), `workers[]` (array of `Pod` objects), `instanceIds` (CPU).

### Billing endpoints (REST API)

| Function | Method & Endpoint | Purpose |
| --- | --- | --- |
| Serverless billing | `GET /billing/endpoints` | Endpoint billing history |
| Pod billing | `GET /billing/pods` | Pod billing history |
| Network volume billing | `GET /billing/networkvolumes` | Volume billing history |

---

## 12. GraphQL Management API

**Source pages:** `sdks/graphql/configurations`, `sdks/graphql/manage-endpoints`, `sdks/graphql/manage-pods`, `sdks/graphql/manage-pod-templates`, `references/graphql-spec`

### Main concepts

- **GraphQL API**: legacy programmatic management at `https://api.runpod.io/graphql`. Full schema in `references/graphql-spec`. Used for Pods, templates, and Serverless endpoints.
- **Pod mutations**: `podFindAndDeployOnDemand` (create), `podResume` (start), `podStop`, `podTerminate`.
- **Pod queries**: `myself { pods { ... } }` (list), `pod(input: {podId})` (get).
- **GPU queries**: `gpuTypes(input: {id})` returns `id`, `displayName`, `memoryInGb`, `secureCloud`, `communityCloud`, `lowestPrice.uninterruptablePrice`, `stockStatus` (`High`/`Medium`/`Low`/`None`).
- **Endpoint mutations**: `saveEndpoint` (create/update), `deleteEndpoint`.
- **Endpoint fields**: `id`, `name`, `gpuIds` (GPU pool string, e.g. `AMPERE_16`), `workersMin`, `workersMax`, `idleTimeout`, `locations` (region), `networkVolumeId`, `scalerType`, `scalerValue`, `templateId`, `type` (`QB`/`LB`), `pods[]`, `serverlessDiscount`.

### GPU pools (GraphQL / Hub)

| Pool ID | GPUs Included | Memory |
| --- | --- | --- |
| `AMPERE_16` | A4000, A4500, RTX 4000, RTX 2000 | 16 GB |
| `AMPERE_24` | L4, A5000, 3090 | 24 GB |
| `ADA_24` | 4090 | 24 GB |
| `AMPERE_48` | A6000, A40 | 48 GB |
| `ADA_48_PRO` | L40, L40S, 6000 Ada | 48 GB |
| `AMPERE_80` | A100 | 80 GB |
| `ADA_80_PRO` | H100 | 80 GB |
| `HOPPER_141` | H200 | 141 GB |

### Pod runtime query fields

`runtime.uptimeInSeconds`, `runtime.ports[].{ip,isIpPublic,privatePort,publicPort,type}`, `runtime.gpus[].{id,gpuUtilPercent,memoryUtilPercent}`, `container.{cpuPercent,memoryPercent}`.

---

## 13. Authentication & API Keys

**Source pages:** `get-started/api-keys`, `api-reference/overview`, `references/security-and-compliance`

### Main concepts

- **API key**: required for all requests, in `Authorization: Bearer {key}` header (REST) or `Authorization: {key}` (Serverless native — Bearer preferred). Created in console Settings › API Keys. RunPod does not store the full key after creation.
- **Key permissions**: `All` (full access), `Restricted` (customize per Runpod API — None/Restricted/Read-Write/Read-Only per endpoint), `Read Only`.
- **Legacy keys**: pre-Nov 11 2024 keys have Read/Write or Read-Only GraphQL access + full AI API access. Generate new Restricted keys for least privilege.
- **Key management**: create, edit permissions, enable/disable (toggle), delete (revoke) from the console.
- **Multi-tenant isolation**: Pods/workers run in containerized isolation; Secure Cloud uses T3/T4 data centers.
- **Data security/GDPR**: GDPR-compliant regions (EU); consent management, data subject rights (access/rectify/erase/restrict). SCC for lawful data transfer.
- **Host access policies**: documented in security-and-compliance.

### Key scoping (Serverless endpoints)

When creating a Restricted key, per-endpoint access can be set to:
- `None` — no access
- `Restricted` — customize per endpoint
- `Read/Write` — full access to your endpoints
- `Read Only` — read access without write

---

## 14. Billing, Pricing & Limits

**Source pages:** `accounts-billing/billing`, `serverless/pricing`, `pods/pricing`, `flash/pricing`, `accounts-billing/cost-centers`

### Main concepts

- **Credits**: prepaid balance; if balance reaches $0, Pods with volumes are stopped (preserved), Pods without volumes are terminated (data lost). Serverless endpoints stop scaling.
- **Spend limit**: default $80/hour across all resources; contact support to increase.
- **Pod pricing**: on-demand (pay-as-you-go, by the second/minute) or savings plans (3/6 month upfront, GPU compute only, storage at standard rate). Savings plan auto-applies to next deployment of the same GPU type when a Pod stops.
- **Serverless pricing**: pay-per-second from worker start to full stop, rounded up. Billed phases: start time + execution time + idle timeout duration. No idle costs when no requests. No upfront costs.
- **Flash pricing**: same as Serverless (Flash deploys to Serverless). Same billing phases.
- **Public Endpoints pricing**: per-token for text models (e.g. $10/1M tokens), per-request/per-second for media.
- **Cost centers**: track and organize spending by team/project/department.
- **Billing support**: include endpoint ID, request ID, approximate time in support tickets.

### Cost optimization strategies

- Choose the smallest GPU that fits VRAM needs (RTX 4090/L4 for 24 GB, not A100).
- Use quantized models (AWQ/GPTQ) to reduce memory 50–75%.
- Use model caching / FlashBoot to reduce cold-start costs.
- Configure idle timeout (30–60s cost-optimized; 60–120s balanced; 120–300s latency-optimized).
- Use `workersMin=0` to allow scale-to-zero (with cold starts).
- Use `workersMin=1` to avoid cold starts (always-on, charged at lower rate).
- Attach network volumes to share models across workers (no re-download).
- Reduce `MAX_MODEL_LEN` if context window causes OOM.
- Use `QUEUE_DELAY` scaler for max-latency guarantees; `REQUEST_COUNT` for queue-depth scaling.

---

## 15. GPU Types & Selection

**Source pages:** `references/gpu-types`, `references/cpu-types`, `flash/configuration/gpu-types`, `flash/configuration/cpu-types`

### Main concepts

- **GPU types**: full catalog of ~50 NVIDIA and AMD GPUs available on RunPod, identified by exact string IDs (e.g. `NVIDIA A100 80GB PCIe`, `NVIDIA GeForce RTX 4090`, `AMD Instinct MI300X OAM`, `NVIDIA H200`, `NVIDIA B200`).
- **GPU pools**: grouped by architecture + VRAM for flexible provisioning (see §12 GraphQL pools).
- **CPU flavors**: `cpu3c`, `cpu3g`, `cpu3m`, `cpu5c`, `cpu5g`, `cpu5m` — with variant suffixes for vCPU/RAM (e.g. `cpu5c-4-8` = 4 vCPU, 8 GB RAM).
- **CUDA versions**: `11.8`, `12.0`–`12.9`, `13.0`. Filter hosts by `allowedCudaVersions` or `minCudaVersion` for compatibility.
- **Data centers**: 26 regions across US, CA, EU (CZ/FR/NL/RO/SE/IS/NO), OC (AU), AP (JP). Select by latency, compliance, GPU availability.

### Notable GPUs for LLM inference

| GPU | VRAM | Architecture | Use case |
| --- | --- | --- | --- |
| RTX 4090 | 24 GB | Ada | 7B–13B quantized inference |
| L4 | 24 GB | Ampere | Cost-effective inference |
| A6000 / A40 | 48 GB | Ampere | 13B–30B |
| L40S | 48 GB | Ada Pro | 13B–30B optimized |
| A100 (PCIe/SXM) | 80 GB | Ampere | 30B–70B, multi-GPU |
| H100 | 80 GB | Hopper | 30B–70B, fast |
| H200 | 141 GB | Hopper | Large models, long context |
| B200 | 180 GB | Blackwell | Largest models |
| MI300X | 192 GB | AMD CDNA | Largest models (AMD) |
| B300 | 288 GB | Blackwell | Largest models |

---

## 16. Known Limitations

### Pods

- **No Docker Compose**: RunPod runs Docker for you; cannot spin up your own Docker instance.
- **No UDP**: Pods support TCP and HTTP only.
- **No Windows**: Pods do not support Windows.
- **Docker-in-Docker**: not natively supported (use Bazel or alternative workflows).

### Serverless

- **`/serverless/reference` and `/serverless/requests`**: these doc URLs return 404 — content has moved to `serverless/endpoints/operation-reference` and `serverless/endpoints/send-requests`.
- **Load balancing endpoints**: no queuing mechanism — if all workers are busy, requests fail ("No workers available"). No guaranteed execution or retries.
- **Concurrent writes to network volume**: writing to the same volume from multiple workers simultaneously may cause data corruption — handle concurrency in app logic.
- **Auto-scale-down**: inactive endpoints may have `workersMax` reduced automatically; must manually raise it to use the endpoint again.
- **Results retention**: sync results expire after 1 minute (5 min max); async after 30 min. Jobs cannot be retried after expiry.
- **Max payload**: sync 20 MB, async 10 MB, stream chunk 1 MB.

### Flash

- **Python 3.10/3.11/3.13 overhead**: ~7 GB additional cold-start overhead on GPU endpoints (alternative interpreter installed alongside base PyTorch).
- **Python 3.10 EOL**: 2026-10-31 — migrate to 3.11+.
- **`.env` not passed to deployed endpoints**: must declare env vars explicitly via `env` parameter.
- **One network volume per datacenter**: multiple volumes in the same DC cause deployment failure.
- **All resources in a Flash app must use the same Python version**: conflicting versions fail the build.

### Public Endpoints

- **OpenAI differences**: token counting, rate limits, tool calling, and multimodal support differ from OpenAI's API.
- **Model availability**: only models listed in the Public Endpoints catalog are available pre-deployed; for other models, deploy your own vLLM Serverless endpoint.

### General

- **GraphQL API** is legacy — the REST API (`rest.runpod.io/v1`) is the current primary management surface with OpenAPI 3.0 spec.
- **Balance exhaustion**: at $0, Pods without volumes are terminated (data unrecoverable); enable low-balance notifications.
- **Network volume size**: can be increased but not decreased; >4 TB requires support contact.