I Read Hermes Agent's Memory System, and It Fixes What OpenClaw Got Wrong
If you've read my previous posts on ChatGPT memory, Claude memory, and OpenClaw memory, you already know I keep coming back to the same question: how do these agents actually remember?
Hermes Agent was particularly interesting to me because this time, I did not have to reverse engineer everything from behavior alone. Hermes is open source, and both the repo and the docs are public. So instead of poking a black box with prompts, I went straight to the code paths that build prompt state, persist sessions, flush memories, and query past conversations.
The short version is this: Hermes does not have one memory system. It has four.
1. A very small, curated prompt memory stored in `MEMORY.md` and `USER.md`.
2. A searchable SQLite archive of past sessions exposed through `session_search`.
3. Agent-managed skills that act like procedural memory
4. An optional Honcho layer for deeper user modeling
And the key design choice tying all of this together is simple: keep the prompt stable for caching, and push everything else to tools.
Let's dive right in.
Hermes's Context Structure
Before understanding memory, it helps to understand what Hermes is actually sending to the model.
The system prompt is assembled roughly like this:
[0] Default agent identity
[1] Tool-aware behavior guidance
[2] Honcho integration block (optional)
[3] Optional system message
[4] Frozen MEMORY.md snapshot
[5] Frozen USER.md snapshot
[6] Skills index
[7] Context files (AGENTS.md, SOUL.md, .cursorrules, .cursor/rules/*.mdc)
[8] Date/time + platform hints
[9] Conversation history
[10] Current user message
This matters because Hermes is optimizing for provider-side prompt caching. The prompt builder is very explicit about this in the source: the stable prefix should stay stable for as long as possible.
That one decision explains most of Hermes's memory architecture.
If a piece of information should be available on every turn, Hermes tries to keep it tiny and inject it once. If it is large, historical, or only occasionally useful, Hermes pushes it out of the prompt and retrieves it on demand.
Layer 1: Frozen Prompt Memory
The built-in memory system is surprisingly small.
Hermes stores durable memory in two files under `~/.hermes/memories/`:
| File | Purpose | Limit |
|------|---------|-------|
| `MEMORY.md` | Agent notes about environment, conventions, tool quirks, lessons learned | 2,200 chars |
| `USER.md` | User profile: preferences, communication style, identity | 1,375 chars |
That is not much. Roughly 1,300 tokens combined.
And that is deliberate.
At session start, Hermes loads both files, renders them into a prompt block, and then freezes that snapshot for the rest of the session. Mid-session writes are persisted to disk immediately, but they do not mutate the already-built system prompt. Those changes only show up when a new session starts, or after a compression-triggered prompt rebuild.
The rendered format looks like this:
══════════════════════════════════════════════
MEMORY (your personal notes) [67% — 1,474/2,200 chars]
══════════════════════════════════════════════
User's project is a Rust web service at ~/code/myapi using Axum + SQLx
§
This machine runs Ubuntu 22.04, has Docker and Podman installed
§
User prefers concise responses, dislikes verbose explanations
There are a few subtle design choices here that I like:
1. It uses character limits, not token limits
This keeps the memory logic model-agnostic. Hermes does not need model-specific tokenization just to decide whether memory is full.
2. It uses a simple delimiter based file format
Entries are separated with `§`. No vector DB. No custom binary store. Just plain text files.
3. It keeps system-prompt memory intentionally tiny
This is probably the most important point in the entire design. Hermes is not trying to stuff its entire history into prompt memory. It wants only the highest-value facts there.
4. It treats memory as a curated state, not a diary
This is where Hermes is very different from OpenClaw.
OpenClaw had an append-only flavor to its daily logs. Hermes explicitly pushes in the opposite direction. The tool schema and tests say:
Save user preferences
Save the environment facts
Save recurring corrections
Save stable conventions
Do not save task progress
Do not save session outcomes
Do not save the temporary TODO state
The truth is that Hermes wants `MEMORY.md` and `USER.md` to stay hot, compact, and cache-friendly.
The `memory` Tool
Hermes manages these files through a single `memory` tool with three actions:
`add`
`replace`
`remove`
There is no real read action in the current tool surface because the memory is already injected into the prompt at session start.
One nice usability detail is that `replace` and `remove` use substring matching. You do not need an internal ID. You just pass a unique substring from the existing entry.
Example:
python
memory(
    action="replace",
    target="memory",
    old_text="dark mode",
    content="User prefers light mode in VS Code, dark mode in terminal"
)
The system also rejects exact duplicates and blocks dangerous content before it ever enters prompt memory. The source scans memory entries for prompt-injection patterns, credential exfiltration strings, SSH backdoor hints, and invisible Unicode characters.
That makes sense. Anything written to memory is effectively becoming part of the future system prompt.
Layer 2: `session_search` for Episodic Recall
If `MEMORY.md` and `USER.md` are Hermes's hot memory, then `session_search` is its long-tail recall system.
All past sessions are stored in `~/.hermes/state.db`, a SQLite database with:
A `sessions` table
A `messages` table
An FTS5 full-text search index
Lineage links through `parent_session_id`
When the model needs to recall something from a previous conversation, Hermes does not search `MEMORY.md`. It searches the session database.
The pipeline looks like this:
FTS5 search over past messages
-> group results by session
-> resolve parent/child lineage
-> load top matching sessions
-> truncate transcript around relevant matches
-> summarize each session with a cheap auxiliary model
-> return focused recaps to the main model
This is a very different philosophy from systems that try to semantically index every memory note.
Hermes is basically saying:
Keep the always-injected memory tiny
Store the real history in SQLite
Search that history only when needed
Summarize the results before handing them back
That is a pragmatic design.
It is also cheaper than blindly stuffing long histories into every prompt.
The docs describe `session_search` as a way to answer questions like:
"Did we discuss this last week?"
"What did we do about X?"
"As I mentioned before..."
In other words, `MEMORY.md` is for durable facts, while `session_search` is for episodic recall.
Layer 3: Compression and the Memory Flush
Another clever part of Hermes is what happens before it compresses a long conversation.
As sessions grow, Hermes eventually summarizes the middle portion of the conversation to stay within the model's context window. But summarization is lossy. Important facts can disappear.
So Hermes does a memory flush first.
Before compression, it injects a synthetic system/user instruction that basically says:
The session is being compressed.
Save anything worth remembering.
Prioritize user preferences, corrections, and recurring patterns over task-specific details.
Then it runs one extra model call with only the `memory` tool available.
If the model decides something should survive compression, it writes it to `MEMORY.md` or `USER.md` before the conversation gets summarized away.
That is a genuinely good pattern.
It gives the model one last chance to distill the durable bits before the middle of the conversation is collapsed.
Even better, after compression, Hermes invalidates and rebuilds the cached system prompt, reloading memory from disk. That means anything flushed right before compression becomes part of the next stable prompt snapshot.
So the flow is:
Long conversation
→ flush durable facts to memory
→ compress old turns
→ rebuild prompt
→ continue with smaller context and updated memory
This is the kind of thing that makes Hermes feel like an actual memory architecture instead of a bolt-on note store.
Layer 4: Skills as Procedural Memory
Hermes's memory story is not just facts and transcripts.
It also has skills.
Skills live under `~/.hermes/skills/` and act like reusable knowledge documents. The docs explicitly describe them as the agent's procedural memory.
When Hermes discovers a non-trivial workflow, fixes a tricky issue, or learns a better way to do something, it can save that as a skill and reuse it later.
This is a big deal.
Most memory systems focus only on semantic recall: names, preferences, facts, and summaries. But agents also need to remember how to do things, not just what happened.
Hermes handles that by separating procedural knowledge from prompt memory:
`MEMORY.md` / `USER.md` for compact, durable facts
`session_search` for episodic recall
`skills` for reusable workflows
There is also a nice token-efficiency trick here. Hermes does not blindly inject every skill into the prompt. It injects a compact skills index and only loads full skill content when needed.
That keeps procedural memory available without paying the full token cost on every turn.
Layer 5: Honcho for Deeper User Modeling
Then there is the optional Honcho layer.
If local memory is Hermes's curated notebook, Honcho is its attempt at a richer user model.
Honcho runs alongside the built-in memory system in `hybrid` mode by default. It adds:
Cross-session user modeling
Cross-machine and cross-platform continuity
Semantic search over user context
Dialectic, LLM-generated answers about the user or the AI peer
The interesting part is how Hermes integrates it without wrecking prompt caching.
First turn vs later turns
On the first turn of a session, prefetched Honcho context can be baked into the cached system prompt.
On later turns, Hermes avoids mutating that stable system prompt. Instead, it attaches Honcho recall to the current user's turn at API-call time only. That means:
The stable prefix stays stable
Prompt caching still works
Turn N can consume context prefetched in the background after turn N-1
This is a very smart compromise.
Honcho itself also models two peers:
The user
The AI assistant
So Hermes is not just trying to remember you. It can also build a representation of itself over time.
That is both cool and slightly wild.
How Hermes Differs from OpenClaw
Since I wrote about OpenClaw recently, this comparison is worth making explicit.
OpenClaw
Memory is much closer to Markdown-first storage
Daily logs and long-term memory files act as the primary source of truth
Memory recall leans on hybrid search over stored notes
Hermes
Prompt memory is aggressively bounded
Session history lives in SQLite, not in prompt memory files
Past work is recalled through `session_search`
Procedural memory is pushed into skills
Deeper user modeling is optionally delegated to Honcho
The key insight here is that Hermes is more cache-aware than OpenClaw.
OpenClaw leans harder into "memory as searchable stored knowledge." Hermes leans harder into "memory as a hot working set plus cold retrieval layers."
I actually think this is the right direction for production agents.
Not everything deserves to live in the system prompt.
What Hermes Gets Right
After going through the repo and docs, I think Hermes gets three big things right.
1. It separates hot memory from cold recall
This is the core architectural win.
Small prompt memory for what always matters. Search for what only sometimes matters.
2. It treats prompt stability as a first-class constraint
A lot of agent systems talk about memory without talking about caching. Hermes clearly cares about both.
Frozen snapshots, delayed prompt updates, turn-level Honcho injection, and compressed-session rebuilds all point to the same design principle: don't casually mutate your prompt if you want good latency and cost.
3. It acknowledges that memory is plural
Hermes does not pretend that one store solves everything.
It has:
Semantic profile memory
Episodic session recall
Procedural memory through skills
Optional higher-order user modeling through Honcho
That is a much more realistic view of what agents actually need.
Conclusion
Hermes's memory system is not a giant knowledge base, and it is not a glorified vector store. It is a layered continuity architecture.
At the center is a tiny curated prompt memory: `MEMORY.md` and `USER.md`. Around that sits a searchable SQLite history for episodic recall. Beyond that sits a skills system for procedural reuse. And if you enable Honcho, Hermes adds a deeper user model on top of everything else.
The design principle underneath all of it is what impressed me the most: memory should help the agent stay useful without destroying prompt stability.
That is the real trick.
Not remembering more. Remembering the right things, in the right layer, at the right cost.
References
Hermes Agent GitHub Repository
Hermes Persistent Memory Docs
Hermes Prompt Assembly Docs
Hermes Session Storage Docs
Hermes Skills Docs
Hermes Honcho Docs
