# OpenAI API — Cross-Cutting Concerns Analysis

> Source: documentation reachable from https://developers.openai.com/api/docs
> Scope: cross-cutting / administrative capabilities only. Core AI capabilities (text generation, chat completions inference semantics, embeddings, fine-tuning model behavior, image/video/audio generation, realtime voice, assistants, moderation scoring, structured outputs, function calling, tools behavior, RAG/file search, evals/grading, prompt engineering, deep research, reasoning, code interpreter, computer use) are intentionally excluded.
> Generated: 2026-07-14

This document analyzes the cross-cutting capabilities exposed by the OpenAI API Platform. For each capability it lists the **main concepts**, the **API functions/endpoints**, and the **parameters** that govern behavior.

---

## Table of Contents

1. [Tenant Isolation & Data Residency](#1-tenant-isolation--data-residency)
2. [Authentication & Security](#2-authentication--security)
3. [Workload Identity Federation](#3-workload-identity-federation)
4. [Permissions & Role-Based Access Control](#4-permissions--role-based-access-control)
5. [Admin APIs (Organization & Project Management)](#5-admin-apis-organization--project-management)
6. [Cost Optimization & Billing](#6-cost-optimization--billing)
7. [Batch Operations](#7-batch-operations)
8. [Quotas & Rate Limits](#8-quotas--rate-limits)
9. [Logs, Audit & Data Retention](#9-logs-audit--data-retention)
10. [Network Security (Private Link, IP Egress, mTLS, Tunnels)](#10-network-security-private-link-ip-egress-mtls-tunnels)
11. [Orchestration & Integrations (Agents SDK, MCP, Webhooks)](#11-orchestration--integrations-agents-sdk-mcp-webhooks)
12. [Sandboxes & Code Execution Isolation](#12-sandboxes--code-execution-isolation)
13. [Guardrails, Approvals & Safety Controls](#13-guardrails-approvals--safety-controls)
14. [Performance Optimization](#14-performance-optimization)
15. [Versioning & Deprecations](#15-versioning--deprecations)
16. [API Conventions & Streaming](#16-api-conventions--streaming)

---

## 1. Tenant Isolation & Data Residency

**Source pages:** `guides/your-data`, `guides/admin-apis` (data retention), `guides/private-link`

### Main concepts

- **Organization**: top-level account container. Org-level settings (data residency default, EKM, abuse-monitoring policy) apply across all projects unless overridden per-project.
- **Project**: workspace for keys, files, and resources. Data residency is configured at the **project level**; each request carries a regional host prefix when residency is enabled.
- **Data Residency Controls**: project-level configuration controlling the location of infrastructure (storage at rest and, where supported, regional processing). Configured per-project; a domain prefix is added to the request host.
- **Regional storage vs. regional processing**: storage at rest in the selected region is distinct from inference processing location. Some regions support only storage (processing happens elsewhere).
- **System Data**: account/metadata/usage data not containing Customer Content; not subject to data residency.
- **Abuse Monitoring Logs**: logs generated from platform use, retained up to 30 days by default; may contain customer content and derived metadata.
- **Modified Abuse Monitoring (MAM)**: excludes customer content (except image/file inputs in rare cases) from abuse monitoring logs while preserving full platform capabilities.
- **Zero Data Retention (ZDR)**: excludes customer content from abuse monitoring logs AND forces `store=false` on `/v1/responses` and `/v1/chat/completions`. Some endpoints still store application state (ineligible endpoints).
- **Safety Retention**: OpenAI may make `gpt-5.5`, `gpt-5.5-pro`, and future models ineligible for ZDR/MAM for specific customers to investigate severe risk.
- **Enterprise Key Management (EKM)**: encrypt customer content at OpenAI using keys from your own external KMS (AWS KMS, GCP, Azure Key Vault). Applies to application state.
- **Extended Prompt Caching**: stores encrypted key/value tensors to GPU-local storage; required for `gpt-5.5`/pro and future models.
- **`store` parameter**: boolean controlling whether response/application state is retained (forced `false` under ZDR).

### Regional endpoints

| Region | Domain | Storage | Processing | MAM/ZDR required |
| --- | --- | --- | --- | --- |
| United States | `us.api.openai.com` | Yes | Yes | No |
| Europe (EEA+Switzerland) | `eu.api.openai.com` | Yes | Yes | Yes |
| Australia | `au.api.openai.com` | Yes | No | Yes |
| Canada | `ca.api.openai.com` | Yes | No | Yes |
| Japan | `jp.api.openai.com` | Yes | No | Yes |
| India | `in.api.openai.com` | Yes | No | Yes |
| Singapore | `sg.api.openai.com` | Yes | No | Yes |
| South Korea | `kr.api.openai.com` | Yes | No | Yes |
| United Kingdom | `gb.api.openai.com` | Yes | No | Yes |
| United Arab Emirates | `ae.api.openai.com` | Yes | Yes | Yes (extra approval) |

### Data retention per endpoint (ZDR eligibility)

| Endpoint path | ZDR eligible |
| --- | --- |
| `POST /v1/chat/completions` | Yes (with limitations; cannot `store=true` in non-US regions) |
| `POST /v1/responses` | Yes (with limitations; `computer-use-preview` only US/EU; cannot `background=true` in EU) |
| `POST /v1/embeddings` | Yes |
| `POST /v1/audio/transcriptions` | Yes |
| `POST /v1/audio/translations` | Yes |
| `POST /v1/audio/speech` | Yes |
| `POST /v1/moderations` | Yes |
| `POST /v1/completions` | Yes |
| `POST /v1/images/generations` | Yes (limitations) |
| `POST /v1/images/edits` | Yes (limitations) |
| `POST /v1/images/variations` | Yes (limitations) |
| `WS /v1/realtime` | Yes (tracing not EU-residency compliant) |
| `POST/GET /v1/conversations` | No |
| `POST/GET /v1/assistants` | No |
| `POST/GET /v1/threads` | No |
| `POST/GET /v1/files` | No |
| `POST/GET /v1/fine_tuning/jobs` | No |
| `POST/GET /v1/evals` | No |
| `POST/GET /v1/batches` | No |
| `POST /v1/videos` | No |

### Key parameters

- `store` (boolean): for `/v1/responses` and `/v1/chat/completions`. Under ZDR always treated as `false`. `/v1/responses` stores response data ≥30 days when `true`.
- `expires_after` (object): for `/v1/files`; auto-deletes files.
- `background` (boolean): for `/v1/responses`; stores response data ~10 min for polling. Cannot be `true` in EU region.
- `external_web_access` (boolean): for Responses API `web_search` tool; `false` = offline/cache-only (BAA-eligible under ZDR).
- `prompt_cache_retention` (string): `in_memory` value causes request error for `gpt-5.5`/pro+ under extended caching requirements.
- Data residency domain prefix: added as the request host (e.g. `eu.api.openai.com`) when the project has residency configured.

### EKM limitations

- Supported KMS: AWS KMS, Google Cloud (GCP), Azure Key Vault (others must sync to these).
- Not supported: `/v1/assistants`, Vision fine-tuning.

---

## 2. Authentication & Security

**Source pages:** `guides/rbac` (auth context), `guides/safety-best-practices`, `guides/production-best-practices`, `guides/ip-addresses`

### Main concepts

- **API keys**: authentication mechanism for all API requests. Must be stored securely (env vars / secret management), never in code/public repos. Keys generated before Dec 20, 2023 lack tracking (usage shows as `Untracked`).
- **Admin API keys**: separate key type (`OPENAI_ADMIN_KEY`) for administration endpoints only; cannot be used for non-admin endpoints. Created at `platform.openai.com/settings/organization/admin-keys`.
- **Safety identifiers**: privacy-preserving hashed user/session identifiers sent with requests to help OpenAI monitor/detect abuse and trace activity to individual end users. Should be a hashed username/email (to avoid sending identifying info); for non-logged-in preview users, send a session ID instead.
- **Requesting organization header**: pass a header to specify which org is billed for an API request.
- **Staging projects**: separate projects for staging vs. production environments with isolated rate/spend limits.
- **Horizontal/vertical scaling**: distribute load across multiple nodes/containers or increase single-node resources.
- **Caching**: store frequently accessed data to avoid repeated API calls.
- **Revoke compromised API keys**: prompt revocation via Security settings.

### API functions/endpoints

| Function | Method & Endpoint | Purpose |
| --- | --- | --- |
| Standard API request | Any `/v1/*` with `Authorization: Bearer $OPENAI_API_KEY` | Authenticate all API requests |
| Admin API request | Any `/v1/organization/*` with `OPENAI_ADMIN_KEY` | Authenticate admin endpoints |
| Chat Completions with safety identifier | `POST /v1/chat/completions` | Send `safety_identifier` in body |
| Realtime API ephemeral secret creation | `POST` (server-side) | Include `OpenAI-Safety-Identifier` header |
| Realtime API direct connection | `WS/WebRTC` | Include `OpenAI-Safety-Identifier` header |

### Key parameters

- `Authorization` (header, string): `Bearer <API_KEY>` — standard auth.
- `OpenAI-Organization` (header, string): specify which organization is billed for a request.
- `safety_identifier` (body, string): hashed username/email or session ID uniquely identifying each user; recommended (not required) for individual-user products.
- `OpenAI-Safety-Identifier` (header, string): same stable, privacy-preserving identifier for Realtime API; bound to the session at creation time; does not carry over between APIs/sessions.
- `max_completion_tokens` (body, integer): limit output tokens to reduce misuse.

### SDK initialization (Admin key)

```python
OpenAI(admin_api_key=os.environ["OPENAI_ADMIN_KEY"])
```
```javascript
new OpenAI({ adminAPIKey: process.env.OPENAI_ADMIN_KEY })
```

---

## 3. Workload Identity Federation

**Source pages:** `guides/workload-identity-federation` (+ provider sub-pages: `aws`, `github-actions`, `google-cloud`, `kubernetes`, `microsoft-azure`, `spiffe`)

### Main concepts

- **Workload identity provider**: describes the trusted issuer; stores the expected OIDC issuer, audience, and key source used to verify external subject tokens. Supports OIDC JWT subject tokens.
- **Service account mapping**: authorizes specific external token attributes (claims/derived attributes) to mint tokens for a particular OpenAI service account within a project.
- **Token exchange**: sends an external subject token to OpenAI and returns a short-lived OpenAI access token. Reference: `/api/reference/workload-identity-federation`.
- **OIDC subject tokens**: externally-issued JWTs (including SPIFFE JWT-SVIDs) that workloads exchange for OpenAI access tokens.
- **Attribute transformations**: optional CEL (Common Expression Language) expressions that derive custom `openai.*` attributes from token claims for mapping decisions. Root object is `assertion` (the verified JWT claim set).
- **JWKS (JSON Web Key Set)**: key source for token verification — either fetched via OIDC discovery or uploaded manually. Discovery/JWKS cached for 600 seconds; key refresh on miss; multiple keys allowed with unique non-empty `kid`.
- **Bearer credential**: the OpenAI-issued short-lived access token used to authenticate API requests.
- **Wildcard matching**: mapping values may use one trailing wildcard (non-empty prefix), e.g. `repo:example/*`. `*` alone is unsupported.
- **Mapping resolution**: OpenAI evaluates enabled mappings for a `(identity_provider_id, service_account_id)` pair and issues a token only if exactly one mapping matches all attributes; multiple matches → rejection.
- **Supported workload providers**: Kubernetes, AWS, Microsoft Azure, Google Cloud, GitHub Actions, SPIFFE/SPIRE.

### API functions/endpoints

- Token exchange endpoint: documented at `/api/reference/workload-identity-federation` (separate reference page). The OpenAI access token is consumed as a bearer token on normal OpenAI API requests.
- This guide page is conceptual/configuration-oriented (dashboard configuration).

### Key parameters — Workload Identity Provider configuration

| Parameter | Type | Description |
| --- | --- | --- |
| `Name` | string | Unique name for the Workload Identity Provider in the organization |
| `OIDC Issuer URL` | string (URL) | Expected OIDC issuer URL; trailing slash ignored in comparison |
| `Audience` | string | Expected `aud` claim on the external subject token |
| `Description` | string (optional) | Description for the provider |
| `Use uploaded JWKS for token verification` | boolean | When enabled, verifies against uploaded JWKS instead of OIDC discovery |
| `JWKS JSON` | object (JWKS) | Uploaded public JWKS with non-empty `keys` array and no private key material |
| `Attribute transformations` | array of CEL expressions | Derive custom `openai.*` attributes from claims |

### Key parameters — Service account mapping configuration

| Parameter | Type | Description |
| --- | --- | --- |
| `Name` | string | Unique mapping name within the Workload Identity Provider |
| `Key` | string | Attribute key to match — raw claim (`sub`, `aud`, `iss`) or derived attribute (`openai.subject`) |
| `Value` | scalar string (with optional trailing wildcard) | Attribute value that must match before token issuance |
| `Description` | string (optional) | Mapping description |
| `Project` | string/object | Project that owns the target service account |
| `Service account` | string/object | Service account the workload can use (new or existing) |
| `Permissions` | array (optional) | API permissions (OAuth scopes) narrowing access tokens minted from this mapping |

### CEL transformation object shape

| Field | Type | Description |
| --- | --- | --- |
| `attribute` | string | Derived attribute name (e.g. `openai.subject`, `openai.repository_ref`) |
| `expression` | string (CEL) | CEL expression over `assertion` (the JWT claim set), e.g. `assertion.sub` |

---

## 4. Permissions & Role-Based Access Control

**Source pages:** `guides/rbac`

### Main concepts

- **Organization**: top-level account. Org roles grant access across all projects.
- **Project**: workspace for keys/files/resources. Project roles grant access only within that project.
- **Groups**: collections of users assignable to roles; syncable from IdP via SCIM.
- **Roles**: bundles of permissions (e.g., Models Request, Files Write). Created at org or project level; assignable to users/groups. Multiple roles union into effective access.
- **Permissions**: specific actions a role allows (Read/Write/Request/Use/Manage).
- **Preset roles**: Org owner, Org reader, Project owner, Project member, Project viewer.
- **Custom roles**: eligible permissions marked in the permissions table.
- **API key permissions**: API keys carry permissions (e.g., `api.model.read`); user must also hold a project role granting that permission.
- **SCIM**: syncs group membership from identity provider.
- **Union evaluation**: effective permissions = union of all org + project roles (direct and via groups).
- **Propagation**: role/group changes take up to 30 minutes to propagate.

### Permissions catalog

| Area | What it allows | Levels | Custom-eligible |
| --- | --- | --- | --- |
| List models | List models org has access to | Read | Yes |
| Groups | View/manage groups | Read, Write | (Read for viewers) |
| Roles | View/manage roles | Read, Write | (Read for viewers) |
| Organization Admin | Manage org users, projects, invites, admin API keys, rate limits | Read, Write | (org owner only) |
| Usage | View usage dashboard/export | Read | Yes |
| External Keys | Manage EKM keys | Read, Write | (org owner) |
| IP allowlist | View/manage IP allowlist | Read, Write | (org owner) |
| mTLS | View/manage mutual TLS settings | Read, Write | (org owner) |
| OIDC | View/manage OIDC config | Read, Write | (org owner) |
| Model capabilities | Requests to chat/audio/embeddings/images | Request | Yes |
| Assistants | Create/retrieve Assistants | Read, Write | Yes |
| Threads | Create/retrieve Threads/Messages/Runs | Read, Write | Yes |
| Evals | Create/retrieve/delete Evals | Read, Write | Yes |
| Fine-tuning | Create/retrieve fine-tuning jobs | Read, Write | Yes |
| Files | Create/retrieve files | Read, Write | Yes |
| Vector Stores | Create/retrieve vector stores | Read, Write | Yes |
| Responses API | Create responses | Read, Write | Yes |
| Prompts | Create/retrieve prompts | Read, Write | Yes |
| Webhooks | Create/view webhooks | Read, Write | Yes |
| Datasets | Create/retrieve Datasets | Read, Write | Yes |
| Apps | Create/manage/submit apps | Read, Write | Yes |
| Tunnels | Inspect/use/manage org tunnels | Read, Use, Manage | Yes |
| Project API Keys | User manages own API keys | Read, Write | Yes |
| Project Administration | Manage project users/service accounts/keys/rate limits via management API | Read, Write | (project owner) |
| Batch | Create/manage batch jobs | Read, Write | (Read for viewers) |
| Service Accounts | View/manage project service accounts | Read, Write | (project owner) |
| Videos | Create/retrieve videos | Read, Write | (Read for viewers) |
| Voices | Create/retrieve voices | Read, Write | Yes |
| Agent Builder | Create/manage agents/workflows | Read, Write | Yes |

---

## 5. Admin APIs (Organization & Project Management)

**Source pages:** `guides/admin-apis`

### Main concepts

- **Admin API keys**: separate key type (`OPENAI_ADMIN_KEY`) for administration endpoints only; cannot be used for non-admin endpoints.
- **Admin API reference resources**: Admin API keys, Invites, Users, Projects, Audit Logs (under `/api/reference/administration/overview`).
- **Project model permissions**: allowlist/denylist of model IDs per project.
- **Spend limit alerts**: per-project spend alerts; threshold in cents; email notification channel.
- **Data retention controls (project)**: override or inherit org retention policy per project.
- **Invites**: send org invitations by email with a role.
- **Audit logs**: list recent user actions and configuration changes for the org.
- **SDK support**: Node 6.36.0, Python 2.34.0, Go 3.34.0, Ruby 0.61.0, Java 4.34.0+.

### API functions/endpoints

| Function | SDK method (Python) | Inferred REST | HTTP method |
| --- | --- | --- | --- |
| Update project model permissions | `client.admin.organization.projects.model_permissions.update(project_id, ...)` | `/v1/organization/projects/{project_id}/model_permissions` | PATCH/PUT |
| Create project spend alert | `client.admin.organization.projects.spend_alerts.create(project_id, ...)` | `/v1/organization/projects/{project_id}/spend_alerts` | POST |
| Update project data retention | `client.admin.organization.projects.data_retention.update(project_id, ...)` | `/v1/organization/projects/{project_id}/data_retention` | PATCH/PUT |
| Create org invite | `client.admin.organization.invites.create(...)` | `/v1/organization/invites` | POST |
| List audit logs | `client.admin.organization.audit_logs.list(...)` | `/v1/organization/audit_logs` | GET |

### Key parameters

**1. Model Permissions Update**

| Parameter | Type | Description |
| --- | --- | --- |
| `project_id` | string (path) | e.g. `"proj_abc"` |
| `mode` | enum: `"allow_list"` \| `"deny_list"` | allow only listed models, or block listed models |
| `model_ids` | array\<string\> | model IDs (must be visible to org, incl. fine-tuned snapshots), e.g. `["gpt-4.1", "o3"]` |

**2. Spend Alert Create**

| Parameter | Type | Description |
| --- | --- | --- |
| `project_id` | string (path) | — |
| `currency` | enum | e.g. `"USD"` |
| `interval` | enum | e.g. `"month"` |
| `threshold_amount` | int (cents) | e.g. `50000` = $500.00 |
| `notification_channel` | object | `{type: "email", recipients: [...], subject_prefix: "..."}` |

**3. Data Retention Update**

| Parameter | Type | Description |
| --- | --- | --- |
| `project_id` | string (path) | — |
| `retention_type` | enum: `"organization_default"` \| `"zero_data_retention"` \| `"modified_abuse_monitoring"` \| `"none"` | `organization_default` inherits org setting |

**4. Invite Create**

| Parameter | Type | Description |
| --- | --- | --- |
| `email` | string | invitee email, e.g. `"user@example.com"` |
| `role` | string/enum | e.g. `"reader"` |

**5. Audit Logs List**

| Parameter | Type | Description |
| --- | --- | --- |
| `limit` | int | max number of logs to return, e.g. `10` |

---

## 6. Cost Optimization & Billing

**Source pages:** `guides/cost-optimization`, `guides/flex-processing`, `guides/priority-processing`, `pricing`, `guides/prompt-caching`

### Main concepts

- **Processing tiers**: Standard, Batch (50% discount), Flex (discounted, flexible capacity), Priority (uplift, faster, more consistent latency).
- **`service_tier`**: request parameter controlling the processing tier (`"flex"`, `"priority"`, `"auto"`, `"default"`). Response echoes the tier actually used.
- **Flex processing**: beta-tier service mode that trades slower response times and occasional resource unavailability for lower (Batch API-rate) pricing. Ideal for non-production / lower-priority tasks (evals, data enrichment, async workloads).
- **Priority processing**: delivers significantly lower and more consistent latency vs Standard while keeping pay-as-you-go flexibility. Ideal for high-value, user-facing apps with regular traffic. Not for data processing, evaluations, or erratic traffic.
- **Ramp rate limit** (Priority): if traffic ramps too quickly, Priority requests may be downgraded to Standard (billed at Standard rates; response shows `service_tier="default"`). Currently applies at ≥1 million TPM and >50% TPM increase within 15 minutes.
- **Prompt caching**: OpenAI routes API requests to servers that recently processed the same prompt prefix. Reduces latency up to 80% and input token costs up to 90%. Automatic (no code changes), no additional fees. Enabled for `gpt-4o` and newer.
- **Cache routing**: requests routed to a machine based on a hash of the initial prompt prefix (typically first 256 tokens). `prompt_cache_key` combines with prefix hash to influence routing.
- **Cache retention policies**:
  - **In-memory**: held in volatile GPU memory; 5–10 min of inactivity, up to 1 hour max.
  - **Extended (`24h`)**: offloads key/value tensors to GPU-local storage when memory is full; up to 24 hours max. Available for: `gpt-5.5`, `gpt-5.5-pro`, `gpt-5.4`, `gpt-5.2`, `gpt-5.1` family, `gpt-5` family, `gpt-4.1`.
- **Prices per 1M tokens** (text/multimodal) unless noted otherwise; video priced per second; realtime audio sometimes per minute.
- **Short context vs. Long context**: different input/output prices by context window tier.
- **Cached input / Cache writes**: distinct pricing for cached prompt input and writing to cache.
- **Regional processing uplift**: 10% uplift for models released on/after March 5, 2026, eligible for data residency.
- **GB definition**: binary gigabytes (gibibytes), 1 GB = 2^30 bytes.

### API functions/endpoints

| Function | Method & Endpoint | Purpose |
| --- | --- | --- |
| Create response (flex) | `POST /v1/responses` with `service_tier: "flex"` | Flex-processed response |
| Create chat completion (flex) | `POST /v1/chat/completions` with `service_tier: "flex"` | Flex-processed completion |
| Create response (priority) | `POST /v1/responses` with `service_tier: "priority"` | Priority-processed response |
| Create chat completion (priority) | `POST /v1/chat/completions` with `service_tier: "priority"` | Priority-processed completion |
| Create response (with caching) | `POST /v1/responses` with `prompt_cache_key`, `prompt_cache_retention` | Prompt-cached response |
| Create chat completion (with caching) | `POST /v1/chat/completions` with `prompt_cache_key`, `prompt_cache_retention` | Prompt-cached completion |

### Key parameters

| Parameter | Type | Description |
| --- | --- | --- |
| `service_tier` | enum (`"flex"` \| `"priority"` \| `"auto"` \| `"default"`) | Request-level opt-in to processing tier. Omit to use Project default. Response echoes the tier actually used |
| `prompt_cache_key` | string | Combined with prefix hash to influence cache routing; use consistently across requests sharing prefixes; keep each prefix-key combination below ~15 RPM to avoid overflow |
| `prompt_cache_retention` | enum (`"in_memory"` \| `"24h"`) | Cache retention policy. For `gpt-5.5`/`gpt-5.5-pro`/future models only `24h` supported. Default depends on org ZDR policy |
| `timeout` | number | SDK-level request timeout for flex requests (recommended `900.0` / 15 min); curl uses `--max-time 900` |

### Response usage object

| Field | Type | Description |
| --- | --- | --- |
| `usage.prompt_tokens` | integer | Total prompt tokens |
| `usage.completion_tokens` | integer | Completion tokens |
| `usage.total_tokens` | integer | Total tokens |
| `usage.prompt_tokens_details.cached_tokens` | integer | Number of prompt tokens that were a cache hit (0 if < 1024 tokens) |
| `usage.completion_tokens_details.reasoning_tokens` | integer | Reasoning tokens |
| `usage.completion_tokens_details.accepted_prediction_tokens` | integer | Accepted predicted output tokens |
| `usage.completion_tokens_details.rejected_prediction_tokens` | integer | Rejected predicted output tokens |

### Pricing reference (selected, per 1M tokens, Standard tier, short context)

| Model | Input | Cached input | Cache writes | Output | Long-ctx Input | Long-ctx Output |
| --- | --- | --- | --- | --- | --- | --- |
| `gpt-5.6-sol` | $5.00 | $0.50 | $6.25 | $30.00 | $10.00 | $45.00 |
| `gpt-5.6-terra` | $2.50 | $0.25 | $3.125 | $15.00 | $5.00 | $22.50 |
| `gpt-5.6-luna` | $1.00 | $0.10 | $1.25 | $6.00 | $2.00 | $9.00 |
| `gpt-5.5` | $5.00 | $0.50 | – | $30.00 | $10.00 | $45.00 |
| `gpt-5.5-pro` | $30.00 | – | – | $180.00 | $60.00 | $270.00 |
| `gpt-5.4` | $2.50 | $0.25 | – | $15.00 | $5.00 | $22.50 |
| `gpt-5.4-mini` | $0.75 | $0.075 | – | $4.50 | – | – |
| `gpt-5.4-nano` | $0.20 | $0.02 | – | $1.25 | – | – |

Batch tier = 50% of Standard. Flex tier = 50% of Standard. Priority tier (short context only): e.g. `gpt-5.6-sol` Input $10.00, Output $60.00.

### Tools pricing

| Tool | Pricing |
| --- | --- |
| Web search (all models) | $10.00 / 1k calls + search content tokens at model rates |
| Web search preview (non-reasoning models) | $25.00 / 1k calls + search content tokens free |
| Containers (Hosted Shell + Code Interpreter) | 1 GB $0.03, 4 GB $0.12, 16 GB $0.48, 64 GB $1.92 per 20-min session per container (5-min min, billed by minute) |
| File search storage | $0.10 / GB/day (1 GB free) |
| File search tool call | $2.50 / 1k calls (Responses API only) |

### Error responses (Flex)

- `408 Request Timeout` — auto-retried twice by SDKs.
- `429 Resource Unavailable` — no charge; retry with exponential backoff (stay on flex) or retry with `service_tier: "auto"` (fall back to standard).

---

## 7. Batch Operations

**Source pages:** `guides/batch`

### Main concepts

- **Batch API**: collect a set of requests into one `.jsonl` file, kick off a batch job, poll status, retrieve results. Benefits: 50% cost discount vs synchronous APIs; substantially higher separate rate-limit pool; 24-hour completion window.
- **`.jsonl` input file**: one JSON object per line, each line = one request; must target a single model per file; max 200 MB; max 50,000 requests per batch (embeddings also capped at 50,000 embedding inputs across the whole batch).
- **`custom_id`**: unique per-request identifier used to map input → output (output order may differ from input order).
- **Batch object**: the returned resource tracking a batch's lifecycle.
- **Batch status lifecycle**: `validating` → `failed` / `in_progress` → `finalizing` → `completed` (or `expired`); cancellation states `cancelling` → `cancelled`.
- **Completion window**: currently only `24h`.
- **Output file**: `.jsonl` with one response per successful input request; auto-deleted 30 days after completion.
- **Error file**: failed requests written to `error_file_id`; expired requests written here with `code: "batch_expired"`.
- **Batch creation rate limit**: up to 2,000 batches/hour; separate from per-model standard limits.
- **Batch expiration**: incomplete batches move to `expired`; completed requests still billed and made available; unfinished requests cancelled.

### Supported target endpoints

Each becomes the `url` field in an input line: `/v1/responses`, `/v1/chat/completions`, `/v1/embeddings`, `/v1/completions`, `/v1/moderations`, `/v1/images/generations`, `/v1/images/edits`, `/v1/videos`.

### API functions/endpoints

| Function | Method & Endpoint | Purpose |
| --- | --- | --- |
| Upload input file | `POST /v1/files` | Upload the `.jsonl` batch input file |
| Create batch | `POST /v1/batches` | Kick off a batch job |
| Retrieve batch status | `GET /v1/batches/{batch_id}` | Poll batch lifecycle |
| Retrieve batch results | `GET /v1/files/{file_id}/content` | Download the `.jsonl` output |
| Cancel a batch | `POST /v1/batches/{batch_id}/cancel` | Cancel an in-flight batch |
| List batches | `GET /v1/batches` | Paginated list of batches |

### Key parameters

**Upload input file (`POST /v1/files`)**

| Parameter | Type | Description |
| --- | --- | --- |
| `file` | file stream | the `.jsonl` batch input file |
| `purpose` | string | must be `"batch"` |

**Create batch (`POST /v1/batches`)**

| Parameter | Type | Description |
| --- | --- | --- |
| `input_file_id` | string (required) | File ID from the upload step (e.g. `file-abc123`) |
| `endpoint` | string (required) | target endpoint path (e.g. `/v1/chat/completions`) |
| `completion_window` | string (required) | only `"24h"` currently supported |
| `metadata` | object (optional) | arbitrary key/value metadata (e.g. `{"description": "nightly eval job"}`) |

**Batch object fields (response)**: `id`, `object` (`"batch"`), `endpoint`, `errors`, `input_file_id`, `completion_window`, `status`, `output_file_id`, `error_file_id`, `created_at`, `in_progress_at`, `expires_at`, `completed_at`, `failed_at`, `expired_at`, `request_counts` (`{total, completed, failed}`), `metadata`.

**List batches (`GET /v1/batches`)**

| Parameter | Type | Description |
| --- | --- | --- |
| `limit` | query, int (optional) | pagination page size |
| `after` | query (optional) | pagination cursor |

**Input-line schema (per line of the `.jsonl`)**

| Field | Type | Description |
| --- | --- | --- |
| `custom_id` | string (required) | unique request reference |
| `method` | string (required) | e.g. `"POST"` |
| `url` | string (required) | target endpoint path |
| `body` | object (required) | same parameters as the underlying endpoint; for `/v1/moderations` must include `input` |

**Output-line schema**

| Field | Type | Description |
| --- | --- | --- |
| `id` | string | e.g. `batch_req_123` |
| `custom_id` | string | echoes the input `custom_id` |
| `response` | object\|null | `{status_code, request_id, body}` |
| `error` | object\|null | `{code, message}`; e.g. `{"code": "batch_expired"}` |

---

## 8. Quotas & Rate Limits

**Source pages:** `guides/rate-limits`

### Main concepts

- **Rate limit metrics**: RPM (requests per minute), RPD (requests per day), TPM (tokens per minute), TPD (tokens per day), IPM (images per minute), audio minutes per minute (streaming audio). Limit hit = whichever metric is reached first.
- **Org/project level**: rate limits defined at organization and project level (not user level).
- **Per-model limits**: vary by model.
- **Long-context limits**: separate rate limit for long-context requests (e.g., GPT-5.5).
- **Usage limits (monthly spend cap)**: max org spend per month.
- **Shared limits**: some model families share a single TPM pool.
- **Batch API queue limits**: calculated by total input tokens queued per model; completed jobs free up the limit.
- **Vector store ingestion limit**: 300 RPM per vector store ID.
- **Usage tiers**: auto-graduated by cumulative paid spend; higher tiers → higher limits.
- **Exponential backoff**: retry strategy with jitter to handle 429s; failed requests still count against per-minute limit.

### Usage tiers

| Tier | Qualification (paid) | Usage limit (monthly) |
| --- | --- | --- |
| Free | Allowed geography | $100/month |
| Tier 1 | $5 paid | $100/month |
| Tier 2 | $50 paid | $500/month |
| Tier 3 | $100 paid | $1,000/month |
| Tier 4 | $250 paid | $5,000/month |
| Tier 5 | $1,000 paid | $200,000/month |

### API functions/endpoints

| Function | Method & Endpoint | Purpose |
| --- | --- | --- |
| Retrieve fine-tuning rate limits | `GET /v1/fine_tuning/model_limits` | Get fine-tuning rate limits (Auth: `Bearer $OPENAI_API_KEY`) |

### Rate limit HTTP response headers

| Header field | Type | Description |
| --- | --- | --- |
| `x-ratelimit-limit-requests` | int | Max requests permitted (e.g., 60) |
| `x-ratelimit-limit-tokens` | int | Max tokens permitted (e.g., 150000) |
| `x-ratelimit-remaining-requests` | int | Remaining requests permitted |
| `x-ratelimit-remaining-tokens` | int | Remaining tokens permitted |
| `x-ratelimit-reset-requests` | duration | Time until request limit resets (e.g., `1s`) |
| `x-ratelimit-reset-tokens` | duration | Time until token limit resets (e.g., `6m0s`) |
| `x-ratelimit-limit-project-tokens` | int | Project-scoped token limit |
| `x-ratelimit-remaining-project-tokens` | int | Remaining project-scoped tokens |
| `x-ratelimit-reset-project-tokens` | duration | Time until project token limit resets |

### Key parameters affecting limits

- `max_tokens` (int): rate limit calculated as max(`max_tokens`, estimated tokens from character count). Set close to expected response size.
- `prompt` (string or list): batch multiple prompts as a list to increase TPM throughput.
- Batch input tokens queued (for `/v1/batch`) count against queue limit until completion.

---

## 9. Logs, Audit & Data Retention

**Source pages:** `guides/admin-apis` (audit logs), `guides/your-data` (retention), `guides/agents/integrations-observability` (tracing), `guides/secure-mcp-tunnels` (audit events)

### Main concepts

- **Audit logs**: list recent user actions and configuration changes for the org. Retrieved via Admin API.
- **Audit log events (tunnels)**: `tunnel.created`, `tunnel.updated`, `tunnel.deleted`.
- **Tracing**: built into the Agents SDK, enabled by default. Emits structured record of model calls, tool calls, handoffs, guardrails, and custom spans. Inspectable in the Traces dashboard.
- **Trace contents**: overall run/workflow, each model call, tool calls + outputs, handoffs + guardrails, custom spans.
- **Trace wrapping**: wrap multiple runs in one trace using `withTrace` (TS) / `trace` (Python).
- **Abuse Monitoring Logs**: retained up to 30 days by default; may contain customer content and derived metadata. MAM/ZDR controls modify this.
- **Application State**: data persisted by some API features to fulfill a task/request.
- **`expires_after` parameter**: files auto-deletion.

### API functions/endpoints

| Function | Method & Endpoint | Purpose |
| --- | --- | --- |
| List audit logs | `GET /v1/organization/audit_logs` | List recent user actions and config changes (Admin API) |

### Key parameters

**Audit Logs List**

| Parameter | Type | Description |
| --- | --- | --- |
| `limit` | int | max number of logs to return, e.g. `10` |

### SDK tracing functions

| Function/Class | Language | Purpose |
| --- | --- | --- |
| `withTrace(name, async fn)` | TypeScript | Wrap multiple runs in one trace |
| `trace(name)` (context manager) | Python | Wrap multiple runs in one trace |

---

## 10. Network Security (Private Link, IP Egress, mTLS, Tunnels)

**Source pages:** `guides/private-link`, `guides/ip-addresses`, `guides/secure-mcp-tunnels`

### 10.1 Private Link (Azure private-network connectivity)

#### Main concepts

- **Private Link**: lets Azure workloads reach regional OpenAI API endpoints through Azure Private Link instead of public endpoints. Not self-service — requires contacting OpenAI/sales.
- **Legacy Private Link (v1)**: cluster-specific host names (e.g. `privatelink.enterprise.unified-1.api.openai.com`), pinned to one API cluster, older v1 health check paths.
- **Regional Private Link**: regional host names (e.g. `southcentralus.privatelink.api.openai.com`), regional private-edge gateway routing to multiple backing clusters.
- **Private Endpoint**: Azure resource connecting a customer VNet to the OpenAI Private Link Service; must share the region of the customer VNet.
- **Private DNS**: maps regional host names to Private Endpoint IP addresses inside the network.
- **Regional private-edge gateway / rail**: routes requests to enterprise-enabled backing OpenAI API clusters; can route around unavailable clusters within a rail.
- **Fail over**: must be configured by the customer (not automatic); probe each region with the health check.
- Incompatible with IP allowlist controls and mutual TLS (mTLS); Azure-specific (AWS/GCP must route through Azure).

#### Regional host names

| Region label | Customer host name |
| --- | --- |
| South Central US | `southcentralus.privatelink.api.openai.com` |
| West US | `westus.privatelink.api.openai.com` |
| East US 2 | `eastus2.privatelink.api.openai.com` |
| Spain Central / EU | `spaincentral.privatelink.api.openai.com` |

#### API functions/endpoints

| Function | Method & Endpoint | Purpose |
| --- | --- | --- |
| Regional health check | `GET /v2/privatelink_healthcheck` | Returns HTTP `200` with `{"message": "Service is up"}` when healthy. Rate limit: ≤ 1 QPS per regional endpoint |
| Normal API surface via Private Link | `https://<region>.privatelink.api.openai.com/v1/*` | Standard `/v1` API accessed via regional Private Link host |

#### Endpoint compatibility matrix (per-region backing availability)

| Endpoint family | South Central US | West US | East US 2 | Spain Central/EU |
| --- | --- | --- | --- | --- |
| `/v1/responses` | Yes | Yes | Yes | Yes |
| `/v1/chat/completions` | Yes | Yes | Yes | Yes |
| `/v1/completions` | Yes | Yes | Yes | Yes |
| `/v1/embeddings` | Yes | Yes | Yes | Yes |
| `/v1/audio/*` (Inference) | Yes | Yes | Yes | Yes |
| `/v1/audio/*` (management) | Yes | No | No | Yes |
| `/v1/models` | Yes | Yes | Yes | Yes |
| `/v1/files`, `/v1/uploads` | Yes | Yes | Yes | Yes |
| `/v1/batches` | Yes | Yes | Yes | Yes |
| `/v1/images/*` | Yes | Yes | Yes | Yes |
| `/v1/moderations` | Yes | Yes | Yes | Yes |
| `/v1/vector_stores` | Yes | Yes | Yes | Yes |
| `/v1/organization/audit_logs` | Yes | Yes | Yes | Yes |
| Other `/v1/organization/*`, `/v1/usage` | Yes | No | No | Yes |
| `/v1/realtime` | Yes | Yes | Yes | Yes |

#### Key parameters

| Parameter | Type | Description |
| --- | --- | --- |
| `base_url` | string (URL) | Regional Private Link base URL, e.g. `https://southcentralus.privatelink.api.openai.com/v1` (SDK client) |
| `--private-connection-resource-id` | string | OpenAI-provided Private Link Service resource identifier (Azure CLI) |
| `--manual-request true` | flag | Required for alias connections; subscriptions on the access list may still get auto-approval |

### 10.2 IP Egress Ranges (Outbound IP Allowlists)

#### Main concepts

- **IP egress ranges**: published outbound IP ranges for OpenAI products that make requests to services you control; used to configure network allowlists.
- **IP allowlisting caveats**: identifies traffic from an OpenAI-operated network, NOT a specific user/workspace; does not replace request auth/authz.
- **mTLS (mutual TLS)**: use to authenticate ChatGPT as MCP client for ChatGPT apps.
- **OAuth 2.1**: use to authenticate/authorize the user when app requires user auth.
- **Dynamic ranges**: ranges change as OpenAI infrastructure changes; fetch JSON regularly and auto-update allowlist.

#### Published ranges (JSON manifest URLs)

| Product | Used for | Published ranges (JSON URL) |
| --- | --- | --- |
| ChatGPT integrations | Apps SDK, connectors, GPT Actions, Agentic Commerce | `https://openai.com/chatgpt-connectors.json` |
| Codex cloud | Connections from Codex cloud to services such as GitHub | `https://openai.com/chatgpt-agents.json` |

#### JSON manifest shape

| Field | Type | Description |
| --- | --- | --- |
| `creationTime` | string/timestamp | When the prefix list was generated |
| `prefixes` | array | List of CIDR IP ranges to allowlist |

### 10.3 Secure MCP Tunnels

#### Main concepts

- **MCP tunnel**: an outbound-only HTTPS connection from a host inside your network to an OpenAI-hosted MCP endpoint. Lets OpenAI products reach a private/on-prem/firewalled MCP server without public ingress.
- **`tunnel-client`**: a binary run inside the network that can reach the private MCP server; long-polls OpenAI for queued MCP work, forwards JSON-RPC requests locally, posts responses back through the same tunnel.
- **Control plane**: OpenAI-hosted tunnel endpoint that OpenAI products call; `tunnel-client` authenticates to it.
- **Trust boundary / network boundary**: the private MCP server stays inside the customer environment; only `tunnel-client` reaches OpenAI over outbound HTTPS.
- **mTLS (control-plane)**: optional mutual TLS for the control-plane connection (`mtls.api.openai.com:443`).
- **Harpoon**: embedded MCP server inside `tunnel-client` that exposes narrowly scoped, allowlisted HTTP callout targets by label (bounded request/response limits; not a general proxy).
- **Tunnel associations**: a tunnel can be associated with one or more Platform organizations or ChatGPT workspaces.
- **OAuth discovery through tunnel**: the tunnel can carry OAuth discovery so the MCP server stays private.

#### API functions/endpoints

| Function | Method & Endpoint | Purpose |
| --- | --- | --- |
| Tunnel control-plane (default) | `HTTPS GET/POST https://api.openai.com:443/v1/tunnel/*` | Long-poll for queued work and response posting |
| Tunnel control-plane (mTLS) | `HTTPS GET/POST https://mtls.api.openai.com:443/v1/tunnel/*` | Polling/response posting when control-plane mTLS configured |
| Local admin /healthz | `GET /healthz` | Health check (loopback-only) |
| Local admin /readyz | `GET /readyz` | Readiness check (loopback-only) |
| Local admin /metrics | `GET /metrics` | Metrics (loopback-only) |
| Local admin /ui | `GET /ui` | Local admin UI (loopback-only) |

#### `tunnel-client` CLI commands

| Command | Purpose |
| --- | --- |
| `tunnel-client help quickstart` | Quickstart help |
| `tunnel-client init` | Initialize a profile |
| `tunnel-client doctor --profile <name> --explain` | Validate a profile and explain issues |
| `tunnel-client run --profile <name>` | Start the long-polling client |

#### Key parameters

| Parameter | Type | Description |
| --- | --- | --- |
| `--sample <sample_name>` | string | sample profile to use, e.g. `sample_mcp_stdio_local` |
| `--profile <name>` | string | named local profile, e.g. `local-stdio` |
| `--tunnel-id <tunnel_id>` | string | tunnel identity from Platform tunnel settings, format `tunnel_0123...` (hex) |
| `--mcp-command "<command>"` | string | local stdio command to launch the MCP server. Mutually exclusive with `--mcp-server-url` |
| `--mcp-server-url <url>` | string (URL) | HTTP MCP server address. Mutually exclusive with `--mcp-command` |
| `CONTROL_PLANE_API_KEY` | string (env var) | runtime API key for `tunnel-client` to authenticate to the OpenAI control plane |

#### RBAC permissions (Tunnels)

| Permission | Description |
| --- | --- |
| Tunnels **Read** | View tunnels |
| Tunnels **Manage** | Create/edit/delete tunnels (requires Read) |
| Tunnels **Use** | Run `tunnel-client` or select a tunnel in connector settings (requires Read) |

#### Audit log events

`tunnel.created`, `tunnel.updated`, `tunnel.deleted`.

---

## 11. Orchestration & Integrations (Agents SDK, MCP, Webhooks)

**Source pages:** `guides/agents/orchestration`, `guides/agents/integrations-observability`, `guides/webhooks`, `guides/developer-mode`, `guides/tools-connectors-mcp`

### 11.1 Orchestration and Handoffs

#### Main concepts

- **Handoffs**: a specialist takes over the conversation for that branch (control moves to specialist agent). Best when a specialist should own the next response.
- **Agents as tools**: a manager stays in control and calls specialists as bounded capabilities (manager keeps ownership of the reply). Best when the manager should synthesize the final answer.
- **Design guidance**: start with one agent; add specialists only when they materially improve capability isolation, policy isolation, prompt clarity, or trace legibility.

#### SDK functions/classes

| Function/Class | Language | Purpose |
| --- | --- | --- |
| `Agent({...})` / `Agent.create({...})` | TypeScript | Create an agent |
| `Agent(name=..., handoffs=[...])` | Python | Create an agent |
| `handoff(agent)` | TS/Python | Wrap an agent as a handoff target |
| `agent.asTool({toolName, toolDescription})` | TypeScript | Expose an agent as a callable tool |
| `agent.as_tool(tool_name=..., tool_description=...)` | Python | Expose an agent as a callable tool |

#### Key parameters

| Parameter | Type | Description |
| --- | --- | --- |
| `name` | string | Agent name |
| `instructions` | string | Agent instructions |
| `handoffs` | array | List of agents (or `handoff(agent)` wrappers) that this agent can hand off to |
| `tools` | array | Tools, including agents-as-tools |
| `handoffDescription` | string | Short, concrete description used for routing |
| `toolName` / `tool_name` | string | Name exposed to the manager model |
| `toolDescription` / `tool_description` | string | Description for the manager model |

### 11.2 MCP Integration & Observability

#### Main concepts

- **Hosted MCP**: remote server runs through the model/platform surface. Use for public remote servers fitting the platform trust model.
- **Local MCP**: your runtime owns connection, approvals, network boundaries. Use when your runtime should own connectivity, filtering, or approvals.
- **Tracing**: built into the Agents SDK, enabled by default. Emits structured record of model calls, tool calls, handoffs, guardrails, and custom spans.

#### SDK functions/classes

| Function/Class | Language | Purpose |
| --- | --- | --- |
| `hostedMcpTool({...})` | TypeScript | Attach a hosted (remote) MCP server as a tool |
| `HostedMCPTool(tool_config={...})` | Python | Attach a hosted (remote) MCP server as a tool |
| `MCPServerStdio({...})` | TypeScript | Connect a local MCP server over stdio |
| `MCPServerStdio(params={...})` | Python | Connect a local MCP server over stdio |
| `withTrace(name, async fn)` | TypeScript | Wrap multiple runs in one trace |
| `trace(name)` (context manager) | Python | Wrap multiple runs in one trace |
| `server.connect()` / `server.close()` | TypeScript | Manage MCP server lifecycle |

#### Key parameters

**`hostedMcpTool` / `HostedMCPTool`**

| Parameter | Type | Description |
| --- | --- | --- |
| `serverLabel` / `server_label` | string | Label for the MCP server (e.g. `"gitmcp"`) |
| `serverUrl` / `server_url` | string | URL of the hosted MCP server |
| `require_approval` | string | Approval requirement (e.g. `"never"`) |
| `type` | string | Python tool_config: `"mcp"` |

**`MCPServerStdio`**

| Parameter | Type | Description |
| --- | --- | --- |
| `name` | string | Server display name |
| `fullCommand` (TS) / `params.command`+`params.args` (Python) | string / array | Command + args to launch the MCP server |

### 11.3 Webhooks

#### Main concepts

- **Webhooks**: HTTP POST notifications delivered to an endpoint you control when subscribed API events occur (e.g. batch completes, background response generated, fine-tuning job finishes).
- **Standard Webhooks specification**: OpenAI webhooks follow the [Standard Webhooks spec](https://github.com/standard-webhooks/standard-webhooks).
- **Per-project configuration**: webhooks are configured per project in the dashboard (Settings → Project → Webhooks).
- **Signing secret**: provided once at endpoint creation; used to verify inbound webhook signatures; rotatable if lost/exposed.
- **Event types**: subscribe to one or more event types per endpoint.
- **Idempotency / deduplication**: use the `webhook-id` header as an idempotency key (rare duplicate deliveries possible).
- **Retry policy**: endpoint must return `2xx` quickly; non-2xx / timeouts trigger retries with exponential backoff for up to 72 hours; `3xx` redirects treated as failures.
- **Signature verification**: via SDK `webhooks.unwrap()` helper; raises `InvalidWebhookSignatureError` on mismatch.

#### API functions/endpoints

| Function | Method & Endpoint | Purpose |
| --- | --- | --- |
| Create background response (triggers webhook) | `POST /v1/responses` with `background: true` | Run async and emit `response.completed` |
| Retrieve completed response | `GET /v1/responses/{response_id}` | Called from webhook handler using `event.data.id` |

#### Inbound webhook HTTP request

Headers:
- `user-agent: OpenAI/1.0 (+https://platform.openai.com/docs/webhooks)`
- `content-type: application/json`
- `webhook-id` (string) — unique event ID, use for idempotency/dedup
- `webhook-timestamp` (unix integer)
- `webhook-signature` (string) — prefixed `v1,` + base64 HMAC

Body (Event object):

| Field | Type | Description |
| --- | --- | --- |
| `object` | string | `"event"` |
| `id` | string | event ID (e.g. `evt_...`) |
| `type` | string | event type (e.g. `response.completed`) |
| `created_at` | unix integer | — |
| `data` | object | event-specific payload; for `response.completed`: `{"id": "resp_abc123"}` |

#### SDK helper functions

| Function | Language | Purpose |
| --- | --- | --- |
| `client.webhooks.unwrap(payload, headers, { secret })` | JS | Verifies signature and returns parsed Event object |
| `client.webhooks.unwrap(request.data, request.headers, secret=...)` | Python | Verifies signature and returns parsed Event object |

#### Webhook endpoint configuration (dashboard)

| Field | Type | Description |
| --- | --- | --- |
| Name | string | label for reference |
| URL | string | public HTTPS endpoint to receive POSTs |
| Event types | list | one or more event types to subscribe to |

### 11.4 ChatGPT Developer Mode

#### Main concepts

- **ChatGPT developer mode**: provides full MCP client support for all tools, both read and write; powerful but dangerous; intended for developers who understand safe app configuration/testing.
- **Eligibility**: Pro, Plus, Business, Enterprise, and Education accounts on the web.
- **Developer-mode apps**: created from remote MCP servers; appear under "Drafts" in app settings.
- **Supported MCP protocols**: SSE and streaming HTTP.
- **Authentication supported**: OAuth, No Authentication, and Mixed Authentication.
  - **OAuth**: static credentials OR Client ID Metadata Documents (CIMD) when the authorization server advertises support. CIMD supports `none` and `private_key_jwt` token exchange. DCR (Dynamic Client Registration) also supported.
  - **Mixed authentication**: OAuth + No Authentication — initialize/list-tools APIs use no auth; individual tools use OAuth or no auth.
- **Tool call review/confirmation**: write actions require confirmation by default; respects the `readOnlyHint` MCP tool annotation; tools without this hint are treated as write actions.

#### Key parameters (MCP tool annotations consumed by ChatGPT)

| Parameter | Type | Description |
| --- | --- | --- |
| `readOnlyHint` | boolean | marks a tool as read-only; absence → treated as a write action requiring confirmation |

---

## 12. Sandboxes & Code Execution Isolation

**Source pages:** `guides/agents/sandboxes`

### Main concepts

- **Sandbox**: an isolated, Unix-like execution environment with filesystem, shell, installed packages, mounted data, exposed ports, snapshots, and controlled external access. Available in TypeScript and Python Agents SDKs (beta).
- **Harness vs. compute boundary**: Harness = control plane (agent loop, model calls, tool routing, handoffs, approvals, tracing, recovery, run state). Compute = sandbox execution plane (files, commands, packages, mounts, ports, snapshots).
- **`SandboxAgent`**: still an `Agent` — keeps `instructions`, `prompt`, `tools`, `handoffs`, MCP servers, model settings, structured output, guardrails, hooks. Adds sandbox execution boundary.
- **Manifest**: fresh-session workspace contract — desired starting contents/layout. Paths are workspace-relative (no absolute paths or `..`).
- **Manifest entry types**: `File`, `Dir`, local file/directory, Git repo, storage mounts (`S3Mount`, `GCSMount`, `R2Mount`, `AzureBlobMount`, `BoxMount`, `S3FilesMount`), `environment`, `users`, `groups`.
- **Capabilities**: sandbox-native behavior attached to a `SandboxAgent`. Defaults: filesystem, shell, compaction. Passing a `capabilities` list replaces defaults.
  - `Shell` — command execution + interactive input
  - `Filesystem` — `apply_patch`, `view_image`; patch paths workspace-root-relative
  - `Skills` — skill discovery/materialization (lazy local dir, local dir, Git repo)
  - `Memory` — persist lessons across runs (requires `Shell`; live updates require `Filesystem`)
  - `Compaction` — context trimming for long-running flows
- **Sandbox client**: provider integration (Unix-local, Docker, hosted providers).
- **Sandbox session**: live execution environment.
- **Sandbox run config**: per-run sandbox session source, client options, fresh inputs.
- **Saved state surfaces**: `RunState` (harness-side state), Session state (serialized sandbox session for reconnection), `snapshot` (saved workspace contents to seed a fresh sandbox).
- **Session resolution order**: (1) live session → (2) `RunState`-stored session → (3) explicit serialized state → (4) fresh session (per-run manifest or agent's default manifest).
- **Sandbox memory**: distinct from SDK `Session` memory (message history). Memory = reusable guidance distilled from prior workspace runs into files (`memory_summary.md`, `MEMORY.md`, `raw_memories.md`, `rollout_summaries/`). Progressive disclosure.
- **Ports/previews**: agent starts a service in the sandbox; client exposes the port; app shares/inspects the preview URL.
- **Composition**: sandbox agents compose with handoffs and agents-as-tools.
- **Sandbox providers**: Blaxel, Cloudflare, Daytona, Docker, E2B, Modal, Runloop, Unix-local, Vercel.

### SDK functions/classes

| Function/Class | Language | Purpose |
| --- | --- | --- |
| `Manifest({entries})` / `Manifest(entries={...})` | TS/Python | Define fresh-session workspace contents |
| `SandboxAgent({...})` | TS/Python | Create a sandbox agent with manifest + capabilities |
| `run(agent, input, {sandbox})` | TypeScript | Run a sandbox agent |
| `Runner.run(agent, input, run_config=...)` | Python | Run a sandbox agent |
| `RunConfig(sandbox=SandboxRunConfig(...))` | Python | Per-run configuration |
| `SandboxRunConfig(client=..., options=..., session=...)` | Python | Per-run sandbox config |
| `client.create({manifest})` | TS | Create a sandbox session |
| `client.serializeSessionState(session.state)` / `client.deserializeSessionState(...)` | TS/Python | Serialize/deserialize session state for resume |
| `client.resume(frozenState)` | TS/Python | Resume from serialized session state |
| `client.delete(session)` | TS/Python | Delete a sandbox session |
| `session.close()` | TS/Python | Close a sandbox session |
| `Capabilities.default()` | TS/Python | Default capability set (filesystem, shell, compaction) |
| `Shell()` / `shell()` | TS/Python | Shell capability |
| `Filesystem()` / `filesystem()` | TS/Python | Filesystem capability |
| `Memory()` / `memory()` | TS/Python | Memory capability |
| `Skills(from_=...)` / `skills({from: ...})` | TS/Python | Skills capability |
| `File({content})` / `file({content})` | TS/Python | Manifest entry for a file |
| `GitRepo({repo, ref})` / `gitRepo({repo, ref})` | TS/Python | Manifest entry / skill source for a Git repo |
| `UnixLocalSandboxClient({...})` | TS/Python | Unix-local sandbox client |
| `DockerSandboxClient({...})` | TS/Python | Docker sandbox client |
| Provider clients (Cloudflare, E2B, Modal, Daytona, Vercel, Blaxel, Runloop) | TS/Python | Hosted provider sandbox clients |

### Key parameters

**`SandboxAgent` constructor**

| Parameter | Type | Description |
| --- | --- | --- |
| `name` | string | Agent name |
| `model` | string | e.g. `"gpt-5.5"` |
| `instructions` | string | Agent instructions |
| `defaultManifest` (TS) / `default_manifest` (Python) | Manifest | Default workspace contract for fresh sessions |
| `capabilities` | array | Capability list (replaces defaults: filesystem, shell, compaction) |

**`SandboxRunConfig`**

| Parameter | Type | Description |
| --- | --- | --- |
| `client` | SandboxClient | Provider sandbox client instance |
| `options` | object | Provider-specific options (e.g. `DockerSandboxClientOptions(image=...)`) |
| `session` | SandboxSession | Live/injected session to reuse |

**`DockerSandboxClient` / `DockerSandboxClientOptions`**

| Parameter | Type | Description |
| --- | --- | --- |
| `image` | string | Docker image (e.g. `"node:22-bookworm-slim"`, `DEFAULT_PYTHON_SANDBOX_IMAGE`) |

### Sandbox providers

| Provider | SDK Client Class |
| --- | --- |
| Blaxel | `BlaxelSandboxClient` |
| Cloudflare | `CloudflareSandboxClient` |
| Daytona | `DaytonaSandboxClient` |
| Docker | `DockerSandboxClient` |
| E2B | `E2BSandboxClient` |
| Modal | `ModalSandboxClient` |
| Runloop | `RunloopSandboxClient` |
| Unix-local | `UnixLocalSandboxClient` |
| Vercel | `VercelSandboxClient` |

### Memory layout (in sandbox workspace)

```
workspace/
  sessions/
    <id>.jsonl
  memories/
    memory_summary.md
    MEMORY.md
    raw_memories.md
    phase_two_selection.json
    raw_memories/
      <id>.md
    rollout_summaries/
      <id>.md
  skills/
```

---

## 13. Guardrails, Approvals & Safety Controls

**Source pages:** `guides/agents/guardrails-approvals`, `guides/agent-builder-safety`, `guides/safety-best-practices`

### 13.1 Guardrails and Human-in-the-Loop Approvals

#### Main concepts

- **Guardrails**: automatic validation of input, output, or tool behavior. Tripwire-triggered result decides whether a run continues/stops.
  - **Input guardrails**: run before the main agent starts; validate user request.
  - **Output guardrails**: validate/redact final output before it leaves the system; run only on the agent producing the final output.
  - **Tool guardrails**: check arguments or results around a function tool call; attached to the specific tool.
- **`runInParallel`**: guardrail option; `false` = blocking (sequential, stops main agent if tripped), `true` = parallel (lower latency, speculative work).
- **Tripwire**: boolean (`tripwireTriggered` / `tripwire_triggered`) that signals a guardrail blocked the request.
- **Human-in-the-loop approvals**: pause a run before a side-effecting tool call; person/policy approves or rejects.
- **Interruptions**: when a tool needs review, the run records an approval interruption instead of executing; result returns `interruptions` + resumable `state`.
- **State (resumable)**: serialized run state; resume the same run from `state` after approving/rejecting (no new user turn). Can be stored for delayed review.
- **`needsApproval` / `needs_approval`**: tool option marking a function tool as requiring approval before execution.
- **Workflow boundaries**: input guardrails run only for the first agent in a chain; output guardrails only for the final-output agent; tool guardrails only on their attached tool.

#### SDK functions/classes

**TypeScript (`@openai/agents`)**

| Function/Class | Purpose |
| --- | --- |
| `new Agent({ name, instructions, outputType, inputGuardrails, tools })` | Create an agent |
| `run(agent, input, { context })` | Run an agent; returns result with `finalOutput`, `interruptions`, `state` |
| `tool({ name, description, parameters, needsApproval, execute })` | Define a function tool requiring approval |
| `InputGuardrailTripwireTriggered` | Exception thrown when an input guardrail trips |
| `state.approve(interruption)` | Approve a pending interruption |

**Python (`agents`)**

| Function/Class | Purpose |
| --- | --- |
| `Agent(name=, instructions=, output_type=, input_guardrails=, tools=)` | Create an agent |
| `Runner.run(agent, input, context=)` | Run an agent; returns result with `final_output`, `interruptions`, `to_state()` |
| `@input_guardrail` | Decorator to define an input guardrail function |
| `@function_tool(needs_approval=True)` | Decorator to define a tool requiring approval |
| `GuardrailFunctionOutput(output_info=, tripwire_triggered=)` | Guardrail return type |
| `InputGuardrailTripwireTriggered` | Exception for tripped input guardrail |
| `state.approve(interruption)` | Approve pending interruption |
| `result.to_state()` | Get resumable state from result |

#### Key parameters

**Guardrail object (TS)**

| Parameter | Type | Description |
| --- | --- | --- |
| `name` | string | guardrail name |
| `runInParallel` | boolean | `false` = blocking, `true` = parallel |
| `execute({ input, context })` | async function | returns `{ outputInfo, tripwireTriggered }` |
| `outputInfo` | any | guardrail result detail |
| `tripwireTriggered` | boolean | whether to block the run |

**`tool` (TS) / `@function_tool` (Python)**

| Parameter | Type | Description |
| --- | --- | --- |
| `name` / `name` | string | tool name |
| `description` | string | tool description |
| `parameters` | Zod schema | e.g. `z.object({ orderId: z.number() })` |
| `needsApproval` / `needs_approval` | boolean | require human approval before execution |
| `execute({ ...params })` | async function | tool implementation |

**Run result (both languages)**

| Field | Type | Description |
| --- | --- | --- |
| `interruptions` | array | pending approval items |
| `state` / `to_state()` | object | resumable state with `.approve(interruption)` |
| `finalOutput` / `final_output` | string/object | final agent output |

### 13.2 Agent Builder Safety

#### Main concepts

- **Prompt injections**: untrusted text/data entering an AI system that attempts to override instructions; goals include data exfiltration via tool calls, misaligned actions, behavior changes.
- **Private data leakage**: agent accidentally sharing private data (e.g., sending more data to an MCP than intended).
- **Agent Builder (legacy/deprecated)**: shutting down November 30, 2026; ChatKit remains available.
- **Developer messages precedence**: developer messages take precedence over user/assistant messages; never inject untrusted input into developer messages.
- **Structured outputs for data flow**: define structured outputs (enums, fixed schemas, required fields) between nodes to eliminate freeform channels attackers exploit.
- **Tool approvals**: always enable tool approvals for MCP tools so users review every operation (reads/writes).
- **Guardrails for user inputs**: sanitize inputs (PII redaction, jailbreak detection) via Agent Builder guardrails nodes.
- **Trace graders / evals**: score/annotate agent trace parts (decisions, tool calls, reasoning) to catch/prevent mistakes.
- **Model choice**: GPT-5 / GPT-5-mini recommended for stronger jailbreak/injection robustness.

*This page is purely conceptual guidance — no API endpoints or parameters are documented.*

### 13.3 Safety Best Practices

#### Main concepts

- **Moderation API**: free-to-use API to reduce unsafe content in completions.
- **Moderation scores in generation request**: request moderation scores inline with Responses API / Chat Completions.
- **Adversarial testing ("red-teaming")**: test robustness against adversarial/prompt-injection inputs.
- **Human in the loop (HITL)**: human review of outputs before use, especially in high-stakes domains.
- **Know your customer (KYC)**: registration/login requirements; linking to existing accounts; requiring credit card/ID.
- **Constrain user input / limit output tokens**: limit input text length and output token count to reduce misuse.
- **Validated inputs/outputs**: dropdowns over open-ended text; backend validated material over novel generated content.
- **Safety identifiers**: privacy-preserving hashed user/session identifiers sent with requests.

#### Key parameters

| Parameter | Type | Where | Description |
| --- | --- | --- | --- |
| `safety_identifier` | string | Chat Completions / Responses API request body | Hashed username/email or session ID uniquely identifying each user |
| `OpenAI-Safety-Identifier` | header string | Realtime API connection / ephemeral secret creation request | Same stable, privacy-preserving identifier; bound to the session |
| `max_completion_tokens` | integer | Chat Completions request body | Limit output tokens to reduce misuse |

---

## 14. Performance Optimization

**Source pages:** `guides/latency-optimization`, `guides/prompt-caching`, `guides/predicted-outputs`, `guides/compaction`, `guides/websocket-mode`, `guides/background`, `guides/streaming-responses`, `guides/model-optimization`, `guides/deployment-checklist`

### 14.1 Latency Optimization

#### Main concepts (seven principles)

1. **Process tokens faster** — inference speed (TPM/TPS); primarily driven by model size. Techniques: longer/detailed prompts, few-shot examples, fine-tuning/distillation, Predicted outputs, faster hardware, lower engine saturation.
2. **Generate fewer tokens** — token generation is highest-latency step; ~50% output token reduction ≈ ~50% latency reduction. Techniques: ask for brevity, minimize structured-output syntax, use `max_tokens`/`stop_tokens`.
3. **Use fewer input tokens** — minor factor (50% prompt cut ≈ 1–5% latency). Techniques: fine-tuning to replace instructions/examples, filtering context (RAG pruning, HTML cleaning), maximizing shared prompt prefix (put dynamic content later for KV-cache friendliness).
4. **Make fewer requests** — avoid per-step round-trip latency; combine sequential steps into a single prompt/response.
5. **Parallelize** — split non-sequential steps into parallel calls; use speculative execution for sequential steps.
6. **Make your users wait less** — streaming (most effective), chunking, show steps, loading states.
7. **Don't default to an LLM** — hard-coding constrained outputs, pre-computing constrained inputs, traditional optimization.

#### Key parameters

| Parameter | Type | Description |
| --- | --- | --- |
| `max_tokens` | integer | End generation early to reduce output tokens |
| `stop_tokens` | array of string | Stop sequences to end generation early |

### 14.2 Predicted Outputs

#### Main concepts

- **Predicted Outputs**: provide anticipated output text to speed up generation when most output is already known (e.g., code/text refactoring with minor changes).
- **`prediction` request parameter**: object on Chat Completions requests carrying the predicted content.
- **`accepted_prediction_tokens`**: usage field counting prediction tokens the model actually used (latency saved).
- **`rejected_prediction_tokens`**: usage field counting prediction tokens NOT used; still billed as completion tokens.
- **Position independence**: predicted text can appear anywhere within the generated response.
- **Streaming compatibility**: works with `stream: true` for even greater latency gains.

#### API functions/endpoints

| Function | Method & Endpoint | Purpose |
| --- | --- | --- |
| Chat Completions with predicted output | `POST /v1/chat/completions` | Create a chat completion with a predicted output |

#### Key parameters

| Parameter | Type | Description |
| --- | --- | --- |
| `model` | string | Must be `gpt-4o`, `gpt-4o-mini`, `gpt-4.1`, `gpt-4.1-mini`, or `gpt-4.1-nano` series |
| `prediction` | object | `{ type: "content", content: "<predicted_text>" }` |
| `prediction.type` | string | Must be `"content"` |
| `prediction.content` | string | The predicted output text |
| `store` | boolean | Whether to store the completion |
| `stream` | boolean | Enable streaming for additional latency gains |

#### Limitations when using Predicted Outputs

- `n` > 1 not supported
- `logprobs` not supported
- `presence_penalty` > 0 not supported
- `frequency_penalty` > 0 not supported
- `audio` not compatible
- `modalities` — only `text` supported
- `max_completion_tokens` not supported
- `tools` (function calling) not supported

### 14.3 Compaction (Context Management)

#### Main concepts

- **Compaction**: reduce context size while preserving state needed for subsequent turns; balances quality, cost, latency as conversations grow.
- **Server-side compaction**: enabled in a Responses create request via `context_management` with `compact_threshold`. When rendered token count crosses threshold, server runs compaction automatically and emits a compaction output item in the same stream. ZDR-friendly with `store=false`.
- **Compaction item**: encrypted, opaque (not human-interpretable) item carrying forward key prior state and reasoning using fewer tokens.
- **Stateless input-array chaining**: append output items (including compaction items) to the next input array. Can drop items before the most recent compaction item to reduce latency.
- **`previous_response_id` chaining**: pass only the new user message each turn; do not manually prune.
- **Standalone compaction endpoint (`/responses/compact`)**: fully stateless, ZDR-friendly. Send a full context window; returns a new compacted context window (including compaction item + retained items). Do not prune the returned output — pass as-is into the next `/responses` call.

#### API functions/endpoints

| Function | Method & Endpoint | Purpose |
| --- | --- | --- |
| Responses create with compaction | `POST /v1/responses` with `context_management` | Enable server-side compaction |
| Standalone compact | `POST /v1/responses/compact` | Stateless compaction; returns compacted input window |

#### Key parameters

**Responses create**

| Parameter | Type | Description |
| --- | --- | --- |
| `model` | string | e.g. `"gpt-5.6"` |
| `input` | array | Conversation input items |
| `store` | boolean | `false` for ZDR-friendly flow |
| `context_management` | array | List of context management configs |

**`context_management` entry**

| Parameter | Type | Description |
| --- | --- | --- |
| `type` | string | `"compaction"` |
| `compact_threshold` | integer | Token count threshold that triggers server-side compaction (e.g. `200000`) |

**Standalone compact**

| Parameter | Type | Description |
| --- | --- | --- |
| `model` | string | e.g. `"gpt-5.6"` |
| `input` | array | Full context window (messages, tools, items) to compact |

**Response of `/responses/compact`**

| Field | Type | Description |
| --- | --- | --- |
| `output` | array | Compacted context window — includes compaction item + retained items; pass as-is into next `/responses` `input` |

### 14.4 WebSocket Mode

#### Main concepts

- **WebSocket mode**: persistent connection to `/v1/responses` for long-running, tool-call-heavy workflows (~40% faster for 20+ tool calls). Send only incremental input items per turn plus `previous_response_id`.
- **ZDR / `store=false` compatibility**: works with both since previous-response state is kept only in-memory (not persisted to disk).
- **`response.create` event**: the client event sent each turn. Payload mirrors the normal Responses create body, except transport-specific fields (`stream`, `background`) are not used.
- **Warmup / `generate: false`**: optionally pre-prepare request state (tools, instructions) without generating model output; returns a response ID for chaining.
- **Connection-local in-memory cache**: the service retains the most recent response in memory for fast continuation.
- **Connection limits**: 60-minute max duration; one in-flight response at a time (sequential, no multiplexing); use multiple connections for parallel runs.
- **Reconnect patterns**: (1) continue with `previous_response_id` if persisted; (2) start fresh with `previous_response_id: null` and full input context; (3) use a compacted window from `/responses/compact` as base input.

#### API functions/endpoints

| Function | Method & Endpoint | Purpose |
| --- | --- | --- |
| WebSocket Responses connection | `WS wss://api.openai.com/v1/responses` | Persistent connection; auth via `Authorization: Bearer <key>` header |
| `response.create` (client→server event) | WS event | Sent over the WebSocket; payload mirrors Responses create body |

#### Key parameters (`response.create` event payload)

| Parameter | Type | Description |
| --- | --- | --- |
| `type` | string | Event type; must be `"response.create"` |
| `model` | string | Model ID, e.g. `"gpt-5.6"` |
| `store` | boolean | Whether to persist responses (`false` for ZDR-compatible) |
| `previous_response_id` | string\|null | Prior response ID for continuation; omit/`null` to start a new chain |
| `input` | array | Input items — only NEW items when continuing |
| `tools` | array | Tool definitions for the turn |
| `generate` | boolean (optional) | `false` warms up request state without producing model output |

### 14.5 Background Mode

#### Main concepts

- **Background mode**: kick off long-running model tasks asynchronously (for models like GPT-5.2 / GPT-5.2 Pro that take minutes).
- **Polling**: poll the GET Responses endpoint while status is `queued` or `in_progress`; terminal state reached when it leaves those states.
- **Cancelling**: idempotent cancel of an in-flight response; subsequent calls return the final `Response` object.
- **Streaming + background**: create with both `background=true` and `stream=true`; track a "cursor" = `sequence_number` per streaming event to resume after disconnect.
- **ZDR**: background mode stores response data ~10 minutes for polling, so it is **not** ZDR-compatible; MAM projects can safely use it.
- **Resuming a stream**: reconnect with `starting_after=<sequence_number>` query param.

#### API functions/endpoints

| Function | Method & Endpoint | Purpose |
| --- | --- | --- |
| Generate background response | `POST /v1/responses` with `background: true` | Run async |
| Retrieve/poll background response | `GET /v1/responses/{response_id}` | Poll status |
| Cancel in-flight response | `POST /v1/responses/{response_id}/cancel` | Idempotent cancel |
| Stream a background response | `POST /v1/responses` with `background: true, stream: true` | Start streaming immediately |
| Resume a background stream | `GET /v1/responses/{response_id}?stream=true&starting_after={sequence_number}` | Resume after disconnect |

#### Key parameters

| Parameter | Type | Description |
| --- | --- | --- |
| `model` | string (required) | e.g. `"gpt-5.6"` |
| `input` | string (required) | the prompt |
| `background` | boolean (required) | `true` to run async |
| `stream` | boolean (optional) | `true` to also start streaming immediately |
| `starting_after` | integer (query) | last `sequence_number` cursor for stream resume |

#### Limits

1. Background sampling requires `store=true`; stateless requests are rejected.
2. To cancel a synchronous response, terminate the connection.
3. You can only start a new stream from a background response if you created it with `stream=true`.

### 14.6 Streaming Responses

#### Main concepts

- **HTTP streaming (`stream=true`)**: server-sent events (SSE) transport.
- **WebSocket mode**: separate persistent transport with incremental inputs via `previous_response_id`.
- **Semantic events**: the Responses API emits typed, predefined-schema events (type-safe).
- **Event lifecycle**: some events emitted once (lifecycle), others multiple times (deltas) as the response is generated.
- **`delta` field** (Chat Completions streaming): holds a role token, content token, or nothing; vs. non-streaming `message` field.
- **`type` property**: identifies individual streaming events.
- **Moderation risk**: partial completions are harder to moderate in production.

#### API functions/endpoints

| Function | Method & Endpoint | Purpose |
| --- | --- | --- |
| Create streaming response (Responses) | `POST /v1/responses` with `stream: true` | SSE streaming |
| Create streaming chat completion | `POST /v1/chat/completions` with `stream: true` | SSE streaming |

#### Key parameters

| Parameter | Type | Description |
| --- | --- | --- |
| `stream` | boolean (required for streaming) | `true` enables SSE streaming |
| `model` | string | e.g. `"gpt-5.6"` |
| `input` (Responses) / `messages` (Chat Completions) | array | conversation input |
| `content` | string | user message text |

#### Responses API streaming event types

Lifecycle (once): `ResponseCreatedEvent`, `ResponseInProgressEvent`, `ResponseFailedEvent`, `ResponseCompletedEvent`, `Error`
Output items: `ResponseOutputItemAdded`, `ResponseOutputItemDone`
Content parts: `ResponseContentPartAdded`, `ResponseContentPartDone`
Text: `ResponseOutputTextDelta`, `ResponseOutputTextAnnotationAdded`, `ResponseTextDone`
Refusal: `ResponseRefusalDelta`, `ResponseRefusalDone`
Function calls: `ResponseFunctionCallArgumentsDelta`, `ResponseFunctionCallArgumentsDone`
File search: `ResponseFileSearchCallInProgress`, `ResponseFileSearchCallSearching`, `ResponseFileSearchCallCompleted`
Code interpreter: `ResponseCodeInterpreterInProgress`, `ResponseCodeInterpreterCallCodeDelta`, `ResponseCodeInterpreterCallCodeDone`, `ResponseCodeInterpreterCallInterpreting`, `ResponseCodeInterpreterCallCompleted`

Common events to listen for (text): `response.created`, `response.output_text.delta`, `response.completed`, `error`.

### 14.7 Token Counting

#### Main concepts

- **Input token count**: exact number of input tokens a request will use, computed before sending to the model.
- **Counting API vs. local tokenizers** (e.g., `tiktoken`): local tokenizers can't handle images/files/tools/model-specific behavior; the API returns the model's exact processed count.
- **Formatting tokens**: tokens for message roles/boundaries that don't appear in visible text.
- **Output token counts**: `output_tokens` (Responses API) vs `completion_tokens` (Chat Completions); includes non-visible formatting/delimiter/channel/tool-call tokens.
- **Non-visible tokens**: formatting tokens not in message content or `logprobs`, may exceed `reasoning_tokens=0` baseline.
- **`max_output_tokens` / `max_completion_tokens`**: limit ALL tokens generated including non-visible ones.
- **Image token consumption**: based on size and detail level; counted exactly by the API.
- **Tool/schema tokens**: function definitions, MCP servers add tokens; counted together with input.

#### API functions/endpoints

| Function | Method & Endpoint | Purpose |
| --- | --- | --- |
| Count input tokens | `POST /v1/responses/input_tokens` | Exact input token count before sending to the model |

SDK methods:
- Python: `client.responses.input_tokens.count(...)`
- TypeScript: `client.responses.input_tokens.count({...})`
- CLI: `openai responses:input-tokens count ...`

#### Key parameters

| Parameter | Type | Description |
| --- | --- | --- |
| `model` | string (required) | e.g. `"gpt-5.5"`; the model whose tokenizer/behavior applies |
| `input` | string \| array (required) | same format as Responses API `input`; supports multimodal content parts |
| `instructions` | string (optional) | system instructions prepended to input |
| `tools` | array (optional) | tool definitions whose schema tokens should be counted |

#### Response shape

| Field | Type | Description |
| --- | --- | --- |
| `input_tokens` | integer | the exact input token count |
| `object` | string | `"response.input_tokens"` |

### 14.8 Model Optimization

#### Main concepts

- **LLM non-determinism**: output changes between snapshots/families; requires constant measurement and tuning.
- **Optimization flywheel**: Evals → prompt engineering → fine-tuning → measure → tweak → repeat.
- **Evals**: systematically measure model output against test inputs; build via API or dashboard; uses **graders** to score results.
- **Prompt engineering best practices**: include relevant context; provide clear instructions; provide example outputs (few-shot learning).
- **Fine-tuning**: take a base model, provide expected inputs/outputs to get a task-specialized model. Benefits: more examples than fit in a context window, shorter prompts (lower cost/latency), train on proprietary data, train smaller/cheaper models for a task.
- **Fine-tuning methods**:
  - **Supervised fine-tuning (SFT)**: provide correct response examples (human "ground truth").
  - **Vision fine-tuning**: image inputs for SFT.
  - **Direct preference optimization (DPO)**: provide correct + incorrect example responses; indicate the correct one.
  - **Reinforcement fine-tuning (RFT)**: generate response, provide expert grade, reinforce chain-of-thought for higher-scored responses. Requires expert graders; **reasoning models only**.
- **Deprecation note**: fine-tuning platform winding down; not accessible to new users; fine-tuned models available for inference until base models deprecated.

*This page is a conceptual guide — no REST endpoints are directly defined. References fine-tuning API at `/api/docs/api-reference/fine-tuning` and evals/graders guides.*

### 14.9 Deployment Checklist

#### Main concepts

- **Responses API**: OpenAI's flagship API; recommended starting point for all new deployments.
- **`reasoning.effort`**: controls how much model thinking precedes the answer (`"none"` | `"low"` | `"medium"` | `"high"` | `"xhigh"`).
- **`text.verbosity`**: controls brevity vs. completeness of output (`"low"` | `"medium"` | `"high"`).
- **Assistant `phase` parameter**: labels assistant messages as `"commentary"` (intermediate) or `"final_answer"` (completed).
- **`tool_search`**: deferred tool loading mechanism; model loads only the tools it needs at runtime.
  - Hosted tool search — simpler; OpenAI manages discovery.
  - Client-executed tool search — app controls what tools are available per context.
  - `defer_loading: true` — flag on tool definitions to mark them as deferred.
- **Compaction**: context engineering to reduce context size while preserving state (server-side via `context_management` + `compact_threshold`, or manual via `client.responses.compact()`).
- **`prompt_cache_key`**: routes requests sharing the same prompt prefix to the same cache.
- **`reasoning.encrypted_content`**: stateless handoff of reasoning items for ZDR compliance; add to `include` array.
- **`background=True`**: async job mode for long-running requests; requires `store=True`.
- **WebSocket mode**: persistent connection for long-running tool-call-heavy workflows (~40% faster for 20+ tool calls).

#### Key parameters (Responses API create)

| Parameter | Type | Description |
| --- | --- | --- |
| `model` | string | e.g. `"gpt-5.6"` |
| `input` | string/array | User input or full input items array |
| `instructions` | string | Developer/system instructions |
| `reasoning` | object | `{ effort: "none" \| "low" \| "medium" \| "high" \| "xhigh" }` |
| `text` | object | `{ verbosity: "low" \| "medium" \| "high" }` |
| `tools` | array | Tool definitions; include `{"type": "tool_search"}` for deferred loading |
| `defer_loading` | boolean | Mark a function tool as deferred (loaded on-demand by model) |
| `prompt_cache_key` | string | Stable key to route related requests to same cache; keep < 15 req/min per key-prefix pair |
| `include` | array | e.g. `["reasoning.encrypted_content"]` to get encrypted reasoning in output |
| `background` | boolean | `true` for async job mode; requires `store: true` |
| `store` | boolean | Whether to persist response; required `true` for `background` |
| `previous_response_id` | string | Continue from prior response (server-side compaction) |
| `context_management` | object | Server-side context management config with `compact_threshold` |
| `stream` | boolean | Streaming for background progress events |

#### Compaction function (`client.responses.compact()`)

| Parameter | Type | Description |
| --- | --- | --- |
| `model` | string | Model ID |
| `input` | array | Full conversation window (user messages, assistant outputs, tool calls, tool outputs) |

Returns object with `.output` array — pass forward as-is into next `responses.create()` input; do not edit.

#### Background response job statuses

`queued`, `in_progress` (poll via `retrieve`), then completed/failed/canceled.

---

## 15. Versioning & Deprecations

**Source pages:** `deprecations`, `changelog`

### 15.1 Deprecations

#### Main concepts

- **Deprecation**: the process of retiring a model or endpoint; immediately "deprecated" upon announcement.
- **Shutdown / sunset**: used interchangeably; the date a model/endpoint becomes inaccessible.
- **Legacy**: models/endpoints no longer receiving updates; expected to be deprecated eventually.
- **Model deprecation notice periods** (minimum, unless safety/compliance requires faster):
  - Generally available models: at least 6 months
  - Specialized variants of GA models (chat variants like `gpt-5.1-chat-latest`, Codex variants, deep research variants): at least 3 months
  - Preview models (model name contains `preview`): may be retired with much shorter notice, e.g. 2 weeks — not recommended for production
- **Notification**: email + documentation; larger changes via blog posts.
- **Dedicated capacity**: after shutdown, may be possible to provision dedicated capacity (contact sales).
- **Upcoming vs Past deprecations**: page is organized chronologically; each entry has a shutdown date, model/system, and recommended replacement.

#### Deprecated endpoints (selected)

| Endpoint | Shutdown date | Replacement |
| --- | --- | --- |
| `POST /v1/prompts` (Reusable prompts API) | Nov 30, 2026 | Migrate prompt content into application code |
| `POST /v1/fine-tunes` (legacy fine-tunes) | Jan 4, 2024 | `/v1/fine_tuning/jobs` |
| `/v1/edits` (Edit models endpoint) | Jan 4, 2024 | `/v1/chat/completions` |
| `/v1/engines` | Dec 3, 2022 | `/v1/models` |
| `/v1/search`, `/v1/classifications`, `/v1/answers` | Dec 3, 2022 | Transition guides |
| Assistants API (v1) | Dec 18, 2024 | v2 |
| Assistants API (whole) | Aug 26, 2026 | Responses API / Conversations API |
| Realtime API Beta (`OpenAI-Beta: realtime=v1`) | May 12, 2026 | Realtime API GA |
| Videos API + Sora 2 models | Sep 24, 2026 | (no replacement listed) |
| Evals platform (dashboard + API) | Nov 30, 2026 | Promptfoo |
| Agent Builder | Nov 30, 2026 | Agents SDK / ChatGPT Workspace Agents |
| Self-serve fine-tuning (new job creation) | Jan 6, 2027 | (inference continues until base model deprecated) |

#### Deprecated → replacement model mappings (selected)

| Deprecated model | Recommended replacement |
| --- | --- |
| `gpt-5-2025-08-07` | `gpt-5.5` |
| `gpt-5-mini-2025-08-07` | `gpt-5.4-mini` |
| `gpt-5-nano-2025-08-07` | `gpt-5.4-nano` |
| `gpt-5-pro-2025-10-06` | `gpt-5.5-pro` |
| `o3-2025-04-16` | `gpt-5.5` |
| `o3-pro-2025-06-10` | `gpt-5.5-pro` |
| `gpt-image-1-mini`, `gpt-image-1.5`, `chatgpt-image-latest`, `gpt-image-1` | `gpt-image-2` |
| `dall-e-2`, `dall-e-3` | `gpt-image-2` / `gpt-image-1` / `gpt-image-1-mini` |
| `gpt-4o-realtime-preview` (and snapshots) | `gpt-realtime-1.5` |
| `gpt-4o-mini-realtime-preview` | `gpt-realtime-mini` |
| `gpt-4o-audio-preview` | `gpt-audio-1.5` |
| `text-moderation-007` / `text-moderation-stable` / `text-moderation-latest` | `omni-moderation` |
| `o1-preview` | `o3` |
| `o1-mini` | `o4-mini` |
| `gpt-4.5-preview` | `gpt-4.1` |
| Legacy first-gen embeddings (`text-similarity-*`, `text-search-*`, `code-search-*`) | `text-embedding-3-small` |

### 15.2 Changelog

#### Main concepts

- **Monthly changelog**: dated entries (date, type tag, affected models, affected endpoints, description).
- **Entry types**: `Feature`, `Update`, `Fix` (and deprecations linked separately).
- **Affected-endpoint tags**: e.g. `v1/responses`, `v1/chat/completions`, `v1/batch`, `v1/realtime`, `v1/videos`, `v1/images/generations`, `v1/images/edits`.
- **Model snapshots**: `chat-latest` / `gpt-5.3-chat-latest` rolling pointers to the ChatGPT Instant snapshot.
- **Span**: entries from October 2023 → June 2026.

#### Key parameters surfaced in changelog entries

| Parameter | Type | Description |
| --- | --- | --- |
| `moderation` | object (on generation requests) | Returns moderation scores for input + generated output (Jun 4 2026) |
| `return_token_budget` | boolean (Responses API web search tool) | Opts in to longer GPT-5+ reasoning web search runs (May 11 2026) |
| `phase` | string (Responses API assistant message) | Labels message as `commentary` (intermediate) or `final_answer` (Feb 2026) |
| `prompt_cache_retention` | enum | Now defaults to `24h` instead of `in_memory` for non-ZDR orgs (May 29 2026) |
| `stream_options: {"include_usage": true}` | object (Chat Completions/Completions) | Enables usage stats during streaming (May 2024) |
| `safety_identifier` | string | Sent on requests to identify end users; surfaces in the Safety Usage Dashboard (Jun 23 2026) |

---

## 16. API Conventions & Streaming

**Source pages:** `guides/streaming-responses`, `guides/production-best-practices`, `changelog`

### Main concepts

- **Organization setup**: organization name (label) and organization ID (unique identifier for API requests).
- **API keys**: authentication mechanism; must be stored securely.
- **API key tracking**: keys generated before Dec 20, 2023 lack tracking; untracked usage shows as `Untracked`.
- **`n` and `best_of`**: number of completions returned / generated for consideration; total generated tokens = `max_tokens * max(n, best_of)`.
- **Cost management**: costs as function of (number of tokens) × (cost per token); reduce by switching models, shorter prompts, fine-tuning, caching queries.
- **MLOps**: data/model management, model monitoring, model retraining, model deployment.
- **Security and compliance**: data storage/transmission/retention, encryption, anonymization, input sanitization.

### Key parameters (general Chat Completions)

| Parameter | Type | Description |
| --- | --- | --- |
| `max_tokens` / `max_completion_tokens` | integer | Upper limit on generated tokens; lower values reduce latency |
| `n` | integer | Number of completions to generate per prompt (default 1) |
| `best_of` | integer | Number of completions generated for consideration (returned = highest log-prob) |
| `stream` | boolean | `true` to stream tokens as generated |
| `stop` | array/string | Stop sequences to prevent generating unneeded tokens |
| `stream_options: {"include_usage": true}` | object | Enables usage stats during streaming |

### Chat Completions streaming chunk shape

`{ role: 'assistant', content: '', refusal: null }` then `{ content: 'Why' }`, etc., ending with `{}` (empty delta).

---

## Summary: Cross-Cutting Capability Matrix

| Capability | Primary API surface | Key parameters |
| --- | --- | --- |
| Tenant isolation & data residency | Regional host prefixes; `store`, `expires_after`, `background`, `external_web_access`, `prompt_cache_retention` | Region domain, ZDR/MAM policy, EKM |
| Authentication & security | `Authorization` header, `OpenAI-Organization` header, `safety_identifier`, `OpenAI-Safety-Identifier` | API key, Admin API key, safety identifier |
| Workload identity federation | Token exchange (`/api/reference/workload-identity-federation`); dashboard config | OIDC issuer, audience, JWKS, CEL transformations, service account mapping |
| Permissions & RBAC | Dashboard + API key permissions; SCIM | Roles, permissions (Read/Write/Request/Use/Manage), groups |
| Admin APIs | `/v1/organization/projects/{id}/model_permissions`, `/spend_alerts`, `/data_retention`, `/invites`, `/audit_logs` | `mode`, `model_ids`, `threshold_amount`, `retention_type`, `email`, `role`, `limit` |
| Cost optimization & billing | `service_tier` (`"flex"`/`"priority"`), `prompt_cache_key`, `prompt_cache_retention` | Tier pricing, cache retention, timeout |
| Batch operations | `POST /v1/files`, `POST /v1/batches`, `GET /v1/batches/{id}`, `POST /v1/batches/{id}/cancel`, `GET /v1/batches` | `input_file_id`, `endpoint`, `completion_window`, `metadata`, `custom_id` |
| Quotas & rate limits | `GET /v1/fine_tuning/model_limits`; HTTP response headers | RPM/TPM/IPM, usage tiers, `max_tokens` |
| Logs, audit & data retention | `GET /v1/organization/audit_logs`; tracing SDK | `limit`, trace wrapping, abuse monitoring logs |
| Network security | Private Link (`/v2/privatelink_healthcheck`), IP egress JSON, `tunnel-client` CLI, `/v1/tunnel/*` | `base_url`, `--tunnel-id`, `--mcp-command`, `CONTROL_PLANE_API_KEY` |
| Orchestration & integrations | Agents SDK (`Agent`, `handoff`, `run`), MCP (`hostedMcpTool`, `MCPServerStdio`), Webhooks (`webhooks.unwrap`) | `handoffs`, `tools`, `serverLabel`, `require_approval`, webhook headers |
| Sandboxes & code execution | `SandboxAgent`, `Manifest`, `SandboxRunConfig`, provider clients | `defaultManifest`, `capabilities`, `client`, `image` |
| Guardrails & safety | Agents SDK guardrails (`@input_guardrail`, `GuardrailFunctionOutput`), approvals (`needsApproval`) | `tripwire_triggered`, `runInParallel`, `needs_approval`, `safety_identifier` |
| Performance optimization | `prompt_cache_key`, `prediction`, `context_management`, WebSocket `response.create`, `background`, `stream`, `POST /v1/responses/input_tokens`, `POST /v1/responses/compact` | `service_tier`, `prompt_cache_retention`, `compact_threshold`, `previous_response_id`, `starting_after` |
| Versioning & deprecations | Deprecations catalog, changelog | Notice periods (6mo GA / 3mo specialized / 2wk preview) |
| API conventions | `max_tokens`, `n`, `best_of`, `stream`, `stop`, `stream_options` | General request parameters |