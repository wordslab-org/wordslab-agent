# WSLc — Windows Subsystem for Linux Containers: How It Works, and What It Simplifies in wordslab-platform

*Study of WSL containers (WSLc), generally available September 29, 2026 in WSL 3.0.1 — mechanics from Microsoft's architecture deep-dive and the `microsoft/WSL` / `microsoft/mxc` repositories, operational findings from running a real production workload (the Hermes agent gateway) in a wslc container, and a grounded analysis of what the wordslab-platform design (ADR-0001…0031) can and cannot simplify with it. October 2026.*

---

## 1. Executive summary

WSLc is WSL's built-in Linux container runtime: **`wslc.exe`** (alias `container.exe`) plus a native Windows API (`Microsoft.WSL.Containers`), shipped with WSL 3.0.1 after `wsl --update`. No Docker Desktop, no engine to install, no license questions. Three architectural moves distinguish it from both WSL distros and Docker-on-Windows:

1. **Per-user session VMs with a privilege split** — the privileged `wslservice.exe` creates the VM but does not keep it; a child `wslcsession.exe` running as the invoking user owns all container operations. Sessions are strongly isolated from each other and run with less privilege than the WSL service.
2. **Two storage primitives** — Windows paths shared over **virtiofs** (~2× faster than the plan9 used by WSL distros) and **VHD-backed volumes** (native ext4, size-limited) — with all session state (images, containers, networks, volumes) in a per-session VHD under `%AppData%\Local\wslc\sessions`.
3. **Consommé networking** — the VM has no virtual NIC on the Windows network; every Ethernet frame exits to a virtio queue served by a **user-mode Windows process** that does DNS, TCP/UDP routing and port mapping. Container traffic behaves like traffic from a normal Windows app — it follows VPNs and firewalls instead of fighting them.

For wordslab-platform the honest conclusion is **not** "replace the platform's WSL distro with containers." The core, the services and their SQLite databases, and GPU inference are exactly what long-lived distro-native processes are for, and ADR-0003/0004's design stands. WSLc's value is a **second, lighter zone** for everything the platform currently has to hand-roll around the distro: engine/harness packaging, agent and workflow execution sandboxes, per-user disk quotas, published-app deployment, and try-before-install evaluation — with the resident launcher (ADR-0014 §4) as the natural Windows-side wslc API client. Section 7 works through that mapping ADR by ADR, including what WSLc does *not* solve (restart policies, UDP, boot-time autostart, per-session RAM cost).

---

## 2. Lineage: WSL1 → WSL2 → WSLc

- **WSL1 (2016):** syscall translation, no Linux kernel. Linux binaries ran on Windows but against a translation layer — compatibility gaps, no containers.
- **WSL2 (2019):** a real Linux kernel in a lightweight utility VM managed by the privileged `wslservice.exe` via the Host Compute Service (HCS). Full syscall compatibility; distros are user-lands sharing one kernel/VM (one `uname -r`, shared network, shared `/mnt/c`). Docker Desktop on Windows is itself a WSL2 distro — containers on Windows have meant *a Docker engine inside that shared VM*.
- **WSL containers (Build 2026 preview → **GA September 29, 2026**, WSL 3.0.1):** Microsoft's own container platform one level down — a purpose-built container host that does not reuse the shared WSL utility VM and does not need Docker at all. GA ships the CLI + API, `wslc container restart`, `wslc container cp`, `wslc system info`, `wslc network connect/disconnect`, driver options on `network create`, `wslc events` (live activity stream), container health checks, `--stop-timeout` (incl. `-1` infinite), `--mount`, and a configurable storage path for the default session.

WSLc is open source in `microsoft/WSL`. Per Microsoft (Craig Loewen), it is "ready for use for full development and production use cases," runs on Windows 10 and Windows 11, and is supported in production on Windows Server.

---

## 3. Architecture

### 3.1 The session model and the privilege split

In classic WSL, every client talks to `wslservice.exe`, a privileged Windows service that creates VMs through HCS and *keeps ownership* of them. WSLc changes the ownership rule:

- `wslservice.exe` still creates the VM (it holds the HCS privilege), **but does not retain it**.
- It spawns a child process, **`wslcsession.exe`**, running on behalf of the calling user, which then performs all session operations: creating containers, mounting directories, binding ports.
- Result: **one session process per user**, strong isolation between sessions (different processes, different VMs), and session operations executing with *user* privilege, not service privilege. A compromised session process has the user's rights, not the WSL service's.

