# Global Architecture — Consolidated AI Platform Services Map

> **Derived from:** the nine `summary.md` studies in `./platform-studies/{text,images,voice,documents,agents,tools,admin,observability,gpu}`.
> **Purpose:** a single hierarchical architecture diagram and reference covering every service (API endpoint / specific capability) found across all nine studies, grouped into five levels: **Layers → Domains → Products → Modules → Services**.

---

## How to read this document

The platform is described as **five layers**, ordered bottom-up by dependency: a lower layer is *implemented on top of* (called by) the layer above, or is *supervised by* a governance layer that sits across the stack. Inside each layer the services are further grouped by **domain** (business function), then **product** (a coherent shippable family), then **module** (functional sub-component), down to **service** (a specific endpoint or capability).

For every service we record: the concept, the main API surface, the principal providers that implement it, the dependencies on other layers/modules, and notable design choices.

Legend used in the Provider column (abbreviations):

| Abbr | Provider | Abbr | Provider |
|---|---|---|---|
| OAI | OpenAI | Mst | Mistral |
| Ant | Anthropic | Grok | xAI Grok |
| Goo | Google Gemini | OR | OpenRouter |
| Az | Azure AI Language / Computer Vision | BFL | Black Forest Labs (FLUX) |
| Ideo | Ideogram | Recr | Recraft |
| Reve | Reve | DV | DaVinci.ai |
| Rep | Replicate | GCV | Google Cloud Vision |
| 11L | ElevenLabs | Cart | Cartesia |
| Deep | Deepgram | Grad | Gradium |
| Bas | Baseten | Fw | Fireworks |
| HF | Hugging Face | Neb | Nebius |
| RunP | RunPod | Tog | Together |
| Doc | docling | Data | datalab |
| Weav | weaviate | mxb | mixedbread |
| Ligh | LightOn | KG | knowledgegraph |
| IBM | IBM watsonx / DPE | docE | docetl |

---

## Overall architecture at a glance

```
┌─────────────────────────────────────────────────────────────────────────────┐
│ L5  GOVERNANCE · SAFETY · OPERATIONS  (supervises all layers below)          │
│   Identity/Admin · Network/Residency · Billing/Quotas · Telemetry/Traces    │
│   Moderation/Guardrails · Approvals · Evaluation/Datasets · Versioning       │
├─────────────────────────────────────────────────────────────────────────────┤
│ L4  AGENTIC ORCHESTRATION  (built on L2 inference + L3 modalities)            │
│   Agent Definition · Sessions/Runs · Agent Loop/Events · Tools/MCP/Skills    │
│   Permissions/Hooks · Multi-Agent Orchestration · Memory/RAG · Workflows      │
│   Channels · Sandboxes/Environments                                           │
├─────────────────────────────────────────────────────────────────────────────┤
│ L3  AI MODALITY PRODUCTS  (built on L2 intelligence APIs)                     │
│   Text & Conversation · Images & Video · Voice · Documents                    │
├─────────────────────────────────────────────────────────────────────────────┤
│ L2  MODEL INFERENCE & INTELLIGENCE APIs  (built on L1 compute)               │
│   Model Catalog · Generation APIs · Reasoning · Structured Output             │
│   Function/Tool Calling · Streaming · Context Mgmt · Multimodal Input         │
│   Embeddings/Rerank · Files · Batch · Grounding/Citations                     │
├─────────────────────────────────────────────────────────────────────────────┤
│ L1  COMPUTE & MODEL SERVING INFRASTRUCTURE                                    │
│   GPU Provisioning · Model Packaging · Inference Engines · Deployment LCM     │
│   Routing/Gateway · Request Execution · Output Control · Reliability/Sec     │
└─────────────────────────────────────────────────────────────────────────────┘
```

### Layer dependency rules used

1. **L1 → L2**: L1 provisions GPUs/engines/deployments; L2 inference endpoints are served *by* those deployments.
2. **L2 → L3**: L3 modality products (text/image/voice/document) call L2 generation/embedding/structured-output primitives.
3. **L2 + L3 → L4**: L4 agents call L2 model APIs (model call + tools) and consume L3 products (voice channels, file search, image generation tool, code execution container).
4. **L5 → all**: L5 governs (auth, billing, rate limits, moderation, tracing, approvals, evals) and wraps every other layer; it is neither below nor above in the call graph but supervisory.

### Where cross-cutting concerns live

- **Files & Object Storage** — primitive Files API lives in L2 (shared resource); *data retention/residency governance* of those files lives in L5.
- **Embeddings & Rerank** — the embedding/rerank *API primitive* lives in L2; the document *indexing/retrieval pipeline* built on it lives in L3 Documents.
- **Tool calling** — the `tools`/`tool_choice` primitive lives in L2; the built-in tool catalog, MCP connectors, Skills, programmatic calling, tool search, and approval gating live in L4.
- **Moderation/Guardrails** — the moderation API and guardrail engine live in L5; their *attachment points* appear at L2 (inline generation) and L4 (agent input/output/tool boundaries).
- **Conversation state** — `previous_response_id`/Conversations API (model-level state) lives in L3 Text; agent sessions/threads/runs (orchestration-level state) live in L4.
- **Sandboxes** appear in three distinct places: L1 (GPU-platform code-exec envs, Nebius Sandboxes), L4 agent execution environments (managed cloud / self-hosted / git-worktree), and L4 code-execution containers as a built-in tool.
- **Observability** — telemetry/metrics/traces plumbing lives in L5; usage/cost accounting returned inline by inference endpoints is noted at L2 but attributed/billed via L5.

---

# LAYER 1 — COMPUTE & MODEL SERVING INFRASTRUCTURE

> **Purpose.** Rent and operate the physical/virtual GPU hardware, package and deploy models onto it, route inference requests to running replicas, and keep the whole fleet reliable and secure. This layer is the substrate: every L2 inference call is ultimately served by an L1 deployment.
> **Sources:** `gpu/summary.md` (Baseten, Fireworks, HF, Nebius, RunPod, Together).

## Domain L1.A — Account, Access & Hardware Catalog

### Product L1.A.1 — Identity & Key Management

**Module — API Key Management**
- Service: `POST /v1/api_keys` — create key with type/scope/expiration. *Providers*: Bas, Fw, HF, Neb, RunP, Tog. *Dep*: L5 Identity.
- Service: `GET /v1/api_keys` — list (prefix+name only). *Providers*: all L1. *Dep*: L5 Identity.
- Service: `DELETE /v1/api_keys/{id_or_prefix}` — revoke. *Providers*: all L1.
- Service: `POST /v1/api_keys/register` — register an externally-owned Ed25519-signed key. *Providers*: Bas (Frontier Gateway).

**Module — Organization / Project / Group**
- Service: `POST /v1/organizations` — create org/tenant. *Providers*: all (workspace/project scoping).
- Service: `POST /v1/organizations/{org}/projects` — create sub-tenant. *Providers*: Tog, Neb (`ai_project_id`).
- Service: `POST /v1/gateway/groups` — hierarchical billing entity with inherited limits. *Providers*: Bas.
- Service: `GET /v1/gateway/groups/{id}/usage` — usage against limits. *Providers*: Bas.
- Service: cost attribution header `X-HF-Bill-To`/`bill_to`. *Providers*: HF. *Dep*: L5 Billing.

### Product L1.A.2 — Hardware Discovery

**Module — Model & Hardware Catalog**
- Service: `GET /v1/models` — list models with per-provider pricing/latency/throughput/features. *Providers*: all L1.
- Service: `GET /v1/models/{model_id}` — one model with all variants/providers.
- Service: `GET /v1/models?provider=&task=` — filter. *Providers*: HF router.
- Service: `GET /v0/templates` — deployable model+flavor+gpu+region combos (dedicated). *Providers*: Neb.
- Service: `GET /v1/hardware?model={id}` — hardware options for a model. *Providers*: Tog.
- Service: `GET /v1/availability-zones`. *Providers*: Fw, Neb, Tog, RunP.

**Module — Variant & Flavor Selection**
- Service: Fast variant suffix (`-fast`, `:fastest`, `routers/...-fast`). *Providers*: Neb, Fw, HF.
- Service: `service_tier` enum (auto/default/over-limit/flex/no-limit/priority). *Providers*: Neb, Fw.
- Service: Deployment shape templates (Fast/Throughput/Minimal). *Providers*: Fw.
- Service: Provider routing policy (`:fastest`/`:cheapest`/`:preferred`/`:<provider>`). *Providers*: HF.

## Domain L1.B — Model Packaging & Artifact Management

### Product L1.B.1 — Weights Supply

**Module — Weight Sources**
- Service: HF repo reference (`hf://org/repo@revision`, `MODEL_NAME`). *Providers*: Bas, HF, RunP, Tog, Neb, Fw.
- Service: 4-step signed upload (register → getUploadEndpoint → PUT → validateUpload). *Providers*: Fw.
- Service: Files API archive upload (`upload-custom-model-archive`, `client.files`). *Providers*: Neb, Tog.
- Service: `weights[].source` block with URI schemes (hf/s3/gs/azure/r2/bt/cw) + auth + allow_patterns. *Providers*: Bas.
- Service: Custom container image (Docker Hub/ECR/ACR/GCR/GHCR/NGC). *Providers*: HF, RunP, Tog, Bas.
- Service: Custom handler (`handler.py`/`EndpointHandler`/`Sprocket`/Truss `Model`). *Providers*: HF, RunP, Tog, Bas.

**Module — LoRA Adapters**
- Service: Live merge (one LoRA at deploy time, no runtime overhead). *Providers*: Fw.
- Service: Multi-LoRA (base + addons loaded at request time, `model#deployment`, `lora_adapters`, `enable_lora`). *Providers*: Fw, Bas (Engine-Builder), HF vLLM, RunP.
- Service: Per-request image-gen LoRA (`loras:[{scale,url}]`). *Providers*: Neb.

**Module — Quantization**
- Service: FP16/BF16 (`no_quant`). *Providers*: Bas, HF, RunP, Tog, Neb.
- Service: FP8/FP8_KV. *Providers*: Bas (TRT-LLM), Fw, Tog, Neb, HF vLLM.
- Service: FP4/FP4_KV/FP4_MLP_ONLY (Blackwell). *Providers*: Bas (B200), Tog (MXFP4), Fw.
- Service: INT8/SmoothQuant. *Providers*: Bas Engine-Builder v1.
- Service: AWQ/GPTQ pre-quantized checkpoints. *Providers*: HF, RunP, Tog.
- Service: GGUF (llama.cpp). *Providers*: HF, RunP.
- Service: bitsandbytes. *Providers*: HF TGI.

**Module — Cold-Start Mitigation (artifact side)**
- Service: BDN multi-tier weight mirroring (blob→cluster→node). *Providers*: Bas.
- Service: FlashBoot state retention / image pre-loading. *Providers*: RunP.
- Service: HF model cache on hosts. *Providers*: RunP, HF.
- Service: `b10cache` compilation caching (torch.compile / CUDA graphs). *Providers*: Bas.
- Service: Network volumes shared across workers. *Providers*: RunP.

## Domain L1.C — Compute Provisioning & Deployment

### Product L1.C.1 — Deployment Creation

**Module — Deployment Modes**
- Service: Serverless / shared public endpoints (per-token, no cold start). *Providers*: all L1.
- Service: Dedicated endpoint / deployment (per-minute/GPU-second, reserved GPUs, autoscaling). *Providers*: Bas, Fw, HF, Neb, Tog.
- Service: Provisioned throughput PTU (SLA-backed reserved capacity, 1-month min). *Providers*: Tog.
- Service: Dedicated containers (bring Docker image + job queue). *Providers*: Tog.
- Service: GPU clusters (K8s/Slurm, full control). *Providers*: Tog.
- Service: Pods (raw GPU instance, no autoscaling). *Providers*: RunP.
- Service: Sandboxes (Git-branchable code-exec envs). *Providers*: Neb.

**Module — Hardware Selection**
- Service: Named instance types (`L4:4x16`, `H100:8`). *Providers*: Bas.
- Service: Accelerator enum + count. *Providers*: Fw, Neb, Tog.
- Service: GPU type string IDs (~50 NVIDIA/AMD). *Providers*: RunP.
- Service: GPU pools grouped by architecture+VRAM (`AMPERE_80`). *Providers*: RunP.
- Service: Cloud provider + instance (AWS/Azure/GCP). *Providers*: HF.
- Service: Template-driven valid combos. *Providers*: Neb.

**Module — Region & Placement**
- Service: Region enum at creation (immutable). *Providers*: Fw, Neb, Tog, RunP.
- Service: Cloud + region. *Providers*: HF.
- Service: Per-endpoint subdomain per region. *Providers*: HF, Neb.
- Service: Data center priority / country restriction. *Providers*: RunP.

**Module — Parallelism**
- Service: Tensor parallelism (`tensor_parallel_count/size`, `gpu_count`). *Providers*: Bas, HF, RunP, Neb.
- Service: Data parallelism (`Data Parallel Size`). *Providers*: HF vLLM.
- Service: Pipeline parallelism. *Providers*: HF SGLang.
- Service: Expert parallelism (MoE). *Providers*: HF SGLang.

**Module — Deployment CRUD**
- Service: `POST /v1/deployments` — create (returns id/routing_key/state). *Providers*: all dedicated.
- Service: `GET /v1/deployments`, `GET /v1/deployments/{id}`. *Providers*: all.
- Service: `PATCH /v1/deployments/{id}` (region immutable). *Providers*: Fw, Neb, Tog.
- Service: `DELETE /v1/deployments/{id}`. *Providers*: all.
- Service: `POST /v1/deployments/{id}/start` / `/stop` / `/wake`. *Providers*: Bas, HF, Neb, RunP, Tog.

## Domain L1.D — Inference Engine Configuration

### Product L1.D.1 — Engine Selection & Tuning

**Module — Engine Choice**
- Service: vLLM (PagedAttention, continuous batching). *Providers*: HF, RunP, Neb, Tog.
- Service: TGI (legacy, migrating to vLLM). *Providers*: HF.
- Service: SGLang (RadixAttention, structured output, multi-LoRA batching). *Providers*: HF.
- Service: llama.cpp (GGUF, CPU+GPU). *Providers*: HF.
- Service: TEI (embeddings/rerank). *Providers*: HF.
- Service: TensorRT-LLM (via BIS-LLM / Engine-Builder / BEI). *Providers*: Bas.
- Service: BIS-LLM (MoE+dense, token autoscaling, KV-aware routing, disaggregated serving, spec decoding). *Providers*: Bas.
- Service: BEI / BEI-Bert (embeddings, rerank, classification, NER). *Providers*: Bas.
- Service: Inference Toolkit fallback. *Providers*: HF.
- Service: Custom container (must expose `/health`). *Providers*: HF, RunP, Tog, Bas.
- Service: Ollama / custom inside a Pod. *Providers*: RunP.

**Module — Performance Optimizations**
- Service: PagedAttention / paged KV cache. *Providers*: Bas, HF, Neb, RunP.
- Service: Flash Attention. *Providers*: HF TGI/SGLang, Neb.
- Service: Continuous batching. *Providers*: all (via engines).
- Service: Chunked prefill. *Providers*: Bas, HF SGLang, vLLM.
- Service: Context/prompt caching (automatic serverless; configurable dedicated). *Providers*: all.
- Service: KV cache quantization (`fp8`, `fp8_e5m2`, `fp8_e4m3`). *Providers*: Bas, HF, Neb.
- Service: Speculative decoding (lookahead/Eagle/MTP/N-gram/draft model). *Providers*: Bas, Fw, Tog (default on), Neb, HF vLLM.
- Service: Disaggregated serving (prefill/decode split). *Providers*: Bas BIS-LLM (Enterprise).
- Service: KV-aware routing (route to replica with cached prefix). *Providers*: Bas BIS-LLM, Fw (session affinity).

**Module — Engine Args**
- Service: `max_num_seqs`, `max_num_batched_tokens`, `tensor_parallel_size`, `data_parallel_size`, `kv_cache_dtype`, `gpu_memory_utilization`, `enable_chunked_prefill`, `enable_lora`, `block_size`, `swap_space`, `max_concurrency`, `served_model_name`. *Providers*: configurable on dedicated (Bas, HF, RunP, Tog).

## Domain L1.E — Deployment Lifecycle & Traffic Management

### Product L1.E.1 — Lifecycle Operations

**Module — Environments & Promotion**
- Service: Environments (dev/staging/production) with stable endpoints. *Providers*: Bas.
- Service: `POST /v1/models/{id}/environments/{env}/promote`. *Providers*: Bas.
- Service: Rolling deployments (gradual shift, max_surge/unavailable, pause/resume/cancel/force_roll_forward). *Providers*: Bas.
- Service: Development deployment with `truss push --watch` live-reload. *Providers*: Bas.
- Service: Labels (key-value metadata on deployments). *Providers*: Bas.
- Service: Frontier Gateway endpoint re-point. *Providers*: Bas.
- Service: Publishing / managing default deployments. *Providers*: Fw.
- Service: Pause/resume (stop billing, keep config). *Providers*: HF, Neb, Tog, RunP Pods.
- Service: Activate/deactivate, reset/restart. *Providers*: Bas, RunP.

**Module — Autoscaling**
- Service: Concurrency per replica (`concurrency_target`, `MAX_CONCURRENCY`). *Providers*: Bas, RunP.
- Service: Tokens in flight (`target_in_flight_tokens`). *Providers*: Bas BIS-LLM.
- Service: GPU utilization threshold. *Providers*: HF.
- Service: Pending requests. *Providers*: HF.
- Service: Queue delay / request count scalers. *Providers*: RunP.
- Service: Load targets (tokens_gen/s, prompt_tokens/s, rps, concurrent). *Providers*: Fw.
- Service: `PATCH /v1/deployments/{id}/autoscaling_settings` (min/max replicas, windows, delays). *Providers*: all dedicated.

**Module — Scale-to-Zero & Cold Starts**
- Service: `min_replica=0` / `workersMin=0` / `min_replicas=0`. *Providers*: all.
- Service: Auto-delete after N days idle (Fw 7d). *Providers*: Fw.
- Service: Wake endpoint before use (`POST /wake`, `resume`, `enabled`, `start`). *Providers*: Bas, HF, Neb, RunP, Tog.
- Service: Pre-warming (raise min replicas ahead of spikes). *Providers*: Bas.
- Service: `X-Scale-Up-Timeout` header (hold during scale-up). *Providers*: HF.
- Service: 503 DEPLOYMENT_SCALING_UP (immediate, retry). *Providers*: Fw.
- Service: Parking (request waits at routing layer). *Providers*: Bas.

**Module — Health Checks**
- Service: Startup probe (model finished initializing). *Providers*: Bas, HF.
- Service: Readiness probe. *Providers*: Bas, HF, RunP, Tog.
- Service: Liveness probe (restart on failure). *Providers*: Bas.
- Service: Custom `is_healthy()` logic. *Providers*: Bas.
- Service: `/health` endpoint (200 ready / 503 loading). *Providers*: HF, Tog.

## Domain L1.F — Request Routing, Gateway & Rate Limiting

### Product L1.F.1 — Routing

**Module — Replica Selection**
- Service: Least-utilized replica (concurrency headroom). *Providers*: Bas.
- Service: KV-aware routing. *Providers*: Bas BIS-LLM.
- Service: Session affinity / sticky routing (`x-session-affinity`, `x-multi-turn-session-id`, `user`). *Providers*: Fw, HF, Bas.
- Service: Load balancing (direct, no queue). *Providers*: RunP LB.
- Service: Queue-based (built-in job queue + handler). *Providers*: RunP QB, Tog dedicated containers.
- Service: Regional data-plane URL. *Providers*: Neb, HF.
- Service: Frontier Gateway (branded URL, federated keys, hierarchical groups). *Providers*: Bas.
- Service: `POST /v1/gateway/endpoints` — map slug → target. *Providers*: Bas.
- Service: `POST /v1/gateway/groups/{id}/api_keys` — mint federated key. *Providers*: Bas.

### Product L1.F.2 — Rate Limiting

**Module — Limit Types**
- Service: Static account-tier RPM/TPM. *Providers*: Bas Model APIs.
- Service: Adaptive/dynamic (15-min buckets, ×1.2 scale-up / ÷1.5 scale-down / 20× ceiling). *Providers*: Fw, Neb, Tog.
- Service: Per-group per-model with inheritance (INDEPENDENT vs CASCADING). *Providers*: Bas.
- Service: Service tiers (`auto`/`default`/`over-limit`/`flex`/`no-limit`, `priority`). *Providers*: Neb, Fw.
- Service: Token-based vs request-based (TOKEN/REQUEST, SECOND/MINUTE/DAY). *Providers*: Bas.
- Service: Daily usage windows (reset midnight UTC). *Providers*: Bas.
- Service: Rate-limit response headers (`x-ratelimit-*`, `Retry-After`). *Providers*: Neb, Tog, Fw.

## Domain L1.G — Inference Request Execution

### Product L1.G.1 — Execution Modes

**Module — Sync / Streaming / Async / Batch**
- Service: Synchronous (block for result). *Providers*: all L1.
- Service: Streaming (SSE, token-by-token, `stream:true`, `stream_options.include_usage`). *Providers*: all L1.
- Service: Async (`async_predict`, `/run`+`/status`, queue API, `background:true`). *Providers*: Bas, RunP, Tog, Neb.
- Service: Batch (JSONL, 50% off, 24h window). *Providers*: Fw (`batchInferenceJobs`), Tog (`/v1/batches`).
- Service: RL rollout (session affinity, hot-load, MoE router replay). *Providers*: Fw.
- Service: Responses API (stateful, `previous_response_id`, built-in tools, MCP). *Providers*: Fw, HF, Neb, Tog.

### Product L1.G.2 — Task Types

**Module — Core Tasks**
- Service: `POST /v1/chat/completions`. *Providers*: all L1.
- Service: `POST /v1/completions` (legacy). *Providers*: all L1.
- Service: `POST /v1/responses`. *Providers*: Fw, HF, Neb, Tog.
- Service: `POST /v1/messages` (Anthropic-compat). *Providers*: Bas, Fw.
- Service: `POST /v1/embeddings`. *Providers*: all (HF via `/models/{id}`).
- Service: `POST /v1/rerank`. *Providers*: Fw, Neb, Tog, Bas BEI (`/rerank`); HF via logits.
- Service: `POST /v1/images/generations`. *Providers*: Neb, Tog; HF via `/models/{id}`.
- Service: `POST /v1/audio/transcriptions`, `/v1/audio/translations`, `/v1/audio/speech`. *Providers*: Tog; HF via `/models/{id}`.
- Service: Summarization / classification / NER / QA / fill-mask / object detection / zero-shot / table QA / image-to-image / text-to-video. *Providers*: HF via `/models/{id}`; Bas BEI (`/predict`, `/predict_tokens`).

**Module — Multimodal Input**
- Service: Image (`image_url` block, URL or base64). *Providers*: all L1 (Fw max 30 images).
- Service: Video (`video_url` base64). *Providers*: Fw, Tog (dedicated).
- Service: Audio (`audio_url` base64). *Providers*: Fw (dedicated).
- Service: File inputs (base64, URL, `preprocess()`). *Providers*: Bas, HF, RunP.

### Product L1.G.3 — Async Lifecycle

**Module — Async Job Management**
- Service: `POST /v1/async_predict` → `{request_id}`. *Providers*: Bas.
- Service: `GET /v1/async_request/{id}` (QUEUED/IN_PROGRESS/SUCCEEDED/FAILED/EXPIRED/CANCELED/WEBHOOK_FAILED). *Providers*: Bas.
- Service: `DELETE /v1/async_request/{id}`, `GET /v1/async_queue_status`. *Providers*: Bas.
- Service: `POST /run` + `GET /status/{id}` + `POST /cancel/{id}`. *Providers*: RunP.
- Service: Tog dedicated containers queue API. *Providers*: Tog.
- Service: Async priority levels (0/1/2). *Providers*: Bas.
- Service: Async retry config (`inference_retry_config`). *Providers*: Bas.
- Service: Webhook delivery (signed HMAC, retries 2 attempts ~2s backoff). *Providers*: Bas, RunP.

## Domain L1.H — Output Control & Generation Parameters

### Product L1.H.1 — Sampling & Penalties

**Module — Sampling**
- Service: `temperature`, `top_p`, `top_k`, `min_p`, `typical_p`, `mirostat_target`/`mirostat_lr`, `seed`, `n`, `best_of`, `use_beam_search`, `length_penalty`, `ignore_eos`, `skip_special_tokens`, `watermark`. *Providers*: configurable per platform union.

**Module — Penalties**
- Service: `frequency_penalty`, `presence_penalty`, `repetition_penalty` (exponential on Tog), `logit_bias`.

**Module — Length & Stop**
- Service: `max_tokens` / `max_completion_tokens` / `max_output_tokens`, `stop` (up to 4), `truncate` (input, HF).

### Product L1.H.2 — Reasoning, Structured Output & Tools

**Module — Reasoning Control**
- Service: `reasoning:{enabled}` toggle (hybrid models). *Providers*: Tog.
- Service: `reasoning_effort` (low/medium/high/xhigh/minimal/none). *Providers*: Fw, HF, Neb, Tog.
- Service: `thinking:{type:"enabled",budget_tokens}` (Anthropic-compat). *Providers*: Fw.
- Service: `chat_template_kwargs` (thinking/enable_thinking/clear_thinking/medium_effort). *Providers*: Tog.
- Service: `reasoning` object (Responses API, gpt-5/o-series). *Providers*: Neb.
- Service: Preserved / turn-level / interleaved thinking (GLM-5). *Providers*: Tog.

**Module — Structured Output**
- Service: `response_format:{type:json_schema|json_object|text}` + strict. *Providers*: all OpenAI-compat.
- Service: `response_format:{type:regex,pattern}`. *Providers*: Tog.
- Service: `grammar:{type,value}` (TGI-style). *Providers*: HF.
- Service: `output_config:{format:{type:json_schema,schema}}` (Anthropic-compat). *Providers*: Fw.
- Service: Pydantic via `client.beta.chat.completions.parse`. *Providers*: Bas, HF, Neb.

**Module — Function / Tool Calling**
- Service: `tools` array (OpenAI function schema). *Providers*: all L1.
- Service: `tool_choice` (auto/none/required/minimal/low/medium/high/xhigh/{function}). *Providers*: all; graduated on Neb.
- Service: `max_tool_calls`, `parallel_tool_calls`. *Providers*: Fw, Neb.
- Service: Tool-call parser env (`mistral`/`hermes`/`llama3_json`/`granite`). *Providers*: RunP vLLM.
- Service: Built-in tools (file search, code interpreter, web search, computer use, MCP, image generation, local shell). *Providers*: Neb Responses.
- Service: MCP tools (`type:"mcp"`, `server_url`, `allowed_tools`, `require_approval`). *Providers*: Fw, HF, Neb.
- Service: SSE server-executed tools (`type:"sse"`, `server_url`). *Providers*: Fw.
- Service: Function client-executed tools (`type:"function"`). *Providers*: Fw, HF, Neb.

### Product L1.H.3 — Inspection & Service Tier

**Module — Logprobs / Token Inspection**
- Service: `logprobs`, `top_logprobs`, `echo`, `return_token_ids`, `raw_output`, `decoder_input_details`, `details`, `include_routing_matrix` (MoE expert selection). *Providers*: per platform.

**Module — Service / Store**
- Service: `service_tier`, `user`, `store`, `metadata`, `previous_response_id`, `background`, `prompt_cache_key`, `truncation`, `include`, `perf_metrics_in_response`. *Providers*: per platform.

## Domain L1.I — Observability, Metrics & Cost Attribution

### Product L1.I.1 — Metrics & Logs

**Module — Metrics Export**
- Service: OpenMetrics API (Prometheus text). *Providers*: HF (Team/Enterprise).
- Service: Prometheus + Grafana. *Providers*: Neb.
- Service: Prometheus-style metrics endpoint (dedicated). *Providers*: Fw.
- Service: Analytics dashboard (web UI). *Providers*: all L1.
- Service: Response headers (`fireworks-prompt-tokens`, `-cached-prompt-tokens`, `-server-time-to-first-token`, `-sampling-options`, `X-Ratelimit-*`). *Providers*: Fw, Neb, Tog.
- Service: `perf_metrics_in_response` (streaming body). *Providers*: Fw.
- Service: `X-BASETEN-MODEL-PREDICTION-ATTEMPTS` (retry count). *Providers*: Bas.

**Module — Runtime Logs**
- Service: Runtime logs tab (real-time, filterable by timestamp/level/content/replica). *Providers*: HF (30-day), Bas, RunP, Tog.
- Service: `tg beta jig logs --follow`. *Providers*: Tog dedicated containers.

### Product L1.I.2 — Billing & Cost Attribution

**Module — Billing Models**
- Service: Per token (input/output/cached). *Providers*: Bas, Fw, HF (routed), Neb, Tog (serverless).
- Service: Per GPU-second. *Providers*: Fw (on-demand).
- Service: Per minute (dedicated). *Providers*: Bas, HF, Neb, Tog.
- Service: Per PTU. *Providers*: Tog ($0.05/PTU/min).
- Service: Per-GPU hourly / reservation. *Providers*: Tog clusters, RunP Pods.
- Service: Per-replica resources. *Providers*: Tog dedicated containers.
- Service: Per successful response (batch 50% off). *Providers*: Fw, Tog.
- Service: Per megapixel (image) / per second of video / per second-char of audio. *Providers*: Tog.
- Service: Per compute time × hardware price/sec. *Providers*: HF Inference.

**Module — Cost Attribution & Controls**
- Service: Cached input discount (automatic, 50% default). *Providers*: Bas, Fw, Tog.
- Service: `X-HF-Bill-To` / `bill_to` (org). *Providers*: HF.
- Service: Project cost analytics (`api_key_id` per-key tracking). *Providers*: Tog.
- Service: Cost centers (team/project/department). *Providers*: RunP, Tog.
- Service: Resource Groups (Enterprise). *Providers*: HF.
- Service: Billing webhooks (per-request token counts, `externalEntityId`, idempotency key, HMAC-signed). *Providers*: Bas Frontier Gateway.
- Service: Spending tiers → higher rate-limit upper bounds. *Providers*: Fw, Tog.
- Service: Spend limit (default $80/hr). *Providers*: RunP.
- Service: Prepaid credits / automatic recharge. *Providers*: HF, RunP.
- Service: Savings plans (3/6 month upfront). *Providers*: RunP Pods.

## Domain L1.J — Reliability, Security, Compliance & Sandboxes

### Product L1.J.1 — Reliability & Retries

**Module — Retry Mechanisms**
- Service: Internal routing retries (502/503/504, exponential backoff, 500ms start ×1.5 cap 60s, 15min total). *Providers*: Bas.
- Service: Circuit breaker (disable retries under memory pressure). *Providers*: Bas.
- Service: Client-side retry (exp backoff on 429/503). *Providers*: all.
- Service: Hedge requests (duplicate after delay to reduce p99). *Providers*: Bas Performance Client.
- Service: Async retry config (`inference_retry_config`: max_attempts/initial_delay/max_delay). *Providers*: Bas.
- Service: Webhook delivery retries (2 attempts ~2s backoff 10s timeout). *Providers*: Bas async.
- Service: Billing webhook retries (exponential 1s→5s, 15s max, dead-letter queue). *Providers*: Bas Frontier Gateway.
- Service: Parking (request waits for a replica). *Providers*: Bas.
- Service: Load shedding (429 when queued payloads exceed memory). *Providers*: Bas.
- Service: Request cancellation (client disconnect → cancel in-flight, 499). *Providers*: Bas, RunP, Tog.
- Service: Rolling deployment rollback (cancel → traffic returns to previous). *Providers*: Bas.
- Service: Reserved capacity (guarantee during scale-up). *Providers*: Fw, Neb.
- Service: Reservations (guarantee for clusters). *Providers*: Tog.

### Product L1.J.2 — Security & Compliance

**Module — Network & Data Security**
- Service: TLS/SSL in transit. *Providers*: all L1.
- Service: PrivateLink / VPC (private IP intra-region). *Providers*: HF (AWS PrivateLink), Tog (clusters).
- Service: Global private networking (Pod-to-Pod). *Providers*: RunP.
- Service: Non-root containers. *Providers*: HF.
- Service: Model security (private repos, malware/pickle scans). *Providers*: HF.
- Service: Data privacy (no payload storage; logs 30 days). *Providers*: HF, Bas.
- Service: Data residency (fixed region, immutable). *Providers*: Neb dedicated.
- Service: Container isolation (Secure Cloud T3/T4). *Providers*: RunP.
- Service: Project-nested observability permissions. *Providers*: Neb.

**Module — Compliance Certifications**
- Service: SOC 2. *Providers*: Bas, Fw, HF, Tog.
- Service: HIPAA. *Providers*: Fw, HF.
- Service: GDPR / DPA / SCC. *Providers*: HF, RunP.
- Service: Enterprise SLA. *Providers*: Neb (Enterprise), Tog (Provisioned 99%).
- Service: SSO/SCIM (Enterprise identity federation). *Providers*: Bas.

