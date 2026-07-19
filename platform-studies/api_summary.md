# AI Platform — API Entry Points Summary

> Companion to `api.md` (OpenAPI 3.1 spec). Each row lists the HTTP method, the entry-point URL, and a short description, grouped by layer/domain tag (L0–L5). Base URL is `https://api.ai-platform.example/v1` (see `api.md` servers).

**Total endpoints:** 486

---

## L0.A API Keys

| Method | Path | Description |
|---|---|---|
| POST | `/api_keys` | Create API Key |
| GET | `/api_keys` | List API Keys (prefix+name only) |
| DELETE | `/api_keys/{id_or_prefix}` | Revoke API Key |
| POST | `/api_keys/register` | Register Externally-Owned Key (Ed25519-signed) |
| POST | `/organizations/service_accounts` | Create Service Account |

## L0.A Federation & SSO

| Method | Path | Description |
|---|---|---|
| POST | `/organizations/federation_issuers` | Create Federation Issuer (Workload Identity Federation) |
| GET | `/organizations/federation_issuers` | List federation issuers |
| POST | `/sts/token` | Token Exchange (OAuth 2.0 jwt-bearer / STS) |
| GET | `/compliance/access_events` | Access Transparency Query |

## L0.B Organizations & Tenancy

| Method | Path | Description |
|---|---|---|
| POST | `/organizations` | Create Organization (top-level billing & identity container) |
| GET | `/organizations` | List organizations |
| GET | `/organizations/{id}` | Retrieve organization |
| PATCH | `/organizations/{id}` | Update organization |
| DELETE | `/organizations/{id}` | Delete organization |
| POST | `/organization/invites` | Create Org Invite |

## L0.B Workspaces

| Method | Path | Description |
|---|---|---|
| POST | `/organizations/workspaces` | Create Workspace (sub-tenant boundary) |
| GET | `/organizations/workspaces` | List workspaces |
| GET | `/organizations/workspaces/{id}` | Retrieve workspace |
| PATCH | `/organizations/workspaces/{id}` | Update workspace |
| DELETE | `/organizations/workspaces/{id}` | Delete workspace |
| POST | `/api/admin/workspaces/{id}/add-users` | Invoke add workspace members |
| POST | `/organizations/{org}/projects` | Create project |
| POST | `/gateway/groups` | Create hierarchical billing entity with inherited limits |
| GET | `/gateway/groups/{id}/usage` | Retrieve gateway group usage |
| GET | `/api/admin/users` | List users |
| POST | `/api/admin/users` | Create user |
| GET | `/api/admin/users/{id}` | Retrieve user |
| PATCH | `/api/admin/users/{id}` | Update user |
| DELETE | `/api/admin/users/{id}` | Delete user |
| POST | `/workspaces/{id}/tenants` | Tenant lifecycle (ACTIVE → INACTIVE → OFFLOADED) |

## L0.C Roles & Permissions

| Method | Path | Description |
|---|---|---|
| GET | `/api/admin/roles` | List assignable roles at org + workspace scope |
| GET | `/api/admin/user-groups` | List User Groups (SCIM-syncable) |

## L0.D Outbound Network Policies

| Method | Path | Description |
|---|---|---|
| POST | `/v1/outbound-policies` | Create Outbound Policy (network domain rules) |
| POST | `/v1/outbound-policies/check` | Invoke check outbound policy |

## L0.D Private Connectivity

| Method | Path | Description |
|---|---|---|
| GET | `/v2/privatelink_healthcheck` | Private Link / PSC Healthcheck (regional) |
| POST | `/v1/tunnel/{tunnel_id}` | Tunnel-Client Invocation (outbound-only HTTPS to platform-hosted MCP) |

## L0.E Files API

| Method | Path | Description |
|---|---|---|
| POST | `/v1/files` | Upload File (multipart / URL / base64) |
| GET | `/v1/files` | List files |
| POST | `/v1/files/bulk-delete` | Invoke bulk delete files |
| POST | `/v1/files/uploads` | Start resumable upload session |
| POST | `/v1/files/uploads/complete` | Complete resumable upload |
| POST | `/v1/files/uploads/abort` | Abort resumable upload |
| POST | `/v1/files/prechunked` | Prechunked upload (MXJSON) |
| GET | `/v1/files/{id}` | Get File (metadata + status) |
| PATCH | `/v1/files/{id}` | Patch File metadata |
| DELETE | `/v1/files/{id}` | Delete file |
| GET | `/v1/files/{id}/content` | Get download file content |
| GET | `/v1/files/{id}/download` | Get download file original or pdf |
| GET | `/v1/files/{id}/chunks` | Get download file chunks |

## L0.F Vector Stores

| Method | Path | Description |
|---|---|---|
| POST | `/v1/vector_stores` | Create vector store |
| GET | `/v1/vector_stores` | List vector stores |
| GET | `/v1/vector_stores/{id}` | Retrieve vector store |
| DELETE | `/v1/vector_stores/{id}` | Delete vector store |
| POST | `/v1/vector_stores/{id}/files` | Attach file to vector store |
| GET | `/v1/vector_stores/{id}/files` | List vector store files |
| POST | `/v1/vector_stores/{id}/search` | Retrieve from Vector Store (max_num_results ≤ 50) |
| POST | `/v1/agents/documents` | Documents API (up to 100 bulk), metadata filters, top_k |

## L0.G Environments

| Method | Path | Description |
|---|---|---|
| POST | `/v1/environments` | Create Environment (managed cloud / self-hosted / git worktree / container cache) |
| GET | `/v1/environments` | List environments |
| GET | `/v1/environments/{id}` | Retrieve environment |
| POST | `/v1/environments/{id}` | Update Environment |
| DELETE | `/v1/environments/{id}` | Delete environment |
| POST | `/v1/environments/{id}/archive` | Archive environment |

## L0.G Sandboxes

| Method | Path | Description |
|---|---|---|
| POST | `/v1/sandboxes` | Create Sandbox (Git-like branching, OCI images) |
| POST | `/v1/sandboxes/{id}/executions` | Run Code in Sandbox (async operation) |
| POST | `/v1/sandboxes/{id}/checkpoints/{cid}/branch` | Branch Sandbox (fork at checkpoint) |
| POST | `/v1/sandboxes/{id}/checkpoints/{cid}/rollback` | Rollback Sandbox to checkpoint |
| GET | `/v1/sandboxes/{id}/operations` | Get poll sandbox operations |

## L0.H Cron & Scheduled Tasks

