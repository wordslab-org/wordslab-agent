# OpenAI API — Agents: Integrations, Observability, Guardrails & Evaluation

Analysis of the agent-related capabilities offered by the OpenAI API, based on four official guides in the **Agents SDK** documentation set:

- **Integrations and observability** — `developers.openai.com/api/docs/guides/agents/integrations-observability`
- **Guardrails and human review** — `developers.openai.com/api/docs/guides/agents/guardrails-approvals`
- **Evaluate agent workflows** — `developers.openai.com/api/docs/guides/agent-evals`
- **Evaluation best practices** — `developers.openai.com/api/docs/guides/evaluation-best-practices`

Scope of this study: capabilities relevant to **agents** — i.e. capabilities exposed through the OpenAI **Agents SDK** (`@openai/agents` for TypeScript, `agents` for Python) that sit around the model + tools + handoffs loop. Each capability is broken down into main concepts, API/SDK surface (functions, classes, parameters, fields), and notes/constraints.

Four distinct surfaces are covered:

- **MCP integrations** attach Model Context Protocol tool surfaces to an agent. Two transports exist: **hosted MCP** (the remote server runs through the OpenAI platform trust model) and **SDK-managed MCP** (your runtime owns the connection over stdio or streamable HTTP).
- **Tracing** is **built into the Agents SDK and enabled by default**. Every run emits a structured record of model calls, tool calls, handoffs, guardrails, and custom spans, viewable in the OpenAI **Traces dashboard**. There is no separate tracing API to call — observability is automatic.
- **Guardrails & human-in-the-loop approvals** add automatic validation and human review boundaries to SDK workflows. Guardrails validate input/output/tool behavior automatically; approvals pause the run so a person/policy can approve or reject a sensitive tool call. Both share a resumable **state** model.
- **Agent workflow evaluation** is the platform-level surface for scoring agent quality. It moves from **traces** (debugging) → **trace grading** (graders scoring traces) → **datasets and eval runs** (repeatable, comparable evaluations). The companion **Evaluation best practices** guide is framework-agnostic guidance for designing evals for single-turn, workflow, single-agent, and multi-agent architectures.

> Note: The OpenAI **Evals platform** is being deprecated. Existing evals content remains available during a transition window; evals become read-only for existing users on **October 31, 2026**, and the platform is scheduled to shut down on **November 30, 2026**. The agent-evals guide points newer code-first SDK workflows toward trace grading + datasets rather than the legacy Evals API.

---

## Table of Contents

