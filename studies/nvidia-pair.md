# NVIDIA PAIR (Personal AI Router) — Technical Study

*Study of `github.com/NVIDIA/Personal-AI-Router` (Apache-2.0, Go + Electron), release `services 2.1.7` / commit `1a0b606`, October 2026. Based on the source at `/opt/data/nvidia-pair` — 12 Go service binaries (~142K lines) plus an Electron/React desktop application. All claims verified against code; file references throughout.*

---

## 1. What PAIR Is

PAIR turns several machines on a LAN into **one place to send inference requests**. It is a *router*, not a runtime: it runs no models, pools no GPU memory, shards nothing, and never splits an in-flight request. Each request goes **whole to exactly one node**; adding machines increases *concurrent throughput*, not single-request latency.

The target workload is concurrent local inference — multi-agent applications (Hermes Agent, OpenClaw/NemoClaw, Perplexity Portable Computer all have documented PAIR setup flows) where one task fans out into several independent sub-queries. On a single machine those queue behind one GPU; PAIR spreads them over every idle paired PC.

**Key design inversion**: PAIR does not ask applications to point at a new endpoint. The proxy **takes over the engines' default ports** (Ollama `11434`, LM Studio `1234`, llama.cpp `8080`) and moves the engines behind it (`11435`, `1235`, `8081`). Existing clients work unchanged and transparently gain the cluster.

Supported: Windows 11 / Linux / macOS, x64 + arm64, mixed-OS clusters. Engines: **Ollama, LM Studio, llama.cpp**.

### 1.1 Vocabulary

| Term | Meaning |
|---|---|
| **Node** | One machine running PAIR. No server, no controller, no primary. |
| **Cluster** | The set of paired nodes. Symmetric membership; a node belongs to at most one cluster. |
| **Engine** | A local inference server on a node (Ollama / LM Studio / llama.cpp). Any combination; none running = not eligible to serve. |
| **Model** | Belongs to an engine on a specific node. Nothing is shared. The same model on two nodes = two independent copies; that duplication is what makes nodes interchangeable. |
| **Proxy / facade** | The reverse proxy behind each local endpoint. One process hosting one *facade* per engine dialect. |
| **Job / workload** | One routed request. |
| **Broker** | The parent service per node; supervises the other services and exposes the JSON-RPC API. |

Eligibility is evaluated at the **bottom** of the hierarchy: a request names a model, so it can only be served by a node whose *running* engine holds it. Cluster membership makes a node reachable, not capable.

---

## 2. Process Architecture

Twelve Go binaries; each component is its own module with tests beside the source. Shared code in `services/shared/` (imported as `nvpair-shared/…`): logging, wire types, JSON-RPC, discovery records, mDNS, network monitoring, node identity, cluster trust, the split listener, CORS.

```
Electron desktop ──stdio JSON-RPC──▶ nvpair-ui-broker ──spawns/supervises──▶ 10 workers
nvpair-tui (headless) ──owns its own broker──▶ same workers
```

**The broker (`nvpair-ui-broker`)** is the parent process. Electron starts *only* the broker from `desktop/cli-bin/`; the broker spawns the workers. Only the scanner's first spawn is fatal — everything else is optional and non-fatal. Spawn order: scanner → errors → settings → engine-manager → (port planning) → node-info → proxy → workload-manager → manual-nodes → cluster-manager → scheduler → `app:ready`.

**Supervision is restart-based** (`supervisor.go`): exponential backoff `1s → 2s → 4s → 8s → 16s` (capped), budget of 5 attempts per unhealthy streak, 60s recovery window resets the budget. A crash raises a sticky error (`supervisor:subprocess-crashed:<name>`); recovery clears it; an exhausted budget leaves the worker down with the error standing. `nvpair-errors` cannot report its own death, so its crashes go to stderr.

**Deliberate gap**: no `/health` endpoint on any service, and the broker does not heartbeat workers. Process *liveness* only — a wedged-but-running worker is undetected (documented; restart from Settings → Service). Engines are the exception: readiness probe on start, then periodic health probes with a `healthy` flag.

**IPC**: newline-delimited JSON-RPC 2.0 over stdio (optional `--ipc` Unix socket / Windows named pipe, parent-owned). Stdout is reserved for JSON-RPC; all logging goes to stderr. Frame caps: 1 MiB inbound, 8 MiB on broker↔engine-manager links (model catalogs ≈1.9 MiB).

**Desktop tree discipline**: `services/` is the source of truth for runtime behavior; `desktop/` is a client. The renderer never reimplements routing, scheduling, discovery, or crypto in TypeScript — verified in code: *"Proxy node selection is owned by the backend nvpair-job-scheduler… PAIR UI never pins node/select"* (`empty-handlers.ts`). Electron's one sanctioned direct HTTP call is a 2-second telemetry poll of each discovered node's `/v1/node-info` (backoff ladder 4/8/16/30s on failure).

### 2.1 Port Map

| Port | Listener |
|---|---|
| `11434` | Ollama-compatible proxy (Ollama moves to `11435`+) |
| `1234` | OpenAI-compatible proxy (LM Studio moves to `1235`+) |
| `8080` | llama.cpp proxy (managed router on `8081`) |
| `14318` | Node hardware/telemetry (`ni`) — plaintext |
| `14319` | Service-error sync (`er`) — mTLS |
| `14320` | Workload propagation (`wl`) — mTLS |
| `14321` | Pairing + cluster membership (`cl`) — split channel |
| `14322` | Model inventory to peers (`em`) — split channel |
| `14323` | Remote engine control (`ec`) — mTLS only |

