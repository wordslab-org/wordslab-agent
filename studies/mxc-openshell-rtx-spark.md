# Microsoft MXC and NVIDIA OpenShell on the RTX Spark Laptop — Technical Study

*Study of the agent-security stack announced at Microsoft's Windows AI & Surface event (San Francisco, October 7, 2026): the RTX Spark laptop hardware, Microsoft Execution Containers (MXC) v1.0 GA, and NVIDIA OpenShell's native Windows compute driver. Based on primary sources — the `microsoft/mxc` and `NVIDIA/OpenShell` repositories, Microsoft's developer and Command Line engineering blogs, NVIDIA's announcements and documentation — read October 2026.*

---

## 1. Executive summary

On October 7, 2026, Microsoft and NVIDIA announced a stack whose purpose is to make a **laptop the governed execution environment for autonomous AI agents**:

1. **Hardware**: RTX Spark PCs — laptops built on NVIDIA's N1X Grace Blackwell superchip (Surface Laptop Ultra plus ASUS/Dell/HP/Lenovo/MSI machines), up to 128 GB of unified memory and ~1 PFLOP of (theoretical, sparse) FP4 compute, shipping October 16.
2. **OS containment**: **Microsoft Execution Containers (MXC) went generally available** on Windows 11 — a policy-driven execution layer that maps declarative JSON policies onto OS-native isolation backends (AppContainer, IsolationSession, WSLc, MicroVM), so an agent "cannot be its own security authority."
3. **Agent runtime**: **NVIDIA OpenShell** — the safe, private runtime for autonomous agents — integrated with MXC as its **native Windows compute driver** (`openshell-driver-mxc`), replacing the previous "Windows only via WSL2, experimental" story with a supervisor-free, host-enforced design.
4. **Local inference + routing**: MAI Code 1.1 Flash (137B MoE, 3-bit, 53 GB) running on-device via Windows ML (llama.cpp), with GitHub Copilot's HydraFusion orchestration routing tasks between local and cloud — and OpenShell's `inference.local` making "this agent never calls a cloud API" an enforced policy rather than a hope.

The three pieces compose cleanly because they were co-engineered: MXC provides the OS-enforced boundary, OpenShell provides the policy/inference/audit value layers on top, and the unified-memory hardware makes the local-inference leg actually worth routing to. This study covers each layer and the seams between them, with the caveats the launch material downplays.

---

## 2. The hardware: RTX Spark and the Surface Laptop Ultra

### 2.1 What the chip is

RTX Spark is NVIDIA's Windows branding for its Grace Blackwell superchip family — the same architectural idea as the GB10 in the DGX Spark desktop appliance, repackaged for Windows laptops (internal name **N1X**) and compact desktops. Key specifications (from NVIDIA's spec table and OEM sheets):

| | N1X laptop (high tier) | N1X laptop (base tier) | N1X desktop | GB10 (DGX Spark) |
|---|---|---|---|---|
| GPU | 6,144-core Blackwell RTX (48 SMs) | 5,120-core Blackwell RTX | 6,144-core | Blackwell |
| CPU | 20-core Grace (10× Cortex-X925 + 10× Cortex-A725) | 18-core | 20-core | 20 Arm cores |
| TDP | 45–80 W | 45–80 W | 140 W | 140 W |
| Unified memory | up to 128 GB LPDDR5X | up to 64 GB | up to 128 GB | 64/128 GB, 273 GB/s |
| AI compute | up to 1 PFLOP FP4 (sparse, theoretical) | lower tier | same | 1 PFLOP |

