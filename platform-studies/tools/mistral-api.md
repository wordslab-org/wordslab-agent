# Mistral API — Agent Tools & Connectors Capabilities

Analysis of the agent-related capabilities offered by the Mistral Studio API, based on the official developer guides at https://docs.mistral.ai/studio-api/agents/agent-tools and https://docs.mistral.ai/studio-api/connectors. Each capability is broken down into main concepts, API surface (functions, parameters, response fields), and notes/constraints.

Mistral exposes agentic capabilities through two complementary APIs: the **Agents API** (`client.beta.agents.*`) and the **Conversations API** (`client.beta.conversations.*` via `/v1/conversations`), distinct from the lower-level **Chat Completions API** (`/v1/chat/completions`). Most built-in tools are only available through the Conversations/Agents APIs because their rich outputs (references, files, tool executions) are not expressible in Chat Completions responses.

---

## Table of Contents

1. [Agents & Conversations core model](#1-agents--conversations-core-model)
2. [Web Search](#2-web-search)
3. [Code Interpreter](#3-code-interpreter)
4. [Image Generation](#4-image-generation)
5. [Function Calling (custom local tools)](#5-function-calling-custom-local-tools)
6. [Handoffs (agent-to-agent workflows)](#6-handoffs-agent-to-agent-workflows)
7. [Connectors — MCP server management](#7-connectors--mcp-server-management)
8. [Connectors — Use in conversations & Agents](#8-connectors--use-in-conversations--agents)
9. [Connectors — Direct tool calling](#9-connectors--direct-tool-calling)
10. [Connectors — Human-in-the-loop confirmation](#10-connectors--human-in-the-loop-confirmation)
11. [Connectors Debugger](#11-connectors-debugger)

---

## 1. Agents & Conversations core model

**Summary** — Agents are preconfigured bundles (model + instructions + tools + completion args) and Conversations are persistent, entry-based interaction histories that can be driven by an Agent or directly by a model.

### Main concepts
- **Agent**: Pre-selected values augmenting a model — `model`, `description`, `name`, `instructions` (system prompt), `tools`, `handoffs`, and `completion_args`.
- **Conversation**: A persistent history of entries (messages + tool executions + handoffs). Can be started with either an `agent_id` or a bare `model`; the two APIs are independent.
- **Entry**: A typed action in a conversation, either user- or assistant-produced (`message.output`, `tool.execution`, `function.call`, `agent.handoff`, etc.). More expressive than plain chat messages.
- **Persistent state**: Conversations are stored on Mistral's cloud by default; pass `store=False` to opt out for a given append.
- **API surface**:
  - Agents: `client.beta.agents.create`, `.update`, `.get`, `.list`, `.delete`.
  - Conversations: `client.beta.conversations.start`, `.append`, `.retrieve`, `.restart`, `.stream`.

### API functions & parameters

**Create an Agent** — `client.beta.agents.create`:

| Parameter | Type | Description |
|---|---|---|
| `model` | string | Underlying chat completion model (e.g. `"mistral-medium-latest"`, `"mistral-large-latest"`). |
| `name` | string | Agent name. |
| `description` | string | Short description of the agent's task/use case. |
| `instructions` | string (optional) | System prompt describing the agent's main task. |
| `tools` | array (optional) | List of tool descriptors (built-in tools, `function`, or `connector`). |
| `handoffs` | array (optional) | List of agent IDs the agent can hand the conversation off to. |
| `completion_args` | object (optional) | Standard chat completion sampler args (e.g. `temperature`, `top_p`, `response_format`). |

Tool `type` values accepted in `tools`:
- `"web_search"` / `"web_search_premium"` — built-in web search.
- `"code_interpreter"` — built-in sandboxed code execution.
- `"image_generation"` — built-in image generation.
- `"document_library"` — built-in RAG over uploaded Libraries.
- `"function"` — user-defined function (custom local tool).
- `"connector"` — registered MCP server Connector.

**Start a Conversation** — `client.beta.conversations.start`:

| Parameter | Type | Description |
|---|---|---|
| `agent_id` | string (optional) | ID of an existing Agent. Mutually exclusive with `model`. |
| `model` | string (optional) | A model to drive the conversation without an Agent. |
| `inputs` | string \| array | First user message, or a list of message entries. |
| `store` | boolean (optional) | Default `true`; `false` disables cloud storage of the new history. |
| `tools` | array (optional) | Ad-hoc tools for this conversation (when not using an Agent). |
| `handoff_execution` | enum (optional) | `"server"` (default — runs handoffs internally) or `"client"` (returns handoff to caller). |
| `guardrails` | object (optional) | Override agent guardrails for a model-driven conversation. |

**Continue a Conversation** — `client.beta.conversations.append`: takes `conversation_id` and `inputs` (string or list of entries, including `FunctionResultEntry` / tool confirmations). Returns a new conversation ID.

### Notes
- Conversations can be read/modified only by the creator of the API key used in the request.
- A new `conversation_id` is produced at each append; the older ID remains valid for retrieval.
- Guardrails configured on an Agent are automatically applied to all its conversations; they can be overridden per-request when using `model` directly.
- For full API specs see the `beta.agents` and `beta.conversations` tags in the API reference.

---

## 2. Web Search

**Summary** — Built-in tool allowing models to search the web for up-to-date information and return answers with inline source references (citations).

### Main concepts
- **Two variants**:
  - `web_search` — basic access to a search engine.
  - `web_search_premium` — adds access to news articles via integrated news-provider verification.
- **Citations / references**: The assistant message `content` is a list of chunks interleaving `text` chunks and `tool_reference` chunks (URL, title, source) used as RAG-style citations.
- **API availability**: Only via the Conversations API (`/v1/conversations`) and Agents API — **not** Chat Completions, because Chat Completions responses cannot carry the search result references.

### API functions & parameters

**Agent creation** — enable via `tools=[{"type": "web_search"}]` (or `"web_search_premium"`). Typical `completion_args` shown in docs: `{"temperature": 0.3, "top_p": 0.95}`.

**Conversation** — `client.beta.conversations.start(agent_id=..., inputs="...")`. No tool-specific request parameters beyond the `type`.

**Response entries** (in `outputs`):
- `tool.execution` entry — tool run metadata: `name` (`"web_search"`), `object` (`"entry"`), `type` (`"tool.execution"`), `id`, `created_at`, `completed_at`.
- `message.output` entry — generated answer with `content` chunks:
  - `text` chunk: `{ "type": "text", "text": "..." }`.
  - `tool_reference` chunk: `{ "type": "tool_reference", "tool": "web_search", "title": "...", "url": "...", "source": "..." }`.
  - Also includes `object`, `type` (`"message.output"`), `id`, `agent_id`, `model`, `role` (`"assistant"`), `created_at`, `completed_at`.

### Notes
- `web_search` / `web_search_premium` are unsupported in the Chat Completions API.
- The Document Library tool is the other reference-emitting tool; see the Libraries guide and the citations guide for citation rendering details.
- Multiple tools can be enabled at the same time on a single agent/conversation.

---

## 3. Code Interpreter

**Summary** — Built-in tool that lets agents safely execute code in an isolated container on demand — useful for data analysis, plotting, math, code validation, and sandboxed execution.

### Main concepts
- **`code_interpreter`**: Execution happens server-side in an isolated container; the model decides when to run code.
- **API availability**: Conversations API and Agents API only — **not** Chat Completions.
- **Tool execution info**: The `tool.execution` entry carries an `info` block with `code` (the code that ran) and `code_output` (the execution stdout/result).

### API functions & parameters

**Agent creation** — `tools=[{"type": "code_interpreter"}]`. Typical `completion_args`: `{"temperature": 0.3, "top_p": 0.95}`.

**Conversation** — `client.beta.conversations.start(agent_id=..., inputs="...")`.

**Response entries** (in `outputs`):
- `message.output` (initial) — assistant acknowledges the task.
- `tool.execution` entry — `name` (`"code_interpreter"`), `object` (`"entry"`), `type` (`"tool.execution"`), `id`, `created_at`, `completed_at`, and `info`:
  - `info.code`: the executed source code (e.g. a Python `fibonacci(n)` function and its invocation).
  - `info.code_output`: the output produced by the executed code.
- `message.output` (final) — assistant summarizes the result.

### Notes
- The interpreter is built into the platform; no container configuration or file uploads are exposed in the documented surface.
- Works with the Conversations API (`/v1/conversations`) and Agents API; not supported in Chat Completions.
- Composable with other tools (web search, image generation, function calling, connectors) on the same agent.

---

## 4. Image Generation

**Summary** — Built-in tool that enables agents to generate images on demand; generated images are returned as files that must be downloaded via the Files endpoint.

### Main concepts
- **`image_generation`**: The model decides when to generate an image; execution is server-side.
- **Tool file chunk**: The assistant's `message.output` content includes a `tool_file` chunk referencing the generated file by `file_id`, `file_name`, and `file_type` (e.g. `png`).
- **Download**: Generated files are retrieved through the standard Files endpoint (`client.files.download(file_id=...)`).
- **API availability**: Works with Conversations API, Agents API, **and** the Chat Completions API.

### API functions & parameters

**Agent creation** — `tools=[{"type": "image_generation"}]`. Typical `completion_args`: `{"temperature": 0.3, "top_p": 0.95}`.

**Conversation** — `client.beta.conversations.start(agent_id=..., inputs="Generate an orange cat in an office.")`.

**Response entries** (in `outputs`):
- `tool.execution` entry — `name` (`"image_generation"`), `object` (`"entry"`), `type` (`"tool.execution"`), `id`, `created_at`, `completed_at`.
- `message.output` entry — `content` list of chunks:
  - `text` chunk: `{ "type": "text", "text": "..." }`.
  - `tool_file` chunk: `{ "type": "tool_file", "tool": "image_generation", "file_id": "...", "file_name": "...", "file_type": "png" }`.
  - Plus `object` (`"entry"`), `type` (`"message.output"`), `id`, `agent_id`, `model`, `role` (`"assistant"`), `created_at`, `completed_at`.

**Download** — iterate `response.outputs[-1].content`, filter for `ToolFileChunk` instances, and download each:
```python
file_bytes = client.files.download(file_id=chunk.file_id).read()
with open(f"image_generated_{i}.png", "wb") as f:
    f.write(file_bytes)
```

### Notes
- No generation parameters (size, quality, format, etc.) are exposed on the tool object in the documented surface; configuration is implicit.
- The `ToolFileChunk` type lives in `mistralai.client.models`.
- Multiple tools can be combined on the same agent; the model chooses when to generate images.

---

## 5. Function Calling (custom local tools)

**Summary** — Standard function-calling mechanism letting you define custom tools via a JSON schema that the model can call; execution happens locally in your environment, and you return the result back to the conversation.

### Main concepts
- **Function definition**: A Python (or local) function in your codebase whose behavior you implement.
- **Schema**: A JSON descriptor passed as a `"function"` tool with `name`, `description`, and `parameters` (JSON schema object). Names/parameters must match the function.
- **Round-trip**: The model emits a `function.call` entry (with `tool_call_id`, `name`, `arguments` as JSON string). You execute the function locally and return a `FunctionResultEntry` via `conversations.append`.
- **API availability**: Usable through Agents and Conversations; full function calling details live in the dedicated function calling docs.

### API functions & parameters

**Agent creation** with a custom function tool:
```python
client.beta.agents.create(
    model="mistral-medium-latest",
    description="...",
    name="...",
    tools=[{
        "type": "function",
        "function": {
            "name": "get_european_central_bank_interest_rate",
            "description": "Retrieve the real interest rate of European central bank.",
            "parameters": {
                "type": "object",
                "properties": {
                    "date": {"type": "string"}
                },
                "required": ["date"]
            }
        }
    }]
)
```

`function` tool descriptor fields:

| Field | Type | Description |
|---|---|---|
| `type` | string | `"function"` (fixed). |
| `function.name` | string | Function name; must match your local function. |
| `function.description` | string | Description shown to the model. |
| `function.parameters` | JSON schema | Argument schema (`type: "object"` with `properties` and `required`). |

**Conversation & result handling**:
```python
response = client.beta.conversations.start(agent_id=..., inputs=[{"role": "user", "content": "..."}])

if response.outputs[-1].type == "function.call" and response.outputs[-1].name == "...":
    function_result = json.dumps(my_function(**json.loads(response.outputs[-1].arguments)))
    entry = FunctionResultEntry(
        tool_call_id=response.outputs[-1].tool_call_id,
        result=function_result,
    )
    response = client.beta.conversations.append(
        conversation_id=response.conversation_id,
        inputs=[entry],
    )
```

**`function.call` entry fields** (in `outputs`): `type` (`"function.call"`), `tool_call_id`, `name`, `arguments` (JSON string), and (with confirmation enabled) `confirmation_status`.

### Notes
- Function calling is also available directly via the Conversations API without creating an Agent.
- For more advanced flows (parallel calls, schema validation, etc.) see the dedicated function calling guide (`/studio-api/conversations/function-calling`).
- Local functions can be combined with built-in tools and Connectors on the same agent.

---

## 6. Handoffs (agent-to-agent workflows)

**Summary** — Lets an Agent hand a conversation over to other Agents mid-task, enabling multi-agent workflows where specialized agents (with different tools/models) cooperate to fulfill a request.

### Main concepts
- **Handoff**: An Agent is configured with a `handoffs` list of other agent IDs it may delegate to. There is no limit on the number of chained handoffs.
- **`agent.handoff` entry**: A conversation entry recording the handoff (`agent_id`, `agent_name`, `id`, `created_at`).
- **`handoff_execution`**: Controls where handoffs run:
  - `"server"` (default) — executed internally on Mistral's cloud.
  - `"client"` — the handoff is returned to the caller, who handles it with control.
- **Workflow construction**: Create all agents first (with their tools/completion args), then update each agent's `handoffs` list to wire the graph.

### API functions & parameters

**Define handoffs** — `client.beta.agents.update(agent_id=..., handoffs=[<other_agent_id>, ...])`.

**Run a workflow** — start a conversation against the entry agent:
```python
response = client.beta.conversations.start(
    agent_id=finance_agent.id,
    inputs="Fetch the current US bank interest rate and calculate the compounded effect if investing for the next 10y",
    # handoff_execution="client"  # optional override
)
```

**Output events** (in order, illustrative):
1. `agent.handoff` — to the chosen sub-agent (e.g. `websearch-agent`).
2. `tool.execution` — e.g. `web_search` runs.
3. `message.output` — sub-agent answer with `TextChunk` + `ToolReferenceChunk` citations, plus handoff-to-next reasoning text.
4. `agent.handoff` — to the next agent (e.g. `calculator-agent`).
5. `message.output` — final answer (optionally structured JSON when `response_format` is set on the agent's `completion_args`).

### Notes
- Agents in a workflow can mix built-in tools, custom functions, and different models.
- `completion_args.response_format` can be set per-agent (e.g. `json_schema` with a pydantic `model_json_schema()`) so a sub-agent returns structured output.
- In `client` execution mode the caller inspects each handoff and decides how to proceed, useful for human-in-the-loop or custom orchestration.
- Handoffs are independent of Connectors — Connectors expose MCP tools, while Handoffs delegate entire sub-conversations to other Agents.

---

## 7. Connectors — MCP server management

**Summary** — Connectors are registered MCP (Model Context Protocol) servers used as tools in conversations and Agents without managing MCP transport locally. Tools are discovered automatically and executed server-side. This section covers the Connector lifecycle management API.

> **Status**: Connectors are a **Public Preview** feature; the API interface may change.

### Main concepts
- **Connector**: A registered MCP server identified by a unique name (≤64 chars, alphanumeric + `_`/`-`) and a UUID. Names are unique within a Workspace.
- **Visibility scopes**:
  - `private` — only the creator can use it.
  - `shared_workspace` — anyone in the same Workspace.
  - `shared_org` — anyone in the organization (org admins only can create/manage these).
- **Auth modes**: Static HTTP headers (`headers`) and OAuth2 (`auth_data` with `client_id`/`client_secret`). OAuth tokens are obtained via an interactive auth URL flow (not programmatically passable).
- **Tool discovery**: The MCP server's tools are auto-discovered and exposed on demand to the model. List/refresh via the list-tools endpoint.
- **API key scoping**: API keys can be scoped to call only Workspace-shared Connectors, or both private and shared ones.

### API functions & parameters

**Create** — `client.beta.connectors.create_async`:

| Parameter | Type | Description |
|---|---|---|
| `name` | string | Unique Connector name (≤64 chars, alphanumeric/`_`/`-`). |
| `description` | string (optional) | Human-readable description. |
| `server` | string | MCP server URL. |
| `visibility` | enum | `"private"` \| `"shared_workspace"` \| `"shared_org"`. |
| `icon_url` | string (optional) | URL of an icon to associate. |
| `headers` | object (optional) | HTTP headers sent with every MCP request (e.g. static API keys). |
| `auth_data` | object (optional) | OAuth2 `{ "client_id", "client_secret" }`. Required when server uses OAuth. |
| `system_prompt` | string (optional) | System prompt injected when the Connector's tools are used. |

Response fields: `id`, `name`, `description`, `server`, `auth_type`, `created_at`, `modified_at`, and `tools[]` (once discovered).

**Get auth URL (OAuth)** — `client.beta.connectors.get_auth_url_async`:

| Parameter | Type | Description |
|---|---|---|
| `connector_id_or_name` | string | Connector name or UUID. |
| `app_return_url` | string | URL the user is redirected to after granting access. |

Returns `{ "auth_url", "ttl" (seconds) }`. Tokens must be obtained via the Studio UI; passing tokens programmatically is not supported.

**Retrieve one** — `client.beta.connectors.get_async(connector_id_or_name=...)` — accepts name or UUID; response includes the `tools[]` array if already discovered.

**List all** — `client.beta.connectors.list_async`:

| Parameter | Type | Description |
|---|---|---|
| `page_size` | int | Items per page. |
| `cursor` | string (optional) | Pagination cursor from `pagination.next_cursor`. |
| `query_filters` | object (optional) | Filter results, e.g. `{"active": true}`. |

**List tools** — `client.beta.connectors.list_tools_async(connector_id_or_name=...)`:

| Query parameter | Default | Description |
|---|---|---|
| `page` | `1` | Offset-based page number. |
| `page_size` | `100` | Tools per page. |
| `refresh` | `false` | Re-fetch tools from the MCP server (bypass cache). |
| `pretty` | `false` | Simplified payload (`name`, `description`, `annotations`, compact `inputSchema`). |

**Update** — `client.beta.connectors.update_async(connector_id=<UUID>, ...)` — updatable fields: `name`, `description`, `server`, `icon_url`, `system_prompt`, `headers`, `auth_data`. Requires the UUID (not the name).

**Delete** — `client.beta.connectors.delete_async(connector_id=<UUID>)` — permanent; any Agents referencing the Connector lose access to its tools.

### Notes
- The full Connector lifecycle is: test in Debugger → create → authenticate (if OAuth) → list tools → use in conversations / direct calls → update/delete.
- Connectors can be referenced interchangeably by name or UUID in read/list-tools operations; updates and deletes require the UUID.
- Connector names are unique within a Workspace.
- If the Connector requires authentication, the user must complete the auth flow before listing or calling its tools.

---

## 8. Connectors — Use in conversations & Agents

**Summary** — After a Connector is registered (and authenticated if required), it can be attached to a conversation or to an Agent so the model automatically discovers and calls its tools based on the query. Connectors can be mixed with built-in tools and tool-filtered per request.

### Main concepts
- **Connector as a tool**: Pass `{"type": "connector", "connector_id": "<name or UUID>"}` in a conversation's or Agent's `tools` array. The model automatically discovers the Connector's tools and picks the right one per query.
- **Tool filtering**: Use `tool_configuration.include` (allowlist) or `tool_configuration.exclude` (blocklist) — mutually exclusive — to control which of the Connector's tools the model may call.
- **Confirmation**: `tool_configuration.requires_confirmation` lists tool names that must be approved before execution (see §10).
- **Agent-attached Connectors**: Add the Connector at Agent creation; every conversation started with that Agent then has access to its tools without passing a `tools` array.
- **Mixing tools**: Built-in tools (`web_search`, `code_interpreter`, `image_generation`, `document_library`) can be passed alongside Connectors in the same `tools` array.

### API functions & parameters

**Attach to a conversation** — `client.beta.conversations.start_async`:
```python
response = await client.beta.conversations.start_async(
    model="mistral-small-latest",
    inputs=[{"role": "user", "content": "..."}],
    tools=[{"type": "connector", "connector_id": "my_deepwiki"}],
)
```

Connector tool descriptor fields:

| Field | Type | Description |
|---|---|---|
| `type` | string | `"connector"` (fixed). |
| `connector_id` | string | Connector name or UUID. |
| `tool_configuration` | object (optional) | Tool filtering & confirmation config. |

`tool_configuration` fields (use `include` **or** `exclude`, not both):

| Field | Type | Description |
|---|---|---|
| `include` | array of strings | Allowlist of tool names the model may call. |
| `exclude` | array of strings | Blocklist of tool names the model may not call. |
| `requires_confirmation` | array of strings | Tool names that require human approval before running. |

**Mix with built-in tools**:
```python
tools=[
    {"type": "web_search"},
    {"type": "connector", "connector_id": "my_deepwiki"},
]
```

**Attach to an Agent** — add the connector tool at creation:
```python
agent = await client.beta.agents.create_async(
    name="deepwiki_agent",
    description="...",
    model="mistral-small-latest",
    instructions="...",
    tools=[{"type": "connector", "connector_id": "my_deepwiki"}],
)
# Then conversations can be started with agent_id and no tools array.
```

### Notes
- When using an Agent, pass `agent_id` instead of `model` when starting a conversation — you cannot pass both.
- `web_search`, `web_search_premium`, and `code_interpreter` are Conversations/Agents API only (not Chat Completions); `image_generation` also works on Chat Completions.
- All of a Connector's tools are available by default; restrict with `include`/`exclude`.
- Use Agent-attached Connectors when an Agent always needs the same external tools (e.g. a support agent that always queries a CRM).

---

## 9. Connectors — Direct tool calling

**Summary** — The `call_tool` method invokes a specific MCP tool on a Connector directly, without starting a conversation or involving the model. Useful when you already know which tool to call, want raw output for downstream processing, or are building programmatic pipelines.

### Main concepts
- **Direct invocation**: Bypasses the model's tool selection; you specify the exact tool and arguments.
- **Typed content blocks**: The response `content` is an array of content blocks (text, image, audio, or resource) returned by the MCP server.
- **Tool name accuracy**: Tool names must match exactly what the Connector exposes — use the list-tools endpoint to verify.
- **Prerequisite**: If the Connector requires auth, the user must complete the auth flow first.

### API functions & parameters

**Call a tool** — `client.beta.connectors.call_tool_async`:

| Parameter | Type | Description |
|---|---|---|
| `connector_id_or_name` | string | Connector name or UUID. |
| `tool_name` | string | Exact tool name as exposed by the Connector. |
| `arguments` | object | Arguments matching the tool's input schema. |

Example:
```python
result = await client.beta.connectors.call_tool_async(
    connector_id_or_name="my_deepwiki",
    tool_name="read_wiki_structure",
    arguments={"repoName": "sqlite/sqlite"},
)
for item in result.content:
    if hasattr(item, "text"):
        print(item.text)
```

**Chaining**: Call multiple tools in sequence to build a pipeline without involving the model at each step (e.g. first `read_wiki_structure`, then `ask_question`).

### Notes
- For scenarios where the model picks the tools, use Connectors in conversations instead (§8).
- Direct calls return raw MCP content blocks; you handle parsing/processing downstream.
- Useful for debugging or verifying Connector tools before using them in conversations.

---

## 10. Connectors — Human-in-the-loop confirmation

**Summary** — The `requires_confirmation` parameter intercepts tool calls before they run, so a user or system can approve or deny each action. Works for all tool types: Connectors, built-in tools (e.g. `web_search_premium`), and local functions (Python SDK only).

### Main concepts
- **Confirmation configuration**: Add `requires_confirmation` to a tool's `tool_configuration`, listing the tool names that require approval.
- **Pending `function.call`**: When a tool requires confirmation, the API returns a `function.call` entry with `confirmation_status: "pending"` instead of running the tool.
- **Approval/denial**: Resume the conversation by sending `tool_confirmations` (REST) with `"allow"` or `"deny"` per `tool_call_id`. Multiple pending calls can be resolved individually or as a batch.
- **Python SDK helpers**: `RunContext` + `DeferredToolCallsException` (requires `mistralai` v2.4+ with the `mcp` extra) handle the plumbing; `dc.confirm()` / `dc.reject("reason")` resume the loop.
- **Stateless serialization**: `deferred.to_dict()` / `deferred.executed_results` let web APIs serialize the deferred state and resume across separate requests.

### API functions & parameters

**Configuration** (in the `tools` array):
```json
[
  {
    "type": "connector",
    "connector_id": "gmail",
    "tool_configuration": { "requires_confirmation": ["gmail_search"] }
  },
  {
    "type": "web_search_premium",
    "tool_configuration": { "requires_confirmation": ["web_search", "news_search"] }
  }
]
```

**REST API flow**:
1. `POST /v1/conversations` with the tools; the response contains:
```json
{
  "conversation_id": "conv_...",
  "outputs": [{
    "type": "function.call",
    "tool_call_id": "WJfo42Ow3",
    "name": "gmail_search",
    "arguments": "{\"limit\": 1}",
    "confirmation_status": "pending"
  }]
}
```
2. Approve or deny by appending to that conversation:
```bash
curl -X POST "https://api.mistral.ai/v1/conversations/$CONVERSATION_ID" \
  -H "Authorization: Bearer $MISTRAL_API_KEY" \
  -H "Content-Type: application/json" \
  -d '{
    "tool_confirmations": [
      {"tool_call_id": "'$TOOL_CALL_ID'", "confirmation": "allow"}
    ]
  }'
```
`confirmation` accepts `"allow"` (runs the tool, returns the model response) or `"deny"` (rejects; the model handles the rejection gracefully).

**Python SDK flow** (local functions):
```python
async with RunContext(model="mistral-large-latest") as run_ctx:
    run_ctx.conversation_id = conversation_id
    run_ctx.register_func(get_weather, requires_confirmation=False)
    run_ctx.register_func(book_flight, requires_confirmation=True)
    try:
        result = await client.beta.conversations.run_async(
            run_ctx=run_ctx, inputs=pending_inputs, instructions="..."
        )
        break
    except DeferredToolCallsException as deferred:
        conversation_id = deferred.conversation_id
        pending_inputs = []
        for dc in deferred.deferred_calls:
            approved = input("Approve? (y/n): ").strip().lower() == "y"
            pending_inputs.append(dc.confirm() if approved else dc.reject("Denied by user"))
```

`run_async` raises `DeferredToolCallsException` whenever a tool requires confirmation (works identically for Connector tools via the `tools` array).

**Stateless resume**: serialize with `deferred.to_dict()` and reconstruct later; always prepend `deferred.executed_results` (results from tools that ran before deferral) when resuming so the model has full context.

### Notes
- The confirmation flow applies to **all tool types** — Connectors, built-in tools, and local Python functions (the latter only via the Python SDK).
- Batch approvals/denials: multiple `tool_confirmations` entries can be sent in a single request.
- `DeferredToolCallsException` requires `mistralai` v2.4+ and `pip install mistralai[mcp]`; older versions must use the REST flow.
- Best for sensitive actions like sending emails or modifying data.

---

## 11. Connectors Debugger

**Summary** — A Studio UI tool to validate an MCP Connector server before production use — checking reachability, authentication, tool discovery, and MCP compatibility — without persisting credentials.

> **Status**: The Connectors Debugger is in **Public Preview**; validation coverage and error details may change.

### Main concepts
- **Pre-flight validation**: Run a diagnostic against any MCP server URL to confirm it is compatible with Studio before registering it as a Connector.
- **Non-persistent credentials**: Headers or OAuth2 credentials entered in the Debugger last only for the current session and are not stored.
- **Step-based report**: The diagnostic produces a per-step report (`transport_detection`, tool discovery, etc.) with `Likely cause`, `Suggested fix`, `Raw response`, `Copy as curl`, and a JSON report for failed steps.
- **OAuth redirect URI**: For OAuth2 servers, configure `https://console.mistral.ai/build/connectors/debugger/oauth-callback` in your OAuth application.

### How it works
1. Open Studio → `Connectors` → `Debugger` (or go to `https://console.mistral.ai/build/connectors/debugger`).
2. Enter the MCP server URL.
3. (Optional) Click the settings icon next to `Run diagnostic` to add credentials:
   - `Custom header` (e.g. `Authorization: Bearer <token>`), or
   - `OAuth 2.0` (`Client ID`, `Client Secret`).
4. Click `Run diagnostic`.

**Successful report** includes:
- `MCP server ready`, MCP protocol version (e.g. `2025-11-25`), `View official docs`.
- `Tools`/`Prompts`/`Resources` counts discovered, `Duration`, `Checks` (e.g. `3/3`), `Instructions`.
- Per-tool inspection: name, description, input schema (e.g. `echo` → `Echo back the provided message.` with `{ "type": "object", "properties": { "message": { "description": "Text to echo", "type": "string" } }, "required": ["message"] }`).

**Failed-step report example** (`not_mcp_server`):
```json
{
  "step": "transport_detection",
  "status": "error",
  "duration": 259,
  "data": {
    "attempt": {
      "request": { "method": "POST", "url": "https://chaos-mcp.example.com/" },
      "response": { "status": 200, "headers": { "content-type": "text/html; charset=utf-8" }, "body": "<!DOCTYPE html>..." }
    }
  },
  "error": { "type": "not_mcp_server" }
}
```

### Notes
- Only test Connector servers you trust — the Debugger sends configured headers/credentials to the provided server URL.
- The Debugger covers common failure categories: HTML/non-JSON-RPC responses, malformed JSON, wrong content type/gzip errors, timeouts/500/429, 401 auth failures, discovery/token-exchange failures, post-consent token rejection, CORS errors, session resume/reinit loops, MCP initialize (capabilities/protocol version/serverInfo), tool discovery, and tool-call hangs/payloads/`isError`/malformed content.
- After a successful test, the Connector creation flow itself is **out of scope** for the Debugger — proceed to the management API (§7) to register it.