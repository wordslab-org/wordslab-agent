# Google Gemini API - Cross-Cutting Concerns Analysis

> **Source**: Documentation reachable from https://ai.google.dev/gemini-api/docs  
> **Scope**: Only pages documenting cross-cutting/administrative concerns (authentication, security, billing, quotas, batch operations, logs, orchestration, versioning, performance optimization, etc.). Core AI capabilities (text generation, image, video, embeddings, etc.) are excluded.  
> **Date of analysis**: July 14, 2026

---

## Table of Contents

1. [Authentication and Security](#1-authentication-and-security)
   - 1.1 [API Keys](#11-api-keys)
   - 1.2 [OAuth 2.0 Authentication](#12-oauth-20-authentication)
   - 1.3 [Ephemeral Tokens](#13-ephemeral-tokens)
   - 1.4 [Safety Settings](#14-safety-settings)
   - 1.5 [Abuse Monitoring and Usage Policies](#15-abuse-monitoring-and-usage-policies)
2. [Cost Optimization and Billing](#2-cost-optimization-and-billing)
   - 2.1 [Pricing](#21-pricing)
   - 2.2 [Billing](#22-billing)
   - 2.3 [Context Caching](#23-context-caching)
3. [Quotas and Limits](#3-quotas-and-limits)
   - 3.1 [Rate Limits](#31-rate-limits)
4. [Batch Operations](#4-batch-operations)
   - 4.1 [Batch API](#41-batch-api)
5. [Logs and Data Retention](#5-logs-and-data-retention)
   - 5.1 [Logs and Datasets](#51-logs-and-datasets)
   - 5.2 [Data Logging and Sharing Policy](#52-data-logging-and-sharing-policy)
6. [Orchestration and Integrations](#6-orchestration-and-integrations)
   - 6.1 [Webhooks](#61-webhooks)
   - 6.2 [Background Execution](#62-background-execution)
   - 6.3 [Files API](#63-files-api)
   - 6.4 [Partner and Library Integrations](#64-partner-and-library-integrations)
   - 6.5 [Migrate to Cloud (Developer API vs Enterprise Agent Platform)](#65-migrate-to-cloud)
7. [Performance Optimization](#7-performance-optimization)
   - 7.1 [Optimization Overview](#71-optimization-overview)
   - 7.2 [Flex Inference](#72-flex-inference)
   - 7.3 [Priority Inference](#73-priority-inference)
8. [Versioning and Deprecation](#8-versioning-and-deprecation)
   - 8.1 [Release Notes (Changelog)](#81-release-notes-changelog)
   - 8.2 [Deprecations](#82-deprecations)
9. [License and Usage Rules](#9-license-and-usage-rules)
   - 9.1 [Available Regions](#91-available-regions)
10. [Tenant Isolation](#10-tenant-isolation)

---

## 1. Authentication and Security

### 1.1 API Keys

**Source**: https://ai.google.dev/gemini-api/docs/api-key

#### Main Concepts
- Every Gemini API request requires authentication via an **API key** (Standard or Auth type).
- The API is transitioning from **Standard keys** to **Auth keys** for improved security.
- Each API key is tied to a **Google Cloud project** which manages billing, collaborators, and permissions.
- Google AI Studio provides a lightweight UI over Google Cloud projects. Not all Cloud projects are visible by default — projects must be **imported** into AI Studio.
- API keys should be treated like passwords. Compromise leads to quota consumption, unexpected billing charges, and access to private resources.

#### API Functions and Parameters

| Method / Endpoint | Parameters | Description |
|---|---|---|
| `genai.Client(api_key="...")` (Python) | `api_key` (string) | Initialize client with explicit key |
| `new GoogleGenAI({ apiKey: "..." })` (JS) | `apiKey` (string) | Initialize client with explicit key |
| `genai.NewClient(ctx, &ClientConfig{APIKey, Backend: genai.BackendGeminiAPI})` (Go) | `APIKey`, `Backend` | Go client init |
| `POST /v1beta/interactions` (REST) | Header: `x-goog-api-key: YOUR_API_KEY`; Body: `{model, input}` | REST API call |

#### Environment Variables (Recommended)
| Variable | Behavior |
|---|---|
| `GEMINI_API_KEY` | Auto-detected by client libraries |
| `GOOGLE_API_KEY` | Also auto-detected; **takes precedence** if both are set |

#### Key Restriction Mechanisms
- **Request origin restrictions**: IP addresses, websites (HTTP referrers), applications — minimize damage from compromised keys.
- **API restriction**: Restrict key to only the Gemini API (done in AI Studio, requires `apikeys.keys.update` IAM permission).
- Restricting for other Google APIs is done in Cloud Console but will **break Gemini API requests**.

#### IAM Permissions for Key Management
| Action | Required IAM Permissions |
|---|---|
| Create API key | `resourcemanager.projects.get`, `apikeys.keys.create`, `serviceusage.services.enable`, `iam.serviceAccounts.create`, `iam.serviceAccountApiKeyBindings.create` |
| Restrict key in AI Studio | `apikeys.keys.update` |
| List API keys | `apikeys.keys.list`, `serviceusage.services.get` |
| Delete API keys | `apikeys.keys.delete` |
| Rename API keys | `apikeys.keys.update` |

#### Security Deadlines
- **May 7, 2026**: Dormant unrestricted keys blocked.
- **June 19, 2026**: All unrestricted keys must be secured or they stop working.

---

### 1.2 OAuth 2.0 Authentication

**Source**: https://ai.google.dev/gemini-api/docs/oauth

#### Main Concepts
- OAuth 2.0 provides **stricter access controls** than API keys, suitable for user-level authorization.
- Uses **OAuth 2.0 Client IDs** (one per platform; "Desktop app" type for testing).
- **Application Default Credentials (ADC)**: `gcloud auth application-default login` converts `client_secret.json` into credentials stored at a well-known location.
- **Token caching**: A `token.json` file caches access/refresh tokens to minimize consent prompts. Expired tokens are refreshed via `Credentials.refresh(Request())`.

#### API Functions and Parameters

| Method / Command | Parameters | Description |
|---|---|---|
| `gcloud auth application-default login --client-id-file=client_secret.json --scopes='...'` | `--client-id-file`, `--scopes`, `--no-browser` | Converts client secret into ADC token |
| `gcloud auth application-default print-access-token` | — | Prints bearer access token for curl |
| `GET /v1/models` (REST) | Headers: `Authorization: Bearer <token>`, `x-goog-user-project: <project_id>` | Test endpoint |
| `genai.Client()` (Python) | None (auto-detects ADC) | Client with ADC |
| `genai.Client(credentials=creds)` (Python) | `credentials` | Client with explicit creds |
| `InstalledAppFlow.from_client_secrets_file('client_secret.json', SCOPES)` | Client secret path, scopes | OAuth flow |
| `creds.refresh(Request())` | — | Refresh expired token |
| `creds.to_json()` | — | Serialize for caching |

#### Key Parameters
| Parameter | Values | Description |
|---|---|---|
| Scopes | `https://www.googleapis.com/auth/cloud-platform`, `https://www.googleapis.com/auth/generative-language.retriever` | Access level requested |
| `client_secret.json` | File | Downloaded OAuth client credentials |
| `token.json` | File | Cached access + refresh tokens |
| OAuth consent screen | User type = External, test user = self | Consent screen config |
| Client ID type | Desktop app | Application type |

---

### 1.3 Ephemeral Tokens

**Source**: https://ai.google.dev/gemini-api/docs/live-api/ephemeral-tokens

#### Main Concepts
- **Ephemeral tokens** are short-lived authentication tokens for **client-to-server** implementations (browser/mobile → Gemini API via WebSocket).
- Designed for the **Live API** only; requires **`v1alpha`** API version.
- Unlike long-lived API keys, tokens expire quickly and can be scoped/restricted, reducing security risks in production.
- Flow: Client auths with your backend → Backend requests ephemeral token from Gemini → Gemini issues token → Backend sends token to client → Client uses token as API key for WebSocket connections.

#### API Functions and Parameters

| Method | SDK | Parameters | Description |
|---|---|---|---|
| `client.auth_tokens.create(config={...})` | Python | See config below | Provisions a short-lived token; returns `token.name` |
| `client.authTokens.create({config: {...}})` | JS | See config below | Same, JavaScript |
| `ai.live.connect({model, config, callbacks})` | JS | `model`, `config`, `callbacks` | Uses `token.name` as `apiKey` |

#### CreateAuthTokenConfig Parameters

| Parameter | Default | Description |
|---|---|---|
| `uses` | `1` | Number of sessions the token can start |
| `expire_time` | 30 min in future | Token expiration (datetime or ISO 8601 string) |
| `new_session_expire_time` | 1 min in future | Window to start new Live API sessions |
| `http_options.api_version` | — | Must be `v1alpha` |
| `live_connect_constraints.model` | — | Locks token to specific model (e.g. `gemini-3.1-flash-live-preview`) |
| `live_connect_constraints.config` | — | Locks config: `session_resumption`, `temperature`, `response_modalities` |
| `lock_additional_fields` | — | Lock a subset of fields |

#### Token Usage Without SDK
- Pass as `access_token` query parameter, OR
- In HTTP `Authorization` header with auth-scheme `Token` (per RFC 7235).

---

### 1.4 Safety Settings

**Source**: https://ai.google.dev/gemini-api/docs/safety-settings

#### Main Concepts
- The Gemini API provides **adjustable safety filters** across 4 categories.
- Content is blocked based on **probability** of being unsafe (not severity).
- **Default filters are Off** for Gemini 2.5 and 3 models.
- Built-in protections against core harms (e.g. child safety) are always blocked and cannot be adjusted.

#### Safety Categories (HarmCategory)

| Category | Description |
|---|---|
| `HARM_CATEGORY_HARASSMENT` | Negative or harmful comments targeting identity and/or protected attributes |
| `HARM_CATEGORY_HATE_SPEECH` | Content that is rude, disrespectful, or profane |
| `HARM_CATEGORY_SEXUALLY_EXPLICIT` | Contains references to sexual acts or other lewd content |
| `HARM_CATEGORY_DANGEROUS_CONTENT` | Promotes, facilitates, or encourages harmful acts |

#### Block Threshold Settings (HarmBlockThreshold)

| AI Studio Name | API Value | Description |
|---|---|---|
| Off | `OFF` | Turn off the safety filter |
| Block none | `BLOCK_NONE` | Always show regardless of probability |
| Block few | `BLOCK_ONLY_HIGH` | Block when high probability of unsafe content |
| Block some | `BLOCK_MEDIUM_AND_ABOVE` | Block when medium or high probability |
| Block most | `BLOCK_LOW_AND_ABOVE` | Block when low, medium, or high probability |
| N/A | `HARM_BLOCK_THRESHOLD_UNSPECIFIED` | Threshold unspecified; use default |

#### Probability Levels
`HIGH`, `MEDIUM`, `LOW`, `NEGLIGIBLE`

#### API Functions and Parameters

| Method | Parameters | Description |
|---|---|---|
| `client.models.generate_content(model, contents, config)` | `config.safety_settings: [{category, threshold}]` | Generate with safety settings (Python) |
| `ai.models.generateContent({model, contents, config})` | `config.safetySettings: [{category, threshold}]` | JavaScript equivalent |
| `POST /v1beta/models/{model}:generateContent` | Body: `safetySettings: [{category, threshold}]` | REST endpoint |

#### Safety Feedback in Responses
| Location | Field | Meaning |
|---|---|---|
| `promptFeedback` | `blockReason` | If set, the prompt was blocked |
| `Candidate` | `finishReason` | If `SAFETY`, response content was blocked |
| `Candidate` | `safetyRatings` | Array with category + probability details |

---

### 1.5 Abuse Monitoring and Usage Policies

**Source**: https://ai.google.dev/gemini-api/docs/usage-policies

#### Main Concepts
- **Responsible AI use** policy framework to ensure safety/integrity of Gemini API and Google AI Studio.
- **Trust & Safety Team** enforces policies via automated and manual **abuse monitoring** processes.
- **Safety filters** flag prompts/outputs automatically.
- **Human review**: authorized Google employees assess flagged content via an internal governance/review platform.
- Flagged data is retained for **55 days** for detecting/preventing policy violations and legal/regulatory disclosures.
- Flagged data is **not** used to train/fine-tune AI/ML models (except those used for policy enforcement).
- Enforcement actions may be taken if usage doesn't align with policies; appeal links provided on suspension/closure.

#### Related Documents
- Gemini API Additional Terms of Service (`/gemini-api/terms`)
- Generative AI Prohibited Use Policy (`https://policies.google.com/terms/generative-ai/use-policy`)

---

## 2. Cost Optimization and Billing

### 2.1 Pricing

**Source**: https://ai.google.dev/gemini-api/docs/pricing

#### Main Concepts
- **Three plan tiers**: Free, Paid, Enterprise.
- **Token-based pricing** per 1M tokens (USD), split into Input price, Output price (including thinking tokens), Context caching price (write cost + hourly storage cost), and Grounding (Google Search/Maps).
- **Inference modes**: Standard (1.0x baseline), Batch (~0.5x, async, 50% reduction), Flex (~0.5x, best-effort latency), Priority (~1.8x, guaranteed throughput).
- Audio tokens typically 2x text rate. Pro models have prompt-size tiers (≤200k vs >200k tokens).
- Media models use per-image / per-second / per-song pricing.
- **Data use**: Free tier = content used to improve products; Paid tier = content NOT used.

#### Plan Tiers

| Feature | Free | Paid | Enterprise |
|---|---|---|---|
| Models | Limited access | Advanced models | All Paid + extras |
| Tokens | Free input/output | Paid | Paid |
| Context caching | No | Yes | Yes |
| Batch API (50% off) | No | Yes | Yes |
| Data used to improve products | Yes | No | No |
| Support | — | — | Dedicated channels |
| Throughput | — | — | Provisioned |
| Security/compliance | — | — | Advanced |

#### Model Pricing (per 1M tokens, USD, Paid Tier Standard)

| Model | Input | Output | Caching (write) | Storage | Notes |
|---|---|---|---|---|---|
| `gemini-3.5-flash` | $1.50 | $9.00 | $0.15 | $1.00/hr | Free tier: free |
| `gemini-3.1-flash-lite` | $0.25 (text), $0.50 (audio) | $1.50 | $0.025 | $1.00/hr | Free tier: free |
| `gemini-3.1-pro-preview` (≤200k) | $2.00 | $12.00 | $0.20 | $4.50/hr | No free tier |
| `gemini-3.1-pro-preview` (>200k) | $4.00 | $18.00 | $0.40 | $4.50/hr | — |
| `gemini-3-flash-preview` | $0.50 (text), $1.00 (audio) | $3.00 | $0.05 | $1.00/hr | Free tier: free |
| `gemini-2.5-pro` (≤200k) | $1.25 | $10.00 | $0.125 | $4.50/hr | Free tier: free input/output |
| `gemini-2.5-pro` (>200k) | $2.50 | $15.00 | $0.25 | $4.50/hr | — |
| `gemini-2.5-flash` | $0.30 (text), $1.00 (audio) | $2.50 | $0.03 | $1.00/hr | Free tier: free |
| `gemini-2.5-flash-lite` | $0.10 (text), $0.30 (audio) | $0.40 | $0.01 | $1.00/hr | Free tier: free |
| `gemini-embedding-2` | $0.20 (text), $0.45 (image) | — | — | — | Free tier: free |

#### Inference Mode Multipliers

| Mode | Price relative to Standard | Latency | Reliability |
|---|---|---|---|
| Standard | 1.0x | Seconds to minutes | High / Medium-high |
| Batch | ~0.5x | Up to 24 hours | High (throughput) |
| Flex | ~0.5x | Minutes (1–15 min target) | Best-effort (Sheddable) |
| Priority | ~1.8x | Seconds | High (Non-sheddable) |

#### Grounding Pricing (Google Search)

| Model generation | Free allowance | Paid price |
|---|---|---|
| Gemini 2.5 | 1,500 RPD free | $35/1,000 grounded prompts |
| Gemini 3.x | 5,000 prompts/month free (shared) | $14/1,000 search queries |

#### Grounding Pricing (Google Maps)

| Model | Free allowance | Paid price |
|---|---|---|
| Gemini 2.5 Flash/Flash-Lite | 1,500 RPD free | $25/1,000 grounded prompts |
| Gemini 2.5 Pro | 10,000 RPD free | $25/1,000 grounded prompts |

#### Tool Pricing

| Tool | Free Tier | Paid Tier |
|---|---|---|
| Google Search | 500 RPD free (shared) | Per grounding pricing above |
| Google Maps | 500 RPD free | Per grounding pricing above |
| Code execution | Free | Billed at standard model token rates |
| URL context | Free | Charged as input tokens per model pricing |
| File search | Free | Embeddings $0.15/1M tokens; retrieved doc tokens as regular model tokens |

---

### 2.2 Billing

**Source**: https://ai.google.dev/gemini-api/docs/billing

#### Main Concepts
- Billing is based on **payment history** — accounts progress through usage tiers based on cumulative spend and account age.
- **Billing plans** (when you pay): **Prepay** (purchase credits in advance; default for new users) and **Postpay** (accrue costs, charged monthly; available at Tier 3).
- **Cloud Billing accounts** back Gemini API billing; managed in AI Studio or Cloud Console.
- **Spend caps** operate at two levels: project spend caps (per-project, experimental) and billing account tier spend caps (monthly, aggregated across all linked projects).
- **Processing latency**: ~10 minutes for credit usage/tier upgrades; up to 24h for cost breakdown graphs.
- **Overages**: Long-running tasks (batch, agents) may exceed caps during ~10 min latency.
- **Refunds**: Prepay credits are non-refundable except when switching to Postpay. Credits expire after 12 months.

#### Usage Tiers and Billing Caps

| Usage tier | Qualification | Billing tier cap (monthly) |
|---|---|---|
| Free | Active project or free trial | N/A |
| Tier 1 | Set up & link an active billing account | $250 |
| Tier 2 | Paid $100 + 3 days from first successful payment | $2,000 |
| Tier 3 | Paid $1,000 + 30 days from first successful payment | $20,000 – $100,000+ |

#### Prepay Credits

| Limit | Value |
|---|---|
| Minimum purchase | $10 |
| Maximum balance | $5,000 |
| Credit expiry | 12 months after purchase |
| Welcome credit | $300 (not usable for Gemini API if account opened after March 2, 2026) |

#### Auto-Reload (Prepay)

| Parameter | Description |
|---|---|
| Trigger balance | Threshold to trigger auto-reload (e.g. $30) |
| Reload value | Amount to add (e.g. $100) |
| Payment method | Credit card on file |
| Monthly Auto-Charge Limit | Caps auto-reloads per cycle (manual purchases excluded) |

#### API Functions

| Method | Description |
|---|---|
| `GenerativeModel.count_tokens` (Python) | Counts tokens; `GetTokens` API is **not billed** and doesn't count against inference quota |

#### Billing Management URLs
| URL | Purpose |
|---|---|
| https://aistudio.google.com/billing | Buy credits, auto-reload, balance, transactions, switch to Postpay |
| https://aistudio.google.com/usage | Usage dashboard |
| https://aistudio.google.com/rate-limit | Quota/rate limits view |

---

### 2.3 Context Caching

**Source**: https://ai.google.dev/gemini-api/docs/caching

#### Main Concepts
- **Implicit caching** is enabled by default for all Gemini 2.5 and newer models.
- Automatically passes cost savings when requests hit caches — nothing to enable.
- Supported for both **stateful** (using `previous_interaction_id`) and **stateless** conversation modes.

#### Minimum Token Count for Caching

| Model | Min token limit |
|---|---|
| Gemini 3.5 Flash | 4,096 |
| Gemini 3.1 Pro Preview | 4,096 |
| Gemini 2.5 Flash | 2,048 |
| Gemini 2.5 Pro | 2,048 |

#### Key Parameters

| Parameter | Location | Description |
|---|---|---|
| `usage.total_cached_tokens` | Response object | Number of tokens which were cache hits |
| `previous_interaction_id` | Request | Enable stateful caching across turns |

---

## 3. Quotas and Limits

### 3.1 Rate Limits

**Source**: https://ai.google.dev/gemini-api/docs/rate-limits

#### Main Concepts
- Rate limits regulate request volume within a timeframe to maintain fair usage, protect against abuse, and protect system performance.
- Limits are evaluated across **three dimensions**: RPM (requests per minute), TPM (tokens per minute, input), RPD (requests per day).
- Exceeding *any* one limit triggers a **429 RESOURCE_EXHAUSTED** error, even if others are not exceeded.
- Limits are applied **per project, not per API key**.
- **RPD quotas reset at midnight Pacific time.**
- Specified limits are **not guaranteed**; actual capacity may vary.

#### Rate Limit Dimensions

| Dimension | Description |
|---|---|
| RPM | Requests per minute |
| TPM | Tokens per minute (input) |
| RPD | Requests per day |
| IPM | Images per minute (model-specific, e.g. Nano Banana) |
| TPD | Tokens per day (some models) |

#### Spend-Based Rate Limits (rolling 10-minute window)

| Usage tier | Spend rate limit (per 10 minutes) |
|---|---|
| Free | N/A |
| Tier 1 | $10 |
| Tier 2 | $200 |
| Tier 3 | $200 |

#### Priority Inference Rate Limits
- Priority consumption has its **own rate limits** even though it counts toward overall interactive traffic limits.
- **Default: 0.3× the standard rate limit** for each model and tier.

#### Batch API Rate Limits

| Limit | Value |
|---|---|
| Concurrent batch requests | 100 |
| Input file size limit | 2 GB |
| File storage limit | 20 GB |
| Enqueued tokens per model | Varies by tier and model (see below) |

#### Batch Enqueued Tokens by Tier (selected models)

| Model | Tier 1 | Tier 2 | Tier 3 |
|---|---|---|---|
| Gemini 3.1 Pro Preview | 5,000,000 | 500,000,000 | ~1B+ |
| Gemini 3.5 Flash | 3,000,000 | — | 1,000,000,000 |
| Gemini 2.5 Pro | 5,000,000 | 500,000,000 | — |
| Gemini 2.5 Flash | 3,000,000 | — | — |
| Gemini 2.5 Flash Lite | 10,000,000 | — | — |
| Gemini Embedding | 500,000 | 5,000,000 | 10,000,000 |

#### Tier Upgrade Request
- URL: https://forms.gle/ETzX94k8jf7iSotH9
- No guarantees of approval.

---

## 4. Batch Operations

### 4.1 Batch API

**Source**: https://ai.google.dev/gemini-api/docs/batch-api

#### Main Concepts
- Batch API processes large volumes of requests **asynchronously at 50% of standard cost**.
- **Target turnaround: 24 hours** (often faster).
- Use for large-scale, non-urgent tasks: data pre-processing, running evaluations.
- Two submission modes: **inline requests** (small sets, < 20MB for images) and **input file** (large sets via File API, JSONL format up to 2GB).
- Returns a **job name** used for monitoring and retrieving results.
- Results retained for **6 weeks**, then permanently deleted.
- Supports batch image generation (Nano Banana) and batch embeddings.

#### API Methods and Endpoints

| Method | SDK | REST Endpoint | Description |
|---|---|---|---|
| Create batch | `client.batches.create(model, src, config)` / `ai.batches.create({model, src, config})` | `POST /v1beta/models/{model}:batchGenerateContent` | Create a batch job |
| Create embeddings batch | `client.batches.create_embeddings(model, src, config)` / `ai.batches.createEmbeddings(...)` | — | Batch embeddings job |
| Get (poll) | `client.batches.get(name)` / `ai.batches.get(name)` | `GET /v1beta/{BATCH_NAME}` | Poll job status |
| List | `client.batches.list()` / `ai.batches.list()` | `GET /v1beta/batches` | List recent batch jobs |
| Cancel | `client.batches.cancel(name)` / `ai.batches.cancel(name)` | `POST /v1beta/{BATCH_NAME}:cancel` | Cancel ongoing job |
| Delete | `client.batches.delete(name)` / `ai.batches.delete(name)` | `DELETE /v1beta/{BATCH_NAME}` | Delete a job |
| Upload file | `client.files.upload(file, config)` / `ai.files.upload(...)` | `POST /upload/v1beta/files` | Upload input JSONL |
| Download result | `client.files.download(file)` / `ai.files.download(...)` | `GET /download/v1beta/{file_name}:download?alt=media` | Download result file |

#### Key Parameters

| Parameter | Description |
|---|---|
| `model` | Model name (e.g. `gemini-3.5-flash`) |
| `src` | Either a list of inline request dicts, or a file name string; for embeddings: `{file_name}` or `{inlined_requests}` |
| `config.display_name` | Human-readable job name |
| `config.page_size` | Pagination (list) |
| `metadata.key` | Per-request key to correlate responses |
| Each request | Full `GenerateContentRequest` fields: `contents`, `generation_config`/`config`, `tools`, `response_modalities`, `response_mime_type`, `response_schema` |

#### Batch Job States (Lifecycle)

```
JOB_STATE_PENDING → JOB_STATE_RUNNING → 
  JOB_STATE_SUCCEEDED | JOB_STATE_FAILED | JOB_STATE_CANCELLED | JOB_STATE_EXPIRED (after 48h)
```

#### Input/Output Configuration

| Config | Inline mode | File mode |
|---|---|---|
| Input | `batch.input_config.requests.requests[]` | `batch.input_config.file_name` (JSONL) |
| Output | `dest.inlined_responses` | `dest.file_name` (downloadable) |
| JSONL format | Each line: `{"key": "...", "request": {GenerateContentRequest}}` | — |
| Max file size | — | 2 GB |

#### Embeddings Batch Output
- `dest.inlined_embed_content_responses` / `inlinedEmbedContentResponses`

---

## 5. Logs and Data Retention

### 5.1 Logs and Datasets

**Source**: https://ai.google.dev/gemini-api/docs/logs-datasets

#### Main Concepts
- Logging lets you view Gemini API usage in the AI Studio dashboard to understand model behavior and user interaction. Optionally share usage feedback with Google.
- **Logs storage only available for paid-tier projects.**
- Supported calls: `GenerateContent`, `BatchGenerateContent`, `StreamGenerateContent`, and Interactions API calls (excluding Managed Agents). Includes OpenAI-compatibility endpoints.
- **Interactions API**: stores requests **by default** (`store=true`).
- **Generate Content API**: does **NOT** store requests by default (`store=false`).
- Toggle logging in AI Studio Settings panel, independently for `generateContent` and Interactions API.

#### Request-Level Logging Parameter

| API | Parameter | Default | How to enable |
|---|---|---|---|
| `generateContent` | `config={'store': True}` (Python) / `config: { store: true }` (JS) | `False` | Set `store=True` |
| Interactions | `store=True` (Python/JS) | `True` | Set `store=False` to disable |

#### Storage Retention

| Setting | Value |
|---|---|
| Default retention | 55 days |
| Configurable options | 7, 14, 28, or 55 days max |
| Logs in datasets | **No expiry** (do not expire) |
| Default dataset storage cap | 1,000 logs per project |

#### Datasets (Create & Share)

| Capability | Description |
|---|---|
| Create | Filter logs → select via checkboxes → Create dataset → name + optional description |
| Export formats | CSV, JSONL, Google Sheets |
| Use cases | Challenge sets, sample sets, evaluation sets |
| Share with Google | Optional; contributes to R&D as demonstration data |

#### What Does NOT Support Logging
- Imagen and Veo models
- Gemini embedding models
- Gemini Robotics model
- Inputs containing videos, GIFs, or PDFs
- Public Preview Agents in the Gemini API

#### Management URLs
| URL | Purpose |
|---|---|
| https://aistudio.google.com/logs | Logs and datasets page |

---

### 5.2 Data Logging and Sharing Policy

**Source**: https://ai.google.dev/gemini-api/docs/logs-policy

#### Main Concepts
- Logs are **developer-owned API data** from supported Gemini API calls for **billing-enabled projects**.
- Logging is **opt-in** by the project owner — for own use and/or feedback/sharing with Google.
- Because logging is only for billing-enabled projects, prompts/responses in logs are **NOT used for product improvement or development by default** (per Terms on data use).
- If you **choose to share datasets**, they're used as real-world demonstration data. May improve model quality, inform training/evaluation.
- Processed under "Unpaid Services" data-use terms. **Human reviewers may read, annotate, and process** shared API inputs/outputs.
- **Privacy protection**: Google disconnects data from your Google Account, API key, and Cloud project before reviewers see/annotate it.
- **Do not include personal, sensitive, or confidential information** in shared logs.

#### Key Retention and Limits

| Setting | Value |
|---|---|
| Logs expire after | 55 days by default |
| Datasets | No set expiry |
| Default per-project log cap | 1,000 logs |

#### Data Permissions
- Opting in confirms you have necessary permissions for Google to process/use data.
- Do not contribute logs containing sensitive, confidential, or proprietary information obtained through the paid service.
- License granted under "Submission of Content" section of API Terms extends to content submitted (prompts, system instructions, cached content, files) and generated responses.

---

## 6. Orchestration and Integrations

### 6.1 Webhooks

**Source**: https://ai.google.dev/gemini-api/docs/webhooks

#### Main Concepts
- Webhooks allow the Gemini API to push real-time notifications to your server when asynchronous or Long-Running Operations (LROs) complete.
- Replaces the need to poll `GET /operations` for status updates, reducing latency and overhead.
- Available for Batch jobs, Interactions API, and video generation.
- Two webhook types: **Static** (registered per project) and **Dynamic** (bound to a specific request configuration).

#### Static Webhooks API

| Method | SDK | REST Endpoint | Description |
|---|---|---|---|
| Create | `client.webhooks.create(name, subscribed_events, uri)` | `POST /v1/webhooks` | Create webhook; returns signing secret (only once) |
| Get | `client.webhooks.get(id)` | `GET /v1/webhooks/{id}` | Retrieve webhook details |
| List | `client.webhooks.list()` | `GET /v1/webhooks` | List all webhooks with pagination |
| Update | `client.webhooks.update(id, subscribed_events)` | `PATCH /v1/webhooks/{id}` | Update properties |
| Delete | `client.webhooks.delete(id)` | `DELETE /v1/webhooks/{id}` | Remove webhook |
| Rotate secret | `client.webhooks.rotate_signing_secret(id, revocation_behavior)` | `POST /v1/webhooks/{id}/rotate_secret` | Rotate signing secret |

#### Static Webhook Parameters

| Parameter | Description |
|---|---|
| `name` | Display name for webhook |
| `uri` | Target listener URL (HTTPS) |
| `subscribed_events` | Array of event types to subscribe to |
| `new_signing_secret` | Returned only once on create/rotate; store securely |
| `revocation_behavior` | `REVOKE_PREVIOUS_SECRETS_AFTER_H24` or immediate |

#### Static Webhook Verification
- Follows **Standard Webhooks** specification (https://github.com/standard-webhooks/standard-webhooks).
- Verify payload using signed header signatures and stored signing secret.
- Endpoint must respond with **2xx status code within a few seconds**.
- Automatic retries for **24 hours** using exponential backoff.

#### Dynamic Webhooks

| Concept | Description |
|---|---|
| Purpose | Bind webhook to a specific request configuration; ideal for agent-orchestration queues |
| Signature type | Asymmetric public-key **JWKS** signatures (not symmetric secrets) |
| Usage | Add `webhook_config` when triggering an async job (e.g. creating a Batch) |

#### Dynamic Webhook Parameters

| Parameter | Description |
|---|---|
| `webhook_config.uris` | Array of callback URIs |
| `webhook_config.user_metadata` | Custom metadata (e.g. `{"job_group": "...", "priority": "..."}`) |
| `background` | Must be `true` when `webhook_config` is specified |

#### Dynamic Webhook Verification (JWKS)
- Extract JWT signature from `Webhook-Signature` header.
- Verify using Google's public certificate endpoint: `https://generativelanguage.googleapis.com/.well-known/jwks.json`.
- Algorithms: `RS256`; verify audience against your configured audience.

#### Webhook Envelope (Thin Payload)

```json
{
  "type": "batch.succeeded",
  "version": "v1",
  "timestamp": "2026-01-22T12:00:00Z",
  "data": {
    "id": "batch_123456",
    "output_file_uri": "gs://my-bucket/results.jsonl"
  }
}
```

#### Event Catalog

| Event type | Trigger | Payload data |
|---|---|---|
| `batch.succeeded` | Processing finished successfully | `id`, `output_file_uri` |
| `batch.cancelled` | User cancelled request | `id` |
| `batch.expired` | Batch not processed in 24h | `id` |
| `batch.failed` | Batch job failed | `id`, `error_code`, `error_message` |
| `interaction.requires_action` | Function call, user action needed | `id` |
| `interaction.completed` | LRO in interactions succeeded | `id` |
| `interaction.failed` | LRO in interactions failed | `id`, `error_code`, `error_message` |
| `interaction.cancelled` | LRO in interactions cancelled | `id` |
| `video.generated` | Video generation LRO completed | `id`, `output_file_uri`, `file_name` |

---

### 6.2 Background Execution

**Source**: https://ai.google.dev/gemini-api/docs/background-execution

#### Main Concepts
- The Interactions API provides **background execution** to run long-running tasks asynchronously, avoiding HTTP connection timeouts (~60 seconds).
- Setting `background=true` when creating an interaction returns an **interaction ID** immediately.
- Clients can then **poll for status**, **stream progress**, or **reconnect to a disconnected stream**.
- Supported models: standard models (`gemini-3.5-flash`, `gemini-3.1-pro-preview`), Managed Agents (`antigravity-preview-05-2026`), Deep Research models.

#### Interaction States (Lifecycle)

```
in_progress → requires_action → completed | failed | cancelled
```

#### API Methods and Endpoints

| Method | SDK | REST Endpoint | Description |
|---|---|---|---|
| Create (background) | `client.interactions.create(model, input, background=True)` | `POST /v1beta/interactions` | Start async interaction; returns ID |
| Poll (non-blocking) | `client.interactions.get(id)` | `GET /v1beta/interactions/{id}` | Check status |
| Stream with reconnect | `client.interactions.get(id, stream=True, last_event_id=...)` | `GET /v1beta/interactions/{id}?stream=true&last_event_id={id}` | Stream deltas |
| Cancel | `client.interactions.cancel(id)` | `POST /v1beta/interactions/{id}/cancel` | Cancel running interaction |
| Delete | `client.interactions.delete(id)` | `DELETE /v1beta/interactions/{id}` | Remove interaction record (404 on subsequent GETs) |

#### Key Parameters

| Parameter | Description |
|---|---|
| `background` (bool) | Set `true` to run asynchronously |
| `model` | Model identifier |
| `input` | Prompt text or structured input |
| `environment` | `"remote"` — sandbox environment for managed agents |
| `previous_interaction_id` | Chain to prior interaction for multi-turn; requires prior interaction in `completed` state |
| `last_event_id` | Resume a disconnected stream from last received event |
| `event_id` | Unique ID in each streamed delta payload (for reconnection) |
| `event_type` | Stream event types: `step.delta` (text), `interaction.completed` |

#### Multi-turn Chaining Constraints
- `previous_interaction_id` chains subsequent interactions to a background conversation.
- Chaining requires the prior interaction to be in `completed` state (attempting while `in_progress` → **400 Bad Request**).
- For managed agents, pass `environment: "remote"` to maintain sandbox state across turns.

---

### 6.3 Files API

**Source**: https://ai.google.dev/gemini-api/docs/files

#### Main Concepts
- The Files API handles uploading and managing media files (audio, images, video, documents, PDFs) for use with the Gemini API.
- Supports **resumable uploads**.
- Files are **automatically deleted after 48 hours**.
- File download is **not supported** (can get metadata but not download).

#### Storage Limits

| Limit | Value |
|---|---|
| Max storage per project | 20 GB |
| Max per-file size | 2 GB |
| File retention | 48 hours (auto-delete) |
| Inline request size threshold (use Files API) | > 100 MB (50 MB for PDFs) |
| Cost | Free |

#### API Methods

| Method | Python SDK | REST Endpoint | Description |
|---|---|---|---|
| Upload | `client.files.upload(file=...)` | `POST /upload/v1beta/files` | Upload media file; returns `.uri`, `.name`, `.mime_type` |
| Get metadata | `client.files.get(name=...)` | `GET /v1beta/{name}` | Retrieve file metadata |
| List | `client.files.list()` | `GET /v1beta/files` | List all uploaded files |
| Delete | `client.files.delete(name=...)` | `DELETE /v1beta/{name}` | Manually delete a file |

#### Resumable Upload Flow (REST)

| Step | Headers | Description |
|---|---|---|
| 1. Initiate | `X-Goog-Upload-Protocol: resumable`, `X-Goog-Upload-Command: start`, `X-Goog-Upload-Header-Content-Length`, `X-Goog-Upload-Header-Content-Type` | Returns upload URL in `x-goog-upload-url` header |
| 2. Upload bytes | `Content-Length`, `X-Goog-Upload-Offset: 0`, `X-Goog-Upload-Command: upload, finalize` | Upload actual bytes to upload URL |

#### File Object Fields
| Field | Description |
|---|---|
| `.name` | File identifier |
| `.uri` | File URI for use in interactions |
| `.mime_type` / `.mimeType` | MIME type |

#### Using Files in Interactions
Files referenced in `interactions.create` input as structured parts:
```python
{"type": "audio", "uri": myfile.uri, "mime_type": myfile.mime_type}
```

---

### 6.4 Partner and Library Integrations

**Source**: https://ai.google.dev/gemini-api/docs/partner-integration

#### Main Concepts
- Guide for **partners** building libraries, platforms, gateways on top of the Gemini API.
- Four partner archetypes: Ecosystem framework, Runtime and edge platform, Aggregator, Enterprise gateway.
- Three integration paths: Google GenAI SDK, Direct API (REST/gRPC), OpenAI compatibility layer.
- **All partners MUST send the `x-goog-api-client` header** regardless of path chosen.

#### Integration Paths Comparison

| Partner type | Recommended path | Key benefit | Key trade-off |
|---|---|---|---|
| Enterprise gateway, ecosystem framework | **Google GenAI SDK** | GCP parity & speed; built-in types/auth/file uploads; seamless migration | Dependency weight; limited languages (Python/Node/Go/Java) |
| Ecosystem framework, edge platforms, aggregators | **Direct API** (REST/gRPC) | Zero deps; full control; full feature access | High dev overhead; manual validation |
| Aggregator (text-only, legacy) | **OpenAI compatibility** | Instant portability; reuse OpenAI-compatible code | Feature ceiling — no native video, caching, etc. |

#### Key Endpoints

| Endpoint | Purpose |
|---|---|
| `https://generativelanguage.googleapis.com/$discovery/OPENAPI3_0` | OpenAPI spec for type generation |
| `https://generativelanguage.googleapis.com/v1beta/models/{model}:generateContent?key=...` | Direct API example |
| `https://generativelanguage.googleapis.com/v1beta/openai/` | OpenAI-compat base URL |

#### Required Header
| Header | Format | Purpose |
|---|---|---|
| `x-goog-api-client` | `company-product/version` (e.g. `acme-framework/1.2.0`) | Mandatory client identification; lets Google identify traffic and proactively debug |

#### SDK Compatibility
- **GenAI SDK supported languages**: Python, Node (JavaScript/TypeScript), Go, Java.
- Direct API works with any HTTP client.
- OpenAI-compat works with any OpenAI SDK / OpenAI-compatible library.

---

### 6.5 Migrate to Cloud

**Source**: https://ai.google.dev/gemini-api/docs/migrate-to-cloud

#### Main Concepts
Two API products for building generative AI with Gemini:
1. **Gemini Developer API** — fastest path to build, productionize, scale. **Recommended default** for most developers.
2. **Gemini Enterprise Agent Platform API** — comprehensive enterprise-ready features backed by Google Cloud Platform (GCP). Use when you need enterprise controls.
- Both accessible through the **Google Gen AI SDK** (one library, switch via config flags).
- Model IDs are **identical** across both APIs.

#### Client Configuration Comparison

| Language | Developer API | Enterprise Agent Platform |
|---|---|---|
| Python | `genai.Client()` | `genai.Client(vertexai=True, project='your-project-id', location='us-central1')` |
| JavaScript | `new GoogleGenAI({})` | `new GoogleGenAI({ vertexai: true, project: 'your_project', location: 'your_location' })` |
| Go | `genai.NewClient(ctx, nil)` | `genai.NewClient(ctx, &ClientConfig{Project, Location, Backend: genai.BackendVertexAI})` |

#### Migration Considerations
- **Auth migration**: API key → Google Cloud **service accounts**.
- Reuse existing GCP project or create new one.
- **Supported regions differ** between Developer API and Enterprise Agent Platform.
- **Models created in Google AI Studio must be retrained** in Enterprise Agent Platform (no automatic transfer).
- Delete unused API key via Cloud Console (`console.cloud.google.com/apis/credentials`); deletion propagates in minutes.
- Recover deleted key via `gcloud beta services api-keys undelete`.

---

## 7. Performance Optimization

### 7.1 Optimization Overview

**Source**: https://ai.google.dev/gemini-api/docs/optimization

#### Main Concepts
The Gemini API offers optimization mechanisms to balance speed, cost, and reliability.

#### Optimization Modes Comparison

| Feature | Standard | Flex | Priority | Batch | Caching |
|---|---|---|---|---|---|
| **Pricing** | Full Price | 50% discount | 75-100% more | 50% discount | 90% discount + prorated storage |
| **Latency** | Seconds to minutes | Minutes (1-15 min target) | Seconds | Up to 24 hours | Faster time-to-first-token |
| **Reliability** | High / Medium-high | Best-effort (Sheddable) | High (Non-sheddable) | High (throughput) | N/A |
| **Interface** | Synchronous | Synchronous | Synchronous | Asynchronous | Saved state |
| **Best use case** | General workflows | Non-urgent sequential chains | Production, user-facing apps | Massive datasets, offline evals | Recurring queries over same file |

#### Inference Service Tiers (Synchronous)
- Shift between reliability-optimized and cost-optimized traffic by passing `service_tier` parameter in standard generation calls.
- **Standard** (default): normal response times, no premium.
- **Priority**: high-criticality compute queues, non-sheddable. Graceful downgrade to Standard if limits exceeded.
- **Flex**: 50% discount, off-peak "sheddable" capacity, synchronous. May be preempted during standard traffic spikes.

---

### 7.2 Flex Inference

**Source**: https://ai.google.dev/gemini-api/docs/flex-inference

#### Main Concepts
- Flex inference offers a **50% cost reduction** vs standard rates for latency-tolerant workloads requiring synchronous processing.
- Uses **off-peak, "sheddable" compute capacity**.
- Traffic counts towards general rate limits (no extended limits like Batch API).

#### API Usage
Set `service_tier` to `flex` in request:

| Method | Parameter |
|---|---|
| `client.interactions.create(model, input, service_tier='flex')` (Python) | `service_tier='flex'` |
| `ai.interactions.create({model, input, service_tier: 'flex'})` (JS) | `service_tier: 'flex'` |
| `POST /v1beta/interactions` (REST) | Body: `"service_tier": "flex"` |

#### Key Parameters

| Parameter | Description |
|---|---|
| `service_tier` | Set to `flex` to use Flex tier |
| `timeout` (client config) | Must cover Flex wait queues (e.g. 600s+ / 900000ms) |

#### Error Codes
- When Flex capacity unavailable, API returns **503 UNAVAILABLE**.
- Implement retries with exponential backoff; fall back to Standard if exhausted.

#### Supported Models
Gemini 3.5 Flash, Gemini 3.1 Flash-Lite, Gemini 3.1 Pro Preview, Gemini 3 Flash Preview, Gemini 2.5 Pro, Gemini 2.5 Flash, Gemini 2.5 Flash-Lite.

---

### 7.3 Priority Inference

**Source**: https://ai.google.dev/gemini-api/docs/priority-inference

#### Main Concepts
- Priority inference is a **premium tier** for business-critical workloads requiring lower latency and highest reliability.
- Traffic is **prioritized above Standard and Flex** tier traffic.
- **Non-sheddable**: never preempted by other tiers.
- **Graceful downgrade**: if Priority limits exceeded, overflow requests are automatically downgraded to Standard processing (billed at Standard rate, not Priority premium).
- Response header `x-gemini-service-tier` indicates actual tier used (`priority` or `standard`).

#### API Usage
Set `service_tier` to `priority` in request:

| Method | Parameter |
|---|---|
| `client.interactions.create(model, input, service_tier='priority')` (Python) | `service_tier='priority'` |
| `ai.interactions.create({model, input, service_tier: 'priority'})` (JS) | `service_tier: 'priority'` |
| `POST /v1beta/interactions` (REST) | Body: `"service_tier": "priority"` |

#### Rate Limits
- Priority consumption has its **own rate limits**: **0.3× standard rate limit** per model and tier.
- Counts towards overall interactive traffic rate limits.

#### Supported Models
Same as Flex inference: Gemini 3.5 Flash, Gemini 3.1 Flash-Lite, Gemini 3.1 Pro Preview, Gemini 3 Flash Preview, Gemini 2.5 Pro, Gemini 2.5 Flash, Gemini 2.5 Flash-Lite.

---

## 8. Versioning and Deprecation

### 8.1 Release Notes (Changelog)

**Source**: https://ai.google.dev/gemini-api/docs/changelog

#### Main Concepts
- Single-page release log organized by date (newest first), spanning **December 13, 2023 → June 30, 2026**.
- **API version channels**:
  - `v1`: stable API channel
  - `v1beta`: beta channel with features that may be under development
- **Model lifecycle stages** (introduced Jan 12, 2026):
  - **Experimental** (`-exp-`)
  - **Preview** (`-preview`)
  - **Stable / GA** (generally available, often `-001` or versionless)
- **Latest alias models**: `gemini-pro-latest`, `gemini-flash-latest` — switch as new models GA.
- **Interactions API**: generally available; recommended for all latest features/models.
- **Breaking changes** tracked explicitly with "Upcoming breaking change" tags.

#### Key Versioning Parameters

| Parameter | Description |
|---|---|
| API version | `v1` (stable) or `v1beta` (beta) in URL path |
| `Api-Revision` header | e.g. `2026-05-20` — specifies API revision to avoid breaking changes |
| Model lifecycle stage | Experimental / Preview / Stable — indicated by model ID suffix |

#### Notable Breaking Changes
| Date | Change |
|---|---|
| Dec 19, 2025 | `total_reasoning_tokens` → `total_thought_tokens` (v1beta) |
| May 6, 2026 | Interactions API schema `outputs` → `steps`; `response_format` change (default May 26, removed June 8) |

#### Key Feature Releases (selected)

| Date | Release |
|---|---|
| Dec 13, 2023 | Initial Gemini launch (`gemini-pro`, `gemini-pro-vision`, `embedding-001`, `aqa`); `v1`/`v1beta` channels |
| Feb 6, 2025 | Google Gen AI SDK for Python GA |
| Mar 12, 2025 | Project-level spend caps |
| Apr 17, 2025 | Tuning shut down on all models |
| Jun 5, 2025 | `gemini-2.5-pro` & `gemini-2.5-flash` stable |
| Nov 4, 2025 | File Search API public preview |
| Dec 11, 2025 | Interactions API launched |
| Jan 12, 2026 | Model lifecycle feature |
| Apr 1, 2026 | Flex & Priority inference tiers |
| May 4, 2026 | Webhooks |
| May 19, 2026 | `gemini-3.5-flash` GA; Managed Agents + Antigravity Agent |
| Jun 30, 2026 | `gemini-omni-flash-preview` (Omni Flash) public preview |

---

### 8.2 Deprecations

**Source**: https://ai.google.dev/gemini-api/docs/deprecations

#### Main Concepts
- Lists known **deprecation schedules** for Stable (GA) and Preview models.
- A **deprecation** is the announcement that a model is no longer supported and will be **shut down** in the near future.
- Once a model is **shutdown**, the endpoint is completely turned off.
- Deprecation announcements are made on the Release Notes page; earliest shutdown dates tracked on the Deprecations page.

#### Deprecation Schedule (selected, currently active models with shutdown dates)

| Model | Release date | Shutdown date | Recommended replacement |
|---|---|---|---|
| `gemini-3.1-flash-lite` | May 7, 2026 | May 7, 2027 | — |
| `gemini-2.5-pro` | Jun 17, 2025 | Oct 16, 2026 | `gemini-3.1-pro-preview` |
| `gemini-2.5-flash` | Jun 17, 2025 | Oct 16, 2026 | `gemini-3.5-flash` |
| `gemini-2.5-flash-image` | Oct 2, 2025 | Oct 2, 2026 | `gemini-3.1-flash-image-preview` |
| `gemini-2.5-flash-lite` | Jul 22, 2025 | Oct 16, 2026 | `gemini-3.1-flash-lite` |
| `gemini-2.0-flash` | Feb 5, 2025 | Jun 1, 2026 | `gemini-3.5-flash` |
| `gemini-2.0-flash-lite` | Feb 25, 2025 | Jun 1, 2026 | `gemini-3.1-flash-lite` |
| `imagen-4.0-generate-001` | Jun 24, 2025 | Aug 17, 2026 | `gemini-3.1-flash-image` |
| `veo-3.0-generate-001` | Sep 9, 2025 | Jun 30, 2026 | `veo-3.1-generate-preview` |
| `veo-2.0-generate-001` | Apr 9, 2025 | Jun 30, 2026 | `veo-3.1-generate-preview` |
| `gemini-embedding-001` | Jul 14, 2025 | Jul 14, 2026 | `gemini-embedding-2` |

#### Models with no shutdown date announced (current GA/active)
- `gemini-3.5-flash`, `gemini-3.1-flash-image`, `gemini-3-pro-image`, `gemini-3.1-pro-preview`, `gemini-embedding-2`, `veo-3.1-generate-preview`, `lyria-3-clip-preview`, `lyria-3-pro-preview`, `gemini-robotics-er-1.6-preview`

---

## 9. License and Usage Rules

### 9.1 Available Regions

**Source**: https://ai.google.dev/gemini-api/docs/available-regions

#### Main Concepts
- The Gemini API and Google AI Studio are available in specific countries and territories.
- If not in an available region, users are directed to the **Gemini API in Gemini Enterprise Agent Platform** (which may have different regional availability).
- AI Studio usage is **free in all available regions**.
- Further requirements detailed in Terms of Service (`/gemini-api/terms`).

#### Licensing
- Content is licensed under the **Creative Commons Attribution 4.0 License**.
- Code samples are licensed under the **Apache 2.0 License**.
- Java is a registered trademark of Oracle and/or its affiliates.
- See Google Developers Site Policies for details.

---

## 10. Tenant Isolation

There is **no dedicated documentation page** for tenant isolation within the Gemini Developer API documentation. Based on analysis across all pages, the following tenant isolation-related concepts emerge:

#### Project-Level Isolation
- Every API key is tied to a **Google Cloud project** which manages billing, collaborators, and permissions.
- Rate limits are applied **per project, not per API key** — providing isolation at the project level.
- Spend caps operate at both **project level** (project spend caps) and **billing account level** (tier spend caps aggregated across all linked projects).
- File storage is limited **per project** (20 GB max).
- Logs are stored **per project** with a 1,000 log default cap.

#### Authentication Scoping
- API keys can be **restricted** by request origin (IP, website, application) to limit exposure.
- OAuth 2.0 provides **user-level authorization** with scoped access.
- **Ephemeral tokens** can be scoped to specific models and configurations via `live_connect_constraints`.
- Service accounts (Enterprise Agent Platform) provide identity-based isolation.

#### Data Isolation
- Free tier: content **used to improve products** (not isolated from training).
- Paid tier: content **NOT used to improve products** (isolated from training).
- Shared datasets are **disconnected from your Google Account, API key, and Cloud project** before human review.
- Flagged abuse data retained 55 days for policy enforcement only.

#### Enterprise Migration
- For stronger tenant isolation, security, and compliance, Google recommends migrating to the **Gemini Enterprise Agent Platform** which provides:
  - Advanced security & compliance
  - Service account authentication
  - GCP project/region isolation
  - Provisioned throughput

---

## Cross-Reference Summary

| Concern | Primary Pages | Key Mechanism |
|---|---|---|
| Authentication | api-key, oauth, ephemeral-tokens | API key (header), OAuth 2.0 (Bearer), ephemeral tokens (scoped, short-lived) |
| Security | api-key, safety-settings, usage-policies | Key restrictions, safety filters, abuse monitoring |
| Billing | billing, pricing | Prepay/Postpay, tiers, spend caps, token-based pricing |
| Cost optimization | optimization, flex-inference, priority-inference, caching | `service_tier` parameter, Batch API, implicit caching |
| Quotas | rate-limits | RPM/TPM/RPD per project, spend-based limits, tier-based caps |
| Batch operations | batch-api | `BatchGenerateContent`, async, 50% cost, 24h turnaround |
| Logs | logs-datasets, logs-policy | `store` parameter, 55-day retention, datasets (no expiry) |
| Orchestration | webhooks, background-execution, files | Webhooks (static/dynamic), `background=true`, Files API |
| Integrations | partner-integration, migrate-to-cloud | GenAI SDK, Direct API, OpenAI compat; Developer → Enterprise |
| Versioning | changelog, deprecations | `v1`/`v1beta` channels, model lifecycle stages, deprecation schedules |
| Performance | optimization, flex-inference, priority-inference | Standard/Flex/Priority/Batch/Caching tiers |
| Tenant isolation | (distributed across pages) | Project-level isolation, auth scoping, data isolation, Enterprise migration |