The session VM is per-user and per-session: community measurement puts a fresh session VM at **~1.5 GB RAM**. Sessions are meant to be disposable ("containers disappear when you're done") — with the caveat (§5) that the *session VM* outlives the containers until explicitly terminated.

### 3.2 Storage: three layers

| Primitive | What it is | When to use |
|---|---|---|
| **Session VHD** | One VHD per session holding the session's entire state — available images, containers, networks, volumes. With `wslc.exe` it lives under `%AppData%\Local\wslc\sessions`; the GA release allows configuring the storage path per session (put container storage on the drive you want). | Automatic; the session's scratch universe |
| **Path volumes (virtiofs)** | A Windows directory shared into the VM over virtiofs, mounted under `/mnt` in the VM, then bind-mounted into the container at the requested path. Boulay: **virtiofs is about twice as fast as plan9** (which is what WSL distros use for `/mnt/c`). | Sharing host files with a container; anything where the Windows side must still see/edit the data |
| **VHD volumes** | `wslc volume create --driver vhd -o SizeBytes=…` → a real ext4 filesystem inside a VHD, mountable by name into one or more containers; enforces a **hard size limit**. | A container that needs a native Linux filesystem, or a volume whose size must be capped |

Container scratch space (the writable layer) is discarded when the container is deleted; volumes are how data outlives containers.

### 3.3 Networking: Consommé

The VM's traffic never touches a Windows virtual NIC. Instead:

1. All the VM's traffic is emitted as **raw Ethernet frames into a virtio queue**.
2. A **Windows process running as the session's user** reads the frames and provides the network services itself: **answering DNS queries**, **routing TCP and UDP**, and **port mapping** (`-p 127.0.0.1:8642:8642` style, as with Docker).
3. Outbound traffic is reinjected into the Windows stack *as the user's process*.

The payoff: container traffic behaves like traffic from any other Windows application — it **automatically follows corporate VPN clients, respects Windows firewall policy, and traverses proxies**, the classic pain points of VM-based container networking on Windows. Microsoft's own caveat: admins should still test their specific network policies rather than assume identical behavior everywhere.

### 3.4 Surfaces: CLI, API, ecosystem

**CLI** (`wslc.exe` / `container.exe`) — Docker-familiar grammar:

```
wslc run --rm -it ubuntu:latest bash -c "echo Hello world from WSL container!"
wslc run -it --rm -d -p 8080:80 --name web nginx
wslc container ps | stop | restart | cp
wslc volume create --driver vhd -o SizeBytes=2000000000 my-volume
wslc network create | connect | disconnect
wslc events        # live activity stream
wslc system info
```