### Product L1.J.3 — Sandboxes & Code Execution (GPU platform)

**Module — Nebius Sandboxes**
- Service: `POST /v1/sandboxes` — create sandbox (Git-like branching, OCI images). *Providers*: Neb.
- Service: `POST /v1/sandboxes/{id}/executions` — run code (async operation). *Providers*: Neb.
- Service: `POST /v1/sandboxes/{id}/checkpoints/{cid}/branch` — fork. *Providers*: Neb.
- Service: `POST /v1/sandboxes/{id}/checkpoints/{cid}/rollback` — rollback. *Providers*: Neb.
- Service: `GET /v1/sandboxes/{id}/operations` — poll async ops. *Providers*: Neb.
- Service: Contree SDK / CLI / MCP. *Providers*: Neb.
- Service: SWE-agent preloaded environments. *Providers*: Neb.
- Service: Code interpreter (built-in tool in Responses API). *Providers*: Neb.
- Service: Local shell / apply-patch tools (Responses). *Providers*: Neb.

---
# LAYER 2 — MODEL INFERENCE & INTELLIGENCE APIs

> **Purpose.** The provider-facing intelligence primitives: model catalog, generation endpoints (legacy + modern), reasoning control, structured output, tool-calling primitive, streaming, context management, multimodal input, embeddings/rerank, files, batch, grounding/citations. These are the building blocks L3 modality products and L4 agents call. *Depends on*: L1 (deployments serve these endpoints). *Supervised by*: L5 (auth, billing, moderation, tracing).
> **Sources:** `text/summary.md`, `tools/summary.md`, `images/summary.md` (vision input), `voice/summary.md` (audio parts), `gpu/summary.md` (OpenAI-compat surface).

## Domain L2.A — Model Catalog & Selection

### Product L2.A.1 — Model Listing & Metadata

**Module — Catalog API**
- Service: `GET /v1/models` — list models. *Providers*: OAI, Ant, Goo, Mst, Grok, OR (400+), all L1.
- Service: `GET /v1/models/{id}` — single model with all variants/providers. *Providers*: OR, HF.
- Service: Model catalog with standardized metadata (`id`, `context_length`, `architecture` modalities/tokenizer, `pricing`, `supported_parameters`, `reasoning` efforts/mandatory, `benchmarks`, `expiration_date`). *Providers*: OR.
- Service: Query filters (`output_modalities`, `supported_parameters`, `sort` by pricing/context/throughput/latency/popularity/newest). *Providers*: OR.
- Service: `GET /api/v1/model/{author}/{slug}` — resolve aliases. *Providers*: OR.
- Service: Model lifecycle stages (Experimental/Preview/Stable). *Providers*: Goo.
- Service: Dated snapshots vs rolling aliases (`-latest`, `~openai/gpt-latest`, `mistral-large-latest`, `modelVersion:"latest"`). *Providers*: OAI, Mst, Az.

**Module — Model Variants (OpenRouter)**
- Service: `:nitro` (fastest), `:floor` (cheapest), `:thinking` (extended reasoning), `:free`, `:extended` (larger context), `:exacto` (best tool-calling), `:online` (deprecated). *Providers*: OR.

### Product L2.A.2 — Model Selection & Routing

**Module — Per-Agent / Per-Run Model Selection**
- Service: `model` parameter / URL path selection. *Providers*: all.
- Service: Run-level / turn-level model override (`RunConfig(model)`, `turn/start model`, `agent_with_overrides.model`, `conversations.start(model)`). *Providers*: OAI, Grok (Codex), Ant, Mst.
- Service: Process-wide fallback (`OPENAI_DEFAULT_MODEL`, `fallback_model`). *Providers*: OAI, Ant (Claude).
- Service: Model policies / governance (`/v1/model_policy`). *Providers*: IBM watsonx.
- Service: Provider capability bounds (`modelProvider/capabilities/read`). *Providers*: Grok (Codex).

**Module — Multi-Provider Routing & Fallbacks**
- Service: `models[]` fallback array (context-length/moderation/rate-limit/downtime triggers). *Providers*: OR, Ant (`fallbacks` safety refusal up to 3).
- Service: Router slugs (`openrouter/auto`, `openrouter/free`, `openrouter/fusion`, Pareto Router). *Providers*: OR.
- Service: Provider-level routing (`provider.order`, `allow_fallbacks`, `require_parameters`, `data_collection:deny`, `zdr:true`, `sort:price/throughput/latency`, `preferred_min_throughput`, `preferred_max_latency`, `max_price`, `quantizations`). *Providers*: OR.
- Service: Session stickiness (pin resolved model+provider per conversation). *Providers*: OR.
- Service: Multi-agent model (`grok-4.20-multi-agent`, reasoning.effort controls agent count 4/16). *Providers*: Grok.
- Service: Gateway passthrough (`/v1/gateway/model/chat/completions`, `/embeddings`, OpenAI-compatible). *Providers*: IBM watsonx.
- Service: Model rerouting / safety buffering (`model/rerouted`, `model/safetyBuffering/updated`). *Providers*: Grok (Codex).

## Domain L2.B — Generation API Surfaces

### Product L2.B.1 — Modern / Recommended APIs

**Module — Responses / Interactions / Messages**
- Service: `POST /v1/responses` (typed `output[]` Items, `instructions`, `previous_response_id`, reasoning items, encrypted content). *Providers*: OAI, Grok.
- Service: `interactions.create` (typed `steps[]`, `system_instruction`, `previous_interaction_id`, thought steps with signatures). *Providers*: Goo.
- Service: `POST /v1/messages` (`content[]` typed blocks, top-level `system`, thinking blocks with signatures). *Providers*: Ant.
- Service: `POST /v1/chat/completions` (`choices[].message`, primary for Mst/OR). *Providers*: Mst, OR.

**Module — Legacy / Low-Level APIs**
- Service: `POST /v1/chat/completions` (`messages[]`, `choices[].message`). *Providers*: OAI (legacy), Grok (deprecated), OR (primary).
- Service: `models.generate_content` (`candidates[].content.parts[]`). *Providers*: Goo (legacy).
- Service: `:analyze-text` / `jobs` endpoints with `kind` discriminator. *Providers*: Az.

### Product L2.B.2 — System Instructions & Configuration

**Module — System Prompt**
- Service: `instructions` (top-level, OAI Responses / Grok Responses). *Providers*: OAI, Grok.
- Service: `system_instruction` (top-level, Goo). *Providers*: Goo.
- Service: `system` (top-level param string or text blocks, Ant). *Providers*: Ant.
- Service: `system` role first message (Mst). *Providers*: Mst.
- Service: `system` / `developer` role in `messages[]` (OAI Chat, OR). *Providers*: OAI, OR.
- Service: Mid-conversation system messages (Opus 4.8, append `system` mid-`messages[]` preserving cache). *Providers*: Ant.

**Module — Generation / Sampling Parameters**
- Service: `temperature`, `top_p`, `top_k`, `min_p`. *Providers*: per platform (Goo recommends defaults for Gemini 3.x; Ant deprecates post-Opus 4.6).
- Service: `max_tokens` / `max_output_tokens` / `max_completion_tokens`. *Providers*: all (required on Ant).
- Service: `stop` / `stop_sequences` (up to 4; not reasoning models). *Providers*: per platform.
- Service: `frequency_penalty`, `presence_penalty`, `repetition_penalty`. *Providers*: OAI, Grok, OR, Tog.
- Service: `seed` / `random_seed` (determinism). *Providers*: Mst, Grok, OR, L1 platforms.
- Service: `n` (multiple candidates). *Providers*: OAI Chat, Grok Chat, Fw, HF, Neb, RunP, Tog.
- Service: `logprobs`, `top_logprobs`. *Providers*: OAI, Grok, OR, L1 platforms.
- Service: `logit_bias`. *Providers*: OR, L1 (limited).
- Service: `verbosity` (low/medium/high/xhigh/max). *Providers*: OR.
- Service: `prediction` (predicted output, accepted/rejected tokens). *Providers*: OAI, OR.
- Service: Output Constraining / Prefill (assistant message last entry; deprecated on Ant newer models, supported on Mst/OR). *Providers*: Ant (deprecated), Mst (`prefix:true`), OR.

## Domain L2.C — Reasoning / Thinking Configuration

### Product L2.C.1 — Reasoning Modes & Effort

**Module — Effort-Based Reasoning**
- Service: `reasoning.effort` (low/medium/high/minimal/none). *Providers*: OAI, Grok, OR.
- Service: `thinking_level` (minimal/low/medium/high). *Providers*: Goo.
- Service: `output_config.effort` (low/medium/high/xhigh/max) + `thinking` (enabled/adaptive/disabled). *Providers*: Ant.
- Service: `reasoning_effort` (high/none). *Providers*: Mst.
- Service: `reasoning` object (effort or max_tokens). *Providers*: OR.
- Service: `reasoning_effort` (minimal/none/low/medium/high/max/xhigh/ultra). *Providers*: Grok (Codex).

**Module — Budget-Based & Adaptive**
- Service: `thinking.budget_tokens` (manual mode, Ant legacy). *Providers*: Ant, OR (`reasoning.max_tokens`).
- Service: `output_config.task_budget` (loop-level advisory total-work budget, min 20000). *Providers*: Ant (beta).
- Service: Adaptive thinking (`thinking:{type:"adaptive"}`, model decides). *Providers*: Ant.
- Service: Mandatory (cannot disable): Grok `grok-4.5`, Ant Fable 5/Mythos 5, OR mandatory models.

**Module — Reasoning Content in Output**
- Service: `reasoning` Items in `output[]` with `summary` (auto/concise/detailed). *Providers*: OAI.
- Service: `thought` steps in `steps[]` with `signature` + optional `summary` (text/image). *Providers*: Goo.
- Service: `thinking` blocks in `content[]` (text + `signature`, `redacted_thinking` encrypted opaque). *Providers*: Ant.
- Service: `ThinkChunk` in content chunk list. *Providers*: Mst.
- Service: `reasoning` items with `summary` / `encrypted_content`; `reasoning_content` in Chat. *Providers*: Grok.
- Service: `reasoning` field (string) + `reasoning_details` array (summary/encrypted/text types). *Providers*: OR.

**Module — Thought Signatures & Multi-Turn Continuity**
- Service: Stateful (server manages via `previous_response_id`/`previous_interaction_id`). *Providers*: OAI, Grok, Goo.
- Service: Stateless (resend all thought/thinking blocks exactly as received). *Providers*: Goo, Ant, OR.
- Service: Encrypted reasoning replay (`include:["reasoning.encrypted_content"]`). *Providers*: OAI, Grok, Ant, OR.
- Service: Reasoning context mode (`reasoning.context`: auto/all_turns/current_turn, GPT-5.6+). *Providers*: OAI.
- Service: Reasoning mode `reasoning.mode:"pro"` (deeper multi-pass, OpenRouter). *Providers*: OR.

**Module — Fast Mode**
- Service: `speed:"fast"` + beta header (up to 2.5× output tokens/sec on Opus 4.8/4.7; same weights; invalidates cache; not Batch/Priority). *Providers*: Ant.

**Module — Reasoning Token Billing**
- Service: Reasoning tokens counted/billed as output tokens, count against `max_output_tokens` and context window. *Providers*: all.

## Domain L2.D — Structured Outputs

### Product L2.D.1 — JSON Schema Enforcement

**Module — Schema-Enforced Structured Output**
- Service: `text.format` (Responses) / `response_format` (Chat) with `json_schema` + `strict:true`. *Providers*: OAI, Grok, OR.
- Service: `response_format` with `schema` (no strict flag). *Providers*: Goo.
- Service: `output_config.format` with `json_schema` (grammar compilation + 24h caching). *Providers*: Ant.
- Service: `client.chat.parse()` with Pydantic/Zod/JSON Schema. *Providers*: Mst, OAI (`client.responses.parse()`), Ant (`client.messages.parse()`), L1 platforms.
- Service: Strict mode (`additionalProperties:false`, all fields required, optional via union null). *Providers*: OAI, Ant (tools), Mst, Grok, OR.

**Module — JSON Mode (No Schema)**
- Service: `json_object` type. *Providers*: OAI, Mst, Grok, OR.
- Service: `response_mime_type:"application/json"` (without schema). *Providers*: Goo.

**Module — Grammar / CFG Constraints**
- Service: Custom tools with `format:{type:"grammar",syntax:"lark"|"regex",definition}`. *Providers*: OAI.
- Service: Constrained decoding (compiled grammars, 24h cache). *Providers*: Ant.
- Service: `response_format:{type:"grammar"}` (from OpenAPI). *Providers*: OR.
- Service: `grammar:{type:"json"|"regex"|"json_schema",value}`. *Providers*: HF.
- Service: `response_format:{type:regex,pattern}`. *Providers*: Tog.

**Module — Refusals in Structured Outputs**
- Service: `choices[0].message.refusal` (Chat). *Providers*: OAI.
- Service: Message content item `type==="refusal"` (Responses). *Providers*: OAI.
- Service: `stop_reason:"refusal"` + `stop_details`, output may not match schema. *Providers*: Ant.

**Module — Schema Complexity Limits**
- Service: Max object properties (OAI 5000, xAI 64), nesting 10, enum 1000, strict tools 20 (Ant), optional params 24 (Ant), union 16 (Ant), compilation timeout 180s (Ant). *Providers*: per platform.

**Module — Structured Outputs + Tools**
- Service: Combine structured outputs with built-in tools + function calling in same request. *Providers*: Goo (Gemini 3), Grok (4).

## Domain L2.E — Function Calling & Tool Configuration (primitive)

### Product L2.E.1 — Tool Declarations

**Module — Function Tool Definition**
- Service: Modern flat `{type:"function",name,description,parameters,strict}`. *Providers*: OAI Responses, Goo, Grok Responses.
- Service: `{type:"function",name,description,input_schema,strict}`. *Providers*: Ant.
- Service: Legacy externally tagged `{type:"function",function:{name,description,parameters,strict}}`. *Providers*: OAI Chat, Mst, Grok Chat, OR.
- Service: Custom tools (free-form text input/output, no JSON Schema). *Providers*: OAI.
- Service: `input_examples` (validates against schema, ~20-50 tokens/example). *Providers*: Ant.
- Service: `strict:true` (guaranteed schema conformance; not with recursive `$ref`). *Providers*: Ant, OAI.
- Service: `cache_control` ephemeral on tools (cannot combine with `defer_loading`). *Providers*: Ant.
- Service: `defer_loading` (withhold schema until tool search loads). *Providers*: Ant, OAI.
- Service: `allowed_callers` (`direct`/`code_execution`/`programmatic`, not a security boundary). *Providers*: Ant, OAI.
- Service: `eager_input_streaming` (fine-grained tool-input streaming, user-defined only). *Providers*: Ant.
- Service: `output_schema` (programmatic; describes JSON in function_call_output). *Providers*: OAI.
- Service: `requires_confirmation` (tool names needing approval). *Providers*: Mst.

**Module — Tool Namespaces**
- Service: `{type:"namespace",name,description,tools:[...]}` with `defer_loading` on inner functions; keep <10 functions/namespace. *Providers*: OAI.

### Product L2.E.2 — Tool Choice & Routing

**Module — tool_choice**
- Service: `auto` (model decides). *Providers*: all.
- Service: `required` / `any` (must call some tool). *Providers*: OAI (`required`), Goo/Mst/Ant (`any`), Grok (`required`).
- Service: `none` (must not call). *Providers*: all.
- Service: Specific function `{type:"function",name}` / `{type:"tool",name}` / `{allowed_tools:{mode:"any",tools:[]}}`. *Providers*: per platform.
- Service: Subset `{type:"allowed_tools",mode:"auto",tools:[]}`. *Providers*: OAI, Goo.
- Service: `validated` (schema-validated, preview). *Providers*: Goo.
- Service: Graduated eagerness (minimal/low/medium/high/xhigh). *Providers*: Neb.
- Service: Force hosted tool typed `tool_choice` (e.g. `{type:"image_generation"}`). *Providers*: OAI.
- Service: `max_tool_calls` (Responses). *Providers*: Fw, Neb.

**Module — Parallel & Compositional**
- Service: `parallel_tool_calls:true` (default). *Providers*: OAI.
- Service: Parallel + compositional (chained calls, result informs next). *Providers*: Goo.
- Service: `disable_parallel_tool_use` option. *Providers*: Ant.

**Module — Multimodal Function Responses**
- Service: Function responses include multimodal content (images) in `result`. *Providers*: Goo (Gemini 3).

## Domain L2.F — Streaming

### Product L2.F.1 — Streaming Formats

**Module — Delta Chunks (Chat-style)**
- Service: `stream:true` SSE with `delta.content`, `delta.tool_calls`, `delta.reasoning_content`, `delta.reasoning_details`. *Providers*: OAI Chat, Grok Chat, OR Chat, Mst.

**Module — Typed SSE Events (Responses/Interactions-style)**
- Service: `response.created`, `response.output_text.delta`, `response.completed`, `response.function_call_arguments.delta`. *Providers*: OAI Responses.
- Service: `interaction.created`, `step.start`, `step.delta` (text/thought_summary/thought_signature/arguments), `step.stop`, `interaction.completed`, `done`. *Providers*: Goo.
- Service: `response.output_text.delta`, `response.reasoning_text.delta`, `response.reasoning_summary_text.delta`. *Providers*: Grok Responses.
- Service: `response.created`, `response.content_part.delta`, `response.reasoning.delta`, `response.done`. *Providers*: OR Responses.

**Module — Content Block Events (Anthropic)**
- Service: `message_start` → `content_block_start` → `content_block_delta` (text_delta/thinking_delta/signature_delta/citations_delta/input_json_delta/compaction_delta) → `content_block_stop` → `message_delta` → `message_stop` → `ping` → `error`. *Providers*: Ant.

**Module — Streaming with Reasoning**
- Service: `thought_summary` deltas + `thought_signature` delta. *Providers*: Goo.
- Service: `thinking_delta` events; `signature_delta` before `content_block_stop`; `display:"omitted"` suppresses. *Providers*: Ant.
- Service: Phase 1 (ThinkChunk) → transition → Phase 3 (plain string). *Providers*: Mst.
- Service: `delta.reasoning_details`; `response.reasoning.delta`. *Providers*: OR.

**Module — Streaming Structured Outputs**
- Service: Streamed chunks are valid partial JSON strings; concatenate to form final; parse after stream completes. *Providers*: all.

**Module — SSE Comments / Keep-Alive / Cancellation / Errors**
- Service: SSE comments (`: OPENROUTER PROCESSING`) to prevent timeouts. *Providers*: OR.
- Service: Stream cancellation (abort connection; supported providers stop immediately; unsupported continue+billed). *Providers*: OR.
- Service: Error before tokens → JSON error; after tokens → SSE error event + `finish_reason:"error"`, HTTP stays 200. *Providers*: OR.

## Domain L2.G — Context Management (Caching, Compaction, Editing)

### Product L2.G.1 — Prompt Caching

**Module — Automatic Caching**
- Service: Automatic routing to cached prefix servers. *Providers*: OAI.
- Service: Automatic (single `cache_control` top-level, system manages breakpoints). *Providers*: Ant.
- Service: Implicit caching (Gemini 2.5+). *Providers*: Goo.
- Service: Provider sticky routing + `cache_control`. *Providers*: OR.
- Service: Automatic (`cached_tokens`, `prompt_cache_key` sticky routing). *Providers*: Grok.

**Module — Explicit Cache Breakpoints**
- Service: `cache_control` on individual content blocks (max 4 breakpoints). *Providers*: Ant.
- Service: `prompt_cache_breakpoint` (GPT-5.6+). *Providers*: OAI.
- Service: Context caching (cache files, pay per hour). *Providers*: Goo.

**Module — Cache Retention & Pricing**
- Service: 5-min default TTL. *Providers*: Ant, OR (Anthropic), Goo.
- Service: 1-hour TTL (`cache_control:{type:ephemeral,ttl:"1h"}`, `ENABLE_PROMPT_CACHING_1H=1`). *Providers*: Ant.
- Service: In-memory (5-10 min) vs Extended `24h` (GPU-local storage, `prompt_cache_retention`). *Providers*: OAI.
- Service: Cache write cost 1.25× (5m) / 2× (1h) (Ant); free (OAI automatic). *Providers*: Ant, OAI.
- Service: Cache read cost 0.1× (Ant) / 0.25-0.50× (OAI) / 0.25× (Goo, Grok) / 0.1× (DeepSeek via OR). *Providers*: per platform.
- Service: Cache diagnostics (`cache_miss_reason`, beta, ZDR-eligible). *Providers*: Ant.

**Module — Cache Invalidation**
- Service: Byte-for-byte prefix identity required; reordered tool/interpolated timestamp/edited earlier message silently invalidates. *Providers*: Ant.
- Service: Tool definitions change → invalidates all levels; tool_choice change → invalidates messages; images/thinking param change → invalidates messages. *Providers*: Ant.
- Service: Cache hierarchy: tools → system → messages; longer TTL must appear before shorter. *Providers*: Ant.

### Product L2.G.2 — Context Compaction

**Module — Server-Side Compaction**
- Service: `context_management.edits` with `compact_20260112` (`trigger:{type:"input_tokens",value:150000}`, min 50000); emits `compaction` block; `pause_after_compaction:true` → `stop_reason:"compaction"`; `usage.iterations` records compaction + message iterations. *Providers*: Ant.
- Service: `context_management` with `compact_threshold` on `/responses`. *Providers*: OAI.
- Service: Standalone `POST /v1/responses/compact` (stateless, ZDR-friendly). *Providers*: OAI.
- Service: `POST /v1/responses/compact` (returns opaque compaction item with `encrypted_content`; pass verbatim into next request; at most one per call; conversation must already fit). *Providers*: Grok.
- Service: Native compaction at ~135k tokens. *Providers*: Goo.
- Service: Configurable threshold + sliding window (`compaction_settings.context_compaction_threshold`/`compaction_sliding_window`/`large_message_threshold`). *Providers*: IBM watsonx.
- Service: Agent-overridable compaction prompt. *Providers*: Vibe Code.

### Product L2.G.3 — Context Editing

**Module — Tool Result Clearing (Anthropic)**
- Service: `clear_tool_uses_20250919` with `trigger` (input_tokens or tool_uses, default 100000), `keep` (recent pairs, default 3), `clear_at_least`, `exclude_tools`, `clear_tool_inputs`. *Providers*: Ant.

**Module — Thinking Block Clearing (Anthropic)**
- Service: `clear_thinking_20251015` with `keep` (recent thinking_turns or "all"); must be listed first in `edits` array; invalidates cache when cleared. *Providers*: Ant.

### Product L2.G.4 — Token Counting & Predicted Outputs

**Module — Token Counting**
- Service: `POST /v1/messages/count_tokens` (same params as Messages; supports `context_management`; returns `input_tokens` after editing + `original_input_tokens`; free but RPM-limited). *Providers*: Ant.
- Service: `POST /v1/responses/input_tokens` (model-exact, handles images/files/tools). *Providers*: OAI.
- Service: `GenerativeModel.count_tokens` (not billed). *Providers*: Goo.
- Service: Tokenizer tool (tiktoken). *Providers*: OAI (local).

**Module — Predicted Outputs**
- Service: `prediction:{type:"content",content}` to speed up generation when most output known; `accepted_prediction_tokens`/`rejected_prediction_tokens` usage. *Providers*: OAI.

### Product L2.G.5 — Response Caching (OpenRouter)

**Module — Whole-Response Caching**
- Service: `X-OpenRouter-Cache:true` header (cache identical API responses at gateway; hits free); `X-OpenRouter-Cache-TTL` (1-86400s); `X-OpenRouter-Cache-Clear:true`; no request coalescing. *Providers*: OR.

## Domain L2.H — Multimodal Input

### Product L2.H.1 — Images

**Module — Image Input**
- Service: `input_image` (Responses) / image content (Chat). *Providers*: OAI, Grok.
- Service: Image parts (`inline_data` base64 / `file_data` URI / URL). *Providers*: Goo.
- Service: `image` content blocks (base64/URL/file_id). *Providers*: Ant.
- Service: `image_url` content parts (URL/base64). *Providers*: Mst, OR.
- Service: `input_image` parts. *Providers*: Grok.

**Module — Image Detail / Resolution Control**
- Service: `detail: low` (512×512 fast) / `high` / `original` / `auto`. *Providers*: OAI.
- Service: Resolution tiers (high-resolution max long edge 2576px / 4784 tokens; standard 1568px / 1568 tokens; 28×28 patches). *Providers*: Ant.
- Service: `media_resolution` (max tokens per image/video frame). *Providers*: Goo.

**Module — Max Images**
- Service: 1500 images/request (OAI), 100 (Ant 200k) / 600 (other), 3600 (Goo), up to 14 refs (Goo 3.1 Flash). *Providers*: per platform.

### Product L2.H.2 — PDF & Documents

**Module — PDF/Document Input**
- Service: `input_file` (Responses) / `file` (Chat) — PDF/DOCX/PPTX/TXT/code/spreadsheets; base64/file_id/URL; PDF detail auto/low/high; spreadsheets 1000 rows/sheet augmentation. *Providers*: OAI.
- Service: Document parts (`inline_data`/`file_data`); page images + text. *Providers*: Goo.
- Service: `document` content blocks (base64/URL/file_id); 600 pages (100 for 200k); 32 MB. *Providers*: Ant.
- Service: `input_file` parts (URL/file_id); agentic document search. *Providers*: Grok.

### Product L2.H.3 — Video & Audio (Google)

**Module — Video Input**
- Service: Video parts (`inline_data`/`file_data`); video Q&A, memory, real-time transcription/translation. *Providers*: Goo.

**Module — Audio Input**
- Service: Audio parts (native audio understanding, no separate STT). *Providers*: Goo.

## Domain L2.I — Files API (shared resource)

### Product L2.I.1 — File Upload & Management

**Module — Files API**
- Service: `POST /v1/files` (`purpose: user_data` for OAI; `purpose:"vision"` for image inputs; `purpose:"batch"` for Mst; `purpose:"assistants"` for vector stores). *Providers*: OAI, Ant (beta), Goo (resumable upload → uri+name, poll PROCESSING→ACTIVE), Grok (bidirectional input+output), Mst (`client.files.upload`).
- Service: `GET /v1/files/{id}` / `GET /v1/files/{id}/content` / `DELETE /v1/files/{id}`. *Providers*: OAI.
- Service: Google raw `File` deleted after 48h; embeddings persist. *Providers*: Goo.
- Service: `client.files.download(file_id)` for tool-generated files. *Providers*: Mst.

## Domain L2.J — Embeddings & Rerank (primitive)

### Product L2.J.1 — Embeddings

**Module — Embedding APIs**
- Service: `POST /v1/embeddings` (`text-embedding-3-small` 1536d, `text-embedding-3-large` 3072d; `dimensions` MRL shortening; `encoding_format` float/base64). *Providers*: OAI, all L1.
- Service: `vo.embed()` / `POST https://api.voyageai.com/v1/embeddings` (`voyage-4-large`/`voyage-4`/`voyage-4-lite`/`voyage-4-nano`; 1024/256/512/2048d). *Providers*: Ant (Voyage AI third-party).
- Service: `client.embeddings.create(model,inputs)` (`mistral-embed`). *Providers*: Mst.
- Service: `POST /api/v1/embeddings` (unified across providers). *Providers*: OR.
- Service: Auto-vectorization on insert (vectorizer modules: OpenAI/Cohere/Google/HF/Ollama/Jina/NVIDIA/Mst/AWS/Voyage/Transformers). *Providers*: Weav.
- Service: Named vectors (multi-model per collection). *Providers*: Weav.
- Service: Multi-vector (ColBERT/ColPali). *Providers*: Weav (v1.30+).
- Service: Multimodal embeddings (text+images unified vector space). *Providers*: Goo (`gemini-embedding-2`).
- Service: Vision (VLM) embeddings over page images. *Providers*: Ligh.
- Service: Whole-document embeddings (`mixedbread-ai/mxbai-wholembed-v3`; required for audio/video). *Providers*: mxb.

**Module — Embedding Input Types**
- Service: `input_type="document"` (indexing) / `input_type="query"` (queries). *Providers*: Voyage AI.
- Service: Contextualized chunk embeddings `contextualized_embed()`. *Providers*: Voyage AI.
- Service: Batch (array of strings); multimodal (image inputs). *Providers*: per platform.

### Product L2.J.2 — Rerankers

**Module — Rerank API**
- Service: `rerank-2.5`, `rerank-2.5-lite` (re-ranking retrieved documents). *Providers*: Voyage AI.
- Service: `POST /v1/rerank` (L1 platforms).
- Service: `Reranker` (Mistral) / `relevance_scoring` (Ligh) / `rerank` (mxb). *Providers*: Mst, Ligh, mxb.

## Domain L2.K — Batch Processing

### Product L2.K.1 — Batch APIs

**Module — Batch Endpoints**
- Service: `POST /v1/batches` (targets various endpoints; JSONL). *Providers*: OAI.
- Service: `POST /v1/messages/batches` (50% discount; 100k requests or 256MB; expire 24h; `.jsonl` via `results_url`; not ZDR). *Providers*: Ant.
- Service: `POST /v1/batch/jobs` (50% discount; 1M requests file / 10k inline; `output_file`+`error_file`; statuses QUEUED/RUNNING/SUCCESS/FAILED/TIMEOUT_EXCEEDED/CANCELLATION_REQUESTED/CANCELLED; supported endpoints: embeddings/chat/fim/moderations/chat-moderations/ocr/classifications/conversations/audio-transcriptions). *Providers*: Mst.
- Service: `deferred:true` (Chat) → poll `GET /v1/chat/deferred-completion/{request_id}` (200 completed / 202 pending). *Providers*: Grok.

**Module — Batch Request Structure**
- Service: `{custom_id, params/body}`; match results by `custom_id` (may not match input order). *Providers*: Ant, Mst, OAI.

**Module — Batch + Prompt Caching**
- Service: Prompt caching stacks with batch discount (Ant); cache hits best-effort 30-98%; consider 1-hour cache for batch. *Providers*: Ant.

## Domain L2.L — Grounding, Citations & RAG (primitive)

### Product L2.L.1 — Document Citations

**Module — Document Citation**
- Service: `citations:{enabled:true}` on each `document` content block; returns `cited_text` with char/page/content_block locations; incompatible with structured outputs; `cited_text` not counted toward output tokens. *Providers*: Ant.

### Product L2.L.2 — Search Result Citations

**Module — Search Result Citation**
- Service: `search_result` content blocks; `search_result_location` (source/title/cited_text/search_result_index/start_block_index/end_block_index); all-or-nothing. *Providers*: Ant.
- Service: `ReferenceChunk` via tool calls (define tool returning references → model emits tool_call → execute → return JSON map → model produces TextChunks + ReferenceChunks with `reference_ids`). *Providers*: Mst.

### Product L2.L.3 — Built-in Web Search Grounding (tool-level)

> The built-in web search *tool* is catalogued in L4 Tools (built-in tools). Here we record the citation/annotation output it produces at the inference layer.

**Module — Web Search Citations**
- Service: `url_citation` annotations (start_index/end_index). *Providers*: Goo, OAI.
- Service: `search_suggestions` widget HTML (must render). *Providers*: Goo.
- Service: `sources` list incl. real-time feeds `oai-sports`/`oai-weather`/`oai-finance`. *Providers*: OAI.
- Service: `tool_reference` chunks. *Providers*: Mst.

---

# LAYER 3 — AI MODALITY PRODUCTS

> **Purpose.** End-user-facing products built on L2 primitives: text & conversation, images & video, voice, documents. Each domain packages L2 generation/embedding/structured-output APIs into shippable modality-specific capabilities. *Depends on*: L2 (and transitively L1). *Supervised by*: L5.
> **Sources:** `text/summary.md`, `images/summary.md`, `voice/summary.md`, `documents/summary.md`.

---

## Domain L3.A — Text & Conversation

### Product L3.A.1 — Text Generation

**Module — Single-Turn Generation**
- Service: Single-turn text generation (prompt → response). *Providers*: all generative LLMs.
- Service: Output structure handling (OpenAI `output[]` typed Items; use `output_text` helper; Goo iterate `steps` for interleaved; Mst `message.content` string or chunk list). *Providers*: per platform.

**Module — Multi-Turn Conversations**
- Service: Multi-turn accumulation (user message + assistant response per turn); use prompt caching / context compaction for long conversations. *Providers*: all.

**Module — Multiple Candidates**
- Service: `n` parameter (multiple completions; billed across all). *Providers*: OAI Chat, Grok Chat. (Not on Responses.)

### Product L3.A.2 — Conversation State Management

