# Unified AI Platform Administration API — Aggregated Specification

> **Purpose.** This document aggregates the cross-cutting / administrative capabilities described by four individual studies of the most popular AI platforms — **OpenAI**, **Anthropic Claude**, **Google Gemini**, and **Mistral AI** — into a single, exhaustive specification of a "super complete" AI platform API. It is written for the **end user / API consumer**: an introduction explains every concept in approachable terms, then a processing-pipeline specification lists every capability, where in the request lifecycle it applies, the alternative approaches that exist for the same step, and the different names the four systems use for the same concept.
>
> **Sources aggregated.**
> - `openai-api.md` — OpenAI API cross-cutting concerns
> - `anthropic-api.md` — Anthropic Claude API cross-cutting concerns
> - `google-api.md` — Google Gemini API cross-cutting concerns
> - `mistral-api.md` — Mistral AI API cross-cutting concerns
>
> **Notation.** Throughout this spec, terminology aliases are written as `OpenAI: "project"` / `Anthropic: "workspace"` / `Google: "Cloud project"` / `Mistral: "workspace"` to make cross-vendor mapping explicit. Where a vendor does not expose a concept, it is marked `(n/a)`.

---

## Table of Contents

1. [Introduction & Core Concepts](#1-introduction--core-concepts)
2. [The End-to-End Processing Pipeline](#2-the-end-to-end-processing-pipeline)
3. [Pipeline Stage 0 — Account & Tenant Setup](#3-pipeline-stage-0--account--tenant-setup)
4. [Pipeline Stage 1 — Identity, Authentication & Key Management](#4-pipeline-stage-1--identity-authentication--key-management)
5. [Pipeline Stage 2 — Access Control, RBAC & Groups](#5-pipeline-stage-2--access-control-rbac--groups)
6. [Pipeline Stage 3 — Tenant Isolation, Data Residency & Encryption](#6-pipeline-stage-3--tenant-isolation-data-residency--encryption)
7. [Pipeline Stage 4 — Network Security & Connectivity](#7-pipeline-stage-4--network-security--connectivity)
8. [Pipeline Stage 5 — Cost, Subscription, Spend & Billing Management](#8-pipeline-stage-5--cost-subscription-spend--billing-management)
9. [Pipeline Stage 6 — Quotas, Rate Limits & Usage Tiers](#9-pipeline-stage-6--quotas-rate-limits--usage-tiers)
10. [Pipeline Stage 7 — Request Preparation: Model Selection & Lifecycle](#10-pipeline-stage-7--request-preparation-model-selection--lifecycle)
11. [Pipeline Stage 8 — Processing Tiers & Cost Optimization](#11-pipeline-stage-8--processing-tiers--cost-optimization)
12. [Pipeline Stage 9 — Prompt Caching & Context Management](#12-pipeline-stage-9--prompt-caching--context-management)
13. [Pipeline Stage 10 — Safety, Guardrails, Moderation & Approvals](#13-pipeline-stage-10--safety-guardrails-moderation--approvals)
14. [Pipeline Stage 11 — Orchestration: Agents, Tools, MCP, Webhooks, Sandboxes](#14-pipeline-stage-11--orchestration-agents-tools-mcp-webhooks-sandboxes)
15. [Pipeline Stage 12 — Execution Modes: Sync, Streaming, Background, WebSocket, Batch](#15-pipeline-stage-12--execution-modes-sync-streaming-background-websocket-batch)
16. [Pipeline Stage 13 — Files, Storage & Data Lifecycle](#16-pipeline-stage-13--files-storage--data-lifecycle)
17. [Pipeline Stage 14 — Logging, Audit, Tracing & Observability](#17-pipeline-stage-14--logging-audit-tracing--observability)
18. [Pipeline Stage 15 — Compliance, Privacy, Data Retention & Legal](#18-pipeline-stage-15--compliance-privacy-data-retention--legal)
19. [Pipeline Stage 16 — Versioning, Deprecation & Changelog](#19-pipeline-stage-16--versioning-deprecation--changelog)
20. [Pipeline Stage 17 — Errors, Conventions & Client SDKs](#20-pipeline-stage-17--errors-conventions--client-sdks)
21. [Cross-Vendor Terminology Alias Reference](#21-cross-vendor-terminology-alias-reference)
22. [Capability Coverage Matrix](#22-capability-coverage-matrix)

---

## 1. Introduction & Core Concepts

Modern AI platforms (OpenAI, Anthropic, Google Gemini, Mistral) all expose an HTTP API that lets your application send prompts to a generative model and receive generated text, images, audio, or structured data. Around that core inference loop, every platform also exposes a large set of **cross-cutting / administrative** capabilities that govern *who* can call the API, *how much* it costs, *where* the data lives, *how fast* the answer comes back, *what* is logged, and *how* the system evolves over time.

This specification treats the entire lifetime of an API request — from setting up an account to retiring a deprecated model — as a single **processing pipeline** with ordered stages. Each stage is a place where the platform lets you configure behavior. The same conceptual step often has different names in different systems, and several stages offer **alternative approaches** you can choose between (for example, static API keys *or* short-lived federated tokens; synchronous streaming *or* asynchronous background jobs). The spec calls out both the aliases and the alternatives explicitly.

### 1.1 The core objects you will encounter

| Concept | What it is | Aliases across vendors |
| --- | --- | --- |
| **Account / Tenant** | The top-level container that owns billing, identity, and resources. | OpenAI: `organization`; Anthropic: `organization` (parent + linked orgs); Google: `Google Cloud organization` / `billing account`; Mistral: `organization` (under an `Enterprise Account`). |
| **Workspace / Project** | A sub-tenant boundary that isolates keys, files, rate limits, spend, and data-residency settings. | OpenAI: `project`; Anthropic: `workspace`; Google: `Cloud project`; Mistral: `workspace`. |
| **API key** | A secret string used to authenticate an API call. | All four: `API key` (prefixes differ: `sk-...`, `sk-ant-api03-...`, `GEMINI_API_KEY`, no prefix for Mistral). |
| **Admin key** | A separate key type that can only call administration endpoints. | OpenAI: `OPENAI_ADMIN_KEY`; Anthropic: `sk-ant-admin01-...` (Admin API key); Mistral: Admin API key (`x-api-key`); Google: (n/a — uses IAM service accounts). |
| **Service account** | A non-human identity used by workloads, often federated from an external IdP. | OpenAI: `service account` (within a project); Anthropic: `svac_...`; Google: `service account` (Cloud IAM); Mistral: (n/a). |
| **Model** | A named inference engine you send prompts to. | All four: `model`. Lifecycle suffixes differ (`-preview`, `-latest`, dated snapshots). |
| **Request** | A single API call (sync, streaming, background, or batched). | All four: `request` / `interaction` (Google) / `response` (OpenAI Responses API). |
| **Token** | The unit of consumption billed and counted (≈ a few characters). | All four: `token`; variants: input, output/cached, reasoning/thinking, accepted/rejected prediction. |

### 1.2 How a request flows through the pipeline

A request travels through every stage, in order:

```
Stage 0  Account/Tenant exists
Stage 1  Caller is authenticated (key or federated token)
Stage 2  Caller is authorized (RBAC / scopes / key permissions)
Stage 3  Request is routed to the right tenant + region + encryption envelope
Stage 4  Network path is established (public, Private Link, mTLS, tunnel)
Stage 5  Billing & spend policy is consulted
Stage 6  Quota / rate limit is checked
Stage 7  Model is selected (and must still be supported)
Stage 8  Processing tier is chosen (standard / flex / priority / batch)
Stage 9  Prompt cache is consulted / written
Stage 10 Safety, guardrails, moderation, approvals run
Stage 11 Orchestration layer (agents, tools, MCP, webhooks) is invoked
Stage 12 Execution mode runs (sync / streaming / background / WebSocket / batch)
Stage 13 Files & storage are read/written
Stage 14 Logs, audit, traces are emitted
Stage 15 Compliance & retention rules are applied to the stored data
Stage 16 Versioning/deprecation governs whether the model/endpoint still exists
Stage 17 Errors, conventions, SDKs shape the client experience
```

Each stage below is structured as: **Concept → Parameters → Endpoints → Alternative approaches → Vendor aliases**.

---

## 2. The End-to-End Processing Pipeline

The remaining sections walk through each pipeline stage in order. For every stage the spec lists:

- **Main concepts** — what the stage is about and why it exists.
- **Unified parameters** — the request/response fields that govern the stage, normalized to a single canonical name, with vendor aliases noted.
- **Endpoints** — representative REST endpoints from each vendor (where the stage is API-driven).
- **Alternative approaches** — when the same step can be done in more than one way.
- **Vendor aliases** — explicit mapping of equivalent names across the four systems.

---

## 3. Pipeline Stage 0 — Account & Tenant Setup

### 3.1 Main concepts

Before any API call can be made, an **account/tenant** must exist and contain at least one **workspace/project**. Some vendors layer a grouping *above* the standard organization for stronger isolation.

- **Organization** (`OpenAI`, `Anthropic`, `Mistral`) / **Cloud organization** (`Google`) — top-level billing & identity container.
- **Enterprise Account / Backoffice** (`Mistral`) / **parent organization + linked organizations** (`Anthropic`) — a grouping above one or more organizations, used when stronger isolation between groups of users is required than workspaces provide.
- **Workspace / Project** — sub-tenant boundary scoping keys, files, rate limits, spend, and data residency.
- **Default workspace** — every organization starts with one that cannot be archived (`Mistral`: default Workspace; `OpenAI`: default project).
- **Workspace limits** — `Mistral`: up to 500 active workspaces per organization; names unique within the org.

### 3.2 Unified parameters

| Parameter | Type | Description | Aliases |
| --- | --- | --- | --- |
| `organization_id` | string | Unique ID of the tenant | `OpenAI: org id`; `Anthropic: organization_id`; `Google: Cloud organization id`; `Mistral: organization uuid` |
| `workspace_id` / `project_id` | string | Unique ID of the sub-tenant | `OpenAI: project_id`; `Anthropic: workspace_id`; `Google: project_id`; `Mistral: workspace_uuid` |
| `name`, `description`, `icon` | string | Display attributes of the workspace | All four |
| `is_default` | boolean | Whether this is the non-archivable default workspace | `Mistral` |
| `is_archived` | boolean | Filter / archive state | `Mistral` |
| `members[]` | array | `{user_id, role_names}` to add/update members | `Mistral: members[]`; `OpenAI: invites`; `Anthropic: workspace members endpoints` |

### 3.3 Endpoints

| Function | OpenAI | Anthropic | Google | Mistral |
| --- | --- | --- | --- | --- |
| Create workspace/project | (dashboard) | `POST /v1/organizations/workspaces` | `Google Cloud project` (Cloud Console) | `POST /api/admin/workspaces` |
| List | (dashboard) | `GET /v1/organizations/workspaces` | (Cloud Console) | `GET /api/admin/workspaces` |
| Update | (dashboard) | `POST /v1/organizations/workspaces/{id}` | (Cloud Console) | `PATCH /api/admin/workspaces/{id}` |
| Archive/Delete | (dashboard) | (archive action) | (Cloud Console) | `DELETE /api/admin/workspaces/{id}` |
| Manage members | (invites) | Workspace Members endpoints | (Cloud IAM) | `POST /api/admin/workspaces/{id}/add-users`, `PATCH .../users`, `DELETE .../users` |

### 3.4 Alternative approaches

- **Grouping above the organization.** `Mistral` (Enterprise Account / Backoffice) and `Anthropic` (parent + linked organizations) provide an extra layer; `OpenAI` and `Google` do not (use separate orgs/projects instead).
- **Workspace creation surface.** `OpenAI` exposes it primarily via the dashboard (plus Admin API project management); `Anthropic`, `Mistral` expose dedicated Admin API endpoints; `Google` uses Cloud Console / Cloud Resource Manager.

### 3.5 Vendor aliases

`organization` = `organization` (OpenAI/Anthropic/Mistral) = `Cloud organization` (Google). `project` (OpenAI) = `workspace` (Anthropic/Mistral) = `Cloud project` (Google).

---

## 4. Pipeline Stage 1 — Identity, Authentication & Key Management

### 4.1 Main concepts

Every request must prove *who* is calling. Four authentication mechanisms coexist across the platforms:

1. **Static API key** — a long-lived secret sent in a header. Simplest; best for local dev / single-tenant servers.
2. **Admin API key** — a separate key type restricted to administration endpoints.
3. **OAuth 2.0 / user credentials** — user-level authorization via OAuth Client IDs (`Google`), or OAuth 2.1 (`OpenAI` for ChatGPT apps).
4. **Workload Identity Federation (WIF)** — short-lived bearer token exchanged from an IdP-issued OIDC JWT; eliminates long-lived keys in production workloads.
5. **Ephemeral tokens** — short-lived, scoped tokens for client-to-server (browser/mobile → API) use, primarily for realtime/live APIs (`Google` Live API; `OpenAI` Realtime API ephemeral secrets).

Supporting concepts:

- **Key expiration** — set `expires_at` to bound a leaked credential's lifetime (`Anthropic`, `Mistral`).
- **Key restriction** — restrict by request origin (IP, website referrer, application) (`Google`); restrict by API product (`Google`: Gemini only).
- **Safety identifiers** — privacy-preserving hashed user/session identifiers sent with requests to trace activity to end users (`OpenAI: safety_identifier`, `OpenAI-Safety-Identifier` header).
- **Domain verification** — prove control of an email domain via DNS TXT before enabling SSO (`Mistral`).
- **SAML SSO** — organization-level single sign-on through an IdP (`Mistral`).
- **SCIM** — sync group membership from an IdP (`OpenAI`; planned in `Mistral`).
- **Customer-Managed Encryption Keys (CMEK) / Enterprise Key Management (EKM)** — encrypt data at rest with keys you control in AWS KMS / GCP / Azure Key Vault (`Anthropic: CMEK`; `OpenAI: EKM`).
- **Access Transparency** — log of every access to your data by staff/systems (`Anthropic`).

### 4.2 Unified parameters & headers

| Parameter / Header | Type | Description | Aliases |
| --- | --- | --- | --- |
| `Authorization: Bearer <key>` | header | Standard bearer auth | `OpenAI`, `Mistral` (inference); `Anthropic` (WIF token) |
| `x-api-key: <key>` | header | Admin / direct key auth | `Anthropic` (admin/compliance keys), `Mistral` (admin) |
| `x-goog-api-key: <key>` | header | Google API key | `Google` |
| `anthropic-version: 2023-06-01` | header | Required API version header | `Anthropic` |
| `OpenAI-Organization` | header | Specify which org is billed | `OpenAI` |
| `OpenAI-Safety-Identifier` | header | Realtime API safety identifier | `OpenAI` |
| `safety_identifier` | body | Hashed user/session identifier | `OpenAI` |
| `expires_at` | RFC 3339 | Key expiration | `Anthropic`, `Mistral` |
| `x-goog-user-project` | header | Specify billing project for OAuth | `Google` |
| `anthropic-beta` / `betas` | header / SDK | Beta feature access | `Anthropic` |
| `managed-agents-2026-04-01` | header | Managed Agents beta | `Anthropic` |

### 4.3 Endpoints (keys & federation)

| Function | OpenAI | Anthropic | Google | Mistral |
| --- | --- | --- | --- | --- |
| Create API key | dashboard | `POST /v1/organizations/api_keys` | AI Studio / Cloud Console | `POST /api/admin/api-keys` |
| List keys | dashboard | `GET /v1/organizations/api_keys` | Cloud Console | `GET /api/admin/api-keys` |
| Delete/revoke key | dashboard | `DELETE /v1/organizations/api_keys/{id}` | Cloud Console | `DELETE /api/admin/api-keys/{id}` |
| Create service account | dashboard | `POST /v1/organizations/service_accounts` | Cloud IAM | (n/a) |
| Federation issuer / rule | dashboard + token exchange ref | `POST/GET /v1/organizations/federation_issuers`, `federation_rules` | Cloud Workload Identity | (n/a) |
| Token exchange | `/api/reference/workload-identity-federation` | OAuth 2.0 jwt-bearer grant | Cloud STS | (n/a) |
| Ephemeral token | (Realtime ephemeral secret) | (n/a) | `client.auth_tokens.create` (Live API) | (n/a) |
| Access Transparency query | (n/a) | `GET /v1/compliance/access_events` | (n/a) | (n/a) |

### 4.4 Alternative approaches

- **Static key vs. federated token.** Use static keys for dev; use WIF for production cloud workloads (`OpenAI`, `Anthropic`, `Google`). `Mistral` only supports static keys.
- **User auth vs. workload auth.** `Google` offers OAuth 2.0 (user) and service accounts (workload); others focus on key-based with optional WIF.
- **Symmetric vs. asymmetric webhook signing.** `Anthropic`, `Google` (static webhooks) use HMAC symmetric secrets; `Google` dynamic webhooks use asymmetric JWKS signatures.
- **CMEK vs. EKM.** `Anthropic` calls it CMEK; `OpenAI` calls it EKM; both support AWS KMS / GCP / Azure Key Vault. `Google` / `Mistral` rely on Cloud KMS implicitly.
- **Key immutability.** `Mistral` keys are immutable after creation (workspace, connector scope, expiration) — create a new key and delete the old to change anything. `Anthropic` scopes are immutable after creation (must rotate keys to change).

### 4.5 Vendor aliases

`API key` = `API key` (all). `Admin API key` = `OPENAI_ADMIN_KEY` (OpenAI) = `sk-ant-admin01-...` (Anthropic) = Admin API key / `x-api-key` (Mistral). `service account` = `service account` (OpenAI/Google) = `svac_...` (Anthropic). `workload identity federation` = `Workload Identity Federation` (OpenAI/Anthropic) = Cloud Workload Identity (Google). `ephemeral token` = `auth_tokens` (Google) ≈ Realtime ephemeral secret (OpenAI).

---

## 5. Pipeline Stage 2 — Access Control, RBAC & Groups

### 5.1 Main concepts

Once authenticated, the caller must be *authorized* to perform the action. All four platforms implement role-based access control (RBAC), but with different scopes and granularity.

- **Roles** — bundles of permissions, assigned at org and/or workspace scope.
- **Preset vs. custom roles** — `OpenAI` has preset (org owner, project owner, member, viewer) + custom roles; `Mistral` has predefined org + workspace roles; `Anthropic` uses scopes (Compliance key scopes, Admin key scopes); `Google` uses Cloud IAM roles.
- **Groups** — collections of users assignable to roles, syncable from an IdP via SCIM (`OpenAI: groups`; `Mistral: User Groups`).
- **Key permissions** — API keys carry their own permissions (e.g. `api.model.read`) that intersect with the user's role (`OpenAI`).
- **Union evaluation** — effective permissions = union of all org + project roles (direct and via groups) (`OpenAI`).
- **Propagation delay** — role/group changes take up to 30 minutes to propagate (`OpenAI`).

### 5.2 Permission catalogs

| Area | OpenAI levels | Anthropic scopes | Google IAM | Mistral roles |
| --- | --- | --- | --- | --- |
| Models / inference | Request | (model access per workspace) | `aiplatform.user` etc. | `user`, `dev`, `workspace_contributor` |
| Files / vector stores | Read, Write | (workspace-scoped) | (project-scoped) | (workspace-scoped) |
| Fine-tuning | Read, Write | (n/a — winding down) | (Vertex) | (workspace-scoped) |
| Admin / org management | Read, Write | `org:admin` OAuth | Owner/Editor/Viewer | `organization_admin`, `billing_manager`, `member` |
| Compliance | (audit logs) | `read:compliance_*`, `delete:compliance_user_data` | (Cloud IAM) | (audit logs) |
| Tunnels | Read, Use, Manage | (n/a) | (n/a) | (n/a) |
| Observability | (tracing) | (n/a) | (n/a) | `observability_viewer` |

### 5.3 Endpoints

| Function | OpenAI | Anthropic | Google | Mistral |
| --- | --- | --- | --- | --- |
| List roles | dashboard | (scopes on keys) | Cloud IAM | `GET /api/admin/roles` |
| Manage groups | dashboard + SCIM | (n/a) | Cloud IAM groups | `/api/admin/user-groups` |
| Invite users | `POST /v1/organization/invites` | (invites) | (Cloud IAM) | `POST /api/admin/users/invite` |
| Manage users | dashboard | (Admin API users) | Cloud IAM | `GET/POST/PATCH/DELETE /api/admin/users[/{id}]` |

### 5.4 Alternative approaches

- **SCIM sync vs. manual invites.** `OpenAI` supports SCIM group sync from IdP; `Mistral` plans SCIM; `Anthropic` centralizes identity via parent org + SSO.
- **Key-scoped permissions vs. user-scoped roles.** `OpenAI` keys carry permissions that intersect with user roles; `Anthropic` uses immutable scopes set at key creation; `Mistral` keys are workspace-scoped with connector-access scope; `Google` uses Cloud IAM roles on service accounts.
- **Preset vs. custom roles.** `OpenAI` allows custom roles; `Mistral` uses predefined roles only.

### 5.5 Vendor aliases

`role` = `role` (OpenAI/Mistral) = `scope` (Anthropic compliance/admin keys) = `IAM role` (Google). `group` = `group` (OpenAI) = `User Group` (Mistral). `invite` = `invite` (OpenAI/Mistral/Anthropic).

---

## 6. Pipeline Stage 3 — Tenant Isolation, Data Residency & Encryption

### 6.1 Main concepts

This stage controls *where* the request's data is stored and processed and *how* it is encrypted.

- **Data residency** — configure the geographic location of storage at rest and (where supported) processing. `OpenAI`: project-level residency with regional host prefixes (`eu.api.openai.com`, etc.); `Anthropic`: workspace geo (`default_inference_geo`, `allowed_inference_geos`) + per-request `inference_geo` (`"global"` | `"us"`); `Google`: Cloud project region (Developer API) / Vertex region (Enterprise); `Mistral`: (n/a — no dedicated residency page).
- **Regional storage vs. regional processing** — distinct concepts; some regions support only storage (`OpenAI`: Australia, Canada, Japan, etc.).
- **Zero Data Retention (ZDR)** — customer content not stored after the response returns (`OpenAI`: ZDR forces `store=false`; `Anthropic`: ZDR arrangement for eligible features; `Google`: paid tier content not used for training).
- **Modified Abuse Monitoring (MAM)** — excludes customer content from abuse monitoring logs while preserving capabilities (`OpenAI`).
- **Application State** — data persisted by some API features to fulfill a task (`OpenAI`).
- **Customer-Managed Encryption Keys (CMEK) / EKM** — encrypt at rest with your own KMS key (`Anthropic: CMEK`; `OpenAI: EKM`).
- **Access Transparency** — staff/system access logs (`Anthropic`).
- **`store` parameter** — boolean controlling whether response/application state is retained (`OpenAI: store`; `Google: store` on generateContent/Interactions; `Mistral`: not exposed as request param).

### 6.2 Unified parameters

| Parameter | Type | Description | Aliases |
| --- | --- | --- | --- |
| `data_residency` / region host prefix | string | Regional endpoint host | `OpenAI: eu.api.openai.com`; `Anthropic: inference_geo`; `Google: location` |
| `store` | boolean | Persist response/application state | `OpenAI: store`; `Google: store` |
| `expires_after` | object | Auto-delete files after a duration | `OpenAI` |
| `background` | boolean | Store response ~10 min for polling (ZDR-incompatible) | `OpenAI` |
| `external_web_access` | boolean | Offline/cache-only web search (BAA-eligible under ZDR) | `OpenAI` |
| `prompt_cache_retention` | enum | `in_memory` / `24h` cache policy | `OpenAI` |
| `inference_geo` | enum | `"global"` / `"us"` per-request inference location | `Anthropic` |
| `default_inference_geo` / `allowed_inference_geos` | enum / array | Workspace-level geo defaults & allowlist | `Anthropic` |
| `previous_interaction_id` | string | Chain stateful conversations across turns | `Google` |

### 6.3 Endpoints

| Function | OpenAI | Anthropic | Google | Mistral |
| --- | --- | --- | --- | --- |
| Set data residency | project config | workspace update (`default_inference_geo`) | Cloud project region | (n/a) |
| Set data retention policy | `PATCH /v1/organization/projects/{id}/data_retention` | workspace privacy controls | (n/a) | privacy controls in Admin Panel |
| ZDR eligibility per endpoint | documented table | feature-level eligibility | (n/a) | (n/a) |

### 6.4 Alternative approaches

- **Residency at project vs. workspace vs. request level.** `OpenAI`: project-level host prefix; `Anthropic`: workspace default + per-request override; `Google`: project/region at setup; `Mistral`: (n/a).
- **ZDR vs. MAM vs. default.** `OpenAI` offers a spectrum (ZDR ⊂ MAM ⊂ default); `Anthropic` offers ZDR arrangement + per-workspace 30-day retention override; `Google` ties data-use to plan tier (Free = used for improvement, Paid = not).
- **CMEK vs. EKM.** Different names, same idea: bring your own KMS key. `OpenAI` calls it EKM and lists eligible KMS providers; `Anthropic` calls it CMEK and requires a cross-account key policy.

### 6.5 Vendor aliases

`data residency` = `data residency` (OpenAI) = `workspace geo` / `inference geo` (Anthropic) = `region` (Google). `ZDR` = `Zero Data Retention` (OpenAI/Anthropic). `MAM` = `Modified Abuse Monitoring` (OpenAI). `CMEK` = `CMEK` (Anthropic) = `EKM` (OpenAI).

---

## 7. Pipeline Stage 4 — Network Security & Connectivity

### 7.1 Main concepts

Controls how traffic travels between your network and the AI platform.

- **Private Link** — reach regional endpoints through Azure Private Link instead of public endpoints (`OpenAI`; Azure-specific).
- **IP egress ranges / outbound IP allowlists** — published IP ranges for the platform's products so you can allowlist them (`OpenAI`).
- **mTLS (mutual TLS)** — authenticate the platform as an MCP client, or the tunnel control plane (`OpenAI`).
- **Secure MCP Tunnels** — outbound-only HTTPS connection from inside your network to a platform-hosted MCP endpoint; lets platform products reach a private/on-prem MCP server without public ingress (`OpenAI`).
- **Environment networking controls** — `unrestricted` (default, full outbound minus safety blocklist) vs `limited` (restrict to `allowed_hosts`) for managed agent sandboxes (`Anthropic`).

### 7.2 Unified parameters

| Parameter | Type | Description | Aliases |
| --- | --- | --- | --- |
| `base_url` | URL | Regional Private Link base URL | `OpenAI` |
| `--private-connection-resource-id` | string | OpenAI-provided Private Link Service resource ID (Azure CLI) | `OpenAI` |
| `--tunnel-id` | string | Tunnel identity | `OpenAI` |
| `--mcp-command` / `--mcp-server-url` | string / URL | Local stdio command or HTTP MCP server address (mutually exclusive) | `OpenAI` |
| `CONTROL_PLANE_API_KEY` | env | Runtime API key for tunnel-client | `OpenAI` |
| `networking.type` | enum | `unrestricted` / `limited` for sandbox envs | `Anthropic` |
| `networking.allowed_hosts` | array | Bare hostnames or wildcards | `Anthropic` |
| `networking.allow_mcp_servers` / `allow_package_managers` | boolean | Outbound to MCP / PyPI/npm | `Anthropic` |

### 7.3 Endpoints

| Function | OpenAI | Anthropic | Google | Mistral |
| --- | --- | --- | --- | --- |
| Regional health check | `GET /v2/privatelink_healthcheck` | (n/a) | (n/a) | (n/a) |
| API surface via Private Link | `https://<region>.privatelink.api.openai.com/v1/*` | (n/a) | (Private Service Connect) | (n/a) |
| IP egress JSON manifest | `https://openai.com/chatgpt-connectors.json` etc. | (n/a) | (n/a) | (n/a) |
| Tunnel control plane | `HTTPS https://api.openai.com:443/v1/tunnel/*` (or `mtls.api.openai.com:443`) | (n/a) | (n/a) | (n/a) |
| Tunnel RBAC permissions | Tunnels Read/Use/Manage | (n/a) | (n/a) | (n/a) |
| Tunnel audit events | `tunnel.created/updated/deleted` | (n/a) | (n/a) | (n/a) |

### 7.4 Alternative approaches

- **Public endpoint vs. Private Link.** `OpenAI` offers Azure Private Link (regional or legacy v1); `Google` offers Private Service Connect on Vertex; `Anthropic`/`Mistral` rely on public endpoints + CMEK/access transparency.
- **Inbound MCP server exposure.** `OpenAI` tunnel-client provides outbound-only tunneling (no public ingress needed); `Anthropic` uses cloud/self-hosted sandboxes with networking controls; `Mistral` uses registered Connectors (platform-managed MCP transport).
- **mTLS on control plane.** `OpenAI` tunnel control plane supports optional mTLS; `Anthropic`/`Mistral` use standard HTTPS.

### 7.5 Vendor aliases

`Private Link` = `Private Link` (OpenAI/Azure) = `Private Service Connect` (Google/Vertex). `tunnel` = `tunnel` (OpenAI). `environment networking` = `networking.type` (Anthropic).

---

## 8. Pipeline Stage 5 — Cost, Subscription, Spend & Billing Management

### 8.1 Main concepts

Governs *how much* you pay, *when* you pay, and *who* is budget-capped.

- **Usage & Cost APIs** — programmatic monitoring, analysis, reconciliation (`OpenAI`: usage dashboard/export; `Anthropic`: `GET /v1/organizations/usage_report`, `cost_report`; `Google`: billing/usage dashboards; `Mistral`: `GET /api/admin/usage`).
- **Spend limits** — monetary caps at org / workspace / user scope, with monthly reset (`OpenAI`: spend alerts; `Anthropic`: spend limits API with `(scope, period)` pairs, increase requests; `Google`: project spend caps + billing-account tier spend caps; `Mistral`: Organization + Workspace spending limits).
- **Billing plans** — Prepay (purchase credits in advance) vs Postpay (accrue, charged monthly) (`Google`); credits expire after 12 months; non-refundable except when switching to Postpay.
- **Subscriptions** — product-surface plans (`Mistral`: Vibe plans, Mistral Code / Vibe Code seats, API Plan Free/Scale; `OpenAI`: usage tiers; `Anthropic`: usage tiers + Priority Tier; `Google`: Free/Paid/Enterprise).
- **Priority Tier** — committed capacity tier with burndown rates (`Anthropic`, `OpenAI`).
- **Pricing model** — per-token (input/output), with feature multipliers (caching, batch, data residency); media models use per-image/per-second/per-song pricing; session runtime priced per hour (Managed Agents).
- **Monetary values** — strings in minor units (cents for USD) to avoid floating-point errors (`Anthropic`, `Mistral` spend limits).

### 8.2 Unified parameters

| Parameter | Type | Description | Aliases |
| --- | --- | --- | --- |
| `service_tier` | enum | Request-level processing tier (`"flex"`, `"priority"`, `"auto"`, `"default"`, `"standard_only"`) | `OpenAI`, `Anthropic`, `Google` |
| `threshold_amount` | int (cents) | Spend alert threshold | `OpenAI: spend_alerts`; `Anthropic: amount` (minor units) |
| `currency` | enum | e.g. `"USD"` | `OpenAI` |
| `interval` | enum | e.g. `"month"` | `OpenAI`, `Anthropic: period` |
| `notification_channel` | object | `{type: "email", recipients, subject_prefix}` | `OpenAI` |
| `retention_type` | enum | `"organization_default"` / `"zero_data_retention"` / `"modified_abuse_monitoring"` / `"none"` | `OpenAI` |
| `starting_at` / `ending_at` | RFC 3339 | Time bounds for usage/cost reports | `Anthropic`, `Google`, `Mistral` |
| `group_by[]` | array | Multi-dimensional grouping (workspace, model, etc.) | `Anthropic` |
| `period_to_date_spend` | string | Spend accrued in current period | `Anthropic` |
| `suppress_notification` | boolean | Suppress notify on approve | `Anthropic` |

### 8.3 Endpoints

| Function | OpenAI | Anthropic | Google | Mistral |
| --- | --- | --- | --- | --- |
| Usage report | usage dashboard | `GET /v1/organizations/usage_report` | billing/usage URLs | `GET /api/admin/usage` |
| Cost report | usage dashboard | `GET /v1/organizations/cost_report` | billing URLs | (usage) |
| Spend alert create | `POST /v1/organization/projects/{id}/spend_alerts` | (spend limits endpoints) | (project spend caps) | `POST /api/admin/spend-limit` |
| Spend limits list/effective | (n/a) | `GET /v1/organizations/spend_limits/effective` | (n/a) | `GET /api/admin/spend-limit` |
| Spend increase request approve/deny | (n/a) | `POST .../spend_limit_increase_requests/{id}/approve|deny` | (n/a) | (n/a) |
| Rate limits | `GET /v1/fine_tuning/model_limits` | `GET /v1/organizations/rate_limits`, `.../workspaces/{id}/rate_limits` | (rate-limit URL) | `GET /api/admin/rate-limit` |
| Analytics | (n/a) | `GET /v1/organizations/claude_code/analytics` | (n/a) | `/analytics/vibe` |

### 8.4 Alternative approaches

- **Prepay vs. Postpay billing.** `Google` offers both (Postpay available at Tier 3); others generally postpay.
- **Spend cap granularity.** `Anthropic` has the richest hierarchy (org → workspace → user, with increase-request workflow); `Google` has project + billing-account tiers; `Mistral` has org + workspace; `OpenAI` has project spend alerts + monthly usage limit.
- **Cost report inclusion of Priority Tier.** `Anthropic` cost report *excludes* Priority Tier (different billing model); others bundle.

### 8.5 Vendor aliases

`spend limit` = `spend alert` (OpenAI) = `spend limit` (Anthropic/Mistral) = `spend cap` (Google). `subscription` = (n/a OpenAI) = `Customer Type: api vs subscription` (Anthropic) = `plan tier` (Google) = `subscription` (Mistral). `usage tier` = `usage tier` (OpenAI/Anthropic/Google) = `subscription tier` (Mistral).

---

## 9. Pipeline Stage 6 — Quotas, Rate Limits & Usage Tiers

### 9.1 Main concepts

Limits on request volume within a timeframe to maintain fair usage and protect the system.

- **Rate limit dimensions** — RPM (requests/min), RPD (requests/day), TPM (tokens/min input), OTPM/OTPM (output tokens/min), TPD (tokens/day), IPM (images/min), audio minutes/min. Limit hit = whichever metric is reached first.
- **Scope** — per org / per project / per workspace / per model; `Google`: per project (not per key); `OpenAI`: org + project level; `Anthropic`: org + workspace level (overrides inherit org value when absent); `Mistral`: per organization.
- **Usage tiers** — auto-graduated by cumulative paid spend; higher tiers → higher limits (`OpenAI`: Free → Tier 1–5; `Anthropic`: Start/Build/Scale; `Google`: Free → Tier 1–3; `Mistral`: Free/Scale).
- **Long-context limits** — separate rate limit for long-context requests (`OpenAI`).
- **Shared limits** — some model families share a single TPM pool (`OpenAI`).
- **Batch API queue limits** — total input tokens queued per model; completed jobs free up the limit (`OpenAI`); `Anthropic`: max 200k batch requests in queue; `Google`: enqueued tokens per tier/model; `Mistral`: batch does not count against real-time limits.
- **Vector store ingestion limit** — 300 RPM per vector store ID (`OpenAI`).
- **Spend-based rate limits** — rolling 10-minute window spend cap (`Google`).
- **Acceleration limits** — sharp usage spikes trigger rejection even within steady limits (`Anthropic`).
- **Cache-aware ITPM** — cache-read tokens counted differently for ITPM budgeting (`Anthropic`).
- **Exponential backoff with jitter** — retry strategy for 429s; failed requests still count against per-minute limit.

### 9.2 Unified response headers

| Header | Description | Aliases |
| --- | --- | --- |
| `x-ratelimit-limit-requests` | Max requests permitted | `OpenAI` |
| `x-ratelimit-remaining-requests` | Remaining requests | `OpenAI` |
| `x-ratelimit-reset-requests` | Time until request limit resets | `OpenAI` |
| `x-ratelimit-limit-tokens` / `-project-tokens` | Max tokens / project-scoped tokens | `OpenAI` |
| `anthropic-ratelimit-requests-limit/remaining/reset` | Request limits | `Anthropic` |
| `anthropic-ratelimit-input-tokens-*` / `-output-tokens-*` | Input/output token limits | `Anthropic` |
| `anthropic-priority-input-tokens-limit/remaining/reset` | Priority tier token limits | `Anthropic` |
| `retry-after` | On 429 | `Anthropic` |
| `X-RateLimit-Remaining` | Remaining rate budget | `Mistral` |
| `x-gemini-service-tier` | Actual tier used (`priority`/`standard`) | `Google` |

### 9.3 Endpoints

| Function | OpenAI | Anthropic | Google | Mistral |
| --- | --- | --- | --- | --- |
| Retrieve rate limits | `GET /v1/fine_tuning/model_limits` | `GET /v1/organizations/rate_limits`, `.../workspaces/{id}/rate_limits` | (rate-limit URL) | `GET /api/admin/rate-limit` |

### 9.4 Alternative approaches

- **Per-key vs. per-project vs. per-org enforcement.** `Google`: per project; `OpenAI`: org + project; `Anthropic`: org + workspace overrides; `Mistral`: per organization.
- **Tier graduation trigger.** `OpenAI`/`Google`/`Mistral`: cumulative paid spend + account age; `Anthropic`: auto-placed, graduate over time.
- **Priority tier rate limits.** `Google`: 0.3× standard per model/tier; `OpenAI`: ramp rate limit may downgrade Priority to Standard at ≥1M TPM and >50% TPM increase in 15 min; `Anthropic`: drawn against both Priority and regular limits.

### 9.5 Vendor aliases

`RPM` = `RPM` (all). `TPM` = `TPM` (OpenAI/Mistral) = `ITPM` (Anthropic input) / `OTPM` (output). `RPD` = `RPD` (OpenAI/Google). `usage tier` = `usage tier` (OpenAI/Anthropic/Google) = `subscription tier` (Mistral).

---

## 10. Pipeline Stage 7 — Request Preparation: Model Selection & Lifecycle

### 10.1 Main concepts

Before a request is sent, a model must be chosen that still exists and is permitted for the caller.

- **Model lifecycle stages** — Experimental / Preview / Stable (GA) (`Google`); dated snapshots + `-latest` aliases (`OpenAI`, `Mistral`); deprecation windows differ.
- **Model allowlist/denylist per project** — `OpenAI`: project model permissions (`mode: "allow_list" | "deny_list"`, `model_ids`).
- **`reasoning.effort`** — controls how much model thinking precedes the answer (`"none" | "low" | "medium" | "high" | "xhigh"`) (`OpenAI`).
- **`text.verbosity`** — controls brevity vs. completeness (`"low" | "medium" | "high"`) (`OpenAI`).
- **`phase`** — labels assistant messages as `"commentary"` (intermediate) or `"final_answer"` (completed) (`OpenAI`).
- **`tool_search` / `defer_loading`** — deferred tool loading; model loads only the tools it needs at runtime (`OpenAI`).
- **`reasoning.encrypted_content`** — stateless handoff of reasoning items for ZDR compliance (`OpenAI`).

### 10.2 Unified parameters

| Parameter | Type | Description | Aliases |
| --- | --- | --- | --- |
| `model` | string | Model ID (e.g. `"gpt-5.6"`, `"claude-fable-5"`, `"gemini-3.5-flash"`, `"mistral-large-latest"`) | All |
| `mode` | enum | `"allow_list"` / `"deny_list"` | `OpenAI` |
| `model_ids` | array | Model IDs visible to org | `OpenAI` |
| `reasoning` | object | `{ effort: "none"|"low"|"medium"|"high"|"xhigh" }` | `OpenAI` |
| `text` | object | `{ verbosity: "low"|"medium"|"high" }` | `OpenAI` |
| `defer_loading` | boolean | Mark tool as deferred | `OpenAI` |
| `include` | array | e.g. `["reasoning.encrypted_content"]` | `OpenAI` |
| `safety_settings` / `safetySettings` | array | `[{category, threshold}]` | `Google` |

### 10.3 Endpoints

| Function | OpenAI | Anthropic | Google | Mistral |
| --- | --- | --- | --- | --- |
| Update project model permissions | `PATCH /v1/organization/projects/{id}/model_permissions` | (n/a) | (n/a) | (n/a) |
| List models | `GET /v1/models` | `GET /v1/models` | `GET /v1/models` | `GET /v1/models` |

### 10.4 Alternative approaches

- **Model gating.** `OpenAI` allows per-project allow/deny lists; others rely on workspace/key scope or Cloud IAM.
- **Reasoning effort control.** `OpenAI` exposes `reasoning.effort`; `Anthropic` exposes extended thinking; `Google`/`Mistral` rely on model choice + sampling params.
- **Deferred tool loading.** `OpenAI` `tool_search` (hosted or client-executed) + `defer_loading: true`; others load all tools upfront.

### 10.5 Vendor aliases

`model` = `model` (all). `model permissions` = `model_permissions` (OpenAI) = (workspace scope, others). `reasoning effort` = `reasoning.effort` (OpenAI) ≈ extended thinking (Anthropic).

---

## 11. Pipeline Stage 8 — Processing Tiers & Cost Optimization

### 11.1 Main concepts

Once a model is selected, the platform lets you choose a *processing tier* that trades latency, reliability, and price.

- **Standard** — baseline (1.0×) price, high reliability, seconds-to-minutes latency.
- **Batch** — asynchronous, ~0.5× price, up to 24h turnaround, separate rate-limit pool (`OpenAI`, `Anthropic`, `Google`, `Mistral` all offer 50% discount).
- **Flex** — ~0.5× price, best-effort "sheddable" capacity, minutes latency target, synchronous, may be preempted (`OpenAI`, `Google`).
- **Priority** — ~1.8× price (Google) / uplift (OpenAI), guaranteed throughput, non-sheddable, lower and more consistent latency (`OpenAI`, `Anthropic`, `Google`).
- **`service_tier` request parameter** — request-level opt-in to a tier; response echoes the tier actually used (`OpenAI: "flex"|"priority"|"auto"|"default"`; `Anthropic: "auto"|"standard_only"`; `Google: "flex"|"priority"|"standard"`).
- **Ramp rate limit (Priority)** — if traffic ramps too quickly, Priority may downgrade to Standard (`OpenAI`: ≥1M TPM and >50% TPM increase in 15 min; `Google`: graceful downgrade, billed at Standard).
- **Regional processing uplift** — 10% uplift for models released on/after March 5, 2026, eligible for data residency (`OpenAI`); 1.1× for US-only inference on Opus 4.6 / Sonnet 4.6+ (`Anthropic`).
- **Prompt caching** — see Stage 9.

### 11.2 Unified parameters

| Parameter | Type | Description | Aliases |
| --- | --- | --- | --- |
| `service_tier` | enum | Tier selection | `OpenAI`, `Anthropic`, `Google` |
| `timeout` | number | SDK-level timeout (Flex: ~900s) | `OpenAI`, `Google` |

### 11.3 Response usage object

| Field | Description | Aliases |
| --- | --- | --- |
| `usage.prompt_tokens` / `input_tokens` | Total prompt/input tokens | `OpenAI: prompt_tokens`; `Anthropic: input_tokens` |
| `usage.completion_tokens` / `output_tokens` | Completion/output tokens | `OpenAI: completion_tokens`; `Anthropic: output_tokens` |
| `usage.total_tokens` | Total tokens | `OpenAI` |
| `usage.prompt_tokens_details.cached_tokens` | Cache hit tokens | `OpenAI` |
| `usage.cache_read_input_tokens` / `cache_creation_input_tokens` | Cache read/creation | `Anthropic` |
| `usage.completion_tokens_details.reasoning_tokens` | Reasoning tokens | `OpenAI` |
| `usage.completion_tokens_details.accepted_prediction_tokens` / `rejected_prediction_tokens` | Predicted output tokens | `OpenAI` |
| `usage.service_tier` | Tier actually used | `Anthropic` |
| `usage.total_cached_tokens` | Cache hits | `Google` |

### 11.4 Alternative approaches

- **Flex vs. Priority vs. Standard vs. Batch.** Each tier optimizes a different axis (cost, latency, reliability, throughput). Batch is always async; Flex/Priority are sync.
- **Priority downgrade behavior.** `OpenAI`: ramp-triggered downgrade to Standard billed at Standard; `Google`: limits-exceeded graceful downgrade to Standard; `Anthropic`: drawn against both Priority and regular limits.
- **Tier eligibility.** Flex/Priority supported on a subset of models (`Google`: Gemini 3.5 Flash, 3.1 Flash-Lite, 3.1 Pro Preview, etc.); Priority short-context only (`OpenAI`).

### 11.5 Vendor aliases

`service_tier` = `service_tier` (OpenAI/Anthropic/Google). `processing tier` = `service tier` (all). `Flex` = `flex` (OpenAI/Google). `Priority` = `priority` (OpenAI/Anthropic/Google). `Standard` = `default` (OpenAI) = `standard_only` (Anthropic) = `standard` (Google).

---

## 12. Pipeline Stage 9 — Prompt Caching & Context Management

### 12.1 Main concepts

Reduce cost and latency by reusing previously computed prompt prefixes or by compacting context.

- **Prompt caching** — cache prompt prefixes to cut cost and latency. Two modes:
  - **Automatic caching** — single `cache_control` at top level; system manages breakpoints (`Anthropic`); implicit caching enabled by default for Gemini 2.5+ (`Google`); automatic routing to servers with cached prefix (`OpenAI`).
  - **Explicit cache breakpoints** — `cache_control` on individual content blocks for fine-grained control (`Anthropic`).
- **Cache routing** — requests routed to a machine based on a hash of the initial prompt prefix (typically first 256 tokens); `prompt_cache_key` combines with prefix hash to influence routing (`OpenAI`).
- **Cache retention policies** — In-memory (5–10 min of inactivity, up to 1 hour max) vs. Extended `24h` (offloads key/value tensors to GPU-local storage) (`OpenAI`). TTLs: 5-minute (default) and 1-hour cache writes (`Anthropic`).
- **Cache hit pricing** — Cache read costs 10% of standard input price (`Anthropic`); up to 90% input cost reduction, 80% latency reduction (`OpenAI`).
- **Cache invalidation** — byte-for-byte prefix identity required; reordered tool, interpolated timestamp, or edited earlier message silently invalidates (`Anthropic`).
- **Cache diagnostics** — per-request fingerprint comparison reveals `cache_miss_reason` (`Anthropic`, beta; ZDR-eligible).
- **Compaction** — reduce context size while preserving state. Server-side via `context_management` with `compact_threshold` (emits opaque compaction item in stream) or standalone endpoint `/responses/compact` (stateless, ZDR-friendly) (`OpenAI`). `Anthropic` exposes `usage` splits and context-window accounting.
- **Context window** — everything counts (system prompt, messages, tool results, images, docs, output, extended thinking); overflow → 400 (`Anthropic`). Context rot: accuracy degrades as token count grows.
- **Token Counting** — exact input token count before sending; separate endpoint, free, RPM-limited (`OpenAI: POST /v1/responses/input_tokens`; `Anthropic: POST /v1/messages/count_tokens`; `Google: GenerativeModel.count_tokens`, not billed; `Mistral`: token budgeting guidance).
- **Predicted Outputs** — provide anticipated output text to speed up generation when most output is already known (`OpenAI`).
- **`previous_response_id` chaining** — pass only the new user message each turn; do not manually prune (`OpenAI`). `previous_interaction_id` for stateful caching across turns (`Google`).

### 12.2 Unified parameters

| Parameter | Type | Description | Aliases |
| --- | --- | --- | --- |
| `prompt_cache_key` | string | Cache routing key | `OpenAI` |
| `prompt_cache_retention` | enum | `"in_memory"` / `"24h"` | `OpenAI` |
| `cache_control` | object | `{ "type": "ephemeral" }` + optional `ttl: "5m"|"1h"` | `Anthropic` |
| `diagnostics` | beta field | Enable cache diagnostics | `Anthropic` |
| `previous_message_id` | string | Response `id` for fingerprint comparison | `Anthropic` |
| `context_management` | array | `[{type: "compaction", compact_threshold}]` | `OpenAI` |
| `previous_response_id` | string | Continue from prior response | `OpenAI` |
| `previous_interaction_id` | string | Chain stateful conversations | `Google` |
| `prediction` | object | `{ type: "content", content: "<predicted_text>" }` | `OpenAI` |
| `accepted_prediction_tokens` / `rejected_prediction_tokens` | integer | Usage fields | `OpenAI` |

### 12.3 Endpoints

| Function | OpenAI | Anthropic | Google | Mistral |
| --- | --- | --- | --- | --- |
| Count input tokens | `POST /v1/responses/input_tokens` | `POST /v1/messages/count_tokens` | `GenerativeModel.count_tokens` | (guidance) |
| Standalone compaction | `POST /v1/responses/compact` | (n/a) | (n/a) | (n/a) |
| Responses with compaction | `POST /v1/responses` with `context_management` | (n/a) | (n/a) | (n/a) |

### 12.4 Alternative approaches

- **Automatic vs. explicit cache breakpoints.** `Anthropic` recommends starting with automatic (single top-level `cache_control`); switch to explicit breakpoints on individual blocks for fine-grained control.
- **In-memory vs. extended cache retention.** `OpenAI` offers `in_memory` (5–10 min) and `24h` (GPU-local storage); `gpt-5.5`/pro require `24h`.
- **Server-side compaction vs. standalone compact endpoint vs. manual pruning.** `OpenAI` offers all three; standalone `/responses/compact` is fully stateless and ZDR-friendly.
- **Stateless input-array chaining vs. `previous_response_id`.** `OpenAI` supports both; do not manually prune when using `previous_response_id`.
- **Token counting via API vs. local tokenizers.** API returns model-exact count (handles images/files/tools); local tokenizers (e.g. `tiktoken`) cannot.

### 12.5 Vendor aliases

`prompt caching` = `prompt caching` (OpenAI/Anthropic) = `context caching` (Google). `cache_control` = `cache_control` (Anthropic) ≈ `prompt_cache_key` (OpenAI). `compaction` = `compaction` (OpenAI) ≈ context-window management (Anthropic). `token counting` = `input_tokens` count (OpenAI) = `count_tokens` (Anthropic/Google).

---

## 13. Pipeline Stage 10 — Safety, Guardrails, Moderation & Approvals

### 13.1 Main concepts

Controls that validate input, output, and tool behavior, and that require human approval before side-effecting actions.

- **Moderation API** — free-to-use API to reduce unsafe content in completions (`OpenAI: /v1/moderations`; `Google: safety_settings`).
- **Moderation scores in generation request** — request moderation scores inline with Responses/Chat Completions (`OpenAI: moderation object`).
- **Safety filters / settings** — adjustable filters across harm categories, blocked based on probability; default Off for Gemini 2.5+; built-in protections (child safety) always on (`Google`).
- **Guardrails** — automatic validation of input, output, or tool behavior. Input guardrails run before the main agent; output guardrails validate/redact final output; tool guardrails check arguments/results around a function tool call (`OpenAI` Agents SDK).
- **`runInParallel`** — `false` = blocking (sequential, stops main agent if tripped), `true` = parallel (lower latency, speculative work) (`OpenAI`).
- **Tripwire** — boolean signaling a guardrail blocked the request (`OpenAI: tripwireTriggered / tripwire_triggered`).
- **Human-in-the-loop approvals** — pause a run before a side-effecting tool call; person/policy approves or rejects (`OpenAI: needsApproval / needs_approval`; `Anthropic: user.tool_confirmation`; `Mistral: tool_confirmations[] / DeferredToolCallsException`; `Google: interaction.requires_action`).
- **Interruptions & resumable state** — when a tool needs review, the run records an approval interruption; result returns `interruptions` + resumable `state`; resume via `state.approve(interruption)` (`OpenAI`).
- **Prompt injections** — untrusted text/data entering an AI system to override instructions; mitigate via developer-message precedence, structured outputs for data flow, tool approvals, guardrails for user inputs, trace graders/evals (`OpenAI` Agent Builder safety).
- **Safety identifiers** — privacy-preserving hashed user/session identifiers (`OpenAI`).
- **Know your customer (KYC)** — registration/login requirements; linking to existing accounts; credit card/ID (`OpenAI`).
- **Adversarial testing (red-teaming)** — test robustness against adversarial/prompt-injection inputs (`OpenAI`).
- **Permission policies (managed agents)** — control whether server-executed tools (agent toolset + MCP toolset) run automatically or wait for human approval; custom tools are app-executed (not governed) (`Anthropic`).

### 13.2 Unified parameters

| Parameter | Type | Description | Aliases |
| --- | --- | --- | --- |
| `safety_identifier` | body string | Hashed username/email or session ID | `OpenAI` |
| `OpenAI-Safety-Identifier` | header | Realtime API safety identifier | `OpenAI` |
| `max_completion_tokens` / `max_tokens` | integer | Limit output tokens to reduce misuse | `OpenAI`, all |
| `safety_settings` / `safetySettings` | array | `[{category, threshold}]` | `Google` |
| `threshold` (HarmBlockThreshold) | enum | `OFF`, `BLOCK_NONE`, `BLOCK_ONLY_HIGH`, `BLOCK_MEDIUM_AND_ABOVE`, `BLOCK_LOW_AND_ABOVE`, `HARM_BLOCK_THRESHOLD_UNSPECIFIED` | `Google` |
| `runInParallel` | boolean | Guardrail blocking vs. parallel | `OpenAI` |
| `tripwireTriggered` / `tripwire_triggered` | boolean | Guardrail tripwire | `OpenAI` |
| `needsApproval` / `needs_approval` | boolean | Tool requires approval | `OpenAI` |
| `tool_confirmations[]` | array | `[{tool_call_id, confirmation: "approve"|"deny"}]` | `Mistral` |
| `require_approval` | string | MCP tool approval requirement (e.g. `"never"`) | `OpenAI` |
| `readOnlyHint` | boolean | MCP tool annotation marking read-only; absence → write requiring confirmation | `OpenAI` |
| `result` (`user.tool_confirmation`) | enum | `"allow"` / `"deny"` + `deny_message` | `Anthropic` |

### 13.3 Endpoints

| Function | OpenAI | Anthropic | Google | Mistral |
| --- | --- | --- | --- | --- |
| Moderation | `POST /v1/moderations` | (n/a) | (safety_settings in generateContent) | `POST /v1/moderations`, `/v1/chat/moderations` |
| Inline moderation scores | `moderation` object on generation request | (n/a) | `safetyRatings` in response | (n/a) |
| Approve interruption | `state.approve(interruption)` | `user.tool_confirmation` event | (interaction.requires_action) | `tool_confirmations[]` in response |

### 13.4 Alternative approaches

- **Blocking vs. parallel guardrails.** `runInParallel: false` (blocking, stops main agent if tripped) vs. `true` (parallel, lower latency, speculative work) (`OpenAI`).
- **Preventive (input guardrails) vs. reactive (output guardrails) vs. tool guardrails.** `OpenAI` Agents SDK supports all three; workflow boundaries: input guardrails run only for the first agent in a chain; output guardrails only for the final-output agent; tool guardrails only on their attached tool.
- **Approval model.** `OpenAI`: `needsApproval` on function tools + resumable `state`; `Anthropic`: `user.tool_confirmation` events (allow/deny); `Mistral`: `DeferredToolCallsException` + `tool_confirmations[]`; `Google`: `interaction.requires_action` webhook event.
- **Safety filter defaults.** `Google`: default Off for Gemini 2.5+ (built-in core-harm protections always on); others: moderation opt-in.

### 13.5 Vendor aliases

`guardrails` = `guardrails` (OpenAI Agents SDK) ≈ `safety filters` (Google) ≈ `moderation` (OpenAI/Mistral). `approval` = `needsApproval` (OpenAI) = `tool_confirmation` (Anthropic) = `tool_confirmations[]` (Mistral) = `requires_action` (Google). `tripwire` = `tripwireTriggered` (OpenAI).

---

## 14. Pipeline Stage 11 — Orchestration: Agents, Tools, MCP, Webhooks, Sandboxes

### 14.1 Main concepts

Layers that let the model call external tools, hand off to specialists, run in sandboxes, and emit events.

- **Agent** — reusable, versioned configuration (tools, MCP servers, permission policies, model) (`Anthropic` Managed Agents; `OpenAI` Agents SDK `Agent` / `SandboxAgent`; `Mistral` Agents).
- **Session** — stateful agent execution in a managed cloud sandbox (or self-hosted); transcripts persist until deleted (`Anthropic`).
- **Environment** — sandbox template (cloud or self-hosted) with networking controls (`Anthropic`).
- **Handoffs** — a specialist takes over the conversation for that branch (`OpenAI` Agents SDK).
- **Agents as tools** — a manager stays in control and calls specialists as bounded capabilities (`OpenAI`).
- **MCP (Model Context Protocol)** — connect MCP servers to agents for external tools/data. Hosted MCP (remote server runs through platform surface) vs. Local MCP (your runtime owns connection/approvals/network) (`OpenAI`, `Mistral` Connectors, `Anthropic` MCP connector).
- **MCP connector** — up to 20 MCP servers per agent; names unique; every `mcp_servers` entry must be referenced by an `mcp_toolset` (`Anthropic`).
- **Vault** — collection of credentials registered once, referenced by ID at session creation to authenticate MCP servers (`Anthropic`).
- **Webhooks** — HTTP POST notifications delivered to an endpoint you control when subscribed API events occur (`OpenAI`, `Anthropic`, `Google`). Standard Webhooks specification (`OpenAI`, `Google` static). Static (registered per project) vs. Dynamic (bound to a specific request config) webhooks (`Google`).
- **Scheduled deployments** — run an agent on a recurring cron schedule; emits webhook events on lifecycle changes and run outcomes (`Anthropic`).
- **Sandboxes & code execution isolation** — isolated, Unix-like execution environment with filesystem, shell, packages, mounts, ports, snapshots (`OpenAI` Agents SDK `SandboxAgent`).
- **Connectors Debugger** — validate a Connector server from Studio before production (`Mistral`).
- **ChatGPT Developer Mode** — full MCP client support for all tools, both read and write; powerful but dangerous; OAuth/No Auth/Mixed Auth; tool call review/confirmation respects `readOnlyHint` (`OpenAI`).

### 14.2 Unified parameters

| Parameter | Type | Description | Aliases |
| --- | --- | --- | --- |
| `name`, `instructions`, `model` | string | Agent attributes | All |
| `handoffs` | array | Agents this agent can hand off to | `OpenAI` |
| `tools` | array | Tools incl. agents-as-tools | `OpenAI` |
| `handoffDescription` | string | Short description for routing | `OpenAI` |
| `toolName` / `tool_name`, `toolDescription` / `tool_description` | string | Expose agent as callable tool | `OpenAI` |
| `serverLabel` / `server_label` | string | MCP server label | `OpenAI` |
| `serverUrl` / `server_url` | string | Hosted MCP server URL | `OpenAI` |
| `require_approval` | string | Approval requirement (`"never"` etc.) | `OpenAI` |
| `mcp_servers` | array | Up to 20 MCP servers per agent | `Anthropic` |
| `mcp_toolset.configs[]` | array | Per-tool overrides keyed by tool name | `Anthropic` |
| `networking.type` / `allowed_hosts` / `allow_mcp_servers` / `allow_package_managers` | enum / array / bool | Environment networking | `Anthropic` |
| `vault` credential auth types | enum | `environment_variable`, `mcp_oauth` | `Anthropic` |
| `webhook_config.uris` / `user_metadata` | array / object | Dynamic webhook config | `Google` |
| `revocation_behavior` | enum | `REVOKE_PREVIOUS_SECRETS_AFTER_H24` or immediate | `Google` |
| `defaultManifest` / `default_manifest` | Manifest | Default workspace contract for fresh sessions | `OpenAI` |
| `capabilities` | array | Replaces defaults (filesystem, shell, compaction) | `OpenAI` |
| `image` | string | Docker image for sandbox | `OpenAI` |

### 14.3 Endpoints

| Function | OpenAI | Anthropic | Google | Mistral |
| --- | --- | --- | --- | --- |
| Create agent | Agents SDK `Agent` | `POST /v1/agents` | (Managed Agents via Interactions) | `POST /v1/agents` (deprecated) |
| Create environment | (n/a) | `POST /v1/environments` | (n/a) | (n/a) |
| Create session | (n/a) | `POST /v1/sessions` | (Interactions) | (n/a) |
| Stream session events | (n/a) | `GET /v1/sessions/{id}/stream` | (Interactions stream) | (n/a) |
| Create deployment | (n/a) | `POST /v1/deployments` | (n/a) | (n/a) |
| List deployment runs | (n/a) | `GET /v1/deployment_runs` | (n/a) | (n/a) |
| Create vault / credential | (n/a) | `POST /v1/vaults`, `POST /v1/vaults/credentials` | (n/a) | (n/a) |
| Create webhook | dashboard | (n/a) | `POST /v1/webhooks` | (n/a) |
| Rotate webhook secret | (n/a) | (n/a) | `POST /v1/webhooks/{id}/rotate_secret` | (n/a) |
| List/create connectors | (n/a) | (n/a) | (n/a) | `GET/POST /v1/connectors` |
| Call connector tool directly | (n/a) | (n/a) | (n/a) | `POST /v1/connectors/{id}/tools/{tool}/call` |
| Set/delete connector credentials | (n/a) | (n/a) | (n/a) | `POST/DELETE /v1/connectors/{id}/user|workspace/credentials/{name}` |

### 14.4 Webhook inbound HTTP

| Header / Field | Description | Aliases |
| --- | --- | --- |
| `webhook-id` | Unique event ID (idempotency key) | `OpenAI`, `Google` |
| `webhook-timestamp` | Unix integer | `OpenAI`, `Google` |
| `webhook-signature` / `X-Webhook-Signature` | `v1,` + base64 HMAC (static) / JWKS JWT (dynamic) | `OpenAI` (symmetric), `Anthropic` (`whsec_`-prefixed 32-byte secret), `Google` (static symmetric, dynamic JWKS RS256) |
| `user-agent: OpenAI/1.0...` | — | `OpenAI` |
| Event `type` | e.g. `response.completed`, `batch.succeeded`, `vault.archived` | All |
| Event `data` | Event-specific payload | All |
| Retry policy | 2xx quickly; non-2xx/timeouts retry with exponential backoff (72h `OpenAI`, 24h `Google`) | `OpenAI`, `Google` |

### 14.5 Webhook event catalogs (union)

| Event type | Vendor | Trigger |
| --- | --- | --- |
| `response.completed` | OpenAI | Background response generated |
| `tunnel.created/updated/deleted` | OpenAI | Tunnel lifecycle |
| `vault.archived/deleted`, `vault_credential.archived/deleted/refresh_failed` | Anthropic | Vault lifecycle |
| Deployment lifecycle / run events, session events | Anthropic | Managed Agents |
| `batch.succeeded/cancelled/expired/failed` | Google | Batch jobs |
| `interaction.requires_action/completed/failed/cancelled` | Google | Interactions LRO |
| `video.generated` | Google | Video generation LRO |

### 14.6 Alternative approaches

- **Hosted vs. local MCP.** `OpenAI` hosted (`hostedMcpTool`) vs. local (`MCPServerStdio`); `Anthropic` MCP connector (up to 20 servers); `Mistral` registered Connectors (platform-managed transport, no local MCP management).
- **Handoffs vs. agents-as-tools.** Handoffs = specialist takes over the branch; agents-as-tools = manager keeps ownership and calls specialists as bounded capabilities (`OpenAI`).
- **Static vs. dynamic webhooks.** `Google` static (symmetric secret, registered per project) vs. dynamic (asymmetric JWKS, bound to a request config, `background: true` required); `OpenAI`/`Anthropic` static only.
- **Cloud vs. self-hosted sandboxes.** `Anthropic` Environment `config.type: cloud | self-hosted`; `OpenAI` sandbox providers (Blaxel, Cloudflare, Daytona, Docker, E2B, Modal, Runloop, Unix-local, Vercel).
- **OAuth discovery through tunnel.** `OpenAI` tunnel can carry OAuth discovery so MCP server stays private; `Anthropic` vault `mcp_oauth` credential type.

### 14.7 Vendor aliases

`agent` = `Agent` (OpenAI/Anthropic/Mistral). `session` = `session` (Anthropic) ≈ `interaction` (Google). `environment` = `environment` (Anthropic) ≈ `sandbox` (OpenAI). `vault` = `vault` (Anthropic). `MCP connector` = `mcp_servers` + `mcp_toolset` (Anthropic) = `Connectors` (Mistral) = `hostedMcpTool`/`MCPServerStdio` (OpenAI). `webhook` = `webhook` (all). `handoff` = `handoff` (OpenAI).

---

## 15. Pipeline Stage 12 — Execution Modes: Sync, Streaming, Background, WebSocket, Batch

### 15.1 Main concepts

How the request is actually executed and how results are delivered.

- **Synchronous** — request blocks until response ready (default).
- **Streaming (`stream: true`)** — server-sent events (SSE) transport; typed, predefined-schema events; partial completions harder to moderate (`OpenAI`, `Anthropic`, `Google`, `Mistral`).
- **WebSocket mode** — persistent connection for long-running, tool-call-heavy workflows (~40% faster for 20+ tool calls); send only incremental input items per turn plus `previous_response_id`; 60-min max duration, one in-flight response at a time (`OpenAI`).
- **Background mode** — kick off long-running model tasks asynchronously; poll GET endpoint while `queued`/`in_progress`; idempotent cancel; streaming + background; ZDR-incompatible (stores ~10 min for polling) (`OpenAI`, `Google: background=true`).
- **Batch operations** — collect requests into a `.jsonl` file, kick off a batch job, poll status, retrieve results; 50% cost discount; separate rate-limit pool; 24-hour completion window (`OpenAI`, `Anthropic`, `Google`, `Mistral`).

### 15.2 Batch specifics

| Aspect | OpenAI | Anthropic | Google | Mistral |
| --- | --- | --- | --- | --- |
| Discount | 50% | 50% | ~50% | 50% |
| Max requests per batch | 50,000 | 100,000 | (varies) | 100,000 |
| Max file size | 200 MB | 256 MB | 2 GB | 512 MB |
| Completion window | 24h | 24h | 24h | (queue-based) |
| Results retention | 30 days | 24h download | 6 weeks | 24h download |
| Target endpoints | `/v1/responses`, `/v1/chat/completions`, `/v1/embeddings`, `/v1/completions`, `/v1/moderations`, `/v1/images/*`, `/v1/videos` | Messages API (`/v1/messages/batches`) | `:batchGenerateContent`, embeddings | `/v1/chat/completions`, `/v1/embeddings`, `/v1/fim/completions`, `/v1/moderations`, `/v1/ocr`, `/v1/classifications`, `/v1/conversations`, `/v1/audio/transcriptions` |
| `custom_id` constraints | unique | 1–64 chars `^[a-zA-Z0-9_-]{1,64}$` | `metadata.key` | unique |
| Batch creation rate limit | 2,000 batches/hour | 1,000 RPM to all Batch endpoints | 100 concurrent | (varies) |
| Inline batch | (no) | (no) | (yes, <20MB for images) | (yes, `inline_batch_data`) |

### 15.3 Batch lifecycle states

| OpenAI | Anthropic | Google | Mistral |
| --- | --- | --- | --- |
| `validating` → `failed` / `in_progress` → `finalizing` → `completed` (or `expired`); cancel: `cancelling` → `cancelled` | `in_progress` → (`canceling`) → `ended`; expire 24h | `JOB_STATE_PENDING` → `JOB_STATE_RUNNING` → `JOB_STATE_SUCCEEDED|FAILED|CANCELLED|EXPIRED` (after 48h) | (status field) |

### 15.4 Unified parameters

| Parameter | Type | Description | Aliases |
| --- | --- | --- | --- |
| `stream` | boolean | Enable SSE streaming | All |
| `stream_options: {"include_usage": true}` | object | Usage stats during streaming | `OpenAI`, `Mistral` |
| `background` | boolean | Run async | `OpenAI`, `Google` |
| `starting_after` | integer (query) | Last `sequence_number` cursor for stream resume | `OpenAI` |
| `last_event_id` | string | Resume a disconnected stream | `Google` |
| `event_id` | string | Unique ID in each streamed delta | `Google` |
| `model`, `input`, `tools`, `generate` | various | WebSocket `response.create` payload | `OpenAI` |
| `input_file_id` / `input_files` | string/array | File ID from upload | `OpenAI`, `Mistral` |
| `endpoint` | string | Target endpoint path | `OpenAI`, `Mistral` |
| `completion_window` | string | Only `"24h"` (OpenAI) | `OpenAI` |
| `metadata` | object | Arbitrary key/value | `OpenAI`, `Mistral` |
| `inline` | boolean | Return results inline | `Mistral` |
| `previous_interaction_id` | string | Chain background conversations | `Google` |

### 15.5 Endpoints (batch & background)

| Function | OpenAI | Anthropic | Google | Mistral |
| --- | --- | --- | --- | --- |
| Upload input file | `POST /v1/files` (`purpose: "batch"`) | (inline `requests[]`) | `POST /upload/v1beta/files` | `POST /v1/files` (`purpose: "batch"`) |
| Create batch | `POST /v1/batches` | `POST /v1/messages/batches` | `POST /v1beta/models/{model}:batchGenerateContent` | `POST /v1/batch/jobs` |
| Retrieve batch | `GET /v1/batches/{id}` | `GET /v1/messages/batches/{id}` | `GET /v1beta/{BATCH_NAME}` | `GET /v1/batch/jobs/{id}` |
| List batches | `GET /v1/batches` | `GET /v1/messages/batches` | `GET /v1beta/batches` | (list files) |
| Cancel batch | `POST /v1/batches/{id}/cancel` | `POST /v1/messages/batches/{id}/cancel` | `POST /v1beta/{BATCH_NAME}:cancel` | (cancel) |
| Delete batch | (n/a) | (n/a) | `DELETE /v1beta/{BATCH_NAME}` | (n/a) |
| Download results | `GET /v1/files/{file_id}/content` | `GET {results_url}` | `GET /download/v1beta/{file_name}:download` | `GET /v1/files/{file_id}/content` |
| Generate background response | `POST /v1/responses` (`background: true`) | (n/a) | `POST /v1beta/interactions` (`background: true`) | (n/a) |
| Poll background response | `GET /v1/responses/{id}` | (n/a) | `GET /v1beta/interactions/{id}` | (n/a) |
| Cancel background response | `POST /v1/responses/{id}/cancel` | (n/a) | `POST /v1beta/interactions/{id}/cancel` | (n/a) |
| WebSocket Responses | `WS wss://api.openai.com/v1/responses` | (n/a) | (n/a) | (n/a) |

### 15.6 Streaming event types (OpenAI Responses API)

Lifecycle (once): `ResponseCreatedEvent`, `ResponseInProgressEvent`, `ResponseFailedEvent`, `ResponseCompletedEvent`, `Error`.
Output items: `ResponseOutputItemAdded`, `ResponseOutputItemDone`.
Content parts: `ResponseContentPartAdded`, `ResponseContentPartDone`.
Text: `ResponseOutputTextDelta`, `ResponseOutputTextAnnotationAdded`, `ResponseTextDone`.
Refusal: `ResponseRefusalDelta`, `ResponseRefusalDone`.
Function calls: `ResponseFunctionCallArgumentsDelta`, `ResponseFunctionCallArgumentsDone`.
File search / code interpreter: (see source).
Google stream events: `step.delta`, `interaction.completed`.

### 15.7 Alternative approaches

- **Sync vs. streaming vs. background vs. WebSocket vs. batch.** Choose by latency tolerance and scale. Streaming = lowest user-perceived latency; background = long-running async; WebSocket = tool-call-heavy persistent; batch = massive offline at 50% cost.
- **File-based vs. inline batch.** `Google` and `Mistral` support inline batch (`inline_batch_data` / `inlined_requests`); `OpenAI`/`Anthropic` file-based only (Anthropic inline `requests[]` in body).
- **Background + streaming combo.** `OpenAI`: create with `background: true, stream: true`; resume with `starting_after=<sequence_number>`. `Google`: stream with reconnect via `last_event_id`.
- **Reconnect patterns (WebSocket).** `OpenAI`: (1) continue with `previous_response_id` if persisted; (2) start fresh with `previous_response_id: null` + full input; (3) use compacted window from `/responses/compact`.

### 15.8 Vendor aliases

`streaming` = `stream: true` (all). `background` = `background: true` (OpenAI/Google). `batch` = `batch` (OpenAI) = `Message Batch` (Anthropic) = `batchGenerateContent` (Google) = `batch/jobs` (Mistral). `custom_id` = `custom_id` (OpenAI/Anthropic) = `metadata.key` (Google). `completion_window` = `completion_window` (OpenAI) = `expires_at 24h` (Anthropic) = `JOB_STATE_EXPIRED 48h` (Google).

---

## 16. Pipeline Stage 13 — Files, Storage & Data Lifecycle

### 16.1 Main concepts

Handling uploaded files, batch result files, and their retention.

- **Files API** — upload, list, get metadata, delete media files (`Google`: resumable uploads, 48h auto-delete, 20GB/project, 2GB/file, no download; `Mistral`: 512MB max, 30-day retention unless deleted; `OpenAI`: `purpose: "batch"`, `expires_after` auto-deletion; `Anthropic`: Files Beta).
- **Batch result files** — `.jsonl` output/error files; auto-deleted after a retention window (`OpenAI`: 30 days; `Anthropic`: 24h download; `Google`: 6 weeks; `Mistral`: 24h download).
- **File retention** — `OpenAI: expires_after` (auto-deletes files); `Google`: 48h auto-delete; `Mistral`: 30 days unless deleted.
- **Storage limits** — `Google`: 20GB/project, 2GB/file; `OpenAI`: vector store ingestion 300 RPM; `Mistral`: 512MB file.
- **Resumable uploads** — `Google` supports resumable upload protocol.

### 16.2 Unified parameters

| Parameter | Type | Description | Aliases |
| --- | --- | --- | --- |
| `file` | file stream | The file to upload | All |
| `purpose` | string | e.g. `"batch"` | `OpenAI`, `Mistral` |
| `expires_after` | object | Auto-deletes files after a duration | `OpenAI` |

### 16.3 Endpoints

| Function | OpenAI | Anthropic | Google | Mistral |
| --- | --- | --- | --- | --- |
| Upload file | `POST /v1/files` | `POST /v1/files` (Beta) | `POST /upload/v1beta/files` | `POST /v1/files` |
| Get metadata | `GET /v1/files/{id}` | `GET /v1/files/{id}` | `GET /v1beta/{name}` | `GET /v1/files` |
| List | `GET /v1/files` | `GET /v1/files` | `GET /v1beta/files` | `GET /v1/files` |
| Download | `GET /v1/files/{id}/content` | (n/a) | `GET /download/v1beta/{name}:download` | `GET /v1/files/{id}/content` |
| Delete | `DELETE /v1/files/{id}` | `DELETE /v1/files/{id}` | `DELETE /v1beta/{name}` | `DELETE /v1/files/{id}` |

### 16.4 Alternative approaches

- **Auto-delete vs. manual delete.** `OpenAI: expires_after`; `Google`: 48h auto-delete; `Mistral`: 30-day retention; `Anthropic`: manual delete.
- **Download support.** `Google`: file download NOT supported (metadata only); others support download.

### 16.5 Vendor aliases

`file` = `file` (all). `purpose` = `purpose` (OpenAI/Mistral). `expires_after` = `expires_after` (OpenAI).

---

## 17. Pipeline Stage 14 — Logging, Audit, Tracing & Observability

### 17.1 Main concepts

Recording what happened, for debugging, compliance, and quality improvement.

- **Audit logs** — list recent user actions and configuration changes for the org (`OpenAI: GET /v1/organization/audit_logs`; `Mistral: GET /api/admin/audit-logs`).
- **Audit log events (tunnels)** — `tunnel.created/updated/deleted` (`OpenAI`).
- **Tracing** — built into Agents SDK, enabled by default; emits structured record of model calls, tool calls, handoffs, guardrails, custom spans (`OpenAI`). `Anthropic` Managed Agents: sessions stateful, transcripts persist until deleted. `Mistral`: OpenTelemetry traces with spans, searchable.
- **Trace wrapping** — wrap multiple runs in one trace (`OpenAI: withTrace` TS / `trace` Python).
- **Abuse Monitoring Logs** — retained up to 30 days by default; may contain customer content and derived metadata; MAM/ZDR controls modify (`OpenAI`). `Google`: flagged data retained 55 days for policy enforcement; not used to train/fine-tune models (except policy enforcement).
- **Observability suite** — Enterprise-tier only (`Mistral`): Explorer (search/filter/inspect chat completion events), Judges (automated scoring criteria, versioned, live-judging), Campaigns (batch annotations on live traffic using a Judge + filters, 100–10,000 events), Datasets (curated JSONL collections for eval/fine-tuning). Workflow observability via OTel activities.
- **Logs & Datasets** — logging view in dashboard; configurable retention (7/14/28/55 days max); datasets no expiry; default 1,000 logs per project; export CSV/JSONL/Sheets (`Google`).
- **Compliance Activity Feed** — queryable within 1 minute of occurring, retained 6 years (`Anthropic`).
- **Access Transparency** — log of every access to your data by staff/systems, plus preservation events (`Anthropic`).
- **Cache diagnostics** — per-request fingerprint comparison reveals `cache_miss_reason` (`Anthropic`, beta; ZDR-eligible).
- **Workflow events** — events emitted during workflow executions, listable with cursor pagination (`Mistral: GET /v1/workflows/events`).
- **Observability logs/traces search** — production chat completion events searchable (`Mistral: POST /v1/observability/logs/search`, `POST /v1/observability/traces/search`).

### 17.2 Unified parameters

| Parameter | Type | Description | Aliases |
| --- | --- | --- | --- |
| `limit` | int | Max logs to return | `OpenAI`, `Anthropic`, `Google`, `Mistral` |
| `activity_types[]` | array | Filter by activity type | `Anthropic` |
| `search_expression` | string | Search logs/traces | `Mistral` |
| `order` | enum | `"asc" | "desc"` | `Mistral` |
| `from` / `to` | date-time | Time bounds | `Mistral`, `Anthropic` |
| `after_id` / `before_id` | string | Cursor pagination | `Anthropic`, `Mistral` |
| `next_cursor` | string | Cursor pagination | `Mistral`, `Google` (next_page) |
| `page` / `page_size` | string/int | Page-token pagination | `Anthropic`, `Mistral` |
| `store` | boolean | Request-level logging toggle | `Google` |

### 17.3 Endpoints

| Function | OpenAI | Anthropic | Google | Mistral |
| --- | --- | --- | --- | --- |
| List audit logs | `GET /v1/organization/audit_logs` | (Activity Feed via Compliance) | (Cloud Audit Logs) | `GET /api/admin/audit-logs` |
| Access Transparency | (n/a) | `GET /v1/compliance/access_events` | (n/a) | (n/a) |
| Search observability logs | (n/a) | (n/a) | (n/a) | `POST /v1/observability/logs/search` |
| Search observability traces | (tracing SDK) | (n/a) | (n/a) | `POST /v1/observability/traces/search` |
| Get trace/span | (tracing SDK) | (n/a) | (n/a) | `GET /v1/observability/traces/{id}`, `.../spans/{id}` |
| List workflow events | (n/a) | (n/a) | (n/a) | `GET /v1/workflows/events` |
| Judges CRUD | (n/a) | (n/a) | (n/a) | `POST/GET/PUT/DELETE /v1/observability/judges[/{id}]`, `.../live-judging` |
| Campaigns | (n/a) | (n/a) | (n/a) | `GET/POST/DELETE /v1/observability/campaigns[/{id}][/status][/selected-events]` |
| Datasets | (n/a) | (n/a) | (n/a) | `/v1/observability/datasets` |

### 17.4 Alternative approaches

- **Built-in tracing vs. OTel export.** `OpenAI` Agents SDK tracing (enabled by default, Traces dashboard); `Mistral` OTel traces with configurable sampling; `Anthropic` stateful session transcripts.
- **Log search model.** `Mistral` Observability suite (Explorer + Judges + Campaigns + Datasets) is the richest; `Google` Logs & Datasets with 55-day retention + share-with-Google; `OpenAI` audit logs + tracing; `Anthropic` Compliance Activity Feed (6-year retention) + Access Transparency.
- **Pagination schemes.** Cursor (`after_id`/`before_id`, returns `has_more`/`first_id`/`last_id`) vs. page-token (`page`, returns `next_page`) — endpoint-family dependent (`Anthropic`, `Mistral`).

### 17.5 Vendor aliases

`audit log` = `audit_log` (OpenAI/Mistral) = `Activity Feed` / `access_events` (Anthropic) = `Cloud Audit Logs` (Google). `trace` = `trace` (OpenAI) = `traces` (Mistral OTel) ≈ `session transcript` (Anthropic). `observability` = `Observability suite` (Mistral) ≈ `Logs & Datasets` (Google) ≈ `Tracing` (OpenAI).

---

## 18. Pipeline Stage 15 — Compliance, Privacy, Data Retention & Legal

### 18.1 Main concepts

Legal and regulatory controls on data handling.

- **Compliance API** — read chat content/attachments and delete on demand; enumerate org directory; support eDiscovery, DLP, account-deletion, SIEM correlation, chain-of-custody exports (`Anthropic`). Shared rate limit: 600 RPM per parent org across every `/v1/compliance/` endpoint.
- **Compliance key types & scopes** — Compliance Access Key (`sk-ant-api01-...`, all scopes, immutable), Admin API key (`sk-ant-admin01-...`, `read:compliance_activities` only). Scopes: `read:compliance_activities`, `read:compliance_user_data`, `delete:compliance_user_data`, `read:compliance_org_data` (`Anthropic`).
- **HIPAA / BAA** — self-serve enablement (download BAA + Implementation Guide, accept as authorized legal rep); permanent (cannot be disabled); API auto-enforces feature restrictions (`Anthropic`). PHI guidance: PHI in message content / files / metadata; not in workspace names, user info, billing, support tickets. `strict: true` schemas cached separately — not PHI-protected.
- **GDPR rights** — access, rectification, erasure, data portability, object, restriction (`Mistral`).
- **Privacy & data controls** — data usage policies, training opt-out, data retention settings, Labs model access (`Mistral`). Vibe privacy controls (model training opt-out, public chat sharing, user feedback, chat retention); API privacy controls (model training opt-out, data retention settings, Labs model access).
- **ZDR-eligible features** — Messages (default for many features), Token Counting, Cache Diagnostics, Inference geo `us` (`Anthropic`). `OpenAI`: ZDR eligibility table per endpoint.
- **Non-ZDR / stateful resources** — Claude Managed Agents (sessions persist until deleted), Code execution tool (container data retained up to 30 days) (`Anthropic`).
- **Workspace-level data retention override** — orgs with ZDR can enable 30-day data retention per workspace (`Anthropic`).
- **CMEK legal retention exceptions** — Anthropic may retain records where required by law (NCMEC reports under 18 U.S.C. § 2258A, exigent risk of serious harm, ToS violations) (`Anthropic`).
- **Supported regions** — Claude API accessible from a defined list of countries/territories; access unsupported from others (`Anthropic`, `Google`). `Google`: AI Studio free in all available regions.
- **Data use by tier** — `Google`: Free tier = content used to improve products; Paid/Enterprise = NOT used.
- **Licensing** — `Google`: content under CC BY 4.0, code samples under Apache 2.0.

### 18.2 Endpoints (Compliance API — Anthropic)

| Function | Endpoint |
| --- | --- |
| List activities | `GET /v1/compliance/activities` |
| List chats | `GET /v1/compliance/apps/chats` |
| Get chat messages | `GET /v1/compliance/apps/chats/{chat_id}/messages` |
| Delete chat | `DELETE /v1/compliance/apps/chats/{chat_id}` |
| Download file content | `GET /v1/compliance/apps/chats/files/{file_id}/content` |
| Get file metadata | `GET /v1/compliance/apps/chats/files/{file_id}` |
| Delete file | `DELETE /v1/compliance/apps/chats/files/{file_id}` |
| Download generated file | `GET /v1/compliance/apps/chats/generated_files/{gen_file_id}/content` |
| Download artifact content | `GET /v1/compliance/apps/artifacts/{artifact_version_id}/content` |
| List projects | `GET /v1/compliance/apps/projects` |
| Delete project | `DELETE /v1/compliance/apps/projects/{project_id}` |

### 18.3 Compliance API errors

- `400` — bad cursor (`after_id` under mismatched `order_by`), bad time-filter/sort-key pairing.
- `401` — bad key.
- `403` — `permission_error` with `Missing required scopes...`.
- `409` — `conflict_error` (project delete with attached chats).
- `429` — shared 600/min/parent-org limit.
- `5xx` — server errors.

### 18.4 Alternative approaches

- **Self-serve vs. custom BAA.** `Anthropic` offers both; self-serve is permanent once enabled.
- **Read + delete key separation.** Recommended: two keys (read + delete) so a leaked read key cannot delete (`Anthropic`).
- **Data retention scope.** `OpenAI`: per-project retention override (`organization_default` / `zero_data_retention` / `modified_abuse_monitoring` / `none`); `Anthropic`: per-workspace 30-day override; `Mistral`: privacy controls in Admin Panel; `Google`: tier-based (Free = used for improvement).

### 18.5 Vendor aliases

`Compliance API` = `Compliance API` (Anthropic) ≈ `Cloud Audit Logs` (Google). `BAA` = `BAA` (Anthropic). `GDPR rights` = `GDPR rights` (Mistral). `ZDR` = `Zero Data Retention` (OpenAI/Anthropic). `training opt-out` = `training opt-out` (Mistral/Google).

---

## 19. Pipeline Stage 16 — Versioning, Deprecation & Changelog

### 19.1 Main concepts

How the platform evolves and retires models/endpoints over time.

- **API version header** — `anthropic-version: 2023-06-01` required on all requests (`Anthropic`); `v1`/`v1beta` URL channels (`Google`).
- **Beta headers** — `anthropic-beta` header (or SDK `betas` parameter) accesses experimental features before GA; each feature doc states the exact beta name; invalid/inaccessible beta → 400 (`Anthropic`). `managed-agents-2026-04-01` for Managed Agents.
- **Api-Revision header** — specifies API revision to avoid breaking changes (`Google`, e.g. `2026-05-20`).
- **Model lifecycle stages** — Experimental (`-exp-`), Preview (`-preview`), Stable/GA (often `-001` or versionless) (`Google`). Latest alias models: `gemini-pro-latest`, `gemini-flash-latest`.
- **Model deprecation notice periods**:
  - `OpenAI`: GA models ≥6 months; specialized variants ≥3 months; Preview models may retire with ~2 weeks notice (not recommended for production).
  - `Anthropic`: ≥60 days notice before retiring publicly released models; notices sent to customers with active deployments; migration guide provided.
  - `Google`: deprecation announcements on Release Notes page; earliest shutdown dates on Deprecations page; Stable + Preview schedules tracked.
  - `Mistral`: old aliases deprecated with three-month sunset window before removal; dated suffixes (`mistral-small-2402`) + `-latest` aliases.
- **Endpoint deprecation lifecycle** — deprecated endpoints grouped under `/api/endpoint/deprecated/*`; newer equivalents under `/api/endpoint/beta/*` (`Mistral`).
- **SDK major versions** — semantic versioning; `Mistral` V1 → V2 migration (unified `mistralai` package).
- **Dedicated capacity** — after shutdown, may be possible to provision dedicated capacity (contact sales) (`OpenAI`).
- **Model snapshots** — `chat-latest` / `gpt-5.3-chat-latest` rolling pointers to the ChatGPT Instant snapshot (`OpenAI`).
- **Monthly changelog** — dated entries with type tag (Feature/Update/Fix), affected models/endpoints, description (`OpenAI`, `Google`, `Mistral`).
- **Judge versioning** — `base_revision`, `up_revision`, `down_revision` for revision tracking (`Mistral`).

### 19.2 Deprecated → replacement model mappings (selected)

| Vendor | Deprecated | Replacement |
| --- | --- | --- |
| OpenAI | `gpt-5-2025-08-07` | `gpt-5.5` |
| OpenAI | `gpt-5-mini-2025-08-07` | `gpt-5.4-mini` |
| OpenAI | `o3-2025-04-16` | `gpt-5.5` |
| OpenAI | `dall-e-2`, `dall-e-3` | `gpt-image-2` |
| Anthropic | `claude-sonnet-4-20250514` | `claude-sonnet-4-6` |
| Anthropic | `claude-3-haiku-20240307` | `claude-haiku-4-5-20251001` |
| Google | `gemini-2.5-pro` | `gemini-3.1-pro-preview` |
| Google | `gemini-2.5-flash` | `gemini-3.5-flash` |
| Google | `gemini-embedding-001` | `gemini-embedding-2` |
| Mistral | old aliases | `-latest` aliases |

### 19.3 Deprecated endpoints (OpenAI selected)

| Endpoint | Shutdown date | Replacement |
| --- | --- | --- |
| `POST /v1/prompts` | Nov 30, 2026 | Migrate prompt content into application code |
| Assistants API (whole) | Aug 26, 2026 | Responses API / Conversations API |
| Evals platform | Nov 30, 2026 | Promptfoo |
| Agent Builder | Nov 30, 2026 | Agents SDK / ChatGPT Workspace Agents |
| Self-serve fine-tuning (new job creation) | Jan 6, 2027 | (inference continues until base model deprecated) |

### 19.4 Alternative approaches

- **Versioning via URL channel vs. header.** `Google`: `v1`/`v1beta` URL path + `Api-Revision` header; `Anthropic`: `anthropic-version` header; `OpenAI`/`Mistral`: dated model suffixes + SDK semantic versioning.
- **Preview model risk.** `OpenAI`: Preview models may retire with ~2 weeks notice (not for production); `Google`: Preview models have deprecation schedules; `Anthropic`: ≥60 days notice.

### 19.5 Vendor aliases

`deprecation` = `deprecation` (all). `shutdown` / `sunset` = used interchangeably (`OpenAI`). `legacy` = models/endpoints no longer receiving updates (`OpenAI`). `changelog` = `changelog` (all) = `release notes` (Anthropic/Google). `beta` = `anthropic-beta` (Anthropic) = `v1beta` (Google) = `/api/endpoint/beta/*` (Mistral) = `preview` (OpenAI).

---

## 20. Pipeline Stage 17 — Errors, Conventions & Client SDKs

### 20.1 Main concepts

How the API communicates failures and how clients should be built.

- **HTTP error codes** — 400 `invalid_request_error`, 401 `authentication_error`, 403 `permission_error` (missing scopes), 404 `not_found_error`, 409 `conflict_error`, 413 `request_too_large`, 422 `Unprocessable Entity` (Mistral), 429 `rate_limit_error` / `RESOURCE_EXHAUSTED` (Google) / `Too Many Requests` (Mistral), 5xx `api_error`.
- **Error shape** — always JSON; `type: "error"`, `error: { type, message }`, `request_id` (`Anthropic`). `OpenAI` errors similar. `Mistral`: message, type, code/params as applicable.
- **`request_id`** — always quote when contacting support (`Anthropic`, `Mistral`).
- **Max request sizes** — Messages 32 MB, Token Counting 32 MB, Batch 256 MB (`Anthropic`); Batch 200 MB (`OpenAI`); Batch 512 MB (`Mistral`); Batch 2 GB, Files 500 MB / 2 GB (`Google`).
- **Retry guidance** — exponential backoff with jitter for transient errors (429, 5xx); failed requests still count against per-minute limit (`OpenAI`, `Mistral`).
- **SDK languages** — `OpenAI`: Node, Python, Go, Ruby, Java; `Anthropic`: Python, JS/TS; `Google`: Python, Node, Go, Java; `Mistral`: Python, JS (V1/V2).
- **`n` and `best_of`** — number of completions returned/generated; total generated tokens = `max_tokens * max(n, best_of)` (`OpenAI`).
- **Cost management** — costs as function of (number of tokens) × (cost per token); reduce by switching models, shorter prompts, fine-tuning, caching (`OpenAI`).
- **Pagination conventions** — cursor (`after_id`/`before_id`, `has_more`/`first_id`/`last_id`) vs. page-token (`page`, `next_page`); endpoint-family dependent (`Anthropic`, `Mistral`).
- **Role fields** — prefer plural `roles`/`role_names` over deprecated singular `role`/`role_name` (`Mistral`).
- **Required partner header** — `x-goog-api-client: company-product/version` mandatory for partners building on Gemini (`Google`).
- **Integration paths** — Google GenAI SDK vs. Direct API (REST/gRPC) vs. OpenAI compatibility layer (`Google`).

### 20.2 Error shape (Anthropic)

```json
{
  "type": "error",
  "error": { "type": "not_found_error", "message": "The requested resource could not be found." },
  "request_id": "req_011CSHoEeqs5C35K2UUqR7Fy"
}
```

### 20.3 Retry pattern (Mistral)

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

### 20.4 General Chat Completions parameters

| Parameter | Type | Description |
| --- | --- | --- |
| `max_tokens` / `max_completion_tokens` | integer | Upper limit on generated tokens; lower values reduce latency |
| `n` | integer | Number of completions to generate per prompt (default 1) |
| `best_of` | integer | Number of completions generated for consideration (returned = highest log-prob) |
| `stream` | boolean | `true` to stream tokens as generated |
| `stop` | array/string | Stop sequences to prevent generating unneeded tokens |
| `stream_options: {"include_usage": true}` | object | Enables usage stats during streaming |

### 20.5 Alternative approaches

- **SDK vs. Direct API vs. OpenAI-compat.** `Google` recommends Google GenAI SDK for ecosystem frameworks/enterprise gateways, Direct API for edge platforms/aggregators, OpenAI-compat for text-only aggregators (feature ceiling — no native video, caching, etc.).
- **Pagination scheme.** Cursor vs. page-token — endpoint-family dependent; cursors are opaque (never parse; bound to sort key).
- **Developer API vs. Enterprise Agent Platform.** `Google` offers both through one SDK (switch via config flags); model IDs identical; auth migrates API key → service accounts; AI Studio models must be retrained in Enterprise.

### 20.6 Vendor aliases

`request_id` = `request_id` (Anthropic/Mistral). `429` = `rate_limit_error` (Anthropic) = `RESOURCE_EXHAUSTED` (Google) = `Too Many Requests` (Mistral). `413` = `request_too_large` (Anthropic). `cursor pagination` = `after_id`/`before_id` (Anthropic/Mistral) = `next_cursor` (Mistral/Google). `page-token pagination` = `page`/`next_page` (Anthropic/Mistral).

---

## 21. Cross-Vendor Terminology Alias Reference

A consolidated mapping of the same concept across all four systems.

| Canonical concept | OpenAI | Anthropic | Google | Mistral |
| --- | --- | --- | --- | --- |
| Top-level tenant | organization | organization (parent + linked) | Cloud organization / billing account | organization (under Enterprise Account) |
| Sub-tenant boundary | project | workspace | Cloud project | workspace |
| Standard API key | API key (`sk-...`) | API key (`sk-ant-api03-...`) | API key (`GEMINI_API_KEY`) | API key (no prefix) |
| Admin key | `OPENAI_ADMIN_KEY` | `sk-ant-admin01-...` | (n/a — IAM) | Admin API key (`x-api-key`) |
| Service account | service account | `svac_...` | service account | (n/a) |
| Workload federation | Workload Identity Federation | WIF | Cloud Workload Identity | (n/a) |
| Ephemeral token | Realtime ephemeral secret | (n/a) | auth_tokens (Live API) | (n/a) |
| Data residency | regional host prefix | workspace geo / inference_geo | region | (n/a) |
| ZDR | ZDR | ZDR | (paid tier not used for training) | (n/a) |
| MAM | MAM | (n/a) | (n/a) | (n/a) |
| Customer-managed encryption | EKM | CMEK | Cloud KMS | (n/a) |
| Spend cap | spend alert / usage limit | spend limit | spend cap | spending limit |
| Usage tier | usage tier (Free → Tier 5) | tier (Start/Build/Scale) | tier (Free → Tier 3) | subscription tier (Free/Scale) |
| Rate limit (requests/min) | RPM | RPM | RPM | RPS |
| Rate limit (tokens/min) | TPM | ITPM (input) / OTPM (output) | TPM | TPM |
| Processing tier | service_tier | service_tier | service_tier | (n/a) |
| Flex | flex | (n/a) | flex | (n/a) |
| Priority | priority | Priority Tier | priority | (n/a) |
| Batch | batch | Message Batch | batchGenerateContent | batch/jobs |
| Prompt caching | prompt caching | prompt caching (cache_control) | context caching | cached tokens |
| Compaction | compaction | (context window mgmt) | (n/a) | (n/a) |
| Token counting | input_tokens endpoint | count_tokens | count_tokens | (guidance) |
| Moderation | /v1/moderations | (n/a) | safety_settings | /v1/moderations |
| Guardrails | guardrails (Agents SDK) | (n/a) | safety filters | (n/a) |
| Approval | needsApproval | tool_confirmation | requires_action | tool_confirmations[] |
| Agent | Agent / SandboxAgent | Agent (Managed) | (Managed Agents) | agent (deprecated) |
| Session | (n/a) | session | interaction | (n/a) |
| Sandbox | SandboxAgent | environment | (n/a) | (n/a) |
| MCP | hostedMcpTool / MCPServerStdio | MCP connector (mcp_servers) | (n/a) | Connectors |
| Vault | (n/a) | vault | (n/a) | (n/a) |
| Webhook | webhook | webhook | webhook (static/dynamic) | (n/a) |
| Background | background=true | (n/a) | background=true | (n/a) |
| WebSocket | WS /v1/responses | (n/a) | (n/a) | (n/a) |
| Streaming | stream=true | stream | stream | stream |
| Audit log | audit_logs | Activity Feed / access_events | Cloud Audit Logs | audit-logs |
| Tracing | Traces dashboard | session transcripts | (n/a) | OTel traces |
| Observability suite | (tracing) | (n/a) | Logs & Datasets | Observability (Explorer/Judges/Campaigns/Datasets) |
| Compliance API | (audit logs) | Compliance API | (Cloud Audit Logs) | (n/a) |
| BAA/HIPAA | (n/a) | BAA (permanent) | (Cloud compliance) | (n/a) |
| API versioning | (dated model suffixes) | anthropic-version header | v1/v1beta + Api-Revision | SDK semver + /beta/* |
| Beta access | preview models | anthropic-beta header | v1beta channel | /api/endpoint/beta/* |
| Deprecation notice | 6mo GA / 3mo specialized / 2wk preview | ≥60 days | deprecation schedule | 3-month sunset |
| Changelog | changelog | release notes | changelog | changelogs |
| Private connectivity | Private Link (Azure) | (n/a) | Private Service Connect | (n/a) |
| Tunnel | tunnel-client | (n/a) | (n/a) | (n/a) |
| IP egress allowlist | published JSON | (n/a) | (n/a) | (n/a) |

---

## 22. Capability Coverage Matrix

Which vendor exposes each capability (✓ = documented, ✗ = not documented, ~ = partial / conceptual only).

| Capability | OpenAI | Anthropic | Google | Mistral |
| --- | --- | --- | --- | --- |
| Tenant isolation (org + workspace/project) | ✓ | ✓ | ~ (Cloud project) | ✓ |
| Grouping above organization (Enterprise Account) | ✗ | ✓ (parent + linked) | ✗ | ✓ (Enterprise Account/Backoffice) |
| Standard API key auth | ✓ | ✓ | ✓ | ✓ |
| Admin API key | ✓ | ✓ | ✗ (IAM) | ✓ |
| Workload Identity Federation | ✓ | ✓ | ✓ | ✗ |
| Ephemeral tokens (client-to-server) | ~ (Realtime) | ✗ | ✓ (Live API) | ✗ |
| OAuth 2.0 user auth | ~ (ChatGPT apps) | ~ (admin OAuth) | ✓ | ✗ |
| RBAC roles + groups + SCIM | ✓ | ~ (scopes) | ✓ (IAM) | ✓ |
| Custom roles | ✓ | ✗ | ✓ (IAM custom) | ✗ |
| Data residency | ✓ (project) | ✓ (workspace + request) | ✓ (project/region) | ✗ |
| ZDR | ✓ | ✓ | ~ (paid tier) | ✗ |
| MAM | ✓ | ✗ | ✗ | ✗ |
| CMEK/EKM | ✓ (EKM) | ✓ (CMEK) | ~ (Cloud KMS) | ✗ |
| Access Transparency | ✗ | ✓ | ✗ | ✗ |
| Private Link / Private Service Connect | ✓ (Azure) | ✗ | ~ (Vertex) | ✗ |
| IP egress allowlist | ✓ | ✗ | ✗ | ✗ |
| Secure MCP tunnels | ✓ | ✗ | ✗ | ✗ |
| Usage & Cost APIs | ~ (dashboard) | ✓ | ~ (dashboard) | ✓ |
| Spend limits (org/workspace/user) | ~ (alerts) | ✓ (full hierarchy + increase requests) | ✓ (project + billing) | ✓ (org + workspace) |
| Prepay/Postpay billing | ✗ | ✗ | ✓ | ✗ |
| Subscriptions (product-surface plans) | ~ (tiers) | ~ (tiers + Priority) | ✓ (Free/Paid/Enterprise) | ✓ (Vibe/Code/API) |
| Priority Tier | ✓ | ✓ | ✓ | ✗ |
| Flex tier | ✓ | ✗ | ✓ | ✗ |
| Batch operations | ✓ | ✓ | ✓ | ✓ |
| Inline batch | ✗ | ✗ | ✓ | ✓ |
| Quotas & rate limits (RPM/TPM/RPD) | ✓ | ✓ (RPM/ITPM/OTPM) | ✓ | ✓ (RPS/TPM) |
| Usage tiers | ✓ (5) | ✓ (Start/Build/Scale) | ✓ (3) | ✓ (Free/Scale) |
| Spend-based rate limits | ✗ | ✗ | ✓ (rolling 10-min) | ✗ |
| Acceleration limits | ✗ | ✓ | ✗ | ✗ |
| Prompt caching | ✓ | ✓ (automatic + explicit) | ✓ (implicit) | ~ (cached tokens reported) |
| Cache diagnostics | ✗ | ✓ (beta) | ✗ | ✗ |
| Compaction | ✓ (server-side + standalone) | ~ (context mgmt) | ✗ | ✗ |
| Token counting API | ✓ | ✓ | ✓ | ~ (guidance) |
| Predicted Outputs | ✓ | ✗ | ✗ | ✗ |
| WebSocket mode | ✓ | ✗ | ✗ | ✗ |
| Background mode | ✓ | ✗ | ✓ | ✗ |
| Streaming (SSE) | ✓ | ✓ | ✓ | ✓ |
| Moderation API | ✓ | ✗ | ✓ (safety_settings) | ✓ |
| Guardrails (input/output/tool) | ✓ (Agents SDK) | ✗ | ✗ | ✗ |
| Human-in-the-loop approvals | ✓ (needsApproval) | ✓ (tool_confirmation) | ✓ (requires_action) | ✓ (tool_confirmations) |
| Sandboxes & code execution | ✓ (SandboxAgent + providers) | ✓ (environment) | ✗ | ✗ |
| Agents (managed) | ~ (Agents SDK) | ✓ (Managed Agents) | ~ (Managed Agents) | ✓ (deprecated) |
| MCP integration | ✓ (hosted/local) | ✓ (MCP connector) | ✗ | ✓ (Connectors) |
| Webhooks | ✓ | ✓ | ✓ (static/dynamic) | ✗ |
| Scheduled deployments | ✗ | ✓ | ✗ | ✗ |
| Vaults (credentials) | ✗ | ✓ | ✗ | ✗ |
| Audit logs | ✓ | ✓ (Activity Feed) | ~ (Cloud Audit Logs) | ✓ |
| Tracing | ✓ (Agents SDK) | ~ (session transcripts) | ✗ | ✓ (OTel) |
| Observability suite (Explorer/Judges/Campaigns/Datasets) | ✗ | ✗ | ~ (Logs & Datasets) | ✓ |
| Compliance API (content read/delete) | ✗ | ✓ | ✗ | ✗ |
| HIPAA/BAA | ✗ | ✓ (permanent) | ~ | ✗ |
| GDPR rights | ✗ | ✗ | ✗ | ✓ |
| Files API | ✓ | ✓ (Beta) | ✓ | ✓ |
| File auto-delete | ✓ (expires_after) | ✗ | ✓ (48h) | ✓ (30 days) |
| Resumable uploads | ✗ | ✗ | ✓ | ✗ |
| Versioning (header/channel) | ~ (model suffixes) | ✓ (anthropic-version) | ✓ (v1/v1beta + Api-Revision) | ~ (SDK semver + /beta) |
| Beta headers | ~ (preview models) | ✓ (anthropic-beta) | ✓ (v1beta) | ✓ (/beta) |
| Deprecation notice periods | ✓ (6mo/3mo/2wk) | ✓ (≥60 days) | ✓ (schedules) | ✓ (3-month) |
| Changelog | ✓ | ✓ (release notes) | ✓ | ✓ |
| Errors & request_id | ✓ | ✓ | ~ | ✓ |
| Retry guidance (backoff + jitter) | ✓ | ~ | ~ | ✓ |
| SDK languages | ✓ (5) | ✓ (2) | ✓ (4) | ✓ (2, V1/V2) |
| OpenAI-compat layer | ✗ | ✗ | ✓ | ✗ |
| Partner integration header | ✗ | ✗ | ✓ (x-goog-api-client) | ✗ |

---

*End of specification. This document aggregates the cross-cutting capabilities documented in `openai-api.md`, `anthropic-api.md`, `google-api.md`, and `mistral-api.md` into a single exhaustive processing-pipeline specification, with explicit vendor terminology aliases and alternative approaches per step.*