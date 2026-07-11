# Mixedbread API Analysis — Search Platform

> **Base URL:** `https://api.mixedbread.com` | **Docs:** `https://www.mixedbread.com/docs` | **Auth:** Bearer token (`MIXEDBREAD_API_KEY` / `MXBAI_API_KEY`)
> **Description:** Upload PDFs, images, documents, code, audio, or video in any format and instantly make them searchable with natural-language queries. No document parsing, no embedding models to manage, no vector databases to set up. Multimodal semantic search across 100+ languages, optimized for AI/agent applications.

---

## Table of Contents

1. [Platform Overview & Core Concepts](#1-platform-overview--core-concepts)
2. [Authentication & API Keys](#2-authentication--api-keys)
3. [Stores — Search Index Management](#3-stores--search-index-management)
4. [Store Files — Document Ingestion & Lifecycle](#4-store-files--document-ingestion--lifecycle)
5. [Multipart Uploads — Large File Ingestion](#5-multipart-uploads--large-file-ingestion)
6. [Files — Raw File Storage (Non-Search)](#6-files--raw-file-storage-non-search)
7. [Search — Semantic Retrieval](#7-search--semantic-retrieval)
8. [Reranking — Second-Stage Ranking](#8-reranking--second-stage-ranking)
9. [Agentic Search — Multi-Round Retrieval](#9-agentic-search--multi-round-retrieval)
10. [Question Answering — RAG Generation](#10-question-answering--rag-generation)
11. [Grep — Regex Pattern Matching](#11-grep--regex-pattern-matching)
12. [List Chunks — Metadata-Only Retrieval](#12-list-chunks--metadata-only-retrieval)
13. [Metadata Filtering & Facets](#13-metadata-filtering--facets)
14. [Generated Metadata — Auto-Extracted Structure](#14-generated-metadata--auto-extracted-structure)
15. [MXJSON — Pre-Chunked Ingestion Format](#15-mxjson--pre-chunked-ingestion-format)
16. [Web Store — Hybrid Web + Internal Search](#16-web-store--hybrid-web--internal-search)
17. [Cross-Cutting: Pagination, Errors, Rate Limits](#17-cross-cutting-pagination-errors-rate-limits)
18. [SDKs, CLI & Integrations](#18-sdks-cli--integrations)
19. [Production: Bring Your Own Bucket](#19-production-bring-your-own-bucket)
20. [Capability Summary & Cross-Reference](#20-capability-summary--cross-reference)

---

## 1. Platform Overview & Core Concepts

### Main Concepts

Mixedbread is an **AI-native multimodal search API** built around a managed retrieval pipeline. The platform exposes three core objects and two read operations:

- **Store** — A named, searchable container (search index) holding your files. Every search runs against one or more Stores. Identified by UUID **or** a unique human-readable name (`[a-z0-9.-]`).
- **Store File** — A file uploaded into a Store. Tracks original filename, user metadata, processing status, version, and usage. Mixedbread parses, chunks, and embeds it automatically.
- **Chunk** — A searchable segment produced from a Store File. Every search/list response is a ranked list of chunks. Chunks inherit file metadata and carry auto-generated metadata.
- **Search** — Natural-language semantic query returning top-k ranked chunks.
- **Agentic Search** — Multi-round, agent-driven retrieval that decomposes complex questions into sub-queries and merges/reranks results.

### End-to-End Flow

1. Create a **Store** (search index).
2. Upload files → become **Store Files** → Mixedbread parses (OCR, layout understanding, transcription), chunks, and embeds them.
3. Once a file reaches `completed`, its chunks are searchable.
4. Query with **Search** / **Agentic Search** / **Question Answering** → receive ranked chunks.

### Key Differentiators

- **Multimodal by default** — text, images, tables, audio, video, code in a single index.
- **Multilingual** — 100+ languages, cross-lingual query → content matching without translation.
- **Zero infrastructure** — no servers, models, or vector DBs to manage; scales automatically.
- **AI-native ranking** — optimized to feed verifiable context to LLMs/agents, reducing hallucinations.
- **Default embedding model:** `mixedbread-ai/mxbai-wholembed-v3` (co-designed with the retrieval infra; powers all new stores by default). Audio/video require this encoder.

---

## 2. Authentication & API Keys

### Main Concepts

- **Bearer Token Auth** — All endpoints require `Authorization: Bearer YOUR_API_KEY`.
- **API Key Management** — Keys created in the [Mixedbread platform](https://platform.mixedbread.com); stored in `.env`.
- **SDK env var** — `MIXEDBREAD_API_KEY` (Python) / `MXBAI_API_KEY` (Vercel integration).

### Analysis

A single, flat API-key model (no documented workspace scoping or key roles). Keys are org-level credentials. The Vercel integration auto-injects `MXBAI_API_KEY` and `MXBAI_STORE_ID` into linked projects.

---

## 3. Stores — Search Index Management

### Main Concepts

- **Store = Search Index** — Primary container for searchable content; manages access permissions and file relationships.
- **Dual Identifiers** — Every Store has a UUID `id` and an optional unique `name`; either works in all API calls.
- **Access Modes** — `private` (default, org-only, owner pays all costs) vs `public` (anyone with a valid API key can search; searcher pays for their own searches).
- **Expiration Policy** — Optional `expires_after` (e.g. `{seconds: 86400}`) auto-expires stores; `expires_at` reflects computed expiry.
- **Usage Tracking** — `usage_bytes`, `usage_tokens`, `file_counts` per status state.
- **Configuration** — `config.contextualization` (`with_metadata`, `with_file_context`) and `config.save_content`.

### Endpoints & Parameters

| Endpoint | Method | Purpose | Key Parameters |
|---|---|---|---|
| `/v1/stores` | `POST` | Create a store | `name`, `description`, `is_public`, `license`, `expires_after`, `metadata`, `file_ids`, `config` |
| `/v1/stores/{store_identifier}` | `GET` | Retrieve store (by ID or name) | — |
| `/v1/stores` | `GET` | List stores (paginated, searchable) | Pagination (`limit`, `after`/`before`), search |
| `/v1/stores/{store_identifier}` | `PUT` | Update store config | `name`, `description`, `is_public`, `license`, `expires_after`, `metadata` |
| `/v1/stores/{store_identifier}` | `DELETE` | Permanently delete store + all contents | — |

### Store Response Model

| Field | Type | Description |
|---|---|---|
| `id` | string | Unique identifier (`vs_...`) |
| `object` | string | `"store"` |
| `name` | string | Unique name (`[a-z0-9.-]` only) |
| `description` | string | Purpose/contents |
| `is_public` | boolean | Public read access |
| `license` | string | License for public stores |
| `metadata` | object | Key-value metadata |
| `config` | object | `contextualization{with_metadata, with_file_context}`, `save_content` |
| `file_counts` | object | Counts per status: `pending`, `in_progress`, `completed`, `failed`, `cancelled`, `total` |
| `status` | string | `expired` / `in_progress` / `completed` |
| `usage_bytes` / `usage_tokens` | integer | Storage usage |
| `expires_after` / `expires_at` | object / datetime | Expiration policy / computed expiry |
| `created_at` / `updated_at` / `last_active_at` | datetime | Timestamps |

### Analysis

Stores are the top-level scoping and billing unit. The dual-identifier design (ID or name) simplifies SDK ergonomics. Public stores enable shared knowledge bases with cost-transfer to the searcher — useful for documentation or community corpora. Expiration policies support ephemeral/dev stores. `contextualization` flags let the platform enrich chunk context at index time using file metadata and surrounding file context, improving retrieval for sparse or ambiguous chunks.

---

## 4. Store Files — Document Ingestion & Lifecycle

### Main Concepts

- **Universal Ingestion** — PDFs, images (jpg/png/webp/avif), Office docs, code files, audio (mp3/wav/ogg/m4a/aac/flac), video (mp4/webm/mov/avi/ogv), MXJSON/MXJSONL pre-chunked.
- **Automatic Processing** — Parsing (OCR for scans, layout understanding for PDFs, transcription for audio/video) → chunking → embedding → indexing.
- **File Status Lifecycle** — `pending` → `in_progress` → `completed` (or `failed` / `cancelled`). `last_error` holds failure detail.
- **Metadata Propagation** — User-defined file metadata propagates to every chunk derived from that file (filter at file-level, applied to chunk search).
- **External ID** — Optional `external_id` for idempotent/mapped uploads; supports slashes for path-like identifiers.
- **Parsing Strategy** — Selectable (e.g. `fast`).
- **Versioning** — `version` increments on updates.
- **Polling** — `upload_and_poll` / `uploadAndPoll` blocks until `completed`; alternatively poll `retrieve`.

### Supported File Types (highlights)

| Category | Formats |
|---|---|
| Documents | PDF, Word, PowerPoint, Excel, OpenDocument, HTML, Markdown, TXT, CSV |
| Images | JPEG, PNG, WebP, AVIF |
| Code | `.py .js .ts .java .cs .c .h .cpp .go .html .rb .rs` (+more) |
| Audio | MP3, WAV, OGG, M4A, WebM Audio, AAC, FLAC *(requires `mxbai-wholembed-v3`)* |
| Video | MP4, WebM, QuickTime, AVI, OGG Video *(requires `mxbai-wholembed-v3`)* |
| Specialized | `.mxjson`, `.mxjsonl` (pre-chunked content; see §15) |

### Endpoints & Parameters

| Endpoint | Method | Purpose | Key Parameters |
|---|---|---|---|
| `/v1/stores/{store_identifier}/files` | `POST` | Upload file to store | `file`, `metadata`, `external_id`, `parsing_strategy`, `multipart_upload` (SDK option) |
| `/v1/stores/{store_identifier}/files/{file_identifier}` | `GET` | Retrieve store file | `return_chunks` (bool or list of indices) |
| `/v1/stores/{store_identifier}/files` | `GET` | List store files (paginated) | `limit`, `after`/`before`, `statuses[]`, `metadata_filter` |
| `/v1/stores/{store_identifier}/files/{file_identifier}` | `PATCH`/`POST` | Update file metadata | `metadata` |
| `/v1/stores/{store_identifier}/files/{file_identifier}` | `DELETE` | Delete file + chunks | — |

### Store File Response Model

| Field | Type | Description |
|---|---|---|
| `id` | string | Unique file ID |
| `external_id` | string | Optional external identifier |
| `object` | string | `"store.file"` |
| `filename` | string | Name with extension |
| `metadata` | object | User-defined metadata (propagates to chunks) |
| `status` | string | `pending` / `in_progress` / `cancelled` / `completed` / `failed` |
| `last_error` | string\|null | Error message if failed |
| `store_id` | string | Containing store |
| `usage_bytes` / `usage_tokens` | integer | Storage usage |
| `config` | object | e.g. `parsing_strategy` |
| `version` | integer | File version (≥1) |
| `created_at` | datetime | Creation timestamp |
| `content_url` | string | Presigned URL for file content |
| `chunks` | array | Returned when `return_chunks` set |

### Analysis

Store Files abstract the entire ingestion pipeline. The `upload_and_poll` helper collapses the async wait into a single SDK call — important because files are not searchable until `completed`. Metadata propagation is a notable design choice: attaching metadata once at the file level automatically filters all derived chunks, avoiding per-chunk tagging. The `return_chunks` query param lets you inspect the parsed/chunked output of a file (optionally specific indices), useful for debugging ingestion and validating chunk boundaries. `external_id` with slash support enables path-like naming for filesystem-mirrored corpora.

---

## 5. Multipart Uploads — Large File Ingestion

### Main Concepts

- **Presigned-URL Multipart** — Create an upload session, receive presigned part URLs, upload parts directly to object storage, then complete (or abort).
- **SDK-Managed (Automatic)** — `upload` / `upload_and_poll` accept `multipart_upload` options and auto-split files above a threshold.
- **Manual Flow** — `create` → upload to each `part_urls[].url` → `complete` (or `abort`).

### Endpoints & Parameters

| Endpoint | Method | Purpose | Key Parameters |
|---|---|---|---|
| `/v1/files/uploads` | `POST` | Create multipart upload | `filename`, `file_size`, `mime_type`, `part_count` (1–10000, default 1) |
| `/v1/files/uploads/{upload_id}` | `GET` | Get upload status + fresh presigned URLs for pending parts | — |
| `/v1/files/uploads` | `GET` | List in-progress multipart uploads | Pagination |
| `/v1/files/uploads/{upload_id}/abort` | `POST` | Abort + cleanup uploaded parts | — |
| *(complete)* | `POST` | Complete multipart upload after all parts uploaded | Part numbers + ETags |

### SDK `MultipartUploadOptions`

| Option | Default | Description |
|---|---|---|
| `threshold` | 100 MB | Min file size to trigger multipart |
| `part_size` / `partSize` | 100 MB | Size of each upload chunk |
| `concurrency` | 5 | Parallel part uploads |
| `on_part_upload` / `onPartUpload` | — | Callback after each part (`PartUploadEvent`: `part_number`, `total_parts`, `part_size`, `uploaded_bytes`, `total_bytes`) |

### Create Multipart Response

```json
{
  "id": "a1b2c3d4-e5f6-7890-abcd-ef1234567890",
  "part_urls": [
    {"part_number": 1, "url": "https://storage.../upload?partNumber=1&uploadId=..."},
    {"part_number": 2, "url": "https://storage.../upload?partNumber=2&uploadId=..."}
  ]
}
```

### Analysis

The dual-mode design (SDK-automatic vs. manual presigned URLs) serves both casual and high-performance users. The manual flow lets clients upload parts directly to object storage (bypassing Mixedbread's API for the heavy bytes), reducing API throughput and enabling resumable uploads — `get` returns fresh URLs for any parts not yet uploaded. The `PartUploadEvent` callback enables progress UIs. The 10,000-part ceiling with default 100 MB parts supports files up to ~1 TB.

---

## 6. Files — Raw File Storage (Non-Search)

### Main Concepts

A separate **Files** API (`/v1/files`) provides raw file storage **independent of Stores/search** — store file content without indexing. These files are not parsed, chunked, or embedded.

### Endpoints & Parameters

| Endpoint | Method | Purpose | Key Parameters |
|---|---|---|---|
| `/v1/files` | `POST` | Upload a raw file | `file` (binary) |
| `/v1/files/{file_id}` | `GET` | Retrieve file metadata | — |
| `/v1/files` | `GET` | List all files (paginated) | `limit`, pagination |
| `/v1/files/{file_id}` | `POST` | Update/replace file content | `file` (binary) |
| `/v1/files/{file_id}/content` | `GET` | Download file content (streaming) | — |
| `/v1/files/{file_id}` | `DELETE` | Delete a file | — |

### File Response Model

| Field | Type | Description |
|---|---|---|
| `id` | string | Unique file ID |
| `filename` | string | Name with extension |
| `bytes` | integer | Size in bytes |
| `mime_type` | string | MIME type |
| `version` | integer | Version (≥1, increments on update) |
| `created_at` / `updated_at` | datetime | Timestamps |
| `object` | string | `"file"` |

### Analysis

The standalone Files API decouples raw storage from search ingestion. This is useful for keeping source-of-truth copies (e.g. contracts, media assets) separately from the searchable index, or for a two-stage workflow where files are uploaded raw first then selectively added to Stores. The `update` endpoint creates a new version rather than mutating in place, preserving audit history. Content download streams via `iter_bytes()`.

---

## 7. Search — Semantic Retrieval

### Main Concepts

- **Natural-Language Query** — Find content by meaning, not keywords; multimodal (text, images, tables) matched semantically.
- **Ranked Chunks** — Returns top-k chunks scored by relevance (`score` field).
- **Extensions** (via `search_options`): metadata filters, file-ID filters, multi-store search, rerank, query rewriting, agentic search, web search.
- **Same Response Shape** across Search, Agentic Search, and Web Store — drop-in upgrades.

### Endpoint & Parameters

**`POST /v1/stores/search`**

| Parameter | Type | Required | Default | Description |
|---|---|---|---|---|
| `query` | string | yes* | — | Search query text (*or extracted from `messages` for QA) |
| `store_identifiers` | array | yes | — | Store IDs/names; include `mixedbread/web` for web search |
| `top_k` | integer | no | 10 | Number of results to return (min 1) |
| `file_ids` | array\|tuple | no | — | Filter by file IDs: inclusion list or `["not_in", [...]]` |
| `filters` | object | no | — | Filter conditions (see §13) |
| `search_options` | object | no | — | `rerank`, `rewrite_query`, `agentic`, `score_threshold`, `return_metadata`, `media_content` |

### `search_options` Sub-Fields

| Option | Type | Description |
|---|---|---|
| `rerank` | bool\|object | Enable/configure second-stage reranking (see §8) |
| `rewrite_query` | bool | Let a model rewrite the query into a more search-friendly form |
| `agentic` | bool\|object | Enable multi-round agentic search (see §9); **overrides** `rerank` and `rewrite_query` |
| `score_threshold` | number | Minimum relevance score; filters final results |
| `return_metadata` | bool | Include file metadata in results |
| `media_content` | string | `"auto"` / `"always"` / `"never"` — whether to return media (image/audio/video) content |

### Search Response Model

```json
{
  "object": "list",
  "data": [
    {
      "chunk_index": 0,
      "mime_type": "text/plain",
      "model": "mxbai-omni",
      "score": 0.8512,
      "file_id": "...",
      "filename": "auth_guide.pdf",
      "store_id": "...",
      "metadata": {"source": "upload", "page": 1},
      "type": "text",
      "text": "Authentication is handled through JWT tokens..."
    },
    {
      "chunk_index": 2,
      "mime_type": "image/jpeg",
      "model": "mxbai-omni",
      "score": 0.8234,
      "file_id": "...",
      "filename": "security_diagram.pdf",
      "store_id": "...",
      "metadata": {"category": "security", "year": 2024},
      "type": "image_url",
      "image_url": {"url": "..."}
    }
  ]
}
```

### Chunk Properties

| Property | Type | Description |
|---|---|---|
| `chunk_index` | integer | Position within source file |
| `mime_type` | string | Content type (`text/plain`, `image/png`, ...) |
| `model` | string | Model that generated the chunk's vector (e.g. `mxbai-omni`) |
| `score` | number | Relevance score (in search results) |
| `file_id` | string | Source file ID |
| `filename` | string | Source file name |
| `store_id` | string | Containing store ID |
| `external_id` | string | Optional external ID of source file |
| `metadata` | object | User-defined metadata inherited from file |
| `generated_metadata` | object | Auto-generated ingestion metadata (see §14) |
| `type` | string | `text` / `image_url` / `audio_url` / `video_url` |
| `text` / `image_url` / `ocr_text` | varies | The content payload (depends on `type`) |

### Analysis

Search is the platform's centerpiece. The unified chunk response model means a single query can return text excerpts, images, and media references ranked together — true multimodal retrieval. The `search_options` bag is the control plane for quality/latency tradeoffs: `rewrite_query` (pre-retrieval optimization), `rerank` (post-retrieval precision), `agentic` (multi-round for hard queries), and `score_threshold` (precision guard). Multi-store search merges and re-ranks across stores in one call, enabling federated knowledge bases. The `media_content` flag lets callers avoid large media payloads when only text is needed.

---

## 8. Reranking — Second-Stage Ranking

### Main Concepts

- **Two-Stage Retrieval** — First-stage semantic search returns candidates; a specialized reranker re-evaluates and reorders them for sharper relevance.
- **Boolean or Object Config** — `rerank: true` (defaults) or `rerank: {model, top_k, with_metadata}`.
- **Metadata-Aware Reranking** — `with_metadata` lets the reranker consider specified metadata fields when scoring.
- **Instruction-Steerable (v3 listwise)** — The listwise model follows natural-language instructions (e.g. "prefer recent docs", "prioritize primary sources").

### Configuration Options

| Option | Type | Description |
|---|---|---|
| `model` | string | Reranker model ID |
| `top_k` | integer | Number of chunks to rerank (defaults to search `top_k`); set higher than search top_k to re-rank a wider candidate pool |
| `with_metadata` | array | Metadata fields to include in reranking context |

### Available Models

| Model | Type | Notes |
|---|---|---|
| `mixedbread-ai/mxbai-rerank-large-v2` | Pointwise | **Default.** 1.5B-param model, 100+ languages |
| `mixedbread-ai/mxbai-rerank-v3-listwise` | Listwise | Reads candidate set as a whole; follows NL instructions (prefer recent, primary sources). Preview. Improves all 56 Vidore v3 runs (+11% NDCG@10 avg). |

### Analysis

Reranking is the primary lever for retrieval quality. The pointwise default is robust and multilingual; the listwise v3 preview is notable for **instruction-following** — callers can inject ranking policies as natural language, turning the reranker into a policy-aware ranker without custom training. `with_metadata` enables signals like recency or authority to influence rank. Because reranking adds latency, it's opt-in per query, letting developers pay for precision only when needed. Web Store results are always reranked regardless of the flag.

---

## 9. Agentic Search — Multi-Round Retrieval

### Main Concepts

- **Agent-Driven Loop** — For complex, multi-part, or ambiguous queries, an agent decomposes the question, runs multiple sub-queries, evaluates candidates, and iterates.
- **Drop-in Upgrade** — Same response shape as normal Search (ranked chunk list).
- **Self-Managed Ranking** — When `agentic` is enabled, `rewrite_query` and `rerank` are **ignored** (the agent owns decomposition + ranking).
- **Steerable** — `instructions` encode domain knowledge the agent can't infer (entity disambiguation, preferred source types).
- **Hybrid Capable** — Include `mixedbread/web` to let the agent pull web results alongside internal stores.

### Configuration (`search_options.agentic`)

| Parameter | Default | Description |
|---|---|---|
| `max_rounds` | 3 | Max agent iterations |
| `queries_per_round` | 4 | Sub-queries generated per round |
| `instructions` | — | Natural-language steering directives |
| `media_content` | `"auto"` | Whether to return media content |
| `strict_top_k` | false | If true, enforce exact result size (drop irrelevant) |

### Parameter Interactions

- `top_k`: target result size; exact when `strict_top_k=true`.
- `filters` / `file_ids`: applied to each round's local retrievals.
- `store_identifiers`: all listed stores searched; `mixedbread/web` enables web rounds.
- `search_options.score_threshold`: final post-rank filter; can return fewer than `top_k` even with `strict_top_k=true`.

### Observability & Tips

- Inspect the agent **trace** to see rounds/sub-queries before tuning.
- If it converges in 2 rounds, lower `max_rounds` to save time.
- If it always burns the budget, raise `queries_per_round`.
- Defaults (`max_rounds=3`, `queries_per_round=4`, no instructions) are a strong baseline.
- Slower and more expensive than single search — use when single search falls short.

### Analysis

Agentic Search addresses the hardest retrieval cases: multi-hop questions ("numbers for 2020–2025?"), entity-heavy queries, and cases requiring query decomposition. By absorbing `rewrite_query` and `rerank` into the agent loop, it reduces configuration surface for complex needs. The `strict_top_k` + `score_threshold` combination gives precise control over output size vs. precision. The trace observability is critical for debugging cost/quality. This is the most "agentic" capability in the platform and positions Mixedbread for LLM-agent retrieval workflows where a single embedding lookup is insufficient.

---

## 10. Question Answering — RAG Generation

### Main Concepts

- **Answer Generation** — Instead of returning raw chunks, generates a complete answer grounded in retrieved store content.
- **Citations** — Optional `cite: true` inserts `<cite i="n"/>` markers in the answer, mapping to `sources[n]` chunks.
- **Multimodal** — Optional `multimodal: true` includes image/OCR content in the answer context.
- **Streaming** — Optional streaming of the generated answer.
- **Instructions** — Steer answer style/audience (e.g. "Answer for an API evaluator. Use concise bullets...").
- **Messages Support** — If `query` not provided, the question is extracted from passed `messages`.

### Endpoint & Parameters

**`POST /v1/stores/question-answering`**

| Parameter | Type | Required | Default | Description |
|---|---|---|---|---|
| `query` | string | no* | — | Question (*or extracted from `messages`*) |
| `store_identifiers` | array | yes | — | Stores to search |
| `top_k` | integer | no | 10 | Chunks to use as context |
| `filters` | object | no | — | Filter conditions |
| `file_ids` | array | no | — | Inclusion file-ID filter |
| `search_options` | object | no | — | Same options as Search (rerank, etc.) |
| `instructions` | string | no | — | Answer-style steering |
| `qa_options` | object | no | — | `cite` (bool), `multimodal` (bool), `stream` (bool) |

### QA Response Model

```json
{
  "answer": "Based on the provided source, the key decision involved finalizing the budget for Project Alpha <cite i=\"0\"/>.",
  "sources": [
    {
      "chunk_index": 2,
      "mime_type": "text/plain",
      "model": "mxbai-omni",
      "score": 0.92,
      "file_id": "...",
      "filename": "meeting_notes_q1.docx",
      "store_id": "...",
      "metadata": {},
      "type": "text",
      "text": "In the Q1 board meeting, we agreed to finalize the budget..."
    }
  ]
}
```

### Analysis

Question Answering is Mixedbread's built-in RAG — it collapses retrieve-then-generate into one API call. The citation mechanism (`<cite i="n"/>` → `sources[n]`) is designed for verifiable, hallucination-resistant answers, letting downstream UIs render inline source links. `multimodal` extends grounding to images/OCR, useful for diagram-heavy or scanned docs. The `messages` fallback enables conversational QA where the question is the latest user message. Streaming support is essential for chat UX. Reusing `search_options` means QA inherits all retrieval-quality knobs (rerank, agentic, filters).

---

## 11. Grep — Regex Pattern Matching

### Main Concepts

- **Literal Regex Search** — Unlike semantic Search, Grep runs an **RE2 regular expression** against the literal text of each chunk.
- **Use Cases** — Find specific tokens, identifiers, error codes, or exact phrases (e.g. `ERR-\d{4}`).
- **Content Groups** — Choose what to match: `text` (original chunk text) and/or `generated` (ingestion-derived fields: transcription, OCR text, summaries).
- **Case Sensitivity** — Optional.

### Endpoint & Parameters

**`POST /v1/stores/grep`**

| Parameter | Type | Required | Default | Description |
|---|---|---|---|---|
| `store_identifiers` | array | yes | — | Stores to search |
| `pattern` | string (RE2) | yes | — | Regex matched against chunk text |
| `top_k` | integer | no | 10 | Number of results |
| `content_groups` | array | no | — | `["text"]`, `["generated"]`, or both |
| `case_sensitive` | boolean | no | false | Case-sensitive matching |
| `file_ids` | array\|tuple | no | — | Inclusion/`not_in` file filter |
| `filters` | object | no | — | Filter conditions |
| `return_metadata` | boolean | no | true | Include file metadata |

### Grep Response

Same chunk list shape as Search (`object: "list"`, `data: [chunks]`); matches have `score: 1`.

### Analysis

Grep complements semantic search with exact-match capability — critical for structured identifiers, error codes, log analysis, or compliance keyword scanning that embeddings cannot reliably locate. The `content_groups` distinction is thoughtful: searching `generated` lets you regex-match OCR/transcription output that may not be in the primary `text`. RE2 syntax guarantees linear-time evaluation (safe on untrusted patterns). This makes Mixedbread a hybrid semantic + lexical search platform.

---

## 12. List Chunks — Metadata-Only Retrieval

### Main Concepts

- **No Embeddings, No Similarity, No Reranking** — `list_chunks` retrieves chunks purely by metadata filters and sorting.
- **Ranked Retrieval Over Metadata** — Useful for browsing/exports ordered by numeric metadata attributes (dates, priorities, counts).
- **No Pagination** — Returns up to `top_k` chunks directly.

### Endpoint & Parameters

**`POST /v1/stores/list-chunks`**

| Parameter | Type | Required | Default | Description |
|---|---|---|---|---|
| `store_identifiers` | array | yes | — | Stores to list from |
| `top_k` | integer | no | 10 | Chunks to return (min 1; no pagination) |
| `file_ids` | array\|tuple | no | — | Inclusion/`not_in` file filter |
| `sort_by` | string\|array | no | — | Metadata field path, or `(field_path, ascending)` tuple. Unprefixed dot paths → file metadata; `generated_metadata.` prefix → chunk metadata |
| `filters` | object | no | — | Filter conditions |
| `return_metadata` | boolean | no | — | Include file metadata |

### Analysis

List Chunks fills the gap between semantic search and raw enumeration. It's the tool for deterministic, ordered access — e.g. "all chunks from file X sorted by page number" or "top 50 chunks by priority metadata". The `sort_by` tuple syntax `(field, ascending)` and the prefix convention (bare path = file metadata, `generated_metadata.` = chunk metadata) mirror the filtering conventions. The lack of pagination (capped by `top_k`) suggests it's intended for bounded, filter-driven extracts rather than full corpus dumps.

---

## 13. Metadata Filtering & Facets

### Main Concepts

- **Two Filter Shapes** — Single-field (direct condition) or multi-field (logical operators `all`/`any`/`none`).
- **Three Metadata Layers** — User file metadata (bare keys), generated chunk metadata (`generated_metadata.` prefix), and system fields (`file_id`, `chunk_index`).
- **Dot Notation** — Drill into nested objects (e.g. `generated_metadata.chunk_headings.level`, `city.name`).
- **Type-Aware Operators** — Equality/pattern for strings, comparisons for numbers/dates, boolean conditions, array containment.
- **Facets** — Aggregate chunk counts grouped by metadata values (single or multiple fields); supports nested fields and query-time faceting.

### Filter Condition Structure

**Direct (single field):**
```json
{"key": "category", "operator": "eq", "value": "documentation"}
```

**Logical (multi-field):**
```json
{
  "all": [
    {"key": "status", "value": "published", "operator": "eq"},
    {"any": [
      {"key": "priority", "value": 3, "operator": "gte"},
      {"all": [
        {"key": "category", "value": "important", "operator": "eq"},
        {"key": "reviewed", "value": true, "operator": "eq"}
      ]}
    ]}
  ]
}
```

### Logical Operators

| Operator | Semantics |
|---|---|
| `all` | AND — all conditions true |
| `any` | OR — at least one true |
| `none` | NOT — none true |

### Comparison Operators

| Operator | Applies To | Description |
|---|---|---|
| `eq` / `ne` | string, number, boolean, date | Equality / inequality |
| `gt` / `gte` / `lt` / `lte` | number, date | Comparisons |
| `in` / `not_in` | arrays, `file_ids` | Membership |
| `like` / `not_like` | string | Pattern match |

### Supported Metadata Value Types

| Type | Best For | Filter Operators |
|---|---|---|
| String | Categories, names, status | `eq`, `ne`, `like`, `not_like` |
| Numeric | Scores, versions, counts | `>`, `<`, `>=`, `<=`, `eq` |
| Boolean | Flags, toggles | True/false conditions |
| Date/Time (ISO 8601) | Timestamps, deadlines | `>`, `<`, `>=`, `<=` |
| Array/List | Tags, authors, categories | Containment |

### Facets Endpoint & Usage

`mxbai.stores.metadata_facets(store_identifiers, query?, top_k?, filters?, facets?)`

- Without `facets` param: aggregates counts across **all** metadata fields.
- With `facets=["author"]`: scopes to specific field(s) (recommended for high-cardinality fields).
- Aggregates across all specified stores (unified view).
- Supports query-time faceting (pass search params to facet the filtered result set).
- Nested fields via dot notation.

**Facet Response:**
```json
{
  "facets": {
    "author": {"John Doe": 1, "Jane Doe": 2},
    "language": {"english": 2, "spanish": 1}
  }
}
```

### Analysis

The filtering system is a structured query language over three metadata layers, enabling precise pre-ranking control of both file listing and search. The `generated_metadata.` prefix is the key to filtering on auto-extracted structure (language, page numbers, media duration, code line ranges) — turning ingestion byproducts into first-class query dimensions. The `none` operator and arbitrary nesting give expressive power (e.g. "exclude chunks where any tag is deprecated"). Facets enable guided-search UIs (drill-down by author/category/date) and corpus analytics. The recommendation to scope facets to specific fields for high-cardinality keys indicates performance-aware design.

---

## 14. Generated Metadata — Auto-Extracted Structure

### Main Concepts

Mixedbread automatically generates typed metadata for each ingested file/chunk, exposed via the `generated_metadata` field. The object is discriminated by `type`; if absent, inferred from `file_type` (MIME), defaulting to `text`. Every object includes `file_extension`.

### Supported `type` Values

`markdown`, `text`, `pdf`, `code`, `audio`, `video`, `image`

### Per-Type Fields

| Type | Key Fields |
|---|---|
| **text** / **markdown** | `language`, `word_count`, `file_size`, `chunk_size` |
| **pdf** | `total_pages`, `total_size`, page-level chunk info |
| **code** | `language` (detected), `word_count`, `file_size`, `start_line`, `num_lines` |
| **image** | `file_type`, `file_size`, `width`, `height` |
| **audio** | `total_duration_seconds`, `sample_rate`, `channels`, `audio_format`, `bpm` (optional), `start_time_seconds`, `end_time_seconds`, `duration_seconds`, `chunk_size_bytes` |
| **video** | `total_duration_seconds`, `fps`, `width`, `height`, `frame_count`, `has_audio_stream`, `bpm` (optional), `start_time_seconds`, `end_time_seconds`, `duration_seconds`, `chunk_size_bytes` |

### Type Inference Rules

- If `type` present → use it.
- Else infer from `file_type` (MIME) when recognized.
- Else default to `"text"`.
- Unknown extra fields may appear; clients should ignore them.

### Analysis

Generated metadata transforms Mixedbread from a black-box embedder into a structured content platform. The per-type schema means you can filter/rank by media-specific dimensions: `total_pages` for PDFs, `start_line`/`num_lines` for code, `duration_seconds`/`bpm` for audio/video. This enables queries like "code chunks in Python after line 100" or "video segments under 30 seconds with audio". The temporal fields (`start_time_seconds`, `end_time_seconds`) for audio/video effectively make the platform a media-clip retrieval system. The defensive "ignore unknown fields" guidance indicates the schema is extensible/forward-compatible.

---

## 15. MXJSON — Pre-Chunked Ingestion Format

### Main Concepts

- **Bypass Parsing** — `.mxjson` / `.mxjsonl` let you ingest **pre-chunked** content directly, skipping Mixedbread's parsing/chunking.
- **Use Cases** — Custom chunking logic, preserving specific chunk boundaries, including pre-computed metadata (OCR text, transcriptions).
- **Chunk Types** — `text`, `image_url`, `audio_url`, `video_url`.

### Chunk Object Schema

```json
{
  "type": "text",
  "text": "Autolyse is a rest period after mixing flour and water.",
  "mime_type": "text/plain",
  "chunk_index": 0,
  "generated_metadata": {
    "type": "text",
    "file_type": "text/plain",
    "language": "en",
    "word_count": 10,
    "file_size": 2048
  }
}
```

- `text`: 1–65,536 characters.
- `image_url` / `audio_url` / `video_url`: `{ "url": "..." }` (URL or data URI).
- File-level metadata (set at upload) applies to all chunks and participates in contextualization if enabled.

### Validation Errors

| Error | Cause |
|---|---|
| `type` required | Missing `type` field |
| `text` must be 1–65536 chars | Empty or exceeds limit |
| Invalid URL format | Malformed URL/data URI |
| Unknown chunk type | Not one of `text`, `image_url`, `audio_url`, `video_url` |

### Analysis

MXJSON is the escape hatch for advanced users who already have a chunking pipeline (e.g. LangChain splitters, custom OCR, or pre-transcribed media). It turns Mixedbread into a pure embedding+indexing+search backend over caller-defined chunks, while still leveraging the platform's embedding models, metadata filtering, and search. This is critical for interoperability with existing data-processing stacks and for cases where preserving exact chunk boundaries matters (legal citations, code blocks). The 65,536-char text limit and data-URI support for media accommodate most chunk shapes.

---

## 16. Web Store — Hybrid Web + Internal Search

### Main Concepts

- **Virtual Store `mixedbread/web`** — Include `"mixedbread/web"` in `store_identifiers` to search the open web alongside your stores.
- **Unified Ranking** — Web + internal results are merged and reranked together for consistent relevance scoring.
- **Fresh Content** — Web results reflect current content at search time; metadata filters (`file_ids`) do not apply to web results.
- **Always Reranked** — Web search results are always reranked for optimal relevance.
- **Works with Agentic Search** — The agent can pull web results in its rounds.

### Web Result Shape

Web results follow the same chunk structure:
- `text`: page title + relevant excerpts
- `filename`: source URL
- `metadata.url`: source URL
- `metadata.title`: page title
- `metadata.source`: `"web_search"`
- `store_id`: fixed web-store ID

### Analysis

The Web Store elegantly extends the same Search/QA/Agentic APIs to web retrieval — no separate web-search endpoint to learn. By modeling the web as just another store identifier, Mixedbread enables hybrid RAG (internal knowledge + live web) in a single ranked list, which is increasingly important for agents that need current information. The "always reranked" rule and the exclusion of `file_ids` filtering on web results reflect that web content isn't pre-indexed. Usage counts toward standard rate limits.

---

## 17. Cross-Cutting: Pagination, Errors, Rate Limits

### Pagination

**Cursor-based**, supported on list endpoints (List Stores, List Store Files, List Files, List Multipart Uploads, List Store Events).

| Parameter | Type | Default | Description |
|---|---|---|---|
| `before` | string | — | Cursor: items before this cursor |
| `after` | string | — | Cursor: items after this cursor |
| `limit` | integer | 20 | Max items (1–100) |

- Use `before` **or** `after`, not both.
- Cursors are opaque — don't parse/modify.

**Response fields:** `data`, `first_cursor`, `last_cursor`, `has_more`, plus `pagination` object (`has_more`, `last_cursor`) in SDK.

**Navigation:** forward (no cursor → `after=last_cursor`) or backward (`before=first_cursor`).

### Error Handling

| Code | Meaning | Description |
|---|---|---|
| 200 | OK | Success |
| 201 | Created | Resource created (POST) |
| 400 | Bad Request | Malformed/missing params |
| 401 | Unauthorized | Invalid/expired/missing API key |
| 402 | Payment Required | Insufficient balance |
| 403 | Forbidden | Valid creds, insufficient permissions |
| 404 | Not Found | Resource doesn't exist |
| 409 | Conflict | State conflict |
| 422 | Unprocessable Entity | Correct format, unprocessable |
| 429 | Rate Limit Exceeded | Too many requests |

SDK typed errors: `AuthenticationError`, `RateLimitError`, etc.

### Rate Limiting

Per-operation-type token-bucket limits (1-minute window):

| Operation Type | Limit | Burst | Window |
|---|---|---|---|
| Read | 1,200 | 1,000 | 1 min |
| List | 600 | 200 | 1 min |
| Write | 360 | 120 | 1 min |
| Update | 480 | 160 | 1 min |
| Delete | 240 | 80 | 1 min |

- Exceeding → `429` with `Retry-After` header.
- Higher limits available on request.

### Analysis

The cross-cutting concerns are conventional and well-documented. Cursor pagination (vs. offset) is the right choice for mutable, append-heavy indexes. The operation-type-tiered rate limits are notable: read-heavy workloads (search/QA) get generous limits (1,200/min) while destructive operations (delete) are tightly throttled (240/min) — a safety-by-default design. The `402 Payment Required` status is a clear signal for usage/balance gating. SDK retries (default 3) plus typed errors make transient failures easy to handle.

---

## 18. SDKs, CLI & Integrations

### SDKs

| SDK | Install | Client | Config |
|---|---|---|---|
| **Python** | `pip install mixedbread` | `from mixedbread import Mixedbread; mxbai = Mixedbread(api_key=...)` | `api_key`, `max_retries` (3), `timeout` (30s), `base_url` |
| **TypeScript** | `npm i mixedbread` | `import { Mixedbread } from "mixedbread"; const mxbai = new Mixedbread({apiKey})` | `apiKey`, `maxRetries` (3), `timeout` (30000ms), `baseURL` |

- Both support **async** clients.
- Env vars: `MIXEDBREAD_API_KEY`.
- Built-in error handling, retries, type safety.

### CLI

Terminal tool for Store operations: bulk uploads, directory sync, CI/CD integration.

| Command area | Highlights |
|---|---|
| `files` | Upload/manage files |
| `stores` | Create/manage stores |
| `search` | `--top-k` (1–100), `--threshold` (0.0–1.0), `--return-metadata`, `--rerank`, `--file-search` (files vs chunks) |

### MCP Server

Model Context Protocol server bridging AI assistants (Claude Desktop, Cursor, Claude Code) to Stores — enables in-conversation search/retrieval.

### Vercel Integration

- Team install → creates Mixedbread org linked to Vercel.
- Products → create Stores in the org.
- Connect projects → auto-injects `MXBAI_API_KEY` + `MXBAI_STORE_ID`.
- Unified billing via Vercel.
- Multi-project connections (staging/prod sharing one store).
- Uninstall deletes the org + stores.

### Agent Skills

Installable `SKILL.md` packages for coding agents:
- `mixedbread-search` — working with Stores via SDKs (create, upload, search, filter, rerank).
- `mxbai-cli` — terminal-based workflows (setup, uploads, sync, scripting).

### Machine-Readable Docs

- `GET /question?q=...&limit=...` — live Q&A over all Mixedbread docs (pricing, API, blog, cookbook).
- `/llms.txt` and `/llms-full.txt` — broader agent-readable docs.

### Analysis

The integration surface is agent-first: official SDKs (Python/TS) for app integration, a CLI for ops/CI, an MCP server for in-IDE assistant search, Agent Skills for coding-agent guidance, and machine-readable doc endpoints (`/question`, `/llms.txt`). The Vercel integration is the one-click deploy path. This ecosystem reflects Mixedbread's positioning as retrieval infrastructure for the AI/agent era — every common integration path is pre-built.

---

## 19. Production: Bring Your Own Bucket

### Main Concepts

- **BYOB** — Enterprise feature; use your own object storage as the backend instead of Mixedbread-managed storage.
- **Scope** — Your bucket holds all user content + derived artifacts (chunks, generated content, search index). Mixedbread uses ephemeral compute and retains nothing beyond in-memory usage metering.
- **Enabled per organization** (contact to activate).
- **Supported providers** — major cloud object-storage providers.

### Key Considerations

- **Performance** — No impact if bucket is in `us-east-1` (colocated with Mixedbread compute). Other regions add cross-region latency + data-transfer cost.
- **Existing stores** — Connecting a bucket changes where **new** content lives; existing stores keep reading/writing Mixedbread-owned storage (not auto-migrated). Recreate stores to move data.
- **Cost** — Mixedbread charges for indexing/search compute; you pay cloud provider for storage/requests/transfer/KMS.
- **Access control** — You revoke access from your cloud at any time; revocation speed depends on method.
- **Compliance** — SOC + ISO 27001 certified; BYOB supports data-residency/sovereignty requirements.

### Analysis

BYOB is the enterprise data-sovereignty story: customers who cannot let content leave their cloud (regulated industries, EU GDPR residency, air-gapped-adjacent setups) can keep all content in their bucket while still using Mixedbread's retrieval compute. The "retain nothing beyond memory" claim and ephemeral-compute model is a strong privacy posture. The colocation guidance (us-east-1) is honest about the latency tradeoff. The lack of auto-migration is a current friction point (acknowledged as coming soon).

---

## 20. Capability Summary & Cross-Reference

### Capability → Endpoint Map

| Capability | Endpoint(s) | SDK Method |
|---|---|---|
| Create/manage stores | `/v1/stores` (CRUD) | `mxbai.stores.create/retrieve/list/update/delete` |
| Upload to store | `POST /v1/stores/{id}/files` | `mxbai.stores.files.upload` / `upload_and_poll` |
| Manage store files | `/v1/stores/{id}/files/{fid}` | `mxbai.stores.files.retrieve/list/update` |
| Multipart upload | `/v1/files/uploads/*` | `mxbai.files.uploads.create/get/list/abort/complete` |
| Raw file storage | `/v1/files/*` | `mxbai.files.create/retrieve/list/update/content` |
| Semantic search | `POST /v1/stores/search` | `mxbai.stores.search` |
| Reranking | `POST /v1/stores/search` (`search_options.rerank`) | `search_options={"rerank": ...}` |
| Agentic search | `POST /v1/stores/search` (`search_options.agentic`) | `search_options={"agentic": ...}` |
| Question answering | `POST /v1/stores/question-answering` | `mxbai.stores.question_answering` |
| Grep (regex) | `POST /v1/stores/grep` | `mxbai.stores.grep` |
| List chunks (metadata) | `POST /v1/stores/list-chunks` | `mxbai.stores.list_chunks` |
| Metadata facets | (facets endpoint) | `mxbai.stores.metadata_facets` |
| Web search | `POST /v1/stores/search` (`store_identifiers` incl. `mixedbread/web`) | include `"mixedbread/web"` |
| Query rewrite | `POST /v1/stores/search` (`search_options.rewrite_query`) | `search_options={"rewrite_query": True}` |

### Retrieval Mode Comparison

| Mode | Trigger | Ranking | Cost/Latency | Best For |
|---|---|---|---|---|
| Basic Search | default | semantic (1-stage) | lowest | fast lookups |
| + Rerank | `rerank: true` | semantic + reranker | +latency | precision on hard queries |
| + Rewrite Query | `rewrite_query: true` | rewritten + semantic | +1 model call | keyword-ish queries |
| Agentic Search | `agentic: true` | agent-managed (multi-round) | highest | multi-hop, complex questions |
| Question Answering | `/question-answering` | retrieval + LLM generation | +generation | grounded answers with citations |
| Grep | `/stores/grep` | regex match (no semantic) | low | exact tokens, error codes |
| List Chunks | `/stores/list-chunks` | metadata sort (no semantic) | low | deterministic browsing/export |
| Web Store | `mixedbread/web` in stores | always reranked | +web fetch | fresh/hybrid internal+web |

### Core Object Relationships

```
Organization
  └── Store (search index) ──── identified by id OR name
        ├── config (contextualization, save_content, expiration)
        ├── metadata (user-defined)
        └── Store File ────────── identified by id OR external_id
              ├── status: pending → in_progress → completed/failed/cancelled
              ├── metadata (propagates to all chunks)
              ├── generated_metadata (auto-typed: text/pdf/code/audio/video/image)
              └── Chunk ────────── the searchable unit
                    ├── inherits file metadata
                    ├── generated_metadata (chunk-level: lines, times, page)
                    ├── type: text / image_url / audio_url / video_url
                    └── score (in search results)
```

### Key Design Principles

1. **Unified chunk response** — Search, Agentic, QA, Grep, and Web Store all return the same ranked-chunk shape, making capabilities composable and upgradeable.
2. **Metadata as a first-class dimension** — Three layers (user, generated, system) with a consistent filter/sort/facet language across listing and search.
3. **Quality knobs are per-query** — Rerank, rewrite, agentic, score_threshold are opt-in via `search_options`, not store-level toggles, letting each query balance cost vs. precision.
4. **Managed pipeline with escape hatches** — Automatic parse/chunk/embed by default; MXJSON bypasses parsing; BYOB bypasses managed storage.
5. **Agent-native** — MCP server, Agent Skills, `/question` and `/llms.txt` doc endpoints, and Agentic Search position the platform for LLM-agent retrieval workflows.

---

*Sources: [Mixedbread Docs](https://www.mixedbread.com/docs), [API Reference](https://www.mixedbread.com/api-reference), [Concepts](https://www.mixedbread.com/docs/concepts), [Data Models](https://www.mixedbread.com/docs/stores/data-models), [Search](https://www.mixedbread.com/docs/stores/search), [Reranking](https://www.mixedbread.com/docs/stores/search/rerank), [Agentic Search](https://www.mixedbread.com/docs/stores/search/agentic-search), [Question Answering](https://www.mixedbread.com/docs/stores/search/question-answering), [Web Store](https://www.mixedbread.com/docs/stores/search/web-store), [Metadata Filtering](https://www.mixedbread.com/docs/stores/metadata-filtering), [Generated Metadata](https://www.mixedbread.com/docs/stores/store-files/generated-metadata), [MXJSON](https://www.mixedbread.com/docs/stores/store-files/mxjson), [Pagination](https://www.mixedbread.com/api-reference/pagination), [Rate Limits](https://www.mixedbread.com/api-reference/rate-limits), [BYOB](https://www.mixedbread.com/docs/production/bring-your-own-bucket), [SDKs](https://www.mixedbread.com/api-reference/sdks/python), [Vercel](https://www.mixedbread.com/api-reference/integrations/vercel). Last reviewed: July 2026.*