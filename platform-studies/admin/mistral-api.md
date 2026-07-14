# Mistral AI API — Cross-Cutting Concerns Analysis

> Source: documentation reachable from https://docs.mistral.ai/
> Scope: cross-cutting / administrative capabilities only. Core AI capabilities (chat completions, embeddings, FIM, OCR, audio, classifiers, agents, RAG, fine-tuning, models, prompt engineering, etc.) are intentionally excluded.
> Generated: 2026-07-14

This document analyzes the cross-cutting capabilities exposed by the Mistral platform. For each capability it lists the **main concepts**, the **API functions/endpoints**, and the **parameters** that govern behavior.

Mistral exposes two distinct API surfaces relevant here:
- **La Plateforme / Studio API** (`https://api.mistral.ai/v1/...`) — the inference and application API, authenticated with standard API keys in the `Authorization: Bearer` header.
- **Admin API** (`https://api.mistral.ai/...`, `/api/admin/...` paths) — Enterprise-only account management API, authenticated with dedicated Admin API keys in the `x-api-key` header. Managed in Preview.

---

## Table of Contents

1. [Tenant Isolation & Workspaces](#1-tenant-isolation--workspaces)
2. [Authentication & Security](#2-authentication--security)
3. [Cost Optimization & Billing](#3-cost-optimization--billing)
4. [Subscription Management](#4-subscription-management)
5. [Batch Operations](#5-batch-operations)
6. [Quotas & Limits](#6-quotas--limits)
7. [Logs, Audit & Data Retention](#7-logs-audit--data-retention)
8. [Observability](#8-observability)
9. [Orchestration & Integrations (Connectors)](#9-orchestration--integrations-connectors)
10. [Privacy & Compliance](#10-privacy--compliance)
11. [Performance Optimization](#11-performance-optimization)
12. [Versioning](#12-versioning)
13. [Errors & API Conventions](#13-errors--api-conventions)
14. [Known Limitations](#14-known-limitations)

---

## 1. Tenant Isolation & Workspaces

**Source pages:** `admin/overview`, `admin/set-up-organization/create-organization`, `admin/set-up-organization/enterprise-accounts`, `admin/workspaces/your-first-workspace`, `admin/admin-api/manage-workspaces`

### Main concepts

- **Organization**: the top-level account. Created automatically at signup. Holds members, billing, subscriptions, audit logs, and privacy controls. Each Organization has a unique Organization ID. An Organization can be permanently deleted from the Danger Zone (deletes resources, data, Workspaces, API keys, usage history — but not user accounts).
- **Enterprise Account**: a grouping above one or more Organizations, used when stronger isolation is needed between groups of users than Workspaces provide. Managed via the **Backoffice** (`backoffice.mistral.ai`). Enterprise Account admins can be promoted to Organization admin on linked Organizations. Admin API keys are created at the Enterprise Account level but always scoped to a single Organization.
- **Workspace**: a shared environment inside an Organization that scopes resources, access, usage, and limits. Used to model teams, products, environments, budgets, or business units. An Organization can have up to **500 active Workspaces**. Workspace names must be unique within an Organization. Every Organization starts with a default Workspace (which cannot be archived). Organization admins always have full access to every Workspace. Archiving a Workspace makes it unavailable to all members and cannot be undone.
- **Backoffice vs Admin Panel**:
  - `admin.mistral.ai` (Admin Panel) administers a single Organization.
  - `backoffice.mistral.ai` (Backoffice) manages an Enterprise Account and the Organizations under it (members, linked Organizations, Admin API keys).
- **User Group**: bundles users so they can be managed together — created, members added, then assigned to a Workspace with a given role. Provisioning a group to a Workspace grants every member the specified Workspace role.
- **RBAC model**: roles are predefined and apply at two scopes — the Organization (org-wide) and each Workspace (resource-scoped). Organization roles and Workspace roles are independent (e.g. an Organization Member can be a Workspace Admin in a specific Workspace).

### API functions (Admin API — Workspaces)

| Function | Method & Endpoint | Purpose |
| --- | --- | --- |
| List Workspaces | `GET /api/admin/workspaces` | Enumerate workspaces; `is_archived` boolean filters archived |
| Create Workspace | `POST /api/admin/workspaces` | Provision a new workspace |
| Update Workspace | `PATCH /api/admin/workspaces/{workspace_uuid}` | Modify name, description, icon |
| Delete Workspace | `DELETE /api/admin/workspaces/{workspace_uuid}` | Remove a workspace |
| Add members | `POST /api/admin/workspaces/{workspace_uuid}/add-users` | Add members with roles |
| Add or update members (idempotent) | `PATCH /api/admin/workspaces/{workspace_uuid}/users` | Upsert members and roles |
| Remove users | `DELETE /api/admin/workspaces/{workspace_uuid}/users` | Remove members |
| List roles | `GET /api/admin/roles` | Enumerate available roles and UUIDs (RBAC) |

### Key parameters

- `is_archived` (boolean) — include archived workspaces in list
- `name`, `description`, `icon` — workspace attributes on create/update
- `members[]` — array of `{user_uuid, role_names}` for add/update operations
- `role_names` (array) — preferred plural form over deprecated singular `role`/`role_name`
- `workspace_role` (UUID) or `workspace_role_name` (name) — role for group-to-workspace provisioning

### Workspace response fields

`uuid`, `name`, `description`, `icon`, `is_default` (boolean), `members_count` (integer), `spend_limit` (WorkspaceSpendLimit object).

---

## 2. Authentication & Security

**Source pages:** `admin/identity-access/api-keys`, `admin/identity-access/user-management`, `admin/identity-access/roles-permissions`, `admin/admin-api/authentication`, `admin/set-up-organization/verify-domain`, `resources/security-advisories`

### Main concepts

- **Two key classes** (separate keys, separate privileges):
  1. **Standard API key** — authenticates inference requests to `api.mistral.ai/v1/...` via `Authorization: Bearer <key>`. Scoped to the Workspace where created; never grants admin access.
  2. **Admin API key** — authenticates Admin API requests via `x-api-key` header. Created in the Backoffice, scoped to a single Organization, can have an expiration date or none.
- **API key types** (product surfaces):
  - `Studio` — standard keys for Mistral API usage
  - `Vibe` — keys used by Vibe Code
  - `Mistral Code` — legacy keys
- **Key scope**: API keys are scoped to the Workspace where they were created. Requests use that Workspace's quota, rate limits, and resources. To separate environments (dev/staging/prod), create separate Workspaces with dedicated keys.
- **Connector access scope**: when creating an API key, choose whether it can call only Workspace-shared Connectors or both private and Workspace-shared Connectors.
- **Key immutability**: after creation you cannot change a key's Workspace, Connector access scope, or expiration date — create a new key and delete the old one. The full key is shown only once.
- **Domain verification**: required before enabling email domain authentication and SAML SSO. Proves control of an email domain via a DNS TXT record (`mistral-domain-verification=xxxxxx`). Keep verified DNS TXT records active or enterprise access features break.
- **SAML SSO**: Organization-level single sign-on through a configured identity provider. Users join via the IdP. Admins can enable automatic seat assignment.
- **SCIM**: System for Cross-domain Identity Management support planned for automating onboarding/offboarding (not yet GA).
- **Security advisories**: published for compromised SDK package versions (e.g. malicious code injected into `mistralai` PyPI/NPM packages). Remediation: stop using affected versions, clean systems, rotate all secrets, check cloud audit logs, monitor listed C2 indicators.

### API functions (Admin API — API Keys)

| Function | Method & Endpoint | Purpose |
| --- | --- | --- |
| Get API Keys | `GET /api/admin/api-keys` | List workspace API keys |
| Create API Key | `POST /api/admin/api-keys` | Create a Workspace API key |
| Delete API Key | `DELETE /api/admin/api-keys/{key_id}` | Revoke a key |

### Key parameters (create API key)

- `expiration` (date|null) — expiration date for the key
- `name` (string|null) — optional name
- `user_id` (string) — ID of the user the key is created for
- `workspace_uuid` (string) — Workspace ID for the key

### API key response fields

`key_id`, `hidden_key`, `name`, `product` (e.g. `"API"`), `created_at`, `created_by`, `expiration_date`, `last_used`, `workspace_id`, `workspace_name`, `can_delete`, `actions` (per-action availability map).

### Roles

**Organization roles** (assigned via `role_names`):

| UI label | API role name | Grants |
| --- | --- | --- |
| Member | `member` | Product usage; manages own profile/preferences |
| Billing | `billing_manager` | Subscriptions, invoices, payment methods, usage reports; no org settings/invites |
| Admin | `organization_admin` | Full control: settings, members, security, billing, audit logs |

**Workspace roles**:

| UI label | API role name | Grants |
| --- | --- | --- |
| User | `user` | Access to Vibe features; no Studio access |
| Developer | `dev` | Access to Studio and primitives (agents, fine-tuning); no Vibe access |
| Mistral Vibe Code User | `mistral_code_user` | Access to Mistral Code (requires seat) |
| Workspace Contributor | `workspace_contributor` | All product features combined; no management/admin/observability |
| Admin | `workspace_admin` | Everything a Contributor has, plus Workspace administration |
| Observability Viewer | `observability_viewer` | Access to the Observability suite |

### User management endpoints

| Function | Method & Endpoint | Purpose |
| --- | --- | --- |
| Get Users | `GET /api/admin/users` | List members |
| Create Users | `POST /api/admin/users` | Bulk create users |
| Get User | `GET /api/admin/users/{user_id}` | Retrieve a member |
| Update User | `PATCH /api/admin/users/{user_id}` | Update roles/subscriptions |
| Delete User | `DELETE /api/admin/users/{user_id}` | Remove a member |
| Invite Users | `POST /api/admin/users/invite` | Send invitations |
| Get Invite | `GET /api/admin/users/invite/{invite_id}` | Retrieve an invitation |
| Delete Invite | `DELETE /api/admin/users/invite/{invite_id}` | Cancel invitation |
| Get Roles | `GET /api/admin/roles` | List roles and UUIDs |

---

## 3. Cost Optimization & Billing

**Source pages:** `admin/billing-usage/billing`, `admin/billing-usage/usage-limits`, `admin/admin-api/usage-metrics`, `api/endpoint/beta/admin/billing`

### Main concepts

- **Billing**: manages payment methods, billing information, credits, monthly spending limits, current usage, and invoices. Requires the Admin or Billing Organization role.
- **Payment methods**: add/update payment method per Organization; if none configured, Billing page indicates this.
- **Billing information**: billing profile appearing on future invoices.
- **Credits**: review credit balance; redeem gift codes.
- **Monthly spending limit (Organization)**: caps API and Vibe consumption at the Organization level, in the billing currency. If reached, API access can be suspended until the next month or an admin increases the limit. Failed invoice payments can also affect access.
- **Workspace spending limits**: cap consumption for a single Workspace; cannot exceed the Organization spending limit. Used to model team/project/environment budgets.
- **Usage dashboard**: shows Organization usage for a selected billing period with breakdowns by day, model, capability, and Workspace.
- **Batch discount**: batch processing runs at a **50% discount** vs. synchronous inference.
- **Admin API billing**: programmatic access to spend limits, rate limits, and usage retrieval.

### API functions (Admin API — Billing)

| Function | Method & Endpoint | Purpose |
| --- | --- | --- |
| Get Rate Limits | `GET /api/admin/rate-limit` | Retrieve current rate limits |
| Get Spend Limits | `GET /api/admin/spend-limit` | Retrieve spend limits |
| Update Spend Limits | `POST /api/admin/spend-limit` | Set/update spend caps |
| Get Usage | `GET /api/admin/usage` | Retrieve billing usage for a period |

### Key parameters / response fields (billing)

- `requests_per_second` (integer) — maximum API requests per second
- `tokens_limits_by_model` (map) — per-model token limits:
  - `tokens_per_minute` (integer)
  - `tokens_per_month` (integer)

### Usage metrics (Admin API)

Two kinds of consumption data:
1. **Billing usage** — cost and consumption for the Organization over a billing period.
2. **Product analytics** — Le Chat activity and Vibe Code usage metrics over a time range.

**Vibe Code analytics** endpoints (`/analytics/vibe`) accept `start_time` and `end_time` as Unix timestamps (seconds), and an optional `workspace_id` to scope reports. Return daily series for: sessions (CLI, ACP, programmatic), active users, consumed tokens (input/output/cached, per model), tool calls, session durations, next-edit suggestions (by outcome), modified lines of code.

---

## 4. Subscription Management

**Source pages:** `admin/billing-usage/subscriptions`, `admin/billing-usage/billing`

### Main concepts

- **Subscriptions are separated** by product surface:
  - **Vibe plans** — control access to Vibe features and seats (Team seats tracked as used/available). Student/teacher discounts available.
  - **Mistral Code / Vibe Code** — Enterprise seats for the coding assistant (legacy `Mistral Code`; current direction is `Vibe Code` covering CLI, editor extensions, coding workflows).
  - **API Plan** — controls Studio and API usage, separate from the Vibe subscription.
- **API Plan modes**:
  - **Free mode** — create API keys and use the free tier within the limits shown on the Usage and limits page. Included with a Vibe subscription if present; available by default otherwise.
  - **Scale plan** — pay-as-you-go API plan giving access to paid usage, higher limits, frontier models, agents, fine-tuning, and preview/beta features.
- **Seat management**: seats assigned per member; can include Vibe, Mistral Code seats. With SAML SSO enabled, admins can turn on automatic seat assignment.

### Management surfaces

- Admin Panel › Subscriptions › Plans — review plans and seats
- Admin Panel › Subscriptions › Billing — payment methods, credits, spending limits, invoices
- Upgrade to Scale via the Scale upgrade page

---

## 5. Batch Operations

**Source pages:** `studio-api/batch-processing`, `api/endpoint/batch`, `api/endpoint/files`

### Main concepts

- **Batch processing** runs asynchronous inference on large inputs in parallel, reducing compute costs while running at a **50% discount**.
- **Batch structure**: a list of API requests, each with:
  - A unique `custom_id` for identifying the request and referencing results after completion
  - A `body` object with the raw request payload (same as calling the original endpoint directly)
- **Supported batch endpoints** (the `endpoint` field on job creation):
  `/v1/chat/completions`, `/v1/embeddings`, `/v1/fim/completions`, `/v1/moderations`, `/v1/chat/moderations`, `/v1/ocr`, `/v1/classifications`, `/v1/chat/classifications`, `/v1/conversations`, `/v1/audio/transcriptions`
- **File-based batch**: upload a JSONL file (via the Files API with `purpose: "batch"`), then create a batch job referencing the file IDs. Results are written to output/error files available for download.
- **Inline batch**: pass `inline_batch_data` array directly to the SDK `batch.jobs.create()` method.
- **Batch does not count against real-time rate limits** — batch jobs are processed asynchronously.
- **Batch limitations** (see Known Limitations): max batch file size 512 MB, max 100,000 requests per batch, results available for download 24 hours after completion.

### API functions

| Function | Method & Endpoint | Purpose |
| --- | --- | --- |
| Create Batch Job | `POST /v1/batch/jobs` | Queue a new batch job |
| Get Batch Job | `GET /v1/batch/jobs/{job_id}` | Retrieve job details; `inline=true` returns results inline |
| Upload file | `POST /v1/files` | Upload JSONL batch input file |
| List files | `GET /v1/files` | Enumerate uploaded files |
| Download file | `GET /v1/files/{file_id}/content` | Download batch results |

### Key parameters (create batch job)

- `endpoint` (string, required) — the target inference endpoint
- `input_files` (array<string>|null) — file IDs containing batch requests
- `model` (string|null) — model to use
- `agent_id` (string|null) — for batch inference with a specific (deprecated) agent
- `metadata` (map|null) — arbitrary metadata (e.g. `{"job_type": "testing"}`)
- `inline` (boolean|null, on GET) — return results inline in the response

### Batch job response fields

`id`, `object` (`"batch"`), `status`, `endpoint`, `model`, `created_at`, `started_at`, `completed_at`, `input_files` (array), `output_file` (string|null), `error_file` (string|null), `completed_requests`, `failed_requests`, `errors` (array<BatchError>), `outputs` (array|null), `metadata`.

---

## 6. Quotas & Limits

**Source pages:** `admin/billing-usage/usage-limits`, `resources/known-limitations`, `api/endpoint/beta/admin/billing`

### Main concepts

- **Three layers of limits**:
  1. **Organization spending limit** — monthly cap on API and Vibe consumption across the Organization (in billing currency).
  2. **Workspace spending limits** — per-Workspace caps, cannot exceed the Organization limit.
  3. **API rate limits** — enforced per Organization (not per key/workspace), varying by subscription tier and model.
- **Rate limit enforcement**: requests per second (RPS) and tokens per minute (TPM) are enforced independently. When exceeded, the API returns `429 Too Many Requests`.
- **Rate limit headers**: check `X-RateLimit-Remaining` to monitor usage before hitting the limit.
- **Batch processing** does not count against real-time rate limits.
- **Context window limits**: vary by model (128k–256k tokens); requests exceeding the context window return `400 Bad Request`.
- **Streaming timeout**: streaming connections time out after 10 minutes of inactivity; `stream_options.include_usage` must be explicitly set to receive token usage in stream events.

### Admin Panel pages for limits

| Task | Admin Panel page |
| --- | --- |
| Review usage and cost breakdowns | Admin Panel › API › Usage |
| Review API rate limits | Admin Panel › API › Limits |
| Set Organization spending limit | Admin Panel › Subscriptions › Billing |
| Set Workspace spending limit | Admin Panel › Administration › Workspaces |

### Rate limit response fields (Admin API)

- `requests_per_second` (integer)
- `tokens_limits_by_model` (map):
  - `tokens_per_minute` (integer)
  - `tokens_per_month` (integer)

---

## 7. Logs, Audit & Data Retention

**Source pages:** `admin/admin-api/overview`, `api/endpoint/beta/admin/audit-logs`, `admin/monitor-comply/privacy-data-controls`, `api/endpoint/events`, `api/endpoint/beta/observability/logs`, `api/endpoint/beta/observability/traces`, `resources/known-limitations`

### Main concepts

- **Audit logs**: Organization-level audit log entries retrievable via the Admin API. Provide a record of administrative actions.
- **Workflow events**: events emitted during workflow executions, listable with cursor-based pagination via `GET /v1/workflows/events` (or `/api/endpoint/events`).
- **Observability logs**: production chat completion events searchable via `POST /v1/observability/logs/search`, with field definitions retrievable via `GET /v1/observability/logs/fields` and per-field options via `GET /v1/observability/logs/fields/{field_name}/options`.
- **Observability traces**: OpenTelemetry traces with spans, searchable via `POST /v1/observability/traces/search`, retrievable by ID, with spans via `GET /v1/observability/traces/{trace_id}/spans`, and individual spans via `GET /v1/observability/spans/{span_id}`.
- **Data retention**:
  - Uploaded files retained for **30 days** unless deleted earlier.
  - Batch results available for download for **24 hours** after completion.
- **Privacy data controls** (see §10): training opt-out, data retention settings, GDPR rights (access, rectification, erasure, portability, object, restriction).

### API functions

| Function | Method & Endpoint | Purpose |
| --- | --- | --- |
| Get Audit Logs | `GET /api/admin/audit-logs` | List Organization audit log entries |
| List workflow events | `GET /v1/workflows/events` | List workflow execution events (cursor pagination) |
| Search logs | `POST /v1/observability/logs/search` | Search production chat completion events |
| Get log field definitions | `GET /v1/observability/logs/fields` | List searchable log fields |
| Get log field options | `GET /v1/observability/logs/fields/{field_name}/options` | Retrieve distinct values for a field |
| Search traces | `POST /v1/observability/traces/search` | Search OTel traces |
| Get trace by id | `GET /v1/observability/traces/{trace_id}` | Retrieve a specific trace |
| Get trace spans | `GET /v1/observability/traces/{trace_id}/spans` | List spans in a trace |
| Get trace field definitions | `GET /v1/observability/traces/fields` | List trace fields |
| Get trace field options | `GET /v1/observability/traces/fields/{field_name}/options` | Retrieve distinct values (`from`, `to` date-time params) |
| Get span by id | `GET /v1/observability/spans/{span_id}` | Retrieve a specific span |

### Key parameters

- Audit logs: requires Admin API key (`x-api-key` header)
- Log/traces search: `search_expression` (string|null), `order` (`"asc"` | `"desc"`, default `"desc"`)
- Trace field options: `field_name` (path), `from` / `to` (date-time|null) query params
- Workflow events: `next_cursor` (string|null) for pagination

---

## 8. Observability

**Source pages:** `studio-api/observability`, `studio-api/observability/quickstart`, `studio-api/observability/judges`, `studio-api/observability/campaigns`, `studio-api/observability/datasets`, `studio-api/workflows/observability`, `resources/observability-integrations`

### Main concepts

The Observability suite (Enterprise-tier only) provides three core capabilities:
- **Visibility**: see production traffic event by event (Explorer).
- **Quality signals**: automatically score/classify responses with LLM-powered **Judges**.
- **Iteration loops**: use **Campaigns** to annotate traffic at scale and build quality-tagged **Datasets**.

**Four components:**
1. **Explorer** — search, filter, inspect every chat completion event (messages, tool calls, metadata); export filtered slices to Datasets.
2. **Judges** — automated scoring criteria defined by instructions + model + output schema (classification or regression). Versioned (base/up/down revisions). Can be run live on a conversation via `live-judging`.
3. **Campaigns** — batch annotations on live production traffic using a Judge + search filters. Run asynchronously in the background. Max events per campaign: 100–10,000.
4. **Datasets** — curated collections of conversation records (JSONL format with `messages` + `properties`), importable from Explorer or Campaigns. Used for evaluation and fine-tuning.

**Workflow observability** (OpenTelemetry):
- Activities automatically generate spans; use the `name` parameter to make them readable.
- Trace sampling configurable.
- SDK helpers: `get_workflow_execution_trace_otel()`, `get_workflow_execution_trace_summary()`, `get_workflow_execution_trace_events(include_internal_events=False)`.

**Observability metrics** (integrations):
- Token and cost metrics at the individual call level (input prompt, model, output).
- Application-level workflow patterns: chaining, flow engineering, multi-agent systems.
- Service Level Objectives (SLOs), alerts, and monitoring can be set across steps.

### API functions — Judges

| Function | Method & Endpoint | Purpose |
| --- | --- | --- |
| Create judge | `POST /v1/observability/judges` | Define a scoring/classification judge |
| Get judge | `GET /v1/observability/judges/{judge_id}` | Retrieve a judge (includes revision fields) |
| Update judge | `PUT /v1/observability/judges/{judge_id}` | Modify instructions, model, output |
| Delete judge | `DELETE /v1/observability/judges/{judge_id}` | Remove a judge |
| Live judging | `POST /v1/observability/judges/{judge_id}/live-judging` | Run a saved judge on a conversation |

### Judge parameters

- `name`, `description`, `instructions` (prompt template with `{{ conversation_history }}`, `{{ user_message }}`, `{{ assistant_message }}`)
- `model_name` (e.g. `"mistral-medium-latest"`)
- `output` — scoring schema:
  - Classification: `{"type": "CLASSIFICATION", "class_descriptions": [...]}`
  - Regression: `{"type": "REGRESSION", "min": 1, "max": 5, "min_description": ..., "max_description": ...}`
- `tools` (array) — tools available to the judge
- Judge response fields: `id`, `name`, `description`, `instructions`, `model_name`, `output`, `tools`, `workspace_id`, `owner_id`, `base_revision`, `up_revision`, `down_revision`, `created_at`, `updated_at`, `deleted_at`

### API functions — Campaigns

| Function | Method & Endpoint | Purpose |
| --- | --- | --- |
| List campaigns | `GET /v1/observability/campaigns` | Enumerate campaigns (`page`, `page_size`, `q`) |
| Create campaign | `POST /v1/observability/campaigns` | Define filters + attach a judge |
| Get campaign status | `GET /v1/observability/campaigns/{campaign_id}/status` | Check progress |
| List selected events | `GET /v1/observability/campaigns/{campaign_id}/selected-events` | Get annotated event IDs (`page`, `page_size`) |
| Delete campaign | `DELETE /v1/observability/campaigns/{campaign_id}` | Remove a campaign |

### Campaign parameters

- `name`, `description`
- `judge_id` — the judge to apply
- `search_params` — filters object with `AND`/`OR` conditions:
  - `field` (e.g. `timestamp`, `model_name`)
  - `op` (e.g. `gte`, `lt`, `eq`)
  - `value`
- `max_nb_events` (integer, 100–10,000)

### API functions — Datasets

Datasets support create, import records (from Explorer or Campaigns via `datasets.import_from_explorer()`), list, and manage. Records are JSONL with `messages` (array) and `properties` (map, e.g. `expected_output`, `category`).

---

## 9. Orchestration & Integrations (Connectors)

**Source pages:** `studio-api/connectors`, `studio-api/connectors/management`, `studio-api/connectors/tool_calling`, `studio-api/connectors/confirmation`, `studio-api/connectors/debugger`, `api/endpoint/beta/connectors`

### Main concepts

- **Connectors** are registered [MCP (Model Context Protocol)](https://modelcontextprotocol.io/) servers usable as tools in conversations and Agents via the API, **without managing MCP transport locally**. Public Preview — interface can change.
- Once registered, a Connector exposes its tools to the model on demand; the model discovers and calls the right tool based on the user's request.
- **Visibility scopes**: Workspace-shared (available to all Workspace members) or private (user-only). Connector names are unique within a Workspace (up to 64 chars, alphanumeric + underscores/dashes).
- **Connector access scope on API keys**: when creating a key, choose Workspace-shared-only or private + Workspace-shared.
- **Credentials management**: credentials stored at user level or Workspace level per connector; managed via dedicated endpoints (set/delete).
- **Human-in-the-loop**: intercept tool calls for user approval before execution. Multiple pending tool calls can be approved/denied individually or in a batch. SDK raises `DeferredToolCallsException` (requires `mistralai` v2.4+ with the `mcp` extra).
- **Connectors Debugger**: validate a Connector server from Studio before production. Supports custom header auth (`Authorization: Bearer <token>`) or OAuth 2.0 (Client ID + Client Secret). Credentials not stored (session only). Diagnostic output includes step-by-step transport detection, request/response details, and error types (e.g. `not_mcp_server`).

### API functions

| Function | Method & Endpoint | Purpose |
| --- | --- | --- |
| List Connectors | `GET /v1/connectors` | List with cursor pagination (`next_cursor`); filter via `query_filters` (e.g. `active: true`) |
| Create Connector | `POST /v1/connectors` | Register an MCP server |
| Get Connector | `GET /v1/connectors/{connector_id_or_name}` | Retrieve by ID or name |
| Update Connector | `PATCH /v1/connectors/{connector_id_or_name}` | Modify settings |
| Delete Connector | `DELETE /v1/connectors/{connector_id_or_name}` | Remove a connector |
| Call tool directly | `POST /v1/connectors/{connector_id_or_name}/tools/{tool_name}/call` | Invoke a specific MCP tool without a conversation |
| List tools | `GET /v1/connectors/{connector_id_or_name}/tools` | Discover available tools |
| Set user credentials | `POST /v1/connectors/{connector_id_or_name}/user/credentials/{credentials_name}` | Store per-user credentials |
| Delete user credentials | `DELETE /v1/connectors/{connector_id_or_name}/user/credentials/{credentials_name}` | Remove per-user credentials |
| Set workspace credentials | `POST /v1/connectors/{connector_id_or_name}/workspace/credentials/{credentials_name}` | Store per-workspace credentials |
| Delete workspace credentials | `DELETE /v1/connectors/{connector_id_or_name}/workspace/credentials/{credentials_name}` | Remove per-workspace credentials |

### Key parameters

- `name` (string, ≤64 chars, alphanumeric + `_`/`-`)
- `connector_id_or_name` (path) — reference by UUID or unique name
- `query_filters` (list) — e.g. `{"active": true}`
- `next_cursor` (string) — pagination cursor
- `tool_confirmations[]` (human-in-the-loop) — array of `{tool_call_id, confirmation: "approve"|"deny"}`

### Workflows (Orchestration API)

Workflows provide a programmatic orchestration layer with:
- **Activities** (decorated functions generating OTel spans automatically, customizable via `name` parameter)
- **Deployments**, **Schedules**, **Executions**, **Events**, **Metrics**, **Runs** endpoints under `/api/endpoint/workflows/*`
- Trace retrieval helpers for execution-level diagnostics

---

## 10. Privacy & Compliance

**Source pages:** `admin/monitor-comply/privacy-data-controls`

### Main concepts

- **Privacy and data controls** manage how data is handled across Vibe and the API. Three areas: data usage policies, training opt-out, and GDPR rights.
- **Vibe privacy controls** (Admin Panel › Vibe › Privacy):
  - Model training opt-out
  - Public chat sharing
  - User feedback
  - Chat retention
- **API privacy controls** (Admin Panel › API › Privacy):
  - Model training opt-out
  - Data retention settings
  - Labs model access
- **GDPR rights**:

| Right | Description |
| --- | --- |
| Access | Obtain copies of personal data |
| Rectification | Correct inaccurate personal information |
| Erasure | Delete account and associated data (permanent, irreversible) |
| Data portability | Request data in a transferable format |
| Object | Opt out of certain processing activities |
| Restriction | Request limitations on data processing |

- **Legal resources**: Terms of Service, Privacy Policies, Data Processing Agreements in the Legal Center (`legal.mistral.ai`) and Trust Center (`trust.mistral.ai`).
- **Data export**: download from Vibe or Studio. Account changes via Admin Panel.

---

## 11. Performance Optimization

**Source pages:** `studio-api/batch-processing`, `models/best-practices/sampling`, `resources/known-limitations`, `resources/error-glossary`

### Main concepts

- **Batch processing** for cost/performance optimization of large workloads: 50% discount, async parallel processing, no real-time rate limit consumption.
- **Sampling parameters** (`models/best-practices/sampling`): control generation behavior via `temperature`, `top_p`, `max_tokens`, etc.
- **Streaming**: for lower latency on long outputs; `stream_options.include_usage` must be set explicitly to receive token usage in stream events. Streaming connections time out after 10 minutes of inactivity. Ensure chunked transfer encoding is handled correctly by client HTTP libraries.
- **Token budgeting**: token counts include both input and output tokens; plan `max_tokens` accordingly to avoid `400 Bad Request` when exceeding the context window.
- **Retry with exponential backoff** (from error glossary): recommended for transient errors (429, 5xx), e.g. `wait = (2 ** attempt) + random.uniform(0, 1)`.
- **Trace sampling**: configurable OpenTelemetry trace sampling for workflow observability to balance detail vs. overhead.
- **Cached tokens**: Vibe Code analytics report consumed tokens split into input, output, and **cached** (per model) — cached tokens offer cost/latency savings.

---

## 12. Versioning

**Source pages:** `resources/changelogs`, `resources/migration-guides`, `resources/deprecated/finetuning`, `resources/deprecated/customization`, `api/endpoint/deprecated/agents`, `api/endpoint/deprecated/fine-tuning`

### Main concepts

- **Changelog** (`resources/changelogs`): chronological log of API updates, model releases, and other changes (dated entries). Tracks endpoint renames, new capabilities (JSON mode, function calling), payment system changes, and admin management features.
- **Model versioning**: models use dated suffixes (e.g. `mistral-small-2402`, `mistral-large-2407`) and `-latest` aliases. Old aliases are deprecated with a three-month sunset window before removal.
- **API versioning / SDK major versions**: SDKs use semantic versioning. The `mistralai` Python/JS SDK has a **V1 → V2 migration**:
  - V1: separate packages (`mistralai-azure`, etc.) with distinct client classes.
  - V2: unified `from mistralai.client import Mistral` with provider-specific sub-modules (`mistralai.azure.client.MistralAzure`). Azure uses `server_url` + `api_key` instead of `azure_endpoint` + `azure_api_key`.
- **Endpoint deprecation lifecycle**: deprecated endpoints grouped under `/api/endpoint/deprecated/*` (e.g. `deprecated/agents`, `deprecated/fine-tuning`). Newer equivalents live under `/api/endpoint/beta/*` (e.g. `beta/agents`, `beta/conversations`, `beta/observability/*`).
- **Beta endpoints**: Public Preview features under `/api/endpoint/beta/*` — interface can change.
- **Judge versioning**: judges carry `base_revision`, `up_revision`, `down_revision` fields for revision tracking.

### Migration examples

**Azure AI client (V1 → V2)**:

```python
# V1
# pip install mistralai-azure>=1.0.0
from mistralai_azure import MistralAzure
client = MistralAzure(
    azure_endpoint=os.environ["AZUREAI_ENDPOINT"],
    azure_api_key=os.environ["AZUREAI_API_KEY"],
)

# V2
# pip install mistralai>=2.0.0
from mistralai.azure.client import MistralAzure
client = MistralAzure(
    server_url=os.environ["AZUREAI_ENDPOINT"],
    api_key=os.environ["AZUREAI_API_KEY"],
)
```

**Standard client (V1 → V2)**:

```python
# V1
from mistralai.client import Mistral  # (old location)

# V2
from mistralai.client import Mistral
```

---

## 13. Errors & API Conventions

**Source pages:** `resources/error-glossary`, `resources/known-limitations`

### Error structure

All errors return a JSON body with a consistent structure (message, type, code/params as applicable).

### HTTP status codes

**Client errors (4xx):**
- `400 Bad Request` — malformed request, context window exceeded
- `401 Unauthorized` — missing/invalid API key
- `403 Forbidden` — insufficient permissions
- `404 Not Found` — resource doesn't exist
- `422 Unprocessable Entity` — validation error
- `429 Too Many Requests` — rate limit exceeded (RPS or TPM independently)

**Server errors (5xx):**
- `500 Internal Server Error`
- `502 Bad Gateway`
- `503 Service Unavailable`
- `504 Gateway Timeout`

### Retry guidance

Use exponential backoff with jitter for transient errors (429, 5xx):

```python
def call_with_retry(func, max_retries=5):
    for attempt in range(max_retries):
        try:
            return func()
        except Exception as e:
            if attempt == max_retries - 1:
                raise
            wait = (2 ** attempt) + random.uniform(0, 1)
            time.sleep(wait)
```

### API conventions

- **Inference API auth**: `Authorization: Bearer <key>` header
- **Admin API auth**: `x-api-key: <admin_key>` header
- **Pagination**: cursor-based (`next_cursor`) for Connectors/events; page-based (`page`, `page_size`) for admin lists
- **Role fields**: prefer plural `roles`/`role_names` over deprecated singular `role`/`role_name`
- **Response headers**: `X-RateLimit-Remaining` for rate limit monitoring

---

## 14. Known Limitations

**Source page:** `resources/known-limitations`

### Context window

| Model | Max context length |
| --- | --- |
| Mistral Large 3 | 256k tokens |
| Mistral Medium 3.5 | 256k tokens |
| Mistral Medium 3.1 | 128k tokens |
| Mistral Small 4 | 256k tokens |
| Codestral | 128k tokens |
| Devstral 2 | 256k tokens |
| Magistral Medium 1.2 | 128k tokens |
| Ministral 3 (3B / 8B / 14B) | 256k tokens |
| Mistral Nemo 12B | 128k tokens |

### Rate limits

- Vary by subscription tier and model; enforced per Organization.
- Requests per second and tokens per minute enforced independently.
- `429 Too Many Requests` when exceeded.
- Batch processing does not count against real-time rate limits.
- Check `X-RateLimit-Remaining` header.

### File uploads

- Maximum file size: **512 MB**
- Supported OCR formats: PDF, PNG, JPG, JPEG, TIFF, BMP, GIF, WEBP
- Uploaded files retained for **30 days** unless deleted earlier.

### Batch processing

- Maximum batch file size: **512 MB**
- Maximum requests per batch: **100,000**
- Batch jobs processed asynchronously; completion time depends on queue depth and complexity.
- Batch results available for download for **24 hours** after completion.

### Streaming

- Streaming connections time out after **10 minutes** of inactivity.
- `stream_options.include_usage` must be explicitly set to receive token usage in stream events.
- Some client HTTP libraries may buffer streamed responses; ensure chunked transfer encoding is handled correctly.

### Workspaces

- Maximum **500 active Workspaces** per Organization.
- Workspace names must be unique within an Organization.
- Default Workspace cannot be archived.

### Admin API

- Enterprise-plan only, in Preview; endpoints and fields can change.
- Admin API keys are scoped to a single Organization (created in the Backoffice).
- Standard API keys never grant admin access.

### Observability suite

- Available to **Enterprise-tier organizations** only.

---

## Appendix: API Surface Summary

### Inference / Studio API (`api.mistral.ai/v1/...`)

Auth: `Authorization: Bearer <standard_api_key>`

| Concern | Endpoint group | Notable paths |
| --- | --- | --- |
| Batch operations | `/v1/batch/jobs` | create, get (inline results) |
| Files | `/v1/files` | upload, list, download (30-day retention) |
| Observability (logs) | `/v1/observability/logs` | search, fields, field options |
| Observability (traces) | `/v1/observability/traces`, `/v1/observability/spans` | search, get, spans, field options |
| Observability (judges) | `/v1/observability/judges` | create, get, update, delete, live-judging |
| Observability (campaigns) | `/v1/observability/campaigns` | list, create, status, selected-events, delete |
| Observability (datasets) | `/v1/observability/datasets` | create, import, manage records |
| Connectors | `/v1/connectors` | list, create, get, update, delete, tools, call_tool, credentials |
| Workflow events | `/v1/workflows/events` | list with cursor pagination |
| Workflow executions | `/v1/workflows/executions` | trace retrieval (OTel, summary, events) |

### Admin API (`api.mistral.ai/...`, `/api/admin/...`)

Auth: `x-api-key: <admin_api_key>` (Enterprise only, Preview)

| Concern | Endpoint group | Notable paths |
| --- | --- | --- |
| Users | `/api/admin/users` | list, create (bulk), get, update, delete, invite, roles |
| Workspaces | `/api/admin/workspaces` | list, create, update, delete, add-users, remove-users |
| User groups | `/api/admin/user-groups` | list, create, assign-to-workspace, manage assignments |
| API keys | `/api/admin/api-keys` | list, create, delete |
| Billing | `/api/admin/rate-limit`, `/api/admin/spend-limit`, `/api/admin/usage` | get/update spend & rate limits, usage |
| Audit logs | `/api/admin/audit-logs` | list Organization audit entries |
| Analytics | `/analytics/vibe` | Vibe Code metrics (start_time, end_time, workspace_id) |
| Roles | `/api/admin/roles` | list roles and UUIDs |