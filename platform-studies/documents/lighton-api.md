# LightOn API Analysis — Paradigm Platform

> **Base URL:** `https://api.lighton.ai` | **Version:** 3.x (v3) | **Auth:** Bearer token  
> **Description:** Upload a PDF. Search it in milliseconds. Parse any document to clean Markdown. Build knowledge-retrieval pipelines without managing vector databases or OCR models.

---

## Table of Contents

1. [Authentication & API Keys](#1-authentication--api-keys)
2. [Workspaces](#2-workspaces)
3. [Files — Document Ingestion & Management](#3-files--document-ingestion--management)
4. [Tags](#4-tags)
5. [Search — Hybrid Vector + Text Retrieval](#5-search----hybrid-vector--text-retrieval)
6. [Ask — Retrieval-Augmented Generation](#6-ask----retrieval-augmented-generation-rag)
7. [Parse — Document-to-Markdown Conversion](#7-parse----document-to-markdown-conversion)
8. [Extract — Structured Data Extraction](#8-extract----structured-data-extraction)
9. [Facets — Content-Type Classification & Attributes](#9-facets----content-type-classification--attributes)
10. [Models Management](#10-models-management)
11. [Budget Management](#11-budget-management)
12. [Integrations Ecosystem](#12-integrations-ecosystem)

---

## 1. Authentication & API Keys

### Main Concepts

- **Bearer Token Auth** — All endpoints require `Authorization: Bearer <token>` header.
- **Scoped API Keys** — Keys can be scoped to specific workspaces, with granular roles (`owner`, `editor`, `viewer`).
- **Lifecycle Management** — Full CRUD for keys: create, list, retrieve details, update name/scopes, and revoke.

### Endpoints & Parameters

| Endpoint | Method | Purpose | Key Parameters |
|---|---|---|---|
| `/api/v3/api-keys` | `POST` | Create a new API key | Name, workspace scope (optional) |
| `/api/v3/api-keys` | `GET` | List all keys for authenticated user | Pagination params |
| `/api/v3/api-keys/{id}` | `GET` | Retrieve single key with workspace scope | — |
| `/api/v3/api-keys/{id}` | `PATCH` | Update an existing key | Name, scopes |
| `/api/v3/api-keys/{id}` | `DELETE` | Revoke a key (can no longer authenticate) | — |

### Analysis

API keys provide the foundation for service-to-service auth. Workspace-scoped keys enable multi-tenant architectures where each integration (SharePoint sync, third-party app, etc.) receives its own restricted credential set. Keys are company-wide resources but can optionally inherit workspace scope from their parent.

---

## 2. Workspaces

### Main Concepts

- **Isolated Containers** — Access-controlled partitions for documents per team, customer, or tenant.
- **Types** — `shared` (team-accessible) and `personal` (single-user).
- **Datasource Sync** — Alpha support for automated ingestion from external sources (Google Drive, SharePoint).
- **Roles & Storage Tracking** — Owner/Editor/Viewer permissions; per-workspace file count and storage metrics.
- **Scoped API Keys** — Workspace-level keys with separate credential management.

> ⚠️ Workspaces endpoints are marked as ALPHA and subject to breaking changes.

### Endpoints & Parameters

| Endpoint | Method | Purpose | Key Parameters |
|---|---|---|---|
| `/api/v3/workspaces` | `GET` | List all workspaces | Pagination, filters |
| `/api/v3/workspaces` | `POST` | Create a workspace | `name`, `description` |
| `/api/v3/workspaces/{id}` | `GET` | Retrieve single workspace details | — |
| `/api/v3/workspaces/{id}` | `PATCH` | Update workspace name/description | `name`, `description` |
| `/api/v3/workspaces/{id}` | `DELETE` | Delete a workspace (cascades to files) | — |

### Workspace Response Model

- **Core:** `id`, `name`, `workspace_type` (`shared`/`personal`), `document_upload_method`
- **Metadata:** `description`, `created_at`, `updated_at`, `files_count`, `used_storage`
- **Access:** `user_role` (`owner`/`editor`/`viewer`), `scoped_api_keys[]`
- **Multi-language Summaries:** Array of `{language, summary}` pairs (en, fr, es, it, ar, nl, sv, de, ja, zh, ko)
- **Sync Config** (datasource workspaces only): `datasource_type`, `source_name`, `last_status`, `next_import_date`, `filter_criteria`

### Analysis

Workspaces are the foundational scoping unit. Search and Ask queries target workspace_ids directly. They represent a multi-tenant tenant isolation boundary, supporting summary generation in 11 languages for global teams. The sync configuration on datasource workspaces enables set-and-forget ingestion pipelines from connected services.

---

## 3. Files — Document Ingestion & Management

### Main Concepts

- **Universal File Upload** — Accepts PDF, Office (doc/x, pptx, xlsx), OpenDocument (odp, odt), images (png, jpeg), HTML, Markdown, TXT, CSV.
- **Async Processing Pipeline** — Files queue through: `pending` → `converting` → `parsing` → `embedding` → `embedded`. Vision processing runs in parallel (`status_vision`).
- **Idempotent Uploads** — When `external_metadata.external_id` is set, re-uploads return the existing document (200) instead of creating duplicates.
- **Custom Ingestion Pipeline** — Selectable parser version (`v2.2.1`) to control processing behavior.
- **Searchable Indexing** — All uploaded files automatically enter the searchable index within the workspace.

### Endpoints & Parameters

| Endpoint | Method | Purpose | Key Parameters |
|---|---|---|---|
| `/api/v3/files` | `POST` | Upload a file (multipart) | `file`, `workspace_id` (required), `filename`, `title`, `parser`, `tags[]`, `external_metadata` |
| `/api/v3/files` | `GET` | List accessible files (paginated, searchable) | Pagination, `search`, ordering params |
| `/api/v3/files/{id}` | `GET` | Retrieve single file by ID | — |
| `/api/v3/files/{id}` | `PATCH` | Update mutable metadata | `title`, etc. |
| `/api/v3/files/{id}` | `DELETE` | Permanently delete a file + embeddings | — |

#### Batch & Tag Operations

| Endpoint | Method | Purpose | Key Parameters |
|---|---|---|---|
| `/api/v3/files/bulk-delete` | `POST` | Delete multiple files in one request | Array of file IDs |
| `/api/v3/files/{id}/tags` | `POST` | Add tags to a file | Tag IDs list, `auto_assigned= false` for manual assignment |
| `/api/v3/files/{id}/tags/{tag_id}` | `DELETE` | Remove a tag from a file | — |

#### Download

| Endpoint | Method | Purpose | Key Parameters |
|---|---|---|---|
| `/api/v3/files/{id}/download` | `GET` | Download original or rendered version | `purpose` param (`original`, `rendered_pdf`) |

### Upload Request Schema (`FileCreateRequestSerializerV3`)

- **file** `[binary, required]` — Binary file data
- **workspace_id** `[int, required]` — Target workspace ID (single source of truth)
- **filename** `[string, optional, max 255]` — Override the uploaded filename
- **title** `[string, optional, max 255]` — Custom document title; defaults to filename without extension
- **parser** `[string, optional]` — Ingestion pipeline; only `'v2.2.1'` accepted; omits uses platform default
- **tags[]** `[int[], optional]` — Tag IDs for manual assignment at creation time
- **external_metadata** `[object|null, optional]` — External source metadata:
  - `external_id` `[string, required within object]` — Source system document ID; enables idempotent uploads
  - `doc_type` `[string, optional]` — e.g., `"incident"`, `"page"`
  - `additional_metadata` `[object,optional]` — Arbitrary JSON passed through as-is (`external_url`, `created_at`, `modified_at`, etc.)

### File Status Lifecycle

```
pending → pending_conversion → converting → parsing → embedding → embedded
                                                    ↓              ↓
                                             parsing_failed    embedding_failed → fail
```

Parallel vision processing status: `pending` → `processing` → `embedded` (or `fail`, `-` = N/A)

### Analysis

Files are the core resource. Upload is intentionally simple (just `file` + `workspace_id`) but extensible through optional metadata, tagging, parser selection, and external system tracking. The async pipeline decouples upload acceptance from processing time. The idempotency mechanism via `external_id` is critical for batch syncing from CMS/ITSM systems where the same document may be pushed on every sync cycle.

---

## 4. Tags

### Main Concepts

- **Flat Labels** — Company-wide, non-hierarchical labels used to group documents across workspaces.
- **Auto vs Manual Assignment** — Controls who can assign a tag: system auto-assignment (`auto_assign=true`) or manual-only (by users).
- **Scoping Filter** — Tags sc Search and Ask queries. Multiple tags are OR'd together within queries.

### Endpoints & Parameters

| Endpoint | Method | Purpose | Key Parameters |
|---|---|---|---|
| `/api/v3/tags` | `GET` | List all company tags (newest first) | Pagination params |
| `/api/v3/tags` | `POST` | Create a new tag | `name`, `description` (required), `auto_assign` (opt, default true) |

### Tag Response Model

- Core: `id`, `name`, `description` (`max 500 chars`)
- Flags: `auto_assign` — Can the system auto-assign this tag?
- Stats: `document_count` — Number of visible documents carrying this tag
- Timestamps: `created_at`, `updated_at`

### Analysis

Tags provide coarse-grained, cross-workspace groupings. Unlike Facets (tree hierarchies), tags are flat and universally applicable. They complement faceted metadata by enabling quick ad-hoc collections ("Q4 reports", "Project Alpha") without upfront schema design.

---

## 5. Search — Hybrid Vector + Text Retrieval

### Main Concepts

- **Hybrid Architecture** — Combines embedding-based vector similarity with keyword/text matching, then optionally re-ranks using a cross-encoder model.
- **No LLM Generation** — Returns raw chunks only; generation is handled by the separate `Ask` endpoint.
- **Vision Mode** — Uses VLM (vision-language models) embeddings over page images for searching visual documents.
- **Provenance Tracking** — Every chunk returns file metadata, page ranges, tags, content types, workspace info, and external metadata.

### Endpoint & Parameters

| Endpoint | Method | Purpose | Key Parameters |
|---|---|---|---|
| `/api/v3/search` | `POST` | Hybrid search query against indexed corpus | See below |

**Billing:** 1 retrieval credit per request.

### Request Schema (`SearchRequest`)

| Parameter | Type | Required | Default | Description |
|---|---|---|---|---|
| `query` | `string (max 1500)` | **Yes** | — | Natural-language search query |
| `max_results` | `int (1–50)` | No | 10 | Maximum chunks to return after reranking |
| `workspace_id[]` | `int[]` | No | All workspaces | Restrict results to specific workspace(s) |
| `tag_id[]` | `int[]` | No | None | Filter by tag IDs (OR logic; incompatible with `file_id`) |
| `file_id[]` | `int[]` | No | None | Search within specific files only (mutually exclusive with `workspace_id`) |
| `mode` | `enum: text, vision` | No | `text` | `text`: hybrid text search; `vision`: VLM page image search |
| `relevance_scoring` | `enum: none, scoring_only, scoring_and_filtering` | No | `scoring_and_filtering` | Controls cross-encoder reranking behavior |
| `include_image` | `boolean` | No | `false` | Include base64-encoded page image per result |
| `include_bboxes` | `boolean` | No | `false` | Include merged PDF bounding boxes (text mode only) |
| `content_type[]` | `string[]` | No | None | Filter by content type path with wildcards (e.g., `legal:contract:*`) |
| `attribute[]` | `string[]` | No | None | Facet attribute filters; multiple entries ANDed together; pipe-separated values within entry use OR logic |

### Response Schema (`SearchResponse`)

```json
{
  "results": [
    {
      "chunk_id": "...",
      "content": "The actual text of the retrieved chunk...",
      "score": 0.92,
      "scores": {
        "text": {...},       // Text similarity score
        "vision": {...},     // Vision model relevance (if applicable)
        "keyword": {...},    // Keyword match quality
        "multivector": {...}, // Multi-vector embedding scores
        "relevance": {...}   // Cross-encoder reranking score
      },
      "source": {
        "file_id": 12345,
        "filename": "document.pdf",
        "title": "Document Title",
        "mime_type": "application/pdf",
        "size_bytes": 245120,
        "page_start": 3,
        "page_end": 4,
        "total_pages": 10,
        "tags": [...],
        "content_types": [...]
      },
      "image_b64_content": "...",     // Optional base64-encoded page image
      "bboxes": [...]                 // Optional bounding box coordinates for merged PDF cells
    }
  ]
}
```

### Analysis

Search is the retrieval backbone. The hybrid scoring architecture (text vector + keyword + vision + cross-encoder reranking) produces higher-quality results than any single approach. The `scores` breakdown exposes each component's contribution, enabling tuning and debugging. Vision mode adds a parallel modality for image-heavy documents (scanned PDFs, screenshots, diagrams). Scoping via `workspace_id`, `tag_id`, or `file_id`, plus facet filtering on content types/attributes, provides surgical precision over the corpus.

---

## 6. Ask — Retrieval-Augmented Generation (RAG)

### Main Concepts

- **Two-Stage Pipeline** — First retrieves relevant chunks via Search, then generates an LLM answer grounded in those passages.
- **Source Attribution** — Returns both the generated answer and the ranked result chunks with full provenance for verification.
- **Streaming Support** — SSE-based streaming allows partial answers to appear immediately while chunks are still being retrieved and processed.
- **Custom Models** — Supports built-in models, LightOn fine-tunes (`alfred-ft5`), and custom registered ML models.

### Endpoint & Parameters

| Endpoint | Method | Purpose | Key Parameters |
|---|---|---|---|
| `/api/v3/ask` | `POST` | RAG over indexed corpus | See below |

**Billing:** 1 search-with-generation credit per request.

### Request Schema (`AskRequest`)

| Parameter | Type | Required | Default | Description |
|---|---|---|---|---|
| `query` | `string (max 1500)` | **Yes** | — | Natural-language question to answer over documents |
| `model` | `string` | No | `mistral-large-latest` | LLM: `mistral-large-latest`, `alfred-ft5`, or custom (`custom-{company_id}-{uuid}`) |
| `max_results` | `int (1–50)` | No | 10 | Max chunks to retrieve as context for the LLM |
| `stream` | `boolean` | No | `false` | If true, returns SSE stream of tokens and final answer |
| `workspace_id[]` | `int[]` | No | All workspaces | Scope retrieval to specific workspace(s) |
| `tag_id[]` | `int[]` | No | None | Filter by tags; OR logic across values |
| `file_id[]` | `int[]` | No | None | Restrict context to specified files only |
| `content_type[]` | `string[]` | No | None | Filter retrieval by content type path (e.g., `legal:contract`) |
| `attribute[]` | `string[]` | No | None | Facet attribute filters; multiple entries ANDed together |

### Response Schema (`AskResponse`)

```json
{
  "answer": "Generated LLM answer grounded in retrieved passages...",
  "results": [
    { /* SearchResultItem — same schema as /search endpoint */ }
  ]
}
```

Each `SearchResultItem` includes: chunk text, scores, page range, file metadata, tags, content types, workspace data, and optionally vision images with bounding boxes.

### Streaming Response (SSE)

When `stream=true`, the response is an SSE stream with events for partial token output culminating in a final structured answer object. The format mirrors standard OpenAI-compatible streaming.

### Analysis

Ask abstracts away retrieval complexity while providing full transparency through result inclusion. Users get both "the answer" and "why we think so." Model selection makes it extensible without additional integration — just register a custom model, then point Ask at it via `custom-{company_id}-{uuid}`. Streaming enables better UX for interactive applications where perceived latency matters as much as accuracy.

---

## 7. Parse — Document-to-Markdown Conversion

### Main Concepts

- **Universal Parser** — Handles PDF, PNG, JPG, PPTX, DOCX, ODT, HTML and more without manual OCR pipeline management.
- **Structured Markdown Output** — Each page gets clean Markdown conversion with tables, lists, headings, etc.
- **Two Modes:** Synchronous (20 MB / 15 pages max) vs Asynchronous (100 MB / 1000 pages, poll for status).
- **Flexible Input** — Accepts file upload or a publicly accessible document URL.

### Endpoints & Parameters

| Endpoint | Method | Purpose | Key Parameters |
|---|---|---|---|
| `/api/v3/parse` | `POST` | Submit parse job (file via multipart, or URL as JSON body) | See below |
| `/api/v3/parse/{id}` | `GET` | Poll async job status and result | — |

### Request Schema (`ParseRequest`)

| Parameter | Source | Required | Default | Description |
|---|---|---|---|---|
| `file` | multipart/form-data binary | Yes* | — | Binary file for parsing (*either this or `document` URL) |
| `document` | JSON body URI string | Yes* | — | Publicly accessible document URL as an alternative to file upload |
| `options.async` | either | No | `false` | When true, returns 202 Accepted + job ID for async polling; when false blocks until completion (up to 20 MB / 15 pages) |

### Response Schema (Synchronous — HTTP 200: `ParseResponse`)

```json
{
  "id": "parse_<token>",
  "status": "completed",
  "created_at": "ISO-8601 datetime",
  "completed_at": "ISO-8601 datetime",
  "processing_time_ms": 2840,
  "document": {
    "filename": "invoice.pdf",
    "page_count": 3,
    "file_size_bytes": 245120,
    "mime_type": "application/pdf"
  },
  "result": {
    "pages": [
      {"index": 1, "markdown": "# Invoice\n\nTable with extracted columns..."},
      {"index": 2, "markdown": "..."}
    ]
  },
  "usage": {
    "pages_processed": 3
  }
}
```

### Async Response (HTTP 202: `ParseAsyncResponse`)

```json
{
  "id": "parse_<token>",
  "status": "pending",       // or "failed" | "processing" | "completed"
  "created_at": "ISO-8601 datetime"
}
```

### Analysis

Parse provides standalone document-to-markdown conversion without requiring the file to be in a workspace. This is valuable for one-off document processing, pre-ingestion preview workflows (send rendered Markdown before committing to upload), or pipeline front-ends that need structured content from uploaded files before passing them to downstream consumers. The dual-mode (sync/async) design with page and size limits provides predictable SLAs on small documents while supporting large batch jobs in async mode.

---

## 8. Extract — Structured Data Extraction

### Main Concepts

- **Schema-Guided Extraction** — Provide a JSON Schema describing what fields to extract; the API uses an LLM to pull matching values from document content.
- **Page-Aware Results** — Output contains one extraction result per page, with `null` for fields that don't appear on specific pages.
- **Schema Validation** — Returned data conforms to the provided JSON Schema structure.

### Same Input/Output Limits as Parse

| Mode | Max Size / Pages | Returns |
|---|---|---|
| Synchronous | ≤ 20 MB, ≤ 15 pages | HTTP 200 with result immediately |
| Asynchronous | ≤ 100 MB, ≤ 1000 pages | HTTP 202 + poll via `GET /api/v3/extract/{id}` |

### Endpoints & Parameters

| Endpoint | Method | Purpose | Key Parameters |
|---|---|---|---|
| `/api/v3/extract` | `POST` | Submit extraction job (file upload or document URL) | See below |
| `/api/v3/extract/{id}` | `GET` | Poll async extraction job status and result | — |

### Request Schema (`ExtractRequest`)

| Parameter | Source | Required | Default | Description |
|---|---|---|---|---|
| `schema` | JSON body object | **Yes** | — | JSON Schema dict defining fields to extract with types, titles, descriptions, required flags |
| `file` | multipart binary | Yes* | — | File to extract from (*either this or `document`) |
| `document` | JSON body URI string | Yes* | — | Document URL as alternative input method |
| `options.async` | either | No | `false` | For large files: true returns 202 + job ID; false blocks until completion |

### Response Schema (Synchronous — HTTP 200: `ExtractJobResponse`)

```json
{
  "id": "ext_<token>",
  "status": "completed",       // or "pending" for async jobs in-progress
  "created_at": "ISO-8601 datetime",
  "completed_at": "ISO-8601 datetime",
  "processing_time_ms": 3200,
  "document": {
    "filename": "invoice.pdf",
    "page_count": 3,
    "file_size_bytes": 245120,
    "mime_type": "application/pdf"
  },
  "result": {
    "data": [                  // One object per page; fields match schema definition, null if not found
      {
        "invoice_number": "INV-2026-001",
        "total_amount": null,
        "date": "2026-07-08"
      },
      {
        "invoice_number": null,
        "total_amount": 1250.00,
        "date": null
      }
    ],
    "pagination": {
      "page": 1,
      "page_size": 15,
      "total_items": 3,
      "total_pages": 1,
      "has_next": false,
      "has_prev": false
    }
  },
  "usage": {                  // Resource usage tracking
    "pages_processed": 3
  },
  "progress": {              // For async in-progress jobs:
    "percentage": 100,        //   progress percentage and pages processed so far
    "pages_processed": 3
  }
}
```

### Domain-Specific Error Codes

| Code | Status | Description |
|---|---|---|
| `extraction_failed` | 500 | Pipeline failed unexpectedly. Retry. If persists, contact support. |
| `extraction_timeout` | 504 | Extraction exceeded allowed time. Try fewer pages at a time. |
| `model_timeout` | 504 | AI model did not respond within timeout. Reduce query complexity or document scope. |

### JSON Schema Example Input

```json
{
  "schema": {
    "type": "object",
    "properties": {
      "invoice_number": {"type": "string"},
      "date": {"type": "string", "format": "date"},
      "total_amount": {"type": "number"}
    },
    "required": ["invoice_number"]
  }
}
```

### Analysis

Extract turns unstructured documents into typed data without coding custom parsers for each document format. This is the key capability for building forms-from-documents, invoice processing pipelines, and data-entry automation. The page-aware granularity means partial information spread across pages doesn't produce ambiguous results — you know exactly where each field was found. Schema-based extraction leverages the LLM's natural language understanding while constraining output to structured data consumers can consume directly.

---

## 9. Facets — Content-Type Classification & Attributes

### Main Concepts

- **Tree-Based Taxonomies** — Hierarchical content types (`legal → contract → nda`) form independent classification trees alongside each other.
- **Multi-Taxonomy Documents** — A single document can belong to multiple trees simultaneously (e.g., `legal:contract:nda` AND `finance:investment`).
- **Custom Attributes Per Node** — Each tree node defines metadata fields with typed values, inherited from parent nodes down the hierarchy.
- **Seed Templates** — Ready-made taxonomies for common domains (`finance`, `healthcare`, `legal`, `manufacturing`, `tech`) available via adopt action.

### Endpoints & Parameters

#### Content-Type Tree Management

| Endpoint | Method | Purpose | Key Parameters |
|---|---|---|---|
| `/api/v3/content-types` | `GET` | List classification trees (full catalog) | `query`, `path`,  `depth`, `include_attributes` (opt, default true) |
| `/api/v3/content-types` | `POST` | Action-dispatched tree/attribute CRUD: `adopt`, `define_content_type`, `undefine_content_type`, `define_attribute`, `undefine_attribute` | See below |
| `/api/v3/content-types/templates` | `GET` | Browse seed templates for adoption | — |

**Query Filtering:** `?path=legal,finance&depth=2&query=5G antennas&include_attributes=true`  
- Combines with AND semantics: only content types under specified paths that match query retrieval.

#### File-Level Classification

| Endpoint | Method | Purpose | Key Parameters |
|---|---|---|---|
| `/api/v3/files/{id}/facets` | `POST` | Action-dispatched file classification & attribute CRUD: `classify`, `unclassify`,  `set_value`, `clear_value` | See below |
| `/api/v3/files/{id}/facets` | `GET` | Read back assigned content types and attribute values for a file | — |

### Content-Type Management Actions (`POST /api/v3/content-types`)

#### Management Actions (`POST /api/v3/content-types`)

| Action | Body Fields | Effect |
|---|---|---|
| `adopt` | `content_types[]` (seed names: e.g. `["legal", "finance"]`) | Bulk-import taxonomies from starter templates |
| `define_content_type` | `code`, `label`, optional `parent_path`, `description`, `inherit_attributes` | Create or update a tree node; idempotent if `(parent_path, code)` exists |
| `undefine_content_type` | `content_type_path` (e.g., `"legal:contract"`) | Delete the node and cascade its entire subtree |
| `define_attribute` | `content_type_path`, (`name`, `attribute_type`, optional `choices[]`, `required`, `description`) | Add or update a metadata field on the specified content type; types include `text`, `number`,  `date`, `boolean`, `select`, `multi-select`, `rich-text` + aliases |
| `undefine_attribute` | `content_type_path`, (`name`) | Remove an attribute column from a node |

#### File Facet Actions (`POST /api/v3/files/{id}/facets`)

| Action | Body Fields | Effect |
|---|---|---|
| `classify` | `content_type_path` (e.g., `"legal:contract"`) | Assign the content type to this file; idempotent |
| `unclassify` | `content_type_path` | Remove classification + cascade all attribute values under that path |
| `set_value` | `content_type_path`, `attribute_name`, (`value` — type-specific) | Set/update an attribute value on the file for a given content type |
| `clear_value` | `content_type_path`,  `attribute_name` | Remove a single attribute value |

**Value Types:**
- `text` / `rich-text` → string
- `number` → number (or numeric string)
- `date` → `"YYYY-MM-DD"`
- `boolean` → `true`/`false`
- `select` → single string from the defined `choices` list
- `multi-select` → array of strings from the defined `choices`

#### File Facet Constraints & Error Codes

| Constraint | Details |
|---|---|
| Sibling Conflict | A file can have only one content type per level within a given tree (e.g., `legal:contract` and `legal:compliance` conflict; use different trees or unclassify first) |
| Max Depth | 4 levels maximum (`depth_3` is the deepest nesting allowed) |

### Analysis

Facets is LightOn's most sophisticated metadata system, offering hierarchical taxonomy management coupled with typed attributes on individual documents. The multi-tenancy aspects (tree-based isolation), starter templates accelerate domain-specific deployments without manual schema design). The search and Ask queries can filter by content type path (`content_type` param) or specific attribute values using `attribute[]` arrays that support complex expressions. Sibling conflict rules within a tree prevent ambiguous single-paths in the scope filtering prevents invalid assignments while still allowing cross-classification from different trees.

---

## 10. Models Management

### Main Concepts

- **Custom ML Model Registry** — Register, update, and delete custom LLM endpoints that your company manages as first-class models within the platform.
- **Use in Ask** — Once registered, a model becomes selectable via `model: "custom-{company_id}-{uuid}"` on Ask queries.
- **Scoped to Company** — Models are isolated per company/organization boundary.

### Endpoints & Parameters

| Endpoint | Method | Purpose | Key Parameters |
|---|---|---|---|
| `/api/v3/models` | `GET` | List registered models for this company | Pagination params |
| `/api/v3/models` | `POST` | Register a new LL model endpoint | `name`, `endpoint_url`, `headers` (opt, optional auth config, etc) |
| `/api/v3/models/{uuid}` | `GET` | Retrieve single model details | — |
| `/api/v3/models/{uuid}` | `PATCH` | Update model configuration | `name`, `endpoint_url`, `headers`, etc. |
| `/api/v3/models/{uuid}` | `DELETE` | Remove a registered model from the company catalog | — |

### Model Configuration Fields (Inferred)

- **Name** `[string]` — Human-readable name displayed in UI and API responses
- **Endpoint URL** `[string, required]` — Base URL of the external LLM inference endpoint
- **Headers** `[object]` — Additional request headers to forward (e.g., `Authorization: Bearer ...`)
- **Temperature, `max_tokens`, `top_p`, `frequency_penalty`, etc. — Request body parameters forwarded with each generation call

### Error Handling

| Code | Status | Description |
|---|---|---|
| `model_not_found` | 404 | Model UUID not found in company catalog |
| `model_not_allowed` | 403 | Caller lacks permission to manage models (likely admin-only) |
| `model_config_invalid` | 422 | Endpoint URL, authentication header format error. |

### Analysis

The Models endpoint enables Bring-Your-Own-Model (BYOM). Organizations with existing LLM infrastructure can plug their proprietary or government-approved model as an inference target for LightOn's Ask queries without proxying through supported providers. This supports compliance requirements where data sovereignty mandates that generated answers stay within controlled environments.

---

## 11. Budget Management

### Main Concepts

- **Monthly Spend Cap** — Set a hard limit that blocks all API requests when reached.
- **Percentage Alerts** — Configure email notifications at custom spend thresholds (e.g., notify at `70%` usage).
- **Organization-Level Only** — Budget controls apply per company/organization, not per workspace or user.

### Endpoints & Parameters

| Endpoint | Method | Purpose | Key Parameters |
|---|---|---|---|
| `/api/v3/budgets` | `GET` | List budget configurations for this organization | Pagination params |
| `/api/v3/budgets/{id}` | `PATCH` | Update a budget (set cap, alerts) | `cap`, `alerts[]` |

### Budget Configuration Fields (Inferred)

- **Cap** `[number]` — Maximum monthly spend threshold in currency units (e.g., USD). Requests are rejected once this is hit.
- **Alerts** `[array of {percentage: number, method: "email", recipients: []}]` — Percentage thresholds that trigger email notifications. For example, alert at 50%, 80% spent.

### Error Codes

| Code | Status | Description |
|---|-----------------------|
| `budget_capped` | 429 | Monthly budget cap has been exceeded; all API requests blocked until next billing cycle or admin intervention. |
| `budget_not_found` | 404 | Budget configuration not found for this organization. |
| `budget_permission_denied` | 403 | User lacks permission to modify the budget (admin-level access required). |

### Analysis

Budget management protects against runaway costs from API usage at scale. The hard cap mechanism is particularly important for multi-tenant deployments where shared teams share a common billing pool and each needs their own safety net. Alert thresholds enable proactive monitoring before hitting limits. The per-month granularity means organizations can allocate budgets monthly, while the ability to set caps makes it suitable for cost-control environments.

---

## 12. Integrations Ecosystem

### Main Concepts

LightOn integrates as a retrieval layer across multiple inference providers and data sources:

| Integration Category | Supported Platforms | Description |
|---|---|---|
| **Datasource Sync** | Google Drive, Microsoft SharePoint | Create workspaces that automatically sync with external document storage. Documents are ingested at scale. |
| **LLM Inference Targets** | Inceptron, Lyceum Serverless, OVHcloud AI Endpoints, Scaleway Generative APIs | Use LightOn search as the retrieval layer for models hosted on these providers. Enables BYOM where you host LLMs but want semantic search context injection. |
| **Embedding Providers** (Inferred) | — | Custom model registration allows pointing to any endpoint supporting standard chat/generate interfaces. |

### MCP Server (Model Context Protocol)

LightOn exposed an MCP server for IDE/code-assistant integration:

| Endpoint | Purpose |
|---|------------------|
| `mcp-server/quickstart` | Connect to the LightOn MCP server from any MCP-compatible client
| Supported Clients | Claude Code, Cursor, Windsurf, Mistral VibeOpenCode, OpenAI Codex |
| `mcp-server/tools` | Reference for all tools exposed by the MCP server (search, ask, parse) |

### Analysis

The integrations position LightOn within existing developer workflows are focused on: **(1)** Ingestion side — connecting to document storage systems people already use, reducing manual upload friction; **(2)** Reasoning side — being a retrieval layer behind multiple inference providers, making it agnostic about where generation happens. The MCP server bridging documents into AI coding assistants as ground truth reference material during development.

---

## Architecture Summary & Key Takeaways

### Conceptual Layer Model

```
┌─────────────────────────────────────────────────────┐
│                  GENERATION LAYER                   │
│   Ask: RAG with source attribution, streaming,       │
│         custom model support, multi-model routing    │
└───────────┬─────────────────────────────────────────┘
            │ (retrieves chunks from)
┌────────────▼────────────────────────────────────────┐
│                RETRIEVAL LAYER                      │
│   Search: Hybrid vector + keyword + vision +         │
│           cross-encoder reranking, fine-grained      │
│           scoping (workspace/tag/file/facet filters) │
└───────────┬─────────────────────────────────────────┘
            │ (indexes from)
┌────────────▼────────────────────────────────────────┐
│              INGESTION LAYER                        │
│   Files: Universal upload with auto-parse, embed    │
│           Parse/Bypass workspace                   │
│   Extract: Schema-governed structured data           │
│           Parse: Document → Markdown                │
└───────────┬──────────────────────────────────────────┐
            │ (organize by)
┌────────────▼─────────────────────────────────────────┐
│             ORGANIZATION LAYER                       │
│   Workspaces, Tags, Facets: Hierarchical              │
│           content types with typed attributes         │
└───────────────────────────────────────────────────────┘

            │ (controlled by)
┌────────────▼─────────────────────────────────────────┐
│              ADMINISTRATION LAYER                    │
│   API Keys, Models, Budgets: Per-organization       │
│           auth credentials and limits                │
└───────────────────────────────────────────────────────┘

            │ (connected to)
┌────────────▼─────────────────────────────────────────┐
│              INTEGRATIONS LAYER                      │
│   Ingestion: Google Drive, M365 SharePoint        │
│   Inference: Inceptron, Lyceum Serverless AI       │
│          Endpoints, OVHcloud, Scaleway             │
│   IDEs: MCP server for Claude Code, Cursor, etc.    │
└───────────────────────────────────────────────────────┘
```

### Key Capabilities vs Traditional Pipelines

| Capability | Traditional Approach | LightOn Approach |
|---|---|---|
| Document OCR & Parsing | Custom Tesseract/DocLayout + manual pipeline maintenance | One-call Parse endpoint with auto-detection and quality guarantees |
| Vector Database Management | Self-hosted Pinecone/Milvus/etc. with scaling concerns, shard management, embedding model versioning | Managed embeddings generated automatically post-upload; no VDB config required |
| Chunk Strategy | Custom implementations, overlap/window tuning | Platform-managed chunking; configurable via parser versions and pipeline params) |
| RAG Orchestration | Build retriever + prompt template + LLM call chain | Single Ask endpoint handles retrieval, grounding, and generation in one call with source attribution included automatically |
| Metadata & Classification | Manual tagging systems or custom faceted search stack | Managed Facets system with hierarchical taxonomies, starter templates, typed attributes on document types |
| Cost Control | Application-level rate limiting + manual monitoring | Built-in API quotas with admin-manageable budget caps and percentage-based alerts |

### Unique Differentiators

1. **Vision-Based Search** — VLM (vision-language model) embeddings for visual content make charts, diagrams, screenshots searchable alongside text documents without separate processing paths.
2. **Multi-Language Document Summaries** — Upload → automatic summaries in 11 languages via parallel async pipeline (`status_vision`).
3. **Idempotent File Upload with External ID Mapping** — Designed for batch syncs and ETL pipelines where re-running jobs produces duplicate documents (critical for ERP/ITSM integration).
4. **Zero VDB Required** — The platform manages embeddings, indexing, sharding, retrieval — consumer only sends queries.
