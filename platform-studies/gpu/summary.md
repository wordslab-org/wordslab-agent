# Unified GPU Inference Platform — Aggregated Capability Specification

> **Purpose:** This document aggregates the capabilities of six popular AI inference platforms — **Baseten**, **Fireworks AI**, **Hugging Face** (Inference Providers + Inference Endpoints), **Nebius Token Factory**, **RunPod**, and **Together AI** — into a single, exhaustive specification. It is organized as a complete processing pipeline that a user follows to run LLM (and other AI model) inference on rented GPU infrastructure. For every step, it lists the concepts, the alternative approaches offered by different systems, and a cross-platform name-mapping table so you can recognize the same idea under different labels.
>
> **Date of analysis:** 2026-07-14
> **Source files:** `baseten-api.md`, `fireworks-api.md`, `huggingface-api.md`, `nebius-api.md`, `runpod-api.md`, `together-api.md`

---

## Table of Contents

1. [Introduction: Core Concepts (Read Me First)](#1-introduction-core-concepts-read-me-first)
2. [The End-to-End Processing Pipeline (Overview)](#2-the-end-to-end-processing-pipeline-overview)
3. [Stage 0 — Account, Authentication & Access Control](#stage-0--account-authentication--access-control)
4. [Stage 1 — Model Discovery & Selection](#stage-1--model-discovery--selection)
5. [Stage 2 — Model Packaging & Artifact Management](#stage-2--model-packaging--artifact-management)
6. [Stage 3 — Compute Provisioning & Deployment Creation](#stage-3--compute-provisioning--deployment-creation)
7. [Stage 4 — Inference Engine Configuration](#stage-4--inference-engine-configuration)
8. [Stage 5 — Deployment Lifecycle & Traffic Management](#stage-5--deployment-lifecycle--traffic-management)
9. [Stage 6 — Request Routing, Gateway & Rate Limiting](#stage-6--request-routing-gateway--rate-limiting)
10. [Stage 7 — Inference Request Execution (Tasks & Modalities)](#stage-7--inference-request-execution-tasks--modalities)
11. [Stage 8 — Output Control & Generation Parameters](#stage-8--output-control--generation-parameters)
12. [Stage 9 — Observability, Metrics & Cost Attribution](#stage-9--observability-metrics--cost-attribution)
13. [Stage 10 — Reliability, Security, Compliance & Sandboxes](#stage-10--reliability-security-compliance--sandboxes)
14. [Cross-Platform Glossary (Alphabetical Name Mapping)](#cross-platform-glossary-alphabetical-name-mapping)
15. [Deployment-Mode Decision Matrix](#deployment-mode-decision-matrix)
16. [Capability Coverage Matrix Per Platform](#capability-coverage-matrix-per-platform)

---

## 1. Introduction: Core Concepts (Read Me First)

If you have never used a GPU inference platform, this section explains the mental model you need before reading the rest of the spec. Every platform in this study exposes the same fundamental building blocks; they just name them differently and expose different subsets.

### 1.1 What "on-demand GPU inference" means

You want to run a model (typically a Large Language Model, but also embedding, reranking, image, video, audio, or classification models) without buying GPUs. A platform rents you either **shared** access to GPUs it already operates, or **dedicated** GPUs reserved only for you, and exposes an HTTP API that accepts your input and returns the model's output. You pay either per token of text processed, or per unit of time that a GPU is allocated to you.

### 1.2 The two fundamental serving models

Every platform offers one or both of these:

| Serving model | Idea | Billing | Cold starts | Control |
|---|---|---|---|---|
| **Serverless / shared / public** | The platform runs a catalog of popular models on a shared fleet. You just call an API. | Per token (or per request / per second of media) | None (platform keeps models warm) | None over hardware/engine |
| **Dedicated / reserved / on-demand deployment** | You pick a model and the platform reserves GPUs only for you. You control hardware, engine, scaling, and can deploy custom/fine-tuned weights. | Per GPU-second or per minute of reserved hardware | Possible when scaling from zero | Full |

A handful of platforms add intermediate tiers:

- **Provisioned throughput** (Together) — you commit to a fixed slice of guaranteed throughput for a stock model, with an SLA. Between serverless and dedicated.
- **Dedicated containers** (Together) — you bring a Docker image; the platform runs it on managed GPU infra with a job queue.
- **GPU clusters** (Together) / **Pods** (RunPod) — raw GPU instances or full Kubernetes/Slurm clusters you fully control, for training or custom stacks.
- **Sandboxes** (Nebius) — secure, Git-like-branchable code-execution environments for AI agents (complementary to inference).

### 1.3 The processing pipeline at a glance

No matter which platform you use, getting inference running follows the same ordered stages. This spec walks through each one:

```
Stage 0  Account & auth          → who are you, what can you do
Stage 1  Model discovery          → which model, which flavor, which provider
Stage 2  Model packaging          → weights, LoRAs, containers, quantization
Stage 3  Compute provisioning     → GPU type, count, region, deployment creation
Stage 4  Engine configuration     → vLLM/TGI/SGLang/TRT-LLM params, spec decoding
Stage 5  Lifecycle & traffic      → promote, roll, scale, wake, health checks
Stage 6  Routing & gateway        → replica selection, rate limits, service tiers
Stage 7  Request execution        → chat, embeddings, rerank, vision, batch, async
Stage 8  Output control           → sampling, structured output, tools, reasoning
Stage 9  Observability & cost     → metrics, logs, billing, attribution
Stage 10 Reliability & security  → retries, webhooks, TLS, compliance, sandboxes
```

### 1.4 Key vocabulary you will meet repeatedly

- **Model** — a specific AI model identified by a string slug. Slugs differ wildly between platforms (see Stage 1).
- **Deployment / Endpoint / Pod** — a running instance of a model on GPUs. The most overloaded word in this space; see the glossary.
- **Replica / Worker** — one process/container holding the model in GPU VRAM and serving requests. Deployments scale by adding/removing replicas.
- **Inference engine** — the software that loads weights and runs generation: vLLM, TGI, SGLang, llama.cpp, TEI, TensorRT-LLM, or a custom container.
- **KV cache** — the cached key/value tensors from attention; reusing them across requests with overlapping prefixes is the main latency/cost win ("prompt caching").
- **Cold start** — the delay when a fresh replica must pull an image, load weights, and compile before it can serve.
- **Autoscaling** — adding/removing replicas based on a metric (concurrency, tokens in flight, GPU utilization, queue depth).
- **Scale-to-zero** — releasing all replicas when idle to stop billing; the next request pays a cold start.

---

## 2. The End-to-End Processing Pipeline (Overview)

The diagram below shows how a request flows from the user through every stage. Stages 0–6 are *setup* (done once or rarely); Stage 7–8 happen *per request*; Stages 9–10 wrap around everything.

```
        ┌─────────────────────────────────────────────────────────────┐
        │  Stage 0: Account, API keys, org/project, RBAC              │
        └─────────────────────────────────────────────────────────────┘
                                  │
        ┌─────────────────────────────────────────────────────────────┐
        │  Stage 1: Discover model (catalog, flavors, provider route)  │
        └─────────────────────────────────────────────────────────────┘
                                  │
        ┌─────────────────────────────────────────────────────────────┐
        │  Stage 2: Package (weights, LoRA, container, quantize)       │
        └─────────────────────────────────────────────────────────────┘
                                  │
        ┌─────────────────────────────────────────────────────────────┐
        │  Stage 3: Provision compute (GPU, region, create deployment) │
        └─────────────────────────────────────────────────────────────┘
                                  │
        ┌─────────────────────────────────────────────────────────────┐
        │  Stage 4: Configure engine (batch, KV cache, spec decoding)  │
        └─────────────────────────────────────────────────────────────┘
                                  │
        ┌─────────────────────────────────────────────────────────────┐
        │  Stage 5: Lifecycle (promote, roll, scale, wake, health)    │
        └─────────────────────────────────────────────────────────────┘
                                  │
   User request ──▶ ┌───────────────────────────────────────────────┐
        │  Stage 6: Route (gateway, affinity, rate limit, tier)       │
        └─────────────────────────────────────────────────────────────┘
                                  │
        ┌─────────────────────────────────────────────────────────────┐
        │  Stage 7: Execute (chat / embed / rerank / vision / batch)   │
        └─────────────────────────────────────────────────────────────┘
                                  │
        ┌─────────────────────────────────────────────────────────────┐
        │  Stage 8: Control output (sampling, JSON, tools, reasoning) │
        └─────────────────────────────────────────────────────────────┘
                                  │
        ┌─────────────────────────────────────────────────────────────┐
        │  Stage 9: Observe (metrics, logs, billing, attribution)     │
        └─────────────────────────────────────────────────────────────┘
                                  │
        ┌─────────────────────────────────────────────────────────────┐
        │  Stage 10: Reliability & security (retries, TLS, compliance) │
        └─────────────────────────────────────────────────────────────┘
```

The rest of this document is one section per stage. Each section contains: (a) the **concepts**, (b) the **alternative approaches** different platforms offer for the same step, (c) a **name-mapping table**, and (d) the **unified API surface** that a "super complete" platform would expose.

---

## Stage 0 — Account, Authentication & Access Control

### Concepts

Before any inference, you must establish identity and scope. All six platforms use a secret **API key** sent in an HTTP header on every request. Keys are created in a web console (and sometimes via a management API), shown in full only once, and can be revoked. Around the key, platforms layer **organizations**, **projects**, **groups**, **cost centers**, and **role-based access control (RBAC)** to scope who can do what and to attribute spending.

### Alternative approaches for authentication

| Approach | Platforms | Notes |
|---|---|---|
| `Authorization: Bearer <key>` | Baseten, Fireworks, HF, Nebius, RunPod, Together | The de-facto standard; OpenAI-compatible. |
| `Authorization: Api-Key <key>` | Baseten (legacy), Baseten Frontier Gateway | Legacy variant still accepted. |
| `Authorization: <key>` (bare) | RunPod (Serverless native) | Bearer preferred. |
| HF fine-grained token (`hf_...`) | Hugging Face | Scoped per-repo/per-task permissions. |
| Federated API key (minted under a group) | Baseten Frontier Gateway | For reselling a hosted model under a branded URL. |
| Register-an-existing-key (Ed25519-signed) | Baseten Frontier Gateway | Bring your own key with a signature. |
| SSO + SCIM | Baseten (Enterprise) | Enterprise identity federation. |

### Alternative approaches for scoping & RBAC

| Approach | Platforms |
|---|---|
| Workspace/personal key types (`PERSONAL`, `WORKSPACE_MANAGE_ALL`, `WORKSPACE_INVOKE`, `WORKSPACE_EXPORT_METRICS`) | Baseten |
| Per-key scope to specific model IDs | Baseten |
| Hierarchical groups with inherited rate/usage limits (`INDEPENDENT` vs `CASCADING`) | Baseten Frontier Gateway |
| Project-scoped keys | Together, Nebius (`ai_project_id`) |
| Per-endpoint key permissions (`None`/`Restricted`/`Read-Write`/`Read-Only`) | RunPod |
| Organizations + Resource Groups (Enterprise) | Hugging Face |
| Cost centers (team/project/department) | RunPod, Together (project cost analytics) |
| `X-HF-Bill-To` / `bill_to` header for org cost attribution | Hugging Face |

### Name-mapping table — Stage 0

| Concept | Baseten | Fireworks | HF | Nebius | RunPod | Together |
|---|---|---|---|---|---|---|
| Secret credential | API key | API key | HF token | API key | API key | API key |
| Header | `Authorization: Bearer` | `Authorization: Bearer` | `Authorization: Bearer hf_` | `Authorization: Bearer` | `Authorization: Bearer` | `Authorization: Bearer` |
| Scoping unit | Workspace | Account | Organization | Project (`ai_project_id`) | (account) | Project |
| Key shown once | yes | yes | yes | yes | yes | yes |
| Federated/resellable key | Federated API key | — | — | — | — | — |
| Enterprise SSO/SCIM | yes | — | — | — | — | — |

### Unified API surface — Stage 0

```
POST   /v1/api_keys                      # create key (type, name, scope, expiration)
GET    /v1/api_keys                       # list keys (prefix + name only)
DELETE /v1/api_keys/{key_id_or_prefix}    # revoke
POST   /v1/api_keys/register              # register an externally-owned key (signed)
# Org / project / group / cost-center management
POST   /v1/organizations
POST   /v1/organizations/{org}/projects
POST   /v1/gateway/groups                 # hierarchical billing entity (Baseten-style)
GET    /v1/gateway/groups/{id}/usage      # usage against limits
```

Request header on every call: `Authorization: Bearer <key>` (or `Api-Key` legacy). Optional `X-Bill-To: <org>` for cost attribution.

---

## Stage 1 — Model Discovery & Selection

### Concepts

Once authenticated, you must choose **which model** to run and **which variant/flavor** of it. Platforms expose a browsable catalog (web UI + `GET /v1/models`), and most let you disambiguate between providers, performance tiers, and quantization levels using a **suffix** or **separate slug** on the model id. Some platforms route the same model across multiple backend providers (HF router) and let you pick by speed, price, or preference.

### Alternative approaches for model identification

| Approach | Platforms | Example |
|---|---|---|
| Namespace/name slug | Together, Nebius | `moonshotai/Kimi-K2.6`, `meta-llama/Meta-Llama-3.1-70B-Instruct` |
| `accounts/fireworks/models/<id>` path | Fireworks | `accounts/fireworks/models/deepseek-v3p1` |
| HF Hub id with `:policy`/`:provider` suffix | Hugging Face | `Qwen/Qwen3-32B:fastest`, `...:groq` |
| `-fast` suffix on slug | Nebius | `meta-llama/...-fast` |
| Router path for Fast variant | Fireworks | `accounts/fireworks/routers/<id>-fast` |
| Deployment name as `model` (leading slash) | Together dedicated | `/Qwen/Qwen3.5-9B-FP8-bb04c904` |
| Deployment `Name` as `model` | Fireworks on-demand | `accounts/<acct>/deployments/<id>` |
| `model#deployment` composite | Fireworks multi-LoRA | `accounts/.../ft-model#accounts/.../dep` |
| `routing_key` returned at endpoint creation | Nebius dedicated | passed as `model` to data plane |
| Endpoint subdomain | Baseten, HF | `https://model-{id}.api.baseten.co`, `https://{id}.{region}.{cloud}.endpoints.huggingface.cloud` |
| Model slug per environment | Baseten | `your-org/your-model` (Frontier Gateway slug) |

### Alternative approaches for variant/flavor selection

| Approach | Platforms | Notes |
|---|---|---|
| **Fast / Base flavors** via suffix | Nebius (`-fast`), Fireworks (`routers/...-fast`), HF (`:fastest`) | Identical outputs, lower latency, higher price. |
| **Deployment shapes** (Fast / Throughput / Minimal) | Fireworks | Pre-configured templates bundling hardware + quantization. |
| **Serving paths** (Standard / Priority / Fast) | Fireworks | Standard = default; Priority = `service_tier: "priority"`; Fast = different model id. |
| **Service tiers** | Nebius (`auto`/`default`/`over-limit`/`flex`/`no-limit`), Fireworks (`priority`) | Controls rate-limit/over-limit behavior per request. |
| **Provider routing policy** | HF (`:fastest`/`:cheapest`/`:preferred`/`:<provider>`) | Multi-provider router picks backend. |
| **Templates** (model + flavor + gpu + region combos) | Nebius | Source of truth for valid combinations. |
| **Quantization variant in slug** | Together (`Qwen3.5-9B-FP8`), RunPod (`Qwen3-32B-AWQ`) | Baked into model id. |

### Name-mapping table — Stage 1

| Concept | Baseten | Fireworks | HF | Nebius | RunPod | Together |
|---|---|---|---|---|---|---|
| Catalog endpoint | `GET /v1/models` | `GET /v1/models` | `GET /v1/models` (router) + Hub API | `GET /v1/models` | `GET /v2/{model}/openai/v1/models` | `GET /v1/models` |
| Model id field | `model` slug | `model` (accounts/...) | `model` (Hub id + suffix) | `model` (HF-style or routing_key) | `MODEL_NAME` env / `model` | `model` (namespace/name) |
| Fast variant | (engine choice) | `routers/<id>-fast` | `:fastest` suffix | `-fast` suffix | — | (Turbo suffix) |
| Performance tier param | — | `service_tier: "priority"` | — | `service_tier` enum | — | — |
| Provider selection | — | — | `:provider` suffix / `provider` client | — | — | — |
| Deployment shape | — | `--deployment-shape` | — | template `flavor_name` | template | hardware id |
| Model "warm" status | — | Serverless tag | `inference: "warm"` | — | — | — |

### Unified API surface — Stage 1

```
GET  /v1/models                                  # list all (with per-provider pricing/latency/throughput/features)
GET  /v1/models/{model_id}                       # one model with all variants/providers
GET  /v1/models?provider={p}&task={t}            # filter
GET  /v0/templates                               # deployable model+flavor+gpu+region combos (dedicated)
```

Model id grammar (unified): `<namespace>/<model>[:<flavor>][@<snapshot>][#<deployment>]` with optional `:provider`/`:policy` suffixes for routed serverless.

---

## Stage 2 — Model Packaging & Artifact Management

### Concepts

If you go beyond the serverless catalog, you must give the platform your model artifacts: weights, LoRA adapters, a Docker container, or a custom handler. Platforms differ in how much they take from you (a HuggingFace repo URL, a signed upload, a container image, or a Python function) and how they cache/mirror those artifacts close to GPUs to reduce cold starts.

### Alternative approaches for supplying weights

| Approach | Platforms | Notes |
|---|---|---|
| Reference a HuggingFace repo (no upload) | Baseten (`hf://`), HF Endpoints, RunPod (`MODEL_NAME`), Together, Nebius (template `huggingface_url`), Fireworks (`huggingFaceUrl`) | Most common; platform pulls weights. |
| 4-step signed upload (create → get URLs → PUT → validate) | Fireworks | For custom models not on HF. |
| Files API upload (archive) | Nebius (`upload-custom-model-archive`), Together (`client.files`) | For custom weights. |
| `weights` block with `source` URI (`hf://`, `s3://`, `gs://`, `azure://`, `r2://`, `bt://`, `cw://`) | Baseten | BDN mirrors to blob storage. |
| Custom container image (Docker Hub, ECR, ACR, GCR, GHCR, NGC) | HF, RunPod, Together (dedicated containers), Baseten (`base_image`) | You build; platform runs. |
| Custom handler (`handler.py` / `EndpointHandler` / `Sprocket` / Truss `Model`) | HF Toolkit, RunPod, Together, Baseten | Python class with `__init__`/`load` + `__call__`/`predict`. |

### Alternative approaches for LoRA adapters

| Approach | Platforms | Notes |
|---|---|---|
| **Live merge** — LoRA weights merged into base at deploy time | Fireworks | One LoRA per deployment; no runtime overhead. |
| **Multi-LoRA** — base deployed with addon support; adapters loaded at request time | Fireworks (`--enable-addons`, BF16 only), Baseten Engine-Builder-LLM (`lora_adapters`), HF vLLM (`enable_lora`), RunPod vLLM | Switch adapter via `model` field; some overhead. |
| **LoRA in image-gen request** | Nebius (`loras: [{scale, url}]`) | Per-request LoRAs for diffusion. |
| **Merge into base weights** (single adapter) | Baseten recommendation | Better perf than multi-LoRA for one adapter. |

### Alternative approaches for quantization

| Format | Memory reduction | Platforms | Notes |
|---|---|---|---|
| FP16/BF16 (`no_quant`) | none | Baseten, HF, RunPod, Together, Nebius | Baseline. |
| FP8 / FP8_KV | ~50–60% | Baseten (TRT-LLM), Fireworks, Together, Nebius, HF vLLM (`kv_cache_dtype`) | Most common compression. |
| FP4 / FP4_KV / FP4_MLP_ONLY | ~75–80% | Baseten (B200 only), Together (`MXFP4`), Fireworks | Blackwell GPUs. |
| INT8 / SmoothQuant | ~50% | Baseten Engine-Builder-LLM v1 | |
| AWQ / GPTQ | ~50–75% | HF (TGI/SGLang), RunPod (`Qwen3-32B-AWQ`), Together | Pre-quantized checkpoints. |
| GGUF | varies | HF llama.cpp, RunPod | For llama.cpp engine. |
| bitsandbytes | varies | HF TGI | |

### Alternative approaches for cold-start mitigation (artifact side)

| Approach | Platforms |
|---|---|
| **BDN** (Baseten Delivery Network) — mirrors weights to multi-tier cache (blob → cluster → node) | Baseten |
| **FlashBoot** — state retention / image pre-loading | RunPod |
| **Model caching** on hosts (HF model cache) | RunPod, HF |
| **Compilation caching** (`b10cache`) — persists `torch.compile`/CUDA graph artifacts | Baseten |
| **Network volumes** shared across workers (no re-download) | RunPod |

### Name-mapping table — Stage 2

| Concept | Baseten | Fireworks | HF | Nebius | RunPod | Together |
|---|---|---|---|---|---|---|
| Packaging framework | Truss (`config.yaml`) | (deployment create) | Endpoint config | template + custom_weights_id | Template + handler | Jig + Sprocket |
| Weights source | `weights[].source` | `huggingFaceUrl` + upload | Hub repo | `custom_weights_id` | `MODEL_NAME` env | model id |
| Custom code entrypoint | `Model.predict` | — | `EndpointHandler.__call__` | — | `handler(job)` | `Sprocket.predict` |
| Init hook | `Model.load` | — | `EndpointHandler.__init__` | — | (worker start) | `Sprocket.setup` |
| LoRA runtime switch | `model` field | `model#deployment` | `enable_lora` | — | vLLM env | — |
| Quantization config | `quantization_type` | shape | engine params | flavor | AWQ/GPTQ in slug | in slug |
| Artifact cache | BDN | — | HF cache | — | FlashBoot/model cache | — |

### Unified API surface — Stage 2

```
# Custom model upload (Fireworks-style 4-step)
POST   /v1/accounts/{acct}/models                              # register
POST   /v1/accounts/{acct}/models/{id}:getUploadEndpoint        # signed URLs
PUT    <signed-url>                                              # upload file
GET    /v1/accounts/{acct}/models/{id}:validateUpload            # finalize → READY

# Weight sources (Baseten-style)
weights:
  - source: hf://org/repo@revision
    mount_location: /weights
    auth: {auth_method, auth_secret_name}
    allow_patterns: ["**/*.safetensors"]

# LoRA
POST /v1/deployments/{id}/lora_adapters          # load adapter (multi-LoRA)
# request-time selection: model = "<base>#<adapter>"

# Quantization
config.build.quantization_type: fp8 | fp8_kv | fp4 | fp4_kv | fp4_mlp_only | no_quant | int8
config.build.calib_size, calib_dataset, calib_max_seq_length
```

---

## Stage 3 — Compute Provisioning & Deployment Creation

### Concepts

This is the step where you actually reserve GPUs. You choose a **GPU type** and **count**, a **region/data center**, a **deployment mode** (serverless needs nothing; dedicated/cluster needs creation), and any **template/shape** that bundles hardware + quantization. The platform returns an identifier you then use as the `model` field in inference calls (or a subdomain you call directly).

### Alternative approaches for deployment modes

| Mode | Platforms | Billing | Key trait |
|---|---|---|---|
| Serverless / public endpoints | All six | per token | No setup; shared fleet. |
| Dedicated endpoint / deployment | Baseten, Fireworks, HF, Nebius, Together | per minute/GPU-second | Reserved GPUs; autoscaling. |
| Provisioned throughput (PTU) | Together | per PTU/min, 1-month min | SLA-backed reserved capacity. |
| Dedicated containers | Together | per-replica resources | You bring Docker image + queue. |
| GPU clusters (K8s/Slurm) | Together | per-GPU hourly or reservation 1–90 days | Full cluster control. |
| Pods (raw GPU instance) | RunPod | per second/minute | Single instance, no autoscaling. |
| Sandboxes (code execution) | Nebius | (beta) | Git-branchable code envs. |

### Alternative approaches for hardware selection

| Approach | Platforms |
|---|---|
| Named instance types (`L4:4x16`, `H100:8`, `A100:8x96x1152`) | Baseten |
| Accelerator enum + count (`NVIDIA_H100_80GB`, `--accelerator-count`) | Fireworks, Nebius (`gpu-h100-sxm`), Together (`1x_nvidia_h100_80gb_sxm`) |
| GPU type string IDs (~50 NVIDIA/AMD) | RunPod |
| GPU pools grouped by architecture+VRAM (`AMPERE_80`, `HOPPER_141`) | RunPod |
| Cloud provider + instance (AWS/Azure/GCP, `nvidia-a100`, `inf2`) | Hugging Face |
| Template-driven (valid gpu_type + flavor + region combos) | Nebius |

### Alternative approaches for region/placement

| Approach | Platforms |
|---|---|
| Region enum set at creation, immutable | Fireworks (`GLOBAL`/`US`/`EUROPE`/`APAC`), Nebius (`eu-north1`/`us-central1`/...), Together (`us-central-8`), RunPod (26 data centers) |
| Cloud + region (AWS/Azure/GCP, e.g. `East US`) | Hugging Face |
| Per-endpoint subdomain per region | HF, Nebius (data-plane base URL per region) |
| Data center priority (`availability` vs `custom`) | RunPod |
| Country code restriction | RunPod |

### Alternative approaches for parallelism

| Strategy | Platforms | Notes |
|---|---|---|
| Tensor parallelism (split weights across GPUs per layer) | Baseten (`tensor_parallel_count`/`tensor_parallel_size`), HF vLLM, RunPod vLLM (`TENSOR_PARALLEL_SIZE`), Nebius (`gpu_count`) | Required when model > 1 GPU VRAM. |
| Data parallelism (independent model copies) | HF vLLM (`Data Parallel Size`) | Higher throughput when model fits on fewer GPUs. |
| Pipeline parallelism | HF SGLang | |
| Expert parallelism (MoE) | HF SGLang | |
| Combined TP × DP = total GPUs | HF vLLM | Trade-off: more DP → more throughput, less KV cache; more TP → longer context. |

### Name-mapping table — Stage 3

| Concept | Baseten | Fireworks | HF | Nebius | RunPod | Together |
|---|---|---|---|---|---|---|
| Deployment unit | Deployment | Deployment | Endpoint | Dedicated endpoint | Serverless endpoint / Pod | Dedicated endpoint / cluster |
| Creation return id | deployment id | deployment `Name` | endpoint name | `endpoint_id` + `routing_key` | endpoint id / pod id | endpoint id + `name` |
| GPU selection | `instance_type` | `--accelerator-type` | instance type | `gpu_type` + `gpu_count` | `gpuTypeIds` + `gpuCount` | `--hardware` |
| Region | (per env host) | `--region` | cloud + region | `region` (immutable) | `dataCenterIds` | `--region` |
| Shape/template | (config.yaml) | `--deployment-shape` | (engine + instance) | template | `templateId` | hardware id |
| Min/max replicas | `min_replica`/`max_replica` | `--min/max-replica-count` | min/max replicas | `scaling.min/max_replicas` | `workersMin`/`workersMax` | `--min/max-replicas` |
| GPUs per replica | (instance_type) | `--accelerator-count` | instance size | `gpu_count` | `gpuCount` | multi-GPU hardware id |

### Unified API surface — Stage 3

```
# List hardware / templates
GET  /v0/templates                  # Nebius-style: valid model+flavor+gpu+region combos
GET  /v1/hardware?model={id}        # Together-style: hardware options for a model
GET  /v1/availability-zones

# Create dedicated deployment
POST /v1/deployments
body: {
  name, model_name, flavor_name, gpu_type, gpu_count, region,
  scaling: {min_replicas, max_replicas},
  custom_weights_id, quantization, tensor_parallel_size,
  autoscaling_settings: {...}, engine_config: {...}
}
→ returns {id, routing_key/name, state}

# Lifecycle
GET    /v1/deployments
GET    /v1/deployments/{id}
PATCH  /v1/deployments/{id}         # update (region immutable)
DELETE /v1/deployments/{id}
POST   /v1/deployments/{id}/start
POST   /v1/deployments/{id}/stop
POST   /v1/deployments/{id}/wake     # wake scaled-to-zero
```

---

## Stage 4 — Inference Engine Configuration

### Concepts

The inference engine is the software that loads weights and runs generation. You either pick a managed engine (the platform's default for your model) or, on dedicated deployments, configure engine-specific parameters that control batching, KV cache, speculative decoding, and parallelism. This is where most performance tuning happens.

### Alternative approaches for engine choice

| Engine | Platforms | Best for |
|---|---|---|
| **vLLM** | HF, RunPod (primary), Nebius (underlying), Together (underlying) | High throughput; PagedAttention; continuous batching; multi-backend. |
| **TGI** (Text Generation Inference) | HF (maintenance mode) | Legacy; migrating to vLLM. |
| **SGLang** | HF | RadixAttention prefix caching; structured outputs; multi-LoRA batching. |
| **llama.cpp** | HF | GGUF models; CPU+GPU. |
| **TEI** (Text Embeddings Inference) | HF | Production embeddings/rerank. |
| **TensorRT-LLM** (via BIS-LLM / Engine-Builder-LLM / BEI) | Baseten | Optimized for MoE, dense LLMs, embeddings. |
| **BIS-LLM** (Baseten Inference Stack v2) | Baseten | MoE + large dense; token-based autoscaling, KV-aware routing, disaggregated serving, speculative decoding. |
| **Engine-Builder-LLM** (v1) | Baseten | Dense text gen; lookahead decoding. |
| **BEI / BEI-Bert** | Baseten | Embeddings, reranking, classification, NER. |
| **Inference Toolkit** (Transformers/Sentence-Transformers/Diffusers wrapper) | HF | Fallback for unsupported models. |
| **Custom container** | HF, RunPod, Together, Baseten | Any Docker image; must expose `/health`. |
| **Ollama / custom** | RunPod Pods | You run the server inside a Pod. |

### Alternative approaches for performance optimizations

| Technique | Platforms | Notes |
|---|---|---|
| **PagedAttention / paged KV cache** | Baseten (`paged_kv_cache`), HF vLLM, Nebius, RunPod vLLM | Splits sequences into chunks; reduces memory. |
| **Flash Attention** | HF TGI/SGLang, Nebius | Modified attention; fewer computations. |
| **Continuous batching** | All (via vLLM/TGI/SGLang/TRT-LLM) | Batches multiple sequences dynamically. |
| **Chunked prefill** | Baseten (`enable_chunked_context`), HF SGLang, vLLM | Process prefill in chunks alongside decode. |
| **Context/prompt caching** | All (automatic on serverless; configurable on dedicated) | Reuse KV from overlapping prefixes. |
| **KV cache quantization** (`fp8`, `fp8_e5m2`, `fp8_e4m3`) | Baseten (`FP8_KV`), HF vLLM/SGLang, Nebius | Compress KV cache. |
| **Speculative decoding** | Baseten (lookahead/Eagle/MTP/N-gram), Fireworks, Together (default on), Nebius, HF vLLM | Draft tokens verified by main model. |
| **Disaggregated serving** (prefill/decode on separate worker groups) | Baseten BIS-LLM (Enterprise) | |
| **KV-aware routing** | Baseten BIS-LLM, Fireworks (session affinity) | Route to replica with cached prefix. |
| **Tensor parallelism / data parallelism** | Baseten, HF, RunPod, Nebius | See Stage 3. |

### Alternative approaches for speculative decoding

| Method | Platforms | Notes |
|---|---|---|
| **Lookahead decoding** (n-gram) | Baseten Engine-Builder-LLM (`LOOKAHEAD_DECODING`, `enable_b10_lookahead`) | 2–4× faster; sampling disabled (temp=0). |
| **Eagle / MTP** (model-based) | Baseten BIS-LLM | Draft model predicts tokens. |
| **N-gram speculation** | Baseten BIS-LLM | |
| **Draft model speculation** | Fireworks, HF vLLM | |
| **Default-on speculation** | Together dedicated (disable with `--no-speculative-decoding`) | |

### Name-mapping table — Stage 4

| Concept | Baseten | Fireworks | HF | Nebius | RunPod | Together |
|---|---|---|---|---|---|---|
| Engine selector | `trt_llm.inference_stack` (v1/v2) | shape | engine choice | (managed) | worker image | (managed) |
| Max batch size | `max_batch_size` | shape | `Max Number of Sequences` | (managed) | `MAX_CONCURRENCY` | (managed) |
| Max tokens per batch | `max_num_tokens` | — | `Max Number of Batched Tokens` | — | — | — |
| KV cache fraction | `kv_cache_free_gpu_mem_fraction` | — | `KV Cache DType` | — | `GPU_MEMORY_UTILIZATION` | — |
| Chunked prefill | `enable_chunked_context` | — | SGLang `chunked prefill size` | — | — | — |
| Speculative decoding | `speculator` block | shape option | vLLM arg | (flavor) | — | default on, `--no-speculative-decoding` |
| Tensor parallel | `tensor_parallel_count/size` | `--accelerator-count` | `Tensor Parallel Size` | `gpu_count` | `TENSOR_PARALLEL_SIZE` | multi-GPU hardware id |

### Unified API surface — Stage 4

```
config.engine: vllm | tgi | sglang | llama_cpp | tei | trt_llm | custom
config.engine_args:
  max_num_seqs, max_num_batched_tokens, tensor_parallel_size, data_parallel_size,
  kv_cache_dtype (auto|fp8|fp8_e4m3), gpu_memory_utilization,
  enable_chunked_prefill, enable_lora, block_size, swap_space,
  max_concurrency, served_model_name
config.speculative_decoding:
  mode: lookahead | eagle | mtp | ngram | draft_model | none
  lookahead_ngram_size, lookahead_verification_set_size, lookahead_windows_size, enable_b10_lookahead
config.quantization_type: fp8 | fp8_kv | fp4 | fp4_kv | fp4_mlp_only | no_quant | int8
config.runtime:
  batch_scheduler_policy, webserver_default_route
```

---

## Stage 5 — Deployment Lifecycle & Traffic Management

### Concepts

After a deployment exists, you manage its lifecycle: promoting versions through environments, rolling traffic from old to new, scaling replicas up/down (including to zero), waking sleeping deployments, and running health checks. This stage is where "devops for models" lives.

### Alternative approaches for environments & promotion

| Approach | Platforms |
|---|---|
| **Environments** (dev/staging/production) with stable endpoints; promote deployments between them | Baseten |
| **Rolling deployments** (gradual traffic shift, max_surge/max_unavailable, pause/resume/cancel) | Baseten |
| **Development deployment** (mutable, live-reload via `truss push --watch`) | Baseten |
| **Labels** (key-value metadata on deployments) | Baseten |
| **Re-point endpoint target** (Frontier Gateway `PATCH endpoint`) | Baseten |
| **Publishing deployments** (make accessible to other users) | Fireworks |
| **Managing default deployments** (which handles model-name-only queries) | Fireworks |
| **Pause/resume** (stop billing, keep config) | HF, Nebius, Together, RunPod Pods |
| **Activate/deactivate** | Baseten |
| **Reset/restart** | RunPod Pods |

### Alternative approaches for autoscaling

| Metric | Platforms | Notes |
|---|---|---|
| **Concurrency per replica** (`concurrency_target`, `target_utilization_percentage`) | Baseten, RunPod (`MAX_CONCURRENCY`) | |
| **Tokens in flight** (`target_in_flight_tokens`) | Baseten BIS-LLM | Mixed-length workloads. |
| **GPU utilization** (threshold %, 1-min window) | HF (default 80%) | |
| **Pending requests** (per replica, 20s window) | HF (default 1.5) | Leading indicator. |
| **Queue delay** (seconds a request waits) | RunPod (`QUEUE_DELAY` scaler) | |
| **Request count** (queue depth) | RunPod (`REQUEST_COUNT` scaler) | |
| **Load targets** (`tokens_generated_per_second`, `prompt_tokens_per_second`, `requests_per_second`, `concurrent_requests`, `default`) | Fireworks | Max replica count across all targets used. |
| **Min/max replicas + windows** (`scale_up_window`, `scale_down_window`, `scale_to_zero_window`) | Fireworks, HF, Together (`inactive_timeout`) | |
| **Scale-down delay / half-life** | Baseten (`scale_down_delay`, `scale_down_half_life_seconds`) | |
| **Idle timeout** | RunPod (`idleTimeout`), Together (`inactive_timeout`) | |
| **K8s Cluster Autoscaler** | Together GPU clusters | Scales nodes. |

### Alternative approaches for scale-to-zero & cold starts

| Approach | Platforms |
|---|---|
| `min_replica=0` / `workersMin=0` / `min_replicas=0` (scale-to-zero) | Baseten, Fireworks, HF, Nebius, RunPod, Together |
| Auto-delete after N days idle (Fireworks 7d) | Fireworks |
| **Wake** endpoint before use | Baseten (`POST /wake`), HF (`resume`), Nebius (`enabled`), Together (`start`) |
| **Pre-warming** (raise min replicas ahead of spikes) | Baseten |
| **X-Scale-Up-Timeout** header (hold request during scale-up) | HF |
| **503 DEPLOYMENT_SCALING_UP** (immediate, must retry) | Fireworks |
| **Parking** (request waits at routing layer for a replica) | Baseten |
| **FlashBoot / BDN / model caching / compilation caching** | RunPod, Baseten, HF |

### Alternative approaches for health checks

| Probe | Platforms | Notes |
|---|---|---|
| **Startup probe** (model finished initializing) | Baseten (30–50 min), HF (`/health` returns 503 until loaded) | |
| **Readiness probe** (can accept traffic) | Baseten, HF, RunPod (`/ping`), Together (`/health`) | |
| **Liveness probe** (restart on failure) | Baseten | |
| **Custom `is_healthy()`** logic | Baseten | |
| **`/health` endpoint** (custom container) | HF, Together | 200 ready / 503 loading. |

### Name-mapping table — Stage 5

| Concept | Baseten | Fireworks | HF | Nebius | RunPod | Together |
|---|---|---|---|---|---|---|
| Stable endpoint across versions | Environment | deployment `Name` | endpoint name | routing_key | endpoint id | endpoint `name` |
| Promote | `POST .../promote` | (re-deploy) | (update endpoint) | (PATCH) | (update) | (update state) |
| Rolling deploy | rolling_deploy config | — | — | — | — | — |
| Scale-to-zero | `min_replica=0` | `--min-replica-count 0` | min replicas 0 | `min_replicas=0` | `workersMin=0` | `--min-replicas 0` |
| Wake | `POST /wake` | (auto on request) | `POST .../resume` | `enabled` toggle | `POST /pods/{id}/start` | `together endpoints start` |
| Health check | startup/readiness/liveness | (READY state) | `/health` | readiness status | `/ping`, `/health` | `/health` |
| Autoscale metric | concurrency/tokens | load-targets | GPU util/pending | (managed) | queue delay/count | (managed) |

### Unified API surface — Stage 5

```
# Environments & promotion
POST   /v1/models/{id}/environments/{env}/promote       # promote deployment to env
PATCH  /v1/models/{id}/environments/{env}              # rolling deploy config
POST   /v1/models/{id}/environments/{env}/{pause|resume|cancel|force_cancel|force_roll_forward}_promotion

# Lifecycle
POST   /v1/deployments/{id}/activate
POST   /v1/deployments/{id}/deactivate
POST   /v1/deployments/{id}/wake
DELETE /v1/deployments/{id}

# Autoscaling
PATCH  /v1/deployments/{id}/autoscaling_settings
body: {min_replica, max_replica, autoscaling_window, scale_down_delay,
       concurrency_target, target_utilization_percentage,
       target_in_flight_tokens, load_targets, scale_up_window, scale_down_window,
       scale_to_zero_window, idle_timeout}

# Health
GET    /health                  # readiness (200 ready / 503 loading)
```

---

## Stage 6 — Request Routing, Gateway & Rate Limiting

### Concepts

When a request arrives, the platform must authenticate it, decide which replica handles it, enforce rate limits, and possibly apply a service tier or gateway policy. This stage is mostly invisible on serverless but configurable on dedicated deployments (session affinity, KV-aware routing) and on managed gateways (Baseten Frontier Gateway: hierarchical groups, federated keys, inherited limits, billing webhooks).

### Alternative approaches for routing

| Approach | Platforms |
|---|---|
| Least-utilized replica (by concurrency headroom) | Baseten |
| **KV-aware routing** (route to replica with cached prefix) | Baseten BIS-LLM |
| **Session affinity / sticky routing** (`x-session-affinity`, `x-multi-turn-session-id`, `user` field) | Fireworks, HF (via `user`), Baseten |
| **Load balancing** (direct routing, no queue) | RunPod LB endpoints |
| **Queue-based** (built-in job queue + handler) | RunPod QB endpoints, Together dedicated containers (queue API) |
| **Regional data-plane URL** (avoid global routing) | Nebius, HF |
| **Frontier Gateway** (branded URL, federated keys, hierarchical groups) | Baseten |
| **Endpoint slug → target mapping** | Baseten Frontier Gateway |

### Alternative approaches for rate limiting

| Approach | Platforms | Notes |
|---|---|---|
| **Static account-tier RPM/TPM** | Baseten Model APIs | |
| **Adaptive/dynamic** (scales with sustained usage, 15-min buckets) | Fireworks, Nebius (×1.2 scale-up, ÷1.5 scale-down, 20× ceiling), Together (dynamic per org per model) | |
| **Per-group, per-model with inheritance** (`INDEPENDENT` vs `CASCADING`) | Baseten Frontier Gateway | |
| **Service tiers** (`auto`/`default`/`over-limit`/`flex`/`no-limit`, `priority`) | Nebius, Fireworks | |
| **Token-based vs request-based** (`TOKEN`/`REQUEST`, `SECOND`/`MINUTE`/`DAY`) | Baseten Frontier Gateway | |
| **Daily usage windows** (reset at midnight UTC) | Baseten Frontier Gateway | |
| **No account-level limits on dedicated** (429 = capacity signal) | Fireworks, Together, Nebius dedicated | |
| **Spend tiers → higher upper bounds** | Fireworks, Together | |
| **Rate-limit response headers** (`x-ratelimit-*`, `Retry-After`, `x-ratelimit-reset`) | Nebius, Together, Fireworks | |

### Name-mapping table — Stage 6

| Concept | Baseten | Fireworks | HF | Nebius | RunPod | Together |
|---|---|---|---|---|---|---|
| Sticky routing | (KV-aware) | `x-session-affinity` | `user` field | — | — | — |
| RL trajectory id | — | `x-multi-turn-session-id` | — | — | — | — |
| Service tier | — | `service_tier` | — | `service_tier` | — | — |
| Rate limit scope | workspace / group | account + model | (provider) | project + model | (account) | org + model |
| Dynamic scaling | — | adaptive | — | dynamic (15-min) | — | dynamic |
| Gateway/resell | Frontier Gateway | — | — | — | — | — |
| Queue vs direct | (parking) | — | — | — | QB vs LB | queue API |

### Unified API surface — Stage 6

```
# Per-request
headers: Authorization, x-session-affinity, x-multi-turn-session-id,
         fireworks-model, fireworks-deployment, X-Bill-To, X-Scale-Up-Timeout
body:    service_tier (auto|default|over-limit|flex|no-limit|priority)

# Gateway management (Baseten Frontier Gateway)
POST   /v1/gateway/endpoints          # map slug → target (provider, model_id, env)
GET    /v1/gateway/endpoints
PATCH  /v1/gateway/endpoints/{id}     # re-point
DELETE /v1/gateway/endpoints/{id}
POST   /v1/gateway/groups             # hierarchical group (metadata, models, hierarchy)
GET    /v1/gateway/groups/{id}/usage
POST   /v1/gateway/groups/{id}/api_keys           # mint federated key
POST   /v1/gateway/groups/{id}/api_keys/register  # register existing key (signed)

# Rate/usage limits (per group, per model)
models[].rate_limits[]:  {type: TOKEN|REQUEST, unit: SECOND|MINUTE, threshold}
models[].usage_limits[]: {type: TOKEN|REQUEST, unit: DAY, threshold}
hierarchy.limit_enforcement: INDEPENDENT | CASCADING
```

---

## Stage 7 — Inference Request Execution (Tasks & Modalities)

### Concepts

This is the actual inference call. All platforms are **OpenAI-compatible** at the core (`POST /v1/chat/completions`), and most also expose legacy completions, embeddings, rerank, image generation, audio (STT/TTS), and a richer **Responses API** (stateful, built-in tools, reasoning). Beyond synchronous streaming, platforms offer **async** (webhook/poll), **batch** (large offline jobs at a discount), and specialized workloads (RL rollout, MoE router replay, vision, video/audio).

### Alternative approaches for execution modes

| Mode | Platforms | Notes |
|---|---|---|
| **Synchronous** (block for result) | All | |
| **Streaming** (SSE, token-by-token) | All | `stream: true`; `stream_options.include_usage`. |
| **Async** (return id, poll/webhook) | Baseten (`async_predict`), RunPod (`/run` + `/status`), Together dedicated containers (queue API), Nebius Responses (`background: true`) | |
| **Batch** (JSONL file, 50% off, 24h window) | Fireworks (`batchInferenceJobs`), Together (`/v1/batches`) | |
| **RL rollout** (session affinity, hot-load, MoE router replay) | Fireworks | |
| **Responses API** (stateful, `previous_response_id`, built-in tools, MCP) | Fireworks, HF, Nebius, Together (via OpenAI compat) | |

### Alternative approaches for task types

| Task | Endpoint | Platforms |
|---|---|---|
| Chat completions | `POST /v1/chat/completions` | All |
| Legacy completions | `POST /v1/completions` | Baseten, Fireworks, HF, Nebius, RunPod, Together |
| Responses API | `POST /v1/responses` | Fireworks, HF, Nebius, Together |
| Embeddings | `POST /v1/embeddings` | All (HF via `/models/{id}`) |
| Rerank | `POST /v1/rerank` | Fireworks, Nebius, Together; Baseten BEI (`/rerank`); HF via logits | Together supports JSON-object documents with `rank_fields` (Llama-Rank-V1). |
| Image generation | `POST /v1/images/generations` | Nebius, Together; HF via `/models/{id}` |
| Video generation | `POST /v1/images/generations` (video) | Together; HF via `/models/{id}` |
| Audio STT | `POST /v1/audio/transcriptions` | Together; HF via `/models/{id}` |
| Audio TTS | `POST /v1/audio/speech` | Together |
| Translation | `POST /v1/audio/translations` | Together; HF via `/models/{id}` |
| Summarization | `/models/{id}` | HF |
| Text/Image classification | `/predict` (BEI), `/models/{id}` | Baseten BEI, HF |
| Token classification (NER) | `/predict_tokens` (BEI-Bert), `/models/{id}` | Baseten, HF |
| Question answering | `/models/{id}` | HF |
| Fill-mask | `/models/{id}` | HF |
| Object detection / image segmentation | `/models/{id}` | HF |
| Zero-shot classification | `/models/{id}` | HF |
| Table QA | `/models/{id}` | HF |
| Image-to-image | `/models/{id}` | HF |
| Text-to-video | `/models/{id}` | HF |

### Alternative approaches for multimodal inputs

| Modality | Platforms | Notes |
|---|---|---|
| **Image** (URL or base64 data URI) | All (via `image_url` content block) | Fireworks max 30 images, <10MB base64, <5MB URL, 1.5s download timeout. |
| **Video** (`video_url` base64) | Fireworks (Qwen3 Omni, Molmo2 — dedicated only), Together (dedicated only) | Fireworks: ≤60s, .mp4, ffmpeg preprocessing (1 FPS, 360p). |
| **Audio** (`audio_url` base64) | Fireworks (Qwen3 Omni — dedicated only) | .ogg Opus, 16kHz mono. |
| **PDF** | None natively (convert pages to images) | Fireworks note. |
| **File inputs** (base64, URL, `preprocess()`) | Baseten, HF, RunPod | |

### Alternative approaches for async & batch

| Feature | Baseten async | RunPod async | Together batch | Fireworks batch | Nebius Responses background |
|---|---|---|---|---|---|
| Submit | `POST /async_predict` | `POST /run` | upload JSONL + `POST /v1/batches` | `POST /batchInferenceJobs` | `background: true` |
| Status | `GET /async_request/{id}` | `GET /status/{id}` | `GET /v1/batches/{id}` | `GET /batchInferenceJobs/{id}` | poll response |
| Cancel | `DELETE /async_request/{id}` | `POST /cancel/{id}` | `POST /v1/batches/{id}/cancel` | — | — |
| Webhook | yes (signed HMAC) | yes (`webhook` field) | — | — | — |
| Priority | `priority` (0/1/2) | — | — | — | — |
| Max queue time | 72h | configurable TTL | 24h window | 12/24/48/72h | — |
| Discount | — | — | 50% off serverless | 50% off | — |
| Retry config | `inference_retry_config` | — | — | — | — |

### Name-mapping table — Stage 7

| Concept | Baseten | Fireworks | HF | Nebius | RunPod | Together |
|---|---|---|---|---|---|---|
| Chat | `/v1/chat/completions` | `/v1/chat/completions` | `/v1/chat/completions` | `/v1/chat/completions` | `/openai/v1/chat/completions` | `/v1/chat/completions` |
| Completions | `/v1/completions` (BIS) | `/v1/completions` | `/models/{id}` (text-gen) | `/v1/completions` | `/openai/v1/completions` | `/v1/completions` |
| Responses | — | `/v1/responses` | `/v1/responses` | `/v1/responses` | — | (via OpenAI compat) |
| Embeddings | `/v1/embeddings` (BEI) | `/v1/embeddings` | `/models/{id}` | `/v1/embeddings` | (vLLM) | `/v1/embeddings` |
| Rerank | `/rerank` (BEI) | `/v1/rerank` | (via logits) | `/v1/rerank` | — | `/v1/rerank` |
| Image gen | — | — | `/models/{id}` | `/v1/images/generations` | (media endpoints) | `/v1/images/generations` |
| Async | `async_predict` | — | — | `background` | `/run` + `/status` | queue API |
| Batch | — | `batchInferenceJobs` | — | — | — | `/v1/batches` |
| Anthropic-compat | `/v1/messages` | `/inference` (Anthropic) | — | — | — | — |

### Unified API surface — Stage 7

```
# Core inference (OpenAI-compatible)
POST /v1/chat/completions
POST /v1/completions
POST /v1/responses
POST /v1/embeddings
POST /v1/rerank
POST /v1/images/generations
POST /v1/audio/transcriptions
POST /v1/audio/translations
POST /v1/audio/speech
GET  /v1/models

# Anthropic-compatible
POST /v1/messages

# Async
POST /v1/async_predict            → {request_id}
GET  /v1/async_request/{id}       → status (QUEUED|IN_PROGRESS|SUCCEEDED|FAILED|EXPIRED|CANCELED|WEBHOOK_FAILED)
DELETE /v1/async_request/{id}
GET  /v1/async_queue_status

# Batch
POST /v1/batches                  (upload JSONL, model, params)
GET  /v1/batches/{id}
POST /v1/batches/{id}/cancel
GET  /v1/batches                  (list)
# Files API for input/output download

# RL rollout (Fireworks-style)
headers: x-multi-turn-session-id, x-session-affinity, fireworks-model, fireworks-deployment
body: include_routing_matrix, logprobs, echo
POST /hot_load/v1/models/hot_load  (reset_prompt_cache: all|new_session|none)
```

---

## Stage 8 — Output Control & Generation Parameters

### Concepts

This stage covers every knob that shapes *what* the model generates, applied per request on top of the engine config from Stage 4. It groups into: sampling, length/stop, penalties, logprobs, reasoning control, structured output, and tool calling.

### Alternative approaches for sampling

| Parameter | Platforms | Notes |
|---|---|---|
| `temperature` | All | 0 = deterministic. |
| `top_p` (nucleus) | All | |
| `top_k` | Fireworks, HF, Nebius, RunPod, Together | |
| `min_p` | Fireworks | Exclude tokens below threshold. |
| `typical_p` | Fireworks, HF | Typical decoding. |
| `mirostat_target`, `mirostat_lr` | Fireworks | Perplexity-targeted adaptive sampling. |
| `seed` | HF, Nebius, Together, RunPod | Best-effort determinism. |
| `n` (multiple completions) | Fireworks, HF, Nebius, RunPod, Together | |
| `best_of` | HF | Generate N, return best. |
| `use_beam_search`, `length_penalty` | RunPod vLLM | |
| `ignore_eos` | Fireworks, RunPod | Force `max_tokens` generation. |
| `skip_special_tokens` | RunPod vLLM | |
| `watermark` | HF TGI | |

### Alternative approaches for penalties

| Parameter | Platforms |
|---|---|
| `frequency_penalty` | All (OpenAI-compatible) |
| `presence_penalty` | All |
| `repetition_penalty` | Fireworks, HF, Nebius, RunPod, Together (exponential) |
| `logit_bias` | Fireworks, Nebius, Together (limited) |

### Alternative approaches for length & stop

| Parameter | Platforms |
|---|---|
| `max_tokens` | All |
| `max_completion_tokens` (incl. reasoning) | Nebius |
| `max_output_tokens` (Responses) | Nebius |
| `stop` / `stop sequences` | All (up to 4) |
| `truncate` (input) | HF |

### Alternative approaches for logprobs & token inspection

| Parameter | Platforms |
|---|---|
| `logprobs` (bool/int) | All |
| `top_logprobs` | All |
| `echo` (return prompt) | Fireworks, HF, Nebius, RunPod, Together |
| `return_token_ids` | Fireworks |
| `raw_output` | Fireworks |
| `decoder_input_details` | HF |
| `details` | HF |
| `include_routing_matrix` (MoE expert selection) | Fireworks |

### Alternative approaches for reasoning control

| Parameter | Platforms | Notes |
|---|---|---|
| `reasoning: {enabled: bool}` (toggle for hybrid models) | Together | |
| `reasoning_effort` (`low`/`medium`/`high`/`xhigh`/`minimal`/`none`) | Fireworks, HF, Nebius, Together | |
| `thinking: {type: "enabled", budget_tokens}` (Anthropic-compat) | Fireworks | |
| `chat_template_kwargs` (`thinking`, `enable_thinking`, `clear_thinking`, `medium_effort`) | Together | |
| `reasoning` object (Responses API, gpt-5/o-series style) | Nebius | |
| Reasoning output field: `reasoning` vs `reasoning_content` vs `<think>` tags | Together (all three variants) | |
| **Preserved thinking** (retain reasoning across turns) | Together (GLM-5) | |
| **Turn-level thinking** (toggle per turn) | Together (GLM-5) | |
| **Interleaved thinking** (default) | Together (GLM-5) | |
| Reasoning tokens billed as completion tokens | Together | |

### Alternative approaches for structured output

| Mode | Platforms | Notes |
|---|---|---|
| `response_format: {type: "json_schema", json_schema: {name, schema, strict}}` | All (OpenAI-compat) | Strict schema enforcement. |
| `response_format: {type: "json_object"}` | All | Valid JSON, no schema. |
| `response_format: {type: "regex", pattern}` | Together | Regex-constrained output. |
| `response_format: {type: "text"}` | HF, Nebius | Plain text. |
| `grammar: {type: "json"|"regex"|"json_schema", value}` | HF (text-gen) | TGI-style. |
| `output_config: {format: {type: "json_schema", schema}}` (Anthropic-compat) | Fireworks | |
| Pydantic model via `client.beta.chat.completions.parse` → `parsed` attribute | Baseten, HF, Nebius | |
| `text.format` / `text_format` (Pydantic, Responses API) | HF, Nebius | |
| `strict: true` + `additionalProperties: false` | HF, Nebius | Strictest. |

### Alternative approaches for function/tool calling

| Feature | Platforms |
|---|---|
| `tools` array (OpenAI function schema) | All |
| `tool_choice` (`auto`/`none`/`required`/`{function}`) | All |
| Graduated tool-choice eagerness (`minimal`/`low`/`medium`/`high`/`xhigh`) | Nebius |
| `max_tool_calls` (Responses API) | Fireworks, Nebius |
| `parallel_tool_calls` | Nebius |
| Tool-call parser env (`mistral`, `hermes`, `llama3_json`, `granite`, ...) | RunPod vLLM |
| **Built-in tools** (file search, code interpreter, web search, computer use, MCP, image generation, local shell) | Nebius Responses |
| **MCP tools** (`type: "mcp"`, `server_url`, `allowed_tools`, `require_approval`) | Fireworks, HF, Nebius |
| **SSE server-executed tools** (`type: "sse"`, `server_url`) | Fireworks |
| **Function client-executed tools** (`type: "function"`) | Fireworks, HF, Nebius |
| Tool-call streaming (incremental `delta.tool_calls`) | HF |

### Name-mapping table — Stage 8

| Concept | Baseten | Fireworks | HF | Nebius | RunPod | Together |
|---|---|---|---|---|---|---|
| Temperature | `temperature` | `temperature` | `temperature` | `temperature` | `temperature` | `temperature` |
| Nucleus | `top_p` | `top_p` | `top_p` | `top_p` | `top_p` | `top_p` |
| Top-k | `top_k` (some models) | `top_k` | `top_k` | — | `top_k` | `top_k` |
| Max tokens | `max_tokens` | `max_tokens` | `max_tokens` | `max_tokens`/`max_completion_tokens` | `max_tokens` | `max_tokens` |
| Stop | — | `stop` | `stop` | `stop` | `stop` | `stop` |
| Logprobs | `logprobs` | `logprobs`+`top_logprobs` | `logprobs`+`top_logprobs` | `logprobs`+`top_logprobs` | `logprobs` | `logprobs` |
| Reasoning toggle | — | `reasoning_effort` | `reasoning_effort` | `reasoning` obj | — | `reasoning:{enabled}` |
| JSON schema | `response_format` | `response_format` | `response_format` | `response_format` | — | `response_format` |
| Regex mode | — | — | `grammar` | — | — | `response_format: regex` |
| Tools | `tools` | `tools` | `tools` | `tools` | (vLLM env) | `tools` |
| Built-in tools | — | MCP/SSE | MCP | file search/code/web/computer/MCP | — | — |

### Unified API surface — Stage 8

```
body: {
  # sampling
  temperature, top_p, top_k, min_p, typical_p, seed, n, best_of,
  mirostat_target, mirostat_lr, use_beam_search, length_penalty,
  ignore_eos, skip_special_tokens, watermark,
  # penalties
  frequency_penalty, presence_penalty, repetition_penalty, logit_bias,
  # length/stop
  max_tokens | max_completion_tokens | max_output_tokens, stop, truncate,
  # inspection
  logprobs, top_logprobs, echo, return_token_ids, raw_output,
  decoder_input_details, details, include_routing_matrix,
  # reasoning
  reasoning: {enabled, effort}, reasoning_effort, thinking: {type, budget_tokens},
  chat_template_kwargs: {thinking, enable_thinking, clear_thinking, medium_effort},
  # structured output
  response_format: {type: text|json_object|json_schema|regex, json_schema:{name,schema,strict}, pattern},
  grammar: {type, value},
  text: {format: {type}},
  # tools
  tools: [{type: function|mcp|sse|file_search|code_interpreter|web_search|computer_use|image_generation|local_shell, ...}],
  tool_choice: auto|none|required|minimal|low|medium|high|xhigh|{function:{name}},
  max_tool_calls, parallel_tool_calls,
  # streaming
  stream, stream_options: {include_usage},
  perf_metrics_in_response,
  # service
  service_tier, user, store, metadata, previous_response_id, background,
  prompt_cache_key, truncation, include
}
```

---

## Stage 9 — Observability, Metrics & Cost Attribution

### Concepts

Once traffic flows, you need to see latency, throughput, errors, GPU utilization, and how much you're spending — and attribute spend to teams/projects/customers. Platforms expose dashboards, OpenMetrics/Prometheus endpoints, runtime logs, and billing webhooks.

### Alternative approaches for metrics

| Metric category | Platforms |
|---|---|
| **Traffic** (RPM, input/output tokens/min, tokens per request) | All |
| **Latency** (p50/p90/p95/p99, TTFT, end-to-end, TPS) | Baseten, HF, Nebius, Together, RunPod |
| **Autoscaling/capacity** (active replicas, ready replicas) | Baseten, HF, Nebius, RunPod |
| **Errors** (by status code, error rate) | All |
| **Engine-level** (`tps_per_request`, `speculation_rate`, `kv_cache_hit_rate`, `cpu/gpu/memory usage`) | Baseten BIS-LLM |
| **Cold start time** | RunPod |
| **Execution/delay time percentiles** | RunPod |

### Alternative approaches for metrics export

| Approach | Platforms |
|---|---|
| **OpenMetrics API** (Prometheus text format) | HF (Team/Enterprise) |
| **Prometheus + Grafana** integration | Nebius |
| **Prometheus-style metrics** endpoint | Fireworks (dedicated deployments) |
| **Analytics dashboard** (web UI) | All |
| **Response headers** (`fireworks-prompt-tokens`, `fireworks-cached-prompt-tokens`, `fireworks-server-time-to-first-token`, `fireworks-sampling-options`, `X-Ratelimit-*`) | Fireworks, Nebius, Together |
| **`perf_metrics_in_response`** (streaming body) | Fireworks |
| **`X-BASETEN-MODEL-PREDICTION-ATTEMPTS`** (retry count) | Baseten |

### Alternative approaches for logs

| Approach | Platforms |
|---|---|
| Runtime logs tab (real-time, filterable) | HF (30-day retention), Baseten, RunPod, Together |
| `tg beta jig logs --follow` | Together dedicated containers |
| Log filtering (timestamp, level, content, replica) | HF |

### Alternative approaches for billing

| Billing model | Platforms | Unit |
|---|---|---|
| Per token (input / cached input / output) | Baseten, Fireworks, HF (routed), Nebius, Together (serverless) | per 1M tokens |
| Per GPU-second | Fireworks (on-demand) | |
| Per minute (dedicated) | Baseten, HF, Nebius, Together | per active replica |
| Per PTU (provisioned throughput) | Together | $0.05/PTU/min, 1-month min |
| Per-GPU hourly / reservation | Together GPU clusters, RunPod Pods | |
| Per-replica resources | Together dedicated containers | cpu/memory/storage/gpu |
| Per successful response (batch) | Fireworks, Together | 50% off serverless |
| Per megapixel (image) | Together | |
| Per second of video | Together | |
| Per second/char of audio | Together | |
| Per compute time × hardware price/sec | HF Inference | |

### Alternative approaches for cost attribution

| Approach | Platforms |
|---|---|
| **Cached input discount** (automatic, 50% default) | Baseten, Fireworks, Together |
| **`X-HF-Bill-To` / `bill_to`** (org) | HF |
| **Project cost analytics** (`api_key_id` per-key tracking) | Together |
| **Cost centers** (team/project/department) | RunPod, Together |
| **Resource Groups** (Enterprise) | HF |
| **Billing webhooks** (per-request token counts, group external id, idempotency key, HMAC-signed) | Baseten Frontier Gateway |
| **`externalEntityId`** on billing events | Baseten Frontier Gateway |
| **Spending tiers → higher rate-limit upper bounds** | Fireworks, Together |
| **Spend limit** (default $80/hr) | RunPod |
| **Prepaid credits / automatic recharge** | HF, RunPod |
| **Savings plans** (3/6 month upfront) | RunPod Pods |

### Name-mapping table — Stage 9

| Concept | Baseten | Fireworks | HF | Nebius | RunPod | Together |
|---|---|---|---|---|---|---|
| Metrics dashboard | console | app usage | analytics | Observability UI | Metrics tab | cost analytics |
| Prometheus | — | dedicated | OpenMetrics API | Prometheus+Grafana | — | — |
| TTFT header | — | `fireworks-server-time-to-first-token` | — | — | — | — |
| Cached tokens | `kv_cache_hit_rate` | `fireworks-cached-prompt-tokens` | `cached_tokens` | `cached_tokens` | — | `cached_tokens` |
| Billing webhook | Frontier Gateway | — | — | — | — | — |
| Per-key spend | — | — | — | — | — | `api_key_id` |
| Cost org | workspace | account | `X-HF-Bill-To` | project | cost center | project |

### Unified API surface — Stage 9

```
GET /v1/endpoints/{id}/open-metrics        # Prometheus text
GET /v1/endpoints/{id}/logs                # runtime logs (filterable)
GET /v1/billing/endpoints                  # billing history
GET /v1/billing/pods
GET /v1/gateway/groups/{id}/usage          # usage against limits
# Billing webhook (inbound to your endpoint)
POST {your_webhook_url}
headers: X-Signature (HMAC-SHA256), X-Request-ID
body: {type: "API_BILLING_USAGE", data: {events: [{idempotencyKey, timestamp, requestId, modelSlug, externalEntityId, apiKeyPrefix, tokens:{input,output,cachedInput}}]}}
```

---

## Stage 10 — Reliability, Security, Compliance & Sandboxes

### Concepts

The final stage wraps around everything: how the platform retries failed requests, how it secures data in transit and at rest, which compliance certifications it holds, and the complementary compute capabilities (sandboxes for code execution).

### Alternative approaches for retries & reliability

| Mechanism | Platforms | Notes |
|---|---|---|
| **Internal retries** (routing layer retries 502/503/504 with exponential backoff) | Baseten (500ms start, ×1.5, cap 60s, 15min total) | |
| **Circuit breaker** (disables retries under memory pressure) | Baseten | |
| **Client-side retry** (exponential backoff on 429/503) | All | |
| **Hedge requests** (duplicate after delay to reduce p99) | Baseten Performance Client | |
| **Async retry config** (`inference_retry_config`: max_attempts, initial_delay, max_delay) | Baseten | |
| **Webhook delivery retries** (2 attempts, ~2s backoff, 10s timeout) | Baseten async | |
| **Billing webhook retries** (exponential 1s→5s, 15s max, dead-letter queue) | Baseten Frontier Gateway | |
| **Parking** (request waits for a replica) | Baseten | |
| **Load shedding** (429 when queued payloads exceed memory) | Baseten | |
| **Request cancellation** (client disconnect → cancel in-flight work, logged 499) | Baseten, RunPod (`/cancel`), Together | |
| **Rolling deployment rollback** (cancel → traffic returns to previous) | Baseten | |
| **Reserved capacity** (guarantee availability during scale-up) | Fireworks, Nebius | |
| **Reservations** (guarantee capacity for clusters) | Together | |

### Alternative approaches for security & compliance

| Feature | Platforms |
|---|---|
| **TLS/SSL in transit** | All |
| **PrivateLink / VPC** (private IP, intra-region) | HF (AWS PrivateLink), Together (clusters) |
| **Global private networking** (Pod-to-Pod) | RunPod |
| **SOC 2** | Baseten, Fireworks, HF, Together |
| **HIPAA** | Fireworks, HF |
| **GDPR / DPA / SCC** | HF, RunPod |
| **Data privacy** (no payload storage; logs 30 days) | HF, Baseten |
| **Model security** (private repos, malware/pickle scans) | HF |
| **RBAC** | HF, Baseten, RunPod, Together |
| **SSO/SCIM** | Baseten (Enterprise) |
| **Non-root containers** | HF |
| **Data residency** (fixed region) | Nebius dedicated |
| **Project-nested observability permissions** | Nebius |
| **Container isolation** | RunPod (Secure Cloud T3/T4) |
| **Enterprise SLA** | Nebius (Enterprise tier), Together (Provisioned Throughput 99%) |

### Alternative approaches for sandboxes & code execution

| Feature | Platforms |
|---|---|
| **Sandboxes** (Git-like branching, checkpoint fork/rollback, OCI images, async operations, resource metrics) | Nebius |
| **Contree SDK / CLI / MCP** | Nebius |
| **SWE-agent preloaded environments** | Nebius |
| **Code interpreter** (built-in tool in Responses API) | Nebius |
| **Local shell / apply-patch tools** | Nebius Responses |

### Name-mapping table — Stage 10

| Concept | Baseten | Fireworks | HF | Nebius | RunPod | Together |
|---|---|---|---|---|---|---|
| Retry header | `X-BASETEN-MODEL-PREDICTION-ATTEMPTS` | — | — | — | — | `x-ratelimit-reset` |
| Circuit breaker | yes | — | — | — | — | — |
| Hedge | Performance Client | — | — | — | — | — |
| PrivateLink | — | — | AWS PrivateLink | — | global networking | clusters |
| SOC2 | yes | yes | yes | — | — | yes |
| HIPAA | — | yes | yes | — | — | — |
| GDPR | — | — | yes | — | yes | — |
| SSO/SCIM | yes | — | — | — | — | — |
| Sandbox | — | — | — | Sandboxes | — | — |
| SLA | — | — | — | Enterprise | — | Provisioned 99% |

### Unified API surface — Stage 10

```
# Reliability
headers: X-BASETEN-MODEL-PREDICTION-ATTEMPTS, x-ratelimit-reset, Retry-After
retry_policy: {max_attempts, initial_delay_ms, max_delay_ms, multiplier, retryable_codes, non_retryable_codes}
hedge: {hedge_delay, hedge_budget_pct}
circuit_breaker: {memory_threshold_pct, cooldown_seconds}

# Security
TLS required; PrivateLink toggle (AWS account id, VPC service name, sharing)
access_level: private | public | authenticated
compliance: SOC2 | HIPAA | GDPR (DPA/SCC)
SSO/SCIM (Enterprise)

# Sandboxes (Nebius-style)
POST /v1/sandboxes                 # create sandbox
POST /v1/sandboxes/{id}/executions  # run code (async operation)
POST /v1/sandboxes/{id}/checkpoints/{cid}/branch  # fork
POST /v1/sandboxes/{id}/checkpoints/{cid}/rollback
GET  /v1/sandboxes/{id}/operations   # poll async ops
```

---

## Cross-Platform Glossary (Alphabetical Name Mapping)

This glossary lists every concept that appears under different names across the six platforms. Use it to translate between systems.

| Unified concept | Baseten | Fireworks | HF | Nebius | RunPod | Together |
|---|---|---|---|---|---|---|
| **Account/workspace** | Workspace | Account | Organization | Project | Account | Project |
| **API key** | API key | API key | HF token | API key | API key | API key |
| **Async inference** | `async_predict` | — | — | `background` | `/run` + `/status` | queue API |
| **Autoscaling metric: concurrency** | `concurrency_target` | `concurrent_requests` | — | — | `MAX_CONCURRENCY` | — |
| **Autoscaling metric: GPU util** | — | — | hardware utilization | — | — | — |
| **Autoscaling metric: pending requests** | — | — | pending requests | — | `REQUEST_COUNT` | — |
| **Autoscaling metric: queue delay** | — | — | — | — | `QUEUE_DELAY` | — |
| **Autoscaling metric: tokens** | `target_in_flight_tokens` | `tokens_generated_per_second` | — | — | — | — |
| **Batch inference** | — | `batchInferenceJobs` | — | — | — | `/v1/batches` |
| **Billing unit (dedicated)** | per minute per replica | per GPU-second | per minute | per GPU/hour (per-minute) | per second/minute | per minute by hardware |
| **Billing unit (serverless)** | per 1M tokens | per 1M tokens | per request | per token | per token | per token/megapixel/second |
| **Cached input discount** | automatic | 50% default | (provider) | — | — | automatic, prefix-based |
| **Chat endpoint** | `/v1/chat/completions` | `/v1/chat/completions` | `/v1/chat/completions` | `/v1/chat/completions` | `/openai/v1/chat/completions` | `/v1/chat/completions` |
| **Cold start mitigation (artifact)** | BDN | — | HF cache | — | FlashBoot | — |
| **Cold start mitigation (compile)** | b10cache | — | — | — | — | — |
| **Completions (legacy)** | `/v1/completions` (BIS) | `/v1/completions` | `/models/{id}` | `/v1/completions` | `/openai/v1/completions` | `/v1/completions` |
| **Custom code entrypoint** | `Model.predict` | — | `EndpointHandler.__call__` | — | `handler(job)` | `Sprocket.predict` |
| **Custom weights** | `weights` block | upload API | Hub repo | `custom_weights_id` | `MODEL_NAME` | model id |
| **Deployment** | Deployment | Deployment | Endpoint | Dedicated endpoint | Serverless endpoint / Pod | Dedicated endpoint / cluster |
| **Deployment id (control plane)** | deployment id | deployment `Name` | endpoint name | `endpoint_id` | endpoint id / pod id | endpoint id |
| **Deployment id (data plane)** | subdomain / env path | deployment `Name` as `model` | endpoint URL | `routing_key` | endpoint id in URL | endpoint `name` as `model` |
| **Development deployment** | development deployment | — | — | — | — | — |
| **Embeddings endpoint** | `/v1/embeddings` (BEI) | `/v1/embeddings` | `/models/{id}` | `/v1/embeddings` | (vLLM) | `/v1/embeddings` |
| **Engine: TRT-LLM** | BIS-LLM / Engine-Builder-LLM / BEI | — | — | (underlying) | — | (underlying) |
| **Engine: vLLM** | (custom container) | (underlying) | vLLM | (underlying) | vLLM (primary) | (underlying) |
| **Environment (stable endpoint)** | Environment | — | — | — | — | — |
| **Fast variant** | (engine choice) | `routers/<id>-fast` | `:fastest` | `-fast` suffix | — | (Turbo suffix) |
| **Federated/resellable key** | Federated API key | — | — | — | — | — |
| **Function calling** | `tools` | `tools` | `tools` | `tools` | (vLLM env) | `tools` |
| **GPU count per replica** | (instance_type) | `--accelerator-count` | instance size | `gpu_count` | `gpuCount` | multi-GPU hardware id |
| **GPU selection** | `instance_type` | `--accelerator-type` | instance type | `gpu_type` | `gpuTypeIds` | `--hardware` |
| **Health check** | startup/readiness/liveness | (READY state) | `/health` | readiness status | `/ping`, `/health` | `/health` |
| **Hierarchical billing** | Groups (INDEPENDENT/CASCADING) | — | — | — | — | — |
| **Idle timeout** | `scale_down_delay` | `--scale-down-window` | scale-to-zero timeout | — | `idleTimeout` | `--inactive-timeout` |
| **Image input** | `image_url` block | `image_url` block | `image_url` block | `image_url` block | (model-dependent) | `image_url` block |
| **JSON schema output** | `response_format` | `response_format` | `response_format` | `response_format` | — | `response_format` |
| **KV cache quantization** | `FP8_KV`/`FP4_KV` | — | `kv_cache_dtype` | (managed) | — | — |
| **LoRA: live merge** | — | live merge | — | — | — | — |
| **LoRA: multi-adapter** | `lora_adapters` | `--enable-addons` | `enable_lora` | — | vLLM env | — |
| **Max replicas** | `max_replica` | `--max-replica-count` | max replicas | `scaling.max_replicas` | `workersMax` | `--max-replicas` |
| **Min replicas** | `min_replica` | `--min-replica-count` | min replicas | `scaling.min_replicas` | `workersMin` | `--min-replicas` |
| **Model catalog** | `GET /v1/models` | `GET /v1/models` | `GET /v1/models` | `GET /v1/models` | Public Endpoints catalog | `GET /v1/models` |
| **Model id (serverless)** | slug (`deepseek-ai/...`) | `accounts/fireworks/models/...` | Hub id + suffix | HF-style or `-fast` | `MODEL_NAME` | `namespace/name` |
| **MCP tools** | — | `type: "mcp"` | `type: "mcp"` | `type: "mcp"` | — | — |
| **MoE router replay** | — | `include_routing_matrix` | — | — | — | — |
| **Packaging framework** | Truss | (deployment create) | endpoint config | template | template + handler | Jig + Sprocket |
| **Pause/resume** | deactivate/activate | — | pause/resume | `enabled` toggle | stop/start | `state: STOPPED/STARTED` |
| **Performance tier** | — | `service_tier: "priority"` | — | `service_tier` enum | — | — |
| **Provisioned throughput** | — | — | — | — | — | PTU |
| **Quantization: FP8** | `fp8` | shape | engine param | flavor | AWQ/GPTQ in slug | in slug |
| **Rate limit (serverless)** | account-tier RPM/TPM | adaptive TPM | (provider) | dynamic 15-min | (account) | dynamic per org per model |
| **Rate limit (dedicated)** | — | no account limits (capacity) | — | no standard limits | — | no shared-fleet limits |
| **Reasoning toggle** | — | `reasoning_effort` | `reasoning_effort` | `reasoning` obj | — | `reasoning:{enabled}` |
| **Region** | (per env host) | `--region` | cloud + region | `region` (immutable) | `dataCenterIds` | `--region` |
| **Rerank endpoint** | `/rerank` (BEI) | `/v1/rerank` | (via logits) | `/v1/rerank` | — | `/v1/rerank` |
| **Responses API** | — | `/v1/responses` | `/v1/responses` | `/v1/responses` | — | (via OpenAI compat) |
| **Rolling deployment** | rolling_deploy | — | — | — | — | — |
| **Scale-to-zero** | `min_replica=0` | `--min-replica-count 0` | min replicas 0 | `min_replicas=0` | `workersMin=0` | `--min-replicas 0` |
| **Service tier** | — | `service_tier` | — | `service_tier` | — | — |
| **Session affinity** | (KV-aware routing) | `x-session-affinity` | `user` field | — | — | — |
| **Speculative decoding** | `speculator` block | shape option | vLLM arg | (flavor) | — | default on |
| **Streaming** | `stream: true` | `stream: true` | `stream: true` | `stream: true` | `stream: true` | `stream: true` |
| **Structured output (regex)** | — | — | `grammar` | — | — | `response_format: regex` |
| **Template/shape** | (config.yaml) | `--deployment-shape` | (engine + instance) | template | `templateId` | hardware id |
| **Tensor parallelism** | `tensor_parallel_count/size` | `--accelerator-count` | `Tensor Parallel Size` | `gpu_count` | `TENSOR_PARALLEL_SIZE` | multi-GPU hardware id |
| **Wake** | `POST /wake` | (auto on request) | `POST .../resume` | `enabled` | `POST /pods/{id}/start` | `together endpoints start` |
| **Webhook (async result)** | yes (HMAC-signed) | — | — | — | `webhook` field | — |
| **Webhook (billing)** | Frontier Gateway | — | — | — | — | — |

---

## Deployment-Mode Decision Matrix

Use this to pick the right mode for your workload.

| Need | Recommended mode | Why |
|---|---|---|
| Quick prototype, popular model, no infra | **Serverless** (any platform) | No setup; per-token; no cold starts. |
| Higher reliability during peak (serverless) | **Priority tier** (Fireworks `service_tier: "priority"`) | Less likely to be load-shed. |
| Lowest latency (serverless) | **Fast variant** (Fireworks `routers/...-fast`, Nebius `-fast`, HF `:fastest`) | Optimized for speed. |
| Custom/fine-tuned model | **Dedicated endpoint** (all) or **Dedicated containers** (Together) | Serverless doesn't support custom weights. |
| LoRA adapters | **Dedicated** (Fireworks multi-LoRA/live-merge, Baseten, HF vLLM) | Serverless doesn't support LoRA. |
| Predictable latency, no rate limits | **Dedicated endpoint** | Reserved hardware. |
| SLA-backed stock model | **Provisioned throughput** (Together) | Defined throughput + 99% reliability. |
| Full control, training, custom stack | **GPU clusters** (Together) or **Pods** (RunPod) | K8s/Slurm; run anything. |
| Custom Docker inference, managed infra | **Dedicated containers** (Together) or **Custom container** (HF/RunPod/Baseten) | Bring image; platform runs it. |
| Large offline batch, 50% off | **Batch** (Fireworks, Together) | Async JSONL; 24h window. |
| RL rollout traffic | **Dedicated** (Fireworks rollout features) | Session affinity, hot-load, MoE replay. |
| Code execution for AI agents | **Sandboxes** (Nebius) | Git-branchable isolated envs. |
| Cost optimization (scale-to-zero) | `min_replica=0` (any dedicated) | Pay nothing when idle; cold start on next request. |
| Cost optimization (always warm) | `min_replica>=1` (any dedicated) | No cold starts; charged at lower rate (RunPod). |
| Data residency / fixed region | **Dedicated** (Nebius, Fireworks, Together) | Region set at creation, immutable. |
| Resell a hosted model under your brand | **Frontier Gateway** (Baseten) | Federated keys, hierarchical limits, billing webhooks. |

---

## Capability Coverage Matrix Per Platform

| Capability | Baseten | Fireworks | HF | Nebius | RunPod | Together |
|---|:---:|:---:|:---:|:---:|:---:|:---:|
| Serverless chat | ✅ | ✅ | ✅ | ✅ | ✅ | ✅ |
| Legacy completions | ✅ | ✅ | ✅ | ✅ | ✅ | ✅ |
| Responses API | — | ✅ | ✅ | ✅ | — | ✅ |
| Anthropic-compat | ✅ | ✅ | — | — | — | — |
| Embeddings | ✅ (BEI) | ✅ | ✅ | ✅ | (vLLM) | ✅ |
| Rerank | ✅ (BEI) | ✅ | (logits) | ✅ | — | ✅ |
| Image generation | — | — | ✅ | ✅ | (media) | ✅ |
| Video generation | — | — | ✅ | — | (media) | ✅ |
| Audio STT/TTS | — | — | ✅ | — | (media) | ✅ |
| Vision (images) | ✅ | ✅ | ✅ | ✅ | (model) | ✅ |
| Video/audio input | — | ✅ (dedicated) | — | — | — | ✅ (dedicated) |
| Function calling | ✅ | ✅ | ✅ | ✅ | (vLLM) | ✅ |
| Structured output (JSON schema) | ✅ | ✅ | ✅ | ✅ | — | ✅ |
| Structured output (regex) | — | — | ✅ | — | — | ✅ |
| Reasoning control | — | ✅ | ✅ | ✅ | — | ✅ |
| MCP tools | — | ✅ | ✅ | ✅ | — | — |
| Built-in tools (file search/code/web/computer) | — | — | — | ✅ | — | — |
| Streaming | ✅ | ✅ | ✅ | ✅ | ✅ | ✅ |
| Async inference | ✅ | — | — | ✅ | ✅ | ✅ (containers) |
| Batch inference | — | ✅ | — | — | — | ✅ |
| RL rollout / MoE replay | — | ✅ | — | — | — | — |
| LoRA (live merge) | — | ✅ | — | — | — | — |
| LoRA (multi-adapter) | ✅ | ✅ | ✅ | — | ✅ | — |
| Custom model upload | ✅ (weights) | ✅ (upload API) | ✅ (Hub) | ✅ (beta) | ✅ (image) | ✅ (image) |
| Custom container | ✅ | — | ✅ | — | ✅ | ✅ |
| Quantization (FP8/FP4) | ✅ | ✅ | ✅ | ✅ | ✅ (AWQ/GPTQ) | ✅ |
| Speculative decoding | ✅ | ✅ | ✅ | ✅ | — | ✅ |
| Dedicated endpoint | ✅ | ✅ | ✅ | ✅ | ✅ (Pod) | ✅ |
| Provisioned throughput (SLA) | — | — | — | — | — | ✅ |
| GPU clusters (K8s/Slurm) | — | — | — | — | — | ✅ |
| Pods (raw instance) | — | — | — | — | ✅ | — |
| Sandboxes (code exec) | — | — | — | ✅ | — | — |
| Environments (dev/staging/prod) | ✅ | — | — | — | — | — |
| Rolling deployments | ✅ | — | — | — | — | — |
| Autoscaling (concurrency) | ✅ | ✅ | — | — | ✅ | ✅ |
| Autoscaling (tokens) | ✅ (BIS) | ✅ | — | — | — | — |
| Autoscaling (GPU util) | — | — | ✅ | — | — | — |
| Scale-to-zero | ✅ | ✅ | ✅ | ✅ | ✅ | ✅ |
| Wake endpoint | ✅ | (auto) | ✅ | ✅ | ✅ | ✅ |
| Health checks (startup/readiness/liveness) | ✅ | — | ✅ | ✅ | ✅ | ✅ |
| KV-aware routing | ✅ (BIS) | ✅ (session affinity) | — | — | — | — |
| Session affinity | ✅ | ✅ | ✅ (user) | — | — | — |
| Service tiers | — | ✅ | — | ✅ | — | — |
| Adaptive/dynamic rate limits | — | ✅ | — | ✅ | — | ✅ |
| Hierarchical billing/groups | ✅ (Frontier) | — | — | — | — | — |
| Federated API keys | ✅ | — | — | — | — | — |
| Billing webhooks | ✅ (Frontier) | — | — | — | — | — |
| Per-key spend tracking | — | — | — | — | — | ✅ |
| Cost attribution (org/project) | ✅ | ✅ | ✅ | ✅ | ✅ | ✅ |
| OpenMetrics/Prometheus | — | ✅ (dedicated) | ✅ | ✅ | — | — |
| Runtime logs | ✅ | — | ✅ | ✅ | ✅ | ✅ |
| PrivateLink/VPC | — | — | ✅ | — | ✅ (global net) | ✅ (clusters) |
| SOC2 | ✅ | ✅ | ✅ | — | — | ✅ |
| HIPAA | — | ✅ | ✅ | — | — | — |
| GDPR | — | — | ✅ | — | ✅ | — |
| SSO/SCIM | ✅ | — | — | — | — | — |
| Enterprise SLA | — | — | — | ✅ | — | ✅ (Provisioned) |
| OpenAI SDK drop-in | ✅ | ✅ | ✅ | ✅ | ✅ | ✅ |
| Native Python SDK | ✅ (perf client) | ✅ (fireworks-ai) | ✅ (huggingface_hub) | — | ✅ (runpod) | ✅ (together) |
| Native CLI | ✅ (baseten) | ✅ (firectl) | ✅ (hf) | — | — | ✅ (tg/together) |
| OpenAPI spec published | ✅ | — | ✅ | ✅ | ✅ | ✅ |

---

*End of specification. This document is the union of capabilities described in `baseten-api.md`, `fireworks-api.md`, `huggingface-api.md`, `nebius-api.md`, `runpod-api.md`, and `together-api.md`. Each stage's alternatives and name-mapping tables are derived directly from those source files; nothing has been added beyond what at least one platform documents.*