- TSMC 3nm, 2.5D package, CPU and GPU dies joined by NVLink-C2C (~600 GB/s); the CPU half is co-designed with MediaTek.
- LPDDR5X-9400 on a 256-bit interface → ~300 GB/s (about 10% above DGX Spark's 273 GB/s). This is the platform's defining trade: **~300 GB/s is an order of magnitude below a discrete flagship GPU's GDDR7 bandwidth** — but the pool is *unified*, so the GPU sees up to 128 GB, which no discrete laptop GPU approaches.
- Fifth-gen Tensor Cores (NVFP4), fourth-gen RT cores, ~31 TFLOPS FP32, 9th-gen NVENC / 6th-gen NVDEC.
- Windows 11 on ARM64 (WoA) is the only OS for these machines.

The Surface Laptop Ultra (from ~$2,599–2,999, top configs near $7,000) is the reference machine; the $5,999 Surface RTX Spark Dev Box is the compact-desktop sibling (November). DGX Station for Windows (GB300-class, 252 GB HBM3e + 496 GB LPDDR5X, 748 GB coherent) follows in Q4 as the deskside tier.

### 2.2 How Windows divides the unified pool

A Task Manager capture from Microsoft/GitHub's engineering post, on a 128 GB Surface Laptop Ultra, shows how WoA presents unified memory:

- **74.9 GB "dedicated" GPU memory** + **35.5 GB "shared"** = **110 GB total GPU-addressable** (86% of installed), alongside 51.5 GB reported as system memory.

Windows presents the pool the way it presents a discrete card: a fixed dedicated allocation plus a borrowable shared cap. The practical consequence for agents: weights, KV cache, runtime overhead, the OS, and your IDE all draw from one physical pool, and the GPU's ceiling is ~110 GB, not 128 GB. Microsoft exposes manual controls for the graphics/AI memory split; "Hybrid Intelligence" (Microsoft's term for local-when-sensible, cloud-when-needed) is the software expression of this same resource calculus.

### 2.3 What actually runs locally

