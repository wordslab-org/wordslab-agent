# Google Gemini API Analysis — Text Generation & Conversation Capabilities

> **Base URL:** `https://generativelanguage.googleapis.com/v1beta` | **Docs:** `https://ai.google.dev/gemini-api/docs` | **Auth:** API key (`GEMINI_API_KEY`, header `x-goog-api-key`)
> **SDKs:** `google-genai` (Python / JavaScript / Go / Java / Dart / Swift / Apps Script) | **Playground:** Google AI Studio
> **Description:** Google's Gemini API exposes text and conversation capabilities through two API surfaces — the newer **Interactions API** (`client.interactions.create`, recommended, GA) and the legacy **generateContent API** (`client.models.generate_content`). The Interactions API introduces a first-class **steps** model where each turn is a chronologically-ordered array of typed steps (thoughts, function calls, model outputs, user inputs). The platform covers text generation from prompts and multimodal inputs, thinking/reasoning with controllable levels and signed thought state, structured JSON output via JSON Schema, function calling (parallel + compositional), multi-turn conversation with server-side or client-side state, streaming via SSE, and long-context windows of 1M+ tokens with context caching.

---

## Table of Contents

1. [Platform Overview & Core Concepts](#1-platform-overview--core-concepts)
2. [API Surfaces — Interactions vs generateContent](#2-api-surfaces--interactions-vs-generatecontent)
3. [Text Generation](#3-text-generation)
4. [System Instructions & Configuration](#4-system-instructions--configuration)
5. [Thinking (Reasoning)](#5-thinking-reasoning)
6. [Structured Outputs (JSON Schema)](#6-structured-outputs-json-schema)
7. [Function Calling](#7-function-calling)
8. [Conversation State Management](#8-conversation-state-management)
9. [Streaming](#9-streaming)
10. [Long Context](#10-long-context)
11. [Multimodal Inputs](#11-multimodal-inputs)
12. [Capability Summary & Cross-Reference](#12-capability-summary--cross-reference)

---

## 1. Platform Overview & Core Concepts

### Main Concepts

Google's Gemini text/conversation platform is organized around these core abstractions:

- **Model** — The underlying generative model. Gemini 3.x and 2.5.x series models are "thinking models" that reason internally before responding. Model IDs include `gemini-3.5-flash`, `gemini-3.1-pro-preview`, `gemini-3-flash-preview`, `gemini-2.5-pro`, `gemini-2.5-flash`, `gemini-2.5-flash-lite`.
- **Interaction** — The central object of the Interactions API. An interaction is a single request to the model that returns a chronologically ordered array of **steps**. Each step is a typed unit (thought, function_call, function_result, user_input, model_output, built-in tool call/result).
- **Step** — A first-class typed unit in the `steps` array. Types: `user_input`, `thought`, `function_call`, `function_result`, `model_output`, and built-in tool steps (e.g. `google_search_call`/`google_search_result`). Steps replace the flat `contents`/`parts` structure of the generateContent API with a richer, ordered representation.
- **Input** — The prompt to the model. In the Interactions API, `input` is either a string, a single content-part dict, an array of content parts, or an array of steps (for stateless history replay). In generateContent, `contents` is an array of `Content` objects (each with `role` + `parts`).
- **Content / Part** (generateContent API) — `Content` is a single turn with a `role` (`user` or `model`); `Part` is a piece of data within a turn (text, image, file_data, etc.). The `inline_data` blob holds raw media bytes + MIME type.
- **System Instruction** — A top-level parameter (`system_instruction`) for high-level guidance (persona, behavior rules). Takes priority over user input.
- **Generation Config** — A config object for generation parameters: `temperature`, `top_p`, `top_k`, `max_output_tokens`, `thinking_level`, `thinking_summaries`, `tool_choice`, etc.
- **Response Format** — Configures structured output. Object with `type`, `mime_type`, and `schema` fields.
- **Tool / Function** — External functionality the model can call. Defined by a JSON Schema declaration (`type: "function"`, `name`, `description`, `parameters`). The model emits a `function_call` step; the app executes it and returns a `function_result` step.
- **Thought** — A dedicated step type representing the model's internal reasoning. Contains an encrypted `signature` (always present) and an optional `summary` (text/image content). Signatures are required for reasoning continuity across turns.
- **Context Window** — The maximum tokens usable in a single request. Many Gemini models support 1M+ tokens. No separate reasoning token budget cap; thought tokens count toward total output tokens.
- **Usage** — Token usage metadata including `total_tokens`, `total_input_tokens`, `total_output_tokens`, and `total_thought_tokens`.

### Text & Conversation Tasks

| Task | Description | API |
|------|-------------|-----|
| **Text generation** | Generate text from text/image/video/audio prompts | Interactions / generateContent |
| **Thinking (reasoning)** | Model reasons internally; control level; surface thought summaries | Interactions / generateContent |
| **Structured output** | Force model output to conform to a JSON Schema | Interactions (`response_format`) / generateContent (`generation_config.response_mime_type` + `response_schema`) |
| **Function calling** | Let the model call external functions (parallel + compositional) | Interactions / generateContent |
| **Multi-turn conversation** | Maintain context across multiple turns | Interactions (`previous_interaction_id`) / generateContent (chat SDK or manual `contents`) |
| **Streaming** | Receive generation progress incrementally via SSE | Interactions (`stream=True`) / generateContent (`generate_content_stream` / `streamGenerateContent`) |
| **Long context** | Process 1M+ tokens in a single request | Both (no code changes needed) |
| **Multimodal input** | Combine text with images, video, audio, documents | Both |

### Platform Architecture

```
Interactions API (client.interactions.create):
  input (string | part[] | step[]) + system_instruction + generation_config + tools + response_format
       │
       ▼
     Model ──▶ interaction.steps[] (chronologically ordered typed steps)
                │
       ┌────────┼────────────┬──────────────┬───────────────┬─────────────┐
       ▼        ▼            ▼              ▼               ▼             ▼
    thought   user_input  function_call  function_result  model_output  built-in tool
   (signature+           (name+args)     (name+call_id     (content[]    (google_search
    summary)                              +result[])        text/image)   _call/result)

State management:
  Stateful:  store=true (default) + previous_interaction_id chaining (server manages state)
  Stateless: store=false + pass full step history in input each turn

generateContent API (legacy):
  contents[] (Content{role, parts[]}) + system_instruction + generation_config + tools
       │
       ▼
     Model ──▶ candidates[] (Candidate{content{role, parts[]}, finishReason})
               + usageMetadata
```

---

## 2. API Surfaces — Interactions vs generateContent

The **Interactions API** is Google's recommended API (now GA). The **generateContent API** remains supported as a legacy surface.

### Key Differences

| Concept | generateContent (Legacy) | Interactions (Recommended) |
|---------|--------------------------|----------------------------|
| **Method** | `client.models.generate_content()` / `:generateContent` REST | `client.interactions.create()` |
| **Input** | `contents` — array of `Content` objects (`role` + `parts`) | `input` — string, part, part array, or step array |
| **Output** | `candidates[]` with `content.parts[]` | `interaction.steps[]` — typed, chronologically ordered |
| **Text helper** | `response.text` | `interaction.output_text` (joins consecutive text blocks in last model output) |
| **Thoughts** | No dedicated thought blocks; signatures are metadata attached to parts | First-class `thought` steps; signatures limited to thought steps and built-in tool steps |
| **Function calls** | Parts within candidate content (`functionCall` part) | Dedicated `function_call` steps with `id`, `name`, `arguments` |
| **Function results** | `functionResponse` part in contents | `function_result` steps with `name`, `call_id`, `result[]` |
| **Multi-turn** | SDK `client.chats.create()` or manual `contents` replay | `previous_interaction_id` (server-side state) or manual step replay |
| **Streaming** | `generate_content_stream()` / `streamGenerateContent` (SSE) | `stream=True` with typed `step.start`/`step.delta`/`step.stop` events |
| **System instruction** | `config.system_instruction` | `system_instruction` top-level param |
| **Config object** | `GenerateContentConfig` | `generation_config` dict |
| **Live/real-time** | `BidiGenerateContent` (WebSocket) | Live API (WebSocket, bi-directional streaming) |
| **State storage** | No server-side conversation state (SDK chat is client-side only) | Server-side state via `store=true` + `previous_interaction_id` |

### Interactions API Benefits

- **First-class steps**: Thoughts, function calls, and outputs are dedicated, typed steps — not metadata on parts.
- **Server-side state**: `store=true` + `previous_interaction_id` lets the server manage full conversation history (including thought signatures) automatically.
- **Richer streaming**: Typed SSE events (`interaction.created`, `step.start`, `step.delta`, `step.stop`, `interaction.completed`, `done`) with structured delta types.
- **Simpler signature handling**: In stateful mode, thought signatures are managed entirely server-side.
- **Future-proofed**: All latest features and models land on Interactions first.

---

## 3. Text Generation

### Main Concepts

Text generation is the foundational capability — prompting a model to produce text. The Interactions API returns an `interaction` object whose `steps` array contains the model's response. The `output_text` convenience property joins consecutive text blocks from the last model output step.

**Important**: `output_text` does **not** include text blocks separated by non-text content (thoughts, images, audio, tool calls). For interleaved multimodal responses, manually iterate over `interaction.steps`.

### API Functions & Parameters

#### Interactions API — `client.interactions.create()`

| Parameter | Type | Description |
|-----------|------|-------------|
| `model` | string | **Required.** Model ID (e.g. `gemini-3.5-flash`). |
| `input` | string \| dict \| array | **Required.** A prompt string, a single content-part dict, an array of parts, or an array of steps (for stateless history replay). |
| `system_instruction` | string | High-level system guidance (persona, behavior). |
| `generation_config` | object | Generation parameters (see §4 & §5). |
| `tools` | array | Tool/function declarations and built-in tools (see §7). |
| `response_format` | object | Structured output config (see §6). |
| `previous_interaction_id` | string | ID of a previous interaction to chain context (stateful mode). |
| `store` | boolean | Whether to store the interaction server-side (default: `true`). Set `false` for stateless. |
| `stream` | boolean | Enable streaming. |

#### generateContent API — `client.models.generate_content()`

| Parameter | Type | Description |
|-----------|------|-------------|
| `model` | string | **Required.** Model ID. |
| `contents` | string \| array | **Required.** A string or array of `Content` objects (`role` + `parts`). |
| `config` | `GenerateContentConfig` | Config object with `system_instruction`, `max_output_tokens`, `temperature`, `top_p`, `top_k`, `thinking_config`, `tools`, `tool_config`, `response_mime_type`, `response_schema`, etc. |

**REST endpoints:**
- `POST …/models/{model}:generateContent` — single response
- `POST …/models/{model}:streamGenerateContent` — SSE streaming

### Example (Interactions API)

```python
from google import genai

client = genai.Client()

interaction = client.interactions.create(
    model="gemini-3.5-flash",
    input="How does AI work?"
)
print(interaction.output_text)
```

### Example (generateContent API)

```python
from google import genai
from google.genai import types

client = genai.Client()

response = client.models.generate_content(
    model="gemini-3.5-flash",
    contents="How does AI work?",
    config=types.GenerateContentConfig(
        max_output_tokens=1000
    )
)
print(response.text)
```

### Response Structure (generateContent REST)

```json
{
  "candidates": [
    {
      "content": {
        "parts": [
          { "text": "At its core, Artificial Intelligence works by..." }
        ],
        "role": "model"
      },
      "finishReason": "STOP",
      "index": 0
    }
  ],
  "usageMetadata": { "promptTokenCount": 5, "candidatesTokenCount": 42 }
}
```

### Prompting Tips

- Consult the prompt engineering guide for Gemini-specific strategies.
- For multimodal file prompting, see the file prompting strategies guide.
- With long context, place the query at the **end** of the prompt for best performance.

---

## 4. System Instructions & Configuration

### Main Concepts

System instructions guide model behavior (persona, tone, rules). Generation config controls sampling and output parameters.

### API Parameters

#### System Instruction

| API | Parameter | Location |
|-----|-----------|----------|
| Interactions | `system_instruction` | Top-level param of `interactions.create()` |
| generateContent | `system_instruction` | Field of `GenerateContentConfig` |

```python
# Interactions API
interaction = client.interactions.create(
    model="gemini-3.5-flash",
    system_instruction="You are a cat. Your name is Neko.",
    input="Hello there"
)
```

#### Generation Config Parameters

| Parameter | Type | Description |
|-----------|------|-------------|
| `temperature` | float | Sampling temperature. **Google recommends keeping defaults (1.0) for Gemini 3.x** — changing (e.g. <1.0) can cause looping or degraded reasoning. |
| `top_p` | float | Nucleus sampling. Keep default for Gemini 3.x. |
| `top_k` | int | Top-k sampling. Keep default for Gemini 3.x. |
| `max_output_tokens` | int | Maximum tokens to generate. |
| `thinking_level` | string | Thinking effort: `minimal` / `low` / `medium` / `high` (see §5). |
| `thinking_summaries` | string | Thought summary verbosity: `none` / `auto` (see §5). |
| `tool_choice` | string \| object | Controls tool usage (see §7). |

> **Critical Gemini 3.x note:** Google strongly recommends keeping `temperature`, `top_p`, and `top_k` at their defaults. Setting temperature below 1.0 can cause unexpected behavior (looping, degraded performance) especially in math/reasoning tasks.

---

## 5. Thinking (Reasoning)

### Main Concepts

Gemini 3 and 2.5 series models use a **thinking process** that improves reasoning and multi-step planning. When thinking is enabled (default for most models), the model reasons internally before responding. The Interactions API surfaces this as **thought steps** — dedicated steps in the `steps` array that appear chronologically alongside function calls, user inputs, and model outputs.

**Every thought step contains two fields:**

| Field | Required | Description |
|-------|----------|-------------|
| `signature` | ✅ Yes | Encrypted representation of the model's internal reasoning state. **Always present**, even with minimal reasoning. Required for reasoning continuity across turns. |
| `summary` | ❌ No | Array of content (text and/or images) summarizing the reasoning. May be empty depending on `thinking_summaries` config, whether enough reasoning occurred, or content type (image latents may lack text summaries). |

**Interactions vs generateContent — thought handling:**

| Aspect | generateContent | Interactions |
|--------|-----------------|--------------|
| Thought blocks | No dedicated thought blocks; signatures are metadata attached to any part (e.g. inside `functionCall` parts or the final part) | First-class `thought` steps in the `steps` array |
| Signature locations | Can appear on any part | Limited exclusively to thought steps and built-in tool steps (e.g. `google_search_call`/`google_search_result`). Never on user inputs, model outputs, or standard function calls. |

### Controlling Thinking

By default, Gemini models engage in **dynamic thinking** — automatically adjusting reasoning effort based on request complexity. The `thinking_level` parameter provides explicit control:

| Model | Default Thinking | Levels Supported |
|-------|-----------------|------------------|
| `gemini-3.1-pro-preview` | On (high) | `low`, `medium`, `high` |
| `gemini-3.1-flash-lite-image` | On (minimal) | `minimal`, `high` |
| `gemini-3-flash-preview` | On (high) | `minimal`, `low`, `medium`, `high` |
| `gemini-3-pro-preview` | On (high) | `low`, `high` |
| `gemini-3.5-flash` | On (medium) | `minimal`, `low`, `medium`, `high` |
| `gemini-2.5-pro` | On | `low`, `medium`, `high` |
| `gemini-2.5-flash` | On | `low`, `medium`, `high` |
| `gemini-2.5-flash-lite` | Off | `low`, `medium`, `high` |

### API Parameters

| Parameter | Location | Description |
|-----------|----------|-------------|
| `thinking_level` | `generation_config` (Interactions) / `thinking_config` in `GenerateContentConfig` (generateContent) | Controls reasoning effort: `minimal` / `low` / `medium` / `high`. |
| `thinking_summaries` | `generation_config` (Interactions) | Controls whether thought summaries are returned: `"none"` (default, summaries disabled) / `"auto"` (summaries enabled). |

### Thought Summaries

By default, only the final output is returned (no thought summaries). Enable with `thinking_summaries: "auto"`:

```python
interaction = client.interactions.create(
    model="gemini-3.5-flash",
    input="What is the sum of the first 50 prime numbers?",
    generation_config={"thinking_summaries": "auto"}
)

for step in interaction.steps:
    if step.type == "thought":
        if step.summary:
            for block in step.summary:
                if block.type == "text":
                    print(block.text)
    elif step.type == "model_output":
        for block in step.content:
            if block.type == "text":
                print(block.text)
```

**Handle empty summaries**: A thought step may contain only a signature with no summary when:
- The request was simple (not enough reasoning to generate a summary)
- `thinking_summaries: "none"` (explicitly disabled)
- Certain content types (images) may not have text summaries

### Thought Signatures & Multi-Turn

Thought signatures are **required** to maintain reasoning continuity across multi-turn interactions.

| Mode | Signature Handling |
|------|-------------------|
| **Stateful (recommended)** | `store=true` + `previous_interaction_id`. Server automatically manages all thought blocks and signatures. **No client-side action needed.** |
| **Stateless** | `store=false` + manual history. **Must resend all thought blocks exactly as received.** Do not remove or modify them. When switching models within a session, still resend the previous model's thought blocks — the backend manages compatibility. Built-in tool result signatures (e.g. Google Search) must also be resent. |

### Streaming with Thinking

Thought blocks are delivered via SSE with two distinct delta types:

| Delta type | Contains | When sent |
|------------|----------|-----------|
| `thought_summary` | Text or image summary content | One or more deltas with incremental summary |
| `thought_signature` | The cryptographic signature | The last delta before `step.stop` |

Streaming SSE event sequence example:
```
event: interaction.created
event: step.start          (type: thought, with summary text)
event: step.delta          (type: thought_signature)
event: step.stop
event: step.start          (type: model_output, with text)
event: step.delta          (type: text)
event: step.stop
event: interaction.completed  (includes usage with total_thought_tokens)
event: done
```

### Pricing

When thinking is on, pricing = output tokens + thinking tokens. The `total_thought_tokens` field in usage metadata gives the count. **Pricing is based on full thought tokens generated**, not just the summary output. Use lower `thinking_level` for lengthy outputs to save tokens.

### Best Practices

| Task complexity | Recommended `thinking_level` | Example |
|----------------|------------------------------|---------|
| Simple (fact retrieval, classification) | `minimal` / `low` | "Where was DeepMind founded?" |
| Moderate (comparing concepts, creative reasoning) | Default (`medium`) | "Compare electric and hybrid cars" |
| Complex (coding, math, multi-step planning) | `high` | "Solve AIME math problems" |

- Review thought summaries to understand failures and improve prompts.
- Control thinking budget: prompt the model to think less for lengthy outputs.

---

## 6. Structured Outputs (JSON Schema)

### Main Concepts

Structured Outputs configures Gemini to generate responses that adhere to a supplied **JSON Schema**, ensuring predictable, type-safe results. Ideal for:

- **Data extraction** — Pull specific information (names, dates) from text.
- **Structured classification** — Classify text into predefined categories.
- **Agentic workflows** — Generate structured inputs for tools/APIs.

SDKs support defining schemas via **Pydantic** (Python) and **Zod** (JavaScript), in addition to raw JSON Schema in the REST API.

### API Parameters

#### Interactions API — `response_format`

| Field | Type | Description |
|-------|------|-------------|
| `type` | string | `"text"` |
| `mime_type` | string | `"application/json"` |
| `schema` | object | JSON Schema object defining the output structure. |

```python
interaction = client.interactions.create(
    model="gemini-3.5-flash",
    input=prompt,
    response_format={
        "type": "text",
        "mime_type": "application/json",
        "schema": Recipe.model_json_schema()
    },
)
recipe = Recipe.model_validate_json(interaction.output_text)
```

#### generateContent API — `GenerateContentConfig`

| Field | Description |
|-------|-------------|
| `response_mime_type` | Set to `"application/json"`. |
| `response_schema` | JSON Schema object (or Pydantic/Zod schema via SDK). |

### Supported JSON Schema Subset

**Supported `type` values:**

| Type | Description |
|------|-------------|
| `string` | Text |
| `number` | Floating-point numbers |
| `integer` | Whole numbers |
| `boolean` | True/false |
| `object` | Structured key-value pairs |
| `array` | Lists of items |
| `null` | Allow null via type array: `{"type": ["string", "null"]}` |

**Descriptive properties:** `title`, `description`.

**Type-specific constraints:**

| Type | Supported constraints |
|------|-----------------------|
| **object** | `properties`, `required`, `additionalProperties` (boolean or schema) |
| **string** | `enum`, `format` (`date-time`, `date`, `time`) |
| **number / integer** | `enum`, `minimum`, `maximum` |
| **array** | `items`, `prefixItems` (tuple-like), `minItems`, `maxItems` |

**Supported features:**
- `anyOf` — conditional schemas (for union types; see Content Moderation example)
- Recursive schemas — self-referencing (e.g. organization charts; see Recursive Structures example)

### Schema Examples

**Basic extraction (Recipe):**
```python
class Ingredient(BaseModel):
    name: str = Field(description="Name of the ingredient.")
    quantity: str = Field(description="Quantity, including units.")

class Recipe(BaseModel):
    recipe_name: str = Field(description="The name of the recipe.")
    prep_time_minutes: Optional[int] = Field(description="Optional prep time.")
    ingredients: List[Ingredient]
    instructions: List[str]
```

**Conditional schema (anyOf) — Content Moderation:**
```python
class SpamDetails(BaseModel):
    reason: str
    spam_type: Literal["phishing", "scam", "unsolicited promotion", "other"]

class NotSpamDetails(BaseModel):
    summary: str
    is_safe: bool

class ModerationResult(BaseModel):
    decision: Union[SpamDetails, NotSpamDetails]  # anyOf
```

**Recursive schema — Organization chart:**
```python
class Employee(BaseModel):
    name: str
    employee_id: int
    reports: List["Employee"] = Field(default_factory=list)
```

### Streaming Structured Outputs

Streamed chunks are **valid partial JSON strings** that can be concatenated to form the final JSON object:

```python
stream = client.interactions.create(
    model="gemini-3.5-flash",
    input=prompt,
    response_format={...},
    stream=True
)
for event in stream:
    if event.event_type == "step.delta":
        if event.delta.type == "text" and event.delta.text:
            print(event.delta.text, end="", flush=True)
```

### Structured Outputs with Tools (Gemini 3 only — Preview)

Gemini 3 models can combine Structured Outputs with built-in tools (Google Search, URL Context, Code Execution, File Search) and Function Calling in the same request:

```python
interaction = client.interactions.create(
    model="gemini-3.1-pro-preview",
    input="Search for all details for the latest Euro.",
    tools=[{"type": "google_search"}, {"type": "url_context"}],
    response_format={...},
)
```

### Structured Outputs vs Function Calling

| Feature | Primary Use Case |
|---------|-----------------|
| **Structured Outputs** | Formatting the final response. Use when you want the model's answer in a specific format. |
| **Function Calling** | Taking action during conversation. Use when the model needs to ask you to perform a task before providing a final answer. |

### Best Practices & Limitations

- Use `description` fields to guide the model.
- Use specific types (`integer`, `string`, `enum`).
- Always validate values in your application — output is syntactically correct JSON but may be semantically incorrect.
- **Limitations:** Not all JSON Schema features supported; very large or deeply nested schemas may be rejected.

---

## 7. Function Calling

### Main Concepts

Function calling connects models to external tools and APIs. The model determines when to call functions and provides the necessary parameters. Three primary use cases:

1. **Take Actions** — Interact with external systems (schedule appointments, send emails, control devices).
2. **Augment Knowledge** — Access external data (databases, APIs, knowledge bases).
3. **Extend Capabilities** — Perform computations, create charts, extend model limitations.

### The Function Calling Flow (4 Steps)

1. **Define function declaration** — name, parameters, purpose.
2. **Call model with declarations** — send user prompt + function declarations.
3. **Execute function code** (your responsibility) — extract name and args, execute in your app.
4. **Send result back to model** — for a final, user-friendly response.

This can repeat over multiple turns. The model supports **parallel** (multiple functions in one turn) and **compositional** (chained) function calling.

### Function Declaration Schema

| Field | Description |
|-------|-------------|
| `type` | Must be `"function"` for custom functions. |
| `name` | Unique function name (use underscores or camelCase; no spaces/special chars). |
| `description` | Clear explanation of the function's purpose. |
| `parameters` | JSON Schema object: `type: "object"`, `properties` (individual params with type + description), `required` (mandatory param names). |

```python
set_light_values_declaration = {
    "type": "function",
    "name": "set_light_values",
    "description": "Sets the brightness and color temperature of a light.",
    "parameters": {
        "type": "object",
        "properties": {
            "brightness": {"type": "integer", "description": "Light level from 0 to 100"},
            "color_temp": {"type": "string", "enum": ["daylight", "cool", "warm"], "description": "Color temperature"},
        },
        "required": ["brightness", "color_temp"],
    },
}
```

### Handling Function Calls (Interactions API)

**Step 2 — Receive function call:**
```python
interaction = client.interactions.create(
    model="gemini-3.5-flash",
    input="Turn the lights down to a romantic level",
    tools=[set_light_values_declaration],
)
fc_step = next(s for s in interaction.steps if s.type == "function_call")
# fc_step.name = 'set_light_values'
# fc_step.arguments = {'color_temp': 'warm', 'brightness': 25}
```

**Step 4 — Send result back (stateful):**
```python
final_interaction = client.interactions.create(
    model="gemini-3.5-flash",
    input=[{
        "type": "function_result",
        "name": fc_step.name,
        "call_id": fc_step.id,
        "result": [{"type": "text", "text": json.dumps(result)}],
    }],
    tools=[set_light_values_declaration],
    previous_interaction_id=interaction.id,
)
print(final_interaction.output_text)
```

### `function_result` Step Fields

| Field | Description |
|-------|-------------|
| `type` | `"function_result"` |
| `name` | Function name |
| `call_id` | The `id` from the corresponding `function_call` step |
| `result` | Array of content blocks (text, image, etc.) |

### Function Calling Modes (`tool_choice`)

Control how the model uses tools via `tool_choice` in `generation_config`:

| Value | Behavior |
|-------|----------|
| `"auto"` (default) | Model decides whether to call a function or respond directly. |
| `"any"` | Model is constrained to always predict a function call. |
| `"none"` | Model is prohibited from making function calls. |
| `"validated"` (Preview) | Model ensures function schema adherence. |
| `{"allowed_tools": {"mode": "any", "tools": ["func_name"]}}` | Restrict to a subset of tools. |

```python
generation_config = {
    "tool_choice": {
        "allowed_tools": {
            "mode": "any",
            "tools": ["get_current_temperature"]
        }
    }
}
```

### Parallel Function Calling

Call multiple independent functions at once:

```python
interaction = client.interactions.create(
    model="gemini-3.5-flash",
    input="Turn this place into a party!",
    tools=[power_disco_ball, start_music, dim_lights],
    generation_config={"tool_choice": "any"},
)
# Multiple function_call steps returned
```

### Compositional Function Calling

Chain function calls — the model calls one function, uses its result to decide the next call (e.g. get weather first, then set thermostat based on result). The model handles this automatically within a single interaction's steps.

### Function Calling with Thinking Models

Gemini 3 series models use internal thinking to improve function calling. The SDKs automatically handle thought signatures. In stateless mode, thought steps must be preserved and resent alongside function call/result steps.

### Multi-Tool Use (Built-in + Custom)

Gemini 3 models can combine built-in tools (Google Search, URL Context, etc.) with custom function calling in the same request:

```python
tools = [
    {"type": "google_search"},
    get_weather_function
]
interaction = client.interactions.create(
    model="gemini-3.5-flash",
    input="What is the northernmost city in the US? What's the weather there?",
    tools=tools
)
```

Passing `previous_interaction_id` automatically circulates the built-in tool context.

### Multimodal Function Responses (Gemini 3)

Function responses can include multimodal content (images) in the `result` field:

```python
input=[{
    "type": "function_result",
    "name": tool_call.name,
    "call_id": tool_call.id,
    "result": [
        {"type": "text", "text": "instrument.jpg"},
        {"type": "image", "mime_type": "image/jpeg", "data": base64_image_data},
    ],
}]
```

### Remote MCP (Model Context Protocol)

The Interactions API supports connecting to remote MCP servers for external tools/services.

| Field | Type | Required | Description |
|-------|------|----------|-------------|
| `type` | string | Yes | Must be `"mcp_server"`. |
| `name` | string | No | Display name (no `-` character; use snake_case). |
| `url` | string | No | Full URL for the MCP server endpoint. |
| `headers` | object | No | Key-value HTTP headers (e.g. auth tokens). |
| `allowed_tools` | array | No | Restrict which tools the model may call. |

**Constraints:** Only Streamable HTTP servers supported (no SSE servers). Server names must not include `-`.

### Streaming Function Calls

Tool calls arrive as `step.delta` events with `arguments` delta type. Aggregate partial arguments to reconstruct complete calls:

| Event | Delta type | Content |
|-------|------------|---------|
| `step.start` | — | `step.type == "function_call"` with `id`, `name`, initial `arguments` |
| `step.delta` | `arguments` | `delta.partial_arguments` (append to aggregate) |
| `step.delta` | `text` | Text output (if any) |
| `interaction.completed` | — | Final aggregated tool calls ready to execute |

### Best Practices

- **Descriptions**: Be clear and specific in function and parameter descriptions.
- **Naming**: Descriptive names without spaces or special characters.
- **Strong typing**: Use specific types (`integer`, `string`, `enum`).
- **Tool selection**: Keep active set to **10–20 tools maximum**.
- **Prompt engineering**: Provide context and instructions.
- **Validation**: Validate function calls before executing.
- **Error handling**: Implement robust error handling.
- **Security**: Use appropriate authentication for external APIs.

### Notes & Limitations

- Only a subset of the OpenAPI Schema is supported.
- Very large or deeply nested schemas may be rejected.
- Supported parameter types in Python are limited.

---

## 8. Conversation State Management

### Main Concepts

Each generation request is independent. The Interactions API provides server-side state management via `previous_interaction_id`, while the generateContent API relies on client-side history management.

### Method 1: Stateful — `previous_interaction_id` (Interactions API, Recommended)

The server automatically manages conversation history, including all thought blocks and signatures. Each turn is a separate interaction chained by ID.

```python
interaction1 = client.interactions.create(
    model="gemini-3.5-flash",
    input="I have 2 dogs in my house.",
)

interaction2 = client.interactions.create(
    model="gemini-3.5-flash",
    input="How many paws are in my house?",
    previous_interaction_id=interaction1.id,
)
```

**Key points:**
- Requires `store=true` (default).
- Server manages all state — no need to handle thought signatures.
- Streaming can be combined with `previous_interaction_id`.

### Method 2: Stateless — Manual History (Interactions API)

Opt out of server-side storage with `store=false` and manage history client-side:

1. Set `store=false`.
2. Maintain conversation history as an array of steps.
3. Pass accumulated steps in `input`, append new turn as `user_input` step.

```python
history = [
    {"type": "user_input", "content": [{"type": "text", "text": "I have 2 dogs."}]}
]

interaction1 = client.interactions.create(
    model="gemini-3.5-flash",
    store=False,
    input=history,
)

for step in interaction1.steps:
    history.append(step.model_dump())  # preserve ALL steps incl. thoughts

history.append({
    "type": "user_input",
    "content": [{"type": "text", "text": "How many paws?"}]
})

interaction2 = client.interactions.create(
    model="gemini-3.5-flash",
    store=False,
    input=history,
)
```

**Critical**: If the model uses thinking or tools, you **must** preserve and resend all model-generated steps (thought, function_call) **exactly as received** — they contain signatures required to continue. Built-in tool result signatures must also be resent.

### Method 3: Chat SDK (generateContent API)

The SDKs provide a `client.chats.create()` interface that manages the `contents` array client-side. Behind the scenes, it uses generateContent and sends the full history each turn.

```python
chat = client.chats.create(model="gemini-3.5-flash")
response = chat.send_message("I have 2 dogs in my house.")
response = chat.send_message("How many paws are in my house?")

for message in chat.get_history():
    print(f'{message.role}: {message.parts[0].text}')
```

Streaming variant: `chat.send_message_stream()`.

### Method 4: Manual Contents Replay (generateContent REST)

For the REST API, maintain `contents` as an array of `Content` objects, alternating `role: "user"` and `role: "model"`:

```json
{
  "contents": [
    {"role": "user", "parts": [{"text": "Hello."}]},
    {"role": "model", "parts": [{"text": "Hello! How can I help?"}]},
    {"role": "user", "parts": [{"text": "Write a poem about the ocean."}]}
  ]
}
```

### State Management Decision Matrix

| Scenario | Recommended Approach |
|----------|---------------------|
| Simple multi-turn (Interactions API) | `previous_interaction_id` with `store=true` |
| Full control over context (trimming, modification) | Stateless: `store=false` + manual step replay |
| Stateless + thinking/tools | `store=false` + resend all steps exactly as received |
| generateContent API | SDK `client.chats.create()` or manual `contents` replay |

---

## 9. Streaming

### Main Concepts

Both APIs support streaming to surface generation progress incrementally.

### Interactions API Streaming

Set `stream=True`. Returns typed SSE events. Consumers must branch on `event.event_type`:

| Event Type | Description |
|------------|-------------|
| `interaction.created` | Interaction object created (with `id`, `status`, `model`) |
| `step.start` | A new step begins (includes `index` and step data: `type`, initial content) |
| `step.delta` | Incremental delta within a step (delta `type`: `text`, `thought_summary`, `thought_signature`, `arguments`) |
| `step.stop` | Step complete |
| `interaction.completed` | Interaction finished (includes `usage` with token counts) |
| `done` | Stream end marker (`[DONE]`) |

**Delta types:**

| Delta type | Content |
|------------|---------|
| `text` | `delta.text` — incremental text output |
| `thought_summary` | `delta.content.text` — incremental thought summary |
| `thought_signature` | `delta.signature` — cryptographic signature (last delta before `step.stop`) |
| `arguments` | `delta.partial_arguments` — incremental function call arguments |

```python
stream = client.interactions.create(
    model="gemini-3.5-flash",
    input="Explain how AI works",
    stream=True
)
for event in stream:
    if event.event_type == "step.delta":
        if event.delta.type == "text":
            print(event.delta.text, end="")
```

### generateContent API Streaming

**SDK:** `client.models.generate_content_stream()` — iterates over `GenerateContentResponse` chunks.

```python
response = client.models.generate_content_stream(
    model="gemini-3.5-flash",
    contents=["Explain how AI works"]
)
for chunk in response:
    print(chunk.text, end="")
```

**REST:** `streamGenerateContent` — SSE stream of `GenerateContentResponse` instances, each with a `responseId` tying the stream together.

### Live API (BidiGenerateContent)

A stateful **WebSocket**-based API for bi-directional real-time streaming (see Live API guide).

---

## 10. Long Context

### Main Concepts

Many Gemini models have context windows of **1 million or more tokens**. The code for text generation and multimodal inputs works without changes with long context — it's the same API, just with more input.

**What 1M tokens looks like:**
- 50,000 lines of code (80 chars/line)
- All text messages sent in the last 5 years
- 8 average-length English novels
- Transcripts of 200+ average podcast episodes

### Paradigm Shift

Historically, limited context required strategies like dropping old messages, summarizing content, or RAG with vector databases. Gemini's large context invites a **direct approach**: provide all relevant information upfront. Gemini models demonstrate powerful **in-context learning** — e.g. learning to translate English to Kalamang (a Papuan language with <200 speakers) from in-context materials alone.

### Long Context Use Cases

#### Long form text
- **Summarizing large corpuses** — No sliding window needed; pass everything at once.
- **Question & answering** — Over large document sets without RAG.
- **Agentic workflows** — More context about the world and agent goals improves reliability.
- **Many-shot in-context learning** — Scale from few-shot to hundreds/thousands of examples. Research shows many-shot can perform similarly to fine-tuned models for specific tasks.

#### Long form video
- Video Q&A, video memory (Project Astra), video captioning, recommendation systems, customization, content moderation, real-time processing.
- Token processing for video affects billing and usage limits.

#### Long form audio
- Real-time transcription/translation, podcast/video Q&A, meeting summarization, voice assistants.
- Native audio understanding (no separate speech-to-text + text-to-text pipeline needed).

### Long Context Optimizations

**Context caching** is the primary optimization:
- Cache uploaded files (PDFs, videos, documents) and pay to store them on a per-hour basis.
- Input/output cost per request with cached context is **~4x less** than standard cost (Gemini Flash example).
- Significant savings for "chat with your data" apps where users query the same context repeatedly.

### Long Context Limitations

- **Multi-needle retrieval**: Single needle-in-a-haystack retrieval achieves ~99% accuracy, but performance degrades with multiple needles to find.
- **Cost tradeoff**: You pay input token cost every time you send a query. For 100 pieces of information at 99% accuracy, you'd send 100 requests — context caching significantly reduces this cost.
- **Latency**: Longer queries have higher latency (time to first token), though there's some fixed latency regardless of size.

### FAQs

| Question | Answer |
|----------|--------|
| Best place for query in context? | **At the end** of the prompt (after all other context), especially for long contexts. |
| Performance with more tokens? | If you don't need tokens, avoid passing them. But the model is highly capable of extracting specific info (up to 99% accuracy). |
| Lower cost? | Use **context caching** for reused context. |
| Latency impact? | Longer queries = higher latency (time to first token). |

---

## 11. Multimodal Inputs

### Main Concepts

The Gemini API supports multimodal inputs — combining text with images, video, audio, and documents. The same text generation API handles multimodal inputs without separate endpoints.

### Input Methods

| Method | Description |
|--------|-------------|
| **File API upload** | Upload file → reference by `uri` + `mime_type`. Best for larger/reusable files. |
| **inline_data** | Base64-encoded bytes + MIME type directly in the part. Best for small images. |
| **file_data** | Reference uploaded file by URI in a part. |

### Example (Interactions API with image)

```python
uploaded_file = client.files.upload(file="path/to/organ.jpg")

interaction = client.interactions.create(
    model="gemini-3.5-flash",
    input=[
        {"type": "text", "text": "Tell me about this instrument"},
        {"type": "image", "uri": uploaded_file.uri, "mime_type": uploaded_file.mime_type}
    ]
)
```

### Example (generateContent with inline image)

```python
image = Image.open("/path/to/organ.png")
response = client.models.generate_content(
    model="gemini-3.5-flash",
    contents=[image, "Tell me about this instrument"]
)
```

### REST inline_data structure

```json
{
  "contents": [{
    "parts": [
      {"inline_data": {"mime_type": "image/jpeg", "data": "<base64>"}},
      {"text": "What is in this picture?"}
    ]
  }]
}
```

### Supported Media Types

Text, images, video, audio, and documents (PDFs, etc.). See the respective understanding guides (image, video, audio, document processing) and file prompting strategies.

---

## 12. Capability Summary & Cross-Reference

### Complete Capability Matrix

| Capability | generateContent (Legacy) | Interactions (Recommended) | Section |
|-----------|--------------------------|----------------------------|---------|
| Text generation | Yes | Yes (recommended) | §3 |
| System instructions | `config.system_instruction` | `system_instruction` param | §4 |
| Temperature / top_p / top_k | Yes (keep defaults for Gemini 3.x) | Yes (keep defaults) | §4 |
| `max_output_tokens` | Yes | Yes | §4 |
| Thinking (reasoning) | Yes (signatures as part metadata) | Yes (first-class thought steps) | §5 |
| Thinking level control | `thinking_config.thinking_level` | `generation_config.thinking_level` | §5 |
| Thought summaries | Limited | `thinking_summaries: "auto"` | §5 |
| Thought signatures | Part metadata (manual handling) | Thought step field (server-managed in stateful mode) | §5 |
| Structured Outputs (JSON Schema) | `response_mime_type` + `response_schema` | `response_format` with `schema` | §6 |
| Pydantic / Zod schema support | Yes (SDK) | Yes (SDK) | §6 |
| Structured Outputs + tools | — | Yes (Gemini 3, preview) | §6 |
| Function calling | Yes | Yes | §7 |
| Parallel function calling | Yes | Yes | §7 |
| Compositional function calling | Yes | Yes | §7 |
| `tool_choice` modes | `auto` / `any` / `none` | `auto` / `any` / `none` / `validated` / `allowed_tools` | §7 |
| Multimodal function responses | Yes (Gemini 3) | Yes (Gemini 3) | §7 |
| Remote MCP | — | Yes (`mcp_server` tool type) | §7 |
| Multi-tool (built-in + custom) | — | Yes (Gemini 3) | §7 |
| Server-side conversation state | No (SDK chat is client-side) | Yes (`previous_interaction_id` + `store=true`) | §8 |
| Stateless mode | Manual `contents` replay | `store=false` + step replay | §8 |
| Streaming | `generate_content_stream` / `streamGenerateContent` | `stream=True` (typed SSE events) | §9 |
| Live/real-time streaming | `BidiGenerateContent` (WebSocket) | Live API (WebSocket) | §9 |
| Long context (1M+ tokens) | Yes (no code changes) | Yes (no code changes) | §10 |
| Context caching | Yes | Yes | §10 |
| Multimodal inputs (image/video/audio/docs) | Yes | Yes | §11 |
| File API upload | Yes | Yes | §11 |
| `output_text` helper | `response.text` | `interaction.output_text` | §3 |

### State Management Decision Matrix

| Scenario | Recommended Approach |
|----------|---------------------|
| Simple multi-turn (Interactions) | `previous_interaction_id` + `store=true` |
| Full control / context trimming | `store=false` + manual step replay |
| Stateless + thinking/tools | `store=false` + resend all steps exactly (incl. thoughts) |
| generateContent API | SDK `client.chats.create()` or manual `contents` |

### Thinking Level Decision Matrix

| Task complexity | `thinking_level` | Example |
|----------------|------------------|---------|
| Simple (fact retrieval, classification) | `minimal` / `low` | "Where was DeepMind founded?" |
| Moderate (comparison, creative reasoning) | Default (`medium`) | "Compare electric and hybrid cars" |
| Complex (coding, math, multi-step planning) | `high` | "Solve AIME math problems" |

### Structured Output vs Function Calling Decision Matrix

| Need | Recommended Feature |
|------|---------------------|
| Format the model's final response | Structured Outputs (`response_format` with schema) |
| Take action during conversation | Function Calling |
| Both (Gemini 3) | Combine Structured Outputs + Function Calling |

### Tool Type Decision Matrix

| Need | Tool Type |
|------|-----------|
| Custom external function | `{"type": "function", ...}` |
| Web search | `{"type": "google_search"}` |
| URL content fetching | `{"type": "url_context"}` |
| Code execution | `{"type": "code_execution"}` |
| File search | `{"type": "file_search"}` |
| Remote MCP server | `{"type": "mcp_server", "name": ..., "url": ...}` |

### Key Differences from OpenAI

| Aspect | Google Gemini | OpenAI |
|--------|---------------|--------|
| Recommended API | Interactions API (`interactions.create`) | Responses API (`/v1/responses`) |
| Legacy API | generateContent (`models.generate_content`) | Chat Completions (`/v1/chat/completions`) |
| Output structure | `steps[]` (typed, chronological) | `output[]` (typed Items) |
| Reasoning | Thinking steps with `signature` + `summary`; levels: minimal/low/medium/high | Reasoning Items with `effort` (low/medium/high) + `summary` |
| Reasoning continuity | Thought signatures (encrypted, resent in stateless) | Encrypted reasoning content (stateless) |
| Server-side state | `previous_interaction_id` + `store=true` | `previous_response_id` + `store=true` / Conversations API |
| Structured output | `response_format` with `schema` (no `strict` flag) | `text.format` with `json_schema` + `strict: true` |
| Function definition | Flat: `type` + `name` + `description` + `parameters` | Responses: flat (internally tagged); Chat: nested (externally tagged) |
| Tool choice | `auto` / `any` / `none` / `validated` / `allowed_tools` | `auto` / `required` / `none` / specific function / `allowed_tools` |
| Temperature guidance | **Keep defaults (1.0) for Gemini 3.x** | Configurable |
| Long context | 1M+ tokens native; context caching | 128k typical; compaction |
| Multimodal | Native text/image/video/audio/document | Text/image/PDF/file |