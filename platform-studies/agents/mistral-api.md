# Mistral API Analysis — Agent Capabilities (Agents & Conversations API)

> **Docs:** `https://docs.mistral.ai/studio-api/agents/introduction` | **API spec:** `https://docs.mistral.ai/api` (tags `beta.agents`, `beta.conversations`, `beta.connectors`, `beta.libraries`)
> **Base API:** `https://api.mistral.ai/v1` | **Auth:** `Authorization: Bearer $MISTRAL_API_KEY`
> **SDKs:** Python (`mistralai`), TypeScript; beta namespace (`client.beta.agents`, `client.beta.conversations`, `client.beta.connectors`, `client.beta.libraries`) | async variants: `*_async`
> **Description:** Mistral exposes agent capabilities through the **Agents & Conversations API** — a hosted, resource-oriented platform where **Agents** are reusable pre-configured model bundles (model + instructions + tools + completion args) and **Conversations** are persistent, entry-based interaction histories. Unlike a raw Chat Completions loop, the platform runs a server-side agent loop: the model autonomously discovers and calls built-in tools (web search, code interpreter, image generation, document library) and custom functions, and can hand off a conversation to other agents. Communication is organized as **Entries** (typed interaction events) rather than plain messages. External capabilities are extended through **Connectors** (managed MCP servers) and **Libraries** (RAG knowledge bases). The platform is organized around six capability areas: agent management, conversations, built-in tools, handoffs, connectors, and libraries.

---

## Table of Contents

