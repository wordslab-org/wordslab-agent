# Anthropic API — Observability & Moderation Agent Capabilities

Analysis of the agent-related capabilities offered by the Anthropic / Claude API, based on the official docs for **Observability** (`code.claude.com/docs/en/agent-sdk/observability`, `code.claude.com/docs/en/monitoring-usage`, `code.claude.com/docs/en/agent-sdk/cost-tracking`) and **Content Moderation** (`platform.claude.com/docs/en/about-claude/use-case-guides/content-moderation`, `platform.claude.com/docs/en/api/messages`). Each capability is broken down into main concepts, API surface (functions, parameters, fields), and notes/constraints.

Scope of this study: capabilities relevant to **agents** — i.e. observability of agentic runs produced by the Claude Agent SDK / Claude Code CLI (which runs the model + tools + hooks loop), and content moderation of user-generated or agent-facing content via the Messages API.

Two distinct surfaces are covered:

- **Observability** is built on **OpenTelemetry**. Anthropic does not expose a dedicated observability REST API; instead, the Agent SDK / Claude Code CLI emits OTel signals (metrics, logs/events, traces) that you export to any OTLP-compatible backend. A complementary in-process surface lets you read cost/usage from the SDK message stream.
- **Content Moderation** is **not** a dedicated moderation model/endpoint. It is implemented as a prompting pattern on top of the standard **Messages API** (`POST /v1/messages`), where Claude itself classifies content against a list of unsafe categories you define.

---

## Table of Contents

