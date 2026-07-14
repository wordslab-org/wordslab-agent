# Gemini API — Agent Tools Capabilities

Analysis of the agent-related capabilities offered by the Gemini API, based on the official developer guides (https://ai.google.dev/gemini-api/docs/tools and the per-tool sub-pages). Each capability is broken down into main concepts, API surface (functions, parameters, response fields), and notes/constraints.

All tools are enabled via the `tools` array on the **Interactions API** (`POST https://generativelanguage.googleapis.com/v1beta/interactions` / `client.interactions.create(...)`). Built-in tools are executed server-side by Google; custom tools (Function Calling, Computer Use) are executed client-side by the application, which returns results back to the model. Gemini 3 series models additionally support combining built-in and custom tools in a single interaction via tool context circulation.

---

## Table of Contents

1. [Function Calling](#1-function-calling)
2. [Google Search](#2-google-search)
3. [Google Maps](#3-google-maps)
4. [Code Execution](#4-code-execution)
5. [URL Context](#5-url-context)
6. [Computer Use](#6-computer-use)
7. [File Search](#7-file-search)
8. [Tool Combination](#8-tool-combination)

---

## 1. Function Calling

**Summary** — Connects models to external tools/APIs you define: instead of generating text, the model decides when to call functions and emits structured arguments that your application executes, then returns results back to the model.

### Main concepts
- **Function declarations**: Passed as items in the `tools` array; each declares a function the model may call (`name`, `description`, `parameters` JSON schema).
- **The function-call loop** (multi-turn, repeatable):
  1. Declare a function and send it in `tools` with the user `input`.
  2. Model returns a `function_call` step (`name`, `arguments`, `id`).
  3. You execute the function locally using `arguments`.
  4. Send the result back as a `function_result` step (`name`, `call_id`, `result`), referencing the prior interaction via `previous_interaction_id` (stateful) or by passing the full history in `input` with `store=false` (stateless).
  5. Model produces final text (`interaction.output_text`) or another function call.
- **Parallel function calling**: Model emits multiple `function_call` steps in one turn when calls are independent; triggered by `generation_config={"tool_choice": "any"}`.
- **Compositional function calling**: Model chains calls in sequence (one call's output informs the next); no special config — the model decides from the declared tools.
- **Thought signatures**: Gemini 3 series models use internal "thinking"; steps can include a `thought` step. The SDKs automatically handle thought signatures, but in stateless mode the `thought` steps and `signature` fields must be replayed exactly in the input history.
- **Remote MCP**: Tool entry `type="mcp_server"` with `name`, `url`, optional `headers`, optional `allowed_tools`.

### API functions & parameters

**Request top-level fields** (Interactions API — `POST /v1beta/interactions`):

| Field | Type | Notes |
|---|---|---|
| `model` | string | e.g. `"gemini-3.5-flash"` |
| `input` | string \| array | User prompt (string) or array of steps (stateless/multimodal) |
| `tools` | array | Function declarations, built-in tools, or MCP servers |
| `generation_config` | object | Holds `tool_choice` |
| `previous_interaction_id` | string | Links turns in stateful mode |
| `store` | boolean | `false` for stateless mode |
| `stream` | boolean | SSE streaming |

**FunctionDeclaration** (item in `tools`):

| Field | Type | Required | Notes |
|---|---|---|---|
| `type` | string | Yes | `"function"` |
| `name` | string | Yes | Function name |
| `description` | string | Yes | What the function does |
| `parameters` | object | — | JSON schema object |

**`parameters` (JSON schema)**:

| Field | Type | Notes |
|---|---|---|
| `type` | string | `"object"` |
| `properties` | object | Map of param name → schema |
| `required` | array | List of required property names |

Property schema fields: `type` (`"string"`, `"integer"`, `"number"`, `"boolean"`, `"array"`), `description`, `enum`, `items` (schema for array elements, e.g. `{"type":"string"}` / `{"type":"number"}`).

**`function_call` response step**:

| Field | Type | Notes |
|---|---|---|
| `type` | string | `"function_call"` |
| `name` | string | The function to call |
| `arguments` | object | Parsed argument dict |
| `id` | string | Unique call ID; used as `call_id` in the `function_result` |

**`function_result` input step** (you → model):

| Field | Type | Notes |
|---|---|---|
| `type` | string | `"function_result"` |
| `name` | string | Same function name as the call |
| `call_id` | string | The `id` from the `function_call` |
| `result` | array | Content blocks (see below) |

`result` content blocks: `{ "type": "text", "text": ... }` or (multimodal, Gemini 3) `{ "type": "image", "mime_type": "image/jpeg", "data": "<base64>" }`.

**Tool choice / mode** (`generation_config.tool_choice`):

| Mode | Description |
|---|---|
| `auto` | Model decides whether to call a function (default) |
| `any` | Model must call a function |
| `none` | Model must not call any function |
| `validated` | (Preview) Model ensures function-schema adherence |

Structured form to restrict allowed functions:
```json
"generation_config": {
  "tool_choice": {
    "allowed_tools": { "mode": "any", "tools": ["get_current_temperature"] }
  }
}
```

**Streaming tool calls**: Use `stream=true` (REST: `?alt=sse`). Events: `step.start` (`step.id`, `step.name`, partial `arguments`), `step.delta` (`delta.type` = `"arguments"` with `partial_arguments`, or `"text"`), `interaction.completed`. Aggregate `partial_arguments` per `event.index`, then `JSON.parse` at completion to reconstruct each call.

### Notes
- Examples use `gemini-3.5-flash`; thinking + multimodal-function-response features are Gemini 3 series.
- **Stateless mode**: `store=false`; pass full history (initial `user_input` + all model steps incl. `thought` & `function_call` exactly as received + your `function_result`) in `input` each turn.
- Best practices / limitations sections on the page exist but are empty (no enumerated content).
- This Interactions-API page documents `tool_choice` / `function_call`-`function_result` steps / `call_id`. The legacy `tool_config` / `function_calling_config` / `allowed_function_names` / `FunctionCall`-`FunctionResponse` part schema (older `generateContent` API) and OpenAI-compatible mapping are not covered on this page.

---

## 2. Google Search

**Summary** — Grounds model responses in real-time web content from Google Search, with verifiable inline citations, to reduce hallucinations and answer questions about recent events.

### Main concepts
- **Grounding with Google Search**: Connects Gemini to real-time web content; the model handles the entire workflow of searching, processing, and citing information automatically.
- **Citations / `url_citation` annotations**: Inline annotations on the model's text blocks give full UI control over how sources are displayed.
- **Search suggestions widget**: Raw results include `search_suggestions` HTML/CSS markup for rendering a search-suggestions widget.
- **Workflow steps**: `thought`, `google_search_call`, `google_search_result`, `model_output`.
- **Combinations**: Can be combined with URL Context (search online, then deep-analyze specific pages), Code Execution, and (Gemini 3) custom function-calling tools.

### API functions & parameters

**Tool config object** — passed in the `tools` array:

| Parameter | Type | Description |
|---|---|---|
| `type` | string | `"google_search"` (fixed) |

```python
client.interactions.create(
    model="gemini-3.5-flash",
    input="Who won the euro 2024?",
    tools=[{"type": "google_search"}]
)
```

REST:
```json
{ "model": "gemini-3.5-flash", "input": "...", "tools": [{"type": "google_search"}] }
```

**Response step fields**:

| Step `type` | Key fields | Description |
|---|---|---|
| `thought` | `summary` (list of `{type:"text", text}`), `signature` | Model's internal reasoning before searching |
| `google_search_call` | `arguments.queries` (list of strings) | Search query/queries the model issues |
| `google_search_result` | `call_id` (e.g. `"search_001"`), `result` (list) | Raw search results; `result[].search_suggestions` holds HTML/CSS for the widget |
| `model_output` | `content` (list of content blocks) | Final answer content |

**`url_citation` annotation** (on text content blocks):

| Field | Type | Notes |
|---|---|---|
| `type` | string | `"url_citation"` |
| `url` | string | Source URL |
| `title` | string | Source title |
| `start_index` | integer | Start offset into the `text` for the cited span |
| `end_index` | integer | End offset (exclusive) for the cited span |

### Notes
- **Pricing (Gemini 3 models)**: billed **per search query** the model executes. Multiple queries in one prompt = multiple billable uses. Empty queries are ignored when counting unique queries for billing.
- **Pricing (Gemini 2.5 and older)**: billed **per prompt** (not per query).
- Supported models: Gemini 3.5 Flash, Gemini 3.1 Flash Image Preview, Gemini 3.1 Pro Preview, Gemini 3 Pro Image Preview, Gemini 3 Flash Preview, Gemini 2.5 Pro/Flash/Flash-Lite, Gemini 2.0 Flash.
- The page documents the Interactions API only. The older `google_search_retrieval` tool name, `dynamic_retrieval_config` / `dynamic_threshold`, and the `groundingMetadata` / `groundingChunks` / `groundingSupports` / `searchEntryPoint` / `webSearchQueries` response fields belong to the legacy `generateContent` shape and are not present here.

---

## 3. Google Maps

**Summary** — Connects Gemini's generative capabilities with Google Maps' factual, up-to-date place data (250+ million places worldwide) to build location-aware assistants.

### Main concepts
- **Places grounding**: When a user query has geographical context, the model invokes the Maps grounding tool and generates responses grounded in Google Maps data.
- **Maps context**: Triggered by passing a user location (`latitude`, `longitude`) in the tool config; the model uses that location to bias/ground the answer.
- **`place_citation` annotations**: Every grounded result must display associated Google Maps sources with links; users must be informed that Google Maps sources were used.

### API functions & parameters

**Tool config object** — passed in the `tools` array:

| Parameter | Type | Required | Description |
|---|---|---|---|
| `type` | string | Yes | `"google_maps"` |
| `latitude` | float | No | User location latitude (e.g. `34.050481`). Best practice to include. |
| `longitude` | float | No | User location longitude (e.g. `-118.248526`). Best practice to include. |

```python
client.interactions.create(
    model="gemini-3.5-flash",
    input="...",
    tools=[{"type": "google_maps", "latitude": 34.050481, "longitude": -118.248526}]
)
```

**Response fields** (iterating `interaction.steps`; `model_output` steps contain `content` blocks of type `"text"` with optional `annotations`):

| Field path | Type | Description |
|---|---|---|
| `interaction.steps[]` | object | Each step in the interaction |
| `step.type == "model_output"` | string | The model's output step |
| `step.content[]` | object | Content blocks within the step |
| `content_block.type == "text"` | string | A text response block |
| `content_block.text` | string | The generated text |
| `content_block.annotations[]` | array | Source citations grounding the text |
| `annotation.type == "place_citation"` | string | A Google Maps place citation |
| `annotation.name` | string | Place name (display in attribution) |
| `annotation.url` | string | Google Maps URL for the place (must render as a link preview) |

### Notes
- **Attribution required**: Display `place_citation` annotations with Google Maps links and inform users that Google Maps sources were used; apply Google Maps/Earth legal notices.
- Supported models: Gemini 3.5 Flash, Gemini 3.1 Pro Preview, Gemini 3.1 Flash-Lite, Gemini 3 Flash Preview, Gemini 2.5 Pro/Flash/Flash-Lite.
- Pricing differs by model generation (see pricing page).
- Requires the genai SDK newer than 2.0.0 and the Interactions API.

---

## 4. Code Execution

**Summary** — Lets the model generate and run Python code in a server-side sandbox, then learn iteratively from the execution results until reaching a final output.

### Main concepts
- **Python only**: Gemini can only *run* code in Python. It can still *generate* code in other languages, but cannot execute them with this tool.
- **Iterative learning**: Results are returned as a sequence of steps — `text`, `code_execution_call`, `code_execution_result` — and the model can iterate until it reaches a final output.
- **Code execution vs function calling**: Code execution runs model-generated Python in a managed sandbox server-side (no external calls/definitions from the user). Function calling has the model emit structured arguments that the *caller* executes against external functions.
- **Image code execution (Gemini 3)**: Inspect/manipulate images via code; requires enabling both Code Execution and Thinking. Input includes `image` parts; output may return inline images.
- **Bundled libraries**: Cannot install your own libraries; `matplotlib` is bundled (graph output returned as inline images).

### API functions & parameters

**Tool config object** — passed in the `tools` array (no additional parameters):

| Parameter | Type | Description |
|---|---|---|
| `type` | string | `"code_execution"` (fixed) |

```python
client.interactions.create(
    model="gemini-3.5-flash",
    input="What is the square root of the latest stock price of GOOG?",
    tools=[{"type": "code_execution"}]
)
```

Request-level fields: `model`, `input` (string or content parts incl. `image` parts for image code execution), `tools`, optional `previous_interaction_id` for multi-turn.

**Response step types** (iterate `interaction.steps`):

| Step type | Field | Meaning |
|---|---|---|
| `model_output` | `step.content[]` | Model text/image content blocks (`text` → `.text`; `image` → `.data` base64 + `.mime_type`) |
| `code_execution_call` | `step.arguments.code` | The Python code the model generated to run |
| `code_execution_result` | `step.result` | The stdout/execution outcome from running that code |

### Notes
- Supported models: Gemini 3.5 Flash (and Gemini 3 Flash for image code execution). Multi-turn supported via `previous_interaction_id`.
- **Pricing**: No additional charge for enabling code execution — billed only for input/output tokens at the model's current rate.
- Can be combined with Google Search grounding; Gemini 3 models allow combining built-in Code Execution with custom function-calling tools.
- Server-side sandbox disallows custom library installation.

---

## 5. URL Context

**Summary** — Lets you provide additional context to models in the form of URLs; the model accesses content from those URLs to inform/enhance its response.

### Main concepts
- **Two-step retrieval**: First attempts to fetch content from an internal **index cache** (optimized cache). If the URL isn't in the index (e.g. very new page), it falls back to a **live fetch** that directly accesses the URL in real time.
- **`url_citation` annotations**: Inline citations linking response text segments to source URLs.
- **Combinations**: With Google Search (model searches online, then uses URL context for in-depth analysis of specific found pages); with custom function-calling tools (Gemini 3).
- **Token accounting**: Retrieved URL content counts as **input tokens** (`tool_use_input_tokens`).

### API functions & parameters

**Tool config object** — passed in the `tools` array (no additional parameters documented):

| Field | Type | Value |
|---|---|---|
| `type` | string | `"url_context"` |

```python
client.interactions.create(model="...", input="...", tools=[{"type": "url_context"}])
```

**Response fields**:

`url_citation` annotation (on text content blocks):

| Field | Description |
|---|---|
| `type` | `"url_citation"` |
| `url` | Source URL |
| `title` | Source title |
| `start_index` | Start of linked text segment |
| `end_index` | End of linked text segment |

`url_context_result` step (debugging metadata about each URL retrieval attempt):

| Field | Description |
|---|---|
| `status` | Retrieval status (e.g. `"unsafe"` if content moderation check fails) |
| (retrieved URL) | The URL that was retrieved |

`usage` object (token accounting):

| Field | Example |
|---|---|
| `input_tokens` | `27` |
| `output_tokens` | `45` |
| `thoughts_tokens` | `31` |
| `tool_use_input_tokens` | `10309` (URL content tokens) |
| `tool_use_input_tokens_details` | `[{"modality": "TEXT", "token_count": 10309}]` |
| `total_tokens` | `10412` |

### Notes
- Supported models: Gemini 3.5 Flash, Gemini 3.1 Pro Preview, Gemini 3.1 Flash-Lite, Gemini 3 Flash Preview, Gemini 2.5 Pro/Flash/Flash-Lite.
- **Safety checks**: URLs undergo content moderation; failing URLs return `url_context_result` `status` of `"unsafe"`.
- Pricing: retrieved URL content billed as input tokens; price per token depends on the model.
- Requires SDK version newer than 2.0.0.

---

## 6. Computer Use

**Summary** — (Preview) Lets a model view a computer screen via screenshots and generate UI actions (mouse clicks, keyboard inputs) that your client-side execution environment runs, enabling browser, mobile, and desktop automation agents.

### Main concepts
- **Client-side execution loop**: Like function calling — you implement the environment to receive and execute Computer Use actions:
  1. Send a request (screenshot + prompt).
  2. Receive a `function_call` step (action) with `intent`, possibly `safety_decision`/`require_confirmation`.
  3. Execute the action client-side (e.g. via Playwright).
  4. Capture new state (screenshot + URL/result) and send back `function_result` entries.
  5. Repeat until the task completes or is terminated.
- **Normalized coordinates**: Model predicts coordinates in range **0–999** on both axes; you scale to actual pixels (`x/1000 * screen_width`, `y/1000 * screen_height`). No display size needed in the request.
- **Three environments**: `browser`, `mobile`, `desktop` (Gemini 3.5 Flash). Legacy Gemini 2.5 is browser-focused.
- **`intent`**: Gemini 3.5 Flash includes a tailored reasoning `intent` alongside coordinates on every action.
- **Safety policies**: Built-in safety categories; risky actions surface `safety_decision.decision == "require_confirmation"` requiring human confirmation before execution.
- **Custom user-defined functions**: Add normal `{"type":"function", ...}` declarations alongside the `computer_use` tool (e.g. a `yield_to_user` HITL tool).

### API functions & parameters

**Tool config object** — passed in the `tools` array:

| Field | Type | Description |
|---|---|---|
| `type` | string | `"computer_use"` |
| `environment` | string | `"browser"` / `"mobile"` / `"desktop"` |
| `excluded_predefined_functions` | array<string> | Optional. Predefined action names to exclude (e.g. `["drag_and_drop"]`, `["click"]`) |
| `enable_prompt_injection_detection` | boolean | Optional (3.5 Flash). Scans screenshot pixels for hidden adversarial prompts and blocks when detected |
| `disabled_safety_policies` | array<string> | Optional (3.5 Flash). Safety policy names to override/disable, e.g. `["data_modification"]` |

REST example:
```json
"tools": [{"type": "computer_use", "environment": "browser", "enable_prompt_injection_detection": true}]
```

**Action types — Browser environment (Gemini 3.5 Flash)** — all coordinate args are `int` in range 0–999; every action includes an `intent: string`:

| Command | Arguments |
|---|---|
| `click` | `x`, `y`, `intent` |
| `double_click` | `x`, `y`, `intent` |
| `triple_click` | `x`, `y`, `intent` |
| `middle_click` | `x`, `y`, `intent` |
| `right_click` | `x`, `y`, `intent` |
| `mouse_down` | `x`, `y`, `intent` |
| `mouse_up` | `x`, `y`, `intent` |
| `move` | `x`, `y`, `intent` |
| `type` | `text`, `press_enter` (opt, default false), `intent` |
| `drag_and_drop` | `start_x`, `start_y`, `end_x`, `end_y`, `intent` |
| `wait` | `seconds` (opt, default 1), `intent` |
| `press_key` | `key`, `intent` |
| `key_down` | `key`, `intent` |
| `key_up` | `key`, `intent` |
| `hotkey` | `keys` (array), `intent` |
| `take_screenshot` | `intent` |
| `scroll` | `x`, `y`, `direction` (`"up"`/`"down"`/`"left"`/`"right"`), `magnitude_in_pixels` (opt, default 300), `intent` |
| `go_back` | `intent` |
| `navigate` | `url`, `intent` |
| `go_forward` | `intent` |

**Action types — Mobile environment (3.5 Flash)**:

| Command | Arguments |
|---|---|
| `open_app` | `app_name`, `intent` |
| `list_apps` | `intent` |
| `click` | `x`, `y`, `intent` |
| `wait` | `seconds` (opt, default 1), `intent` |
| `go_back` | `intent` |
| `type` | `text`, `press_enter` (opt, default false), `intent` |
| `drag_and_drop` | `start_x`, `start_y`, `end_x`, `end_y`, `intent` |
| `long_press` | `x`, `y`, `seconds` (opt, default 2), `intent` |
| `press_key` | `key`, `intent` |
| `take_screenshot` | `intent` |

**Action types — Desktop environment (3.5 Flash)**: Same as Browser **except** no `go_back` / `navigate` / `go_forward`.

**Legacy action types (Gemini 2.5 — `gemini-2.5-computer-use-preview-10-2025`)**:

| Command | Arguments |
|---|---|
| `open_web_browser` | none |
| `wait_5_seconds` | none |
| `go_back` | none |
| `go_forward` | none |
| `search` | none |
| `navigate` | `url` |
| `click_at` | `x` (0-999), `y` (0-999) |
| `hover_at` | `x`, `y` |
| `type_text_at` | `x`, `y`, `text`, `press_enter` (opt, default True), `clear_before_typing` (opt, default True) |
| `key_combination` | `keys` (string) |
| `scroll_document` | `direction` |
| `scroll_at` | `x`, `y`, `direction`, `magnitude` (opt, default 800) |
| `drag_and_drop` | `x`, `y`, `destination_x`, `destination_y` |

**Response structure** (`interaction.steps`):
- `"model_output"` → `content`: list of `{type:"text", text:...}` blocks.
- `"function_call"` → `name`, `id`, `arguments` (dict of action fields). For 3.5 Flash, `arguments` includes `intent`; may include `safety_decision` (`{explanation, decision}` where `decision` is `"require_confirmation"`).

**Sending back screenshots / observations** — build `function_result` entries (one per executed `function_call`), pass them as `input` in the next `interactions.create` (with `previous_interaction_id`):

| Field | Value |
|---|---|
| `type` | `"function_result"` |
| `name` | action name |
| `call_id` | the `function_call.id` |
| `result` | list: `[{type:"text", text: JSON of {url, ...action_result}}, {type:"image", data: base64 PNG, mime_type:"image/png"}]` |

If a `safety_decision` was `require_confirmation` and the user confirmed, set `safety_acknowledgement: true` inside the action result. Initial turn may also include an image in `input`: `{type:"image", data: base64, mime_type:"image/png"}` alongside the text prompt.

**Safety policies (Gemini 3.5 Flash built-in)**:

| Category | Description |
|---|---|
| `FINANCIAL_TRANSACTIONS` | Payments, checkout, regulated goods |
| `SENSITIVE_DATA_MODIFICATION` | Health/financial/government records |
| `COMMUNICATION_TOOL` | Sending emails/messages |
| `ACCOUNT_CREATION` | Registering new accounts |
| `DATA_MODIFICATION` | File system modifications, data sharing/deletion |
| `USER_CONSENT_MANAGEMENT` | Cookie/privacy consent banners |
| `LEGAL_TERMS_AND_AGREEMENTS` | Accepting ToS / contracts |

### Notes
- Supported models: `gemini-3.5-flash` (recommended), `gemini-3-flash-preview`, `gemini-2.5-computer-use-preview-10-2025` (legacy).
- SDK: `google-genai` Python ≥ `2.7.0`; `@google/genai` Node.js. Uses the Interactions API (`previous_interaction_id`), not `generateContent`.
- Thinking levels can be configured (3.5 Flash) to balance action quality vs speed; lower levels suit standard automation.
- Safety best practices: Human-in-the-Loop confirmation, custom system instructions, secure sandboxed execution (VM/Docker/dedicated browser profile), input sanitization, content guardrails, allowlists/blocklists, observability/logging, consistent clean environment.
- Recommended client-side handler in examples: Playwright.

---

## 7. File Search

**Summary** — Enables Retrieval Augmented Generation (RAG): imports, chunks, and indexes your data for fast semantic retrieval; retrieved chunks are injected as context to ground the model's response.

### Main concepts
- **Semantic search**: Uses vector embeddings (not keyword search). Files are chunked → embedded → stored in a FileSearchStore; the query is embedded and matched via vector search to the most relevant chunks.
- **Embedding models**: `gemini-embedding-001` (text-only); `gemini-embedding-2` (multimodal — text + image; required for Multimodal File Search).
- **FileSearchStore**: Persistent container for embeddings. Store names are globally scoped. Embeddings persist indefinitely (no TTL) until manually deleted or model deprecated. Raw `File` objects (from Files API) are deleted after 48 hours.
- **Workflow**: (1) Create a FileSearchStore; (2) Upload/import file (auto chunked, embedded, indexed); (3) Query by passing the store as a `file_search` tool in `interactions.create`.

### API functions & parameters

**Tool config object** — passed in the `tools` array of an `interactions.create` request:

| Field | Type | Description |
|---|---|---|
| `type` | string | `"file_search"` |
| `file_search_store_names` | array<string> | List of FileSearchStore resource names to search |
| `metadata_filter` | string | Optional list-filter expression (syntax at google.aip.dev/160), e.g. `author = "Robert Graves"` |

```python
client.interactions.create(
    model="gemini-3.5-flash",
    input="...",
    tools=[{"type": "file_search", "file_search_store_names": ["fileSearchStores/..."]}]
)
```

**FileSearchStore management** (resource: `fileSearchStores`; REST base `https://generativelanguage.googleapis.com/v1beta/fileSearchStores`):

| Operation | REST | Python method | Key parameters |
|---|---|---|---|
| Create | `POST /fileSearchStores` | `client.file_search_stores.create(config=...)` | `display_name` (string), `embedding_model` (string, e.g. `models/gemini-embedding-2`) |
| List | `GET /fileSearchStores` | `client.file_search_stores.list()` | — |
| Get | `GET /fileSearchStores/{name}` | `client.file_search_stores.get(name=...)` | `name` |
| Delete | `DELETE /fileSearchStores/{name}` | `client.file_search_stores.delete(name=..., config={'force': True})` | `name`, `force` (bool) |

Create request body fields: `display_name` (string, human-readable name), `embedding_model` (string, set to `models/gemini-embedding-2` for multimodal).

**File upload / import operations** (two ways):

1. **Direct upload (`uploadToFileSearchStore`)** — resumable upload, no separate Files API file needed:
   - REST: `POST /upload/v1beta/fileSearchStores/{store}:uploadToFileSearchStore` (uses `X-Goog-Upload-*` headers).
   - Python: `client.file_search_stores.upload_to_file_search_store(file=..., file_search_store_name=..., config=...)`.
   - Returns a long-running `operation` (poll via `operations.get` until `done`).

2. **Import existing Files API file (`importFile`)**:
   - REST: `POST /v1beta/fileSearchStores/{store}:importFile` body `{"fileName": "..."}`.
   - Python: `client.file_search_stores.import_file(file_search_store_name=..., file_name=...)`.
   - Also returns a long-running operation.

**Upload config fields**:

| Field | Type | Description |
|---|---|---|
| `display_name` | string | Display name for the imported file |
| `chunking_config` | object | Optional custom chunking config |
| `custom_metadata` | array<{key, string_value/numeric_value}> | Optional key-value metadata for filtering |

`chunking_config` contains `whiteSpaceConfig` with: `maxTokensPerChunk` (int), `maxOverlapTokens` (int).

**File Search Documents API** (`fileSearchStores/{store}/documents`):

| Operation | REST | Python method |
|---|---|---|
| List | `GET /fileSearchStores/{store}/documents` | `client.file_search_stores.documents.list(parent=...)` |
| Get | `GET /fileSearchStores/{store}/documents/{doc}` | `client.file_search_stores.documents.get(name=...)` |
| Delete | `DELETE /fileSearchStores/{store}/documents/{doc}?force=true` | `client.file_search_stores.documents.delete(name=..., config={'force': True})` |

**Response / citation fields** — citations appear in `annotations` of each `model_output` step's text content block.

`file_citation` annotation fields:

| Field | Type | Description |
|---|---|---|
| `type` | string | `"file_citation"` |
| `file_name` | string | Name of the cited source file |
| `source` | string | Source reference/URI |
| `page_number` | int | Page number (for paged docs like PDFs) |
| `media_id` | string | ID of referenced image/media chunk (e.g. `fileSearchStores/{store}/media/{BlobId}`); persistent across calls |
| `custom_metadata` | array<{key, string_value/numeric_value}> | Custom metadata attached to the cited document |

**Media download**: `media_id` from a `file_citation` can be downloaded — REST `GET /v1/fileSearchStores/{store}/media/{BlobId}`; Python `client.file_search_stores.download_media(media_id=...)`; JS `ai.fileSearchStores.downloadMedia(mediaId)`.

### Notes
- **Pricing**: Free at query time; you pay only for embedding creation at index time plus normal model token costs.
- **TTL**: No TTL for embeddings (persist until deleted or model deprecated). Raw `File` objects deleted after 48 hours.
- **Store scope**: FileSearchStore names are globally scoped; create multiple stores to organize documents.
- **Multimodal**: Requires `embedding_model = models/gemini-embedding-2` at store creation; images then uploadable via the same upload/import APIs.
- **Metadata filtering**: Uses list-filter syntax per google.aip.dev/160.
- **Structured output**: Combinable with file search starting with Gemini 3 models.
- Supported models: Gemini 3.5 Flash, Gemini 3.1 Pro Preview, Gemini 3.1 Flash-Lite, Gemini 3 Flash Preview.
- Supported file types: ~250 MIME types including pdf, docx/xlsx/pptx, json, sql, markdown, html, csv, yaml, and many code types (python, java, c, c++, go, rust, kotlin, swift, etc.).

---

## 8. Tool Combination

**Summary** — Lets Gemini combine built-in tools (e.g. `google_search`) with custom function calling in a single interaction by preserving and circulating tool-call context, enabling complex agentic workflows.

### Main concepts
- **Tool context circulation (Gemini 3 models)**: Preserves and exposes built-in tool context and shares it with custom tools within the same interaction.
- **Coordination across environments**: The Interactions API manages context automatically across tool calls. Server-side tools (Google Search, Maps, URL Context, File Search, Code Execution) run on Google's side; client-side tools (Computer Use, custom functions) run on your side. Both have built-in context-circulation solutions.
- **Thought signatures**: Returned `function_call` and `function_response` steps include a `signature` field (an encrypted thought signature). When circulating results back (e.g. via `previous_interaction_id`), you must pass along both `id` and `signature` to maintain tool context — do not strip these fields.
- **Parallel function calls**: Model returns separate steps for built-in tool calls and function calls; the model may emit multiple `function_call` steps.

### API functions & parameters

**Enabling combination support**: There is **no separate combination flag**. Combination is enabled implicitly by passing **both** built-in tools and custom function declarations together in the `tools` array of an Interactions API request.

**Request structure** (`client.interactions.create` / `POST /v1beta/interactions`):

| Field | Purpose |
|---|---|
| `model` | e.g. `gemini-3.5-flash` |
| `input` | User prompt |
| `tools` | Array mixing `{"type": "<built_in>"}` and custom function declarations |

Built-in tool entries: `{"type": "google_search"}`, `{"type": "google_maps"}`, `{"type": "url_context"}`, `{"type": "file_search"}`, `{"type": "code_execution"}`.

Custom function entry: a function declaration object with `type: "function"`, plus `name`, `description`, `parameters`.

REST tip: specify the API revision to avoid breaking changes (uses `/v1beta/interactions`).

**Response flow** — the response is an interaction object containing `steps`. Iterate `interaction.steps`; each step has a `type`:
- `function_call` — the model requesting a custom tool call (has `name`, `arguments`).
- `function_response` — the result you supply back.
- Built-in tool steps (e.g. `code_execution` / `code_execution_result`, search results).

You execute client-side functions and provide results back to the model; the Interactions API manages server-side tool context automatically.

**Critical fields in returned steps**:

| Field | Role |
|---|---|
| `id` | Step identifier; must be echoed back to preserve context |
| `signature` | Encrypted thought signature preserving model reasoning context; must be passed back (e.g. with `previous_interaction_id`) — do not remove |
| `function_call` | Step type carrying custom tool invocation (`name`, `arguments`) |
| `function_response` | Step type carrying your returned custom tool result |
| `thought` | Associated thought content (where present) |

**Tool-specific user-visible data in steps**:

| Tool | Call args | Response |
|---|---|---|
| `google_search` | `queries` | `search_suggestions` |
| `google_maps` | `queries` | `places`, `google_maps_widget_context_token` |
| `url_context` | `urls` (URLs to browse) | `status`, `retrieved_url` |
| `file_search` | None | None |

### Notes
- **Model availability**: Tool context circulation requires **Gemini 3 series** models (example uses `gemini-3.5-flash`). SDK >= 2.0.0 required.
- **Token accounting**: Built-in tool call parts in requests count toward `prompt_token_count` (they're part of conversation history). Responses are not charged this way. Google Search is already priced at the query level, so tokens are not double-charged.
- **Limitations**: `google_search` cannot be combined with a `system_instruction`; instead use `function_declaration.description` to guide behavior.

**Supported tools (context circulation)**:

| Tool | Side | Support |
|---|---|---|
| Google Search | Server | Standard circulation |
| Google Maps | Server | Standard circulation |
| URL Context | Server | Standard circulation |
| File Search | Server | Standard circulation |
| Code Execution | Server | Built-in (`code_execution` / `code_execution_result` steps) |
| Computer Use | Client | Built-in (`function_call` / `function_response` steps) |
| Custom functions | Client | Built-in (`function_call` / `function_response` steps) |

- **Best practice**: Always preserve `id` + `signature` when circulating tool results (e.g. via `previous_interaction_id`); never strip them from the history.