1. [Platform Overview & Core Concepts](#1-platform-overview--core-concepts)
2. [Agent Management](#2-agent-management)
3. [Conversations](#3-conversations)
4. [Built-in Tools](#4-built-in-tools)
5. [Function Calling (Custom Tools)](#5-function-calling-custom-tools)
6. [Connectors (Managed MCP Servers)](#6-connectors-managed-mcp-servers)
7. [Human-in-the-Loop (Tool Confirmation)](#7-human-in-the-loop-tool-confirmation)
8. [Handoffs (Multi-Agent Orchestration)](#8-handoffs-multi-agent-orchestration)
9. [Libraries (RAG Knowledge Bases)](#9-libraries-rag-knowledge-bases)
10. [Citations & References](#10-citations--references)
11. [Capability Summary & Cross-Reference](#11-capability-summary--cross-reference)

---

## 1. Platform Overview & Core Concepts

### Main Concepts

Mistral's agent platform is built around these core abstractions:

- **Agent** — A reusable set of pre-selected values that augment model abilities: `model`, `name`, `description`, `instructions` (system prompt), `tools`, `completion_args`, and `handoffs`. Created once, referenced by ID from every conversation. Not versioned (updates mutate the agent in place).
- **Conversation** — A persistent history of interactions with an assistant, started by an Agent (`agent_id`) or a raw model (`model`). Conversations accumulate **Entries** and can be continued across multiple requests. Optionally stored on Mistral's cloud (`store=False` opts out).
- **Entry** — The atomic unit of a conversation. A typed action created by the user or assistant (messages, tool executions, function calls, handoffs). More flexible and expressive than plain chat messages; allows fine-grained control over describing events.
- **Tool** — A capability the agent can call. Two categories: (1) **built-in tools** executed server-side in Mistral's environment (`web_search`, `web_search_premium`, `code_interpreter`, `image_generation`, `document_library`), and (2) **custom functions** executed locally by your application via JSON-schema function calling. Connectors (MCP) expose tools discovered automatically and executed server-side.
- **Connector** — A registered [Model Context Protocol](https://modelcontextprotocol.io/) (MCP) server used as a tool source. Tools are discovered automatically by the model and executed server-side — no local MCP transport management. Public Preview feature.
- **Library** — A persistent knowledge base of uploaded documents connected to agents for built-in retrieval-augmented generation (RAG). Agents search through Libraries on demand via the `document_library` tool.
- **Handoff** — A delegation mechanism where one agent hands over a conversation to another agent mid-action, enabling multi-agent agentic workflows with chained responsibilities.

### Agent Capabilities Map

| Capability | Description | Docs |
|------------|-------------|------|
| **Agent management** | Create, update, list, and delete reusable agent configurations | [agents-api](https://docs.mistral.ai/studio-api/agents/agents-api) |
| **Conversations** | Start, continue (append), manage persistent interaction histories | [agents-api](https://docs.mistral.ai/studio-api/agents/agents-api) |
| **Built-in tools** | Server-side web search, code interpreter, image generation, document library | [agent-tools](https://docs.mistral.ai/studio-api/agents/agent-tools) |
| **Function calling** | Custom local tools defined by JSON schema, executed in your environment | [function-calling](https://docs.mistral.ai/studio-api/agents/agent-tools/function-calling) |
| **Connectors** | Register MCP servers; model auto-discovers and calls their tools | [connectors](https://docs.mistral.ai/studio-api/connectors) |
| **Human-in-the-loop** | Intercept tool calls for user approval before execution | [confirmation](https://docs.mistral.ai/studio-api/connectors/confirmation) |
| **Handoffs** | Multi-agent orchestration: delegate conversation ownership between agents | [handoffs](https://docs.mistral.ai/studio-api/agents/handoffs) |
| **Libraries** | Document knowledge bases for RAG; connect to agents via `document_library` tool | [libraries](https://docs.mistral.ai/studio-api/libraries) |
| **Citations & references** | Grounded responses with source references for RAG and tool outputs | [citations](https://docs.mistral.ai/studio-api/conversations/citations) |

### Platform Architecture

```
Your application (Python mistralai SDK / TypeScript SDK / curl / REST)
        │
        ▼
   Create Agent  ── model, name, description, instructions, tools[], completion_args, handoffs[]
        │          (update mutates in place; reference by ID)
        ▼
   Start Conversation ── agent_id OR model (mutually exclusive)
        │                 inputs (string | message[]), store, handoff_execution, guardrails
        ▼
   ┌──────────────── Agent Loop (server-side) ─────────────────┐
   │  1. Model receives conversation + available tools          │
   │  2. Model decides to respond and/or call a tool           │
   │  3. Built-in tools → executed in Mistral's environment   │
   │  4. Custom functions → returns function.call entry        │
   │      → your app runs it → append FunctionResultEntry       │
   │  5. Connectors → MCP tools auto-discovered & executed     │
   │  6. Handoff triggered → conversation delegated to another │
   │  7. Returns typed Entries (message.output, tool.execution,│
   │     function.call, agent.handoff, tool.file, etc.)        │
   └────────────────────────────────────────────────────────────┘
        │
        ▼
   Conversation (persistent, appendable by conversation_id)
     ├── message.output entries (text + reference chunks)
     ├── tool.execution entries (built-in tool runs)
     ├── function.call entries (custom tool requests)
     ├── agent.handoff entries (delegations)
     └── usage (tokens, connector_tokens)
```

### Quickstart flow

The minimal end-to-end flow is three steps: (1) `POST /v1/agents` → `agent.id`; (2) `POST /v1/conversations` with `agent_id` + `inputs` → `conversation_id` + `outputs[]`; (3) optionally `POST /v1/conversations/{conversation_id}` to continue with new `inputs`. Conversations can also be started without an Agent by passing `model` instead of `agent_id`.

---

## 2. Agent Management

An **agent** is a reusable configuration that bundles a model with instructions, tools, completion parameters, and handoff targets. The Agents API and Conversations API are **independent** — you can start conversations without creating an agent (using `model` directly).

### Agent configuration fields

| Field | Required | Description |
|-------|----------|-------------|
| `model` | Yes | The chat completion model the agent uses (e.g. `"mistral-medium-latest"`, `"mistral-large-latest"`). |
| `name` | Yes | Human-readable name of the agent. |
| `description` | Yes | Description related to the task/use case the agent addresses. |
| `instructions` | No | Main instructions / system prompt. Must accurately describe the agent's task. |
| `tools` | No | List of tools the model can use. Tool `type` values: `function`, `web_search`, `web_search_premium`, `code_interpreter`, `image_generation`, `document_library`, `connector`. |
| `completion_args` | No | Standard chat completion sampler arguments (e.g. `temperature`, `top_p`). All chat completion arguments accepted. |
| `handoffs` | No | List of agent IDs this agent can hand off the conversation to. |

### Create an agent

`POST /v1/agents` — SDK: `client.beta.agents.create(...)` (sync) / `client.beta.agents.create_async(...)`.

```python
agent = client.beta.agents.create(
    model="mistral-medium-latest",
    name="Simple Agent",
    description="A simple Agent with persistent state.",
)
# Returns an agent object with an agent ID used to start conversations.
```

A web-search agent with tools and completion args:

```python
websearch_agent = client.beta.agents.create(
    model="mistral-medium-latest",
    description="Agent able to search information over the web.",
    name="Websearch Agent",
    instructions="You have the ability to perform web searches with `web_search`.",
    tools=[{"type": "web_search"}],
    completion_args={"temperature": 0.3, "top_p": 0.95},
)
```

### Update an agent

`client.beta.agents.update(agent_id=..., ...)` — mutate the agent in place (e.g. add `handoffs`, change tools). Updates are not versioned; existing references pick up the current state.

### Tool types summary

| `type` | Category | Execution |
|--------|----------|-----------|
| `function` | Custom | Local (your application) |
| `web_search` | Built-in | Server-side |
| `web_search_premium` | Built-in | Server-side (+ news provider) |
| `code_interpreter` | Built-in | Server-side (isolated container) |
| `image_generation` | Built-in | Server-side (also works in Chat Completions) |
| `document_library` | Built-in | Server-side RAG over a Library |
| `connector` | Managed MCP | Server-side (MCP transport handled by Mistral) |

> **Note:** `web_search`, `web_search_premium`, and `code_interpreter` work only with the Conversations API (`/v1/conversations`) and the Agents API — **not** the Chat Completions API (`/v1/chat/completions`) because Chat Completions responses don't include search result references. `image_generation` is also supported in Chat Completions.

---

## 3. Conversations

A **conversation** is a persistent history of interactions. More flexible and expressive than the Chat Completions API, it allows more control over describing events through typed **Entries**.

> **Access scope:** The public Conversations API can read and modify only the conversations owned by the creator of the API key used in the request.

### Starting a conversation

`POST /v1/conversations` — SDK: `client.beta.conversations.start(...)` / `.start_async(...)`.

| Parameter | Description |
|-----------|-------------|
| `agent_id` | ID of a created agent. **Mutually exclusive** with `model`. |
| `model` | Use a model directly without an agent. Mutually exclusive with `agent_id`. |
| `inputs` | First user message — either a string or a list of message objects (`{"role": "user", "content": "..."}`). |
| `store` | `False` opts out of cloud storage (the new history is not stored). |
| `handoff_execution` | `server` (default — runs handoffs internally) or `client` (returns the handoff to the user to handle). |
| `guardrails` | Override agent-configured guardrails (with a model, not an agent). |
| `tools` | Per-conversation tools (when using `model`). Can include built-in tools and connectors. |
| `tool_confirmations` | Approve/deny pending tool calls (human-in-the-loop). |

```python
# With an agent
response = client.beta.conversations.start(
    agent_id=simple_agent.id,
    inputs="Who is Albert Einstein?",
    # inputs=[{"role": "user", "content": "Who is Albert Einstein?"}]  # also valid
    # store=False  # opt out of cloud storage
)

# Without an agent, using a model directly
response = client.beta.conversations.start(
    model="mistral-medium-latest",
    inputs=[{"role": "user", "content": "Who is Albert Einstein?"}],
)
```

Returns a `conversation_id` and an `outputs[]` array of typed entries.

### Continuing a conversation

`POST /v1/conversations/{conversation_id}` — SDK: `client.beta.conversations.append(...)`.

| Parameter | Description |
|-----------|-------------|
| `conversation_id` | The ID from a previous start/append, mapping to stored history. |
| `inputs` | The next message or reply — string or list of messages. Can also include `FunctionResultEntry` objects to return custom tool results. |
| `tool_confirmations` | Approve/deny pending tool calls. |

A **new conversation ID** is provided at each append.

### Response structure — Entries

The `outputs[]` array contains typed **entries**. Common entry types:

| Entry type | Description | Key fields |
|------------|-------------|------------|
| `message.output` | Generated assistant answer | `content` (list of chunks: `TextChunk`, `ToolReferenceChunk`, `ToolFileChunk`), `agent_id`, `model`, `role`, `id`, `created_at`, `completed_at` |
| `tool.execution` | Built-in tool execution | `name`, `id`, `created_at`, `completed_at`, `info` (tool-specific: `code`, `code_output`, references…) |
| `function.call` | Request to call a custom function | `name`, `arguments`, `tool_call_id`, `confirmation_status` |
| `agent.handoff` | Conversation delegated to another agent | `agent_id`, `agent_name`, `id`, `created_at` |

### Content chunk types (inside `message.output.content`)

| Chunk type | Description |
|-----------|-------------|
| `TextChunk` | Plain model response text (`text`, `type: "text"`). |
| `ToolReferenceChunk` | Citation pointing to a tool/source (`tool`, `title`, `url`, `source`, `type: "tool_reference"`). |
| `ToolFileChunk` | Generated file (e.g. image) reference (`tool`, `file_id`, `file_name`, `file_type`, `type: "tool_file"`). |
| `ReferenceChunk` | Reference to an indexed document (Chat Completions citations; `reference_ids`). |

---

## 4. Built-in Tools

Built-in tools are ready out of the box, executed in Mistral's internal environment, and can be called at any point. They're also available via the Conversations API without creating an agent first (pass them in `tools`). Multiple tools can be used simultaneously.

### 4.1 Web Search

Browse the web for up-to-date information and access specific websites. Two versions:

| Tool | Description |
|------|-------------|
| `web_search` | Simple web search tool — access to a search engine. |
| `web_search_premium` | Advanced — search engine + news articles via integrated news provider verification. |

**Usage:** `{"type": "web_search"}` in the agent's `tools` array.

```python
websearch_agent = client.beta.agents.create(
    model="mistral-medium-latest",
    name="Websearch Agent",
    description="Agent able to search information over the web.",
    instructions="You have the ability to perform web searches with `web_search`.",
    tools=[{"type": "web_search"}],
    completion_args={"temperature": 0.3, "top_p": 0.95},
)
response = client.beta.conversations.start(
    agent_id=websearch_agent.id,
    inputs="Who won the last European Football cup?",
)
```

**Output entries:**
- `tool.execution` (`name: "web_search"`) — the search execution with timestamps and ID.
- `message.output` — the answer with `content` chunks: `TextChunk` (response text) interleaved with `ToolReferenceChunk` citations (`tool: "web_search"`, `title`, `url`, `source`).

### 4.2 Code Interpreter

Safely execute code in an isolated container on demand — drawing graphs, data analysis, mathematical operations, code validation.

**Usage:** `{"type": "code_interpreter"}` in the agent's `tools` array.

```python
code_agent = client.beta.agents.create(
    model="mistral-medium-latest",
    name="Coding Agent",
    description="Agent used to execute code using the interpreter tool.",
    instructions="Use the code interpreter tool when you have to run code.",
    tools=[{"type": "code_interpreter"}],
    completion_args={"temperature": 0.3, "top_p": 0.95},
)
response = client.beta.conversations.start(
    agent_id=code_agent.id,
    inputs="Run a fibonacci function for the first 20 values.",
)
```

**Output entries:**
- `message.output` — initial response indicating capability.
- `tool.execution` (`name: "code_interpreter"`) — execution metadata + `info` section:
  - `code` — the actual Python code executed.
  - `code_output` — the output of the executed code.
- `message.output` — final response with results.

### 4.3 Image Generation

Generate images on demand based on a prompt or description.

**Usage:** `{"type": "image_generation"}` in the agent's `tools` array. Also supported in the Chat Completions API.

```python
image_agent = client.beta.agents.create(
    model="mistral-medium-latest",
    name="Image Generation Agent",
    description="Agent used to generate images.",
    instructions="Use the image generation tool when you have to create images.",
    tools=[{"type": "image_generation"}],
    completion_args={"temperature": 0.3, "top_p": 0.95},
)
response = client.beta.conversations.start(
    agent_id=image_agent.id,
    inputs="Generate an orange cat in an office.",
)
```

**Output entries:**
- `tool.execution` (`name: "image_generation"`) — execution metadata.
- `message.output` — answer with `content` chunks including `ToolFileChunk` (`tool: "image_generation"`, `file_id`, `file_name`, `file_type: "png"`).

**Downloading images:** Use the Files endpoint to download by `file_id`.

```python
file_bytes = client.files.download(file_id=file_chunk.file_id).read()
with open("image_generated.png", "wb") as f:
    f.write(file_bytes)
```

### 4.4 Document Library

Built-in RAG tool for knowledge grounding and search on custom data uploaded to a **Library**. Connected per-agent via `library_ids`.

**Usage:** `{"type": "document_library", "library_ids": ["..."]}` in the agent's `tools` array. See [Libraries](#9-libraries-rag-knowledge-bases).

```python
library_agent = client.beta.agents.create(
    model="mistral-medium-latest",
    name="Document Library Agent",
    description="Agent used to access documents from the document library.",
    instructions="Use the library tool to access external documents.",
    tools=[{"type": "document_library", "library_ids": [new_library.id]}],
    completion_args={"temperature": 0.3, "top_p": 0.95},
)
response = client.beta.conversations.start(
    agent_id=library_agent.id,
    inputs="How does the vision encoder for pixtral 12b work",
)
```

**Output entries:** `tool.execution` (search ran) + `message.output` (grounded answer with `text` and `tool_reference` citation chunks). The `usage` object includes `connector_tokens` consumed by the Library search.

---

## 5. Function Calling (Custom Tools)

Custom (user-defined) tools executed **locally** in your environment, using standard JSON-schema function calling. The model emits a structured request; your code runs it; the result flows back via a `FunctionResultEntry`.

### Define a function tool

```python
tools = [{
    "type": "function",
    "function": {
        "name": "get_european_central_bank_interest_rate",
        "description": "Retrieve the real interest rate of European central bank.",
        "parameters": {
            "type": "object",
            "properties": {
                "date": {"type": "string"},
            },
            "required": ["date"],
        },
    },
}]
```

The `function` schema fields:

| Field | Description |
|-------|-------------|
| `name` | Function name the model calls. |
| `description` | What the function does (guides model selection). |
| `parameters` | JSON Schema object (`type`, `properties`, `required`). Standard JSON-schema properties. |

### Create an agent with function tools

```python
ecb_agent = client.beta.agents.create(
    model="mistral-medium-latest",
    description="Can find the current interest rate of the European central bank",
    name="ecb-interest-rate-agent",
    tools=[{
        "type": "function",
        "function": {
            "name": "get_european_central_bank_interest_rate",
            "description": "Retrieve the real interest rate of European central bank.",
            "parameters": {
                "type": "object",
                "properties": {"date": {"type": "string"}},
                "required": ["date"],
            },
        },
    }],
)
```

### Handle the function call

When the model wants to call a function, the response contains a `function.call` entry. Detect it, run your function, and return the result via `FunctionResultEntry`:

```python
response = client.beta.conversations.start(
    agent_id=ecb_agent.id,
    inputs=[{"role": "user", "content": "Whats the current 2025 real interest rate?"}],
)

if (response.outputs[-1].type == "function.call"
        and response.outputs[-1].name == "get_european_central_bank_interest_rate"):

    function_result = json.dumps(
        get_european_central_bank_interest_rate(**json.loads(response.outputs[-1].arguments))
    )

    user_function_calling_entry = FunctionResultEntry(
        tool_call_id=response.outputs[-1].tool_call_id,
        result=function_result,
    )

    response = client.beta.conversations.append(
        conversation_id=response.conversation_id,
        inputs=[user_function_calling_entry],
    )
    print(response.outputs[-1])
```

`FunctionResultEntry` fields:

| Field | Description |
|-------|-------------|
| `tool_call_id` | Matches the `tool_call_id` from the `function.call` entry. |
| `result` | JSON-stringified result of executing your function. |

### Chat Completions API function calling

Function calling also works in the Chat Completions API (`/v1/chat/completions`), where the model returns a `tool_calls` response. `tool_choice` controls behavior: `"auto"` (default — model decides) or `"any"` (force a tool call). The flow: send messages + tools → model returns `tool_calls` → execute locally → append a `tool` role message with `tool_call_id` → re-complete for a natural-language answer.

---

## 6. Connectors (Managed MCP Servers)

**Connectors** are registered [MCP servers](https://modelcontextprotocol.io/) used as tool sources in conversations and Agents — without managing MCP transport locally. Once registered, a Connector exposes its tools to the model on demand; the model discovers them automatically and calls the right one based on the user's request. Tools execute server-side.

> **Public Preview** — the API interface can change.

API keys can be scoped for Connector access (Workspace-shared only, or both private + shared).

### 6.1 Create a Connector

`client.beta.connectors.create_async(...)` — register an MCP server.

| Field | Required | Description |
|-------|----------|-------------|
| `name` | Yes | Unique name (≤64 chars, alphanumeric + `_`/`-`). Referenced by name or UUID. |
| `server` | Yes | MCP server URL. |
| `visibility` | Yes | `private`, `shared_workspace`, or `shared_org`. |
| `description` | No | Connector description. |
| `icon_url` | No | Icon URL. |
| `headers` | No | HTTP headers sent with every MCP request (e.g. static API keys). |
| `auth_data` | No | OAuth2 `client_id` + `client_secret` (when the MCP server uses OAuth). |
| `system_prompt` | No | System prompt injected when the Connector's tools are used. |

```python
connector = await client.beta.connectors.create_async(
    name="my_deepwiki",
    description="DeepWiki MCP connector for code repository exploration",
    server="https://mcp.deepwiki.com/mcp",
    visibility="private",
)
```

### 6.2 Authenticate a Connector (OAuth)

If the MCP server requires OAuth, retrieve the authorization URL and redirect the user. After completion, they're redirected to `app_return_url`. Programmatic token passing is not supported (use Studio).

`client.beta.connectors.get_auth_url_async(connector_id_or_name, app_return_url)` → `auth_url`, `ttl`.

### 6.3 Connector lifecycle

| Operation | SDK method | Notes |
|-----------|-----------|-------|
| Get | `connectors.get_async(connector_id_or_name)` | By name or UUID; includes `tools` array if discovered. |
| List | `connectors.list_async(page_size, cursor, query_filters)` | Cursor pagination (`next_cursor`); filter `active: true`. |
| List tools | `connectors.list_tools_async(connector_id_or_name, page, page_size, refresh, pretty)` | Query params: `page` (default 1), `page_size` (100), `refresh` (re-fetch from server), `pretty` (simplified payload: `name`, `description`, `annotations`, `inputSchema`). |
| Update | `connectors.update_async(connector_id, ...)` | `connector_id` must be UUID. Updatable: `name`, `description`, `server`, `icon_url`, `system_prompt`, `headers`, `auth_data`. |
| Delete | `connectors.delete_async(connector_id)` | Permanent; agents referencing it lose access. |

### 6.4 Use Connectors in conversations

Pass a Connector as a `tool` with `type: "connector"` and `connector_id` (name or UUID). All exposed tools become available.

```python
response = await client.beta.conversations.start_async(
    model="mistral-small-latest",
    inputs=[{"role": "user", "content": "Using deepwiki, tell me about the sqlite/sqlite repository."}],
    tools=[{"type": "connector", "connector_id": "my_deepwiki"}],
)
```

**Filter tools** via `tool_configuration` (use `include` **or** `exclude`, not both):

```python
# Exclude specific tools
{"type": "connector", "connector_id": "my_deepwiki",
 "tool_configuration": {"exclude": ["read_wiki_structure"]}}

# Include only specific tools
{"type": "connector", "connector_id": "my_deepwiki",
 "tool_configuration": {"include": ["ask_question"]}}
```

`tool_configuration` fields:

| Field | Description |
|-------|-------------|
| `include` | Allowlist of tool names the model can use. |
| `exclude` | Blocklist of tool names. |
| `requires_confirmation` | List of tool names requiring human approval before execution. |

**Mix with built-in tools** — pass built-in tools and connectors in the same `tools` array; the model picks based on the query.

**Attach to an Agent** — include the connector in the agent's `tools` at creation; conversations started with that agent get the tools automatically (no `tools` array needed at conversation start).

### 6.5 Direct tool calling (no model)

`client.beta.connectors.call_tool_async(connector_id_or_name, tool_name, arguments)` — call a specific MCP tool directly without starting a conversation or involving the model. The response `content` array contains typed content blocks (`TextContent`, `ImageContent`, etc.). Useful for building pipelines by chaining multiple tool calls in sequence. If the Connector requires auth, complete the auth flow first.

---

## 7. Human-in-the-Loop (Tool Confirmation)

The `requires_confirmation` parameter intercepts tool calls before execution so a user/system can approve or deny each action. Works for all tool types: Connectors, built-in tools (e.g. `web_search_premium`), and local functions (Python SDK only).

### Configuration

Add `requires_confirmation` to `tool_configuration`, listing tool names requiring approval:

```python
tools=[
    {"type": "connector", "connector_id": "gmail",
     "tool_configuration": {"requires_confirmation": ["gmail_search"]}},
    {"type": "web_search_premium",
     "tool_configuration": {"requires_confirmation": ["web_search", "news_search"]}},
]
```

### REST API flow (two steps)

**1. Start the conversation** — the API returns a pending `function.call` entry instead of running the tool:

```json
{
  "conversation_id": "conv_abc123...",
  "outputs": [{
    "type": "function.call",
    "tool_call_id": "WJfo42Ow3",
    "name": "gmail_search",
    "arguments": "{\"limit\": 1}",
    "confirmation_status": "pending"
  }]
}
```

**2. Approve or deny** — `POST /v1/conversations/{conversation_id}` with `tool_confirmations`:

```json
{
  "tool_confirmations": [
    {"tool_call_id": "WJfo42Ow3", "confirmation": "allow"}
  ]
}
```

`confirmation` values: `"allow"` (run the tool) or `"deny"` (reject; the model handles rejection gracefully). Multiple pending calls can be approved/denied individually or batched.

### Python SDK flow

The SDK provides `RunContext` and `DeferredToolCallsException` (requires `mistralai` v2.4+ with the `mcp` extra). `run_async` runs the conversation loop and raises `DeferredToolCallsException` when a tool requires confirmation; call `dc.confirm()` or `dc.reject("reason")` on each deferred call, then loop back to resume.

```python
from mistralai.extra.run.context import RunContext
from mistralai.extra.exceptions import DeferredToolCallsException

async def main():
    client = Mistral(api_key=os.environ["MISTRAL_API_KEY"])
    conversation_id = None
    pending_inputs = [{"role": "user", "content": "I need a vacation somewhere warm next Friday."}]
    while True:
        async with RunContext(model="mistral-large-latest") as run_ctx:
            run_ctx.conversation_id = conversation_id
            run_ctx.register_func(get_weather, requires_confirmation=False)
            run_ctx.register_func(book_flight, requires_confirmation=True)
            try:
                result = await client.beta.conversations.run_async(
                    run_ctx=run_ctx, inputs=pending_inputs,
                    instructions="You are a travel assistant.",
                )
                print(result.output_entries)
                break
            except DeferredToolCallsException as deferred:
                conversation_id = deferred.conversation_id
                pending_inputs = []
                for dc in deferred.deferred_calls:
                    approved = input(f"Approve {dc.tool_name}? (y/n): ").strip().lower() == "y"
                    pending_inputs.append(dc.confirm() if approved else dc.reject("Denied"))
```

**Local functions** are registered with `register_func(func, requires_confirmation=True/False)`. **Connector tools** work the same way. **Stateless resumption** (e.g. web APIs): serialize via `deferred.to_dict()` and reconstruct later; prepend `deferred.executed_results` (results from tools that ran before deferral) when resuming so the model has full context.

---

## 8. Handoffs (Multi-Agent Orchestration)

**Handoffs** let one agent hand over a conversation to other agents mid-action, enabling agentic workflows with chained responsibilities. There is no limit to chained handoffs; you orchestrate your own workflow with diverse tools, models, and handoffs.

### Create multiple agents

Create each specialist agent independently (finance, web search, ECB rate, etc.) with their own tools — built-in or custom functions.

### Define handoff responsibilities

Update each agent with a list of `handoffs` (agent IDs it can delegate to):

```python
finance_agent = client.beta.agents.update(
    agent_id=finance_agent.id,
    handoffs=[ecb_interest_rate_agent.id, web_search_agent.id],
)
ecb_interest_rate_agent = client.beta.agents.update(
    agent_id=ecb_interest_rate_agent.id,
    handoffs=[graph_agent.id, calculator_agent.id],
)
web_search_agent = client.beta.agents.update(
    agent_id=web_search_agent.id,
    handoffs=[graph_agent.id, calculator_agent.id],
)
```

`update` parameters:

| Parameter | Description |
|-----------|-------------|
| `agent_id` | The agent to update. |
| `handoffs` | List of agent IDs this agent can hand off to. |

### Execution modes — `handoff_execution`

| Mode | Behavior |
|------|----------|
| `server` | Runs the handoff internally on Mistral's cloud servers (default). |
| `client` | When a handoff is triggered, a response is provided directly to the user, enabling them to handle the handoff with control. |

Set on `conversations.start` / `conversations.append`.

### Run the workflow

Start a conversation with the entry-point agent; the agent loop chains handoffs server-side:

```python
response = client.beta.conversations.start(
    agent_id=finance_agent.id,
    inputs="Fetch the current US bank interest rate and calculate the compounded effect if investing for the next 10y",
)
```

### Output events

The conversation produces a sequence of typed entries in order:

| Entry type | Description |
|------------|-------------|
| `agent.handoff` | Delegation to another agent (`agent_id`, `agent_name`, `id`, `created_at`). |
| `tool.execution` | Built-in/custom tool execution (e.g. `web_search`) with `created_at`/`completed_at`. |
| `message.output` | Agent message with `content` chunks (`TextChunk`, `ToolReferenceChunk`), `agent_id`, `model`, `role`. |
| `function.call` | Custom function request (when using function tools mid-handoff). |

A typical chain: `agent.handoff` → `tool.execution` → `message.output` (interim) → `agent.handoff` → `message.output` (final).

---

## 9. Libraries (RAG Knowledge Bases)

**Libraries** are persistent knowledge bases filled with documents and connected to agents for built-in RAG. Upload PDFs, papers, or any document; agents search through them on demand via the `document_library` tool. Libraries created in Vibe can also be used via the API (share with the Organization; Org admin required) and vice versa.

### Create a Library

`client.beta.libraries.create(name, description)` — returns metadata including `generated_name`/`generated_description` (auto-updated as files are added). A new library starts empty.

### Document management

| Operation | SDK method | Notes |
|-----------|-----------|-------|
| Upload | `libraries.documents.upload(library_id, file=File(...))` | `File(fileName, content)`. |
| List docs | `libraries.documents.list(library_id)` | Each doc: `name`, `extension`, `number_of_pages`, `summary`. |
| Status | `libraries.documents.status(library_id, document_id)` | `processing_status`: `Running` → `Completed`. |
| Get info | `libraries.documents.get(library_id, document_id)` | Full metadata once processed. |
| Text content | `libraries.documents.text_content(library_id, document_id)` | Extracted text (+ `signed_url` for extracted/raw). |
| Delete | `libraries.delete(library_id)` | Deletes the library. |

### Library listing & access control

| Operation | SDK method | Notes |
|-----------|-----------|-------|
| List libraries | `libraries.list()` | Each with `nb_documents`. |
| List accesses | `libraries.accesses.list(library_id)` | Entities with access and their level. |
| Access params | `org_id`, `level` (`"Viewer"`/`"Editor"`), `share_with_uuid`, `share_with_type` (`"User"`/`"Workspace"`/`"Org"`) | Owner-only sharing/deletion; owners can't delete their own access; Viewers can't edit. |

### Connect a Library to an agent

Create an agent with the `document_library` tool and pass `library_ids`:

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

`document_library` tool fields:

| Field | Description |
|-------|-------------|
| `type` | `"document_library"`. |
| `library_ids` | List of Library IDs the agent can search. |

Querying: `conversations.start(agent_id=library_agent.id, inputs="...")`. Response includes `tool.execution` (search) + `message.output` (grounded answer with `tool_reference` citations). `usage.connector_tokens` reflects Library search consumption.

---

## 10. Citations & References

Citations let models ground responses and provide source references — powerful for RAG and agentic applications. Mistral models are trained to ground on documents and extract references from tool-call data.

### References structure (provided to the model via tool results)

```json
{
  "0": {
    "url": "https://en.wikipedia.org/wiki/2024_Nobel_Peace_Prize",
    "title": "2024 Nobel Peace Prize",
    "snippets": [["...extracted text..."]],
    "description": null,
    "date": "2024-11-26T17:39:55.057454",
    "source": "wikipedia"
  }
}
```

Reference fields: `url`, `title`, `snippets` (list of text passages), `description`, `date`, `source`.

### Flow (Chat Completions)

1. Define a function tool returning reference data (with `strict: true` for schema enforcement).
2. Send the user message + tool → model returns a `tool_calls` response.
3. Execute the function locally, append a `ToolMessage` (`content`, `tool_call_id`, `name`).
4. Re-complete → model returns the answer with `content` chunks: `TextChunk` (answer) and `ReferenceChunk` (`reference_ids` pointing to the references).

### Conversations API citations

In the Conversations/Agents API, `web_search`, `web_search_premium`, and `document_library` produce `ToolReferenceChunk` entries interleaved with `TextChunk` in `message.output.content` — each carrying `tool`, `title`, `url`, `source` for verifiable, traceable outputs.

---

## 11. Capability Summary & Cross-Reference

| Capability | Primary resource(s) | Key endpoints / SDK calls | Core parameters |
|------------|---------------------|---------------------------|-----------------|
| Agent management | Agent | `client.beta.agents.create`, `.update`, `.list`, `.delete` | `model`, `name`, `description`, `instructions`, `tools`, `completion_args`, `handoffs` |
| Conversations | Conversation | `client.beta.conversations.start`, `.append` (`POST /v1/conversations`, `POST /v1/conversations/{id}`) | `agent_id` \| `model`, `inputs`, `store`, `handoff_execution`, `guardrails`, `tools`, `tool_confirmations` |
| Built-in tools | Agent/Conversation `tools[]` | Agent create; conversation start | `type`: `web_search`/`web_search_premium`/`code_interpreter`/`image_generation`/`document_library` (+ `library_ids`) |
| Function calling | Custom function tool | Agent create with `type: "function"`; `conversations.append` with `FunctionResultEntry` | `function.name`, `description`, `parameters` (JSON Schema); `FunctionResultEntry(tool_call_id, result)` |
| Connectors | Connector | `connectors.create_async`, `.get_async`, `.list_async`, `.list_tools_async`, `.update_async`, `.delete_async`, `.call_tool_async` | `name`, `server`, `visibility`, `headers`, `auth_data`, `system_prompt`; usage: `type: "connector"`, `connector_id`, `tool_configuration` (`include`/`exclude`/`requires_confirmation`) |
| Human-in-the-loop | `tool_configuration.requires_confirmation` | `POST /v1/conversations/{id}` with `tool_confirmations`; SDK `RunContext` + `DeferredToolCallsException` | `requires_confirmation: [tool_names]`; `confirmation: "allow"\|"deny"` |
| Handoffs | Agent `handoffs` + conversation `handoff_execution` | `agents.update(handoffs=[...])`; `conversations.start(handoff_execution=...)` | `handoffs` (agent ID list); `handoff_execution`: `server` \| `client` |
| Libraries | Library + documents | `libraries.create`, `.list`, `.delete`; `libraries.documents.upload/list/status/get/text_content` | `name`, `description`; tool: `type: "document_library"`, `library_ids` |
| Citations | Reference data + tool results | Provided via function/`document_library`/`web_search` outputs | `url`, `title`, `snippets`, `source`, `date`; chunks: `ToolReferenceChunk`/`ReferenceChunk` |

### Key design principles

1. **Independent Agents & Conversations APIs** — Agents are optional; conversations can use `model` directly. Both are first-class.
2. **Server-side agent loop** — Built-in tools and Connector (MCP) tools execute in Mistral's environment; custom functions are returned to your app for local execution.
3. **Entry-based conversations** — Interactions are typed Entries (`message.output`, `tool.execution`, `function.call`, `agent.handoff`), more expressive than plain messages.
4. **Reusable agents** — Create once, reference by ID across many conversations; update mutates in place (not versioned).
5. **Managed MCP via Connectors** — Register MCP servers once; tools auto-discovered and called server-side; no local transport management. Direct tool calling also available without the model.
6. **Human-in-the-loop** — `requires_confirmation` intercepts tool calls (Connectors, built-in, local functions) for approval/denial; REST or SDK flow with serializable deferred state.
7. **Multi-agent handoffs** — Chain agents via `handoffs` lists; `server` execution runs autonomously, `client` returns control to the user.
8. **Built-in RAG** — Libraries + `document_library` tool give agents grounded knowledge with citations (`tool_reference` chunks).
9. **Citations & references** — Models trained to ground on documents and provide sources, interleaving `TextChunk` with reference chunks for verifiable outputs.

### API surface notes

- Agents and Conversations live under the `beta` namespace in the SDK (`client.beta.*`), with sync and async variants.
- Built-in tools `web_search`/`web_search_premium`/`code_interpreter` are Conversations/Agents-only (not Chat Completions); `image_generation` also works in Chat Completions.
- `agent_id` and `model` are mutually exclusive when starting a conversation.
- Connectors are a Public Preview feature (interface may change).
- API keys can be scoped for Connector access (shared-only vs. private + shared).