# RTK (Rust Token Killer) — Technical Study

*Study of `github.com/rtk-ai/rtk` (v0.49.0, ~295 Rust files / ~193K lines, October 2026): the open-source CLI proxy that compresses command output before it reaches an LLM's context. Based on direct code study — architecture docs plus six parallel deep-dives into the hook system, core infrastructure, filter ecosystem, LLM-proxy daemon, engine/analytics, and extensibility/security surfaces.*

---

## 1. Executive summary

RTK answers a concrete inefficiency of agentic coding: every `git status`, `cargo test`, or `npm install` dumps thousands of bytes of boilerplate — progress bars, ANSI escapes, passing tests, hints — into the model's context window, where they cost tokens and crowd out useful context. RTK sits between the agent and the shell and **filters command output before the model ever sees it** (each filter must cut ≥20% of bash-output bytes; most cut 60–99%). It is a single Rust binary with no runtime dependencies, adding <10 ms per command, plus a separate source-available daemon that applies the same philosophy to provider-API traffic.

The architecture has four moving parts:

1. **Hooks (18 AI tools)** — a `PreToolUse` hook rewrites `git status` → `rtk git status` before execution. One Rust decision engine (`rtk rewrite` / `rtk hook <agent>`), per-agent JSON adapters; thin delegates (shell scripts, TS/Python plugins) never contain rewrite logic.
2. **Filters (two layers)** — compiled Rust modules per ecosystem (git, cargo, pytest, kubectl…; structured parsing, flag injection, state machines) plus a declarative TOML DSL (62 built-in filters, 8-stage regex pipeline) reached only through the unknown-command fallback.
3. **The recall store** — every elision is recoverable: raw output is gzipped into a content-addressed SQLite DB and the model gets a hint (`[+N hidden: rtk recall 3f9c2a81d4e7]`) to fetch back exactly what was dropped instead of re-running the command.
4. **The LLM proxy** (`proxy/`, BUSL-1.1, separate workspace) — a loopback daemon (`ANTHROPIC_BASE_URL` → `127.0.0.1:8080`) that strips ANSI/progress-redraw/duplicate lines from provider responses *without breaking prompt-cache prefixes*, and can optionally run CCR (Compress-Cache-Retrieve): reversible block compression with an injected `rtk_retrieve` tool.

What makes RTK worth studying is not compression — it's the **discipline**: fail-open everywhere (a panicking filter returns raw output; the release profile keeps unwind tables for exactly this), exit-code preservation for CI, a `never_worse` guard so a filter can never emit more than the raw output, "Default never auto-allows" as a hard security invariant, and unusual metric honesty (no tokenizer — tokens are `bytes/4`, so ratios are reliable but absolute counts are approximate; a 7,500-token input cap excludes inflated savings from headlines; negative savings are stored as negative).

---

## 2. The problem and the proxy pattern

An LLM agent's shell-tool loop consumes tokens for every command output it reads. Most of those bytes are noise that a human never reads and a model doesn't need: `Compiling foo…` lines, test-by-test `ok` lines, ANSI-colored progress bars that render as escape soup, hint lines like `(use "git push" to publish)`. RTK's insight is that the *compression point* belongs between the command and the model, not inside the agent: the agent's harness never changes, the agent itself is unmodified — the hook rewrites the command, the CLI filters the output.

The design principles, from the docs and verified in code:

- **Transparent proxy**: unknown commands pass through unchanged (inherit stdio, tracked as 0% reduction).
- **Fail-safe**: filter failure falls back to raw output — a filter panic is caught (`catch_unwind` in the Claude PostToolUse path), and the release profile deliberately does **not** set `panic = "abort"` because unwinding is the fail-open mechanism.
- **Exit-code preservation**: RTK never swallows a non-zero exit; signals map to 128+n; spawn failure → 127. `Result<i32>` flows to `std::process::exit` — CI reliability is a contract, not a hope.
- **<10 ms startup**: the hook runs on *every* command an agent issues; single-threaded, no async, `LazyLock` regex compilation, config cached in a `OnceLock`, no config I/O at startup.

---

## 3. The hook system — 18 agents, one decision engine

### 3.1 Architecture: thin delegates, one brain