**Module — Stateful Server-Side State**
- Service: `previous_response_id` (server rehydrates prior context; only send new turn). *Providers*: OAI Responses, Grok Responses.
- Service: `previous_interaction_id` (server manages all state including thought signatures). *Providers*: Goo Interactions.
- Service: Chat SDK (`client.chats.create()` manages `contents` client-side). *Providers*: Goo generateContent.

**Module — Persistent Conversations API**
- Service: `POST /v1/conversations` (persistent conversation object across sessions/devices). *Providers*: OAI, Mst (beta).
- Service: `previous_response_id` / `conversationId`. *Providers*: OAI.
- Service: Append with new conversation ID (append-only immutable). *Providers*: Mst.

**Module — Manual Replay (Stateless)**
- Service: Replay full message/item/step history each turn. *Providers*: all generative LLMs (stateless default).

**Module — Encrypted Reasoning Replay**
- Service: `store:false` + pass encrypted reasoning blobs back. *Providers*: OAI, Grok, Ant, Goo, OR.

**Module — Mid-conversation System Messages**
- Service: Append `{"role":"system"}` inside `messages[]` at point new instruction becomes relevant; preserves cache; placement rules (follow user/assistant turn, not first, not between tool_use/tool_result). *Providers*: Ant (Opus 4.8).

**Module — CRUD on Stored Responses**
- Service: `GET /v1/responses/{id}` (retrieve). *Providers*: OAI, Grok.
- Service: `DELETE /v1/responses/{id}` (delete). *Providers*: Grok.
- Service: Conversations API read/modify (owner-only by API key creator). *Providers*: Mst.

### Product L3.A.3 — Classical NLP Analysis (Pre-trained Tasks)

**Module — Language Detection**
- Service: `kind:LanguageDetection` (100+ languages, ISO 639-1, script detection, country/region hint, confidence). *Providers*: Az.

**Module — Named Entity Recognition (NER)**
- Service: `kind:EntityRecognition` (Person/Org/Location/DateTime/etc.; category/subcategory; inclusionList/exclusionList; sync + async batch). *Providers*: Az.

**Module — Custom NER**
- Service: `kind:CustomEntityRecognition` (train on labeled data; authoring API + runtime API; requires Blob Storage). *Providers*: Az.

**Module — PII Detection & Redaction**
- Service: Text PII `kind:PiiEntityRecognition` (characterMask/noMask/entityMask/syntheticReplacement; `piiCategories`, `confidenceScoreThreshold`, `disableEntityValidation`, `excludeExtractionData`). *Providers*: Az.
- Service: Conversation PII `kind:ConversationalPIITask` (text/transcript modality; `redactionSource`; `includeAudioRedaction`). *Providers*: Az.
- Service: Document PII (native files PDF/DOCX; blur-based image redaction GA). *Providers*: Az.

**Module — Text Analytics for Health**
- Service: `kind:Healthcare` (biomedical NER + relation extraction + entity linking UMLS + assertion detection + FHIR output; en/de/fr/it/es/pt/he). *Providers*: Az.

**Module — Sentiment Analysis & Opinion Mining**
- Service: `kind:SentimentAnalysis` (document + sentence sentiment positive/neutral/negative/mixed + confidence; aspect-based opinion mining targets+assessments; retiring March 2029). *Providers*: Az.

**Module — Key Phrase Extraction**
- Service: `kind:KeyPhraseExtraction` (main concepts/topics; retiring March 2029). *Providers*: Az.

**Module — Entity Linking**
- Service: `kind:EntityLinking` (disambiguate via Wikipedia; name/url/bingId/dataSource/matches; retiring September 2028). *Providers*: Az.

**Module — Summarization**
- Service: Extractive `kind:ExtractiveSummarization` (rankScore; sentenceCount 1-20; sortby Offset/Rank). *Providers*: Az.
- Service: Abstractive `kind:AbstractiveSummarization` (summaryLength oneSentence/short/medium/long; query-focused with `query` field). *Providers*: Az.
- Service: Conversation `kind:ConversationalSummarizationTask` (issue/resolution/chapterTitle/narrative/recap/action items). *Providers*: Az.
- Service: Document summarization (native files via Blob Storage). *Providers*: Az.

**Module — Custom Text Classification**
- Service: `kind:CustomSingleLabelClassification` / `CustomMultiLabelClassification` (train custom; requires Blob Storage; retiring March 2029). *Providers*: Az.

**Module — Conversation Language Understanding (CLU)**
- Service: `kind:Conversation` (predict intents + extract entities; Standard English-only / Advanced multilingual; `multilingual` flag; `confidenceThreshold`). *Providers*: Az.

**Module — Custom Question Answering (CQA)**
- Service: `query-knowledgebases` (deployed KB of Q&A pairs; imports from URLs/PDFs/FAQs; layered ranking Azure AI Search → NLP re-ranking → confidence; multi-turn follow-up; metadata filtering; active learning; retiring March 2029). *Providers*: Az.
- Service: `query-text` (prebuilt, no project). *Providers*: Az.

**Module — Orchestration Workflow**
- Service: `projectKind:Orchestration` (route utterances to CLU/CQA sub-projects; top-level dispatcher; retiring March 2029). *Providers*: Az.

**Module — NLP via Generative LLMs**
- Service: Classification/NER/PII/Summarization/Sentiment/QA/Intent/Key-phrase via prompting or structured outputs. *Providers*: all generative LLMs.

### Product L3.A.4 — Custom NLP Model Training & Deployment

**Module — Project Lifecycle**
- Service: `POST :import` (schema + labeled data from Blob Storage). *Providers*: Az.
- Service: `POST :train` (modelLabel, trainingConfigVersion, evaluationOptions split percentages). *Providers*: Az.
- Service: Evaluate (train job includes evaluation; view metrics). *Providers*: Az.
- Service: `PUT deployments/{deploymentName}` (trainedModelLabel). *Providers*: Az.
- Service: Swap deployments (test ↔ production). *Providers*: Az.
- Service: Query runtime (projectName + deploymentName). *Providers*: Az.

**Module — Authoring API Split**
- Service: `/language/authoring/analyze-text/projects/{name}/...` (Custom NER, Custom Text Classification). *Providers*: Az.
- Service: `/language/authoring/analyze-conversations/projects/{name}/...` (CLU, Orchestration). *Providers*: Az.
- Service: `/language/authoring/query-knowledgebases/projects/{name}/...` (CQA). *Providers*: Az.

**Module — Deployment Expiry**
- Service: Custom model deployments expire ~18 months after training-config version deployed. *Providers*: Az.

### Product L3.A.5 — Usage, Billing & Token Accounting

**Module — Billing Models**
- Service: Per token (input/output); reasoning tokens billed as output; cache reads discounted. *Providers*: per platform.
- Service: Per transaction / per document (custom features per text record). *Providers*: Az.

**Module — Service Tiers**
- Service: `service_tier: auto` / `standard_only`; Priority Tier. *Providers*: Ant.
- Service: `service_tier: default` / `priority`. *Providers*: Grok.
- Service: `service_tier: auto` / `default` / `flex` / `priority` / `scale`. *Providers*: OR.

**Module — Rate Limits & Concurrency**
- Service: RPM by usage tier (Start 2000, Build 4000, Scale 8000). *Providers*: Ant.
- Service: Failed/fallback attempts not billed. *Providers*: OR.

### Product L3.A.6 — Organization, Admin & Asset Management

**Module — Workspaces (Anthropic)**
- Service: Admin API endpoints (create/list/archive workspaces, members, usage reports). *Providers*: Ant.
- Service: Roles (Workspace User/Limited Developer/Developer/Admin/Billing). *Providers*: Ant.
- Service: Non-deletable Default Workspace; max 100 workspaces/org; API keys scoped; resources scoped (Files, Batches, Skills, prompt caches). *Providers*: Ant.

**Module — Model Routing & Fallbacks (OpenRouter)**
- Service: Model-level routing (`models[]` fallback array). *Providers*: OR.
- Service: Router slugs (`openrouter/auto`, `openrouter/free`, `openrouter/fusion`, Pareto Router). *Providers*: OR.
- Service: Provider-level routing (`provider.order`, `allow_fallbacks`, `require_parameters`, `data_collection:deny`, `zdr:true`, `sort`, performance floors, `max_price`, `quantizations`). *Providers*: OR.
- Service: Session stickiness. *Providers*: OR.

**Module — Plugins (OpenRouter)**
- Service: `web` (deprecated), `file-parser`, `response-healing`, `context-compression`, `moderation`, `web-fetch`, `fusion`, `auto-router`, `pareto-router`. *Providers*: OR.

**Module — Asset Management**
- Service: Videos (list, delete); Files (upload, list, delete). *Providers*: OAI.

### Product L3.A.7 — Privacy, Data Residency & ZDR

**Module — Zero Data Retention**
- Service: ZDR enforced for ZDR orgs (`store:false`). *Providers*: OAI.
- Service: ZDR arrangement for eligible features (Files API and Batches not). *Providers*: Ant.
- Service: `store:false`. *Providers*: Grok.

**Module — Data Residency**
- Service: `inference_geo` ("global"|"us" per-request); workspace `default_inference_geo`/`allowed_inference_geos`. *Providers*: Ant.
- Service: EU data residency (SOC 2/HIPAA/GDPR). *Providers*: Grok.

**Module — Training Opt-Out**
- Service: Not used for training. *Providers*: Grok.
- Service: `provider.data_collection:deny`. *Providers*: OR.
- Service: `provider.zdr:true`. *Providers*: OR.

**Module — Watermarking**
- Service: SynthID invisible watermark. *Providers*: Goo.

---

## Domain L3.B — Images & Video

### Product L3.B.1 — Image Generation

**Module — Text-to-Image Generation**
- Service: `POST /v1/images/generations` (Dedicated Images endpoint). *Providers*: OAI, Grok, BFL (per-model URL), Ideo, Recr.
- Service: Unified Interactions API (understanding + generation; pick model + `response_format`). *Providers*: Goo.
- Service: `image_generation` built-in tool in chat (mainline model hosts tool; conversational multi-turn editing). *Providers*: OAI Responses.
- Service: `POST /v1/image/create` (text + aspect_ratio + version + postprocessing). *Providers*: Reve.
- Service: Layout-aware create (returns image + echoed `layout`). *Providers*: Reve v2.

**Module — Vector Image Generation**
- Service: Vector model variants (`recraftv4_1_vector`, etc.) with vector styles; `/v1/images/generations/vector` rejects raster. *Providers*: Recr.

**Module — Transparent Background Generation**
- Service: `/v1/ideogram-v3/generate-transparent` (die-cut stickers, logos; `upscale_factor:X1/X2/X4`; no FLASH; `aspect_ratio`). *Providers*: Ideo.
- Service: `remove-background` on generated image (Ideo, Recr), Reve postprocessing `remove_background`. *Providers*: Ideo, Recr, Reve.

**Module — Multi-Turn / Conversational Image Generation**
- Service: `previous_response_id` chains turns; prior `image_generation_call` ids in `input`; `action:auto|generate|edit`. *Providers*: OAI Responses.
- Service: `previous_interaction_id` continues conversation; model retains prior image context. *Providers*: Goo Interactions.
- Service: Multi-turn editing loop (feed output as next input). *Providers*: Grok, most providers implicitly.

**Module — Streaming / Partial Images**
- Service: `partial_images` (0-3; 0=final only); each partial adds 100 output-image tokens; events `response.image_generation_call.partial_image` (index + b64). *Providers*: OAI.

**Module — Interleaved Text & Image Output**
- Service: Iterate `steps` → `model_output` → `content[]` handling each text/image block. *Providers*: Goo Nano Banana Pro.

**Module — Determinism & Reproducibility**
- Service: `seed` / `random_seed`. *Providers*: BFL, Ideo, Recr, Goo (Veo, "slightly improves"), OAI.
- Service: Pinned model versions (`reve-create@20250915`, BFL versions). *Providers*: Reve, BFL.
- Service: `test_time_scaling` (1-15, cost scales linearly, >5 rarely helps). *Providers*: Reve.

### Product L3.B.2 — Image Editing & Transformation

**Module — Prompt-Based Editing**
- Service: `POST /v1/images/edits` or `image_generation` tool with `input_image`. *Providers*: OAI.
- Service: Interactions (text+image input). *Providers*: Goo.
- Service: `POST /v1/images/edits`. *Providers*: Grok.
- Service: `POST /v1/image/edit` (`edit_instruction`). *Providers*: Reve.
- Service: `/v1/images/imageToImage` (with `strength`). *Providers*: Recr.
- Service: `/v1/edit` or `/v1/ideogram-v4/remix` (`image_weight`). *Providers*: Ideo.
- Service: BFL same endpoint + `input_image`. *Providers*: BFL.

**Module — Multi-Image Editing (Compositing)**
- Service: Up to 3 source images (mix url/file_id kinds); `aspect_ratio` override. *Providers*: Grok.
- Service: Up to 10 images (files/URLs). *Providers*: Ideo `/v1/edit`.
- Service: Up to 8 reference images via `input_image_1..8`; 9MP combined budget on [pro]. *Providers*: BFL.
- Service: 1-6 reference images with `<img>N</img>` tags. *Providers*: Reve.
- Service: Up to 14 object + 4 character + 3 style references (roles). *Providers*: Goo 3.1 Flash.
- Service: Source image + style refs. *Providers*: Recr.

**Module — Inpainting (Masked Region Regeneration)**
- Service: `/v1/images/edits` with `mask` (alpha) or Responses `input_image_mask.file_id`; prompt-based. *Providers*: OAI.
- Service: `/v1/flux-pro-1.0-fill` (`mask` B/W or alpha; `guidance` 1.5-100). *Providers*: BFL.
- Service: `/v1/ideogram-v3/inpaint` (`mask` B/W, black=edit). *Providers*: Ideo.
- Service: `/v1/images/inpaint` (`mask` grayscale, white=modify; V3/V3 Vector only). *Providers*: Recr.

**Module — Outpainting / Border Extension**
- Service: Per-side pixel margins (`expand_left/right/top/bottom` 0-4096, mutually exclusive with `size`). *Providers*: Recr.
- Service: BFL FLUX.1 Expand (`top/bottom/left/right` 0-2048). *Providers*: BFL.
- Service: Target size/canvas (`size`, `reference_offset_x/y`, `auto_crop`). *Providers*: Recr, BFL.
- Service: Zoom out (`zoom_out_percentage` 0-100). *Providers*: Recr.
- Service: Reframe (`/v1/ideogram-v3/reframe`, square→target aspect, preserve focal point). *Providers*: Ideo.
- Service: Outpainting mode `high|fast`. *Providers*: BFL.

**Module — Background Operations**
- Service: Remove background (`/v1/remove-background`, `/v1/images/removeBackground`, postprocessing `remove_background`). *Providers*: Ideo, Recr, Reve.
- Service: Replace background auto-detect (`/v1/ideogram-v3/replace-background`, `/v1/images/replaceBackground`). *Providers*: Ideo, Recr.
- Service: Generate background masked (`/v1/images/generateBackground`, mask specifies fill regions). *Providers*: Recr.

**Module — Object Removal / Erase**
- Service: `/v1/flux-tools/erase-v1` (`image`+`mask`, white=remove; `dilate_pixels` 0-25; FLUX.2 Klein 9B). *Providers*: BFL.
- Service: `/v1/images/eraseRegion` (`image`+`mask`, white=erase; content-aware fill; cheapest operation). *Providers*: Recr.
- Service: Object remove/replace within `/app/edit`. *Providers*: DV.

**Module — Deblur**
- Service: `/v1/flux-tools/deblur-v1` (no prompt, no mask; fixed KV BF16 blur-removal LoRA; caller controls input+seed/format/safety). *Providers*: BFL.

**Module — Virtual Try-On (VTO)**
- Service: `/v1/flux-tools/vto-v1` (`prompt`, `person`→`input_image`, `garment`→`input_image_2`; low-latency; FLUX.2 Klein). *Providers*: BFL.

**Module — Restyle / Relight**
- Service: Restyle Image / Relight Image at `/app/edit` (source + style/mood/lighting prompt). *Providers*: DV.

**Module — Remix / Variate / Explore**
- Service: `/v1/image/remix` (1-6 refs + prompt with `<img>N</img>` tags; `aspect_ratio` model-chosen). *Providers*: Reve.
- Service: `/v1/ideogram-v4/remix` or `/v1/ideogram-v3/remix` (`image`+`text_prompt`+`image_weight`; V3 cropped to aspect ratio). *Providers*: Ideo.
- Service: `/v1/images/variateImage` (no prompt; visual content; `size` required; reformat aspect ratios). *Providers*: Recr.
- Service: `/v1/images/explore` (grid of diverse images from prompt; V4/V4.1). *Providers*: Recr.
- Service: `/v1/images/explore/similar` (`source_image_id` from prior explore + `similarity` 1-5). *Providers*: Recr.
- Service: Ultra image remix (`image_prompt` + `image_prompt_strength` 0-1 blend). *Providers*: BFL.

### Product L3.B.3 — Layout-Aware Composition

**Module — Layout Pipeline (Reve v2)**
- Service: `create_layout` (text/refs → `Layout` with regions label/prompt/bbox/region_type). *Providers*: Reve.
- Service: `edit_layout` (layout + text + typed `LayoutCommand`s: add/shift/remove/place/keep/change). *Providers*: Reve.
- Service: `render` (layout → image; echoes layout). *Providers*: Reve.
- Service: `image_to_layout` (image → `Layout`; reverse-engineers structure; only explicit image understanding in Reve). *Providers*: Reve.
- Service: `edit` v2 (image + text → image + echoed layout). *Providers*: Reve.
- Service: `create` v2 (text/refs → image + echoed layout). *Providers*: Reve.

**Module — Structured (JSON) Prompts**
- Service: `V4JsonPrompt` (`high_level_description` + `style_description` (aesthetics/art_style/lighting/medium/photo) + `compositional_deconstruction` (background + ordered elements with type obj/text, desc/text, optional bbox 0-1000); disables magic-prompt). *Providers*: Ideo.
- Service: `Layout` (regions with label/prompt/bbox normalized 0-1, image_index, parent, region_type coarse/medium/fine/text/hand/face; width/height multiples of 32). *Providers*: Reve.
- Service: JSON prompting (subject, background, lighting, style, camera_angle). *Providers*: BFL.
- Service: `text_layout` (array of {text, bbox} 4-point polygons normalized 0-1; V3/V3 Vector only). *Providers*: Recr.

### Product L3.B.4 — Image Understanding (Vision / Analysis)

**Module — Generative Multimodal Vision (chat-based)**
- Service: `input_image` (Responses/Chat). *Providers*: OAI.
- Service: Interactions (input image part). *Providers*: Goo.
- Service: Mainline Grok chat (out of Imagine scope). *Providers*: Grok.

**Module — Classical Fixed-Feature Detection**
- Service: `images:annotate` (batch of `AnnotateImageRequest`s with `features` enum; returns labels+faces+OCR+objects+safe-search simultaneously). *Providers*: GCV.
- Service: `imageanalysis:analyze` (`features` query param: Caption/Tags/Read/Objects/People/SmartCrops/denseCaptions). *Providers*: Az.
- Service: Per-model predictions. *Providers*: Rep.

**Module — Image Classification & Tagging**
- Service: `LABEL_DETECTION` (fixed). *Providers*: GCV.
- Service: `features=Tags` (fixed). *Providers*: Az.
- Service: `<CAPTION>` task (open via prompt). *Providers*: Rep (Florence-2).
- Service: classification mode (text-prompted). *Providers*: Rep (YOLO-World).
- Service: Classification project (user-defined). *Providers*: Az Custom Vision.
- Service: structured output JSON schema (prompt-defined). *Providers*: Goo Gemini.

**Module — Object Detection**
- Service: Open-vocabulary Grounding DINO (`query`, `box_threshold`, `text_threshold`). *Providers*: Rep.
- Service: YOLO-World (`class_names`, `score_thr`, `nms_thr`). *Providers*: Rep.
- Service: OWL-ViT, Florence-2 `<OD>`. *Providers*: Rep.
- Service: `OBJECT_LOCALIZATION` (`NormalizedVertex` 0-1). *Providers*: GCV.
- Service: `Objects` (`boundingBox:{x,y,w,h}` pixels). *Providers*: Az.
- Service: Custom Vision Object Detection project. *Providers*: Az.
- Service: Gemini structured detection (`[ymin,xmin,ymax,xmax]` 0-1000 + `mask` polygon + `label` via JSON schema). *Providers*: Goo.

**Module — Semantic & Instance Segmentation**
- Service: Semantic segmentation (class per pixel; Semantic-Segment-Anything, Mask2Former ADE20k). *Providers*: Rep.
- Service: Instance/promptable segmentation (SAM-2, points/boxes). *Providers*: Rep.
- Service: Grounded segmentation (Grounding DINO + SAM chained). *Providers*: Rep (`schananas/grounded_sam`).
- Service: Gemini segmentation (`box_2d` + `mask` polygon 0-1000 + `label`). *Providers*: Goo.
- Service: Specialized (clothing, hair, salient foreground). *Providers*: Rep.
- Service: Mask encodings: PNG image (SAM-2), COCO RLE (Semantic-Segment-Anything, SAMURAI), polygon (Gemini); quality scores `predicted_iou`, `stability_score`. *Providers*: Rep, Goo.

**Module — Image Captioning & Dense Captioning**
- Service: Caption (`<CAPTION>`/`<DETAILED_CAPTION>`/`<MORE_DETAILED_CAPTION>`). *Providers*: Rep (Florence-2).
- Service: `Caption` (Florence-based, English-only, `gender-neutral-caption`). *Providers*: Az.
- Service: Dense captions (`denseCaptions` up to 10; `<DENSE_REGION_CAPTION>`). *Providers*: Az, Rep.
- Service: Caption-to-phrase grounding (`<CAPTION_TO_PHRASE_GROUNDING>` with `text_input`; Grounded SAM). *Providers*: Rep.
- Service: Structured describe round-trip (`/v1/ideogram-v4/describe` → `V4JsonPrompt` with optional `bbox`; `include_bbox`; reusable as `json_prompt`). *Providers*: Ideo.
- Service: Layout extraction (`/v2/image/image_to_layout` → `Layout` of typed Regions with parent/child hierarchy). *Providers*: Reve.

**Module — OCR**
- Service: `TEXT_DETECTION` (textAnnotations, full string + per-word boxes). *Providers*: GCV.
- Service: `DOCUMENT_TEXT_DETECTION` (hierarchical pages→blocks→paragraphs→words→symbols, `languageHints`). *Providers*: GCV.
- Service: `Read` (synchronous, `blocks[].lines[].words[]` with `boundingPolygon` 4 points + confidence). *Providers*: Az.
- Service: Document Intelligence Read (async). *Providers*: Az.
- Service: `<OCR>` (text string), `<OCR_WITH_REGION>` (quad boxes 8 coords for tilted text). *Providers*: Rep (Florence-2).
- Service: Text layer extraction (`/v1/ideogram-v3/layerize-text` → `base_image_url` + `text_blocks[]` geometry/font/color/alignment/role). *Providers*: Ideo.

**Module — Face & Pose Detection**
- Service: `FACE_DETECTION` (`faceAnnotations[]`: boundingPoly/fdBoundingPoly, landmarks ~30+ types 3D {x,y,z}, rollAngle/panAngle/tiltAngle, detectionConfidence, joy/sorrow/anger/surprise/underExposed/blurred/headwear Likelihood). *Providers*: GCV.
- Service: MediaPipe Face (full facial mesh / pose landmarks). *Providers*: Rep (`chigozienri/mediapipe-face`).
- Service: Faces removed from v4.0 (gated v3.2); dedicated Face API. *Providers*: Az.
- Service: Facial recognition (not supported by Goo; Az via separate Face API). *Providers*: Az.

**Module — Object Tracking in Video**
- Service: Zero-shot tracking (`zsxkib/samurai` SAM 2 + motion-aware memory, COCO RLE per frame keyed by object_id; prompt in first frame point/box). *Providers*: Rep.
- Service: YOLO-World video mode (open-vocab detection+tracking across frames). *Providers*: Rep.
- Service: SAM-2-video (prompt object in one frame, masks across all frames). *Providers*: Rep.

**Module — Specialized Detectors (mostly GCV)**
- Service: `LANDMARK_DETECTION` (name, score, boundingPoly, locations lat/long). *Providers*: GCV.
- Service: `LOGO_DETECTION` (name, score, boundingPoly). *Providers*: GCV.
- Service: `WEB_DETECTION` (webEntities, visuallySimilarImages, fullMatchingImages, partialMatchingImages, pagesWithMatchingImages). *Providers*: GCV.
- Service: `SAFE_SEARCH_DETECTION` (adult/spoof/medical/violence/racy Likelihood). *Providers*: GCV.
- Service: `IMAGE_PROPERTIES` (dominantColors color + pixel fraction + score). *Providers*: GCV.
- Service: `CROP_HINTS` (cropHints boundingPoly). *Providers*: GCV.
- Service: `PRODUCT_SEARCH` (product matches in a product set). *Providers*: GCV.
- Service: NSFW classification (`falcons-ai/nsfw_image_detection` ViT). *Providers*: Rep.
- Service: Az v4.0 removed (Brands, Landmarks, Celebrities, Adult, Color, Image type in v3.2, now other services). *Providers*: Az.

**Module — Multi-Feature Batch Annotation**
- Service: One `images:annotate` call runs batch of features simultaneously; async `files:asyncBatchAnnotate` for up to 2000 files (PDF/TIFF/batch) to Cloud Storage. *Providers*: GCV.
- Service: `features` query param one result block per feature synchronous. *Providers*: Az.

**Module — Confidence / Likelihood Reporting**
- Service: Float 0-1 (`confidence`, `score`, `detectionConfidence`); caller chooses thresholds. *Providers*: Rep, GCV, Az.
- Service: Likelihood enum `UNKNOWN(0)→VERY_LIKELY(5)` for face attributes and safe-search. *Providers*: GCV.
- Service: Threshold parameters (Grounding DINO `box_threshold`/`text_threshold`; YOLO-World `score_thr`/`nms_thr`; SAM-2 `pred_iou_thresh`/`stability_score_thresh`). *Providers*: Rep.

**Module — Custom Vision (Train-your-own)**
- Service: Classification project (train classifier on labeled images). *Providers*: Az (deprecated 2025-2028).
- Service: Object Detection project (regions drawn at training time). *Providers*: Az.
- Service: Export to ONNX/TF/CoreML/Docker. *Providers*: Az.

### Product L3.B.5 — Image Format & Structure Conversion

**Module — Vectorization (Raster → SVG)**
- Service: `/v1/images/vectorize` (deterministic, no model; <10MB; <16MP; max dim <4096px; min dim ≥256px). *Providers*: Recr.

**Module — Text Layer Extraction**
- Service: `/v1/ideogram-v3/layerize-text` (`base_image_url` + `text_blocks[]` role/text/geometry/font_name/font_size/color/alignment/formatting). *Providers*: Ideo.

**Module — Layout Extraction (reverse)**
- Service: `/v2/image/image_to_layout` → `Layout` of typed Regions. *Providers*: Reve.
- Service: `/v1/ideogram-v4/describe` → `V4JsonPrompt` (with optional `bbox`; `include_bbox`). *Providers*: Ideo.

### Product L3.B.6 — Video Generation

**Module — Text-to-Video**
- Service: `POST /v1/videos` (`sora-2`/`sora-2-pro`; 16/20s; 480p/720p/1080p; native audio). *Providers*: OAI.
- Service: `models.generate_videos` (predictLongRunning; Veo 3.1/3.1 Fast/3.1 Lite/Veo 3/3 Fast/Veo 2; 4/6/8s; 720p/1080p/4k; native audio Veo 3.x always on). *Providers*: Goo.
- Service: `POST /v1/videos/generations` (`grok-imagine-video`/`-1.5`; 1-15s; 480p/720p/1080p I2V only on 1.5). *Providers*: Grok.

**Module — Image-to-Video (First Frame)**
- Service: `input_reference` (multipart file or JSON {file_id}/{image_url}; must match `size`). *Providers*: OAI.
- Service: `image` (initial frame); combine with Nano Banana for two-step pipeline. *Providers*: Goo.
- Service: `image` ({url} or {file_id}); `aspect_ratio` defaults to input ratio. *Providers*: Grok.

**Module — Last-Frame Interpolation**
- Service: `image` + `lastFrame`; model generates video transitioning between them (Veo 3.1/3.1 Fast/3/2). *Providers*: Goo.

**Module — Reference-to-Video**
- Service: `reference_images[]` + `<IMAGE_1>`,`<IMAGE_2>`,`<IMAGE_3>` placeholders; `grok-imagine-video` only (1.5 not). *Providers*: Grok.
- Service: `referenceImages[]` (up to 3, `VideoGenerationReferenceImage` with `reference_type:"asset"`; requires `durationSeconds:"8"`; Veo 3.1/3.1 Fast). *Providers*: Goo.

**Module — Character Assets (Reusable Non-Human Subjects)**
- Service: `POST /v1/videos/characters` (upload short MP4 + name); `characters:[{id}]` up to 2 per video + mention name verbatim in prompt; best 2-4s clips 16:9 or 9:16 720p-1080p; not in extensions; human-likeness blocked by default. *Providers*: OAI.

**Module — Native Audio Generation**
- Service: Always on (Sora 2, Veo 3.x, Kling 2.6, Wan 2.6, Seedance 2.0). *Providers*: OAI, Goo, DV-listed.
- Service: Silent only (Veo 2). *Providers*: Goo.
- Service: Prompt audio cues (describe SFX/ambient/dialogue). *Providers*: Goo.

**Module — Video Generation Parameters (union)**
- Service: `prompt`, `model`, `size`/`aspectRatio`, `seconds`/`durationSeconds`/`duration`, `resolution`, `seed` (Veo 3+), `personGeneration` (allow_all/allow_adult/dont_allow), `input_reference`/`image`, `lastFrame`, `referenceImages`/`reference_images`+`<IMAGE_N>`, `characters`, `storage_options`. *Providers*: per platform.

### Product L3.B.7 — Video Editing, Extension & Interpolation

**Module — Video Editing**
- Service: `/v1/videos/edits` (video id/uploaded + prompt; one focused change; model inferred from source for id, set explicitly for upload; uploaded-video editing gated). *Providers*: OAI.
- Service: `/v1/videos/edits` (video {url}/{file_id} + prompt; duration/aspect_ratio/resolution ignored, inherited; resolution capped 720p; duration capped 8.7s). *Providers*: Grok.

**Module — Video Extension**
- Service: `/v1/videos/extensions` (up to 20s per extension; up to 120s 6× max; no characters/image refs; source video + prompt only). *Providers*: OAI.
- Service: `generate_videos` with `video` param (+7s per extension; up to 148s 20× max; 720p only; must be prior Veo generation). *Providers*: Goo Veo 3.1/3/3 Fast.
- Service: `/v1/videos/extensions` (`duration` = extension portion only; total = input + extension; `grok-imagine-video` only). *Providers*: Grok.

**Module — Video-to-Video & Audio-to-Video (Multimodal)**
- Service: `@Image`/`@Video`/`@Audio` tag reference system; modes T2V/I2V/V2V/A2V; seamless scene extensions; structured refinement; auto audio-visual sync. *Providers*: DV / Seedance 2.0.

**Module — Batch Video Generation**
- Service: `POST /v1/batches` targets `/v1/videos` (JSON only, no multipart; upload assets ahead; reference via `input_reference` with `file_id`/`image_url`; `custom_id` map results; batch videos available 24h after batch completes). *Providers*: OAI.

### Product L3.B.8 — Postprocessing & Effects

**Module — Postprocessing Pipeline (Reve)**
- Service: `upscale` (`upscale_factor` 2/3/4; variable cost ≥2 credits). *Providers*: Reve.
- Service: `remove_background` (≥2 credits; transparent output). *Providers*: Reve.
- Service: `fit_image` (`max_dim`/`max_width`/`max_height` 1-4096; **free**; scale-down preserving aspect ratio). *Providers*: Reve.
- Service: `effect` (`effect_name` + optional `effect_parameters`; 3 credits; named preset). *Providers*: Reve.

**Module — Effects System (Reve)**
- Service: `GET /v1/image/effect?source=all|project|preset` (list effects; name/source/description/category). *Providers*: Reve.

**Module — Upscaling**
- Service: `/v1/images/crispUpscale` (crisp, interpolation sharpening, preserves content, min dim ≥32px). *Providers*: Recr.
- Service: `/v1/images/creativeUpscale` (creative, regenerates finer details and faces, min dim ≥256px). *Providers*: Recr.
- Service: `/upscale` (guided, `resemblance`/`detail` controls, optional `prompt`, `magic_prompt_option`). *Providers*: Ideo.
- Service: postprocessing `upscale` (factor 2/3/4). *Providers*: Reve.

