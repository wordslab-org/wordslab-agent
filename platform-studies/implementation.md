# Implementation Research Plan — Self-Hosted AI Agent Platform

> **Companion to:** `architecture_v2.md` (vendor-neutral layered reference, L0–L5) and `api_summary.md` (486 canonical entry points).
> **Purpose:** Turn the cross-layer suggestion set into a concrete, sequenced research and build plan: open-source selection per domain, custom-build scope, validation strategy, and a phased roadmap toward an enterprise-ready, self-hosted AI agent platform.
> **Stance:** Vendor-neutral, OSS-first, sovereignty-oriented (matches the project's stated goals in `prototype/architecture.ipynb`: technological sovereignty, data security, human supervision, differentiated-skill preservation). Every "OSS" choice must be self-hostable; every "custom" choice is the minimum layer of platform IP that no mature OSS covers.

---

## 0. Guiding principles

1. **Layered independence.** Each L0–L5 domain is a separately deployable service exposing the canonical paths in `api_summary.md`. This preserves swap-out optionality and matches the dependency rules in `architecture_v2.md` §A.1.
2. **OSS-first where mature, custom where it's IP.** Use OSS for commodity infrastructure (identity, storage, queues, telemetry, parsing, model serving). Custom-build only the surfaces that differentiate the platform and have no mature OSS equivalent (Responses/Interactions stateful API, agent loop events, HITL approvals, programmatic tool calling, realtime voice session, improvement loop).
3. **Self-hostability is mandatory.** Every selected OSS component must run on-prem with no external SaaS dependency. Exceptions are explicitly flagged and require a mitigation plan.
4. **Conventions are contracts.** The transversal standards (S.1–S.33 in `architecture_v2.md`) are treated as first-class acceptance criteria: headers, error taxonomy, streaming transport, webhooks, structured-output schema limits, citation contracts, retry/backoff, etc. A service is not "done" until it implements the relevant `S.*` clauses.
5. **Validate before scaling.** Each phase ends with a working vertical slice and a written evaluation, not a feature checklist. Re-evaluate OSS picks against the latest releases before committing (the AI infra landscape moves monthly).

---

## 1. Cross-cutting core stack (decided once, reused everywhere)

These are picked at the platform level, not per domain, because they form the shared spine.

| Concern | Choice | Rationale (mid-2026) |
|---|---|---|
| Primary language | **Python 3.12** (services + SDKs) + **Go** (data-plane gateways, hot paths) | Python ecosystem for AI/ML; Go for low-latency routing, rate-limit, MCP tunnels. |
| Service framework | **FastAPI** (Python) + **Fiber/Echo** (Go) | Async, OpenAPI 3.1 first-class, streaming native. |
| Metadata DB | **PostgreSQL 16** + **pgvector**, **pgcrypto**, row-level security | Tenancy, sessions, agents, releases, catalog, vector (small), audit hashchain. Single operational surface. |
| Cache / queues / SSE cursors | **Redis 7** (Streams, Streams `XADD`/`XREAD` for resumable SSE) | Rate-limit token buckets, last_event_id cursors, ephemeral session state. |
| Object storage | **MinIO** (S3-compatible, multipart + lifecycle) | Resumable upload = S3 multipart; BYOB trivial; `expires_after` native. |
| Durable execution | **Temporal** (self-hosted) | Mature (21k+ stars, 7 yrs), strong SDK in Python/Go/TS; matches `/v1/workflows`, batch, async inference, eval campaigns, approval pause/resume. *Alternatives evaluated:* Inngest (cloud-first, weaker self-host), Hatchet (Postgres-native, younger, good for AI concurrency control — revisit if Temporal ops cost is high), Trigger.dev v3 (best DX, self-hostable, TS-only). Decision: Temporal for breadth; revisit Hatchet for the L4.I multi-agent concurrency layer. |
| Identity | **Keycloak** (SSO/SAML/OIDC/SCIM) + **Ory Keto** candidate | Best OSS IdP coverage for S.1 federation + SCIM. *Revisit:* **Zitadel** if Keycloak JVM ops cost is high; Zitadel is lighter and API-first. |
| Secrets / vault | **HashiCorp Vault** (KV v2, Transit, dynamic secrets, leases) | CMEK/EKM, rotation propagation to running sessions via sidecar lease reload. |
| Policy / RBAC | **Cedar** (Rust, embedded) or **OPA/Rego** | `always_allow`/`always_ask` knobs, per-tool annotations, `deny>defer>ask>allow` precedence. Cedar preferred for eval ergonomics and AWS traction; Rego if team has prior expertise. |
| Webhooks | **Svix** (open-source, Standard Webhooks spec, JWKS, retries, DLQ) | Near-1:1 match for `/v1/webhooks`, `/rotate`, `/configure`. |
| Telemetry | **OpenTelemetry Collector** + **Tempo** (traces) + **Loki** (logs) + **Prometheus/Mimir** (metrics) + **Langfuse** (LLM-native layer) | OTLP transport, mTLS, rotating-token headers; Langfuse gives Explorer + datasets + judge prompts + retention gating. |
| Model serving | **vLLM** (primary) + **SGLang** (structured output / multi-LoRA) + **TEI** (embeddings/rerank) + **llama.cpp** (CPU/GGUF edge) + **TensorRT-LLM** (peak perf, optional) | Engine args in `architecture_v2.md` L1.D map directly. |
| Multi-provider gateway (data plane) | **LiteLLM Proxy** (self-hosted) | OpenAI-compat passthrough, fallbacks, presets, response cache, ZDR/data-collection policy. |
| Agent framework (in-process) | **OpenAI Agents SDK** (Python) + **LangGraph** (state graphs, checkpointers) | Versioned agents, handoffs, agents-as-tools, tool-calling. Custom releases/overrides layer on top. |
| Tool / connector protocol | **MCP** (Anthropic spec, reference SDKs Python/TS/Go/Rust, **FastMCP**, **mcp-go**) | Stdio/HTTP/SSE transports, OAuth, elicitation, approvals. De-facto interop standard by mid-2026. |
| Document parsing | **Docling** (IBM, Apache 2.0, donated to Linux Foundation Agentic AI Foundation in 2026) + **MinerU** (optional, AGPL — for table/LaTeX-heavy academic PDFs) + **Surya/PaddleOCR** OCR fallback | Docling matches `processing_mode`, `save_checkpoint`, DoclingDocument/DocTags output; MinerU when table accuracy is critical and AGPL is acceptable. |
| RAG / graph | **LlamaIndex** + **LightRAG**/**GraphRAG** (Microsoft) + **Qdrant** (vector) + **Neo4j Community** (graph) | Qdrant for multi-tenancy 50k+ shards; LightRAG/GraphRAG for knowledge-graph build + export (CSV/Cypher/JSON/HTML). |
| Image/video | **ComfyUI** (node graph, LoRA, layouts) fronted by a thin OpenAI-compat API; **Diffusers** for programmatic; **Florence-2 / Grounding DINO / SAM-2 / YOLO-World** for understanding; **Rembg** for bg removal; **Tesseract/Surya** for OCR; **AnimateDiff/Mochi/CogVideoX** for video | ComfyUI is graph-based, not REST — a thin service exposes `/v1/images/*` and `/v1/videos/*`. |
| Voice | **faster-whisper / WhisperX / Parakeet** (STT), **Kokoro / Piper / XTTS-v2 / CosyVoice** (TTS), **AudioCraft/MusicGen** (music), **RVC/So-VITS-SVC** (voice transform), **Demucs** (stems), **Pipecat** (realtime voice agent framework, Apache 2.0, pipeline model) with **LiveKit** as alternative for WebRTC-first low latency | Pipecat for control; LiveKit if WebRTC room model fits product. |
| Container / sandbox | **Firecracker** (microVM) + **gVisor** (kernel sandbox) + **E2B** (code-sandbox SDK) + **containerd** (OCI) + **git worktree** for FS isolation | Branch/rollback map to checkpoint/branch; overlayfs snapshots for low-security dev. |
| Network / tunnels | **Tailscale/WireGuard** (outbound-only tunnels) or **Headscale** (self-hosted control plane) + **Envoy** (mTLS mesh, rate-limit, retries, hedge) | Matches `tunnel/*` outbound-only design. |
| SDK generation | **Speakeasy** or **Fern** (preferred) / **OpenAPI Generator** (fallback) | Multi-language SDKs from `api.md`; streaming/files/beta-header overlays hand-written. |

### Decision log discipline
Every row above gets an entry in `implementation/decisions/ADR-NNNN-*.md` recording: date, options evaluated, evidence links, choice, expected revisit date. Re-evaluate every 6 months or when a major upstream release lands.

---

## 2. Research tracks (parallelizable)

Each track produces a **track report** (`implementation/tracks/Tn-*.md`) with: (a) OSS shortlist with evidence, (b) integration sketch, (c) gaps requiring custom code, (d) validation plan, (e) risks. Tracks run in parallel where they don't block each other; dependencies are called out.

### Track T1 — L0 foundation hardening
**Scope:** L0.A Identity, L0.B Tenancy, L0.C RBAC, L0.E Files, L0.J Rate limits, L0.L Token counting, L0.P SDK/CLI, L0.Q Gateway control plane.
**OSS to validate:** Keycloak vs Zitadel (ops cost, SCIM completeness, OIDC jwt-bearer flow); MinIO resumable upload performance at 2 GB files; LiteLLM Proxy routing + response-cache behavior under `X-Response-Cache-TTL`; Speakeasy vs Fern output quality for streaming + multipart.
**Custom scope:** API-key registry (per-model scoping, Ed25519 externally-owned key registration, federated/resellable keys); gateway endpoint registry (slug→target) over Postgres; rate-limit adaptive 15-min bucket logic + cache-aware ITPM; token-counting service per-model.
**Validation:** end-to-end `/v1/api_keys` CRUD + S.1 header conformance; `/v1/files` resumable upload at 2 GB; `/v1/gateway/model/chat/completions` OpenAI-compat passthrough to vLLM.
**Deliverables:** running `identity-svc`, `tenancy-svc`, `files-svc`, `gateway-svc`, `ratelimit-svc`, `tokencount-svc`.
**Depends on:** none (this is the base).

### Track T2 — L0 storage, vaults, workflows
**Scope:** L0.F Vector/graph, L0.G Environments/sandboxes, L0.H Workflows/cron, L0.O Webhooks, L0.R Vaults/connections.
**OSS to validate:** Qdrant vs Weaviate vs pgvector at 10M and 100M vectors, multi-tenancy shard count; Firecracker boot time + checkpoint/branch RPC; Temporal vs Hatchet for AI concurrency control (run a 100-workflow parallelism benchmark); Svix JWKS rotation; Vault lease propagation to running sessions.
**Custom scope:** sandbox checkpoint/branch/rollback service; cron DST-aware scheduler with RRULE (≤10s jitter); vault credential types (`mcp_oauth`/`static_bearer`/`environment_variable`) + `.worktreeinclude` scoping.
**Validation:** `/v1/vector_stores/{id}/search` hybrid retrieval; `/v1/sandboxes/{id}/checkpoints/{cid}/branch`; `/v1/deployments` cron with RRULE; `/v1/webhooks` registration + signed delivery + rotate.
**Deliverables:** `storage-svc`, `sandbox-svc`, `workflow-svc`, `webhook-svc`, `vault-svc`.
**Depends on:** T1 (identity, tenancy).

### Track T3 — L0/L5 observability & governance
**Scope:** L0.M Compliance, L0.N Residency/encryption, L5.A–E Observability, L5.D Retention, L5.L Analytics portals.
**OSS to validate:** Langfuse retention windows + redaction gating flags (`OTEL_LOG_USER_PROMPTS`, etc.); OTel Collector mTLS + rotating-token headers; ClickHouse vs Postgres for trace search at 100M spans; Grafana/Superset/Metabase for analytics portals.
**Custom scope:** compliance activity feed (6-year retention, chain-of-custody); residency policy enforcement tied to routing; resource attribute stamping from gateway OIDC; Explorer filter language matching `api_summary.md` L5.E structured condition.
**Validation:** `/v1/observability/logs/search` with structured filter; `/v1/compliance/activities` pagination; residency-routed inference with `inference_geo: us/eu`.
**Deliverables:** `observability-svc`, `compliance-svc`, `residency-svc`, analytics dashboards.
**Depends on:** T1, T2.

### Track T4 — L1 model serving
**Scope:** L1.A Catalog, L1.B Packaging, L1.C Deployment, L1.D Engine config, L1.E Lifecycle/autoscaling, L1.F Routing, L1.G Inference execution, L1.I Metrics plumbing.
**OSS to validate:** vLLM prefix-cache hit rate + sticky routing; SGLang multi-LoRA batching + RadixAttention; KEDA scalers (tokens-in-flight, queue depth) on a real H100 deployment; Knative scale-to-zero cold start with `POST /wake`; Envoy KV-aware routing extension.
**Custom scope:** deployment service (K8s Operator) wrapping Knative/Ray Serve; PTU/provisioned-throughput tier; async inference via Temporal (`/v1/async_predict` + `/run` + `/status` + `/cancel`); rolling promotion with pause/resume/cancel/force_roll_forward.
**Validation:** `/v1/deployments` create → `/v1/chat/completions` served → autoscale under load → `/health` 200/503 lifecycle; `/v1/async_predict` queue + webhook delivery.
**Deliverables:** `catalog-svc`, `deploy-svc`, `inference-svc` (sync/stream/async/batch), `lifecycle-svc`.
**Depends on:** T1, T2 (workflow for async), T3 (metrics plumbing).

### Track T5 — L2 intelligence primitives
**Scope:** L2.A Catalog, L2.B Generation (modern Responses/Interactions + legacy Chat), L2.C Reasoning, L2.D Structured output, L2.E Tool calling, L2.F Streaming, L2.G Context mgmt, L2.H Multimodal, L2.J Embeddings/rerank, L2.K Batch, L2.L Citations.
**OSS to validate:** **open-responses** (Julep AI, Apache 2.0) as a self-hosted Responses API starting point — assess coverage of `previous_response_id`, reasoning items, encrypted content, built-in tools, MCP; Outlines/xgrammar/lm-format-enforcer for S.19 JSON Schema keyword matrix; mem0/Letta for compaction; LiteLLM unified embeddings; tiktoken vs HF tokenizers for newer model families.
**Custom scope (largest track — core platform IP):**
- **Responses/Interactions stateful API** — Postgres state store + replay logic + encrypted reasoning replay blobs (`include:["reasoning.encrypted_content"]`); this is the single biggest build.
- **Context management** — `/v1/responses/compact` opaque compaction item; `context_management.edits` (clear_tool_uses/clear_thinking) with S.27 cache-invalidation rules.
- **Streaming typed event format** — `response.created`/`output_text.delta`/`completed`, resumable via `last_event_id` (Redis Stream per response).
- **Batch** — JSONL on S3, `custom_id` correlation, 50% discount billing, ZDR-ineligibility, deferred-completion polling.
- **Citations** — `url_citation`/`file_citation`/`container_file_citation`/`place_citation` + S.22 visibility contract.
**Validation:** `/v1/responses` create → retrieve → `previous_response_id` continuity → reasoning item encrypted replay; `/v1/responses/compact`; SSE event sequence matches S.21; batch submit + result correlation.
**Deliverables:** `responses-svc`, `streaming-svc`, `context-svc`, `batch-svc`, `citations-svc`.
**Depends on:** T4 (inference engines), T1 (files, rate limits), T2 (workflow for batch).

### Track T6 — L3 modalities
**Scope:** L3.A Text/NLP, L3.B Images/video, L3.C Voice, L3.D Documents.
**Sub-tracks (independent):**
- **T6a Text/NLP:** LLM-prompting wrapper for classification/NER/PII/summarization over L2; spaCy/GLiNER fallback for high-throughput deterministic tasks. Mostly glue.
- **T6b Images/video:** ComfyUI-fronting service exposing `/v1/images/generations`, `/edits`, `/describe`, `/images/vectorize`, `/videos`, `/videos/edits`, `/videos/extensions`; Florence-2/Grounding DINO/SAM-2/YOLO-World for understanding; Temporal for async video jobs; layout/V4JsonPrompt layer is custom.
- **T6c Voice:** faster-whisper server for STT batch + WebSocket realtime; Kokoro/CosyVoice TTS; Pipecat for realtime voice agent session (`/v1/agent/converse`); telephony via Twilio SDK (self-hosted Twilio-compatible via **fonoster** if sovereignty-critical); voice cloning PVC pipeline with consent recording.
- **T6d Documents:** Docling-fronting service exposing `/v1/convert`, `/v1/extract`, `/v1/segment`, `/v1/gen-schemas`, `/v1/validate`, `/v1/fill`, `/v1/track-changes`, `/v1/thumbnails`; LlamaIndex + LightRAG for `/v1/knowledge-graph/build`, `/v1/resolve`, `/v1/equijoin`, `/v1/cluster`; MCP server (`WS /v1/mcp`) exposing ops as tools.
**OSS to validate:** Docling vs MinerU on a 1000-doc internal benchmark (tables, CJK, LaTeX); Pipecat vs LiveKit for realtime latency and tool-calling; ComfyUI throughput under concurrent generation.
**Custom scope (per sub-track):** T6b layout composition; T6c realtime voice session config + VAD + turn-taking + barge-in; T6d schema auto-generation, KVP with ontology, BARGAIN cascade, MOAR MCTS optimization, custom processor lifecycle.
**Validation:** per sub-track, a vertical slice (one image gen + edit; one STT + TTS roundtrip; one document parse + extract + RAG ask with citations).
**Deliverables:** `text-svc`, `image-svc`, `video-svc`, `voice-svc`, `document-svc`, `mcp-doc-svc`.
**Depends on:** T5 (generation/embeddings), T1 (files), T2 (vector store), T3 (observability).

### Track T7 — L4 agent orchestration
**Scope:** L4.A Agent definition, L4.B Models (agent-level), L4.C Sessions, L4.D Loop/events, L4.E Tools, L4.F Connectors/MCP, L4.G Permissions/approvals, L4.H Hooks, L4.I Multi-agent, L4.J Memory/RAG, L4.K Workflows, L4.L Channels, L4.M Plugins/marketplace, L4.N Webhooks.
**OSS to validate:** OpenAI Agents SDK vs LangGraph for sessions/threads/fork/steer/interrupt; MCP reference SDKs + FastMCP + mcp-go for connectors; mem0 vs Letta for memory store + versions + redaction; LiveKit/Pipecat for voice channel; CrewAI/AutoGen/Magentic-One for multi-agent (benchmark handoff accuracy).
**Custom scope (core platform IP, second-largest track):**
- **Agent registry** — Postgres with `version` optimistic concurrency, releases table, deploy-to-environment binding.
- **Sessions service** — Postgres JSONL or `SessionStore` adapter (S3/Redis), threads per session, two-step lifecycle, `previous_interaction_id` continuity.
- **Agent loop events** — symmetric step model, item lifecycle, compaction events, plan/todo streaming, token-usage events.
- **Built-in tool catalog** — file ops, search, shell, web, code execution, image gen, computer use, ToolSearch; tool annotations, parallel execution rules, programmatic tool calling (V8 JS runtime), advisor tool, memory tool.
- **Connectors registry** — `connections` OAuth app store, async tool callback endpoint with correlation ID, secure MCP tunnels.
- **HITL approval flow** — `requires_action` + `user.tool_confirmation` + resume; `canUseTool` callback; auto-review reviewer agent; outside-CWD confirmation. Resumable serialized state via Temporal.
- **Hook event bus** — PreToolUse/PostToolUse/UserPromptSubmit/PreCompact/SessionStart/…; matcher (regex/exact); parallel execution; precedence.
- **Multi-agent** — coordinator/roster, threads with cross-posted blocking events, CSV batch fan-out, dynamic JS workflow runtime (V8 + bounded concurrency).
- **Memory** — workspace-scoped memory mounted under `/mnt/memory/{slug}`, 8 stores/session, optimistic concurrency `content_sha256`, immutable versions + redaction.
- **Workflows** — `@flow` decorator, flow callbacks (OpenAPI/Python/MCP), `is_schedulable` linking to L0.H cron, goal mode, `/loop` in-session.
- **Channels** — Slack/Teams/Twilio/Genesys/web chat connectors; embedded chat config + web chat SDK.
- **Plugins/marketplace** — local/git/npm/remote sources, install/uninstall, plugin-skill on-demand read, template creation, catalog browser.
**Validation:** `/v1/agents` CRUD + versioning + release deploy; `/v1/sessions` create → send events → stream SSE → fork → interrupt; `/v1/connectors` create + OAuth + tool list; HITL pause/resume roundtrip; multi-agent handoff; memory add/list/search/redact.
**Deliverables:** `agent-svc`, `session-svc`, `tools-svc`, `connector-svc`, `approval-svc`, `hook-svc`, `multiagent-svc`, `memory-svc`, `workflow-svc`, `channel-svc`, `plugin-svc`.
**Depends on:** T5 (generation, tools, streaming), T6 (voice channel, file search, image gen tool, code exec container), T2 (sandboxes, vaults, cron, webhooks), T1 (identity, RBAC).

### Track T8 — L5 evals/safety/admin
**Scope:** L5.F Evals/judges/campaigns/datasets, L5.G Feedback, L5.H Improvement loop, L5.I Moderation/guardrails/PII/approvals, L5.J Model lifecycle admin, L5.K Caching diagnostics, L5.M Telemetry backends, L5.N Cost lookup.
**OSS to validate:** Langfuse judges/datasets/campaigns coverage vs `api_summary.md` L5.F; promptfoo for CLI eval matrix; Llama Guard 3 / ShieldGemma / NVIDIA NeMo Guardrails / Presidio / LLM Guard / Guardrails AI for moderation + PII; MLflow vs HF Hub for model lifecycle.
**Custom scope:**
- **Campaign runner** — Temporal workflow for background async eval runs with annotation write-back to Explorer.
- **Improvement loop** — Observe→moderate→approve→record→score→curate→re-run→improve orchestration over L5.E+F+I; no OSS bundles this end-to-end.
- **Guardrail engine** — input/output/tool guardrails with attachment points + tripwire; refusal-fallback chain (up to 3 models) with `usage.iterations`; safety_decision on computer-use.
- **PII service** — Presidio-backed `/v1/pii/redact` with Text/Conversation/Document PII + redaction policies.
- **Model lifecycle admin** — project-level allowlist/denylist, reasoning/verbosity admin.
- **Caching diagnostics** — `cache_miss_reason` per-request fingerprint comparison (instrumented in T5).
- **Cost lookup** — `/api/v1/generation?id=` async token-count + cost over L0.I billing events.
**Validation:** `/v1/observability/judges` create + run on 20 records; `/v1/observability/campaigns` background + annotation write-back; `/v1/moderations` per-category scores; `/v1/guardrails` 403 on trigger; `/v1/pii/redact` with `characterMask`/`syntheticReplacement`; HITL approval resumable state.
**Deliverables:** `judge-svc`, `campaign-svc`, `dataset-svc`, `eval-svc`, `moderation-svc`, `guardrail-svc`, `pii-svc`, `approval-admin-svc`, `lifecycle-svc`, `cache-diag-svc`, `cost-svc`.
**Depends on:** T3 (observability), T7 (sessions/approvals), T5 (generation for evals).

---

## 3. Cross-cutting validation harnesses

These run continuously from phase 2 onward and gate phase exits.

### V1 — Convention conformance suite
A pytest/Go test suite asserting every `S.*` clause from `architecture_v2.md` against the live services:
- S.1 auth headers; S.2 versioning/beta headers; S.4 error taxonomy + retry headers; S.5 pagination; S.7 stop reasons; S.8 usage accounting; S.10 streaming transport; S.11 idempotency; S.16 async state machine; S.17 webhook verification; S.19 strict-mode schema limits; S.20 JSONL batch format; S.21 tool/content/event catalogs; S.22 citation contract; S.27 context-mgmt invalidation; S.30 realtime session config.
**Exit gate:** 100% of applicable `S.*` clauses green against the live stack.

### V2 — OpenAPI/endpoint parity suite
Generated from `api.md` (the OpenAPI 3.1 spec). One test per row in `api_summary.md` (486 endpoints) asserting: path exists, method allowed, auth enforced, request schema validated, response schema matches, error codes match S.4.
**Exit gate:** ≥95% of 486 endpoints return conformant responses (5% allowance for intentionally-unimplemented preview/beta endpoints).

### V3 — Sovereignty & security audit
- No outbound calls to external SaaS except explicitly allowlisted model providers.
- All secrets in Vault; no secrets in env files committed.
- Per-tenant encryption (CMEK) verified at rest and in transit.
- Sandbox escape tests (Firecracker/gVisor).
- Prompt-injection red-team (L5.I) runs as a campaign weekly.
- Data residency: `inference_geo: us/eu` routing verified end-to-end.
**Exit gate:** clean audit report; zero critical findings.

### V4 — Performance SLOs
- Chat Completions p95 < 2× the underlying engine's p95.
- Streaming TTFT within 50 ms of the engine's TTFT.
- Session create < 500 ms p95.
- Files resumable upload at 1 Gbps sustained.
- Rate-limit overhead < 5 ms per request p99.
- Vector search < 100 ms p95 at 10M vectors.
**Exit gate:** SLOs met under a defined load profile (documented in `implementation/perf/`).

### V5 — OSS currency review (monthly)
Automated check of selected OSS components' latest releases, CVEs, license changes, and deprecation notices. Output: a delta report filed in `implementation/reviews/YYYY-MM.md`. If a selected component goes EOL or changes license (e.g., AGPL relicense like MinerU/Firecrawl/Daytona), trigger a track re-evaluation.

---

## 4. Phased roadmap

Each phase ends with a working vertical slice, a written evaluation against V1–V5, and a go/no-go decision.

### Phase 0 — Foundations & decisions (4–6 weeks)
- Stand up shared infra: Postgres+pgvector, Redis, MinIO, Temporal, Keycloak, Vault, OTel Collector, Loki/Tempo/Prometheus, Langfuse.
- Write ADRs for every core-stack row.
- Generate initial SDKs from `api.md` (Speakeasy/Fern).
- Deliverable: empty but conformant `identity-svc`, `tenancy-svc`, `files-svc`, `gateway-svc` returning S.4-compliant errors.
- Gate: V1 green for S.1/S.4/S.5 on the four services.

### Phase 1 — L0 + L1 serving vertical (10–12 weeks)
- Tracks T1, T2, T4 to MVP.
- One model (e.g., Llama 3.x or Mistral) deployed via vLLM behind the gateway.
- End-to-end: `/v1/chat/completions` + `/v1/completions` + `/v1/embeddings` + `/v1/files` + `/v1/deployments` + `/v1/async_predict` + rate limits + token counting + billing events.
- Gate: V2 green for all L0 + L1 endpoints; V3 sovereignty audit clean; V4 SLOs for serving.

### Phase 2 — L2 intelligence primitives (12–16 weeks)
- Track T5 in full.
- Responses/Interactions stateful API with encrypted reasoning replay.
- Streaming typed events; context compaction; batch; citations.
- Gate: V2 green for L2; V1 green for S.7/S.8/S.10/S.18/S.19/S.22/S.27; open-responses evaluated and either adopted as a base or documented why custom-built.

### Phase 3 — L3 modality slices (12–16 weeks, parallelizable sub-tracks)
- Track T6a–d, each delivering a vertical slice:
  - T6a: text/NLP wrapper.
  - T6b: one image generation + one edit + one describe; one video generation async.
  - T6c: one STT + one TTS roundtrip; one realtime voice agent session via Pipecat.
  - T6d: one document convert + extract + RAG ask with citations + knowledge-graph build.
- Gate: V2 green for implemented L3 endpoints; V4 SLOs per modality; sovereignty audit for any new model providers.

### Phase 4 — L4 agent orchestration (16–20 weeks)
- Track T7 in full. Largest custom build.
- Agent registry + sessions + loop events + tools + connectors + approvals + hooks + multi-agent + memory + workflows + channels + plugins.
- Gate: V2 green for L4; V1 green for S.21/S.30; HITL pause/resume roundtrip verified; multi-agent handoff eval ≥ baseline.

### Phase 5 — L5 evals/safety/admin (10–14 weeks)
- Track T8 in full.
- Judges + campaigns + datasets + improvement loop + moderation + guardrails + PII + approvals admin + model lifecycle + caching diagnostics + cost lookup.
- Gate: V2 green for L5; improvement loop demonstrated end-to-end on a real agent; prompt-injection red-team campaign runs weekly.

### Phase 6 — Hardening & scale (8–12 weeks)
- Load test to target concurrency (define in `implementation/perf/targets.md`).
- Multi-region residency routing.
- HA for every stateful service (Postgres HA, Redis Sentinel/Cluster, Temporal HA, MinIO distributed, Qdrant cluster).
- DR runbook + tested restore.
- Final sovereignty audit; SOC 2 / GDPR / HIPAA-readiness checklist (where pursuing certifications).
- Gate: V3 + V4 green at target scale; DR restore time ≤ defined RTO.

**Total: ~72–96 weeks** of focused work for a team of ~6–10 engineers, parallelizable across tracks. Aggressive but realistic given OSS leverage; the two largest custom builds (T5 Responses/Interactions, T7 agent orchestration) are the critical path.

---

## 5. Risk register (top 10)

| # | Risk | Likelihood | Impact | Mitigation |
|---|---|---|---|---|
| R1 | open-responses (Julep) is immature and we build Responses API from scratch | High | High | Timebox a 2-week spike in Phase 2; fallback to custom with documented delta. |
| R2 | Temporal ops cost exceeds budget | Medium | High | Re-evaluate Hatchet at Phase 1 gate; keep workflow interfaces abstracted. |
| R3 | Keycloak JVM footprint too heavy | Medium | Medium | Re-evaluate Zitadel at Phase 0 gate. |
| R4 | ComfyUI throughput insufficient for production image gen | Medium | Medium | Add vLLM-style OpenAI-compat image server (e.g., Diffusers + FastAPI) as parallel path in T6b. |
| R5 | Pipecat realtime latency above product target | Medium | High | Benchmark vs LiveKit in T6c; switch if WebRTC room model wins. |
| R6 | MinerU AGPL contaminates distribution | Low | High | Default to Docling (Apache 2.0); use MinerU only in SaaS/internal deployments where AGPL is acceptable. |
| R7 | Cedar/OPA eval latency adds >20 ms p99 | Medium | Medium | Cache decisions per (subject, action) tuple; warm cache at session start. |
| R8 | S.19 strict-mode schema limits vary across engines | High | Medium | Build a per-engine conformance matrix in T5; reject unsupported keywords with explicit 400. |
| R9 | Encrypted reasoning replay breaks across model versions | High | High | Pin model + version on stored reasoning blobs; refuse cross-model replay with 400 (per S.18). |
| R10 | OSS license/availability shifts mid-project (e.g., Svix, Langfuse) | Medium | High | V5 monthly review; maintain a fallback shortlist per core-stack row in ADRs. |

---

## 6. Open research questions (to resolve during phases)

1. **Responses API provenance.** Does open-responses' state model match S.18 (encrypted reasoning replay, `previous_response_id` continuity, `include:["reasoning.encrypted_content"]`)? If not, what's the cost of forking vs custom-building? (T5, Phase 2)
2. **Temporal vs Hatchet for AI concurrency.** Can Hatchet's per-worker concurrency control replace our custom `max_threads`/`max_depth` enforcement in L4.I? (T2/T7, Phase 1+4)
3. **KV-aware routing.** Is Envoy's extension API sufficient, or do we need a custom router process? (T4, Phase 1)
4. **Voice realtime session config.** Does Pipecat's session config cover the unified realtime config object in S.30, or do we need a custom WebSocket layer? (T6c, Phase 3)
5. **Knowledge-graph build.** LightRAG vs GraphRAG vs custom Pydantic-driven pipeline (per `architecture_v2.md` L3.D Module: Knowledge Graph Capabilities) — which best matches the `processing_mode`/`extraction_contract`/`gleaning`/`provenance` surface? (T6d, Phase 3)
6. **Improvement loop closure.** What's the minimum viable Observe→moderate→approve→record→score→curate→re-run→improve loop that runs unattended with human review only on escalations? (T8, Phase 5)
7. **Sovereignty boundary.** Which model providers (if any) are allowlisted for outbound calls, and what's the on-prem fallback for each modality? (V3, Phase 0 + ongoing)

---

## 7. Deliverables checklist (per phase)

- [ ] `implementation/decisions/ADR-NNNN-*.md` per core-stack row
- [ ] `implementation/tracks/Tn-*.md` per track with OSS evidence + validation results
- [ ] `implementation/conventions/V1-*.md` test suite (S.* clauses)
- [ ] `implementation/parity/V2-*.md` endpoint parity suite (486 endpoints)
- [ ] `implementation/security/V3-*.md` sovereignty audit
- [ ] `implementation/perf/V4-*.md` SLO definitions + results
- [ ] `implementation/reviews/YYYY-MM.md` monthly OSS currency review
- [ ] `implementation/roadmap/phase-N.md` per-phase exit report

---

> **End of implementation.md.** This is a research plan, not a fixed blueprint: every OSS pick is a hypothesis to be validated in its track, every custom build is scoped to platform IP, and every phase exit is gated by conformance + sovereignty + performance evidence. Revisit this document at each phase boundary and amend based on what the track reports found.