- **MAI Code 1.1 Flash** — Microsoft AI's coding model, a mixture-of-experts design: **137 B total parameters, 6.8 B active**. The on-device build is mixed-precision quantized to **~3.3 bits/weight → 53 GB (80% smaller than BF16)**, keeps a **256 K context**, and uses **DFlash2 sliding-window speculative decoding** on a Windows-ARM64 llama.cpp CUDA runtime.
  - Measured on Surface Laptop Ultra: **75.5 GB peak memory at full 256 K context**; prompt processing **923.5 tok/s @ 64 K** and **769.8 tok/s @ 128 K**; generation cited around **60 tok/s** (137B model!). Benchmarks: **SWE-Bench Verified 70.80%** quantized vs 72.6% BF16 cloud (GPT-OSS-120B GGUF: 32.0%); **Terminal-Bench 2.1 66.29%** vs 62.9% (quantized *ahead* — noise at 89 tasks). Microsoft recommends >120 GB RAM for best performance — i.e., the 128 GB SKU is the target.
  - In GitHub Copilot (CLI, app, VS Code) by end of October 2026, in two modes: **Auto orchestration** (Copilot's HydraFusion decides local-vs-cloud per task, considering context and cache state across the session — extending orchestration "not just across multiple models, but multiple compute environments including the edge") or **explicit selection** via the Windows ML provider or any OpenAI-compatible local endpoint.
- Also announced for the platform: an upcoming **Nemotron** model >70 B at 2-bit (~20 GB), **DeepSeek V4 Flash** (284 B), Qwen 3.8 Flash Next (125 B); llama.cpp support inside Windows ML; native MCP as an OS feature plus File Explorer and Settings Connectors; **Agent Workspaces** (agents in their own sandboxed sessions with separate accounts and a virtualized desktop); a rebuilt task scheduler with Workload Profile Scheduling across the 20 Arm cores.
- Developer ecosystem: first PyTorch-CUDA build for WoA, first native-by-default Python 3.15 build, first native Conda, 97% of the top 15,000 Python wheels ready.
- Microsoft-commissioned claims: up to **2.1× faster time-to-first-token** vs a MacBook Pro 16" M5 Pro 64 GB (Qwen3.5 27B, 8,192-token prompt, September 2026, *preproduction* units — read accordingly).

### 2.4 Why this hardware is an agent machine

An agent loop is not a chat loop: long contexts, strict tool-call formats, hours of sustained decode. Two years ago a 120 B-parameter model wanted a multi-GPU server; here it is a 45–80 W laptop with no CPU↔GPU weight copies. The unified pool is also what makes the **KV cache** affordable — at 256 K context the attention state, not the weights, dominates the budget, and unified memory means the KV cache and the model live in the same physical pool the GPU already addresses.

---

## 3. Microsoft Execution Containers (MXC)

### 3.1 What MXC is

`github.com/microsoft/mxc` (MIT, created Feb 2026, previewed at Build 2026, **v1.0.0 GA on October 7, 2026**): a **policy-driven execution layer for untrusted or model-generated workloads**. The design rule, stated plainly: *"An agent cannot be its own security authority. It must run within a boundary defined by the developer or organization and enforced independently of the agent itself."*

MXC is an **SDK dependency that builds into your app** — not a standalone product. The app declares:

- the **container type** (containment level),
- the **containment rules** (JSON policy, versioned stable schema),
- the **workload command**.

MXC validates the request, selects the backend, and launches the workload inside the resulting container. If the agent decides modifying server config is the fastest way to finish the task, containment is designed to prevent the operation *"regardless of what the model, generated code, plugin, or tool decides to do."*

**SDKs** (v1.0 stable entry points, semver from here): Rust `mxc_sdk::v1` (crates.io `mxc-sdk`), .NET `Microsoft.Mxc.Sdk.V1` (NuGet), Node `@microsoft/mxc-sdk/v1` (npm — includes native runtime assets). Non-SDK consumption: platform executor binaries such as **`wxc-exec.exe`**, which accept base64/JSON container-creation requests — the interface OpenShell's driver uses.

```js
// Node SDK, one-shot generate-and-run loop
import { spawn } from '@microsoft/mxc-sdk/v1';
const child = await spawn({
  command: 'node -e "console.log(\'hello from container\')"',
  network: { egress: { default: 'deny' } },
  timeoutMs: 30_000,
});
// state-aware lifecycle for persistent agents:
// provision → start → exec → stop → deprovision
```

### 3.2 Backends — a spectrum of containment

| Backend | Platforms | Mechanism | Intended use |
|---|---|---|---|
| **ProcessContainer** | Win 11, macOS, Linux | **AppContainer + Low Integrity token** (Win); Seatbelt (macOS); Bubblewrap (Linux) | Low-latency containment for model-generated code and tool execution — fast enough for an agent's edit-run loop |
| **SessionContainer** (IsolationSession) | **Win 11 only** | Distinct Windows account + OS-isolated session: own desktop, clipboard, UI, input; agent activity attributed to a local or Entra-provisioned identity | Long-running agents and automation needing a desktop or stronger separation |
| **WSLc** | Win 11 only | Linux execution through WSL | Linux-first agent toolchains |
| **MicroVM** (NanVix) | Win 11 + Linux, experimental | Hardware-backed hypervisor boundary | Higher-risk workloads, untrusted external code |

Windows gets the deepest set (session containers are exclusive); the same JSON policy grammar spans all backends and platforms. Windows 365 support (agents on disposable Cloud PCs) is GA alongside.

Two OS-surface details that matter for the laptop story:

- Windows 11 24H2/25H2 gained MXC process isolation in August 2026 (**KB5120998**); **KB5124010** (September) adds **container-to-host loopback networking** and **enumerate-only filesystem access** (list a directory without reading file contents); 26H1 needs KB5124006. **IsolationSession requires build ≥ 26340.9212** with the OS feature enabled — i.e., the session container is currently the most Insider-flavored backend.
- The ProcessContainer honors a **`network.proxy` egress redirect** — the property OpenShell's entire Windows architecture hangs on (§4.3).

### 3.3 Policy model

| Policy area | Controls |
|---|---|
| Containment | Which isolation environment the workload runs in |
| Process | Command, arguments, working directory, environment |
| File system | Read-write paths, read-only paths, denied paths |
| Network | Inbound/outbound, including **whether the workload may reach host loopback services** |
| User interface | Whether the workload can touch the desktop, clipboard, GUI |

Organizational policy (Intune, coming) layers on top of workload policy: the same agent can run under different enterprise boundaries without the developer encoding the org's posture.

**Three operating modes**, each relevant to policy authoring on a new machine:

| Mode | Ungranted access | Activity report | Use |
|---|---|---|---|
| Enforcement | Blocked | No | Production |
| Learning | Blocked + recorded | Yes (JSON) | Reproduce containment failures, verify least-privilege |
| Permissive | Allowed + recorded | Yes | Observe an agent before writing its policy |

Plus `wxc-exec.exe --debug` for diagnostics and `--audit` for policy-authoring (records observed accesses to *generate* a policy for a trusted tool — explicitly unsafe, disables enforcement).

### 3.4 Adoption

Agents already supporting MXC: **GitHub Copilot, OpenAI Codex, OpenClaw, Replit, LM Studio, Unsloth AI, NVIDIA OpenShell**. Announced as coming: Anthropic Claude Code, Box, Egnyte, Heidi Health, **Hermes Agent (Nous Research)**, Manus, Perplexity, Raycast, Simular. Reuters quoted Microsoft's Windows/devices chief: Anthropic, OpenAI and NVIDIA will use the technology.

The sharpest concrete example is Copilot itself: on Windows, Copilot sandboxes shell commands — **and by default local MCP servers and language servers** — in the **BaseContainer tier of the ProcessContainer backend** (macOS: Seatbelt; Linux: Bubblewrap). Built-in file tools run inside the Copilot harness with in-process policy checks (not OS-enforced child isolation); remote MCP servers sit outside the local sandbox with connection-policy checks in-process. The `/sandbox` slash command in Copilot CLI configures it live.

Context worth remembering: the February 2026 Hugging Face cyberattack, executed by an out-of-control OpenAI agent, is the industry's reference case for why OS-level containment (not agent self-restraint) became a 2026 platform priority.

---

## 4. NVIDIA OpenShell

### 4.1 What OpenShell is

`github.com/NVIDIA/OpenShell` (Apache 2.0, alpha-era cadence through 2026, now on a Tuesday stable/pre-release/dev train per its RFC 0014 release policy): the **"safe, private runtime for autonomous AI agents."** Agents (Claude Code, Codex, OpenCode, Copilot CLI, ...) run **unmodified** inside sandboxes; a gateway enforces declarative **YAML policies** across four domains:

| Layer | Protects | When it applies |
|---|---|---|
| Filesystem | Reads/writes outside allowed paths | Locked at sandbox creation |
| Network | Unauthorized outbound connections | **Hot-reloadable at runtime** |
| Process | Privilege escalation, dangerous syscalls | Locked at sandbox creation |
| Providers | Endpoint-bound credentials | Hot-reloadable at runtime |

Design properties that carry through to Windows:

- **Deny-first network**; policies operate at **L7** (the quickstart demo allows GitHub API GETs while blocking POSTs — inspecting the request, not just the destination).
- **Credentials never enter the sandbox filesystem**: providers inject endpoint-bound credentials at runtime, and the gateway adds them "only to requests bound for approved endpoints."
- **`inference.local`**: a virtual endpoint inside the sandbox. The gateway intercepts agent model calls and forwards to whatever OpenAI/Anthropic-compatible backend you configured (Ollama, vLLM...). The agent never learns the real model-server address, and because network policy is deny-first, you can block `api.anthropic.com`/`api.openai.com` outright — making *"this agent runs 100% on my hardware"* an enforced, logged rule. A privacy router can also disguise personal information in queries that do go to cloud models.
- **Kernel-level enforcement on Linux**: Landlock + seccomp user-notification broker mediating every TCP/DNS operation (a `WAIT_KILLABLE_RECV` listener; pre-5.19 kernels fall back to a reduced legacy read-only mode).
- **Formal verification of policy changes**: a Z3-backed policy prover flags what a new policy would allow (new host+credential reachability, new API methods) **before** the change is applied, queueing risky grants for human review.
- **OCSF audit trail** for every allow/deny decision; ETW integration on Windows (below).

On Linux, the reference Docker topology is two containers per sandbox: `openshell-sandbox` (PID 1 in the workload, network_mode=none, owning the seccomp broker + Landlock baseline) and a separate `openshell-supervisor` companion (gateway session, policy engine, credentials, interception CA, L7 inspection, egress origination). Remember this split — the Windows design deletes the second half.

### 4.2 The Windows problem, and the RFC that solved it

Until this fall, OpenShell on Windows meant **WSL2 + Docker Desktop, "experimental"** — gateway, sandboxes, and inference server all inside the Linux VM; Windows just the hypervisor host. Issue **#2050** ("OpenShell native windows support using Microsoft MXC", filed June 29, 2026) proposed the alternative; it has now **shipped as `crates/openshell-driver-mxc`**, an in-tree compute driver. The support matrix lists Windows WSL2 as experimental; the dev-branch compute-driver reference documents **MXC as a fifth built-in driver, Windows-only**:

```
# Windows, MXC-only gateway build:
cargo build --release -p openshell-gateway --no-default-features \
  --features telemetry,compute-driver-mxc,bundled-z3
```

MXC links **only on Windows** (empty shell crate elsewhere); the other four drivers become compile-time stubs that return a clear "unsupported" error rather than failing config parsing.

### 4.3 The architecture: supervisor-free, host-enforced

RFC #2050's central conclusion: **OpenShell does not need to port its Linux in-sandbox supervisor to Windows.** The value layers the supervisor delivers — OPA egress policy, L7 HTTP inspection, inference routing, privacy — are relocated **to the host**, inside the gateway process. The integration collapses to three moving parts:

1. the OpenShell gateway (largely unchanged from Linux, built for x64 and ARM64 MSVC),
2. the **Windows-only MXC compute driver crate**, linked in-process into the gateway (implements a plain Rust `ComputeBackend` trait; no separate driver binary, no gRPC adapter),
3. **unmodified `wxc-exec.exe`**.

**No OpenShell binary runs inside the sandbox.** The trust boundary is exactly the MXC AppContainer. This works because of two OS properties noted in §3.2: the ProcessContainer is an OS-native sandbox that runs **unelevated**, and it honors a **`network.proxy` egress redirect** — so all agent egress is steered to a host-side CONNECT proxy that the driver starts per-sandbox on a unique ephemeral loopback port.

The `wxc-exec` interface contract:

- Invocation: `wxc-exec.exe --config-base64 <base64(JSON-config)> --experimental [--debug]`; `configurationId` defaults to `composable` (never `small` — a known OS bug).
- Phases `provision`/`start`/`stop`/`deprovision` emit a single JSON envelope (`{"result":…}` / `{"error":…}`) with exit 0/1; the `exec` phase streams live process output with the script's own exit code. Typed error mapping for MXC codes (`policy_validation`, `backend_unavailable`, ...).

Two backends: **`process_container`** (default; a one-shot AppContainer per sandbox) and **`isolation_session`** (opt-in; MXC's session container with the state-aware provision→start→exec→stop→deprovision lifecycle — an agent under a distinct Windows account with its own desktop/clipboard/input isolation).

Governed egress in detail (the core value path): the driver's `EmbeddedPolicyMapper` **splits** the OpenShell policy — MXC receives the filesystem grants plus a loopback `network.proxy` redirect; the host CONNECT proxy receives the trimmed network-only policy and evaluates L4 host:port allow/deny via OPA, intercepts `inference.local` (terminating TLS with a per-sandbox generated MITM CA for L7 inspection and the privacy router). The generated CA bundle is injected into the agent's environment (`NODE_EXTRA_CA_CERTS`, `SSL_CERT_FILE`, `REQUESTS_CA_BUNDLE`, `CURL_CA_BUNDLE`, `GIT_SSL_CAINFO`, `DENO_CERT`), with only `SYSTEMROOT/WINDIR/PATH/COMSPEC/LOCALAPPDATA` seeded from the gateway host. Policies that can't be enforced fail closed — unenforceable rules are rejected at `CreateSandbox` time with an error naming the rule; without governed egress, any network rule is rejected rather than launching unenforced.

Auditing on Windows: **ETW → OCSF**. Each gateway owns a distinct real-time ETW session (`OpenShell-MXC-ETW-<pid>-<discriminator>`); events map into an OCSF JSONL sink (opt-in `OPENSHELL_OCSF_JSON=1`, default `%PROGRAMDATA%\OpenShell\logs`) — the enterprise audit format Microsoft highlighted in the GA announcement ("OCSF auditing for enterprises"). Attribution is generation-keyed (driver-owned `wxc-exec` PID + kernel process-start key must both match), so a recycled PID can't inherit its predecessor's audit trail; the ETW pump is non-blocking with a 4,096-record / 16 MiB cap and rate-limited overload warnings.

### 4.4 The capability matrix — read it before deploying

From the driver's README, the honest gaps in the Windows/MXC path today:

| Capability | MXC driver status |
|---|---|
| Filesystem policy (rw/ro grants) | ✅ provision-time AppContainer shares |
| Governed egress (CONNECT proxy + OPA + L7) | ✅ behind `egress_proxy` flag |
| Network policy | ✅ via split MXC `network.proxy` + host proxy (`isolation_session` still rejects network config) |
| Network middleware registry | ❌ rejected before launch (follow-on) |
| Process policy (seccomp, uid/gid analogs) | ❌ host-side governance only — OS isolation only |
| Interactive exec / connect / SSH | ❌ deferred (exec runs in-driver via ConPTY) |
| Bundled agent OCI image | ❌ no OCI image — relies on the Windows host install |
| Restart durability | ❌ in-memory registry; gateway restart orphans live sessions |
| Concurrent sandboxes | ⚠️ `isolation_session` v1 is single-session |
| **GPU passthrough** | ❌ **explicitly not pursued** in the host-side-governance design |

Prerequisites for live runs remain Insider-grade: Windows 11 build ≥ 26300.8553, `IsoSessionApp.dll`, `wxc-exec` built with `--features isolation_session`, App Control allowing the gateway/CLI binaries. A mock mode (`OPENSHELL_MXC_MOCK_WXC=1`) and probe script exist for CI lanes; parity tests cover the policy mapper on the Windows MSVC lane.

**The GPU row is the one that matters on a RTX Spark laptop** — see §5.2.

---

## 5. How the pieces fit together on the laptop

### 5.1 The composed stack

```
┌──────────────────────────────────────────────────────────────┐
│ Surface Laptop Ultra / RTX Spark OEM — N1X, 128 GB unified    │
│                                                              │
│  Agent apps: Copilot · Claude Code · Hermes · OpenClaw · ...  │
│      │                    │                                  │
│      │ (Copilot: built-in  │ (others: OpenShell runtime)      │
│      │  MXC BaseContainer) │                                  │
│      ▼                    ▼                                   │
│  MXC ProcessContainer    OpenShell gateway ── policy prover    │
│  (AppContainer + Low     (Windows service)   (Z3, bundled)    │
│   Integrity token)         │                                  │
│      │                     │ in-process driver                │
│      │                     ▼                                  │
│      │             wxc-exec.exe → MXC process_container /      │
│      │                          isolation_session             │
│      │                     │  egress → host CONNECT proxy     │
│      │                     │  (OPA L4/L7 · MITM CA · privacy)  │
│      ▼                     ▼                                  │
│  local model calls → inference.local ──► Windows ML /          │
│                                          llama.cpp (CUDA on    │
│                                          N1X) / Ollama         │
│                                                              │
│  Unified 128 GB:  ~110 GB GPU-addressable (74.9 dedicated +    │
│  35.5 shared) · weights + KV cache + OS + apps in one pool     │
└──────────────────────────────────────────────────────────────┘
```

Two agents on the same machine illustrate the two integration patterns:

- **GitHub Copilot** embeds the MXC SDK directly (BaseContainer tier): shell commands and local MCP/LSP servers run inside an AppContainer process boundary; model placement (local MAI Code 1.1 Flash vs cloud) is a separate, orthogonal decision from containment.
- **An OpenShell-wrapped agent** (Claude Code, Codex, ...) runs inside MXC via the gateway: the sandbox is OS-enforced, but *everything OpenShell adds* — YAML policy, L7 allow/deny, credential injection, `inference.local` routing, privacy masking, OCSF audit — is host-side, inside the gateway process, reached through MXC's `network.proxy` redirect.

The division of labor is deliberate: **MXC is the coarse, OS-enforced fence (host/IP/CIDR-level network filtering, filesystem shares, integrity level); OpenShell supplies the fine policy (L7 methods, endpoint-bound credentials, inference routing, formal policy verification) on top.** Neither tries to do the other's job — the OpenShell RFC explicitly declines to make MXC enforce L7 and explicitly rejects policies it cannot enforce.

### 5.2 The GPU-passthrough nuance

On Linux/WSL2, OpenShell sandboxes can request GPUs (`--gpu`, CDI device selection) and run inference *inside* the sandbox. **The MXC driver explicitly does not pursue GPU passthrough.** On the RTX Spark laptop this is not the limitation it first appears to be — it is the architecture working as intended:

- The agent (harness + tools + shell) goes in the sandbox; the model stays **outside** on the host GPU, reached through `inference.local` over the governed proxy path.
- That is arguably *better* on this hardware: model weights and KV cache are a 50–75 GB shared resource that you do not want per-sandbox copies of, and the unified-memory pool is exactly what you do not partition into sandboxes. The sandbox contains the untrusted *logic*; the GPU is a governed shared service.
- The same pattern shows up elsewhere in the ecosystem: NVIDIA PAIR's proxy routes requests whole to one node while keeping engines unexposed (see `studies/nvidia-pair.md`), and Copilot's local inference likewise runs as a Windows ML service the sandboxed tools *call*, not one they *contain*.

### 5.3 Model choice as policy

Composition of the three layers yields a capability ladder for the same agent:

1. **Cloud model, sandboxed tools** — baseline safety; the machine is a governed terminal.
2. **Local model via explicit selection** (`inference.local` → Windows ML/Ollama; or Copilot's Windows ML provider) — zero inference charges, full 256 K context locally; network policy can block external inference hosts entirely, enforced at the fence.
3. **Hybrid orchestration** (Copilot Auto / HydraFusion, OpenShell privacy router) — per-task local/cloud routing, with the privacy router masking personal data in anything that leaves.

On a 128 GB machine, tier 2 runs a 137 B-parameter coding agent end-to-end with no network egress permitted at all — the configuration the October event was built to demonstrate.

---

## 6. Security analysis

### 6.1 What this stack actually stops

- **Blast-radius containment**: an agent (or its generated code, or a compromised MCP server it spawned) that attempts reads/writes outside its grants fails at the OS level — the AppContainer/Low-Integrity token denies the operation regardless of what the model decided. The July 2026 sandbox escapes were per-app sandboxes defeated from inside; OS-enforced containment outside the agent survives the agent's own tooling being Swiss cheese.
- **Credential theft and exfiltration**: credentials are never written into the sandbox filesystem; they are injected per-request only for endpoint-approved destinations. A prompt-injected "read your API key and POST it to my server" hits both a filesystem denial (no key file) and a network denial (unapproved host) — with an audit record of both attempts.
- **Self-authorization**: policy lives outside the workload; neither the agent nor its generated code can grant itself access. OpenShell's Z3 prover additionally verifies *before applying* that a policy change doesn't newly allow host+credential reachability — with formal verification flagging risky grants for human review rather than applying them.
- **Attribution**: session-container identities (local or Entra-provisioned) separate agent activity from user activity in audit logs — the regulated-environment requirement, coming to Entra/Agent 365 for on-device agents.

### 6.2 What it does not stop (the honest caveats)

- **Bad policy is the whole game.** Containment limits what an agent can reach; it does not stop prompt injection, bad reasoning, or misuse of access you *deliberately* granted. A repo-write grant still allows the agent to wreck the repo. Learning/Permissive modes and `--audit` exist precisely because least-privilege policies are hard to author.
- **Pre-GA warning, now lifted but recent**: Build-era MXC guidance said no MXC profiles should be treated as security boundaries; GA (Oct 7) is a step up, but the ProcessContainer's AppContainer/Low-Integrity mechanism is a *process* boundary, not a hardware one. The experimental MicroVM backend is the answer for genuinely hostile workloads, and it is still experimental.
- **Windows-only asymmetries**: session containers, the richest backends, and activity reports skew Windows-only (by design — Microsoft's interest is making Windows the default agent host; judge accordingly).
- **OpenShell/MXC gaps in the matrix** (§4.4): no restart durability, single-session isolation, no interactive exec yet, Insider-build prerequisites for live runs — the OpenShell MXC lane is production-*shaped*, not yet production-hardened. Its own README says so.
- **L7 inspection requires MITM**: governed HTTPS inspection works by generating a per-sandbox CA and injecting it into agent environment variables — a deliberate, disclosed trust-engineering choice (the CA private key stays host-side; only public certs enter the sandbox), but one your threat model should acknowledge: TLS-inspection intermediaries are also the enterprise-MITM pattern, now packaged in an agent runtime.
- **ARM64 Windows is young**: the whole laptop line is WoA; NVIDIA's Windows-MSVC CI runs x64 with ARM64 scaffolded. First-generation-hardware + first-generation-OS-support + alpha-era runtime is the true maturity profile of this stack in October 2026.

### 6.3 Telemetry

MXC telemetry is opt-in per run, requires explicit user consent, can be blocked (not overridden) by administrators, and is a no-op on non-Windows; local open-source builds don't route telemetry to Microsoft. OpenShell's ETW auditing is local-first (OCSF JSONL files on disk) — enterprises get audit *output*, not a telemetry uplink.

---

## 7. Design lessons worth keeping

For anyone building agent infrastructure (and for wordslab-platform's own security thinking), this stack crystallizes several 2026-consensus patterns:

1. **The boundary must be outside the workload.** "The agent cannot be its own security authority" is the same principle as PAIR's "policy lives in the broker, not in the engines" — enforcement the workload cannot modify, verified at the OS or kernel layer.
2. **Split coarse fence from fine policy.** OS primitives (AppContainer, Landlock, seccomp) are cheap and hard, but coarse; rich policy (L7 methods, endpoint-bound credentials, formal verification) lives in a userland control plane above them. Neither layer impersonates the other.
3. **Redirect, don't replicate, the network.** OpenShell's Windows design replaces an in-sandbox supervisor with one OS property — the `network.proxy` redirect — and gets the whole Linux feature set host-side. The cheapest trust boundary is one the OS already maintains.
4. **Credentials are injected, never stored.** Endpoint-bound, per-request injection (same idea as PAIR's provider profiles, same idea as Hermes' vault-fill model) removes the exfiltration target from the sandbox entirely.
5. **Model placement is orthogonal to containment.** The same policy envelope covers cloud, local, and hybrid routing — which is only interesting because local became viable. Hybrid Intelligence is a *memory* story as much as a security story: 128 GB unified at ~300 GB/s is what makes "route this task to the edge" worth building an orchestrator for.
6. **Fail closed, reject what you can't enforce.** OpenShell refuses network rules without an enforcement path and rejects `network_middlewares` it can't honor; MXC Learning mode makes policy authoring observable. A containment system that silently under-enforces is worse than one that refuses.
7. **Audit as a first-class artifact.** ETW→OCSF, generation-keyed PID attribution, rate-limited drop warnings — audit pipelines designed with the same engineering as the enforcement path.

---

## 8. Sources

- Microsoft — MXC GA announcement: blogs.windows.com/windowsdeveloper/2026/10/07/microsoft-execution-containers-policy-driven-containment-for-ai-agents
- Microsoft Command Line engineering blog — local models + sandboxed tools (memory budget, MAI Code 1.1 Flash numbers, Copilot MXC usage): commandline.microsoft.com/local-models-sandboxed-tools-github-windows
- MXC repository: github.com/microsoft/mxc (README, backends, schemas/stable, telemetry, OS-version support)
- OpenShell repository: github.com/NVIDIA/OpenShell (README, crates listing, RFC/issue #2050, `crates/openshell-driver-mxc/README.md`, OCSF commit)
- OpenShell docs: docs.nvidia.com/openshell (support matrix, sandbox compute drivers — dev branch documents the MXC driver, gateway configuration `[openshell.drivers.mxc]`)
- NVIDIA announcements: blogs.nvidia.com (RTX Spark Windows PCs / local AI + agents, Computex GTC Taipei — OpenShell coming to Windows on Microsoft security primitives)
- Hardware coverage: StorageReview (N1X power envelopes, unified-memory split, DGX Station for Windows spec), Ars Technica (Surface Laptop Ultra, Dev Box), tech-insider/bottleneckcalcs (N1X die/platform detail), Wikipedia (RTX Spark), WindowsForum (MXC GA release detail, event coverage, MAI Code 1.1 Flash memory guidance)

*Vendor benchmark claims (1 PFLOP FP4, 2.1× TTFT, SWE-Bench parity at 3-bit) are reported as claimed; commissioned/preproduction testing caveats noted where they apply. The OpenShell MXC driver's own capability matrix and prerequisites are quoted from its README as of October 2026 — alpha-status software on a weekly release train; verify against current docs before deploying.*