| Method | Path | Description |
|---|---|---|
| POST | `/v1/deployments` | Create Cron Deployment (DST-aware; ≤10 s jitter; ≤1000 deployments/org) |
| POST | `/v1/deployments/{id}/{action}` | Pause / Unpause / Archive / Run Deployment |
| GET | `/v1/deployments/{id}/runs` | List deployment runs |
| POST | `/v1/workspace_agents/trigger` | Trigger Workspace Agent (Idempotency-Key) |
| POST | `/v1/scheduled_tasks` | Scheduled Task CRUD (cadence, unattended approval gating) |

## L0.H Workflows & Pipelines

| Method | Path | Description |
|---|---|---|
| POST | `/v1/pipelines` | Create Pipeline (declarative YAML/Python; draft → saved → published; immutable versioned snapshots) |
| POST | `/v1/pipelines/{id}/run` | Run pipeline |
| GET | `/v1/pipelines/{id}/executions` | List pipeline executions |
| POST | `/v1/pipelines/{id}/optimize` | Optimize Pipeline (MOAR — offline MCTS optimization; Pareto frontier) |
| POST | `/v1/workflows` | Create Workflow (Temporal-based; long-running, fault-tolerant) |

## L0.I Billing & Usage

| Method | Path | Description |
|---|---|---|
| GET | `/v1/organizations/usage_report` | Get usage report |
| GET | `/api/admin/usage` | Get admin usage |
| GET | `/v1/organizations/cost_report` | Get cost report |
| GET | `/api/v1/generation` | Generation Stats (async token counts + cost lookup; X-Generation-Id) |
| GET | `/v1/organizations/{surface}/analytics` | Get surface analytics |
| GET | `/v1/billing/endpoints` | Get billing endpoints |
| GET | `/v1/billing/pods` | Get billing pods |

## L0.I Spend Limits

| Method | Path | Description |
|---|---|---|
| POST | `/v1/organization/projects/{id}/spend_alerts` | Create spend alert |
| GET | `/v1/organizations/spend_limits/effective` | Get read effective spend limits |
| POST | `/v1/organizations/spend_limit_increase_requests/{id}/approve` | Approve spend limit increase |
| POST | `/v1/organizations/spend_limit_increase_requests/{id}/deny` | Deny spend limit increase |
| POST | `/api/admin/spend-limit` | Invoke admin spend limit |
| GET | `/api/admin/spend-limit` | Retrieve admin spend limit |

## L0.I Subscriptions

| Method | Path | Description |
|---|---|---|
| POST | `/v1/subscriptions` | Manage Subscription / Billing Plan (prepay vs postpay, credits) |
| POST | `/v1/priority_tier` | Priority Tier (committed capacity with burndown rates) |

## L0.J Rate Limits

| Method | Path | Description |
|---|---|---|
| GET | `/v1/fine_tuning/model_limits` | Get read fine tuning model limits |
| GET | `/v1/organizations/rate_limits` | Get read org rate limits |
| GET | `/v1/organizations/workspaces/{id}/rate_limits` | Get read workspace rate limits |
| GET | `/api/admin/rate-limit` | Get admin rate limit |

## L0.K Processing Tiers

| Method | Path | Description |
|---|---|---|
| GET | `/v1/service_tiers` | List processing tiers (flex / priority / auto / default / standard / standard_only) |

## L0.L Token Counting

| Method | Path | Description |
|---|---|---|
| POST | `/v1/messages/count_tokens` | Messages Count Tokens (same as Messages; `context_management`) |
| POST | `/v1/responses/input_tokens` | Responses Input Tokens (model-exact token count handling images/files/tools) |

## L0.M Compliance

| Method | Path | Description |
|---|---|---|
| GET | `/v1/compliance/activities` | List Compliance Activities (eDiscovery, DLP, SIEM, chain-of-custody) |
| GET | `/v1/compliance/apps/chats` | Get compliance list chats |
| DELETE | `/v1/compliance/apps/chats/{chat_id}` | Delete delete chat |
| GET | `/v1/compliance/apps/chats/{chat_id}/messages` | Get compliance list chat messages |
| GET | `/v1/compliance/apps/chats/files/{file_id}/content` | Get compliance read chat file |
| GET | `/v1/compliance/files/{file_id}` | Get compliance read file |
| DELETE | `/v1/compliance/files/{file_id}` | Delete delete file |
| GET | `/v1/compliance/generated_files/{gen_file_id}/content` | Get compliance read generated file |
| GET | `/v1/compliance/artifacts/{artifact_version_id}/content` | Get compliance read artifact |
| GET | `/v1/compliance/apps/projects` | Get compliance list projects |
| DELETE | `/v1/compliance/apps/projects/{project_id}` | Delete delete project |

## L0.N Data Residency & Encryption

| Method | Path | Description |
|---|---|---|
| PATCH | `/v1/organization/projects/{id}/data_retention` | Set Data Residency / Retention Policy |
| PATCH | `/v1/organization/projects/{id}/model_permissions` | Model Allowlist/Denylist (project-level model permissions) |
| POST | `/v1/encryption/cmek` | Encryption-at-Rest (CMEK/EKM) |

## L0.O Webhooks

| Method | Path | Description |
|---|---|---|
| POST | `/v1/webhooks` | Register Webhook (static HMAC vs dynamic JWKS RS256) |
| GET | `/v1/webhooks` | List webhooks |
| DELETE | `/v1/webhooks/{id}` | Delete webhook |
| POST | `/v1/webhooks/{id}/rotate` | Rotate Webhook Secret |
| POST | `/v1/webhooks/{id}/configure` | Dynamic Webhook Configuration (uris, user_metadata, revocation_behavior) |

## L0.P SDK & CLI

| Method | Path | Description |
|---|---|---|
| GET | `/v1/openapi` | Published OpenAPI spec |

## L0.Q Routing & Gateway

| Method | Path | Description |
|---|---|---|
| POST | `/v1/gateway/endpoints` | Create Gateway Endpoint (slug → target {provider, model_id, env}) |
| GET | `/v1/gateway/endpoints` | List gateway endpoints |
| PATCH | `/v1/gateway/endpoints/{id}` | Re-point target |
| DELETE | `/v1/gateway/endpoints/{id}` | Delete gateway endpoint |
| POST | `/v1/gateway/groups/{id}/api_keys` | Mint Federated Key (under group) |
| POST | `/v1/gateway/groups/{id}/api_keys/register` | Register Existing Signed Key |
| POST | `/v1/gateway/model/chat/completions` | Model Gateway Passthrough (OpenAI-compatible) |
| POST | `/v1/gateway/model/embeddings` | Model Gateway Embeddings Passthrough |