1. [MCP integrations](#1-mcp-integrations)
   - 1.1 [Hosted MCP tools](#11-hosted-mcp-tools)
   - 1.2 [SDK-managed MCP servers (local/private)](#12-sdk-managed-mcp-servers-localprivate)
   - 1.3 [Choosing a transport](#13-choosing-a-transport)
2. [Tracing](#2-tracing)
   - 2.1 [What a default trace contains](#21-what-a-default-trace-contains)
   - 2.2 [Wrapping multiple runs in one trace](#22-wrapping-multiple-runs-in-one-trace)
   - 2.3 [Tracing controls & the Traces dashboard](#23-tracing-controls--the-traces-dashboard)
3. [Guardrails](#3-guardrails)
   - 3.1 [Guardrail types & where they run](#31-guardrail-types--where-they-run)
   - 3.2 [Input guardrail (blocking)](#32-input-guardrail-blocking)
   - 3.3 [Output & tool guardrails](#33-output--tool-guardrails)
   - 3.4 [Parallel vs blocking execution](#34-parallel-vs-blocking-execution)
4. [Human-in-the-loop approvals](#4-human-in-the-loop-approvals)
   - 4.1 [Marking a tool as needing approval](#41-marking-a-tool-as-needing-approval)
   - 4.2 [Approval lifecycle & resumable state](#42-approval-lifecycle--resumable-state)
   - 4.3 [Streaming & delayed review](#43-streaming--delayed-review)
5. [Agent workflow evaluation](#5-agent-workflow-evaluation)
   - 5.1 [Trace grading](#51-trace-grading)
   - 5.2 [Datasets and eval runs](#52-datasets-and-eval-runs)
   - 5.3 [Evaluation surfaces map](#53-evaluation-surfaces-map)
6. [Evaluation best practices](#6-evaluation-best-practices)
   - 6.1 [What are evals](#61-what-are-evals)
   - 6.2 [Design your eval process](#62-design-your-eval-process)
   - 6.3 [Identify where you need evals (by architecture)](#63-identify-where-you-need-evals-by-architecture)
   - 6.4 [Evaluator types](#64-evaluator-types)
   - 6.5 [Handle edge cases](#65-handle-edge-cases)
   - 6.6 [Use evals to improve performance](#66-use-evals-to-improve-performance)

---

## 1. MCP integrations

**Summary** — The Agents SDK lets you attach Model Context Protocol (MCP) tool surfaces to an agent so the model can call remote or local MCP servers' tools. Capability semantics are described in the canonical *Using tools / MCP and Connectors* guide; this surface covers the SDK-specific wiring and the trust-model split between hosted and local transports.

### 1.1 Hosted MCP tools

Use **hosted MCP** when the remote MCP server should run through the OpenAI platform/model surface (public, remotely hosted servers that fit the platform trust model).

| Surface | TypeScript | Python |
|---|---|---|
| Import | `hostedMcpTool` from `@openai/agents` | `HostedMCPTool` from `agents` |
| Attach to | `Agent.tools` array | `Agent.tools` list |

`hostedMcpTool` / `HostedMCPTool` parameters (TypeScript shape):

| Parameter | Type | Description |
|---|---|---|
| `serverLabel` | string | Label identifying the MCP server |
| `serverUrl` | string | Remote MCP server URL |

Python `HostedMCPTool` takes a single `tool_config` dict:

| Key | Type | Description |
|---|---|---|
| `type` | string | `"mcp"` |
| `server_label` | string | Label identifying the MCP server |
| `server_url` | string | Remote MCP server URL |
| `require_approval` | string | Approval policy for the hosted MCP actions, e.g. `"never"` |

### 1.2 SDK-managed MCP servers (local/private)

Use **local or private MCP** when your application should connect to the MCP server directly (you own connectivity, filtering, approvals, and network boundaries). The SDK transports are **stdio** and **streamable HTTP**.

| Surface | TypeScript | Python |
|---|---|---|
| Class | `MCPServerStdio` from `@openai/agents` | `agents.mcp.MCPServerStdio` |
| Attach to | `Agent.mcpServers` array | `Agent.mcp_servers` list |

`MCPServerStdio` parameters:

| Parameter (TS) | Parameter (Py) | Type | Description |
|---|---|---|---|
| `name` | `name` | string | Server display name |
| `fullCommand` | `params.command` + `params.args` | string / list | The command and args to launch the server subprocess |

Lifecycle:

| Method | Purpose |
|---|---|
| `await server.connect()` (TS) / `async with MCPServerStdio(...) as server` (Py) | Establish the connection to the MCP server |
| `await server.close()` (TS) / context-manager exit (Py) | Tear down the connection |

### 1.3 Choosing a transport

| Need | Use |
|---|---|
| Public, remotely hosted MCP tools that fit the platform trust model | **Hosted MCP** tools in the SDK |
| Local or private MCP servers where your runtime owns connection, approvals, network boundaries | **SDK-managed MCP** servers over stdio / streamable HTTP |

The canonical MCP concept, trust model, and product-support story live in *MCP and Connectors*; this page is the SDK wiring layer.

---

## 2. Tracing

**Summary** — Tracing is **built into the Agents SDK and enabled by default** in the normal server-side SDK path. Every run can emit a structured record of model calls, tool calls, handoffs, guardrails, and custom spans, viewable in the [Traces dashboard](https://platform.openai.com/traces). There is no separate tracing API to call — you tune or scope tracing via SDK-level / per-run controls rather than removing observability from the workflow.

### 2.1 What a default trace contains

A default trace usually gives you:

- The overall run or workflow
- Each model call
- Tool calls and their outputs
- Handoffs and guardrails
- Any custom spans you wrap around the workflow

### 2.2 Wrapping multiple runs in one trace

To group multiple `run` calls into a single trace (e.g. a multi-step workflow), wrap them in a trace context:

| Surface | TypeScript | Python |
|---|---|---|
| Import | `withTrace` from `@openai/agents` | `trace` from `agents` |
| Usage | `await withTrace("Joke workflow", async () => { ... })` | `with trace("Joke workflow"):` |

`withTrace` / `trace` parameters:

| Parameter | Type | Description |
|---|---|---|
| name (first positional) | string | Trace/workflow name shown in the dashboard |
| callback (TS) | async fn | The block of `run` calls to capture in one trace |

### 2.3 Tracing controls & the Traces dashboard

- Use SDK-level or per-run tracing controls to reduce tracing rather than removing all observability.
- Traces serve two jobs: (1) **debug one workflow run** and understand what happened; (2) **feed higher-signal examples** into agent workflow evaluation once behavior stabilizes.

---

## 3. Guardrails

**Summary** — Guardrails are **automatic validation checks** that run inside the agent loop. They validate input, output, or tool behavior and decide whether a run should continue or stop. They are the complement to human-in-the-loop approvals (which pause for a person). Use guardrails for fast, automatic checks; use approvals for sensitive, side-effecting actions.

### 3.1 Guardrail types & where they run

| Use case | Start with |
|---|---|
| Block disallowed user requests before the main model runs | **Input guardrails** |
| Validate or redact the final output before it leaves the system | **Output guardrails** |
| Check arguments or results around a function tool call | **Tool guardrails** |
| Pause before side effects (cancellations, edits, shell, sensitive MCP actions) | Human-in-the-loop approvals (see §4) |

**Workflow boundaries matter** — agent-level guardrails don't run everywhere:

- **Input guardrails** run only for the **first agent** in the chain.
- **Output guardrails** run only for the agent that produces the **final output**.
- **Tool guardrails** run on the function tools they're attached to.

For checks around every custom tool call in a manager-style workflow, do not rely only on agent-level input/output guardrails — put validation next to the tool that creates the side effect.

### 3.2 Input guardrail (blocking)

An input guardrail runs a fast validation step before the expensive/side-effecting part of the workflow starts. A common pattern is a **separate guardrail agent** that classifies the input (e.g. "is this math homework?") with a structured output schema, returning whether a tripwire was triggered.

| Surface | TypeScript | Python |
|---|---|---|
| Attach to | `Agent.inputGuardrails` array | `Agent.input_guardrails` list |
| Decorator | (object form, see below) | `@input_guardrail` on an async fn |
| Output type | `GuardrailFunctionOutput` (returned from `execute`) | `GuardrailFunctionOutput` (returned) |
| Exception on block | `InputGuardrailTripwireTriggered` | `InputGuardrailTripwireTriggered` |

TypeScript input guardrail object shape (added to `Agent.inputGuardrails`):

| Field | Type | Description |
|---|---|---|
| `name` | string | Guardrail name |
| `runInParallel` | boolean | Whether to run in parallel with other guardrails |
| `execute` | async fn | `async ({ input, context }) => { outputInfo, tripwireTriggered }` |

Python `@input_guardrail` function signature:

| Parameter | Type | Description |
|---|---|---|
| `ctx` | `RunContextWrapper[T]` | Run context wrapper |
| `agent` | `Agent` | The agent the guardrail is attached to |
| `input` | `str \| list[TResponseInputItem]` | The input being validated |

`GuardrailFunctionOutput` fields:

| Field | Type | Description |
|---|---|---|
| `output_info` / `outputInfo` | any | Structured info from the guardrail (e.g. classification result) |
| `tripwire_triggered` / `tripwireTriggered` | boolean | If `true`, the run is blocked with `InputGuardrailTripwireTriggered` |

The guardrail agent typically uses **structured output** (Pydantic `BaseModel` / Zod schema) to return a typed verdict like `{ is_math_homework: bool, reasoning: str }`.

### 3.3 Output & tool guardrails

- **Output guardrails** validate or redact the final output before it leaves the system; they run only for the agent producing the final output.
- **Tool guardrails** check arguments or results around a specific function tool call; they are attached to the function tool itself rather than the agent.

(Same `GuardrailFunctionOutput` / tripwire contract as input guardrails, applied at the relevant boundary.)

### 3.4 Parallel vs blocking execution

- Use **blocking execution** when the cost or risk of starting the main agent is too high (the guardrail must pass before the main work begins).
- Use **parallel guardrails** when lower latency matters more than avoiding speculative work (guardrails run alongside the main agent; `runInParallel: true`).

---

## 4. Human-in-the-loop approvals

**Summary** — Approvals are the **human-in-the-loop path for tool calls**. The model can still decide an action is needed, but the run **pauses** until you approve or reject it. This is the right control for side effects like cancellations, edits, shell commands, or sensitive MCP actions.

### 4.1 Marking a tool as needing approval

A function tool opts into the approval flow via a flag at definition time.

| Surface | TypeScript | Python |
|---|---|---|
| Define | `tool({ ... })` from `@openai/agents` | `@function_tool(...)` decorator from `agents` |
| Flag | `needsApproval: true` (in the `tool({...})` options) | `needs_approval=True` (decorator kwarg) |

TypeScript `tool({...})` parameters (approval-relevant):

| Parameter | Type | Description |
|---|---|---|
| `name` | string | Tool name |
| `description` | string | Tool description |
| `parameters` | Zod schema | Arguments schema |
| `needsApproval` | boolean | If `true`, the run pauses for approval instead of executing |
| `execute` | async fn | The tool's implementation |

Python `@function_tool` parameters (approval-relevant):

| Parameter | Type | Description |
|---|---|---|
| `needs_approval` | bool | If `True`, the run pauses for approval instead of executing |
| (function signature) | — | Arguments become the tool's parameters; return value is the tool result |

### 4.2 Approval lifecycle & resumable state

When a tool call needs review, the SDK follows the same pattern every time:

1. The run records an **approval interruption** instead of executing the tool.
2. The result returns **`interruptions`** plus a resumable **`state`**.
3. Your application **approves or rejects** the pending items via `state.approve(interruption)`.
4. You **resume the same run** from `state` (e.g. `await run(agent, state)`) instead of starting a new user turn.

| Surface | TypeScript | Python |
|---|---|---|
| Read interruptions | `result.interruptions` | `result.interruptions` |
| Get resumable state | `result.state` | `result.to_state()` |
| Approve an item | `state.approve(interruption)` | `state.approve(interruption)` |
| Resume | `await run(agent, state)` | `await Runner.run(agent, state)` |

The same interruption pattern applies even when the approving tool lives **deeper in the workflow** — e.g. after a handoff or inside a nested `agent.asTool()` call.

If review might take time, **serialize `state`, store it, and resume later** — it is still the same run.

### 4.3 Streaming & delayed review

Streaming does not create a separate approval system. If a streamed run pauses:

1. Wait for it to settle.
2. Inspect `interruptions`.
3. Resolve the approvals.
4. Resume from the same `state`.

If the review happens later, store the serialized `state` and continue the same run when the decision arrives.

---

## 5. Agent workflow evaluation

**Summary** — The OpenAI Platform offers a suite of evaluation tools for agent workflows. The recommended path is a maturity progression: start with **traces** (debugging), move to **trace grading** (graders scoring traces), then to **datasets and eval runs** (repeatable, comparable evaluation). This is the decision point for which evaluation surface matters most for agent workflows.

### 5.1 Trace grading

Trace grading is the **fastest way to identify workflow-level issues**. A trace captures the end-to-end record of model calls, tool calls, guardrails, and handoffs for one run. **Graders** let you score those traces with structured criteria so you can find regressions and failure modes at scale.

Use trace grading to answer questions like:

- Did the agent pick the right tool?
- Did a handoff happen when it should have?
- Did the workflow violate an instruction or safety policy?
- Did a prompt or routing change improve end-to-end behavior?

Trace-grading workflow:

1. Open **Logs > Traces** in the dashboard.
2. Inspect a representative workflow trace from an SDK-based app (or an existing Agent Builder workflow during the transition window).
3. Create a **grader** and run it against the selected traces.
4. Use the results to refine prompts, tool surfaces, routing logic, or guardrails.

For code-first SDK workflows, start with the tracing surface (§2) to get high-signal traces before formalizing graders.

### 5.2 Datasets and eval runs

Once you know what "good" looks like, move from individual traces to **repeatable datasets and eval runs** — the right step when you want to benchmark changes, compare prompts, or run larger-scale evaluations over time.

For advanced features (evaluation against external models, evaluation APIs, larger-scale batch evaluation), use the **Evals** surface alongside datasets.

### 5.3 Evaluation surfaces map

| Surface | When to use |
|---|---|
| **Traces** | Still debugging behavior; understand what happened in one run |
| **Trace grading** (traces + graders) | Find regressions/failure modes at scale; score workflow behavior |
| **Datasets & eval runs** | Need repeatability; benchmark changes, compare prompts, larger-scale evals |
| **Evals** (API) | Advanced: external-model eval, evaluation APIs, larger-scale batch eval |

Related surfaces referenced by the guide: *Getting started with evals: Datasets*, *Working with evals*, *Prompt optimizer*, and the cookbook *Building resilient prompts with evals* (an evaluation flywheel).

---

## 6. Evaluation best practices

**Summary** — Generative AI is variable: models sometimes produce different output from the same input, which makes traditional software testing methods insufficient. **Evals** are structured tests that measure a model's performance despite this nondeterminism. This guide is **framework-agnostic guidance** for designing your own evals (the third type of "evals" — not industry benchmarks or generic numerical scores). It is not a dedicated API surface; it is a methodology layered on top of the Evals API / datasets / graders.

### 6.1 What are evals

Evals are **structured tests for measuring model performance** — the main way to ensure accuracy, performance, and reliability despite nondeterminism, and one of the only ways to **improve** an LLM application's performance (via fine-tuning).

**Types of "evals"** the term can mean:

1. Industry benchmarks for comparing models in isolation (e.g. MMLU, HuggingFace leaderboard).
2. Standard numerical scores used as you design evals (e.g. ROUGE, BERTScore).
3. Specific tests **you** implement to measure your LLM application's performance — **this guide's focus**.

**How to read evals** — combine numerical scores (often 0–1) with human judgment; metrics alone are insufficient.

**Tips**: eval-driven development (evaluate early and often); design task-specific evals; log everything to mine for eval cases; automate scoring when possible; treat evaluation as a continuous journey; calibrate automated scoring with human feedback.

**Anti-patterns**: overly generic metrics (perplexity/BLEU alone); biased datasets that don't reproduce production traffic; "vibe-based" evals ("it seems to work"); ignoring human feedback for calibration.

### 6.2 Design your eval process

Five components of an eval workflow:

1. **Define eval objective** — what's the success criteria?
2. **Collect dataset** — consider synthetic, domain-specific, purchased, human-curated, production, and historical data.
3. **Define eval metrics** — how to check success criteria are met (e.g. ROUGE-L ≥ 0.40, coherence ≥ 80% via G-Eval; context recall ≥ 0.85, context precision > 0.7).
4. **Run and compare evals** — iterate to improve performance (use the Evals API to create/run in the dashboard).
5. **Continuously evaluate (CE)** — run evals on every change, monitor for new nondeterminism, grow the eval set over time.

**Design principle** — LLMs are better at **discriminating** between options than open-ended generation. Prefer evals that are pairwise comparisons, classification, or scoring against specific criteria.

**Worked examples**: transcript summarization (ROUGE-L + G-Eval coherence) and Q&A over docs (context recall + context precision + positive-answer rate). When building an eval dataset, `gpt-5.6` is useful for collecting examples and edge cases; include typical, edge, and adversarial cases; use human expert labelers.

### 6.3 Identify where you need evals (by architecture)

Complexity (and where nondeterminism enters) increases across four architecture patterns. Identify where nondeterminism enters → that's where to implement evals.

| Architecture | New nondeterminism | Area to evaluate |
|---|---|---|
| **Single-turn model interactions** | Developer/user inputs; model outputs | Instruction following; functional correctness |
| **Workflows** (chained model calls) | (no new nondeterminism, but multiple model interactions to eval in isolation) | Instruction following per step; functional correctness per step + end-to-end |
| **Single-agent** (instructions + tools, dynamic tool choice) | **Tools chosen by the model** | Instruction following; functional correctness; **tool selection**; **data precision** (correct arguments extracted from history) |
| **Multi-agent** (triage + handoff among specialized agents) | **Agent handoff** | All of single-agent + **agent handoff accuracy** (decision boundaries for triaging to another agent) |

> The decision to use a multi-agent architecture should be **driven by your evals** — starting multi-agent adds unnecessary complexity that slows time to production.

### 6.4 Evaluator types

Three roles an evaluator can play (combine them):

**Metric-based evals** (quantitative)

- Examples: exact match, string match, ROUGE/BLEU, function-call accuracy, executable evals (e.g. text2sql executed to assess behavior).
- Challenges: may not be tailored to use cases, may miss nuance.

**Human evals** (highest quality, slow/expensive)

- Examples: skim outputs to judge better/worse; randomized blinded tests where labelers rank outputs or grade 1–5.
- Challenges: expert disagreement, expensive, slow.
- Recommendations: multiple review rounds to refine the scorecard; "show rather than tell" with example score levels (1, 3, 8 of 10); include a pass/fail threshold alongside the score; aggregate multiple reviewers via consensus votes.

**LLM-as-a-judge and model graders** (cheaper, scalable)

- Start with `gpt-5.6` as a strong LLM judge, then validate agreement against human labels before optimizing for cost/latency.
- Examples: **pairwise comparison** (which of two responses is better by criteria); **single-answer grading** (score one response against predefined metrics); **reference-guided grading** (judge against a "gold standard" answer).
- Challenges: position bias (response order), verbosity bias (preferring longer responses).
- Recommendations: prefer pairwise comparison or pass/fail for reliability; use the most capable model to grade; control for response length; add reasoning/chain-of-thought before scoring; restructure questions as multiple choice for automated grading while preserving task integrity; keep rubrics clear and detailed.

No strategy is perfect — LLM-as-judge quality varies by problem context, and expert human ground-truth labels are expensive/time-consuming.

### 6.5 Handle edge cases

Beyond happy-path scenarios, evaluate edge cases (key for reliability and UX):

**Input variability** — non-English/multilingual inputs; non-text formats (XML, JSON, Markdown, CSV); non-text input modalities (e.g. images). Instruction-following and functional-correctness evals must accommodate these.

**Contextual complexity** — multiple questions/intents per request; typos/misspellings; short requests with minimal context (e.g. just "returns"); long context / long-running conversations; tool calls returning data with ambiguous property names (e.g. `"on": 123`); multiple tool calls leading to incorrect arguments; multiple agent handoffs sometimes leading to **circular handoffs**.

**Personalization and customization** — jailbreak attempts; formatting requests (JSON, bullet points); cases where user prompts **conflict with system prompts**. Clearly define evals for use cases to specifically support and block.

### 6.6 Use evals to improve performance

Once evals mature into a consistent performance measure, shift to using eval data to **improve** the application — feed eval data into the optimization loop via **reinforcement fine-tuning** to create a data flywheel.

---

## Cross-cutting notes

- The four surfaces form a maturity pipeline: **wire integrations → trace what happens → add guardrails/approvals as boundaries → grade traces → formalize datasets/evals → improve via fine-tuning.**
- **Observability is automatic, not an API.** Tracing is built into the Agents SDK and enabled by default; there is no separate "send trace" REST call — you scope or reduce it via SDK controls, then inspect results in the Traces dashboard.
- **Guardrails (automatic) and approvals (human) share the same resumable `state` model.** Guardrails trip synchronously and raise `InputGuardrailTripwireTriggered`; approvals pause the run and resume via `state.approve(...)` + `run(agent, state)`. Both apply across handoffs and nested `agent.asTool()` calls.
- **Boundaries are positional.** Input guardrails run only for the first agent; output guardrails only for the final-output agent; tool guardrails only on their attached tool. For per-tool validation in manager-style workflows, attach validation to the tool, not the agent.
- **MCP has two transports with different trust ownership.** Hosted MCP = platform trust model (remote public servers); SDK-managed MCP = your runtime owns connectivity/approvals/network (stdio / streamable HTTP).
- **Evaluation is layered on traces, not separate from them.** Trace grading reuses the same end-to-end traces produced by default tracing; datasets/eval runs then add repeatability and comparison. The legacy Evals platform is being deprecated (read-only Oct 31 2026; shutdown Nov 30 2026), with newer code-first workflows pointed toward trace grading + datasets.
- **Eval design is architecture-aware.** Single-turn → eval instruction-following + functional-correctness; workflows → eval each step in isolation + end-to-end; single-agent → add tool-selection and data-precision (argument extraction) evals; multi-agent → add agent-handoff-accuracy evals. Don't start multi-agent without evals driving the decision.