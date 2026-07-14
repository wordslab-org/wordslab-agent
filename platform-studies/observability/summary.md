# Observability, Moderation & Evaluation of AI Chatbots and Agents — Unified Specification

> Aggregated reference synthesizing the capabilities documented in the four platform studies of this folder:
> - **Anthropic** (`anthropic-api.md`) — Claude Agent SDK / Claude Code CLI OpenTelemetry export + Messages-API moderation prompting pattern + in-process cost/usage.
> - **Google Gemini** (`google-api.md`) — AI Studio Logs & Datasets, Data Logging & Sharing Policy, Safety Settings, Safety Feedback, Safety & Factuality guidance.
> - **Mistral** (`mistral-api.md`) — Moderation API, Custom Guardrails, Explorer, Judges, Campaigns, Datasets.
> - **OpenAI** (`openai-api.md`) — Agents SDK MCP integrations, Tracing, Guardrails, Human-in-the-loop approvals, Agent workflow evaluation, Evaluation best practices.
>
> This document is a **super-complete end-user specification**: it takes the union of every capability, use case, parameter, field and concept found in the four systems, orders them along an **exhaustive processing pipeline**, and is written so a developer can read it from top to bottom and understand every available lever. Where the four systems use different names for the same concept, the convergence is stated explicitly. Where the same processing step can be implemented with different approaches, the alternatives are listed with their trade-offs.

---

## Table of Contents