The `143xx` ports are fixed so the broker can tell a worker which port to serve without a handshake.

---

## 3. Discovery & Node Identity

### 3.1 One consolidated mDNS record

`nvpair-node-scanner` advertises **one** `_nvpair-node._tcp` record per host (`services/shared/noderec/noderec.go`). The TXT map:

```
v=1;uuid=<hostUuid>;cluster-uuid=<clusterUuid>;ip=192.168.1.10;ips=<ranked,candidates>;ni=14318;ol=11434;lm=1234;lc=8080;er=14319;wl=14320;cl=14321;em=14322;ec=14323
```

- The SRV port is a **fixed, non-authoritative constant**; consumers must read every real port from TXT. A missing service key means that service is absent.
- **Transport is policy, not data** — derived from the service key + whether the target is clustered, never advertised.
- The **model list is deliberately NOT in TXT** — it can exceed the 255-byte DNS per-string limit (RFC 6763 §6.1). Peers fetch it over HTTP from the engine-manager (`em` → `GET /v1/models`).
- `ips=` carries up to 4 ranked address candidates (`MaxAdvertisedIPs`) — multi-homed nodes publish the order they believe in, and consumers confirm by connecting.

**UUID is the correlation key everywhere** — discovery, cluster membership, workload attribution, error reports, telemetry, routing. Hostnames are display-only; every service advertises a per-host UUID so two machines sharing a hostname cannot silently merge. Records without `uuid=` are skipped.

### 3.2 Node identity minting

