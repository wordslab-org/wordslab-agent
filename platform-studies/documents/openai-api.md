# OpenAI API Analysis — File Search & Vector Embeddings

> **Base URL:** `https://api.openai.com/v1` | **Docs:** `https://developers.openai.com/api/docs` | **Auth:** Bearer token (`OPENAI_API_KEY`)
> **SDKs:** `openai` (Python / TypeScript / .NET) | **CLI:** OpenAI CLI
> **Description:** OpenAI exposes two complementary retrieval capabilities. **File search** is a hosted tool in the Responses API that lets models autonomously search a knowledge base of uploaded files via semantic and keyword search. **Vector embeddings** provide standalone embedding generation for custom search, clustering, classification, and ML pipelines. Both are powered by **vector stores** — managed containers that automatically chunk, embed, and index files.

---

## Table of Contents

1. [Platform Overview & Core Concepts](#1-platform-overview--core-concepts)
2. [Files API — Uploading Content](#2-files-api--uploading-content)
3. [Vector Stores — Index Management](#3-vector-stores--index-management)
4. [Vector Store Files — Indexing & Lifecycle](#4-vector-store-files--indexing--lifecycle)
5. [Batch Operations — Bulk Ingestion](#5-batch-operations--bulk-ingestion)
6. [Chunking Strategies](#6-chunking-strategies)
7. [Attributes & Metadata](#7-attributes--metadata)
8. [File Search Tool — Hosted Retrieval in Responses API](#8-file-search-tool--hosted-retrieval-in-responses-api)
9. [Retrieval API — Direct Semantic Search](#9-retrieval-api--direct-semantic-search)
10. [Attribute Filtering](#10-attribute-filtering)
11. [Ranking Options & Hybrid Search](#11-ranking-options--hybrid-search)
12. [Query Rewriting](#12-query-rewriting)
13. [Embeddings API — Generating Vectors](#13-embeddings-api--generating-vectors)
14. [Embedding Models](#14-embedding-models)
15. [Dimensionality Reduction](#15-dimensionality-reduction)
16. [Embedding Use Cases](#16-embedding-use-cases)
17. [Distance Functions & Vector Math](#17-distance-functions--vector-math)
18. [Tokenization](#18-tokenization)
19. [Supported File Types](#19-supported-file-types)
20. [Pricing, Rate Limits & Expiration](#20-pricing-rate-limits--expiration)
21. [Capability Summary & Cross-Reference](#21-capability-summary--cross-reference)

---

## 1. Platform Overview & Core Concepts

### Main Concepts

OpenAI's retrieval stack is organized around three layers:

- **File** — Raw content uploaded via the Files API (`client.files.create`). Backed by a `file` object with an `id`, `filename`, `bytes`, `mime_type`, and `version`. Used for vector stores, fine-tuning, and other purposes.
- **Vector Store** — A named container (`vector_store`) that serves as a search index. When a file is added, OpenAI automatically parses, chunks, embeds, and indexes it. Identified by `vs_...` IDs.
- **Vector Store File** — A wrapper (`vector_store.file`) representing a `file` that has been chunked and embedded and associated with a `vector_store`. Carries an `attributes` map used for filtering.

Two consumption modes sit on top of vector stores:

- **File Search Tool** — A hosted tool (`type: "file_search"`) passed to the Responses API. The model decides when to invoke it, retrieves information from your files, and returns an answer with citations. No client-side retrieval code required.
- **Retrieval API** — Direct semantic search via `client.vector_stores.search`. Returns ranked chunks with scores; you synthesize responses yourself (or feed results to a chat completion).

A third, lower-level capability operates independently of vector stores:

- **Embeddings API** — `client.embeddings.create` generates vector embeddings for arbitrary text. You store and search them in your own vector database.

### End-to-End Flows

**File Search (hosted tool):**
```
Upload file ──▶ Create vector store ──▶ Add file to store ──▶ Poll until completed
                                                                     │
Responses API create (tools=[{type: "file_search", vector_store_ids: [...]}])
                                                                     │
                                              Model calls tool ──▶ file_search_call + message with citations
```

**Retrieval API (direct search):**
```
Upload file ──▶ Create vector store ──▶ Add file to store ──▶ Poll until completed
                                                                     │
                              vector_stores.search(query=...) ──▶ ranked chunks (score, content, attributes)
                                                                     │
                                          (optional) feed results to chat.completions.create for synthesis
```

**Embeddings (standalone):**
```
Text input ──▶ embeddings.create(model, input, dimensions?) ──▶ vector ──▶ store in your own vector DB
                                                                                    │
                                                         cosine similarity search / clustering / classification
```

### Key Differentiators

- **Hosted tool vs. direct API** — File search is fully managed (model-autonomous); Retrieval API gives you control over search params and response synthesis.
- **Automatic pipeline** — Vector stores handle parsing, chunking, embedding, and indexing with zero infrastructure.
- **Hybrid search** — Reciprocal rank fusion combines semantic embedding matches with sparse keyword matches, with tunable weights.
- **Dimension-shortable embeddings** — `text-embedding-3-*` models support a `dimensions` parameter to trade accuracy for smaller vectors without retraining.
- **Citations built-in** — File search returns `file_citation` annotations mapping answer spans to source files.

---

## 2. Files API — Uploading Content

### Main Concepts

- **Purpose:** Upload raw files to OpenAI storage. Files can then be attached to vector stores, used for fine-tuning, or retrieved directly.
- **`purpose` field:** Must be set to `"assistants"` for files intended for vector stores / file search.
- **Sources:** Local file paths or remote URLs (download first in client code, then upload).

### Endpoints & Parameters

| Endpoint | Method | Purpose | Key Parameters |
|---|---|---|---|
| `/v1/files` | `POST` | Upload a file | `file` (binary), `purpose` (`"assistans"` / `"vision"` / `"fine-tune"` etc.) |
| `/v1/files/{file_id}` | `GET` | Retrieve file metadata | — |
| `/v1/files` | `GET` | List files (paginated) | `limit`, `after` (cursor) |
| `/v1/files/{file_id}/content` | `GET` | Download file content | — |
| `/v1/files/{file_id}` | `DELETE` | Delete a file | — |

### File Object Response Model

| Field | Type | Description |
|---|---|---|
| `id` | string | Unique file ID (`file-...`) |
| `object` | string | `"file"` |
| `filename` | string | Name with extension |
| `bytes` | integer | Size in bytes |
| `mime_type` | string | MIME type |
| `purpose` | string | Upload purpose |
| `status` | string | Processing status |
| `version` | integer | Version (increments on update) |
| `created_at` / `updated_at` | datetime | Timestamps |

### Analysis

The Files API is the ingestion entry point for both file search and retrieval. The `purpose` field gates downstream usage — `"assistants"` is required for vector store integration. The dual upload pattern (local file handle vs. remote URL download) is handled in client code, not by the API itself. File content can be retrieved via a dedicated content endpoint.

---

## 3. Vector Stores — Index Management

### Main Concepts

- **Vector Store = Search Index** — Primary container for searchable content. Files added are automatically chunked, embedded, and indexed.
- **Expiration Policies** — Optional `expires_after` auto-expires stores to control costs; when expired, all associated files are deleted and billing stops.
- **File Counts** — Tracks counts of files in various processing states.
- **Dual API surface** — Used by both the File Search tool (via `vector_store_ids`) and the Retrieval API (via `vector_store_id`).

### Endpoints & Parameters

| Endpoint | Method | Purpose | Key Parameters |
|---|---|---|---|
| `/v1/vector_stores` | `POST` | Create a vector store | `name`, `file_ids`, `expires_after`, `chunking_strategy` |
| `/v1/vector_stores/{vector_store_id}` | `GET` | Retrieve vector store | — |
| `/v1/vector_stores/{vector_store_id}` | `POST` | Update vector store | `name`, `expires_after` |
| `/v1/vector_stores/{vector_store_id}` | `DELETE` | Delete vector store + contents | — |
| `/v1/vector_stores` | `GET` | List vector stores (paginated) | `limit`, `after` (cursor) |

### `expires_after` Object

| Field | Type | Description |
|---|---|---|
| `anchor` | string | Anchor for expiration calculation (e.g. `"last_active_at"`) |
| `days` | integer | Number of days after anchor before expiration |

### Vector Store Response Model

| Field | Type | Description |
|---|---|---|
| `id` | string | Unique identifier (`vs_...`) |
| `object` | string | `"vector_store"` |
| `name` | string | Display name |
| `file_counts` | object | Counts per status (e.g. `completed`, `in_progress`, `failed`) |
| `status` | string | Store status |
| `expires_after` | object | Expiration policy |
| `expires_at` | datetime | Computed expiry timestamp |
| `created_at` / `updated_at` / `last_active_at` | datetime | Timestamps |

### Analysis

Vector stores are the managed-index abstraction that eliminates infrastructure. The `expires_after` policy with `last_active_at` anchoring is a cost-control mechanism — dormant knowledge bases auto-expire, stopping the $0.10/GB/day billing. File counts per status enable monitoring of ingestion progress across batch uploads. The store is the single scoping unit for both consumption modes (file search tool and direct retrieval).

---

## 4. Vector Store Files — Indexing & Lifecycle

### Main Concepts

- **Automatic Processing** — When a file is added, OpenAI parses, chunks, embeds, and indexes it asynchronously.
- **Async Lifecycle** — Operations are asynchronous; use `create_and_poll` / `upload_and_poll` helpers, or poll `list`/`retrieve` until `status` is `completed`.
- **Eventually Consistent Deletes** — Removing a file may leave its content briefly searchable.
- **Attributes** — Each vector store file carries an `attributes` map (see §7) used for pre-search filtering.

### Endpoints & Parameters

| Endpoint | Method | Purpose | Key Parameters |
|---|---|---|---|
| `/v1/vector_stores/{vs_id}/files` | `POST` | Add existing file to store | `file_id`, `attributes`, `chunking_strategy` |
| `/v1/vector_stores/{vs_id}/files` | `POST` (upload) | Upload + add in one call | `file` (binary), `attributes`, `chunking_strategy` |
| `/v1/vector_stores/{vs_id}/files/{file_id}` | `GET` | Retrieve vector store file | — |
| `/v1/vector_stores/{vs_id}/files/{file_id}` | `POST` | Update attributes | `attributes` |
| `/v1/vector_stores/{vs_id}/files/{file_id}` | `DELETE` | Remove file from store | — |
| `/v1/vector_stores/{vs_id}/files` | `GET` | List files in store (paginated) | `limit`, `after` |

### Helper Functions (SDK)

| Helper | Behavior |
|---|---|
| `create_and_poll` | Create vector store file, block until processing completes |
| `upload_and_poll` | Upload file content + create vector store file, block until complete |

### Rate Limits

Requests to `/vector_stores/{vector_store_id}/files` and `/vector_stores/{vector_store_id}/file_batches` share a **per-vector-store limit of 300 requests per minute**.

### File Limits

| Limit | Value |
|---|---|
| Max file size | 512 MB |
| Max tokens per file | 5,000,000 (computed automatically on attach) |

### Analysis

The async lifecycle with polling helpers is the key usability pattern — files are not searchable until `completed`. The per-vector-store rate limit (300 RPM) means large-scale ingestion should use batch operations (§5) to avoid throttling. The `upload_and_poll` convenience method collapses upload + attach + wait into a single SDK call. The 5M-token-per-file ceiling accommodates large documents (roughly 3,750 pages at ~1,333 tokens/page).

---

## 5. Batch Operations — Bulk Ingestion

### Main Concepts

- **Batch = Up to 500 files** per request. Recommended for higher-throughput ingestion to reduce contention and improve latency.
- **Two input modes (mutually exclusive):**
  - `file_ids` — array of IDs with optional shared `attributes` and/or `chunking_strategy`.
  - `files` — array of per-file objects, each with `file_id` + optional `attributes` + optional `chunking_strategy`.
- **Async** — Use `create_and_poll` to block, or poll `retrieve` for status.

### Endpoints & Parameters

| Endpoint | Method | Purpose | Key Parameters |
|---|---|---|---|
| `/v1/vector_stores/{vs_id}/file_batches` | `POST` | Create batch | `files` OR (`file_ids` + optional shared `attributes`/`chunking_strategy`) |
| `/v1/vector_stores/{vs_id}/file_batches/{batch_id}` | `GET` | Retrieve batch status | — |
| `/v1/vector_stores/{vs_id}/file_batches/{batch_id}/cancel` | `POST` | Cancel in-progress batch | — |
| `/v1/vector_stores/{vs_id}/file_batches` | `GET` | List batches (paginated) | `limit`, `after` |

### Batch File Object (in `files` array)

| Field | Type | Description |
|---|---|---|
| `file_id` | string | ID of file to add |
| `attributes` | object | Optional per-file attributes (see §7) |
| `chunking_strategy` | object | Optional per-file chunking config (see §6) |

### Analysis

Batch operations are the throughput optimization layer. Per-file `attributes` and `chunking_strategy` in the `files` array mode enable heterogeneous ingestion — e.g. tagging finance docs with `{department: "finance"}` while using a different chunk size for legal docs in the same batch. The 500-file ceiling balances throughput against atomicity. Cancellation support is important for long-running batches where upstream data changes.

---

## 6. Chunking Strategies

### Main Concepts

- **Automatic Chunking** — By default, files are split into 800-token chunks with 400-token overlap.
- **Configurable** — `chunking_strategy` can be set per file (in create/upload/batch operations).

### Static Chunking Strategy

```json
{
    "type": "static",
    "max_chunk_size_tokens": 1200,
    "chunk_overlap_tokens": 200
}
```

| Parameter | Type | Constraints | Default | Description |
|---|---|---|---|---|
| `type` | string | `"static"` | — | Chunking strategy type |
| `max_chunk_size_tokens` | integer | 100–4096 | 800 | Maximum tokens per chunk |
| `chunk_overlap_tokens` | integer | ≥ 0, ≤ `max_chunk_size_tokens / 2` | 400 | Overlap between adjacent chunks |

### Analysis

The static chunking strategy is the only one documented in the guides. The 100–4096 token range covers both fine-grained (FAQ snippets) and coarse-grained (full sections) chunking. The overlap constraint (≤ half the chunk size) prevents excessive redundancy while maintaining context continuity across boundaries. The default 800/400 split is a balanced general-purpose configuration. The API reference may expose additional strategy types not covered in the guides.

---

## 7. Attributes & Metadata

### Main Concepts

- **Attributes = File-level Metadata** — Key-value pairs attached to `vector_store.file` objects, used for pre-search filtering.
- **Limits** — At most 16 keys, up to 256 characters per key.
- **Value Types** — Strings and numbers (e.g. unix timestamps for dates).

### Setting Attributes

```python
client.vector_stores.files.create(
    vector_store_id="<vector_store_id>",
    file_id="file_123",
    attributes={
        "region": "US",
        "category": "Marketing",
        "date": 1672531200
    }
)
```

### Attributes in Search Results

```json
"attributes": {
    "region": "North America",
    "author": "Wildlife Department"
}
```

### Analysis

Attributes are the filtering dimension for vector stores. The 16-key / 256-char limits keep filter indexes compact. The dual value type (string/number) supports both categorical filtering (`region`, `category`) and range queries (`date` as unix timestamp). Attributes propagate to search results, enabling downstream UIs to display provenance. Unlike Mixedbread's three-layer metadata system (user/generated/system), OpenAI keeps a single flat attributes map — simpler but less expressive for auto-extracted structure.

---

## 8. File Search Tool — Hosted Retrieval in Responses API

### Main Concepts

- **Hosted Tool** — File search is a built-in tool in the Responses API. The model autonomously decides when to call it, retrieves from your vector stores, and generates an answer with citations.
- **No Client-Side Code** — You don't implement search logic; the model handles tool invocation, retrieval, and synthesis.
- **Multi-Store** — Can search multiple vector stores in one call via `vector_store_ids`.
- **Citations** — Returns `file_citation` annotations mapping answer text spans to source files (`file_id`, `filename`).
- **Semantic + Keyword** — Combines semantic search with keyword matching.

### Tool Configuration (in Responses API `tools` array)

```python
response = client.responses.create(
    model="gpt-5.6",
    input="What is deep research by OpenAI?",
    tools=[{
        "type": "file_search",
        "vector_store_ids": ["<vector_store_id>"],
        "max_num_results": 2,
        "filters": {
            "type": "in",
            "key": "category",
            "value": ["blog", "announcement"]
        }
    }]
)
```

| Parameter | Type | Required | Default | Description |
|---|---|---|---|---|
| `type` | string | yes | — | Must be `"file_search"` |
| `vector_store_ids` | array | yes | — | Vector store IDs to search |
| `max_num_results` | integer | no | — | Limit number of results (reduces tokens/latency, may reduce quality) |
| `filters` | object | no | — | Metadata filter conditions (see §10) |

### Including Search Results in Response

By default, the `file_search_call` output item has `search_results: null`. To include them:

```python
response = client.responses.create(
    model="gpt-5.6",
    input="What is deep research by OpenAI?",
    tools=[{"type": "file_search", "vector_store_ids": ["<vector_store_id>"]}],
    include=["file_search_call.results"]
)
```

### Response Output Items

The response `output` array contains:

1. **`file_search_call`** — The tool call record.
2. **`message`** — The model's answer with citations.

### `file_search_call` Output Item

| Field | Type | Description |
|---|---|---|
| `type` | string | `"file_search_call"` |
| `id` | string | Tool call ID (`fs_...`) |
| `status` | string | `"completed"` |
| `queries` | array | The search queries the model generated |
| `search_results` | array\|null | Results (only if `include` set) |

### `message` Output Item with Citations

```json
{
    "type": "message",
    "role": "assistant",
    "content": [{
        "type": "output_text",
        "text": "Deep research is a sophisticated capability...",
        "annotations": [{
            "type": "file_citation",
            "index": 992,
            "file_id": "file-2dtbBZdjtDKS8eqWxqbgDi",
            "filename": "deep_research_blog.pdf"
        }]
    }]
}
```

### `file_citation` Annotation

| Field | Type | Description |
|---|---|---|
| `type` | string | `"file_citation"` |
| `index` | integer | Character offset in the output text where the citation applies |
| `file_id` | string | Source file ID |
| `filename` | string | Source file name |

### Analysis

File search is the zero-code RAG path — the model owns the retrieval loop, including query generation (visible in `queries`). The `max_num_results` knob is the primary cost/latency lever, trading answer quality for fewer retrieved chunks. The `filters` parameter (using the same comparison/compound filter syntax as the Retrieval API) enables scoped searches, e.g. only searching "blog" and "announcement" category files. The citation system with `index` offsets enables precise inline source attribution in UIs. The `include` parameter is opt-in because search results can be large; the annotations alone suffice for most use cases. The `queries` field is valuable for debugging — it reveals how the model reformulated the user's question for retrieval.

---

## 9. Retrieval API — Direct Semantic Search

### Main Concepts

- **Direct Semantic Search** — Query a vector store with `client.vector_stores.search`, get ranked chunks with scores.
- **Semantic + Keyword** — Surfaces semantically similar results even with few or no shared keywords (powered by embeddings).
- **Pagination** — Returns up to 10 results by default (max 50), with `has_more` / `next_page` cursor pagination.
- **Response Synthesis** — You pass results to `chat.completions.create` yourself (the guide shows wrapping results in `<sources>` XML).

### Endpoint & Parameters

```python
results = client.vector_stores.search(
    vector_store_id=vector_store.id,
    query="What is the return policy?",
    max_num_results=10,
    rewrite_query=True,
    attribute_filter={"type": "eq", "key": "region", "value": "us"},
    ranking_options={
        "ranker": "auto",
        "score_threshold": 0.5,
        "hybrid_search": {
            "embedding_weight": 0.5,
            "text_weight": 0.5
        }
    }
)
```

| Parameter | Type | Required | Default | Description |
|---|---|---|---|---|
| `vector_store_id` | string | yes | — | Vector store to search |
| `query` | string | yes | — | Natural language search query |
| `max_num_results` | integer | no | 10 (max 50) | Maximum results to return |
| `rewrite_query` | boolean | no | false | Auto-rewrite query for optimal search (see §12) |
| `attribute_filter` | object | no | — | Comparison or compound filter (see §10) |
| `ranking_options` | object | no | — | Ranking config (see §11) |

### Search Response Model

```json
{
    "object": "vector_store.search_results.page",
    "search_query": "How many woodchucks are allowed per passenger?",
    "data": [
        {
            "file_id": "file-12345",
            "filename": "woodchuck_policy.txt",
            "score": 0.85,
            "attributes": {"region": "North America", "author": "Wildlife Department"},
            "content": [
                {"type": "text", "text": "According to the latest regulations..."},
                {"type": "text", "text": "Ensure that the woodchucks..."}
            ]
        }
    ],
    "has_more": false,
    "next_page": null
}
```

| Response Field | Type | Description |
|---|---|---|
| `object` | string | `"vector_store.search_results.page"` |
| `search_query` | string | The (possibly rewritten) query |
| `data` | array | List of result objects |
| `data[].file_id` | string | Source file ID |
| `data[].filename` | string | Source file name |
| `data[].score` | float | Similarity score |
| `data[].attributes` | object | File attributes |
| `data[].content` | array | Array of `{type: "text", text: "..."}` chunks |
| `has_more` | boolean | Whether more results are available |
| `next_page` | string\|null | Pagination cursor |

### Response Synthesis Pattern

```python
query = f"""Use the below sources to answer the question.

Sources:
<sources>
{format_results(results)}
</sources>

Question: {user_query}"""

response = client.chat.completions.create(
    messages=[
        {"role": "developer", "content": "Produce a concise answer based on the provided sources."},
        {"role": "user", "content": query}
    ],
    model="gpt-5.6",
    temperature=0,
)
```

### Analysis

The Retrieval API is the developer-control counterpart to the hosted file search tool. You own the search parameters (ranking, filtering, rewriting, result count) and the response synthesis prompt. The `content` array in results contains the actual text chunks — multiple per result, enabling the model to see surrounding context. Cursor pagination (`next_page`) supports iterating beyond 50 results. The `format_results` helper wrapping results in `<sources><result file_id=... file_name=...><content>...</content></result></sources>` XML is a proven prompt pattern for grounded generation. Compared to the file search tool, this mode trades convenience for control and observability.

---

## 10. Attribute Filtering

### Main Concepts

- **Pre-Search Filtering** — Narrow results by applying criteria to file `attributes` (or built-in properties like `filename`) before semantic search.
- **Two Filter Types** — Comparison filters (compare a key to a value) and compound filters (combine filters with `and`/`or`).
- **Property vs. Key** — `key` filters on user-defined attributes; `property` filters on built-in fields (e.g. `filename`).

### Comparison Filter

```json
{
    "type": "eq" | "ne" | "gt" | "gte" | "lt" | "lte" | "in" | "nin",
    "key": "attributes_key",
    "value": "target_value"
}
```

| Field | Type | Description |
|---|---|---|
| `type` | string | Comparison operator (see table below) |
| `key` | string | Attributes key to compare (for user-defined attributes) |
| `property` | string | Built-in property name (e.g. `filename`) — used instead of `key` |
| `value` | string\|array\|number | Value to compare against (array for `in`/`nin`) |

### Comparison Operators

| Operator | Description |
|---|---|
| `eq` | Equal |
| `ne` | Not equal |
| `gt` | Greater than |
| `gte` | Greater than or equal |
| `lt` | Less than |
| `lte` | Less than or equal |
| `in` | In a set of values |
| `nin` | Not in a set of values |

### Compound Filter

```json
{
    "type": "and" | "or",
    "filters": [...]
}
```

| Field | Type | Description |
|---|---|---|
| `type` | string | Logical operator: `and` or `or` |
| `filters` | array | Nested comparison or compound filters |

### Example Filters

**Region (eq):**
```json
{"type": "eq", "key": "region", "value": "us"}
```

**Date range (and + gte/lte):**
```json
{
    "type": "and",
    "filters": [
        {"type": "gte", "key": "date", "value": 1704067200},
        {"type": "lte", "key": "date", "value": 1710892800}
    ]
}
```

**Filenames (in, using `property`):**
```json
{"type": "in", "property": "filename", "value": ["example.txt", "example2.txt"]}
```

**Exclude filenames (nin):**
```json
{"type": "nin", "property": "filename", "value": ["draft.txt", "internal_notes.md"]}
```

**Complex nested (or/and/eq):**
```json
{
    "type": "or",
    "filters": [
        {
            "type": "and",
            "filters": [
                {
                    "type": "or",
                    "filters": [
                        {"type": "eq", "key": "project_code", "value": "X123"},
                        {"type": "eq", "key": "project_code", "value": "X999"}
                    ]
                },
                {"type": "eq", "key": "confidentiality", "value": "top_secret"}
            ]
        },
        {"type": "eq", "key": "language", "value": "en"}
    ]
}
```

### Analysis

The filter system is a structured query language over file attributes and built-in properties. The `key` vs. `property` distinction is important — `key` accesses user-defined attributes, while `property` accesses system fields like `filename`. The 8 comparison operators cover equality, inequality, ordering, and set membership. The compound `and`/`or` operators support arbitrary nesting, enabling expressive queries like "top-secret projects X123 or X999, OR any English-language content". Notably absent is a `not` compound operator (you must use `ne`/`nin` at the leaf level). Dates are expressed as unix timestamps (integers), requiring client-side conversion.

---

## 11. Ranking Options & Hybrid Search

### Main Concepts

- **Ranker Selection** — Choose a ranking algorithm (`auto` or versioned like `default-2024-08-21`).
- **Score Threshold** — Filter results by minimum relevance score (0.0–1.0).
- **Hybrid Search** — Reciprocal rank fusion (RRF) blends semantic embedding matches with sparse keyword matches, with tunable weights.

### `ranking_options` Object

| Parameter | Type | Default | Description |
|---|---|---|---|
| `ranker` | string | `"auto"` | Ranker to use (e.g. `"auto"`, `"default-2024-08-21"`) |
| `score_threshold` | float | — | 0.0–1.0; higher limits to more relevant chunks |
| `hybrid_search` | object | — | Hybrid search configuration |
| `hybrid_search.embedding_weight` | float | — | (aka `rrf_embedding_weight`) Weight for semantic matches |
| `hybrid_search.text_weight` | float | — | (aka `rrf_text_weight`) Weight for keyword matches |

**Constraint:** At least one of `embedding_weight` or `text_weight` must be greater than zero.

### Tuning Guidance

| Action | Effect |
|---|---|
| Increase `embedding_weight` | Emphasize semantic similarity |
| Increase `text_weight` | Emphasize textual/keyword overlap |
| Increase `score_threshold` | Fewer, more relevant results (may exclude useful ones) |

### Analysis

Hybrid search is OpenAI's answer to the semantic-vs-lexical tradeoff. RRF is a proven fusion method that normalizes ranks from different retrieval systems and combines them weighted. The dual alias names (`embedding_weight`/`rrf_embedding_weight`) suggest API evolution. The `score_threshold` is a precision guard — useful for RAG where irrelevant context degrades answer quality. The versioned ranker (`default-2024-08-21`) enables reproducibility: pin a ranker version to avoid behavior changes from `auto` updates. The constraint that at least one weight must be > 0 prevents degenerate configurations.

---

## 12. Query Rewriting

### Main Concepts

- **Automatic Query Optimization** — Setting `rewrite_query=true` on `search` rewrites the query into a more search-friendly form before retrieval.
- **Observable** — The rewritten query is returned in the response's `search_query` field.

### Parameter

| Parameter | Type | Default | Description |
|---|---|---|---|
| `rewrite_query` | boolean | false | Auto-rewrite query for optimal search performance |

### Example Rewrites

| Original | Rewritten |
|---|---|
| I'd like to know the height of the main office building. | primary office building height |
| What are the safety regulations for transporting hazardous materials? | safety regulations for hazardous materials |
| How do I file a complaint about a service issue? | service complaint filing process |

### Analysis

Query rewriting is a pre-retrieval optimization that strips conversational filler and restructures questions into keyword-dense queries that match embedding spaces better. The examples show consistent patterns: removing politeness ("I'd like to know"), dropping question words ("What are", "How do I"), and converting to noun-phrase form. The observable `search_query` field enables debugging and A/B testing. This is opt-in (default `false`) because it adds a model call and latency. In the File Search tool context, the model handles query generation automatically (visible in `queries`), so `rewrite_query` is specific to the direct Retrieval API.

---

## 13. Embeddings API — Generating Vectors

### Main Concepts

- **Embeddings = Vector Representations** — An embedding is a vector (list) of floating point numbers measuring text relatedness. Distance between vectors measures relatedness (small = high relatedness).
- **Standalone** — Unlike file search/retrieval, embeddings are generated and returned to you. You store and search them in your own vector database.
- **Batch Input** — `input` can be a single string or an array of strings.
- **Billing** — Priced per input token.

### Endpoint & Parameters

**`POST /v1/embeddings`**

```python
response = client.embeddings.create(
    input="Your text string goes here",
    model="text-embedding-3-small",
    encoding_format="float",
    dimensions=256
)
```

| Parameter | Type | Required | Default | Description |
|---|---|---|---|---|
| `input` | string\|array | yes | — | Text string(s) to embed |
| `model` | string | yes | — | Embedding model ID (e.g. `text-embedding-3-small`) |
| `encoding_format` | string | no | `"float"` | Output format (`"float"` or `"base64"`) |
| `dimensions` | integer | no | model-dependent | Number of dimensions (shorten without losing concept properties) |

### Response Model

```json
{
    "object": "list",
    "data": [
        {
            "object": "embedding",
            "index": 0,
            "embedding": [-0.006929283, -0.005336422, -4.547132e-5, -0.024047505]
        }
    ],
    "model": "text-embedding-3-small",
    "usage": {
        "prompt_tokens": 5,
        "total_tokens": 5
    }
}
```

| Field | Type | Description |
|---|---|---|
| `object` | string | `"list"` |
| `data` | array | Array of embedding objects |
| `data[].object` | string | `"embedding"` |
| `data[].index` | integer | Position in input array |
| `data[].embedding` | array | Vector of floats (length = model default or `dimensions` param) |
| `model` | string | Model used |
| `usage.prompt_tokens` | integer | Tokens in input |
| `usage.total_tokens` | integer | Total tokens billed |

### Analysis

The Embeddings API is the lowest-level retrieval primitive — it gives you vectors and nothing else. The `encoding_format` option (`float` vs. `base64`) trades human-readability for compactness in transit. The `dimensions` parameter (§15) is the standout feature, enabling runtime dimensionality reduction. The `usage` object enables cost tracking per request. The `index` field in each embedding object supports batch inputs (array of strings) with ordered results. The response is self-contained — no citations, no search, no synthesis. This is the building block for custom RAG pipelines where you need control over the vector store (e.g. Pinecone, Weaviate, pgvector).

---

## 14. Embedding Models

### Main Concepts

- **Third-Generation Models** — Denoted by `-3` in the model ID. Trained with MRL (Matryoshka Representation Learning) for dimension shortening.
- **Per-Token Pricing** — Usage billed by input tokens. Max input is 8192 tokens for all current models.

### Model Comparison

| Model | Default Dimensions | ~ Pages per Dollar | MTEB Score | Max Input |
|---|---|---|---|---|
| `text-embedding-3-small` | 1536 | 62,500 | 62.3% | 8192 tokens |
| `text-embedding-3-large` | 3072 | 9,615 | 64.6% | 8192 tokens |
| `text-embedding-ada-002` | 1536 | 12,500 | 61.0% | 8192 tokens |

*(~800 tokens per page assumed for pricing.)*

### Analysis

The `text-embedding-3-small` model is the cost-performance sweet spot — 62,500 pages per dollar with 62.3% MTEB score. The `text-embedding-3-large` model offers the best quality (64.6% MTEB) at ~6.5× the cost but with double the default dimensions (3072 vs 1536). The legacy `text-embedding-ada-002` is superseded by both v3 models on quality, though it's cheaper than `3-large`. The 8192-token input limit (~6 pages) means longer documents must be chunked before embedding — this is where the vector store's automatic chunking adds value. The MTEB (Massive Text Embedding Benchmark) scores provide a standardized comparison across 56 multilingual tasks.

---

## 15. Dimensionality Reduction

### Main Concepts

- **MRL Training** — V3 embedding models were trained with [Matryoshka Representation Learning](https://arxiv.org/abs/2205.13147), allowing developers to shorten embeddings (remove trailing numbers) without losing concept-representing properties.
- **`dimensions` API Parameter** — The recommended approach: specify desired dimensions at embedding creation time.
- **Manual Shortening** — Alternative: truncate the vector post-hoc, but you must L2-normalize afterward.

### Using the `dimensions` Parameter

```python
response = client.embeddings.create(
    model="text-embedding-3-large",
    input="Your text",
    dimensions=1024  # Shorten from 3072 to 1024
)
```

### Manual Shortening with Normalization

```python
import numpy as np

def normalize_l2(x):
    x = np.array(x)
    if x.ndim == 1:
        norm = np.linalg.norm(x)
        if norm == 0:
            return x
        return x / norm
    else:
        norm = np.linalg.norm(x, 2, axis=1, keepdims=True)
        return np.where(norm == 0, x, x / norm)

cut_dim = response.data[0].embedding[:256]
norm_dim = normalize_l2(cut_dim)
```

### Performance Tradeoff

A `text-embedding-3-large` embedding shortened to **256 dimensions** still outperforms an unshortened `text-embedding-ada-002` embedding (1536 dimensions) on MTEB. This enables using the best model with vector databases that have dimension limits.

### Analysis

Dimension shortening is a significant capability. It decouples model quality from storage cost — you can use `text-embedding-3-large` (best quality) even with a vector DB that caps at 1024 dimensions, by setting `dimensions=1024`. The MRL training ensures the leading dimensions capture the most important semantic information, so truncation degrades gracefully. The distinction between API-time `dimensions` (server handles normalization) and manual truncation (client must L2-normalize) matters: the API approach is both simpler and safer. The 256-dim `3-large` outperforming 1536-dim `ada-002` is a compelling data point for migration.

---

## 16. Embedding Use Cases

### Main Concepts

The embeddings guide documents eight representative use cases, all with cookbook notebook references:

### Use Case Catalog

| Use Case | Method | Cookbook |
|---|---|---|
| **Semantic text search** | Cosine similarity between query and document embeddings; return top-N | [Semantic_text_search_using_embeddings](https://developers.openai.com/cookbook/examples/semantic_text_search_using_embeddings) |
| **Code search** | Embed Python functions; cosine similarity between NL query and function embeddings | [Code_search](https://developers.openai.com/cookbook/examples/code_search_using_embeddings) |
| **Recommendations** | Rank strings by embedding distance from a source string | [Recommendation_using_embeddings](https://developers.openai.com/cookbook/examples/recommendation_using_embeddings) |
| **Question answering** | Embedding-based retrieval + context injection into chat completion | [Question_answering_using_embeddings](https://developers.openai.com/cookbook/examples/question_answering_using_embeddings) |
| **Data visualization** | t-SNE dimensionality reduction to 2D for visual clustering | [Visualizing_embeddings_in_2D](https://developers.openai.com/cookbook/examples/visualizing_embeddings_in_2d) |
| **Regression** | Embeddings as ML features for RandomForestRegressor (predict star rating) | [Regression_using_embeddings](https://developers.openai.com/cookbook/examples/regression_using_embeddings) |
| **Classification** | Embeddings as ML features for RandomForestClassifier (5-bucket star rating) | [Classification_using_embeddings](https://developers.openai.com/cookbook/examples/classification_using_embeddings) |
| **Zero-shot classification** | Embed class labels; predict class with highest cosine similarity to input | [Zero-shot_classification_with_embeddings](https://developers.openai.com/cookbook/examples/zero-shot_classification_with_embeddings) |
| **Cold-start recommendation** | Average user/product embeddings from reviews; predict preference by similarity | [User_and_product_embeddings](https://developers.openai.com/cookbook/examples/user_and_product_embeddings) |
| **Clustering** | KMeans on embeddings to discover hidden groupings | [Clustering](https://developers.openai.com/cookbook/examples/clustering) |

### Key Patterns

**Semantic text search:**
```python
embedding = get_embedding(query, model='text-embedding-3-small')
df['similarities'] = df.ada_embedding.apply(lambda x: cosine_similarity(x, embedding))
res = df.sort_values('similarities', ascending=False).head(n)
```

**Zero-shot classification:**
```python
labels = ['negative', 'positive']
label_embeddings = [get_embedding(label, model=model) for label in labels]
prediction = 'positive' if cosine_similarity(review_embedding, label_embeddings[1]) > \
             cosine_similarity(review_embedding, label_embeddings[0]) else 'negative'
```

**Cold-start (averaging embeddings):**
```python
user_embeddings = df.groupby('UserId').ada_embedding.apply(np.mean)
prod_embeddings = df.groupby('ProductId').ada_embedding.apply(np.mean)
```

### Analysis

The use case catalog reveals embeddings as a general-purpose text-understanding primitive, not just for search. The key insight is that embeddings compress semantic meaning into a numeric space where standard operations (cosine similarity, averaging, KMeans, RandomForest) become meaningful. The zero-shot classification pattern (embed labels, compare) is particularly elegant — no training data needed. The cold-start recommendation via averaging is notable: mean-of-embeddings produces a reasonable user/product profile from sparse data. The ML feature encoder use cases (regression/classification) show embeddings are information-dense — the guide explicitly warns that even 10% PCA/SVD reduction worsens downstream performance. The t-SNE visualization is a diagnostic tool for understanding cluster structure before building retrieval systems.

---

## 17. Distance Functions & Vector Math

### Main Concepts

- **Cosine Similarity Recommended** — The guide explicitly recommends cosine similarity as the distance function.
- **Normalized Embeddings** — OpenAI embeddings are normalized to length 1, which means:
  - Cosine similarity can be computed as a simple dot product (faster).
  - Cosine similarity and Euclidean distance yield identical rankings.

### Recommendations

| Question | Answer |
|---|---|
| Which distance function? | Cosine similarity (via dot product) |
| How to find K nearest vectors? | Use a vector database (Pinecone, Weaviate, pgvector, etc.) |
| Can I share embeddings? | Yes — customers own their inputs and outputs |
| Do v3 models know recent events? | No — knowledge cutoff is September 2021 |

### Analysis

The normalization-to-unit-length property is a meaningful design choice: it makes dot product equivalent to cosine similarity (cheaper compute) and makes cosine and Euclidean rankings identical (no need to choose). The recommendation to use a vector database for K-nearest-neighbor search acknowledges that brute-force cosine over large corpora is impractical — this is where the Embeddings API hands off to external infrastructure (vs. the managed vector stores which handle this internally). The September 2021 knowledge cutoff for v3 models is a consideration for time-sensitive content, though the guide notes this is less impactful for embeddings than for generation models.

---

## 18. Tokenization

### Main Concepts

- **`tiktoken`** — OpenAI's Python tokenizer for counting tokens before embedding.
- **`cl100k_base` Encoding** — The encoding used by `text-embedding-3-*` models.

### Counting Tokens

```python
import tiktoken

def num_tokens_from_string(string: str, encoding_name: str) -> int:
    encoding = tiktoken.get_encoding(encoding_name)
    return len(encoding.encode(string))

num_tokens_from_string("tiktoken is great!", "cl100k_base")
```

### Analysis

Pre-embedding token counting is essential for cost estimation and for ensuring inputs don't exceed the 8192-token model limit. The `tiktoken` library is OpenAI-specific (not a general BPE tokenizer), and `cl100k_base` is the encoding shared across v3 embedding models and GPT-4-series models. This shared encoding means token counts are directly comparable between embedding and generation API calls — useful for budgeting end-to-end RAG pipelines.

---

## 19. Supported File Types

### Main Concepts

Both the File Search tool and vector stores support the same set of file formats. For `text/` MIME types, encoding must be `utf-8`, `utf-16`, or `ascii`.

### Supported Formats

| Extension | MIME Type |
|---|---|
| `.c` | `text/x-c` |
| `.cpp` | `text/x-c++` |
| `.cs` | `text/x-csharp` |
| `.css` | `text/css` |
| `.doc` | `application/msword` |
| `.docx` | `application/vnd.openxmlformats-officedocument.wordprocessingml.document` |
| `.go` | `text/x-golang` |
| `.html` | `text/html` |
| `.java` | `text/x-java` |
| `.js` | `text/javascript` |
| `.json` | `application/json` |
| `.md` | `text/markdown` |
| `.pdf` | `application/pdf` |
| `.php` | `text/x-php` |
| `.pptx` | `application/vnd.openxmlformats-officedocument.presentationml.presentation` |
| `.py` | `text/x-python` / `text/x-script.python` |
| `.rb` | `text/x-ruby` |
| `.sh` | `application/x-sh` |
| `.tex` | `text/x-tex` |
| `.ts` | `application/typescript` |
| `.txt` | `text/plain` |

### Analysis

The format list is code- and document-centric: programming languages (C, C++, C#, Go, Java, JS, PHP, Python, Ruby, TypeScript, shell), markup (HTML, Markdown, LaTeX, JSON), and Office formats (Word, PowerPoint). Notably absent are: Excel (`.xlsx`), images (`.png`, `.jpg`), audio, video, and CSV. This makes OpenAI's file search text-document-oriented, in contrast to Mixedbread's multimodal ingestion (images, audio, video). The encoding constraint (`utf-8`/`utf-16`/`ascii`) ensures reliable text extraction but excludes legacy encodings. The dual MIME for Python (`text/x-python` and `text/x-script.python`) reflects different MIME registration conventions.

---

## 20. Pricing, Rate Limits & Expiration

### Vector Store Pricing

| Storage | Cost |
|---|---|
| Up to 1 GB (across all stores) | Free |
| Beyond 1 GB | $0.10 / GB / day |

### Embeddings Pricing

| Model | ~ Pages per Dollar |
|---|---|
| `text-embedding-3-small` | 62,500 |
| `text-embedding-3-large` | 9,615 |
| `text-embedding-ada-002` | 12,500 |

*(~800 tokens per page assumed.)*

### File Search Rate Limits

| Tier | RPM |
|---|---|
| Tier 1 | 100 |
| Tier 2 and 3 | 500 |
| Tier 4 and 5 | 1000 |

### Vector Store File Rate Limits

| Scope | Limit |
|---|---|
| Per vector store (files + file_batches endpoints) | 300 RPM |

### Expiration Policies

Set `expires_after` on vector stores to auto-expire and stop billing:

```python
client.vector_stores.update(
    vector_store_id="vs_123",
    expires_after={"anchor": "last_active_at", "days": 7}
)
```

| Field | Description |
|---|---|
| `anchor` | Expiration anchor (e.g. `"last_active_at"`) |
| `days` | Days after anchor before expiration |

### Analysis

The 1 GB free tier for vector store storage is generous for prototyping and small knowledge bases. The $0.10/GB/day pricing means a 10 GB knowledge base costs ~$1/day — the `expires_after` policy with `last_active_at` anchoring is the key cost-control for ephemeral/dev stores. The tiered rate limits (100–1000 RPM) scale with account tenure/usage. The per-vector-store 300 RPM limit for file ingestion is the bottleneck for large corpora — batch operations help by reducing request count. Embeddings pricing is per-token, making `text-embedding-3-small` extremely cost-effective (62,500 pages/dollar). The `last_active_at` anchor means a store resets its expiry clock on every search, keeping active knowledge bases alive automatically.

---

## 21. Capability Summary & Cross-Reference

### Capability → API Map

| Capability | API / Endpoint | SDK Method |
|---|---|---|
| Upload raw file | `POST /v1/files` | `client.files.create` |
| Create vector store | `POST /v1/vector_stores` | `client.vector_stores.create` |
| Add file to store | `POST /v1/vector_stores/{id}/files` | `client.vector_stores.files.create` / `create_and_poll` |
| Upload + add file | `POST /v1/vector_stores/{id}/files` | `client.vector_stores.files.upload` / `upload_and_poll` |
| Batch add files | `POST /v1/vector_stores/{id}/file_batches` | `client.vector_stores.file_batches.create` / `create_and_poll` |
| Direct semantic search | `POST /v1/vector_stores/{id}/search` | `client.vector_stores.search` |
| File search (hosted tool) | `POST /v1/responses` (tools) | `client.responses.create` |
| Generate embeddings | `POST /v1/embeddings` | `client.embeddings.create` |
| Attribute filtering | (search/filter param) | `attribute_filter` / `filters` |
| Hybrid search | (search param) | `ranking_options.hybrid_search` |
| Query rewriting | (search param) | `rewrite_query=true` |
| Dimension reduction | (embeddings param) | `dimensions=N` |

### Retrieval Mode Comparison

| Mode | API | Who Controls Retrieval | Citations | Cost/Complexity |
|---|---|---|---|---|
| File Search Tool | Responses API | Model (autonomous) | Yes (`file_citation`) | Lowest code, highest abstraction |
| Retrieval API | `vector_stores.search` | Developer (manual) | No (raw chunks) | Medium (you synthesize) |
| Embeddings + Your Vector DB | `embeddings.create` | Developer (full control) | No | Highest (you build everything) |

### Object Relationships

```
Organization
  └── File (raw content, via Files API)
        └── Vector Store (search index, auto chunk + embed)
              ├── expires_after (cost control)
              ├── file_counts (status tracking)
              └── Vector Store File (wrapper)
                    ├── attributes (max 16 keys, 256 chars each)
                    ├── chunking_strategy (static: 100-4096 tokens, overlap)
                    └── Chunks (auto-generated, searchable)
                          │
                          ├──▶ File Search Tool (Responses API)
                          │      └── file_search_call + message with file_citation annotations
                          │
                          └──▶ Retrieval API (vector_stores.search)
                                 └── ranked results (score, content, attributes)
                                        └── (optional) chat.completions for synthesis

Standalone:
  Text ──▶ embeddings.create ──▶ Vector ──▶ Your vector DB
                                           └── cosine similarity / clustering / classification / RAG
```

### Key Design Principles

1. **Three-tier abstraction** — File search tool (fully managed) → Retrieval API (search control) → Embeddings API (full control). Each tier trades convenience for flexibility.
2. **Automatic pipeline with overrides** — Vector stores auto-chunk and embed; `chunking_strategy` and `dimensions` let you override defaults.
3. **Unified filter language** — Comparison (`eq`/`ne`/`gt`/`gte`/`lt`/`lte`/`in`/`nin`) and compound (`and`/`or`) filters work across both File Search tool and Retrieval API.
4. **Hybrid search by default** — Semantic + keyword fusion with tunable weights, not pure embedding similarity.
5. **MRL-trained embeddings** — Dimension shortening at API time enables quality-cost tradeoffs without retraining or losing concept properties.
6. **Citation-first RAG** — File search returns `file_citation` annotations with character offsets, designed for verifiable, hallucination-resistant answers.
7. **Cost-aware defaults** — Expiration policies, free tier (1 GB), and per-token pricing with a cost-effective small model (62,500 pages/dollar).

---

*Sources: [File Search Guide](https://developers.openai.com/api/docs/guides/tools-file-search), [Embeddings Guide](https://developers.openai.com/api/docs/guides/embeddings), [Retrieval Guide](https://developers.openai.com/api/docs/guides/retrieval), [API Reference](https://developers.openai.com/api/reference/overview). Last reviewed: July 2026.*