**Module — Smart Cropping**
- Service: `CROP_HINTS` (`cropHints[].boundingPoly`). *Providers*: GCV.
- Service: `SmartCrops` (suggested crop regions). *Providers*: Az.

### Product L3.B.9 — Style & Asset Management

**Module — Style System**
- Service: Curated style names (`style` Photorealism/Illustration/Vector art; `style_type` REALISTIC/DESIGN/FICTION). *Providers*: Recr, Ideo.
- Service: Style presets (~60 named: ART_DECO/WATERCOLOR/BAUHAUS). *Providers*: Ideo.
- Service: Style codes (8-char hex, shareable, precise). *Providers*: Ideo.
- Service: Style reference images (transfer aesthetic). *Providers*: Recr, Ideo, Goo.
- Service: Custom style `POST /v1/styles` (from up to 5 reference images; V3/V3 Vector only; reusable UUID). *Providers*: Recr.
- Service: Color palette (`controls.colors` Recr; `color_palette` preset or hex+weight members Ideo). *Providers*: Recr, Ideo.
- Service: Structured style description (aesthetics/art_style/lighting/medium/photo). *Providers*: Ideo.
- Service: Controls (`colors`, `background_color`, `artistic_level` 0-5 V3, `no_text` V3). *Providers*: Recr.
- Service: `text_layout` (individual words as 4-point polygons). *Providers*: Recr.
- Service: LoRA finetune (`finetune_id` + `finetune_strength` 0-2). *Providers*: BFL.
- Service: Custom model (`custom_model_uri` trained 15-100 assets). *Providers*: Ideo.
- Service: Azure Custom Vision (train-your-own classifier/detector, export ONNX/TF/CoreML/Docker). *Providers*: Az.

**Module — Style Exclusivity Rules (Ideogram V3)**
- Service: `style_codes` cannot combine with `style_reference_images` or `style_type`; `color_palette` cannot mix preset and members; `resolution` and `aspect_ratio` mutually exclusive. *Providers*: Ideo.

**Module — Recraft Style Compatibility**
- Service: Styles (`style`/`style_id`) only supported on V2 and V3 models — V4/V4.1 ignore style parameters. *Providers*: Recr.

**Module — Character Reference Management**
- Service: `character_reference_images` (currently 1) + optional `character_reference_images_mask` (grayscale, same dims). *Providers*: Ideo.
- Service: Character reference parts (up to 4-5 depending on model). *Providers*: Goo.
- Service: Reusable `characters` assets (non-human, up to 2 per video, mention name in prompt). *Providers*: OAI Sora.
- Service: Locked character consistency across variations. *Providers*: DV/Kling.

**Module — Prompt Enhancement / Rewriting**
- Service: `revised_prompt` (returned, automatic). *Providers*: OAI.
- Service: Prompt Upsampling `disable_pup` (FLUX.2 pro/max) / `prompt_upsampling` (older, default false; flex default true). *Providers*: BFL.
- Service: Magic Prompt `magic_prompt:AUTO/ON/OFF` (V3) / automatic when `text_prompt` (V4) / disabled when `json_prompt`; dedicated `/v1/ideogram-v4/magic-prompt` (returns `V4JsonPrompt`). *Providers*: Ideo.
- Service: `POST /v1/prompts/enhance` (dedicated, ≤2000 chars, returns `enhanced_prompt`). *Providers*: Recr.
- Service: Auto-enhanced (automatic, silent, not surfaced). *Providers*: Reve.
- Service: Thinking process `generation_config.thinking_level:minimal|high` (default minimal; enabled by default, cannot disable; thought steps in `steps`, thought images not billed). *Providers*: Goo.

**Module — Negative Prompts**
- Service: `negative_prompt` (V3). *Providers*: Ideo, Recr V2/V3. (Not BFL FLUX.)

**Module — Reference-Image Tagging in Prompts**
- Service: `<img>N</img>` (0-indexed; also `<ref>N</ref>` v1). *Providers*: Reve.
- Service: `<IMAGE_1>`,`<IMAGE_2>`,`<IMAGE_3>` placeholders. *Providers*: Grok reference-to-video.
- Service: Describe roles ("person of image 1 wearing garments of image 2"). *Providers*: BFL.
- Service: Mention character name verbatim in prompt. *Providers*: OAI Sora.

**Module — Grounding / Real-Time Information (image gen)**
- Service: `google_search` tool in Interactions API; `search_types:["web_search","image_search"]` (3.1 Flash); must render `search_suggestions` HTML; not Flash Lite. *Providers*: Goo.
- Service: FLUX.2 [max] grounding search (trending products, events, scores, weather). *Providers*: BFL.

### Product L3.B.10 — Files API / Storage (image/video)

**Module — Input Files**
- Service: `POST /v1/files` (`purpose:"vision"` → `file_id`). *Providers*: OAI.
- Service: Resumable upload → `uri`+`name`; poll `state` PROCESSING→ACTIVE; recommended >100MB/reuse. *Providers*: Goo.
- Service: Files API bidirectional; `file_id` substitutes url/base64; outputs via `storage_options`. *Providers*: Grok.
- Service: `input_image_blob_path` (hosted blob, FLUX.2 [flex] only). *Providers*: BFL.
- Service: Project references `id:<uuid>` / `reference:@<name>`. *Providers*: Reve.

**Module — Output Persistence (xAI)**
- Service: `storage_options` (`filename` required, `expires_after` 3600-2592000s or permanent, `public_url` bool or `{expires_after}`); `file_output` (`file_id`, `filename`, `expires_at`, `public_url`, `public_url_expires_at`, `public_url_error`); up to 1000 active public URLs per team. *Providers*: Grok.

**Module — Ephemeral URLs**
- Service: All generated URLs expire (BFL, Ideo, Recr ~24h, OAI video 1h / batch 24h, Grok, Reve). Download promptly or persist via Files API. *Providers*: all.

### Product L3.B.11 — Async Job Lifecycle, Storage & Management

**Module — Async Lifecycle**
- Service: BFL `POST` → `AsyncResponse {id, polling_url, cost, input_mp, output_mp}` or `AsyncWebhookResponse`; poll `GET polling_url` (or `GET /v1/get_result?id=`); states Pending→Reasoning→Generating→Ready (or Request Moderated/Content Moderated/Error/Task not found). *Providers*: BFL.
- Service: OAI Videos `POST /videos` → job `id` (`status:queued`); poll `GET /videos/{id}` 10-20s exponential backoff or register webhook (`video.completed`/`video.failed`); states queued/in_progress/completed/failed; `GET /videos/{id}/content?variant=video|thumbnail|spritesheet` streams binary MP4 (URL valid 1h). *Providers*: OAI.
- Service: Goo Veo `POST models/{model}:predictLongRunning` → `{name:"operations/..."}`; poll `GET /v1beta/{name}` until `done:true`; download `video.uri`. *Providers*: Goo.
- Service: Grok Videos `POST /v1/videos/generations` → `{request_id}`; poll `GET /v1/videos/{request_id}`; states pending/done/expired/failed; SDKs abstract polling (timeout default 10min, interval 100ms). *Providers*: Grok.
- Service: Ideo async `POST /v1/ideogram-v4/async/generate?webhook_url=` (HTTPS required, rejects private/loopback) → `{generation_id}`; webhook POSTs result (Ed25519-signed); fallback `GET /v1/generations/{generation_id}` (status pending/completed/failed). *Providers*: Ideo.
- Service: Replicate `POST /v1/predictions {version, input}` → poll `GET /v1/predictions/{id}` or webhooks; states starting/processing/succeeded/failed/canceled. *Providers*: Rep.
- Service: Az Custom Vision training long-running; Prediction synchronous. *Providers*: Az.

**Module — Webhook Verification & Idempotency**
- Service: Ideo Ed25519 (fetch public keys `GET /v1/.well-known/jwks.json`; signed message = `{Generation-Id}\n{User-Id}\n{Timestamp}\n{sha256_hex(raw body)}`; verify hex signature against each public key; `X-Ideogram-Webhook-Key-Id` hint; handlers idempotent by `generation_id`; delivery not guaranteed). *Providers*: Ideo.
- Service: `webhook_url` + optional `webhook_secret` for signature verification. *Providers*: BFL.

**Module — Library & Asset Management**
- Service: `GET /v1/videos?limit=20&after=&order=asc` (paginate + sort); `DELETE /v1/videos/{video_id}`. *Providers*: OAI.
- Service: `GET/DELETE /v1/files/{id}`, `POST /v1/files/{id}/public-url` (create/recreate), `POST /v1/files/{id}/public-url/revoke`. *Providers*: Grok.
- Service: `GET /v1/my_finetunes`, `GET /v1/finetune_details?finetune_id=`, `POST /v1/delete_finetune`. *Providers*: BFL.
- Service: `GET /models` (`scope:owned|shared`, `status` filter), `GET /models/{model_id}`. *Providers*: Ideo.
- Service: `GET /v1/users/me` (credits, email, id, name). *Providers*: Recr.

### Product L3.B.12 — Output Formatting & Delivery

**Module — Image Output Formats**
- Service: PNG (default), JPEG, WebP, SVG (Recr vector). *Providers*: per platform.
- Service: Transparent (OAI `background:transparent` unsupported on gpt-image-2, Ideo `generate-transparent`, Recr removeBackground). *Providers*: per platform.

**Module — Response Delivery Modes**
- Service: Synchronous (image bytes/URL). *Providers*: OAI Images, Goo Interactions, Grok image, all Recr, all Reve, Ideo sync, BFL (poll), classical vision.
- Service: Streaming (partial images). *Providers*: OAI (`stream` + `partial_images`).
- Service: Asynchronous (job id → poll). *Providers*: OAI Videos, Goo Veo, Grok Videos, BFL (all), Rep, Ideo async, Az Custom Vision training.
- Service: Webhook delivery. *Providers*: OAI Videos, BFL, Ideo (Ed25519-signed), Rep.

**Module — Response Shape**
- Service: Image URL (ephemeral). *Providers*: all.
- Service: Base64 inline (`b64_json` OAI/Grok/Recr; Reve JSON default; BFL base64 or URL). *Providers*: per platform.
- Service: Raw image bytes (Reve via `Accept: image/png|jpeg|webp` headers). *Providers*: Reve.

**Module — Video Output**
- Service: MP4 video (`variant=video` OAI, `video.uri` Goo, `video.url` Grok). *Providers*: per platform.
- Service: Thumbnail (`variant=thumbnail` → `thumbnail.webp`). *Providers*: OAI.
- Service: Spritesheet (`variant=spritesheet` → `spritesheet.jpg`). *Providers*: OAI.
- Service: Persisted file (`storage_options` → `file_output.file_id` + optional `public_url`). *Providers*: Grok.

**Module — Response Headers & Metadata**
- Service: Reve headers (`X-Reve-Content-Violation`, `X-Reve-Request-Id`, `X-Reve-Version`, `X-Reve-Credits-Used`, `X-Reve-Credits-Remaining`, `X-Reve-Error-Code`). *Providers*: Reve.
- Service: OAI moderation error (`error.type:image_generation_user_error`, `error.code:moderation_blocked`, `moderation_details:{moderation_stage:input|output|unknown, categories}`). *Providers*: OAI.
- Service: Grok response object (`response.url`, `response.image` base64, `response.model`, `response.respect_moderation`, `response.file_output`, `response.public_url`). *Providers*: Grok.

### Product L3.B.13 — Moderation & Safety (image/video)

**Module — Safety Tolerance**
- Service: `safety_tolerance` 0-5 (FLUX.2) / 0-6 (FLUX.1), 0=strictest. *Providers*: BFL.
- Service: `moderation:auto|low`. *Providers*: OAI.
- Service: `content_violation` / `X-Reve-Content-Violation`; `respect_moderation`; `is_image_safe`. *Providers*: Reve, Grok, Ideo.
- Service: `moderation_stage:input|output|unknown`, `moderation_details.categories`. *Providers*: OAI.
- Service: `enable_copyright_detection` (Hive likeness + logo checks). *Providers*: Ideo.
- Service: `personGeneration:allow_all|allow_adult|dont_allow` (Veo). *Providers*: Goo.
- Service: `SAFE_SEARCH_DETECTION` (adult/spoof/medical/violence/racy Likelihood). *Providers*: GCV.
- Service: NSFW classification (`falcons-ai/nsfw_image_detection`). *Providers*: Rep.
- Service: SynthID invisible watermark (all generated images and Veo outputs). *Providers*: Goo.

### Product L3.B.14 — Billing Units (image/video)

**Module — Billing**
- Service: Credits (BFL 1 credit=$0.01; Recr 1000 units=$1; Reve 750 credits≈$1; Ideo; DV). *Providers*: per platform.
- Service: Per image (flat or megapixel-based on FLUX.2). *Providers*: most.
- Service: Per output-image tokens (OAI GPT Image driven by quality+size). *Providers*: OAI.
- Service: Per patch/tile (vision input, 32×32 patches or 512px tiles). *Providers*: OAI mainline vision.
- Service: Per second (video, Grok/Veo driven by model/resolution/duration). *Providers*: Grok, Goo.
- Service: Per request (Recr transformations, Reve operations, BFL tools). *Providers*: per platform.
- Service: Surcharges (OAI partial images +100 output tokens each; Reve postprocessing upscale/effect adds cost, fit_image free). *Providers*: OAI, Reve.

---

## Domain L3.C — Voice

### Product L3.C.1 — Voice Asset Management

**Module — Prebuilt Voice Library**
- Service: Voice Library (10,000+ community-shared, 11L); Built-in Voices (13, OAI); Prebuilt Voices (30, Goo); Voice Library (Cart); Aura Voice Catalog (40+, Deep); Flagship Voices (60+, Grad). *Providers*: per platform.
- Service: Voice selection (`voice_id`/`voice` name; filter by language/gender/country; search by name/description/ID `q`; preview audio URLs `expand[]=preview_file_url`). *Providers*: Cart, 11L.
- Service: Find similar voices (`/voices/find-similar`). *Providers*: 11L.

**Module — Instant Voice Cloning (IVC)**
- Service: IVC (<2 min, 11L; ≤30s + consent recording, OAI; ≤10s recommended, Cart free; ≥10s recommended, Grad). *Providers*: 11L, OAI, Cart, Grad.
- Service: OAI consent requirement (two recordings: consent phrase in one of 16 languages + actual audio sample ≤30s; returns `consent_id`). *Providers*: OAI.

**Module — Professional Voice Cloning (PVC)**
- Service: PVC (create → add samples → train → verify captcha; speaker separation on samples; verification via voice-captcha). *Providers*: 11L.
- Service: Pro Voice Clone (`/fine-tunes/create`; 1,000,000 credits per fine-tune; ~1.5 credits/char TTS 50% more than standard; datasets API for file management). *Providers*: Cart.

**Module — Voice Design from Text Description**
- Service: `/text-to-voice/design` (20-1000 chars; returns 3 preview voices; create from preview; remix with attribute transformations gender/accent/speaking style/pacing/audio quality; `prompt_strength` low/medium/high/max). *Providers*: 11L.

**Module — Voice Remixing**
- Service: `/text-to-voice/remix` (transform existing voice attributes via natural language, maintains recognizable characteristics; iteratively remixable; backward-compatible with older models). *Providers*: 11L.

**Module — Voice Localization**
- Service: `/voices/localize` (adapt existing voice to new language/dialect; 20 target languages with dialect options: English Australian/Indian/Southern/British/American; Spanish Latin American/Peninsular; Portuguese Brazilian/European; French Standard/Canadian). *Providers*: Cart.

**Module — Voice Metadata Management**
- Service: List/search voices (pagination, filtering by language/gender/ownership); get details; update (name/description/gender/settings); delete; default voice settings get/set; share to library (make public). *Providers*: 11L.

**Module — Pronunciation Dictionaries**
- Service: Versioned sets of text-to-pronunciation mappings (from-file/from-rules 11L; POST with items Cart; POST with rewrite rules Grad); versioning (`version_id` 11L); attachment (`pronunciation_dictionary_locators` up to 3 11L, `pronunciation_dict_id` Cart, `pronunciation_id` Grad); per-request; access control (private/public Cart); full CRUD management API; item format (phonetic rules 11L, `{text,pronunciation}` Cart, language-specific rewrite rules Grad). *Providers*: 11L, Cart, Grad.

**Module — Datasets & Fine-Tuning (Music)**
- Service: Datasets API for organizing fine-tune training files. *Providers*: Cart.
- Service: Music model fine-tunes on own non-copyrighted tracks (5-10 min, copyright screening, captures instrumentation/tempo/production style); curated finetunes (Afro House, Reggaeton, Arabic Groove). *Providers*: 11L.

### Product L3.C.2 — Audio Input Preprocessing

**Module — Audio Format Detection & Encoding**
- Service: Compressed (MP3/AAC/OGG/OPUS/M4A/WebM/FLAC), Uncompressed (WAV/AIFF/raw PCM 16/32-bit), Telephony (μ-law/A-law/G.729/AMR-narrow/wideband), Video (MP4/AVI/MKV/MOV/WMV/FLV/MPEG/3GPP), Raw PCM (requires `encoding`+`sample_rate`). *Providers*: per platform.

**Module — Sample Rate Configuration**
- Service: 8kHz (telephony), 16kHz (low-latency streaming), 22.05kHz, 24kHz (voice, Gemini Live input), 32kHz, 44.1kHz (CD), 48kHz (professional, Gemini Live output, Grad TTS output). *Providers*: per platform.

**Module — Multichannel Handling**
- Service: Independent channel transcription (each channel separate, `use_multi_channel` up to 5 11L, `multichannel` Deep); output style `separate`/`combined` (11L); channel count in streaming setup (Deep `channels`). *Providers*: 11L, Deep.

**Module — Voice Isolation / Noise Removal (pre-processing)**
- Service: Voice Isolator `/audio-isolation` (buffered + streamed; max 500MB, 1 hour; not optimized for vocal isolation from music). *Providers*: 11L.
- Service: `remove_background_noise` boolean (STS option). *Providers*: 11L.
- Service: Speaker separation (PVC sample; Music stem separation). *Providers*: 11L.

### Product L3.C.3 — Speech-to-Text / Audio Understanding

**Module — Batch Transcription (File-Based)**
- Service: Batch STT (`/speech-to-text` 11L; `/audio/transcriptions` OAI; Audio Understanding Goo Interactions; `/stt` Cart; `/v1/listen` Deep Pre-Recorded; `/post/speech/asr` Grad STT REST). *Providers*: all voice. Max file sizes: 3GB 11L, 25MB OAI, 2GB Deep; max duration 10h 11L, 10min Deep Nova/20min Whisper; URL input (YouTube/TikTok) 11L/Deep; video input 11L/OAI.

**Module — Real-Time Streaming Transcription**
- Service: Realtime STT (WebSocket, 11L); Realtime Transcription (session, OAI); Live Streaming STT (`WSS /v1/listen` Deep); STT WebSocket (Cart, Grad). *Providers*: all voice.
- Service: Interim vs final results; `is_final` vs `speech_final` distinction (Deep); VAD events (`SpeechStarted` Deep, `speech_started`/`speech_stopped` OAI); KeepAlive messages (Deep); manual commit (OAI, Cart manual mode). *Providers*: per platform.

**Module — Language Detection & Hinting**
- Service: Smart language detection with confidence (`language_probability` 11L); language hint ISO 639-1/BCP-47 (all); code-switching/multilingual (`multi` Deep Nova-3/2, `flux-general-multi` Deep Flux 10 languages, `language_hints` array Flux); auto-detect with candidate list (Deep `detect_language` array). *Providers*: per platform.

**Module — Diarization (Speaker Identification)**
- Service: Up to 32 speakers (`diarize` + `diarization_threshold`, `num_speakers` hint, `detect_speaker_roles` agent/customer +10% surcharge, `use_speaker_library` match registered). *Providers*: 11L.
- Service: `gpt-4o-transcribe-diarize` model, `diarized_json` format, `known_speaker_names[]` up to 4, `known_speaker_references[]` 2-10s audio data URLs. *Providers*: OAI.
- Service: Structured output schema with `speaker` field. *Providers*: Goo.
- Service: `diarize_model` (latest/v1/v2), `utterances:true` + `utt_split` pause threshold. *Providers*: Deep.

**Module — Vocabulary Boosting / Keyterm Prompting**
- Service: `keyterms` (up to 1000 batch, 50 realtime; plain terms ≤50 chars; +20% surcharge). *Providers*: 11L.
- Service: `prompt` (Whisper, 224 tokens context). *Providers*: OAI.
- Service: `keyterm` (Nova-3). *Providers*: Deep.
- Service: `keywords` (legacy, `term:boost` 0-100). *Providers*: Deep.
- Service: `search` (search for terms, returns hits). *Providers*: Deep.
- Service: `keyterm` (Flux). *Providers*: Cart.

**Module — Text Normalization & Formatting**
- Service: `smart_format` (currency/phone/email/dates, Deep); `punctuate` (Deep); `paragraphs` (Deep); `numerals` (Deep); `dictation` (Deep); `measurements` (Deep); `profanity_filter` (Deep); `filler_words` (Deep); `no_verbatim` (11L remove filler/false starts); `apply_text_normalization` auto/on/off (11L); `apply_language_text_normalization` Japanese-specific (11L); `rewrite_rules` language-specific (Grad); `search` + `replace` (Deep). *Providers*: per platform.

**Module — Entity Detection & Redaction**
- Service: `entity_detection` (PII/PHI/PCI/offensive/names/orgs/dates, 11L; `detect_entities` Deep NAME/PHONE_NUMBER/EMAIL_ADDRESS/ORGANIZATION/CARDINAL).
- Service: Entity redaction modes `redacted`/`entity_type`/`enumerated_entity_type` (11L; +30% surcharge); `redact` pci/pii/numbers/ssn/aggressive_numbers (Deep). *Providers*: 11L, Deep.

**Module — Timestamps**
- Service: Word-level (all); character-level (`timestamps_granularity:character` 11L); phoneme-level (`add_phoneme_timestamps` Cart); segment-level (`timestamp_granularities:segment` OAI Whisper); MM:SS format (Goo prompt-based). *Providers*: per platform.

**Module — Audio Event Tagging**
- Service: `tag_audio_events` (default true; word types: word/spacing/audio_event; tag laughter/applause/footsteps). *Providers*: 11L.

**Module — Audio Intelligence (Post-Transcription Analysis)**
- Service: `summarize` (true/false/"v2"). *Providers*: 11L, Deep.
- Service: `sentiment` (positive/negative/neutral + score per segment + average). *Providers*: 11L, Deep.
- Service: `topics` (auto-detected + `custom_topic` up to 100, `custom_topic_mode:strict/extended`). *Providers*: 11L.
- Service: `intents` (auto-detected + `custom_intent`, `custom_intent_mode:strict/extended`). *Providers*: 11L.
- Service: `detect_entities` (NAME/PHONE_NUMBER/EMAIL_ADDRESS/ORGANIZATION/CARDINAL). *Providers*: Deep.
- Service: Emotion (happy/sad/angry/neutral per segment via structured output). *Providers*: Goo.
- Service: Deepgram Read API (`/v1/read`) — intelligence features applied to text content. *Providers*: Deep.

**Module — Forced Alignment**
- Service: `/forced-alignment` (audio file + plain string transcript; word/phrase timestamps; not with diarized text; max 3GB/10h/675,000 chars; 29 languages). *Providers*: 11L.

**Module — Turn Detection (VAD) — STT Layer**
- Service: Silence-based VAD (`server_vad` OAI; automatic Goo; `endpointing` Deep v1; configurable threshold/padding/silence duration). *Providers*: OAI, Goo, Deep.
- Service: Semantic VAD (`semantic_vad` OAI; `step` messages Grad; classifier scores probability user done; `eagerness` auto/low/medium/high). *Providers*: OAI, Grad.
- Service: Model-native turn detection (Flux v2 Deep, Ink-2 Cart; emits StartOfTurn/Update/EagerEndOfTurn/TurnResumed/EndOfTurn lifecycle). *Providers*: Deep, Cart.
- Service: Manual/push-to-talk (`turn_detection:null` OAI; manual VAD Goo; `/stt/websocket`+`finalize` Cart; `flush` message Grad). *Providers*: OAI, Goo, Cart, Grad.
- Service: Turn lifecycle events (StartOfTurn/turn.start, Update/turn.update cumulative, EagerEndOfTurn/turn.eager_end moderate confidence start reply early, TurnResumed/turn.resume speech continued false alarm cancel prep, EndOfTurn/turn.end definitive). *Providers*: Deep Flux, Cart Ink-2.
- Service: Gradium semantic VAD `step` messages (every 80ms, `vad` array with horizon predictions `horizon_s`/`inactivity_prob`; client decides threshold to flush). *Providers*: Grad.
- Service: Adaptive delay (`delay_in_frames` 0-80, each frame 80ms, Grad; `delay` minimal/low/medium/high/xhigh OAI). *Providers*: Grad, OAI.
- Service: Turn detection config params (`threshold`, `prefix_padding_ms`, `silence_duration_ms`, `eagerness`, `start_of_speech_sensitivity`/`end_of_speech_sensitivity`, `turn_start_threshold`/`turn_eager_end_threshold`/`turn_end_threshold`/`turn_end_timeout_ms` Cart, `eot_threshold`/`eager_eot_threshold`/`eot_timeout_ms` Deep, `activity_handling` START_OF_ACTIVITY_INTERRUPTS/NO_INTERRUPTION Goo, `turn_coverage` ONLY_ACTIVITY/ALL_INPUT/AUDIO_ACTIVITY_AND_ALL_VIDEO Goo). *Providers*: per platform.

**Module — Export Formats**
- Service: SRT (configurable `max_characters_per_line`/`segment_on_silence_longer_than_s`/`max_segment_duration_s`/`max_segment_chars`/`include_speakers` 11L), TXT, DOCX, HTML, PDF, segmented JSON, verbose_json (OAI), VTT (OAI), diarized_json (OAI), NDJSON (Grad). *Providers*: per platform.

**Module — Webhooks / Async Callbacks**
- Service: `webhook=true` in STT returns early with `request_id`/`transcription_id`, results to configured webhooks; `webhook_id`; `webhook_metadata` (max 16KB). *Providers*: 11L.
- Service: `callback` URL parameter on STT/TTS, `callback_method` POST/PUT. *Providers*: Deep.
- Service: Webhook endpoints for call events. *Providers*: Cart Line.

### Product L3.C.4 — Translation & Dubbing

**Module — File-Based Audio Translation**
- Service: `/audio/translations` with `whisper-1` only; output always English text; 25MB limit. *Providers*: OAI.

**Module — Translating Transcription (STT with translation output)**
- Service: `stt-translate` model (set `language`/`target_language`; input any supported → transcript in target; 5 languages en/fr/de/es/pt). *Providers*: Grad.

**Module — Batch Dubbing (Audio/Video Translation)**
- Service: `/dubbing` (translates across 90+ languages preserving emotion/timing/tone/speaker characteristics; speaker separation auto-detect multiple speakers even overlapping; preserve original voices retains speaker identity and emotional tone; keep background audio avoids re-mixing music/effects/ambient; `cloning_strength` configurable default 7 higher=more voice similarity less natural cross-language; sources YouTube/X/TikTok/Vimeo/direct URLs/file uploads; no realtime; watermark free tier none paid; limits 2GB/180min v2, 1GB/45min V1 Studio, up to 9 speakers, 5 concurrent self-serve 100 Enterprise). *Providers*: 11L.

**Module — Live Speech-to-Speech Translation (Real-Time)**
- Service: `gpt-realtime-translate` (continuous WebSocket session, no turn lifecycle, no `response.create`, no conversation lifecycle, model acts as interpreter not assistant; events `session.output_audio.delta`/`session.output_transcript.delta`/`session.input_transcript.delta`; one session per target language; `session.close` to flush; patterns listen-along one source many listeners OR conversational two participants one session per direction). *Providers*: OAI.
- Service: `gemini-3.5-live-translate-preview` (70+ languages; Live API WebSocket continuous stream; `translationConfig:{targetLanguageCode,echoTargetLanguage}`; audio-only input no video/text; no tools/instructions translation only; `echoTargetLanguage:true` echoes target language audio back; ephemeral tokens with `live_connect_constraints`). *Providers*: Goo.
- Service: `s2s-translate` (5 languages en/fr/de/es/pt; duplex WebSocket; pipeline input audio → STT stt-translate → translated text → TTS default → output audio; `voice_id` must match `target_language`; input 24kHz PCM output 48kHz PCM; omit `target_language` for transcription + re-synthesis without translation). *Providers*: Grad.

### Product L3.C.5 — Text-to-Speech Generation

**Module — Single-Speaker TTS**
- Service: TTS request params (union): `text`/`transcript`/`input`, `model`/`model_id`, `voice`/`voice_id`, `language`/`language_code`, `output_format`, `voice_settings`/`generation_config`/`json_config`/`instructions`, `pronunciation_dictionary_locators`/`pronunciation_dict_id`/`pronunciation_id`, `seed`, `previous_text`/`next_text`/`previous_request_ids`/`next_request_ids` (context stitching 11L), `apply_text_normalization`, `apply_language_text_normalization` Japanese, `optimize_streaming_latency` 0-4, `enable_logging`, `speed`, `stream`. *Providers*: all voice.