Every integration funnels into a single Rust decision function; no rewrite logic is duplicated outside the binary:

```
Agent tool fires PreToolUse (JSON on stdin)
        ↓
Hook adapter (shell script / TS plugin / Python plugin / native processor)
        ↓
rtk rewrite "<command>"      ← one subprocess, exit-code protocol
   or rtk hook <agent>       ← in-process, agent-specific JSON
        ↓
src/hooks/decision.rs::decide_for_agent()
        ↓
Agent-specific JSON response (updatedInput / deny / ask / silence)
```

Three adapter shapes, driven by what each agent supports: **native processors** (`rtk hook claude|cursor|gemini|codex|copilot|trae|vibe|antigravity|droid` read stdin, answer in-process), **TS plugins** (OpenCode, Pi, OMP — in-place command mutation), **a Python plugin** (Hermes — calls `rtk rewrite` as a subprocess and mutates `args["command"]`), and **prompt-level rules files** for agents with no hook surface at all (Windsurf `.windsurfrules`, Cline `.clinerules`, Kilo/Kimi rules + `AGENTS.md` awareness — the "full" awareness text is the only one that asks the model to prefix `rtk` itself).

`rtk init` writes per-agent artifacts idempotently (atomic tempfile-rename, `.bak` backups, `write_if_changed`): Claude gets a `settings.json` `PreToolUse` entry pointing at `rtk hook claude` plus an `RTK.md` awareness file referenced from `CLAUDE.md`; Cursor gets a flat `hooks.json` entry; Gemini gets a 2-line shell script whose **SHA-256 hash is baselined at install** and re-verified at runtime; Codex gets hooks + an `@RTK.md` reference in `AGENTS.md` and refuses symlinked hook files (an anti-escape measure); the Hermes plugin lands in `~/.hermes/plugins/rtk-rewrite/` and is enabled by a line-oriented hand-edit of `config.yaml` (handles empty config, missing `enabled:`, inline and block list forms — idempotently, with tests pinning each).

### 3.2 The decision engine

One enum, four outcomes (`decision.rs`): `AllowRewrite(cmd)` / `AskRewrite(cmd)` / `Defer` / `Deny`. The gate order is load-bearing:

