# IBM watsonx Orchestrate API Analysis — Agent Capabilities

> **Docs (UI / concepts):** `https://www.ibm.com/docs/en/watsonx/watson-orchestrate/base?topic=getting-started`
> **Docs (developer / pro-code / API reference):** `https://developer.watson-orchestrate.ibm.com` | **OpenAPI spec:** `https://developer.watson-orchestrate.ibm.com/apis/server_openapi.json` (OpenAPI 3.1.0, 173 paths, 431 schemas; title `WxO Server API`)
> **Base API:** `https://{api_endpoint}` (per-instance) | **Auth:** bearer token from `POST /v1/auth/login` (IAM / MCSP / CPD), passed as `Authorization: Bearer <token>`
> **CLI:** `orchestrate` (Python package `ibm-watsonx-orchestrate`, the Agent Development Kit / ADK) | **Python runtime SDK:** `ibm-watsonx-orchestrate-sdk` (the Agentic SDK, LangChain/LangGraph integration)
> **Description:** IBM watsonx Orchestrate is an agentic control plane whose primary abstraction is the **agent** — a combination of a large language model (LLM), a reasoning **style**, a set of **tools**, optional **collaborator** agents, optional **knowledge bases**, **instructions**, and **channels**. Unlike a hosted-sandbox model, watsonx Orchestrate does *not* execute the agent loop for native agents in a single server-managed container; instead it orchestrates registered agent definitions, versioned **releases**, deployment **environments** (draft/live), and conversational **runs** over **threads** of **messages**. Capabilities are exposed through three complementary surfaces: a **REST API** (resource-oriented CRUD for agents, tools, knowledge, threads, runs, flows, channels), the **ADK CLI** (import/export of YAML/JSON definitions, chat), and the **Agentic SDK** (runtime integration for custom LangGraph agents running *on* the platform). External agents built on other frameworks plug in via the **Agent Connect Framework (ACF)** — an OpenAI-compatible `/chat/completions` interface — or the **A2A** protocol.

---

## Table of Contents