**Module — Multi-Speaker TTS / Dialogue**
- Service: Text to Dialogue (`inputs[]` with text + `voice_id` per turn; unlimited speakers; `eleven_v3` only; ≤2000 chars total; supports audio tags per turn; punctuation for flow interruptions via `"Hello, is this seat-"` trailing ellipsis; up to 2 free regenerations dashboard only). *Providers*: 11L.
- Service: Multi-speaker TTS (`speech_config` with speaker + voice pairs; up to 2 speakers; `gemini-3.1-flash-tts-preview`; speaker names must match prompt transcript; Director-style prompting Audio Profile + Scene + Director's Notes + Transcript). *Providers*: Goo.

**Module — Voice Behavior Control**
- Service: Stability `stability` 0-1 (11L); Similarity `similarity_boost` 0-1 (11L) / `cfg_coef` 1-4 (Grad); Style `style` 0-1 (11L); Speaker boost `use_speaker_boost` (11L); Speed (`speed` 0.7-1.2 11L; via instructions OAI; via Director's Notes Goo; `generation_config.speed` 0.6-1.5 Cart; `padding_bonus` -4 to 4 Grad); Volume `generation_config.volume` 0.5-2.0 (Cart); Emotion (audio tags + `style` 11L; `instructions` OAI; audio tags + Director's Notes Goo; `generation_config.emotion` 60+ values Cart); Temperature (`seed` 11L; `temperature` Goo; `temp` 0.0-1.4 Grad); Instructions (`instructions` accent/emotion/intonation/speed/tone/whispering OAI; Director's Notes Goo); Accent (via instructions OAI, Director's Notes Goo); Whispering (audio tags `[whispering]` 11L, `[whispers]` Goo, via instructions OAI). *Providers*: per platform.

**Module — Inline Audio Directives (Tags)**
- Service: Audio Tags (square brackets `[sad]`/`[laughing]`/`[whispering]`/`[excitedly]`/`[laughs]`/`[giggles]`/`[sighs]`/`[gasp]`/`[cough]`/`[very fast]`/`[very slow]`/`[leaves rustling]`/`[football]`/`[like a cartoon dog]`; use English tags even for non-English transcripts). *Providers*: 11L v3, Goo.
- Service: SSML Tags (`<speed ratio="1.5"/>`, `<volume ratio="0.5"/>`, `<emotion value="angry"/>` beta, `<break time="1s"/>`, `<spell>ABC123</spell>`; nonverbalism `[laughter]`). *Providers*: Cart.
- Service: In-Text Tags (`<flush>` force audio emission, don't flush after every token small fragments reduce prosody; `<break time="Ns"/>` 0.1-2.0s must be preceded/followed by space). *Providers*: Grad.

**Module — Context Stitching & Continuations**
- Service: `previous_text`/`next_text` (strings) or `previous_request_ids`/`next_request_ids` (up to 3 each) for content longer than per-request char limits. *Providers*: 11L.
- Service: WebSocket contexts + continuations (`context_id` identifies context; `continue:true` on partial chunks, `continue:false` on final; all fields except transcript/continue/duration must be identical; transcripts concatenated verbatim include spaces at boundaries; contexts auto-expire 1s after last audio output). *Providers*: Cart.
- Service: WebSocket text streaming; server inserts single whitespace between consecutive text messages; never split inside a word. *Providers*: Grad.

**Module — Buffering Control**
- Service: Managed buffering (`max_buffer_delay_ms` >0, default 3000; server decides when to start generating; stream LLM tokens directly). *Providers*: Cart.
- Service: Custom buffering (`max_buffer_delay_ms:0`; server generates immediately; you aggregate sentences). *Providers*: Cart.

**Module — Multiplexing**
- Service: Multi-context WebSocket (`multi-stream-input`); multiple conversation contexts; only generation time counts toward concurrency. *Providers*: 11L.
- Service: Multiple `context_id` values active simultaneously; audio tagged with `context_id`; only unique active contexts count toward concurrency. *Providers*: Cart.
- Service: `close_ws_on_eos:false` + unique `client_req_id` on every message; server stamps all responses with matching `client_req_id`. *Providers*: Grad.

**Module — Timestamps in TTS Output**
- Service: `with-timestamps` endpoints (character-level 11L); `add_timestamps` (word-level) + `add_phoneme_timestamps` + `use_normalized_timestamps` (Cart SSE+WebSocket); `text` messages with `start_s`/`stop_s` per segment word-level (Grad); metadata headers `dg-char-count`/`dg-request-id` (Deep). *Providers*: per platform.

**Module — Determinism**
- Service: `seed` 0-4294967295 (improves consistency not guaranteed; up to 2 free regenerations identical params 11L); `temp:0.0` deterministic (Grad); pinned model IDs `sonic-3.5-2026-05-04` (Cart). *Providers*: per platform.

**Module — Free Regenerations**
- Service: Up to 2 free regenerations per generation when identical text and parameters; dashboard-only for Text to Dialogue. *Providers*: 11L.

### Product L3.C.6 — Voice Transformation

**Module — Voice Changer (STS, No Translation)**
- Service: `/speech-to-speech/{voice_id}` + `/stream` (1000 chars/min; 5 min max segment; `model_id`, `remove_background_noise`, `audio`). *Providers*: 11L.
- Service: `/voice-changer/bytes` + `/sse` (15 credits/sec; `clip`, `voice[id]`, `output_format`). *Providers*: Cart.

**Module — Voice Isolator (Audio Isolation / Noise Removal)**
- Service: `/audio-isolation` + `/stream` + history management (max 500MB, 1 hour; not optimized for vocal isolation from music). *Providers*: 11L.

**Module — Audio Infill / Bridging**
- Service: `/infill/bytes` (`left_audio` → `transcript` → `right_audio`; at least one of left/right must be provided; longer transcripts give more adaptation flexibility; cost 300 credits + ~1 credit/char). *Providers*: Cart.

**Module — Stem Separation**
- Service: `/music/separate-stems` (separate song into individual instrument/vocal stems). *Providers*: 11L.

### Product L3.C.7 — Sound Effects & Music Generation

**Module — Sound Effects (Text-to-Sound)**
- Service: `/text-to-sound-effect` (model `eleven_text_to_sound_v2`; `text` description; `duration_seconds` 0.1-30s auto if not specified 40 credits/sec when specified; `prompt_influence` high=literal low=creative; `loop` seamless for >30s; prompting categories simple/complex/musical/audio terminology; output MP3 all, WAV 48kHz non-looping). *Providers*: 11L.

**Module — Music Generation**
- Service: `/music/compose` (text-to-music; fine-tunes on own non-copyrighted tracks 5-10 min; curated finetunes Afro House/Reggaeton/Arabic Groove). *Providers*: 11L.

### Product L3.C.8 — Output Formatting & Delivery (voice)

**Module — Output Formats**
- Service: mp3, pcm, wav, opus, μ-law, a-law (11L); mp3, opus, aac, flac, wav, pcm (OAI); raw PCM 24kHz (Goo); raw, wav, mp3 pcm_s16le/f32le/mulaw/alaw (Cart); mp3, linear16, flac, mulaw, alaw, opus, aac (Deep); wav, pcm, opus, ulaw_8000, alaw_8000 (Grad). *Providers*: per platform.

### Product L3.C.9 — Privacy & Data Retention (voice)

**Module — Zero Retention / ZDR**
- Service: `enable_logging=false` (Enterprise only). *Providers*: 11L.
- Service: Zero Data Retention (Enterprise). *Providers*: Cart.
- Service: `mip_opt_out=true`. *Providers*: Deep.

**Module — Data Residency**
- Service: Regional servers US/EU/India/Singapore. *Providers*: 11L.
- Service: Europe/US servers. *Providers*: Grad.

### Product L3.C.10 — Billing Units (voice)

**Module — Billing**
- Service: Characters per text synthesized (11L, Grad, Deep TTS); Credits (Cart, 11L, Grad); Per second of audio STT and voice changing (Cart, Deep, Grad); Per minute dubbing/agent calls (11L, Cart Line); Per generation music/sound effects (11L); Surcharges entity detection +30%, keyterm prompting +20%, speaker roles +10% (11L). *Providers*: per platform.

---

## Domain L3.D — Documents

### Product L3.D.1 — Ingestion & Storage (index time)

**Module — Authentication & Access Control (document)**
- Service: API key bearer (`Authorization: Bearer`/`X-API-Key`, google/lighton/mixedbread/openai/weaviate/mistral/datalab); HTTP Basic + Bearer/Zen JWT (ibm); OIDC tokens (weaviate); model provider keys via headers `X-OPENAI-API-KEY`/`X-COHERE-API-KEY` for self-hosted vectorizer/generator (weaviate); scoped API keys with roles owner/editor/viewer workspace-scoped (lighton). *Providers*: per platform.

**Module — Container / Workspace Management**
- Service: Workspace with type + residency (`shared`/`personal`, `processing_location:us/eu`, embedding model, chunking config, expiration). *Providers*: Ligh, Data.
- Service: Vector store with expiration (`expires_after:{anchor,days}`). *Providers*: OAI, mxb.
- Service: FileSearchStore with immutable embedding model (`embedding_model` set at creation). *Providers*: Goo.
- Service: Collection with schema + vectorizer config (named vectors, distance metric, index type, per-property index flags). *Providers*: Weav.
- Service: Project with ontology (doc types, KeyClasses, validators). *Providers*: IBM.
- Service: Dataset / Frame (lazy, immutable; Python API ↔ YAML). *Providers*: docE.

**Module — File Upload & Ingestion**
- Service: Direct multipart upload. *Providers*: Data, Doc, IBM, Ligh, Mst, mxb, OAI.
- Service: URL-based ingestion (platform fetches). *Providers*: Data, Doc, Goo, Ligh, Mst, mxb.
- Service: Base64 inline. *Providers*: Doc, Goo, Mst.
- Service: Presigned-URL upload (platform issues presigned PUT URL). *Providers*: Data, mxb.
- Service: Resumable/multipart upload (~1TB). *Providers*: Goo, mxb.
- Service: Cloud-storage loaders (S3/Azure Blob/GCS/Google Drive/SharePoint sync). *Providers*: Mst, Ligh.
- Service: Local filesystem/directory batch. *Providers*: Doc, docE, Mst.
- Service: In-memory/stream (`DocumentStream`, `from_list`, `convert_string`). *Providers*: Doc, docE.
- Service: Pre-chunked ingestion (MXJSON/MXJSONL bypass parsing/chunking). *Providers*: mxb.
- Service: Docling JSON round-trip (re-ingest prior Docling JSON to re-export without re-parsing). *Providers*: Doc.
- Service: `datalab://` file references (stable URI). *Providers*: Data.
- Service: Object insertion (JSON directly into vector DB). *Providers*: Weav.
- Service: Key params (`file`/`file_url`/`document`/`base64_string`/`http_sources`, `workspace_id`/`store_id`/`vector_store_id`/`collection`/`project_id`/`parent`, `filename`/`display_name`/`title`, `metadata`/`attributes`/`custom_metadata`/`external_metadata`/`tags`, `external_id` idempotent re-upload overwrites not duplicates, `parser`/`parsing_strategy`, `processing_location`, `chunking_config`/`chunking_strategy`, `purpose`, `max_file_size`). *Providers*: per platform.
- Service: Async processing (job/file ID + status poll/webhook); status lifecycles pending→in_progress→completed|failed|cancelled (mxb/OAI), pending→pending_conversion→converting→parsing→embedding→embedded (Ligh), PROCESSING→ACTIVE|FAILED (Goo), pending→started→success|failure (Doc), In Progress→Completed|Failed (IBM), pending→dispatched→running→completed→failed→skipped (Data pipelines). *Providers*: per platform.
- Service: Webhooks (Data, IBM, Doc via WS); batch ingestion up to 500 files (OAI), bulk operations (mxb), `convert_all` (Doc), batch over collections (Data), concurrent asyncio.gather (Mst). *Providers*: per platform.
- Service: Idempotent ingestion (deterministic UUID5/`external_id`). *Providers*: Weav, Mst, Ligh, mxb.

**Module — Document Segmentation / Boundary Detection**
- Service: Schema-guided segmentation (`segmentation_schema` with segment names + descriptions). *Providers*: Data.
- Service: Automatic boundary detection (`segmentation_strategy:document_boundary`). *Providers*: Data.
- Service: Page-structure segmentation (header-based). *Providers*: IBM.
- Service: Output segments with name, page ranges, confidence (high/medium/low). *Providers*: Data.

### Product L3.D.2 — Document Understanding (parse + extract; checkpoint-reusable)

**Module — Document Parsing, OCR & Layout Analysis**
- Service: Multi-model pipeline (separate models for layout DocLayNet, OCR Tesseract/Surya/RapidOCR, table structure TableFormer; modes fast/balanced/accurate). *Providers*: Doc, Data.
- Service: Single end-to-end VLM (GraniteDocling 258M or remote VLM API replaces entire chain). *Providers*: Doc.
- Service: Native multimodal vision (LLM "sees" PDF pages as images; `media_resolution:low/medium/high`). *Providers*: Goo.
- Service: Managed automatic parsing (platform-internal, not exposed). *Providers*: Ligh, mxb, OAI.
- Service: Dedicated OCR API (`mistral-ocr-latest`/`4-0`; 13 block types with bboxes in reading order; confidence scores; markdown + images + tables + blocks). *Providers*: Mst.
- Service: Word-level OCR with font metadata (per-word coordinates, confidence, bold/italic/font). *Providers*: IBM.
- Service: Format-specific backends (subclassable per format PDF/DOCX/HTML/image/audio). *Providers*: Doc.
- Service: Audio/video transcription (ASR) (Whisper, Voxtral; diarization + timestamps; output WebVTT or text). *Providers*: Doc, Mst.
- Service: Legacy office conversion (`.doc/.ppt/.hwp` → PDF via PyMuPDF Pro → OCR). *Providers*: Mst.
- Service: Output representations — Markdown per-page or full-document with tables/lists/headings (Data/Doc/Mst/Ligh); HTML with `data-block-id` for citation tracking (Data/Doc); JSON/DoclingDocument hierarchical content items/body tree/furniture tree/groups/reading order/bboxes/provenance (Doc/Data/IBM); DocTags compact markup token-efficient (Doc); DocLang XML/`.dclx` schema-validated zipped page images (Doc); Chunks pre-segmented (Data); Blocks paragraph-level with bboxes/types (Mst/Data); Bounding boxes per-word/cell/list-item/block with confidence (Data/Doc/IBM/Mst); Confidence scores page/word/parse-quality 0-5 (Data/IBM/Mst); Images extracted/base64/referenced with captions (Data/Doc/Mst); Tables structured rows/columns/multi-level headers markdown or HTML (Data/Doc/Mst/IBM). *Providers*: per platform.
- Service: Enrichments — code language detection `code_language` (Doc); formula extraction LaTeX (Doc); picture classification chart/diagram/logo/signature (Doc); picture description/captioning local VLM or remote API (Doc); chart understanding/infographics (Data `extras:chart_understanding,infographic`); link extraction hyperlinks (Data/Mst); track changes/redline extraction insertions/deletions/comments from DOCX (Data); headers/footers detection (IBM/Mst/Doc-furniture); reading order detection encoded in body tree (Doc). *Providers*: per platform.
- Service: Key params (`mode`/`processing_mode` fast/balanced/accurate; `do_ocr`/`force_ocr`/`ocr_lang`/`ocr_preset`; `table_format` markdown/html/None; `include_image_base64`/`disable_image_extraction`/`image_export_mode` placeholder/embedded/referenced; `include_blocks`/`add_block_ids`; `confidence_scores_granularity` page/word/None; `bbox_annotation_format`; `paginate`/`page_range`/`max_pages`; `word_bboxes`/`table_cell_bboxes`/`list_item_bboxes`; `token_efficient_markdown`; `keep_pageheader_in_output`/`keep_pagefooter_in_output`; `jsonOptions` comma-separated HR/DC/KVP/TH/OCR/SN/MT/DS/CHAR with dependency chain IBM; `media_resolution`). *Providers*: per platform.
- Service: Checkpoint system (stored parse state `checkpoint_id` from conversion with `save_checkpoint:true`, reusable by extraction/segmentation/schema generation — parse once, run many extractions without re-parsing). *Providers*: Data.

**Module — Data Extraction (fields, tables, KVPs, annotations)**
- Service: JSON-schema-driven LLM extraction (modes turbo image-only no citations, fast citations+scores, balanced verification+reasoning+citations). *Providers*: Data, Goo, Ligh, Mst, mxb, docE.
- Service: BBox annotation (per-image classification/description via schema). *Providers*: Mst.
- Service: Document annotation (document-level structured extraction via schema). *Providers*: Mst.
- Service: KVP extraction with ontology (key+value with coordinates, confidence, KeyClass tagging, validators; three output tiers basic/detailed/verbose). *Providers*: IBM.
- Service: Table/line-item extraction (recursive `ComplexKVPStructure` with nested rows/cells; nested tables). *Providers*: IBM.
- Service: Semantic normalization (cleans/standardizes values names/addresses with `OriginalValue` preservation). *Providers*: IBM.
- Service: Verbatim text extraction (pull source text without synthesis; line_number or regex strategy; lower token cost no hallucination). *Providers*: docE `Extract`.
- Service: Form filling (fill PDF/image forms with field data; AcroForm + visual + image field detection; confidence threshold). *Providers*: Data.
- Service: Schema auto-generation (generate candidate extraction schemas simple/moderate/complex from a checkpoint). *Providers*: Data.
- Service: Pydantic-template extraction (schema-first Pydantic models define extraction schema AND graph structure; one-to-one or many-to-one). *Providers*: KG.
- Service: Dense extraction (two-phase skeleton-then-flesh extraction contract for large documents). *Providers*: KG.
- Service: Output quality signals — per-field citations block IDs traceable to source (Data); per-field verification status PASS/FAIL_UNRESOLVABLE/FAIL_FIX/FAIL_CITATIONS/ITEMS_MISSING (Data); per-field confidence score 1-5 + reasoning (Data); per-field `KeyConfidence`/`ValueConfidence`/`KeyClassConfidence` (IBM); extraction score average (Data); parse quality score 0-5 (Data). *Providers*: per platform.

**Module — Classification & Categorization**
- Service: AI document classification (`Classification.DocumentClass` with `ClassConfidence`, `ClassMatch` Low/Medium/High, `AlternateDocumentClass[]`). *Providers*: IBM.
- Service: Hierarchical content-type taxonomy Facets (multi-tree max depth 4, `:`-path notation; typed inheritable attributes per node; seed templates finance/healthcare/legal/manufacturing/tech; file-level classify/unclassify/set_value/clear_value actions; sibling-conflict rules; filterable in search). *Providers*: Ligh.
- Service: Flat tags (company-wide non-hierarchical labels; `auto_assign` flag; OR'd in queries). *Providers*: Ligh.
- Service: Classification modification via custom processors (classify pages for downstream routing). *Providers*: Data.
- Service: LLM map with enum output (classification via structured output schema with calibration for consistency). *Providers*: docE.
- Service: Filter-based classification (metadata filtering and facets on user-defined metadata). *Providers*: mxb.
- Service: Picture classification (chart types, diagrams, logos, signatures). *Providers*: Doc.
- Service: Zero-shot classification via embeddings (embed labels, compare cosine similarity). *Providers*: OAI.
- Service: Key params (`docClass` override IBM; `content_type_path` Ligh; `attribute_name`/`value` Ligh; `class_match` threshold IBM). *Providers*: per platform.

### Product L3.D.3 — Chunking & Enrichment (prepare for indexing)

**Module — Chunking / Splitting**
- Service: Static/token-count (`max_chunk_size_tokens` 100-4096 default 800, `chunk_overlap_tokens` default 400). *Providers*: OAI, Goo, Mst, docE.
- Service: Character-count (`chunk_size` default 1000). *Providers*: Mst.
- Service: Separator/hierarchical (split on configurable separators paragraph/sentence with fallback). *Providers*: Mst, docE.
- Service: Markdown/header-aware (split by markdown headers, preserve header context). *Providers*: Mst, Doc.
- Service: Hierarchical/structure-pure (one chunk per document element; merges list items; attaches headers/captions). *Providers*: Doc `HierarchicalChunker`.
- Service: Hybrid tokenization-aware (splits oversized, merges undersized; aligned to tokenizer; table-header repetition; production default for RAG). *Providers*: Doc `HybridChunker`.
- Service: Line-based (preserves line boundaries tables/code/logs/lists; repeated prefixes). *Providers*: Doc `LineBasedTokenChunker`.
- Service: Word-count with overlap (`chunk_size` words, `overlap` words). *Providers*: KG AI-KG.
- Service: Structure-preserving Docling-based (token-bounded; tables/lists/section hierarchy intact; sentence→word→char fallback). *Providers*: KG Docling-Graph.
- Service: Automatic/managed (platform-managed, not configurable). *Providers*: Ligh, mxb, Goo default, OAI default.
- Service: Pre-chunked bypass (provide chunks directly via MXJSON/MXJSONL). *Providers*: mxb.
- Service: Gather context enrichment (adds context from surrounding chunks previous/next head/middle/tail + header hierarchy). *Providers*: docE `Gather`.
- Service: Chunk types (`text`, `image_url`, `audio_url`, `video_url` mxb; `content`, `image_annotation`, `summary` Mst). *Providers*: per platform.
- Service: Key params (`max_chunk_size_tokens`/`max_tokens_per_chunk`/`chunk_size`/`chunk_max_tokens`, `chunk_overlap_tokens`/`max_overlap_tokens`/`overlap`, `tokenizer`/`tokenizer_name`/`tokenizer_model`, `merge_list_items`/`merge_peers`/`repeat_table_header`/`omit_header_on_overflow`, `keep_separator`/`strip_whitespace`/`strip_headers`/`headers_to_split_on`, `num_splits_to_group`). *Providers*: per platform.
- Service: Metadata preserved on chunks (`page_number`, `filename`, `filepath`, `start_offset`/`end_offset`, `images`, `chunk_id`, `chunk_index`, `headings`, `token_count`, `text_hash`, `char_length`, `locator` char:/page:/summary: formats Mst). *Providers*: per platform.

**Module — Chunk Enrichment & Contextualization**
- Service: Summary enrichment (generate document/chunk summary, optionally prepend to every chunk `propagate_summary_to_chunks`). *Providers*: Mst.
- Service: Contextualization at index time (`with_metadata`, `with_file_context`). *Providers*: mxb.
- Service: Gather context windows (add peripheral chunks previous/next + reconstructed headers). *Providers*: docE.
- Service: Generated metadata (auto-extract typed metadata per chunk: language, word_count, page info, code line ranges, media durations, BPM, frame counts, temporal boundaries). *Providers*: mxb.
- Service: Custom enrichers (pluggable `ChunkEnricher` interface for arbitrary metadata injection). *Providers*: Mst.

### Product L3.D.4 — Embedding, Indexing & Graph (build the retrievable store)

**Module — Indexing & Storage**
- Service: Managed vector store auto-indexed (platform manages embeddings, indexing, sharding; free storage in some). *Providers*: Goo, Ligh, mxb, OAI.
- Service: Self-hosted vector database (HNSW/Flat/Dynamic/HFresh index types; LSM-Tree storage; WAL + HNSW snapshots; lazy shard loading; async indexing). *Providers*: Weav.
- Service: Vespa vector store (swappable schema with `embedding_dimensions`, `indexing_mode`, `SearchMode`). *Providers*: Mst.
- Service: LanceDB local index (FTS, embedding, or hybrid; no server). *Providers*: docE.
- Service: Native graph database (index-free adjacency; ACID; Bolt protocol; persistent). *Providers*: KG Neo4j.
- Service: File-based storage (CSV, JSON, Cypher script, HTML). *Providers*: KG.
- Service: DPE database (results persist until explicitly deleted; no auto-expiration). *Providers*: IBM.
- Service: S3 integration (source syncing and output writeback). *Providers*: Data.
- Service: BYOB Bring Your Own Bucket (enterprise feature; own object storage backend; ephemeral compute). *Providers*: mxb.
- Service: Vector index types — HNSW (default in-memory graph; `ef_construction`, `max_connections`/M, `ef` dynamic, `dynamicEfMin/Max/Factor`, `flatSearchCutoff`, `filterStrategy` ACORN/sweeping); Flat (brute-force small data); Dynamic (auto flat→HNSW at 10k); HFresh (disk-backed memory-efficient). *Providers*: Weav.
- Service: Vector quantization/compression — PQ (Product Quantization ~24x `segments`/`trainingLimit`/`encoder.type`); BQ (Binary 32x no training); SQ (Scalar 4x min/max training); RQ (Rotational 8-bit 4x/1-bit ~32x training-free 98-99% recall `rescoreLimit`); re-scoring over-fetch compressed candidates re-score with original vectors. *Providers*: Weav.
- Service: Inverted index (roaring bitmaps for filtering; map index for BM25). *Providers*: Weav.
- Service: Per-property index flags (`index_filterable`, `index_searchable`, `index_range_filters`, `index_null_state`, `index_property_length`). *Providers*: Weav.
- Service: Lifecycle/cost control (`expires_after`/`expires_at` auto-expire dormant stores `anchor:last_active_at` OAI/mxb; TTL object expiration relative to DATE property Weav v1.36; result retention 1 hour after completion Data; raw file expiration 48 hours embeddings persist indefinitely Goo). *Providers*: per platform.
- Service: Consistency levels ALL/QUORUM default/ONE per-query. *Providers*: Weav.
- Service: Replication config; backups. *Providers*: Weav.
- Service: Multi-tenancy (per-tenant sharding; 50,000+ active shards per node; auto-tenant creation; tenant lifecycle ACTIVE→INACTIVE→OFFLOADED to S3). *Providers*: Weav.

**Module — Entity Detection, Resolution, Deduplication & Linking**
- Service: Blocking + pairwise LLM comparison + union-find clustering + resolution (reduce comparisons via code conditions and/or embedding similarity thresholds; auto-computed blocking threshold 95% recall target; union-find DSU for grouping; LLM resolution prompt). *Providers*: docE.
- Service: Deterministic node ID registry (fingerprint hashing `ClassName_fingerprint`; same entity always same node ID across batches; cross-document graph merging). *Providers*: KG.
- Service: LLM-based entity standardization (cluster entity name variants into canonical nodes; remove self-referencing triples). *Providers*: KG.
- Service: Dense dedupe (`off`/`standard`/`aggressive` LLM reconciliation of same-entity aliases/OCR noise). *Providers*: KG.
- Service: Link resolve (fix links between items in a knowledge graph one-sided; assumes canonical IDs; embedding blocking + LLM comparison). *Providers*: docE.
- Service: Equijoin fuzzy join (join two datasets by LLM-evaluated semantic similarity; embedding blocking). *Providers*: docE.
- Service: Cross-references (directional links between objects across collections; manual entity-linking). *Providers*: Weav.
- Service: BARGAIN cascade (cheap proxy model + oracle-labeled threshold learning with statistical guarantees for binary resolve/filter). *Providers*: docE.
- Service: Key params (`blocking_keys`, `blocking_threshold`, `blocking_target_recall` 0.95, `blocking_conditions`, `comparison_prompt`, `resolution_prompt`, `embedding_model`, `cascade` BARGAIN). *Providers*: docE.

**Module — Clustering**
- Service: Hierarchical agglomerative clustering (binary tree of embeddings; cluster path annotation most-specific→root; LLM-generated cluster summaries). *Providers*: docE `Cluster`.
- Service: KMeans on embeddings (discover hidden groupings). *Providers*: OAI.
- Service: Louvain community detection (color-coded clusters in visualization). *Providers*: KG.
- Service: Value sampling cluster (k-means for representative subset selection in reduce). *Providers*: docE.
- Service: t-SNE visualization (2D cluster diagnostics). *Providers*: OAI.

**Module — Knowledge Graph Building**
- Service: Schema-validated Pydantic-driven pipeline (Pydantic templates define extraction schema AND graph structure; deterministic provenance ledger; stable cross-batch node IDs; graph conversion to NetworkX DiGraph; export CSV/Cypher/JSON/HTML; extraction contracts direct one-pass / dense skeleton-then-flesh two-phase / auto resolves per document; backends LLM text local/remote or VLM vision local only; gleaning extra "what did you miss?" extraction pass; dense dedupe off/standard/aggressive). *Providers*: KG Docling-Graph.
- Service: Schema-free LLM triple extraction (SPO triples subject/predicate/object/chunk; entity standardization LLM clustering of variants; relationship inference rule-based transitivity + lexical similarity + LLM-assisted subgraph bridging; PyVis HTML visualization with Louvain communities). *Providers*: KG AI-Knowledge-Graph.
- Service: Native graph database storage (Neo4j with Cypher CRUD; index-free adjacency; ACID; GraphRAG; MCP server for agent tool exposure). *Providers*: KG Neo4j.
- Service: Link resolve (fix links between items in a knowledge graph one-sided). *Providers*: docE.
- Service: Cross-references (directional links between objects across collections). *Providers*: Weav.
- Service: Graph export formats CSV (Neo4j-compatible), Cypher script, JSON (node-link), HTML visualization, Docling outputs. *Providers*: KG.
- Service: Graph statistics `node_count`, `edge_count`, `node_types`, `edge_types`, `avg_degree`, `density`. *Providers*: KG.
- Service: Provenance levels `off`/`standard`/`detailed` (with char-offset spans). *Providers*: KG.

### Product L3.D.5 — Query Time (read path, correct order)

**Module — Query Rewriting & Preprocessing**
- Service: Query rewriting `rewrite_query` (auto-rewrite into search-friendly form strips conversational filler converts to noun-phrase; rewritten query observable in `search_query`). *Providers*: OAI, mxb.
- Service: LLM query rewriter `LLMQueryRewriter` (reformulate informal queries). *Providers*: Mst.
- Service: LLM query extension `LLMQueryExtension` (generate multiple sub-queries for broader retrieval). *Providers*: Mst.
- Service: Model-generated queries (File Search tool generates its own queries visible in `queries`). *Providers*: OAI.

**Module — Search (Lexical, Vector, Hybrid, Agentic)**
- Service: Lexical/keyword BM25/BM25F (token frequency + IDF; BM25F weighted multi-field; per-property tokenization WORD/LOWERCASE/WHITESPACE/FIELD/TRIGRAM for fuzzy/typo tolerance; property boosting `^weight`; `and`/`or` operators with `minimum_match`; accent folding; stopword presets). *Providers*: Weav.
- Service: Grep regex (RE2 regex pattern matching against literal chunk text; `content_groups` text/generated; case sensitivity). *Providers*: mxb.
- Service: Vector/semantic `near_text`/`near_vector`/`near_object`/`near_image` (four input modalities Weav); `VectorRetriever` (Mst); semantic via embeddings (Goo/Ligh/mxb/OAI); move parameters curriculum learning `move_to`/`move_away` with force + concepts (Weav); MR Maximal Marginal Relevance diversity selection (Weav v1.37 preview); Autocut/auto_limit detect natural breaks in distance/score distribution (Weav). *Providers*: per platform.
- Service: Hybrid vector+keyword fusion (run both in parallel, fuse weighted by `alpha` 0=pure keyword 1=pure vector 0.5=balanced; two fusion algorithms `RELATIVE_SCORE` default / `RANKED` `1/(RANK+60)` Weav); Reciprocal Rank Fusion RRF blend semantic+keyword tunable `embedding_weight`/`text_weight` (OAI at least one must be >0); `RRFRanker` with `rrf_k` smoothing for multi-retriever fusion (Mst); hybrid vector+keyword+vision score breakdown per component text/vision/keyword/multivector/relevance (Ligh); hybrid web+internal virtual web store in `store_identifiers` web results always reranked merged with internal (mxb). *Providers*: per platform.
- Service: Agentic search — multi-round agent-driven retrieval (decompose complex questions into sub-queries; run multiple rounds; evaluate candidates; iterate; merge/rerank; `max_rounds`, `queries_per_round`, `instructions` natural-language steering, `strict_top_k`, `score_threshold` mxb `agentic`); Query Agent (LLM translates natural language into database operations searches/aggregations/filters/sorts across collections; modes Ask answer+sources / Search raw objects / Suggest query discovery; streaming with progress; `additional_filters`; `view_properties`; pagination reusing searches Weav); Model-autonomous File Search (model decides when to search, generates queries, synthesizes answers OAI). *Providers*: mxb, Weav, OAI.
- Service: Other search modes — List Chunks metadata-only retrieval (no embeddings/similarity/reranking; sort by metadata mxb); Multi-store/federated search across multiple stores in one query (Goo/mxb); Cross-collection Explore GraphQL function (Weav); Cross-document reasoning up to 1000 pages across multiple PDFs (Goo). *Providers*: per platform.
- Service: Key search params (`query`/`input`, `top_k`/`max_results`/`max_num_results`/`limit` 1-50 typical, `mode` text/vision Ligh, `alpha` keyword/vector balance Weav, `fusion_type` RELATIVE_SCORE/RANKED Weav, `ranking_options` ranker/score_threshold/hybrid_search weights OAI, `search_options` rerank/rewrite_query/agentic/score_threshold/return_metadata/media_content mxb, `target_vector` named vector selection Weav, `distance`/`certainty` similarity threshold Weav, `include_image`/`include_bboxes` visual output Ligh, `media_content` auto/always/never mxb). *Providers*: per platform.

**Module — Filtering, Facets & Metadata Scoping**
- Service: Three metadata layers (user file metadata bare keys; generated chunk metadata `generated_metadata.` prefix; system fields `file_id`/`chunk_index` mxb). *Providers*: mxb.
- Service: Comparison filters `eq`/`ne`/`gt`/`gte`/`lt`/`lte`/`in`/`nin`/`like`/`not_like` (OAI/mxb); compound `and`/`or`/`all` AND/`any` OR/`none` NOT (OAI/mxb/Weav); AIP-160 filter expressions `author="Robert Graves" AND year>=1934` (Goo); Weav filter operators `equal`/`not_equal`/`less_than`/`greater_than`/`like` with `*`/`?` wildcards/`contains_any/all/none`/`is_none`/`within_geo_range`/`length`; cross-reference filtering `Filter.by_ref`/`Filter.by_ref_count` (Weav); nested object property filtering index-based `cars[0].make` same-element correlation (Weav v1.38 preview); content-type/facet filters `content_type[]` wildcard path filters `attribute[]` AND across OR within pipe-separated (Ligh); tag filters `tag_id[]` OR'd (Ligh); file ID scoping `file_ids` inclusion or `["not_in",[...]]` (mxb). *Providers*: per platform.
- Service: Facets (aggregate chunk counts grouped by metadata values; query-time faceting; dot notation for nested fields). *Providers*: mxb.
- Service: Filter strategy ACORN default preserves HNSW graph integrity vs sweeping. *Providers*: Weav.
- Service: Attributes (file-level key-value metadata; max 16 keys, 256 chars per key; string/number values). *Providers*: OAI.

**Module — Reranking**
- Service: Rerank (second-stage reordering with more expensive model; runs after search and filtering) — `rerank` (Weav `Rerank`); `Reranker` (Mst); `relevance_scoring` (Ligh); `rerank` (mxb); not separate API in OAI/Goo (via `ranking_options.ranker`). *Providers*: per platform.
- Service: OAI Retrieval API `POST /v1/vector_stores/{id}/search` (`query`, `rewrite_query`, `max_num_results` ≤50, `attribute_filter` comparison/compound, `ranking_options` `ranker` auto/`default-2024-08-21`, `score_threshold`, `hybrid_search` `{embedding_weight,text_weight}`/`{rrf_embedding_weight,rrf_text_weight}`). *Providers*: OAI.
- Service: Mst `RRFRanker` with `rrf_k` smoothing for multi-retriever fusion. *Providers*: Mst.
- Service: mxb `search_options.rerank` and `score_threshold`. *Providers*: mxb.

**Module — Caching**
- Service: (Document query caching where offered — e.g. OpenRouter response caching at L2; per-platform internal result caching.)

**Module — Aggregations, Grouped Search & Analytics**
- Service: Aggregations (per Weav/mxb; grouped search; analytics over metadata). *Providers*: Weav, mxb.

### Product L3.D.6 — Generation & Output

**Module — Answer Generation / RAG / QA**
- Service: Generative search (Weav); Ask (Ligh); Question Answering (mxb); File Search tool (OAI); Document QnA (Mst); File Search (Goo); Custom Question Answering CQA (Az — layered ranking Azure AI Search → NLP re-ranking → confidence). *Providers*: per platform.
- Service: Managed RAG — Mistral Libraries (upload docs → ingest/vectorize/search → `document_library` tool on agents → grounded answers with `tool_reference` citations); OpenAI `file_search` built-in tool; Google context caching (cache large files, query against cached context); Azure CQA knowledge base. *Providers*: per platform.
- Service: RAG from scratch (Mistral guide): get data → load text → split into chunks (by character/tokens/sentences/paragraphs/HTML headers/AST for code) → create embeddings `client.embeddings.create` → load into vector DB e.g. FAISS → embed question same model → retrieve similar chunks `index.search` → combine context + question in prompt → `client.chat.complete`; RAG techniques HyDE hypothetical answer embedding, child/parent chunks, time-weighted retrieval, "lost in the middle" reordering, metadata filtering, BM25, few-shot prompting. *Providers*: Mst.

**Module — Document Transformation & Round-trip**
- Service: Docling JSON round-trip (re-ingest prior Docling JSON to re-export without re-parsing). *Providers*: Doc.
- Service: Form filling (fill PDF/image forms; AcroForm + visual + image field detection; confidence threshold). *Providers*: Data.

### Product L3.D.7 — Cross-Cutting Concerns (document)

**Module — Orchestration, Pipelines & Workflows**
- Service: Pipeline (Data/Mst/docE); Workflow Temporal (Data); operator chain Frame (docE); `run_pipeline` (KG); ingestion pipeline (mxb/Ligh). *Providers*: per platform.
- Service: datalab Workflow (Temporal-based orchestration of document processing steps). *Providers*: Data.

**Module — Evaluation, Quality Assurance & Optimization**
- Service: Per-field verification status PASS/FAIL (Data); per-field confidence score 1-5 + reasoning (Data); extraction score average (Data); parse quality score 0-5 (Data); class confidence + alternate candidates (IBM). *Providers*: per platform.

**Module — Provenance, Citations & Source Tracking**
- Service: `file_citation` (Goo/OAI); citations / block IDs (Data); provenance / `ProvenanceLedger` (KG/Data); source attribution (Ligh/mxb). *Providers*: per platform.

**Module — Visualization**
- Service: PyVis HTML visualization with Louvain communities (KG AI-KG); t-SNE 2D cluster diagnostics (OAI); graph HTML export (KG); Docling output visualization. *Providers*: per platform.

**Module — Multi-tenancy, Security, Residency & Administration**
- Service: Workspace (Ligh); Tenant (Weav per-tenant sharding 50,000+ active shards/node auto-tenant creation tenant lifecycle ACTIVE→INACTIVE→OFFLOADED to S3); Organization (mxb); team context (Data). *Providers*: per platform.
- Service: `processing_location:us/eu` (Data); BYOB Bring Your Own Bucket enterprise own object storage (mxb); consistency levels ALL/QUORUM/ONE per-query (Weav); replication config + backups (Weav). *Providers*: per platform.

### Product L3.D.8 — Billing Units (document)

**Module — Billing**
- Service: Per page — conversion, extraction priced per 1K pages (Data/IBM); per token — embedding, generation usage (OAI/mxb/Ligh); per request/per file — ingestion counts (Goo/mxb); storage — free in some (Goo), per-index in others (Weav/OAI); surcharges — add-on bbox extraction $0.30/1K pages (Data), EU residency premium (Data). *Providers*: per platform.

---

# LAYER 4 — AGENTIC ORCHESTRATION

> **Purpose.** Define agents (model + instructions + tools + skills + permissions), provision sandbox environments, start sessions/runs, drive the autonomous loop, stream progress as events, pause for approvals, register hooks, orchestrate multiple agents, attach memory/RAG, run workflows/scheduled tasks, deliver via channels, and extend via plugins/marketplaces. *Depends on*: L2 (model inference, tool-calling primitive), L3 (voice channels, file search, image generation tool, code execution), L1 (GPU sandboxes). *Supervised by*: L5.
> **Sources:** `agents/summary.md`, `tools/summary.md` (tool catalog/MCP/skills/programmatic), `observability/summary.md` (hooks/approvals), `admin/summary.md` (orchestration).

## Domain L4.A — Platform Onboarding & Authentication

### Product L4.A.1 — Authentication & Access Control (agent)

**Module — API Key / OAuth / Login**
- Service: API-key auth (bearer or `x-api-key` header on every call, all REST systems). *Providers*: Ant Managed, Goo, IBM, Mst, OAI.
- Service: OAuth / login flow (ChatGPT login Codex; Mistral account OAuth Vibe; IAM/MCSP/CPD bearer token from login endpoint IBM). *Providers*: Codex, Vibe, IBM.
- Service: Scoped/ephemeral keys (`CODEX_API_KEY` scoped to one invocation Codex; Workspace Agent access tokens scoped to trigger API Codex; API keys scoped to Connector access only Mst). *Providers*: Codex, Mst.
- Service: Admin vs user key roles (General vs Inference key types; admins manage keys for all users in instance Bob). *Providers*: Bob.
- Service: Beta/version headers (`anthropic-version`, `managed-agents-2026-04-01`, `agent-memory-2026-07-22`, `skills-2025-10-02` Ant; `Api-Revision:2026-05-20` Goo). *Providers*: Ant, Goo.
- Service: Provider routing/gateways (Anthropic via Bedrock/AWS/GCP Agent Platform/Microsoft Foundry/LLM gateway Claude; OpenAI-compatible model gateway passthrough IBM). *Providers*: Ant (Claude), IBM.
- Service: WebSocket auth (capability-token or signed-bearer-token with issuer/audience/clock-skew Codex app-server). *Providers*: Codex.
- Service: License acceptance (`--accept-license` before first non-interactive run Bob). *Providers*: Bob.

## Domain L4.B — Agent Definition & Configuration

### Product L4.B.1 — Agent Object & Versioning

**Module — Reusable Agent Object**
- Service: `POST /v1/agents` (name, description, model, system, tools, mcp_servers ≤20, skills ≤20 per session across agents, multiagent, metadata, tags, structured_output, style, collaborators, knowledge_base, timeout_seconds 120-900, restrictions, hidden, is_schedulable, context_access_enabled, context_variables, bundled_agent_id) → `{id, type, version, created_at, updated_at}`. *Providers*: Ant, Goo, IBM, Mst, OAI.
- Service: File-based agent (`~/.<platform>/agents/<name>.{md,toml,yaml}`, `.<platform>/agents/`; fields name/description/developer_instructions/prompt/model/model_reasoning_effort/sandbox_mode/mcp_servers/skills/tools/nickname_candidates/allowedSubagents/groups/fileRegex/permissionMode). *Providers*: Claude, Codex, Bob, Vibe.
- Service: Code-defined agent (`new Agent({...})` OAI; `AgentDefinition` Claude). *Providers*: OAI, Claude.

**Module — Versioning / Releases**
- Service: Every save produces new version; releases immutable snapshots deployable per environment; optimistic concurrency `version` field required on update mismatch → 409 (Ant); `precondition.content_sha256` for memories (Ant). *Providers*: Ant.
- Service: `POST /v1/agents/{id}/releases` (version_label, semantic_version, version_name, version_description); `POST /v1/agents/{id}/releases/{ver}/environment/{env_id}` deploy; `POST /v1/agents/{id}/releases/{ver}/undeploy`. *Providers*: IBM.
- Service: `version_label`/`semantic_version`. *Providers*: IBM.

**Module — Lifecycle Ops**
- Service: `GET /v1/agents/{id}` (retrieve, specific version via `?version=n`); `GET /v1/agents/{id}/versions` (paginated); `POST /v1/agents/{id}` update (version required, new version on change); `POST /v1/agents/{id}/archive` (one-way read-only); `DELETE /v1/agents/{id}`; `GET /v1/agents` (filters agent_id/order/limit/cursor/include_hidden). *Providers*: Ant, IBM.

**Module — AGENTS.md Layered Discovery**
- Service: Global `~/.codex/AGENTS[.override].md` → project root → cwd; concatenated root-down, later overrides earlier; `AGENTS.override.md` precedence; fallback filenames; max-bytes cap (32 KiB); config `project_doc_fallback_filenames[]`, `project_doc_max_bytes`. *Providers*: Codex, Goo; analogues Claude CLAUDE.md, Bob AGENTS.md, Vibe `.vibe/`.

**Module — Model Selection per Agent**
- Service: String ID or `{id, speed}` object (Ant); `provider/developer/model_id` (IBM); alias `opus`/`sonnet`/`haiku`/`fable`/`inherit` (Claude). *Providers*: per platform.

**Module — Default Toolset**
- Service: Flag enabling all built-in tools (`agent_toolset_20260401` Ant); explicit `tools[]`; mode-gated groups (Bob). *Providers*: Ant, Bob.

**Module — Structured Output (agent)**
- Service: JSON Schema enforced on responses (`outputType` OAI; `structured_output` IBM; `output_format` Ant; `--output-schema` Codex). *Providers*: OAI, IBM, Ant, Codex.

**Module — Metadata / Tags**
- Service: Arbitrary key-value merged at key level (`metadata` Ant); `tags`, `hidden` (IBM). *Providers*: Ant, IBM.

**Module — Description as Routing Metadata**
- Service: `description` drives supervisor/collaborator routing, not just human text (IBM, OAI `handoffDescription`, Codex `description`). *Providers*: IBM, OAI, Codex.

**Module — Agent Styles / Reasoning Modes**
- Service: `default`/`react`/`planner`/`custom`/`react_intrinsic`/`code_act`/`experimental_customer_care` (IBM); built-in personas `default`/`plan`/`accept-edits`/`auto-approve` (Vibe Code); `default`/`worker`/`explorer` (Codex); `Explore`/`Plan`/`general-purpose` (Claude). *Providers*: per platform.

**Module — Context Variables**
- Service: Platform-provided runtime values (`wxo_email_id`, `wxo_user_name`) referenced in instructions as `{var}`. *Providers*: IBM.

**Module — Agent Restrictions / Editability**
- Service: `editable`/`non_editable`/`custom` controlling collaborator editability. *Providers*: IBM.

**Module — Timeout**
- Service: Per-agent `timeout_seconds` min 120 max 900. *Providers*: IBM.

**Module — Bundled / Catalog Agents**
- Service: Derive from base/catalog agent (`bundled_agent_id`, `base_agent`, `create-from-template`). *Providers*: IBM, Goo, Codex.

## Domain L4.C — Models, Providers & Reasoning (agent-level)

### Product L4.C.1 — Model & Reasoning Config (agent)

**Module — Per-Turn / Run Override**
- Service: `RunConfig(model)` (OAI); `turn/start model` override (Codex); `agent_with_overrides.model` (Ant); `RunOrchestrate.llm_params` (IBM); `conversations.start(model)` (Mst). *Providers*: per platform.
- Service: Reasoning effort / thinking config (`model_reasoning_effort` minimal/none/low/medium/high/max/xhigh/ultra Codex; `effort` low/medium/high/xhigh/max Claude; `thinking` ThinkingConfig Claude; `generation_config.thinking_summaries`/`thinking_level` Goo; `modelSettings` reasoning effort OAI; `speed:"fast"` Ant). *Providers*: per platform.
- Service: Thinking events (`agent.thinking` Ant; `reasoning` items with summary+content Codex; `thought` steps with `thought_summary`+`thought_signature` deltas Goo; `thinking chunk` Vibe Workflows; `hide_reasoning` toggle IBM). *Providers*: per platform.
- Service: Model rerouting / safety buffering (`model/rerouted {fromModel,toModel,reason}`; `model/safetyBuffering/updated` with `fasterModel`). *Providers*: Codex.
- Service: Model catalog/list (`model/list` with `supportedReasoningEfforts[]`, `inputModalities`, `supportsPersonality`, `isDefault`, `upgrade` Codex; `/v1/models/list` + `/embeddings` IBM). *Providers*: Codex, IBM.
- Service: Model policies/governance (`/v1/model_policy` controls over which models may be used, public preview). *Providers*: IBM.
- Service: Provider capability bounds (`modelProvider/capabilities/read`). *Providers*: Codex.
- Service: Generation params (`temperature`, `top_p`, `max_tokens`, `n`, `seed`, `stop`, `frequency_penalty`, `presence_penalty`, `logit_bias` IBM, Mst `completion_args`). *Providers*: IBM, Mst.
- Service: Per-turn personality (`personality` override on `turn/start`). *Providers*: Codex.
- Service: Multi-provider gateway (OpenAI-compatible passthrough `/gateway/model/chat/completions`, `/embeddings` IBM; Anthropic via Bedrock/Foundry/GCP Claude). *Providers*: IBM, Claude.
- Service: Verification (`model/verification` additional account verification). *Providers*: Codex.

## Domain L4.D — Environments & Sandboxes

### Product L4.D.1 — Execution Environments (agent sandboxes)

**Module — Managed Cloud Sandbox**
- Service: Fresh Linux container per session from environment config (Ant `cloud`; Goo `remote`; Codex `Cloud`; OAI `DockerSandboxClient`). *Providers*: Ant, Goo, Codex, OAI.
- Service: Pre-installed packages cached across sessions sharing an environment; multiple managers in alphabetical order (apt, cargo, gem, go, npm, pip) with optional version pins (Ant); pre-installed runtimes (Goo: Python 3.12, Node 22, common UNIX tools). *Providers*: Ant, Goo.

**Module — Self-Hosted / Local Sandbox**
- Service: Agent runs on your machine/process; OS-level sandbox (Seatbelt/bubblewrap/seccomp/Windows sandbox). *Providers*: Ant `self_hosted`, Codex Local + OS sandbox, Bob `--sandbox`, Claude `sandbox` option, Vibe CLI.

**Module — Git Worktree Isolation**
- Service: Each background session gets its own worktree; commits/pushes a branch; opens a draft PR (Claude `.claude/worktrees/`; Codex `Worktree`). *Providers*: Claude, Codex.

**Module — Declarative Sources / Mounts**
- Service: `repository` (git clone), `gcs` (Cloud Storage), `inline` (text), local dir, GitRepo, S3/GCS/R2/AzureBlob/Box mounts (Goo `sources`; OAI `Manifest` entries; Codex `.worktreeinclude`). *Providers*: Goo, OAI, Codex.

**Module — Network Policy**
- Service: `unrestricted` | `limited` with `allowed_hosts` (Ant); `network.allowlist` with `transform` header injection via egress proxy (Goo); `network.domains` allow/deny + proxy + SOCKS5 + unix sockets (Codex Permission Profiles); `network:"disabled"` (Goo); `network_proxy` feature (Codex). *Providers*: Ant, Goo, Codex.
- Service: `allow_mcp_servers` bool (default false); `allow_package_managers` bool (default false). *Providers*: Ant.

**Module — Credentials Injection**
- Service: Vault env-var credentials substituted at egress (Ant); egress-proxy header `transform` never exposed inside sandbox (Goo); cloud secrets decrypted only for setup scripts, removed before agent phase (Codex). *Providers*: Ant, Goo, Codex.

**Module — Lifecycle States**
- Service: Created → Active → Idle (auto-snapshot after 15 min) → Offline (retained 7 days) → Deleted (Goo); archive/delete (Ant). *Providers*: Goo, Ant.

**Module — Resource Limits**
- Service: CPU/memory (Goo 4 cores/16 GB); agent `timeout_seconds` (IBM); `max_turns`/`max_budget_usd` (Claude). *Providers*: per platform.

**Module — Container Cache**
- Service: Up to 12h; clones default branch, runs setup, caches state; invalidated on script/env/secret change. *Providers*: Codex Cloud.

**Module — Setup Scripts**
- Service: Run on new worktree creation; platform-specific overrides; automatic (npm/pip/…) or manual custom script. *Providers*: Codex, OAI Manifest `environment`.

**Module — Filesystem Download / Snapshot**
- Service: `GET /files/environment-{id}:download` returns tar archive of full filesystem (Goo); `snapshot()` saves workspace to seed a fresh sandbox (OAI); checkpointing — git snapshot + conversation + tool-call record for rollback (Bob, Claude `enable_file_checkpointing` + `rewind_files`). *Providers*: Goo, OAI, Bob, Claude.

**Module — Permission Profiles**
- Service: Named least-privilege policy combining filesystem rules (`read`/`write`/`deny`) + network rules + inheritance via `extends` (Codex, beta); protected paths (`.git`, `.agents`, `.codex`). *Providers*: Codex.
- Service: Sandbox modes (simpler alternative) `sandbox_mode: "read-only"|"workspace-write"|"danger-full-access"` (Codex). *Providers*: Codex.

**Module — Sandbox Test Command**
- Service: `codex sandbox macos|linux|windows`. *Providers*: Codex.

**Module — Background Terminals / Process Spawn**
- Service: `thread/backgroundTerminals/*`, `process/spawn` outside the sandbox. *Providers*: Codex.

**Module — Antigravity IDE Harness Reuse**
- Service: Managed agent uses the same harness as the IDE. *Providers*: Goo.

### Product L4.D.2 — Environment CRUD

**Module — Environment API**
- Service: `POST /v1/environments` (name, config: type cloud/self_hosted/local/worktree, packages, sources, network, resources, setup_scripts, env, users, groups) → `{id}`. *Providers*: Ant-style unified.
- Service: `GET /v1/environments`, `GET /v1/environments/{id}`, `POST /v1/environments/{id}/archive`, `DELETE /v1/environments/{id}` (only if no sessions reference it). *Providers*: Ant-style.

## Domain L4.E — Sessions, Threads, Runs & Interactions

### Product L4.E.1 — Session Creation & Lifecycle

**Module — Creation**
- Service: `POST /v1/sessions` (agent string | {type:agent,id,version?} | {type:agent_with_overrides,id,model?,system?,tools?,mcp_servers?,skills?}, environment_id required, vault_ids, resources memory_store access, title, previous_session_id, background, store, context, context_variables, llm_params, guardrails) → `{id, status, agent resolved snapshot, environment_id}`. *Providers*: Ant.
- Service: `interactions.create` (Goo); `POST /v1/threads` + `POST /v1/orchestrate/runs` (IBM Thread=history, Run=execution); `conversations.start` (Mst); `Runner.run()` (OAI); `query()` (Claude); `thread/start` + `turn/start` (Codex); `bob -p`/`bob -i` (Bob). *Providers*: per platform.
- Service: Two-step lifecycle (create session provisions sandbox → send first message starts work). *Providers*: Ant.

**Module — Status State Machine**
- Service: `idle`/`running`/`rescheduling`/`terminated` (Ant); `in_progress`/`requires_action`/`completed`/`failed`/`cancelled` (Goo); `pending`/`running`/`completed`/`async_wait`/`async_completed`/`failed`/`cancelled`/`requires_input`/`expired` (IBM); `success`/`error_max_turns`/`error_max_budget_usd`/`error_during_execution`/`error_max_structured_output_retries` (Claude Result subtypes); `turn/started` → `completed`/`interrupted`/`failed` (Codex). *Providers*: per platform.

**Module — Agent Reference Forms**
- Service: Agent ID string (latest version); pinned `{id, version}`; `agent_with_overrides` (per-session model/system/tools/mcp/skills without versioning, Ant). *Providers*: Ant.
- Service: `agent` vs `model` mutually exclusive (Goo, Mst). *Providers*: Goo, Mst.
- Service: `agent_id` + optional `environment_id` + optional `version` pin (IBM). *Providers*: IBM.

**Module — Override Rules**
- Service: Omit → inherit; `null`/empty → clear (exceptions: `model` never clearable); value → full replacement. *Providers*: Ant.

**Module — Continue / Resume / Fork**
- Service: `continue_conversation` (find most recent in cwd, Claude); `resume: session_id` (specific, Claude/Codex/Ant); `fork_session` (new session, copied history, original unchanged, Claude/Codex `thread/fork` with `lastTurnId`); `previous_interaction_id` (Goo server-side state); `previousResponseId`/`conversationId` (OAI); `append` with new conversation ID (Mst, append-only immutable). *Providers*: per platform.

**Module — Multi-Turn**
- Service: Send new messages to existing session; multiple `turn/start` append to a thread. *Providers*: all.

**Module — Mid-Turn Steering**
- Service: `turn/steer` appends user input to active in-flight turn (Codex); `user.interrupt` + `user.message` redirects mid-execution (Ant). *Providers*: Codex, Ant.

**Module — Session Storage / Persistence**
- Service: JSONL on disk `~/.<platform>/projects/<encoded-cwd>/<session-id>.jsonl`; `SessionStore` adapter mirroring to S3/Redis/custom (Claude); SQLite rollouts `session-*.jsonl` (Codex). *Providers*: Claude, Codex.

**Module — Session Utility Functions**
- Service: `list_sessions`, `get_session_messages`, `get_session_info`, `rename_session`, `tag_session` (Claude); `thread/list` with filters (modelProviders, sourceKinds, archived, cwd, searchTerm, parentThreadId) (Codex). *Providers*: Claude, Codex.

**Module — Thread Operations**
- Service: name/set, goal/set, metadata/update, archive/unarchive/delete, unsubscribe, compact/start, shellCommand (outside sandbox), inject_items, rollback. *Providers*: Codex.

**Module — Two Independent State Dimensions**
- Service: Conversation history (`previous_interaction_id`) and sandbox/files (`environment_id`) decoupled and mixable. *Providers*: Goo (implicit in others).

**Module — Background / Async Mode**
- Service: `background: true` returns interaction ID immediately; poll/stream/cancel (Goo); background agents under a supervisor process (Claude `claude --bg`, Codex). *Providers*: Goo, Claude, Codex.

**Module — Goals**
- Service: Long-running task target with progress tracking, pause/resume/edit/clear; keeps shared context across turns. *Providers*: Codex `/goal`.

**Module — Access Scope**
- Service: Public Conversations API can read/modify only conversations owned by the API key creator. *Providers*: Mst.

**Module — Data Retention**
- Service: Paid Tier 55 days (configurable 7/14/28/55), Free Tier 1 day; `store=false` opts out (Goo); `store=False` (Mst). *Providers*: Goo, Mst.

**Module — List / Retrieve**
- Service: `GET /v1/sessions/{id}` includes status; `GET /v1/sessions` paginated with `limit`, `agent_id` filter, `order`, cursor pagination. *Providers*: Ant.

**Module — Session API (unified)**
- Service: `POST /v1/sessions/{id}/events` (send user.message to continue Ant); `POST /v1/sessions/{id}/resume` (Claude/Codex); `POST /v1/sessions/{id}/fork` (body last_turn_id → new session with copied history); `POST /v1/interactions` (with previous_interaction_id + environment_id Goo); `POST /v1/sessions/{id}/steer` (append user input to active turn Codex); `POST /v1/sessions/{id}/interrupt` (cancel mid-execution Ant); `POST /v1/sessions/{id}/goal`, `GET /v1/sessions/{id}/goal`, `POST /v1/sessions/{id}/goal/clear` (Codex); `GET /v1/sessions` (filters), `GET /v1/sessions/{id}`, `POST /v1/sessions/{id}/archive`, `DELETE /v1/sessions/{id}`, `POST /v1/sessions/{id}/name`, `POST /v1/sessions/{id}/metadata` (patch metadata gitInfo/tag/custom_title); thread-level `GET /v1/sessions/{id}/threads`, `GET /v1/sessions/{id}/threads/{tid}/stream`, `GET /v1/sessions/{id}/threads/{tid}/events`, `POST /v1/sessions/{id}/threads/{tid}/archive`; background `GET /v1/interactions/{id}` poll, `GET /v1/interactions/{id}?stream=true&last_event_id=` resumable stream, `POST /v1/interactions/{id}/cancel`. *Providers*: Ant-style unified.

## Domain L4.F — The Agent Loop, Turns, Streaming & Events

### Product L4.F.1 — Loop Execution

**Module — Server-Side Managed Loop**
- Service: Platform runs: provision sandbox → stream events → model picks tools → executes → pauses on `always_ask` → resumes on confirmation → `status_idle` when done. *Providers*: Ant, Goo, Mst, IBM.

**Module — In-Process Loop**
- Service: SDK runs the loop in your process via bundled native binary. *Providers*: Claude, Codex, OAI, Bob.

**Module — Loop Lifecycle**
- Service: Receive prompt → evaluate & respond → execute tools (read-only concurrent, state-modifying sequential) → repeat until no tool calls → return result. *Providers*: Claude.

**Module — Streaming via SSE**
- Service: `GET /v1/sessions/{id}/events/stream` (Ant); `stream: true` (Goo, Mst); `POST /runs?stream=true` (IBM); `async for message in query(...)` (Claude); `codex exec --json` JSONL (Codex). *Providers*: per platform.

**Module — Symmetric Step Model**
- Service: All content flows through `step.start` → `step.delta`(s) → `step.stop` (Goo); analogous `item/started` → `item/completed` (Codex); `event_start` → `event_delta` (Ant). *Providers*: per platform.

**Module — Event Type Catalog (union)**
- Service: User events (send) — `user.message`, `user.interrupt`, `user.custom_tool_result`, `user.tool_confirmation`, `user.define_outcome`, `user.tool_result` (self-hosted) (Ant). *Providers*: Ant.
- Service: Agent events (receive) — `agent.message`, `agent.thinking`, `agent.tool_use`, `agent.tool_result`, `agent.mcp_tool_use`, `agent.mcp_tool_result`, `agent.custom_tool_use`, `agent.thread_context_compacted`, `agent.thread_message_received`, `agent.thread_message_sent` (Ant); `message.output`, `tool.execution`, `function.call`, `agent.handoff` (Mst); `assistantMessage`, `userMessage`, `systemMessage`, `streamEvent`, `resultMessage` (Claude); `userMessage`, `agentMessage`, `plan`, `reasoning`, `commandExecution`, `fileChange`, `mcpToolCall`, `dynamicToolCall`, `collabToolCall`, `webSearch`, `imageView`, `contextCompaction`, `enteredReviewMode`/`exitedReviewMode` (Codex); `model_output`, `thought`, `function_call` + server-side tool steps (Goo); `RunEvent` envelope `{id,event,data}` (IBM). *Providers*: per platform.
- Service: Session events — `session.status_running`, `session.status_idle`, `session.status_rescheduled`, `session.status_terminated`, `session.deleted`, `session.updated`, `session.error`, `session.thread_created`, `session.thread_status_*` (Ant); `turn/started`, `turn/completed`, `turn/diff/updated`, `turn/plan/updated`, `thread/tokenUsage/updated` (Codex); `interaction.created`, `interaction.status_update`, `interaction.completed`, `error`, `done` (Goo). *Providers*: per platform.
- Service: Span events (observability) — `span.model_request_start`/`_end` with `model_usage`, `span.outcome_evaluation_start`/`_ongoing`/`_end` (Ant); `hook/started`/`hook/completed` (Codex); `model/rerouted`, `model/safetyBuffering/updated`, `model/verification` (Codex). *Providers*: Ant, Codex.
- Service: System events (send) — `system.message` (update system prompt between turns, Opus 4.8 only, Ant). *Providers*: Ant.

**Module — Deltas (stream-only, opt-in, never persisted)**
- Service: `event_start` + `event_delta` with `delta.type:"content_delta"` and `index` (Ant); `step.delta` types text/image/audio/thought_summary/thought_signature/arguments_delta/google_search_call/google_search_result (Goo); `item/agentMessage/delta`, `item/plan/delta`, `item/reasoning/summaryTextDelta`/`textDelta`, `item/commandExecution/outputDelta` (Codex); `StreamEvent` raw API streaming (Claude); `TextChunk`, `ToolReferenceChunk`, `ToolFileChunk`, `ReferenceChunk` (Mst/Vibe). *Providers*: per platform.

**Module — Item Lifecycle**
- Service: `item/started` (full item, `item.id` = delta `itemId`) → `item/completed` (authoritative final state) (Codex); reconcile per model request: `span.model_request_start` → `event_start` → `event_delta`s → buffered `agent.message` → `span.model_request_end` (Ant). *Providers*: Codex, Ant.

**Module — Stop Reasons**
- Service: `requires_action` with blocking `event_ids`; `end_turn` for interrupted blocked threads (Ant); Result subtypes (Claude). *Providers*: Ant, Claude.

**Module — Errors**
- Service: `session.error` with typed `error` object + `retry_status`: `mcp_connection_failed_error`, `mcp_authentication_failed_error`, `environment_archived_error`, `agent_archived_error`, `session_rate_limited_error` (Ant); `codexErrorInfo` variants `ContextWindowExceeded`, `UsageLimitExceeded`, `HttpConnectionFailed`, `BadRequest`, `Unauthorized`, `SandboxError`, `InternalServerError`, `Other` (Codex); `error` event with `message` + `code` e.g. `gateway_timeout` (Goo). *Providers*: per platform.

**Module — Sending / Listing Events**
- Service: `POST /v1/sessions/{id}/events` with `{"events":[...]}`; multiple per request; `GET /v1/sessions/{id}/events` paginated with `types[]` filter (Ant); `thread/items/list`, `thread/turns/list` with `itemsView` omit/summarize/fully-load (Codex). *Providers*: Ant, Codex.

**Module — Compaction / Context Window (agent loop)**
- Service: `agent.thread_context_compacted` events (Ant); `SystemMessage(subtype="compact_boundary")` + `PreCompact` hook + `/compact` (Claude); `thread/compact/start` + `contextCompaction` item (Codex); native compaction at ~135k tokens (Goo); `compaction_settings.context_compaction_threshold`/`compaction_sliding_window`/`large_message_threshold` (IBM); agent can override compaction prompt (Vibe Code). *Providers*: per platform.

**Module — Outcome Definition & Evaluation**
- Service: `user.define_outcome` + `span.outcome_evaluation_*` lifecycle. *Providers*: Ant.

**Module — Plan / Todos Streaming**
- Service: `turn/plan/updated` with steps `{step, status: pending/inProgress/completed}` (Codex); `update_todo_list` tool (Bob); `TaskCreate`/`TaskUpdate` tools (Claude); Todos panel (Vibe). *Providers*: per platform.

**Module — Token Usage**
- Service: `thread/tokenUsage/updated` (Codex); `interaction.usage` with `total_tokens`, modality breakdowns, `total_cached_tokens`, `total_thought_tokens`, `total_tool_use_tokens` (Goo); `ResultMessage.usage` + `total_cost_usd` + `num_turns` (Claude); `usage.connector_tokens` (Mst). *Providers*: per platform.

**Module — Resumable Streaming**
- Service: `last_event_id` cursor to resume after disconnect; each delta carries unique `event_id` (Goo). *Providers*: Goo.

**Module — Handling Unknown Events**
- Service: New event/delta types may be added; handle gracefully (log and skip). *Providers*: Goo versioning policy.

**Module — Unified Event API**
- Service: `GET /v1/sessions/{id}/events/stream` (SSE, query `event_deltas[]=agent.message&event_deltas[]=agent.thinking` opt-in deltas); `POST /v1/sessions/{id}/events` (body events array); `GET /v1/sessions/{id}/events` (query types[]/limit/cursor); unified event schema `{id, type, processed_at, data:{content/tool/input/delta/index/stop_reason/error/model_usage}}`; deltas `{type:event_start,event_type,event_id}`/`{type:event_delta,event_id,delta:{type:content_delta,index,text}}`; compaction `POST /v1/sessions/{id}/compact` (emits agent.thread_context_compacted | contextCompaction item | SystemMessage(compact_boundary)); token usage `{type:thread/tokenUsage/updated, usage:{total_tokens,...}}`; resumable `GET /v1/interactions/{id}?stream=true&last_event_id=<cursor>`. *Providers*: Ant-style unified.

## Domain L4.G — Tools (Built-in, Custom & Function Calling)

### Product L4.G.1 — Built-in Tools Catalog

**Module — File Ops Tools**
- Service: `read`/`Read`/`read_file`, `write`/`Write`/`write_file`, `edit`/`Edit`/`apply_diff`/`insert_content`/`search_and_replace`. *Providers*: Ant, Claude, Bob, Vibe.

**Module — Search Tools**
- Service: `glob`/`Glob`, `grep`/`Grep`, `list_files`, `GetSymbolsOverview`, `FindSymbol`, `FindReferencingSymbols`. *Providers*: Ant, Claude, Bob.

**Module — Shell Tools**
- Service: `bash`/`Bash`/`execute_command`/`Shell`/`commandExecution`. *Providers*: Ant, Claude, Codex, Bob, Vibe, OAI sandbox.

**Module — Web Tools**
- Service: `web_search`/`WebSearch`/`google_search`/`web_search_premium`, `web_fetch`/`WebFetch`/`url_context`, `webSearch` items with `action: search`/`openPage`/`findInPage`. *Providers*: Ant, Claude, Codex, Goo, Mst, Vibe.

**Module — Code Execution Tools**
- Service: `code_execution`/`code_interpreter` (isolated container). *Providers*: Goo, Mst, Vibe. (See Containers product.)

**Module — Image Generation Tool**
- Service: `image_generation` (Mistral, Vibe, Google `gemini-3-pro-image`, OpenAI Responses tool). *Providers*: Mst, Vibe, Goo, OAI.

**Module — Discovery Tools**
- Service: `ToolSearch` (large tool sets). *Providers*: Claude.

### Product L4.G.2 — Containers (execution sandbox as tool)

**Module — Container Management**
- Service: `POST /v1/containers` (name, memory_limit `"1g"`/`"4g"`/`"16g"`/`"64g"` OAI / fixed 5 GiB Ant, `expires_after` `{anchor:"last_active_at",minutes}`, skills OAI) → `{id}`. *Providers*: OAI, Ant (code-execution container).
- Service: `GET /v1/containers`, `DELETE /v1/containers/{id}` (any operation refreshes `last_active_at`); OpenAI code-interpreter containers expire after 20 min idle (data discarded); Anthropic idle ~5 min, 30-day cap. *Providers*: OAI, Ant.
- Service: Container files (OpenAI code interpreter) — create (multipart or `file_id`), list, retrieve content. *Providers*: OAI.
- Service: Container reuse via top-level `container` id (Ant); pass back `container` id when a programmatic call is in flight. *Providers*: Ant.

### Product L4.G.3 — Vector Stores / FileSearchStores (agent tool setup)

**Module — Vector Store Management**
- Service: `POST /v1/vector_stores` (OAI) / `POST /v1beta/fileSearchStores` (Goo) — create (name/display_name, embedding_model, file_ids, expires_after). *Providers*: OAI, Goo.
- Service: `GET`/`POST` (update name/expires_after)/`DELETE` (with force). *Providers*: OAI, Goo.
- Service: Add files — `POST /v1/vector_stores/{id}/files` (`file_id`, optional `attributes`, `chunking_strategy`) or upload stream (OAI); `uploadToFileSearchStore` (resumable) or `importFile` (existing file) (Goo); both return long-running operations. *Providers*: OAI, Goo.
- Service: Chunking strategy static (`max_chunk_size_tokens` OAI 100-4096 default 800, `chunk_overlap_tokens` ≥0 ≤ half max default 400; Goo `whiteSpaceConfig.maxTokensPerChunk`/`maxOverlapTokens`). *Providers*: OAI, Goo.
- Service: Attributes/custom_metadata (per-file key/value map OAI max 16 keys 256 chars each; Goo `{key,string_value|numeric_value}` array) for filtering. *Providers*: OAI, Goo.
- Service: Documents API `GET/DELETE /fileSearchStores/{store}/documents[/{doc}]` (Goo). *Providers*: Goo.
- Service: Media download `GET /v1/fileSearchStores/{store}/media/{BlobId}` (Goo). *Providers*: Goo.
- Service: Batch operations `POST /v1/vector_stores/{id}/file_batches` (up to 500 files, shared or per-file attributes/chunking), `GET`, `cancel` (OAI). *Providers*: OAI.

### Product L4.G.4 — Connectors / MCP Servers

**Module — Connector Registration**
- Service: `POST /v1/connectors` (Mst) — create (name unique within workspace ≤64 chars alnum/_/-, description, server/server_url, visibility private/shared_workspace/shared_org, icon_url, headers static HTTP, auth_data OAuth2 {client_id,client_secret}, system_prompt injected when connector tools used, connector_id OAI one of connector_dropbox/gmail/googlecalendar/googledrive/microsoftteams/outlookcalendar/outlookemail/sharepoint). *Providers*: Mst, OAI, Goo, Ant.
- Service: Operations — `get` (by id or name), `list` (paginated, filterable), `list_tools` (paginated, `refresh` to bypass cache, `pretty` for compact schema), `update` (requires UUID), `delete` (permanent; agents referencing it lose tools). *Providers*: Mst, OAI.
- Service: OAuth `get_auth_url` (Mst) returns `{auth_url, ttl}`; user grants in Studio UI; tokens not programmatically passable; configure OAuth redirect URI. *Providers*: Mst.
- Service: Connectors Debugger (Mst, preview) — pre-flight diagnostic against an MCP server URL (transport detection, tool discovery, auth) with per-step report (Likely cause, Suggested fix, Raw response, Copy as curl); non-persistent credentials. *Providers*: Mst.
- Service: Secure MCP Tunnel (OAI `openai/tunnel-client`) — exposes private/on-prem firewalled servers without public exposure. *Providers*: OAI.

**Module — MCP Tool in Request**
- Service: `type:"mcp"` (OAI) / `"connector"` (Mst) / `"mcp_server"` (Goo) / MCP connector (Ant); `connector_id`/`server_url`/`server`; `server_label`; `server_description`; `headers`; `allowed_tools` allowlist; `require_approval` `"always"`/`"never"`/`{"never":{"tool_names":[...]}}` (OAI); `authorization` OAuth token not stored/echoed (OAI); `defer_loading` load defs only when needed (OAI); `tool_configuration` `{include|exclude, requires_confirmation}` (Mst). *Providers*: per platform.

**Module — MCP Approvals**
- Service: `mcp_approval_request`/`mcp_approval_response` items (OAI); `requires_confirmation` + `tool_confirmations` (Mst); `safety_decision:require_confirmation` + `safety_acknowledgement:true` (Goo). *Providers*: OAI, Mst, Goo.

### Product L4.G.5 — Skills

**Module — Skill Management**
- Service: `POST /v1/skills` (OAI) — create (multipart: multiple `files[]` paths under one top-level folder OR single zip part; bundle must contain exactly one `SKILL.md` case-insensitive; front matter `name`, `description` per agentskills.io). *Providers*: OAI.
- Service: `POST /v1/skills/{skill_id}/versions` — new version (zip upload); `POST /v1/skills/{skill_id}` — set `default_version`; delete rules (cannot delete default version; deleting last version deletes skill; cascade). *Providers*: OAI.
- Service: Limits (zip ≤50 MB, ≤500 files/version, uncompressed file ≤25 MB). *Providers*: OAI.
- Service: SKILL.md filesystem-based packages (Claude, Bob, Codex, Goo, IBM Skill binding). *Providers*: Claude, Bob, Codex, Goo, IBM.

### Product L4.G.6 — Tool Search / Deferred Loading

**Module — Tool Search**
- Service: `tool_search_tool_regex_20251119` (Claude builds Python regex, `pattern` ≤200) / `tool_search_tool_bm25_20251119` (NL `query` ≤500, BM25 ranking) (Ant); `defer_loading:true` on tools; ≥1 tool (usually search tool) must stay non-deferred; never defer search tool; keep 3-5 frequent tools non-deferred; max 10,000 deferred tools; ≤5 results/search; `server_tool_use` → `tool_search_tool_result` with `tool_reference` blocks API auto-expands; never return `tool_result` for search id; custom client-side impl return `tool_reference` blocks from custom tool's `tool_result`; not separately metered. *Providers*: Ant.
- Service: `{"type":"tool_search"[, execution:"client", description, parameters]}` (OAI); `defer_loading:true` on functions/MCP tools; namespaces group tools; hosted mode returns `tool_search_call` `{paths:[...]}` + `tool_search_output` `tools[]`; client mode emits `tool_search_call` you fulfill; discovered tools appended at end of context to preserve cache; `additional_tools` input item injects tools mid-conversation; only GPT-5.4+; use `parallel_tool_calls=False`. *Providers*: OAI.

### Product L4.G.7 — Programmatic Tool Calling

**Module — Programmatic Calling**
- Service: Programmatic tool calling (model writes code Python in Ant container, or JS in OAI V8 runtime that calls tools as functions within a single execution; execution pauses on each tool call, app returns result, code resumes; intermediate results never enter model context; `allowed_callers:["programmatic"]` on tools). *Providers*: Ant, OAI.

### Product L4.G.8 — Advisor (model consulting model)

**Module — Advisor Tool**
- Service: `advisor_20260301` (beta header `advisor-tool-2026-03-01`); faster executor model consults higher-intelligence advisor mid-generation; params `model` (advisor id), `max_uses`, `max_tokens` (min 1024), `caching:{type:"ephemeral",ttl:"5m"|"1h"}`; `server_tool_use` (empty input) → `advisor_tool_result` (`advisor_result` text or `advisor_redacted_result` encrypted); billed at advisor rates (`usage.iterations[]`); advisor must be ≥ Sonnet 4.6 and ≥ executor; cannot combine with extended thinking; forcing via `tool_choice:tool` works; resume `pause_turn` by resending paused content; Claude API + AWS only. *Providers*: Ant.

### Product L4.G.9 — Memory Tool

**Module — Memory Tool**
- Service: `memory_20250818` (`name:"memory"`, client, schema-less); store/retrieve info across conversations in `/memories` (your handler maps to real storage); commands `view`/`create`/`str_replace`/`insert`/`delete`/`rename` (paths within `/memories`); auto system-prompt "MEMORY PROTOCOL" (view dir first); pairs with context editing; validate path traversal (`../`, `%2e%2e%2f`); cap sizes; delete stale files; ZDR-eligible. *Providers*: Ant.

## Domain L4.H — Permissions, Approvals & Human Review

### Product L4.H.1 — Permission Policies

**Module — Permission Policy**
- Service: `permission_policy` (`always_allow`/`always_ask`) (Ant); auto-approve toggles (Bob); `permission_mode` (`default`/`acceptEdits`/`plan`/`dontAsk`/`auto`/`bypassPermissions`) (Claude); `sandbox_mode` + `approval_policy` (Codex); `ToolPermission` (IBM); `requires_confirmation` (Mst); `needsApproval`/guardrails (OAI); agent mode (`default`/`plan`/`accept-edits`/`auto-approve`) (Vibe). *Providers*: per platform.

### Product L4.H.2 — Approval Pause & Resume

**Module — Approval Pause**
- Service: `requires_action` + `user.tool_confirmation` (Ant); interactive approve (Bob); `canUseTool` callback / interruption (Claude); `requestApproval` + decision (Codex); `requires_action` + `function_result` (Goo); `requires_input` status (IBM); `confirmation_status:pending` + `tool_confirmations` (Mst); `result.interruptions` + `state.approve/reject` (OAI); `Continue`/`Always allow`/`Decline` (Vibe). *Providers*: per platform.

**Module — Auto-Review**
- Service: `auto` mode (Claude); `auto_review` (Codex). *Providers*: Claude, Codex.

**Module — Resumable Approval State (OpenAI)**
- Service: `needs_approval=True`/`needsApproval:true` at tool definition; run pauses; `result.interruptions` + resumable `result.state`; `state.approve(interruption)` then resume `await run(agent, state)`/`await Runner.run(agent, state)`; works across handoffs and nested `agent.asTool()` calls; serialize `state` and resume later for delayed review; same pattern for streamed runs. *Providers*: OAI.

**Module — Permission Events (Anthropic)**
- Service: `tool_decision` log events, `claude_code.tool.blocked_on_user` spans (`decision` accept/reject, `source`), `permission_mode_changed` events; hook spans capture lifecycle. *Providers*: Ant.

**Module — MCP require_approval**
- Service: `tool_config.require_approval` policy e.g. `"never"` (OAI hosted MCP). *Providers*: OAI.

## Domain L4.I — Hooks & Lifecycle Callbacks

### Product L4.I.1 — Hooks

**Module — Hook Registration**
- Service: Hooks (PreToolUse, PostToolUse, PreCompact, …) (Claude `claude_code.hook` span, detailed beta tracing requires `ENABLE_BETA_TRACING_DETAILED=1` + `BETA_TRACING_ENDPOINT`); `hook/started`/`hook/completed` events (Codex); `plugins`/async callbacks (IBM); `RunHooks`/`AgentHooks` (OAI). *Providers*: Claude, Codex, IBM, OAI.
- Service: Hook attributes `hook_event`, `hook_name`, `num_hooks`, `hook_definitions` (gated), `duration_ms`, `num_success`, `num_blocking`, `num_non_blocking_error`, `num_cancelled`. *Providers*: Claude.

## Domain L4.J — Credentials, Secrets & Vaults

### Product L4.J.1 — Vault / Connection / Credential

**Module — Credential Store**
- Service: Vault + Credential (Ant); API Key (Bob); env vars / MCP headers (Claude); env / secrets / `.worktreeinclude` (Codex); network `transform` egress proxy (Goo); Connection (IBM); Connector `headers`/`auth_data` (Mst); sandbox `environment` (OAI); Connector auth (Vibe). *Providers*: per platform.
- Service: Vault env-var credentials substituted at egress (Ant); egress-proxy header `transform` never exposed inside sandbox (Goo); cloud secrets decrypted only for setup scripts removed before agent phase (Codex). *Providers*: Ant, Goo, Codex.
- Service: Webhooks for vaults (Ant). *Providers*: Ant.

## Domain L4.K — Multi-Agent Orchestration

### Product L4.K.1 — Subagents / Collaborators / Teams

**Module — Subagent / Collaborator**
- Service: Roster entry / `self` (Ant); Subagent `spawn_subagent` / Subtask (Bob); Subagent `Agent` tool (Claude); `collabToolCall` (Codex); — (Goo); Collaborator (IBM); handoff target (Mst/OAI/Vibe). *Providers*: per platform.
- Service: Team / direct messaging — threads (Ant); — (Bob); Agent team `SendMessage` (Claude); CSV fan-out workers (Codex); — (Goo); collaborators + flows (IBM); — (Mst); — (OAI); — (Vibe). *Providers*: per platform.

**Module — Handoff**
- Service: Delegation via threads (Ant); — (Bob); Agent tool delegation (Claude); `collabToolCall` (Codex); — (Goo); collaborator delegation (IBM); Handoff `handoffs[]` (Mst); Handoff / agents-as-tools (OAI); Handoff `handoffs[]` (Vibe). *Providers*: per platform.
- Service: `handoff_execution:"server"` (default) / `"client"` (Mst); unlimited chaining. *Providers*: Mst.

## Domain L4.L — Memory & Knowledge (RAG)

### Product L4.L.1 — Memory Store / Library / Knowledge Base

**Module — Semantic Memory**
- Service: Memory Store (Ant); — (Bob); auto-memory / CLAUDE.md (Claude); — (Codex); — (Goo); `memory_enabled` / `client.memory.*` (IBM); — (Mst); `Memory()` capability (OAI); Chat Memories (Vibe). *Providers*: per platform.

**Module — RAG / Knowledge**
- Service: Memory Store (Ant); — (Bob); CLAUDE.md / auto-memory (Claude); AGENTS.md / web search cache (Codex); sandbox files / google_search (Goo); Knowledge base / Document collection / Vector index (IBM); Library (Mst); file search / `Memory()` (OAI); Library (Vibe). *Providers*: per platform.
- Service: `POST /v1/sessions` resources `[{type:"memory_store", memory_store_id, access:"read_write"|"read_only", instructions?}]`. *Providers*: Ant.

## Domain L4.M — Workflows, Scheduled Tasks & Automation

### Product L4.M.1 — Workflow / Flow

**Module — Workflow**
- Service: — (Ant); Workflow tool (Bob); Dynamic workflow JS (Claude); Goal mode (Codex); — (Goo); Flow `@flow` (IBM); — (Mst); chained voice pipeline (OAI); Workflow Studio (Vibe). *Providers*: per platform.

### Product L4.M.2 — Scheduled Task / Deployment / Routine

**Module — Scheduled Task**
- Service: Deployment cron (Ant); external CI (Bob); Routine / scheduled task (Claude); Scheduled task / Automations (Codex); — (Goo); `is_schedulable` UI-scheduled (IBM); — (Mst); — (OAI); Scheduled Task (Vibe). *Providers*: per platform.

## Domain L4.N — Channels, Voice & Embedded Chat

### Product L4.N.1 — Channels

**Module — Delivery Surfaces**
- Service: — (Ant); IDE/CLI only (Bob); Surfaces CLI/IDE/web/Slack/Chrome (Claude); GitHub/Linear/Slack triggers (Codex); — (Goo); Channels Slack/Teams/Twilio/phone/web (IBM); — (Mst); Realtime API / ChatKit (OAI); web/CLI/VS Code/mobile (Vibe). *Providers*: per platform.

### Product L4.N.2 — Voice (agent channel)

**Module — Voice Configuration**
- Service: — (Ant/Claude/Codex/Goo); TTS/audio models (Goo); Voice configuration / RealtimeAgentSettings (IBM); — (Mst); RealtimeAgent / VoicePipeline (OAI); — (Vibe). *Providers*: per platform.

## Domain L4.O — Extensions, Plugins, Marketplaces & Interoperability

### Product L4.O.1 — Plugin / Marketplace

**Module — Plugin**
- Service: — (Ant); — (Bob); Plugin bundled skills/agents/hooks/MCP (Claude); Plugin / App / marketplace (Codex); — (Goo); Catalog (IBM); — (Mst); — (OAI); — (Vibe). *Providers*: Claude, Codex, IBM.

---

# LAYER 5 — GOVERNANCE · SAFETY · OPERATIONS

> **Purpose.** The supervisory layer that wraps every other layer: who can call what (identity/RBAC), where data lives (residency/encryption), how traffic reaches the platform (network), how much it costs (billing/quotas), what is logged/audited (traces/logs), what is allowed (moderation/guardrails), what needs human sign-off (approvals), and how quality is measured (evaluation/datasets). This layer does not sit below or above the others in the call graph — it governs them. *Supervises*: L1–L4.
> **Sources:** `admin/summary.md`, `observability/summary.md`, `tools/summary.md` (safety/HITL), `agents/summary.md` (permissions/observability), `text/summary.md` (moderation/ZDR), `images/summary.md` (moderation), `voice/summary.md` (privacy).

## Domain L5.A — Account & Tenant Setup

### Product L5.A.1 — Organization / Workspace / Project

**Module — Account/Tenant**
- Service: Organization (OAI/Ant/Mistral) / Cloud organization (Goo) — top-level billing & identity container. *Providers*: OAI, Ant, Goo, Mst.
- Service: Enterprise Account / Backoffice (Mst) / parent organization + linked organizations (Ant) — grouping above organization for stronger isolation. *Providers*: Mst, Ant.
- Service: Workspace / Project — sub-tenant boundary scoping keys/files/rate limits/spend/data residency (OAI `project`; Ant `workspace`; Goo `Cloud project`; Mst `workspace`). *Providers*: all.
- Service: Default workspace (non-archivable; Mst default Workspace; OAI default project). *Providers*: Mst, OAI.
- Service: Workspace limits (Mst up to 500 active workspaces/org; names unique within org). *Providers*: Mst.

**Module — Workspace Management API**
- Service: `POST /v1/organizations/workspaces` (Ant create); `GET /v1/organizations/workspaces` (list); `POST /v1/organizations/workspaces/{id}` (update); archive/delete. *Providers*: Ant.
- Service: `POST /api/admin/workspaces` (Mst create); `GET /api/admin/workspaces` (list); `PATCH /api/admin/workspaces/{id}` (update); `DELETE /api/admin/workspaces/{id}`. *Providers*: Mst.
- Service: Member management `POST /api/admin/workspaces/{id}/add-users`, `PATCH .../users`, `DELETE .../users` (Mst); Workspace Members endpoints (Ant); invites (OAI); Cloud IAM (Goo). *Providers*: per platform.

## Domain L5.B — Identity, Authentication & Key Management

### Product L5.B.1 — API Keys & Federation

**Module — API Key**
- Service: Static API key (long-lived secret in header). *Providers*: all.
- Service: Admin API key (`OPENAI_ADMIN_KEY` OAI; `sk-ant-admin01-...` Ant; Admin API key `x-api-key` Mst). *Providers*: OAI, Ant, Mst.
- Service: Service account (`svac_...` Ant; service account within project OAI; Cloud IAM service account Goo). *Providers*: OAI, Ant, Goo.
- Service: Key expiration `expires_at` (Ant, Mst). *Providers*: Ant, Mst.
- Service: Key restriction by request origin IP/website referrer/application (Goo); restrict by API product (Goo Gemini only). *Providers*: Goo.
- Service: Safety identifiers privacy-preserving hashed user/session (`safety_identifier`, `OpenAI-Safety-Identifier` header OAI). *Providers*: OAI.
- Service: Domain verification (DNS TXT before SSO Mst). *Providers*: Mst.
- Service: SAML SSO org-level single sign-on through IdP (Mst). *Providers*: Mst.
- Service: SCIM sync group membership from IdP (OAI; planned Mst). *Providers*: OAI, Mst.
- Service: Customer-Managed Encryption Keys CMEK / Enterprise Key Management EKM (Ant CMEK; OAI EKM; AWS KMS/GCP/Azure Key Vault). *Providers*: Ant, OAI.
- Service: Access Transparency (log of every access to your data by staff/systems Ant). *Providers*: Ant.

**Module — Federation / Workload Identity**
- Service: Workload Identity Federation WIF (short-lived bearer token exchanged from IdP-issued OIDC JWT; eliminates long-lived keys in production). *Providers*: OAI, Ant, Goo.
- Service: OAuth 2.0 / user credentials (OAuth Client IDs Goo; OAuth 2.1 OAI for ChatGPT apps). *Providers*: Goo, OAI.
- Service: Ephemeral tokens short-lived scoped for client-to-server (browser/mobile → API) — Single Use Token (11L), Ephemeral Client Secret `ek_...` (OAI), Ephemeral Token `access_token` (Goo), Access Token JWT (Cart), Temporary API Token JWT (Deep), Browser Token `?token=` (Grad). *Providers*: per voice/admin platform.
- Service: Token configuration (Goo most advanced) — `uses` number of sessions (0=unlimited), `expire_time` max 20 hours, `new_session_expire_time`, `field_mask` controls which setup fields token overrides, `live_connect_constraints` lock model and configuration, `safety_identifier` bind privacy-preserving user ID (OAI). *Providers*: Goo, OAI.

**Module — Key Management API**
- Service: `POST /v1/organizations/api_keys` (Ant create); `GET /v1/organizations/api_keys` (list); `DELETE /v1/organizations/api_keys/{id}` (revoke). *Providers*: Ant.
- Service: `POST /v1/organizations/service_accounts` (Ant create). *Providers*: Ant.
- Service: `POST/GET /v1/organizations/federation_issuers`, `federation_rules` (Ant); Cloud Workload Identity (Goo); token exchange OAuth 2.0 jwt-bearer grant (OAI `/api/reference/workload-identity-federation`, Ant, Goo Cloud STS). *Providers*: Ant, Goo, OAI.
- Service: `GET /v1/compliance/access_events` (Ant Access Transparency query). *Providers*: Ant.

## Domain L5.C — Access Control, RBAC & Groups

### Product L5.C.1 — Roles & Permissions

**Module — Roles**
- Service: Roles bundles of permissions assigned at org and/or workspace scope; preset vs custom (OAI preset org owner/project owner/member/viewer + custom; Mst predefined org + workspace roles; Ant scopes Compliance key scopes/Admin key scopes; Goo Cloud IAM roles). *Providers*: per platform.
- Service: Permission catalogs — Models/inference Request (OAI) / model access per workspace (Ant) / `aiplatform.user` (Goo) / `user`,`dev`,`workspace_contributor` (Mst); Files/vector stores Read/Write; Fine-tuning Read/Write; Admin/org management Read/Write; Compliance `read:compliance_*`/`delete:compliance_user_data`; Tunnels Read/Use/Manage (OAI); Observability `observability_viewer` (Mst). *Providers*: per platform.
- Service: Groups collections of users assignable to roles syncable from IdP via SCIM (OAI groups; Mst User Groups). *Providers*: OAI, Mst.
- Service: Key permissions API keys carry own permissions (`api.model.read`) intersect with user's role (OAI). *Providers*: OAI.
- Service: Union evaluation effective permissions = union of all org + project roles direct and via groups (OAI); propagation delay up to 30 minutes (OAI). *Providers*: OAI.

**Module — RBAC API**
- Service: `GET /api/admin/roles` (Mst list); `GET /api/admin/user-groups` (Mst groups); `POST /v1/organization/invites` (OAI invite); `POST /api/admin/users/invite` (Mst invite); `GET/POST/PATCH/DELETE /api/admin/users[/{id}]` (Mst users). *Providers*: per platform.

## Domain L5.D — Tenant Isolation, Data Residency & Encryption

### Product L5.D.1 — Data Residency & Retention

**Module — Data Residency**
- Service: Project-level residency with regional host prefixes (`eu.api.openai.com` etc. OAI); workspace geo `default_inference_geo`/`allowed_inference_geos` + per-request `inference_geo` `"global"`|`"us"` (Ant); Cloud project region Developer API / Vertex region Enterprise (Goo); (n/a Mst). *Providers*: OAI, Ant, Goo.
- Service: Regional storage vs regional processing distinct (some regions support only storage OAI: Australia/Canada/Japan). *Providers*: OAI.

**Module — Zero Data Retention (ZDR)**
- Service: ZDR customer content not stored after response returns (OAI ZDR forces `store=false`; Ant ZDR arrangement for eligible features Files API and Batches not; Goo paid tier content not used for training). *Providers*: OAI, Ant, Goo.
- Service: Modified Abuse Monitoring MAM excludes customer content from abuse monitoring logs while preserving capabilities (OAI). *Providers*: OAI.
- Service: Application State data persisted by some API features to fulfill a task (OAI). *Providers*: OAI.

**Module — store Parameter**
- Service: `store` boolean controlling whether response/application state is retained (OAI `store`; Goo `store` on generateContent/Interactions; Mst not exposed as request param). *Providers*: OAI, Goo.

**Module — Encryption at Rest**
- Service: CMEK / EKM encrypt at rest with own KMS key (Ant CMEK requires cross-account key policy; OAI EKM lists eligible KMS providers). *Providers*: Ant, OAI.

**Module — Residency / Retention API**
- Service: `PATCH /v1/organization/projects/{id}/data_retention` (OAI set policy); workspace privacy controls (Ant); privacy controls in Admin Panel (Mst). *Providers*: OAI, Ant, Mst.
- Service: ZDR eligibility per endpoint documented table (OAI); feature-level eligibility (Ant). *Providers*: OAI, Ant.

## Domain L5.E — Network Security & Connectivity

### Product L5.E.1 — Private Connectivity

**Module — Private Link**
- Service: Azure Private Link reach regional endpoints (OAI; Azure-specific); Private Service Connect (Goo/Vertex). *Providers*: OAI, Goo.
- Service: IP egress ranges / outbound IP allowlists (published IP ranges OAI). *Providers*: OAI.
- Service: mTLS mutual TLS authenticate platform as MCP client or tunnel control plane (OAI). *Providers*: OAI.
- Service: Secure MCP Tunnels outbound-only HTTPS from inside your network to platform-hosted MCP endpoint; lets platform products reach private/on-prem MCP server without public ingress (OAI). *Providers*: OAI.
- Service: Environment networking controls `unrestricted` (default full outbound minus safety blocklist) vs `limited` (restrict to `allowed_hosts`) for managed agent sandboxes (Ant). *Providers*: Ant.

**Module — Network API**
- Service: `GET /v2/privatelink_healthcheck` (OAI regional health check); `https://<region>.privatelink.api.openai.com/v1/*` (API surface via Private Link); IP egress JSON manifest `https://openai.com/chatgpt-connectors.json` (OAI); Tunnel control plane `HTTPS https://api.openai.com:443/v1/tunnel/*` or `mtls.api.openai.com:443` (OAI); Tunnel RBAC permissions Tunnels Read/Use/Manage (OAI); Tunnel audit events `tunnel.created/updated/deleted` (OAI). *Providers*: OAI.

## Domain L5.F — Cost, Subscription, Spend & Billing Management

### Product L5.F.1 — Usage & Cost APIs

**Module — Usage/Cost Reports**
- Service: Usage dashboard/export (OAI); `GET /v1/organizations/usage_report` (Ant); billing/usage dashboards (Goo); `GET /api/admin/usage` (Mst). *Providers*: per platform.
- Service: `GET /v1/organizations/cost_report` (Ant; excludes Priority Tier different billing model). *Providers*: Ant.
- Service: `GET /v1/organizations/claude_code/analytics` (Ant); `/analytics/vibe` (Mst). *Providers*: Ant, Mst.

### Product L5.F.2 — Spend Limits

**Module — Spend Limits**
- Service: Spend alert `POST /v1/organization/projects/{id}/spend_alerts` (`threshold_amount` cents, `currency`, `interval` month, `notification_channel` {type:email,recipients,subject_prefix}) (OAI). *Providers*: OAI.
- Service: Spend limits API with `(scope, period)` pairs + increase requests (Ant `GET /v1/organizations/spend_limits/effective`; `POST .../spend_limit_increase_requests/{id}/approve|deny`). *Providers*: Ant.
- Service: Project spend caps + billing-account tier spend caps (Goo). *Providers*: Goo.
- Service: Organization + Workspace spending limits `POST /api/admin/spend-limit` / `GET /api/admin/spend-limit` (Mst). *Providers*: Mst.

### Product L5.F.3 — Billing Plans & Subscriptions

**Module — Billing Plans**
- Service: Prepay (purchase credits in advance) vs Postpay (accrue charged monthly) (Goo); credits expire after 12 months; non-refundable except when switching to Postpay. *Providers*: Goo.
- Service: Subscriptions product-surface plans (Mst Vibe plans, Mistral Code/Vibe Code seats, API Plan Free/Scale; OAI usage tiers; Ant usage tiers + Priority Tier; Goo Free/Paid/Enterprise). *Providers*: per platform.
- Service: Priority Tier committed capacity tier with burndown rates (Ant, OAI). *Providers*: Ant, OAI.
- Service: Pricing model per-token input/output with feature multipliers caching/batch/data residency; media models per-image/per-second/per-song; session runtime per hour Managed Agents. *Providers*: per platform.
- Service: Monetary values strings in minor units cents for USD to avoid floating-point errors (Ant, Mst). *Providers*: Ant, Mst.

## Domain L5.G — Quotas, Rate Limits & Usage Tiers

### Product L5.G.1 — Rate Limits

**Module — Rate Limit Dimensions**
- Service: RPM requests/min, RPD requests/day, TPM tokens/min input, OTPM/OTPM output tokens/min, TPD tokens/day, IPM images/min, audio minutes/min; limit hit = whichever metric reached first. *Providers*: per platform.
- Service: Scope per org/per project/per workspace/per model (Goo per project not per key; OAI org + project; Ant org + workspace overrides inherit org value when absent; Mst per organization). *Providers*: per platform.

**Module — Usage Tiers**
- Service: Auto-graduated by cumulative paid spend; higher tiers → higher limits (OAI Free → Tier 1-5; Ant Start/Build/Scale; Goo Free → Tier 1-3; Mst Free/Scale). *Providers*: per platform.
- Service: Long-context limits separate rate limit for long-context requests (OAI). *Providers*: OAI.
- Service: Shared limits some model families share a single TPM pool (OAI). *Providers*: OAI.
- Service: Batch API queue limits total input tokens queued per model (OAI; Ant max 200k batch requests in queue; Goo enqueued tokens per tier/model; Mst batch does not count against real-time limits). *Providers*: per platform.
- Service: Vector store ingestion limit 300 RPM per vector store ID (OAI). *Providers*: OAI.
- Service: Spend-based rate limits rolling 10-minute window spend cap (Goo). *Providers*: Goo.
- Service: Acceleration limits sharp usage spikes trigger rejection even within steady limits (Ant). *Providers*: Ant.
- Service: Cache-aware ITPM cache-read tokens counted differently for ITPM budgeting (Ant). *Providers*: Ant.
- Service: Exponential backoff with jitter retry strategy for 429s; failed requests still count against per-minute limit. *Providers*: all.

**Module — Rate Limit Response Headers**
- Service: `x-ratelimit-limit-requests`, `x-ratelimit-remaining-requests`, `x-ratelimit-reset-requests`, `x-ratelimit-limit-tokens`/`-project-tokens` (OAI); `anthropic-ratelimit-requests-limit/remaining/reset`, `anthropic-ratelimit-input-tokens-*`/`-output-tokens-*`, `anthropic-priority-input-tokens-limit/remaining/reset`, `retry-after` (Ant); `X-RateLimit-Remaining` (Mst); `x-gemini-service-tier` actual tier used priority/standard (Goo). *Providers*: per platform.

**Module — Rate Limit API**
- Service: `GET /v1/fine_tuning/model_limits` (OAI); `GET /v1/organizations/rate_limits`, `.../workspaces/{id}/rate_limits` (Ant); rate-limit URL (Goo); `GET /api/admin/rate-limit` (Mst). *Providers*: per platform.

## Domain L5.H — Model Lifecycle & Permissions

### Product L5.H.1 — Model Selection & Permissions

**Module — Model Lifecycle Stages**
- Service: Experimental / Preview / Stable GA (Goo); dated snapshots + `-latest` aliases (OAI, Mst); deprecation windows differ. *Providers*: per platform.

**Module — Model Allowlist/Denylist**
- Service: Project model permissions `mode:"allow_list"|"deny_list"`, `model_ids` (OAI `PATCH /v1/organization/projects/{id}/model_permissions`). *Providers*: OAI.

**Module — Reasoning / Verbosity / Phase**
- Service: `reasoning.effort` none/low/medium/high/xhigh (OAI); `text.verbosity` low/medium/high (OAI); `phase` labels assistant messages `commentary`/`final_answer` (OAI); `tool_search`/`defer_loading` deferred tool loading (OAI); `reasoning.encrypted_content` stateless handoff of reasoning items for ZDR (OAI). *Providers*: OAI.

## Domain L5.I — Processing Tiers & Cost Optimization

### Product L5.I.1 — Service Tiers

**Module — Processing Tiers**
- Service: Standard baseline 1.0× price high reliability seconds-to-minutes latency. *Providers*: all.
- Service: Batch asynchronous ~0.5× price up to 24h turnaround separate rate-limit pool (OAI, Ant, Goo, Mst all 50% discount). *Providers*: all.
- Service: Flex ~0.5× price best-effort "sheddable" capacity minutes latency target synchronous may be preempted (OAI, Goo). *Providers*: OAI, Goo.
- Service: Priority ~1.8× price Google / uplift OAI guaranteed throughput non-sheddable lower and more consistent latency (OAI, Ant, Goo). *Providers*: OAI, Ant, Goo.
- Service: `service_tier` request parameter request-level opt-in to a tier; response echoes tier actually used (OAI `"flex"|"priority"|"auto"|"default"`; Ant `"auto"|"standard_only"`; Goo `"flex"|"priority"|"standard"`). *Providers*: per platform.
- Service: Ramp rate limit Priority if traffic ramps too quickly Priority may downgrade to Standard (OAI ≥1M TPM and >50% TPM increase in 15 min; Goo graceful downgrade billed at Standard; Ant drawn against both Priority and regular limits). *Providers*: per platform.
- Service: Regional processing uplift 10% uplift for models released on/after March 5 2026 eligible for data residency (OAI); 1.1× for US-only inference on Opus 4.6/Sonnet 4.6+ (Ant). *Providers*: OAI, Ant.

**Module — Usage Response Object**
- Service: `usage.prompt_tokens`/`input_tokens`, `usage.completion_tokens`/`output_tokens`, `usage.total_tokens`, `usage.prompt_tokens_details.cached_tokens`, `usage.cache_read_input_tokens`/`cache_creation_input_tokens`, `usage.completion_tokens_details.reasoning_tokens`, `usage.completion_tokens_details.accepted_prediction_tokens`/`rejected_prediction_tokens`, `usage.service_tier`, `usage.total_cached_tokens`. *Providers*: per platform.

## Domain L5.J — Prompt Caching & Context Management (governance view)

### Product L5.J.1 — Prompt Caching (admin)

> The technical caching mechanisms live in L2.G. Here we record the governance/billing aspects.

**Module — Cache Retention Policies**
- Service: `prompt_cache_retention` `in_memory`/`24h` cache policy (OAI); `cache_control` `{type:ephemeral}` + `ttl:"5m"|"1h"` (Ant). *Providers*: OAI, Ant.

**Module — Cache Diagnostics**
- Service: Per-request fingerprint comparison reveals `cache_miss_reason` (Ant beta ZDR-eligible). *Providers*: Ant.

### Product L5.J.2 — Token Counting (admin)

**Module — Token Counting API**
- Service: `POST /v1/responses/input_tokens` (OAI model-exact handles images/files/tools); `POST /v1/messages/count_tokens` (Ant same params as Messages supports `context_management` returns `input_tokens` after editing + `original_input_tokens` free but RPM-limited); `GenerativeModel.count_tokens` (Goo not billed); token budgeting guidance (Mst). *Providers*: OAI, Ant, Goo, Mst.

## Domain L5.K — Safety, Guardrails, Moderation & Approvals

### Product L5.K.1 — Moderation API

**Module — Moderation API**
- Service: `POST /v1/moderations` (OAI free-to-use reduce unsafe content). *Providers*: OAI.
- Service: `POST /v1/moderations` raw text + `POST /v1/chat/moderations` conversational (Mst `mistral-moderation-2603`; scores per category; `mistral-moderation-2411` deprecated March 31 2026). *Providers*: Mst.
- Service: `safety_settings`/`safetySettings` array `[{category, threshold}]` (Goo). *Providers*: Goo.

**Module — Moderation in Generation Request**
- Service: `moderation` auto/low; `moderation_details.categories` (OAI). *Providers*: OAI.
- Service: `moderation_stage: input|output|unknown` (OAI). *Providers*: OAI.

### Product L5.K.2 — Guardrails

**Module — Guardrail Engine**
- Service: Input guardrail fast validation step before expensive/side-effecting part; separate guardrail agent classifies input with structured output schema returning whether `tripwire_triggered`; raises `InputGuardrailTripwireTriggered` (OAI `Agent.input_guardrails`/`inputGuardrails` `@input_guardrail` decorator). *Providers*: OAI.
- Service: Custom Guardrails declarative `guardrails` array on request each with `moderation_llm_v2` config (`custom_category_thresholds` 0-1 per category `1` disables, `ignore_other_categories`, `action:"block"`, `model_name` override, `block_on_error`); input-only evaluates incoming request before model; triggered blocks HTTP 403; multiple guardrail objects; blocked if any triggers; attachable inline on `chat.complete`/`conversations.start`/agent-level `agents.create` inherited overridable per-request (Mst). *Providers*: Mst.
- Service: Safety Settings per-request per-category adjustable filters gate prompts; blocking based on probability of content being unsafe; built-in non-adjustable protections child safety `PROHIBITED_CONTENT` always block; default threshold for Gemini 2.5/3 is `Off` (Goo `safetySettings`). *Providers*: Goo.
- Service: Moderation prompting pattern no dedicated moderation model; prompt Claude on Messages API with content + category list instruct strict JSON verdict parse result; supports binary/multi-level risk 0-3/category definitions/batch processing patterns; recommended `claude-haiku-4-5`, `max_tokens=200` single / `2048` batch, `temperature=0`, `output_config.format` json_schema (Ant). *Providers*: Ant.
- Service: Output guardrail validates/redacts final output before leaves system; runs only for agent producing final output; same `GuardrailFunctionOutput`/tripwire contract (OAI `Agent.output_guardrails`). *Providers*: OAI.
- Service: Tool guardrail checks arguments or results around a specific function tool call; attached to tool not agent (OAI). *Providers*: OAI.
- Service: `runInParallel` false = blocking sequential stops main agent if tripped, true = parallel lower latency speculative work (OAI). *Providers*: OAI.
- Service: Tripwire boolean signaling guardrail blocked (`tripwireTriggered`/`tripwire_triggered` OAI). *Providers*: OAI.

**Module — Moderation Categories**
- Service: Mistral 11 fixed: Sexual, Hate and Discrimination, Violence and Threats, Dangerous, Criminal, Self-Harm, Health, Financial, Law, PII, **Jailbreaking**. *Providers*: Mst.
- Service: Google 4 adjustable + built-in: Harassment, Hate speech, Sexually explicit, Dangerous; built-in non-adjustable child safety, prohibited content. *Providers*: Goo.
- Service: Anthropic customizable: Child Exploitation, Conspiracy Theories, Hate, Indiscriminate Weapons, Intellectual Property, Non-Violent Crimes, Privacy, Self-Harm, Sex Crimes, Sexual Content, Specialized Advice, Violent Crimes (append e.g. "Underage Posting"). *Providers*: Ant.
- Service: OpenAI fully custom (you define categories in guardrail agent). *Providers*: OAI.

**Module — Decision Granularity**
- Service: Binary `violation:bool` (Ant, Mst `violated`). *Providers*: Ant, Mst.
- Service: Multi-level integer risk 0-3 (Ant `risk_level`). *Providers*: Ant.
- Service: Probability enum + numeric probability/severity scores (Goo `SafetyRating` `category`/`probability` HIGH/MEDIUM/LOW/NEGLIGIBLE/`probabilityScore`/`severity`/`severityScore`/`blocked`). *Providers*: Goo.
- Service: Classification label set (Mst Judge CLASSIFICATION; OAI grader labels). *Providers*: Mst, OAI.

**Module — Refusal Handling & Fallback (Anthropic)**
- Service: Claude Fable 5 safety classifiers can decline; refusal HTTP 200 with `stop_reason:"refusal"`; `stop_details` identifies policy category (`cyber`/`bio`/`frontier_llm`/`reasoning_extraction`/null). *Providers*: Ant.
- Service: Server-side fallback `fallbacks` parameter up to 3 entries each `{model, max_tokens?, thinking?}`; tried in order; `usage.iterations` records every attempt; only safety-classifier decline triggers fallback; rate limits/overload returned as-is. *Providers*: Ant.
- Service: SDK middleware fallback `BetaRefusalFallbackMiddleware` with shared `BetaFallbackState` to pin follow-ups to accepting model. *Providers*: Ant.

**Module — PII Detection & Redaction (Azure, governance view)**
- Service: Text PII / Conversation PII / Document PII with configurable redaction policies (characterMask/noMask/entityMask/syntheticReplacement), `piiCategories`, `confidenceScoreThreshold` with `default` and `overrides[]`, `disableEntityValidation`, `excludeExtractionData`. *Providers*: Az.

**Module — Safety Filters & Watermarking (images/video)**
- Service: Safety filters (Goo); `personGeneration` controls `allow_all`/`allow_adult`/`dont_allow` (Goo Veo); SynthID watermarking (Goo all generated images and Veo outputs). *Providers*: Goo.

### Product L5.K.3 — Human-in-the-Loop Approvals

**Module — Approval Flow**
- Service: Resumable serialized state `state.approve()` + resume `run(agent, state)` (OAI). *Providers*: OAI.
- Service: Permission events + hooks (Ant `tool_decision`, `permission_mode_changed`, `blocked_on_user`). *Providers*: Ant.
- Service: Human reviewers alter/block content (Goo guidance). *Providers*: Goo.
- Service: `confirmation_status:pending` + `tool_confirmations[]` / `DeferredToolCallsException` (Mst). *Providers*: Mst.
- Service: `interaction.requires_action` (Goo). *Providers*: Goo.

**Module — Prompt Injection Defenses**
- Service: Untrusted text/data entering AI system to override instructions; mitigate via developer-message precedence, structured outputs for data flow, tool approvals, guardrails for user inputs, trace graders/evals (OAI Agent Builder safety). *Providers*: OAI.
- Service: `enable_prompt_injection_detection` (Goo computer use). *Providers*: Goo.
- Service: Dedicated Jailbreaking category (Mst). *Providers*: Mst.
- Service: Domain filtering `allowed_domains`/`blocked_domains` limit web access (Ant/OAI); URL validation restricting fetch to URLs already in context (Ant `web_fetch`). *Providers*: Ant, OAI.
- Service: Path traversal protection for memory/file tools; allowlists for shell; secret injection with placeholder names (OAI `domain_secrets`). *Providers*: OAI.
- Service: Treat all tool/web/page content as untrusted input; only direct user instructions count as permission. *Providers*: all.

**Module — KYC & Adversarial Testing**
- Service: Know your customer registration/login requirements; linking to existing accounts; credit card/ID (OAI). *Providers*: OAI.
- Service: Adversarial testing red-teaming test robustness against adversarial/prompt-injection inputs (OAI). *Providers*: OAI.

### Product L5.K.4 — Permission Policies (Managed Agents)

**Module — Permission Policy (managed agent)**
- Service: Control whether server-executed tools (agent toolset + MCP toolset) run automatically or wait for human approval; custom tools are app-executed not governed (Ant `permission_policy` `always_allow`/`always_ask`). *Providers*: Ant.

## Domain L5.L — Observability, Tracing & Evaluation

### Product L5.L.1 — Telemetry & Observability Backend Wiring

**Module — Master & Signal Enables**
- Service: `enable_telemetry` `CLAUDE_CODE_ENABLE_TELEMETRY=1` master switch (Ant); `enable_enhanced_telemetry_beta` `CLAUDE_CODE_ENHANCED_TELEMETRY_BETA=1` enable span tracing beta (Ant); `metrics_exporter`/`logs_exporter`/`traces_exporter` `console`/`otlp`/`prometheus`(metrics)/`none` (Ant); `store` project toggle / per-request (Goo); tracing scope controls SDK-level/per-run reduce (OAI). *Providers*: Ant, Goo, OAI.

**Module — OTLP Transport**
- Service: `OTEL_EXPORTER_OTLP_PROTOCOL` grpc/http/json/http/protobuf; `OTEL_EXPORTER_OTLP_ENDPOINT` (gRPC :4317; HTTP :4318 + /v1/{metrics,logs,traces}); `OTEL_EXPORTER_OTLP_HEADERS` auth; per-signal overrides `_PROTOCOL`/`_ENDPOINT`; `OTEL_EXPORTER_OTLP_METRICS_TEMPORALITY_PREFERENCE` delta default/cumulative. *Providers*: Ant.

**Module — Export Intervals**
- Service: `OTEL_METRIC_EXPORT_INTERVAL` 60000ms; `OTEL_LOGS_EXPORT_INTERVAL` 5000ms; `OTEL_TRACES_EXPORT_INTERVAL` 5000ms (set all ~1000ms for short-lived `query()`). *Providers*: Ant.

**Module — mTLS & Dynamic Headers**
- Service: Client cert/key/passphrase per protocol http/protobuf/http/json (`CLAUDE_CODE_CLIENT_CERT`/`CLAUDE_CODE_CLIENT_KEY`/`CLAUDE_CODE_CLIENT_KEY_PASSPHRASE` + `NODE_EXTRA_CA_CERTS`) or grpc (`OTEL_EXPORTER_OTLP_CLIENT_KEY`+`OTEL_EXPORTER_OTLP_CLIENT_CERTIFICATE` + `OTEL_EXPORTER_OTLP_CLIENT_CERTIFICATE`); dynamic rotating-token headers via external script (refresh `CLAUDE_CODE_OTEL_HEADERS_HELPER_DEBOUNCE_MS` default 29 min). *Providers*: Ant.

**Module — Hosted Dashboards**
- Service: OpenAI Traces (built-in enabled by default); Mistral Explorer (Enterprise always-on for workspace); Google AI Studio Logs (paid tier billing-enabled project). *Providers*: OAI, Mst, Goo.

### Product L5.L.2 — Identity, Resource Attribution & Tenancy Configuration

**Module — Resource Attributes**
- Service: `service.name` `OTEL_SERVICE_NAME`; `resource_attributes` `OTEL_RESOURCE_ATTRIBUTES` comma-separated key=value; `enduser.id`/`tenant.id`/`user.account_uuid`/`user.id`/`user.email`/`user.groups`/`identity.source`/`app.version`/`app.entrypoint`/`terminal.type` (Ant); `metadata.user_id` opaque external user id no PII abuse detection (Goo); `metadata` custom key-value per-request filterable in Explorer (Mst); `api_agent_id` ID of agent that handled request (Mst); `correlation_id` cross-system tracing identifier (Mst); `organization.id`/`prompt.id`/`workflow.run_id`/`workflow.name`/`workspace.host_paths` (Ant); `include_resource_attributes`/`include_session_id`/`include_account_uuid`/`include_version`/`include_entrypoint` bool controls (Ant). *Providers*: Ant, Goo, Mst.

**Module — Cardinality Discipline**
- Service: Custom keys become labels on every metric series; high-cardinality values inflate storage; exclude custom attributes from datapoint labels (`OTEL_METRICS_INCLUDE_RESOURCE_ATTRIBUTES=false`). *Providers*: Ant.

### Product L5.L.3 — Observability Signals Emitted During the Run

**Module — Metrics**
- Service: `claude_code.session.count` (attr `start_type` fresh/resume/continue/agents_view); `claude_code.lines_of_code.count` (attr type added/removed); `claude_code.pull_request.count`; `claude_code.commit.count`; `claude_code.cost.usage` USD; `claude_code.token.usage` tokens; `claude_code.code_edit_tool.decision` count; `claude_code.active_time.total` seconds (Ant). *Providers*: Ant.
- Service: Other platforms equivalent counts via response metadata (`usageMetadata` Goo, event fields `input_tokens`/`output_tokens`/`total_time_elapsed` Mst). *Providers*: Goo, Mst.

**Module — Log Events**
- Service: Per-prompt events, per-API-request/error events, per-tool events (`tool_result`, `tool_decision`), MCP events (`mcp_server_connection`), `permission_mode_changed`, `assistant_response`; content-bearing events gated by flags (Ant `claude_code.` prefix exported via OTel logs). *Providers*: Ant.
- Service: Security-relevant events (`tool_decision`, `tool_result`, `mcp_server_connection`, `permission_mode_changed`) form per-user audit trail when end-user identity attached — forwardable to SIEM. *Providers*: Ant.

**Module — Traces / Spans**
- Service: Span hierarchy `claude_code.interaction` (one turn prompt→response) → `claude_code.llm_request` (each API call) / `claude_code.hook` (each hook detailed beta) / `claude_code.tool` (each tool invocation) → `claude_code.tool.blocked_on_user` (permission wait) / `claude_code.tool.execution` (execution) / subagent `claude_code.llm_request`/`.tool` spans (Ant). *Providers*: Ant.
- Service: `claude_code.interaction` attributes (`user_prompt` gated, `user_prompt_length`, `interaction.sequence`, `interaction.duration_ms`). *Providers*: Ant.
- Service: `claude_code.llm_request` attributes (`model`, `gen_ai.system`, `gen_ai.request.model`, `query_source`, `agent_id`, `parent_agent_id`, `workflow.run_id`, `workflow.name`, `speed` fast/normal, `llm_request.context` interaction/tool/standalone, `duration_ms`, `ttft_ms`, `input_tokens`, `output_tokens`, `cache_read_tokens`, `cache_creation_tokens`, `request_id`, `gen_ai.response.id`, `client_request_id`, `attempt`, `success`, `status_code`, `error`, `response.has_tool_call`, `stop_reason`, `gen_ai.response.finish_reasons`). *Providers*: Ant.
- Service: `claude_code.tool` attributes (`tool_name`, `duration_ms`, `result_tokens`, `tool_use_id`, `gen_ai.tool.call.id`, `agent_id`, `parent_agent_id`, `workflow.run_id`, `workflow.name` gated, `file_path`, `full_command`, `skill_name`, `subagent_type` gated by `OTEL_LOG_TOOL_DETAILS`; `tool.output` span event input/output truncated 60KB with `OTEL_LOG_TOOL_CONTENT=1`). *Providers*: Ant.
- Service: `claude_code.tool.blocked_on_user` (`duration_ms`, `decision` accept/reject, `source`); `claude_code.tool.execution` (`duration_ms`, `tool_use_id`, `gen_ai.tool.call.id`, `success`, `error` gated); `claude_code.hook` (`hook_event`, `hook_name`, `num_hooks`, `hook_definitions` gated, `duration_ms`, `num_success`, `num_blocking`, `num_non_blocking_error`, `num_cancelled`; requires `ENABLE_BETA_TRACING_DETAILED=1` + `BETA_TRACING_ENDPOINT`). *Providers*: Ant.
- Service: Span status `llm_request`/`tool.execution`/`hook` set `ERROR` on failure; others end `UNSET`; each retry is `gen_ai.request.attempt` span event. *Providers*: Ant.
- Service: OpenAI default trace contents (overall run/workflow; each model call; tool calls and outputs; handoffs and guardrails; custom spans wrapped via `withTrace`/`trace` `name` + `callback`). *Providers*: OAI.

**Module — Cost & Usage (in-process)**
- Service: `message.message.id`/`message_id` nested BetaMessage id dedup by this; `message.message.usage`/`usage` per-step token counts (`input_tokens`, `output_tokens`, `cache_creation_input_tokens`, `cache_read_input_tokens`); `total_cost_usd` cumulative estimated cost across all steps success and error; `usage` cumulative token counts; `modelUsage`/`model_usage` map model → `{costUSD, inputTokens, outputTokens, cacheReadInputTokens, cacheCreationInputTokens}` (Ant). *Providers*: Ant.
- Service: Google `usageMetadata` (`promptTokenCount` etc.); Mistral `input_tokens`/`output_tokens`/`total_time_elapsed` event fields. *Providers*: Goo, Mst.
- Service: Constraints estimates drift with pricing changes/unknown models/unmodelable billing rules — do not bill end users or trigger financial decisions; each `query()` returns own `total_cost_usd` no session-level total accumulate yourself; always read cost from result message regardless of subtype; on `output_tokens` discrepancies for same id use highest value and prefer result message's accumulated estimate. *Providers*: Ant.

**Module — W3C Trace-Context Propagation**
- Service: SDK auto-propagates W3C trace context into CLI subprocess — when OTel span active `TRACEPARENT`/`TRACESTATE` injected into child env, `claude_code.interaction` becomes child of your span; Bash/PowerShell subprocesses inherit `TRACEPARENT` env referencing active tool-execution span → subprocess spans nest under `claude_code.tool.execution`; direct Ant API calls carry W3C `traceparent` header (llm_request span context), API's `traceresponse` header recorded as span link, same for outbound HTTP MCP requests, header not sent to third-party providers; auto-injection skipped if you set `TRACEPARENT` explicitly in `options.env`; interactive CLI ignores inbound `TRACEPARENT` only Agent SDK and `claude -p` honor it; through custom `ANTHROPIC_BASE_URL` proxy `traceparent`/`TRACEPARENT` suppressed by default set `CLAUDE_CODE_PROPAGATE_TRACEPARENT=1` to enable (Ant). *Providers*: Ant.

### Product L5.L.4 — Storage, Logging & Retention

**Module — Store / Log Toggle**
- Service: Telemetry enable + per-signal exporters (Ant); `store` boolean per request / project (Goo); always captured for workspace Enterprise (Mst); tracing enabled-by-default scope via SDK (OAI). *Providers*: per platform.

**Module — Retention**
- Service: You own it OTLP backend (Ant); 7/14/28/55-day window datasets don't expire (Goo); hosted Enterprise (Mst); hosted dashboard (OAI). *Providers*: per platform.

**Module — Sensitive-Content Gating**
- Service: `OTEL_LOG_USER_PROMPTS`, `OTEL_LOG_TOOL_CONTENT`, `OTEL_LOG_RAW_API_BODIES` etc. (Ant); data-use policy opt-in sharing (Goo); Enterprise gating (Mst); tracing scope controls (OAI). *Providers*: per platform.

### Product L5.L.5 — Search, Filter & Inspect Production Traffic

**Module — Traffic Inspection**
- Service: Hosted dashboard search/filter (OAI Traces, Mst Explorer, Goo AI Studio Logs); log filtering by timestamp/level/content/replica (HF). *Providers*: OAI, Mst, Goo, HF (L1).

### Product L5.L.6 — Evaluation & Scoring (Judges / Graders / Campaigns / Evals)

**Module — Judge / Scorer**
- Service: Mistral Judge (CLASSIFICATION or REGRESSION, Jinja2 instructions, model scores outputs against rubrics). *Providers*: Mst.
- Service: OpenAI Grader (trace grader; LLM-as-a-judge / model grader with rubrics). *Providers*: OAI.
- Service: Anthropic classification cookbook. *Providers*: Ant.
- Service: Google safety classifier; LLM-as-judge in guidance. *Providers*: Goo.
- Service: Metric-based (ROUGE, exact match, function-call accuracy). *Providers*: OAI best practices.
- Service: Human labelers. *Providers*: Goo guidance, OAI human evals.

**Module — Eval Runs / Campaigns**
- Service: Mistral Campaign (one Judge over filtered events, background). *Providers*: Mst.
- Service: OpenAI Eval run (over a dataset); legacy Evals API deprecated. *Providers*: OAI.
- Service: Google Batch API re-run of curated dataset. *Providers*: Goo.
- Service: Continuous evaluation CE on every change (OpenAI best practices). *Providers*: OAI.

**Module — Trace Grading**
- Service: Scoring individual end-to-end traces with structured criteria (did agent pick right tool? did handoff happen when should?). *Providers*: OAI.

### Product L5.L.7 — Datasets, Curation & Re-runs

**Module — Dataset**
- Service: Google Dataset (curated logs, no expiry, ≤1000/project). *Providers*: Goo.
- Service: Mistral Dataset (records = conversation + properties + source, JSONL import/export). *Providers*: Mst.
- Service: OpenAI Dataset (golden set, eval runs). *Providers*: OAI.
- Service: Anthropic manual no dedicated surface. *Providers*: Ant.

**Module — Record Metadata**
- Service: Mistral Properties (`expected_output`, `category`, `grading_guidance`, `difficulty`). *Providers*: Mst.
- Service: OpenAI golden-set labels / expected outputs. *Providers*: OAI.

**Module — Dataset Ingest**
- Service: From production traffic/logs (all); Manual entry (Mst); JSONL file upload (Mst); From Playground / Campaign (Mst). *Providers*: per platform.

### Product L5.L.8 — Production Monitoring & Improvement Loop

**Module — Improvement Loop**
- Service: Observe → moderate → approve → record → score → curate datasets → re-run → improve (prompts, routing, fine-tuning). *Providers*: all.

**Module — Privacy / Data-Use**
- Service: Sensitive-content opt-in flags (Ant); Opt-in sharing under "Unpaid Services" terms with account/key/project disconnection (Goo); Enterprise gating (Mst); Tracing scope reduction (OAI). *Providers*: per platform.

## Domain L5.M — Files, Storage & Data Lifecycle (governance)

### Product L5.M.1 — Files & Storage Governance

**Module — File Lifecycle**
- Service: `expires_after` / `expires_at` auto-delete files after a duration (OAI). *Providers*: OAI.
- Service: `background` store response ~10 min for polling ZDR-incompatible (OAI). *Providers*: OAI.
- Service: `external_web_access` offline/cache-only web search BAA-eligible under ZDR (OAI). *Providers*: OAI.
- Service: Raw file expiration 48 hours embeddings persist indefinitely (Goo). *Providers*: Goo.

## Domain L5.N — Versioning, Deprecation & Changelog

### Product L5.N.1 — API Versioning

**Module — Versioning**
- Service: Dated API versions (`api-version=2022-05-01` Az; `anthropic-version: 2023-06-01` Ant). *Providers*: Az, Ant.
- Service: Beta headers (`anthropic-beta: files-api-2025-04-14`, `compact-2026-01-12`, `fast-mode-2026-02-01`, `task-budgets-2026-03-13`, `context-management-2025-06-27`, `server-side-fallback-2026-06-01` Ant). *Providers*: Ant.
- Service: Model versioning pinned dated snapshots vs rolling aliases. *Providers*: all.
- Service: Deprecation notices (Az legacy capabilities retire March 31 2029; LUIS retired March 2026; QnA Maker retired October 2025; OAI Assistants API sunset August 2026; OAI prompts API shutdown November 2026; Sora 2 shuts down 2026-09-24; Imagen shut down 2026-08-17; Az Custom Vision retirement 2025-2028). *Providers*: per platform.

## Domain L5.O — Errors, Conventions & Client SDKs

### Product L5.O.1 — Errors & Conventions

**Module — Error Handling**
- Service: Streaming error recovery (Ant Claude 4.5 and earlier resume by placing partial response as start of new assistant message; 4.6+ place user message instructing model to continue; tool use and extended thinking blocks cannot be partially recovered resume from most recent text block). *Providers*: Ant.
- Service: Azure job errors `errors[]` array in results `status:failed` for async jobs. *Providers*: Az.
- Service: xAI video error codes `invalid_argument`/`permission_denied`/`failed_precondition`/`service_unavailable`/`internal_error`. *Providers*: Grok.
- Service: Reve error hierarchy `ReveAPIError` → `ReveAuthenticationError`(401)/`ReveBudgetExhaustedError`(402)/`ReveRateLimitError`(429 with `.retry_after`)/`ReveValidationError`(400)/`ReveContentViolationError`; error codes `PROMPT_TOO_LONG`/`CONTENT_POLICY_VIOLATION`/`INDEX_OUT_OF_BOUNDS`/`MISSING_REQUIRED_PARAMETER`. *Providers*: Reve.

**Module — Stop Reasons / Finish Reasons**
- Service: OAI Responses `status:completed`/`status:incomplete` (`incomplete_details.reason:max_output_tokens`/`content_filter`); OAI Chat `finish_reason:stop`/`length`/`content_filter`/`tool_calls`; Goo `finishReason:STOP`/`MAX_TOKENS`/`SAFETY`/`RECITATION`; Ant `stop_reason:end_turn`/`max_tokens`/`stop_sequence`/`tool_use`/`pause_turn`/`refusal`/`model_context_window_exceeded`/`compaction`; Mst `finish_reason:stop`/`length`/`tool_calls`; Grok `status:completed`; OR normalized `tool_calls`/`stop`/`length`/`content_filter`/`error` raw in `native_finish_reason`. *Providers*: per platform.

**Module — Usage / Token Accounting**
- Service: OAI `usage.prompt_tokens`/`completion_tokens`/`total_tokens`/`prompt_tokens_details.cached_tokens`/`completion_tokens_details.reasoning_tokens`; Goo `usageMetadata.total_tokens`/`total_input_tokens`/`total_output_tokens`/`total_thought_tokens`; Ant `usage.input_tokens`/`output_tokens`/`cache_creation_input_tokens`/`cache_read_input_tokens`/`cache_creation.ephemeral_5m/1h_input_tokens`/`output_tokens_details.thinking_tokens`/`server_tool_use`/`service_tier`/`iterations`; Mst `usage` standard; Grok `usage.input_tokens`/`output_tokens`/`total_tokens`/`input_tokens_details.cached_tokens`/`output_tokens_details.reasoning_tokens`/`cost_in_usd_ticks`/`cost_in_nano_usd`/`num_sources_used`/`server_side_tool_usage_details`; OR `usage.prompt_tokens`/`completion_tokens`/`total_tokens`/`prompt_tokens_details.cached_tokens/cache_write_tokens`/`completion_tokens_details.reasoning_tokens`/`cost`/`is_byok`/`cost_details`; Az per-document results with `modelVersion`. *Providers*: per platform.

**Module — Refusal Detection**
- Service: OAI Chat `choices[0].message.refusal`; OAI Responses message content item `type==="refusal"`; Ant `stop_reason:"refusal"` + `stop_details:{category,explanation,type:"refusal"}`; OAI moderation `error.code:moderation_blocked`, `moderation_details.categories`. *Providers*: per platform.

**Module — Generation Stats (OpenRouter)**
- Service: `GET /api/v1/generation?id={id}` returns token counts and cost asynchronously; Generation ID also in `X-Generation-Id` response header. *Providers*: OR.

---

## Cross-layer dependency summary

| Layer | Depends on | Supervised by |
|---|---|---|
| L1 Compute & Model Serving Infrastructure | (physical GPUs) | L5 (identity, billing, rate limits, security, compliance, observability) |
| L2 Model Inference & Intelligence APIs | L1 (deployments serve endpoints) | L5 (auth, billing, moderation inline, tracing, token accounting) |
| L3 AI Modality Products | L2 (generation/embedding/structured-output primitives) | L5 (moderation, ZDR, residency, billing, observability) |
| L4 Agentic Orchestration | L2 (model call + tools), L3 (voice channels, file search, image gen tool, code exec), L1 (GPU sandboxes) | L5 (permissions, approvals, guardrails, hooks, observability, evals) |
| L5 Governance · Safety · Operations | (wraps all) | (self — supervisory) |

## Design choices recap

1. **Separation of conversation state from sandbox/filesystem state** (L4): they can be mixed and matched independently — continue a conversation with a fresh sandbox, or reuse a sandbox with a reset conversation (Goo makes explicit; others implicit).
2. **Two paradigms for text** (L3): generative LLMs (steer via prompts/schemas) vs classical NLP (fixed task-specific endpoints). A unified platform offers both.
3. **Understanding vs generation split for images** (L3): understanding via mainline multimodal chat models; generation via dedicated synthesis models; some platforms bridge both (OpenAI Responses, Google Interactions, Ideogram describe round-trip).
4. **Server-managed vs in-process agent loop** (L4): hosted REST platforms run the loop on managed infra; local-first/embedded runtimes run it in your process via SDK/CLI/IDE.
5. **Bring-your-own OTLP backend vs hosted dashboard** (L5): Anthropic emits OTel signals to any collector you own; OpenAI/Mistral/Google capture server-side in web UI.
6. **Sync vs streaming vs async vs batch** (cross-layer): all platforms offer sync + SSE streaming; async (poll/webhook) for long jobs and media; batch (JSONL 50% off) for bulk offline.
7. **Stateful server-side vs stateless replay** (L3/L4): server-managed conversation state (`previous_response_id`/`previous_interaction_id`/Conversations API) vs client-managed full-history replay; encrypted reasoning replay bridges stateless + reasoning.
8. **Sandboxes appear in three places**: L1 (GPU-platform code-exec envs, Nebius), L4 agent execution environments (managed cloud/self-hosted/git-worktree), and L4 code-execution containers as a built-in tool.

*End of architecture document. This is the union of services found across all nine `summary.md` studies in `./platform-studies/`.*
