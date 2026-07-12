# IBM Bob API Analysis — Agent Capabilities

> **Docs root:** `https://bob.ibm.com/docs/ide` (IDE) and `https://bob.ibm.com/docs/shell` (Bob Shell)
> **Product:** IBM Bob — an AI SDLC (Software Development Lifecycle) partner delivered as a VS Code-style IDE extension and a terminal CLI ("Bob Shell"). Bob is an agentic coding assistant, not a hosted REST platform: there is no public REST "agent API" surface in the Anthropic/OpenAI sense. Instead, agent capabilities are exposed through (1) a **CLI** (`bob`) used interactively and non-interactively for automation, (2) **API keys** for headless/inference authentication, (3) a **tool-calling runtime** (the agent loop) driven by a fixed set of built-in tools, and (4) declarative configuration files (`settings.json`, `custom_modes.yaml`, `AGENTS.md`, `.bob/commands/`, `.bob/rules-*/`, MCP server configs).
> **Auth:** API keys (General or Inference type), scoped to a subscription instance + user. General keys require a team-ID header for inference; Inference keys are pre-scoped. Keys are managed at `bob.ibm.com`.
> **Consumption unit:** Bobcoins — a unified billing metric abstracting per-model token costs. Plans: Free trial/Pro (40), Pro+ (160), Ultra (500) Bobcoins/month; Enterprise buys packs.
> **Description:** IBM Bob exposes agent capabilities primarily through a local-first agentic runtime. The "API" is the combination of the `bob` CLI (non-interactive `-p` mode for scripting/CI), the in-IDE chat agent loop, and the configuration layer that defines modes, tools, MCP servers, skills, rules, and slash commands. This document analyses each agent-related capability, its main concepts, and the functions/parameters that compose it.

---

## Table of Contents