0. [How to read this document](#0-how-to-read-this-document)
1. [Introduction — the concepts you need to know](#1-introduction--the-concepts-you-need-to-know)
2. [The unified processing pipeline at a glance](#2-the-unified-processing-pipeline-at-a-glance)
3. [Naming convergence across the four systems](#3-naming-convergence-across-the-four-systems)
4. [Approach alternatives for the same processing step](#4-approach-alternatives-for-the-same-processing-step)
5. [Stage A — Identity, Resource Attribution & Tenancy Configuration](#stage-a--identity-resource-attribution--tenancy-configuration)
6. [Stage B — Telemetry & Observability Backend Wiring](#stage-b--telemetry--observability-backend-wiring)
7. [Stage C — Ingress & Input Pre-Processing (Input Moderation / Guardrails)](#stage-c--ingress--input-pre-processing-input-moderation--guardrails)
8. [Stage D — Agent Run Execution (the model + tools + handoffs + hooks loop)](#stage-d--agent-run-execution-the-model--tools--handoffs--hooks-loop)
9. [Stage E — Observability Signals Emitted During the Run](#stage-e--observability-signals-emitted-during-the-run)
10. [Stage F — Output Post-Processing (Output Moderation / Guardrails)](#stage-f--output-post-processing-output-moderation--guardrails)
11. [Stage G — Human-in-the-Loop Approvals & Permission Boundaries](#stage-g--human-in-the-loop-approvals--permission-boundaries)
12. [Stage H — Storage, Logging & Retention](#stage-h--storage-logging--retention)
13. [Stage I — Search, Filter & Inspect Production Traffic](#stage-i--search-filter--inspect-production-traffic)
14. [Stage J — Evaluation & Scoring (Judges / Graders / Campaigns / Evals)](#stage-j--evaluation--scoring-judges--graders--campaigns--evals)
15. [Stage K — Datasets, Curation & Re-runs](#stage-k--datasets-curation--re-runs)
16. [Stage L — Production Monitoring & the Continuous Improvement Loop](#stage-l--production-monitoring--the-continuous-improvement-loop)
17. [Cross-cutting concerns](#cross-cutting-concerns)
18. [Per-platform capability matrix](#per-platform-capability-matrix)

---

## 0. How to read this document

- §1 is a gentle, concept-first introduction. Read it if any term is unfamiliar.
- §2 is the one-paragraph map of the whole pipeline.
- §3 and §4 are reference tables you will come back to: §3 says "when system X says *A* and system Y says *B*, they mean the same thing"; §4 says "for step S, you may choose between approaches 1/2/3".
- §5–§16 are the pipeline stages, each structured as a **specification block**: purpose, concepts, API surface (the union of all parameters/fields), behavior, options/alternatives, and notes/constraints.
- §17 covers concerns that cut across all stages (privacy, sensitive-content gating, enterprise tiering, trace-context propagation).
- §18 is the final matrix showing which platform exposes which capability.

> Convention: in API surface tables, the **Source** column uses `[A]` Anthropic, `[G]` Google, `[M]` Mistral, `[O]` OpenAI. A field shown without a source is the unified name proposed by this spec; the rows below it list the per-platform equivalents.

---

## 1. Introduction — the concepts you need to know

Modern AI chatbots and **agents** (systems where a model is put in a loop with tools, memory and possibly other agents) are non-deterministic, side-effecting and expensive. Four families of capability have emerged to make them observable, safe and improvable. This section introduces them in the order a developer typically meets them.

### 1.1 Observability (logs, metrics, traces)

Observability means **recording what the agent did at runtime** so you can debug, attribute cost, audit, and feed improvement. The industry distinguishes three signals:

- **Logs / events** — structured records of discrete occurrences: a prompt arrived, an API call started, a tool returned, a permission changed, an error happened. Low cost, high cardinality.
- **Metrics** — aggregatable counters/gauges/histograms: tokens consumed, dollars spent, sessions started, lines of code edited, PRs created, active time. Low cardinality, suited for dashboards and alerting.
- **Traces / spans** — a tree of nested spans describing one end-to-end run: the whole interaction, each model request, each tool call, each hook, each subagent. Spans carry timing (`duration_ms`, `ttft_ms`), identifiers (`session.id`, `request_id`), status (`ERROR`/`UNSET`), and attributes. A trace is what lets you reconstruct "what happened in this one run".

Two delivery models exist:

- **Bring-your-own OTLP backend** (Anthropic): the SDK/CLI emits OpenTelemetry signals and you ship them to any collector (Honeycomb, Datadog, Grafana, Langfuse, self-hosted). You own retention, privacy, querying.
- **Hosted dashboard** (OpenAI Traces, Mistral Explorer, Google AI Studio Logs): the platform captures traffic server-side and you inspect it in a web UI.

### 1.2 Moderation & Guardrails

Moderation is **deciding whether content is allowed** before (or after) the model sees it. Guardrails are the configurable machinery that enforces that decision inside the agent loop.

- **Input moderation** runs before the main model: it classifies the user prompt and may block it. It protects against disallowed requests and **prompt injection / jailbreaks**.
- **Output moderation** runs after the model: it inspects the generated response and may block/redact it before it reaches the user.
- **Tool guardrails** wrap a specific tool call, validating its arguments or its result.

The decision itself can be: a binary *violated / not violated*, a multi-level **risk score** (e.g. 0–3 or 0–1 probability), or a categorical label. Moderation can be implemented by (a) a dedicated classifier model, (b) the chat model itself via a prompting pattern, (c) built-in non-adjustable model safety, or (d) a custom guardrail agent you write. See §4.

### 1.3 Human-in-the-loop approvals

Some actions are too sensitive to auto-execute (cancellations, edits, shell commands, sensitive MCP actions). For these, the run **pauses** and waits for a human (or a policy engine) to approve or reject the pending tool call, then **resumes**. Approvals differ from guardrails: guardrails are automatic and synchronous; approvals are asynchronous and resumable, often backed by serializable **state**.

### 1.4 Evaluation (evals)

Evaluation is **measuring how good the agent is**, despite non-determinism. It spans:

- **Trace grading** — scoring individual end-to-end traces with structured criteria (did the agent pick the right tool? did a handoff happen when it should?).
- **Datasets** — curated, repeatable collections of conversations (with optional expected outputs / properties) used to benchmark changes.
- **Eval runs / campaigns** — batch-applying a scorer (a **Judge** / **Grader** / LLM-as-a-judge) over a filtered slice of traffic or a dataset.
- **Continuous evaluation (CE)** — running evals on every change to catch regressions.

Evals come in three evaluator types: **metric-based** (exact match, ROUGE, function-call accuracy), **human** (labelers rank or score), and **LLM-as-a-judge / model graders** (a strong model scores outputs against rubrics).

### 1.5 The improvement loop

These four families compose into a loop: **observe → moderate → approve → record → score → curate datasets → re-run → improve (prompts, routing, fine-tuning)**. Every platform in this study is a different slice of that loop; the unified spec below is the union.

### 1.6 Agents, subagents, handoffs, hooks

- An **agent** is a model + instructions + tools (+ maybe guardrails/approvals). A **run** is one invocation; a **session** is a series of runs linked by an id.
- A **subagent** is an agent spawned by another agent. Anthropic does this via the Agent/Task tool (its spans nest under the parent's `tool` span); OpenAI does it via **handoffs** between specialized agents; Mistral exposes `api_agent_id` to identify which agent handled a request.
- A **hook** (Anthropic term) is custom code that runs at defined lifecycle events of the agent loop (e.g. before/after a tool). OpenAI guardrails are conceptually adjacent but run at input/output/tool boundaries.
- An **MCP integration** (OpenAI) attaches Model Context Protocol tool surfaces to an agent — either **hosted** (platform trust model) or **SDK-managed** over stdio / streamable HTTP.

### 1.7 Cost & usage

Every run consumes tokens and money. Cost can be tracked **in-process** (a client-side estimate from a bundled price table — Anthropic's `total_cost_usd`) or **authoritatively** via the platform's billing API. Cache tokens (`cache_creation_input_tokens` charged higher, `cache_read_input_tokens` charged lower) are tracked separately from base input tokens.

---

## 2. The unified processing pipeline at a glance

```
┌─────────────────────────────────────────────────────────────────────────────┐
│ A. Identity & resource attribution        (who/what/where is this run for?) │
│ B. Telemetry & observability backend wiring (where do signals go?)         │
│ C. Ingress & input pre-processing         (input moderation / guardrails)   │
│    └─ approval gating may pause here too                                    │
│ D. Agent run execution                     (model + tools + handoffs + hooks)│
│      ├─ E. observability signals emitted inline (metrics/logs/traces/cost)   │
│      └─ G. human-in-the-loop approvals may pause mid-run                    │
│ F. Output post-processing                  (output moderation / guardrails)  │
│ H. Storage, logging & retention            (where records live, how long)   │
│ I. Search, filter & inspect traffic        (find specific runs/events)      │
│ J. Evaluation & scoring                   (Judges/Graders/Campaigns/Evals)   │
│ K. Datasets, curation & re-runs           (repeatable baselines, Batch API) │
│ L. Production monitoring & improvement loop (CE, fine-tuning, grounding)     │
└─────────────────────────────────────────────────────────────────────────────┘
                      Cross-cutting: privacy · sensitive-content gates ·
                      enterprise tiering · W3C trace-context propagation
```

Each stage is specified in §5–§16.

---

## 3. Naming convergence across the four systems

The same concept is called differently by different platforms. The unified name (left) is used throughout the spec.

| Unified concept | Anthropic | Google | Mistral | OpenAI |
|---|---|---|---|---|
| **Run / turn** | `interaction`, `query()` call, step | `generateContent` call, Interactions API call | chat completion event | `run`, trace |
| **Trace** | OTel trace (spans) | (implicit; logs are the unit) | (events; correlation_id links) | trace |
| **Span** | span (`claude_code.interaction`, `.llm_request`, `.tool`, `.hook`) | n/a (log entry) | n/a (event) | span (model call, tool call, handoff, guardrail, custom) |
| **Log event** | log event (`claude_code.*`) | log entry | chat completion event | trace step (rendered in dashboard) |
| **Metric** | OTel metric (`claude_code.*`) | `usageMetadata` (token counts) | event fields (`input_tokens`, `output_tokens`, `total_time_elapsed`) | trace attributes |
| **Input moderation / guardrail** | prompting-pattern moderation on Messages API | Safety Settings (prompt-level block via `promptFeedback`) | Custom Guardrails (`moderation_llm_v2`, input-only) | Input guardrail (`input_guardrails`, tripwire) |
| **Output moderation / guardrail** | post-hoc prompting pattern | Safety feedback (`Candidate.finishReason = SAFETY` / `PROHIBITED_CONTENT`) | n/a (guardrails are input-only) | Output guardrail (`output_guardrails`, tripwire) |
| **Tool guardrail** | (via hooks / tool_decision events) | n/a | n/a | Tool guardrail (attached to function tool) |
| **Block decision** | `violation` (bool) / `risk_level` | `blocked` (bool) / `finishReason` | `violated` (bool), `action: "block"`, HTTP 403 | `tripwire_triggered` (bool), exception `InputGuardrailTripwireTriggered` |
| **Threshold** | (implied in prompt) | `HarmBlockThreshold` (OFF/BLOCK_NONE/.../BLOCK_LOW_AND_ABOVE) | `custom_category_thresholds` (0–1 per category) | (custom in guardrail agent) |
| **Risk score granularity** | binary OR 0–3 risk_level | probability (HIGH/MEDIUM/LOW/NEGLIGIBLE) + probabilityScore + severity + severityScore | 0–1 `category_scores` | classification label OR numeric metric (0–1) |
| **Moderation categories** | customizable list (Child Exploitation, Hate, Self-Harm, …) | 4 fixed: Harassment, Hate speech, Sexually explicit, Dangerous (+ built-in non-adjustable: child safety, PROHIBITED_CONTENT) | 11 fixed: Sexual, Hate and Discrimination, Violence and Threats, Dangerous, Criminal, Self-Harm, Health, Financial, Law, PII, **Jailbreaking** | (custom — you define in guardrail agent) |
| **Judge / scorer** | (classification cookbook) | (safety classifier; LLM-as-judge in guidance) | **Judge** (CLASSIFICATION or REGRESSION, Jinja2 instructions) | **Grader** (trace grader); LLM-as-a-judge / model grader |
| **Batch scoring run** | n/a | Batch API re-run of a dataset | **Campaign** (one Judge over filtered events, background) | **Eval run** (over a dataset); legacy Evals API (deprecated) |
| **Dataset** | (manual; no dedicated surface) | **Dataset** (curated logs, no expiry, ≤1000/project) | **Dataset** (records = conversation + properties + source, JSONL import/export) | **Dataset** (golden set, eval runs) |
| **Record metadata** | (resource attributes) | (log properties) | **Properties** (`expected_output`, `category`, `grading_guidance`, `difficulty`) | golden-set labels / expected outputs |
| **Approval / human review** | `tool_decision`/`permission_mode_changed` events, hooks | human reviewers in guidance | n/a | **Approval** (`needs_approval`, `state.approve()`, resumable) |
| **Subagent / delegation** | Agent/Task tool (subagent spans nest) | (Public Preview Agents — not loggable) | `api_agent_id` (agent identity on events) | **Handoff** (triage → specialized agent) |
| **Hook** (lifecycle middleware) | **Hook** (`claude_code.hook` span) | n/a | n/a | (closest: guardrails at boundaries) |
| **Tool permission** | `tool_decision` event, `blocked_on_user` span | n/a | n/a | `needs_approval` flag, `interruptions` |
| **Telemetry enable** | `CLAUDE_CODE_ENABLE_TELEMETRY=1` | `store` (per-request) + project toggle | (Enterprise; always-on for workspace) | built-in & enabled by default |
| **Trace-context propagation** | W3C `traceparent`/`TRACEPARENT` env, `traceresponse` link | n/a | `correlation_id` | (implicit in dashboard) |
| **Cost field (estimate)** | `total_cost_usd`, `model_usage.costUSD` | `usageMetadata` (tokens only) | `input_tokens`/`output_tokens` event fields | (via trace attributes / billing API) |
| **Cache tokens** | `cache_creation_input_tokens`, `cache_read_input_tokens`, 5m/1h TTL | n/a | n/a | n/a |
| **Store / log toggle** | (telemetry enable + per-signal exporters) | `store` boolean (per request / project) | n/a (always captured for workspace) | tracing enabled-by-default, scope via SDK |
| **Sensitive-content gating** | `OTEL_LOG_USER_PROMPTS`, `OTEL_LOG_TOOL_CONTENT`, `OTEL_LOG_RAW_API_BODIES`, etc. | data-use policy, opt-in sharing | Enterprise gating | tracing scope controls |
| **End-user identity injection** | `OTEL_RESOURCE_ATTRIBUTES` (`enduser.id`, `tenant.id`); gateway OIDC | `metadata.user_id` | `metadata` on request; `api_agent_id` | run context / trace metadata |

---

## 4. Approach alternatives for the same processing step

For several pipeline steps, the four systems offer materially different implementation strategies. This table is the decision aid; the stage sections give the detail.

| Step | Alternative 1 | Alternative 2 | Alternative 3 | Alternative 4 |
|---|---|---|---|---|
| **Moderation engine** | Dedicated classifier model (Mistral `mistral-moderation-2603`; Google built-in safety) | Prompting pattern on the chat model (Anthropic Messages API) | Custom guardrail agent with structured output (OpenAI input guardrail) | Built-in non-adjustable model safety (Google `PROHIBITED_CONTENT`, Anthropic AUP) |
| **Guardrail transport** | Declarative config object on the request (Mistral `guardrails`/`moderation_llm_v2`) | Code-defined async guardrail function attached to agent/tool (OpenAI `@input_guardrail` / `tool({...})`) | Per-request safety settings array (Google `safetySettings`) | Prompt engineering (Anthropic) |
| **Decision granularity** | Binary `violation: bool` (Anthropic, Mistral `violated`) | Multi-level integer risk 0–3 (Anthropic `risk_level`) | Probability enum + numeric probability/severity scores (Google) | Classification label set (Mistral Judge CLASSIFICATION; OpenAI grader labels) |
| **Observability delivery** | Bring-your-own OTLP collector (Anthropic) | Hosted platform dashboard, server-side capture (OpenAI Traces, Mistral Explorer, Google AI Studio Logs) | Hybrid: in-process stream + OTel export (Anthropic Agent SDK message stream) | — |
| **Tracing enable model** | Off until env vars set; opt-in per signal (Anthropic) | On by default, scope via SDK controls (OpenAI) | Project-level toggle + per-request `store` override (Google) | Enterprise-tier, always captured (Mistral) |
| **Cost tracking** | In-process estimate from bundled price table (Anthropic `total_cost_usd`) | Token counts in response metadata (Google `usageMetadata`) | Authoritative platform billing API (all) | — |
| **Eval execution** | Background batch campaign over filtered traffic (Mistral Campaigns) | Trace grading in dashboard (OpenAI graders) | Batch API re-run of a curated dataset (Google) | Continuous evaluation on every change (OpenAI best practices, CE) |
| **Scorer type** | LLM-as-a-judge with Jinja2 templated instructions (Mistral Judge) | LLM grader / model grader with rubrics (OpenAI grader) | Metric-based (ROUGE, exact match, function-call accuracy) (OpenAI best practices) | Human labelers (Google guidance, OpenAI human evals) |
| **Approval flow** | Resumable serialized state, `state.approve()` + resume `run(agent, state)` (OpenAI) | Permission events + hooks (Anthropic `tool_decision`, `permission_mode_changed`, `blocked_on_user`) | Human reviewers alter/block content (Google guidance) | — |
| **Subagent delegation** | Agent/Task tool spawning subagent with nested spans (Anthropic) | Handoff to specialized agent (OpenAI) | `api_agent_id` attribution on events (Mistral) | — |
| **MCP transport** | Hosted MCP (platform trust) (OpenAI) | SDK-managed over stdio / streamable HTTP (OpenAI) | — | — |
| **Trace-context propagation** | W3C `traceparent` env injection + `traceresponse` link (Anthropic) | `correlation_id` cross-system id (Mistral) | — | — |
| **Dataset ingest** | From production traffic/logs (all) | Manual entry (Mistral) | JSONL file upload (Mistral) | From Playground / Campaign (Mistral) |
| **Retention** | You own it (OTLP backend) (Anthropic) | 7/14/28/55-day window, datasets don't expire (Google) | (hosted, Enterprise) (Mistral) | (hosted dashboard) (OpenAI) |
| **Privacy / data-use** | Sensitive-content opt-in flags (Anthropic) | Opt-in sharing under "Unpaid Services" terms with account/key/project disconnection (Google) | Enterprise gating (Mistral) | Tracing scope reduction (OpenAI) |

---

## Stage A — Identity, Resource Attribution & Tenancy Configuration

**Purpose** — Before any run, establish *who* the run is for (end user, tenant, service account), *what* system produced it (service name, version, entrypoint), and *where* it is deployed. These attributes become labels on every metric, every log event, and every span, enabling multi-team attribution, per-user audit trails, and SIEM forwarding.

### Concepts

- **Resource attributes** (OTel term): key-value pairs attached to every signal emitted by a run. Built-in keys (`session.id`, `user.id`, `organization.id`, `app.version`, `app.entrypoint`) coexist with custom keys.
- **End-user attribution**: the credential the agent uses to call the model identifies the *service*, not the human the agent acts on behalf of. You inject the human's identity as resource attributes per call.
- **Gateway OIDC identity**: when the agent calls through an identity-provider gateway, identity is stamped from the OIDC session (`user.id` = IdP subject, `user.email`, `user.groups`, `identity.source: gateway-oidc`) and overrides env-supplied `user.*`/`identity.*`.
- **Tenancy**: multi-tenant systems attach `tenant.id` so traffic can be sliced per customer.
- **Trace metadata / `metadata` on request**: lighter-weight custom key-value pairs attached to a single request (Mistral `metadata`, Google `metadata.user_id`, OpenAI run context).
- **Cardinality discipline**: custom keys become labels on every metric series; high-cardinality values inflate storage. Most systems let you exclude custom attributes from datapoint labels (Anthropic `OTEL_METRICS_INCLUDE_RESOURCE_ATTRIBUTES=false`).

### API surface (union)

| Parameter | Type | Source | Description |
|---|---|---|---|
| `service.name` | string | `[A] OTEL_SERVICE_NAME` | Override default service name (e.g. `support-triage-agent`). |
| `resource_attributes` | map<string,string> | `[A] OTEL_RESOURCE_ATTRIBUTES` | Comma-separated `key=value` applied to all signals. |
| `enduser.id` | string | `[A]` | End-user id (percent-encode before interpolating). |
| `tenant.id` | string | `[A]` | Tenant id. |
| `user.account_uuid` / `user.account_id` | string | `[A] OTEL_METRICS_INCLUDE_ACCOUNT_UUID` | Authenticated account identity. |
| `user.id` | string | `[A]` | Anonymous persisted random id from `~/.claude.json`. |
| `user.email` | string | `[A]` | OAuth user email (when OAuth). |
| `user.groups` | list<string> | `[A]` (gateway OIDC) | IdP group membership. |
| `identity.source` | string | `[A]` | `gateway-oidc` when stamped from gateway. |
| `app.version` | string | `[A] OTEL_METRICS_INCLUDE_VERSION` | SDK/CLI version. |
| `app.entrypoint` | string | `[A] OTEL_METRICS_INCLUDE_ENTRYPOINT` | `cli`, `sdk-cli`, `sdk-ts`, `sdk-py`, `claude-vscode`. |
| `terminal.type` | string | `[A]` | Terminal type (when detected). |
| `metadata.user_id` | string | `[G]` | Opaque external user id for abuse detection (no PII). |
| `metadata` | object | `[M]` | Custom key-value metadata attached to the request; filterable in Explorer. |
| `api_agent_id` | string | `[M]` | ID of the agent that handled the request. |
| `correlation_id` | string | `[M]` | Cross-system tracing identifier. |
| `organization.id` | string | `[A]` | Org UUID (always when available). |
| `prompt.id` | string | `[A]` | UUID correlating a user prompt with subsequent events until next prompt (events only). |
| `workflow.run_id` / `workflow.name` | string | `[A]` | For agents spawned by a Workflow tool run (`wf_`-prefixed). |
| `workspace.host_paths` | string | `[A]` | Host workspace dirs from desktop app (events only). |
| `include_resource_attributes` | bool | `[A] OTEL_METRICS_INCLUDE_RESOURCE_ATTRIBUTES` | If false, custom attrs go in OTLP resource block only, omitted from datapoint labels. |
| `include_session_id` | bool | `[A] OTEL_METRICS_INCLUDE_SESSION_ID` | Attach `session.id` to metrics. |
| `include_account_uuid` | bool | `[A] OTEL_METRICS_INCLUDE_ACCOUNT_UUID` | Attach account identity. |
| `include_version` | bool | `[A] OTEL_METRICS_INCLUDE_VERSION` | Attach app version. |
| `include_entrypoint` | bool | `[A] OTEL_METRICS_INCLUDE_ENTRYPOINT` | Attach entrypoint. |

### Behavior

- Built-in standard attributes always win on collision with custom keys (Anthropic).
- `OTEL_RESOURCE_ATTRIBUTES` formatting: no spaces; values can't contain whitespace, double quotes, commas, semicolons, backslashes, or control chars. Percent-encode as needed.
- Per-call injection (Anthropic): pass `env` in `ClaudeAgentOptions`/`options.env` with percent-encoded values.

### Example (Anthropic)

```python
from urllib.parse import quote
options = ClaudeAgentOptions(env={
    "OTEL_RESOURCE_ATTRIBUTES": f"enduser.id={quote(request.user_id)},tenant.id={quote(request.tenant_id)}",
})
```

### Notes / constraints

- Anthropic identity is the *service's* credential; end-user identity must be injected per call.
- Mistral `metadata` is the lightweight per-request alternative; `api_agent_id` is the agent identifier.
- Google `metadata.user_id` is opaque, no PII, used for abuse detection.
- OpenAI attributes flow via the run context and trace metadata (no env-var surface documented in the studied guides).

---

## Stage B — Telemetry & Observability Backend Wiring

**Purpose** — Decide where telemetry goes and how it gets there. This is the configuration step that precedes Stage E (signals actually being emitted).

### Concepts

- **Three independent signals** with independent enable switches and exporters: metrics, logs/events, traces. Each can target a different endpoint/protocol.
- **Exporter types**: `console`, `otlp`, `prometheus` (metrics only), `none`; comma-separated for multiple (Anthropic). Hosted dashboards are the equivalent "exporter" for OpenAI/Mistral/Google.
- **OTLP transport**: protocol (`grpc`, `http/json`, `http/protobuf`), endpoint, headers, per-signal overrides, temporality (`delta` default, `cumulative`).
- **Export intervals & flush**: batch export intervals per signal; on clean exit, flush within a bounded timeout. Spans can be dropped if the collector is slow; buffered data is lost on kill.
- **mTLS & dynamic headers**: client cert + key + passphrase per protocol; trust collector CA; dynamic rotating-token headers via an external script (runs at startup and every 29 min by default).
- **Master switch & beta gating**: a top-level enable flag (Anthropic `CLAUDE_CODE_ENABLE_TELEMETRY=1`); traces additionally need a beta flag (`CLAUDE_CODE_ENHANCED_TELEMETRY_BETA=1`, alias `ENABLE_ENHANCED_TELEMETRY_BETA`).
- **Project-level vs per-request logging toggle** (Google): `generateContent` defaults `store=false`; Interactions API defaults `store=true`. Toggling Interactions logging off disables automatic history storage/retrieval unless overridden per request.
- **Always-on / built-in** (OpenAI): tracing enabled by default in the Agents SDK; reduce via SDK controls, not via a separate "send trace" API.

### API surface (union)

**Master & signal enables**

| Parameter | Source | Description |
|---|---|---|
| `enable_telemetry` (`CLAUDE_CODE_ENABLE_TELEMETRY=1`) | `[A]` | Master switch for any signal. |
| `enable_enhanced_telemetry_beta` (`CLAUDE_CODE_ENHANCED_TELEMETRY_BETA=1`) | `[A]` | Enable span tracing (beta). |
| `metrics_exporter` (`OTEL_METRICS_EXPORTER`) | `[A]` | `console`/`otlp`/`prometheus`/`none`. |
| `logs_exporter` (`OTEL_LOGS_EXPORTER`) | `[A]` | `console`/`otlp`/`none`. |
| `traces_exporter` (`OTEL_TRACES_EXPORTER`) | `[A]` | `console`/`otlp`/`none`. |
| `store` (project toggle / per-request) | `[G]` | Whether to log this request. Default differs by API surface. |
| tracing scope controls | `[O]` | SDK-level / per-run controls to reduce tracing. |

**OTLP transport**

| Parameter | Source | Description |
|---|---|---|
| `OTEL_EXPORTER_OTLP_PROTOCOL` | `[A]` | `grpc`/`http/json`/`http/protobuf`. |
| `OTEL_EXPORTER_OTLP_ENDPOINT` | `[A]` | Collector endpoint (gRPC `:4317`; HTTP `:4318` + `/v1/{metrics,logs,traces}`). |
| `OTEL_EXPORTER_OTLP_HEADERS` | `[A]` | Auth headers (`Authorization=Bearer ...`). |
| `OTEL_EXPORTER_OTLP_{METRICS,LOGS,TRACES}_PROTOCOL` / `_ENDPOINT` | `[A]` | Per-signal overrides. |
| `OTEL_EXPORTER_OTLP_METRICS_TEMPORALITY_PREFERENCE` | `[A]` | `delta` (default) / `cumulative`. |

**Export intervals**

| Parameter | Default | Source | Description |
|---|---|---|---|
| `OTEL_METRIC_EXPORT_INTERVAL` | 60000 ms | `[A]` | Metrics batch interval. |
| `OTEL_LOGS_EXPORT_INTERVAL` | 5000 ms | `[A]` | Logs batch interval. |
| `OTEL_TRACES_EXPORT_INTERVAL` | 5000 ms | `[A]` | Spans batch interval. |

(For short-lived `query()` calls set all three to ~1000 ms.)

**mTLS**

| Protocol | Client cert | Trust CA |
|---|---|---|
| `http/protobuf`, `http/json` | `CLAUDE_CODE_CLIENT_CERT`, `CLAUDE_CODE_CLIENT_KEY`, `CLAUDE_CODE_CLIENT_KEY_PASSPHRASE?` | `NODE_EXTRA_CA_CERTS` |
| `grpc` | `OTEL_EXPORTER_OTLP_CLIENT_KEY` + `OTEL_EXPORTER_OTLP_CLIENT_CERTIFICATE` (or per-signal) | `OTEL_EXPORTER_OTLP_CERTIFICATE` |

**Dynamic headers** (http/protobuf, http/json only): `.claude/settings.json` `otelHeadersHelper` → script outputting JSON of HTTP headers; refresh interval `CLAUDE_CODE_OTEL_HEADERS_HELPER_DEBOUNCE_MS` (default 29 min).

### Behavior

- Telemetry is off until master switch is set and at least one exporter configured (Anthropic).
- The SDK produces no telemetry itself; it forwards config to the CLI child process which instruments and exports (Anthropic).
- `console` exporter not usable through the SDK (stdout is the message channel); use a local OTLP collector or Jaeger for local inspection.
- Administrators can centrally configure telemetry via managed settings `env` block (MDM-distributed), high precedence, not user-overridable.

### Notes / constraints

- Hosted dashboards (OpenAI/Mistral/Google) eliminate transport config but lock you to their UI and retention.
- Google logging requires the paid tier (billing-enabled project).

---

## Stage C — Ingress & Input Pre-Processing (Input Moderation / Guardrails)

**Purpose** — Classify the incoming user prompt (and any attached content) and decide whether the run should start at all. This is the front gate: it blocks disallowed requests, prompt-injection / jailbreak attempts, and policy violations **before** the main model runs.

### Concepts

- **Input guardrail** (OpenAI): a fast validation step before the expensive/side-effecting part of the workflow. Often a **separate guardrail agent** that classifies input with a structured output schema, returning whether a **tripwire** was triggered. Raises `InputGuardrailTripwireTriggered`.
- **Custom Guardrails** (Mistral): declarative `guardrails` array on the request, each with a `moderation_llm_v2` config. **Input-only** — they evaluate the incoming request before the model sees it; a triggered guardrail blocks with HTTP 403. Multiple guardrail objects per request; blocked if any triggers.
- **Safety Settings** (Google): per-request, per-category adjustable filters that gate prompts. Blocking is based on **probability** of content being unsafe (not severity). Built-in non-adjustable protections (child safety, `PROHIBITED_CONTENT`) always block.
- **Moderation prompting pattern** (Anthropic): no dedicated moderation model; you prompt Claude on the Messages API with the content + a category list, instruct strict JSON verdict, parse the result. Supports binary, multi-level risk (0–3), category definitions, and batch processing patterns.
- **Agent-level inheritance**: guardrails attached to an agent at creation (Mistral, OpenAI) are inherited by all conversations and overridable per request.
- **Parallel vs blocking execution** (OpenAI): blocking = guardrail must pass before main work starts; parallel = guardrails run alongside the main agent (`runInParallel: true`), trading speculative work for latency.
- **Fail-closed**: Mistral `block_on_error` blocks the request if the moderation API itself fails.
- **Anti-prompt-injection**: Mistral has a dedicated **Jailbreaking** category; Google guidance recommends prompt-injection safeguards; OpenAI input guardrails can implement custom injection checks.

### API surface (union)

**OpenAI input guardrail** (attach to `Agent.input_guardrails` / `inputGuardrails`)

| Field | Type | Description |
|---|---|---|
| `name` | string | Guardrail name. |
| `run_in_parallel` (`runInParallel`) | bool | Run alongside main agent. |
| `execute` | async fn | `async ({ input, context }) => { output_info, tripwire_triggered }`. |
| decorator | `@input_guardrail` (Py) | On async fn `(ctx, agent, input) -> GuardrailFunctionOutput`. |

`GuardrailFunctionOutput`: `{ output_info: any, tripwire_triggered: bool }`. On `true`, run blocked with `InputGuardrailTripwireTriggered`.

**Mistral guardrail object** (`guardrails` array element)

| Field | Type | Description |
|---|---|---|
| `block_on_error` | bool | Block if moderation API fails. |
| `moderation_llm_v2` | object | Single per guardrail object. |

`moderation_llm_v2`:

| Field | Type | Description |
|---|---|---|
| `custom_category_thresholds` | map<string,float> | category → threshold 0–1; `1` disables a category. |
| `ignore_other_categories` | bool | If true, only listed categories evaluated. |
| `action` | `"block"` | Block on violation. |
| `model_name` | string? | Override default moderation model. |

Attachable surfaces: inline on `chat.complete(...)`, on `conversations.start(...)`, agent-level on `agents.create(...)`.

**Google SafetySetting** (`safetySettings` array element on `generateContent`/`StreamGenerateContent`)

| Field | Type | Description |
|---|---|---|
| `category` | enum `HarmCategory` | `HARM_CATEGORY_HARASSMENT`/`_HATE_SPEECH`/`_SEXUALLY_EXPLICIT`/`_DANGEROUS_CONTENT`. |
| `threshold` | enum `HarmBlockThreshold` | `OFF`/`BLOCK_NONE`/`BLOCK_ONLY_HIGH`/`BLOCK_MEDIUM_AND_ABOVE`/`BLOCK_LOW_AND_ABOVE`/`HARM_BLOCK_THRESHOLD_UNSPECIFIED`. |

Default threshold for Gemini 2.5/3 is `Off` (model inherent safety covers most cases).

**Anthropic moderation prompting** (on `POST /v1/messages`)

| Pattern | Verdict shape |
|---|---|
| Binary | `{ "violation": bool, "categories": [...], "explanation": str? }` |
| Multi-level risk | `{ "risk_level": 0|1|2|3, "categories": [...], "explanation": str? }` |
| Category definitions | dict `category -> definition` improves precision |
| Batch | `{ "violations": [ { "id": int, "categories": [...], "explanation": str }, ... ] }` |

Recommended: `claude-haiku-4-5`, `max_tokens=200` (single) / `2048` (batch), `temperature=0`. Use `output_config.format` (json_schema) to enforce verdict shape.

### Behavior

- **Workflow boundaries** (OpenAI): input guardrails run only for the **first agent** in a chain. For per-tool validation in manager-style workflows, attach validation to the tool, not the agent.
- Mistral guardrail success response includes a `guardrails` field with per-category `score`/`violated`. Blocked response is HTTP 403 with `decisions` per category (threshold, score, violated).
- Google: if `promptFeedback.blockReason` is set, the prompt was blocked and **no candidates are returned**. Block reasons: `SAFETY`, `OTHER`, `BLOCKLIST`, `PROHIGITED_CONTENT`, `IMAGE_SAFETY`.
- Anthropic: Claude's built-in harmlessness training (AUP) may flag content deemed dangerous regardless of your prompt.

### Reference category lists

- **Mistral (11 fixed)**: Sexual; Hate and Discrimination; Violence and Threats; Dangerous; Criminal; Self-Harm; Health; Financial; Law; PII; **Jailbreaking**.
- **Google (4 adjustable + built-in)**: Harassment; Hate speech; Sexually explicit; Dangerous. Built-in non-adjustable: child safety, prohibited content.
- **Anthropic (customizable)**: Child Exploitation, Conspiracy Theories, Hate, Indiscriminate Weapons, Intellectual Property, Non-Violent Crimes, Privacy, Self-Harm, Sex Crimes, Sexual Content, Specialized Advice, Violent Crimes (append e.g. "Underage Posting").
- **OpenAI**: fully custom (you define categories in your guardrail agent).

### Notes / constraints

- Mistral `mistral-moderation-2603` (`mistral-moderation-2411` deprecated March 31, 2026); continuously improved → custom pipelines keyed on `category_scores` may require recalibration.
- Moderation is not a hard policy gate (Anthropic, OpenAI custom); verdicts must be parsed/validated.
- Google blocking is probability-based; severity scores are surfaced for finer triage but not used for blocking.
- Provide clear user feedback for blocked/flagged input (use the `explanation` field).

---

## Stage D — Agent Run Execution (the model + tools + handoffs + hooks loop)

**Purpose** — The core loop: model call → optional tool calls → optional handoffs/subagents → optional hooks → repeat until a final response. Observability signals (Stage E) and approval pauses (Stage G) happen *inside* this loop.

### Concepts

- **Run** — one invocation (`query()` / `run()` / `generateContent` / `chat.complete`). A **session** is a series of runs linked by a session id.
- **Step** — a single request/response cycle within a run; each step produces assistant messages with token usage.
- **Tool call** — the model emits a tool_use block; the runtime executes the tool and returns a tool_result. Tools may need **approval** (Stage G).
- **Handoff** (OpenAI) — triage agent hands off to a specialized agent; the receiving agent continues the run.
- **Subagent** (Anthropic) — Agent/Task tool spawns a subagent whose spans nest under the parent's `tool` span, making the delegation chain one trace.
- **Hook** (Anthropic) — custom code at defined lifecycle events; emitted as `claude_code.hook` spans (detailed beta tracing).
- **MCP integration** (OpenAI) — attaches Model Context Protocol tool surfaces: **hosted MCP** (platform trust, `hostedMcpTool`/`HostedMCPTool`, `serverLabel`/`serverUrl`, `require_approval`) or **SDK-managed** over stdio / streamable HTTP (`MCPServerStdio`, `name`, `params.command`+`params.args`, `connect()`/`close()`).
- **Streaming** — SSE response; for OpenAI, streaming does not create a separate approval system (resume from same `state`).
- **Cache tokens** (Anthropic) — `cache_creation_input_tokens` (charged higher) and `cache_read_input_tokens` (charged lower); 5-min default TTL, 1-hour with `ENABLE_PROMPT_CACHING_1H=1` (subscription users already get 1h).
- **Deduplication by message id** — parallel tool calls produce multiple assistant messages sharing the same nested `BetaMessage` `id` with identical usage; dedupe by id to avoid inflated totals.

### API surface (union — run entry points)

| Surface | Source | Description |
|---|---|---|
| `query(prompt, ...)` | `[A]` | Stream messages; emits `assistant`/`result` messages. |
| `run(agent, input)` / `Runner.run(...)` | `[O]` | Run an agent; returns interruptions + state when approval needed. |
| `generateContent` / `StreamGenerateContent` | `[G]` | Model call with `contents`, `config` (safety_settings, store, …). |
| `chat.complete(...)` / `conversations.start(...)` | `[M]` | Chat completion / conversation with optional `guardrails`. |

**MCP tool parameters (OpenAI)**

Hosted (`hostedMcpTool`/`HostedMCPTool`): `serverLabel`/`server_url`, `serverUrl`/`server_url`, `require_approval` (Py `tool_config.type="mcp"`).
SDK-managed (`MCPServerStdio`): `name`, `params.command`, `params.args`; lifecycle `connect()`/`close()` or `async with`.

### Notes / constraints

- Input guardrails (Stage C) run only for the first agent; output guardrails (Stage F) only for the final-output agent; tool guardrails only on their attached tool (OpenAI).
- The decision to use a multi-agent architecture should be driven by your evals (Stage J) — starting multi-agent adds unnecessary complexity (OpenAI best practices).
- Google logging limitations: Imagen, Veo, embedding, Robotics models, inputs with videos/GIFs/PDFs, and Public Preview Agents in the Gemini API are **not** loggable.

---

## Stage E — Observability Signals Emitted During the Run

**Purpose** — Capture what happened: token/cost metrics, structured log events, distributed trace spans, and in-process cost/usage. This is the data that feeds Stages H–L.

### E.1 Metrics

**Anthropic metric reference** (`claude_code.*`):

| Metric | Unit | Description |
|---|---|---|
| `claude_code.session.count` | count | CLI sessions started (attr `start_type`: `fresh`/`resume`/`continue`/`agents_view`). |
| `claude_code.lines_of_code.count` | count | Lines modified (attr `type`: `added`/`removed`). |
| `claude_code.pull_request.count` | count | PRs created. |
| `claude_code.commit.count` | count | Git commits created. |
| `claude_code.cost.usage` | USD | Cost of the session. |
| `claude_code.token.usage` | tokens | Tokens used. |
| `claude_code.code_edit_tool.decision` | count | Code-editing tool permission decisions. |
| `claude_code.active_time.total` | s | Total active time. |

Other platforms expose equivalent counts via response metadata (`usageMetadata`, event fields `input_tokens`/`output_tokens`/`total_time_elapsed`).

### E.2 Log events

**Anthropic log event categories** (`claude_code.` prefix, exported via OTel logs): per-prompt events, per-API-request/error events, per-tool events (`tool_result`, `tool_decision`), MCP events (`mcp_server_connection`), `permission_mode_changed`, `assistant_response`. Content-bearing events gated by flags (Stage §17.1).

Security-relevant events (`tool_decision`, `tool_result`, `mcp_server_connection`, `permission_mode_changed`) form a **per-user audit trail** when end-user identity is attached (Stage A) — forwardable to a SIEM.

### E.3 Traces / spans

**Anthropic span hierarchy** (beta tracing):

```
claude_code.interaction              (one turn: prompt -> response)
├── claude_code.llm_request          (each Claude API call)
├── claude_code.hook                 (each hook; detailed beta)
└── claude_code.tool                 (each tool invocation)
    ├── claude_code.tool.blocked_on_user   (permission wait)
    ├── claude_code.tool.execution          (execution)
    └── (Agent tool) subagent claude_code.llm_request / .tool spans
```

**`claude_code.interaction` attributes**: `user_prompt` (gated), `user_prompt_length`, `interaction.sequence`, `interaction.duration_ms`.

**`claude_code.llm_request` attributes**: `model`, `gen_ai.system` (`anthropic`), `gen_ai.request.model`, `query_source`, `agent_id`, `parent_agent_id`, `workflow.run_id`, `workflow.name`, `speed` (`fast`/`normal`), `llm_request.context` (`interaction`/`tool`/`standalone`), `duration_ms`, `ttft_ms`, `input_tokens`, `output_tokens`, `cache_read_tokens`, `cache_creation_tokens`, `request_id`, `gen_ai.response.id`, `client_request_id`, `attempt`, `success`, `status_code`, `error`, `response.has_tool_call`, `stop_reason`, `gen_ai.response.finish_reasons`.

**`claude_code.tool` attributes**: `tool_name`, `duration_ms`, `result_tokens`, `tool_use_id`, `gen_ai.tool.call.id`, `agent_id`, `parent_agent_id`, `workflow.run_id`, `workflow.name` (gated), `file_path`, `full_command`, `skill_name`, `subagent_type` (gated by `OTEL_LOG_TOOL_DETAILS`); `tool.output` span event (input/output bodies truncated 60 KB) with `OTEL_LOG_TOOL_CONTENT=1`.

**`claude_code.tool.blocked_on_user`**: `duration_ms`, `decision` (`accept`/`reject`), `source`.
**`claude_code.tool.execution`**: `duration_ms`, `tool_use_id`, `gen_ai.tool.call.id`, `success`, `error` (gated).
**`claude_code.hook`**: `hook_event`, `hook_name`, `num_hooks`, `hook_definitions` (gated), `duration_ms`, `num_success`, `num_blocking`, `num_non_blocking_error`, `num_cancelled`. Requires `ENABLE_BETA_TRACING_DETAILED=1` + `BETA_TRACING_ENDPOINT`.

Span status: `llm_request`, `tool.execution`, `hook` set `ERROR` on failure; others end `UNSET`. Each retry is a `gen_ai.request.attempt` span event.

**OpenAI default trace contents**: overall run/workflow; each model call; tool calls and outputs; handoffs and guardrails; custom spans wrapped via `withTrace`/`trace`.

**`withTrace` / `trace`** (wrap multiple `run` calls in one trace):

| Parameter | Type | Description |
|---|---|---|
| `name` | string | Trace/workflow name in dashboard. |
| `callback` (TS) | async fn | Block of `run` calls to capture. |

### E.4 Cost & usage (in-process)

**Anthropic message-stream fields**:

| Field | On | Description |
|---|---|---|
| `message.message.id` / `message_id` | `assistant` | Nested `BetaMessage` id; dedup by this. |
| `message.message.usage` / `usage` | `assistant` | Per-step token counts (`input_tokens`, `output_tokens`, `cache_creation_input_tokens`, `cache_read_input_tokens`). |
| `total_cost_usd` | `result` (`SDKResultMessage`/`ResultMessage`) | Cumulative estimated cost across all steps (success **and** error). |
| `usage` (dict) | `result` | Cumulative token counts. |
| `modelUsage` / `model_usage` | `result` | Map model → `{ costUSD, inputTokens, outputTokens, cacheReadInputTokens, cacheCreationInputTokens }`. |

Constraints: estimates drift with pricing changes / unknown models / unmodelable billing rules — **do not** bill end users or trigger financial decisions from these. Each `query()` returns its own `total_cost_usd`; no session-level total — accumulate yourself. Always read cost from the result message regardless of subtype. On `output_tokens` discrepancies for the same id: use the highest value and prefer the result message's accumulated estimate.

**Google `usageMetadata`**: `promptTokenCount`, etc. **Mistral**: `input_tokens`, `output_tokens`, `total_time_elapsed` event fields.

### E.5 W3C trace-context propagation (Anthropic)

- SDK auto-propagates W3C trace context into the CLI subprocess: when an OTel span is active in your app, `TRACEPARENT`/`TRACESTATE` are injected into the child env, and `claude_code.interaction` becomes a child of your span.
- Bash/PowerShell subprocesses inherit a `TRACEPARENT` env referencing the active tool-execution span → subprocess spans nest under `claude_code.tool.execution`.
- Direct Anthropic API calls carry a W3C `traceparent` header (the `llm_request` span context); the API's `traceresponse` header is recorded as a span link. Same for outbound HTTP MCP requests. Header not sent to third-party providers.
- Auto-injection skipped if you set `TRACEPARENT` explicitly in `options.env`. Interactive CLI ignores inbound `TRACEPARENT`; only Agent SDK and `claude -p` honor it.
- Through a custom `ANTHROPIC_BASE_URL` proxy, `traceparent`/`TRACEPARENT` suppressed by default; set `CLAUDE_CODE_PROPAGATE_TRACEPARENT=1` to enable.

---

## Stage F — Output Post-Processing (Output Moderation / Guardrails)

**Purpose** — Inspect the model's generated response and decide whether it may leave the system. Symmetric to Stage C but on the output side.

### Concepts

- **Output guardrail** (OpenAI): validates or redacts the final output before it leaves the system; runs only for the agent producing the final output. Same `GuardrailFunctionOutput`/tripwire contract as input guardrails.
- **Tool guardrail** (OpenAI): checks arguments or results around a specific function tool call; attached to the tool, not the agent.
- **Safety feedback** (Google): `Candidate.finishReason == SAFETY` means the response was blocked by adjustable filters; `PROHIBITED_CONTENT` means built-in protections activated. Blocked content is **not returned**. Per-category `SafetyRating`: `category`, `probability`, `probabilityScore`, `severity`, `severityScore`, `blocked`.
- **Output moderation via prompting** (Anthropic): same Messages-API pattern as input, applied to the generated text.

### API surface (union)

**Google SafetyRating** (per candidate, per category)

| Field | Type | Description |
|---|---|---|
| `category` | enum | `HARM_CATEGORY_*`. |
| `probability` | enum | `HIGH`/`MEDIUM`/`LOW`/`NEGLIGIBLE`. |
| `probabilityScore` | float | Numeric probability. |
| `severity` | enum | `HARM_SEVERITY_*`. |
| `severityScore` | float | Numeric severity. |
| `blocked` | bool | Whether this category caused the block. |

**Google prompt feedback** (`promptFeedback`): `blockReason` (`SAFETY`/`OTHER`/`BLOCKLIST`/`PROHIBITED_CONTENT`/`IMAGE_SAFETY`); `safetyRatings[]`.

**OpenAI output/tool guardrail**: same fields as input guardrail (§ Stage C), attached to `Agent.output_guardrails` / the function tool.

### Notes / constraints

- Mistral guardrails are **input-only** — they do not moderate model outputs.
- Google: blocking decisions keyed on probability; severity surfaced for finer triage but not used for blocking.
- For per-tool validation in manager-style workflows (OpenAI), attach validation to the tool, not the agent.

---

## Stage G — Human-in-the-Loop Approvals & Permission Boundaries

**Purpose** — Pause the run before side-effecting actions (cancellations, edits, shell, sensitive MCP actions) so a human or policy engine can approve or reject. Distinct from guardrails: asynchronous, resumable.

### Concepts

- **Approval** (OpenAI): a function tool opts in via `needs_approval=True` / `needsApproval: true` at definition time. The run pauses; `result.interruptions` plus resumable `result.state` returned; `state.approve(interruption)` then resume `await run(agent, state)` / `await Runner.run(agent, state)`. Works across handoffs and nested `agent.asTool()` calls. Serialize `state` and resume later for delayed review; same pattern for streamed runs.
- **Permission events** (Anthropic): `tool_decision` log events, `claude_code.tool.blocked_on_user` spans (`decision` `accept`/`reject`, `source`), `permission_mode_changed` events. Hook spans capture lifecycle.
- **Human reviewers** (Google guidance): alter or block content; not a runtime API surface.
- **MCP `require_approval`** (OpenAI hosted MCP): `tool_config.require_approval` policy (e.g. `"never"`).

### API surface (union)

**OpenAI approval**

| Surface | TS | Py |
|---|---|---|
| Define | `tool({ ..., needsApproval: true })` | `@function_tool(needs_approval=True)` |
| Read interruptions | `result.interruptions` | `result.interruptions` |
| Get state | `result.state` | `result.to_state()` |
| Approve | `state.approve(interruption)` | `state.approve(interruption)` |
| Resume | `await run(agent, state)` | `await Runner.run(agent, state)` |

`tool({...})` approval-relevant params: `name`, `description`, `parameters` (Zod), `needsApproval`, `execute`.

**Anthropic permission surfaces**: `tool_decision`/`tool_result`/`mcp_server_connection`/`permission_mode_changed` log events; `claude_code.tool.blocked_on_user` span; `claude_code.hook` span.

### Notes / constraints

- Guardrails (automatic) and approvals (human) share the same resumable `state` model (OpenAI).
- Streaming does not create a separate approval system (OpenAI): wait for settle → inspect `interruptions` → resolve → resume from same `state`.

---

## Stage H — Storage, Logging & Retention

**Purpose** — Where the captured signals live, for how long, and under what data-use terms.

### Concepts

- **Bring-your-own** (Anthropic OTLP): you own the collector and backend; retention is your policy.
- **Hosted logs** (Google AI Studio Logs): project-level capture of every supported call (`GenerateContent`/`BatchGenerateContent`/`StreamGenerateContent`/Interactions, incl. OpenAI-compat endpoints); retention default **55 days**, configurable to **7/14/28/55**; logs marked for deletion after the window unless saved to a dataset.
- **Datasets as retention** (Google): curated logs saved to a dataset don't expire; default storage limit **up to 1,000 logs** per project.
- **Hosted events** (Mistral Explorer): every chat completion event in a workspace; Enterprise-tier.
- **Hosted traces** (OpenAI Traces dashboard): captured server-side by default.
- **Per-API default storage** (Google): Interactions API `store=true` by default; `generateContent` `store=false` by default. Toggling Interactions logging off disables automatic history storage/retrieval unless overridden per request.
- **Data-use policy** (Google): logged prompts/responses **not** used by Google for product improvement by default; sharing a dataset with Google opts it into "Unpaid Services" terms (may be used for training/evaluation with human review **after** account/key/project disconnection). Don't contribute sensitive/confidential/proprietary/personal data.

### API surface (union)

| Parameter | Source | Description |
|---|---|---|
| `store` (project toggle / per-request) | `[G]` | Logging enable. |
| retention window (7/14/28/55 days) | `[G]` | Project log retention. |
| dataset (named, no expiry, ≤1000/project) | `[G]` | Curated slice. |
| (OTLP backend retention) | `[A]` | You own. |
| (workspace capture, Enterprise) | `[M]` | Always-on. |
| (traces dashboard) | `[O]` | Built-in. |

### Google logging limitations (not loggable)

Imagen, Veo, embedding, Robotics models; inputs containing videos, GIFs, or PDFs; Public Preview Agents in the Gemini API.

---

## Stage I — Search, Filter & Inspect Production Traffic

**Purpose** — Find specific runs/events/traces to debug, audit, or curate into datasets.

### Concepts

- **Explorer** (Mistral): search/filter/inspect every chat completion event in a workspace; structured filter language (AND/OR/parentheses); inspect full conversation (messages, tool calls, metadata); export filtered slices to Datasets; seed a Judge/Campaign from a filter. Restricted to Workspace administrators (Enterprise).
- **AI Studio Logs page** (Google): filter bar; reverse-chronological; click entry for payload preview (full prompt, response, prior-turn context; Interactions entries link to `previous_interaction_id`).
- **Traces dashboard** (OpenAI): Logs > Traces; inspect a representative workflow trace.
- **OTel backend queries** (Anthropic): filter on `session.id` to see related `query()` calls as one timeline.

### Mistral filter condition object

```json
{ "field": "<filter name>", "op": "<operator>", "value": <typed value> }
```
Combine with `{"AND": [...]}` / `{"OR": [...]}`.

**Operators**: `=`/`eq`, `!=`/`ne`, `contains`, `includes`, `excludes`, `>`/`gt`, `<`/`lt`, `>=`/`gte`, `<=`/`lte`, `isnull`, `length_equals`, `starts_with`, `ends_with`, `matches` (regex).

**Event fields**: `timestamp`, `model_name`, `last_user_message_preview`, `response_messages_preview`, `invoked_tools` (list), `total_time_elapsed` (s), `input_tokens`, `output_tokens`, `api_agent_id`, `event_id`, `correlation_id`, `first_system_message`, `metadata`.

**SDK**: `mistral.beta.observability.chat_completion_events.search(search_params={filters: ...}, extra_fields=[...], page_size=...)`.

### Query design tips (Mistral)

Start broad (time range) → add one business condition (tool/model/topic) → add one technical condition (latency/content) → scan before exporting. Treat exports as snapshots (descriptive names e.g. `support_web_search_2026_02`).

---

## Stage J — Evaluation & Scoring (Judges / Graders / Campaigns / Evals)

**Purpose** — Measure agent quality despite non-determinism. The maturity progression (OpenAI): **traces** (debug) → **trace grading** (graders score traces) → **datasets & eval runs** (repeatable, comparable). Mistral's loop: Explorer → Judges → Campaigns → Explorer (annotations) → Datasets.

### Concepts

- **Judge** (Mistral): LLM-based evaluator with `type` (CLASSIFICATION with `options` `{value, description}`[], or REGRESSION with `min`/`max`/`min_description`/`max_description`), `model_name`, `instructions` (Jinja2), `tools` (e.g. Web Search, Code Interpreter). Conversation history, user message, assistant response, available tools auto-injected. Jinja2 vars: `{{ conversation_history }}`, `{{ user_message }}`, `{{ assistant_message }}`, `{{ system_prompt }}`, `{{ available_tools }}`, `{{ answer_type_definition }}`, `{{ properties.* }}`. Validate on 10–20 real records before scaling.
- **Grader** (OpenAI): scores traces with structured criteria (did the agent pick the right tool? did a handoff happen when it should? did the workflow violate a policy?). Created from Logs > Traces.
- **Campaign** (Mistral): batch-annotate traffic by running one Judge over a filtered set; `max_nb_events` 100–10,000; background; annotations written back into Explorer; linked to original events. SDK: `campaigns.create(name, description, judge_id, search_params, max_nb_events)`, `fetch_status()`, `list_events()`. Filter cannot change after start.
- **Eval run** (OpenAI): over a dataset; for advanced features use the Evals API (deprecated: read-only Oct 31 2026, shutdown Nov 30 2026).
- **Evaluator types** (OpenAI best practices): metric-based (exact match, ROUGE/BLEU, function-call accuracy, executable evals); human (labelers rank/grade 1–5, consensus votes, "show rather than tell"); LLM-as-a-judge / model graders (pairwise comparison, single-answer grading, reference-guided grading; watch position bias, verbosity bias; prefer pairwise or pass/fail; use most capable model; add CoT before scoring).
- **Architecture-aware evals** (OpenAI): single-turn → instruction following + functional correctness; workflows → per-step + end-to-end; single-agent → add **tool selection** + **data precision** (correct arguments extracted from history); multi-agent → add **agent handoff accuracy** (triage decision boundaries).

### API surface (union)

**Mistral Judge** (`mistral.beta.observability.judges.create`)

| Parameter | Type | Description |
|---|---|---|
| `name` | string | Judge name. |
| `description` | string | Judge description. |
| `model_name` | string | Evaluation model (`mistral-medium-latest`, `mistral-small-latest`, ...). |
| `instructions` | string | `# Instructions` block; may include Jinja2. |
| `output` | object | CLASSIFICATION `{type, options:[{value,description}]}` or REGRESSION `{type, min, max, min_description, max_description}`. |
| `tools` | array | Optional (Web Search, Code Interpreter); `[]` for none. |

**Mistral Campaign** (`mistral.beta.observability.campaigns.create`)

| Parameter | Type | Description |
|---|---|---|
| `name` | string | Campaign name. |
| `description` | string | Campaign description. |
| `judge_id` | string | Judge to apply. |
| `search_params` | object | `{filters: <AND/OR tree>}`. |
| `max_nb_events` | int | 100–10,000. |

**OpenAI trace-grading workflow**: Logs > Traces → inspect trace → create grader → run against traces → refine prompts/tools/routing/guardrails.

### Notes / constraints

- A Judge uses a single output type; classification options each need `value` + `description`.
- Be specific in instructions; never assume the Judge understands your context; use boundary examples.
- Campaign runs async; close the tab and check the dashboard later. Deleting a Campaign does not necessarily lose annotations.
- LLM-as-judge quality varies by problem context; expert human ground-truth labels are expensive/slow.
- No strategy is perfect — combine evaluator types.

---

## Stage K — Datasets, Curation & Re-runs

**Purpose** — Build repeatable, comparable baselines from production traffic, manual entry, or file uploads; re-run them to benchmark changes.

### Concepts

- **Dataset** (Mistral): curated, editable collections; record = **Conversation + Properties + Source**. Sources: `EXPLORER`, `UPLOADED_FILE`, `DIRECT_INPUT`, `PLAYGROUND`. Properties: structured metadata (`expected_output`, `category`, `grading_guidance`, `difficulty`) referenced by Judges via `{{ properties.* }}`. JSONL import/export. Editable — fix messages, add expected outputs, remove duplicates.
- **Dataset** (Google): curated logs; no expiry; ≤1,000 logs/project; export to CSV/JSONL/Google Sheets; re-run with **Batch API**. Use cases: challenge sets, sample sets, evaluation sets.
- **Dataset** (OpenAI): golden set for eval runs; expected outputs / labels.
- **Anthropic**: no dedicated dataset surface (manual curation).
- **Best practices**: explicit names with scope/date (e.g. `support_billing_baseline_2025_06`); track sources/curation; version baselines; don't mix unrelated tasks; check class balance; freeze baseline between uses.

### API surface (union)

**Mistral Dataset** (`mistral.beta.observability.datasets.create`)

| Parameter | Type | Description |
|---|---|---|
| `name` | string | Dataset name. |
| `description` | string | Dataset description. |

Related: add records, `import_from_explorer()`, list records, export to JSONL.

**Mistral JSONL import format** (one record per line):
```json
{"messages": [{"role":"user","content":"..."},{"role":"assistant","content":"..."}], "properties": {"expected_output":"...","category":"..."}}
```

**Mistral record fields**: Conversation (system messages, user inputs, assistant responses, tool calls); Properties (custom metadata); Source (`EXPLORER`/`UPLOADED_FILE`/`DIRECT_INPUT`/`PLAYGROUND`).

**Google dataset operations**: filter logs → Create dataset → name + description → export CSV/JSONL/Sheets → re-run with Batch API.

### Notes / constraints

- Mistral imports may take time; check Import Tasks status.
- Campaign → Dataset flow: `campaigns.list_events()` then `datasets.import_from_explorer()`.
- Validate a Judge on a single record before launching a full Campaign (fastest feedback loop).

---

## Stage L — Production Monitoring & the Continuous Improvement Loop

**Purpose** — Close the loop: monitor in production, detect regressions, improve prompts/routing/guardrails, optionally fine-tune.

### Concepts

- **Iterative cycle** (Google guidance): understand risks → adjust/test (repeat until performance appropriate) → monitor in production.
- **Two kinds of testing** (Google): **safety benchmarking** (design safety metrics, test against eval datasets, define minimum acceptable levels) and **adversarial testing** (proactively try to break the app; diverse test data; **automated red-team LLM** to find inputs eliciting harmful outputs).
- **Monitoring** (Google): monitored user-feedback channel (thumbs up/down); user studies with diverse users; especially when usage patterns differ from expectations.
- **Mitigation techniques** (Google, mostly application-side): blocklists; trained classifiers labeling prompts for harms/adversarial signals; unique user IDs + per-user volume limits; prompt-injection safeguards; scope-narrowing (narrower tasks, more human oversight); adjust safety settings; **Grounding with Google Search** for factuality (verifiable citations beyond knowledge cutoff; disable for creative non-information-seeking use cases).
- **Eval-driven development** (OpenAI best practices): evaluate early and often; log everything to mine for eval cases; automate scoring; calibrate with human feedback; treat evaluation as a continuous journey.
- **Continuous evaluation (CE)** (OpenAI): run evals on every change; monitor for new nondeterminism; grow the eval set over time.
- **Data flywheel** (OpenAI): once evals mature, feed eval data into **reinforcement fine-tuning** to improve the application.
- **Moderation analysis** (Anthropic): analyze moderated content to identify trends; continuously evaluate with precision/recall tracking; iteratively refine prompts, keywords, criteria.
- **Mistral Observability loop**: Explorer → Judges → Campaigns → Explorer (filter by annotations) → Datasets → (re-run / improve).

### Notes / constraints

- Subject to platform Prohibited Use Policies and Terms of Service.
- Built-in safety filtering + this guidance are complementary: API provides adjustable filters; developer owns application-level mitigations, testing, and production monitoring.
- Logs/datasets provide the observability substrate for production monitoring.

---

## Cross-cutting concerns

### 17.1 Sensitive-content controls (privacy gates)

Telemetry is structural by default (durations, model names, tool names; token counts when API returns usage). Content read/written by the agent is **not** recorded by default. Opt-in flags (Anthropic):

| Variable | Adds |
|---|---|
| `OTEL_LOG_USER_PROMPTS=1` | Prompt text on `claude_code.user_prompt` events and `claude_code.interaction` span. |
| `OTEL_LOG_ASSISTANT_RESPONSES=1` | Assistant response text on `assistant_response` events (falls back to `OTEL_LOG_USER_PROMPTS`; v2.1.193+). |
| `OTEL_LOG_TOOL_DETAILS=1` | Tool input args (file paths, shell commands, search patterns, skill names, MCP names) on `tool_result` events / span attrs. |
| `OTEL_LOG_TOOL_CONTENT=1` | Full tool input/output bodies as span events on `claude_code.tool` (truncated 60 KB). Requires tracing. |
| `OTEL_LOG_RAW_API_BODIES` | Full Messages API request/response JSON as `api_request_body`/`api_response_body` log events. `1`=inline truncated 60 KB; `file:<dir>`=untruncated on disk with `body_ref` path. Implies consent to everything the three flags above reveal. Extended-thinking content redacted. |

Leave unset unless your observability pipeline is approved to store the data the agent handles.

### 17.2 Data-use & sharing policy (Google)

- Logging is opt-in and developer-owned; available only for billing-enabled projects.
- Default: prompts/responses within logs **not** used for product improvement/development.
- Sharing a dataset with Google opts it into "Unpaid Services" data-use terms (may be used for training/evaluation, including human review).
- Google disconnects data from Account/API key/Cloud project **before** reviewers see/annotate it.
- Do not contribute logs containing sensitive, confidential, proprietary, or personal information.
- License extends to prompts (incl. system instructions, cached content, files) and generated responses.

### 17.3 Enterprise gating

- Mistral: entire Observability suite (Explorer, Judges, Campaigns, Datasets) is **Enterprise-tier only**; SDK under `mistral.beta.observability.*`. Moderation API and Custom Guardrails available to standard users.
- Google: logs storage requires **paid tier** (billing-enabled project); safety settings/feedback available to standard users.
- Anthropic: tracing beta may require **org allowlist** in interactive CLI (not gated in Agent SDK / `-p` sessions).
- OpenAI: tracing built-in and enabled by default; legacy Evals platform being deprecated (read-only Oct 31 2026; shutdown Nov 30 2026).

### 17.4 W3C trace-context propagation (Anthropic)

See Stage E.5. Cross-system correlation is also available via Mistral `correlation_id`.

### 17.5 Cost model caveats

- In-process cost estimates (Anthropic `total_cost_usd`/`costUSD`) are **not authoritative billing** — drift with pricing changes, unknown models, unmodelable rules. Do not bill end users or trigger financial decisions from these.
- Use the platform's Usage/Cost API or console for authoritative billing.
- Anthropic cache: 5-min default TTL (API key / Bedrock / GCP Agent Platform / Foundry); 1-hour with `ENABLE_PROMPT_CACHING_1H=1` (subscription users already get 1h; 1-hour writes billed at higher rate).

### 17.6 Moderation model lifecycle (Mistral)

`mistral-moderation-2603` is current; `mistral-moderation-2411` deprecated March 31, 2026. The model is continuously improved; custom pipelines keyed on `category_scores` may require recalibration.

### 17.7 Anthropic extended thinking & structured output

- `thinking` config: `enabled`/`disabled`/`adaptive` with `budget_tokens` ≥ 1024; `display` `summarized`/`omitted`.
- `output_config.effort`: `low`/`medium`/`high`/`xhigh`/`max` (reasoning effort).
- `output_config.format`: `JSONOutputFormat` (json_schema) — usable to enforce moderation verdict shape.
- `service_tier`: `auto`/`standard_only`.
- `cache_control`: top-level ephemeral breakpoint; per-block `cache_control` with `ttl` `5m`/`1h`.
- Extended-thinking content is **redacted** from `OTEL_LOG_RAW_API_BODIES`.

---

## Per-platform capability matrix

| Capability | Anthropic | Google | Mistral | OpenAI |
|---|---|---|---|---|
| Identity & resource attribution | ✓ (OTel resource attrs, gateway OIDC) | ✓ (`metadata.user_id`) | ✓ (`metadata`, `api_agent_id`) | partial (run context) |
| Bring-your-own OTLP backend | ✓ | — | — | — |
| Hosted dashboard | — | ✓ (AI Studio Logs) | ✓ (Explorer, Enterprise) | ✓ (Traces) |
| Per-request store toggle | — (telemetry enable) | ✓ (`store`) | — | — (built-in) |
| Metrics | ✓ (`claude_code.*` counters) | ✓ (`usageMetadata`) | ✓ (event fields) | ✓ (trace attrs) |
| Log events | ✓ (`claude_code.*`) | ✓ (log entries) | ✓ (chat completion events) | ✓ (trace steps) |
| Traces/spans | ✓ (beta, OTel) | — (logs are unit) | — (events) | ✓ (built-in) |
| Span hierarchy with subagents | ✓ (nested) | — | — | ✓ (handoffs) |
| W3C trace-context propagation | ✓ | — | ✓ (`correlation_id`) | — |
| In-process cost estimate | ✓ (`total_cost_usd`) | ✓ (`usageMetadata` tokens) | ✓ (event fields) | partial |
| Cache token tracking | ✓ (5m/1h TTL) | — | — | — |
| Input moderation (dedicated model) | — (prompt pattern) | ✓ (built-in safety) | ✓ (`mistral-moderation-2603`) | — (custom agent) |
| Input guardrails (declarative) | — | ✓ (`safetySettings`) | ✓ (`moderation_llm_v2`) | ✓ (`input_guardrails`) |
| Jailbreaking / prompt-injection category | — (custom prompt) | — (guidance) | ✓ (dedicated category) | — (custom) |
| Output moderation | ✓ (prompt pattern) | ✓ (safety feedback) | — (input-only) | ✓ (`output_guardrails`) |
| Tool guardrails | partial (hooks) | — | — | ✓ (attached to tool) |
| Human-in-the-loop approvals | partial (permission events) | — (guidance) | — | ✓ (resumable `state`) |
| MCP integrations | — | — | — | ✓ (hosted + SDK-managed) |
| Hooks (lifecycle middleware) | ✓ | — | — | partial (guardrails) |
| Search/filter traffic | ✓ (OTel backend) | ✓ (Logs page filter) | ✓ (Explorer filter lang, Enterprise) | ✓ (Traces dashboard) |
| Judge/Grader (LLM-as-a-judge) | — (classification cookbook) | — (guidance) | ✓ (Judge, Enterprise) | ✓ (Grader) |
| Batch scoring run | — | ✓ (Batch API) | ✓ (Campaign, Enterprise) | ✓ (Eval run; Evals API deprecated) |
| Datasets | — (manual) | ✓ (no expiry, ≤1000/proj) | ✓ (props + source, JSONL, Enterprise) | ✓ (golden set) |
| Dataset re-run | — | ✓ (Batch API) | — (export only) | ✓ (eval runs) |
| Safety benchmarking / adversarial testing | — (precision/recall) | ✓ (guidance, automated red-team) | — | ✓ (best practices) |
| Grounding for factuality | — | ✓ (Google Search) | — | — |
| Continuous evaluation / fine-tuning flywheel | — (refine prompts) | ✓ (iterative cycle) | ✓ (Observability loop) | ✓ (CE, reinforcement FT) |
| Sensitive-content gating | ✓ (opt-in flags) | ✓ (data-use policy) | ✓ (Enterprise) | ✓ (scope controls) |
| mTLS / dynamic headers | ✓ | — | — | — |

---

## Appendix — Reference category lists & thresholds

### Mistral moderation categories (11)
Sexual; Hate and Discrimination; Violence and Threats; Dangerous; Criminal; Self-Harm; Health; Financial; Law; PII; Jailbreaking.

### Google adjustable safety categories (4) + built-in
Adjustable: Harassment; Hate speech; Sexually explicit; Dangerous.
Built-in non-adjustable: child-safety endangering content; `PROHIBITED_CONTENT`.

### Google HarmBlockThreshold enum
`OFF`; `BLOCK_NONE`; `BLOCK_ONLY_HIGH`; `BLOCK_MEDIUM_AND_ABOVE`; `BLOCK_LOW_AND_ABOVE`; `HARM_BLOCK_THRESHOLD_UNSPECIFIED`.

### Google HarmProbability enum
`HIGH`; `MEDIUM`; `LOW`; `NEGLIGIBLE`.

### Google BlockReason enum (prompt-level)
`BLOCK_REASON_UNSPECIFIED`; `SAFETY`; `OTHER`; `BLOCKLIST`; `PROHIBITED_CONTENT`; `IMAGE_SAFETY`.

### Anthropic moderation category list (customizable reference)
Child Exploitation; Conspiracy Theories; Hate; Indiscriminate Weapons; Intellectual Property; Non-Violent Crimes; Privacy; Self-Harm; Sex Crimes; Sexual Content; Specialized Advice; Violent Crimes.

### Anthropic risk scale (multi-level pattern)
0 = No risk; 1 = Low; 2 = Medium; 3 = High.

---

*End of unified specification. This document is the union of the four platform studies in this folder; every capability, parameter, field, category, threshold, and concept found in the source files is represented above. Re-check any item against its source file for verbatim API details.*