# Vibe API Analysis — Agent Capabilities (Vibe Work + Vibe Code)

> **Docs:** `https://docs.mistral.ai/vibe/overview` · [Vibe Code overview](https://docs.mistral.ai/vibe/code/overview) · [Vibe Work get-started](https://docs.mistral.ai/vibe/work/get-started)
> **Entry point:** `https://chat.mistral.ai` (web) · CLI (`vibe`) · VS Code extension · Vibe Code Web (remote sandbox) · iOS / Android
> **Auth:** Mistral account (OAuth in-product); programmatic access via API keys in the underlying Studio API (`Authorization: Bearer $MISTRAL_API_KEY`)
> **Underlying platform:** Vibe is the product surface of the **Studio Agents & Conversations API** — agent-related capabilities reached through Vibe are backed by the REST tags `beta.agents`, `beta.conversations`, `beta.connectors`, `beta.libraries` at `https://api.mistral.ai/v1`.
> **Description:** Vibe is Mistral's **unified agent** for productivity and coding, the rebrand of Le Chat. It exposes agent capabilities through **two task-shaped modes** rather than a single REST surface: **Vibe Work** (multi-stage professional tasks across apps/tools, run server-side in Mistral's environment) and **Vibe Code** (development against a local filesystem or remote sandbox, via CLI / VS Code / Web). A third **Chat** tab keeps legacy agent features. The agent loop is managed for you in both modes: you describe an outcome in natural language, Vibe gathers context, plans and acts, shows a live todos panel and reasoning, and asks for approval before sensitive actions. Agent behavior is configured declaratively — Work uses **Skills**, **Connectors**, **Libraries**, **Workflows**, and **Scheduled Tasks**; Code uses **Agents** (approval-rule bundles), **trusted folders**, and **per-tool permissions**. Developers can also reach the same primitives directly through the Studio API. This study treats each Vibe capability as an "API function" and extracts its concepts, parameters, and behavior.

---

## Table of Contents

