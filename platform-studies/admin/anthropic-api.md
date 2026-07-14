# Anthropic Claude API — Cross-Cutting Concerns Analysis

> Source: documentation reachable from https://platform.claude.com/docs/en/home
> Scope: cross-cutting / administrative capabilities only. Core AI capabilities (messages generation, vision, tool-use, extended thinking, embeddings, RAG, structured outputs, citations, skills authoring, prompt engineering, etc.) are intentionally excluded.
> Generated: 2026-07-14

This document analyzes the cross-cutting capabilities exposed by the Anthropic Claude Platform API. For each capability it lists the **main concepts**, the **API functions/endpoints**, and the **parameters** that govern behavior.

---

## Table of Contents

1. [Tenant Isolation & Workspaces](#1-tenant-isolation--workspaces)
2. [Authentication & Security](#2-authentication--security)
3. [Workload Identity Federation](#3-workload-identity-federation-wif)
4. [Cost Optimization & Billing](#4-cost-optimization--billing)
5. [Subscription / Spend Management](#5-subscription--spend-management)
6. [Batch Operations](#6-batch-operations)
7. [Quotas & Limits](#7-quotas--limits)
8. [Logs, Audit & Data Retention](#8-logs-audit--data-retention)
9. [Compliance API](#9-compliance-api)
10. [Orchestration & Integrations (Managed Agents)](#10-orchestration--integrations-managed-agents)
11. [Licenses & Usage Rules](#11-licenses--usage-rules)
12. [Performance Optimization](#12-performance-optimization)
13. [Versioning](#13-versioning)
14. [Errors & API Conventions](#14-errors--api-conventions)

---

## 1. Tenant Isolation & Workspaces

**Source pages:** `manage-claude/workspaces`, `manage-claude/admin-api`, `api/admin`, `manage-claude/data-residency`

### Main concepts

- **Organization**: the top-level billing/identity container. A Claude Enterprise tenant has one **parent organization** centralizing identity, SSO, and SCIM, with **linked organizations** of two kinds:
  - `claude.ai` organizations (where users chat and store content)
  - Claude Console organizations (where users manage Claude API workloads)
- **Workspace**: a sub-organizational boundary used to isolate resources, API keys, members, rate limits, spend limits, and data-residency settings. API keys are scoped to a single workspace and can only access resources within it.
- **Resources scoped to workspaces**: Files (Files API), Message Batches (Batch API), Skills (Skills API).
- **Workspace geo**: controls where data is stored at rest and where endpoint processing (image transcoding, code execution) happens. Configured at workspace level in Console via `default_inference_geo` and `allowed_inference_geos`.
- **Inference geo**: per-request control of where model inference runs (`inference_geo` parameter on `POST /v1/messages`).

### API functions (Admin API — Workspaces)

| Function | Method & Endpoint | Purpose |
| --- | --- | --- |
| Create workspace | `POST /v1/organizations/workspaces` | Provision a new workspace under the org |
| Get workspace | `GET /v1/organizations/workspaces/{workspace_id}` | Retrieve workspace details |
| List workspaces | `GET /v1/organizations/workspaces` | Enumerate workspaces |
| Update workspace | `POST /v1/organizations/workspaces/{workspace_id}` | Modify settings (incl. geo) |
| Archive workspace | (archive action on workspace) | Retire a workspace |
| Add/update/remove workspace member | Workspace Members endpoints | Manage membership; see Workspace Members API reference |

### Key parameters

- `default_inference_geo` — default geo for inference (`"global"` | `"us"`)
- `allowed_inference_geos` — array restricting which geos keys in the workspace may use
- `inference_geo` (request-level) — `"global"` (default, any available geography) | `"us"` (US-only infrastructure); eligible for Zero Data Retention (ZDR)
- Workspace member role assignments

> Note: On Claude Platform on AWS, only workspace create/get/list/update/archive are available; member/invite/api-key/usage endpoints are not.

---

## 2. Authentication & Security

**Source pages:** `manage-claude/authentication`, `manage-claude/admin-api`, `manage-claude/admin-api-keys`, `manage-claude/cmek`, `manage-claude/access-transparency`

### Main concepts

- **Two authentication methods** for the Claude API:
  1. **API key** — static `sk-ant-api...` secret sent in `x-api-key` header. Best for local dev / single-tenant servers.
  2. **Workload Identity Federation (WIF)** — short-lived bearer token exchanged from an IdP identity token. Best for production cloud workloads (eliminates static secrets).
- **Key types** (distinct prefixes, distinct privileges):
  - `sk-ant-api03-...` — Claude API key (calls models)
  - `sk-ant-admin01-...` — Admin API key (manages org resources + Activity Feed read)
  - `sk-ant-api01-...` — Compliance Access Key (full Compliance API; claude.ai only)
  - Analytics API key (Claude Enterprise Analytics)
- **Key expiration**: an `expires_at` can be set at creation to bound a leaked credential's lifetime.
- **Customer-Managed Encryption Keys (CMEK)**: encrypt workspace data at rest with a key you control in AWS KMS, Google Cloud KMS, or Azure Key Vault. Cross-account key policy required; key registered and validated with Anthropic.
- **Access Transparency**: log of every access to your data by Anthropic staff/systems, plus preservation events (e.g. `cmek_preserve`). Queryable via Compliance Access Key.

### API functions (Admin API — API Keys)

| Function | Method & Endpoint | Purpose |
| --- | --- | --- |
| Create API key | `POST /v1/organizations/api_keys` | Issue a workspace-scoped key |
| Get API key | `GET /v1/organizations/api_keys/{key_id}` | Inspect a key |
| List API keys | `GET /v1/organizations/api_keys` | Enumerate keys |
| Update API key | `POST /v1/organizations/api_keys/{key_id}` | Rotate/name/edit; supports `expires_at` |
| Delete API key | `DELETE /v1/organizations/api_keys/{key_id}` | Revoke immediately (no grace period) |

### Key parameters / headers

- `x-api-key: $ANTHROPIC_ADMIN_KEY` (or `ANTHROPIC_API_KEY`, `ANTHROPIC_AUTH_TOKEN`)
- `anthropic-version: 2023-06-01` (required on all requests)
- `expires_at` — RFC 3339 timestamp bounding key validity

### Access Transparency query

```bash
curl -G "https://api.anthropic.com/v1/compliance/access_events" \
  --data-urlencode "activity_types[]=cmek_preserve" \
  --data-urlencode "limit=50" \
  --header "x-api-key: $ANTHROPIC_COMPLIANCE_ACCESS_KEY"
```

---

## 3. Workload Identity Federation (WIF)

**Source pages:** `manage-claude/workload-identity-federation`, `manage-claude/wif-reference`, `manage-claude/wif-providers/gcp`, `manage-claude/wif-providers/okta`, `api/admin`

### Main concepts

- **Federation removes long-lived Claude API keys** from workloads: the workload exchanges an IdP-issued OIDC JWT for an Anthropic bearer token at runtime. Shrinks blast radius of leaked credentials.
- **Resources** (managed via Admin API, require `org:admin` OAuth token — Admin API keys are NOT accepted for these):
  - **Service account** (`svac_...`) — the identity the minted token acts as
  - **Federation issuer** — represents the trusted IdP
  - **Federation rule** (`fdrl_...`) — binds issuer claims to a service account and workspace(s)
- **Token lifetime**: 60–86400 seconds (default 3600). SDK handles refresh loop.
- **Credential resolution order**: constructor arg → `ANTHROPIC_API_KEY`/`ANTHROPIC_AUTH_TOKEN` → `ANTHROPIC_PROFILE` → federation env vars → active profile → `default`.

### API functions (Admin API — Service Accounts / Federation)

| Function | Method & Endpoint |
| --- | --- |
| Create service account | `POST /v1/organizations/service_accounts` |
| Get service account | `GET /v1/organizations/service_accounts/{service_account_id}` |
| List service accounts | `GET /v1/organizations/service_accounts` |
| Update service account | `POST /v1/organizations/service_accounts/{service_account_id}` |
| Archive service account | `POST /v1/organizations/service_accounts/{service_account_id}/archive` |
| Add workspace to service account | Service Accounts Workspaces endpoints |
| Federation issuers | `POST/GET /v1/organizations/federation_issuers...` |
| Federation rules | `POST/GET /v1/organizations/federation_rules...` |

### Token exchange parameters (OAuth 2.0 jwt-bearer grant)

| Field | Required | Description |
| --- | --- | --- |
| `grant_type` | Yes | `urn:ietf:params:oauth:grant-type:jwt-bearer` |
| `assertion` | Yes | OIDC JWT issued by your IdP |
| `federation_rule_id` | Yes | `fdrl_...` |
| `organization_id` | Yes | UUID of your Anthropic org |
| `service_account_id` | Yes | `svac_...` |
| `workspace_id` | Conditional | `wrkspc_...` or literal `default`; required when rule enabled for >1 workspace |

Response: `access_token` (`sk-ant-oat01-...`), `expires_in`, `token_lifetime_seconds`.

### Provider examples

- **Google Cloud**: attach dedicated service account; request identity token with `format=full` (so `email` claim present); match on `sub` + `email`.
- **Okta**: RFC 7523 `client_assertion` JWT signed with app private key; `grant_type=client_credentials`, `scope=anthropic.access`.

---

## 4. Cost Optimization & Billing

**Source pages:** `build-with-claude/usage-cost-api`, `manage-claude/usage-cost-api`, `about-claude/pricing`, `build-with-claude/prompt-caching`, `api/service-tiers`

### Main concepts

- **Usage & Cost API**: programmatic monitoring, analysis, reconciliation. Data appears within ~5 min of API request completion. Polling recommendation: once per minute sustained.
- **Cost reconciliation**: match internal records with Anthropic billing.
- **Prompt caching multipliers** (relative to base input price):
  - 5-min cache write: **1.25x**
  - 1-hour cache write: **2x**
  - Cache read (hit): **0.1x** — pays off after 1 read (5-min) or 2 reads (1-hour)
- **Batch API discount**: **50% off** standard pricing on batch requests.
- **Data residency modifier**: US-only inference priced at **1.1x** on Opus 4.6 / Sonnet 4.6 and later.
- **Priority Tier**: a committed capacity tier; burndown rates reflect relative pricing per token type. Drawn against both Priority capacity and regular rate limits.

### API functions (Admin API — Usage/Cost Reports)

| Function | Method & Endpoint | Purpose |
| --- | --- | --- |
| Usage report | `GET /v1/organizations/usage_report` | Token usage broken down by dimensions |
| Cost report | `GET /v1/organizations/cost_report` | Billed cost (excludes Priority Tier, which uses different billing model) |

### Key parameters

- `starting_at`, `ending_at` — RFC 3339 time bounds
- `group_by[]` — e.g. `workspace_id`, `description` (multi-dimensional grouping)
- Pagination: cursor-based (`page`/`next_page` or `after_id`/`before_id` depending on endpoint)
- `anthropic-version: 2023-06-01`, `x-api-key: $ANTHROPIC_ADMIN_KEY`

### Service tiers (`service_tier` request parameter)

| Value | Behavior |
| --- | --- |
| `"auto"` (default) | Use Priority Tier capacity if available, fall back to other capacity |
| `"standard_only"` | Only standard tier; skip Priority capacity |

Response `usage.service_tier` reports which tier was assigned. Priority response headers: `anthropic-priority-input-tokens-limit/remaining/reset`, `anthropic-priority-output-tokens-limit/remaining/reset`.

### Claude Code Analytics API

`GET /v1/organizations/claude_code/analytics` — monitors developer productivity and Claude Code adoption. Free for orgs with Admin API access. Tracks only Claude API usage (not AWS/Bedrock/Foundry/Vertex). Retained indefinitely.

Parameters: `starting_at`, `limit` (default 20), `page` (cursor e.g. `page_MjAyNS0wNS0xNFQwMDowMDowMFo=`).

---

## 5. Subscription / Spend Management

**Source pages:** `manage-claude/spend-limits-api`

### Main concepts

- **Spend limit**: identified by the `(scope, period)` pair. Currently only `period: "monthly"` is supported; monthly spend resets at 00:00 UTC on the first of each calendar month.
- **Scope hierarchy**: organization-level limits cascade; workspace-level overrides inherit from org when absent. Each member has an **effective spend limit** with a `source` indicating where it was set.
- **Monetary values**: strings in **minor units** of the org's billing currency (cents for USD). E.g. `"50000"` = 500.00 USD. Parse as decimal, divide by 100; avoid binary floating-point.
- **Increase requests**: members can request a spend-limit increase; admins approve/deny.
- **`period_to_date_spend`**: spend accrued in the current period.

### API functions

| Function | Method & Endpoint | Scope required |
| --- | --- | --- |
| List effective spend limits (per member) | `GET /v1/organizations/spend_limits/effective` | `read:spend_limits` |
| Set/update spend limit | (spend limits endpoints) | write scope |
| List spend limit increase requests | (requests endpoints) | `read:spend_limits` |
| Approve increase request | `POST /v1/organizations/spend_limit_increase_requests/{id}/approve` | write scope |
| Deny increase request | `POST /v1/organizations/spend_limit_increase_requests/{id}/deny` | write scope |

### Key parameters

- `amount` — string, minor units (e.g. `"75000"`)
- `period` — `"monthly"` (treat as open set)
- `suppress_notification` — boolean (approve endpoint)
- `user_ids[]` — filter by member (array form: `user_ids[]=user_01AbCdEfGh&user_ids[]=user_01JkLmNoPq`)
- `scope` — org / workspace / user

### Error handling

Standard error shape with `request_id`; quote it when contacting support.

---

## 6. Batch Operations

**Source pages:** `build-with-claude/batch-processing`, `api/listing-message-batches`, `api/retrieving-message-batches`, `api/canceling-message-batches`, `api/rate-limits`

### Main concepts

- **Message Batch**: a list of independent Messages API requests processed **asynchronously** with a **50% cost discount**. Each batch request is a standard Messages API call (vision, tool use, system messages, multi-turn, extended thinking, most beta features supported).
- **Batch request shape**: `custom_id` (1–64 chars, `^[a-zA-Z0-9_-]{1,64}$`) + `params` (standard Messages params). Requests processed independently; mix types within a batch.
- **Lifecycle**: `in_progress` → (`canceling`) → `ended`. Batches **expire 24h after creation**. Results delivered as a `.jsonl` file (not guaranteed in request order; match via `custom_id`).
- **Result types** (4): `succeeded`, `errored`, `canceled`, `expired`.
- **Unsupported in batches**: a small number of Messages params (validation error if included).

### API functions

| Function | Method & Endpoint | Notes |
| --- | --- | --- |
| Create Message Batch | `POST /v1/messages/batches` | Body: `requests[]` (max 100,000 batch requests per batch) |
| Retrieve Message Batch | `GET /v1/messages/batches/{message_batch_id}` | Idempotent; poll for completion |
| List Message Batches | `GET /v1/messages/batches` | Newest first; pagination |
| Cancel Message Batch | `POST /v1/messages/batches/{message_batch_id}/cancel` | Any time before `ended`; may finish non-interruptible in-progress first |
| Get results | `GET {results_url}` | `.jsonl` file once `ended` |

### List/Retrieve parameters

- `after_id`, `before_id` (optional string) — cursor pagination
- `limit` (optional number) — default 20, range 1–1000
- Path: `message_batch_id` (string)

### Response object — `MessageBatch`

| Field | Type | Description |
| --- | --- | --- |
| `id` | string | unique object ID |
| `type` | `"message_batch"` | constant |
| `created_at` | RFC 3339 | creation time |
| `expires_at` | RFC 3339 | 24h after creation |
| `ended_at` | RFC 3339 | when processing ended |
| `cancel_initiated_at` | RFC 3339 | if cancellation initiated |
| `archived_at` | RFC 3339 | when results became unavailable |
| `processing_status` | `in_progress` \| `canceling` \| `ended` | |
| `request_counts` | object | `{canceled, errored, expired, processing, succeeded}` — sums equal total |
| `results_url` | string | URL to `.jsonl` results (only once ended) |

List response also returns: `first_id`, `last_id`, `has_more`.

### Batch rate limits (shared across all models)

| Metric | Limit |
| --- | --- |
| Max requests per minute (RPM) to all Batch endpoints | 1,000 |
| Max batch requests in processing queue | 200,000 |
| Max batch requests per batch | 100,000 |

---

## 7. Quotas & Limits

**Source pages:** `api/rate-limits`, `manage-claude/rate-limits-api`, `api/overview`, `build-with-claude/token-counting`

### Main concepts

- **Usage tiers**: org auto-placed on a tier and can graduate higher over time. Each tier grants requests-per-minute (RPM), input tokens per minute (ITPM), output tokens per minute (OTPM) per model class.
- **Rate limit dimensions**: RPM, ITPM, OTPM. Exceeding any returns `429` with a `retry-after` header. **Acceleration limits** also apply on sharp usage spikes — ramp gradually.
- **Cache-aware ITPM**: cache-read tokens counted differently for ITPM budgeting.
- **Rate Limits API**: programmatic read of configured limits at org and workspace level. Workspace overrides inherit org value when absent; `org_limit` field shows the org value (or `null`).
- **Token Counting rate limits** (independent from message creation): Start 2,000 / Build 4,000 / Scale 8,000 RPM.
- **Max request sizes**: Messages 32 MB, Token Counting 32 MB, Batch 256 MB, Files 500 MB (else `413 request_too_large`).

### API functions

| Function | Method & Endpoint |
| --- | --- |
| List organization rate limits | `GET /v1/organizations/rate_limits` |
| List workspace rate limits | `GET /v1/organizations/workspaces/{workspace_id}/rate_limits` (per Workspace Rate Limits API reference) |

### Rate limit response headers (every API response)

```
anthropic-ratelimit-requests-limit
anthropic-ratelimit-requests-remaining
anthropic-ratelimit-requests-reset
anthropic-ratelimit-input-tokens-limit
anthropic-ratelimit-input-tokens-remaining
anthropic-ratelimit-input-tokens-reset
anthropic-ratelimit-output-tokens-limit
anthropic-ratelimit-output-tokens-remaining
anthropic-ratelimit-output-tokens-reset
retry-after   # on 429
```

### Response semantics (Rate Limits API)

- Response only includes **overrides**. Missing group → no workspace override (inherits org, NOT unlimited). Within a present group, missing limiter type → no override. `org_limit` = org-level value or `null`.

---

## 8. Logs, Audit & Data Retention

**Source pages:** `manage-claude/api-and-data-retention`, `build-with-claude/api-and-data-retention`, `manage-claude/access-transparency`, `manage-claude/compliance-activity-feed`

### Main concepts

- **Zero Data Retention (ZDR)**: when org has a ZDR arrangement, data sent through ZDR-eligible features is not stored after the API response returns. Eligible features include: Messages (default for many features), Token Counting, Cache Diagnostics, Inference geo (`us`).
- **Non-ZDR / stateful resources**:
  - **Claude Managed Agents** (`/v1/agents`, `/v1/sessions`, `/v1/environments`) — sessions are stateful; transcripts persist until deleted (no automatic deletion). Applies to all sub-features incl. self-hosted sandboxes.
  - **Code execution** tool — container data retained up to **30 days**.
- **Workspace-level data retention override**: orgs with ZDR can enable **30-day data retention** per workspace (Console > Settings > Workspaces > Privacy controls). Other workspaces keep ZDR. Makes Claude Fable 5 / Mythos 5 available in that workspace.
- **Activity Feed retention**: activities queryable within 1 minute of occurring, retained **6 years**.
- **Cache diagnostics**: ZDR-eligible (qualified). Stores only cryptographic hashes + token-count estimates (never raw content), keyed by response `id`, scoped to org+workspace, short expiry.
- **Access Transparency**: records staff/system access to your data; plus preservation events (`cmek_preserve`).

### Token Counting (`POST /v1/messages/count_tokens`)

- Accepts the same structured inputs as message creation (system prompts, tools, images, PDFs). Returns total input tokens. Free but RPM-limited by tier. ZDR-eligible.

### HIPAA readiness

- Self-serve enablement (Console > Settings > Privacy) for eligible orgs: download BAA + Implementation Guide, accept as authorized legal rep. Enablement is **permanent** (cannot be disabled). API auto-enforces feature restrictions (returns errors for non-eligible features).
- Custom BAA via sales team.
- PHI guidance: PHI appears in message content, attached files, filenames/metadata. NOT in workspace names, user info, billing, support tickets.
- `strict: true` schemas compiled to grammars cached separately from message content — do NOT receive the same PHI protections.

---

## 9. Compliance API

**Source pages:** `manage-claude/compliance-api`, `manage-claude/compliance-api-access`, `manage-claude/compliance-activity-feed`, `manage-claude/compliance-content-data`, `manage-claude/compliance-integration-patterns`, `manage-claude/compliance-faq`

### Main concepts

- **Purpose**: read chat content/attachments and delete on demand; enumerate org directory (organizations, users, roles, groups, settings); support eDiscovery, DLP, account-deletion, SIEM correlation, chain-of-custody exports.
- **Tenant model**: parent organization centralizes identity; linked organizations are `claude.ai` (chat content) and Claude Console (API workloads). A key covering the parent covers every linked org. Content endpoints serve `claude.ai` data only; directory endpoints serve both kinds.
- **Shared rate limit**: **600 requests per minute per parent organization**, shared across every key, every linked org, every `/v1/compliance/` endpoint.
- **Pagination schemes** (endpoint-family dependent):

| Endpoint family | Sort order | Scheme | Parameters |
| --- | --- | --- | --- |
| Activities | Newest first | Cursor | `after_id`, `before_id` (returned as `first_id`, `last_id`) |
| Chats & chat messages | Oldest first | Cursor | `after_id`, `before_id` |
| Organizations, projects, project attachments, users, roles, role permissions, groups, group members | Endpoint-specific | Page token | `page` (returned as `next_page`) |

- **Cursors are opaque** — never parse; bound to the sort key (mismatched `order_by` + `after_id` → 400). Time filters must match sort key.
- **Retention**: activities 6 years; content per org retention policy. Soft-deleted (claude.ai) chats still visible with `deleted_at` populated; hard-deleted (via Compliance API or after retention window) are not retrievable.
- **Max page size**: `limit` capped at **5,000**.
- **Chain of custody**: store exported records with provenance (source endpoint, query params, run timestamp, content hash).

### Key types & scopes

| Key type | Created in | Coverage | Scopes available |
| --- | --- | --- | --- |
| Compliance Access Key (`sk-ant-api01-...`) | claude.ai > Org settings > API | parent or single org | all scopes (immutable after creation) |
| Admin API key (`sk-ant-admin01-...`) | Claude Console > Settings > Admin keys | Console org | `read:compliance_activities` only (and only if Compliance API enabled before key creation) |

### Scopes (Compliance Access Key only)

| Scope | Grants |
| --- | --- |
| `read:compliance_activities` | Read Activity Feed |
| `read:compliance_user_data` | Read user chats, messages, files, projects, org users, group members |
| `delete:compliance_user_data` | Delete user chats, files, projects (permanent, immediate, no recovery) |
| `read:compliance_org_data` | Read org metadata (names, types, roles, groups) + effective settings; user listings/group membership also need `read:compliance_user_data` |

> Recommended: smallest scope set; use two keys (read + delete) so a leaked read key cannot delete. Scopes immutable — to change, create new key + delete old. Rotation: create new → update integration → verify → delete old. Cursors are org-scoped (remain valid across rotation).

### API functions — Activity Feed

| Function | Method & Endpoint |
| --- | --- |
| List activities | `GET /v1/compliance/activities` |

**Parameters**: `activity_types[]`, `limit` (default 100, max 5000), `after_id`, `before_id`, time-range bounds.

### API functions — Chats / Files / Projects (claude.ai content)

| Function | Endpoint |
| --- | --- |
| List chats | `GET /v1/compliance/apps/chats` |
| Get chat messages | `GET /v1/compliance/apps/chats/{chat_id}/messages` |
| Delete chat | `DELETE /v1/compliance/apps/chats/{chat_id}` (also removes messages + attached files) |
| Download file content | `GET /v1/compliance/apps/chats/files/{file_id}/content` |
| Get file metadata | `GET /v1/compliance/apps/chats/files/{file_id}` |
| Delete file | `DELETE /v1/compliance/apps/chats/files/{file_id}` |
| Download generated file | `GET /v1/compliance/apps/chats/generated_files/{gen_file_id}/content` |
| Download artifact content | `GET /v1/compliance/apps/artifacts/{artifact_version_id}/content` |
| List projects | `GET /v1/compliance/apps/projects` |
| Get project details | `GET /v1/compliance/apps/projects/{project_id}` |
| List project attachments | `GET /v1/compliance/apps/projects/{project_id}/attachments` |
| Get project document content | `GET /v1/compliance/apps/projects/documents/{proj_doc_id}/content` |
| Delete project document | `DELETE /v1/compliance/apps/projects/documents/{proj_doc_id}` |
| Delete project | `DELETE /v1/compliance/apps/projects/{project_id}` (409 if chats attached) |

### Chat list parameters

- `order_by` — `created_at` (default) or `updated_at`
- `updated_at.gte` / `created_at.gte` (and `gt`/`lt`/`lte`) — must match `order_by`
- `user_ids[]` — 1–10 values; when present, sort is always `created_at` and `project_ids[]` filter becomes available
- `limit`, `after_id`, `before_id`
- Organization-wide (omit `user_ids[]`): forward-only pagination (`after_id`), no `before_id`, no `project_ids[]`

### Chat messages response fields

`id`, `role`, `created_at`, `content[]` (text), `files[]` (user uploads), `generated_files[]` (tool outputs), `artifacts[]` (versioned docs with `version_id`).

### Attachment discriminator

`type`: `project_file` (binary, `claude_file_*` → file content endpoint) vs `project_doc` (plain-text `text/plain`, `claude_proj_doc_*` → project document content endpoint).

### Delete responses

Small envelope: `{ "id": "...", "type": "claude_chat_deleted" }`. Project delete returns 409 `conflict_error` if chats attached — detach/delete chats first via `GET /v1/compliance/apps/chats?user_ids[]={uid}&project_ids[]={pid}`.

---

## 10. Orchestration & Integrations (Managed Agents)

**Source pages:** `managed-agents/overview`, `managed-agents/reference`, `managed-agents/webhooks`, `managed-agents/mcp-connector`, `managed-agents/scheduled-deployments`, `managed-agents/environments`, `managed-agents/permission-policies`, `managed-agents/vaults`, `agents-and-tools/remote-mcp-servers`

> All Managed Agents API requests require the `managed-agents-2026-04-01` beta header (SDK sets it automatically).

### Main concepts

- **Agent**: reusable, versioned configuration (tools, MCP servers, permission policies, model).
- **Session**: stateful agent execution in a managed cloud sandbox (or self-hosted). Transcripts persist until deleted.
- **Environment**: sandbox template (cloud or self-hosted) with networking controls.
- **Vault**: collection of credentials registered once, referenced by ID at session creation to authenticate MCP servers.
- **MCP connector**: connect Model Context Protocol servers to agents for external tools/data. Up to **20 MCP servers** per agent; names unique; every `mcp_servers` entry must be referenced by an `mcp_toolset`.
- **Permission policies**: control whether server-executed tools (agent toolset + MCP toolset) run automatically or wait for human approval. Custom tools are app-executed (not governed).
- **Scheduled deployments**: run an agent on a recurring cron schedule; emits webhook events on lifecycle changes and run outcomes.
- **Webhooks**: HTTPS endpoint (port 443) subscribed to event types; signed with `whsec_`-prefixed 32-byte secret; `X-Webhook-Signature` header; payloads valid ≤5 min.
- **Self-hosted worker**: for self-hosted sandboxes, integration provides `agent_toolset` results (`user.tool_result`).

### API functions

| Function | Method & Endpoint |
| --- | --- |
| Create agent | `POST /v1/agents` |
| List/get/update agent | `GET/POST /v1/agents...` |
| Create environment | `POST /v1/environments` |
| Create session | `POST /v1/sessions` |
| Stream session events | `GET /v1/sessions/{id}/stream` |
| Send session event | (events send) |
| Create deployment | `POST /v1/deployments` |
| List deployment runs | `GET /v1/deployment_runs` (filter: `--deployment-id`, error filter) |
| Get deployment run | `GET /v1/deployment_runs/{deployment_run_id}` |
| Pause / unpause / archive deployment | deployment lifecycle endpoints |
| Trigger manual run | deployment `run` endpoint (allowed while paused) |
| Create vault | `POST /v1/vaults` |
| Create vault credential | `POST /v1/vaults/credentials` |

### Environment parameters

- `config.type` — `cloud` | self-hosted
- `networking.type` — `unrestricted` (default, full outbound minus safety blocklist) | `limited` (restrict to `allowed_hosts`)
- `networking.allowed_hosts` — bare hostnames or wildcard (`.example.com`); no scheme/port/path
- `networking.allow_mcp_servers` — bool, default false (outbound to MCP endpoints beyond `allowed_hosts`)
- `networking.allow_package_managers` — bool, default false (PyPI/npm beyond `allowed_hosts`)
- Does NOT affect `web_search`/`web_fetch` tool allowed domains

### MCP toolset parameters

- `default_config.enabled` — default false to allow explicit per-tool enabling
- `configs[]` — per-tool overrides keyed by bare tool name reported by server
- MCP tool output >100,000 tokens → auto-written to sandbox file; model gets truncated preview + file path

### Vault credential auth types

- `environment_variable` — not yet supported with self-hosted sandboxes
- `mcp_oauth` — OAuth 2.0; `access_token`, `expires_at`, optional `refresh` block
  - `refresh.token_endpoint`, `client_id`, `scope`, `refresh_token`
  - `refresh.token_endpoint_auth.type`: `none` | `client_secret_basic` | `client_secret_post`
- Sensitive write-only fields (`token`, `access_token`, `refresh_token`, `client_secret`, `secret_value`) never returned in responses

### Permission policy actions (session events)

```json
{ "type": "user.tool_confirmation", "tool_use_id": "...", "result": "allow" }
{ "type": "user.tool_confirmation", "tool_use_id": "...", "result": "deny", "deny_message": "..." }
```

### Session event types (`user.*`)

`user.message`, `user.interrupt`, `user.custom_tool_result`, `user.tool_confirmation`, `user.define_outcome`, `user.tool_result` (self-hosted only).

### Webhook event types (vault examples)

`vault.archived`, `vault.deleted`, `vault_credential.archived`, `vault_credential.deleted`, `vault_credential.refresh_failed`, deployment lifecycle events, deployment run events, session events.

### Managed Agents rate limits (per organization)

| Operation | Limit |
| --- | --- |
| Create endpoints (agents, sessions, environments) | 300 RPM |
| Read endpoints (retrieve, list, stream) | 1,200 RPM |

Org-level spend limits and usage-tier rate limits also apply.

---

## 11. Licenses & Usage Rules

**Source pages:** `about-claude/pricing`, `about-claude/model-deprecations`, `api/supported-regions`, `manage-claude/api-and-data-retention` (HIPAA / PHI), `manage-claude/cmek` (legal retention exceptions)

### Main concepts

- **Supported regions**: Claude API accessible from a defined list of countries/territories (Albania, Algeria, … United States, etc. — full enumerated list). Access unsupported from others.
- **Model deprecation policy**: Anthropic gives **≥60 days notice** before retiring publicly released models. Deprecation notices sent to customers with active deployments. Migration guide provided.
  - Audit usage: Console > Usage > Export (CSV by API key and model).
  - Example retirements: `claude-sonnet-4-20250514` → `claude-sonnet-4-6`; `claude-opus-4-20250514` → `claude-opus-4-8`; `claude-3-haiku-20240307` → `claude-haiku-4-5-20251001`.
- **API parameter deprecations**: tracked separately (see Model deprecations page).
- **Pricing model**: per-token (input/output), with feature multipliers (caching, batch, data residency). Session runtime priced per hour (e.g. `$0.08/hr` for Managed Agents).
- **HIPAA / BAA**: self-serve or custom; once enabled, configuration is **permanent**; API enforces eligible-feature restrictions automatically.
- **PHI handling**: PHI in message content / files / metadata; not in workspace names, user info, billing, support tickets. `strict: true` schemas cached separately — not PHI-protected.
- **CMEK legal retention exceptions**: Anthropic may retain records where required by law (e.g. NCMEC reports under 18 U.S.C. § 2258A), exigent risk of serious harm (CBRNE, offensive cyberattacks, imminent violence), or ToS violations.
- **Customer Type** (analytics): `api` (pay-as-you-go) vs `subscription` (Pro/Team plans).

---

## 12. Performance Optimization

**Source pages:** `build-with-claude/prompt-caching`, `build-with-claude/cache-diagnostics`, `build-with-claude/token-counting`, `build-with-claude/context-windows`, `build-with-claude/usage-cost-api`

### Main concepts

- **Prompt caching**: cache prompt prefixes via `cache_control` to cut cost and latency. Two modes:
  - **Automatic caching**: single `cache_control` at top level; system manages breakpoints as conversations grow. Recommended starting point.
  - **Explicit cache breakpoints**: `cache_control` on individual content blocks for fine-grained control.
- **TTLs**: 5-minute (default) and 1-hour cache writes. Cache hit costs 10% of standard input price.
- **Cache invalidation**: byte-for-byte prefix identity required. Reordered tool, interpolated timestamp, or edited earlier message silently invalidates. Only signal without diagnostics: `usage.cache_read_input_tokens` dropping to 0.
- **Cache diagnostics** (beta): per-request fingerprint comparison reveals `cache_miss_reason` (e.g. `_changed` field with `type`/`cache_control`). Pass `previous_message_id` (null on first turn, prior response `id` thereafter). Fingerprints = hashes + token-count estimates (never raw content), scoped to org+workspace, short expiry. ZDR-eligible.
- **Token Counting**: estimate tokens before sending to manage costs/rate limits/context-window fit. Separate, independent rate limits from message creation. Free. Migration note: tokenizer changed at Claude Opus 4.7 — same content consumes ~30% more tokens on Fable 5 / Mythos 5; don't reuse old counts.
- **Context window**: everything counts (system prompt, messages, tool results, images, docs, output, extended thinking). `usage` splits input across `input_tokens`, `cache_read_input_tokens`, `cache_creation_input_tokens` — all count toward the window. Overflow → 400 `invalid_request_error` ("prompt is too long"). Cached prefixes still occupy the window (caching changes price, not count). Context rot: accuracy/recall degrade as token count grows.

### Key parameters

- `cache_control` — `{ "type": "ephemeral" }` (with optional `ttl: "5m"` | `"1h"`)
- `diagnostics` — beta field enabling cache diagnostics
- `previous_message_id` — response `id` for fingerprint comparison; `null` on first turn to opt in
- Token counting endpoint: `model` (e.g. `"claude-fable-5"`), standard message input shape

### Usage object (response)

```json
{
  "usage": {
    "input_tokens": 410,
    "cache_creation_input_tokens": 0,
    "cache_read_input_tokens": 0,
    "output_tokens": 585,
    "service_tier": "priority"
  }
}
```

---

## 13. Versioning

**Source pages:** `api/versioning`, `api/beta-headers`, `api/overview`, `release-notes/api`

### Main concepts

- **API version** sent via `anthropic-version` header (e.g. `anthropic-version: 2023-06-01`). Required on all requests.
- **Backward compatibility**: for any given version with the Messages API, Anthropic preserves documented behavior. May add new optional fields/parameters, add new `enum` values, add new response fields. Generally will not break documented usage. Previous versions deprecated and may be unavailable for new users.
- **Beta headers**: `anthropic-beta` header (or SDK `betas` parameter / `beta` namespace) accesses experimental features before GA. Each feature doc states the exact beta name. Invalid/inaccessible beta → 400 `invalid_request_error`.
- **Managed Agents beta header**: `managed-agents-2026-04-01` required for all Managed Agents requests (SDK sets automatically).
- **Pagination convention differences**:
  - Cursor scheme (`after_id`/`before_id`, returns `has_more`/`first_id`/`last_id`): Message Batches, Files, Models, several Admin endpoints.
  - Page-token scheme (`page`, returns `next_page`): most list endpoints; some (e.g. `GET /v1/skills`) return `has_more` alongside `next_page`.
- **Available APIs (GA vs Beta)**:

| API | Status | Endpoint |
| --- | --- | --- |
| Messages | GA | `POST /v1/messages` |
| Message Batches | GA | `POST /v1/messages/batches` |
| Token Counting | GA | `POST /v1/messages/count_tokens` |
| Models | GA | `GET /v1/models` |
| Files | Beta | `POST /v1/files`, `GET /v1/files` |
| Skills | Beta | `POST /v1/skills`, `GET /v1/skills` |
| Agents | Beta | `POST /v1/agents`, `GET /v1/agents` |
| Sessions | Beta | `POST /v1/sessions`, `GET /v1/sessions/{id}/stream` |
| Environments | Beta | `POST /v1/environments`, `GET /v1/environments` |

- **Release notes timeline highlights**: Admin API (Nov 21, 2024), Messages API ITPM/OTPM split (Nov 20, 2024), Usage & Cost API + Org Info endpoint (Aug 18, 2025), Rate Limits API (Apr 24, 2026), Memory for Managed Agents public beta (Apr 23, 2026).

---

## 14. Errors & API Conventions

**Source pages:** `api/errors`, `api/overview`, `api/supported-regions`

### HTTP error codes

| Code | Type | Meaning |
| --- | --- | --- |
| 400 | `invalid_request_error` | Format/content issue (also default for unlisted 4XX) |
| 401 | `authentication_error` | API key issue (or AWS creds/SigV4 on AWS) |
| 403 | `permission_error` | Missing scopes (`Missing required scopes. Got: [...] Needed: [...]`) |
| 404 | `not_found_error` | Resource not found |
| 409 | `conflict_error` | e.g. deleting a project with attached chats |
| 413 | `request_too_large` | Exceeds max request size (returned by Cloudflare pre-API on direct API) |
| 429 | `rate_limit_error` | Exceeded RPM/ITPM/OTPM; includes `retry-after` header |
| 5xx | `api_error` | Server-side |

### Error shape (always JSON)

```json
{
  "type": "error",
  "error": { "type": "not_found_error", "message": "The requested resource could not be found." },
  "request_id": "req_011CSHoEeqs5C35K2UUqR7Fy"
}
```

Always quote `request_id` when contacting support.

### Max request sizes (else 413)

| Endpoint | Max size |
| --- | --- |
| Messages API | 32 MB |
| Token Counting API | 32 MB |
| Batch API | 256 MB |
| Files API | 500 MB |

### Admin API navigation (endpoint groups from `api/admin`)

Organizations, Invites, Users, Workspaces, API Keys, External Keys, Usage Report, Cost Report, Analytics, Spend Limits, Rate Limits, Service Accounts, Federation Issuers, Federation Rules, MCP Tunnels; Compliance API (Activities, Organizations, Groups, Apps, Code, Completions); plus Claude Code (Trigger a routine).

### Compliance API errors (selected)

- `400` — bad cursor (`after_id` under mismatched `order_by`), bad time-filter/sort-key pairing
- `401` — bad key
- `403` — `permission_error` with `Missing required scopes...` (e.g. Admin key hitting content endpoint)
- `404` — resource not found
- `409` — `conflict_error` (project delete with attached chats)
- `429` — shared 600/min/parent-org limit; response headers + retry contract
- `5xx` — server errors

---

## Appendix — Documentation Source Inventory (cross-cutting pages analyzed)

**Authentication & Security**
- `manage-claude/authentication`
- `manage-claude/workload-identity-federation`
- `manage-claude/wif-reference`
- `manage-claude/wif-providers/gcp`
- `manage-claude/wif-providers/okta`
- `manage-claude/admin-api-keys`
- `manage-claude/cmek`
- `manage-claude/access-transparency`

**Tenant Isolation**
- `manage-claude/workspaces`
- `manage-claude/data-residency`
- `manage-claude/admin-api`
- `api/admin`

**Cost & Billing**
- `build-with-claude/usage-cost-api`
- `manage-claude/usage-cost-api`
- `manage-claude/claude-code-analytics-api`
- `about-claude/pricing`
- `api/service-tiers`

**Spend Management**
- `manage-claude/spend-limits-api`

**Batch**
- `build-with-claude/batch-processing`
- `api/listing-message-batches`
- `api/retrieving-message-batches`
- `api/canceling-message-batches`

**Quotas & Limits**
- `api/rate-limits`
- `manage-claude/rate-limits-api`
- `api/overview`
- `build-with-claude/token-counting`

**Logs, Audit & Retention**
- `manage-claude/api-and-data-retention`
- `build-with-claude/api-and-data-retention`

**Compliance**
- `manage-claude/compliance-api`
- `manage-claude/compliance-api-access`
- `manage-claude/compliance-activity-feed`
- `manage-claude/compliance-content-data`
- `manage-claude/compliance-integration-patterns`
- `manage-claude/compliance-faq`
- `api/compliance`

**Orchestration & Integrations**
- `managed-agents/reference`
- `managed-agents/webhooks`
- `managed-agents/mcp-connector`
- `managed-agents/scheduled-deployments`
- `managed-agents/environments`
- `managed-agents/permission-policies`
- `managed-agents/vaults`
- `agents-and-tools/remote-mcp-servers`

**Licenses & Usage Rules**
- `about-claude/model-deprecations`
- `api/supported-regions`

**Performance**
- `build-with-claude/prompt-caching`
- `build-with-claude/cache-diagnostics`
- `build-with-claude/context-windows`

**Versioning & Errors**
- `api/versioning`
- `api/beta-headers`
- `api/errors`
- `release-notes/api`