1. **Deny first** — a deny rule anywhere in the chain kills the call; a rewrite can never rescue a denied command.
2. **Defer** — matched inside a quoted shell wrapper: RTK steps aside so the host's native deny sees the command as typed.
3. **Unattestable constructs** — `$(...)`, backticks, file-target redirects, heredocs, `$((`: `Defer`, refused rewrite outright (stronger than an Ask — a rewrite could otherwise smuggle an unchecked command past an allow rule).
4. **Rewrite iff a rule matches**, auto-allowed **only** on an explicit `Allow` verdict. The invariant that matters (**#1155**): `Default` (no rule matched) can *never* produce `AllowRewrite` — a machine with zero configured permission rules never auto-approves anything.
5. Identity suppression: an already-`rtk` command rewrites to a no-op → `Defer` (hosts render no-op edits pathologically).

`rtk rewrite`'s exit-code protocol is the universal contract for delegates: **0** = rewritten+allowed (stdout), **3** = rewritten but ask, **1** = no equivalent (passthrough), **2** = deny. `RTK_REWRITE_HOST=openclaw` names a delegate that owns approval itself — it relaxes *only* a Default ask into Allow, never an explicit ask rule, never a deny (pinned by tests). Unknown host values fail closed to the stricter behavior.

### 3.3 Permissions — whose rules govern?

The hook never invents authority; it consults the *host agent's own* permission rules (`permissions.rs`): Claude's `settings.json` `Bash(...)` rules (4 files, project+global, incl. `.local`), Cursor's global `cli-config.json` `Shell()` rules, Gemini's folder-trust-gated settings, Droid's deny-lists (deny-only — RTK never asserts allows for Droid). Codex/Trae/Vibe/Antigravity get no RTK-side rules at all: the host's native approval runs *on the rewritten argv*. Verdict precedence: **Deny > Defer > Ask > Allow > Default(ask)**, with compound-command hardening (every segment must independently match an allow rule — #1213), grammar-residue stripping that can only make verdicts stricter, and a 600+-case generated prefix-matrix test asserting no redirect/grouping/negation prefix can hide a denied command.

### 3.4 The rewrite registry

The matcher is a `const` table of **98 `RtkRule`s** (`src/discover/rules.rs`), each an anchored regex + `rtk` equivalent + pipeline-safety class + category + expected savings, compiled once into a `RegexSet`. The classification pipeline: ignore-lists (`cd`, `echo`, `rm`, shell keywords, `rtk` itself) → strip env prefixes (`sudo` deliberately **not** stripped — #146) → normalize absolute paths → strip global opts (`git -C /tmp status` → `git status`) → redirect guards → match (last match wins) → per-subcommand savings overrides.

Compound commands are the hard part, and the handling is careful (`registry.rs`): a single-pass lexer state machine tokenizes; `&&`/`||`/`;`/`&` split into independently-rewritten segments; **pipes keep producers raw** and rewrite only a pipeline-safe final stage (`grep`/`rg` yes; git/cargo are producer-only; `|&` keeps the whole pipeline raw) — because `rtk git log | wc -l` would break if the producer's output shape changed. Quoted `sh -c` wrappers get one level of dialect-aware inner rewriting (fish `and/or/end` defer). If *any* segment fails to match, it stays raw; a full `None` is returned only when nothing was rewritten. Bash recombines the segments at execution time.

### 3.5 Integrity and trust

Motivated by a real advisory (SA-2025-RTK-001: the PreToolUse hook *auto-allows* commands, so a tampered hook is a command-injection vector): installed hooks are hashed at install (SHA-256, read-only 0o444 baseline) and verified at runtime; tampering blocks execution. Project-local TOML filters and pipeline files are trust-gated (`.rtk/` content hash must match what `rtk trust` recorded — changed-since-trusted files are silently skipped). `rtk hook-audit` parses an opt-in audit log into skip-reason breakdowns (`skip:deny_rule`, `skip:defer`, `skip:no_rewrite`…).

---

## 4. The filter ecosystem — two layers, one boundary

### 4.1 Rust filters (`src/cmds/` — 12 ecosystems, 86 files)

Routed by Clap (the `Commands` enum); each module follows the shared skeleton in `core/runner.rs`: spawn → capture → `filter_fn(&str, exit_code) -> String` → guard → tee-hint → track → return exit code. Representative mechanics:

- **git status**: compact path runs a plain `git status` under `LC_ALL=C` (for state extraction) plus `--porcelain -b`; renders branch line + entries, "clean — nothing to commit", a rebase/merge/bisect state header.
- **git diff**: raw passthrough for `--stat`/`--quiet`/patch-shape flags; compact path runs a probe `git diff --no-patch --stat` and prints the stat header plus a compact diff (hunk headers, ±lines capped at 100/hunk, ≤3 context lines per hunk, per-file counters, `[+N lines omitted]`).
- **git log**: raw-shape flags (`-p`, `--stat`) → passthrough with RTK's default `-10` injected only when the user bounded nothing, plus a second-walk probe that admits the cap bit only when it actually bit; compact path injects a `--pretty=format:…---END---` delimiting format (when the user named none), keeps ≤3 body lines per commit, drops trailer noise (`Signed-off-by:`).
- **pytest**: a state machine (`Header → TestProgress → Failures → Summary`) over ANSI-stripped lines; failures numbered with ≤3 relevant lines each (`>`/`E`/assert lines); all-pass collapses to `Pytest: N passed in Ds`.
- **cargo build/test**: streamed handlers that drop `Compiling`/`Downloading`/`Finished`, emit error/warning blocks live, and merge test suites into one line (`15 passed, 2 ignored (3 suites, 0.04s)`).
- **grep/rg**: runs the invoked engine (never substitutes rg↔grep), mirrors real grep's output shape (filename only when multi-file, line numbers only with `-n`), and uses the grouped `N matches in M files` form only when capping actually shrank output *and* beats the faithful baseline.
- **ls**: forces `LC_ALL=C`, drops noise dirs (`node_modules`, `target`…) unless `-a`, dirs-first `name/` + `name size`, 50-entry cap with a hint holding exactly the hidden entries.

### 4.2 The TOML DSL (`src/filters/*.toml` — 62 built-ins)

A fully data-driven second layer, reached **only** when Clap parsing fails (unknown commands) — the layers are disjoint, and `compile_filter` even warns when a TOML `match_command` overlaps a Rust-handled command ("this filter will never activate"). One file = `schema_version` + N `[filters.<name>]` tables + inline `[[tests]]`:

```toml
[filters.make]
match_command = "^make\\b"          # must be fully anchored (^ on every top-level branch)
strip_ansi = true
strip_lines_matching = ["^make\\[\\d+\\]"]  # or keep_lines_matching; RegexSet
max_lines = 60
on_empty = "make: no output"
[[tests.make]]                      # inline tests, run by `rtk verify`
input = """...real output..."""
expect_contains = ["..."]
```

The **8-stage pipeline**: strip_ansi → chained `replace` → `match_output` short-circuits (with `unless` guards so errors aren't swallowed) → strip/keep lines → truncate_lines_at → head+tail → max_lines → on_empty. A `Lossiness` return value distinguishes tail-recoverable cuts from whole-blob losses — the caller turns them into recall hints, and **a lossy filter without a hint falls back to raw output**: no unrecoverable truncation ever ships.

Filter sources, first-match-wins: project `.rtk/filters.toml` (committable, **trust-gated** — SA-2025-RTK-002) → user `~/.config/rtk/filters.toml` (trust-gated) → built-ins (concatenated alphabetically by `build.rs`, parse-validated and duplicate-checked *at compile time*, embedded via `include_str!`). Anchoring is enforced or the filter is dropped (prevents `timeout 5 /usr/bin/liquibase` from activating a filter mid-command). Admission bar for TOML filters is 60% reduction (higher than the global 20% floor) — TOML is for predictable line-noise; Rust is for JSON/state machines/flag injection.

### 4.3 The 20% floor — enforced in layers, not by a gate

There is no single runtime check. It's a discipline stack: policy docs (CONTRIBUTING admission criteria + PR checklist), **hard unit-test assertions** (`savings >= 20.0` against real captured fixtures; `output * 5 <= input * 4` property tests on metadata-heavy corpora), TOML-side build-time validation (build fails on invalid/duplicate filters; a test fails if any of the 62 built-ins lacks inline tests), a CI job that fails a PR touching a `*_cmd.rs` without `#[cfg(test)]`, a benchmark job that hard-fails only on *negative* savings or empty-where-raw-had-content, and the runtime `never_worse` guard enforcing ≥0% per invocation.

### 4.4 Core plumbing worth stealing

- **`never_worse` guard** (`guard.rs`): if `estimate_tokens(filtered) > estimate_tokens(raw)`, print raw. Ties go to filtered. Waivers are enumerated and bounded (e.g. ruff's injected `[]` clean-output format).
- **The recall store** (`retriever.rs`): SQLite `~/.local/share/rtk/recall.db`, content-addressed by `SHA-256(command || 0x00 || content)[:12]` — re-running the same command rewrites the same entry. Gzipped, byte-faithful, 10 MiB/entry, 200 entries FIFO, 30-day retention. Over-cap entries store the **hidden tail** (not the prefix the model already saw), so a hint never promises more than recall returns. `rtk recall <hash>` defaults to exactly the elided tail; `--grep`/`--from`/`--full` for more. Failure output tees on non-zero exit (≥500 bytes); truncated lists emit `[+N hidden: rtk recall <hash>]`. Posthook writes have their own eviction budget so they can never evict failure-recovery entries. `rtk gain --recalls` reports whether elided output is ever actually consulted — an honest check on over-aggressive filtering.
- **`arg_tokenizer.rs`** (1,448 lines): the reason flag inspection is never string-scanned. Typed tokens over a per-tool, per-flag `takes_value` table with five independent dialect axes (single-dash clusters, attach style, `--` boundary semantics, case folding, MSBuild `/flag`); `-20` stays one token; `git -M` is attached-only; grep's `--` claims dash-dash, git's doesn't. The docs' four rules: one grammar per subcommand; scope lookups to the region the tool parses; inject before the `--` boundary, never append; one rule to detect and act.
- **Language-aware source filtering** (`filter.rs`): Minimal strips comments (Python via a triple-quote string-aware scanner — `"""` is a *string*, not a comment; `/*` inside a glob like `"src/*.rs"` doesn't open a block); Aggressive adds brace-depth function-body elision with a literal `// ... implementation` marker. Explicitly a line/brace heuristic, **not** an AST — cheap by design.

---

## 5. The engine and the two editions

The default `rtk` binary links `rtk-engine` (`engine/`, **BUSL-1.1**): the *pipeline* engine that parses user `pipeline.toml` files (rules/directives/methods/tasks/classifiers positioned at `hook.before_tool` / `hook.after_tool` / `proxy.request`), evaluates them at hook and proxy sites, and runs compression method chains (`term` = `[ansi, progress, dedup, base64-elide, cap]` — "cap is tail-biased — the one deliberately lossy step"). Six reference pipelines ship embedded (`rtk:default`, `rtk:posthook`, `rtk:proxy`, …), each declaring its benchmark status honestly ("Benchmark: not published" — CONTRIBUTING explicitly prefers that to fabrication). A guide-as-tests harness extracts every TOML block from the docs and validates it, making the documentation normative and executable.

`--no-default-features` builds the **Apache-2.0 "oss" edition**: same hook rewrites, same command filters, but no pipeline engine — a `pipeline.toml` on disk is inert, and `pipeline_config_oss.rs` returns exactly the no-file answers. The engine even reserves pro keys (`locked`, `team`, `user_role`, `spend`, a `redact` method) that **parse but never fire** in OSS builds, emitting a W009 "rule is inert; the file still loads" — forward-compat hooks for a commercial tier, handled without breaking the open build. CI asserts OSS purity (no BUSL crate in the `--no-default-features` dependency graph).

Load resolution is trust-gated and fail-open: user files → sorted user `pipelines/*.toml` → trust-gated project files; parse failures drop the culprit file and re-resolve; unattributable errors fall back to compiled `rtk:default` with `fell_back = true`. A per-chain panic in method execution is contained (`run_chain_caught`) → no output → raw passthrough.

**Plugins** are compile-time, not dynamic: an SDK crate (`rtk-plugin`, re-exporting the engine's `Method`/`TaskKind` traits), a `rtk_plugin!` registration macro, and `rtk plugins build` synthesizes a host crate that statically links `rtk` + user plugins and installs a custom binary (still named `rtk` so hook configs keep working). Every method declares a typed contract — `lossy` methods must declare a `Recovery` path (Recall / RetrieveTool / Pointer); built-ins can't be shadowed; a rule referencing a missing plugin method fails open (rule skipped), but a control rule depending on it refuses to load its whole file (fail-closed where it matters).

---

## 6. The LLM proxy — compression at the provider boundary

`proxy/` is a deliberate architectural sibling, not a member of the workspace: the CLI is synchronous with a <10 ms budget, and linking tokio/axum into it would tax every `rtk ls`. So `rtk-proxy` is a separate BUSL-1.1 binary (Change Date 2030-09-09 → auto-converts to Apache-2.0; free production use, no offering it to third parties as a paid product) that the CLI *spawns* and whose SQLite DB it *reads* — two licenses in two artifacts, one install path (`cargo install --git` still works because only one bin package installs).

**Interception** is base-URL rewrite, not MITM: `rtk claude` execs Claude Code with `ANTHROPIC_BASE_URL=http://127.0.0.1:8080`; other on-ramps are `init`-written settings env, `eval "$(rtk llm-proxy start --env)"`, and a SessionStart hook keep-alive. Loopback policy is strict (non-loopback `start` warns, `service install` refuses; `/admin/settings` can't change upstreams — a credential-stealing vector — and requires a custom header so browsers must preflight). `/health` must answer `service: "rtk-proxy"` because 8080 is the most common dev-server port on Earth; a bare 200 proves nothing. A recent fix (4a583f8f) made every CLI surface honor `RTK_PROXY_BIND` so a machine with something else on 8080 still works.

**What it strips** (the `normalize` stage, Anthropic dialect only): ANSI CSI/OSC sequences, `\r`-overwritten progress redraws (keep the final render), runs of ≥3 identical adjacent lines → one + `[xN]`. The constraint that shapes everything: **cache stability** — every transform is a pure function of block bytes so re-sent history prefixes stay byte-identical and prompt-cache hits survive (enforced as a validation error; `Read`/`NotebookRead` tool results are exempt entirely because a byte-shift breaks later `Edit` matching). Blocks already truncated by the hook are never re-cut.

**CCR (Compress-Cache-Retrieve)** is the flagship opt-in: oversized tool-result blocks become head+tail previews plus a marker — `[rtk-proxy: compressed N tokens — call rtk_retrieve("<id>") to get the full content]` — with the original stored in a second SQLite DB (ICM library, topic `ccr/{session}/{id}`, 7-day TTL). The proxy *injects an `rtk_retrieve` tool into the request*; if the model calls it, an internal loop (max 6 rounds) resolves retrieves and re-calls upstream, synthesizing SSE back to a streaming client that never sees the detour. The guards are the interesting part: off by default *twice* (config AND a pipeline rule must name it); **Read results are never compressed** (`read_min_tokens = usize::MAX` — recovery paging would cost more than compression saves); a block retrieved ≥2 times is pinned exempt for the session (an unresolved retrieve would recur every turn).

**Usage accounting** writes a ~30-column SQLite row per request — including cache read/creation tokens, per-stage saved tokens, and `cost_without` (a counterfactual: removed tokens added back, priced at the pre-route model). `rtk gain` reads it read-only (500 ms busy timeout, bounded windows) and deliberately shows no dollars unless the operator supplied a price table — "rtk ships none," so cost claims stay the operator's own.

**Protocol coverage**: Anthropic Messages fully (normalize, cache-align, CCR); OpenAI Chat Completions is forwarded with usage accounting but "every shipped transform is Anthropic-only"; Gemini exists only for usage extraction. Streaming SSE both directions, chunk-tallied; streamed responses are exempt from the 600 s buffered timeout and bounded by read-silence instead. Unknown paths 404 rather than relay a client's `Authorization` header to a guessed host.

---

## 7. Security, telemetry, and honesty

- **Fail-open is a philosophy, not an accident**: hooks exit 0 on every error path (a non-zero exit would block the agent's command); filter panics return raw output; recall-store failure just drops the hint; classifier failure means a fact is absent so a rule doesn't fire. The one place RTK fails *closed* is exactly right: authority (Deny, Default-never-allows, trust-gated filters, tamper-checked hooks).
- **Telemetry is quadruply opt-in**: build-time endpoint (no `RTK_TELEMETRY_URL` in CI → binary has no telemetry at all), env opt-out, explicit consent (GDPR framing), config flag — then a 23 h rate marker, fire-and-forget. Payload: a salted device hash, version/OS/arch, install method, aggregate counts; command names reduced to `rtk <tool>` (operands and paths never stored); recall-stats slugs pass an allowlist or fold into "other." `rtk telemetry forget` withdraws consent, deletes the salt/marker/history DB, and POSTs an erasure request. The leak-guard reasoning is documented per field.
- **SECURITY.md** runs a real process: private reporting, 48 h ack, 90-day embargo, tiered high-risk files needing 2 reviewers, automated cargo-audit/semgrep, a dangerous-pattern table, and a new-dependency policy (download volume, maintainer verification, license, typosquat check).
- **Distribution**: release-please semver on master, dev pre-releases on develop; 5-target × 2-edition release matrix plus a separate proxy archive; brew tap (two formulae, mutually `conflicts_with`), deb/rpm stable aliases, winget, nix, and an `install.sh` that refuses to install without checksums.txt (SHA-256 verification; tar path-traversal check; `RTK_SKIP_CHECKSUM=1` as the explicit at-your-own-risk bypass). Not on crates.io — a *different* "rtk" (Rust Type Kit) squats the name; `rtk gain` working is the identity check.
- **Testing**: ~4,283 inline unit tests colocated with modules (a CI job fails a PR touching a filter without tests); 206 real captured fixtures (no snapshot framework); protocol-pinning tests for the exit-code contract (876 lines); a 683-line seeded differential fuzzer that builds a hermetic git repo and asserts blob classification against `git cat-file -t` ground truth — "written because the bug reopened three times"; a 596-line post-merge bash smoke suite over live tools; integration tests that *fail* if a spawned binary escapes test isolation and writes into the contributor's real home.

### Doc-vs-code discrepancies found (small but real)

1. `tracking.rs` module docs say `tracking.db`; the actual filename is `history.db` (`constants.rs:2`).
2. `analytics/README.md` claims "read-only dashboards," but `rtk gain --reset` DELETEs from five tables.
3. `ccusage.rs` docs claim a 90-day window; the code hardcodes `--since 20250101`.
4. `rtk session` counts adoption more generously than `rtk discover` (no hook-install/permission/`hook_decisions` checks) — the two will disagree on the same history; `discover` is the measured one.
5. The "engine" name is a trap: docs call the TOML filter interpreter "the TOML DSL filter engine" while `engine/` is the unrelated BUSL *pipeline* engine.

---

## 8. Design lessons

1. **Compress at the boundary the model reads, not inside the agent.** RTK never patches an agent — it wraps the command and filters the output. Every agent integration is a ~50-line adapter; the brain is one binary.
2. **Fail-open on quality, fail-closed on authority.** Compression is allowed to break (raw output survives); permission decisions are not (Deny first, Default never auto-allows, unattestable constructs defer). Knowing which category each failure belongs to is the whole security model.
3. **Every lossy operation needs a recovery path.** The recall store turns "we truncated" from data loss into a pointer — and `gain --recalls` audits whether recovery was ever needed. A lossy filter without a hint falls back to raw: truncation without recovery never ships.
4. **Guard the metric you actually claim.** `never_worse` enforces the ≥0 case per invocation; negative savings are stored negative; oversized inputs are excluded from headlines; no tokenizer means ratios are honest and absolutes are labeled approximate. The number users see is engineered not to lie.
5. **One decision engine, N adapters.** The 18-agent surface would be unmaintainable if each hook held logic; instead each adapter only translates JSON shapes, and correctness improvements (compound-command handling, deny-in-wrapper, identity suppression) land once for everyone.
6. **Keep the hot path boring.** Single-threaded, no async, lazy regex, cached config, per-call SQLite with version-gated migrations — because the hook runs on *every* command. The async daemon is a separate binary reached by spawn+HTTP, keeping 10 ms startup a hard budget rather than an aspiration.
7. **Licenses as architecture.** Apache CLI + BUSL engine/proxy in separate workspaces/artifacts lets one repo ship an OSI edition, a paid-tier surface (inert pro keys), and a commercial daemon — with CI asserting the OSS build is actually pure.

---

## 9. Sources

- Repository `github.com/rtk-ai/rtk` @ v0.49.0 (clone at `/opt/data/rtk`), read directly: `docs/contributing/ARCHITECTURE.md` (987 lines), `docs/contributing/TECHNICAL.md` (444), `CLAUDE.md`, `src/hooks/` (decision, hook_cmd + per-agent processors, init, integrity, permissions, rewrite_cmd), `src/discover/` (rules.rs — 98 rules, registry.rs, lexer.rs), `src/core/` (runner, guard, tracker/retriever/tee, arg_tokenizer, filter, toml_filter, pipeline_config[_oss], config), `src/cmds/` (git 6,717 lines, cargo, pytest, search, ls), `src/filters/*.toml` (62), `engine/` (pipeline schema/methods/builtins, pro-key surface), `proxy/` (server, forward, normalize, ccr, store, config), `plugin/`, `openclaw/`, `hooks/hermes/`, plus `.github/workflows/`, `install.sh`, `Formula/`, `release-please-config.json`, `SECURITY.md`, `DISCLAIMER.md`
- Six parallel subsystem deep-dives (subagent reports, deleg_b4c1bc1f): hook system & rewrite engine; core infrastructure; command-filter ecosystem incl. a traced `rtk git log --oneline -20`; the LLM proxy daemon; engine/learn/discover/analytics; extensibility/security/testing

*All claims above are from code or first-line docs in the repository at the studied revision; the five doc-vs-code discrepancies found are listed in §7. RTK's own framing is worth repeating: every percentage measures bash output, not your bill — tokens are estimated `bytes/4`.*