**API** (`Microsoft.WSL.Containers` NuGet, C# and C++/WinRT projections) — four objects: `WslcService` (component check/install), `Session` (the WSL-backed host; manages images; creates containers; configurable CPU count and memory), `Container` (start/stop/inspect/delete; run extra processes inside), `Process` (read output, write stdin, send signals, watch exit codes). Supports file mounts, networking, port mapping and GPU access. A Windows app can therefore ship a Linux-only workload — an inference server, a data tool — and start it in a container **without asking the user to install anything**. GA added stdin support through the SDK and raised the VM termination timeout to 30 s.

**Ecosystem:** the VS Code Dev Containers extension can use `wslc` as its container driver; the VS Code Containers extension and .NET Aspire support it; community tools include a Compose shim (`wslc compose`), terminal dashboards, and a WinUI container-desktop app.

### 3.5 Enterprise controls

- **Intune** can enable/disable WSL containers entirely and enforce an **allow-list restricting image pulls to approved registries**.
- **Microsoft Defender for Endpoint's** WSL plugin now covers containers: process, file, and network activity inside a container is surfaced and linked back to the Windows host.

These matter to wordslab-platform only marginally (home/small-business), but the registry allow-list is the same mechanism an admin-relative ("my employee's workstation") install could reuse.

### 3.6 MXC's WSLc backend — the policy rider

MXC (see `studies/mxc-openshell-rtx-spark.md`) treats WSLc as one of its containment backends (`containment: "wslc"`, schema v0.9 — a *stable* backend, no runtime experimental opt-in; native builds need the `wslc` feature and the external WSLC SDK, `wslcsdk.dll`). Two design points are worth stealing regardless of whether the platform ever adopts MXC:

- **Hermetic execution — MXC never pulls images at run time.** Images are pre-pulled into the SDK cache (`setup-wslc.ps1`); the executor never reaches for the network and its failure modes are decoupled from registry availability. This is exactly the wordslab-platform "download at install, then run" rhythm (ADR-0005's install-disk booking).
- **Fail-closed policy honesty.** WSLc networking is all-or-nothing (`None` vs `Bridged`; no per-host filtering, no `CAP_NET_ADMIN`, no independent ingress/loopback restriction). MXC therefore accepts exactly two coherent postures (deny/deny/deny = isolated; allow/allow/allow = bridged) and **rejects** mixed postures, per-host rules, and legacy fields rather than pretending to enforce them. `networkProxy` is honored for what it is — env-var injection, not a firewall. UDP port mappings are hard-rejected (the SDK returns `E_NOTIMPL`). *A sandbox layer that refuses what it cannot enforce is a platform-compatible stance: no security theater (ADR-0017).*

The state-aware lifecycle (provision → start → exec → stop → deprovision) is backed by a persistent per-user daemon (`wxc-wslc-daemon.exe`) that keeps the SDK session and container warm across phase processes (the SDK has no cross-process re-attach; a keepalive init process keeps the container alive between `exec` phases). `exec` timeouts kill the **process**, not the container. One-shot mode supports only `destroyOnExit: true`.

---

## 4. Operational realities (from running production workloads in wslc)

Running the Hermes agent gateway in a wslc container (s6-supervised multi-service image, published ports, named volumes) surfaced a set of behaviors the docs underplay. These are the facts a platform build plan must budget for:

1. **No `--restart` policy.** `wslc container run --restart …` is not recognized. Containers die on reboot/WSL shutdown and must be restarted — via `wslc container start <name>`, a desktop-app watchdog, or a scheduled task at logon. For an always-on platform service, **the restart story is yours to build** (§7.3).
2. **WSL stops idle VMs after ~15 s.** For always-on sessions, set `instanceIdleTimeout=-1` in `%UserProfile%\.wslconfig`. Without it, a quiet leader looks "down" to clients.
3. **The session VM outlives its containers** — it survives container exit, a manual WSL shutdown signal, and holds its RAM. Reclaiming requires `wslc system session terminate` (three CLI levels deep, per community testing). Budget the ~1.5 GB/session and terminate deliberately.
4. **Bridge networking only; no `network_mode: host`.** Every reachable service needs an explicit `-p` mapping.
5. **The loopback-binding trap.** A published port can be mapped correctly and still refuse connections: if the in-container process binds `127.0.0.1`, the mapping forwards to the container's loopback, where nothing answers from outside. Diagnose in order — Windows `netstat` (mapping present?), then `/proc/net/tcp` inside the container (`00000000:…` = correctly bound to `0.0.0.0`) — and fix by running the service with an explicit `0.0.0.0` bind. Any wordslab service container image must default to a non-loopback bind when a port is published.
6. **`wslc` output contains NUL bytes** when piped — strip them (`tr -d '\000'`) when parsing CLI output programmatically.
7. **Env injection vs file config precedence.** A key set inside a mounted volume's `.env` can override `-e` environment variables — configuration precedence rules must be explicit per service, or "why is my new API key ignored" becomes a support case.
8. **GPU access exists at the API level** (Microsoft Learn documents GPU access in the WSL Containers API; MXC's schema carries a `gpu` flag for wslc). What is *not* established, as of October 2026, is production-hardened CUDA inference inside wslc containers on the CUDA-for-WSL driver stack — see §7.6.

None of these are blockers; all of them are planning inputs.

---

## 5. WSLc versus the alternatives (for the Windows leg)

| | **WSL2 distro** (wordslab-notebooks pattern) | **Docker Desktop** | **WSLc** |
|---|---|---|---|
| What runs | A full long-lived Linux user-land | A Docker engine inside the shared WSL utility VM | Purpose-built per-user session VMs |
| Daemon | none (it *is* the OS) | dockerd + Docker Desktop services | none to install; `wslcsession.exe` per user |
| Isolation | weakest (one shared kernel/VM across distros) | container-level in one shared VM | **session VM per user; privileged service does not own the VM** |
| Networking | shared stack, plan9 `/mnt/c` | Docker NAT inside WSL VM; VPN pain | **Consommé — user-mode frames; VPN/firewall-native** |
| Windows file access | plan9 (~½ of virtiofs speed) | bind mounts | **virtiofs shares + VHD volumes** |
| Persistence | distro VHD | volumes | volumes (path or VHD), session VHD with configurable location |
| Licensing | — | **Docker subscription terms for larger businesses** | none beyond Windows |
| Restart/boot | distro boot; systemd optional | restart policies built in | **no restart policy — build your own** |
| Fit for wordslab | the platform's runtime (ADR-0003) | avoidable dependency | the container zone (§7) |

The strategic point: on Windows, WSLc lets a *native Windows application* own container lifecycle — no engine install, no license surface — which is precisely the shape of wordslab-platform's resident launcher.

---

## 6. wordslab-platform: what WSLc changes, and what it doesn't

The platform's design phase has settled (ADR-0003, 0004): **one Linux runtime per machine** — on Windows via a WSL2 distro — hosting the core, the services, their SQLite DBs, and (layer 2) the heavy implementations. The support matrix is "hosts = Windows 10/11 via WSL2 (primary) + native Linux + rented cloud hosts." ADR-0004 §3 even anticipated container layering: *"All services and capabilities software lives in the distro's filesystem (one filesystem → uv package dedup and docker layer sharing work)."*

The recommendation of this study is **not** to replace that. It is to add a **second execution zone** where the platform currently has to engineer its own answers, and to assign each zone what it is structurally best at:

### 6.1 What stays in the distro (unchanged)

- **The platform core** (control plane: fleet/install/catalog/resources/users/keys/secrets/monitoring/logs/stats/datasets — ADR-0003) — long-lived, stateful, authority-bearing; a per-user ephemeral session VM is the wrong home.
- **The eleven services** — long-running, each with its own SQLite DB (ADR-0002), FastHTML UIs, one filesystem for uv dedup (ADR-0004 §8), realtime low-latency capability loops (voice: STT+TTS+VAD in one process — ADR-0002 §2). Containers buy nothing here and cost a session VM, a restart story, and a networking hop.
- **GPU-bound inference engines in v1** — Ollama/vLLM/sglang class implementations stay distro-native where the CUDA-on-WSL path is proven; GPU-in-wslc is unproven at production grade (§7.6 open question). Engines-in-containers is a *possible* later stage, gated on hardware verification, not a v1 bet.

### 6.2 The container zone: six concrete adoptions

**(a) Agent and workflow execution sandboxes — the biggest win.** ADR-0007 made workflows plain Python invoked by callers; agents graduate into them; ADR-0017's outermost boundary is "the platform must not reach into host OS files," with execution containment deliberately left light. A wslc container is a *pre-built* version of that boundary, stronger than any in-distro mechanism the platform could build itself:

- A **session VM boundary** (not a namespace in a shared kernel): no `/mnt/c`, no Windows PATH, no distro interop — host files are unreachable *by construction*, making ADR-0017's default-deny stance literal rather than configured.
- **Deny-by-default networking** (`None` mode) with explicit enablement — generated code runs offline unless the activity grants egress.
- Ephemeral by design: run workflow/agent code, keep artifacts on a volume, delete the container. No cleanup machinery to write.
- This is the zone where **MXC's WSLc backend** (§3.6) becomes relevant *if* per-tool policy ever needs to be more than "on/off": its two-posture honesty (isolated vs bridged, everything else rejected) fits ADR-0017's anti-theater stance. v1 does not need MXC — wslc directly is enough — but the study of it is done (see `studies/mxc-openshell-rtx-spark.md`).

**(b) Workspace environments with their own libraries.** ADR-0004 §8 already rules that workspace environments (JupyterLab, agent workspaces) install their own library copies, decoupled from platform versions. Today that costs distro disk and risks polluting the shared filesystem. As wslc containers they are: an image pinned per workspace kind, a **VHD volume for `/workspace`** (native ext4, size-limited), network off by default, deleted wholesale when the workspace dies. Nothing to clean up in the distro.

**(c) Per-user and per-content disk quotas as VHD volumes — ADR-0005 §8 nearly for free.** The reservation ledger models "free = measured free minus reservations"; the disk-quota machinery gives users allocated maxima (extended by ADR-0031 to implementations with `storage-space` content parts). A VHD volume *is* an enforced quota: `wslc volume create --driver vhd -o SizeBytes=<quota>` gives the user a mountable, size-capped ext4 volume. The `resources` capability's bookkeeping shifts from *monitoring* quotas (soft) to *provisioning* them (hard, enforced by the storage layer) — one less thing to police in userspace.

**(d) Harness images make the frequent update lane literal (ADR-0016).** The update taxonomy's frequent lane is "additive, side-effect-free" — new models, **harness images**. As wslc images that lane becomes: pull the new image, restart the container, prune the old image — old versions pruned, backward compatible, and *no automatic anything* is preserved (the user's one click triggers the pull). The core's `install` capability gains a second implementation kind: `implementation.toml` gains an optional **container-image source** (alongside local weights and cloud provider references, ADR-0027/0031) — the platform can then treat "an image reference + volume mounts + port map" as a declarable, installable, updatable implementation.

**(e) Published apps: the deployable unit for ADR-0019's catalog.** Generated apps deploy with deploy/start/stop; their sharing is overlay-first (Tailscale/WireGuard inside WSL2 — ADR-0019 §Goal-3). A published app as a wslc container maps cleanly: app image + persistence volume + `-p` port published through Consommé (which, notably, makes the app's traffic behave like normal Windows traffic on the host's VPN — overlay-friendly). The Publishing service manages ACLs on the overlay; the container carries the app.

**(f) Evaluation and try-before-install (ADR-0020).** An evaluation capability that must run an implementation *before committing disk* gets a cheap substrate: pull the candidate image into the eval session, run the eval harness, read results, delete. The session VHD is disposable by design; the distro never sees the experiment. (Same mechanism as MXC's pre-pull-then-run hermeticity.)

### 6.3 The three-zone Windows machine

```
┌─ Windows ──────────────────────────────────────────────────────┐
│  Installer + resident launcher (ADR-0014)                      │
│    └─ wslc API client (Microsoft.WSL.Containers):              │
│       sessions, volumes, containers, restart-at-logon          │
│                                                                │
│  ┌─ Zone 1: WSL2 platform distro ──── long-lived ────────────┐ │
│  │  core (leader/worker) · services · SQLite · CUDA engines  │ │
│  │  data volumes (models/, workspace/ — ADR-0004 §8)         │ │
│  └───────────────────────────────────────────────────────────┘ │
│  ┌─ Zone 2: wslc session VM(s) ─── ephemeral/capped ────────┐  │
│  │  agent+workflow sandboxes (net: none)                    │  │
│  │  workspace environments (VHD volume per workspace)       │  │
│  │  user quota volumes · harness/eval images                │  │
│  │  published apps (ports via Consommé)                     │  │
│  └───────────────────────────────────────────────────────────┘ │
│  ┌─ Zone 3: Windows-native surface ─────────────────────────┐  │
│  │  launcher UI · front-door port bridging · WoL · backups   │  │
│  └───────────────────────────────────────────────────────────┘ │
└──────────────────────────────────────────────────────────────────┘
```

Division of labor: **Zone 1 runs the platform; Zone 2 executes untrusted/heavy/ephemeral work; Zone 3 owns the machine** (ADR-0014 §4's launcher is already "the single surface to start/stop/update/monitor/backup"). The launcher's "Start" already handles "run WSL + platform, open the dashboard" — it naturally extends to "ensure the wslc session and its containers" (it is a Windows app; the wslc API is a Windows API — no interop hop).

### 6.4 The simplification ledger

| ADR / concern | Today's design cost | With the container zone |
|---|---|---|
| ADR-0003 bootstrap | Installs WSL + distro + virtual disks + uv/Python (unchanged for Zone 1) | Unchanged — but engine *dependencies* stop being bootstrap's problem |
| ADR-0004 §3 layer 2 | Implementations installed into the one distro; apt/pip/CUDA conflicts across engines managed by hand | Image-backed implementations: pinned upstream, isolated, `install` = pull |
| ADR-0004 §8 workspaces | Own library copies in distro disk; cleanup manual | Workspace containers + VHD volumes; delete = gone |
| ADR-0005 §3/§8 quotas | Soft ledger bookkeeping + userspace policing | Hard VHD-size enforcement for user/workspace quotas |
| ADR-0007 workflows | Execution boundary of generated code left light (deliberately) | OS-enforced, deny-by-default sandbox per run — without building a sandbox |
| ADR-0014 launcher | Machine lifecycle only | Owns Zone-2 lifecycle via native API; restart-at-logon covers wslc's missing `--restart` |
| ADR-0016 updates | Frequent lane is a taxonomy | Frequent lane = pull/prune images; lanes stay user-clicked |
| ADR-0017 security | "No host access by default" is a distro stance + configured mounts | For agent execution it becomes VM-constructional (no `/mnt/c` at all) |
| ADR-0019 publishing | Deployable apps need a packaging story | App = image + volume + port map; deploy/start/stop map 1:1 to wslc verbs |
| ADR-0020 evaluation | "Run before committing disk" needs isolation machinery | Eval session; disposable VHD; hermetic pre-pull |

### 6.5 What WSLc does *not* solve (keep it honest)

1. **Restart/boot resilience.** No `--restart`; session VMs are per-user (a logged-in user's processes). An always-on leader needs the launcher (or a logon scheduled task) to re-establish sessions — and a "leave it on, nobody logged in" headless leader is a real deployment question (autologon vs Windows-service wrapper). This must be designed, not assumed.
2. **UDP through mapped ports.** The WSLC SDK returns `E_NOTIMPL` for UDP port mappings (MXC hard-rejects them). Any realtime voice/video surface that assumed UDP port mapping needs a different path (overlay-native, or distro-side).
3. **RAM overhead per session.** ~1.5 GB per session VM + the idle-timeout config. The design should use **one platform session** (not per-user sessions — platform "users" are ADR-0003 `users`, not Windows accounts) and terminate deliberately.
4. **Cross-platform leakage.** wslc exists only on Windows. Linux/cloud machines keep the distro-native path; every container-zone adoption must remain an *implementation packaging choice*, never a structural assumption (the same activity must still be satisfiable distro-native on a Linux worker).
5. **GPU inference in containers is unproven** on the CUDA-for-WSL stack; v1 keeps engines in Zone 1 (§6.1).
6. **Session-VM lifetime hygiene** (§4.3) — the platform's `monitoring` capability should surface wslc session state (RAM held, last terminate) so the user sees it on the dashboard (no black box).

### 6.6 Open questions (candidates for tickets)

1. **GPU-in-wslc verification** — does CUDA inference inside a wslc container reach parity with distro-native on the same driver? (Enables/defers "engines as images".)
2. **Headless autostart** — launcher/scheduled-task pattern vs Windows-service wrapper for "always-on leader, no logged-in user"; what does Start mean for Zone 2 at boot?
3. **`implementation.toml` image-source shape** — fields for image reference + registry allow-list + volume/port declaration; interplay with ADR-0031's declaration model v2.
4. **Front door × Consommé** — port publication for published apps vs the front door's single-address/cert rule (ADR-0004); does the overlay (Tailscale inside the distro) see container traffic correctly through Consommé?
5. **Backup semantics for Zone 2** — VHD volumes are file-copyable; define what ADR-0021 backs up (volumes yes, session VHD no) and the restore ritual.
6. **Windows 10 floor** — wslc works on Windows 10 per Microsoft; verify the platform's minimum Windows build against the WSLC SDK's component gate (`WslcGetMissingComponents`).

---

## 7. Sources

- Microsoft — WSL containers GA: blogs.windows.com/windowsdeveloper/2026/09/29/wsl-containers-now-generally-available (Logan Iyer); WSL 3.0.1 release notes: github.com/microsoft/WSL/releases/tag/3.0.1
- Pierre Boulay — WSLC Architecture deep dive: devblogs.microsoft.com/commandline/wslc-architecture-deep-dive (session model, virtiofs/plan9, Consommé)
- WSL Containers API — Microsoft.WSL.Containers (learn.microsoft.com/en-us/windows/wsl/wsl-container; WslcService/Session/Container/Process)
- MXC WSLc backend — github.com/microsoft/mxc: docs/wsl/wsl-container-getting-started.md, docs/wsl/wslc-state-aware.md, docs/schema.md (SDK 2.9.9, daemon model, networking honesty, UDP rejection)
- Operational findings — first-hand from running the Hermes agent gateway image in a wslc container (ports, volumes, s6 supervision): restart-policy absence, `instanceIdleTimeout`, loopback-binding trap, NUL-byte output, env/file precedence (Hermes skill `wslc-hermes-gateway`)
- Community measurement — XDA Developers (session VM ~1.5 GB RAM, session-VM persistence, `wslc system session terminate`); WindowsLatest (Windows 10/Server support, Loewen production quote); WindowsForum (GA feature list, API objects, storage paths); devnews (Intune/Defender coverage)
- wordslab-platform — docs/adr/0003 (bootstrap, core, support matrix), 0004 (topology, two-layer install, workspaces), 0005 (ledger, quotas), 0007 (composition), 0014 (installer/launcher), 0016 (update lanes), 0017 (security model), 0019 (publishing), 0020 (evaluation), 0021 (backup), 0027/0031 (implementations/declaration model v2)

*Claims are sourced as above; the GPU-in-wslc question is explicitly open (§6.6) and should be settled on hardware before any engine-container decision. WSLc is GA on a fast-moving WSL release train — re-verify CLI/API specifics against the current WSL release before building.*
