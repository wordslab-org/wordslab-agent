# Mistral API Analysis — Text Generation & Conversation Capabilities

> **Base URL:** `https://api.mistral.ai/v1` | **Docs:** `https://docs.mistral.ai/studio-api/overview` | **API Spec:** `https://docs.mistral.ai/api` | **Auth:** Bearer token (`Authorization: Bearer $MISTRAL_API_KEY`)
> **SDKs:** `mistralai` (Python / TypeScript) — also raw REST/curl
> **Description:** Mistral exposes text and conversation capabilities primarily through the **Chat Completions API** (`POST /v1/chat/completions`), a single unified surface for text generation, vision, reasoning, structured outputs, and citations. The platform covers chat from text/image inputs, adjustable reasoning effort, custom JSON Schema outputs and JSON mode, citation/reference grounding via tool-call responses, multi-turn conversations (manual state replay), streaming, an asynchronous batch processing API at a 50% discount, a dedicated moderation API with inline guardrails, and a managed RAG stack (Libraries + Embeddings API). Conversations and Agents APIs (`/v1/conversations`, beta) offer a higher-level managed alternative for multi-turn + tool orchestration, but the foundational text capabilities all flow through Chat Completions.

---

## Table of Contents

1. [Platform Overview & Core Concepts](#1-platform-overview--core-concepts)
2. [The Chat Completions API — Core Request/Response Schema](#2-the-chat-completions-api--core-requestresponse-schema)
3. [Text Generation](#3-text-generation)
4. [Message Roles & Instruction Hierarchy](#4-message-roles--instruction-hierarchy)
5. [Multi-turn Conversations](#5-multi-turn-conversations)
6. [Reasoning (Adjustable Thinking)](#6-reasoning-adjustable-thinking)
7. [Vision (Image Inputs)](#7-vision-image-inputs)
8. [Structured Outputs (Custom Schema & JSON Mode)](#8-structured-outputs-custom-schema--json-mode)
9. [Citations & References (Document Grounding)](#9-citations--references-document-grounding)
10. [Streaming](#10-streaming)
11. [Moderation & Guardrailing](#11-moderation--guardrailing)
12. [Batch Processing](#12-batch-processing)
13. [Embeddings API](#13-embeddings-api)
14. [RAG — From Scratch](#14-rag--from-scratch)
15. [Libraries (Managed Knowledge Base / RAG)](#15-libraries-managed-knowledge-base--rag)
16. [Capability Summary & Cross-Reference](#16-capability-summary--cross-reference)

---

## 1. Platform Overview & Core Concepts

### Main Concepts

Mistral's text/conversation platform is organized around these core abstractions:

- **Model** — The underlying large language model. Multiple families are offered: **Mistral Large 3** (`mistral-large-2512` / `mistral-large-latest`), **Mistral Medium 3.1** (`mistral-medium-2508` / `mistral-medium-3-5`), **Mistral Small 3.2** (`mistral-small-2506` / `mistral-small-latest`), and the **Ministral 3** line (`ministral-14b-2512`, `ministral-8b-2512`, `ministral-3b-2512`). Some models support vision; some support adjustable reasoning. A `mistral-moderation-2603` model powers moderation/classification.
- **Chat Completion** — The core request/response cycle: send a list of `messages`, receive a generated assistant message. Accessed via `client.chat.complete(...)` (synchronous), `client.chat.stream(...)` (streaming), or `client.chat.parse(...)` (structured output helper).
- **Message** — A unit of conversation context with a `role` (`system`, `user`, `assistant`, or `tool`) and `content`. Content can be a plain `string` **or** a list of typed chunks (`TextChunk`, `ThinkChunk`, `ReferenceChunk`, image parts, etc.) when advanced features are used.
- **Content Chunk** — A typed element within a message's `content` list. Types include `text` (`TextChunk`), `thinking` (`ThinkChunk`, from reasoning), `reference` (`ReferenceChunk`, citations), and `image_url` (vision inputs). The chunked form appears when reasoning, citations, tool calls, or vision are involved.
- **System Message** — An optional first message (`role: "system"`) that sets behavior, personality, and instructions. Equivalent to OpenAI's `system` role.
- **Response** — The object returned by the API (`object: "chat.completion"`). Contains `choices[]` (each with `index`, `finish_reason`, `message`), plus `id`, `created`, `model`, and `usage`. `choices[0].message.content` holds the generated output.
- **Tool Call** — When function calling is enabled, the assistant message may include `tool_calls[]` (each with `id`, `type: "function"`, `function: { name, arguments }`). A `tool`-role message carries the result back. (Function calling is covered in the Agents/Tools study; citations reuse this mechanism.)
- **Reasoning Effort** — A top-level `reasoning_effort` parameter (`"high"` or `"none"`) controlling whether the model emits a thinking trace before the final answer. Only on select models (`mistral-small-latest`, `mistral-medium-3-5`).
- **Guardrails** — Inline moderation rules passed in a `guardrails` array on chat/conversation requests; evaluated on input before the model runs, blocking (HTTP 403) on violation. Backed by `mistral-moderation-2603`.
- **Library** — A persistent knowledge base (managed RAG): upload documents, Mistral ingests/vectorizes/searches them, and agents (or conversations) query them via a built-in `document_library` tool.
- **Batch Job** — An asynchronous job that runs up to 1M requests against a single endpoint at a 50% cost discount. Managed via `client.batch.jobs.*` and `client.files.*`.

### Text & Conversation Tasks

| Task | Description | API |
|------|-------------|-----|
| **Text generation** | Generate text from a prompt | Chat Completions (`/v1/chat/completions`) |
| **Multi-turn conversation** | Maintain context across multiple messages | Chat Completions (manual replay) / Conversations API (beta) |
| **Reasoning** | Model emits a thinking trace before answering | Chat Completions (`reasoning_effort`) |
| **Vision** | Analyze images alongside text | Chat Completions (image_url content parts) |
| **Structured output** | Force JSON conforming to a schema | `client.chat.parse` (custom) / `response_format` (JSON mode) |
| **Citations / references** | Ground answers in provided source documents | Chat Completions (tool-call flow + ReferenceChunk) |
| **Streaming** | Receive generation progress incrementally | `client.chat.stream` |
| **Moderation** | Classify text / guardrail requests | Moderation API + inline `guardrails` |
| **Batch inference** | Async large-volume inference at a discount | Batch API (`/v1/batch/jobs`) |
| **Embeddings** | Vectorize text for retrieval/clustering | Embeddings API (`/v1/embeddings`) |
| **Managed RAG** | Persistent knowledge base with built-in search | Libraries API (beta) |

### Platform Architecture

```
Chat Completions API (POST /v1/chat/completions):
  messages[] ──▶ Model ──▶ choices[].message
                                │
                content = string  (standard)
                content = [chunks] (reasoning / citations / vision / tool calls)

Advanced request parameters:
  reasoning_effort: "high" | "none"        → emits ThinkChunk(s) before TextChunk
  response_format:  custom schema | json_object
  tools / tool_choice                          (function calling — agents study)
  guardrails: [{ moderation_llm_v2: {...} }]   (inline input moderation)
  safe_prompt: bool                            (legacy moderation flag)
  prefix: bool (on assistant message)          (prepend to model output)
  stop: [strings]                              (stop sequences)

Surrounding services:
  Moderation API   (POST /v1/moderations, /v1/chat/moderations)  — classify text/conversations
  Batch API        (POST /v1/batch/jobs)                          — async discounted jobs
  Embeddings API   (POST /v1/embeddings)                          — mistral-embed vectors
  Libraries API    (beta.libraries.*)                             — managed RAG knowledge base
  Conversations API (beta.conversations.start)                    — higher-level multi-turn + agents
```

---

## 2. The Chat Completions API — Core Request/Response Schema

Mistral's primary text/conversation surface is a single Chat Completions endpoint, consistent with the OpenAI Chat Completions shape but extended with Mistral-specific parameters (`reasoning_effort`, `safe_prompt`, `guardrails`, `prefix`).

### Endpoint

`POST /v1/chat/completions` (API spec tag: `chat`)

### Request Parameters

| Parameter | Type | Description |
|-----------|------|-------------|
| `model` | string | **Required.** Model ID (e.g. `mistral-large-latest`, `mistral-small-latest`, `mistral-medium-3-5`). |
| `messages` | array | **Required.** List of messages, each with `role` + `content` (string or chunk list). |
| `temperature` | number | Sampling temperature (e.g. `0`, `0.2`, `0.8`). |
| `max_tokens` | integer | Maximum generated tokens. |
| `top_p` | number | Nucleus sampling. |
| `stream` | boolean | Enable streaming. |
| `reasoning_effort` | string | `"high"` or `"none"` — controls thinking trace (select models). See §6. |
| `response_format` | object | Structured output: `{"type": "json_object"}` or a custom schema. See §8. |
| `tools` | array | Function definitions (function calling — covered in agents/tools study). |
| `tool_choice` | string \| object | Controls tool usage. |
| `guardrails` | array | Inline moderation rules. See §11. |
| `safe_prompt` | boolean | Legacy flag to force moderation on the prompt. See §11. |
| `stop` | array of strings | Stop sequences; output excludes the stop string. |
| `random_seed` | integer | Seed for reproducibility. |
| `metadata` | object | Optional request metadata. |

### Message Object

| Field | Type | Description |
|-------|------|-------------|
| `role` | string | `"system"`, `"user"`, `"assistant"`, or `"tool"`. |
| `content` | string \| array | Plain string, or a list of typed chunks (`text`, `thinking`, `reference`, `image_url`, ...). |
| `prefix` | boolean | (assistant role) If `true`, the model's response is prepended with this content. |
| `tool_calls` | array | (assistant role) Tool calls emitted by the model. |
| `name` | string | (tool role) Name of the called function. |
| `tool_call_id` | string | (tool role) ID of the tool call this result responds to. |

### Response Object

```json
{
  "id": "cmpl-...",
  "object": "chat.completion",
  "created": 1702256327,
  "model": "mistral-small-latest",
  "choices": [
    {
      "index": 0,
      "finish_reason": "stop",
      "message": {
        "role": "assistant",
        "content": "..." // string OR list of chunks
      }
    }
  ],
  "usage": { ... }
}
```

### SDK Methods

| Method | Description |
|--------|-------------|
| `client.chat.complete(...)` | Synchronous (or async variant) non-streaming completion. |
| `client.chat.stream(...)` | Streaming completion (iterable of SSE events). |
| `client.chat.parse(...)` | Structured-output helper accepting a Pydantic/Zod/JSON-Schema `response_format`; returns parsed object. See §8. |

---

## 3. Text Generation

### Main Concepts

Text generation is the foundational capability — providing a list of messages and receiving a generated assistant message. Use cases span chatbots, classification, data extraction, summarization, code generation, and question answering.

The response content can be:
- A plain **string**: `{'content': '...'}` (standard usage).
- A **list of chunks**: `{'content': [{'type': 'text', 'text': '...'}, {'type': 'thinking', ...}, ...]}` — when reasoning, citations, or other interleaved events are present.

### API Function & Parameters

`client.chat.complete(model, messages, ...)` — see §2 for full parameter table.

### Example (Non-Streaming)

```python
import os
from mistralai.client import Mistral

client = Mistral(api_key=os.environ["MISTRAL_API_KEY"])

chat_response = client.chat.complete(
    model="mistral-large-latest",
    messages=[
        {"role": "user", "content": "How far is the moon from earth?"},
    ],
)
print(chat_response.choices[0].message.content)
```

### Useful Generation Features

- **`prefix` flag** — Set on an assistant message at the end of the list (`"prefix": True`) to force the model to start its response with that exact string. Useful for enforcing format or reducing hallucination. The model continues only after the prefix.
- **`safe_prompt` flag** — Forces the completion to be moderated against sensitive content (legacy mechanism; the newer `guardrails` array is recommended — see §11).
- **`stop` sequences** — An array of strings/tokens that force the model to stop generating. The stop sequence itself is not included in the output.

---

## 4. Message Roles & Instruction Hierarchy

### Main Concepts

Chat `messages` are a collection of prompts/messages, each with a specific role:

| Role | Description |
|------|-------------|
| `system` | **Optional.** Sets behavior, context, personality, and instructions for the assistant. Placed first. |
| `user` | The human's input — a request, question, or comment. |
| `assistant` | The model's reply. Can also appear at the start (e.g. a greeting) or with `prefix: true` to steer the next generation. |
| `tool` | Appears only in function-calling contexts, carrying the tool's output back for final response formulation. |

### system vs user — When to Split

Mistral recommends experimenting with two patterns:
1. Combine `system` + `user` into a single `user` message.
2. Separate them into two distinct messages (`system` then `user`).

There is no strict priority hierarchy documented (unlike OpenAI's developer > user chain of command); the `system` message is treated as optional context/instruction.

### Example

```python
chat_response = client.chat.complete(
    model="mistral-small-latest",
    messages=[
        {"role": "system", "content": "You are a helpful French-speaking assistant."},
        {"role": "user", "content": "What is the capital of France?"},
    ],
)
```

---

## 5. Multi-turn Conversations

### Main Concepts

Chat completions support multi-turn conversations by sending the full message history (user → assistant → user → ...) on each request. The API itself is **stateless** — you manage and replay the conversation history client-side.

Between interactions, different events may be interleaved: tool calls (function calling), citations/references, or reasoning chunks. When using reasoning, the full assistant message **including ThinkChunks** must be replayed (see §6).

For a simplified managed alternative, Mistral offers the **Conversations API** (`client.beta.conversations.start`, `/v1/conversations`) which handles multi-turn state, built-in tools, and connectors — but that is part of the Agents surface (covered in the agents/tools study).

### API Function & Parameters

Same `client.chat.complete(...)` — the multi-turn behavior comes from passing an extended `messages` array.

### Example

```python
chat_response = client.chat.complete(
    model="mistral-medium-latest",
    messages=[
        {"role": "user", "content": "What is the capital of France?"},
        {"role": "assistant", "content": "The Capital of France is **Paris**."},
        {"role": "user", "content": "Translate that to French."},
    ],
)
```

### State Management Summary

| Method | Description |
|--------|-------------|
| **Manual replay** | Append each assistant reply to `messages` and resend on every turn. Default for Chat Completions. |
| **Conversations API (beta)** | `client.beta.conversations.start(agent_id=..., inputs=...)` — server-managed conversation state with built-in tools/connectors. |

---

## 6. Reasoning (Adjustable Thinking)

### Main Concepts

**Reasoning** is Mistral's term for Chain-of-Thought-style logical steps the model generates before the final answer (also called **Test Time Computation**). Reasoning models are trained to freely generate chains of thought before producing the answer, excelling at math and coding but applicable broadly.

The output of a reasoning request includes a **thinking chunk** (`ThinkChunk`) with the model's trace, followed by the final answer (`TextChunk`).

> Note: Native reasoning models `magistral-small-latest` / `magistral-medium-latest` have been **deprecated**. Adjustable reasoning is now accessed via `reasoning_effort` on standard models.

### Supported Models

- `mistral-small-latest` — supports `reasoning_effort`. No extra configuration; just add the parameter.
- `mistral-medium-3-5` — supports `reasoning_effort`. `"high"` recommended for agentic and code use cases.

### API Parameter

| Parameter | Location | Description |
|-----------|----------|-------------|
| `reasoning_effort` | top-level request param | `"high"` → full thinking chunk before the answer (more tokens); `"none"` → minimal thinking, thinking chunk omitted (content is a plain string). |

`reasoning_effort` is also available on the Agents and Conversations endpoints inside the `completion_args` field.

### Output Shape — Handling Thinking Chunks

When `reasoning_effort="high"`, `message.content` is a **list of chunks** instead of a string:

- **`ThinkChunk`** (`type: "thinking"`) — contains the reasoning trace. Its `thinking` field is itself a list of `TextChunk` objects.
- **`TextChunk`** (`type: "text"`) — contains the final answer.

When `reasoning_effort="none"`, `message.content` is a plain `str`.

```python
from mistralai.client.models import TextChunk, ThinkChunk

content = chat_response.choices[0].message.content
if isinstance(content, str):
    print(content)
else:
    for chunk in content:
        if isinstance(chunk, ThinkChunk):
            for inner in chunk.thinking:
                if isinstance(inner, TextChunk):
                    print(inner.text)
        elif isinstance(chunk, TextChunk):
            print(chunk.text)
```

### Streaming Reasoning

When streaming with `reasoning_effort="high"`, `delta.content` changes shape during the response:
1. **Thinking phase** — `delta.content` is a list containing a `ThinkChunk`.
2. **Transition** — a single list with both a closing `ThinkChunk` and the first `TextChunk`.
3. **Answer phase** — `delta.content` is a plain string.

### Multi-turn with Reasoning — Critical Rule

**Always replay the full assistant message (including `ThinkChunk`) back into the message history.** Dropping the reasoning trace across turns degrades model performance. Do not rebuild the message with only the answer text — append the raw `assistant_message` object.

```python
messages.append(user_msg)
response = client.chat.complete(model="mistral-medium-3-5", messages=messages, reasoning_effort="high")
messages.append(response.choices[0].message)  # preserves ThinkChunk
```

> Stripping `ThinkChunk` improves token efficiency but significantly degrades output quality.

---

## 7. Vision (Image Inputs)

### Main Concepts

Vision capabilities enable models to analyze images and provide insights based on visual content alongside text — a multimodal approach. All vision-capable models are accessed via the same Chat Completions API.

> For document parsing, OCR, and structured data extraction, see Document AI (separate study).

### Vision-Capable Models

- Mistral Large 3 (`mistral-large-2512`)
- Mistral Medium 3.1 (`mistral-medium-2508`)
- Mistral Small 3.2 (`mistral-small-2506`)
- Ministral 3: 14B (`ministral-14b-2512`), 8B (`ministral-8b-2512`), 3B (`ministral-3b-2512`)

### Sending an Image — Two Methods

| Method | Description |
|--------|-------------|
| **Image URL** | Provide a publicly accessible URL in an `image_url` content part. No encoding required. |
| **Base64 encoded** | Pass a base64-encoded image (data URI) in the same `image_url` field. |

### Content Part Schema

A user message `content` becomes a list of typed parts:

```json
{
  "role": "user",
  "content": [
    {"type": "text", "text": "What's in this image?"},
    {"type": "image_url", "image_url": "https://.../eiffel-tower.jpg"}
  ]
}
```

| Field | Description |
|-------|-------------|
| `type` | `"text"` or `"image_url"`. |
| `text` | (text part) The textual prompt. |
| `image_url` | (image part) A public URL or base64 data URI of the image. |

### Example (URL)

```python
messages = [
    {
        "role": "user",
        "content": [
            {"type": "text", "text": "What's in this image?"},
            {"type": "image_url", "image_url": "https://docs.mistral.ai/img/eiffel-tower-paris.jpg"},
        ],
    }
]
chat_response = client.chat.complete(model="mistral-small-latest", messages=messages)
```

### Use Cases

- Chart/graph understanding and data extraction
- Image comparison
- Transcription from images
- OCR of old documents
- OCR combined with structured output

---

## 8. Structured Outputs (Custom Schema & JSON Mode)

### Main Concepts

When LLMs are used as steps in a pipeline/chain, outputs often must adhere to a structured format. Mistral offers two mechanisms:

1. **Custom Structured Outputs** (recommended) — supply a JSON Schema (via Pydantic/Zod/JSON Schema); the model is guided to return a JSON object matching it.
2. **JSON Mode** — set `response_format` to `{"type": "json_object"}` to guarantee valid JSON without a strict schema.

> For JSON mode, you must explicitly instruct the model in the prompt to output JSON and specify the format. Custom structured outputs are more reliable; iterate on prompts for best results.

### Custom Structured Outputs

#### API Function & Parameters

`client.chat.parse(...)` — a dedicated helper that accepts a `response_format` as a Pydantic model, Zod schema, or JSON Schema and returns a parsed object.

| Parameter | Description |
|-----------|-------------|
| `response_format` | A Pydantic `BaseModel` class, Zod schema, or JSON Schema dict defining the output structure. |

The SDK prepends the following to the system prompt automatically:
```
Your output should be an instance of a JSON object following this schema: {{ json_schema }}
```

It is recommended to add further explanation and iterate on the system prompt.

#### Example

```python
from pydantic import BaseModel

class Book(BaseModel):
    name: str
    authors: list[str]

chat_response = client.chat.parse(
    model="ministral-8b-latest",
    messages=[
        {"role": "system", "content": "Extract the books information."},
        {"role": "user", "content": "I recently read 'To Kill a Mockingbird' by Harper Lee."},
    ],
    response_format=Book,
    max_tokens=256,
    temperature=0,
)
```

#### Accessing the Output

- **Raw JSON** — `chat_response.choices[0].message.content` (a stringified JSON object).
- **Parsed object** — available via the SDK's parse helper (Pydantic instance).

### JSON Mode

#### API Function & Parameters

Set `response_format` to `{"type": "json_object"}` on a standard `client.chat.complete(...)` call.

#### Example

```python
chat_response = client.chat.complete(
    model="mistral-large-latest",
    messages=[
        {"role": "user", "content": "What is the best French meal? Return the name and the ingredients in short JSON object."},
    ],
    response_format={"type": "json_object"},
)
```

The `content` field is a stringified JSON object. The output is enforced to be valid JSON regardless of the prompt, but explicit instruction is still recommended.

### Comparison

| Feature | Custom Structured Outputs | JSON Mode |
|---------|--------------------------|-----------|
| Guarantees valid JSON | Yes | Yes |
| Adheres to a schema | Yes (schema-enforced) | No (prompt-guided only) |
| SDK helper | `client.chat.parse` | `client.chat.complete` |
| Prompt instruction required | Auto-prepended schema hint | Must explicitly instruct JSON |
| Reliability | Higher (recommended) | Lower; iterate on prompts |

---

## 9. Citations & References (Document Grounding)

### Main Concepts

Citations enable models to ground responses and provide references — a powerful feature for RAG and agentic applications. Models are deeply trained to ground on documents and extract references/citations, providing the source of information extracted from a document or chunk from a tool call.

Citations are surfaced as a **`ReferenceChunk`** in the response content list, pointing back to source documents supplied via a tool-call response.

### How It Works — The Flow

1. **Define a tool** (e.g. `get_information`) that returns references as a JSON string.
2. **Make an initial chat request** with the tool and the user query. The model emits a `tool_call`.
3. **Execute the tool** locally, returning a JSON map of references.
4. **Append a `tool`-role message** (`ToolMessage`) with the tool result to the chat history.
5. **Make a final chat request** with the updated history. The model produces an answer with `TextChunk`(s) and `ReferenceChunk`(s) interleaved.
6. **Extract references** — `ReferenceChunk.reference_ids` indexes into the references map.

### References Data Structure

References are passed as a JSON object keyed by string IDs:

```json
{
  "0": {
    "url": "https://en.wikipedia.org/wiki/...",
    "title": "2024 Nobel Peace Prize",
    "snippets": [["snippet text...", "another snippet..."]],
    "description": null,
    "date": "2024-11-26T17:39:55.057454",
    "source": "wikipedia"
  },
  "1": { ... }
}
```

| Field | Description |
|-------|-------------|
| `url` | Source URL. |
| `title` | Source title. |
| `snippets` | Array of arrays of text snippets from the source. |
| `description` | Optional description. |
| `date` | ISO timestamp. |
| `source` | Source label (e.g. `wikipedia`). |

### Tool Definition (externally tagged, OpenAI-style)

```python
get_information_tool = {
    "type": "function",
    "function": {
        "name": "get_information",
        "description": "Get information from external source.",
        "parameters": {"type": "object", "properties": {}, "additionalProperties": False},
        "strict": True,
    },
}
```

### Response Chunk Types

| Chunk Type | SDK Class | Description |
|-----------|-----------|-------------|
| `text` | `TextChunk` | The answer text. Has `.text`. |
| `reference` | `ReferenceChunk` | A citation marker. Has `.reference_ids` (list of reference keys). |

### Parsing the Response

```python
from mistralai.client.models import TextChunk, ReferenceChunk

refs_used = []
for chunk in chat_response.choices[0].message.content:
    if isinstance(chunk, TextChunk):
        print(chunk.text, end="")
    elif isinstance(chunk, ReferenceChunk):
        refs_used += chunk.reference_ids
```

### Tool Result Message

```python
from mistralai.client import ToolMessage

tool_call_result = ToolMessage(
    content=result,              # JSON string of references
    tool_call_id=tool_call.id,
    name=tool_call.function.name,
)
chat_history.append(tool_call_result)
```

> Citations also appear in the Conversations API and the web search tool via `tool_reference` chunks pointing back to source documents.

---

## 10. Streaming

### Main Concepts

Streaming surfaces generation progress incrementally. Use `client.chat.stream(...)` which returns an iterable of SSE events. Each event's `data.choices[0].delta` contains incremental content.

### Streaming with Reasoning

When `reasoning_effort="high"`, `delta.content` changes shape over the response (see §6):
1. **Thinking phase** — `delta.content` is a list containing a `ThinkChunk`.
2. **Transition** — a list with both a closing `ThinkChunk` and the first `TextChunk`.
3. **Answer phase** — `delta.content` is a plain string.

### Example (Streaming with Reasoning)

```python
for event in client.chat.stream(
    model="mistral-medium-3-5",
    messages=[{"role": "user", "content": "What is 17 * 23?"}],
    reasoning_effort="high",
):
    delta = event.data.choices[0].delta.content
    if not delta:
        continue
    if isinstance(delta, str):
        print(delta, end="", flush=True)
    else:
        for chunk in delta:
            if isinstance(chunk, ThinkChunk):
                for inner in chunk.thinking:
                    if isinstance(inner, TextChunk):
                        print(inner.text, end="", flush=True)
            elif isinstance(chunk, TextChunk):
                print(chunk.text, end="", flush=True)
```

---

## 11. Moderation & Guardrailing

### Main Concepts

When deploying LLMs in production, different verticals require different levels of guardrailing (safe content, PII filtering, jailbreak detection). Mistral provides two mechanisms:

1. **Custom Guardrails** (recommended) — declare moderation rules directly in API requests via a `guardrails` array. No separate calls, no threshold logic in your code. Applies **input moderation only** (runs before the model); blocks with HTTP 403 on violation.
2. **Moderation API** — a dedicated API to classify text across policy categories, for custom pipelines needing raw scores and full control.

Both are backed by the `mistral-moderation-2603` model (`mistral-moderation-2411` was deprecated March 31, 2026).

### Moderation API

#### Endpoints

| Endpoint | Use |
|----------|-----|
| Raw-text (`/v1/moderations`) | Classify raw text chunks directly. Input: a single string or list of strings (small batches). |
| Conversational (`/v1/chat/moderations`) | Classify conversational content. |

#### API Function & Parameters

`client.classifiers.moderate(model, inputs)`

| Parameter | Description |
|-----------|-------------|
| `model` | `"mistral-moderation-2603"`. |
| `inputs` | A single string or a list of strings to classify. |

Returns scores per category. The policy threshold is based on internal test-set optimal performance; you can use the raw score or adjust thresholds per use case. Custom policies depending on `category_scores` may require recalibration as the model improves.

#### Example

```python
response = client.classifiers.moderate(
    model="mistral-moderation-2603",
    inputs=[
        "Such a lovely day today, isn't it?",
        "Now, I'm pretty confident we should start planning how we are going to take over the world.",
    ],
)
```

#### Policy Categories

| Category | Description |
|----------|-------------|
| **Sexual** | Explicit sexual content, nudity, solicitation. Educational/medical exempted. |
| **Hate and Discrimination** | Prejudice/hostility/discrimination against protected groups (race, religion, gender, sexual orientation, disability). Slurs, dehumanizing language, exclusion/harm calls. |
| **Violence and Threats** | Describes/glorifies/incites/threatens physical violence. Graphic injury/death, threats, instructions for violent acts. |
| **Dangerous** | Promotes extremely hazardous behaviors with significant physical-harm risk. |
| **Criminal** | Describes/promotes illegal activities. |
| **Self-Harm** | Promotes/instructs/plans/encourages self-injury, suicide, eating disorders. Methods, glorification, intent, dangerous challenges. |
| **Health** | Contains or elicits detailed/tailored medical advice. |
| **Financial** | Contains or elicits detailed/tailored financial advice. |
| **Law** | Contains or elicits detailed/tailored legal advice. |
| **PII** | Requests/shares/elicits personal identifying information (names, addresses, phone, SSN, financial accounts). |
| **Jailbreaking** | Attempts to bypass safety guidelines via prompt manipulation, role-playing, or other techniques. |

### Custom Guardrails (Inline)

Guardrails are declared directly in `POST /v1/chat/completions` or `POST /v1/conversations` requests via a top-level `guardrails` array. They run on **input only**, before the model. On violation, a `403` is returned.

#### Guardrail Config (`moderation_llm_v2`)

| Field | Description |
|-------|-------------|
| `custom_category_thresholds` | Object mapping category names → threshold values (0–1). Set a category to `1` to explicitly disable it. |
| `ignore_other_categories` | If `true`, only categories in `custom_category_thresholds` are evaluated; others ignored. |
| `action` | `"block"` to block the request on violation. |
| `block_on_error` | If `true`, block the request when the moderation API itself fails (per guardrail). |
| `model_name` | *(Optional)* Override the default moderation model for that config. |

Multiple guardrails can be specified per request — the request is blocked if **any** is triggered. Only one `moderation_llm_v2` config per guardrail object, but multiple guardrail objects allowed.

#### Example (Inline Guardrail on Chat Completion)

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

#### Guardrail Attachment Points

| Surface | How |
|---------|-----|
| Chat Completions | `guardrails` field on `POST /v1/chat/completions`. |
| Conversations API | `guardrails` field on `POST /v1/conversations` (with `model` or overriding an agent's guardrails). |
| Agent-level | `guardrails` set at agent creation; inherited by all conversations using that agent; overridable per-request. |

#### Response — Successful (Non-Blocked)

A `guardrails` field is included in the response with evaluation results per guardrail. Only categories in `custom_category_thresholds` are returned (or all evaluated categories when `ignore_other_categories` is `false`):

```json
{
  "guardrails": [
    {
      "moderation_llm_v2": {
        "action": "pass",
        "categories": {
          "sexual": {"score": 0.03, "violated": false},
          "selfharm": {"score": 0.05, "violated": false},
          "violence_and_threats": {"score": 0.0, "violated": false},
          "hate_and_discrimination": {"score": 0.0, "violated": false}
        }
      }
    }
  ]
}
```

#### Response — Blocked (403)

```json
{
  "error": {"message": "Content blocked by guardrail", "status": 403},
  "guardrails": {
    "results": {
      "moderation_llm_v2": {
        "model_name": "mistral-moderation-2603",
        "decisions": {
          "sexual": {"threshold": 0.1, "score": 0.3, "violated": true},
          "selfharm": {"threshold": 0.1, "score": 0.05, "violated": false}
        },
        "violated": true,
        "action": "block"
      }
    }
  }
}
```

#### Response — Block on Error

If `block_on_error` is `true` and the moderation API fails:

```json
{
  "object": "Error",
  "message": "Request blocked due to error in guardrail evaluation and block_on_error is set to True.",
  "type": "invalid_request_error",
  "code": 3201,
  "guardrails": [
    {"moderation_llm_v2": {"action": "block", "error": {"message": "Moderation API request failed."}}}
  ]
}
```

### Legacy `safe_prompt`

The older `safe_prompt: true` flag on chat completions forces moderation against sensitive content. The `guardrails` array is the recommended successor.

---

## 12. Batch Processing

### Main Concepts

Batching allows asynchronous inference on large inputs in parallel, reducing compute costs — batch jobs run at a **50% discount**. A batch is a list of API requests, each with a unique `custom_id` and a `body` matching the target endpoint's request body. Supports up to **1 million requests** per batch (file batching) or **10,000** requests (inline batching).

### Batch Request Structure

Each line in a batch (JSONL) is:

```json
{"custom_id": "0", "body": {"max_tokens": 128, "messages": [{"role": "user", "content": "..."}]}}
```

| Field | Description |
|-------|-------------|
| `custom_id` | A unique ID for identifying each request and referencing results. |
| `body` | The raw request body for the target endpoint (same shape as the endpoint's normal request), **excluding** `model` (provided at job creation). |

The `body` can be any valid request body for the chosen endpoint. One JSON object per line — no line breaks within an object.

### Supported Endpoints

`/v1/embeddings`, `/v1/chat/completions`, `/v1/fim/completions`, `/v1/moderations`, `/v1/chat/moderations`, `/v1/ocr`, `/v1/classifications`, `/v1/conversations`, `/v1/audio/transcriptions`.

### Two Batching Methods

#### A. File Batching (up to 1M requests)

1. **Upload** a `.jsonl` file via `client.files.upload(file=..., purpose="batch")` (or via Studio › Files with `purpose` = Batch Processing).
2. **Create a job** referencing the file ID.

#### B. Inline Batching (up to 10k requests)

Pass `requests` (a list of `{custom_id, body}` objects) directly at job creation — no file upload needed.

### API Functions & Parameters

#### Upload a Batch File

`client.files.upload(file, purpose)`

| Parameter | Description |
|-----------|-------------|
| `file` | `{"file_name": ..., "content": <file obj>}`. |
| `purpose` | `"batch"`. |

#### Create a Batch Job

`client.batch.jobs.create(input_files / requests, model, endpoint, metadata)`

| Parameter | Description |
|-----------|-------------|
| `input_files` | *(File batching)* List of uploaded file IDs. |
| `requests` | *(Inline batching)* List of `{custom_id, body}` objects. |
| `model` | One model per batch (e.g. `mistral-small-latest`, `codestral-latest`). Run multiple batches on the same files to compare models. |
| `endpoint` | One of the supported endpoints above. |
| `metadata` | Optional custom metadata (e.g. `{"job_type": "testing"}`). |

#### Example (File Batching)

```python
batch_data = client.files.upload(
    file={"file_name": "test.jsonl", "content": open("test.jsonl", "rb")},
    purpose="batch",
)
created_job = client.batch.jobs.create(
    input_files=[batch_data.id],
    model="mistral-small-latest",
    endpoint="/v1/chat/completions",
    metadata={"job_type": "testing"},
)
```

#### Example (Inline Batching)

```python
inline_batch_data = [
    {"custom_id": "0", "body": {"max_tokens": 128, "messages": [{"role": "user", "content": "..."}]}},
    {"custom_id": "1", "body": {"max_tokens": 512, "temperature": 0.2, "messages": [{"role": "user", "content": "..."}]}},
]
created_job = client.batch.jobs.create(
    requests=inline_batch_data,
    model="mistral-small-latest",
    endpoint="/v1/chat/completions",
)
```

#### Retrieve Job Details

`client.batch.jobs.get(job_id=created_job.id)`

#### Get Job Results

```python
output_file_stream = client.files.download(file_id=retrieved_job.output_file)
with open("batch_results.jsonl", "wb") as f:
    f.write(output_file_stream.read())
```

#### List Jobs

`client.batch.jobs.list(status, metadata)` — filter by status and/or metadata.

| Status values | |
|---------------|-|
| `QUEUED`, `RUNNING`, `SUCCESS`, `FAILED`, `TIMEOUT_EXCEEDED`, `CANCELLATION_REQUESTED`, `CANCELLED` | |

#### Cancel a Job

`client.batch.jobs.cancel(job_id=created_job.id)`

### Job Statistics

Each job object includes: `total_requests`, `failed_requests`, `succeeded_requests`, `created_at`, `completed_at`, `output_file`, `error_file`.

### End-to-End Flow

```
1. Create client (Mistral(api_key=...))
2. Generate input data → build JSONL in-memory (BytesIO)
3. Upload file (client.files.upload, purpose="batch")
4. Create job (client.batch.jobs.create, input_files=[id], model, endpoint)
5. Poll status (client.batch.jobs.get) until SUCCESS/FAILED
6. Download output_file + error_file (client.files.download)
```

---

## 13. Embeddings API

### Main Concepts

**Embeddings** are vector representations of text capturing semantic meaning through position in a high-dimensional vector space. Mistral's Embeddings API provides embeddings for **text** and **code**, usable for retrieval (RAG), clustering, classification, semantic search, duplicate detection.

> For a managed feature that ingests/vectorizes/searches documents for you, use **Libraries** (§15). For connected sources (Google Drive, SharePoint), use Connectors.

### Services

- **Text embeddings** — general-purpose text embedding model (`mistral-embed`).
- **Code embeddings** — embed code databases/repositories for code retrieval (separate page).

### API Function & Parameters

`client.embeddings.create(model, inputs)`

| Parameter | Description |
|-----------|-------------|
| `model` | `"mistral-embed"` (text). |
| `inputs` | A single string or a list of strings to embed. |

Returns `data[].embedding` (a vector). For batched requests, multiple inputs can be passed at once.

### Example

```python
def get_text_embedding(input):
    response = client.embeddings.create(model="mistral-embed", inputs=input)
    return response.data[0].embedding

embedding = get_text_embedding("What were the two main things the author worked on?")
```

### Use Cases

- RAG retrieval (combine with a vector DB — see §14)
- Clustering of unorganized data
- Document classification
- Semantic code search / code analytics
- Duplicate detection

---

## 14. RAG — From Scratch

### Main Concepts

Retrieval-Augmented Generation (RAG) combines retrieval with model generation to answer questions or generate content from external knowledge. Two steps:

1. **Retrieval** — retrieve relevant information from a knowledge base using text embeddings stored in a vector store.
2. **Generation** — insert the relevant information into the prompt so the model generates an answer grounded in that context.

This guide builds a RAG pipeline from scratch using the Embeddings API + a vector DB (e.g. FAISS) + Chat Completions. For managed RAG, use **Libraries** (§15).

### Pipeline Steps

1. **Get data** — load a text document (e.g. an essay).
2. **Split into chunks** — e.g. 2048 characters per chunk. Tunable: smaller chunks improve retrieval precision but increase processing; can split by tokens, sentences, paragraphs, HTML headers, or AST (for code).
3. **Create embeddings** — call `client.embeddings.create(model="mistral-embed", inputs=chunk)` for each chunk.
4. **Load into a vector DB** — e.g. FAISS `IndexFlatL2(d)`; `index.add(text_embeddings)`.
5. **Embed the question** — same embedding model as the chunks.
6. **Retrieve similar chunks** — `index.search(question_embeddings, k=2)` returns distances `D` and indices `I`; map indices back to chunk text.
7. **Combine context + question in a prompt** and call `client.chat.complete(...)` to generate the answer.

### Prompt Template

```python
prompt = f"""
Context information is below.
---------------------
{retrieved_chunk}
---------------------
Given the context information and not prior knowledge, answer the query.
Query: {question}
Answer:
"""
```

### Generation Call

```python
def run_mistral(user_message, model="mistral-large-latest"):
    chat_response = client.chat.complete(model=model, messages=[{"role": "user", "content": user_message}])
    return chat_response.choices[0].message.content

run_mistral(prompt)
```

### RAG Technique Notes

| Topic | Guidance |
|-------|----------|
| **Chunk size** | Smaller chunks aid retrieval precision; trade-off is processing time/resources. |
| **Splitting** | By character, tokens, sentences, paragraphs, HTML headers, or AST for code. |
| **Vector DB choice** | Consider speed, scalability, cloud management, advanced filtering, open vs closed source. |
| **HyDE** | Generate a hypothetical answer/document from the query and embed that for better retrieval. |
| **Retrieval methods** | Similarity search (embeddings), metadata filtering, TF-IDF, BM25. |
| **Child/parent chunks** | Retrieve a small "child chunk" but return a larger "parent chunk" for context. |
| **Time-weighted retrieval** | Weight documents by recency. |
| **Lost in the middle** | Reorder docs to place most relevant at beginning/end. Mistral models handle up to 32k context well (passkey/needle-in-haystack). |
| **Prompting** | Few-shot learning, explicit formatting instructions can improve RAG answers. |

### Integrations

- LangChain (Adaptive RAG, Corrective RAG, Self-RAG)
- LlamaIndex (agentic RAG)
- Haystack (chat with documents)

---

## 15. Libraries (Managed Knowledge Base / RAG)

### Main Concepts

**Libraries** are persistent knowledge bases you fill with documents and connect to agents for built-in RAG. Upload PDFs, papers, or any document, and agents search through them on demand — Mistral handles ingestion, vectorization, and search internally.

Libraries are interoperable with Vibe: a Library created in Vibe (`https://chat.mistral.ai/libraries/<library_id>`) is usable via the API (must be shared with the Org by an admin), and vice versa.

### API Functions & Parameters (all under `client.beta.libraries.*`)

#### Create a Library

`client.beta.libraries.create(name, description)`

| Parameter | Description |
|-----------|-------------|
| `name` | Library name. |
| `description` | Optional description. |

Response includes `generated_name` and `generated_description` (auto-updated as files are added).

#### List Libraries

`client.beta.libraries.list()` → `.data` array, each with `name`, `nb_documents`.

#### Upload a Document

`client.beta.libraries.documents.upload(library_id, file)`

| Parameter | Description |
|-----------|-------------|
| `library_id` | Target library ID. |
| `file` | `File(fileName=..., content=<file obj>)`. |

#### List Documents

`client.beta.libraries.documents.list(library_id)` → `.data` array, each with `name`, `extension`, `number_of_pages`, `summary`.

#### Document Status

`client.beta.libraries.documents.status(library_id, document_id)`

Returns `processing_status`: `"Running"` (being processed) or `"Completed"` (ready).

#### Get Document Info

`client.beta.libraries.documents.get(library_id, document_id)` — full metadata once processed.

#### Get Document Content

`client.beta.libraries.documents.text_content(library_id, document_id)` — extracted text. Also provides `signed_url` for extracted text and raw file.

#### Delete

- `client.beta.libraries.delete(library_id)` — delete a library.
- Document deletion — delete individual documents.

### Access Control

Manage who has access to each Library:

| Parameter | Description |
|-----------|-------------|
| `org_id` | Organization ID. |
| `level` | `"Viewer"` or `"Editor"`. |
| `share_with_uuid` | ID of the entity to share with. |
| `share_with_type` | `"User"`, `"Workspace"`, or `"Org"`. |

Rules: must be owner to share; owner can't delete their own access; owner can delete others' access; Viewers can't edit, Editors can.

Functions: `client.beta.libraries.accesses.list(library_id)`, create/update access, delete access.

### Connecting to Agents (Built-in `document_library` Tool)

Document Library is a built-in agent tool. Create an agent with `tools=[{"type": "document_library", "library_ids": [...]}]`:

```python
library_agent = client.beta.agents.create(
    model="mistral-medium-latest",
    name="Document Library Agent",
    description="Agent used to access documents from the document library.",
    instructions="Use the library tool to access external documents.",
    tools=[{"type": "document_library", "library_ids": [new_library.id]}],
    completion_args={"temperature": 0.3, "top_p": 0.95},
)
```

### Querying via Conversations

```python
response = client.beta.conversations.start(
    agent_id=library_agent.id,
    inputs="How does the vision encoder for pixtral 12b work?",
)
```

### Response Entries

- **`tool.execution`** — the Document Library tool ran a search (includes `name`, `created_at`, `completed_at`, `id`).
- **`message.output`** — the agent's answer, grounded in found documents. `content` is a list of chunks: `text` (the response) or `tool_reference` (citations pointing to source documents).
- **`usage`** — token counts including `connector_tokens` consumed by the Library search.

> The agent/tools connection is detailed in the agents/tools study; the text-relevant aspect here is the managed document ingestion + search + grounded answer with citations.

---

## 16. Capability Summary & Cross-Reference

### Complete Capability Matrix

| Capability | Chat Completions | Conversations API (beta) | Section |
|-----------|------------------|--------------------------|---------|
| Text generation | Yes (`client.chat.complete`) | Yes (`client.beta.conversations.start`) | §3 |
| Message roles (system/user/assistant/tool) | Yes | Yes | §4 |
| Multi-turn conversation | Manual replay | Server-managed | §5 |
| Adjustable reasoning (`reasoning_effort`) | Yes (`high`/`none`) | Yes (via `completion_args`) | §6 |
| Vision (image URL / base64) | Yes (`image_url` content part) | Yes | §7 |
| Custom Structured Outputs (schema) | Yes (`client.chat.parse`) | — | §8 |
| JSON Mode | Yes (`response_format: json_object`) | — | §8 |
| Citations & references | Yes (tool-call flow + `ReferenceChunk`) | Yes (`tool_reference` chunks) | §9 |
| Streaming | Yes (`client.chat.stream`) | Yes | §10 |
| Inline guardrails (`guardrails`) | Yes | Yes (override agent) | §11 |
| Agent-level guardrails | — | Yes (at creation) | §11 |
| `safe_prompt` (legacy) | Yes | — | §11 |
| `prefix` flag | Yes (assistant message) | — | §3 |
| `stop` sequences | Yes | — | §3 |
| Batch processing | Via `/v1/batch/jobs` (endpoint `/v1/chat/completions`) | Via `/v1/batch/jobs` (endpoint `/v1/conversations`) | §12 |
| Embeddings | `/v1/embeddings` (`mistral-embed`) | — | §13 |
| RAG from scratch | Embeddings + vector DB + chat | — | §14 |
| Managed RAG (Libraries) | — | Via `document_library` tool on agents | §15 |

### Moderation Category Quick Reference

| Category | Key (guardrail) |
|----------|-----------------|
| Sexual | `sexual` |
| Hate and Discrimination | `hate_and_discrimination` |
| Violence and Threats | `violence_and_threats` |
| Dangerous | `dangerous` |
| Criminal | `criminal` |
| Self-Harm | `selfharm` |
| Health | `health` |
| Financial | `financial` |
| Law | `law` |
| PII | `pii` |
| Jailbreaking | `jailbreaking` |

### Structured Output Decision Matrix

| Need | Recommended Approach |
|------|----------------------|
| Enforce a specific JSON schema | Custom Structured Outputs (`client.chat.parse` + Pydantic/Zod/JSON Schema) |
| Just valid JSON, flexible shape | JSON Mode (`response_format: {"type": "json_object"}`) + explicit JSON instruction in prompt |
| Structured data extraction | Custom Structured Outputs with a typed model |

### RAG Decision Matrix

| Need | Recommended Approach |
|------|----------------------|
| Managed, no infrastructure | Libraries (§15) — upload docs, connect to agent, get grounded answers with citations |
| Connected external sources (Drive, SharePoint) | Connectors |
| Full control over chunking/retrieval | RAG from scratch (§14) — Embeddings API + vector DB + Chat Completions |
| Ground answers in provided documents (no persistence) | Citations & References (§9) — pass references via tool call |

### Reasoning Decision Matrix

| Need | Setting |
|------|---------|
| Math/coding/complex reasoning | `reasoning_effort="high"` (full thinking trace) |
| Fast, simple answers | `reasoning_effort="none"` (plain string output) |
| Multi-turn with reasoning | Replay full assistant message incl. `ThinkChunk` each turn |