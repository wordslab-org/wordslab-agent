# Google Gemini API Analysis — Document Processing & File Search

> **Base URL:** `https://generativelanguage.googleapis.com` | **Docs:** `https://ai.google.dev/gemini-api/docs/document-processing` & `https://ai.google.dev/gemini-api/docs/file-search`
> **Auth:** API key (`GEMINI_API_KEY` / `x-goog-api-key` header)
> **Description:** Two complementary document capabilities in the Gemini API. **Document understanding** uses native multimodal vision to process PDFs (and text files) inline or via the Files API, understanding entire document contexts including charts, layout, and formatting. **File Search** is a managed RAG pipeline that chunks, embeds, indexes, and semantically retrieves from your documents — including multimodal (text + image) search — with citations, metadata filtering, and structured output support.

---

## Table of Contents

1. [Platform Overview & Core Concepts](#1-platform-overview--core-concepts)
2. [Authentication & API Keys](#2-authentication--api-keys)
3. [Document Understanding — Inline PDF Processing](#3-document-understanding--inline-pdf-processing)
4. [Document Understanding — Files API Upload](#4-document-understanding--files-api-upload)
5. [Document Understanding — Multiple PDFs](#5-document-understanding--multiple-pdfs)
6. [Document Understanding — Technical Details & Limits](#6-document-understanding--technical-details--limits)
7. [File Search — Overview & How It Works](#7-file-search--overview--how-it-works)
8. [File Search Stores — Container Management](#8-file-search-stores--container-management)
9. [File Search — Direct Upload to Store](#9-file-search--direct-upload-to-store)
10. [File Search — Importing Files](#10-file-search--importing-files)
11. [File Search — Chunking Configuration](#11-file-search--chunking-configuration)
12. [File Search Documents — Per-Document Management](#12-file-search-documents--per-document-management)
13. [File Search — File Metadata & Filtering](#13-file-search--file-metadata--filtering)
14. [File Search — Multimodal Search](#14-file-search--multimodal-search)
15. [File Search — Citations & Source Tracking](#15-file-search--citations--source-tracking)
16. [File Search — Structured Output](#16-file-search--structured-output)
17. [File Search — Querying with Interactions](#17-file-search--querying-with-interactions)
18. [File Search — Supported Models & File Types](#18-file-search--supported-models--file-types)
19. [Capability Summary & Cross-Reference](#19-capability-summary--cross-reference)

---

## 1. Platform Overview & Core Concepts

### Main Concepts

The Gemini API exposes two distinct but complementary document-processing capabilities:

- **Document Understanding** — Gemini models process PDFs using **native vision**, understanding entire document contexts (charts, tables, layout, formatting) beyond simple text extraction. Documents are passed as `input` parts to `interactions.create`, either inline (base64) or via the Files API (URI reference).
- **File Search** — A managed **Retrieval Augmented Generation (RAG)** pipeline. Documents are imported into a **FileSearchStore**, where they are automatically chunked, embedded, and indexed. At query time, semantic search retrieves relevant chunks to ground the model's response.
- **FileSearchStore** — A persistent container for document embeddings. Raw files expire after 48 hours, but embeddings persist indefinitely until manually deleted. Store names are globally scoped.
- **Interaction** — The unified entry point for model calls (`interactions.create`). Accepts `input` (text/document parts) and optional `tools` (including `file_search`). Returns `steps` containing `model_output` content blocks with text, annotations, and citations.
- **Embedding Models** — `gemini-embedding-001` (text-only) and `gemini-embedding-2` (multimodal: text + images). The embedding model is configured per store at creation time.

### End-to-End Flow

**Document Understanding:**
1. Upload a PDF (inline base64 or via Files API) → pass as `document` part in `interactions.create` → model processes with vision → returns text response.

**File Search:**
1. Create a **FileSearchStore** (with embedding model).
2. Upload or import files → automatic chunking + embedding + indexing.
3. Query via `interactions.create` with `file_search` tool → semantic retrieval → grounded response with citations.

### Key Differentiators

- **Native vision for PDFs** — Gemini "sees" document pages as images, preserving layout, charts, and formatting context that text extraction loses.
- **Managed RAG** — No need to manage chunking, embedding models, or vector databases; File Search handles the entire pipeline.
- **Multimodal search** — `gemini-embedding-2` enables embedding and retrieving images alongside text in a single store.
- **Citations with page numbers** — Responses include `file_citation` annotations with source file name, page number, and media ID.
- **Metadata filtering** — Custom key-value metadata on files enables filtered search at query time.
- **Cost model** — File storage and query-time embedding generation are free; you pay for initial indexing embeddings and standard model token costs.

---

## 2. Authentication & API Keys

### Main Concepts

- **API Key Auth** — All endpoints use `GEMINI_API_KEY`, passed either as a query parameter (`?key=$GEMINI_API_KEY`) or as the `x-goog-api-key` header.
- **SDK Clients** — `genai.Client()` (Python) and `new GoogleGenAI({})` (JavaScript) auto-resolve the key from the environment.

### Analysis

A single, flat API-key model with no documented workspace scoping or key roles. The key is a global credential for the Gemini API. Both header and query-param auth are supported, giving flexibility for different client environments (server-side vs. URL-based curl).

---

## 3. Document Understanding — Inline PDF Processing

### Main Concepts

- **Inline Data** — PDF bytes are base64-encoded and passed directly in the request body as a `document` part.
- **Best For** — Smaller documents or temporary processing where the file doesn't need to be referenced in subsequent requests.
- **MIME Type** — `application/pdf` must be specified on the document part.

### API Function & Parameters

**`interactions.create`** (inline document)

| Parameter | Type | Required | Description |
|---|---|---|---|
| `model` | string | yes | Model ID (e.g. `gemini-3.5-flash`) |
| `input` | array | yes | List of content parts: `document` part (with `data` + `mime_type`) and `text` part (the prompt) |

**Document part structure (inline):**

```json
{
  "type": "document",
  "data": "<base64-encoded PDF bytes>",
  "mime_type": "application/pdf"
}
```

### Code Pattern (Python)

```python
from google import genai
import base64

client = genai.Client()
with open('path/to/document.pdf', 'rb') as f:
    pdf_bytes = f.read()

interaction = client.interactions.create(
    model="gemini-3.5-flash",
    input=[
        {"type": "document", "data": base64.b64encode(pdf_bytes).decode('utf-8'), "mime_type": "application/pdf"},
        {"type": "text", "text": "Summarize this document"}
    ]
)
print(interaction.output_text)
```

### Analysis

Inline processing is the simplest path — no file management, no async polling. The tradeoff is bandwidth and latency: every request re-sends the full PDF bytes. For one-off processing or small documents, this is optimal. For larger files or multi-turn interactions, the Files API (§4) is recommended.

---

## 4. Document Understanding — Files API Upload

### Main Concepts

- **Decoupled Upload** — Files are uploaded once via the Files API, then referenced by URI in subsequent `interactions.create` calls.
- **Benefits** — Improved latency, reduced bandwidth (upload once, reference many times), supports multi-turn interactions.
- **File States** — `PROCESSING` → `ACTIVE` (or `FAILED`). Poll `files.get` until state is no longer `PROCESSING`.
- **Resumable Upload (REST)** — Two-step: initiate upload (get upload URL) → upload bytes to the URL.

### API Functions & Parameters

**`files.upload`**

| Parameter | Type | Required | Description |
|---|---|---|---|
| `file` | string/path/Blob | yes | File path or binary content |
| `config.mime_type` | string | no | MIME type (e.g. `application/pdf`) |
| `config.display_name` | string | no | Human-readable name |

**`files.get`**

| Parameter | Type | Required | Description |
|---|---|---|---|
| `name` | string | yes | File resource name (e.g. `files/abc123`) |

**Document part structure (URI reference):**

```json
{
  "type": "document",
  "uri": "https://generativelanguage.googleapis.com/.../files/abc123",
  "mime_type": "application/pdf"
}
```

### File Response Fields

| Field | Type | Description |
|---|---|---|
| `name` | string | Unique file resource name |
| `uri` | string | URI for use in model requests |
| `mime_type` | string | Detected/specified MIME type |
| `state` | string | `PROCESSING` / `ACTIVE` / `FAILED` |
| `display_name` | string | Optional human-readable name |

### REST Upload Flow (Resumable)

1. **Initiate** — `POST /upload/v1beta/files` with `X-Goog-Upload-Protocol: resumable`, `X-Goog-Upload-Command: start`, content length/type headers → receive upload URL in `X-Goog-Upload-URL` response header.
2. **Upload** — `PUT <upload_url>` with `X-Goog-Upload-Command: upload, finalize` and binary data → receive file metadata JSON (`file.uri`, `file.name`).
3. **Poll** — `GET /v1beta/files/{name}` until `state` is `ACTIVE`.
4. **Use** — Reference `file.uri` in `interactions.create` input.

### Analysis

The Files API is the production-grade path for document understanding. The resumable upload protocol (Google's standard) supports large files and recovery from interruptions. The polling pattern (`PROCESSING` → `ACTIVE`) is a typical async ingestion model. The key design choice is that only `name` (and `uri`) are unique identifiers — `display_name` is not guaranteed unique, so applications must track the returned `name`/`uri`.

---

## 5. Document Understanding — Multiple PDFs

### Main Concepts

- **Multi-Document Processing** — Up to 1000 pages across multiple PDFs in a single request, as long as combined size + prompt fit within the model's context window.
- **Same Input Shape** — Multiple `document` parts in the `input` array, each referencing a different uploaded file.
- **Use Case** — Cross-document comparison (e.g. "What is the difference between benchmarks in these two papers?").

### Code Pattern (Python)

```python
interaction = client.interactions.create(
    model="gemini-3.5-flash",
    input=[
        {"type": "document", "uri": sample_pdf_1.uri, "mime_type": sample_pdf_1.mime_type},
        {"type": "document", "uri": sample_pdf_2.uri, "mime_type": sample_pdf_2.mime_type},
        {"type": "text", "text": "What is the difference between each of the main benchmarks between these two papers?"}
    ]
)
```

### Analysis

Multi-PDF support is a natural extension of the parts-based input model — each document is just another part. The 1000-page ceiling and context-window constraint are the only limits. This enables powerful cross-document reasoning (comparison, synthesis, contradiction detection) without any special API surface.

---

## 6. Document Understanding — Technical Details & Limits

### Limits

| Constraint | Value |
|---|---|
| Max PDF size | 50 MB |
| Max pages per request | 1000 pages (combined across multiple PDFs) |
| Tokens per page | 258 tokens (each document page) |
| Max page resolution | 3072 × 3072 px (larger pages scaled down, preserving aspect ratio) |
| Min page resolution | 768 × 768 px (smaller pages scaled up) |

### Gemini 3 Media Resolution

- **`media_resolution` parameter** — Granular per-part control over vision processing resolution: `low`, `medium`, or `high`.
- **`usage_metadata`** — Tracks whether a part was processed as `IMAGE` or `DOCUMENT`, reflecting the resolution-based processing mode.
- See the [Media resolution guide](https://ai.google.dev/gemini-api/docs/media-resolution) for details.

### Document Types Beyond PDF

- Other MIME types (TXT, Markdown, HTML, XML, etc.) can be passed but are treated as **pure text** — no vision understanding.
- Charts, diagrams, HTML tags, Markdown formatting, etc. are **lost** for non-PDF types.
- Only PDFs receive meaningful document vision processing.

### Analysis

The 258-tokens-per-page metric is important for cost and context-window planning. The resolution scaling (768–3072px) is an automatic optimization — larger pages are downscaled, smaller pages upscaled — with no cost difference for lower resolutions beyond bandwidth. The Gemini 3 `media_resolution` parameter adds fine-grained control for cost-sensitive applications: `low` resolution for quick scans, `high` for detailed chart/diagram analysis. The non-PDF limitation is significant: for rich document understanding, convert to PDF first.

---

## 7. File Search — Overview & How It Works

### Main Concepts

- **Semantic Search** — Unlike keyword-based search, File Search understands meaning and context by converting documents and queries into numerical **embeddings** and finding the most similar document chunks.
- **Automatic Pipeline** — Import a file → it's chunked → embedded → indexed → stored in the FileSearchStore. No manual chunking or embedding management.
- **Persistent Embeddings** — Embeddings have no TTL; they persist until manually deleted or the model is deprecated. Raw `File` objects are deleted after 48 hours, but store data is indefinite.
- **Billing Model** — File storage and query-time embedding generation are **free**. You pay for: (1) embedding creation at index time, (2) standard Gemini model input/output tokens at query time.

### Process Flow

```
Documents
    │
    ▼
Embedding Model (gemini-embedding-001 / gemini-embedding-2)
    │
    ├── (uploadToFileSearchStore) ── bypasses File storage
    │
    └── (Files API → importFile) ── goes through File storage first
    │
    ▼
Chunking → Embedding → Indexing
    │
    ▼
FileSearchStore (persistent embeddings)
    │
    ▼
Query (interactions.create with file_search tool)
    │
    ▼
Semantic Search → Retrieved Chunks → Grounded Model Response + Citations
```

### Analysis

File Search is a fully managed RAG pipeline. The two ingestion paths (direct upload vs. Files API + import) offer flexibility: direct upload is simpler (one step), while the Files API path allows reuse of the uploaded file for other purposes (e.g. also passing it inline for document understanding). The "free storage and query-time embedding" billing is notable — it makes the marginal cost of additional queries essentially zero (only model token costs apply), incentivizing high query volumes over a fixed indexed corpus.

---

## 8. File Search Stores — Container Management

### Main Concepts

- **FileSearchStore** — A persistent container for document embeddings. Multiple stores can be created to organize documents.
- **Globally Scoped Names** — Store names are globally unique across all users.
- **Embedding Model Selection** — Set at creation time; `gemini-embedding-2` for multimodal (text + images), `gemini-embedding-001` for text-only.
- **Persistence** — Store data persists indefinitely until manually deleted. Use `force: true` on delete to cascade-delete all contents.

### API Functions & Parameters

| Function | Method | Purpose | Key Parameters |
|---|---|---|---|
| `file_search_stores.create` | `POST /v1beta/fileSearchStores` | Create a store | `config.display_name`, `config.embedding_model` |
| `file_search_stores.list` | `GET /v1beta/fileSearchStores` | List all stores | — |
| `file_search_stores.get` | `GET /v1beta/fileSearchStores/{name}` | Retrieve a store | `name` |
| `file_search_stores.delete` | `DELETE /v1beta/fileSearchStores/{name}` | Delete store + contents | `name`, `config.force` (bool) |

### Create Parameters

| Parameter | Type | Required | Description |
|---|---|---|---|
| `display_name` / `displayName` | string | yes | Human-readable store name |
| `embedding_model` / `embeddingModel` | string | yes | Embedding model ID (e.g. `models/gemini-embedding-2`) |

### Store Object

| Field | Type | Description |
|---|---|---|
| `name` | string | Globally unique store resource name (e.g. `fileSearchStores/my-store-123`) |
| `display_name` | string | Human-readable name |
| `embedding_model` | string | Configured embedding model |

### Analysis

The store management API is a standard CRUD interface. The globally-scoped naming is an important constraint — store names must be unique across all users, so applications should use namespaced or UUID-suffixed names. The `force: true` delete option is a cascade delete (store + all documents + all embeddings), which is convenient but destructive. The embedding model is immutable after creation (not shown as updatable), so choosing text-only vs. multimodal at creation time is a one-way decision.

---

## 9. File Search — Direct Upload to Store

### Main Concepts

- **One-Step Ingestion** — `uploadToFileSearchStore` simultaneously uploads a file and imports it into a store, bypassing the File storage layer.
- **Temporary File Object** — A temporary `File` object is created internally but is deleted after 48 hours. The indexed data in the store persists.
- **Long-Running Operation** — Returns an `operation` object; poll `operations.get` until `operation.done` is `true`.

### API Function & Parameters

**`file_search_stores.upload_to_file_search_store`**

| Parameter | Type | Required | Description |
|---|---|---|---|
| `file` | string/path | yes | File to upload |
| `file_search_store_name` | string | yes | Target store name |
| `config.display_name` | string | no | Display name for the file in the store |
| `config.chunking_config` | object | no | Custom chunking strategy (see §11) |

### REST Flow (Resumable Upload to Store)

1. **Initiate** — `POST /upload/v1beta/fileSearchStores/{store}:uploadToFileSearchStore` with resumable upload headers + JSON metadata (`displayName`, optional `chunkingConfig`).
2. **Upload** — `PUT <upload_url>` with binary data → returns operation.
3. **Poll** — `operations.get` until `done: true`.

### Analysis

Direct upload is the simplest ingestion path — one call handles upload + chunking + embedding + indexing. The long-running operation pattern (poll until `done`) is standard for async processing. The 48-hour temporary file TTL is an implementation detail: the file object is an intermediate artifact, while the store's embeddings are the durable product. For use cases where you also need the raw file for other purposes (e.g. inline document understanding), use the import path (§10) instead.

---

## 10. File Search — Importing Files

### Main Concepts

- **Two-Step Ingestion** — First upload via the Files API (`files.upload`), then import into a store (`file_search_stores.import_file`).
- **File Reuse** — The uploaded file can be referenced by multiple stores or used for inline document understanding.
- **Long-Running Operation** — Same polling pattern as direct upload.

### API Function & Parameters

**`file_search_stores.import_file`**

| Parameter | Type | Required | Description |
|---|---|---|---|
| `file_search_store_name` | string | yes | Target store name |
| `file_name` | string | yes | Name of the uploaded file (from `files.upload` response) |
| `config.custom_metadata` | array | no | Key-value metadata pairs (see §13) |

### Code Pattern (Python)

```python
sample_file = client.files.upload(file='sample.txt', config={'display_name': 'display_file_name'})
operation = client.file_search_stores.import_file(
    file_search_store_name=file_search_store.name,
    file_name=sample_file.name
)
while not operation.done:
    time.sleep(5)
    operation = client.operations.get(operation)
```

### Analysis

The import path decouples file storage from indexing. This is useful when: (1) the same file needs to be in multiple stores, (2) the file is also used for inline document understanding, or (3) you want to manage file lifecycle separately from search indexing. The tradeoff is an extra API call and managing the intermediate file object (which expires in 48 hours). The `custom_metadata` parameter (only available on import, not direct upload in the documented examples) is a key advantage of this path.

---

## 11. File Search — Chunking Configuration

### Main Concepts

- **Automatic Chunking** — By default, imported files are automatically chunked, embedded, and indexed.
- **Custom Chunking** — `chunking_config` gives control over chunk size and overlap.
- **White Space Config** — The documented chunking strategy uses whitespace-based splitting with configurable token limits.

### Configuration Parameters

**`config.chunking_config.white_space_config`**

| Parameter | Type | Description |
|---|---|---|
| `max_tokens_per_chunk` / `maxTokensPerChunk` | integer | Maximum tokens per chunk |
| `max_overlap_tokens` / `maxOverlapTokens` | integer | Maximum overlapping tokens between adjacent chunks |

### Code Pattern (Python)

```python
operation = client.file_search_stores.upload_to_file_search_store(
    file_search_store_name=file_search_store.name,
    file='sample.txt',
    config={
        'chunking_config': {
            'white_space_config': {
                'max_tokens_per_chunk': 200,
                'max_overlap_tokens': 20
            }
        }
    }
)
```

### Analysis

Chunking configuration is the primary lever for retrieval quality. Smaller chunks with overlap provide finer-grained retrieval (more precise matches) but may fragment context. Larger chunks preserve more context but may dilute relevance signals. The `max_overlap_tokens` parameter prevents information loss at chunk boundaries — critical for documents where important sentences span chunk edges. The documented strategy is "white_space_config" (whitespace-based splitting), suggesting a simple token/whitespace chunker; more sophisticated strategies (semantic chunking, layout-aware chunking for PDFs) may be available but are not documented on this page.

---

## 12. File Search Documents — Per-Document Management

### Main Concepts

- **Document-Level Management** — After files are imported into a store, they become "documents" that can be listed, retrieved, and deleted individually.
- **Document Name Format** — `fileSearchStores/{store}/documents/{document}`.

### API Functions & Parameters

| Function | Method | Purpose | Key Parameters |
|---|---|---|---|
| `file_search_stores.documents.list` | `GET .../documents` | List documents in a store | `parent` (store name) |
| `file_search_stores.documents.get` | `GET .../documents/{name}` | Retrieve a document | `name` |
| `file_search_stores.documents.delete` | `DELETE .../documents/{name}` | Delete a document + its embeddings | `name`, `config.force` (bool) |

### Code Pattern (Python)

```python
for document_in_store in client.file_search_stores.documents.list(parent='fileSearchStores/myfilesearchstore123'):
    print(document_in_store)

client.file_search_stores.documents.delete(
    name='fileSearchStores/myfilesearchstore123/documents/sampletxt123',
    config={'force': True}
)
```

### Analysis

The documents sub-API provides granular management over indexed content. This is essential for maintaining a living corpus — removing outdated documents, inspecting what's indexed, or debugging ingestion. The `force: true` delete cascades to the document's embeddings. The hierarchical naming (`stores/{store}/documents/{doc}`) follows Google's standard resource model.

---

## 13. File Search — File Metadata & Filtering

### Main Concepts

- **Custom Metadata** — Key-value pairs attached to files at import time. Supports `string_value` and `numeric_value` types.
- **Metadata Filtering at Query Time** — `metadata_filter` on the `file_search` tool restricts retrieval to documents matching a filter expression.
- **Filter Syntax** — Follows [Google AIP-160](https://google.aip.dev/160) list filter syntax (e.g. `author="Robert Graves"`).

### Metadata Attachment (Import Time)

```python
operation = client.file_search_stores.import_file(
    file_search_store_name=file_search_store.name,
    file_name=sample_file.name,
    config={
        'custom_metadata': [
            {"key": "author", "string_value": "Robert Graves"},
            {"key": "year", "numeric_value": 1934}
        ]
    }
)
```

### Metadata Field Types

| Field | Type | Description |
|---|---|---|
| `key` | string | Metadata key |
| `string_value` | string | String value (for text metadata) |
| `numeric_value` | number | Numeric value (for numeric metadata, supports range filters) |

### Query-Time Filtering

```python
interaction = client.interactions.create(
    model="gemini-3.5-flash",
    input="Tell me about the book 'I, Claudius'",
    tools=[{
        "type": "file_search",
        "file_search_store_names": [file_search_store.name],
        "metadata_filter": 'author="Robert Graves"',
    }]
)
```

### Tool Configuration with Filter

| Parameter | Type | Description |
|---|---|---|
| `metadata_filter` / `metadataFilter` | string | AIP-160 filter expression (e.g. `author="Robert Graves"`, `year>=1934`) |

### Analysis

Metadata filtering is the mechanism for scoped retrieval within a large corpus. The AIP-160 syntax supports equality, comparison, and logical operators, enabling expressions like `author="Robert Graves" AND year>=1934`. The two value types (string and numeric) cover the common cases for filtering by categorical and range attributes. Metadata is set only at import time (not shown as updatable post-ingestion), so applications should attach metadata proactively. The returned citations also include the custom metadata, allowing downstream logic to access the source context.

---

## 14. File Search — Multimodal Search

### Main Concepts

- **Multimodal Embeddings** — `gemini-embedding-2` natively embeds both text and images in a unified vector space.
- **Image Search** — Upload image files to a multimodal store; queries can semantically retrieve relevant images alongside text.
- **Store Configuration** — The embedding model must be set to `models/gemini-embedding-2` at store creation time.

### Configuration

```python
store = client.file_search_stores.create(
    config={
        "display_name": "Multimodal Catalog",
        "embedding_model": "models/gemini-embedding-2",
    }
)
```

### Image Upload

After creating a multimodal store, images are uploaded using the same APIs (direct upload or import) described in §9 and §10. Image file requirements are referenced but not detailed on the page.

### Analysis

Multimodal search extends RAG beyond text — enabling applications like visual product catalogs, diagram retrieval, or image-based knowledge bases. The unified embedding space means a text query can retrieve relevant images and vice versa, all within the same store. The embedding model choice at store creation is the gating decision: text-only stores use `gemini-embedding-001`, multimodal stores use `gemini-embedding-2`. Since the model is immutable after creation, multimodal capability must be decided upfront.

---

## 15. File Search — Citations & Source Tracking

### Main Concepts

- **File Citations** — Model responses include `file_citation` annotations specifying which documents were used to generate the answer.
- **Page Numbers** — For paginated documents (PDFs), citations include `page_number` indicating where the information was found.
- **Media Citations** — For image chunks, citations include a `media_id` that can be used to download the exact referenced image.
- **Custom Metadata in Citations** — Citations carry the custom metadata attached to the source file, passing context to application logic.
- **Persistent Media IDs** — `media_id` is stable across multiple search calls, enabling caching and reliable retrieval.

### Citation Annotation Structure

| Field | Type | Description |
|---|---|---|
| `type` | string | `"file_citation"` |
| `file_name` / `fileName` | string | Source file name |
| `source` | string | Source reference |
| `page_number` / `pageNumber` | integer | Page number (for PDFs) |
| `media_id` / `mediaId` | string | Media blob ID (for image chunks) |
| `custom_metadata` / `customMetadata` | array | Custom metadata from source file |

### Accessing Citations (Python)

```python
for step in interaction.steps:
    if step.type == "model_output":
        for content in step.content:
            if content.type == "text" and content.annotations:
                for annotation in content.annotations:
                    if annotation.type == "file_citation":
                        print(f"File: {annotation.file_name}, Page: {annotation.page_number}")
```

### Media Download

```python
blob_content = client.file_search_stores.download_media(media_id=annotation.media_id)
```

**REST:** `GET https://generativelanguage.googleapis.com/v1/fileSearchStores/{store}/media/{blobId}`

### Citation Response Example (REST)

```json
{
  "steps": [
    {
      "type": "model_output",
      "content": [
        {
          "type": "text",
          "text": "...",
          "annotations": [
            {
              "type": "file_citation",
              "file_name": "document.pdf",
              "page_number": 1,
              "source": "...",
              "media_id": "fileSearchStores/my-store-123/media/BlobId-456",
              "custom_metadata": [
                {"key": "author", "string_value": "Robert Graves"},
                {"key": "year", "numeric_value": 1934}
              ]
            }
          ]
        }
      ]
    }
  ]
}
```

### Analysis

The citation system is designed for verifiable, hallucination-resistant RAG. Three citation dimensions — file name, page number, and media ID — cover text, paginated, and image sources respectively. The persistent `media_id` is particularly valuable for multimodal applications: it enables image caching and reliable re-retrieval of the exact referenced visual chunk. The inclusion of custom metadata in citations creates a round-trip: metadata set at import time flows through to query-time citations, allowing applications to access source context (URLs, authors, dates) without a separate lookup.

---

## 16. File Search — Structured Output

### Main Concepts

- **Structured Outputs with File Search** — Starting with Gemini 3 models, the `file_search` tool can be combined with `response_format` to produce structured (JSON) output grounded in retrieved documents.
- **Schema-Based** — Define a JSON schema (or Pydantic model / Zod schema) and the model returns JSON conforming to it.
- **Response Format** — `response_format` with `type: "text"`, `mime_type: "application/json"`, and a `schema` object.

### API Function & Parameters

**`interactions.create`** with structured output + file search

| Parameter | Type | Required | Description |
|---|---|---|---|
| `model` | string | yes | Gemini 3+ model (e.g. `gemini-3.5-flash`) |
| `input` | string/array | yes | Query |
| `tools` | array | yes | File search tool config |
| `response_format.type` | string | yes | `"text"` |
| `response_format.mime_type` | string | yes | `"application/json"` |
| `response_format.schema` | object | yes | JSON schema defining the output structure |

### Code Pattern (Python with Pydantic)

```python
from pydantic import BaseModel, Field

class Money(BaseModel):
    amount: str = Field(description="The numerical part of the amount.")
    currency: str = Field(description="The currency of amount.")

interaction = client.interactions.create(
    model="gemini-3.5-flash",
    input="What is the minimum hourly wage in Tokyo right now?",
    tools=[{"type": "file_search", "file_search_store_names": [file_search_store.name]}],
    response_format={
        "type": "text",
        "mime_type": "application/json",
        "schema": Money.model_json_schema()
    },
)
result = Money.model_validate_json(interaction.output_text)
```

### Analysis

Structured output + file search combines grounded retrieval with machine-readable responses — ideal for building data extraction pipelines over document corpora. The schema is a standard JSON Schema, compatible with Pydantic (Python) and Zod (JavaScript) for type-safe parsing. The Gemini 3+ requirement means this is a newer capability not available on older models. The `output_text` contains the JSON string, which can be validated against the schema for type-safe consumption.

---

## 17. File Search — Querying with Interactions

### Main Concepts

- **Unified Entry Point** — File Search queries go through `interactions.create` with a `file_search` tool configuration.
- **Tool Configuration** — The `file_search` tool specifies which store(s) to search and optional metadata filters.
- **Response Structure** — Interactions return `steps`, each with a `type` (`model_output`) and `content` blocks (text + annotations).

### API Function & Parameters

**`interactions.create`** (file search query)

| Parameter | Type | Required | Description |
|---|---|---|---|
| `model` | string | yes | Model ID (e.g. `gemini-3.5-flash`) |
| `input` | string/array | yes | Query text or content parts |
| `tools` | array | yes | Tool configurations (see below) |
| `response_format` | object | no | Structured output config (see §16) |

**`file_search` tool configuration:**

| Parameter | Type | Required | Description |
|---|---|---|---|
| `type` | string | yes | `"file_search"` |
| `file_search_store_names` / `fileSearchStoreNames` | array | yes | List of FileSearchStore names to search |
| `metadata_filter` / `metadataFilter` | string | no | AIP-160 filter expression (see §13) |

### Response Structure

| Field | Type | Description |
|---|---|---|
| `steps` | array | Ordered list of processing steps |
| `steps[].type` | string | Step type (e.g. `model_output`) |
| `steps[].content` | array | Content blocks |
| `steps[].content[].type` | string | Block type (e.g. `text`) |
| `steps[].content[].text` | string | Generated text |
| `steps[].content[].annotations` | array | Citations (see §15) |
| `output_text` | string | Convenience accessor for the final text output |

### Code Pattern (Python)

```python
interaction = client.interactions.create(
    model="gemini-3.5-flash",
    input="Can you tell me about [insert question]",
    tools=[{
        "type": "file_search",
        "file_search_store_names": [file_search_store.name]
    }]
)
for step in interaction.steps:
    if step.type == "model_output":
        for content_block in step.content:
            if content_block.type == "text":
                print(content_block.text)
                if content_block.annotations:
                    for annotation in content_block.annotations:
                        if annotation.type == "file_citation":
                            print(f" - {annotation.file_name}: {annotation.source}")
```

### Analysis

The `interactions.create` API is the unified interface for all Gemini model calls, with File Search being one of several possible tools. The multi-store support (`file_search_store_names` is an array) enables federated search across multiple stores in a single query. The steps-based response model is more structured than a simple text response — it separates the processing pipeline into discrete steps, allowing applications to inspect intermediate results. The `output_text` convenience accessor simplifies the common case where only the final text is needed.

---

## 18. File Search — Supported Models & File Types

### Supported Models

| Model | File Search |
|---|---|
| Gemini 3.5 Flash | ✔️ |
| Gemini 3.1 Pro Preview | ✔️ |
| Gemini 3.1 Flash-Lite | ✔️ |
| Gemini 3 Flash Preview | ✔️ |

### Supported File Types

File Search supports a wide range of MIME types across two categories:

**Application types (highlights):**
- `application/pdf`, `application/json`, `application/xml`, `application/zip`
- `application/msword`, `application/vnd.ms-excel`
- `application/vnd.openxmlformats-officedocument.*` (Word, Excel, PowerPoint)
- `application/vnd.oasis.opendocument.text`
- Code: `application/typescript`, `application/x-php`, `application/x-sh`, `application/x-powershell`, etc.

**Text types (highlights):**
- `text/plain`, `text/html`, `text/markdown`, `text/csv`, `text/tsv`, `text/rtf`
- `text/css`, `text/javascript`, `text/xml`, `text/yaml`
- Code: `text/x-python`, `text/x-java`, `text/x-go`, `text/x-rust`, `text/x-swift`, `text/x-kotlin`, `text/x-scala`, etc.
- `text/x-diff`, `text/x-r-markdown`, `text/vnd.graphviz`

### Analysis

The file type support is extensive — covering documents, spreadsheets, presentations, code in dozens of languages, and structured data formats. The breadth rivals dedicated document processing platforms. Notably, PDFs and Office formats are supported for both document understanding (vision) and file search (text extraction + embedding). For non-PDF types in document understanding, only text is extracted (no vision), but in File Search all supported types are properly chunked and embedded. The code file support (Python, Java, Go, Rust, TypeScript, etc.) makes File Search viable for codebase search applications.

---

## 19. Capability Summary & Cross-Reference

### Capability → API Map

| Capability | API Function | Key Parameters |
|---|---|---|
| Inline PDF processing | `interactions.create` | `input` with `document` part (`data` + `mime_type`) |
| Upload file for processing | `files.upload` | `file`, `config.mime_type`, `config.display_name` |
| Check file status | `files.get` | `name` |
| Multi-PDF processing | `interactions.create` | Multiple `document` parts in `input` |
| Create file search store | `file_search_stores.create` | `config.display_name`, `config.embedding_model` |
| Direct upload to store | `file_search_stores.upload_to_file_search_store` | `file`, `file_search_store_name`, `config.chunking_config` |
| Import file to store | `file_search_stores.import_file` | `file_search_store_name`, `file_name`, `config.custom_metadata` |
| Custom chunking | (upload/import `config.chunking_config`) | `white_space_config.max_tokens_per_chunk`, `max_overlap_tokens` |
| Manage store documents | `file_search_stores.documents.{list,get,delete}` | `parent`/`name`, `config.force` |
| Store management | `file_search_stores.{list,get,delete}` | `name`, `config.force` |
| File search query | `interactions.create` with `file_search` tool | `tools[].file_search_store_names`, `metadata_filter` |
| Metadata filtering | (tool `metadata_filter`) | AIP-160 filter expression |
| Citations | (response `annotations`) | `file_citation` with `file_name`, `page_number`, `media_id` |
| Media download | `file_search_stores.download_media` | `media_id` |
| Multimodal search | Store with `gemini-embedding-2` | Upload images to multimodal store |
| Structured output | `interactions.create` with `response_format` | `type`, `mime_type`, `schema` |

### Document Understanding vs. File Search

| Dimension | Document Understanding | File Search |
|---|---|---|
| **Purpose** | Process PDFs with vision (summarize, extract, reason) | RAG — retrieve relevant chunks to ground answers |
| **Processing** | Per-request (no persistence) | Persistent index (embeddings stored indefinitely) |
| **File size** | Up to 50 MB / 1000 pages | No documented limit (managed pipeline) |
| **Vision** | Yes — native vision for PDFs | No — text/image embedding only |
| **Citations** | No | Yes — file name, page number, media ID |
| **Metadata filtering** | No | Yes — AIP-160 filter expressions |
| **Multimodal** | PDF vision (charts, layout) | Text + image embedding (`gemini-embedding-2`) |
| **Query model** | Single prompt per request | Semantic search + grounded generation |
| **Best for** | One-off document analysis, multi-doc comparison | Repeated queries over a fixed corpus |
| **Cost** | Per-request tokens | Index-time embeddings + query-time tokens (storage free) |

### Core Object Relationships

```
Gemini API
├── Document Understanding (inline / Files API)
│     └── interactions.create
│           └── input: [document part(s), text part]
│                 └── document: {data (base64) | uri, mime_type}
│
└── File Search (managed RAG)
      └── FileSearchStore (globally scoped, persistent)
            ├── embedding_model: gemini-embedding-001 (text) | gemini-embedding-2 (multimodal)
            ├── Documents (imported files → chunked → embedded → indexed)
            │     ├── custom_metadata: [{key, string_value | numeric_value}]
            │     └── chunking_config: white_space_config {max_tokens_per_chunk, max_overlap_tokens}
            └── Query (interactions.create with file_search tool)
                  ├── file_search_store_names: [store names]
                  ├── metadata_filter: AIP-160 expression
                  └── Response
                        └── steps[].content[].annotations[]
                              └── file_citation: {file_name, page_number, media_id, custom_metadata}
```

### Key Design Principles

1. **Unified interaction model** — Both document understanding and file search use `interactions.create` as the entry point, differing only in `input` parts vs. `tools` configuration.
2. **Two ingestion paths** — Direct upload (one-step, bypasses file storage) vs. Files API + import (two-step, enables file reuse and metadata attachment).
3. **Managed pipeline with configuration** — Automatic chunking/embedding/indexing by default, with `chunking_config` for advanced control over chunk size and overlap.
4. **Citations as first-class output** — Every file search response includes structured citations (file, page, media, metadata) for verifiable grounding.
5. **Metadata round-trip** — Custom metadata set at import time propagates to query-time filtering and citation output, enabling scoped retrieval and source context.
6. **Multimodal via embedding model selection** — Store-level choice between text-only (`gemini-embedding-001`) and multimodal (`gemini-embedding-2`) embeddings, decided at creation time.
7. **Free storage, pay for compute** — Embedding storage and query-time embedding generation are free; costs are indexing embeddings and model tokens only.

---

*Sources: [Document Processing](https://ai.google.dev/gemini-api/docs/document-processing), [File Search](https://ai.google.dev/gemini-api/docs/file-search). Last reviewed: July 2026.*