1. [Documentation Map & First-Level Menu](#1-documentation-map--first-level-menu)
2. [Platform Overview & Core Concepts](#2-platform-overview--core-concepts)
3. [Agent Configuration & Registration](#3-agent-configuration--registration)
4. [Agent Releases, Environments & Deployment Lifecycle](#4-agent-releases-environments--deployment-lifecycle)
5. [Running Agents: Runs, Threads & Messages](#5-running-agents-runs-threads--messages)
6. [Chat Completions & Model Gateway](#6-chat-completions--model-gateway)
7. [External Agents & Interoperability (ACF / A2A / watsonx Assistants)](#7-external-agents--interoperability-acf--a2a--watsonx-assistants)
8. [Tools & Toolkits](#8-tools--toolkits)
9. [Agentic Workflows (Flows)](#9-agentic-workflows-flows)
10. [Knowledge Bases, Document Collections & Vector Indices](#10-knowledge-bases-document-collections--vector-indices)
11. [Connections & Credentials](#11-connections--credentials)
12. [Models & Model Policies](#12-models--model-policies)
13. [Channels & Embedded Chat](#13-channels--embedded-chat)
14. [Chat Starter Settings, Voice & Agent UX](#14-chat-starter-settings-voice--agent-ux)
15. [Agent Evaluation & Testing](#15-agent-evaluation--testing)
16. [Monitoring, Governance & Observability](#16-monitoring-governance--observability)
17. [Agentic SDK (Runtime Integration for Custom Agents)](#17-agentic-sdk-runtime-integration-for-custom-agents)
18. [Scheduling](#18-scheduling)
19. [Capability Summary & Cross-Reference](#19-capability-summary--cross-reference)

---

## 1. Documentation Map & First-Level Menu

The IBM watsonx Orchestrate documentation is split across several sources, because the product spans UI-driven no-code authoring and pro-code developer tooling. The **first-level menu** of the core UI documentation (`https://www.ibm.com/docs/en/watsonx/watson-orchestrate/base`) is:

| First-level menu entry | Scope | Agent relevance |
|------------------------|-------|------------------|
| **Release notes** | What's new, known issues, feature parity, deprecations, ADK/on-prem release notes, regional availability | Context only |
| **Getting started** | Logging in, exploring doc sources, getting help, on-prem install, agent tutorials | Entry point to agents |
| **Designing** | Preparing to build AI agents, core components of an AI agent (identity, knowledge, actions, behavior, channels), discovering the catalog | Conceptual agent architecture |
| **Building** | Building agents, building tools, configuring agents, testing agents, deploying agents | Primary agent-build surface (UI) |
| **Using** | Using AI agents in Orchestrate Chat | End-user runtime |
| **Operating** | Agentic Control Plane, monitoring with agent analytics | Governance/runtime ops |
| **Administering** | Managing users, connections, instance security, credentials | Admin config |
| **Learning resources** | Tutorials, IBM Developer content | Guided walkthroughs |
| **Reference** | API reference, ADK reference | The REST surface analyzed below |

The developer/pro-code documentation (`developer.watson-orchestrate.ibm.com`) mirrors this with a different cut, organized as: **Get Started → Environments → Agents → Tools and Connections → Knowledge Bases → Managing → Evaluate → Channels → SDK → MCP Server → Scheduling → Voice → Webchat → Workspaces**. Its `llms.txt` index lists ~480 pages; the agent-related API endpoints are published as an OpenAPI 3.1.0 spec under `apis/server_openapi.json`. The analysis below is built from that machine-readable spec plus the ADK concept pages.

---

## 2. Platform Overview & Core Concepts

### Main Concepts

watsonx Orchestrate is built around these core abstractions (from the ADK "What is watsonx Orchestrate" and "Overview" pages):

- **Agent** — An AI entity that combines an LLM, a reasoning *style*, *instructions*, *tools*, optional *collaborator* agents, and optional *knowledge bases* to execute tasks on behalf of users. The unit of configuration reuse. Registered by name; every save produces a new version.
- **Agent type** — Three types: `native` (built directly in watsonx Orchestrate), `external_chat` (an external agent reachable via an OpenAI-compatible `/chat/completions` endpoint, i.e. ACF), and `watsonx` (a watsonx Assistant). Enum `AgentType: external_chat | watsonx | native`.
- **Style** — The reasoning approach the agent uses. Enum `AgentStyle: default | react | planner | custom | react_intrinsic | experimental_customer_care | code_act`. `default` is tool-centric for simple linear tasks; `react` is ReAct (reason + act) for multi-step problem solving; `react_intrinsic` uses the model's native chain-of-thought; `planner` decomposes tasks. (GPT-OSS-120B ignores all styles.)
- **Tool** — A capability an agent can invoke. Defined by `name`, `description`, `permission`, an `input_schema`, an `output_schema`, and a **binding** that selects the implementation (OpenAPI, Python, Flow, MCP, Skill, client-side, conversational-search, langflow, wxflows). Tools are first-class, reusable across agents, and separately versioned.
- **Toolkit** — A grouping of related tools (Python tools sharing one process, or an MCP server's tools) imported/removed as a unit. Referenced from an agent via `toolkits[]`.
- **Knowledge base** — Domain knowledge for an agent, either *built-in* (a managed Milvus index populated with uploaded documents) or *external* (your own Milvus/Elasticsearch, or a custom conversational-search endpoint). Represented as `auto` or `tool` (`KnowledgeBaseRepresentation`).
- **Collaborator** — Another agent (native, external, or watsonx Assistant) that an agent can delegate to, enabling multi-agent orchestration. Supervisor agents route to collaborators based on their `description`.
- **Environment** — A named deployment target for an agent (e.g. *draft* vs *live*), pointing at a current agent version. Agents are released into environments; channels bind to (agent, environment) pairs.
- **Release / Version** — An immutable snapshot of an agent configuration. `version_label` (integer) and optional `semantic_version` (e.g. `1.2.0`). Releases can be deployed, undeployed, and switched per environment.
- **Thread** — A conversation container (a persisted list of messages) tied to an `agent_id` and/or `assistant_id`. Status enum `ThreadStatus: async_wait | ready | async_slot_request | async_a2a_slot_request`.
- **Message** — A single turn in a thread, with `role`, `content` (string or array of content blocks), optional `mentions`, `document_ids`, `parent_message_id`, `step_history`, and `message_state`.
- **Run** — An execution of an agent (or assistant) over a thread. Status enum `AssistantRunStatus: pending | running | completed | async_wait | async_completed | failed | cancelled | requires_input | expired`. Runs emit a stream of `RunEvent`s.
- **Flow (agentic workflow)** — A directed graph of nodes (tool, agent, branch, loop/foreach, decisions, generative-prompt, timer, user-activity, document-processing) that acts *as a tool* with built-in agentic capabilities. Built with the `@flow` decorator in Python and imported as a tool.
- **Connection** — A credential binding (OAuth2, API key, basic auth) associated with a tool/agent so it can call external services. Registered once, referenced by `connection_id`.
- **Channel** — A delivery surface for an agent in a (agent, environment) pair: Slack, Microsoft Teams, Twilio SMS/WhatsApp, Facebook, Genesys, web chat, phone/voice.
- **Catalog** — A governed library of pre-built agents and tools (HR, IT, procurement, sales, productivity) that can be discovered and reused.
- **Context variables** — Platform-provided runtime values (e.g. `wxo_email_id`, `wxo_user_name`, `wxo_tenant_id`) that agents can read when `context_access_enabled` is true and the variable is listed in `context_variables[]`. Referenced in instructions via `{wxo_email_id}`.
- **Memory** — Agentic memory (`memory_enabled`): persistent per-user information the agent can write to and search during runs, exposed to custom agents through the Agentic SDK `client.memory.*`.

### Agent Capabilities Map

| Capability | Description | Primary surface |
|------------|-------------|-----------------|
| **Agent configuration** | Define native/external/assistant agents: model, style, instructions, tools, collaborators, knowledge, context vars, plugins | REST `/v1/orchestrate/agents`, ADK YAML, UI |
| **Releases & versioning** | Snapshot agent configs; list/get/delete versions; deploy/undeploy/switch per environment | REST `/v1/orchestrate/agents/{id}/releases` |
| **Environments** | Named draft/live targets; per-environment current version & channels | REST `/v1/orchestrate/agents/{agent_id}/environment` |
| **Runs (chat)** | Start/cancel/get/list agent executions; streaming via SSE events | REST `/v1/orchestrate/runs`, `/v1/orchestrate/{agent_id}/chat/completions` |
| **Threads & messages** | Persisted conversation state; CRUD on threads and messages | REST `/v1/threads` |
| **Chat completions gateway** | OpenAI-compatible passthrough to underlying models (with/without an agent) | REST `/v1/orchestrate/gateway/model/chat/completions` |
| **External agents (ACF / A2A)** | Register remote OpenAI-compatible agents or A2A agents as collaborators | REST `/v1/agents/external-chat`, `/v1/a2a/versions` |
| **Tools & toolkits** | CRUD for tools and toolkits; 9 binding types; async tool callbacks | REST `/v1/tools`, `/v1/toolkits` |
| **Agentic workflows** | Build/import flows as tools; run flow instances (sync/async) | REST `/v1/orchestrate/flows`, ADK `@flow` |
| **Knowledge bases** | Create/ingest/patch/delete KBs; built-in or external vector indices; chat-with-docs per thread | REST `/v1/orchestrate/knowledge-bases`, `/v1/document-collections`, `/v1/vector-indices` |
| **Connections** | Create/list/delete credential bindings; OAuth callback | REST `/v1/connections/applications` |
| **Models & policies** | Register/list/update/delete LLMs and embeddings; model policies for governance | REST `/v1/models`, `/v1/model_policy` |
| **Channels** | Bind agents to Slack/Teams/Twilio/Facebook/Genesys/web chat/phone per environment | REST `/v1/orchestrate/agents/{agent_id}/environments/{environment_id}/channels/...` |
| **Chat starter / embedded chat / voice** | Welcome message, starter prompts, icon, embedded chat config, voice configuration | REST `/v1/orchestrate/agents/{id}/chat-starter-settings`, `/embedded-chat-config` |
| **Evaluation & testing** | Upload CSV test cases, run evaluations, list/export results | REST `/v1/orchestrate/agent/{agent_id}/test_case`, `/evaluate`, `/evaluations` |
| **Monitoring** | Enable agent monitoring, fetch governance metrics | REST `/v1/orchestrate/agents/.../monitoring` |
| **Agentic SDK** | Runtime integration for custom LangGraph agents: client, chat models, memory, context compression, embeddings | Python `ibm-watsonx-orchestrate-sdk` |
| **Scheduling** | Recurring agent/flow executions (set `is_schedulable` / `schedulable`) | ADK + scheduling config |

### Platform Architecture

```
Three authoring surfaces, one registered agent
─────────────────────────────────────────────
  UI Agent Builder  ┐
  ADK CLI (YAML/JSON) ├──> POST /v1/orchestrate/agents  (CreateAgent)
  REST / SDK         ┘            │  (version 1)
                                  ▼
                        Release  POST /v1/orchestrate/agents/{id}/releases
                                  │  (version_label N)
                                  ▼
              Create Environment  POST /v1/orchestrate/agents/{agent_id}/environment
                                  │  (draft / live) → current_version
                                  ▼
              Connect Channel     POST .../environments/{env_id}/channels/{slack|teams|...}
                                  │
                                  ▼
   Conversation runtime
   ─────────────────────────────────────────────────────
   POST /v1/threads                         (CreateThread: agent_id, title)
        ▼
   POST /v1/threads/{tid}/messages           (CreateMessage: role, content)
        ▼
   POST /v1/orchestrate/runs                  (RunOrchestrate: agent_id, thread_id, message)
        │   OR  POST /v1/orchestrate/{agent_id}/chat/completions   (ChatCompletion: messages)
        ▼
   SSE stream of RunEvent{ id, event, data } until run.status = completed
        │
        ├── agent calls tools (binding: openapi|python|flow|mcp|skill|...)
        ├── agent delegates to collaborators (native/external/assistant)
        └── agent searches knowledge_base (built-in Milvus / external)
```

### Quickstart flow

The minimal end-to-end flow is: (1) authenticate (`POST /v1/auth/login` → bearer token); (2) register an agent (`POST /v1/orchestrate/agents` with `CreateAgent` — `name`, `description`, `style`, `llm`); (3) create a thread (`POST /v1/threads` with `agent_id`); (4) run the agent (`POST /v1/orchestrate/runs` with `RunOrchestrate` referencing `agent_id` + `thread_id` + a `message`, or stream via `/v1/orchestrate/runs/stream`); (5) consume `RunEvent`s and read the final `AssistantRun`. The ADK equivalent is `orchestrate agents create --name ... --kind native --llm watsonx/ibm/granite-3-8b-instruct --style default` then `orchestrate chat ask --agent-name ...`.

---

## 3. Agent Configuration & Registration

A **native agent** is the primary configurable resource. It is registered (created) once, updated by re-importing/patching, and released into environments. Configuration is expressed identically in three forms: the REST `CreateAgent`/`BaseCreateAgent` body, the ADK YAML/JSON file, and the UI Agent Builder.

### Native agent configuration fields (`CreateAgent` / `BaseCreateAgent`)

| Field | Type | Req | Description |
|-------|------|-----|-------------|
| `name` | string | – | Unique system identifier (also the CLI/UI handle). |
| `display_name` | string | – | Human-friendly name shown in UI. |
| `description` | string | **Y** | What the agent does; used by supervisor agents to route requests; informs collaborators. |
| `instructions` | string | – | Natural-language directives shaping behavior, persona, and tool/collaborator usage. |
| `llm` | string | – | LLM identifier, format `provider/developer/model_id` (e.g. `watsonx/ibm/granite-3-8b-instruct`, `groq/openai/gpt-oss-120b`). |
| `llm_config` | `ModelConfig` | – | Structured model configuration (overrides/extends `llm`). |
| `style` | `AgentStyle` | **Y** | Reasoning style: `default` \| `react` \| `planner` \| `custom` \| `react_intrinsic` \| `experimental_customer_care` \| `code_act`. |
| `tools` | array<string> | – | Tool names available to the agent. |
| `toolkits` | array<string> | – | Toolkit IDs available to the agent. |
| `collaborators` | array<string> | – | Other agent names this agent can delegate to. |
| `knowledge_base` | array<string> | – | Knowledge-base names available to the agent. |
| `guidelines` | array | – | Behavioral guidelines. |
| `glossary` | array | – | Glossary entries. |
| `structured_output` | object (JSON schema) | – | Enforces a JSON-schema structure on agent responses. |
| `custom_join_tool` | string | – | Python tool ID used for custom synthesis of task results. |
| `hidden` | boolean | – | Hide agent from listings. |
| `tags` | array | – | Tags for categorization. |
| `context_access_enabled` | boolean | – | Enable reading platform context variables. |
| `context_variables` | array<string> | – | Whitelist of context-variable names (e.g. `wxo_email_id`, `wxo_user_name`, `wxo_tenant_id`). |
| `connection_ids` | array<string> | – | Direct agent-to-connection bindings. |
| `hide_reasoning` | boolean | – | Show/hide the reasoning trace. |
| `sync_tool_flow_interactions` | boolean | – | Sync user interactions from a tool flow back to the agent. |
| `voice_configuration_id` | string | – | Voice configuration for the agent. |
| `chat_with_docs` | `ChatWithDocsConfig` | – | Per-agent chat-with-documents configuration. |
| `restrictions` | `AgentRestrictionsEnum` | – | `editable` \| `non_editable` \| `custom` (collaborator editability). |
| `bundled_agent_id` | string | – | Base/catalog agent this is derived from. |
| `workspace_id` | string | – | Owning workspace. |
| `plugins` | `Plugins` | – | Plugins and hook points. |
| `memory_enabled` | boolean | – | Enable agentic memory for this agent. |
| `is_schedulable` | boolean | – | Whether the agent can be scheduled for recurring runs. |
| `additional_properties` | `AgentAdditionalPropertiesIn` | – | Starter prompts, welcome content, icon, realtime settings, context settings. |

The ADK YAML form (`spec_version: v1`, `kind: native`) adds `hide_reasoning`, `memory_enabled`, `restrictions: editable`, `is_schedulable`, `icon` (SVG), and `compaction_settings` (context-compaction thresholds) at the top level; see `customer_support_agent` example with `compaction_settings.context_compaction_threshold`, `compaction_sliding_window`, `large_message_threshold`, etc.

### Endpoints — agent registration & management (`/v1/orchestrate/agents`)

| Method | Path | Description | Body → Response |
|--------|------|-------------|-----------------|
| GET | `/v1/orchestrate/agents` | List registered agents | → `ListAgent[]` |
| GET | `/v1/orchestrate/agents/{id}` | Get a registered agent by id | → `ListAgent` |
| PATCH | `/v1/orchestrate/agents/{id}` | Update a registered agent | `PatchAgent` → 200 |
| DELETE | `/v1/orchestrate/agents/{id}` | Delete a registered agent | → 200 |
| GET | `/v1/orchestrate/agents/{id}/tools-by-version` | Tools of an agent for a specific version | → `AgentToolVersionMinimal[]` |
| GET | `/v1/orchestrate/agents/{id}/template-status` | Template-creation status | → 200 |
| POST | `/v1/orchestrate/agents/create-from-template` | Create agent + dependencies from a template | `CreateAgentFromTemplate` → 200 |
| GET | `/v1/orchestrate/agents/{agent_id}/connections` | All connections used by the agent (and collaborators) | → 200 |
| GET | `/v1/orchestrate/agents/{agent_id}/tools/connections` | Connections from tools (sequential/parallel) | `parallel_execution` query → 200 |

`PatchAgent` mirrors `CreateAgent` and adds `timeout_seconds` (min 120, max 900) and `custom_agents_metadata` (language, framework, tool count, connection requirements) for custom (LangGraph) agents.

### V2 listing endpoints

| Method | Path | Description | Notable params |
|--------|------|-------------|----------------|
| GET | `/v2/orchestrate/agents` | List registered agents (v2) | `include_hidden` |
| GET | `/v2/orchestrate/agents/unified` | List unified agents (native + external + assistants) | – |

### Agent style semantics

| Style | When to use | Characteristics |
|-------|-------------|----------------|
| `default` | Simple/linear single-step tasks | Tool-centric; LLM decides tool, args, and when to respond |
| `react` | Multi-step analysis, complex Q&A, support tickets | ReAct: explicit reason→act→observe loop; supports knowledge/data tools + collaborators |
| `react_intrinsic` | Complex tasks on models with native chain-of-thought | ReAct without prompting overhead; requires intrinsic-reasoning models |
| `planner` | Task decomposition + synthesis | Decomposes into sub-tasks |
| `code_act` | Code-execution-oriented tasks | – |
| `experimental_customer_care` | Customer-care scenarios | – |
| `custom` | Bespoke | – |

### Descriptions & instructions guidance

Descriptions are not metadata — they **directly drive routing**. Supervisor agents read collaborator descriptions to decide delegation; tool descriptions are the most specific. Best practices: treat a description as instructions for *how the agent should use the artifact*; keep hierarchies non-overlapping; include geographic/domain context markers; assign clear unique names.

### ADK CLI authoring

```bash
orchestrate agents create \
  --name agent_name --kind native \
  --description "..." \
  --llm watsonx/ibm/granite-3-8b-instruct \
  --style default \
  --collaborators agent_1 --collaborators agent_2 \
  --tools tool_1 --output agent_name.agent.yaml

orchestrate agents import -f my_agent.yaml     # create/update
orchestrate agents export -n <name> -k native -o my_agent.zip
orchestrate agents remove --name my-agent --kind native
orchestrate chat ask --agent-name <name>        # interactive chat
```

---

## 4. Agent Releases, Environments & Deployment Lifecycle

Agents are immutable once released. A **release** snapshots the agent as `version_label` (integer) plus optional `semantic_version`/`version_name`/`version_description`. **Environments** (e.g. *draft* / *live*) point at a current version and own channels. This separates "editing the agent" from "what users talk to".

### Release endpoints (`/v1/orchestrate/agents/{id}/releases`)

| Method | Path | Description | Params |
|--------|------|-------------|--------|
| POST | `/v1/orchestrate/agents/{id}/releases` | Release an agent (new version) | `id` |
| GET | `/v1/orchestrate/agents/{id}/releases` | List all agent versions | `id` |
| GET | `/v1/orchestrate/agents/{id}/releases/status` | Deployment status | `id`, `environment_id` (query, req) |
| GET | `/v1/orchestrate/agents/{id}/releases/{version}` | Get a specific version | `id`, `version` (int) |
| DELETE | `/v1/orchestrate/agents/{id}/releases/{version}` | Delete a version | `id`, `version` |
| POST | `/v1/orchestrate/agents/{id}/releases/{version}/undeploy` | Undeploy a released version | `id`, `version` |
| POST | `/v1/orchestrate/agents/{id}/releases/{version}/environment/{env_id}` | Switch an environment to a new version | `id`, `env_id`, `version` |
| GET | `/v1/orchestrate/agents/environment/{environment_name}/releases` | List versioned agents in an environment | `environment_name`, `include_hidden`, `show_bundled` |

`AgentVersion` / `ListAgentVersion` schemas expose: `version_label`, `semantic_version`, `version_name`, `version_description`, `points_to` (environment IDs), `mapped_environments`, `deployment_status` (v1) / `deployment_statuses` (v2, per-environment), `comments`.

### Environment endpoints (`/v1/orchestrate/agents/{agent_id}/environment`)

| Method | Path | Description |
|--------|------|-------------|
| POST | `/v1/orchestrate/agents/{agent_id}/environment` | Create an environment for an agent |
| GET | `/v1/orchestrate/agents/{agent_id}/environment` | List all environments of an agent |
| GET | `/v1/orchestrate/agents/{agent_id}/environment/{environment_id}` | Get a specific environment |
| PATCH | `/v1/orchestrate/agents/{agent_id}/environment/{environment_id}` | Update an environment |
| DELETE | `/v1/orchestrate/agents/{agent_id}/environment/{environment_id}` | Delete an environment |

`CreateEnvironment` body:

| Field | Type | Req | Description |
|-------|------|-----|-------------|
| `name` | string | Y | Environment name (e.g. draft/live) |
| `description` | string | Y | Description |
| `voice` | `Voice` | – | Voice configuration |
| `current_version` | integer | – | Agent version this environment points to |
| `enable_wxg_integration` | boolean | – | Enable watsonx Governance integration |

`PatchEnvironment` is identical plus the same fields. `CreateAttachedAgentConfig` (`agent_id` + `environment_id`, both required) is used to attach agents across environments.

---

## 5. Running Agents: Runs, Threads & Messages

Conversation state is persisted as **threads** of **messages**; an agent execution over a thread is a **run**. Runs are the primary runtime API and support SSE streaming.

### Thread endpoints (`/v1/threads`)

| Method | Path | Description | Notable params |
|--------|------|-------------|----------------|
| POST | `/v1/threads` | Create message thread | `CreateThread` |
| GET | `/v1/threads` | List message threads | `limit`, `offset`, `agent_id` (filter) |
| GET | `/v1/threads/{thread_id}` | Get thread by id | → `Thread` |
| PATCH | `/v1/threads/{thread_id}` | Update thread | `PatchThread` |
| DELETE | `/v1/threads/{thread_id}` | Delete thread | – |

`CreateThread` body: `assistant_id?`, `agent_id?`, `title?`, `context?` (object).
`PatchThread` body: `assistant_id?`, `agent_id?`, `knowledge_base_id?`, `title?`, `status?` (`ThreadStatus`), `updated_at?`.
`Thread` response adds: `id`, `tenant_id`, `tenant_name`, `created_by(_username)`, `created_on`, `updated_at`, `deleted_at`, `deleted_by`, `last_read_at`, `last_agent_message_at`.

`ThreadStatus` enum: `async_wait | ready | async_slot_request | async_a2a_slot_request`.

### Message endpoints (`/v1/threads/{thread_id}/messages`)

| Method | Path | Description |
|--------|------|-------------|
| POST | `/v1/threads/{thread_id}/messages` | Create message |
| GET | `/v1/threads/{thread_id}/messages` | List messages |
| GET | `/v1/threads/{thread_id}/messages/{message_id}` | Get message by id |
| PATCH | `/v1/threads/{thread_id}/messages/{message_id}` | Update message |
| DELETE | `/v1/threads/{thread_id}/messages/{message_id}` | Delete message |

`CreateMessage` body:

| Field | Type | Req | Description |
|-------|------|-----|-------------|
| `role` | string | Y | Role (user/assistant/...) |
| `content` | array \| string | Y | Content blocks or text |
| `mentions` | array | – | Mentions |
| `document_ids` | array | – | Attached document IDs |
| `parent_message_id` | string | – | Parent message (threading/replies) |
| `additional_properties` | `AdditionalProperties` | – | – |
| `assistant_id` | string | – | Assistant id |
| `context` | `Context` | – | – |
| `step_history` | array | – | Step history (tool/agent traces) |
| `message_state` | object | – | Message state |

`Message` (response) adds: `id`, `tenant_id`, `thread_id`, `created_by(_username)`, `created_on`, `updated_at`, `assistant` (`AssistantInfo`). The simpler `CreateMessageV2` (`role` + `content`) is used by the chat-completions endpoints.

### Run endpoints (`/v1/orchestrate/runs`)

| Method | Path | Description | Notable params | Response |
|--------|------|-------------|----------------|----------|
| POST | `/v1/orchestrate/runs` | Chat with Orchestrate assistant (agent) | `stream` (query, bool), `multiple_content`, `stream_timeout` | `AssistantRunResponse` (or SSE stream when `stream=true`) |
| POST | `/v1/orchestrate/runs/stream` | Chat with Orchestrate assistant as stream | `stream_timeout`, `multiple_content` | SSE `RunEvent` |
| GET | `/v1/orchestrate/runs` | List runs | `limit`, `offset` | `PaginatedAssistantRunResponse` |
| POST | `/v1/orchestrate/runs/cancel/{run_id}` | Cancel a run | `run_id` | `AssistantRunCancelResponse` |
| GET | `/v1/orchestrate/runs/{run_id}` | Get a run | `run_id` | `AssistantRun` |
| GET | `/v1/orchestrate/runs/{run_id}/events` | Get run events | `run_id`, `stream_timeout` | 200 (event list/stream) |

`RunOrchestrate` body (the core "run an agent" payload):

| Field | Type | Req | Description |
|-------|------|-----|-------------|
| `message` | `CreateMessage` | – | The user message to send |
| `thread_id` | string | – | Existing thread to continue |
| `agent_id` | string | – | Agent to run |
| `environment_id` | string | – | Target environment (draft/live) |
| `version` | integer | – | Pin a specific agent version |
| `llm_params` | `TextGenerationParameters` | – | Generation params override |
| `guardrails` | `Guardrails` | – | Guardrail config |
| `context` | object | – | Context dictionary |
| `context_variables` | object | – | Context-variable values, e.g. `{"wxo_email_id":"user@domain.com","wxo_user_name":"John Doe"}` |
| `additional_parameters` | object | – | Extra params |

`AssistantRun` response:

| Field | Type | Req | Description |
|-------|------|-----|-------------|
| `id` | string | Y | Run id |
| `thread_id` | string | Y | Thread id |
| `assistant_id` | string | – | Assistant id |
| `agent_id` | string | – | Agent id |
| `status` | `AssistantRunStatus` | – | Run status |
| `created_at`/`started_at`/`completed_at`/`failed_at`/`cancelled_at` | string | – | Lifecycle timestamps |
| `llm_params` | `TextGenerationParameters` | – | – |
| `usage` | object | – | Token usage |
| `last_error` | string | – | Last error |
| `result` | object | – | Run result |
| `guardrails` | `Guardrails` | – | – |
| `additional_parameters` | object | – | – |
| `trace_id` | string | – | Trace id (observability) |

`AssistantRunStatus` enum: `pending | running | completed | async_wait | async_completed | failed | cancelled | requires_input | expired`.
`AssistantRunResponse` (sync start): `thread_id`, `run_id`, `task_id`, `message_id`.
`RunEvent` (SSE): `{ id: string, event: EventType, data: object }` — the streaming envelope emitted during a run.

### Per-agent chat completions

| Method | Path | Description | Body | Stream |
|--------|------|-------------|------|--------|
| POST | `/v1/orchestrate/{agent_id}/chat/completions` | Chat with a specific agent (OpenAI-style) | `ChatCompletion` | optional (`stream` field) |

`ChatCompletion` body: `messages` (array<`CreateMessageV2`>, req), `additional_parameters` (object), `context` (object), `stream` (boolean).

---

## 6. Chat Completions & Model Gateway

watsonx Orchestrate provides an OpenAI-compatible surface both for running *agents* (above) and for raw model access through a **gateway**.

### Model gateway endpoints

| Method | Path | Description | Notable params |
|--------|------|-------------|----------------|
| POST | `/v1/orchestrate/gateway/model/chat/completions` | Pure passthrough to chat completions | `llm_as_judge` (query, bool) |
| POST | `/v1/orchestrate/gateway/model/embeddings` | Pure passthrough to embeddings | – |

### Standalone completions (model-level, no agent)

These mirror OpenAI's completion endpoints and let callers invoke models directly:

`ChatCompletionRequest`:

| Field | Type | Req | Description |
|-------|------|-----|-------------|
| `model` | string | Y | Model id |
| `messages` | array<`ChatMessage`> | Y | Messages (`ChatMessage`: `role`, `content`, `name?`) |
| `frequency_penalty` / `presence_penalty` | number | – | Sampling penalties |
| `logit_bias` | object | – | Logit bias |
| `temperature` | number | – | Temperature |
| `top_p` | number | – | Top-p |
| `max_tokens` | integer | – | Max tokens |
| `n` / `seed` | integer | – | Samples / seed |
| `stop` | array | – | Stop sequences |
| `stream` | boolean | – | Stream |
| `tools` | array | – | Tools |
| `tool_choice` | string | – | Tool choice |
| `user` | string | – | User id |
| `extra_model_args` | object | – | Extra model args |

`ChatCompletionResponse`: `id`, `object`, `created`, `model`, `choices` (array), `thread_id`.
`CompletionRequest` (text completion): `model`, `prompt` (req), plus `logprobs`, `best_of`, `suffix`, and the same sampling fields. `CompletionResponse`: `content`, `response_metadata`.

The gateway endpoints and the `/completions`/`/completions/chat` paths make Orchestrate a drop-in replacement for OpenAI-style clients, enabling integration of agents into IDEs and third-party systems.

---

## 7. External Agents & Interoperability (ACF / A2A / watsonx Assistants)

watsonx Orchestrate treats foreign agents as first-class collaborators. Three integration modes exist:

1. **External chat-completions agents (ACF)** — any agent exposing an OpenAI-compatible `/chat/completions` endpoint. Registered, then referenced as a collaborator.
2. **A2A agents** — agents speaking the Agent-2-Agent protocol (`provider/protocol/version`, e.g. `external_chat/A2A/0.3.0`).
3. **watsonx Assistants** — registered watsonx Assistant instances, usable as collaborators.

### External-chat agent endpoints (`/v1/agents/external-chat`)

| Method | Path | Description | Body / Response |
|--------|------|-------------|-----------------|
| POST | `/v1/agents/external-chat` | Register an external chat-completions agent | `CreateExternalChatAgent` → 200 |
| GET | `/v1/agents/external-chat` | List registered external agents | → `ExternalChatAgent[]` |
| GET | `/v1/agents/external-chat/{id}` | Get an external agent by id | → `ExternalChatAgent` |
| PATCH | `/v1/agents/external-chat/{id}` | Update an external agent | `PatchExternalChatAgent` → 200 |
| DELETE | `/v1/agents/external-chat/{id}` | Delete an external agent | → 200 |

`CreateExternalChatAgent` body:

| Field | Type | Req | Description |
|-------|------|-----|-------------|
| `name` | string | – | Unique name |
| `title` | string | – | Title |
| `description` | string | – | Description (used for routing) |
| `tags` | array | – | Tags |
| `api_url` | string | **Y** | URL of the external agent's `/chat/completions` |
| `auth_scheme` | `ExternalAgentAuthScheme` | – | `BEARER_TOKEN` \| `API_KEY` \| `NONE` |
| `auth_config` | object | – | e.g. `{ "token": "<key>" }` |
| `chat_params` | object | – | Per-request params (e.g. `{ "stream": true }`) |
| `instructions` | string | – | Instructions forwarded to the external agent |
| `config` | `ExternalChatAgentConfig` | – | `{ hidden: bool, enable_cot: bool }` |
| `nickname` | string | – | Nickname |
| `connection_id` | string | – | Connection id |
| `provider` | string | – | Provider identifier (e.g. `external_chat/A2A/0.3.0` for A2A, `wx.ai` for watsonx.ai) |
| `context_access_enabled` | boolean | – | – |
| `context_variables` | array<string> | – | – |

`ExternalChatAgent` (response) adds: `id`, `tenant_id`, `type`, `hidden`, `enable_cot`.
`ExternalChatAgentConfig`: `hidden` (bool), `enable_cot` (bool — whether the external agent returns internal steps/tool calls).
`ExternalAgentAuthScheme` enum: `BEARER_TOKEN | API_KEY | NONE`.

The ADK YAML for an external agent uses `spec_version: v1`, `kind: external`, with `provider`, `api_url`, `auth_scheme`, `auth_config.token`, `chat_params.stream`, and `config.hidden`/`config.enable_cot`.

### A2A protocol

| Method | Path | Description |
|--------|------|-------------|
| GET | `/v1/a2a/versions` | Get supported A2A protocol versions (for a role: client or server) |

### watsonx Assistant endpoints (`/v1/assistants/watsonx`)

| Method | Path | Description |
|--------|------|-------------|
| GET | `/v1/assistants/watsonx` | List registered watsonx Assistant instances |
| GET | `/v1/assistants/watsonx/{assistant_id}` | Get a watsonx Assistant instance |
| PATCH | `/v1/assistants/watsonx/{assistant_id}` | Update a watsonx Assistant instance |
| DELETE | `/v1/assistants/watsonx/{assistant_id}` | Delete a watsonx Assistant instance |
| GET | `/v1/assistants/watsonx/wxassistant/listfromwxa` | Fetch assistants available in a WxA instance |

The ADK `kind: assistant` form (`my_assistant`) uses `config.assistant_id`, `config.environment_id`, `config.api_key`, `config.service_instance_url`, `config.api_version` (e.g. `2023-06-15`), `config.auth_type` (e.g. `MCSP`), `config.authorization_url`.

### Custom assistants (`/v1/assistants/custom`)

| Method | Path | Description |
|--------|------|-------------|
| GET | `/v1/assistants/custom` | List custom assistants |
| POST | `/v1/assistants/custom` | Create custom assistant |
| GET | `/v1/assistants/custom/{assistant_id}` | Get custom assistant |
| PATCH | `/v1/assistants/custom/{assistant_id}` | Update custom assistant |
| DELETE | `/v1/assistants/custom/{assistant_id}` | Delete custom assistant |
| POST | `/v1/assistants/custom/{assistant_id}/files` | Upload file to a custom assistant |
| GET | `/v1/assistants/custom/{assistant_id}/files` | List custom-assistant files |
| DELETE | `/v1/assistants/custom/{assistant_id}/files/{document_id}` | Delete a file |

### Assistant runs (`/v1/assistants/{assistant_id}/runs`)

| Method | Path | Description | Body |
|--------|------|-------------|------|
| POST | `/v1/assistants/{assistant_id}/runs` | Run assistant | `RunOrchestrate` |
| GET | `/v1/assistants/{assistant_id}/runs` | List assistant runs | – |
| GET | `/v1/assistants/{assistant_id}/runs/{run_id}` | Get assistant run by id | – |

---

## 8. Tools & Toolkits

**Tools** are the action layer. A tool is defined by its name, description, permission, JSON-schema input/output, and a **binding** that selects one of nine implementation types. **Toolkits** bundle tools for import/remove as a unit and enable performance optimizations (Python toolkits run tools in a single persistent process; MCP toolkits group an MCP server's tools).

### Tool endpoints (`/v1/tools`)

| Method | Path | Description | Notable params / body |
|--------|------|-------------|------------------------|
| POST | `/v1/tools` | Create a tool | `CreateTool` |
| GET | `/v1/tools` | Fetch tools | `ids`, `names`, `show_bundled` |
| PUT | `/v1/tools/{id}` | Patch (update) a tool | `CreateTool` |
| GET | `/v1/tools/{id}` | Get tool by id | → `ToolOut` |
| DELETE | `/v1/tools/{id}` | Delete a tool | – |
| POST | `/v1/tools/{id}/upload` | Upload supporting artifact | binary `file` |
| GET | `/v1/tools/{id}/download` | Download supporting artifact | – |
| POST | `/v1/tools/create-from-template` | Create tool from template | `CreateToolFromTemplate` |
| POST | `/v1/tools/{tenant_id}/callback/{correlation_id}` | Post async-tool response | – |
| POST | `/v1/tools/{tenant_id}/callback/{original_correlation_id}/correlation_objects` | Create sub-correlation object | `SearchCorrelationObject` |
| GET | `/v1/tools` (v2) | Fetch tools (v2) | `tools-v2` |

`CreateTool` body:

| Field | Type | Req | Description |
|-------|------|-----|-------------|
| `name` | string | – | Tool name |
| `display_name` | string | – | Display name |
| `description` | string | **Y** | What the tool does (drives LLM selection) |
| `permission` | `ToolPermission` | **Y** | `read_only` \| `write_only` \| `read_write` \| `admin` |
| `input_schema` | `ToolRequestBody` | – | JSON schema (object/string, properties, required) |
| `output_schema` | `ToolResponseBody` | – | JSON schema (type/description/properties/items/required/format/anyOf/oneOf/allOf) |
| `binding` | `ToolBinding` | – | Implementation selector (see below) |
| `tags` | array | – | Tags |
| `is_async` | boolean | – | Whether execution is asynchronous (uses `callbackUrl`) |
| `restrictions` | string | – | Editability |
| `bundled_agent_id` | string | – | Base agent id |

`ToolPermission` enum: `read_only | write_only | read_write | admin`.

### Tool bindings (`ToolBinding`)

A `ToolBinding` is a discriminated object with one of these keys set:

| Binding | Schema | Key fields |
|---------|--------|------------|
| `openapi` | `OpenApiToolBinding` | `http_method` (GET/POST/PUT/PATCH/DELETE, req), `http_path` (req), `security`, `servers`, `connection_id`, `callback`, `acknowledgement` |
| `python` | `PythonToolBinding` | `function` (fully-qualified name, req), `requirements`, `connections` |
| `flow` | `FlowToolBinding` | `flow_id`, `version`, `dependencies` (`{tools, agents}`), `http_method` (GET/POST/PUT), `http_path`, `connection_id`, `callback`, `acknowledgement`, `model` |
| `mcp` | `MCPToolBinding` | `server_url`, `source` (`public-registry`/`files`), `env`, `command`, `args`, `transport`, `connections` |
| `skill` | `SkillToolBinding` | `skillset_id`, `skill_id`, `skill_operation_path`, `http_method` (all req) |
| `client_side` | `ClientSideToolBinding` | (client-executed tool) |
| `conversational_search` | `ConversationalSearchToolBinding` | (knowledge conversational search) |
| `langflow` | `LangflowToolBinding` | `langflow_id`, `project_id`, `langflow_version`, `connections` |
| `wxflows` | `WxFlowsToolBinding` | `endpoint` (req), `flow_name` (req), `security` |

`ToolOut` (response) adds: `id`, `tenant_id`, `uid`, `collab_idf`, `environment_id`, `toolkit_id`, `is_async`, `restrictions`, `bundled_agent_id`, plus created/updated audit fields.

### Toolkit endpoints (`/v1/toolkits`)

| Method | Path | Description | Body / params |
|--------|------|-------------|---------------|
| POST | `/v1/toolkits` | Create a toolkit | `CreateToolKit` |
| GET | `/v1/toolkits` | Fetch toolkits | `ids`, `names` |
| GET | `/v1/toolkits/{id}` | Get toolkit by id | → `ToolKitOut` |
| PATCH | `/v1/toolkits/{id}` | Patch a toolkit | `PatchToolKit` |
| DELETE | `/v1/toolkits/{id}` | Delete a toolkit | – |
| POST | `/v1/toolkits/prepare/list-tools` | List tools from an MCP server without importing | `list_toolkit_obj`, `file` |
| POST | `/v1/toolkits/{id}/upload` | Upload supporting artifact | binary `file` |
| GET | `/v1/toolkits/{id}/download` | Download supporting artifact | – |

`CreateToolKit` body: `name` (req), `description` (req), `mcp` (`MCPToolKitConfig`).

`MCPToolKitConfig`:

| Field | Type | Description |
|-------|------|-------------|
| `source` | enum `files` / `public-registry` | Source of the MCP |
| `server_url` | string | URL of the remote MCP server |
| `transport` | string | Transport protocol for remote MCP |
| `command` | string | Command to start the MCP server |
| `args` | array | Startup arguments |
| `env` | object | Environment variables |
| `tools` | array<string> | Tools to import (`['*']` for all) |
| `connections` | object | Credential connections |

`ToolKitOut` adds: `id`, `tenant_id`, `permission`, `input_schema`, `output_schema`, `binding`, `mcp`, `uid`, `collab_idf`, `tools[]`.

### Tool concepts

- **OpenAPI tools** — consume remote web services via an OpenAPI spec. Async tools require a `callbackUrl` header parameter; the only supported callback content type is `application/json`. Default OpenAPI specs often perform poorly as agent tools (designed for developers, not agents).
- **Python tools** — `@tool`-decorated functions running in isolated containers with their own virtualenv; support multi-file packages and `requirements.txt`. `PythonToolBinding.function` is the fully-qualified function name.
- **Agentic workflow tools (Flows)** — a flow exposed as a tool (see §9).
- **MCP tools** — Model Context Protocol tools, local (stdio, Python/Node via npx/uvx) or remote (HTTP SSE / streamable HTTP). Imported via toolkits.
- **Skill tools** — call a skillset/skill operation by id + path + method.
- **Async tools** — return immediately; post results to `/v1/tools/{tenant_id}/callback/{correlation_id}` with sub-correlation objects.

---

## 9. Agentic Workflows (Flows)

An **agentic workflow** is a directed graph of nodes that acts *as a tool* with built-in agentic capabilities. Built in Python with the `@flow` decorator, imported as a `flow`-kind tool, and run either directly or as a tool invoked by an agent.

### Flow endpoints (`/v1/orchestrate/flows`)

| Method | Path | Description | Notable params |
|--------|------|-------------|----------------|
| POST | `/v1/orchestrate/flows/{flow_id}/run` | Run the latest published flow version | headers: `x-ibm-thread-id`, `x-ibm-environment-id`, `x-ibm-agent-id`, `x-ibm-agent-version`, `x-ibm-flow-execution-summary`; query: `request_timeout` |
| POST | `/v1/orchestrate/flows/{flow_id}/run/async` | Asynchronously run the latest flow version | headers: `x-ibm-wxo-thread-id`, `x-ibm-flow-instance-id`, `callbackUrl` |
| GET | `/v1/orchestrate/flows/` | List flow instances | `flow_id`, `version`, `state` (`completed`/`in_progress`/`interrupted`/`failed`), `instance_id`, `root_only`, `initiators`, `page`, `page_size` |

### Flow nodes

The flow graph composes these node types (from the ADK flows docs):

- **Tool node** — invoke a tool.
- **Agent node** — invoke a registered agent.
- **Generative prompt node** — LLM prompt step.
- **Branch node** — conditional routing (Python `evaluator` expression).
- **Parallel branch node** — concurrent branches.
- **Foreach node** — iterate over a collection.
- **Loop** — repeat until a condition holds.
- **Decisions node** — rule-based decisions.
- **Timer node** — wait/delay.
- **User activity node** — pause for user input (not allowed in callback flows).
- **Document classifier / field extractor / text extractor** (public preview) — document processing.
- **Data map** — map inputs/outputs between nodes.
- **Error handling / masking sensitive data** — control-flow and security nodes.

`FlowToolBinding` exposes `flow_id`, `version`, `dependencies` (`FlowDependencies`: `tools[]`, `agents[]`), plus OpenAPI-style `http_method`/`http_path`/`servers`/`connection_id` and `callback`/`acknowledgement` for async flows.

### Flow concepts

- **Callbacks** support four tool types (OpenAPI, Python, Flow, MCP), executed fire-and-forget. OpenAPI is the recommended callback mechanism. Flow callbacks must **not** contain user-activity nodes (correlation-ID propagation complexity).
- **Multi-language support** — target locales, translations.
- **Multi-user activities** — configure concurrent user interactions.
- **MCP workflows** (public preview) — run agentic workflows with MCP.
- **Expressions** — agentic-workflow expression language for branch/loop conditions.
- **Scheduling** — set `schedulable=True` in `@flow` to enable recurring runs.
- The `@flow` decorator parameters: `name`, `display_name`, `description`, `input_schema`, `output_schema`, `initiators`, `schedulable`, `llm_model`, `agent_conversation_memory_turns_limit`.

### Langflow & wxflows

`LangflowToolBinding` (`langflow_id`, `project_id`, `langflow_version`, `connections`) lets you import visually-built Langflow flows as tools. `WxFlowsToolBinding` (`endpoint`, `flow_name`, `security`) integrates watsonx flows.

---

## 10. Knowledge Bases, Document Collections & Vector Indices

Knowledge lets agents ground answers in domain content. Two layers exist: the high-level **knowledge base** (managed by `/v1/orchestrate/knowledge-bases`) and the lower-level **document collections** + **vector indices** (`/v1/document-collections`, `/v1/vector-indices`).

### Knowledge-base endpoints (`/v1/orchestrate/knowledge-bases`)

| Method | Path | Description | Notable params |
|--------|------|-------------|----------------|
| POST | `/v1/orchestrate/knowledge-bases/documents` | Create a KB by uploading documents or providing an external vector index | `knowledge_base` (string name, req), `file_urls`, `files` |
| PUT | `/v1/orchestrate/knowledge-bases/{kb_id}/documents` | Ingest additional documents | `file_urls`, `files` |
| PATCH | `/v1/orchestrate/knowledge-bases/{kb_id}/documents` | Patch a KB (replace docs / external index) | `knowledge_base`, `file_urls`, `files` |
| DELETE | `/v1/orchestrate/knowledge-bases/{id}/documents` | Delete some ingested documents | – |
| DELETE | `/v1/orchestrate/knowledge-bases/{id}` | Delete a KB and its documents | – |
| GET | `/v1/orchestrate/knowledge-bases/{id}` | Get a KB by id | → `ListKnowledgeBase` |
| GET | `/v1/orchestrate/knowledge-bases/{kb_id}/status` | KB status | – |
| GET | `/v1/orchestrate/knowledge-bases` | Fetch KBs | `names`, `ids`, `query`, `limit`, `offset`, `sort` |

`ListKnowledgeBase`:

| Field | Type | Req | Description |
|-------|------|-----|-------------|
| `id` | string | Y | KB id |
| `name` / `display_name` / `description` | string | Y | – |
| `prioritize_built_in_index` | boolean | Y | Prefer built-in index when both built-in and external are provided |
| `representation` | `KnowledgeBaseRepresentation` | Y | `auto` \| `tool` |
| `vector_index` | `VectorIndexOut` | Y | Built-in index config |
| `conversational_search_tool` | `ConversationalSearchConfig` | Y | External search config |

`KnowledgeBaseRepresentation` enum: `auto | tool` — whether the KB is surfaced as an automatic retrieval augmentation or as an explicit tool.

### Chat with documents (per-agent/thread)

| Method | Path | Description | Body |
|--------|------|-------------|------|
| POST | `/v1/orchestrate/agents/{agent_id}/threads/{thread_id}/chat-with-docs` | Create a transient KB from text/files for a thread | `text`, `files` |
| GET | `/v1/orchestrate/agents/{agent_id}/threads/{thread_id}/chat-with-docs/status` | Status of the thread's chat-with-docs KB | – |

### Document collections (`/v1/document-collections`)

| Method | Path | Description |
|--------|------|-------------|
| POST | `/v1/document-collections` | Create a document collection |
| GET | `/v1/document-collections` | List collections |
| GET | `/v1/document-collections/{collection_id}` | Get a collection |
| PATCH | `/v1/document-collections/{collection_id}` | Update a collection |
| DELETE | `/v1/document-collections/{collection_id}` | Delete a collection |

`CreateDocumentCollection`: `name` (req), `title`, `description`, `tags`.

### Documents (`/v1/documents`)

| Method | Path | Description |
|--------|------|-------------|
| POST | `/v1/documents` | Create a document |
| GET | `/v1/documents` | List documents |
| GET | `/v1/documents/{doc_id}` | Get a document |
| PATCH | `/v1/documents/{doc_id}` | Update a document |
| DELETE | `/v1/documents/{doc_id}` | Delete a document |
| POST | `/v1/documents/{doc_id}/upload` | Upload document content (binary `file`) |
| GET | `/v1/documents/{doc_id}/download` | Download document content |

`CreateDocument`: `name` (req), `collection_id` (req), `title`, `description`, `tags`, `meta`, `source_uri`, `source_date`, `source_name`, `source_title`, `text`, `content_type`.

### Vector indices (`/v1/vector-indices`)

| Method | Path | Description |
|--------|------|-------------|
| POST | `/v1/vector-indices` | Create a vector index |
| GET | `/v1/vector-indices` | List vector indices |
| GET | `/v1/vector-indices/{index_id}` | Get a vector index |
| PATCH | `/v1/vector-indices/{index_id}` | Update a vector index |
| DELETE | `/v1/vector-indices/{index_id}` | Delete a vector index |
| POST | `/v1/vector-indices/{index_id}/collections` | Attach collections to an index |
| POST | `/v1/vector-indices/{index_id}/refresh` | Refresh an index |
| POST | `/v1/vector-indices/{index_id}/rebuild` | Rebuild an index |
| GET | `/v1/vector-indices/{index_id}/retrieve` | Retrieve from an index |

`CreateVectorIndex`: `name` (req), `embeddings_model_name` (req), `chunk_size`, `chunk_overlap`, `top_k`, `extraction_strategy` (`ExtractionStrategy`), `title`, `description`, `tags`.
`VectorIndex` adds: `id`, `tenant_id`, `status` (`VectorIndexStatus`), `status_msg`.
`VectorIndexStatus` enum: `ready | not_ready | rebuilding | error | update_pending`.

### Knowledge concepts

- **Built-in KB** — managed Milvus instance; you upload documents, choose an `embeddings_model_name` (e.g. `ibm/slate-125m-english-rtrvr-v2`), set chunking (`chunk_size`, `chunk_overlap`) and `top_k`.
- **External KB** — your own Milvus/Elasticsearch, or a `custom_search` URL with `filter` and `metadata`; exposed via `conversational_search_tool.index_config[]` (supports `astrodb`, `custom_search`, etc.).
- **prioritize_built_in_index** — when both a built-in index and external index exist, choose which wins.
- The ADK `kind: knowledge_base` YAML mirrors this with `spec_version: v1`, `name`, `description`, `documents[]`, `vector_index.embeddings_model_name`, optional `prioritize_built_in_index`, and `conversational_search_tool.index_config[]` (with `astradb`/`custom_search` entries supporting `embedding_mode: server|client`, `search_mode: vector|lexical|hybrid`).

---

## 11. Connections & Credentials

Connections bind credentials to tools/agents so they can call external services. Registered once, referenced by `connection_id`.

### Connection endpoints (`/v1/connections/applications`)

| Method | Path | Description |
|--------|------|-------------|
| GET | `/v1/connections/applications` | Get connections (by app id, shared flag) |
| POST | `/v1/connections/applications` | Create connection (validation, duplicate check, OAuth2 processing) |
| DELETE | `/v1/connections/applications/{connection_id}` | Delete connection |
| GET | `/v1/connections/callback` | OAuth callback |
| GET | `/v1/connections/applications/list` | Get list of connections (by app id) |

### Outbound policies (`/v1/outbound-policies`)

| Method | Path | Description |
|--------|------|-------------|
| POST | `/v1/outbound-policies` | Create policies |
| GET | `/v1/outbound-policies` | Fetch outbound policy for tenant |
| PUT | `/v1/outbound-policies` | Update policies by ids |
| DELETE | `/v1/outbound-policies` | Delete policies by ids |
| POST | `/v1/outbound-policies/check` | Check if a URL/set of URLs is allowed |

### Connection concepts

- Connections support OAuth2 (with callback flow), API key, and basic auth.
- `connection_id` is referenced from `OpenApiToolBinding`, `PythonToolBinding.connections`, `MCPToolBinding.connections`, `FlowToolBinding`, `LangflowToolBinding.connections`, and directly from agents via `connection_ids[]`.
- OpenAPI connections can have at most one `app_id` associated.
- Global variables can be bound to connections (Langflow / agentic workflows).
- Customer-care platform connections are managed separately.

---

## 12. Models & Model Policies

### Model endpoints (`/v1/models`)

| Method | Path | Description |
|--------|------|-------------|
| POST | `/v1/models` | Create a model |
| GET | `/v1/models` | Fetch models |
| GET | `/v1/models/list` | List models |
| GET | `/v1/models/list/embeddings` | List embeddings models |
| PUT | `/v1/models/{model_id}` | Update a model |
| DELETE | `/v1/models/{model_id}` | Delete a model by id |

### Model policy endpoints (`/v1/model_policy`)

| Method | Path | Description |
|--------|------|-------------|
| POST | `/v1/model_policy` | Create a model policy |
| GET | `/v1/model_policy` | Fetch models policy |
| PUT | `/v1/model_policy/{model_policy_id}` | Update a model policy |
| DELETE | `/v1/model_policy/{model_policy_id}` | Delete a model policy by id |

### Model concepts

- **Model IDs** follow `provider/developer/model_id`, e.g. `watsonx/ibm/granite-3-8b-instruct`, `watsonx/meta-llama/llama-3-3-70b-instruct`, `groq/openai/gpt-oss-120b`. The `watsonx/` prefix denotes models supported by watsonx Orchestrate.
- **Virtual models** — managed aliases/configurations over base models.
- **Model policies** — governance controls over which models can be used (public preview).
- **Embeddings** — used by knowledge bases (`embeddings_model_name`, e.g. `ibm/slate-125m-english-rtrvr-v2`).
- **LLM observability** — Langfuse or IBM telemetry integration in Developer Edition.

---

## 13. Channels & Embedded Chat

Channels deliver an agent to end users in a given environment. Every channel binds to a `(agent_id, environment_id)` pair.

### Agent environment channels (`/v1/orchestrate/agents/{agent_id}/environments/{environment_id}/channels`)

A uniform CRUD pattern per channel type (36 operations total):

| Channel type | Operations |
|--------------|------------|
| Genesys Bot Connector | create / list / update / get / delete |
| Microsoft Teams | create / list / update / get / delete |
| Twilio WhatsApp | create / list / update / get / delete |
| Twilio SMS | create / list / update / get / delete |
| Facebook Messenger | create / list / update / get / delete |
| Genesys Audio Connector | create / list / update / get / delete |
| Slack | create / list / update / get / delete |
| (All) | GET `/channels` — list all channels for the environment |

### Phone channel (`/v1/channels/phone`)

| Method | Path | Description |
|--------|------|-------------|
| POST | `/v1/channels/phone` | Create phone config |
| GET | `/v1/channels/phone` | List phone configs |
| PATCH | `/v1/channels/phone/{config_id}` | Update phone config |
| GET | `/v1/channels/phone/{config_id}` | Get phone config |
| DELETE | `/v1/channels/phone/{config_id}` | Delete phone config |
| POST | `/v1/channels/phone/{config_id}/numbers` | Add phone numbers |
| GET | `/v1/channels/phone/{config_id}/numbers` | List phone numbers |
| PATCH | `/v1/channels/phone/{config_id}/numbers/{number}` | Patch phone number |
| DELETE | `/v1/channels/phone/{config_id}/numbers/{number}` | Delete phone number |

### Channel integration / Slack app instances

`/v1/channel-integration/...` provides generic app registration plus Slack app-instance CRUD (`Register App`, `List App`, `Register Slack App Instance`, etc.).

### Embedded chat config

| Method | Path | Description | Body |
|--------|------|-------------|------|
| PUT | `/v1/orchestrate/agents/{id}/embedded-chat-config` | Create/update embedded chat config | `EmbeddedChatConfigIn` |
| GET | `/v1/orchestrate/agents/{id}/embedded-chat-config` | Get embedded chat config | – |

`EmbeddedChatConfigIn`: `layout` (`LayoutConfig`), `is_live` (boolean). The web chat SDK (`webchat/`) provides a JS embed with events (`pre:send`, `pre:receive`, `chat:ready`, `feedback`, `view:change`, etc.) and instance methods (`send()`, `restartConversation()`, `loadThreadById()`, `updateAuthToken()`, …).

### Embed settings

| Method | Path | Description |
|--------|------|-------------|
| POST | `/v1/embed-settings/config` | Create embed config |
| GET | `/v1/embed-settings/config` | Get embed config |
| POST | `/v1/embed-settings/generate-key-pair` | Generate key pair |

---

## 14. Chat Starter Settings, Voice & Agent UX

Per-agent UX configuration: welcome message, starter prompts, icon, and voice.

### Chat starter settings (`/v1/orchestrate/agents/{id}/chat-starter-settings`)

| Method | Path | Description | Notable params |
|--------|------|-------------|----------------|
| PUT | `/v1/orchestrate/agents/{id}/chat-starter-settings` | Create/update chat starter settings | `AgentChatStarterSettingsIn` |
| GET | `/v1/orchestrate/agents/{id}/chat-starter-settings` | Get chat starter settings | `environment_id`, `version_id` (query) |
| DELETE | `/v1/orchestrate/agents/{id}/chat-starter-settings` | Delete settings | `setting_type` (req: `starter_prompts` \| `welcome_content` \| `icon` \| `all`), `prompt_id` |
| PUT | `/v1/orchestrate/agents/{id}/chat-starter-settings/icon` | Upload icon (SVG/binary) | binary `file` |

`AgentChatStarterSettingsIn`:
- `starter_prompts` → `AgentCustomizedPromptsIn` (`customize`: array of `AgentPrompt`).
- `welcome_content` → `AgentWelcomeContentIn` (`welcome_message`, `description`, `is_default_voice_greeting`).
- `icon` → `AgentSvgIcon` (SVG string).

`AgentChatStarterSettingsOut` adds audit fields (`id`, `tenant_id`, `created_by`, `created_on`, `updated_at`, `updated_by`).

### Voice configurations (`/v1/voice-configurations`)

| Method | Path | Description |
|--------|------|-------------|
| POST | `/v1/voice-configurations` | Create voice configuration |
| GET | `/v1/voice-configurations` | List voice configurations |
| GET | `/v1/voice-configurations/{id}` | Get voice configuration |
| PATCH | `/v1/voice-configurations/{id}` | Update voice configuration |
| DELETE | `/v1/voice-configurations/{id}` | Delete voice configuration |

Voice is referenced from agents via `voice_configuration_id` and from environments via `voice`. `AgentIdleHandler`/`AgentIdleHandlerMessages` configure on-hold behaviour (`pre_hold_message`, `hold_message`, `typing_enabled`, `typing_duration_seconds`, `audio_clip_id`, `hold_audio_seconds`). `RealtimeAgentSettings` (in `additional_properties`) configure realtime/voice agents.

### Agent additional properties

`AgentAdditionalPropertiesIn` bundles: `starter_prompts`, `welcome_content`, `icon`, `realtime_agent_settings` (`RealtimeAgentSettingsIn`), `context_settings` (`ContextSettings` — context-management/compaction controls).

---

## 15. Agent Evaluation & Testing

watsonx Orchestrate provides a test-case + evaluation pipeline per agent: upload CSV test cases, run evaluations, list/export results, and bulk-delete.

### Test case & evaluation endpoints (`/v1/orchestrate/agent/{agent_id}`)

| Method | Path | Description | Notable params |
|--------|------|-------------|----------------|
| GET | `/v1/orchestrate/agent/test_case/templates` | Get a sample test-case CSV | – |
| POST | `/v1/orchestrate/agent/{agent_id}/test_case` | Upload test cases (CSV binary) | binary `file` → `UploadTestCasesResponse` |
| GET | `/v1/orchestrate/agent/{agent_id}/test_case` | List test cases | `query`, `limit`, `offset`, `sortkey`, `sort` |
| POST | `/v1/orchestrate/agent/{agent_id}/test_case/bulk_delete` | Bulk delete test cases | – |
| POST | `/v1/orchestrate/agent/{agent_id}/evaluate` | Evaluate test cases for an agent | – → `EvaluateTestcasesResponse` |
| GET | `/v1/orchestrate/agent/{agent_id}/evaluations` | List evaluations | `limit`, `offset`, `sortkey`, `sort` |
| POST | `/v1/orchestrate/agent/{agent_id}/evaluations/bulk_delete` | Bulk delete evaluations | – |
| POST | `/v1/orchestrate/agent/{agent_id}/evaluations/export` | Export evaluations | `ExportEvaluationRequest` (`evaluation_ids[]`) |
| GET | `/v1/orchestrate/agent/{agent_id}/evaluations/status` | Current running evaluation | – |
| GET | `/v1/orchestrate/agent/{agent_id}/evaluations/{evaluation_id}` | Get evaluation details | – |
| GET | `/v1/orchestrate/agent/{agent_id}/evaluations/{evaluation_id}/test_cases` | Get test cases of an evaluation | `status` filter, `limit`, `offset`, `sortkey`, `sort` |

### Evaluation concepts (ADK `evaluate/`)

- **Analyzing agents and tools** — inspect behavior before evaluating.
- **Creating an evaluation dataset** — author test cases.
- **Quick evaluation** — fast, lightweight runs.
- **Rubric evaluations** — score against defined rubrics.
- **LLM agent vulnerability testing** — adversarial/red-team testing.
- The CLI exposes `orchestrate` commands to create evaluation datasets and run rubric evaluations; the REST endpoints above back the UI/programmatic path.

---

## 16. Monitoring, Governance & Observability

### Agent monitoring (`/v1/orchestrate/agents/.../monitoring`, governance-monitoring group)

| Method | Path | Description |
|--------|------|-------------|
| GET | `…/check-monitoring-availability` | Check if monitoring is available |
| POST | `…/setup-agent-monitoring` | Setup agent monitoring (`SetupAgentMonitoringRequest`) |
| GET | `…/get-agent-monitoring-details` | Get agent monitoring details (`AgentMonitoringStatusResponse`: `monitoring_enabled`, `wxg_metrics_url`) |

### LLM analytics (`/v1/llm-analytics`)

| Method | Path | Description |
|--------|------|-------------|
| POST | `/v1/llm-analytics` | Create/update LLM analytics config |
| GET | `/v1/llm-analytics` | Get LLM analytics config |
| DELETE | `/v1/llm-analytics` | Delete LLM analytics config |

### Observability concepts

- **watsonx Governance (WXG) integration** — toggled per environment (`enable_wxg_integration`) and per agent (monitoring setup); returns a `wxg_metrics_url`.
- **Langfuse / IBM telemetry** — Developer Edition observability backends; `-l/--with-langfuse`, `-i/--with-ibm-telemetry`.
- **Traces** — ADK traces via CLI and Python; `trace_id` propagates through `AssistantRun`.
- **Agent analytics / Agentic Control Plane** — UI dashboards for alerts, incidents, and insights.
- **Outbound policies** — URL allow/deny lists (§11).

---

## 17. Agentic SDK (Runtime Integration for Custom Agents)

The **Agentic SDK** (`ibm-watsonx-orchestrate-sdk`) is a runtime integration layer for *custom agents* (e.g. LangGraph graphs) that execute *on* the platform. It is distinct from the ADK CLI: the ADK builds/imports definitions; the Agentic SDK runs inside a custom agent's process to provide platform context, chat models, memory, and context compression.

### Execution modes

| Mode | Use | Auth / context |
|------|-----|----------------|
| **Runs-on** | Agent runs inside the watsonx Orchestrate runtime | Runtime provides `execution_context` (`access_token`, `thread_id`, `api_proxy_url`); init via `Client.from_runnable_config(config)` or `Client(execution_context=...)` |
| **Runs-elsewhere** | Agent/app runs outside the runtime | Instance credentials: `Client(api_key=..., instance_url=...)`; supports CPD (username/password), HIPAA (`auth_type="mcsp_v2"`, `iam_url`), AWS production |
| **Local** | Developer Edition | `Client()` (local config) |

### Client & initialization

`Client` is the entry point. `api_proxy_url` is used as-is (the SDK does not append `/v1/orchestrate` or an instance path). `WXO_API_PROXY_URL` env var overrides `execution_context["api_proxy_url"]`. Runs-on defaults to `verify=False` unless overridden.

### Chat models (`sdk/chat_wxo`)

LangChain-compatible chat model that routes completions through watsonx Orchestrate. Factory methods:
- `from_instance_credentials(instance_url, api_key, model, **kwargs)` — standalone/runs-elsewhere.
- `from_execution_context(execution_context, model, **kwargs)` — runtime/runs-on.
- `from_session(session, model, **kwargs)` — from `AgenticSession` (runs-on).
- `from_runnable_config(config, model, **kwargs)` — from LangGraph `RunnableConfig`.

Model IDs use `provider/model-name` (e.g. `watsonx/meta-llama/llama-3-2-90b-vision-instruct`). Supports structured outputs, tool calling, and streaming.

### Memory (`sdk/memory`)

Persist user information and retrieve it during execution. Endpoints called: `/memories`, `/memories/search`. APIs:
- **Write:** `client.memory.add_messages(...)`
- **Read:** `client.memory.search(...)`, `client.memory.list(...)`, `client.memory.retrieve(...)`
- **Delete:** `client.memory.delete(...)`

Requires a runs-on agent with `memory_enabled`; package and import with `orchestrate agents import --experimental-package-root <dir>`.

### Context (`sdk/context`)

Context compression for long conversations: summarize older messages while preserving recent context to stay within token limits. `client.context.compress(messages=...)` returns compressed messages; recommended before LLM calls when token count exceeds a threshold.

### Embeddings (`sdk/wxo_embeddings`)

Embeddings interface routed through watsonx Orchestrate (for knowledge/retrieval inside custom agents).

### Custom-agent metadata

`PatchAgent.custom_agents_metadata` (`CustomAgentsMetadata`) records `language`, `framework`, `tool count`, `tool names`, `connection requirements` for custom agents so the platform can package and run them correctly.

---

## 18. Scheduling

Scheduling enables recurring agent/flow executions.

### Concepts (ADK `scheduling/`)

- Set `is_schedulable: true` on an agent or `schedulable=True` on a `@flow` to make it schedulable.
- Schedules are created through the Chat UI using natural language.
- New scheduling approach supersedes the legacy one (migration guide provided).
- Scheduling for both **agentic workflows** and **agents** is supported.

The capability is primarily UI/ADK-driven; the REST surface integrates via the agent's `is_schedulable` flag and the flow run endpoints.

---

## 19. Capability Summary & Cross-Reference

### Endpoints by capability (counts from the OpenAPI spec)

| Capability | Path prefix | # ops | Key schemas |
|------------|-------------|-------|-------------|
| Orchestrate runs (chat) | `/v1/orchestrate/runs` | 6 | `RunOrchestrate`, `AssistantRun`, `RunEvent`, `AssistantRunResponse` |
| Per-agent chat completions | `/v1/orchestrate/{agent_id}/chat/completions` | 1 | `ChatCompletion`, `CreateMessageV2` |
| Model gateway | `/v1/orchestrate/gateway/model` | 2 | `ChatCompletionRequest`/`Response`, `CompletionRequest`/`Response` |
| External-chat agents | `/v1/agents/external-chat` | 5 | `CreateExternalChatAgent`, `ExternalChatAgent`, `ExternalChatAgentConfig` |
| A2A | `/v1/a2a` | 1 | – |
| Agent registration/releases/env/tools/connections/chat-starter/embed | `/v1/orchestrate/agents` | 29 | `CreateAgent`/`BaseCreateAgent`, `PatchAgent`, `ListAgent`, `AgentVersion`, `CreateEnvironment`, `AgentChatStarterSettingsIn`, `EmbeddedChatConfigIn` |
| Agents v2 | `/v2/orchestrate/agents` | 2 | – |
| Assistants (custom + watsonx + runs) | `/v1/assistants` | 17 | `RunOrchestrate`, `AssistantRun` |
| Threads & messages | `/v1/threads` | 10 | `CreateThread`, `CreateMessage`, `Message`, `Thread`, `ThreadStatus` |
| Document collections | `/v1/document-collections` | 5 | `CreateDocumentCollection`, `DocumentCollection` |
| Documents | `/v1/documents` | 7 | `CreateDocument`, `DocumentOut` |
| Vector indices | `/v1/vector-indices` | 10 | `CreateVectorIndex`, `VectorIndex`, `VectorIndexStatus` |
| Knowledge bases | `/v1/orchestrate/knowledge-bases` | 8 | `ListKnowledgeBase`, `KnowledgeBaseRepresentation`, `KnowledgeBaseBuiltInVectorIndexConfig` |
| Chat-with-docs (per thread) | `/v1/orchestrate/agents/{agent_id}/threads/{thread_id}/chat-with-docs` | 2 | – |
| Tools v1 | `/v1/tools` | 10 | `CreateTool`, `ToolOut`, `ToolBinding`, `ToolPermission`, 9 binding schemas |
| Tools v2 | `/v1/tools` (v2) | 1 | – |
| Toolkits | `/v1/toolkits` | 8 | `CreateToolKit`, `ToolKitOut`, `MCPToolKitConfig` |
| Flows | `/v1/orchestrate/flows` | 3 | `FlowToolBinding`, `FlowDependencies` |
| Catalog | `/v1/catalog` | 3 | – |
| Connections | `/v1/connections` | 5 | – |
| Models & model policy | `/v1/models`, `/v1/model_policy` | 10 | – |
| Channels (phone) | `/v1/channels/phone` | 9 | – |
| Agent environment channels | `…/environments/{env_id}/channels` | 36 | – |
| Agent evaluation | `/v1/orchestrate/agent/{agent_id}` | 11 | `ExportEvaluationRequest` |
| Monitoring | governance-monitoring | 3 | `AgentMonitoringStatusResponse`, `SetupAgentMonitoringRequest` |

### Key enums

| Enum | Values |
|------|--------|
| `AgentType` | `external_chat`, `watsonx`, `native` |
| `AgentStyle` | `default`, `react`, `planner`, `custom`, `react_intrinsic`, `experimental_customer_care`, `code_act` |
| `AgentRestrictionsEnum` | `editable`, `non_editable`, `custom` |
| `ExternalAgentAuthScheme` | `BEARER_TOKEN`, `API_KEY`, `NONE` |
| `ToolPermission` | `read_only`, `write_only`, `read_write`, `admin` |
| `AssistantRunStatus` | `pending`, `running`, `completed`, `async_wait`, `async_completed`, `failed`, `cancelled`, `requires_input`, `expired` |
| `ThreadStatus` | `async_wait`, `ready`, `async_slot_request`, `async_a2a_slot_request` |
| `KnowledgeBaseRepresentation` | `auto`, `tool` |
| `VectorIndexStatus` | `ready`, `not_ready`, `rebuilding`, `error`, `update_pending` |

### Cross-capability reference

- **Agent ↔ Tools:** agent `tools[]` (names) + `toolkits[]` (ids); tools resolve via `ToolBinding` (9 types); connections via `connection_ids[]` or tool-level `connection_id`.
- **Agent ↔ Knowledge:** agent `knowledge_base[]` (names); KBs backed by `vector_index` (built-in) or `conversational_search_tool` (external); per-thread transient KBs via `chat-with-docs`.
- **Agent ↔ Agents (multi-agent):** `collaborators[]` references native/external/assistant agents; external via `CreateExternalChatAgent` (`api_url` + `auth_scheme`); assistants via `/v1/assistants/watsonx`; A2A via `provider: external_chat/A2A/0.3.0`.
- **Agent ↔ Flows:** flows are tools (`FlowToolBinding`); flows can themselves call agents (`Agent node`) and tools (`Tool node`), with `FlowDependencies.{tools, agents}`.
- **Agent ↔ Channels:** channels bind to `(agent_id, environment_id)`; environments point at a released `version`; releases are immutable snapshots of `CreateAgent`.
- **Agent ↔ Runs:** `RunOrchestrate` selects `agent_id` + `environment_id` + optional `version` + `thread_id` + `message`; emits `RunEvent` SSE stream; `AssistantRun` records status/usage/trace.
- **Agent ↔ Context:** `context_access_enabled` + `context_variables[]` whitelist platform variables (`wxo_email_id`, `wxo_user_name`, `wxo_tenant_id`); values supplied per-run via `RunOrchestrate.context_variables`; agents reference them in instructions as `{wxo_email_id}`.
- **Agent ↔ Memory:** `memory_enabled` toggles agentic memory; custom agents access it via `client.memory.*` (SDK) hitting `/memories` and `/memories/search`.
- **Agent ↔ Evaluation:** `is_schedulable` + test cases (`/test_case`) + evaluations (`/evaluate`, `/evaluations`).
- **Agent ↔ Governance:** environment `enable_wxg_integration` + agent monitoring setup → `wxg_metrics_url`; outbound policies gate URL access.

### Comparison notes (vs. hosted-sandbox agent platforms)

Unlike platforms that provision a server-side sandbox per session and stream an event-based agent loop, watsonx Orchestrate is a **control plane**: it stores registered agent definitions, versioned releases, environments, and channel bindings, and orchestrates *runs* over *threads*. Tool execution is delegated to bindings (remote OpenAPI endpoints, isolated Python containers, MCP servers, flows, skills) rather than a single managed shell. Custom LangGraph agents are supported via the Agentic SDK's runs-on mode, where the platform injects an `execution_context` into the agent's graph. External agents are integrated as collaborators through the OpenAI-compatible ACF surface rather than being re-hosted. This makes watsonx Orchestrate closer to an "agent registry + runtime orchestrator + integration hub" than a hosted agent sandbox.