## L0.R Connections

| Method | Path | Description |
|---|---|---|
| POST | `/v1/connections/applications` | Create Connection Application (OAuth2 / API key / basic auth) |
| GET | `/v1/connections/callback` | OAuth Callback |

## L0.R Vaults & Credentials

| Method | Path | Description |
|---|---|---|
| POST | `/v1/vaults` | Create Vault (write-only; ≤20 credentials/vault; keys immutable) |
| GET | `/v1/vaults` | List vaults |
| GET | `/v1/vaults/{id}` | Retrieve vault |
| DELETE | `/v1/vaults/{id}` | Delete vault |
| POST | `/v1/vaults/{id}/credentials` | Create Credential (mcp_oauth / static_bearer / environment_variable) |
| GET | `/v1/vaults/{id}/credentials/{cid}` | Retrieve credential |
| DELETE | `/v1/vaults/{id}/credentials/{cid}` | Delete credential |
| POST | `/v1/vaults/{id}/credentials/{cid}/rotate` | Rotate Credential (propagates to running sessions) |
| POST | `/v1/vaults/{id}/credentials/{cid}/mcp_oauth_validate` | Validate mcp o auth |

## L1.A Hardware & Model Catalog

| Method | Path | Description |
|---|---|---|
| GET | `/v1/models` | List Models (pricing/latency/throughput/features) |
| GET | `/v1/models/{model_id}` | Get Model (all variants) |
| GET | `/v0/templates` | List Deployable Templates (model+flavor+gpu+region combos, dedicated) |
| GET | `/v1/hardware` | List Hardware Options for a model |
| GET | `/v1/availability-zones` | List availability zones |

## L1.B Model Packaging

| Method | Path | Description |
|---|---|---|
| POST | `/v0/models/custom/register` | 4-step signed upload step 1 (register → getUploadEndpoint → PUT → validateUpload) |
| GET | `/v0/models/custom/{id}/upload-endpoint` | Retrieve upload endpoint |
| POST | `/v0/models/custom/{id}/validate` | Validate upload |

## L1.C Deployment CRUD

| Method | Path | Description |
|---|---|---|
| POST | `/v1/deployments` | Create Deployment (returns id/routing_key/state) |
| GET | `/v1/deployments` | List deployments |
| GET | `/v1/deployments/{id}` | Retrieve deployment |
| PATCH | `/v1/deployments/{id}` | Patch Deployment (mutable fields; region immutable) |
| DELETE | `/v1/deployments/{id}` | Delete deployment |
| POST | `/v1/deployments/{id}/{action}` | Start / Stop / Wake / Activate / Deactivate |
| POST | `/wake` | Wake a scaled-to-zero deployment |

## L1.E Autoscaling

| Method | Path | Description |
|---|---|---|
| PATCH | `/v1/deployments/{id}/autoscaling_settings` | Patch Autoscaling Settings |

## L1.E Health Checks

| Method | Path | Description |
|---|---|---|
| GET | `/health` | Health Endpoint (200 ready / 503 loading) |

## L1.E Lifecycle & Promotion

| Method | Path | Description |
|---|---|---|
| POST | `/v1/models/{id}/environments/{env}/promote` | Invoke promote to environment |
| PATCH | `/v1/models/{id}/environments/{env}` | Patch rolling-deploy config (max_surge / max_unavailable) |
| POST | `/v1/models/{id}/environments/{env}/{action}_promotion` | pause / resume / cancel / force_cancel / force_roll_forward promotion |

## L1.G Async Inference

| Method | Path | Description |
|---|---|---|
| POST | `/v1/async_predict` | Submit Async Predict (→ request_id) |
| GET | `/v1/async_request/{id}` | QUEUED/IN_PROGRESS/SUCCEEDED/FAILED/EXPIRED/CANCELED/WEBHOOK_FAILED |
| DELETE | `/v1/async_request/{id}` | Delete async request |
| GET | `/v1/async_queue_status` | Get async queue status |
| POST | `/run` | Generic async execution primitive (run) |
| GET | `/status/{id}` | Retrieve status |
| POST | `/cancel/{id}` | Cancel run |

## L1.G Batch Inference

