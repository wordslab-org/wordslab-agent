# AI Agent gateways

Here’s a comprehensive, up-to-date comparison of open-source AI agent gateways (not frameworks, not agents themselves — but the infrastructure layer that routes, connects, secures, and exposes agents).

Key Insights (important) - The ecosystem is splitting into 3 categories
- Infra-first → Agent Gateway
- Marketplace-first → AgentGate
- Platform-first → ForgeAI

🧠 What is an “AI agent gateway” (quick clarity)

An AI agent gateway is analogous to an API gateway, but for:

agent ↔ agent communication (A2A)
agent ↔ tool / LLM routing
observability, auth, billing, policy
discovery / orchestration

Think: “Kong / Envoy, but for agents”

🧩 Major Open-Source AI Agent Gateways (2026)
1. Agent Gateway

(Linux Foundation / Solo.io ecosystem)

🔑 Positioning
The most “canonical” open standard gateway
Built around MCP + A2A protocols
⚙️ Core features
Agent-to-agent + agent-to-tool routing
LLM routing (OpenAI-compatible API)
RBAC, JWT, TLS security
Observability + policy engine
Written in Rust (high performance)
✅ Strengths
Strong enterprise-grade governance
Backed by major players (Microsoft, AWS, Cisco)
Aligns with emerging open standards
❌ Weaknesses
Still early ecosystem maturity
Less focus on monetization / marketplace
🧠 Best for

Enterprises building production multi-agent systems

2. AgentGate
🔑 Positioning
“App Store for agents”
Focus on discovery + monetization
⚙️ Core features
Agent registry (“Agent Cards”)
Agent-to-agent chaining
Built-in billing (price per task)
Marketplace + ratings
SDKs (Python, TS)
✅ Strengths
Dead simple deployment (agentgate deploy)
Native economic layer
Great for open ecosystems
❌ Weaknesses
Less enterprise-grade security/policy
Centralized marketplace model
🧠 Best for

Startups & devs building agent ecosystems or SaaS agents

3. ForgeAI Gateway
🔑 Positioning
Full-stack self-hosted agent platform
Gateway is part of a larger system
⚙️ Core features
140+ REST endpoints + WebSocket
Multi-LLM routing (OpenAI, Claude, Gemini, local)
Built-in dashboard + RAG + tools
Strong security modules (17 layers)
✅ Strengths
All-in-one platform (gateway + agents + UI)
Strong local / private deployment
Built-in failover across LLM providers
❌ Weaknesses
Heavy (monolithic)
Less “plug-and-play” as a pure gateway
🧠 Best for

Teams wanting a self-hosted agent stack with gateway included

4. OpenClaw ecosystem gateways
🔑 Positioning
Rapidly growing open-source agent ecosystem
Multiple gateway-like implementations emerging
⚙️ Core features
Agent orchestration + tool execution
Integrations (WeChat, enterprise tools)
Variants like:
Tencent ClawBot
Nvidia NemoClaw
Alibaba Wukong
✅ Strengths
Explosive adoption globally
Rich ecosystem of extensions
❌ Weaknesses
Fragmented standards
Security concerns flagged by authorities
🧠 Best for

Experimental / cutting-edge agent ecosystems

5. Emerging / “Gateway-like” OSS tools

These are not pure gateways but often used as gateways in practice:

🧰 LiteLLM (proxy gateway)
Open-source LLM router
Unified API across providers
Often used as agent backend gateway

👉 Strength: lightweight, flexible
👉 Weakness: no native agent-to-agent protocols

🧰 Helicone
Observability-first gateway
Tracks usage, latency, failures

👉 Strength: analytics
👉 Weakness: not full orchestration

🧰 Kong AI Gateway (OSS core)
Extension of API gateway model
Familiar infra for backend teams

👉 Strength: production reliability
👉 Weakness: not agent-native

🧰 OmniRoute (community project)
“Wi-Fi router for AI calls”
Multi-account + fallback routing

👉 Interesting idea from community experimentation

⚖️ Side-by-Side Comparison
Feature	Agent Gateway	AgentGate	ForgeAI	OpenClaw ecosystem
Focus	Infra (proxy)	Marketplace	Full platform	Ecosystem
Protocols	MCP, A2A	Custom	Custom	Mixed
Agent discovery	❌	✅	Partial	Partial
Monetization	❌	✅	❌	❌
Security	⭐⭐⭐⭐	⭐⭐	⭐⭐⭐⭐	⭐⭐
Ease of use	Medium	Very easy	Heavy	Varies
Self-host	✅	✅	✅	✅
Maturity	High (enterprise)	Growing	Strong	Fragmented
🧠 Key Insights (important)
1. The ecosystem is splitting into 3 categories
Infra-first → Agent Gateway
Marketplace-first → AgentGate
Platform-first → ForgeAI
2. Protocol war is starting
MCP + A2A → becoming standard layer
Proprietary schemas → still dominant elsewhere