1. [Platform Overview & Core Concepts](#1-platform-overview--core-concepts)
2. [Vibe Work — Task Entry & The Work Agent Loop](#2-vibe-work--task-entry--the-work-agent-loop)
3. [Skills (Reusable Specialized Behavior)](#3-skills-reusable-specialized-behavior)
4. [Connectors (External Tools & Data Sources)](#4-connectors-external-tools--data-sources)
5. [Libraries (RAG Knowledge Bases)](#5-libraries-rag-knowledge-bases)
6. [Workflows (Studio-Built Automations Called as Tools)](#6-workflows-studio-built-automations-called-as-tools)
7. [Scheduled Tasks (Time-Based Autonomous Runs)](#7-scheduled-tasks-time-based-autonomous-runs)
8. [Files, Canvas & Context Inputs](#8-files-canvas--context-inputs)
9. [Safety, Approvals & Supervision (Work)](#9-safety-approvals--supervision-work)
10. [Vibe Code — Coding Agent Loop & Surfaces](#10-vibe-code--coding-agent-loop--surfaces)
11. [Code Agents (Approval-Rule Bundles)](#11-code-agents-approval-rule-bundles)
12. [Code Tools, Per-Tool Permissions & Trusted Folders](#12-code-tools-per-tool-permissions--trusted-folders)
13. [Underlying Studio API (Developer Surface for the Same Primitives)](#13-underlying-studio-api-developer-surface-for-the-same-primitives)
14. [Capability Summary & Cross-Reference](#14-capability-summary--cross-reference)

---

## 1. Platform Overview & Core Concepts

### Main Concepts

Vibe is not a single endpoint but a product organized around a few recurring abstractions shared by its Work and Code modes:

- **Mode** — One of three task surfaces: **Work** (productivity, multi-tool delegation), **Code** (software engineering), **Chat** (turn-based + legacy features). Each shapes the same underlying agent loop differently.
- **Task** — The unit of work in Work. You describe an outcome; Work gathers context, breaks the task into steps, calls tools, and asks for approval on sensitive actions. Tasks are stateful and show a live todos panel.
- **Agent (Work sense)** — A reusable specialized assistant configurable in the legacy Chat tab (instructions + tools + knowledge). Superseded in Work by **Skills**.
- **Agent (Code sense)** — A *configuration override bundle* that defines the system prompt, tool selection, and approval rules for Vibe Code. Four built-in agents (`default`, `plan`, `accept-edits`, `auto-approve`) plus custom agents.
- **Skill** — A reusable, filesystem-based package (`SKILL.md` + supporting files) following the open [Agent Skills](https://agentskills.io) standard. Loaded via progressive disclosure (discovery → activation → execution). Tells Work *how* to approach a class of tasks.
- **Connector** — A secure bridge between Work and an external tool/data source (Gmail, GitHub, Notion, Linear, Slack, Google Drive, SharePoint…). Featured Connectors use OAuth; Knowledge Connectors index files server-side; MCP Connectors register arbitrary MCP servers.
- **Library** — A persistent knowledge base of uploaded documents/web pages connected to agents for built-in RAG. Work searches it in real time and returns answers with numbered source references.
- **Workflow** — A coded, deterministic automation built in Studio by developers and published as a chat-compatible assistant that surfaces in Work's tool menu. Distinct from Skills (which instruct behavior).
- **Scheduled Task** — A prompt scheduled to run automatically (once / daily / weekly / monthly / yearly). Runs unattended in Work mode using the Workflows infrastructure.
- **Canvas** — A reviewable, editable document surface where Work renders structured outputs (briefs, drafts, tables) for human review before reuse.
- **Project** — A scoped work area grouping related Work conversations around a team, customer, initiative, or topic.
- **Custom Instructions** — Persistent behavior rules (tone, format, language, constraints) applied to all Work conversations; overridden when an Agent/Skill is active.
- **Todos panel / Reasoning summary** — Live supervision surfaces in Work: a real-time checklist of steps plus a per-step reasoning trace and per-tool-call transparency.
- **Approval** — A human-in-the-loop checkpoint. Work stops before sensitive actions and offers `Continue` / `Always allow` / `Decline`; Code asks inline with keyboard shortcuts.

### Agent Capabilities Map

| Capability | Mode | Description | Docs |
|------------|------|-------------|------|
| **Task entry & agent loop** | Work | Natural-language task → context gathering → planning → tools → review | [get-started](https://docs.mistral.ai/vibe/work/get-started) |
| **Skills** | Work | Reusable `SKILL.md` instructions + files; progressive disclosure; auto/explicit activation | [skills](https://docs.mistral.ai/vibe/work/skills) |
| **Connectors** | Work | OAuth/MCP bridges to external tools; per-function pre-authorization | [connectors](https://docs.mistral.ai/vibe/work/connectors) |
| **Libraries** | Work | Persistent RAG knowledge bases with cited answers | [libraries](https://docs.mistral.ai/vibe/work/libraries) |
| **Workflows** | Work | Studio-built deterministic automations surfaced as tools | [workflows](https://docs.mistral.ai/vibe/work/workflows) |
| **Scheduled tasks** | Work | Time-based autonomous prompt runs (5 frequencies) | [scheduled-tasks](https://docs.mistral.ai/vibe/work/scheduled-tasks) |
| **Files & Canvas** | Work | Upload docs; review/edit structured outputs | [files-and-canvas](https://docs.mistral.ai/vibe/work/files-and-canvas) |
| **Web search & Open URL** | Work | Public information retrieval / read a specific page | [web-search-open-url](https://docs.mistral.ai/vibe/work/web-search-open-url) |
| **Custom instructions** | Work | Persistent preferences applied to all conversations | [custom-instructions](https://docs.mistral.ai/vibe/work/custom-instructions) |
| **Projects** | Work | Scoped conversation grouping | [projects](https://docs.mistral.ai/vibe/work/projects) |
| **Safety & approvals** | Work | Approval flow, per-function permissions, supervision | [safety-and-approvals](https://docs.mistral.ai/vibe/work/safety-and-approvals) |
| **Code loop & surfaces** | Code | CLI / VS Code / Web sandbox; read-write FS, shell, tools | [code overview](https://docs.mistral.ai/vibe/code/overview) |
| **Code Agents** | Code | Approval-rule + tool/prompt override bundles | [safety-approvals-permissions](https://docs.mistral.ai/vibe/code/safety-approvals-permissions) |
| **Per-tool permissions** | Code | Enable/disable + per-tool ask/always + bash allow/deny | [safety-approvals-permissions](https://docs.mistral.ai/vibe/code/safety-approvals-permissions) |
| **Trusted folders** | Code | Allow loading project-level `.vibe/` config | [safety-approvals-permissions](https://docs.mistral.ai/vibe/code/safety-approvals-permissions) |
| **Chat legacy Agents** | Chat | Specialized assistants (instructions + tools + knowledge) | [chat-legacy agents](https://docs.mistral.ai/vibe/chat-legacy/agents) |

### Platform Architecture

```
User (web chat.mistral.ai / vibe CLI / VS Code / Vibe Code Web / mobile)
        │
        ├── Vibe Work  ── describe outcome in natural language
        │       │
        │       ▼
        │   Work agent loop (server-side, managed)
        │     1. Gather context (prompt, files, Libraries, Connectors, web)
        │     2. Plan → live todos panel + reasoning summary
        │     3. Call tools (Connectors / Workflows / web search / image gen)
        │     4. Pause for approval on sensitive actions (Continue / Always allow / Decline)
        │     5. Render result (chat or Canvas) for review
        │       │  configurable via: Skills, Custom instructions, Projects, Libraries,
        │       │                  Connector per-function permissions, Scheduled tasks
        │
        └── Vibe Code ── describe coding task in natural language
                │
                ▼
            Code agent loop (local FS + shell, or remote sandbox)
              1. Active Agent sets system prompt, tool selection, approval rules
              2. Read files / run commands / edit code / open PRs under supervision
              3. Ask approval per agent mode + per-tool permission + outside-CWD rule
              4. Trust gating: load `.vibe/` config only from trusted folders
                │  configurable via: Agents, config.toml (per-tool perms, bash allow/deny),
                │                  trusted folders, MCP servers
```

Both modes ultimately rest on the **Studio Agents & Conversations API** (agents, conversations/entries, built-in tools, connectors, libraries, handoffs). Vibe is the managed, productized surface; the Studio API is the developer escape hatch for the same primitives.

---

## 2. Vibe Work — Task Entry & The Work Agent Loop

**Concept:** Work is the productivity mode for delegating complex, multi-step tasks across your apps and tools. You describe the *outcome*; Work reasons through it, breaks it into steps, calls the right tools, and asks for approval before sensitive actions. You stay in control throughout via a live todos panel, a reasoning summary, and tool-call transparency.

### "Function": Start a Work task

There is no public REST call for this from the Vibe surface — you start a task through the UI at `chat.mistral.ai/work` (sidebar mode selector → **Work**). The conceptual parameters of a good first prompt:

| Parameter | Required | Description |
|-----------|----------|-------------|
| Outcome | Yes | The result you want, in natural language. |
| Source material / tools | No | Point Work at specific files, Libraries, Connectors, or web pages. |
| Audience | No | Who the output is for (shapes tone/format). |
| Constraints | No | Length, tone, format, deadline. |

Example prompts the docs give:
- `Research this topic and create a one-page brief I can share with my team.`
- `Summarize this PDF and extract owners, deadlines, and action items.`
- `Compare these two documents and highlight the main differences.`
- `Create a draft response based on this source material.`

### "Return value": The task run

Work surfaces the following during a run:

- **Todos panel** (right-hand) — live checklist of completed/pending steps; appears for longer multi-step tasks, not one-shot queries.
- **Reasoning summary** — per-step explanation of what Work is reading, comparing, deciding, ruling out.
- **Tool-call transparency** — each tool call shows: which tool, inputs, outputs, status (pending/succeeded/failed), expandable for inspection.
- **Follow-up questions** — when the prompt is ambiguous, Work asks clarifying questions (e.g. *Which Project?*, *One page or full report?*) before committing.
- **Result** — final summary plus generated content in Canvas or files; treated as a draft until reviewed.

### Control checkpoints

- Watch todos as they appear.
- Approve/deny sensitive actions when prompted.
- Stop with the **stop** button (black square) to redirect.
- Redirect with a follow-up message.

### When to use Work vs Code vs Chat

- **Work**: tasks across apps, documents, chats, meetings, business tools.
- **Code**: source-code work (files, repos, commands, diffs, PRs).
- **Chat**: quick turn-based conversation, or legacy features (Agents, Think mode, Code Interpreter, Deep Research, Memories).

---

## 3. Skills (Reusable Specialized Behavior)

**Concept:** Skills are reusable instructions and resources that give Work specialized capabilities for specific tasks. They follow the open **Agent Skills** standard (`SKILL.md` + supporting files). Use them when a task has a known method, checklist, template, output format, or domain-specific procedure. They are the Work-mode successor to legacy Chat Agents.

### How Skills work — progressive disclosure

Three stages keep context lean:

1. **Discovery** — at session start, Work loads only each Skill's name + description (~100 tokens each).
2. **Activation** — when a task matches a Skill's description, Work reads the full `SKILL.md` into context.
3. **Execution** — Work follows the instructions, optionally loading referenced files/examples on demand.

This means many Skills can be enabled without overloading context; only relevant ones expand.

### Skill parameters (creation form)

| Field | Required | Description |
|-------|----------|-------------|
| Title | Yes | Short, descriptive name (e.g. `Brand voice copywriting`). |
| Description | Yes | **Trigger text**: when Work should use the Skill. Phrased as *"Use when…"*. What Work reads at discovery to decide activation — must be specific. |
| SKILL.md | Yes | Full instructions Work follows on activation (tone, format, steps, examples). |
| Files / folders | No | Supporting references, templates, brand guidelines, sample outputs, datasets. Loaded as needed on activation (subject to a file-count limit; use progressive disclosure for large methods). |
| Scope | No | Personal (only you) or Workspace (shared). Studio-created Skills can be `Publish to Vibe`. |

### Built-in Skills (ship with Work)

| Skill | Triggers on |
|-------|-------------|
| `challenge-my-thinking` | Pressure-test a plan, devil's advocate, "what am I missing", risk review |
| `data-analysis` | Inspect/clean/transform/aggregate data (csv, json, tables, metrics, anomalies) |
| `deep-research` | Thorough research, source-backed synthesis, market/technical landscape |
| `doc-coauthoring` | Co-author proposals, technical specs, decision docs, RFCs |
| `document-review` | Review documents for completeness, compliance, consistency, quality |
| `internal-comms` | Status reports, leadership updates, 3P updates, FAQs, incident reports |
| `meeting-prep` | Prepare for upcoming meetings, brief on calendar items |
| `research-synthesis` | Synthesize multiple sources/notes/files into a structured brief |
| `skill-creator` | Create/update/delete Skills (loaded before any Skill edit) |
| `stakeholder-translator` | Reframe content for a different audience (exec, engineering, leadership) |
| `structured-extraction` | Extract structured data (tables, JSON) from emails, PDFs, free-text |
| `vibe-work-onboarding` | Walk through Vibe Work capabilities for first-time users |

### Activation modes

- **Auto-match**: enabled Skills activate automatically when a task matches their description.
- **Slash command**: type `/` and pick a Skill, or `/{skill-name}` directly (e.g. `/deep-research`).
- **Natural reference**: mention the Skill by name ("Use the contract-review Skill on this PDF").

### Skill scopes & admin controls

- **Personal** — only you see/use them.
- **Workspace** — shared across the workspace; toggled per user.
- **Force-enabled** — workspace admins can force specific Skills always-on (cannot be disabled by individuals).
- **Studio → Vibe** — Skills created in Studio can be `Publish to Vibe` to surface in Work; `Share` controls privacy/workspace visibility.

### Creation routes

1. **From the editor** — `Context` > `Skills` > `New Skill`, fill the form, add files, `Create Skill`.
2. **From a task (recommended)** — run a task, refine the approach, ask Work to *"turn this into a Skill"*; Work drafts a `SKILL.md` for review.

### Management

Edit title/description/`SKILL.md`/files; disable without deleting; delete personal Skills; share/private. Edits take effect in new chats; active chats keep the previous version.

### Skills vs Workflows (key distinction)

| | Skills | Workflows |
|---|--------|----------|
| What it is | Instructions + resources (`SKILL.md` + files) | A coded process built in Studio |
| Who creates | Any user, often from a chat | Developers in Studio |
| What it does | Tells Work *how* to approach a task | Runs deterministic logic, returns structured output |
| Best for | Style, procedure, checklist, repeatable method | Data pipelines, automations, internal-system integrations |
| Triggered by | Auto-match / slash / natural reference | Explicit selection via `+` > `Workflows` |

---

## 4. Connectors (External Tools & Data Sources)

**Concept:** Connectors are secure bridges between Work and your external tools/data sources. Work decides which Connector to call, asks for authentication if missing, and requests approval before sensitive actions. There are three families: **Featured** (direct OAuth), **Knowledge** (indexed server-side, e.g. Google Drive/SharePoint), and **MCP** (registered Model Context Protocol servers).

### Available Featured Connectors

| Connector | What it does |
|-----------|--------------|
| Atlassian | Confluence + Jira: search, summarize, project actions |
| Box | Search, analyze, insights from Box files |
| GitHub App | Search repos, review issues, manage PRs |
| Gmail | Include email in chats |
| Google Calendar | Include calendar in chats |
| Linear | Search/summarize/manage issues and projects |
| Notion | Search/summarize/author content |
| Outlook | Read and send emails |
| Outlook Calendar | Search events, schedule/delete meetings, accept/decline |
| SharePoint Search API | Search/open SharePoint via Microsoft Graph |
| Slack | Search messages, read channels, send messages, manage canvases |
| Stripe | Access/manage payments, customers, transactions |

Knowledge Connectors (Google Drive, SharePoint Online) are Team/Enterprise, require admin setup, and index files server-side in European data centers.

### "Function": Connect a service

| Step | Description |
|------|-------------|
| 1 | Open `Connectors` page from sidebar. |
| 2 | Find the Connector card, click `Connect`. |
| 3 | Complete OAuth 2.0 flow (password never shared). |
| 4 | Green `Connected` indicator confirms; disconnect anytime. |

### "Function": Use a Connector in a task

| Step | Description |
|------|-------------|
| 1 | Click `+` or type `/` in chat. |
| 2 | Select `Tools`, enable the Connector. |
| 3 | Describe the task naturally; Work picks the right Connector from the query. |

Typical prompts:
- *"What meetings do I have tomorrow?"*
- *"Find the latest Q3 revenue deck in Google Drive and prepare a one-page summary."*
- *"Create a calendar event for Friday at 2pm with the product team."*
- *"Check my unread emails from the legal team and draft replies for me to review."*

### Per-function Connector permissions

Each Connector exposes many functions (e.g. Linear exposes dozens). Per user, per function, you can pre-authorize:

| Function group | Risk | Recommendation |
|----------------|------|----------------|
| Interactive (create/update/delete/send/post) | Higher | Manual approval until trusted |
| Read-only (get/list/search) | Lower | Safe to pre-authorize |

Managed via `Connectors` > `My Connectors` > card > `Functions` tab > `Always allow` toggle per function. `Refresh tools` reloads the function list. Permissions are **per-user** (don't affect teammates).

### Approving Connector actions

When a Connector performs an action on your behalf (send email, create event, modify file), Work stops and offers:

| Option | Behavior |
|--------|---------|
| Continue | Approve this action only; ask again next time. |
| Always allow | Pre-authorize this function for the session. |
| Decline | Cancel; Work skips or asks how to proceed. |

### Data & privacy

- **Featured Connectors**: data fetched in real time, not stored; disconnect revokes immediately.
- **Knowledge Connectors**: files indexed and stored in European data centers.
- **Training**: data accessed via Connectors is **never** used to train/fine-tune models, regardless of plan.

---

## 5. Libraries (RAG Knowledge Bases)

**Concept:** Libraries are persistent knowledge bases you build from your own documents and web pages. Attach a Library to any task and Work searches its contents in real time, returning answers based on your data with direct numbered references to source material. Unlike file uploads (single chat), Libraries persist across sessions.

### Common use cases

Internal docs, product specs/release notes, research/reports, client/project knowledge, onboarding material.

### "Function": Create a Library

| Step | Description |
|------|-------------|
| 1 | Open `Libraries` page, click `New Library`. |
| 2 | Give it a descriptive name (visible if shared). |

### "Function": Populate a Library

| Method | Details |
|--------|---------|
| Upload documents | Drag-drop or `Upload`; up to **100 files at once**, max **100 MB/file**. |
| Add web pages | Click `Webpage`, enter URL(s), `Index`. Only text content indexed (no child pages/images/charts). |

### Supported formats

| Category | Formats |
|----------|---------|
| Documents | PDF, Word (.docx/.doc), PowerPoint (.pptx/.ppt), ODT, EPUB, RTF |
| Spreadsheets | Excel (.xlsx/.xls), CSV, ODS, Numbers |
| Images | PNG, JPEG, WebP, GIF |
| Text & markup | TXT, Markdown, RST, LaTeX |
| Data formats | JSON, JSONL, XML, YAML |
| Code files | Python, JS/TS, Java, Go, Rust, C/C++, Ruby, PHP, SQL, … |
| Email | EML, MSG |

### "Function": Attach & query

| Step | Description |
|------|-------------|
| Attach | In chat, click `+` → `Libraries` → search or create. Multiple Libraries attachable to one chat. |
| Query single doc | `+` → `Upload Files` → pick Library → specific document; or mention the document by name. |
| Query | Ask naturally (*"What does our refund policy say about international orders?"*). |

### "Return value": Cited answers

Each response includes **numbered footnotes** linked to source passages; the `Sources` button shows every reference used. Enables verification before reuse.

### Limits & processing

- No restriction on number of Libraries.
- Up to 100 files per upload, 100 MB/file.
- Monthly processing allowance shared across Libraries; resets automatically; pauses new uploads at limit (existing Libraries stay usable).

### Sharing & access (mirrors Studio Libraries API)

| Level | Capabilities |
|-------|-------------|
| Collaborator | Use, upload/delete documents, modify settings |
| Viewer | Use in own chats; cannot modify |
| Entire organization | Toggle + choose Collaborator/Viewer for all |

### Vibe ↔ API bridge

A Library's ID is visible in the URL (`https://chat.mistral.ai/libraries/<library_id>`). To let an API agent access a Vibe Library, an Org admin must share it with the Organization. Reverse works too: create via API, share with the team in Vibe.

---

## 6. Workflows (Studio-Built Automations Called as Tools)

**Concept:** Workflows are internal automations (data pipelines, report generators, ticketing flows) built by developers in Mistral Studio and published as chat-compatible assistants that surface in Work's tool menu. They are *not* built inside Work — they are coded processes Work calls as tools, like any other, with the Workflow name surfaced in the chat.

### How Workflows work in Work

- Developers build in Studio and publish as chat-compatible assistants.
- Users see them in Work's `+` tool menu, ready to attach to any chat.
- When attached, Work calls the Workflow as part of its reasoning, with the Workflow name visible in chat.
- The Workflow owns its inputs, logic, and output format; Work routes the prompt and renders the result.

### "Function": Use a Workflow in a chat

| Step | Description |
|------|-------------|
| 1 | Click `+` in chat. |
| 2 | Select `Workflows`. |
| 3 | Pick the Workflow from the list. |
| 4 | Workflow name appears as a **tag in the chat input** (attached). |
| 5 | Send prompt; Work calls the Workflow alongside its thinking chunk. |
| 6 | Tag also appears **above your message in chat history** for traceability. |

### "Return value" during a run

- **Thinking chunk** — Work reasons about arguments to pass.
- **`Workflow started: {workflow-name}`** — invocation confirmation.
- **Workflow output** (or failure) — rendered inline or in Canvas.
- **`Workflow failed: {workflow-name}`** — error reported (missing access, invalid input, tier restriction); Work surfaces it in plain language.

Example failure:
```
Workflow started: vibe-nuage
Your account tier (free) does not have access to this feature
Workflow failed: vibe-nuage
```

### Limitations

- Must be **published to your workspace** by a developer; if none appear, ask your Studio dev/admin.
- Some Workflows are **gated by account tier / org plan**.
- Invoked **explicitly** via `+` menu; Work does *not* auto-trigger them from the prompt (unlike Skills).

### Developer surface

For building/publishing Workflows that appear in Work, see Studio docs: `Conversational Workflows`, `Publish in Vibe` (must return `ChatAssistantWorkflowOutput`), plus `Forms and confirmations`, `Progress tracking`, `Canvas` for richer chat UX.

---

## 7. Scheduled Tasks (Time-Based Autonomous Runs)

**Concept:** Scheduled tasks (Public Preview) let you run a prompt automatically at a future date or on a recurring schedule. Work picks up the prompt at the scheduled time, executes it like any other task, and notifies you when the result is ready. Runs **in Work mode only**, using the Workflows infrastructure under the hood — so all Work capabilities apply (Skills, Connectors, web search, Libraries, Projects).

### Common use cases

- Weekly digest (Monday: summarize unread email + Slack from past week).
- Release summary (Friday: from Linear, Notion, Slack).
- Meeting briefing (daily 9am).
- Tech/competitive watch (weekly).
- One-off reminders (single future prompt).

### "Function": Create a scheduled task

Two routes:

| Route | Steps |
|-------|-------|
| From Tasks page | Sidebar `Scheduled` → `New Task` → write prompt → `Schedule` → pick frequency + time → confirm. |
| From a chat | Ask Work directly: `Schedule this to run every Monday at 9am.` Work proposes a schedule; you confirm. |

Suggested schedules may appear based on recent conversations/connected tools.

### Scheduling parameters (frequency)

| Frequency | What you pick |
|-----------|---------------|
| `Once` | A single calendar date and time |
| `Daily` | A time of day (runs every day) |
| `Weekly` | One or several days of the week + a time |
| `Monthly` | A day of the month + a time |
| `Yearly` | A day and a month (one date/year) + a time |

Triggers are **time-based** for now; event-based may follow.

### Approvals for unattended runs

Because scheduled tasks run unattended, pre-authorize sensitive Connector actions with `Always allow` for that Connector (or use read-only prompts). Recommended pattern: **read-only** prompts for unattended runs; **write actions** reserved for prompts you review before pre-authorizing.

### "Return value": notifications & results

- Result appears in sidebar with an **unread dot** (like any conversation).
- Open to review output, ask follow-ups, or jump to the associated chat to refine the prompt for the next run.

### Management

Edit (prompt/frequency/next run), Pause (stop runs, keep schedule), Delete (permanently; past results stay in history).

### Limits

Number of concurrent scheduled tasks depends on **plan**; pause/delete to free a slot or upgrade.

---

## 8. Files, Canvas & Context Inputs

**Concept:** Beyond Libraries and Connectors, Work accepts direct context inputs for a single chat: uploaded files and the Canvas review surface. These complement persistent Libraries.

### Files (single-chat context)

- Upload documents, spreadsheets, presentations, PDFs, images for Work to read, summarize, extract, or turn into reviewable outputs.
- Same supported formats as Libraries (see §5).
- Persist for the single chat only — use a Library for cross-session reuse.

### Canvas

- A reviewable, editable document surface where Work renders structured outputs (briefs, drafts, tables, reports).
- Outputs are treated as drafts until reviewed; you can read, edit, share, or reuse after verification.
- Some Workflows and Skills (e.g. `deep-research`, `doc-coauthoring`) open results in Canvas.

### Other context inputs

- **Web search & Open URL** — use public information or have Work read a specific web page.
- **Custom instructions** — persistent preferences (tone, format, language, recurring constraints) applied to all conversations.
- **Projects** — scoped groupings of conversations around a team/customer/initiative/topic.

---

## 9. Safety, Approvals & Supervision (Work)

**Concept:** Work is built for **delegation with supervision**. You see what the agent intends, follow progress in real time, and approve sensitive steps before they happen. Nothing important happens in a connected tool without your sign-off.

### Approval flow (sensitive actions)

Triggers on anything that creates, modifies, sends, posts, or deletes: send email (Gmail/Outlook), post to Slack, create/update/delete issues (Linear/GitHub), calendar events, Notion/SharePoint edits.

| Option | Behavior |
|--------|---------|
| Continue | Approve this action only; ask again next time. |
| Always allow | Pre-authorize this function for the session. |
| Decline | Cancel; Work skips or asks how to proceed. |

The **stop** button (black square) can interrupt a running action, but if it already reached the external system, consider it done.

### Per-function Connector permissions (detail)

- Interactive tools (create/update/delete/send/post): higher risk → manual approval recommended.
- Read-only tools (get/list/search): lower risk → safe to pre-authorize.
- Toggles are **per-user**.
- Revisit periodically as a function proves safer/riskier.

### Clarifying questions (pre-commit)

For non-trivial tasks Work asks follow-up questions before acting, e.g. *Which Project?*, *One page or full report?*, *Notion or Google Drive?*. Select an answer to continue, or rewrite the prompt.

### Supervision surfaces

| Surface | Purpose |
|---------|---------|
| Todos panel (right-hand) | Live checklist of completed/pending steps (no upfront plan to approve) |
| Reasoning summary | Per-step explanation (why, not just what); enables early correction |
| Tool-call transparency | Per tool: which tool, inputs, outputs, status; expandable for inspection |
| Stop button | Interrupt early; redirect with follow-up message |

### Org/workspace perimeter

- **Workspace admins** can disable specific Connectors workspace-wide, or force-enable specific Skills.
- **Organization admins** can toggle features per workspace (Skills, specific Connectors, MCP Connectors).
- **Knowledge Connectors** (Google Drive, SharePoint) require admin setup (Team/Enterprise).

### Pre-share checklist

- Did Work follow the todos it showed?
- Are the tool calls/Connectors the expected ones?
- Are facts, names, dates, numbers, citations accurate against source?
- For anything going to a customer/teammate/external party, was it read end-to-end?

---

## 10. Vibe Code — Coding Agent Loop & Surfaces

**Concept:** Vibe Code is Vibe's coding mode. With read/write access to your filesystem, a shell, and a configurable set of tools, it can read files, run commands, write code, and open PRs on your behalf, under your supervision. You stay in the loop: Vibe Code surfaces its plan, requests approval before sensitive actions, and can be interrupted at any step.

### Why use Vibe Code

- **Write code**: describe what to build; Vibe applies changes following project structure/conventions.
- **Explore codebases**: inspect files, explain architecture, trace how systems fit.
- **Review code**: find bugs, logic errors, missing edge cases, risky changes.
- **Debug**: drop in an error/failing test/stack trace; Vibe inspects context and proposes a targeted fix.
- **Automate dev tasks**: refactors, tests, migrations, setup, PR preparation.

### Three surfaces

| Surface | Where it runs |
|---------|---------------|
| CLI (`vibe`) | Local checkout (install + setup) |
| VS Code extension | Local checkout (install + authenticate) |
| Vibe Code Web | Remote sandbox against a GitHub repository |

### Behavior is configurable per agent and per environment

See [Safety, approvals, and permissions](https://docs.mistral.ai/vibe/code/safety-approvals-permissions). The approval model is consistent across CLI, VS Code, and Web; only how you act on approvals differs.

### CLI approval keyboard controls

| Key | Action |
|-----|--------|
| `Enter` / `1` / `Y` | Approve the action |
| `2` / `3` / `4` | Approve with broader scope (e.g. always allow this tool) |
| `N` | Reject |
| `Up` / `Down` | Move between approval options |
| `Escape` | Interrupt the agent |

The active agent and approval style are shown in the status line; switch mid-session with `/config` or restart with `vibe --agent <name>`.

### Programmatic mode

`vibe --prompt "…"` runs non-interactively. When `--agent` is omitted, Vibe falls back to **`auto-approve`** and interactive tools such as `ask_user_question` are disabled. Pass `--agent plan` explicitly for a read-only programmatic run.

---

## 11. Code Agents (Approval-Rule Bundles)

**Concept:** In Vibe Code, an **Agent** is a configuration override bundle: it bundles a system prompt, tool selection, and approval rules. It defines what gets auto-approved and can override the compaction prompt, model, available tools, and other settings. Distinct from Work/Chat "Agents" (which are specialized assistants).

### Built-in Agents

| Agent | Approval behavior | Use for |
|-------|-------------------|---------|
| `default` | Asks before running any tool | Day-to-day work where you vet each action |
| `plan` | Read-only. Auto-approves safe read tools, blocks edits/commands | Exploring an unfamiliar codebase or designing before changes |
| `accept-edits` | Auto-approves file edits; still asks for shell/sensitive tools | Larger refactors already scoped |
| `auto-approve` | Auto-approves every tool execution, including shell | Trusted, sandboxed environments only — **use with care** |

### Key rule

`default_agent` only applies in **interactive** sessions. In **programmatic mode** (`vibe --prompt …`), Vibe falls back to `auto-approve` when `--agent` is not provided, and interactive tools (e.g. `ask_user_question`) are disabled.

### Outside-CWD confirmation

Regardless of the active agent, Vibe asks for confirmation whenever a tool tries to **read, write, or run a command outside the current working directory**.

### Custom agents

Create/switch/set defaults via the CLI (`/vibe/code/cli/agents`) or VS Code extension (`/vibe/code/vs-code-extension/agents`). Custom agents can override system prompt, compaction prompt, model, tools, and approval rules.

---

## 12. Code Tools, Per-Tool Permissions & Trusted Folders

Vibe Code uses **layered controls**: Agents (approval bundles) + Trusted folders (project config) + Per-tool permissions (granular tool access).

### Per-tool permissions (`config.toml`)

Applies to built-in tools (`read_file`, `write_file`, `bash`, `grep`, etc.), MCP tools, and connector tools.

**Enable/disable tools globally** (exact names, globs, or regex):
```toml
enabled_tools = ["read_file", "grep", "task"]
disabled_tools = ["bash"]
```

**Per-tool permission** (`always` or `ask`):
```toml
[tools.read_file]
permission = "always"

[tools.bash]
permission = "ask"
```

**Bash allow/deny lists** (safe commands like `ls`, `pwd`, `cat`, `echo` auto-allowed by default):
```toml
[tools.bash]
permission = "ask"
allow = ["git status", "pnpm test"]
deny = ["rm -rf *"]
```

**MCP / connector tools** are addressed as `{server_name}_{tool_name}`:
```toml
[tools.fetch_server_get]
permission = "always"
```

### Trusted folders

A project can ship Vibe configuration (agents, skills, MCP servers, tool permissions) in a `.vibe/` directory at the repo root. Because that changes Vibe's behavior, it loads only from **explicitly trusted** directories.

- On starting an interactive session in a directory with trustable files (e.g. `.vibe/config.toml`) and unknown trust state, Vibe asks whether to trust it.
- Trusted directories are remembered in `~/.vibe/trusted_folders.toml`.
- Declined/untrusted → project config ignored (warning printed); user-level `~/.vibe/` config still applies.
- **Temporary trust** (CLI): `vibe --trust` trusts the working directory for the current invocation only, not persisted. Programmatic mode never prompts for trust — use `--trust` when scripting.

Before trusting, review `.vibe/` config like any repo code — especially MCP server definitions and tool permissions, which can grant access to external services or local commands.

### Best practices (Code)

- Start in `plan` for unfamiliar code.
- Commit before risky runs (instant rollback via `git reset --hard`).
- Use `accept-edits` for scoped refactors; review the diff before committing.
- Reserve `auto-approve` for disposable environments (containers, CI, ephemeral VMs); never on machines with SSH keys/cloud creds/production access.
- Trust deliberately; review `.vibe/` on first trust and whenever it changes.
- Tighten MCP/connector tools to `permission = "ask"` even under `accept-edits`.
- Version-control `.vibe/` config so the team can review/reproduce safety decisions.
- Revoke unused access (Web GitHub connections, MCP creds, connector authorizations).

---

## 13. Underlying Studio API (Developer Surface for the Same Primitives)

Vibe is the managed product surface. The same agent primitives are reachable programmatically through the **Studio Agents & Conversations API** (`https://api.mistral.ai/v1`), SDK beta namespace (`client.beta.agents`, `client.beta.conversations`, `client.beta.connectors`, `client.beta.libraries`). This section summarizes the developer-facing functions that back Vibe's agent capabilities (full detail in the sibling `mistral-api.md` study).

### Agents & Conversations

| Function | SDK | Key parameters |
|----------|-----|----------------|
| Create Agent | `client.beta.agents.create` | `model`, `name`, `description`, `instructions` (opt), `tools[]` (opt), `completion_args` (opt), `handoffs[]` (opt) |
| Update Agent | `client.beta.agents.update` | `agent_id`, plus fields to change (incl. `handoffs[]`) |
| Start Conversation | `client.beta.conversations.start` | `agent_id` **or** `model`, `inputs` (string or message list), `store` (opt, default true), `handoff_execution` (opt: `server`/`client`), `guardrails` (opt override) |
| Append to Conversation | `client.beta.conversations.append` | `conversation_id`, `inputs` (next message/reply/`FunctionResultEntry`) |

Three core objects: **Agent** (reusable pre-selected values), **Conversation** (persistent interaction history), **Entry** (typed action — message, tool execution, function call, handoff).

### Built-in tools (server-side, in `tools[]`)

| Tool `type` | Description | API support |
|-------------|-------------|-------------|
| `web_search` | Search engine access | Conversations + Agents (not Chat Completions) |
| `web_search_premium` | Search engine + news via verified providers | Conversations + Agents (not Chat Completions) |
| `code_interpreter` | Sandboxed code execution | Conversations + Agents (not Chat Completions) |
| `image_generation` | Generate images; returns `tool_file` chunk with `file_id` | Conversations, Agents, **and** Chat Completions |
| `document_library` | RAG over Libraries; takes `library_ids[]` | Conversations + Agents |
| `function` | Custom local tools via JSON schema; executed in your env | All |

### Connectors (managed MCP servers — Public Preview)

| Function | SDK (async) | Key parameters |
|----------|-------------|----------------|
| Create Connector | `client.beta.connectors.create_async` | `name` (≤64 chars, alnum/underscore/dash), `description`, `server` (MCP URL), `visibility` (`private`/`shared_workspace`/`shared_org`), opt `icon_url`, `headers`, `auth_data` (OAuth client_id/secret), `system_prompt` |
| Get auth URL | `client.beta.connectors.get_auth_url_async` | `connector_id_or_name`, `app_return_url` → returns `auth_url`, `ttl` |
| Get / List | `client.beta.connectors.get_async` / `list_async` | by name or UUID; list uses cursor pagination + `query_filters` (e.g. `active:true`) |
| List tools | `client.beta.connectors.list_tools_async` | `connector_id_or_name`; query: `page`, `page_size`, `refresh`, `pretty` |
| Update / Delete | `update_async` / `delete_async` | update by UUID; fields: `name`, `description`, `server`, `icon_url`, `system_prompt`, `headers`, `auth_data` |

**Attach in conversations/agents:** `{"type": "connector", "connector_id": "<name or UUID>", "tool_configuration": {...}}`. `tool_configuration` supports `include`/`exclude` (mutually exclusive) and `requires_confirmation` (list of tool names → human-in-the-loop).

### Human-in-the-loop (tool confirmation)

- Add `requires_confirmation: [<tool names>]` to `tool_configuration`.
- Conversation returns a pending `function.call` entry with `confirmation_status: "pending"`.
- Resume with `tool_confirmations: [{"tool_call_id", "confirmation": "allow" | "deny"}]` on the next append.
- Python SDK (v2.4+, `mistralai[mcp]`): `RunContext` + `register_func(requires_confirmation=...)` + `run_async`; raises `DeferredToolCallsException` with `dc.confirm()` / `dc.reject()`. Supports stateless serialize/resume via `deferred.to_dict()`.

### Handoffs (multi-agent orchestration)

- Create multiple agents, then `client.beta.agents.update(agent_id, handoffs=[other_agent_ids])`.
- No limit on chained handoffs.
- `handoff_execution`: `server` (default, runs on Mistral's cloud) or `client` (returns handoff to user to handle).
- Output events: `agent.handoff`, `tool.execution`, `message.output` (with `TextChunk`/`ToolReferenceChunk`/`ToolFileChunk` content).

### Libraries (RAG — backs Vibe Libraries)

| Function | SDK | Key parameters |
|----------|-----|----------------|
| Create Library | `client.beta.libraries.create` | `name`, `description` (opt) |
| List Libraries | `client.beta.libraries.list` | — (returns `nb_documents`) |
| Upload document | `client.beta.libraries.documents.upload` | `library_id`, `file` (`File(fileName, content)`) |
| List / Get / Status | `documents.list` / `.get` / `.status` | `library_id`, `document_id`; status `processing_status` = `Running`/`Completed` |
| Text content | `documents.text_content` | `library_id`, `document_id` → extracted text + signed URLs |
| Delete | `libraries.delete` / `documents.delete` | `library_id`[, `document_id`] |
| Access control | `libraries.accesses.list` (+ create/update/delete) | `org_id`, `level` (`Viewer`/`Editor`), `share_with_uuid`, `share_with_type` (`User`/`Workspace`/`Org`) |
| Connect to agent | tools entry | `{"type": "document_library", "library_ids": [<id>]}` |

### Vibe ↔ API bridge notes

- A Vibe Library's ID is in its URL; share with the Org (admin) for API agents to access. Reverse works too.
- Skills created in Studio can be `Publish to Vibe` to surface in Work.
- Workflows built in Studio and returning `ChatAssistantWorkflowOutput` can be `Publish in Vibe` to appear in Work's `+` > `Workflows` menu.
- Chat-legacy Agents (instructions + tools + knowledge) are **not** API-accessible; for API access, create agents via Studio or the REST API (`#tag/beta.agents`).

---

## 14. Capability Summary & Cross-Reference

| Capability | Vibe surface | Backing primitive | Configurable via | Approval model |
|-----------|--------------|-------------------|------------------|----------------|
| Task loop (Work) | Work chat | Conversations API + built-in tools | Custom instructions, Projects | Per-action (Continue/Always allow/Decline) |
| Specialized behavior | Skills (`SKILL.md`) | Agent Skills standard | Personal/Workspace/Force-enable | N/A (instructions) |
| External tools/data | Connectors | MCP servers (Connectors API) | Per-function `Always allow` | Per-action + per-function pre-auth |
| Knowledge grounding | Libraries | Libraries API + `document_library` tool | Library sharing/access | N/A (read-only search) |
| Deterministic automations | Workflows | Studio Conversational Workflows | Developer-built; tier-gated | Workflow-internal forms/confirmations |
| Autonomous scheduled runs | Scheduled tasks | Workflows infra (time triggers) | 5 frequencies; pause/edit/delete | Pre-authorize `Always allow` for unattended |
| Single-chat file context | Files & Canvas | File upload + Canvas | Per-chat | N/A |
| Public web info | Web search / Open URL | `web_search`/`web_search_premium` tools | Tool attach | N/A (built-in) |
| Coding agent loop | Vibe Code (CLI/VSCode/Web) | Local FS + shell + tools | Agents + config.toml | Per-tool + outside-CWD + agent mode |
| Code approval bundles | Code Agents | Agent config overrides | Built-in 4 + custom | `default`/`plan`/`accept-edits`/`auto-approve` |
| Code tool lockdown | Per-tool permissions | `config.toml` | `enabled/disabled_tools`, `permission`, bash allow/deny | `always`/`ask` |
| Code project config trust | Trusted folders | `.vibe/` directory | `--trust` (temp) or remembered | Trust prompt on first use |
| Multi-agent orchestration | (via Workflows / API) | Handoffs | `handoffs[]` + `handoff_execution` | `server` (auto) or `client` (manual) |
| Legacy specialized assistants | Chat Agents | (Chat only, not API) | Instructions + tools + knowledge | Per-conversation |

### Cross-platform notes

- **Same models, same patterns, same control** across all Vibe modes.
- Vibe Work and Vibe Code share the supervision philosophy (approve before sensitive actions, interruptible, transparent tool calls) but differ in surface and toolset.
- The Studio Agents & Conversations API is the developer escape hatch: anything configured in Vibe (Libraries, Connectors, Skills published to Vibe, Workflows published in Vibe) can be reached programmatically with the same IDs.
- Work is the recommended successor to Chat-legacy Agents; Skills replace Agents for reusable specialized behavior in Work.
- Programmatic Vibe Code (`vibe --prompt`) defaults to `auto-approve` and disables interactive tools — pass `--agent plan` for safe read-only scripted runs.