1. [Platform Overview & Core Concepts](#1-platform-overview--core-concepts)
2. [Authentication & API Keys](#2-authentication--api-keys)
3. [Bob Shell CLI — The Automation API](#3-bob-shell-cli--the-automation-api)
4. [Non-Interactive Sessions (Headless Agent Invocations)](#4-non-interactive-sessions-headless-agent-invocations)
5. [The Agent Tool Runtime](#5-the-agent-tool-runtime)
6. [Read Tools](#6-read-tools)
7. [Write Tools](#7-write-tools)
8. [Command Tools](#8-command-tools)
9. [Subagent Tools](#9-subagent-tools)
10. [Subtask Tools](#10-subtask-tools)
11. [MCP Tools (Model Context Protocol)](#11-mcp-tools-model-context-protocol)
12. [Mode Tools](#12-mode-tools)
13. [Skill Tools](#13-skill-tools)
14. [Workflow Tools](#14-workflow-tools)
15. [Todo Tools](#15-todo-tools)
16. [Question Tools](#16-question-tools)
17. [Modes (Agent Personas)](#17-modes-agent-personas)
18. [Custom Modes](#18-custom-modes)
19. [Custom Rules & Context Files](#19-custom-rules--context-files)
20. [Slash Commands](#20-slash-commands)
21. [Context Mentions (`@`)](#21-context-mentions-)
22. [Auto-Approve / Approval Modes](#22-auto-approve--approval-modes)
23. [Checkpointing & Rollback](#23-checkpointing--rollback)
24. [Context Window Management](#24-context-window-management)
25. [Bobalytics (Agent Observability)](#25-bobalytics-agent-observability)
26. [Capability Summary & Cross-Reference](#26-capability-summary--cross-reference)

---

## 1. Platform Overview & Core Concepts

IBM Bob is built around these core abstractions:

- **Agent** — The agentic runtime that drives an autonomous tool-calling loop. Bob decides which tool to call, executes it, observes the result, and continues until the task is complete. The loop is the same in the IDE chat and in Bob Shell.
- **Mode** — A specialized persona that scopes Bob's behavior, role definition, available tool groups, and allowed subagents. Built-in modes: **Agent**, **Plan**, **Ask** (IDE); **Code**, **Ask**, **Plan**, **Advanced** (Shell). Custom modes are defined in YAML.
- **Tool** — A capability the agent can call. Tools are grouped: read, write, command, subagent, subtask, MCP, mode, skill, workflow, todo, question. Mode configuration controls which groups are available.
- **Subagent** — An independent agent spawned (`spawn_subagent`) to run a self-contained task in its own isolated context window, returning only a summary. Two types: `explore` (read-only, lighter model) and `general` (full tools, default model).
- **Subtask** — A visible, interactive named task (`start_subtask`) with its own conversation thread, breadcrumb, and optional todo list. Unlike a subagent, a subtask is tracked in the UI and can be continued.
- **Skill** — A reusable instruction set (`SKILL.md` + supporting files) loaded into context on demand via `use_skill`. Provides domain-specific workflows (e.g., Carbon Design, DITA, Jira, create-plan).
- **MCP Server** — An external Model Context Protocol server connected to Bob to extend its tools. Configured in `settings.json` under `mcpServers`; supports STDIO, Streamable HTTP, and legacy SSE transports.
- **Rule** — A custom instruction loaded into the system context, scoped globally (`~/.bob/rules/`), per-mode (`rules-agent/`, `rules-plan/`, …), or via `AGENTS.md`.
- **Slash Command** — A reusable prompt template invoked with `/`, defined as markdown files in `.bob/commands/` or `~/.bob/commands/`, with optional frontmatter (`description`, `argument-hint`).
- **Context Mention (`@`)** — A mechanism to inject file/folder/git/problems/terminal content directly into the conversation context.
- **Checkpoint** — An automatic git snapshot + conversation history + tool-call record taken before file-modifying operations (Shell), enabling `/restore`.
- **API Key** — A credential for headless/automation/inference authentication without browser sign-in. Two types: General and Inference.
- **Bobcoin** — The consumption unit billed per agent action, abstracting underlying model token costs.

### Agent Capability Map

| Capability | Description | Docs |
|------------|-------------|------|
| **Authentication (API keys)** | General & Inference keys for headless/CI/inference auth | [api-keys](https://bob.ibm.com/docs/ide/account/api-keys) |
| **Bob Shell CLI** | `bob` command with flags for interactive/non-interactive sessions | [configuring](https://bob.ibm.com/docs/shell/configuration/configuring) |
| **Non-interactive sessions** | `bob -p` headless single-prompt invocations for scripting/CI | [non-interactive](https://bob.ibm.com/docs/shell/getting-started/start-bobshell-non-interactive) |
| **Agent tool runtime** | Built-in tool-calling loop: read/write/execute/subagent/MCP/skill/mode/workflow/todo/question | [tools](https://bob.ibm.com/docs/ide/core-concepts/tools) |
| **Subagents** | `spawn_subagent` — isolated-context background agents (explore/general) | [subagents](https://bob.ibm.com/docs/ide/features/subagents) |
| **Subtasks** | `start_subtask` — visible, interactive tracked task threads | [tools](https://bob.ibm.com/docs/ide/core-concepts/tools) |
| **Modes** | Agent/Plan/Ask personas with scoped tools & subagents | [modes](https://bob.ibm.com/docs/ide/features/modes) |
| **Custom modes** | YAML-defined modes with tool groups, fileRegex, subagent presets | [custom-modes](https://bob.ibm.com/docs/ide/configuration/custom-modes) |
| **Custom rules** | Per-mode/global/`AGENTS.md` instruction injection | [rules](https://bob.ibm.com/docs/ide/configuration/rules) |
| **Skills** | `use_skill` + `SKILL.md` reusable instruction sets | [skills](https://bob.ibm.com/docs/ide/features/skills) |
| **MCP** | Connect external MCP servers (STDIO/Streamable HTTP/SSE) | [mcp-in-bob](https://bob.ibm.com/docs/ide/configuration/mcp/mcp-in-bob) |
| **Slash commands** | Reusable `/` prompt templates with argument hints | [slash-commands](https://bob.ibm.com/docs/shell/features/slash-commands) |
| **Context mentions** | `@` injection of files/folders/git/problems/terminal | [context-mentions](https://bob.ibm.com/docs/ide/features/context-mentions) |
| **Auto-approve** | Per-action approval modes (read/edit/execute/MCP/skill/todo/subtask/subagent/mode) | [auto-approve](https://bob.ibm.com/docs/ide/features/auto-approving-actions) |
| **Checkpointing** | Auto git snapshots + `/restore` (Shell) | [checkpointing](https://bob.ibm.com/docs/shell/features/checkpointing) |
| **Context window mgmt** | 270,000-token window budgeting across system/tools/rules/skills/messages | [context-window](https://bob.ibm.com/docs/ide/core-concepts/context-window-management) |
| **Bobalytics** | Enterprise analytics: adoption rate, Bob factor, Bobcoin spend | [bobalytics](https://bob.ibm.com/docs/ide/features/bobalytics) |

### Platform Architecture

```
┌─ IDE extension (VS Code-style) ──────────────┐   ┌─ Bob Shell (CLI) ───────────────┐
│  Chat interface / slash commands / @mentions │   │  bob (interactive TUI)          │
│  Modes selector / auto-approve toolbar        │   │  bob -p "..." (non-interactive)  │
└──────────────────┬───────────────────────────┘   └────────────────┬────────────────┘
                   │          Shared agent runtime                   │
                   ▼                                                ▼
          ┌───────────────────────────────────────────────────────────────┐
          │  Agent loop: decide → call tool → observe → repeat            │
          │  Mode-scoped tool groups + subagent/subtask spawning          │
          │  Approval gate (auto-approve / ask) before each tool call     │
          └───────────────────────────────────────────────────────────────┘
                   │
        ┌──────────┼───────────┬───────────────┬─────────────┬───────────┐
        ▼          ▼           ▼               ▼             ▼           ▼
   Built-in tools  MCP tools  Skill loader   Mode switcher  Subagent    Subtask
   (read/write/    (external   (use_skill)   (switch_mode)  (spawn_     (start_
    execute/...)   servers)                                  subagent)   subtask)
        │
        ▼
  Configuration layer: settings.json · custom_modes.yaml · AGENTS.md
                       .bob/commands/ · .bob/rules-*/ · mcpServers · .bobignore
        │
        ▼
  Auth: API key (General | Inference) → subscription instance + team
  Billing: Bobcoins (per-action consumption unit)
```

---

## 2. Authentication & API Keys

### Main Concepts

IBM Bob authenticates automation/inference workloads with **API keys** instead of browser-based IBMid sign-in. Keys are owned by a user and scoped to a subscription instance.

- **General key** — Scoped to a subscription instance, usable across multiple services. For inference requests, a **team-ID header** must accompany the key.
- **Inference key** — Pre-scoped to a specific instance + team; limited to inference requests only; no extra headers required.
- **Admin scope** — Instance admins can list and revoke keys for all users in the instance; individual users manage only their own.

### API Key Operations

| Operation | Who | Notes |
|-----------|-----|-------|
| Create key | User (UI at `bob.ibm.com`) | Choose type (General/Inference); value shown once. |
| List keys | User (own) / Admin (all in instance) | Search/filter active/revoked/expired. |
| Revoke key | User (own) / Admin (any in instance) | Deactivates the key. |
| View details | User/Admin | Metadata only; secret not retrievable after creation. |

### Usage with Bob Shell

API keys authenticate non-interactive (`bob -p`) sessions, CI/CD pipelines, and scripted workflows, as well as inference requests. See [§3](#3-bob-shell-cli--the-automation-api) and the install/setup docs.

> ⚠️ Keys are shown only once at creation; store securely immediately.

Docs: [api-keys](https://bob.ibm.com/docs/ide/account/api-keys)

---

## 3. Bob Shell CLI — The Automation API

### Main Concepts

The `bob` CLI is Bob's programmatic surface for automation. It runs the same agent runtime as the IDE, in either an interactive TUI or a non-interactive single-shot mode suitable for scripts and CI/CD.

### `bob` Command Flags

| Flag | Short | Description | Example |
|------|-------|-------------|---------|
| `--prompt` | `-p` | Non-interactive prompt; run once and exit | `bob -p "Explain this code"` |
| `--prompt-interactive` | `-i` | Interactive session seeded with an initial prompt | `bob -i "Help me debug"` |
| `--sandbox` | `-s` | Enable sandbox mode (isolated execution) | `bob -s` |
| `--debug` | `-d` | Enable debug mode | `bob -d` |
| `--yolo` | — | Auto-approve all tool calls (enables write/execute in non-interactive) | `bob --yolo` |
| `--approval-mode` | — | Set tool approval mode | `bob --approval-mode=auto_edit` |
| `--allowed-tools` | — | Specific tools to auto-approve | `bob --allowed-tools="git status"` |
| `--include-directories` | — | Add extra directories to the workspace | `bob --include-directories=../lib` |
| `--chat-mode` | — | Choose the interaction mode | `bob --chat-mode` |
| `--hide-intermediary-output` | — | Hide intermediate tool output | `bob --hide-intermediary-output` |
| `--show-license` | — | Display the license agreement | `bob --show-license` |
| `--accept-license` | — | Accept the license (required before first non-interactive run) | `bob --accept-license -p "…"` |
| `--instance-id` | — | Target a specific subscription instance | `bob --instance-id=my-instance` |
| `--team-id` | — | Target a specific team (required for General API keys on inference) | `bob --team-id=my-team` |

### Notes

- Non-interactive sessions default to **non-destructive tools only** (reads). Use `--yolo` to enable writes/edits; Bob Shell still refuses to write outside the start directory.
- Input can be piped: `cat buildError.txt | bob -p "Explain this build error"`.
- Output can be redirected: `bob -p "Review @bigFile.java" > review.md`.
- `@` references inject project files into the prompt context.

Docs: [configuring](https://bob.ibm.com/docs/shell/configuration/configuring), [non-interactive](https://bob.ibm.com/docs/shell/getting-started/start-bobshell-non-interactive), [usage examples](https://bob.ibm.com/docs/shell/getting-started/bobshell-examples)

---

## 4. Non-Interactive Sessions (Headless Agent Invocations)

### Main Concepts

A non-interactive session is the primary "API call" pattern for automation: a single prompt dispatched to the agent runtime, which runs the full tool-calling loop and returns the result to stdout.

### Functions & Parameters

| Function | Parameters | Behavior |
|----------|------------|----------|
| `bob -p "<prompt>"` | `prompt` (string) | Run the agent once with the prompt; emit answer + thinking steps to stdout. |
| Pipe input | stdin text + `prompt` | Combine piped content with the prompt (e.g., error logs). |
| `@<path>` mentions | file/folder path inside prompt | Inject referenced content into context (bypasses `.bobignore`/`.gitignore` for direct mentions). |
| `--yolo` | flag | Enable destructive tools (write/execute); still workspace-scoped. |
| `--accept-license` | flag | Required before first run. |
| Auth via API key | env/config | General key needs `--team-id`; Inference key is self-scoped. |

### When to use

- CI/CD pipelines and scheduled automation
- Scripted workflows avoiding browser sign-in
- Batch processing of files/projects
- Single-shot analysis (e.g., `npm run start 2>&1 | bob -p "Help me understand and fix this error"`)

Docs: [non-interactive](https://bob.ibm.com/docs/shell/getting-started/start-bobshell-non-interactive)

---

## 5. The Agent Tool Runtime

### Main Concepts

Bob's agent loop is a tool-calling runtime. For each step Bob: (1) decides the appropriate action, (2) proposes a tool call, (3) requests approval (unless auto-approved), (4) runs the tool and shows results, and (5) continues until the task is complete. Tools are grouped into categories; the active **mode** determines which groups are available.

### Tool Categories (Summary)

| Category | Tools (IDE) | Tools (Shell) |
|----------|-------------|---------------|
| Read | `read_file`, `glob`, `grep`, `list_files`, `GetSymbolsOverview`, `FindSymbol`, `FindReferencingSymbols` | (shell read tools) |
| Write | `write_file`, `apply_diff`, `insert_content`, `search_and_replace` | `write_to_file`, `apply_diff`, `insert_content` |
| Command | `execute_command` | `execute_command` |
| Subagent | `spawn_subagent` | `spawn_subagent` |
| Subtask | `start_subtask` | — |
| MCP | MCP server tools | `use_mcp_tool` (MCP server tools) |
| Mode | `switch_mode` | `switch_mode` |
| Skill | `use_skill` | — |
| Workflow | `start_workflow` | — |
| Todo | `update_todo_list` | — |
| Question | `ask_followup_question` | `ask_followup_question` |

> Note: Tool names differ slightly between IDE and Shell (e.g., `write_file` vs `write_to_file`); the Shell omits some IDE-only tools (subtask, skill, workflow, todo) and exposes `use_mcp_tool` as a named tool.

Docs: [IDE tools](https://bob.ibm.com/docs/ide/core-concepts/tools), [Shell tools](https://bob.ibm.com/docs/shell/core-concepts/tools)

---

## 6. Read Tools

### Main Concepts

Read tools let Bob access file content and code structure without making changes. Used for reviewing code, searching patterns, and examining project structure.

### Functions & Parameters

| Tool | Purpose | Parameters (inferred) | Example |
|------|---------|------------------------|---------|
| `read_file` | Read file contents, optional line ranges | `path`, optional `start_line`/`end_line` | View config files, examine source |
| `glob` | Find files by name pattern | `pattern` (e.g., `**/*.ts`) | Locate test files, find all components |
| `grep` | Search file content with regex | `pattern`, optional `include`/paths | Find function definitions, locate TODOs |
| `list_files` | List files and directories | `path` | Browse a directory's structure |
| `GetSymbolsOverview` | Top-level symbols in a file | `path` | Understand file structure before diving in |
| `FindSymbol` | Look up a symbol by name path, optional depth | `symbolPath`, optional `depth` | Navigate to a class/method/function |
| `FindReferencingSymbols` | Find all references to a symbol | `symbol` | Understand where a function/type is used |

Docs: [read-tools](https://bob.ibm.com/docs/ide/core-concepts/tools#read-tools)

---

## 7. Write Tools

### Main Concepts

Write tools let Bob create or modify files. Enabled only in destructive-capable modes (Agent/Code) and gated by approval unless auto-approved.

### Functions & Parameters

| Tool | Purpose | Parameters (inferred) | Example |
|------|---------|------------------------|---------|
| `write_file` (IDE) / `write_to_file` (Shell) | Create a new file or fully rewrite an existing one | `path`, `content` | Generate components, create config files |
| `apply_diff` | Apply precise, targeted changes to parts of a file | `path`, diff hunks | Update function logic, fix bugs, refactor |
| `insert_content` | Insert new lines at a specific position | `path`, `position`, `content` | Add imports, insert new functions |
| `search_and_replace` (IDE) | Find/replace text or regex across a file | `path`, `search`, `replace` (text or regex) | Rename variables, update repeated patterns |

Docs: [write-tools](https://bob.ibm.com/docs/ide/core-concepts/tools#write-tools)

---

## 8. Command Tools

### Main Concepts

Command tools let Bob run shell/CLI commands in the workspace for system operations.

### Functions & Parameters

| Tool | Purpose | Parameters (inferred) | Example |
|------|---------|------------------------|---------|
| `execute_command` | Run CLI commands in the workspace | `command` (string), optional `cwd` | Install dependencies, run tests, build projects |

Approval behavior: gated by the **Execute** auto-approve setting; high risk (system-level operations possible).

Docs: [command-tools](https://bob.ibm.com/docs/ide/core-concepts/tools#command-tools)

---

## 9. Subagent Tools

### Main Concepts

A **subagent** is an independent agent spawned to run a focused, self-contained task in its own isolated context window, returning only a summary to the main conversation. Useful for work that would pollute the main context. Bob uses subagents sparingly — only when the task is clearly self-contained, would add significant irrelevant content, and cannot be done with one or two direct tool calls.

### Subagent Types

| Type | Description | Best for |
|------|-------------|----------|
| `explore` | Read-only codebase exploration, runs on a lighter model | Searching/summarizing code, finding files, understanding structure |
| `general` | Full tool access, runs on the default model | Any self-contained task needing reads/writes/commands |

### Functions & Parameters

| Tool | Purpose | Parameters | Notes |
|------|---------|------------|-------|
| `spawn_subagent` | Create an independent agent with its own context window | `type` (`explore`/`general`), task `description`, optional `fork_context: true` | `fork_context` passes parent conversation history into the subagent; default is no shared history/state/files. |

### Approval

Spawning a subagent is gated by the **Subagent** auto-approve setting (workflow delegation, not direct system access).

Docs: [subagents](https://bob.ibm.com/docs/ide/features/subagents), [subagent-tools](https://bob.ibm.com/docs/ide/core-concepts/tools#subagent-tools)

---

## 10. Subtask Tools

### Main Concepts

A **subtask** is a visible, interactive named task with its own breadcrumb and conversation thread in the UI. Unlike a subagent (silent, summary-only), a subtask can be followed, reviewed, and continued. Used when work is complex enough to benefit from dedicated tracking.

### Functions & Parameters

| Tool | Purpose | Parameters | Example |
|------|---------|------------|---------|
| `start_subtask` | Create a new task with its own conversation thread | `title` (UI breadcrumb), `message` (instructions), optional `todo list`, optional `mode` (e.g., Plan) | Break a large request into a tracked subtask |

### Approval

Gated by the **Subtask** auto-approve setting (workflow organization, not system access).

Docs: [subtask-tools](https://bob.ibm.com/docs/ide/core-concepts/tools#subtask-tools)

---

## 11. MCP Tools (Model Context Protocol)

### Main Concepts

MCP is a standardized client-server protocol letting Bob connect to external tools/services ("like a USB-C port"). Bob acts as MCP client; configured servers expose tools Bob can call. MCP tools are accessed via "MCP server tools" (IDE) / `use_mcp_tool` (Shell).

### Server Transports

| Transport | How it works | When to use |
|-----------|--------------|------------|
| **STDIO** | Local server process communicating over stdin/stdout JSON | Local tools, full machine access, dev work |
| **Streamable HTTP** | Remote server over HTTP/HTTPS (modern standard) | Remote servers, shared/team servers, production |
| **SSE (legacy)** | Deprecated HTTP+SSE with `/events` + `/message` endpoints | Backwards compatibility only |

### Configuration (`settings.json` → `mcpServers`)

**STDIO parameters:**

| Parameter | Required | Description |
|-----------|----------|-------------|
| `command` | Yes | Executable to run (`node`, `python`, `npx`) |
| `args` | No | Array of arguments |
| `cwd` | No | Working directory for the server process |
| `env` | No | Environment variables for the server process |
| `alwaysAllow` | No | Array of tool names to auto-approve |
| `disabled` | No | `true` to disable this server |

**Streamable HTTP parameters:**

| Parameter | Required | Description |
|-----------|----------|-------------|
| `url` | Yes | Full URL of the MCP endpoint |
| `headers` | No | Custom HTTP headers (e.g., auth tokens) |
| `alwaysAllow` | No | Array of tool names to auto-approve |
| `disabled` | No | `true` to disable this server |

Example STDIO config:
```json
{
  "mcpServers": {
    "local-server": {
      "command": "node",
      "args": ["server.js"],
      "cwd": "/path/to/project/root",
      "env": { "API_KEY": "your_api_key" },
      "alwaysAllow": ["tool1", "tool2"],
      "disabled": false
    }
  }
}
```

### Functions & Parameters (runtime)

| Tool | Purpose | Parameters (inferred) |
|------|---------|------------------------|
| `use_mcp_tool` (Shell) / MCP server tools (IDE) | Call a tool exposed by a connected MCP server | `server`, `tool_name`, tool-specific `arguments` |

### Security

Never hardcode credentials in MCP config; use `env`/secret references and `.gitignore` config files. The **MCP** auto-approve setting controls whether Bob calls MCP servers without asking.

Docs: [understanding-mcp](https://bob.ibm.com/docs/ide/configuration/mcp/understanding-mcp), [server-transports](https://bob.ibm.com/docs/ide/configuration/mcp/server-transports), [mcp-in-bob](https://bob.ibm.com/docs/ide/configuration/mcp/mcp-in-bob)

---

## 12. Mode Tools

### Main Concepts

Mode tools let Bob switch between specialized modes whose tool groups/subagents are optimized for the current task.

### Functions & Parameters

| Tool | Purpose | Parameters | Example |
|------|---------|------------|---------|
| `switch_mode` | Change to a different mode | `mode` slug (e.g., `agent`, `plan`, `ask`, custom slug) | Switch to Plan mode for design, Agent mode for implementation |

### Approval

Gated by the **Mode** auto-approve setting (workflow organization, not system access).

Docs: [mode-tools](https://bob.ibm.com/docs/ide/core-concepts/tools#mode-tools)

---

## 13. Skill Tools

### Main Concepts

**Skills** are reusable instruction sets (`SKILL.md` + supporting files) that teach Bob specialized workflows. Loaded into context once per conversation via `use_skill` when a request matches the skill's domain (e.g., Carbon Design System, DITA docs, Jira sprints, `create-plan`). Skills are only available in **Advanced mode** (Shell) / modes that include the `skill` tool group.

### Functions & Parameters

| Tool | Purpose | Parameters | Example |
|------|---------|------------|---------|
| `use_skill` | Load detailed instructions for a named skill into current context | `skill_name` (string) | Activate the Carbon builder skill, load Jira workflow guidance |

### Skill file structure

```markdown
---
name: api-documentation
description: Generate API documentation following OpenAPI standards
---
Generate API documentation that includes:
- Endpoint descriptions
- Request/response schemas
- Authentication requirements
Follow the style guide in `api-style-guide.md`.
```

Keep `SKILL.md` focused on the core workflow; move reference material to supporting files.

Docs: [skills](https://bob.ibm.com/docs/ide/features/skills), [skill-tools](https://bob.ibm.com/docs/ide/core-concepts/tools#skill-tools)

---

## 14. Workflow Tools

### Main Concepts

Workflow tools launch curated, multi-step processes for recurring tasks (e.g., generating a PR description from a git diff, reviewing code against a base branch).

### Functions & Parameters

| Tool | Purpose | Parameters | Example |
|------|---------|------------|---------|
| `start_workflow` | Launch a named workflow with structured steps | `workflow_name`, optional workflow-specific args | Create a pull request, run a code review |

Docs: [workflow-tools](https://bob.ibm.com/docs/ide/core-concepts/tools#workflow-tools)

---

## 15. Todo Tools

### Main Concepts

Todo tools let Bob maintain a visible markdown checklist for multi-step tasks, marking items complete and keeping one item in-progress at a time.

### Functions & Parameters

| Tool | Purpose | Parameters (inferred) | Example |
|------|---------|------------------------|---------|
| `update_todo_list` | Create/update a markdown checklist of task steps | list of items with status (`pending`/`in_progress`/`completed`), optional new items | Track implementation steps, mark completed items |

Docs: [todo-tools](https://bob.ibm.com/docs/ide/core-concepts/tools#todo-tools)

---

## 16. Question Tools

### Main Concepts

Question tools let Bob request clarification or additional detail from the user before proceeding.

### Functions & Parameters

| Tool | Purpose | Parameters (inferred) | Example |
|------|---------|------------------------|---------|
| `ask_followup_question` | Request clarification or additional detail | `question` text | Ask about preferred implementation approach, request missing info |

Docs: [question-tools](https://bob.ibm.com/docs/ide/core-concepts/tools#question-tools)

---

## 17. Modes (Agent Personas)

### Main Concepts

Modes are specialized personas that tailor Bob's behavior, role definition, available tool groups, and allowed subagents. Switching is done via `/mode <slug>`, the mode selector, or `switch_mode`.

### Built-in Modes (IDE)

| Mode | Purpose | When to use | Key capabilities | Tool access | Allowed subagents |
|------|---------|-------------|-----------------|-------------|-------------------|
| **Agent** | Write/modify code | Implementing features, fixing bugs, file ops | Full tool access for implementation/debugging/complex workflows | `Read`, `Edit`, `Execute`, `MCP`, `Skill`, `Todo`, `Subtask`, `Subagent`, `Mode` | `All` |
| **Plan** | Plan/design | Before implementation; architecture, specs, breaking down problems | System design, problem decomposition, specifications | `Read`, `Edit`, `MCP`, `Skill`, `Subagent`, `Mode` | `Explore` |
| **Ask** | Q&A/explanations | Understanding concepts, analyzing code, recommendations | Read-only analysis, external resources, Mermaid diagrams | `Read`, `MCP`, `Skill`, `Subagent`, `Mode` | `Explore` |

### Built-in Modes (Shell)

| Mode | Purpose |
|------|---------|
| **Code** | Generate, modify, refactor code from the command line |
| **Ask** | Answers about codebase and dev questions |
| **Plan** | Design/plan before running |
| **Advanced** | Extended capabilities including MCP tools |

### Mode role definitions (IDE examples)

- **Agent:** "You are Bob, a highly skilled software engineer with extensive knowledge in many programming languages, frameworks, design patterns, and best practices."
- **Plan:** "You are Bob, an experienced technical leader who is inquisitive and an excellent planner… uses the create-plan skill… before they switch into another mode to implement."
- **Ask:** "You are Bob, a knowledgeable technical assistant focused on answering questions and providing information about software development…"

Docs: [modes](https://bob.ibm.com/docs/ide/features/modes)

---

## 18. Custom Modes

### Main Concepts

Custom modes are YAML-defined personas with scoped tool groups, optional file-edit restrictions, and allowed subagent presets. Configured globally (`~/.bob/settings/custom_modes.yaml`) or per-project (`.bob/custom_modes.yaml`).

### Mode Configuration Properties

| Property | Description |
|----------|-------------|
| `slug` | Unique mode identifier (letters, numbers, hyphens only) |
| `name` | Display name |
| `roleDefinition` | System role/persona text |
| `customInstructions` | Mode-specific instructions |
| `groups` | Array of enabled tool-group names (see below) |
| `allowedSubagents` | Array of permitted subagent presets (e.g., `Explore`) |
| `fileRegex` (edit group) | Restricts edit tools to matching file paths |

### Available Tool Groups

| Group | Grants |
|-------|--------|
| `read` | Read files and directories |
| `edit` | Modify files (restrictable via `fileRegex`) |
| `execute` | Run terminal commands |
| `mcp` | Access MCP servers |
| `skill` | Load skills |
| `workflow` | Launch pre-defined workflows |
| `todo` | Update task todo lists |
| `subtask` | Create subtasks |
| `subagent` | Spawn subagents |
| `mode` | Switch to another mode |

### Validation Rules

- `slug` must be unique, letters/numbers/hyphens only.
- Only supported group names grant access; unknown names are ignored.
- Omitting `groups` yields no grouped tools.
- `allowedSubagents` restricts subagent presets to those listed.
- Invalid `fileRegex` prevents the mode file from loading.

### Mode-specific instructions

Add via directory structure (preferred): `01-style-guide.md`, `02-formatting.txt`, … under the mode's rules directory; or a single file. Default modes can be overridden.

Docs: [custom-modes](https://bob.ibm.com/docs/ide/configuration/custom-modes)

---

## 19. Custom Rules & Context Files

### Main Concepts

Rules are custom instructions injected into the system context to standardize Bob's behavior. They are loaded hierarchically and scoped by mode, project, or globally.

### Rule Scopes & Directories

| Directory | Purpose |
|-----------|---------|
| `rules/` | General rules for all modes |
| `rules-agent/` | Agent mode only |
| `rules-plan/` | Plan mode only |
| `rules-ask/` | Ask mode only |
| `rules-{mode}/` | Any custom mode |
| `~/.bob/rules/` | Global (user-wide) rules |

Load order: global → mode-specific → `AGENTS.md` → general workspace rules.

### `AGENTS.md`

- Auto-loaded by default from the workspace root; version-controllable for team standardization.
- Disable with `"bob-code.useAgentRules": false`.
- Loaded after mode-specific rules but before general workspace rules.

### Context files hierarchy

`~/.bob/AGENTS.md` (global) → `AGENTS.md` (project) — loaded hierarchically into the model's system context.

### Memory commands (Shell)

| Command | Purpose |
|---------|---------|
| `/memory refresh` | Reload context/memory files |
| `/memory show` | Display loaded memory |

Docs: [rules](https://bob.ibm.com/docs/ide/configuration/rules), [configuring](https://bob.ibm.com/docs/shell/configuration/configuring)

---

## 20. Slash Commands

### Main Concepts

Slash commands are reusable prompt templates invoked with `/`, defined as markdown files in `.bob/commands/` (project) or `~/.bob/commands/` (global). They work identically across Bob Shell and Bob IDE.

### Command Sources

| Command type | Source | Purpose |
|--------------|--------|---------|
| Custom workflow commands | `.bob/commands/` or `~/.bob/commands/` | User-created automation |
| Mode commands | Built-in and custom modes | Switch operational context (e.g., `/mode code`) |

### File Format with Frontmatter

```markdown
---
description: Create a new API endpoint
argument-hint: <endpoint-name> <http-method>
---
Create a new API endpoint called $1 that handles $2 requests.
Include proper error handling and documentation.
```

### Frontmatter Fields

| Field | Purpose | Example |
|-------|---------|---------|
| `description` | Appears in the command menu | "Create a new API endpoint" |
| `argument-hint` | Shows expected arguments next to the command | `<endpoint-name> <http-method>` |

### Argument substitution

Positional placeholders `$1`, `$2`, … in the body are replaced by arguments typed after the command. Argument hints are visual only (not inserted).

### Built-in examples

`/mode code`, `/mode ask`, `/review`, `/api-endpoint <endpoint-name> <http-method>`, `/instance`, `/restore <checkpoint_file>`, `/init`, `/memory refresh`, `/memory show`.

Docs: [slash-commands](https://bob.ibm.com/docs/shell/features/slash-commands)

---

## 21. Context Mentions (`@`)

### Main Concepts

`@` mentions inject specific project elements directly into the conversation context for precise, context-aware assistance. They bypass `.bobignore` and `.gitignore` (for direct file/folder mentions); git-based mentions respect `.gitignore`.

### Mention Types

| Format | Provides | Notes |
|--------|----------|-------|
| `@/path/to/file` | Full contents of a single file | Bypasses ignore rules |
| `@/path/to/folder` (no trailing slash) | Contents of all non-binary text files directly in the folder | **Non-recursive**; watch context-window limits |
| `@git-changes` | Current uncommitted git changes | Respects `.gitignore` |
| `@<commit-hash>` | Contents changed in a specific commit | Respects `.gitignore` |
| `@problems` | All errors/warnings from the Problems panel | File paths, line numbers, diagnostic messages, grouped by file |
| `@terminal` | Last command + its complete output | Preserves terminal state; limited to visible buffer |
| Editor selection | Selected text via `Cmd+L`/`Ctrl+L` | Adds highlighted text to chat |

Docs: [context-mentions](https://bob.ibm.com/docs/ide/features/context-mentions)

---

## 22. Auto-Approve / Approval Modes

### Main Concepts

Auto-approve eliminates repetitive permission prompts for specific action classes. Each action has an independent toggle and risk level. Configurable via the auto-approve toolbar (IDE) or `--yolo`/`--approval-mode`/`--allowed-tools` flags (Shell).

### Available Actions & Risk

| Action | Description | Risk |
|--------|-------------|------|
| **Read** | Read files without asking | Low |
| **Edit** | Create/modify/delete files without asking | High (directly modifies codebase) |
| **Execute** | Run terminal commands automatically | High (system-level operations possible) |
| **MCP** | Use configured MCP servers without asking | Depends on server access |
| **Skill** | Activate skills without asking | Workflow organization |
| **Todo** | Update todo lists without asking | Workflow organization |
| **Subtask** | Create/complete subtasks without confirmation | Workflow organization |
| **Subagent** | Spawn subagents without asking | Workflow delegation (isolated context) |
| **Mode** | Change modes automatically | Workflow organization |

### Shell approval flags

| Flag | Behavior |
|------|----------|
| `--yolo` | Auto-approve all tool calls |
| `--approval-mode=<mode>` | Set a specific approval mode (e.g., `auto_edit`) |
| `--allowed-tools="git status"` | Auto-approve a specific tool set |

Docs: [auto-approve](https://bob.ibm.com/docs/ide/features/auto-approving-actions)

---

## 23. Checkpointing & Rollback

### Main Concepts (Shell)

Checkpointing auto-snapshots the project before file-modifying operations (`write_file`, `replace`, …), enabling rollback. Disabled by default.

### Checkpoint contents

When a file-modifying tool is approved, Bob Shell automatically records:
1. A **Git snapshot** in a shadow repo (`~/.bob/history/<project_hash>`) — separate from the project's Git repo.
2. The **conversation history** up to that point.
3. The **tool call** about to run.

Data is stored locally at `~/.bob/history/<project_hash>` and `~/.bob/tmp/<project_hash>/checkpoints`.

### Functions & Parameters

| Function | Purpose | Parameters |
|----------|---------|------------|
| `/restore` | List all saved checkpoints for the current project | none |
| `/restore <checkpoint_file>` | Restore the project to a specific checkpoint | `checkpoint_file` name (e.g., `2025-06-22T10-00-00_000Z-my-file.txt-write_file`) |

### Enabling

Via `settings.json`:
```json
{ "general": { "checkpointing": { "enabled": true } } }
```

Docs: [checkpointing](https://bob.ibm.com/docs/shell/features/checkpointing)

---

## 24. Context Window Management

### Main Concepts

Bob operates within a **270,000-token context window**. Budget is consumed across several categories; understanding them keeps tasks focused and cost-efficient.

### Token Categories

| Category | What it includes | What makes it grow |
|----------|------------------|-------------------|
| System prompt | Bob's core instructions for the session | Set at task start; stays flat |
| Tool definitions | Built-in tool schemas + connected MCP tool definitions | Set at task start; stays flat |
| MCP Tools | Instructions/descriptions for MCP-server tools | Grows when adding servers/tools |
| Rules | Custom instructions from project/mode rule files (`AGENTS.md`, `.bob/rules-*`) | Set at task open |
| Skills | Instructions from skills Bob loaded | Can grow if a skill activates mid-thread |
| Messages | Prompts, replies, tool activity, `@` mentions, tool results | Grows every turn and with repo exploration |

### Best practices

- Reference specific files/line ranges (`@/src/utils/validation.ts:45-67`), not broad swaths.
- Work in stages: find → inspect → plan → change → validate.
- Use **subagents** for broad repo reads so the main task receives condensed summaries instead of every `read_file` landing in Messages.
- Reset or condense when Messages fills up.
- When sources conflict, trust running code and tests over stale comments/READMEs.

Docs: [context-window-management](https://bob.ibm.com/docs/ide/core-concepts/context-window-management)

---

## 25. Bobalytics (Agent Observability)

### Main Concepts

Bobalytics provides analytics on Bob's impact, Bobcoin spending, and team adoption. Enterprise-plan only; accessed via the web portal (Admin → Bobalytics). Designed for team-level insights without exposing individual user data to admins.

### Levels

1. **Workspace view** — org-wide overview with KPI cards.
2. **Team view** — per-team breakdown.
3. **User view** — individual (admins see only their own).

### Key Performance Indicators

| KPI | Description |
|-----|-------------|
| **Adoption rate** | Measure of team/org adoption of Bob |
| **Bob factor** | Productivity impact metric |
| **Bobcoin spend** | Consumption/billing tracking |

### Filtering

- Time range selector controls the period for all charts (with automatic previous-period comparison on KPI cards).
- Teams dropdown (workspace view) aggregates data across selected teams.

Docs: [bobalytics](https://bob.ibm.com/docs/ide/features/bobalytics), [bobcoins](https://bob.ibm.com/docs/ide/account/bobcoins)

---

## 26. Capability Summary & Cross-Reference

| Capability | Primary surface | Key functions/params | Auth/Approval |
|------------|----------------|---------------------|---------------|
| Headless agent invocation | `bob -p` | `-p`, `--yolo`, `--allowed-tools`, `--approval-mode`, `--instance-id`, `--team-id`, `@mentions`, pipe stdin | API key (General/Inference) |
| Read | `read_file`, `glob`, `grep`, `list_files`, `GetSymbolsOverview`, `FindSymbol`, `FindReferencingSymbols` | `path`, `pattern`, optional ranges/depth | Read auto-approve (low risk) |
| Write | `write_file`/`write_to_file`, `apply_diff`, `insert_content`, `search_and_replace` | `path`, `content`, diff/position, search/replace | Edit auto-approve (high risk) |
| Execute | `execute_command` | `command`, optional `cwd` | Execute auto-approve (high risk) |
| Subagent | `spawn_subagent` | `type` (`explore`/`general`), `description`, `fork_context` | Subagent auto-approve |
| Subtask | `start_subtask` | `title`, `message`, `todo list`, `mode` | Subtask auto-approve |
| MCP | `use_mcp_tool` / MCP server tools; config `mcpServers` | STDIO (`command`,`args`,`cwd`,`env`,`alwaysAllow`,`disabled`) / HTTP (`url`,`headers`,`alwaysAllow`,`disabled`) | MCP auto-approve |
| Mode switch | `switch_mode`, `/mode <slug>` | `mode` slug | Mode auto-approve |
| Skill | `use_skill` | `skill_name` | Skill auto-approve; Advanced mode required |
| Workflow | `start_workflow` | `workflow_name`, args | — |
| Todo | `update_todo_list` | items with status | Todo auto-approve |
| Question | `ask_followup_question` | `question` | — |
| Custom modes | `custom_modes.yaml` | `slug`, `name`, `roleDefinition`, `groups`, `allowedSubagents`, `fileRegex` | — |
| Custom rules | `rules/`, `rules-{mode}/`, `AGENTS.md` | markdown instruction files | — |
| Slash commands | `.bob/commands/*.md` | `description`, `argument-hint`, `$1..$N` body | — |
| Context mentions | `@` | file/folder/git/problems/terminal | — |
| Checkpointing | `/restore`, `settings.json` | `checkpointing.enabled`, `<checkpoint_file>` | — |
| Observability | Bobalytics (web portal) | KPIs: adoption rate, Bob factor, Bobcoin spend | Enterprise plan |
| Billing | Bobcoins | per-action consumption unit; plans 40/160/500/month | — |

### Key takeaways

- **No public REST agent API.** Bob's "API" is the `bob` CLI plus configuration files. Programmatic integration is via non-interactive `bob -p` sessions in scripts/CI, authenticated with API keys.
- **Tool-calling runtime is the core.** The agent loop is governed by modes (tool-group scoping) and approval gates (auto-approve/`--yolo`).
- **Extensibility is configuration-driven.** MCP servers, custom modes, rules, skills, and slash commands are all declarative files (`settings.json`, YAML, markdown) rather than code you deploy.
- **Multi-agent = subagents + subtasks.** `spawn_subagent` (silent, isolated, summary-only) vs `start_subtask` (visible, interactive, tracked thread).
- **Context discipline matters.** The 270k-token window is budgeted across system/tools/rules/skills/messages; `@` mentions, subagents, and staged workflows keep it efficient.