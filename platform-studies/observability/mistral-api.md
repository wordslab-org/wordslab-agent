# Mistral API — Observability & Moderation Agent Capabilities

Analysis of the agent-related capabilities offered by the Mistral Studio API, based on the official docs for **Observability** (`/studio-api/observability`) and **Moderation & Guardrailing** (`/studio-api/safety-moderation`). Each capability is broken down into main concepts, API surface (functions, parameters, response fields), and notes/constraints.

Scope of this study: capabilities that are relevant to **agents** — i.e. agentic (conversational / chat-completion) traffic and agents that can be equipped with guardrails and observed/scored. The Observability suite is the main agent-observability surface (it tracks `chat completion events` produced by agents and lets you score them at scale); the Moderation/Custom Guardrails surface lets you block unsafe/jailbreaking inputs before they reach an agent.

The entire Observability suite (Explorer, Judges, Campaigns, Datasets) is available to **Enterprise-tier organizations** only. The SDK access lives under `mistral.beta.observability.*`.

---

## Table of Contents

1. [Moderation API](#1-moderation-api)
2. [Custom Guardrails](#2-custom-guardrails)
3. [Explorer](#3-explorer)
4. [Judges](#4-judges)
5. [Campaigns](#5-campaigns)
6. [Datasets](#6-datasets)

---

## 1. Moderation API

**Summary** — A dedicated classifier service (`mistral-moderation-2603`) that scores text across 11 policy categories (including `jailbreaking`). Two endpoints: raw-text classification and conversational classification. Used to build custom guardrail pipelines where you keep raw scores and threshold logic in your own code.

### Main concepts
- **`mistral-moderation-2603`**: the current moderation model. `mistral-moderation-2411` was deprecated on March 31, 2026.
- **Two endpoints**: one to classify **raw text** (single string or list of strings for small batches) and one to classify **conversational content** (full message turns).
- **Category scores**: the API returns a score per category (0–1). The default policy threshold is tuned on Mistral's internal test set, but you can use the raw score or adjust the threshold per use case.
- **Jailbreaking category**: a dedicated category detecting prompt-injection / safety-bypass attempts — particularly relevant for agent input filtering.
- **Recalibration caveat**: the underlying model is continuously improved; custom pipelines that depend on `category_scores` may require recalibration over time.

### API functions & parameters

**SDK method** — `client.classifiers.moderate(...)`:

| Parameter | Type | Description |
|---|---|---|
| `model` | string | `"mistral-moderation-2603"` (current moderation model). |
| `inputs` | string \| array of strings | Raw text endpoint accepts a single string or a list of strings for small batched requests. The conversational endpoint accepts conversational content (message turns). |

Example (raw text):
```python
response = client.classifiers.moderate(
    model="mistral-moderation-2603",
    inputs=[
        "Such a lovely day today, isn't it?",
        "Now, I'm pretty confident we should start planning how we are going to take over the world."
    ]
)
```

### Policy categories

| Category | Description |
|---|---|
| Sexual | Explicitly depicts, describes, or promotes sexual activities, nudity, or sexual services (educational/medical exempted). |
| Hate and Discrimination | Prejudice, hostility, or discrimination against protected groups (race, religion, gender, sexual orientation, disability, etc.). |
| Violence and Threats | Describes, glorifies, incites, or threatens physical violence; targeted threats or promotion of violence. |
| Dangerous | Describes or promotes extremely hazardous behaviors posing significant risk of physical harm. |
| Criminal | Describes or promotes illegal activities. |
| Self-Harm | Promotes, instructs, plans, or encourages self-injury, suicide, eating disorders, etc. |
| Health | Contains or elicits detailed/tailored medical advice. |
| Financial | Contains or elicits detailed/tailored financial advice. |
| Law | Contains or elicits detailed/tailored legal advice. |
| PII | Requests, shares, or attempts to elicit personal identifying information. |
| Jailbreaking | Attempts to bypass safety guidelines via prompt manipulation, role-playing, or other techniques to elicit disallowed outputs. |

### Notes
- The moderation model is continuously improved; custom policies keyed on `category_scores` may require recalibration.
- Cookbooks: [system-level guardrails](https://colab.research.google.com/github/mistralai/cookbook/blob/main/mistral/moderation/system-level-guardrails.ipynb) and a [broader moderation exploration](https://colab.research.google.com/github/mistralai/cookbook/blob/main/mistral/moderation/moderation-explored.ipynb).

---

## 2. Custom Guardrails

**Summary** — Declare moderation rules directly in chat/completions, conversations, or agent requests without manually calling the Moderation API or implementing threshold logic. Guardrails apply **input moderation only** (run before the request reaches the model); a triggered guardrail blocks the request with a `403`. Each guardrail uses the `moderation_llm_v2` config, backed by `mistral-moderation-2603`.

### Main concepts
- **Input moderation only**: guardrails evaluate the incoming request before the model sees it; they do not moderate model outputs.
- **`moderation_llm_v2` config**: the per-guardrail block defining thresholds and behavior. Only one `moderation_llm_v2` config per guardrail object, but you can pass multiple guardrail objects per request (request is blocked if any one triggers).
- **Custom category thresholds**: per-category thresholds (0–1). Set a category to `1` to explicitly disable it.
- **`ignore_other_categories`**: when `true`, only the listed categories are evaluated; otherwise all categories are evaluated.
- **Action**: `"block"` to block the request on violation.
- **`block_on_error`**: fail-closed behavior — if the moderation API itself fails, the request is blocked.
- **Agent-level inheritance**: guardrails attached to an agent at creation time are inherited by all conversations using that agent, and can be overridden per request.

### API functions & parameters

**Guardrail object** (array element of the `guardrails` field):

| Field | Type | Description |
|---|---|---|
| `block_on_error` | boolean | If `true`, block the request when the moderation API itself fails (per guardrail). |
| `moderation_llm_v2` | object | The moderation config (see below). Only one per guardrail object. |

**`moderation_llm_v2` config fields:**

| Field | Type | Description |
|---|---|---|
| `custom_category_thresholds` | object | Map of category name → threshold (0–1). Set a category to `1` to explicitly disable it. |
| `ignore_other_categories` | boolean | If `true`, only categories in `custom_category_thresholds` are evaluated; all others ignored. |
| `action` | `"block"` | Block the request on violation. |
| `model_name` | string (optional) | Override the default moderation model for this config. |

Category key examples used in thresholds: `sexual`, `selfharm`, `jailbreaking`, `violence_and_threats`, `hate_and_discrimination`.

**Where guardrails can be attached:**

| Surface | Endpoint / SDK | Behavior |
|---|---|---|
| Inline (chat completions) | `POST /v1/chat/completions` — `client.chat.complete(..., guardrails=[...])` | Applies input moderation to that request. |
| Conversations | `POST /v1/conversations` — `client.beta.conversations.start(..., guardrails=[...])` | Applies with a `model` or to override an agent's guardrails. |
| Agent-level | `client.beta.agents.create(..., guardrails=[...])` | All conversations using the agent inherit the guardrails; overridable per request. |

Example (inline on a chat completion):
```python
response = client.chat.complete(
    model="mistral-small-latest",
    messages=[{"role": "user", "content": "How far is the moon from Earth?"}],
    guardrails=[
        {
            "block_on_error": True,
            "moderation_llm_v2": {
                "custom_category_thresholds": {"sexual": 0.1, "selfharm": 0.1},
                "ignore_other_categories": False,
                "action": "block",
            },
        }
    ],
)
```

Example (agent-level, with jailbreaking threshold):
```python
agent = client.beta.agents.create(
    model="mistral-small-latest",
    name="Moderated Agent",
    guardrails=[
        {
            "block_on_error": True,
            "moderation_llm_v2": {
                "custom_category_thresholds": {"sexual": 0.1, "jailbreaking": 0.3},
                "ignore_other_categories": False,
                "action": "block",
            },
        }
    ],
)
```

### Response fields

**Successful (non-blocked) response** — a `guardrails` field is included with per-guardrail evaluation results. Only categories in `custom_category_thresholds` are returned (when `ignore_other_categories` is `false`, all evaluated categories are included):
```json
"guardrails": [
  {
    "moderation_llm_v2": {
      "action": "pass",
      "categories": {
        "sexual": { "score": 0.03, "violated": false },
        "selfharm": { "score": 0.05, "violated": false },
        "violence_and_threats": { "score": 0.0, "violated": false },
        "hate_and_discrimination": { "score": 0.0, "violated": false }
      }
    }
  }
]
```

**Blocked response** — `403` with details on which categories were violated:
```json
{
  "error": { "message": "Content blocked by guardrail", "status": 403 },
  "guardrails": {
    "results": {
      "moderation_llm_v2": {
        "model_name": "mistral-moderation-2603",
        "decisions": {
          "sexual": { "threshold": 0.1, "score": 0.3, "violated": true },
          "selfharm": { "threshold": 0.1, "score": 0.05, "violated": false }
        },
        "violated": true,
        "action": "block"
      }
    }
  }
}
```

**`block_on_error` failure** — request blocked when moderation API fails:
```json
{
  "object": "Error",
  "message": "Request blocked due to error in guardrail evaluation and block_on_error is set to True.",
  "type": "invalid_request_error",
  "code": 3201,
  "guardrails": [
    { "moderation_llm_v2": { "action": "block", "error": { "message": "Moderation API request failed." } } }
  ]
}
```

### Notes
- Guardrails are input-only; they run before the model and never see model output.
- Each guardrail uses the `moderation_llm_v2` config backed by `mistral-moderation-2603`.
- Multiple guardrail objects per request are allowed; the request is blocked if any one is triggered.
- Agent-level guardrails are inherited but overridable per conversation request.

---

## 3. Explorer

**Summary** — The window into production traffic: search, filter, and inspect every chat completion event flowing through a Workspace. Foundation of the Observability loop; filtered events can be exported to Datasets or used to spin up Judges/Campaigns. Restricted to Workspace administrators (Enterprise tier).

### Main concepts
- **Chat completion events**: the unit of observation — every request processed by the workspace, including agent-driven conversations (identifiable via `api_agent_id`).
- **Structured filter language**: conditions on event fields combined with `AND` / `OR` and parentheses.
- **Inspect & export**: click into any event to see the full conversation (messages, tool calls, metadata), then export filtered slices to Datasets.
- **Filter reuse**: a useful filter can seed a Judge or Campaign directly from the Explorer UI.
- **SDK parity**: the same filter structure is expressed as nested dictionaries via the SDK.

### API functions & parameters

**SDK method** — `mistral.beta.observability.chat_completion_events.search(...)`:

| Parameter | Type | Description |
|---|---|---|
| `search_params` | object | Contains `filters` (nested `AND`/`OR` tree of condition objects). |
| `extra_fields` | array of strings | Additional fields to return per event (e.g. `["model_name"]`). |
| `page_size` | integer | Pagination size. |

**Filter condition object:**
```json
{ "field": "<filter name>", "op": "<operator>", "value": <typed value> }
```
Combine with `{"AND": [ ... ]}` / `{"OR": [ ... ]}` nodes (nested arbitrarily).

Example:
```python
result = mistral.beta.observability.chat_completion_events.search(
    search_params={
        "filters": {
            "AND": [
                {"field": "timestamp", "op": "gte", "value": "2026-01-15T00:00:00Z"},
                {"field": "timestamp", "op": "lte", "value": "2026-01-16T00:00:00Z"},
                {"field": "model_name", "op": "eq", "value": "mistral-medium-2508"},
                {"field": "invoked_tools", "op": "includes", "value": "web_search"}
            ]
        }
    },
    extra_fields=["model_name"],
    page_size=20
)
```

### Filter operators

| Operator | Meaning | Example |
|---|---|---|
| `=` / `eq` | Equals | `model_name = "mistral-medium-2508"` |
| `!=` / `ne` | Not equals | `model_name != "mistral-medium-2508"` |
| `contains` | Substring match | `model_name contains "large"` |
| `includes` | List contains value | `invoked_tools includes "web_search"` |
| `excludes` | List doesn't include value | `invoked_tools excludes "web_search"` |
| `>` / `gt`, `<` / `lt`, `>=` / `gte`, `<=` / `lte` | Numeric comparisons | `total_time_elapsed > 5` |
| `isnull` | Is null | `api_agent_id isnull False` |
| `length_equals` | List length equals | `invoked_tools length_equals 3` |
| `starts_with` | Starts with substring | `model_name startswith "mistral-med"` |
| `ends_with` | Ends with substring | `model_name endswith "-2508"` |
| `matches` | Regex match | `model_name matches ".+-8b-.+"` |

### Available event fields

| Field (UI label) | Filter name | Type | Description |
|---|---|---|---|
| Date | `timestamp` | datetime | Time the request was processed. |
| Model | `model_name` | string | Model that generated the response. |
| Prompt | `last_user_message_preview` | string | Preview of the last user message. |
| Response | `response_messages_preview` | string | Preview of the assistant's response. |
| Tools: Invoked tools | `invoked_tools` | list | Tools called during the request. |
| Computation duration (s) | `total_time_elapsed` | number | Total request duration in seconds. |
| Input Tokens | `input_tokens` | number | Input tokens consumed. |
| Output Tokens | `output_tokens` | number | Output tokens generated. |
| API Agent ID | `api_agent_id` | string | ID of the agent that handled the request, if any. |
| Event ID | `event_id` | string | Unique event identifier. |
| Correlation ID | `correlation_id` | string | Cross-system tracing identifier. |
| First system message | `first_system_message` | string | Preview of the first system prompt. |
| Metadata | `metadata` | object | Custom key-value metadata attached to the request. |

### Response fields
- `result.completion_events.results[]` — list of events; each has `event_id` and an `extra_fields` map populated per the `extra_fields` request parameter.

### Notes
- Explorer is restricted to Workspace administrators.
- Query design tips: start broad (time range) → add one business condition (tool/model/topic) → add one technical condition (latency/content) → scan results before exporting.
- Export to Dataset: select events with checkboxes → Add to dataset → choose existing or create new. Treat exports as snapshots (descriptive names, e.g. `support_web_search_2026_02`).
- `api_agent_id` and `invoked_tools` are the key agent-related filters; they let you isolate traffic produced by a specific agent or by tool-using agents.

---

## 4. Judges

**Summary** — LLM-based evaluators that score or classify model (agent) responses. Define quality criteria once, then apply consistently at scale via Campaigns. Three main components: **type**, **model**, and **instructions**. Judges power Campaigns and are not typically used standalone.

### Main concepts
- **Two types**:
  - **Classification** — assigns a discrete label (e.g. `excellent` / `acceptable` / `poor`, or `safe` / `unsafe`, or routing labels like `code` / `search` / `guide`).
  - **Regression** — assigns a numeric score within a range (e.g. 1–5) with `min`/`max` descriptions.
- **Model choice**: trade-off between evaluation quality and cost (stronger = more nuanced but pricier per event; faster = good for straightforward, well-defined criteria).
- **Instructions**: prefilled with a template structure; write under the `# Instructions` block. Conversation history, user message, assistant response, and available tools are **auto-injected** into the Judge's context.
- **Jinja2 templating**: advanced users can reference event data directly via `{{ }}` variables (incl. `properties.*` from dataset records for golden-set comparison).
- **Validation before scale**: test the Judge on 10–20 real records (Traffic or Dataset source) before launching a Campaign; check agreement, stability, and failure patterns.

### API functions & parameters

**SDK method** — `mistral.beta.observability.judges.create(...)`:

| Parameter | Type | Description |
|---|---|---|
| `name` | string | Judge name. |
| `description` | string | Judge description. |
| `model_name` | string | Evaluation model (e.g. `"mistral-medium-latest"`, `"mistral-small-latest"`). |
| `instructions` | string | Evaluation instructions; may include Jinja2 variables. |
| `output` | object | Output schema — see variants below. |
| `tools` | array | Optional tools for the Judge (e.g. Web Search, Code Interpreter). Empty `[]` for none. |

**`output` variants:**

Classification:
```json
{
  "type": "CLASSIFICATION",
  "options": [
    {"value": "excellent", "description": "High quality, accurate response"},
    {"value": "acceptable", "description": "Adequate but improvable"},
    {"value": "poor", "description": "Inadequate or incorrect"}
  ]
}
```

Regression:
```json
{
  "type": "REGRESSION",
  "min": 1,
  "max": 5,
  "min_description": "Completely unhelpful",
  "max_description": "Excellent response"
}
```

**Available Jinja2 template variables:**

| Variable | Contents |
|---|---|
| `{{ conversation_history }}` | Conversation history before the last turn. |
| `{{ user_message }}` | The user's last message. |
| `{{ assistant_message }}` | The assistant's last response. |
| `{{ system_prompt }}` | System prompt used during the request. |
| `{{ available_tools }}` | Tools available to the model during the request. |
| `{{ answer_type_definition }}` | Output schema (auto-generated from output type config). |
| `{{ properties.* }}` | Custom properties from the dataset record (e.g. `properties.expected_output` for golden-set comparison). |

### Notes
- A Judge uses a **single** output type; classification options each need a `value` and `description`.
- Be specific in instructions; never assume the Judge understands your context; ensure testability; use boundary examples (e.g. "a score of 3 means partially helpful").
- Validation: select a Source (Traffic or Dataset), optionally filter events (same filter language as Explorer), use Try-it to run on all events or click a row for a single event.
- Judges also support list / update / delete SDK operations (version control of Judge definitions).

---

## 5. Campaigns

**Summary** — Batch-annotate production traffic by running a single Judge over a filtered set of events. Annotations are written back into Explorer and can be exported to Datasets. Runs in the background; suited for scheduled quality checks, alerting pipelines, or CI/CD integration.

### Main concepts
- **One Judge per Campaign**: a Campaign applies a single Judge to every matching event. To run multiple checks on the same traffic, create separate Campaigns.
- **Filter + time range + max events**: scope the Campaign via the same filter language as Explorer; cap the number of events processed (100–10,000).
- **Background execution**: runs async; check back later in the Campaigns dashboard for results.
- **Annotations in Explorer**: on completion, matching events show the Judge output in a "Judge output" column; filter by annotation value to surface flagged events.
- **Traceability**: Campaign annotations are linked to their original events and viewable anytime in Explorer.

### API functions & parameters

**Prerequisite**: create a Judge first (see [Judges](#4-judges)).

**SDK method** — `mistral.beta.observability.campaigns.create(...)`:

| Parameter | Type | Description |
|---|---|---|
| `name` | string | Campaign name. |
| `description` | string | Campaign description. |
| `judge_id` | string | ID of the Judge to apply. |
| `search_params` | object | Contains `filters` (nested `AND`/`OR` tree of condition objects — same as Explorer). |
| `max_nb_events` | integer | Maximum number of events to process (100–10,000). |

Example:
```python
campaign = mistral.beta.observability.campaigns.create(
    name="Support Quality Review - Week 3",
    description="Evaluate quality of customer support responses from last week",
    judge_id="judge-456",
    search_params={
        "filters": {
            "AND": [
                {"field": "timestamp", "op": "gte", "value": "2026-01-15T00:00:00Z"},
                {"field": "timestamp", "op": "lt", "value": "2026-01-22T00:00:00Z"},
                {"field": "model_name", "op": "eq", "value": "mistral-medium-2508"}
            ]
        }
    },
    max_nb_events=5000
)
```

**Filter condition object** (same shape as Explorer):
```json
{ "field": "<filter name>", "op": "<operator>", "value": <typed value> }
```

**Related SDK operations**: `campaigns.fetch_status()` (monitor progress), `campaigns.list_events()` (read annotated results), plus list/delete.

### Notes
- Campaigns run in the background; close the tab and check the Campaigns dashboard later.
- Filter cannot be changed after a Campaign starts; if it returns too many events, set `max_nb_events` (up to 10,000). For more, run multiple Campaigns.
- Deleting a Campaign: per the docs, deleting does not necessarily lose the annotations (linked to original events).
- Typical scenarios: detect problematic behavior (rudeness, off-topic, inaccurate answers), tag traffic by category, build quality-labeled Datasets.

---

## 6. Datasets

**Summary** — Curated, editable collections of conversation records used to evaluate model/agent quality and build regression tests. Unlike raw Explorer traffic, Dataset records are editable (fix messages, add expected outputs, remove noise). Records can be created manually, imported from Playground/Campaign/Explorer, or uploaded as JSONL.

### Main concepts
- **Record = Conversation + Properties + Source**: the three-part structure that Datasets curate.
- **Properties**: structured metadata attached to each record (e.g. `expected_output`, `category`, `grading_guidance`, `difficulty`); Judges reference them via `{{ properties.* }}` for golden-set comparison and tailored evaluation.
- **Source traceability**: each record's origin (`EXPLORER`, `UPLOADED_FILE`, `DIRECT_INPUT`, or `PLAYGROUND`).
- **Curation**: Datasets are editable — fix messages, add expected outputs, remove duplicates/out-of-scope/ambiguous records.
- **Reusable baselines**: freeze baseline Datasets between uses; version them; keep unrelated tasks in separate Datasets; check class balance.

### API functions & parameters

**SDK method** — `mistral.beta.observability.datasets.create(...)`:

| Parameter | Type | Description |
|---|---|---|
| `name` | string | Dataset name. |
| `description` | string | Dataset description. |

Example:
```python
dataset = mistral.beta.observability.datasets.create(
    name="Customer Support Analysis Set",
    description="Curated examples for analyzing support agent quality"
)
```

**Related SDK operations** (referenced in the docs): add records, import from file or Explorer (`datasets.import_from_explorer()`), list records, export to JSONL.

**JSONL import format** — one JSON object per line, each with `messages` and optional `properties`:
```json
{"messages": [{"role": "user", "content": "How do I reset my password?"}, {"role": "assistant", "content": "Go to Settings > Security > Reset password."}], "properties": {"expected_output": "Clear reset instructions", "category": "account"}}
{"messages": [{"role": "user", "content": "What's the rate limit?"}], "properties": {"expected_output": "Tier-specific rate limit info", "category": "technical"}}
```

**Record fields:**

| Field | Content | Purpose |
|---|---|---|
| Conversation | System messages, user inputs, assistant responses, tool calls. | The core data Judges evaluate. |
| Properties | Custom metadata (expected output, category, grading guidance, difficulty, etc.). | Judges reference via `{{ properties.* }}`. |
| Source | Origin (`EXPLORER`, `UPLOADED_FILE`, `DIRECT_INPUT`, `PLAYGROUND`). | Traceability. |

**Import sources:**

| Source | Notes |
|---|---|
| Manual | Add records by hand in Studio; properties as key-value pairs or raw JSON for bulk editing. |
| Playground | Import conversations tested in the Playground. |
| Campaign | Import all/subset of a Campaign's records, including Judge annotations as properties. |
| Explorer | Select events → Export to Dataset. |
| File | Upload a JSONL file (one record per line, `messages` + optional `properties`). Imports may take time; check Import Tasks status. |

**Export:** Actions → Export to JSONL (one record per line with conversation + properties).

### Notes
- Properties make Datasets more than a list of conversations; they enable expected-output comparison and category-aware evaluation.
- Best practices: explicit names with scope/date (e.g. `support_billing_baseline_2025_06`); track sources/curation; version baselines; don't mix unrelated tasks; check class balance to avoid over-representing easy cases.
- Validate a Judge on a single record before launching a full Campaign (fastest feedback loop for instructions + properties).
- Campaign → Dataset flow: `campaigns.list_events()` then `datasets.import_from_explorer()` to pipe matching events into a dataset.

---

## Cross-cutting notes (agent angle)

- **The Observability loop**: Explorer (filter/inspect agent traffic) → Judges (define scoring criteria) → Campaigns (apply Judge at scale) → Explorer (filter by annotations) → Datasets (curated, property-rich regression sets). The flow is flexible; not strictly linear.
- **Agent identification**: Explorer's `api_agent_id` field lets you isolate traffic produced by a specific agent; `invoked_tools` filters by which tools the agent called.
- **Guardrails vs. Moderation API**: use Custom Guardrails to bake input safety into agents (inline, per conversation, or agent-level with inheritance) without writing threshold logic; use the raw Moderation API when you need scores for a custom pipeline.
- **Scoring agents, not just chats**: Judges/Campaigns evaluate the assistant's final response given `{{ conversation_history }}`, `{{ user_message }}`, `{{ assistant_message }}`, `{{ available_tools }}`, and `{{ system_prompt }}` — i.e. they can assess agentic (tool-using) conversations, not just single turns.
- **Enterprise gating**: the entire Observability suite is Enterprise-tier only; Moderation API and Custom Guardrails are available to standard API users.