# OpenAI API Analysis — Agent Capabilities (Agents SDK)

> **Docs:** `https://developers.openai.com/api/docs/guides/agents` | **SDK repos:** [TypeScript](https://github.com/openai/openai-agents-js) · [Python](https://github.com/openai/openai-agents-python)
> **Packages:** `@openai/agents` (npm) · `openai-agents` (pip) | **Auth:** Bearer token (`OPENAI_API_KEY`)
> **Description:** OpenAI exposes agent capabilities through the **Agents SDK** — a code-first framework available in TypeScript and Python that manages the agent loop (model calls, tool execution, handoffs, approvals) on behalf of the developer. It sits on top of the **Responses API** (`/v1/responses`), which can also be used directly when the developer wants to own the loop. The SDK organizes agent work into twelve capability areas: agent definitions, model/provider selection, the runtime loop, sandboxed execution, multi-agent orchestration, guardrails & human review, results & state, integrations & observability, evaluation, and voice agents. A hosted visual builder (**Agent Builder**, legacy) and a browser UI component (**ChatKit**) round out the platform.

---

## Table of Contents

1. [Platform Overview & Core Concepts](#1-platform-overview--core-concepts)
2. [Agents SDK vs Responses API](#2-agents-sdk-vs-responses-api)
3. [Agent Definitions](#3-agent-definitions)
4. [Models & Providers](#4-models--providers)
5. [Running Agents — The Agent Loop, State & Streaming](#5-running-agents--the-agent-loop-state--streaming)
6. [Sandbox Agents — Container-Based Execution](#6-sandbox-agents--container-based-execution)
7. [Orchestration & Handoffs](#7-orchestration--handoffs)
8. [Guardrails & Human Review](#8-guardrails--human-review)
9. [Results & State](#9-results--state)
10. [Integrations & Observability (MCP, Tracing, Hooks)](#10-integrations--observability-mcp-tracing-hooks)
11. [Evaluating Agent Workflows](#11-evaluating-agent-workflows)
12. [Voice Agents](#12-voice-agents)
13. [Capability Summary & Cross-Reference](#13-capability-summary--cross-reference)

---

## 1. Platform Overview & Core Concepts

### Main Concepts

OpenAI's agent platform is organized around these core abstractions:

- **Agent** — The core unit of an SDK-based workflow. Packages a model, instructions, and optional runtime behavior (tools, guardrails, MCP servers, handoffs, structured output). Defined in code as a reusable object.
- **Run** — One application-level turn executed by the SDK runner. The runner loops (model call → inspect output → execute tools / switch agents / return result) until a stopping point is reached.
- **Tool** — A capability the agent can call directly. Three categories: (1) **function tools** (typed application code), (2) **hosted OpenAI tools** (web search, file search, code interpreter, shell, computer use, image generation), (3) **MCP-backed tools** (local or hosted Model Context Protocol servers).
- **Handoff** — A delegation mechanism where one agent transfers conversation ownership to another specialist agent. The receiving agent owns subsequent replies.
- **Agents-as-tools** — An alternative to handoffs where a manager agent calls a specialist as a bounded tool, keeping ownership of the final reply.
- **Guardrail** — An automatic validation check on input, output, or tool behavior. Can block the workflow or run in parallel with the main agent.
- **Approval** — A human-in-the-loop pause point. The run interrupts, stores state, and resumes after a person or policy approves/rejects a pending action.
- **Session** — SDK-managed conversation history that persists across runs. An alternative to manually replaying `result.history`.
- **Run State** — A resumable snapshot of a paused run (model items, tool state, approvals, active agent position). Used to resume after interruptions.
- **Trace** — A structured record of a run emitted by the SDK (model calls, tool calls, handoffs, guardrails, custom spans). Viewable in the Traces dashboard.
- **Sandbox** — An isolated, container-based execution environment with a filesystem, shell, packages, mounts, ports, and snapshots. Used when the agent needs to manipulate files or run commands.
- **Manifest** — The fresh-session workspace contract for a sandbox agent. Describes files, directories, mounts, environment, users, and groups that should exist when a new sandbox session starts.
- **Run Context** — Local application state (user info, DB clients, loggers) passed into a run and available to tools, but **not** sent to the model. Keeps conversation history (model-visible) separate from runtime context (code-only).

### Agent Capabilities Map

| Capability | Description | Guide |
|------------|-------------|-------|
| **Agent definitions** | Configure a single specialist (model, instructions, tools, output type, guardrails) | [define-agents](https://developers.openai.com/api/docs/guides/agents/define-agents) |
| **Models & providers** | Choose models, defaults, transport strategy (OpenAI, WebSocket, third-party) | [models](https://developers.openai.com/api/docs/guides/agents/models) |
| **Running agents** | The agent loop, conversation state strategies, streaming, pause/failure handling | [running-agents](https://developers.openai.com/api/docs/guides/agents/running-agents) |
| **Sandbox agents** | Container-based execution with files, commands, packages, snapshots, mounts | [sandboxes](https://developers.openai.com/api/docs/guides/agents/sandboxes) |
| **Orchestration** | Multi-agent patterns: handoffs (delegated ownership) and agents-as-tools (manager-style) | [orchestration](https://developers.openai.com/api/docs/guides/agents/orchestration) |
| **Guardrails & approvals** | Input/output/tool validation + human-in-the-loop approval pauses | [guardrails-approvals](https://developers.openai.com/api/docs/guides/agents/guardrails-approvals) |
| **Results & state** | Final output, history, last agent, response chaining, interruptions, resumable state | [results](https://developers.openai.com/api/docs/guides/agents/results) |
| **Integrations & observability** | MCP server wiring (hosted/local), built-in tracing, lifecycle hooks | [integrations-observability](https://developers.openai.com/api/docs/guides/agents/integrations-observability) |
| **Evaluation** | Trace grading, datasets, eval runs, graders for agent quality measurement | [agent-evals](https://developers.openai.com/api/docs/guides/agent-evals) |
| **Voice agents** | Speech-to-speech (Realtime API) and chained voice pipelines (STT → text agent → TTS) | [voice-agents](https://developers.openai.com/api/docs/guides/voice-agents) |

### Platform Architecture

```
Application code (TypeScript / Python)
        │
        ▼
   Agent definition  ── model, instructions, tools[], handoffs[], outputType, guardrails, mcpServers
        │
        ▼
   Runner.run(agent, input, { context, session, runConfig })
        │
        ▼
   ┌──────────────── Agent Loop ────────────────┐
   │  1. Call model with prepared input          │
   │  2. Inspect model output                    │
   │  3. If tool calls  → execute tools → loop   │
   │  4. If handoff     → switch agent → loop    │
   │  5. If final answer → return result         │
   │                                             │
   │  Guardrails run at input/output/tool points │
   │  Approvals pause loop → store state → resume│
   └─────────────────────────────────────────────┘
        │
        ▼
   RunResult
     ├── finalOutput / final_output
     ├── history / to_input_list()
     ├── lastAgent / last_agent
     ├── lastResponseId / last_response_id
     ├── interruptions + state (if paused)
     └── trace (emitted to Traces dashboard)

  State strategies:
    result.history     → app-owned replay
    session            → SDK-managed persistent history
    conversationId     → OpenAI Conversations API (server-managed)
    previousResponseId → cheapest response chaining
```

---

## 2. Agents SDK vs Responses API

OpenAI offers two paths for building agentic applications. The **Responses API** is the lower-level REST primitive; the **Agents SDK** is the higher-level framework that manages the loop for you.

### When to choose each

| | Responses API | Agents SDK |
|---|---|---|
| **Best for** | Custom model-powered features and workflows | Bounded conversational/transactional workflows with defined tools and recurring orchestration |
| **Core abstraction** | A model response | An agent run |
| **Loop ownership** | You manage custom loops and branching | The SDK provides the agent loop and lifecycle |
| **Tools** | Platform tools, function calling, remote MCP | Platform tools + function tool wrappers + local MCP + agents-as-tools |
| **Multi-agent** | Build routing/delegation yourself | Built-in handoffs and agents-as-tools |
| **State** | Manual history, `previous_response_id`, Conversations API | Same options + SDK sessions + resumable run state |
| **Safety** | Tool-specific approvals; you build broader controls | Input/output/tool guardrails + resumable approval flows |
| **Debugging** | Response objects and API logs | Built-in traces across model calls, tools, agents, guardrails, handoffs |

### Decision rule

- **Responses API** when you want direct control over model interactions, output items, tools, state, and orchestration.
- **Agents SDK** when you want the SDK to manage the agent loop and recurring orchestration (repeated tool calls, branching), or when you need built-in sessions, tracing, guardrails, or resumable approvals.

---

## 3. Agent Definitions

**Docs:** [define-agents](https://developers.openai.com/api/docs/guides/agents/define-agents)

### Main Concepts

An agent is the core unit of an SDK workflow. It packages everything a single specialist needs:

- **Static vs dynamic instructions** — Start with a static string. When guidance depends on the current user/tenant/runtime, switch to a dynamic instructions callback.
- **Structured output** — Use `outputType` (TS) / `output_type` (Py) when downstream code needs typed data (Zod schema / Pydantic model) instead of free-form prose.
- **Local context vs model context** — Run context (user info, DB clients, loggers) is available to tools but never sent to the model. Conversation history is what the model sees.
- **Splitting agents** — Split when a specialist needs different tools, MCP surfaces, approval policies, models, or output styles, or when you want explicit routing in traces.

### API Functions & Parameters

#### `Agent` constructor

| Property | Type (TS / Py) | Description |
|----------|----------------|-------------|
| `name` | `string` | Human-readable identity; appears in traces and tool/handoff surfaces. |
| `instructions` | `string` \| `async (ctx, agent) => string` | The job, constraints, and style for this agent. Can be a dynamic callback. |
| `prompt` | stored prompt config | References a stored Responses API prompt configuration instead of embedding the full system prompt in code. |
| `model` | `string` | Model ID (e.g. `"gpt-5.6"`). Explicit per-agent choice. |
| `modelSettings` / `model_settings` | object | Tuning: reasoning effort, verbosity, tool behavior. |
| `tools` | `Tool[]` | Capabilities the agent can call directly (function tools + hosted tools). |
| `handoffDescription` / `handoff_description` | `string` | Short, concrete hint for routing agents to know when to delegate here. |
| `handoffs` | `Agent[]` \| `Handoff[]` | Agents this one can delegate to via handoff. |
| `outputType` / `output_type` | `ZodSchema` / `Pydantic BaseModel` | Structured output schema. Replaces free-form text with typed data. |
| `mcpServers` / `mcp_servers` | `MCPServer[]` | Local or hosted MCP servers attached to this agent. |
| Guardrails | config | Input/output guardrails and approval policies (see §8). |

#### Function tools

| Decorator / Helper | Language | Description |
|--------------------|----------|-------------|
| `tool({ name, description, parameters, execute, needsApproval })` | TypeScript | Defines a typed function tool. `parameters` is a Zod schema. |
| `@function_tool` | Python | Decorator that wraps a Python function into a tool. Type hints generate the schema. |

| `tool()` parameter | Type | Description |
|--------------------|------|-------------|
| `name` | `string` | Tool name (model-visible). |
| `description` | `string` | What the tool does; the model uses this to decide when to call it. |
| `parameters` | `ZodSchema` | Argument schema (TS). In Python, derived from type hints. |
| `execute` | `async (args, runContext?) => string` | The tool implementation. Receives typed args and optional run context. |
| `needsApproval` | `boolean` | If `true`, the run pauses before executing this tool (see §8). |

#### Run context

| Mechanism | Description |
|-----------|-------------|
| `RunContext<T>` / `RunContextWrapper[T]` (TS) | Generic wrapper carrying typed local context into tool `execute` calls. |
| `context=UserInfo(...)` on `Runner.run()` (Py) | Passed as a keyword arg; tools receive `wrapper: RunContextWrapper[UserInfo]`. |

### Example (Python — structured output)

```python
from pydantic import BaseModel
from agents import Agent, Runner

class CalendarEvent(BaseModel):
    name: str
    date: str
    participants: list[str]

agent = Agent(
    name="Calendar extractor",
    instructions="Extract calendar events from text.",
    output_type=CalendarEvent,
)

result = await Runner.run(agent, "Dinner with Priya and Sam on Friday.")
print(result.final_output)  # CalendarEvent(name="Dinner", date="Friday", ...)
```

### Example (TypeScript — local context in tools)

```typescript
const fetchUserAge = tool({
  name: "fetch_user_age",
  description: "Return the age of the current user.",
  parameters: z.object({}),
  async execute(_args, runContext?: RunContext<UserInfo>) {
    return `User ${runContext?.context.name} is 47 years old`;
  },
});

const agent = new Agent<UserInfo>({ name: "Assistant", tools: [fetchUserAge] });
const result = await run(agent, "What is the age of the user?", {
  context: { name: "John", uid: 123 },
});
```

---

## 4. Models & Providers

**Docs:** [models](https://developers.openai.com/api/docs/guides/agents/models)

### Main Concepts

Every SDK run resolves a model and a transport. The guidance is to keep this straightforward: choose models explicitly, use the standard OpenAI path by default, and reach for provider/transport overrides only when needed.

- **Explicit model selection** — Prefer explicit `model` on each agent over relying on the SDK's runtime default.
- **Model selection layers** — Per-agent `model` (highest priority) → run-level default (`RunConfig.model`) → process-wide fallback (`OPENAI_DEFAULT_MODEL`).
- **Transport options** — Standard HTTP (default), Responses WebSocket transport (for many repeated round trips over a socket), and provider/adapter surfaces for non-OpenAI models.
- **WebSocket vs voice** — The Responses WebSocket transport is for text-and-tools agent loops. Live audio sessions (WebRTC/WebSocket) are a separate path for voice agents (see §12).

### API Functions & Parameters

#### Model selection strategies

| Strategy | Mechanism | Use when |
|----------|-----------|----------|
| Per-agent model | `Agent({ model: "gpt-5.6-terra" })` | A specialist consistently needs a different quality/latency/cost profile. |
| Run-level default | `RunConfig(model="gpt-5.6")` / `new Runner({ model: "gpt-5.6" })` | One workflow should override several agents at once. |
| Process-wide fallback | `OPENAI_DEFAULT_MODEL` env var | Agents that omit `model` should still resolve predictably. |
| Mixed models | Different `model` per agent | A fast triage agent and a slower deep specialist coexist. |

#### Provider & transport configuration

| Need | Start with |
|------|------------|
| Standard SDK runs on OpenAI | Default OpenAI provider path |
| Many repeated Responses round trips over a socket | Responses WebSocket transport |
| Non-OpenAI models or mixed-provider stack | Provider/adapter surface (language-specific SDK docs) |

#### Model settings

| Setting surface | Description |
|-----------------|-------------|
| `modelSettings` / `model_settings` | Tuning: reasoning effort, verbosity, tool behavior. |
| `prompt` | Stored prompt configuration from the Responses API, instead of embedding the full system prompt in code. |

### Example (Python — per-agent and run-level models)

```python
from agents import Agent, RunConfig, Runner

fast_agent = Agent(name="Fast support agent", model="gpt-5.6-terra")
general_agent = Agent(name="General support agent")  # uses run-level default

await Runner.run(fast_agent, "Summarize ticket 123.")
result = await Runner.run(
    general_agent, "Investigate the billing issue.",
    run_config=RunConfig(model="gpt-5.6"),
)
```

---

## 5. Running Agents — The Agent Loop, State & Streaming

**Docs:** [running-agents](https://developers.openai.com/api/docs/guides/agents/running-agents)

### Main Concepts

Defining an agent is only setup. The runtime questions are: what does a single run do, how does the next turn continue, and how does the workflow behave when it pauses?

- **One run = one application-level turn** — The runner loops until it reaches a real stopping point.
- **The agent loop** — (1) Call the current agent's model with prepared input. (2) Inspect output. (3) If tool calls, execute and continue. (4) If handoff, switch agents and continue. (5) If final answer, return result.
- **Four conversation state strategies** — `result.history` (app-owned), `session` (SDK-managed), `conversationId` (OpenAI Conversations API), `previousResponseId` (cheapest chaining).
- **Streaming** — Same loop and state strategies; the only difference is consuming events while the run is still happening.
- **Pauses vs failures** — Failures (max-turn limits, guardrail exceptions, tool errors) vs expected pauses (human approval requests). Pauses resume from the same `state`; they are not new turns.
- **Approvals are paused runs** — Treating approvals as paused runs (not new turns) keeps turn counts, history, and server-managed continuation IDs consistent.

### API Functions & Parameters

#### `Runner.run()` / `run()`

| Parameter | Type | Description |
|-----------|------|-------------|
| `agent` | `Agent` | The starting agent for this run. |
| `input` | `string` \| `Item[]` | The user's input for this turn. |
| `context` | `T` (generic) | Local run context (available to tools, not sent to model). |
| `session` | `Session` | SDK-managed conversation history. E.g. `MemorySession`. |
| `runConfig` / `run_config` | `RunConfig` | Run-level config: model override, max turns, sandbox config, tracing settings. |
| `sandbox` | `SandboxRunConfig` | Sandbox session + client (see §6). |
| `conversationId` / `conversation_id` | `string` | OpenAI Conversations API ID for server-managed state. |
| `previousResponseId` / `previous_response_id` | `string` | Cheapest response-to-response chaining. |

#### Conversation state strategies

| Strategy | Where state lives | Best for | What you pass next turn |
|----------|-------------------|----------|------------------------|
| `result.history` | Your application | Small chat loops, maximum control | The replay-ready history |
| `session` | Your storage + SDK | Persistent chat state, resumable runs | The same session object |
| `conversationId` | OpenAI Conversations API | Shared server-managed state across workers/services | The same conversation ID + only the new turn |
| `previousResponseId` | OpenAI Responses API | Lightest server-managed continuation | The last response ID + only the new turn |

#### Streaming

| Aspect | Description |
|--------|-------------|
| Mechanism | Same agent loop + same state strategies; consume events as they arrive. |
| Settled state | Wait for the stream to finish before treating the run as settled. |
| Approval during stream | If paused, resolve `interruptions` and resume from `state` (not a fresh turn). |
| Cancellation | Resume the unfinished turn from `state` if you want it to continue later. |

#### Pause & failure handling

| Outcome class | Examples | Handling |
|---------------|----------|----------|
| Runtime/validation failures | Max-turn limits, guardrail exceptions, tool errors | Treat as errors; inspect diagnostics. |
| Expected pauses | Human approval requests | Resume from `state` after resolving `interruptions`. |

### Example (TypeScript — session-based multi-turn)

```typescript
import { Agent, MemorySession, run } from "@openai/agents";

const agent = new Agent({ name: "Tour guide", instructions: "Answer with compact travel facts." });
const session = new MemorySession();

const firstTurn = await run(agent, "What city is the Golden Gate Bridge in?", { session });
console.log(firstTurn.finalOutput);

const secondTurn = await run(agent, "What state is it in?", { session });
console.log(secondTurn.finalOutput);
```

---

## 6. Sandbox Agents — Container-Based Execution

**Docs:** [sandboxes](https://developers.openai.com/api/docs/guides/agents/sandboxes) · **Status:** Beta (TS + Python)

### Main Concepts

A sandbox gives an agent an isolated, Unix-like execution environment with a filesystem, shell, installed packages, mounted data, exposed ports, snapshots, and controlled access to external systems. Use sandboxes when the agent needs to manipulate files, run commands, mount a data room, produce artifacts, expose a service, or continue stateful work later.

- **Harness vs compute boundary** — The **harness** is the control plane (agent loop, model calls, tool routing, handoffs, approvals, tracing, run state). **Compute** is the sandbox execution plane (files, commands, packages, mounts, ports, snapshots). Orchestration stays in the trusted runtime; the sandbox is an execution surface.
- **Manifest** — The fresh-session workspace contract. Describes files, directories, mounts, environment, users, and groups that exist when a new sandbox session starts.
- **Mounts** — External storage (S3, GCS, R2, Azure Blob, Box) or local directories materialized into the sandbox. Mounts are ephemeral: snapshot/persistence flows skip mounted remote storage.
- **Capabilities** — `Filesystem()` (read/write workspace files), `Shell()` (list files, run commands, search), `Compaction()` (manage long-running context), `Memory()` (store reusable workflow lessons).
- **Three state surfaces** — `RunState` (harness-side: model items, tool state, approvals, active agent), Session state (serialized sandbox session for reconnection), `snapshot` (saved workspace contents to seed a fresh sandbox).

### API Functions & Parameters

#### `Manifest` entries

| Entry type | Use it for |
|------------|------------|
| `File`, `Dir` | Small synthetic inputs, helper files, or output directories. |
| Local file or directory | Host files/directories to materialize into the sandbox. |
| `GitRepo` | A repository to fetch into the workspace. |
| `LocalDir` | A local host directory to mount. |
| `S3Mount`, `GCSMount`, `R2Mount`, `AzureBlobMount`, `BoxMount`, `S3FilesMount` | External storage to make available inside the sandbox. |
| `environment` | Environment variables the sandbox needs at startup. |
| `users`, `groups` | Sandbox-local OS accounts/groups for providers that support account provisioning. |

#### `SandboxAgent` & run config

| Parameter | Type | Description |
|-----------|------|-------------|
| `sandbox.client` | `SandboxClient` | The sandbox provider client (e.g. `UnixLocalSandboxClient`, `DockerSandboxClient`). |
| `sandbox.session` | `SandboxSession` | An existing sandbox session to resume into. |
| `manifest` | `Manifest` | The workspace contract for a fresh session. |
| `maxTurns` | `number` | Maximum loop iterations for the run. |

#### Sandbox client methods (resume flow)

| Method | Description |
|--------|-------------|
| `serializeSessionState(state)` | Freeze sandbox session state for storage. |
| `deserializeSessionState(frozen)` | Restore frozen session state. |
| `resume(restoredState)` | Create a new session that resumes from restored state. |
| `snapshot()` | Save workspace contents for seeding a fresh sandbox later. |

### Example (TypeScript — manifest + local sandbox)

```typescript
import { run } from "@openai/agents";
import { Manifest, SandboxAgent, file, shell } from "@openai/agents/sandbox";
import { UnixLocalSandboxClient } from "@openai/agents/sandbox/local";

const manifest = new Manifest({
  entries: {
    "account_brief.md": file({ content: "# Northwind Health\n..." }),
    "output/": { type: "dir" },
  },
});

const agent = new SandboxAgent({
  name: "Analyst",
  instructions: "Read account_brief.md and write a summary to output/summary.md.",
  tools: [shell(), /* filesystem() */],
});

const result = await run(agent, "Summarize the account.", {
  sandbox: { client: new UnixLocalSandboxClient(), manifest },
});
```

### Example (Python — Docker sandbox)

```python
result = await Runner.run(agent, "Inspect the workspace.", sandbox={
    "client": DockerSandboxClient(image="node:22-bookworm-slim"),
})
```

### Sandbox examples (from docs)

| Example | Description |
|---------|-------------|
| Data room Q&A | Answer questions over a mounted data room. |
| Data room table extraction | Extract a table from a mounted data room. |
| Repository code review | Clone a repo, inspect it, produce code review artifacts. |
| Vision website clone | Clone a website using the Vision API and screenshot feedback. |
| Sandbox resume | Resume work in a pre-existing sandbox. |

---

## 7. Orchestration & Handoffs

**Docs:** [orchestration](https://developers.openai.com/api/docs/guides/agents/orchestration)

### Main Concepts

Multi-agent workflows are useful when specialists should own different parts of the job. The first design choice is deciding **who owns the final user-facing answer** at each branch.

| Pattern | Use it when | What happens |
|---------|-------------|--------------|
| **Handoffs** | A specialist should take over the conversation for that branch | Control moves to the specialist agent; it owns subsequent replies. |
| **Agents as tools** | A manager should stay in control and call specialists as bounded capabilities | The manager keeps ownership of the reply; specialists return narrow results. |

- **Handoffs** — The clearest fit when a specialist should own the next response rather than merely helping behind the scenes. The `handoffs` array on an agent lists the specialists it can delegate to.
- **Agents as tools** — Use `agent.asTool()` (TS) / `agent.as_tool()` (Py) when the main agent should stay responsible for the final answer and call specialists as helpers. The specialist runs as a tool, returns a narrow result, and the orchestrator decides what to do next.
- **`handoff()` wrapper** — Advanced handoffs can carry structured metadata or filtered history. The `handoff()` function wraps an agent with additional config.
- **Routing surface** — Keep each specialist's job narrow, `handoffDescription` short and concrete, and split only when the next branch truly needs different instructions, tools, or policy.

### API Functions & Parameters

#### Handoffs

| API | Language | Description |
|-----|----------|-------------|
| `Agent({ handoffs: [agentA, agentB] })` | TypeScript | Lists agents this one can hand off to. |
| `Agent(handoffs=[agent_a, agent_b])` | Python | Same. |
| `handoff(agent, metadata=..., ...)` | Both | Wraps an agent with structured metadata or filtered history for advanced routing. |

#### Agents as tools

| API | Language | Description |
|-----|----------|-------------|
| `agent.asTool({ name, description })` | TypeScript | Converts an agent into a callable tool for a manager agent. |
| `agent.as_tool(name=..., description=...)` | Python | Same. The specialist runs, returns a result, and the manager stays in control. |

### Example (Python — handoffs)

```python
from agents import Agent, handoff

billing_agent = Agent(name="Billing agent")
refund_agent = Agent(name="Refund agent")

triage_agent = Agent(
    name="Triage agent",
    handoffs=[billing_agent, handoff(refund_agent)],
)
```

### Example (TypeScript — agents as tools)

```typescript
const specialist = new Agent({ name: "Researcher", instructions: "..." });
const manager = new Agent({
  name: "Manager",
  instructions: "Delegate research tasks to the researcher tool.",
  tools: [specialist.asTool({ name: "research", description: "Research a topic." })],
});
```

### Ownership rules

| Pattern | Ownership | When to use |
|---------|-----------|-------------|
| `agent.as_tool(...)` | Manager stays final-response owner | Main orchestrator should remain in control. |
| `handoffs=[...]` | Specialist takes over, owns final response | Specialist should own the next response. |

---

## 8. Guardrails & Human Review

**Docs:** [guardrails-approvals](https://developers.openai.com/api/docs/guides/agents/guardrails-approvals)

### Main Concepts

Use guardrails for automatic checks and human review for approval decisions. Together, they define when a run should continue, pause, or stop.

- **Input guardrails** — Validate user requests before the main model runs. By default run **in parallel** with the main agent for lower latency; set `run_in_parallel=False` for blocking validation that must complete before tool use.
- **Output guardrails** — Validate or redact the final output before it leaves the system.
- **Tool guardrails** — Check arguments or results around a specific function tool call. Put validation next to the tool that creates the side effect.
- **Human-in-the-loop approvals** — Pause the run before side effects (cancellations, edits, shell commands, sensitive MCP actions). The run interrupts, stores state, and resumes after approval/rejection.
- **Agent-level guardrail scope** — Input guardrails run only for the first agent in the chain. Output guardrails run only for the agent that produces the final output. Tool guardrails run on the function tools they're attached to.
- **Streaming uses the same state model** — If a streamed run pauses, wait for it to settle, inspect `interruptions`, resolve approvals, and resume from the same `state`.

### API Functions & Parameters

#### Guardrail types

| Use case | Start with | Mechanism |
|----------|------------|-----------|
| Block disallowed user requests before the main model runs | Input guardrails | Fast validation step before the expensive/side-effecting part. |
| Validate or redact the final output before it leaves | Output guardrails | Check the agent's final output. |
| Check arguments or results around a function tool call | Tool guardrails | Per-tool validation. |
| Pause before side effects (cancellations, edits, shell, sensitive MCP) | Human-in-the-loop approvals | `needsApproval: true` on tools + `interruptions`/`state` resume. |

#### Approval flow

| Step | API | Description |
|------|-----|-------------|
| 1. Mark tool as needing approval | `tool({ ..., needsApproval: true })` | The run pauses before executing this tool. |
| 2. Detect interruption | `result.interruptions` | Array of pending tool calls needing decisions. |
| 3. Save state | `result.state` (TS) / `result.to_state()` (Py) | Resumable snapshot of the paused run. |
| 4. Approve/reject | `state.approve(interruption)` / `state.reject(interruption)` | Resolve each pending item. |
| 5. Resume | `run(agent, state)` | Pass the state back to resume the same run (not a new turn). |

#### Input guardrail parameters

| Parameter | Type | Description |
|-----------|------|-------------|
| `run_in_parallel` | `boolean` | If `True` (default), runs alongside the main agent. If `False`, blocks before the agent can call tools. |

### Example (TypeScript — approval flow)

```typescript
const cancelOrder = tool({
  name: "cancel_order",
  description: "Cancel a customer order.",
  parameters: z.object({ orderId: z.number() }),
  needsApproval: true,
  async execute({ orderId }) { return `Cancelled order ${orderId}`; },
});

const agent = new Agent({ name: "Support agent", tools: [cancelOrder] });

let result = await run(agent, "Cancel order 123.");
if (result.interruptions?.length) {
  const state = result.state;
  for (const interruption of result.interruptions) {
    state.approve(interruption);
  }
  result = await run(agent, state);
}
console.log(result.finalOutput);
```

---

## 9. Results & State

**Docs:** [results](https://developers.openai.com/api/docs/guides/agents/results)

### Main Concepts

When you run an agent, the result is more than the final answer. It's also the handoff boundary, the next-turn continuation surface, and the resumable snapshot when a run pauses for review.

- **Five high-level result surfaces** — `finalOutput`, `history`, `lastAgent`, `lastResponseId`, `interruptions` + `state`.
- **Paused runs** — `finalOutput` can be empty because the run hasn't finished. `interruptions` tells you which pending tool calls need a decision. `state` is the saved snapshot to resume from.
- **Richer diagnostics** — The SDK also exposes item-level tool/handoff records, raw model responses, guardrail results, and usage details for applications that need more.

### API Functions & Parameters

#### Result properties

| If you need | Use (TypeScript) | Use (Python) |
|-------------|------------------|--------------|
| The final answer to show the user | `result.finalOutput` | `result.final_output` |
| Local replay-ready history | `result.history` | `result.to_input_list()` |
| The specialist that should own the next turn | `result.lastAgent` | `result.last_agent` |
| OpenAI-managed response chaining | `result.lastResponseId` | `result.last_response_id` |
| Pending approvals + resumable snapshot | `result.interruptions` + `result.state` | `result.interruptions` + `result.to_state()` |

#### What to carry into the next turn

| Goal | Carry forward |
|------|---------------|
| Keep the full conversation in your app | `result.history` |
| Let the SDK manage history | The same `session` object |
| Let OpenAI manage continuation | `result.lastResponseId` or `conversationId` |
| Continue after handoff | `result.lastAgent` (so that specialist stays in control) |
| Resume after pause | `result.state` (after resolving `interruptions`) |

#### Richer surfaces (advanced)

| Surface | Description |
|---------|-------------|
| Item-level tool/handoff records | Detailed records of each tool call and handoff within the run. |
| Raw model responses | The underlying Responses API output objects. |
| Guardrail results | Outcomes of input/output/tool guardrail checks. |
| Usage details | Token usage and cost information. |

---

## 10. Integrations & Observability (MCP, Tracing, Hooks)

**Docs:** [integrations-observability](https://developers.openai.com/api/docs/guides/agents/integrations-observability)

### Main Concepts

This page covers SDK-specific MCP wiring and the observability loop. Tool capability semantics live in the [Using tools](https://developers.openai.com/api/docs/guides/tools) guide.

- **MCP (Model Context Protocol)** — Two integration paths: (1) **hosted MCP tools** for public remote servers that fit the platform trust model, (2) **local/private MCP servers** managed by your runtime over stdio or streamable HTTP.
- **Tracing** — Built into the Agents SDK and enabled by default in the normal server-side SDK path. Every run emits a structured record viewable in the Traces dashboard.
- **Lifecycle hooks** — `RunHooks` and `AgentHooks` are mainly for lifecycle side effects (logging, tracing). For blocking, approval, or execution-shaping logic, use guardrails, approvals, or filters instead.

### API Functions & Parameters

#### MCP integration

| Need | Start with | Why |
|------|------------|-----|
| Public, remotely hosted MCP tools | Hosted MCP tools in the SDK | The model calls the remote MCP server through the hosted surface. |
| Local or private MCP servers | SDK-managed MCP servers over stdio or streamable HTTP | Your runtime owns the connection, approvals, and network boundaries. |

| MCP class | Transport | Description |
|-----------|-----------|-------------|
| `MCPServerStdio` | stdio (subprocess) | Launches a local MCP server as a subprocess (e.g. `npx @modelcontextprotocol/server-filesystem`). |
| `MCPServerStreamableHttp` / `MCPServerStreamableHttpParams` | Streamable HTTP | Connects to an HTTPS MCP endpoint (e.g. Databricks managed MCP). |

| MCP method | Description |
|------------|-------------|
| `server.connect()` | Establish connection to the MCP server. |
| `server` in `Agent({ mcpServers: [server] })` | Attach the server to an agent; its tools become available. |

#### Tracing

| Trace component | Description |
|-----------------|-------------|
| Overall run/workflow trace | The top-level trace for one run. |
| Model call spans | Each model call within the run. |
| Tool call spans | Tool calls and their outputs. |
| Handoff spans | Agent-to-agent handoffs. |
| Guardrail spans | Guardrail evaluations. |
| Custom spans | Spans you wrap around workflow segments. |
| `trace(name, metadata=...)` | Python helper to create a named workflow trace with metadata. |
| `gen_trace_id()` | Python helper to generate a trace ID. |

#### Lifecycle hooks

| Hook type | Use for |
|-----------|---------|
| `RunHooks` | Run-level lifecycle side effects (logging, tracing, audit events). |
| `AgentHooks` | Agent-level lifecycle side effects. |
| **Not for** | Blocking, approval, or execution-shaping logic — use guardrails/approvals/filters instead. |

### Example (Python — local MCP server via stdio)

```python
from agents import Agent, Runner
from agents.mcp import MCPServerStdio

async with MCPServerStdio(
    name="Filesystem MCP Server",
    params={"command": "npx", "args": ["-y", "@modelcontextprotocol/server-filesystem", "./sample_files"]},
) as server:
    agent = Agent(
        name="Filesystem assistant",
        instructions="Read files with the MCP tools before answering.",
        mcp_servers=[server],
    )
    result = await Runner.run(agent, "What files are in the workspace?")
    print(result.final_output)
```

### Example (TypeScript — local MCP via stdio)

```typescript
import { Agent, MCPServerStdio, run } from "@openai/agents";

const server = new MCPServerStdio({
  name: "Filesystem MCP Server",
  fullCommand: "npx -y @modelcontextprotocol/server-filesystem ./sample_files",
});
await server.connect();
try {
  const agent = new Agent({
    name: "Filesystem assistant",
    instructions: "Read files with the MCP tools before answering.",
    mcpServers: [server],
  });
  const result = await run(agent, "List the files.");
  console.log(result.finalOutput);
} finally {
  await server.close();
}
```

---

## 11. Evaluating Agent Workflows

**Docs:** [agent-evals](https://developers.openai.com/api/docs/guides/agent-evals) · [trace-grading](https://developers.openai.com/api/docs/guides/trace-grading)

### Main Concepts

The OpenAI Platform offers evaluation tools to ensure agents perform consistently and accurately. The progression is: **traces** (debugging) → **trace grading** (scoring) → **datasets & eval runs** (repeatability).

- **Start with traces** — A trace captures the end-to-end record of model calls, tool calls, guardrails, and handoffs for one run. The fastest way to identify workflow-level issues.
- **Graders** — Score traces with structured criteria to find regressions and failure modes at scale. Questions like: did the agent pick the right tool? did a handoff happen when it should have? did the workflow violate an instruction?
- **Datasets & eval runs** — Move from individual traces to repeatable datasets and eval runs when you need to benchmark changes, compare prompts, or run larger-scale evaluations over time.
- **Trace evals vs black-box evals** — Unlike black-box evaluations, trace evals provide more data to understand why an agent succeeds or fails.

### API Functions & Parameters

#### Trace-grading workflow

| Step | Action |
|------|--------|
| 1 | Open Logs > Traces in the dashboard. |
| 2 | Inspect a representative workflow trace (from SDK app or Agent Builder). |
| 3 | Create a grader and run it against selected traces. |
| 4 | Use results to refine prompts, tool surfaces, routing logic, or guardrails. |

#### Eval run configuration

| Option | Description |
|--------|-------------|
| Model | Which model to use for the eval run. |
| Date range | Filter traces by date. |
| Tool calls | Filter by tool call presence/absence. |
| Test criteria | Grader criteria added/edited in the evaluation dashboard. |
| "Grade all" | Bulk-grade traces and push to the evaluation dashboard. |

#### Eval questions (examples)

| Question | What it checks |
|----------|----------------|
| Did the agent pick the right tool? | Tool selection accuracy. |
| Did a handoff happen when it should have? | Routing correctness. |
| Did the workflow violate an instruction or safety policy? | Compliance. |
| Did a prompt or routing change improve end-to-end behavior? | Regression testing. |

#### SDK helpers (Python)

| Helper | Description |
|--------|-------------|
| `trace(name, metadata=...)` | Create a named workflow trace with metadata. |
| `gen_trace_id()` | Generate a trace ID for linking. |

---

## 12. Voice Agents

**Docs:** [voice-agents](https://developers.openai.com/api/docs/guides/voice-agents) · [realtime](https://developers.openai.com/api/docs/guides/realtime)

### Main Concepts

Voice agents turn the same agent concepts into spoken, low-latency interactions. The key design choice is whether the model works directly with live audio or whether the application explicitly chains speech-to-text, text reasoning, and text-to-speech.

- **Two architectures** — (1) **Speech-to-speech with live audio sessions** (Realtime API) for natural, low-latency conversations; (2) **Chained voice pipeline** (STT → text agent → TTS) for predictable workflows or extending an existing text agent.
- **Agent Builder doesn't support voice** — Voice stays an SDK-first surface.
- **Same core building blocks** — Voice agents still use tools, running agents, orchestration, guardrails, and observability. The voice surface only changes the transport and audio loop.
- **Language-specific helpers** — TypeScript: `RealtimeAgent` + `RealtimeSession` (fastest path to browser voice). Python: `VoicePipeline` + `SingleAgentVoiceWorkflow` (simplest path to extend a text agent).
- **Connection methods** — WebRTC (recommended for browser), WebSocket, SIP.

### API Functions & Parameters

#### Architecture choice

| Architecture | Best for | Why |
|--------------|----------|-----|
| Speech-to-speech (live audio) | Natural, low-latency conversations | Model handles live audio input/output directly. |
| Chained voice pipeline | Predictable workflows, extending text agents | App controls transcription, text reasoning, and speech output explicitly. |

#### Python — Chained voice pipeline

| Class | Description |
|-------|-------------|
| `VoicePipeline` | Interface for transcribing audio input, executing an agent workflow, and generating TTS response. |
| `SingleAgentVoiceWorkflow` | Wraps an existing text agent into a voice workflow. |
| `AudioInput` | Container for raw audio buffer (e.g. 24kHz PCM int16). |
| `TTSModelSettings` | Custom TTS settings (instructions for personality, tone, accent). |
| `VoicePipelineConfig` | Pipeline configuration (models, tracing, etc.). |

| Pipeline method | Description |
|-----------------|-------------|
| `pipeline.run(audio_input)` | Execute the pipeline; returns a streamable result. |
| `result.stream()` | Async iterator of voice events (`voice_stream_event_audio`, etc.). |

#### TypeScript — Realtime agent

| Class | Description |
|-------|-------------|
| `RealtimeAgent` | Agent configured for live audio sessions via the Realtime API. |
| `RealtimeSession` | Manages the WebRTC/WebSocket connection and session lifecycle. |

#### Realtime API endpoints

| Endpoint | Description |
|----------|-------------|
| `POST /v1/realtime` | Establish a realtime session (WebSocket upgrade). |
| `POST /v1/realtime/calls` | Establish a WebRTC session. |
| `POST /v1/realtime/client_secrets` | Create ephemeral credentials for browser/mobile clients. |

#### Models

| Model | Use |
|-------|-----|
| `gpt-realtime-2.1` | Low-latency voice agent (speech-to-speech). |
| `gpt-realtime-translate` | Live translation. |
| `gpt-realtime-whisper` | Realtime transcription. |
| `gpt-4o-mini-transcribe` | Speech-to-text (Transcription / Realtime API). |
| `gpt-4o-mini-tts` | Text-to-speech (Speech API). |
| `gpt-audio-mini` | Native speech-to-speech via Chat Completions API. |

### Example (Python — chained voice pipeline)

```python
import numpy as np
from agents import Agent
from agents.voice import VoicePipeline, SingleAgentVoiceWorkflow, AudioInput

agent = Agent(name="Guide", instructions="Answer travel questions concisely.")
pipeline = VoicePipeline(workflow=SingleAgentVoiceWorkflow(agent))

audio_input = AudioInput(buffer=np.zeros(24000 * 3, dtype=np.int16))
result = await pipeline.run(audio_input)
async for event in result.stream():
    if event.type == "voice_stream_event_audio":
        print("Received audio bytes", len(event.data))
```

### Design rule

Choose the audio architecture first, then design the rest of the agent workflow (tools, orchestration, guardrails, observability) the same way you would for text.

---

## 13. Capability Summary & Cross-Reference

### Capability → Key API surfaces

| Capability | Key API surfaces |
|------------|-----------------|
| Agent definitions | `Agent`, `tool()` / `@function_tool`, `outputType` / `output_type`, `RunContext` |
| Models & providers | `Agent.model`, `RunConfig.model`, `OPENAI_DEFAULT_MODEL`, `modelSettings`, `prompt` |
| Running agents | `Runner.run()` / `run()`, `Session` (`MemorySession`), `conversationId`, `previousResponseId`, streaming events |
| Sandbox agents | `Manifest`, `SandboxAgent`, `SandboxRunConfig`, `File`/`Dir`/`GitRepo`/`*Mount`, `UnixLocalSandboxClient`, `DockerSandboxClient`, `Shell()`/`Filesystem()`/`Compaction()`/`Memory()` |
| Orchestration | `Agent.handoffs`, `handoff()` wrapper, `agent.asTool()` / `agent.as_tool()` |
| Guardrails & approvals | Input/output/tool guardrails, `needsApproval: true`, `result.interruptions`, `result.state`, `state.approve()` / `state.reject()` |
| Results & state | `finalOutput`, `history`, `lastAgent`, `lastResponseId`, `interruptions`, `state` / `to_state()` |
| Integrations & observability | `MCPServerStdio`, `MCPServerStreamableHttp`, tracing (auto), `trace()`, `gen_trace_id()`, `RunHooks`, `AgentHooks` |
| Evaluation | Traces dashboard, graders, datasets, eval runs, "Grade all" |
| Voice agents | `VoicePipeline`, `SingleAgentVoiceWorkflow`, `AudioInput`, `TTSModelSettings`, `RealtimeAgent`, `RealtimeSession`, `/v1/realtime`, `/v1/realtime/calls` |

### Cross-capability concepts

| Concept | Where it appears |
|---------|------------------|
| **Agent loop** | Running agents (core); tools, handoffs, approvals, streaming all build on it |
| **State** | Running agents (4 strategies), Results & state (5 surfaces), Sandbox (3 state surfaces), Guardrails (resume from state) |
| **Tools** | Agent definitions (function tools), Integrations (MCP tools), Orchestration (agents-as-tools), Using tools guide (hosted tools) |
| **Tracing** | Integrations & observability (emission), Evaluation (grading), Quickstart (inspect early) |
| **Local vs model context** | Agent definitions (RunContext), Sandbox (harness vs compute), Integrations (local vs hosted MCP) |
| **Ownership** | Orchestration (handoff vs agent-as-tool), Results (lastAgent), Guardrails (agent-level scope) |
| **Pause & resume** | Guardrails (approvals), Running agents (pause vs failure), Results (interruptions + state), Sandbox (session resume) |

### Decision guide — "If you want to…"

| If you want to | Start here |
|----------------|------------|
| Build a code-first agent app | [Quickstart](https://developers.openai.com/api/docs/guides/agents/quickstart) |
| Define one specialist cleanly | Agent definitions (§3) |
| Choose models, defaults, and transport | Models & providers (§4) |
| Understand the runtime loop and state | Running agents (§5) |
| Run work in a container-based environment | Sandbox agents (§6) |
| Design specialist ownership | Orchestration (§7) |
| Add validation or human review | Guardrails & human review (§8) |
| Understand what a run returns | Results & state (§9) |
| Add hosted tools, function tools, or MCP | [Using tools](https://developers.openai.com/api/docs/guides/tools) + Integrations (§10) |
| Inspect and improve runs | Integrations & observability (§10) + Evaluation (§11) |
| Build a voice-first workflow | Voice agents (§12) |