👉 This is similar to early Kubernetes vs custom orchestration

3. Missing piece (big opportunity)

No project yet fully nails:

open discovery
secure execution
economic incentives
standard protocols

👉 Whoever combines all 4 = likely winner

🏁 Recommendations (by use case)
🏢 Enterprise / production

→ Agent Gateway

🚀 Startup / monetized agents

→ AgentGate

🔒 Fully self-hosted stack

→ ForgeAI

🧪 Research / experimental ecosystems

→ OpenClaw-based stacks

If you want, I can:

map these to your specific architecture (LangGraph, CrewAI, etc.)
or 
design a reference architecture (gateway + agents + memory + tools) tailored to your stack

[Agent Gateway](https://agentgateway.dev/)

An open source data plane built on AI-native protocols (A2A & MCP) to connect, secure, and observe agent-to-agent and agent-to-tool communication across any framework and environment.

https://github.com/agentgateway/agentgateway

[AgentGate](https://agentgate.sh/)

The unified gateway to deploy, connect, and monetize AI agents via A2A + MCP + UCP.

AgentGate is an open-source infrastructure layer for the AI agent economy. Register agents, route tasks between them, charge per invocation, and deploy with one command. Think of it as Vercel + npm + Stripe, but for AI agents.

https://github.com/agentgatesh/agentgate

AI agents are everywhere, but there's no standard way to discover, connect, or pay them. AgentGate solves this:
- One-command deploy — agentgate deploy ./my-agent and it's live with HTTPS, health checks, and an A2A Agent Card
- Registry & discovery — Search agents by skill, tag, or name. A2A-compliant .well-known/agent.json
- Task routing — Route tasks between agents through a single gateway (REST, SSE streaming, WebSocket)
- Agent chaining — Build multi-step pipelines: agent A -> agent B -> agent C
- Built-in billing — Set a price per task, wallets auto-charge, transaction ledger included
- Organizations — Multi-tenant: create orgs with their own API keys, rate limits, and billing
- Marketplace — Browse and discover agents at agentgate.sh/marketplace
- Reviews & ratings — Community-driven quality signals on every agent
- Plugin system — Pre/post task hooks for logging, filtering, or custom logic
- Health monitoring — Background health checks every 60s for all registered agents
- Metrics & dashboards — Live request/latency/error tracking per agent
- Rate limiting — Token bucket per IP (global) + per-org custom limits

[Forge AI](https://www.getforgeai.com/)

Run your own AI assistant. Connect any messaging app. Use any LLM. Own every byte of your data.

ForgeAI is a production-ready, fully self-hosted AI assistant platform built from scratch in TypeScript. Connect any LLM to WhatsApp, Telegram, Discord, Slack, Teams, Google Chat, WebChat & IoT devices — all managed through a modern 19-page dashboard with 17 security modules.

Dashboard (19 Pages)
- Overview	System health, uptime, active agent info (model, thinking level, temperature), security module status (clickable toggles), alerts, OpenTelemetry spans/metrics
- Chat	Interactive chat with session history sidebar, real-time execution step viewer (tool calls + results with expandable details), session persistence across restarts, agent selector for multi-agent
- Tools	Built-in tools explorer with parameters + MCP Servers tab (add/connect/reconnect, list tools and resources from connected servers)
- Usage	Token consumption by provider and model, estimated cost tracking, real-time provider credit balances (Moonshot, DeepSeek), usage history table with latency
- Plugins	Plugin store with categories, enable/disable toggle, template generator (Plugin SDK scaffolding)
- Channels	Per-channel status, token configuration via encrypted Vault, DM Pairing panel (generate/revoke FORGE-XXXX invite codes)
 -Agents	Multi-agent CRUD, per-agent model/provider/persona/tools config, routing bindings
 -Workspace	Live editor for 5 prompt files: AGENTS.md, SOUL.md, IDENTITY.md, USER.md, AUTOPILOT.md
- Gmail	Inbox viewer (paginated), compose with To/Subject/Body, search, mark read/unread, thread view
- Calendar	Google Calendar integration: list/create/edit/delete events, quick add (natural language), free/busy check
- Memory	Cross-session memory browser (MySQL-persistent), semantic search (OpenAI embeddings + TF-IDF fallback), importance scoring, entity extraction, consolidate duplicates
- API Keys	Create keys with 12 granular scopes, set expiration (days), view usage count, revoke/delete
- Webhooks	Outbound webhooks (URL + events), inbound webhooks (path + handler), event log with status/duration/timestamp
- Audit Log	Security event viewer with risk level color coding, action filtering, detail expansion
- Settings	Provider API key management (validated via test call before saving, stored encrypted), system configuration
- RAG	Document upload (PDF/TXT/MD/code), semantic search with score display, config editor (chunk size, embedding provider), persistence
- Voice	Text-to-Speech (OpenAI TTS) and Speech-to-Text (Whisper), voice input/output for agent interactions
- Canvas	ForgeCanvas: live visual artifacts (HTML, React, SVG, Mermaid, Charts, Markdown, Code) rendered in sandboxed iframes with bidirectional agent↔artifact interaction
- Recordings	Session Recording & Replay: record full agent sessions, timeline player with play/pause/scrub, step-by-step visualization (messages, tool calls, thinking, progress)

ForgeAI offers two desktop applications with different purposes: Desktop App (packages/desktop)	 //  Companion (packages/companion)
- Framework	Electron	Tauri 2 + React + Rust
- Purpose	Lightweight wrapper for the Dashboard — opens it as a native desktop window instead of a browser tab	Smart client that connects to the Gateway and lets the AI control your Windows PC (desktop automation, voice, file management)
- Platforms	Windows, macOS, Linux	Windows 10/11 only
- Requires Gateway?	No — embeds the Dashboard UI	Yes — pairs with a Gateway via WebSocket
- Key features	System tray, global hotkeys, auto-update, startup on boot	Pairing, voice mode, desktop automation, dual-environment routing, safety system
- When to use	You want a native app to access the Dashboard without opening a browser	You want the AI to interact with your Windows PC remotely (e.g., Gateway on VPS, Companion on your desktop)

# AI agent sandboxes

[AI Agent Sandboxes](https://rywalker.com/research/ai-agent-sandboxes) - Published February 17, 2026

Market Leaders (11 platforms): E2B, Sprites, Daytona, Modal, Northflank, OpenSandbox, Quilt, Runloop, CodeSandbox SDK, ComputeSDK, Zeroboot

Key Findings:
- E2B dominates with 200M+ sandboxes started and Fortune 100 adoption
- Sprites (Fly.io) challenges the ephemeral model with persistent VMs and instant checkpoint/restore
- Daytona offers the fastest creation (90ms) and unique Computer Use support
- Modal leads for GPU workloads with serverless Python infrastructure
- Northflank brings enterprise-grade VPC deployment with GPU support and flexible ephemeral/persistent modes
- OpenSandbox (Alibaba) introduces a protocol-driven open-source approach with multi-language SDKs and Kubernetes-native scaling
- Zeroboot achieves sub-millisecond spawns (0.79ms) via copy-on-write forking of Firecracker snapshots — 190x faster than E2B

The market is splitting into ephemeral (security-first) vs persistent (productivity-first) camps

Rather than retrofitting a container orchestration tool, hopx.ai starts from the assumption that the code being executed is untrusted, ephemeral, and needs a full development environment — not just a compute shell.

Firecracker microVMs

Every sandbox runs in a Firecracker microVM — the same technology AWS Lambda uses. Dedicated kernel per session, hardware-level isolation, ~100ms cold start. This is critical because Docker is not a sandbox — there is strong community consensus that shared-kernel containers do not provide adequate isolation for untrusted AI-generated code.

Full-Stack Environments

A coding agent sandbox on hopx.ai can include databases, API servers, Redis, message queues — anything your application needs. This addresses the ephemeral vs persistent debate: E2B argues ephemerality is essential for security, Fly.io says it is obsolete for stateful agent work. hopx.ai supports both modes so you choose per workload.

MCP Server for AI IDEs
hopx.ai provides a native Model Context Protocol (MCP) server, following the approach Anthropic pioneered for code execution via MCP. Unlike Cloudflare Code Mode (which gives the agent one tool that executes TypeScript) or AIO Sandbox (an all-in-one Docker container combining Browser, Shell, File, MCP, and VSCode Server), hopx.ai MCP provides granular sandbox controls that AI IDEs like Cursor, Windsurf, and Claude Desktop discover automatically.

Claude Code Skill
A dedicated Claude Code skill lets Claude create, execute, and destroy sandboxes mid-conversation. This is especially important because Claude Code ships with sandboxing off by default (Bubblewrap on Linux, Seatbelt on macOS) — the skill provides a stronger isolation boundary than the built-in opt-in sandbox, with full environment support.

Per-Sandbox Network Policies
Configure egress and ingress rules per sandbox. Whitelist specific domains, block all outbound by default, or open specific ports for multi-service communication. This also addresses the environment variable gap — the biggest security blind spot the community has identified — by letting you control what secrets are injected and what network paths are available to exfiltrate them.

BYOC Deployment
Run hopx.ai on your own cloud account (AWS, GCP, Azure) or on-prem. Your agent code and data never leave your infrastructure. E2B is also self-hostable (Apache-2.0), but only provides compute sandboxes. Other self-hosted options like microsandbox (libkrun-based) and OpenSandbox (Kubernetes-native) are emerging but lack full-stack and MCP support.

Integration Patterns: Connecting Your AI Agent to a Sandbox
There are three primary ways to connect a coding agent to a hopx.ai sandbox, depending on your architecture:

Model Context Protocol (MCP)
The recommended path for AI IDEs. Add the hopx.ai MCP server to Cursor, Windsurf, Claude Desktop, or any MCP-compatible client. Unlike approaches like Cloudflare Code Mode — which gives the agent a single tool that executes TypeScript in a sandbox, dramatically reducing token usage — hopx.ai MCP exposes granular sandbox lifecycle controls. The AI model discovers sandbox tools automatically and can create, execute, read files, and destroy sandboxes as part of its normal tool-use flow. Zero custom code required.

💡
MCP is the recommended integration path for AI IDEs. The AI model discovers sandbox tools automatically — no custom code required.

REST API + SDKs
For custom agent frameworks and backend services. The hopx.ai REST API lets you programmatically create sandboxes, execute commands, upload/download files, and stream stdout/stderr. SDKs are available for TypeScript, Python, and Go. Create a sandbox in three lines of code and wire it into your agent's tool-use layer.

Claude Code Skill / CLI
For developers using Claude Code directly. Install the Bunnyshell skill and Claude gains the ability to spin up sandboxes on demand during any conversation. Use the hopx CLI for scripting and CI/CD pipelines — create sandboxes, run commands, and tear down environments in shell scripts or GitHub Actions workflows.

https://www.bunnyshell.com/guides/coding-agent-sandbox/

[NVIDIA OpenShell](https://docs.nvidia.com/openshell/latest/)

NVIDIA OpenShell is the safe, private runtime for autonomous AI agents. It provides sandboxed execution environments that protect your data, credentials, and infrastructure. Agents run with exactly the permissions they need and nothing more, governed by declarative policies that prevent unauthorized file access, data exfiltration, and uncontrolled network activity.

https://github.com/NVIDIA/OpenShell

[OpenShell community sandboxes](https://docs.nvidia.com/openshell/latest/sandboxes/community-sandboxes.html)

community ecosystem around OpenShell -- a hub for contributed skills, sandbox images, launchables, and integrations that extend its capabilities.

https://github.com/NVIDIA/OpenShell-Community

[AIO Sandbox - All-in-One Agent Sandbox Environment](https://sandbox.agent-infra.com/)

One Docker container with shared filesystem. Files downloaded in the browser are instantly accessible in Terminal and VSCode.

Built‑in VNC browser, VS Code, Jupyter, file manager, and terminal—accessible directly via API/SDK.

Isolated Python and Node.js sandboxes. Safe code execution without system risks.

Pre-configured MCP Server with Browser, File, Terminal, Markdown, Ready-to-use for AI agents.

Cloud-based VSCode with persistent terminals, intelligent port forwarding(via `${Port}-${domain}/` or `/proxy`), and instant frontend/backend previews

Enterprise-grade Docker deployment. Lightweight, scalable.

https://github.com/agent-infra/sandbox

[LLM Sandbox](https://vndee.github.io/llm-sandbox/)

LLM Sandbox is a lightweight and portable sandbox environment designed to run Large Language Model (LLM) generated code in a safe and isolated mode. It provides a secure execution environment for AI-generated code while offering flexibility in container backends and comprehensive language support, simplifying the process of running code generated by LLMs.

https://github.com/vndee/llm-sandbox

Self-host open-source LLM agent sandbox on your own cloud

https://blog.skypilot.co/skypilot-llm-sandbox/

[Alibaba - OpenSandbox](https://open-sandbox.ai/)

OpenSandbox - Universal Sandbox Infrastructure for AI Applications
Securely run commands, filesystems, code interpreters, browsers, and developer tools in isolated runtime environments.

https://github.com/alibaba/OpenSandbox

The OpenSandbox architecture consists of four main layers:

SDKs Layer - Client libraries for interacting with sandboxes
Specs Layer - OpenAPI specifications defining the protocols
Runtime Layer - Server implementations managing sandbox lifecycle
Sandbox Instances Layer - Running sandbox containers with injected execution daemons

https://open-sandbox.ai/overview/architecture

[E2B - AI Sandboxes](https://e2b.dev/)

https://github.com/e2b-dev/infra/blob/main/self-host.md

[Modal Sandboxes](https://modal.com/products/sandboxes)

https://modal.com/docs/guide/sandboxes

[Daytona - run AI code](https://www.daytona.io/)

https://github.com/daytonaio/daytona/tree/main/docker

[KAgent and KMCP - Bringing Agentic AI to cloud native](https://kagent.dev/)

https://github.com/kagent-dev/kagent