| Method | Path | Description |
|---|---|---|
| POST | `/v1/batches` | Submit Batch (multi-endpoint; targets /v1/responses, /v1/chat/completions, /v1/embeddings, /v1/completions, /v1/moderations, /v1/images/*, /v1/videos) |
| GET | `/v1/batches` | List batches |
| GET | `/v1/batches/{id}` | Retrieve batch |
| POST | `/v1/batches/{id}/cancel` | Cancel batch |
| POST | `/batchInferenceJobs` | BatchInferenceJob (inline or file; 1 M requests file / 10k inline; output_file + error_file) |
| GET | `/batchInferenceJobs/{id}` | Retrieve batch inference job |
| POST | `/v1/messages/batches` | Messages Batch (50% discount; 100k requests or 256 MB; expire 24 h; not ZDR) |
| POST | `/v1/batch/jobs` | Batch Job (embeddings/chat/fim/moderations/chat-moderations/ocr/classifications/conversations/audio-transcriptions) |

## L1.G Inference Execution

| Method | Path | Description |
|---|---|---|
| POST | `/v1/chat/completions` | Chat Completions (primary or legacy surface) |
| GET | `/v1/chat/deferred-completion/{request_id}` | Poll deferred completion (200 completed / 202 pending) |
| POST | `/v1/completions` | Completions (legacy) |
| POST | `/v1/messages` | Messages (compat) — content[] typed blocks, thinking blocks with signatures |
| POST | `/inference` | Inference (compat alternate route) |
| POST | `/v1/embeddings` | Embeddings |
| POST | `/v1/rerank` | Rerank (variants: /rerank) |
| POST | `/v1/images/generations` | Image Generations (proxy; canonical L3.B) |
| POST | `/v1/audio/transcriptions` | Audio Transcription |
| POST | `/v1/audio/translations` | Audio Translation |
| POST | `/v1/audio/speech` | Audio Speech (TTS) |

## L1.G RL Rollout

| Method | Path | Description |
|---|---|---|
| POST | `/hot_load/v1/models/hot_load` | RL rollout hot-load / MoE router replay |

## L1.I Observability Plumbing

| Method | Path | Description |
|---|---|---|
| GET | `/v1/endpoints/{id}/open-metrics` | OpenMetrics API (Prometheus text format) |
| GET | `/v1/endpoints/{id}/logs` | Runtime Logs (real-time, filterable; 30-day retention) |

## L2.A Model Catalog

| Method | Path | Description |
|---|---|---|
| GET | `/api/v1/model/{author}/{slug}` | Resolve alias/slug to canonical model id |
| GET | `/v1/model_policy` | Read model-selection governance policies |
| POST | `/v1/model_policy` | Apply model-selection governance policies |

## L2.B Generation (Legacy)

| Method | Path | Description |
|---|---|---|
| POST | `/v1/interactions/{id}/cancel` | Generate Content (legacy) — candidates[].content.parts[] |
| POST | `/v1/interactions/{id}/cancel` | Analyze Text (typed `kind` discriminator; sync) |

## L2.B Generation (Modern)

| Method | Path | Description |
|---|---|---|
| POST | `/v1/responses` | Responses Create (stateful; typed output[] Items, reasoning items, encrypted content) |
| GET | `/v1/responses/{id}` | Stored Responses CRUD — retrieve |
| DELETE | `/v1/responses/{id}` | Stored Responses CRUD — delete |
| POST | `/v1/conversations` | Persistent Conversations API (owner, conversation ID, append-only new turn) |
| GET | `/v1/interactions/{id}` | Background Interaction Poll / Stream (stream=true, last_event_id=) |
| POST | `/v1/interactions/{id}/cancel` | Cancel interaction |

## L2.F Streaming

| Method | Path | Description |
|---|---|---|
| GET | `/v1/responses` | WebSocket Responses (persistent connection; ~40% faster for 20+ tool calls) |

## L2.G Context Management

| Method | Path | Description |
|---|---|---|
| POST | `/v1/responses/compact` | Responses Compact (stateless, ZDR-friendly; opaque compaction item) |

## L2.J Embeddings & Rerank

| Method | Path | Description |
|---|---|---|
| POST | `/api/v1/embeddings` | Embeddings (unified multi-provider) |

## L3.A Classical NLP

| Method | Path | Description |
|---|---|---|
| POST | `/api/v1/embeddings` | NLP Analysis Job (async; dispatches by task `kind`) |
| POST | `/api/v1/embeddings` | Custom Question Answering — query-knowledgebases |
| POST | `/api/v1/embeddings` | Custom Question Answering — query-text (prebuilt) |
| POST | `/api/v1/embeddings` | Orchestration Workflow (projectKind:Orchestration; routes utterances to CLU/CQA) |

## L3.A Custom NLP Training

| Method | Path | Description |
|---|---|---|
| POST | `/api/v1/embeddings` | Import Project (schema + labeled data via Blob Storage) |
| POST | `/api/v1/embeddings` | Invoke train model |
| PUT | `/language/authoring/analyze-text/projects/{name}/deployments/{deploymentName}` | Deploy trained model; swap deployments test↔prod |

## L3.B Image Editing

| Method | Path | Description |
|---|---|---|
| POST | `/v1/images/edits` | Image Edit (prompt-based editing + compositing; multi-image ≤N refs) |
| POST | `/v1/images/imageToImage` | Invoke image to image |
| POST | `/v1/edit` | Invoke edit alias |
| POST | `/v1/images/remix` | Remix (1-6 refs + `<img>N</img>`; image_weight) |

## L3.B Image Generation

| Method | Path | Description |
|---|---|---|
| POST | `/v1/images/generations` | Image Generation (text→raster image; conversational multi-turn variant) |
| POST | `/v1/images/generations/vector` | Vector Image Generation (text→SVG; rejects raster) |
| POST | `/v1/images/generate-transparent` | Transparent-Background Image Generation (die-cut stickers/logos) |

## L3.B Image Postprocessing

| Method | Path | Description |
|---|---|---|
| POST | `/v1/images/crispUpscale` | Image Upscale — Crisp (interpolation sharpening preserving content) |
| POST | `/v1/images/creativeUpscale` | Image Upscale — Creative (regenerates finer details/faces) |
| GET | `/v1/image/effect` | List available visual filters |
| POST | `/v1/image/effect` | Effect Apply (named visual filter; effect_parameters filterId:uniformId:value) |
| POST | `/v1/images/vectorize` | Image Vectorization (raster → SVG; deterministic, no model) |

## L3.B Image Understanding

| Method | Path | Description |
|---|---|---|
| POST | `/v1/images/describe` | Image Annotation Batch (labels+faces+OCR+objects+safe-search simultaneously) |
| POST | `/v1/images/describe` | Async batch annotation (up to 2000 files to Cloud Storage) |
| POST | `/v1/images/describe` | Invoke image analysis |
| POST | `/v1/images/layerize-text` | Text Layer Extraction (base_image_url + text_blocks[] role/text/geometry/font/color/alignment) |

## L3.B Layout Composition

| Method | Path | Description |
|---|---|---|
| POST | `/v1/images/create_layout` | Layout-Aware Create (typed Layout regions; returns image + echoed layout) |
| POST | `/v1/images/edit_layout` | Layout-Aware Edit (LayoutCommand ops: add\|shift\|remove\|place\|keep\|change) |
| POST | `/v1/images/render` | Layout Render (Layout → image; echoes layout) |
| POST | `/v1/images/describe` | Image-to-Layout Extraction (reverse-engineer Layout/V4JsonPrompt) |

## L3.B Style & Assets

| Method | Path | Description |
|---|---|---|
| POST | `/v1/styles` | Custom Style Creation (≤5 reference images; returns reusable UUID style) |
| POST | `/v1/videos/characters` | Character Reference Asset (Video) — reusable non-human character asset |
| POST | `/v1/prompts/enhance` | Prompt Enhancement (≤2000 chars) |
| POST | `/v1/images/magic-prompt` | Magic Prompt |

## L3.B Video Editing

| Method | Path | Description |
|---|---|---|
| POST | `/v1/videos/edits` | Video Editing (one focused change per edit) |
| POST | `/v1/videos/extensions` | Video Extension (source video, prompt, seconds/duration extension portion) |

## L3.B Video Generation

| Method | Path | Description |
|---|---|---|
| POST | `/v1/videos` | Video Generation (text→video) |
| POST | `/v1/videos/generations` | Invoke generate video alt |
| POST | `/v1/videos/generations` | predictLongRunning video generation |
| GET | `/v1/videos/{id}` | Poll video async operation |

## L3.C Audio Preprocessing

| Method | Path | Description |
|---|---|---|
| POST | `/v1/audio-isolation` | Voice Isolator (max 500 MB / 1 hour) |
| POST | `/v1/audio-isolation/stream` | Streamed noise removal |

## L3.C Sound & Music

| Method | Path | Description |
|---|---|---|
| POST | `/v1/text-to-sound-effect` | Sound Effects Generation (duration 0.1-30s; prompt_influence; loop) |
| POST | `/v1/music/compose` | Music Composition (text→music) |
| POST | `/v1/music/compose-detailed` | Invoke compose music detailed |
| POST | `/v1/music/create-composition-plan` | Create composition plan |
| POST | `/v1/music/video-to-music` | Invoke video to music |

## L3.C Speech-to-Text

| Method | Path | Description |
|---|---|---|
| POST | `/v1/speech-to-text` | Batch Transcription (file-based with full option suite) |
| POST | `/v1/audio/transcriptions` | Invoke audio transcriptions |
| POST | `/v1/listen` | Pre-Recorded STT |
| POST | `/v1/stt` | Invoke stt |
| POST | `/post/speech/asr` | Invoke legacy asr |
| GET | `/v1/stt/stream` | Real-Time Streaming Transcription (WebSocket; is_final/speech_final, VAD, KeepAlive, manual commit) |
| POST | `/v1/forced-alignment` | Forced Alignment (audio file + plain string transcript; 29 languages) |
| POST | `/v1/read` | Read API (STT intelligence features applied to text) |

## L3.C Text-to-Speech

| Method | Path | Description |
|---|---|---|
| POST | `/v1/text-to-speech` | Text-to-Speech (single-speaker) |
| POST | `/v1/text-to-speech/multi-speaker` | Multi-Speaker TTS / Dialogue (inputs[] with text + voice_id per turn) |
| GET | `/v1/text-to-speech/stream` | WebSocket TTS Control (flush:true / cancel:true / context_id) |

## L3.C Translation & Dubbing

| Method | Path | Description |
|---|---|---|
| POST | `/v1/audio/translations` | File-Based Audio Translation (output always English text; 25 MB) |
| POST | `/v1/stt-translate` | Translating Transcription (transcript in target language) |
| POST | `/v1/dubbing` | Batch Dubbing (90+ languages; preserve emotion/timing/tone/speaker) |
| GET | `/v1/gpt-realtime-translate` | Live Speech-to-Speech Translation (WebSocket; one session per target language) |

## L3.C Voice Agents

| Method | Path | Description |
|---|---|---|
| GET | `/v1/agent/converse` | Conversational Voice Agent Session (WebSocket; unified realtime session config) |
| GET | `/v1/speech-engine` | Get speech engine |
| GET | `/v1/realtime` | Get realtime session |
| POST | `/v1/realtime` | Invoke realtime post |
| GET | `/v1/realtime/translations` | Get realtime translations session |
| POST | `/v1/agent/configs` | Agent Configuration CRUD (voice, LLM, tools, telephony) |
| GET | `/v1/agent/configs/{id}` | Retrieve agent config |
| POST | `/v1/agents` | Persisted agent definitions (voice platform) |
| POST | `/v1/agents/calls/create-outbound` | Outbound Call (phone number, agent config) |
| POST | `/v1/agents/call-batches/create-call-batch` | Call Batch (telephony outbound) |
| POST | `/v1/agents/documents` | Agent Documents / Knowledge Base Upload (up to 100 bulk) |
| POST | `/v1/agents/webhooks` | Agent Webhooks CRUD (call-event webhooks) |
| POST | `/v1/realtime/calls` | Realtime Call (issue ephemeral tokens for browser/mobile) |
| POST | `/v1/realtime/client_secrets` | Realtime Ephemeral Client Secret |
| POST | `/v1/realtime/translations/calls` | Invoke realtime translations calls |
| POST | `/v1/realtime/translations/client_secrets` | Invoke realtime translations client secrets |

## L3.C Voice Assets

| Method | Path | Description |
|---|---|---|
| GET | `/v1/voices` | Voice Library Browse & Search (filter by language/gender/country; expand[]=preview_file_url) |
| POST | `/v1/voices/find-similar` | Invoke find similar voices |
| POST | `/v1/voices/localize` | Voice Localization (adapt voice to new language) |
| GET | `/v1/voices/{voice_id}` | Retrieve voice |
| PATCH | `/v1/voices/{voice_id}` | Voice Metadata CRUD (name/description/gender/settings, share-to-library) |
| POST | `/v1/voices/clone` | Instant Voice Cloning (IVC) — short audio sample (+ consent recording) |
| POST | `/v1/voices/fine-tunes/create` | Professional Voice Cloning (PVC) — multi-step with speaker separation |
| POST | `/v1/text-to-voice/design` | Voice Design from Text (20-1000 char description; returns 3 preview voices) |
| POST | `/v1/text-to-voice/remix` | Voice Remixing (existing voice + NL attribute transforms) |
| POST | `/v1/pronunciation-dictionaries` | Pronunciation Dictionary CRUD (phonetic rules, versioning, attachment) |
| POST | `/v1/music/fine-tunes` | Music Datasets & Fine-Tuning (non-copyrighted tracks 5-10 min) |

## L3.C Voice Transformation

| Method | Path | Description |
|---|---|---|
| POST | `/v1/speech-to-speech/{voice_id}` | Voice Changer (STS, no translation) |
| POST | `/v1/speech-to-speech/{voice_id}/stream` | Invoke voice changer stream |
| POST | `/v1/infill/bytes` | Audio Infill / Bridging (left_audio, transcript, right_audio) |
| POST | `/v1/music/separate-stems` | Stem Separation (instrument/vocal stems) |

## L3.D Chunking & Enrichment

| Method | Path | Description |
|---|---|---|
| POST | `/v1/chunk` | Chunking Operation (static/token-count/character/separator/markdown/hierarchical/hybrid/line/word/structure-pure) |
| POST | `/v1/chunk/enrich` | Chunk Enrichment / Contextualization (propagate_summary_to_chunks, with_metadata, with_file_context) |

## L3.D Custom Processors

| Method | Path | Description |
|---|---|---|
| POST | `/v1/custom-processors` | Custom Processor CRUD (AI-generated lifecycle) |
| GET | `/v1/custom-processors` | List custom processors |
| GET | `/v1/custom-processors/{id}` | Retrieve custom processor |
| POST | `/v1/custom-processors/{id}/iterate` | Invoke iterate custom processor |
| POST | `/v1/custom-processors/{id}/describe` | Invoke describe custom processor |
| POST | `/v1/custom-processors/{id}/execute` | Invoke execute custom processor |
| GET | `/v1/custom-processors/{id}/pipelines` | List custom processor pipelines |

## L3.D Document Ingestion

| Method | Path | Description |
|---|---|---|
| POST | `/v1/workspaces` | Workspace / Container CRUD (type, residency, embedding model, chunking, expiration, access_mode) |
| GET | `/v1/workspaces` | List document workspaces |
| GET | `/v1/workspaces/{id}` | Retrieve document workspace |
| PUT | `/v1/workspaces/{id}` | Replace document workspace |
| DELETE | `/v1/workspaces/{id}` | Delete document workspace |
| GET | `/v1/workspaces/{id}/documents` | List workspace documents |
| GET | `/v1/workspaces/{id}/documents/{doc}` | Retrieve workspace document |
| DELETE | `/v1/workspaces/{id}/documents/{doc}` | Delete workspace document |

## L3.D Document Transformation

| Method | Path | Description |
|---|---|---|
| POST | `/v1/create-document` | DOCX Generation from Markdown (track changes tags <ins>/<~~>/<comment>) |

## L3.D Document Understanding

| Method | Path | Description |
|---|---|---|
| POST | `/v1/convert` | Document Convert / Parse (returns 202 + request_id) |
| GET | `/v1/convert/{request_id}` | Poll convert operation |
| POST | `/v1/extract` | Data Extraction (JSON-schema-driven LLM extraction with citations + per-field verification + confidence) |
| GET | `/v1/extract/{request_id}` | Get poll extract |
| POST | `/v1/annotate` | BBox / Document Annotation (per-image / document-level) |
| POST | `/v1/gen-schemas` | Schema Auto-Generation (returns simple/moderate/complex candidate schemas) |
| GET | `/v1/gen-schemas/{request_id}` | Get poll gen schemas |
| POST | `/v1/segment` | Document Segmentation (segmentation_schema; document_boundary strategy; page-structure header-based) |
| POST | `/v1/fill` | Form Filling (AcroForm + visual + image field detection; PDF/PNG) |
| POST | `/v1/validate` | KVP Validation (Python-expression-based with retries; ValidatorResult Pass/Fail) |
| POST | `/v1/track-changes` | Track Changes Extraction (DOCX redlines + comments) |
| GET | `/v1/thumbnails/{lookup_key}` | Thumbnail Generation (thumb_width, track_changes, page_range) |
| GET | `/v1/content-types` | Classification / Facets — list content types |
| POST | `/v1/content-types` | adopt \| define_content_type \| undefine_content_type \| define_attribute \| undefine_attribute |
| GET | `/v1/content-types/templates` | List content type templates |
| GET | `/v1/tags` | List tags |
| POST | `/v1/tags` | Create tag |
| POST | `/v1/files/{id}/facets` | classify \| unclassify \| set_value \| clear_value |
| GET | `/v1/files/{id}/facets` | Retrieve file facets |

## L3.D Generation & Output

| Method | Path | Description |
|---|---|---|
| POST | `/v1/ask` | RAG Ask (retrieve + generate; with citations) |
| POST | `/v1/generate` | RAG Generate |

## L3.D Indexing & Graph

| Method | Path | Description |
|---|---|---|
| POST | `/v1/knowledge-graph/build` | Knowledge Graph Build (Pydantic-driven pipeline; output graph_id + stats) |
| GET | `/v1/knowledge-graph/{id}` | Retrieve knowledge graph |
| GET | `/v1/knowledge-graph/{id}/export` | Export (format=CSV/Cypher/JSON/HTML/Docling) |
| GET | `/v1/knowledge-graph/{id}/visualize` | Get visualize knowledge graph |
| POST | `/v1/visualize/embeddings` | t-SNE 2D visualization |
| POST | `/v1/resolve` | Entity Resolution (blocking + pairwise LLM + union-find clustering) |
| POST | `/v1/equijoin` | Equijoin (fuzzy join; LLM-evaluated semantic join of two datasets) |
| POST | `/v1/cluster` | Clustering (hierarchical agglomerative / KMeans / Louvain / value sampling) |

## L3.D MCP Tools

| Method | Path | Description |
|---|---|---|
| GET | `/v1/mcp` | MCP Server (WS; exposes search/ask/convert/extract/graph_query as agent tools) |

## L3.D Query Time

| Method | Path | Description |
|---|---|---|
| POST | `/v1/search` | Search (unified: semantic \| keyword \| hybrid \| agentic \| grep \| list) |
| POST | `/v1/grep` | Grep (regex RE2 over literal chunk text) |
| POST | `/v1/list-chunks` | List Chunks (metadata-only retrieval; no embeddings) |
| POST | `/v1/metadata-facets` | Metadata Facets (aggregate chunk counts grouped by metadata) |
| POST | `/v1/query-agent/ask` | Query Agent Ask (LLM translates NL → database operations across collections) |
| POST | `/v1/query-agent/search` | Query agent search |
| POST | `/v1/query-agent/ask-stream` | Query agent ask stream |
| POST | `/v1/aggregate` | Aggregate queries + grouped search |
| POST | `/v1/rank` | Rank (full sorting by latent attribute) |

## L3.D Schema Management

| Method | Path | Description |
|---|---|---|
| POST | `/v1/schemas` | Schema CRUD (soft delete; create_new_version) |
| GET | `/v1/schemas` | List schemas |
| GET | `/v1/schemas/{id}` | Retrieve schema |
| PUT | `/v1/schemas/{id}` | Replace schema |
| DELETE | `/v1/schemas/{id}` | Delete schema |

## L4.A Agent Definition

| Method | Path | Description |
|---|---|---|
| POST | `/v1/agents` | Create Agent (returns {id, version, created_at, updated_at}) |
| GET | `/v1/agents` | List agents |
| GET | `/v1/agents/{id}` | Retrieve Agent (optionally pinned version) |
| POST | `/v1/agents/{id}` | Update Agent (version required; produces new version on change) |
| DELETE | `/v1/agents/{id}` | Delete agent |
| POST | `/v1/agents/{id}/archive` | Archive Agent (one-way read-only archival) |
| GET | `/v1/agents/{id}/versions` | List agent versions |
| POST | `/v1/agents/{id}/releases` | Create Agent Release (immutable release snapshot) |
| POST | `/v1/agents/{id}/releases/{ver}/environment/{env_id}` | Deploy Agent Release to Environment |
| POST | `/v1/agents/{id}/releases/{ver}/undeploy` | Invoke undeploy agent release |

## L4.A Skills

| Method | Path | Description |
|---|---|---|
| POST | `/v1/skills` | Upload Skill (Custom; multipart files; returns {skill_id, latest_version}) |
| POST | `/v1/skills/config/write` | Skills Config Write (enable/disable a skill) |
| POST | `/v1/skills/list` | List Skills (cwds, forceReload, perCwdExtraUserRoots) |

## L4.B Models (agent-level)

| Method | Path | Description |
|---|---|---|
| GET | `/v1/models/list` | List Models (agent-level; supportedReasoningEfforts, inputModalities, supportsPersonality, isDefault, upgrade) |

## L4.C Sessions

| Method | Path | Description |
|---|---|---|
| POST | `/v1/sessions` | Create Session (provisions sandbox + starts conversation history) |
| GET | `/v1/sessions` | List sessions |
| GET | `/v1/sessions/{id}` | Retrieve session |
| DELETE | `/v1/sessions/{id}` | Delete session |
| POST | `/v1/sessions/{id}/events` | Send Session Events (user.message / user.tool_confirmation / user.custom_tool_result / system.message) |
| GET | `/v1/sessions/{id}/events` | List session events |
| GET | `/v1/sessions/{id}/events/stream` | Stream Session Events (SSE; event_deltas[] opt-in) |
| POST | `/v1/sessions/{id}/resume` | Invoke resume session |
| POST | `/v1/sessions/{id}/fork` | Fork Session (new session with copied history; last_turn_id) |
| POST | `/v1/sessions/{id}/steer` | Steer Session (append user input to in-flight turn) |
| POST | `/v1/sessions/{id}/interrupt` | Interrupt Session (cancel mid-execution) |
| POST | `/v1/sessions/{id}/compact` | Compaction Trigger |
| POST | `/v1/sessions/{id}/goal` | Session Goal CRUD (long-running target with progress) |
| GET | `/v1/sessions/{id}/goal` | Retrieve session goal |
| POST | `/v1/sessions/{id}/goal/clear` | Invoke clear session goal |
| POST | `/v1/sessions/{id}/name` | Invoke rename session |
| POST | `/v1/sessions/{id}/metadata` | Update Session Metadata (gitInfo/tag/custom_title patch) |
| POST | `/v1/sessions/{id}/archive` | Archive session |
| GET | `/v1/sessions/{id}/threads` | List session threads |
| GET | `/v1/sessions/{id}/threads/{tid}/stream` | Get stream thread |
| GET | `/v1/sessions/{id}/threads/{tid}/events` | List thread events |
| POST | `/v1/sessions/{id}/threads/{tid}/archive` | Archive thread |

## L4.E Containers

| Method | Path | Description |
|---|---|---|
| POST | `/v1/containers` | Create Container (returns {id}) |
| GET | `/v1/containers` | List containers |
| DELETE | `/v1/containers/{id}` | Delete container |
| POST | `/v1/containers/{id}/files` | Container File Create (multipart or file_id) |
| GET | `/v1/containers/{id}/files` | Container File List (generated files) |

## L4.F Connectors / MCP

| Method | Path | Description |
|---|---|---|
| POST | `/v1/connectors` | Create Connector / MCP Server Registration |
| GET | `/v1/connectors` | List connectors |
| GET | `/v1/connectors/{idOrName}` | Retrieve connector |
| POST | `/v1/connectors/{idOrName}` | Update connector |
| DELETE | `/v1/connectors/{idOrName}` | Delete connector |
| GET | `/v1/connectors/{id}/tools` | List connector tools |
| POST | `/v1/connectors/{id}/auth_url` | Returns {auth_url, ttl} |
| POST | `/v1/mcp_servers/{name}/oauth/login` | Returns auth_url |
| POST | `/v1/sessions/{id}/mcp/{name}/reconnect` | Invoke mcp runtime reconnect |
| POST | `/v1/sessions/{id}/mcp/{name}/toggle` | Invoke mcp runtime toggle |
| GET | `/v1/sessions/{id}/mcp/status` | Get mcp status |
| POST | `/v1/mcp/config/reload` | Invoke mcp config reload |
| POST | `/v1/mcp/resource/read` | Invoke mcp resource read |
| POST | `/v1/toolkits/prepare/list-tools` | Invoke prepare list tools |
| POST | `/v1/tools/{tenant_id}/callback/{correlation_id}` | Async Tool Callback (tenant_id, correlation_id, result) |

## L4.I Multi-Agent

| Method | Path | Description |
|---|---|---|
| POST | `/v1/spawn_agents_on_csv` | CSV Batch Fan-Out (one worker per row; combined CSV export) |

## L4.J Memory & Knowledge

| Method | Path | Description |
|---|---|---|
| POST | `/v1/memory_stores` | Create Memory Store (workspace-scoped; returns {id}) |
| POST | `/v1/memory_stores/{id}/memories` | Seed Memory (no overwrite) |
| GET | `/v1/memory_stores/{id}/memories` | List Memories (path_prefix, depth) |
| GET | `/v1/memory_stores/{id}/memories/{mem_id}` | Retrieve memory |
| POST | `/v1/memory_stores/{id}/memories/{mem_id}` | Update Memory (content and/or path rename; precondition.content_sha256) |
| DELETE | `/v1/memory_stores/{id}/memories/{mem_id}` | Delete memory |
| GET | `/v1/memory_stores/{id}/memory_versions` | List memory versions |
| GET | `/v1/memory_stores/{id}/memory_versions/{vid}` | Retrieve memory version |
| POST | `/v1/memory_stores/{id}/memory_versions/{vid}/redact` | Redact Memory Version (scrubs content, preserves audit) |
| POST | `/v1/memories` | Agentic Memory add |
| GET | `/v1/memories` | Get agentic memory list |
| POST | `/v1/memories/search` | Invoke agentic memory search |
| POST | `/v1/libraries` | Create Library (returns {library_id, generated_name, generated_description}) |
| POST | `/v1/libraries/{id}/documents` | Invoke upload library document |
| GET | `/v1/libraries/{id}/documents` | List library documents |
| GET | `/v1/libraries/{id}/documents/{doc_id}/status` | Get library document status |
| GET | `/v1/libraries/{id}/documents/{doc_id}/text_content` | Get library document text content |
| GET | `/v1/libraries/accesses/{id}` | List library accesses |
| POST | `/v1/orchestrate/knowledge-bases` | Create Knowledge Base (built-in managed Milvus or external) |
| GET | `/v1/orchestrate/knowledge-bases/{kb_id}/status` | Get knowledge base status |
| POST | `/v1/vector-indices` | Create Vector Index (embeddings model + chunking + retrieval) |
| POST | `/v1/vector-indices/{id}/collections` | Attach collections to vector index |
| POST | `/v1/vector-indices/{id}/refresh` | Invoke refresh vector index |
| POST | `/v1/vector-indices/{id}/rebuild` | Invoke rebuild vector index |
| GET | `/v1/vector-indices/{id}/retrieve` | Retrieve from vector index |

## L4.K Workflows

| Method | Path | Description |
|---|---|---|
| POST | `/v1/orchestrate/flows/{id}/run` | Run Flow (sync) |
| POST | `/v1/orchestrate/flows/{id}/run/async` | Run Flow Async (callbackUrl) |

## L4.L Channels

| Method | Path | Description |
|---|---|---|
| POST | `/v1/agents/{agent_id}/environments/{env_id}/channels` | Bind Channel to (Agent, Environment) |
| POST | `/v1/channels/phone` | Phone Channel CRUD |
| POST | `/v1/channels/phone/{id}/numbers` | Numbers Management (add/list/patch/delete) |
| PUT | `/v1/agents/{id}/embedded-chat-config` | Embedded Chat Config (layout, is_live) + web chat SDK |
| PUT | `/v1/embed-settings/config` | Embed Settings CRUD |
| POST | `/v1/embed-settings/generate-key-pair` | Invoke generate embed key pair |
| PUT | `/v1/agents/{id}/chat-starter-settings` | Chat Starter Settings (starter_prompts/welcome_content/icon) |

## L4.L Voice Channel

| Method | Path | Description |
|---|---|---|
| POST | `/v1/voice-configurations` | Voice Configuration CRUD (AgentIdleHandler, RealtimeAgentSettings) |

## L4.M External Agents

| Method | Path | Description |
|---|---|---|
| POST | `/v1/agents/external-chat` | Create External-Chat Agent (api_url, auth_scheme, auth_config) |
| GET | `/v1/a2a/versions` | A2A Protocol Versions (client/server role) |
| POST | `/v1/externalAgentConfig/detect` | External Agent Config Detect (discovers & migrates artifacts) |
| POST | `/v1/externalAgentConfig/import` | Import external agent config |

## L4.M Plugins & Marketplace

| Method | Path | Description |
|---|---|---|
| POST | `/v1/agents/create-from-template` | Create Agent from Template |
| GET | `/v1/agents/{id}/template-status` | Get agent template status |
| POST | `/v1/marketplaces/add` | Add Marketplace (source: local\|git\|npm\|remote) |
| POST | `/v1/marketplaces/{name}/remove` | Remove marketplace |
| POST | `/v1/marketplaces/{name}/upgrade` | Upgrade marketplace |
| GET | `/v1/plugins` | List plugins |
| GET | `/v1/plugins/{id}` | Retrieve plugin |
| POST | `/v1/plugins/install` | Install Plugin (marketplacePath \| remoteMarketplaceName) |
| POST | `/v1/plugins/{id}/uninstall` | Uninstall plugin |
| POST | `/v1/plugins/{id}/skill/read` | Read Plugin Skill (remote plugin skill Markdown on demand) |
| POST | `/v1/tools/create-from-template` | Create tool from template |
| GET | `/v1/catalog` | Browse Catalog (governed library of pre-built agents and tools) |

## L5.A Hosted UI

| Method | Path | Description |
|---|---|---|
| GET | `/v1/dashboard` | Hosted Dashboard Service (Traces / Explorer / AI Studio Logs) |

## L5.A Observability Export

| Method | Path | Description |
|---|---|---|
| PUT | `/v1/telemetry/config` | Telemetry Configuration Service |

## L5.B Attribution

| Method | Path | Description |
|---|---|---|
| PUT | `/v1/observability/attribution` | Resource Attributes Service |

## L5.E Traffic Inspection

| Method | Path | Description |
|---|---|---|
| POST | `/v1/observability/logs/search` | Observability Logs Search Service (structured filter condition) |
| POST | `/v1/observability/traces/search` | Search observability traces |
| GET | `/v1/observability/traces/{id}` | Retrieve trace |
| GET | `/v1/observability/traces/{id}/spans/{spanId}` | Retrieve trace span |
| GET | `/v1/workflows/events` | Workflow Events Service (cursor pagination) |

## L5.F Datasets

| Method | Path | Description |
|---|---|---|
| GET | `/v1/agent/test_case/templates` | Test Case Template Service (returns sample CSV) |
| POST | `/v1/observability/datasets` | Dataset Service (record = Conversation + Properties + Source) |

## L5.F Eval Campaigns

| Method | Path | Description |
|---|---|---|
| POST | `/v1/observability/campaigns` | Campaign Service (background async; annotations written back into Explorer) |
| POST | `/v1/agent/{id}/test_case` | Test Case Upload Service (CSV body) |
| POST | `/v1/agent/{id}/evaluate` | Evaluate Service (rubric evaluations; LLM agent vulnerability testing incl. adversarial/red-team) |
| GET | `/v1/agent/{id}/evaluations` | List evaluations |
| POST | `/v1/agent/{id}/evaluations/export` | Evaluations Export Service (evaluation_ids:[...]) |

## L5.F Judges & Evaluators

| Method | Path | Description |
|---|---|---|
| POST | `/v1/observability/judges` | Judge Service |

## L5.G Feedback

| Method | Path | Description |
|---|---|---|
| POST | `/v1/feedback/upload` | Feedback Upload Service (classification, reason, logs, conversation_id) |
| POST | `/v1/attestation/generate` | Attestation Service (opt-in via requestAttestation capability) |

## L5.I Guardrails

| Method | Path | Description |
|---|---|---|
| POST | `/v1/guardrails` | Custom Guardrails Service (declarative guardrails array; input-only; 403 on trigger) |

## L5.I Moderation

| Method | Path | Description |
|---|---|---|
| POST | `/v1/moderations` | Moderation Service (reduce unsafe content; returns per-category scores) |
| POST | `/v1/chat/moderations` | Invoke moderate chat |

## L5.I PII & Redaction

| Method | Path | Description |
|---|---|---|
| POST | `/v1/pii/redact` | PII Detection & Redaction (Text / Conversation / Document PII) |

## L5.M Telemetry Backends

| Method | Path | Description |
|---|---|---|
| GET | `/v1/llm-analytics` | Retrieve llm analytics config |
| POST | `/v1/llm-analytics` | Create llm analytics config |