1. [Agent Observability — OpenTelemetry export](#1-agent-observability--opentelemetry-export)
   - 1.1 [Enabling telemetry](#11-enabling-telemetry)
   - 1.2 [Three signals (metrics, logs, traces)](#12-three-signals-metrics-logs-traces)
   - 1.3 [OTLP exporter & transport configuration](#13-otlp-exporter--transport-configuration)
   - 1.4 [Export intervals & flush behavior](#14-export-intervals--flush-behavior)
   - 1.5 [Standard attributes](#15-standard-attributes)
   - 1.6 [Metrics reference](#16-metrics-reference)
   - 1.7 [Log events reference](#17-log-events-reference)
   - 1.8 [Traces / span hierarchy & attributes](#18-traces--span-hierarchy--attributes)
   - 1.9 [W3C trace-context propagation](#19-w3c-trace-context-propagation)
   - 1.10 [Resource attributes, tagging & multi-team](#110-resource-attributes-tagging--multi-team)
   - 1.11 [End-user attribution & audit events](#111-end-user-attribution--audit-events)
   - 1.12 [Sensitive-content controls](#112-sensitive-content-controls)
   - 1.13 [mTLS & dynamic headers](#113-mtls--dynamic-headers)
2. [Cost & Usage tracking (in-process)](#2-cost--usage-tracking-in-process)
3. [Content Moderation via the Messages API](#3-content-moderation-via-the-messages-api)
   - 3.1 [Messages API endpoint & parameters](#31-messages-api-endpoint--parameters)
   - 3.2 [Binary moderation pattern](#32-binary-moderation-pattern)
   - 3.3 [Multi-level risk assessment pattern](#33-multi-level-risk-assessment-pattern)
   - 3.4 [Category definitions pattern](#34-category-definitions-pattern)
   - 3.5 [Batch processing pattern](#35-batch-processing-pattern)
   - 3.6 [Operational considerations](#36-operational-considerations)

---

## 1. Agent Observability — OpenTelemetry export

**Summary** — The Claude Agent SDK and Claude Code CLI emit OpenTelemetry-compatible telemetry (metrics, log events, and distributed traces) describing agent runs: which tools they called, model-request latency and tokens, where failures occurred, and hook executions. Telemetry is off by default and configured entirely through **environment variables**; the CLI exports directly to any OTLP-compatible collector (Honeycomb, Datadog, Grafana, Langfuse, self-hosted collectors, etc.). There is no Anthropic-hosted observability API to call — you own the collector and backend.

The SDK itself produces no telemetry; it runs the Claude Code CLI as a child process, passes telemetry configuration through, and the CLI instruments and exports. Configuration can be supplied at the **process level** (shell/container/orchestrator env — recommended for production) or **per-call** via `ClaudeAgentOptions.env` (Python) / `options.env` (TypeScript).

### 1.1 Enabling telemetry

| Variable | Required | Description |
|---|---|---|
| `CLAUDE_CODE_ENABLE_TELEMETRY` | yes | Master switch. Must be `1` for any signal to be emitted. |
| `CLAUDE_CODE_ENHANCED_TELEMETRY_BETA` | for traces only | `1` enables span tracing (beta). `ENABLE_ENHANCED_TELEMETRY_BETA` is also accepted. Metrics and log events do not need this. |

Telemetry is off until `CLAUDE_CODE_ENABLE_TELEMETRY=1` is set and at least one exporter is configured. The Agent SDK does not have its own telemetry; it forwards config to the CLI child process.

> Note: `console` exporter is not usable through the SDK because the SDK uses stdout as its message channel. Use a local OTLP collector or Jaeger container for local inspection.

### 1.2 Three signals (metrics, logs, traces)

The CLI exports three independent OTel signals, each with its own enable switch and exporter:

| Signal | Contents | Enable with |
|---|---|---|
| Metrics | Counters for tokens, cost, sessions, lines of code, tool decisions | `OTEL_METRICS_EXPORTER` |
| Log events | Structured records for each prompt, API request, API error, tool result, security-relevant audit events | `OTEL_LOGS_EXPORTER` |
| Traces (beta) | Spans for interaction, model request, tool call, hook execution | `OTEL_TRACES_EXPORTER` + `CLAUDE_CODE_ENHANCED_TELEMETRY_BETA=1` |

Exporter values: `console`, `otlp`, `prometheus` (metrics only), `none`. Comma-separated for multiple exporters.

### 1.3 OTLP exporter & transport configuration

| Variable | Description | Example values |
|---|---|---|
| `OTEL_EXPORTER_OTLP_PROTOCOL` | Protocol for all signals | `grpc`, `http/json`, `http/protobuf` |
| `OTEL_EXPORTER_OTLP_ENDPOINT` | Collector endpoint for all signals | `http://localhost:4317` |
| `OTEL_EXPORTER_OTLP_HEADERS` | Auth headers | `Authorization=Bearer token` |
| `OTEL_EXPORTER_OTLP_METRICS_PROTOCOL` / `_ENDPOINT` | Per-signal overrides (metrics) | overrides general |
| `OTEL_EXPORTER_OTLP_LOGS_PROTOCOL` / `_ENDPOINT` | Per-signal overrides (logs) | overrides general |
| `OTEL_EXPORTER_OTLP_TRACES_PROTOCOL` / `_ENDPOINT` | Per-signal overrides (traces) | overrides general |
| `OTEL_EXPORTER_OTLP_METRICS_TEMPORALITY_PREFERENCE` | Metrics temporality | `delta` (default), `cumulative` |

Per-signal endpoint suffixes: gRPC typically `:4317`; HTTP paths are typically `/v1/metrics`, `/v1/logs`, `/v1/traces` on `:4318`.

### 1.4 Export intervals & flush behavior

The CLI batches telemetry and exports on an interval. On clean exit it flushes pending data within a short bounded timeout; spans can be dropped if the collector is slow, and anything in the batch buffer is lost if the process is killed. Lowering intervals reduces both windows.

| Variable | Default | Description |
|---|---|---|
| `OTEL_METRIC_EXPORT_INTERVAL` | 60000 ms | Metrics batch export interval |
| `OTEL_LOGS_EXPORT_INTERVAL` | 5000 ms | Logs/events batch export interval |
| `OTEL_TRACES_EXPORT_INTERVAL` | 5000 ms | Span batch export interval |

For short-lived `query()` calls, set all three to ~1000 ms so data reaches the collector while the task is still running.

### 1.5 Standard attributes

All metrics and events share these standard attributes (controlled for cardinality via flags):

| Attribute | Controlled by (default) | Description |
|---|---|---|
| `session.id` | `OTEL_METRICS_INCLUDE_SESSION_ID` (true) | Unique session identifier |
| `app.version` | `OTEL_METRICS_INCLUDE_VERSION` (false) | Claude Code version |
| `app.entrypoint` | `OTEL_METRICS_INCLUDE_ENTRYPOINT` (false) | Launch mode: `cli`, `sdk-cli`, `sdk-ts`, `sdk-py`, `claude-vscode` |
| `organization.id` | always when available | Org UUID |
| `user.account_uuid` / `user.account_id` | `OTEL_METRICS_INCLUDE_ACCOUNT_UUID` (true) | Authenticated account identity |
| `user.id` | always | Anonymous persisted random id from `~/.claude.json` |
| `user.email` | always when OAuth | OAuth user email |
| `terminal.type` | always when detected | Terminal type |
| keys from `OTEL_RESOURCE_ATTRIBUTES` | `OTEL_METRICS_INCLUDE_RESOURCE_ATTRIBUTES` (true) | Custom attributes |

Events additionally carry high-cardinality-only attributes that are never attached to metrics:
- `prompt.id` — UUID correlating a user prompt with subsequent events until next prompt.
- `workspace.host_paths` — host workspace dirs from desktop app.
- `workflow.run_id` (`wf_`-prefixed) and `workflow.name` — for agents spawned by a Workflow tool run (Claude Code v2.1.202+).

When signed in to a Claude apps gateway, identity is stamped from the gateway OIDC session (`user.id` = IdP subject, `user.email`, `user.groups`, `identity.source: gateway-oidc`); gateway identity is applied last and overrides `user.*`/`identity.*` from `OTEL_RESOURCE_ATTRIBUTES`.

### 1.6 Metrics reference

| Metric name | Unit | Description |
|---|---|---|
| `claude_code.session.count` | count | CLI sessions started |
| `claude_code.lines_of_code.count` | count | Lines of code modified (attr `type`: `added`/`removed`) |
| `claude_code.pull_request.count` | count | PRs created |
| `claude_code.commit.count` | count | Git commits created |
| `claude_code.cost.usage` | USD | Cost of the session |
| `claude_code.token.usage` | tokens | Tokens used |
| `claude_code.code_edit_tool.decision` | count | Code-editing tool permission decisions |
| `claude_code.active_time.total` | s | Total active time |

Session counter adds `start_type` (`fresh`, `resume`, `continue`, `agents_view`).

### 1.7 Log events reference

Log events are structured records exported via the OTel logs protocol, named with a `claude_code.` prefix. Categories include per-prompt events, per-API-request/error events, per-tool events (`tool_result`, `tool_decision`), MCP events (`mcp_server_connection`), `permission_mode_changed`, and `assistant_response`. Several content-bearing events are gated by content-logging flags (see [§1.12](#112-sensitive-content-controls)). Security-relevant events (`tool_decision`, `tool_result`, `mcp_server_connection`, `permission_mode_changed`) form an audit trail when end-user identity is attached (see [§1.11](#111-end-user-attribution--audit-events)).

### 1.8 Traces / span hierarchy & attributes

Tracing is beta. With `CLAUDE_CODE_ENHANCED_TELEMETRY_BETA=1` and `OTEL_TRACES_EXPORTER` set, each step of the agent loop becomes a span. Span hierarchy:

```
claude_code.interaction                 (one turn: prompt -> response)
├── claude_code.llm_request             (each Claude API call)
├── claude_code.hook                    (each hook execution; needs detailed beta tracing)
└── claude_code.tool                    (each tool invocation)
    ├── claude_code.tool.blocked_on_user   (permission wait)
    ├── claude_code.tool.execution          (execution itself)
    └── (Agent tool) subagent claude_code.llm_request / claude_code.tool spans
```

When the Agent/Task tool spawns a subagent, the subagent's spans nest under the parent's `claude_code.tool` span, so the full delegation chain is one trace. Spans carry a `session.id` attribute by default; filter on it in your backend to see related `query()` calls as one timeline.

**`claude_code.interaction` attributes**

| Attribute | Description | Gated by |
|---|---|---|
| `user_prompt` | Prompt text (`<REDACTED>` unless gate set) | `OTEL_LOG_USER_PROMPTS` |
| `user_prompt_length` | Prompt length in chars | — |
| `interaction.sequence` | 1-based interaction counter in session | — |
| `interaction.duration_ms` | Wall-clock turn duration | — |

**`claude_code.llm_request` attributes**

| Attribute | Description |
|---|---|
| `model`, `gen_ai.system` (`anthropic`), `gen_ai.request.model` | Model identification (GenAI semantic convention) |
| `query_source`, `agent_id`, `parent_agent_id`, `workflow.run_id`, `workflow.name` | Source/subagent/workflow attribution |
| `speed` | `fast` or `normal` |
| `llm_request.context` | `interaction`, `tool`, or `standalone` |
| `duration_ms`, `ttft_ms` | Total latency, time to first token |
| `input_tokens`, `output_tokens`, `cache_read_tokens`, `cache_creation_tokens` | Token counts from API usage block |
| `request_id`, `gen_ai.response.id`, `client_request_id` | Request identifiers (Anthropic `request-id` header) |
| `attempt`, `success`, `status_code`, `error` | Retry/failure info |
| `response.has_tool_call`, `stop_reason`, `gen_ai.response.finish_reasons` | Response characteristics |

Each retry is also a `gen_ai.request.attempt` span event. `llm_request`, `tool.execution`, and `hook` spans set OTel status `ERROR` on failure; others end `UNSET`.

**`claude_code.tool` attributes**

| Attribute | Description | Gated by |
|---|---|---|
| `tool_name`, `duration_ms`, `result_tokens` | Tool identity, duration (incl. permission wait), result size | — |
| `tool_use_id`, `gen_ai.tool.call.id` | Model's tool_use block id (joins to tool_result/tool_decision events) | — |
| `agent_id`, `parent_agent_id`, `workflow.run_id`, `workflow.name` | Delegation/workflow attribution (`workflow.name` gated) | `OTEL_LOG_TOOL_DETAILS` for `workflow.name` |
| `file_path`, `full_command`, `skill_name`, `subagent_type` | Tool-specific detail | `OTEL_LOG_TOOL_DETAILS` |

With `OTEL_LOG_TOOL_CONTENT=1`, a `tool.output` span event records tool input/output bodies truncated at 60 KB.

**`claude_code.tool.blocked_on_user` attributes**: `duration_ms`, `decision` (`accept`/`reject`), `source`.

**`claude_code.tool.execution` attributes**: `duration_ms`, `tool_use_id`, `gen_ai.tool.call.id`, `success`, `error` (full message gated by `OTEL_LOG_TOOL_DETAILS`).

**`claude_code.hook` span** — emitted only with detailed beta tracing (`ENABLE_BETA_TRACING_DETAILED=1` + `BETA_TRACING_ENDPOINT`); not gated in Agent SDK / `-p` sessions, but requires org allowlist in interactive CLI. Attributes: `hook_event`, `hook_name`, `num_hooks`, `hook_definitions` (gated), `duration_ms`, `num_success`, `num_blocking`, `num_non_blocking_error`, `num_cancelled`.

### 1.9 W3C trace-context propagation

The SDK auto-propagates W3C trace context into the CLI subprocess. When you call `query()` while an OTel span is active in your application, the SDK injects `TRACEPARENT`/`TRACESTATE` into the child process environment, and the CLI's `claude_code.interaction` span becomes a child of your span — the agent run appears inside your application's trace rather than as a disconnected root.

- When tracing is active, Bash/PowerShell subprocesses inherit a `TRACEPARENT` env var referencing the active tool-execution span, so subprocess-emitted spans nest under `claude_code.tool.execution`.
- When connected directly to the Anthropic API, each model request carries a W3C `traceparent` header (the `claude_code.llm_request` span context); the API's `traceresponse` header is recorded as a span link, connecting client-side to server-side traces. Same for outbound HTTP MCP requests. The header is not sent to third-party providers.
- Auto-injection is skipped if you set `TRACEPARENT` explicitly in `options.env` (lets you pin a specific parent context). Interactive CLI sessions ignore inbound `TRACEPARENT`; only Agent SDK and `claude -p` runs honor it.
- When running through a custom `ANTHROPIC_BASE_URL` proxy, `traceparent` and the subprocess `TRACEPARENT` are suppressed by default (some proxies reject unknown headers). Set `CLAUDE_CODE_PROPAGATE_TRACEPARENT=1` to enable propagation through the proxy.

### 1.10 Resource attributes, tagging & multi-team

By default `service.name` is `claude-code`. Override and add deployment metadata with `OTEL_SERVICE_NAME` and `OTEL_RESOURCE_ATTRIBUTES`. These are applied as OTel resource attributes on every span, metric, and event.

| Variable | Description |
|---|---|
| `OTEL_SERVICE_NAME` | Override the service name (e.g. `support-triage-agent`) |
| `OTEL_RESOURCE_ATTRIBUTES` | Comma-separated `key=value` pairs applied to all signals |

`OTEL_RESOURCE_ATTRIBUTES` formatting constraints:
- No spaces; values can't contain whitespace, double quotes, commas, semicolons, backslashes, or control chars.
- Use underscores/camelCase, or percent-encode special chars (you can percent-encode any char).
- Custom keys never override built-in standard attributes (`user.id`, `session.id`, etc.) on collision — the built-in value wins.
- Custom keys become labels on every metric series, so high-cardinality values increase storage cost. Set `OTEL_METRICS_INCLUDE_RESOURCE_ATTRIBUTES=false` to send custom attributes in the OTLP resource block only and omit them from datapoint labels.

### 1.11 End-user attribution & audit events

The CLI attaches identity attributes based on the credential it uses to call Anthropic — identifying the *service's* credential, not the end user the agent acts on behalf of. To attribute tool calls and MCP activity to your application's end users, inject end-user identity as resource attributes per `query()` call (percent-encode values before interpolating into `OTEL_RESOURCE_ATTRIBUTES`):

```python
from urllib.parse import quote
options = ClaudeAgentOptions(env={
    # ... exporter config ...
    "OTEL_RESOURCE_ATTRIBUTES": f"enduser.id={quote(request.user_id)},tenant.id={quote(request.tenant_id)}",
})
```

With end-user identity attached, the `tool_decision`, `tool_result`, `mcp_server_connection`, and `permission_mode_changed` log events become a per-user audit trail forwardable to a SIEM platform.

### 1.12 Sensitive-content controls

Telemetry is structural by default — durations, model names, tool names recorded; token counts recorded when the API returns usage. Content read/written by the agent is **not** recorded by default. These opt-in flags add content:

| Variable | Adds |
|---|---|
| `OTEL_LOG_USER_PROMPTS=1` | Prompt text on `claude_code.user_prompt` events and the `claude_code.interaction` span |
| `OTEL_LOG_ASSISTANT_RESPONSES=1` | Assistant response text on `assistant_response` events (falls back to `OTEL_LOG_USER_PROMPTS` value; v2.1.193+) |
| `OTEL_LOG_TOOL_DETAILS=1` | Tool input arguments (file paths, shell commands, search patterns, skill names, MCP names) on `tool_result` events / span attributes |
| `OTEL_LOG_TOOL_CONTENT=1` | Full tool input/output bodies as span events on `claude_code.tool` (truncated 60 KB). Requires tracing. |
| `OTEL_LOG_RAW_API_BODIES` | Full Messages API request/response JSON as `api_request_body` / `api_response_body` log events. `1` = inline truncated 60 KB; `file:<dir>` = untruncated on disk with `body_ref` path. Implies consent to everything the three flags above would reveal. Extended-thinking content is redacted. |

Leave these unset unless your observability pipeline is approved to store the data the agent handles.

### 1.13 mTLS & dynamic headers

mTLS client certificate config depends on the OTLP protocol for the signal:

| Protocol | Client cert variables | Trust collector CA with |
|---|---|---|
| `http/protobuf`, `http/json` | `CLAUDE_CODE_CLIENT_CERT`, `CLAUDE_CODE_CLIENT_KEY`, optional `CLAUDE_CODE_CLIENT_KEY_PASSPHRASE` | `NODE_EXTRA_CA_CERTS` |
| `grpc` | `OTEL_EXPORTER_OTLP_CLIENT_KEY` + `OTEL_EXPORTER_OTLP_CLIENT_CERTIFICATE` (or per-signal variants) | `OTEL_EXPORTER_OTLP_CERTIFICATE` |

**Dynamic headers** (for enterprise envs requiring rotating tokens) apply only to `http/protobuf` and `http/json`; `grpc` uses static `OTEL_EXPORTER_OTLP_HEADERS`. Configure in `.claude/settings.json`:

```json
{ "otelHeadersHelper": "/bin/generate_opentelemetry_headers.sh" }
```

The script must output valid JSON of string key-value HTTP headers. It runs at startup and every 29 min by default (customize with `CLAUDE_CODE_OTEL_HEADERS_HELPER_DEBOUNCE_MS`). Errors surface in `/status`, the debug log, and stderr (in `-p` non-interactive sessions).

Administrators can centrally configure all telemetry via the managed settings file `env` block, distributed via MDM; these have high precedence and cannot be overridden by users.

---

## 2. Cost & Usage tracking (in-process)

**Summary** — For cost/usage without an external backend, the Agent SDK exposes token-usage and cost data directly in the message stream returned by `query()`. This is a **client-side estimate** computed locally from a bundled price table — not authoritative billing. Use the Usage and Cost API or the Claude Console for authoritative billing.

### Main concepts

- **`query()` call** — one invocation of `query()`; can span multiple steps; produces one `result` message at the end.
- **Step** — a single request/response cycle within a `query()` call; each step produces assistant messages with token usage.
- **Session** — a series of `query()` calls linked by a session id (`resume` option); each call reports its own cost independently.
- **Deduplication by message id** — parallel tool calls produce multiple assistant messages sharing the same nested `BetaMessage` `id` with identical usage; deduplicate by id to avoid inflated totals.
- **Cache tokens** — `cache_creation_input_tokens` (charged higher) and `cache_read_input_tokens` (charged lower) tracked separately from `input_tokens`.
- **1-hour cache TTL** — set `ENABLE_PROMPT_CACHING_1H=1` to request 1-hour TTL on cache writes (default 5 min with API key / Bedrock / GCP Agent Platform / Foundry). Subscription users already get 1-hour TTL. 1-hour writes are billed at a higher rate.

### API surface (message stream)

The SDK emits a stream of messages; `query()` is the entry point. Relevant message types/fields (TypeScript names; Python equivalents noted):

| Field (TS) / Field (Python) | On message type | Description |
|---|---|---|
| `message.message.id` / `message_id` | `assistant` | Nested `BetaMessage` id; dedup by this |
| `message.message.usage` / `usage` | `assistant` | Per-step token counts (`input_tokens`, `output_tokens`, `cache_creation_input_tokens`, `cache_read_input_tokens`) |
| `total_cost_usd` | `result` (`SDKResultMessage` / `ResultMessage`) | Cumulative estimated cost across all steps in the call (present on success **and** error results) |
| `usage` (dict) | `result` | Cumulative token counts |
| `modelUsage` / `model_usage` | `result` | Map of model name → per-model token counts and cost (`costUSD`, `inputTokens`, `outputTokens`, `cacheReadInputTokens`, `cacheCreationInputTokens`) |

Total cost of a query (read from the result message):

```python
from claude_agent_sdk import query, ResultMessage
async for message in query(prompt="Summarize this project"):
    if isinstance(message, ResultMessage):
        print(f"Total cost: ${message.total_cost_usd or 0}")
```

Per-step usage with deduplication (TypeScript):

```typescript
const seenIds = new Set<string>();
let totalInputTokens = 0, totalOutputTokens = 0;
for await (const message of query({ prompt: "..." })) {
  if (message.type === "assistant") {
    const id = message.message.id;
    if (!seenIds.has(id)) {
      seenIds.add(id);
      totalInputTokens += message.message.usage.input_tokens;
      totalOutputTokens += message.message.usage.output_tokens;
    }
  }
}
```

### Notes / constraints

- `total_cost_usd` / `costUSD` are **estimates** and drift when pricing changes, the SDK version doesn't recognize a model, or billing rules the client can't model apply. Do **not** bill end users or trigger financial decisions from these fields.
- Each `query()` call returns its own `total_cost_usd`; the SDK provides no session-level total — accumulate totals yourself across calls.
- Always read cost from the result message regardless of its `subtype`; failed conversations still consumed tokens up to the failure point.
- On rare `output_tokens` discrepancies for the same id: use the highest value and prefer the result message's accumulated estimate.

---

## 3. Content Moderation via the Messages API

**Summary** — Anthropic does not offer a dedicated moderation model or moderation endpoint. Content moderation is implemented as a **prompting pattern on the standard Messages API** (`POST /v1/messages`): you send Claude the content to evaluate plus a list of unsafe categories (and optionally definitions), instruct it to respond with a strict JSON verdict, and parse the result. Claude's built-in harmlessness training (per the Acceptable Use Policy) means it may flag content it deems dangerous regardless of your prompt.

### 3.1 Messages API endpoint & parameters

**Endpoint** — `POST /v1/messages` (Anthropic Messages API). Auth via API key; optional `anthropic-user-profile-id` header (requires `user-profiles` beta header) to attribute the request to a party other than your organization.

**Key body parameters** (relevant to moderation use):

| Parameter | Type | Required | Description |
|---|---|---|---|
| `model` | string / Model enum | yes | Model to use. For cost-sensitive moderation at scale, `claude-haiku-4-5` (or `claude-haiku-4-5-20251001`) is recommended. |
| `max_tokens` | number | yes | Max tokens to generate. Moderation uses small values (e.g. `200`); batch processing uses larger values (e.g. `2048`). `0` pre-warms the cache without generating. |
| `messages` | array of `MessageParam` | yes | Conversation turns. For moderation, typically a single `user` message containing the assessment prompt. `content` is a string or array of content blocks (text, image, document, tool_use, etc.). Max 100,000 messages per request. |
| `temperature` | number | no | `0.0`–`1.0`, default `1.0`. Use `0` for moderation consistency. Not fully deterministic even at `0`. |
| `system` | string / array of `TextBlockParam` | no | System prompt (role/goal). There is no `system` role inside `messages`. |
| `stream` | boolean | no | Stream response via SSE. |
| `stop_sequences` | array of string | no | Custom stop strings → `stop_reason: "stop_sequence"`. |
| `metadata.user_id` | string | no | Opaque external user id for abuse detection (no PII). |
| `tools` / `tool_choice` | array / object | no | Tool definitions and use policy (not used in plain moderation, but available for agentic moderation flows). |
| `thinking` | `ThinkingConfigParam` | no | Extended-thinking config (`enabled`/`disabled`/`adaptive` with `budget_tokens` ≥ 1024; `display` `summarized`/`omitted`). |
| `output_config.effort` | `low`/`medium`/`high`/`xhigh`/`max` | no | Reasoning effort. |
| `output_config.format` | `JSONOutputFormat` (json_schema) | no | Structured-output schema — usable to enforce the JSON moderation verdict shape. |
| `service_tier` | `auto` / `standard_only` | no | Priority vs standard capacity. |
| `cache_control` | `CacheControlEphemeral` | no | Top-level cache breakpoint (auto-applies to last cacheable block); per-block `cache_control` with `ttl` `5m`/`1h`. |

**Response** includes `id`, `model`, `content` (array of `TextBlock` / `ToolUseBlock` / `ThinkingBlock` / etc.), `stop_reason` (`end_turn`, `tool_use`, `max_tokens`, `stop_sequence`, `pause_turn`, `refusal`), and a `usage` block (`input_tokens`, `output_tokens`, `cache_creation_input_tokens`, `cache_read_input_tokens`).

### 3.2 Binary moderation pattern

Single-message, single-category-list prompt returning a strict JSON verdict.

**Prompt structure** — wrap the message in `<message>...</message>` and the categories in `<categories>...</categories>`, then instruct: "Respond with ONLY a JSON object" with shape:

```json
{
  "violation": <boolean>,
  "categories": [<comma-separated list of violated categories>],
  "explanation": "<only if there is a violation>"
}
```

**API call**:

```python
client = anthropic.Anthropic()
response = client.messages.create(
    model="claude-haiku-4-5-20251001",
    max_tokens=200,
    temperature=0,
    messages=[{"role": "user", "content": assessment_prompt}],
)
assessment = json.loads(response.content[0].text)
```

Returns `contains_violation` (bool), `violated_categories` (list), `explanation` (optional). Requires nuance — e.g. "killed it" is metaphorical (no violation), while a veiled threat should be flagged.

### 3.3 Multi-level risk assessment pattern

Instead of binary classification, assign a numeric risk level to tune moderation aggressiveness (e.g. auto-block high risk, human-review repeated medium risk).

**Risk scale** in the prompt:
- `0` — No risk
- `1` — Low risk
- `2` — Medium risk
- `3` — High risk

**JSON response shape**:

```json
{
  "risk_level": <number>,
  "categories": [<violated categories>],
  "explanation": "<only if risk level > 0>"
}
```

Example outcomes: a dangerous threat → high risk; a conspiracy claim → medium risk.

### 3.4 Category definitions pattern

Improves precision by pairing each category with a detailed definition in the prompt (a dict `category -> definition` rather than a bare list). Definitions let you scope categories narrowly — e.g. adding that `Specialized Advice` covers investment guidance causes a benign-seeming "great time to invest in gold" post to trigger a violation that the bare-list pattern missed. The prompt includes "It's important that you remember all unsafe categories and their definitions."

### 3.5 Batch processing pattern

Reduces cost when real-time moderation isn't required: include multiple messages in one prompt and ask Claude to assess all of them in a single API call.

- Each message wrapped as `<message id={idx}>{msg}</message>` with a positional id.
- Instruct Claude to return a JSON object listing all violating messages by id:

```json
{
  "violations": [
    { "id": <id>, "categories": [...], "explanation": "..." },
    ...
  ]
}
```

- Remind Claude to analyze every message and select any number of applicable violations.
- Increase `max_tokens` (e.g. `2048`) to accommodate longer batched responses; tune batch size experimentally (larger batches lower cost but may reduce quality).

### 3.6 Operational considerations

**Unsafe categories (reference list)**: Child Exploitation, Conspiracy Theories, Hate, Indiscriminate Weapons, Intellectual Property, Non-Violent Crimes, Privacy, Self-Harm, Sex Crimes, Sexual Content, Specialized Advice, Violent Crimes — customizable (e.g. append "Underage Posting").

**When to prefer Claude over traditional ML / rules-based moderation** (per the guide): cost-effective rapid implementation; semantic understanding with quick decisions; consistent policy application; easily evolvable policies without relabeling; interpretable reasoning/justifications; multilingual support without per-language models; multimodal (text + image) support.

**Model choice & cost estimate** (1bn posts/month, 100 chars/post, 3% flagged, 50 output tokens/flagged message):

| Model | Input cost | Output cost | Monthly total |
|---|---|---|---|
| Claude Haiku 4.5 ($1.00/MTok in, $5.00/MTok out) | $28,600 | $7,500 | $36,100 |
| Claude Opus 4.8 ($5.00/MTok in, $25.00/MTok out) | $143,000 | $37,500 | $180,500 |

Actual costs vary; reduce output tokens further by omitting the `explanation` field.

**Deployment best practices**:
- Provide clear user feedback for blocked/flagged input (via the `explanation` field).
- Analyze moderated content to identify trends and improvement areas.
- Continuously evaluate with precision/recall tracking and iteratively refine prompts, keywords, and criteria. Treat moderation as a classification problem (see the classification cookbook) — or as multi-level risk classes to tune aggressiveness.

**Constraints / caveats**:
- Claude is trained to be honest/helpful/harmless and may moderate content deemed dangerous under the AUP regardless of your prompt (e.g. explicit sexual content may still be flagged even if you instruct otherwise). Review the AUP before building.
- The Messages API is not a hard policy gate; verdicts are model judgments and must be parsed/validated from JSON output (consider `output_config.format` json_schema to enforce shape).
- This guide covers moderating user-generated content *within* your application, not moderating interactions *with* Claude (for that, see the guardrails guide).

---

## Cross-cutting notes

- The two surfaces are complementary: **observability** watches what the agent *does* at runtime (tools, model calls, costs, traces) and exports to OTLP; **content moderation** is a pre-filter pattern on the Messages API that can sit in front of an agent (or moderate its user-generated outputs) using the same model/endpoint.
- Both rely on the Anthropic Messages API as the foundation; observability adds the Claude Code CLI / Agent SDK OTel layer on top, while moderation is purely a prompting discipline on `POST /v1/messages`.