`shared/nodeid/nodeid.go` `Resolve()`: prefers `cluster/identity.json` (the cluster manager's persisted principal) → else `node-id.json` → else mints a crypto/rand UUIDv4 and persists (tmp+rename, 0600). The cluster manager's `loadOrMintIdentity` **adopts** the node-id.json an earlier-starting worker minted, so one host presents one UUID everywhere. If `identity.json` exists but the keypair is missing, it **fails loudly** rather than re-minting — a re-mint would orphan every peer pin.

### 3.3 The custom mDNS responder

PAIR runs its own responder because (a) Windows ships none, and (b) zeroconf's design — one socket bound to the multicast group for both send and receive — **fails to transmit on Windows** (Windows refuses to send from a socket whose local address is a multicast group; the WriteTo errors are swallowed inside the library). The fix: keep zeroconf's receive trick (join group per multicast interface), but **send everything from a fresh per-interface unicast-bound UDP 5353 socket** with explicit `SetMulticastInterface`.

Socket reuse, per-OS (`shared/mdns/`, build-tagged):
- Unix + Windows: `SO_REUSEADDR` only — coexists with Bonjour/Avahi/sibling PAIR processes. Linux **deliberately omits `SO_REUSEPORT`**: it can load-balance unicast datagrams into a short-lived send socket and steal replies from the long-lived receiver.
- Darwin: sender sockets add `SO_REUSEPORT` — required to coexist with the system mDNS responder.

Protocol: RFC 6762 §8.3 two startup announcements ~1s apart, re-announce every 60s, TTL 120, cache-flush bit on unsolicited announcements, goodbye (TTL=0 PTR) on shutdown. `netmon.Watch` re-announces on interface changes.

### 3.4 Scan cadence and enrichment

Scan every **5s** (timeout 3s), plus a per-interface unicast PTR re-send (the Windows workaround). Node map keyed by `uuid=`; comparison is order-insensitive multiset so multi-homed address reordering doesn't churn spurious "updated" events.

**Enrichment over HTTP** after announcement: node-info (`ni` → `GET /v1/node-info`, plain HTTP, 3s timeout) and model inventory (`em` → `GET /v1/models`, 6s timeout — deliberately above engine-manager's own 5s sweep cap; **pin-based mTLS for peers, plaintext loopback for self**). Last-good caches prevent a transient miss from blanking a node card. A 15s mDNS-independent convergence sweep guarantees eventual consistency; telemetry sampled at ~2s (per-UUID jittered ±250ms, backoff 4→8→16→30s on failure).

### 3.5 Eviction — network proof, not mDNS claims

Multicast is lossy (Wi-Fi, VPN adapters, sleeping interfaces), so eviction is deliberately slow and evidence-based:

- After **3 missed scans** (~15s), the node is checked on *every* scan; any answer resets the counter.
- Drop only after **12 consecutive misses** (~60s) — raised from 15s after real deployments showed an inference-saturated node starves its control plane while still serving.

The four eviction checks, in code order (`daemon.go` `reachable()`):
1. Fresh node-info enrichment within 10s → keep (no dial)
2. Inference response bytes within 60s → keep (no dial) — *the only liveness signal that strengthens under load*
3. TCP dial of the node-info port (1s timeout), each candidate address
4. TCP dial of the engine-manager port (fallback)

**Inference ports are never probed** — SYN spam there caused accept-queue exhaustion on macOS. When a dial succeeds, `identityAlive()` asks the responder *who it is*: a wiped-and-reinstalled node keeps its address and rebinds the same ports, so a connect alone would keep a dead record alive forever. A conclusive mismatched identity evicts.

**Record replacement (ghosts)**: a record that looks superseded (same mDNS instance name, different UUID, ≥300s older) is only *nominated* — eviction requires two node-info answers proving the old node is gone and the machine at the new address names someone else. Shared address alone is not evidence (direct-connect links reuse the same subnet). Removals emit **before** the replacement's insert so no consumer sees one machine under two identities.

### 3.6 Manual nodes

For multicast-blocked networks: `nvpair-manual-nodes` probes user-entered addresses every **10s** (three legs: Ollama `/api/tags`, LM Studio `/v1/models`, node-info). 3 consecutive all-services failures (~30s) surface an error rather than evicting. `DisableKeepAlives` forces hostname re-resolution each probe. Entries are re-keyed from the typed address to the peer's real UUID once node-info reveals it, so a manual machine also found via mDNS collapses into one entry.

---

## 4. Pairing & Cluster Trust

The most interesting subsystem: a **symmetric, CA-less, pin-based mTLS mesh** bootstrapped by a six-digit PIN over an EAP-NOOB (RFC 9140) exchange. Owned by `nvpair-cluster-manager` (port `14321`).

### 4.1 The PIN

- Generated with CSPRNG, uniform over 000000–999999 (~19.9 bits), zero-padded. Encoded as the EAP-NOOB `Noob` OOB nonce: the decimal value as a 16-byte big-endian integer.
- **Never appears on any wire channel** — returned only in the inviter's `cluster:invite-node` JSON-RPC result; every notification path strips it.
- Lives exactly as long as its pairing session; pending invites expire at **TTL 5 min** (swept every 15s). Single-shot: one completion attempt consumes the session; a wrong PIN destroys it on both sides. No retry counter needed.
- Bounded unauthenticated surface: max 64 concurrent pre-invite sessions (`429` beyond), 1 MiB body cap.
- **Honest security positioning** (the code's own spec, §4): the PIN defeats passive eavesdroppers and accidental joins — it is *known debt* that it is offline-brute-forceable by an active LAN MITM who relays the in-band exchange (NoobId leaks a verifiable oracle). "Pair only over networks and with machines you trust."

### 4.2 The pairing protocol (EAP-NOOB, vendored at `services/eap-noob`)

EAP-NOOB suites: X25519 and P-256 (server prefers P-256); one-step KDF (SP 800-56C); 17-element Hoob/MAC input array; MACs truncated to 32 bytes. Roles are fixed: **inviter = EAP Server, joiner = EAP Peer**, but the HTTP client flips per phase:

1. **Initial exchange — inviter drives HTTP.** `POST /v1/cluster/pairing` with envelope `{inviteId, phase:"initial", msg: base64(eap-noob blob)}`. Discovery → Negotiation (server assigns a 16-byte `PeerId`) → KeyExchange (ECDHE JWKs + nonces) → both reach Waiting. The inviter then injects the PIN: `server.OOBOutputWith(noobFromPIN(pin))`.
2. **The joiner's inbound state**: it captures the inviter's authenticated `PairingInfo` — `{nodeUuid, nodeId, name, clusterId, admissionEpoch, addr, cert(PEM)}` — which rides *inside* the MAC'd transcript, so **the certificate and UUID are cryptographically bound into the pairing**. A cert whose principal ≠ `nodeUuid` is rejected. An already-clustered joiner refuses pre-session with HTTP 409.
3. **Completion exchange — joiner drives HTTP.** Human types the PIN → `peer.OOBInputNoob(...)` → Type-5 NoobId → Type-6 MACs → EAP-Success. On the inviter, the commit happens under `rosterMu` with epoch and session-liveness rechecks — a completion landing after leave/removal commits nothing (fail-closed 409, never returning the EAP-Success frame).
4. **Post-commit handshake**: joiner persists provisional roster/pins, activates admission as the **final durable write**, adopts the inviter's `clusterId`, then POSTs `phase:"ack"` **over the new mTLS** (pinned to the inviter's exact cert DER). The inviter requires the mTLS caller's UUID == the pinned joiner UUID.

EAP sessions are **memory-only** — a restart mid-pairing fails the invite.

### 4.3 Certificates and the trust store

- **Ed25519** self-signed leaf (not RSA/ECDSA), random 128-bit serial, `CN = nodeUUID`, **URI SAN `urn:nvpair:node:<uuid>`**, NotAfter = now + 2 years, EKU ServerAuth+ClientAuth, PKCS#8 key.
- **Pinning**: trust = **byte-for-byte DER equality** against the pinned cert — no CA chain, no signature check, no expiry check at this layer. `clustertrust.go`: *"Because only paired cluster members hold a mutual pin, 'can complete the handshake' is exactly 'is a trusted member of my cluster'."* A re-pin of a *different* cert for a pinned UUID is rejected (no silent rotation; explicit re-invite required).
- On disk (`<configDir>/cluster/`, 0700): `node.key` (0600), `node.crt`, `identity.json`, `admission.json`, `members.json`, `tombstones.json`, `removal-proofs.json`, and `trusted/<peerUUID>.json` (0600): `{nodeUuid, nodeId, name, clusterId, admissionEpoch, certPem, certFingerprint, pinnedAt, endorsements[]}`. Pin validation requires filename == inner `nodeUuid` == cert principal (a renamed/tampered file can't masquerade).
- **Transitive trust via endorsements**: an Ed25519 signature by a trusted member over `nvpair-endorse:v2 ‖ introducedUUID ‖ fingerprint ‖ clusterId ‖ epochs ‖ issuedAt`. Roster gossip pins a peer only if a currently-trusted node's endorsement verifies over the exact cert fingerprint + admission epochs. Endorsement and tombstone signature domains are separated (one can't be replayed as the other).

### 4.4 Membership operations

- **Found**: founder mints a cluster UUIDv4, records self-member, activates admission atomically. **Join**: joiner *adopts the inviter's clusterId* from the authenticated transcript.
- **Admission epochs**: a node-global monotonic counter reserved before pairing, activated as the last durable write of a join. This is what makes an *old* removal proof fail against a newer readmission ("monotonicity, not density, is the security property").
- **Leave**: build a signed self-targeted `RemovalProof`, fan out to reachable peers, then journal + teardown: clear admission (retired=true, counter retained), delete all pins, reset members/tombstones/proofs. Crash-consistent via `teardown.pending` journal.
- **Remove (unpair a peer)**: persist the admission-targeted RemovalProof **before** notifying; best-effort mTLS POST; revalidate; de-pin. Offline victims converge via proof-bearing 403s on roster reconcile — requiring unanimous 403 **and** at least one authenticated proof.
- **Identity reset** = deleting the cluster dir; it orphans every pin peers hold for you and requires re-pairing. Missing keypair with existing identity fails loudly rather than re-minting.

### 4.5 The dual-personality listener

One port (`14321`), two personalities, demultiplexed by the connection's **first byte** (`shared/splitlisten`):

```go
const tlsHandshakeByte = 0x16   // TLS record content-type; no HTTP method starts with 0x16

func (s *Splitter) dispatch(conn net.Conn) {
    _ = conn.SetReadDeadline(time.Now().Add(peekTimeout)) // 5s
    b := make([]byte, 1)
    conn.Read(b)                                        // read exactly one byte
    pc := &prefixConn{Conn: conn, prefix: b[:n]}        // splice it back — no byte lost
    if b[0] == tlsHandshakeByte { s.tls.push(pc) } else { s.plain.push(pc) }
}
```

The plaintext personality serves *only* the pairing path (PIN-authenticated, pre-trust); the mTLS personality serves pairing acks, member removal, and roster reconcile. Cluster-gated verification is per-request: `tls.RequireAnyClientCert` at handshake (deliberately not rejecting there), then `VerifyClientPin` → extract claimed UUID from the URI SAN → `MatchDER(uuid, cert.Raw)` → real **403** on mismatch. Pins re-derived per request, so joining/leaving takes effect with no restart.

---

## 5. The Inference Proxy & Routing

`nvpair-proxy` is the heart. One process hosting one **facade** per engine dialect, enabled at runtime by the broker via `facade/enable {engine, port, aliasAddresses, ignorePersistedPort}` — a flag cannot carry per-engine port plans.

### 5.1 The same dual-personality listener, inverted

Each facade binds **all interfaces** on its port (default the engine's stock port) and serves two personalities:
- **Plaintext HTTP → loopback only.** Non-loopback gets `403 {"code":"loopback-only"}` — enforced, not conventional; without it the port would be an open relay.
- **mTLS → pinned cluster peers** — and crucially, peer traffic forwards **straight to the local engine** and *never re-enters candidate selection*. A peer cannot chain a request onward through a third node.

Two consequences that define the client model: (1) **a machine that is not a node has no way in** — not loopback, no pinned cert; (2) **the routing decision belongs to the machine the request originates on** — to give a workstation the whole cluster, make it a node.

Engines started by PAIR bind to loopback only, so the proxy is the sole network-facing entry point. While unclustered, the TLS handshake itself is refused (`ErrNotClustered` from `GetConfigForClient`) — no rebind needed on join/leave.

### 5.2 Port takeover and yielding

The broker plans ports (`planManagedEnginePorts`):
- **Ollama is *adopted*** — never moved while running. If it already holds `:11434`, the takeover is *blocked* (the classic failure: start Ollama desktop and it claims the port, so requests reach local Ollama without ever being routed).
- **LM Studio / llama.cpp are *managed*** — engine-manager can stop/reposition them (`lms server stop` for LM Studio).
- The Ollama backend move to `11435` is **deferred until a `ready` proves the proxy holds `:11434`**; if the move fails, the broker vacates and rolls back.
- `set-port` is a **live rebind**: bind new → persist → start fresh split listener → close old; in-flight connections drain. No restart, ever.
- An inherited `OLLAMA_HOST` naming a local port is honored as a loopback-only *alias* on the same router — clients configured through the variable enter the routing path without reconfiguration.

### 5.3 The capability gate (model matching)

Model extraction is dialect-independent: buffer the body once, `json.Unmarshal` into `{"model"}` — works for Ollama `/api/chat|generate|embed(ings)`, OpenAI `/v1/chat/completions|embeddings|responses`, and Anthropic `/v1/messages` (all carry top-level `model`). The model gates routing **only on classified inference routes**; the route table is a *classifier, not an allowlist* — unlisted paths forward verbatim (`/api/show`, `/api/pull`, `/api/ps` keep working).

Normalization per engine: Ollama `llama3` ≡ `llama3:latest` (implicit `:latest`); LM Studio and llama.cpp byte-for-byte exact IDs. The gate normalizes both the request and each advertised model, so *a model the proxy lists is one it will route*.

Candidates come from a **request-local copy** of the discovery snapshot (subscribed per-engine), so a mid-request discovery update can't mutate the in-flight list. Empty owner set → **local 502 without contacting any engine** — no broadening, no synchronous inventory refresh.

### 5.4 The ordered failover walk

The proxy produces an **ordered failover list**, not a single winner:
1. Manual pin (`node/select`) — only if still *eligible*; a pin cannot override the capability gate.
2. Scheduler priority order (`node/set-priority`).
3. Remaining owners by stable node ID (deterministic cold start).

**Retryable statuses**: 408, 429, 502, 503, 504, any ≥500, and **404 only for inference** (advertised inventory can be stale). 400/401/422 are never retried — they'd fail identically everywhere. 3xx forwarded verbatim.

**The commit point is the first body byte, not headers** (`awaitFirstBody`, 120s budget). Rationale from measurement (spec §5.2): LM Studio sends 200 + headers ~12ms after accept, then sits silent behind a busy slot — headers alone are not evidence an engine started work. Before commit, a retryable answer tears down and advances the walk with **nothing streamed to the client**; after commit, the request is bound to that node ("a stream truncated from there is the caller's to resume"). Commit vs. abandon is decided by one atomic CAS so a delivered first byte always wins the claim.

**Bounds**: 5 dispatch attempts, 10-minute job deadline, backoff 1/2/4/8s with ±25% jitter. Only *inference* re-resolves between rounds (non-inference gets one pass — prevents replaying state-changing `POST /api/pull`). Client disconnect aborts immediately.

**Latency effects**: request bodies are fully buffered once (`io.ReadAll`) and replayed per attempt — this is what makes failover-with-replay possible, and bodies are prompt+model (small), so it's cheap; consequence: no streaming *uploads*. No dial probes on the hot path (confirmed address preferred instantly; background confirmation, transport error → forget + failover). Pooled transports: one plain for self/manual, one mTLS transport **per pinned peer**. A 30s zombie guard arms around each client write so a dead client can't pin a copy goroutine.

### 5.5 Model-list fan-out & merge

`GET /api/tags` / `GET /v1/models` / `GET /models` fan out to **every** candidate concurrently (per-candidate goroutine, 16 MiB cap, `Authorization`/`Cookie` stripped so caller credentials never cross engines), merged in **candidate order for deterministic dedupe** (normalized by the engine's naming rule), re-emitted in the caller's dialect. This is why the answer is the *cluster's* inventory, not one node's.

### 5.6 Per-proxy reservations (burst spread)

Ranking can't close one gap alone: a node's pending count rises only after its workload report arrives, so requests dispatched at the same moment would all see the same idle node. Fix: each proxy adds the requests **it just dispatched** to its own estimate — `pending + gpuPressure + localReservations` — and moves the least-loaded scheduler-listed candidate to the front, holding a reservation until first byte / release on backoff / request end. The reservation map is **process-wide**, which is *why one process hosts all facades*: two facades bursting are competing for the same GPUs, so they must see each other's in-flight dispatches. Snapshots carry a monotonic generation; a redelivered broker RPC can't clear live reservations.

### 5.7 CORS — permissions are never invented

Ordinary forwards relay upstream headers verbatim, including *missing* headers and denials; proxy-generated errors carry no CORS headers (a browser may see a generic CORS failure where curl shows the real status — deliberate). Preflights fan out and **intersect**: any real denial → 403; ≥5xx = "no opinion" (skipped so one offline node can't deny every online one); credentials require unanimous explicit support; `Max-Age: 60` because the intersection is over a changing engine set.

---

## 6. Scheduling & Workload Replication

### 6.1 The scheduler (`nvpair-job-scheduler`)

The scheduler **never touches a request** and holds no socket; it only computes a preference order. Composition is one-directional: the scheduler is **model-blind** (its `workload` struct has no Model field; zero `model` matches in the package); the proxy enforces model capability *before* applying the scheduler's order. Load balancing happens **among the nodes that can serve the request**, not across the cluster.

**Inputs** (fed by the broker): `discovery:nodes-changed`, `workloads:upsert/remove`, `scheduler:telemetry`.

**Ranking**: a workload is pending while `queued` or `running`, attributed to its `scheduledOn` node. All engines count in **one node-wide pool** (work through one facade affects every other facade's ordering). Sort ascending by `pending + gpuPressure`, then lower pressure, then node ID (deterministic cold start).

**GPU pressure**: input is the **max** utilization across the node's GPUs (conservative — avoids steering work to a node with one saturated device, at the cost of overlooking idle capacity on multi-GPU nodes). Smoothed with **EWMA α = 0.35**; banded 0–3 with hysteresis — up at 40/70/85%, down at 35/65/80% (rank-thrash avoidance). Missing/invalid/stale (>10s) telemetry → **neutral pressure 1**, never treated as idle.

**Publication**: recompute on meaningful change (pending-attribution change, effective pressure change, node set change); reconcile on a 1s timer; **emit only when order, pending counts, or pressure actually changed**. One `schedule:priority {engine, nodes, ranks}` per engine — today all engines get the same node-wide ordering; the per-engine envelope exists so the contract can diverge later without a wire change.

**Relay path**: scheduler → broker caches one process-wide snapshot (deduped before minting a monotonic generation) → `node/set-priority {generation, nodes, ranks}` to each distinct live proxy → generation-gated apply (≤ current ignored; apply resets reservations). After a proxy respawn, the cached snapshot is re-sent **ungated** so a snapshot that arrived mid-death isn't lost. Scheduler restart: the broker replays active workloads, cached telemetry, and the discovery baseline before live fanout.

### 6.2 Workload replication (`nvpair-workload-manager`, port 14320)

States: `queued | running | completed | failed | cancelled`. Identity is `(originatedFrom, engine, runID, id)` — the proxy's per-process nonce (`runID`) disambiguates per-facade counters that restart at 1. The proxy stamps `scheduledOn` at the running commit, **re-points it on each dispatch and clears it between attempts** so an abandoned node stops looking busy.

Relay: `POST /v1/workloads/events`, **unconditional cluster mTLS, no plaintext personality** — leaf resolved per handshake (unclustered = handshake refused), per-request pin verify → 403. Dedup: bounded LRU (10k) keyed with the event's `seq`. Broadcast: single ordered worker (queue 1024, so a remove can't overtake its upsert), per-peer delivery with 3 attempts at 200/400/800ms backoff. **Anti-entropy**: 30s heartbeat re-asserts active + recently-terminal workloads (`resync: true` bypasses dedup); new peers get an immediate full snapshot; broker staleness backstop at 5 minutes.

**Stated limitations** (honest and useful to read in full, `architecture.mdx`): the scheduler sees pressure, not capacity (no VRAM, no latency); every workload counts as one (a three-token completion = a long generation); model load state is ignored (a request can cold-load while a warm node sits one place lower); only PAIR-routed work counts (direct-to-engine traffic is invisible to pending counts); every node ranks from its own lagging view — per-proxy reservations cover the local burst but one node cannot see what another just dispatched, so the system **self-corrects** through relay and reconciliation rather than preventing the collision.

---

## 7. Engine Management (`nvpair-engine-manager`)

Manifest-driven, engine-agnostic runner: "adding an engine is a manifest, not code." Bundled manifests are `//go:embed`-ed (`ollama.json`, `lmstudio.json`, `llamacpp.json`), overlaid by per-user override deltas deep-merged from `<appdata>/engines/*.json`. A port pin is a minimal delta `{"engine":"...","runtime":{"port":N}}`.

### 7.1 Detection

Walks the platform's `detect` array (expanded `{install_dir}`, `%VAR%`/`$VAR`, `~`); first existing file wins. Ollama probes vendor/user install locations (adopts an install you already have); llama.cpp probes **only** its own `{install_dir}` — always PAIR-managed.

### 7.2 Adoption — a real distinction, not a label

If something is already serving the engine's port and **identifies via the engine's readiness probe**, PAIR *adopts* it (reports running, never spawns a second copy). If it answers but doesn't identify → merely *occupied*: never adopted, never spawned over.

**PAIR never kills a process it did not start.** Consequences:
- Stopping an adopted engine is refused: "running under external management (pid N); stop it in its own application, then retry." (Exception: a PAIR orphan on our own port is reclaimed.)
- **Port change on adopted Ollama** → error rather than two listeners. **Adopted LM Studio** can be moved because it publishes an official stop command (`lms server stop`) — stop, persist, restart on the new port.
- Uninstall requires ownership (process path or an `installed-by-pair.json` provenance marker).

### 7.3 Readiness & health

| Engine | Ready probe | Health | Interval |
|---|---|---|---|
| Ollama | `GET /api/version` → 200 | same | 5s |
| LM Studio | `GET /v1/models` → 200 | same | 10s |
| llama.cpp | `GET /props` → 200 **+ `role:"router"`** | `GET /health` | 5s |

Poll every 300ms (3s per-attempt timeout) until ready; then a generation-gated health ticker writes the `healthy` flag — a `true→false` transition raises `engine-manager:unhealthy:<engine>`, recovery clears it. All probes carry `X-NVPAIR-Engine-Identity-Probe: 1` — which the proxy answers with **409**, so a facade can never be mistaken for the real engine.

### 7.4 Model inventory & peers

Local actions per manifest: Ollama `GET /api/tags` (`name`), LM Studio `GET /api/v1/models` (`key`), llama.cpp `GET /models` (`id`); loaded-set via row filters (`loaded_instances` nonempty / `status.value == "loaded"`). A loaded-model watcher polls every 5s, diffs order-insensitively, and mut residency-affecting actions *poke* it for sub-tick latency.

Served to peers on port **14322** (`em`) via the split listener: plaintext → loopback only (so this node's own scanner can self-enrich unclustered); LAN → cluster mTLS pin-verified. `GET /v1/models` returns `{models[union], modelsByEngine{…}, loadedByEngine{…}}` — a present-empty key means "running, none"; a missing key means "not running/not queryable".

### 7.5 The llama.cpp managed router

Readiness requires `/props` to answer `role: "router"` — a single-model `llama-server` squatting on `:8081` is **rejected, not adopted**, because its `/models` contract doesn't expose router residency state. Launch: `llama-server --sleep-idle-seconds 300 --host 127.0.0.1 --port 8081`, env `LLAMA_CACHE=~/.llamacpp` (deliberately outside PAIR's data dir so models survive uninstall/reset). Downloads are checksum-pinned release archives (CUDA 12.4/12.8 x64, 13.4 arm64, ~0.3–0.7 GiB; Metal arm64 / CPU x64 on macOS, ~11 MiB). Model pulls are SSE-subscribed (`GET /models/sse` first, then `POST /models {"model":"repo:quant"}`); a loaded model **sleeps after 5 idle minutes**, releasing model + KV-cache memory, and wakes on the next request.

### 7.6 Remote engine control (port 14323, `ec`)

mTLS-only surface (no plaintext personality; leaf per handshake; pin re-verified, and even re-verified **mid-request** for long settings operations). A trusted peer can drive: status, install (+start), pull/load/unload/delete models, start (**loopback-bound — "inference engines are never exposed on the LAN directly"**), stop, and engine settings get/preview/apply — **except CORS changes, refused remotely**. Restart and uninstall stay local-only, by design. Progress streams as NDJSON `{type: progress|result|error, opId, …}`.

---

## 8. Supporting Services

### 8.1 Node info (`nvpair-node-info`, port 14318)

`GET /v1/node-info` → `{gpus[] {name, vram_bytes, vram_used_bytes, utilization_percent}, cpu {name, cores, utilization_percent}, memory {total_bytes, used_bytes}, telemetryValid, msSince, hostUuid, clusterUuid (tri-state: absent = not pushed, present-empty = unclustered)}`. GPU utilization: Linux `nvidia-smi` (3s timeout, UMA fallback to system memory), Windows PDH counters + `GlobalMemoryStatusEx`, macOS IORegistry + gopsutil.

**Plaintext by design** — a considered trade-off, not an oversight: it's the lowest-sensitivity inter-node surface, read by consumers holding no cluster identity (the desktop's 2s telemetry poll, baseline LAN discovery). The documented consequence: *hostname, hardware inventory, and utilization for a node running PAIR are readable by anything on the same subnet that asks. Do not run PAIR on a network where that is unacceptable.* (A cluster-gated mode exists behind a flag but is deliberately unenabled.)

### 8.2 Errors (`nvpair-errors`, port 14319)

Records `{id, message, timestamp, nodeId, severity, action, engineType, operation, modelName}`; keyed `(nodeID, id)`; upsert highest-timestamp-wins, **incoming-wins-on-tie** ("still broken right now"); a local clear cannot delete a peer's entry. Cross-node sync is push-model cluster-mTLS: on local change, on peer discovery, and on a 30s heartbeat — the full local-origin envelope to `https://<host>/v1/errors`; inbound reconcile is authoritative-replace-by-origin with self-origin rejected. Peer departure evicts its entries.

### 8.3 Node settings (`nvpair-node-settings`)

A typed 4-key store (`force_ports` default true, `cluster_auto_sync`, `cluster_id`, `cluster_friendly_name`), persisted atomically (`settings.json`, tmp+rename, 0600). Malformed file → renamed `.corrupt-<ts>`, defaults load — degraded but responsive.

### 8.4 The dispatcher tool (`scripts/inference-dispatcher`)

A stdlib-only test client that deliberately knows nothing about PAIR — pointing it at a proxy is indistinguishable from pointing it at a native server, which is what makes it a valid end-to-end probe. Notable hard rule it embodies: **no PAIR component ever logs prompts or response bodies** — prompts are identified by `sha256:<prefix> len=<n>` digests, upstream failures by status + size + digest. "There is no flag that turns it off."

---

## 9. Security Model Summary

Trust boundaries (SECURITY.md, verified in code):

| Surface | Transport |
|---|---|
| Proxy, local clients | Plaintext HTTP, **loopback only** (non-loopback → 403) |
| Proxy, cluster ingress | Mutual TLS (pin-based) → forwarded to local engine only, never re-routed |
| Model inventory (`em`), remote engine control (`ec`) | Split / mTLS pin-gated |
| Workload + error sync | Unconditional mTLS |
| Pairing | Plaintext, PIN-authenticated (pre-trust by definition) |
| Host/GPU telemetry | **Plaintext, unauthenticated — documented risk acceptance** |

- **Discovery is not trust.** Every machine on the network is visible; pairing is the explicit, mutual trust decision (invitation + human-typed PIN). After pairing, "handshake completes + pin matches" *is* membership — no separate check.
- **The PIN is a bootstrap convenience**, ~20 bits, brute-forceable by an active LAN MITM (the code says so itself); it protects the cert exchange, nothing before pairing, and nothing against a compromised member.
- **mTLS scope is limited** — it does not automatically cover loopback HTTP, plain discovery metadata, node-info, Electron IPC, or third-party engines. Workers share the user's OS account.
- No relay: a non-node has no path in; ports are never to be forwarded through a router or fronted by an unauthenticated proxy.
- Privacy posture: prompts/responses stay on the LAN **when every client, engine, and node is local** — "local-first describes the intended topology. It does not prove that no data leaves the machine or LAN" (engines, catalogs, and update systems have their own network behavior).

---

## 10. Design Lessons Worth Stealing

1. **Take over the default port, not the client config.** Compatibility by port inversion beats "just point your tools at a new URL."
2. **UUIDs as correlation keys, hostnames as decoration.** A stable UUID minted once and reused across every subsystem (discovery, pins, workloads, telemetry, routing) makes identity survive renames and duplicate hostnames.
3. **Symmetric mesh, no CA.** Byte-for-byte cert pinning makes "handshake completes" == "is a member" — eliminating an entire membership-check layer. The cost is re-invite on cert rotation; for a LAN product that's the right trade.
4. **Commit at first body byte, not headers.** Measured reality (LM Studio's instant-headers-then-silence) beats the intuitive header-based commit; it makes failover invisible to the client.
5. **Evidence over claims for liveness.** Eviction requires network proof (identity-answering probes), never mDNS assertions; liveness evidence ranks *work served* above probes; inference ports are never probed (probing the serving path poisons it).
6. **Local reservations close the feedback-loop gap.** A distributed scheduler can't see same-moment dispatches; process-local reservation counters make bursts spread without waiting for replicated state. And the reservation map being process-wide is *why* one process hosts all facades — a nice example of a shared-fate decision made for correctness, not convenience.
7. **Never kill what you didn't start.** Adoption with refusal beats false control: PAIR would rather report "external management, stop it yourself" than fake a stop it can't perform.
8. **State the limitations in the docs.** The "Scheduler Limitations" section enumerates six known gaps (pressure-not-capacity, one-workload-one-unit, load-state-blindness, routed-work-only counting, lagging views, identity-order fallback) — that section is arguably more useful than the feature list for predicting real behavior.
9. **Prompts never touch logs — as a hard rule, not a setting.** Digest-based correlation proves you can debug without content.
10. **One record per node on the wire.** mDNS TXT limits forced the model list to HTTP — the right reaction to a size limit is a second fetch, not a fragmented advertisement.

---

## Appendix A — Key Constants (verified in source)

| Constant | Value | Where |
|---|---|---|
| Scan cadence / timeout | 5s / 3s | `shared/discovery` |
| Eviction: probe after / drop after | 3 misses / 12 misses | `shared/discovery` |
| Ghost nomination age / empty-scan grace | 300s / 6 scans | `directory.go` / `discovery.go` |
| Enrichment: node-info / models timeout | 3s / 6s | `node-scanner/daemon.go` |
| Telemetry cadence / staleness | ~2s jittered / 10s | `daemon.go` / `telemetry.go` |
| EWMA α / pressure bands up / down | 0.35 / 40,70,85 / 35,65,80 | `telemetry.go` |
| Neutral (unknown) pressure | 1 | `telemetry.go` |
| Scheduler interval default / floor | 1s / 200ms | `job-scheduler` |
| Worker restart: base→max backoff / budget / recovery | 1s→16s / 5 / 60s | `supervisor.go` |
| PIN TTL / sweep / max pre-invite sessions | 5min / 15s / 64 | `cluster-manager` |
| Leaf: key / lifetime | Ed25519 / 2 years | `identity.go` |
| Dispatch attempts / job deadline / first-body budget | 5 / 10min / 120s | `nvpair-proxy` |
| Retry backoff / jitter | 1,2,4,8s / ±25% | `nvpair-proxy` |
| Workload: resync / terminal retention / dedup LRU | 30s / 60s / 10000 | `workload-manager` |
| Model-list fan-out cap / CORS preflight concurrency | 16 MiB / 8 | `nvpair-proxy` |
| Frame caps (client / broker-worker) | 1 MiB / 8 MiB | `jsonrpc` / `codec.go` |
| Desktop telemetry poll / backoff max | 2s / 30s (4,8,16,30 ladder) | `node-info-poller.ts` |
| Broker→worker call timeout | 5s | `rpcworker.go` |

## Appendix B — JSON-RPC Method Surface (selected)

- **Client→broker**: `ping`, `discovery:get-nodes|subscribe`, `workloads:subscribe|get-initial`, `errors:get-initial|report|clear`, `engine:get-settings|set-port`, `node/add|remove`, `cluster:*`, `nodes:*`, `shutdown`
- **Broker→proxy**: `facade/enable`, `node/set-priority {generation, nodes, ranks}`, `node/select`, `node/set-local-backend`, `set-port`
- **Scheduler→broker**: `schedule:priority {engine, nodes[], ranks[]{id, pending, gpuPressure, rank}}`
- **Proxy→broker (workloads)**: `workload:submitted|started|completed|errored {id, model, engine, runId, state, originatedFrom, scheduledOn, seq, …}`
- **Broker→workers**: transparent pass-through relays (`<engine>-proxy:*`, `engine:*`, `cluster:*`, `settings/*`), each with its own timeout policy (engine-manager gets *no* deadline — installs run minutes)

## Appendix C — On-Disk State (per user data dir)

```
<NVPAIR data>/                    (Windows: %LocalAppData%\Nvidia Corporation\Personal AI Router;
                                   Linux: ~/.config/Nvidia Corporation/Personal AI Router)
├── node-id.json                  {node_uuid, created_at}           0600
├── cluster/                      0700
│   ├── identity.json             {node_uuid, created_at}           0600
│   ├── node.crt / node.key       Ed25519 leaf                      0644/0600
│   ├── admission.json            {counter, activated, retired, clusterId, epoch}
│   ├── members.json              sorted confirmed members
│   ├── tombstones.json / removal-proofs.json / invite-created-cluster.json
│   ├── teardown.pending          crash-consistent leave journal
│   └── trusted/<peerUUID>.json   {nodeUuid, certPem, certFingerprint, admissionEpoch, endorsements[]}  0600
├── engines/<engine>.json         per-user manifest override deltas (port pins)
├── proxy-port.json               per-facade persisted port (one file per engine)
├── settings.json                 node-settings typed KV
├── manual-nodes.json
└── engine-bin/<engine>/          PAIR-installed engine binaries
```

Model weights stay in engine-owned storage (`~/.ollama`, `~/.lmstudio/models`, `~/.llamacpp`) — deliberately outside PAIR's data dir so they survive uninstall and reset.
