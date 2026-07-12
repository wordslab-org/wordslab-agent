# Unified Document Intelligence API Specification

> **Aggregated from:** datalab, docetl, docling, google, ibm, knowledgegraph, lighton, mistral, mixedbread, openai, and weaviate
> **Purpose:** A single, exhaustive specification that encompasses every document-intelligence capability, concept, parameter, and processing step described across the eleven platform studies in `./platform-studies/documents/`.

---

## How to Read This Document

This document is written for end users — developers, product managers, and architects who want to understand the **full landscape** of document-intelligence capabilities before choosing a provider or building a system. It is organized as follows:

1. **Part I — Concepts & Vocabulary** — An approachable introduction to every concept you will encounter, with plain-language explanations and a glossary that maps the many different names providers use for the *same* idea.
2. **Part II — The Exhaustive Processing Pipeline** — Every document-intelligence capability ordered into a single exhaustive end-to-end pipeline, organized into **modular phases**. Each stage lists the alternative approaches available and the synonyms used by each platform.
3. **Part III — The Unified API Specification** — A detailed, provider-agnostic API reference that describes every endpoint, parameter, data structure, and response format needed to implement a "super complete" document-intelligence platform.
4. **Part IV — Cross-Reference & Coverage** — Naming-variant and alternative-approach cross-reference tables, end-to-end reference flows, and a capability coverage matrix.

Each source system is referenced by its short name in parentheses, e.g. `(weaviate)`, `(datalab)`, `(docling)`.

---

# Part I — Concepts & Vocabulary

## 1. What Is Document Intelligence?

Document Intelligence is the set of techniques that turn **unstructured or semi-structured documents** (PDFs, images, Office files, HTML, email, spreadsheets, audio, video, code, XML schemas) into **structured, machine-readable, searchable, and answerable** data. It spans the entire journey from a raw file on disk to a grounded answer in a chatbot, covering ingestion, parsing, layout understanding, chunking, enrichment, extraction, classification, entity resolution, embedding, indexing, retrieval, reranking, generation, knowledge-graph construction, and evaluation.

## 2. Core Concepts (Provider-Agnostic)

### 2.1 The Document Lifecycle
A document moves through three broad time horizons on the platform:
- **Index time** (write path) — the document is uploaded, parsed, chunked, enriched, embedded, and stored so it can be searched later. Most of the pipeline runs here.
- **Query time** (read path) — a user issues a query; the system rewrites it, searches the index, filters, reranks, and generates an answer.
- **Cross-cutting** — concerns that touch both paths: orchestration, evaluation, provenance, tenancy, security, visualization.

### 2.2 Document / File / Data Object
The unit of input. Variants: *File* `(datalab, openai, mistral, mixedbread)`, *Data Object* `(weaviate)`, *Document* `(google, docling)`, *Item* / *row* `(docetl)`, *Analyzer* (a processing job) `(ibm)`.

### 2.3 Workspace / Store / Collection
A named container holding documents and their index. Variants: *Workspace* `(lighton)`, *Store* / *search index* `(mixedbread)`, *Vector Store* `(openai)`, *FileSearchStore* `(google)`, *Collection* / *schema* / *class* `(weaviate)`, *Project* `(ibm)`, *Dataset* / *Frame* `(docetl)`.

### 2.4 Chunk
A retrievable text/media segment produced from a document. Variants: *Chunk* `(weaviate, mixedbread, mistral, openai)`, *DocumentChunk* `(mistral)`, *block* `(datalab, docling, ibm)`, *content item* `(docling)`.

### 2.5 Embedding / Vector
A numeric array representing semantic content. Variants: *Vector* `(weaviate, openai)`, *embedding* `(google, mistral, mixedbread, lighton, docetl)`.

### 2.6 Vectorizer / Embedder
The component that produces vectors. Variants: *Vectorizer module* `(weaviate)`, *Embedder* `(mistral)`, *Embedding model* `(google, openai)`.

### 2.7 Schema / Template / Ontology
A definition of the structure to extract or enforce. Variants: *extraction schema* / *page_schema* / *schema_id* `(datalab)`, *Pydantic template* `(knowledgegraph)`, *ontology* / *KeyClass* `(ibm)`, *JSON Schema* `(google, lighton, mixedbread, mistral)`, *output schema* `(docetl)`, *collection schema* `(weaviate)`.

### 2.8 Citation / Provenance
Linking outputs back to source document locations. Variants: *file_citation* `(google, openai)`, *citations* / *block IDs* `(datalab)`, *provenance* / *ProvenanceLedger* `(knowledgegraph, datalab)`, *source attribution* `(lighton, mixedbread)`.

### 2.9 RAG / Generative search / Ask / QA
Retrieval + LLM generation. Variants: *Generative search* `(weaviate)`, *Ask* `(lighton)`, *Question Answering* `(mixedbread)`, *File Search tool* `(openai)`, *Document QnA* `(mistral)`, *File Search* `(google)`.

### 2.10 Reranking
Second-stage reordering of retrieved results with a more expensive model. Variants: *Rerank* `(weaviate)`, *Reranker* `(mistral)`, *relevance_scoring* `(lighton)`, *rerank* `(mixedbread)`. Not present in `(openai, google)` as a separate API.

### 2.11 Query Rewriting
Improving a user query *before* retrieval (stripping conversational filler, generating sub-queries). This is a **pre-search** step — it logically precedes search and reranking.

### 2.12 Pipeline / Workflow / Operator chain
Orchestration of processing steps. Variants: *Pipeline* `(datalab, mistral, docetl)`, *Workflow (Temporal)* `(datalab)`, *operator chain* / *Frame* `(docetl)`, *run_pipeline* `(knowledgegraph)`, *ingestion pipeline* `(mixedbread, lighton)`.

### 2.13 Checkpoint
A stored intermediate parse state reusable by downstream steps — parse once, run many extractions without re-parsing. `(datalab)` unique.

### 2.14 Facet / Content-type / Tag
Classification metadata used to scope search. Variants: *Facet* / *content-type* `(lighton)`, *tag* `(lighton)`, *attribute* `(openai, lighton, weaviate)`, *custom_metadata* `(google, mixedbread)`.

### 2.15 Tenancy / Isolation
Multi-tenant data separation. Variants: *Workspace* `(lighton)`, *Tenant* `(weaviate)`, *Organization* `(mixedbread)`, *team context* `(datalab)`.

### 2.16 Quantization / Compression
Reducing vector storage. `(weaviate)` unique (PQ/BQ/SQ/RQ).

### 2.17 Billing Units
- **Per page** — conversion, extraction priced per 1K pages `(datalab, ibm)`.
- **Per token** — embedding, generation usage `(openai, mixedbread, lighton)`.
- **Per request / per file** — ingestion counts `(google, mixedbread)`.
- **Storage** — free in some `(google)`, per-index in others `(weaviate, openai)`.
- **Surcharges** — add-on bbox extraction `$0.30/1K pages` `(datalab)`; EU residency premium `(datalab)`.

## 3. Cross-Provider Synonym Glossary

| Concept | Variant names (system) |
|---|---|
| Document container | Workspace (lighton), Store / search index (mixedbread), Vector Store (openai), FileSearchStore (google), Collection / schema / class (weaviate), Project (ibm), Dataset / Frame (docetl) |
| Document / input unit | File (datalab, openai, mistral, mixedbread), Data Object (weaviate), Document (google, docling), Item / row (docetl), Analyzer (ibm) |
| Chunk | Chunk (weaviate, mixedbread, mistral, openai), DocumentChunk (mistral), block (datalab, docling, ibm), content item (docling) |
| Embedding | Vector (weaviate, openai), embedding (google, mistral, mixedbread, lighton, docetl) |
| Embedding component | Vectorizer module (weaviate), Embedder (mistral), Embedding model (google, openai) |
| Schema | page_schema / schema_id (datalab), Pydantic template (knowledgegraph), ontology / KeyClass (ibm), JSON Schema (google, lighton, mixedbread, mistral), output schema (docetl), collection schema (weaviate) |
| Citation / provenance | file_citation (google, openai), citations / block IDs (datalab), provenance / ProvenanceLedger (knowledgegraph), source attribution (lighton, mixedbread) |
| RAG / QA | Generative search (weaviate), Ask (lighton), Question Answering (mixedbread), File Search tool (openai), Document QnA (mistral), File Search (google) |
| Reranking | Rerank (weaviate), Reranker (mistral), relevance_scoring (lighton), rerank (mixedbread) |
| Query rewriting | rewrite_query (openai, mixedbread), LLMQueryRewriter / LLMQueryExtension (mistral) |
| Pipeline / orchestration | Pipeline (datalab, mistral, docetl), Workflow / Temporal (datalab), operator chain / Frame (docetl), run_pipeline (knowledgegraph), ingestion pipeline (mixedbread, lighton) |
| Classification metadata | Facet / content-type (lighton), tag (lighton), attribute (openai, lighton, weaviate), custom_metadata (google, mixedbread) |
| Tenancy / isolation | Workspace (lighton), Tenant (weaviate), Organization (mixedbread), team context (datalab) |
| Quantization / compression | Quantization (weaviate) — PQ/BQ/SQ/RQ |
| Segmentation | Document Segmentation (datalab), DS/SHW (ibm) |
| Entity dedup | Resolve (docetl), entity standardization (knowledgegraph), entity resolution / dense dedupe (knowledgegraph), link_resolve (docetl) |
| Filter | where clause / pre-filter (weaviate), attribute_filter / filters (openai, mixedbread), metadata_filter (google), content_type[]/attribute[] (lighton) |
| Hybrid fusion | RELATIVE_SCORE / RANKED (weaviate), RRF (openai, mistral), hybrid (lighton) |
| Expiration | expires_after (openai, mixedbread), TTL (weaviate), result retention (datalab), raw file 48h (google) |
| Chunking | Chunking (openai, google, docling, mistral), Split (docetl), TextSplitter (mistral), DocumentChunker (knowledgegraph) |

## 4. Input Formats (Union)

The union of accepted input formats across all systems. No single system supports all of these; the unified API accepts all and routes to format-specific backends.

- **Documents:** PDF, DOCX, DOC, PPTX, PPT, XLSX, XLS, ODT, ODP, ODS, ODF, RTF, CSV, TSV, HTML, HTM, Markdown, TXT, AsciiDoc, LaTeX, Box Notes, XML (JATS, USPTO, XBRL), Email (EML, MSG), NUMBERS, HWP/HWPX.
- **Images:** PNG, JPG/JPEG, WebP, AVIF, TIFF.
- **Audio:** MP3, WAV, M4A, AAC, OGG, FLAC, WebM Audio.
- **Video:** MP4, AVI, MOV, QuickTime, WebM, OGG Video.
- **Code:** Python, Java, Go, Rust, Swift, Kotlin, Scala, TypeScript, JavaScript, PHP, C, C++, C#, Ruby, Shell, PowerShell, CSS, diff, R Markdown, Graphviz, YAML, JSON.
- **Structured:** JSON, JSONL, Parquet, CSV, MXJSON/MXJSONL (pre-chunked format `(mixedbread)`), DocLang / `.dclx` `(docling)`, Docling JSON `(docling)`.
- **Other:** ZIP archives, WebVTT (timed text).

## 5. The Two Archetypes

The surveyed systems cluster into (often overlapping) archetypes:

| Archetype | Description | Examples |
|---|---|---|
| **Document Parsing / Conversion** | Turns raw files into Markdown/JSON/HTML/chunks with layout, tables, bboxes. | docling, datalab, mistral-OCR, ibm |
| **Managed Document-RAG / Search Platform** | Upload → auto-parse → chunk → embed → index → semantic search → grounded answers, zero infrastructure. | google (File Search), lighton, mixedbread, openai (Vector Stores) |
| **Vector Database / Search Engine** | Store pre-chunked objects + vectors; provide vector/keyword/hybrid search, RAG, reranking, aggregations. | weaviate |
| **Declarative Transformation Framework** | Map-reduce-style pipelines over documents with LLM operators (map, filter, reduce, resolve, cluster, rank, extract). | docetl |
| **Knowledge-Graph Builder** | Document → entities + relationships → graph (NetworkX / Neo4j) with provenance. | knowledgegraph (Docling-Graph, AI-Knowledge-Graph, Neo4j) |
| **Enterprise Capture / Extraction** | Classification + OCR + KVP/table extraction with ontology, validation, verification. | ibm (DPE) |

A **super-complete** system encompasses all of these archetypes in one coherent API.

---

# Part II — The Exhaustive Processing Pipeline

The pipeline is organized into **modular phases**. Phases A–F form the linear end-to-end flow from raw input to final answer/output. Phase G groups cross-cutting concerns that wrap or span the whole pipeline. Every stage is **optional** — stages can be combined, reordered, or bypassed depending on the use case.

```
PHASE A — Ingestion & Storage (index time)
  A1 Authentication & Access Control
  A2 Container / Workspace Management
  A3 File Upload & Ingestion
  A4 Document Segmentation / Boundary Detection
            │
            ▼
PHASE B — Document Understanding (parse + extract; checkpoint-reusable)
  B1 Document Parsing, OCR & Layout Analysis
  B2 Data Extraction (fields, tables, KVPs, annotations)
  B3 Classification & Categorization
            │
            ▼
PHASE C — Chunking & Enrichment (prepare for indexing)
  C1 Chunking / Splitting
  C2 Chunk Enrichment & Contextualization
            │
            ▼
PHASE D — Embedding, Indexing & Graph (build the retrievable store)
  D1 Embedding / Vectorization
  D2 Indexing & Storage
  D3 Entity Detection, Resolution, Deduplication & Linking
  D4 Clustering
  D5 Knowledge Graph Building
            │
   ┌────────┴───────── query-time boundary ─────────┐
   ▼                                                 │
PHASE E — Query Time (read path, correct order)       │
  E1 Query Rewriting & Preprocessing   ← runs BEFORE search
  E2 Search (lexical / vector / hybrid / agentic)
  E3 Filtering, Facets & Metadata Scoping
  E4 Reranking
  E5 Caching
  E6 Aggregations, Grouped Search & Analytics
            │                                         │
            ▼                                         │
PHASE F — Generation & Output                         │
  F1 Answer Generation / RAG / QA                     │
  F2 Document Transformation & Round-trip             │
            │                                         │
            ▼                                         │
PHASE G — Cross-Cutting Concerns (wrap the whole pipeline) │
  G1 Orchestration, Pipelines & Workflows             │
  G2 Evaluation, Quality Assurance & Optimization    │
  G3 Provenance, Citations & Source Tracking          │
  G4 Visualization                                    │
  G5 Multi-tenancy, Security, Residency & Administration │
```

---

## Phase A — Ingestion & Storage

### A1 Authentication & Access Control

**Purpose:** Authenticate requests and scope them to a tenant/team/organization.

**Alternative approaches:**
1. **API key (bearer token)** — `Authorization: Bearer <key>` or `X-API-Key` header. `(google, lighton, mixedbread, openai, weaviate, mistral, datalab)`
2. **HTTP Basic + Bearer/Zen JWT** — `(ibm)`
3. **OIDC tokens** — `(weaviate)`
4. **Model provider keys via headers** — `X-OPENAI-API-KEY`, `X-COHERE-API-KEY`, etc. for self-hosted vectorizer/generator modules. `(weaviate)`
5. **Scoped API keys with roles** — owner/editor/viewer, workspace-scoped. `(lighton)`

---

### A2 Container / Workspace Management

**Purpose:** Create the named container that holds documents and their index.

**Alternative approaches:**
1. **Workspace with type + residency** — `shared`/`personal`, `processing_location: us/eu`, embedding model, chunking config, expiration. `(lighton, datalab)`
2. **Vector store with expiration** — `expires_after: {anchor, days}`. `(openai, mixedbread)`
3. **FileSearchStore with immutable embedding model** — `embedding_model` set at creation. `(google)`
4. **Collection with schema + vectorizer config** — named vectors, distance metric, index type, per-property index flags. `(weaviate)`
5. **Project with ontology** — doc types, KeyClasses, validators. `(ibm)`
6. **Dataset / Frame** — lazy, immutable; Python API ↔ YAML. `(docetl)`

---

### A3 File Upload & Ingestion

**Purpose:** Get raw documents into the system.

**Alternative ingestion approaches (union):**

| # | Approach | Description | Systems |
|---|---|---|---|
| 1 | Direct multipart upload | Binary file in request body | datalab, docling, ibm, lighton, mistral, mixedbread, openai |
| 2 | URL-based ingestion | Pass a public/internal URL; platform fetches it | datalab, docling, google, lighton, mistral, mixedbread |
| 3 | Base64 inline | File bytes encoded inline in JSON body | docling, google, mistral |
| 4 | Presigned-URL upload | Platform issues a presigned PUT URL; client uploads to object storage | datalab, mixedbread |
| 5 | Resumable / multipart upload | Two-step initiate-then-upload for large files (~1 TB) | google, mixedbread |
| 6 | Cloud-storage loaders | S3, Azure Blob, GCS, Google Drive, SharePoint sync | mistral, lighton |
| 7 | Local filesystem / directory batch | Recursive directory reads | docling, docetl, mistral |
| 8 | In-memory / stream | `DocumentStream`, `from_list`, `convert_string` | docling, docetl |
| 9 | Pre-chunked ingestion (MXJSON/MXJSONL) | Bypass parsing/chunking; provide chunks directly | mixedbread |
| 10 | Docling JSON round-trip | Re-ingest a prior Docling JSON to re-export without re-parsing | docling |
| 11 | `datalab://` file references | Stable URI for previously uploaded files | datalab |
| 12 | Object insertion (JSON) | Insert pre-parsed JSON objects directly into a vector DB | weaviate |

**Key parameters (union):**
- `file` / `file_url` / `document` / `base64_string` / `http_sources` — input source.
- `workspace_id` / `store_id` / `vector_store_id` / `collection` / `project_id` / `parent` — target container.
- `filename` / `display_name` / `title` — human label.
- `metadata` / `attributes` / `custom_metadata` / `external_metadata` / `tags` — attached metadata propagated to chunks.
- `external_id` / `external_metadata.external_id` — idempotent identifier; re-upload returns existing doc instead of duplicating. `(lighton, mixedbread)` — slash-supported path-like IDs `(mixedbread)`.
- `parser` / `parsing_strategy` — selectable ingestion pipeline version or strategy (e.g. `fast`). `(lighton, mixedbread)`
- `processing_location` — data residency (`us`/`eu`). `(datalab)`
- `chunking_config` / `chunking_strategy` — chunking config at upload time. `(google, openai, mixedbread)`
- `purpose` — gates usage (e.g. `assistants` for vector stores). `(openai)`
- `max_file_size` — limits (200 MB `(datalab)`, 250 MB `(ibm)`, 512 MB `(openai)`, 50 MB PDF `(google)`, ~1 TB multipart `(mixedbread)`, 100 MB async `(lighton, mistral)`).

**Async processing pattern:** Upload returns a job/file ID + status; client polls or receives webhook until terminal state. Status lifecycles:
- `pending → in_progress → completed | failed | cancelled` `(mixedbread, openai)`
- `pending → pending_conversion → converting → parsing → embedding → embedded` `(lighton)`
- `PROCESSING → ACTIVE | FAILED` `(google)`
- `pending → started → success | failure` `(docling)`
- `In Progress → Completed | Failed` `(ibm)`
- `pending → dispatched → running → completed → failed → skipped` `(datalab pipelines)`

**Webhooks:** `(datalab, ibm, docling-via-WS)` — alternative to polling. Configure default in account settings or override per-request with `webhook_url`.

**Batch ingestion:** Up to 500 files per batch `(openai)`; bulk operations `(mixedbread)`; `convert_all` `(docling)`; batch runs over collections `(datalab)`; concurrent `asyncio.gather` `(mistral)`.

**Idempotent ingestion:** Deterministic UUID5 / `external_id` — re-ingesting overwrites, not duplicates. `(weaviate, mistral, lighton, mixedbread)`

---

### A4 Document Segmentation / Boundary Detection

**Purpose:** Split a multi-document PDF into logical sections (each segment = a separate document with page ranges). This is distinct from *chunking* — segmentation splits *documents within a file*; chunking splits *content within a document*.

**Naming variants:** *Segmentation* / *Document Segmentation* `(datalab)`; *Document Segmentation* (`DS`/`SHW`) `(ibm)`.

**Alternative approaches:**
1. **Schema-guided segmentation** — provide `segmentation_schema` with segment names + descriptions. `(datalab)`
2. **Automatic boundary detection** — `segmentation_strategy: document_boundary` for auto-detection. `(datalab)`
3. **Page-structure segmentation** — header-based page structure segmentation. `(ibm)`

**Output:** Segments with name, page ranges, confidence (`high`/`medium`/`low`). `(datalab)`

---

## Phase B — Document Understanding

### B1 Document Parsing, OCR & Layout Analysis

**Purpose:** Convert raw bytes into structured text + layout + tables + images + metadata.

**Naming variants:** *Document conversion* `(datalab, docling)`, *OCR Processor* `(mistral)`, *Document Understanding* `(google)`, *parsing* `(lighton, mixedbread, openai)`, *backend* `(docling)`.

**Alternative parsing approaches (union):**

| # | Approach | Description | Systems |
|---|---|---|---|
| 1 | Multi-model pipeline | Separate models for layout (DocLayNet), OCR (Tesseract/Surya/RapidOCR), table structure (TableFormer). Modes: `fast`/`balanced`/`accurate` | docling, datalab |
| 2 | Single end-to-end VLM | One vision-language model (GraniteDocling 258M, or remote VLM API) replaces the entire chain | docling |
| 3 | Native multimodal vision | The LLM "sees" PDF pages as images; `media_resolution: low/medium/high` | google |
| 4 | Managed automatic parsing | Platform-internal, not exposed (OCR + layout + transcription) | lighton, mixedbread, openai |
| 5 | Dedicated OCR API | Endpoint returning markdown + images + tables + blocks; 13 block types with bboxes in reading order; confidence scores | mistral (`mistral-ocr-latest`/`4-0`) |
| 6 | Word-level OCR with font metadata | Per-word coordinates, confidence, bold/italic/font | ibm |
| 7 | Format-specific backends | Subclassable per format (PDF, DOCX, HTML, image, audio) | docling |
| 8 | Audio/video transcription (ASR) | Whisper, Voxtral; diarization + timestamps; output as WebVTT or text | docling, mistral |
| 9 | Legacy office conversion | `.doc/.ppt/.hwp` → PDF via PyMuPDF Pro → OCR | mistral |

**Output representations (union):**
- **Markdown** — per-page or full-document, with tables/lists/headings. `(datalab, docling, mistral, lighton)`
- **HTML** — with `data-block-id` attributes for citation tracking. `(datalab, docling)`
- **JSON / DoclingDocument** — hierarchical: content items (texts, tables, pictures, key_value_items), body tree, furniture tree (headers/footers separated), groups, reading order, bboxes, provenance. `(docling, datalab, ibm)`
- **DocTags** — compact markup separating content from layout; token-efficient for LLMs. `(docling)`
- **DocLang XML / `.dclx`** — schema-validated XML interchange with zipped page images. `(docling)`
- **Chunks** — pre-segmented for vector DBs. `(datalab)`
- **Blocks** — paragraph-level structural units with bboxes, types (text, title, list, table, image, equation, caption, code, references, aside_text, header, footer, signature). `(mistral, datalab)`
- **Bounding boxes** — per-word, per-cell, per-list-item, per-block; with confidence. `(datalab, docling, ibm, mistral)`
- **Confidence scores** — page-level, word-level, parse-quality (0–5). `(datalab, ibm, mistral)`
- **Images** — extracted, base64-embedded, or referenced; with captions. `(datalab, docling, mistral)`
- **Tables** — structured with rows/columns/multi-level headers; markdown or HTML format. `(datalab, docling, mistral, ibm)`

**Enrichments (post-parse, optional):**
- **Code language detection** — `code_language` on code blocks. `(docling)`
- **Formula extraction** — LaTeX on formula items. `(docling)`
- **Picture classification** — chart types, diagrams, logos, signatures. `(docling)`
- **Picture description / captioning** — local VLM or remote API. `(docling)`
- **Chart understanding / infographics** — `(datalab)` (`extras: chart_understanding, infographic`).
- **Link extraction** — hyperlinks. `(datalab, mistral)`
- **Track changes / redline extraction** — insertions/deletions/comments from DOCX. `(datalab)`
- **Headers/footers detection** — `(ibm, mistral, docling-furniture)`
- **Reading order detection** — encoded in body tree. `(docling)`

**Key parameters (union):**
- `mode` / `processing_mode` — `fast`/`balanced`/`accurate`. `(datalab)`
- `do_ocr` / `force_ocr` / `ocr_lang` / `ocr_preset` — OCR control. `(docling)`
- `table_format` — `markdown`/`html`/`None`. `(mistral)`
- `include_image_base64` / `disable_image_extraction` / `image_export_mode` (`placeholder`/`embedded`/`referenced`). `(docling, mistral, datalab)`
- `include_blocks` / `add_block_ids` — structural block output. `(mistral, datalab)`
- `confidence_scores_granularity` — `page`/`word`/`None`. `(mistral)`
- `bbox_annotation_format` — schema for per-image classification/description. `(mistral)`
- `paginate` / `page_range` / `max_pages` — page-level control. `(datalab, docling, mistral)`
- `word_bboxes` / `table_cell_bboxes` / `list_item_bboxes` — granular bbox output. `(datalab)`
- `token_efficient_markdown` — `(datalab)`
- `keep_pageheader_in_output` / `keep_pagefooter_in_output` — furniture retention. `(datalab)`
- `jsonOptions` — comma-separated capability toggles (`HR,DC,KVP,TH,OCR,SN,MT,DS,CHAR`). `(ibm)` — with a formal dependency chain.
- `media_resolution` — `low`/`medium`/`high`. `(google)`

**Checkpoint system:** A stored parse state (`checkpoint_id`) from a conversion with `save_checkpoint=true`, reusable by extraction, segmentation, schema generation — avoids re-parsing. `(datalab)` — unique.

---

### B2 Data Extraction (Fields, Tables, KVPs, Annotations)

**Purpose:** Pull structured data (typed fields, tables, KV pairs) from documents using schemas or models.

**Naming variants:** *Structured extraction* / *Extract* `(datalab, lighton, mistral, docetl)`, *Annotations* (BBox / Document) `(mistral)`, *KVP extraction* `(ibm)`, *structured output* `(google, openai)`, *Map operator* `(docetl)`.

**Alternative extraction approaches (union):**

| # | Approach | Description | Systems |
|---|---|---|---|
| 1 | JSON-schema-driven LLM extraction | Provide a JSON Schema / Pydantic / Zod model; LLM fills it. Modes: `turbo` (image-only, no citations), `fast` (citations + scores), `balanced` (verification + reasoning + citations) | datalab, google, lighton, mistral, mixedbread, docetl |
| 2 | BBox annotation | Per-image classification/description via schema | mistral |
| 3 | Document annotation | Document-level structured extraction via schema | mistral |
| 4 | KVP extraction with ontology | Key + value with coordinates, confidence, KeyClass tagging, validators; three output tiers (basic/detailed/verbose) | ibm |
| 5 | Table / line-item extraction | Recursive `ComplexKVPStructure` with nested rows/cells; nested tables | ibm |
| 6 | Semantic normalization | Cleans/standardizes values (names, addresses) with `OriginalValue` preservation | ibm |
| 7 | Verbatim text extraction | Pull source text without synthesis; line_number or regex strategy; lower token cost, no hallucination | docetl (`Extract`) |
| 8 | Form filling | Fill PDF/image forms with field data; AcroForm + visual + image field detection; confidence threshold | datalab |
| 9 | Schema auto-generation | Generate candidate extraction schemas (simple/moderate/complex) from a checkpoint | datalab |
| 10 | Pydantic-template extraction | Schema-first Pydantic models define extraction schema AND graph structure; one-to-one or many-to-one | knowledgegraph |
| 11 | Dense extraction | Two-phase skeleton-then-flesh extraction contract for large documents | knowledgegraph |

**Extraction modes (datalab):**
- `turbo` — image-only, no citations, lowest price.
- `fast` — per-field citations + `_score` (1–5) confidence with reasoning.
- `balanced` — per-field independent verification (PASS/FAIL) + reasoning + citations.

**Output quality signals (union):**
- Per-field citations (block IDs traceable to source). `(datalab)`
- Per-field verification status (`PASS`/`FAIL_UNRESOLVABLE`/`FAIL_FIX`/`FAIL_CITATIONS`/`ITEMS_MISSING`). `(datalab)`
- Per-field confidence score (1–5) + reasoning. `(datalab)`
- Per-field `KeyConfidence`/`ValueConfidence`/`KeyClassConfidence`. `(ibm)`
- Extraction score average. `(datalab)`
- Parse quality score (0–5). `(datalab)`

---

### B3 Classification & Categorization

**Purpose:** Assign documents/chunks to categories, manage taxonomies, and use categories as search filters.

**Alternative approaches (union):**
1. **AI document classification** — assign to a known document class with confidence + alternate candidates. `(ibm)` — `Classification.DocumentClass` with `ClassConfidence`, `ClassMatch` (Low/Medium/High), `AlternateDocumentClass[]`.
2. **Hierarchical content-type taxonomy (Facets)** — multi-tree taxonomy (max depth 4, `:`-path notation); typed, inheritable attributes per node; seed templates (finance, healthcare, legal, manufacturing, tech); file-level classify/unclassify/set_value/clear_value actions; sibling-conflict rules; filterable in search. `(lighton)`
3. **Flat tags** — company-wide, non-hierarchical labels; `auto_assign` flag; OR'd in queries. `(lighton)`
4. **Classification modification via custom processors** — classify pages for downstream routing. `(datalab)`
5. **LLM map with enum output** — classification via structured output schema. `(docetl)` — with calibration for consistency.
6. **Filter-based classification** — use metadata filtering and facets on user-defined metadata. `(mixedbread)`
7. **Picture classification** — chart types, diagrams, logos, signatures. `(docling)`
8. **Zero-shot classification via embeddings** — embed labels, compare cosine similarity. `(openai)`

**Key parameters:** `docClass` (override) `(ibm)`; `content_type_path` `(lighton)`; `attribute_name`, `value` `(lighton)`; `class_match` threshold `(ibm)`.

---

## Phase C — Chunking & Enrichment

### C1 Chunking / Splitting

**Purpose:** Divide a parsed document into retrievable/embedding-ready segments.

**Naming variants:** *Chunking* `(openai, google, docling, mistral)`, *Split* operator `(docetl)`, *TextSplitter* `(mistral)`, *chunking_strategy* `(openai)`, *chunking_config* `(google)`, *DocumentChunker* `(knowledgegraph)`, *platform-managed chunking* `(lighton, mixedbread)`.

**Alternative chunking approaches (union):**

| Approach | Description | Systems |
|---|---|---|
| Static / token-count | Fixed token window with overlap. `max_chunk_size_tokens` (100–4096, default 800), `chunk_overlap_tokens` (default 400) | openai, google, mistral, docetl |
| Character-count | Fixed character window. `chunk_size` (default 1000) | mistral |
| Separator / hierarchical | Split on configurable separators (paragraph, sentence) with fallback | mistral, docetl |
| Markdown / header-aware | Split by markdown headers (`#`, `##`...); preserve header context | mistral, docling |
| Hierarchical / structure-pure | One chunk per document element; merges list items; attaches headers/captions | docling (`HierarchicalChunker`) |
| Hybrid (tokenization-aware) | Splits oversized chunks, merges undersized peers; aligned to tokenizer; table-header repetition. Production default for RAG | docling (`HybridChunker`) |
| Line-based | Preserves line boundaries (tables, code, logs, lists); supports repeated prefixes | docling (`LineBasedTokenChunker`) |
| Word-count with overlap | `chunk_size` words, `overlap` words | knowledgegraph (AI-KG) |
| Structure-preserving (Docling-based) | Token-bounded; tables/lists/section hierarchy intact; sentence→word→char fallback | knowledgegraph (Docling-Graph) |
| Automatic / managed | Platform-managed; not configurable | lighton, mixedbread, google (default), openai (default) |
| Pre-chunked (bypass) | Provide chunks directly via MXJSON/MXJSONL | mixedbread |
| Gather (context enrichment) | Adds context from surrounding chunks (previous/next, head/middle/tail) + header hierarchy | docetl (`Gather`) |

**Chunk types (union):** `text`, `image_url`, `audio_url`, `video_url` `(mixedbread)`; `content`, `image_annotation`, `summary` `(mistral)`.

**Key parameters (union):**
- `max_chunk_size_tokens` / `max_tokens_per_chunk` / `chunk_size` / `chunk_max_tokens` — chunk size.
- `chunk_overlap_tokens` / `max_overlap_tokens` / `overlap` — overlap.
- `tokenizer` / `tokenizer_name` / `tokenizer_model` — tokenizer alignment.
- `merge_list_items` / `merge_peers` / `repeat_table_header` / `omit_header_on_overflow` — structure preservation. `(docling)`
- `keep_separator` / `strip_whitespace` / `strip_headers` / `headers_to_split_on` — separator control. `(mistral)`
- `num_splits_to_group` — grouping split chunks. `(docetl)`

**Metadata preserved on chunks (union):** `page_number`, `filename`, `filepath`, `start_offset`/`end_offset`, `images`, `chunk_id`, `chunk_index`, `headings`, `token_count`, `text_hash`, `char_length`, `locator` (`char:`, `page:`, `summary:` formats) `(mistral)`.

---

### C2 Chunk Enrichment & Contextualization

**Purpose:** Add metadata, summaries, or surrounding context to chunks during or after ingestion.

**Alternative approaches:**
1. **Summary enrichment** — generate a document/chunk summary and optionally prepend it to every chunk (`propagate_summary_to_chunks`). `(mistral)`
2. **Contextualization at index time** — `with_metadata` and `with_file_context` enrich chunk context. `(mixedbread)`
3. **Gather (context windows)** — add peripheral chunks (previous/next) + reconstructed headers to each chunk. `(docetl)`
4. **Generated metadata** — auto-extract typed metadata per chunk (language, word_count, page info, code line ranges, media durations, BPM, frame counts, temporal boundaries). `(mixedbread)`
5. **Custom enrichers** — pluggable `ChunkEnricher` interface for arbitrary metadata injection. `(mistral)`

---

## Phase D — Embedding, Indexing & Graph

### D1 Embedding / Vectorization

**Purpose:** Convert text/images/audio/video into numeric vectors for semantic search.

**Naming variants:** *Vectorization* / *Vectorizer module* `(weaviate)`, *Embedder* `(mistral)`, *embedding* `(google, openai, mixedbread, lighton, docetl)`.

**Alternative approaches (union):**

| # | Approach | Description | Systems |
|---|---|---|---|
| 1 | Auto-vectorization on insert | Vectorizer module generates vectors from object properties automatically. Providers: OpenAI, Cohere, Google, Hugging Face, Ollama, Jina, NVIDIA, Mistral, AWS, Voyage AI, Transformers (self-hosted) | weaviate |
| 2 | Managed automatic embedding | Platform embeds during ingestion; not exposed | google, lighton, mixedbread, openai (Vector Stores) |
| 3 | Standalone embedding API | Generate vectors for arbitrary text; return to client for own vector DB. Models: `text-embedding-3-small` (1536d), `text-embedding-3-large` (3072d), `ada-002`; `dimensions` (MRL shortening); `encoding_format` (float/base64) | openai |
| 4 | Configurable embedder | Pluggable `Embedder` ABC. Models: 1024-dim, 256-dim, 128-dim | mistral |
| 5 | Named vectors (multi-model) | Multiple vector spaces per collection, each with independent vectorizer + index + compression; multi-model search on same data | weaviate |
| 6 | Multi-vector embeddings | ColBERT/ColPali-style multi-vector | weaviate (v1.30+) |
| 7 | Multimodal embeddings | Text + images in unified vector space; text query retrieves images and vice versa | google (`gemini-embedding-2`) |
| 8 | Vision (VLM) embeddings | VLM embeddings over page images for searching visual documents; parallel `status_vision` pipeline | lighton |
| 9 | Whole-document embeddings | `mixedbread-ai/mxbai-wholembed-v3`; required for audio/video | mixedbread |
| 10 | Self-provided vectors | User supplies vectors; no vectorizer | weaviate |
| 11 | Embeddings as ML features | For classification, clustering, regression, recommendations | openai |

**Key parameters (union):**
- `model` / `embedding_model` / `model_name` — model selection.
- `dimensions` — dimension shortening (MRL). `(openai)`
- `encoding_format` — `float`/`base64`. `(openai)`
- `source_properties` — which properties to vectorize. `(weaviate)`
- `vectorize_property_name` / `skip_vectorization` — per-property control. `(weaviate)`
- `distance_metric` — `COSINE`/`DOT`/`L2_SQUARED`/`HAMMING`/`MANHATTAN`. `(weaviate)`
- `index_types` — `fts`/`embedding`/`hybrid`. `(docetl)`

**Embedding is free at query time** in some systems `(google, mixedbread)`; pay only at index time.

---

### D2 Indexing & Storage

**Purpose:** Persist documents, vectors, inverted indexes, and graph structures for fast retrieval.

**Alternative index/storage approaches (union):**
1. **Managed vector store (auto-indexed)** — platform manages embeddings, indexing, sharding; zero infrastructure. Free storage in some. `(google, lighton, mixedbread, openai)`
2. **Self-hosted vector database** — HNSW/Flat/Dynamic/HFresh index types; LSM-Tree storage; WAL + HNSW snapshots; lazy shard loading; async indexing. `(weaviate)`
3. **Vespa vector store** — swappable schema with `embedding_dimensions`, `indexing_mode`, `SearchMode`. `(mistral)`
4. **LanceDB local index** — FTS, embedding, or hybrid; no server. `(docetl)`
5. **Native graph database** — index-free adjacency; ACID; Bolt protocol; persistent. `(knowledgegraph — Neo4j)`
6. **File-based storage** — CSV, JSON, Cypher script, HTML. `(knowledgegraph)`
7. **DPE database** — results persist until explicitly deleted; no auto-expiration. `(ibm)`
8. **S3 integration** — source syncing and output writeback. `(datalab)`
9. **BYOB (Bring Your Own Bucket)** — enterprise feature; own object storage backend; ephemeral compute. `(mixedbread)`

**Vector index types (weaviate):**
- **HNSW** (default, in-memory graph) — `ef_construction`, `max_connections`/M, `ef` (dynamic), `dynamicEfMin/Max/Factor`, `flatSearchCutoff`, `filterStrategy` (ACORN/sweeping).
- **Flat** (brute-force, small data).
- **Dynamic** (auto flat→HNSW at 10k).
- **HFresh** (disk-backed, memory-efficient).

**Vector quantization / compression (weaviate):**
- **PQ** (Product Quantization) — ~24x; `segments`, `trainingLimit`, `encoder.type`.
- **BQ** (Binary Quantization) — 32x; no training.
- **SQ** (Scalar Quantization) — 4x; min/max training.
- **RQ** (Rotational Quantization) — 8-bit 4x / 1-bit ~32x; training-free; 98–99% recall; `rescoreLimit` for re-scoring.
- Re-scoring: over-fetch compressed candidates, re-score with original vectors.

**Inverted index** — roaring bitmaps for filtering; map index for BM25. `(weaviate)`

**Per-property index flags:** `index_filterable`, `index_searchable`, `index_range_filters`, `index_null_state`, `index_property_length`. `(weaviate)`

**Lifecycle / cost control:**
- `expires_after` / `expires_at` — auto-expire dormant stores; `anchor: last_active_at`. `(openai, mixedbread)`
- TTL — object expiration relative to a DATE property. `(weaviate)` (v1.36)
- Result retention — 1 hour after completion. `(datalab)`
- Raw file expiration — 48 hours; embeddings persist indefinitely. `(google)`

**Consistency levels:** `ALL`, `QUORUM` (default), `ONE` — per-query. `(weaviate)`

**Replication config** `(weaviate)`; **backups** `(weaviate)`.

**Multi-tenancy:** Per-tenant sharding; 50,000+ active shards per node; auto-tenant creation; tenant lifecycle (ACTIVE → INACTIVE → OFFLOADED to S3). `(weaviate)`

---

### D3 Entity Detection, Resolution, Deduplication & Linking

**Purpose:** Identify entities, canonicalize duplicates, fix links, and build entity relationships.

**Naming variants:** *Resolve* / *entity deduplication* `(docetl)`, *entity standardization* `(knowledgegraph)`, *entity resolution* / *dense dedupe* `(knowledgegraph)`, *link_resolve* `(docetl)`, *NodeIDRegistry* `(knowledgegraph)`, *Natural Language extractors* / `allSystemTKVPs` `(ibm)`.

**Alternative approaches (union):**
1. **Blocking + pairwise LLM comparison + union-find clustering + resolution** — reduce comparisons via code conditions and/or embedding similarity thresholds; auto-computed blocking threshold (95% recall target); union-find (DSU) for grouping; LLM resolution prompt. `(docetl)`
2. **Deterministic node ID registry** — fingerprint hashing (`ClassName_fingerprint`); same entity always gets same node ID across batches; enables cross-document graph merging. `(knowledgegraph)`
3. **LLM-based entity standardization** — cluster entity name variants into canonical nodes; remove self-referencing triples. `(knowledgegraph)`
4. **Dense dedupe** — `off`/`standard`/`aggressive` LLM reconciliation of same-entity aliases / OCR noise. `(knowledgegraph)`
5. **Link resolve** — fix links between items in a knowledge graph (one-sided; assumes canonical IDs); embedding blocking + LLM comparison. `(docetl)`
6. **Equijoin (fuzzy join)** — join two datasets by LLM-evaluated semantic similarity; embedding blocking. `(docetl)`
7. **Cross-references** — directional links between objects (across collections); manual entity-linking. `(weaviate)`
8. **BARGAIN cascade** — cheap proxy model + oracle-labeled threshold learning with statistical guarantees for binary resolve/filter. `(docetl)`

**Key parameters:** `blocking_keys`, `blocking_threshold`, `blocking_target_recall` (0.95), `blocking_conditions`, `comparison_prompt`, `resolution_prompt`, `embedding_model`, `cascade` (BARGAIN). `(docetl)`

---

### D4 Clustering

**Purpose:** Group items by semantic similarity.

**Alternative approaches (union):**
1. **Hierarchical agglomerative clustering** — binary tree of embeddings; cluster path annotation (most-specific → root); LLM-generated cluster summaries. `(docetl — `Cluster` operator)`
2. **KMeans on embeddings** — discover hidden groupings. `(openai)`
3. **Louvain community detection** — color-coded clusters in visualization. `(knowledgegraph)`
4. **Value sampling cluster** — k-means for representative subset selection in reduce. `(docetl)`
5. **t-SNE visualization** — 2D cluster diagnostics. `(openai)`

---

### D5 Knowledge Graph Building

**Purpose:** Build a graph of entities and relationships from documents.

**Alternative approaches (union):**
1. **Schema-validated Pydantic-driven pipeline** — Pydantic templates define extraction schema AND graph structure; deterministic provenance ledger; stable cross-batch node IDs; graph conversion to NetworkX `DiGraph`; export to CSV/Cypher/JSON/HTML. `(knowledgegraph — Docling-Graph)`
   - Extraction contracts: `direct` (one-pass), `dense` (skeleton-then-flesh two-phase), `auto` (resolves per document).
   - Backends: LLM (text, local/remote) or VLM (vision, local only).
   - Gleaning: extra "what did you miss?" extraction pass.
   - Dense dedupe: `off`/`standard`/`aggressive`.
2. **Schema-free LLM triple extraction** — SPO triples (`subject`, `predicate`, `object`, `chunk`); entity standardization (LLM clustering of variants); relationship inference (rule-based transitivity + lexical similarity + LLM-assisted subgraph bridging); PyVis HTML visualization with Louvain communities. `(knowledgegraph — AI-Knowledge-Graph)`
3. **Native graph database storage** — Neo4j with Cypher CRUD; index-free adjacency; ACID; GraphRAG; MCP server for agent tool exposure. `(knowledgegraph — Neo4j)`
4. **Link resolve** — fix links between items in a knowledge graph (one-sided). `(docetl)`
5. **Cross-references** — directional links between objects (across collections). `(weaviate)`

**Graph export formats:** CSV (Neo4j-compatible), Cypher script, JSON (node-link), HTML visualization, Docling outputs. `(knowledgegraph)`

**Graph statistics:** `node_count`, `edge_count`, `node_types`, `edge_types`, `avg_degree`, `density`. `(knowledgegraph)`

**Provenance levels:** `off` / `standard` / `detailed` (with char-offset spans). `(knowledgegraph)`

---

## Phase E — Query Time (Read Path)

> The stages below are ordered as they execute at query time: the user query is **rewritten first**, then **searched**, then **filtered**, then **reranked**, with **caching** and **aggregations** available throughout.

### E1 Query Rewriting & Preprocessing

**Purpose:** Improve user queries *before* retrieval. This stage logically precedes search and reranking.

**Alternative approaches:**
1. **Query rewriting** — auto-rewrite into search-friendly form (strips conversational filler, converts to noun-phrase). `(openai — `rewrite_query`)`, `(mixedbread — `rewrite_query`)` — rewritten query observable in `search_query`.
2. **LLM query rewriter** — reformulate informal queries. `(mistral — `LLMQueryRewriter`)`
3. **LLM query extension** — generate multiple sub-queries for broader retrieval. `(mistral — `LLMQueryExtension`)`
4. **Model-generated queries** — File Search tool generates its own queries (visible in `queries`). `(openai)`

---

### E2 Search (Lexical, Vector, Hybrid, Agentic)

**Purpose:** Find relevant chunks/documents given a query.

**Naming variants:** *Search* `(lighton, mixedbread, openai)`, *near_\** methods* `(weaviate)`, *Retrieval* `(openai, mistral)`, *File Search* `(google, openai)`, *Query* `(weaviate)`.

#### E2.1 Lexical / Keyword Search
- **BM25 / BM25F** — token frequency + IDF; BM25F extends to weighted multi-field. `(weaviate)` — per-property tokenization (WORD, LOWERCASE, WHITESPACE, FIELD, TRIGRAM for fuzzy/typo tolerance); property boosting (`^weight`); `and`/`or` operators with `minimum_match`; accent folding; stopword presets.
- **Grep (regex)** — RE2 regex pattern matching against literal chunk text; `content_groups` (text/generated); case sensitivity. `(mixedbread)`

#### E2.2 Vector / Semantic Search
- `near_text`, `near_vector`, `near_object`, `near_image` `(weaviate)` — four input modalities.
- `VectorRetriever` `(mistral)` — embedding-based semantic search.
- Semantic search via embeddings `(google, lighton, mixedbread, openai)`.
- **Move parameters** (curriculum learning) — `move_to` / `move_away` with force + concepts to steer results. `(weaviate)`
- **MMR (Maximal Marginal Relevance)** — diversity selection balancing relevance and diversity. `(weaviate)` (v1.37 preview)
- **Autocut / auto_limit** — detect natural breaks in distance/score distribution. `(weaviate)`

#### E2.3 Hybrid Search
- **Vector + keyword fusion** — run both in parallel, fuse weighted by `alpha` (0=pure keyword, 1=pure vector, 0.5=balanced). `(weaviate)` — two fusion algorithms: `RELATIVE_SCORE` (default) / `RANKED` (`1/(RANK+60)`).
- **Reciprocal Rank Fusion (RRF)** — blend semantic + keyword with tunable `embedding_weight`/`text_weight`. `(openai)` — at least one must be > 0.
- **RRF for multi-retriever fusion** — `RRFRanker` with `rrf_k` smoothing. `(mistral)`
- **Hybrid vector + keyword + vision** — vector similarity + keyword/text matching + optional cross-encoder reranking + vision mode. `(lighton)` — score breakdown per component (text/vision/keyword/multivector/relevance).
- **Hybrid web + internal** — virtual web store in `store_identifiers`; web results always reranked, merged with internal. `(mixedbread)`

#### E2.4 Agentic Search
- **Multi-round, agent-driven retrieval** — decompose complex questions into sub-queries; run multiple rounds; evaluate candidates; iterate; merge/rerank. `(mixedbread — `agentic`)` — `max_rounds`, `queries_per_round`, `instructions` (natural-language steering), `strict_top_k`, `score_threshold`.
- **Query Agent** — LLM translates natural language into database operations (searches, aggregations, filters, sorts across collections). `(weaviate)` — modes: Ask (answer + sources), Search (raw objects), Suggest (query discovery); streaming with progress; `additional_filters`; `view_properties`; pagination reusing searches.
- **Model-autonomous File Search** — model decides when to search, generates queries, synthesizes answers. `(openai)`

#### E2.5 Other Search Modes
- **List Chunks** — metadata-only retrieval (no embeddings, no similarity, no reranking); sort by metadata. `(mixedbread)`
- **Multi-store / federated search** — search across multiple stores in one query. `(google, mixedbread)`
- **Cross-collection Explore** — GraphQL function. `(weaviate)`
- **Cross-document reasoning** — up to 1000 pages across multiple PDFs. `(google)`

**Key search parameters (union):**
- `query` / `input` — natural language query.
- `top_k` / `max_results` / `max_num_results` / `limit` — result count (1–50 typical).
- `mode` — `text`/`vision` `(lighton)`.
- `alpha` — keyword/vector balance. `(weaviate)`
- `fusion_type` — `RELATIVE_SCORE`/`RANKED`. `(weaviate)`
- `ranking_options` — `ranker`, `score_threshold`, `hybrid_search` weights. `(openai)`
- `search_options` — `rerank`, `rewrite_query`, `agentic`, `score_threshold`, `return_metadata`, `media_content`. `(mixedbread)`
- `target_vector` — named vector selection. `(weaviate)`
- `distance` / `certainty` — similarity threshold. `(weaviate)`
- `include_image` / `include_bboxes` — visual output. `(lighton)`
- `media_content` — `auto`/`always`/`never`. `(mixedbread)`

---

### E3 Filtering, Facets & Metadata Scoping

**Purpose:** Narrow search results by metadata before or after retrieval.

**Three metadata layers (mixedbread):** User file metadata (bare keys), generated chunk metadata (`generated_metadata.` prefix), system fields (`file_id`, `chunk_index`).

**Alternative filter approaches:**
1. **Comparison filters** — `eq`, `ne`, `gt`, `gte`, `lt`, `lte`, `in`, `nin`, `like`, `not_like`. `(openai, mixedbread)`
2. **Compound filters** — `and`, `or`, `all` (AND), `any` (OR), `none` (NOT). `(openai, mixedbread, weaviate)`
3. **AIP-160 filter expressions** — `author="Robert Graves" AND year>=1934`. `(google)`
4. **Weaviate filter operators** — `equal`, `not_equal`, `less_than`, `greater_than`, `like` (with `*`/`?` wildcards), `contains_any/all/none`, `is_none`, `within_geo_range`, `length`. `(weaviate)`
5. **Cross-reference filtering** — `Filter.by_ref`, `Filter.by_ref_count`. `(weaviate)`
6. **Nested object property filtering** — index-based (`cars[0].make`), same-element correlation. `(weaviate)` (v1.38 preview)
7. **Content-type / facet filters** — `content_type[]` (wildcard path filters), `attribute[]` (AND across, OR within pipe-separated). `(lighton)`
8. **Tag filters** — `tag_id[]` (OR'd). `(lighton)`
9. **File ID scoping** — `file_ids` (inclusion or `["not_in", [...]]`). `(mixedbread)`

**Facets:** Aggregate chunk counts grouped by metadata values; query-time faceting; dot notation for nested fields. `(mixedbread)`

**Filter strategy:** ACORN (default, preserves HNSW graph integrity) vs sweeping. `(weaviate)`

**Attributes:** File-level key-value metadata; max 16 keys, 256 chars per key; string/number values. `(openai)`

---

### E4 Reranking

**Purpose:** Second-stage reordering of retrieved results with a more expensive model for sharper relevance. Runs *after* search and filtering.

**Naming variants:** *Rerank* `(weaviate)`, *Reranker* `(mistral)`, *relevance_scoring* `(lighton)`, *rerank* `(mixedbread)`. Not present in `(openai, google)` as a separate API.

**Alternative reranking approaches (union):**
1. **Cross-encoder reranking** — pointwise model re-evaluates query-chunk pairs. `(weaviate, mistral, lighton, mixedbread)` — providers: Cohere, Hugging Face, Voyage AI, Contextual AI, Transformers; models: `mxbai-rerank-large-v2` (pointwise), `cross-encoder/ms-marco-MiniLM-L-6-v2`.
2. **Listwise reranking** — instruction-steerable via natural language (`mxbai-rerank-v3-listwise`); inject ranking policies ("prefer recent docs", "prioritize primary sources"). `(mixedbread)`
3. **LLM reranking** — deep LLM scoring, 1 call per chunk. `(mistral — `LLMReRanker`)`
4. **RRF fusion** — Reciprocal Rank Fusion for multi-retriever results. `(mistral — `RRFRanker`)`
5. **Relevance scoring modes** — `none` / `scoring_only` / `scoring_and_filtering`. `(lighton)`
6. **Sequential reranker chaining** — progressive refinement, each receiving previous output. `(mistral)`
7. **Boost (soft ranking)** — lightweight reordering by property values, recency, popularity, soft filters; no external model. `(weaviate)` (v1.38 preview)

**Key parameters:** `rerank` (bool/object), `model`, `top_k` (chunks to rerank), `with_metadata` (metadata fields in reranking context), `prop` (property to pass to reranker), `query` (can differ from retrieval query). `(mixedbread, weaviate)`

**Works with all search types** `(weaviate)` — near_text, near_vector, near_object, near_image, bm25, hybrid.

---

### E5 Caching

**Purpose:** Skip embedding/retrieval/reranking on cache hits. Caching spans both index and query time.

**Alternative approaches:**
1. **Semantic cache** — match queries by meaning via cosine similarity threshold (0.99 strict / 0.95 balanced / 0.90 permissive); skip retrieval on hit; eviction policies (LRU/LFU/FIFO); TTL; metrics tracking. `(mistral — `CachedQueryEngine`, `SemanticCache`)`
2. **Result caching** — `skip_cache` override. `(datalab)`
3. **LLM call caching** — cache in `~/.cache/docetl/llm`. `(docetl)`
4. **Memoized terminal actions** — repeated calls with unchanged config reuse results. `(docetl)`
5. **Persistent media IDs** — stable blob IDs for image chunks enable caching. `(google)`
6. **HNSW snapshots** — point-in-time HNSW state for fast startup. `(weaviate)`

---

### E6 Aggregations, Grouped Search & Analytics

**Purpose:** Compute metrics over result sets; group results.

**Alternative approaches:**
1. **Aggregate queries** — counts, statistics (sum, max, min, mean, median, mode), frequency distributions (top_occurrences), boolean percentages, reference counts; GroupByAggregate for per-group metrics. `(weaviate)`
2. **Grouped search (GroupBy)** — organize results into groups by property or cross-reference; `objects_per_group`, `number_of_groups`. `(weaviate)`
3. **Reduce operator** — group by `reduce_key`, produce one output per group; incremental folding; scratchpad; value sampling. `(docetl)`
4. **Facets** — aggregate chunk counts by metadata values. `(mixedbread)`
5. **Rank operator** — full sorting by latent attribute (not top-k retrieval); "picky window" refinement; O(n) scaling. `(docetl)`

---

## Phase F — Generation & Output

### F1 Answer Generation / RAG / QA

**Purpose:** Generate grounded answers from retrieved chunks with citations.

**Naming variants:** *Generative search* / *RAG* `(weaviate)`, *Ask* `(lighton)`, *Question Answering* `(mixedbread)`, *File Search tool* `(openai)`, *Document QnA* `(mistral)`, *File Search* `(google)`.

**Alternative RAG approaches (union):**

| # | Approach | Description | Systems |
|---|---|---|---|
| 1 | Hosted RAG (model-autonomous) | Model decides when to search, retrieves, generates with inline citations. `file_citation` annotations with character offsets | openai (File Search tool) |
| 2 | Managed RAG (one-call) | Retrieve-then-generate in one API call; citations via `<cite i="n"/>` markers mapping to `sources[n]`; multimodal, streaming, instructions | mixedbread (Question Answering) |
| 3 | Two-stage retrieve + generate | Search then Ask; SSE streaming (OpenAI-compatible); source attribution | lighton (Ask) |
| 4 | Generative search in search calls | Single Prompt (per-object) + Grouped Task (per-group); `{property_name}` interpolation; multimodal (images in prompts); query-time model override | weaviate |
| 5 | Retrieval API + manual synthesis | Direct search returns chunks; developer feeds to `chat.completions.create` with `<sources>` XML pattern | openai |
| 6 | Document QnA | Chat Completions with `document_url` content block; multi-document queries | mistral |
| 7 | RAG via retriever + map/reduce | Attach retriever to map/filter/reduce/extract; inject `{{ retrieval_context }}` | docetl |
| 8 | GraphRAG | Vector search + graph traversal for multi-hop reasoning; patterns: vector+graph, agentic retrieval, entity-centric RAG, ontology-driven RAG | knowledgegraph (Neo4j) |
| 9 | Query Agent Ask mode | Natural language → answer + sources across collections | weaviate |
| 10 | Structured grounded output | JSON Schema + file search for machine-readable grounded responses | google (Gemini 3+) |

**Citation mechanisms (union):**
- `file_citation` with `file_name`, `page_number`, `media_id`, `custom_metadata`. `(google, openai)`
- `<cite i="n"/>` markers → `sources[n]` chunks. `(mixedbread)`
- `{field}_citations` arrays of block IDs. `(datalab)`
- Source chunks returned alongside answer. `(lighton, weaviate)`
- Character-offset annotations. `(openai)`

**Streaming:** SSE token streaming. `(lighton, mixedbread, weaviate)` — OpenAI-compatible format.

**Key RAG parameters (union):**
- `query` / `input` / `messages` — question (supports conversational history).
- `model` — generation model (built-in, fine-tune, or custom BYOM). `(lighton, mixedbread)`
- `stream` — boolean.
- `instructions` — answer-style steering. `(mixedbread)`
- `cite` — boolean. `(mixedbread)`
- `multimodal` — include image/OCR in context. `(mixedbread)`
- `single_prompt` / `grouped_task` / `grouped_properties` — generation modes. `(weaviate)`
- `generative_provider` — query-time model override. `(weaviate)`
- `qa_options` — cite, multimodal, stream. `(mixedbread)`

---

### F2 Document Transformation & Round-trip

**Purpose:** Generate or transform documents (not just extract from them).

**Alternative approaches (union):**
1. **DOCX generation from markdown** — `create-document`; native Word formatting; track changes revision marks (`<ins>`, `~~`, `<comment>` tags with author/datetime). `(datalab)`
2. **Form filling** — fill PDF/image forms with structured data; AcroForm + visual + image field detection; confidence threshold; PDF or PNG output. `(datalab)`
3. **Track changes extraction** — extract redlines (insertions/deletions) and comments from DOCX as Markdown/HTML/chunks with annotation tags. `(datalab)`
4. **Document round-trip** — DOCX → track-changes → markdown → create-document → DOCX with redlines. `(datalab)`
5. **Thumbnail generation** — page thumbnails from prior conversion; `thumb_width`, `track_changes` toggle. `(datalab)`
6. **Markdown-to-DOCX / DOCX-to-Markdown** — via parsing and generation endpoints. `(datalab, docling)`
7. **Synthetic data generation** — `output.n > 1` multiplies dataset. `(docetl)`

---

## Phase G — Cross-Cutting Concerns

> These concerns wrap or span the entire pipeline. They are not linear stages but apply across phases A–F.

### G1 Orchestration, Pipelines & Workflows

**Purpose:** Chain multiple processing steps into reusable, versioned executions.

**Alternative orchestration approaches (union):**

| # | Approach | Description | Systems |
|---|---|---|---|
| 1 | Declarative pipelines (YAML/Python) | Ordered operators with draft→saved→published lifecycle; immutable versioned snapshots; per-step intermediate results; eval integration; per-step billing | datalab |
| 2 | Temporal workflows | Temporal-engine workflow definitions; long-running, fault-tolerant | datalab |
| 3 | Declarative map-reduce framework | Lazy, immutable Frame; chained operations; terminal actions trigger execution; Python API ↔ YAML convertible. Operators: map, filter, reduce, resolve, equijoin, rank, extract, cluster, split, gather, unnest, sample, topk, parallel_map, link_resolve, code_map, code_reduce, code_filter. Tool-equipped agents on map/filter/reduce | docetl |
| 4 | Pipeline + RoutedPipeline | Ingestion orchestration with checkpointing + progress callbacks; multi-format routing by extension/MIME | mistral |
| 5 | QueryEngine + CachedQueryEngine | Retrieval orchestration with optional caching | mistral |
| 6 | run_pipeline(config) | Template loading → extraction → Docling export → graph conversion → export → statistics | knowledgegraph |
| 7 | Pipeline class | Ingestion orchestrator: load → extract → chunk → enrich → embed → index | mistral |
| 8 | MCP server | Expose operations as tools for AI agents | docling, mixedbread, weaviate, knowledgegraph-Neo4j, lighton |
| 9 | Automatic managed pipeline | No manual orchestration needed (chunk → embed → index automatic) | google, lighton, mixedbread, openai |

**Orchestration features:**
- **Checkpointing** — skip already-processed documents on restart. `(mistral, datalab)`
- **Progress callbacks** — tqdm progress. `(mistral)`
- **Parallelization** — `max_threads`, `parallel_workers`, `concurrent_thread_count`. `(docetl, knowledgegraph)`
- **Caching** — LLM calls + optimized plans cached. `(docetl, datalab)`
- **Retries / timeouts** — `max_retries_per_timeout`, `skip_on_error`. `(docetl)`
- **Cost & token tracking** — `frame.total_cost`, `frame.token_usage`. `(docetl)`

---

### G2 Evaluation, Quality Assurance & Optimization

**Purpose:** Measure and improve pipeline quality.

**Alternative approaches (union):**

| # | Approach | Description | Systems |
|---|---|---|---|
| 1 | Parse quality score | 0–5 self-assessment of conversion quality | datalab |
| 2 | Eval rubrics | block/page/document rules scoring 0–5; `eval_rubric_id` on convert/extract/pipeline steps; generation from user feedback | datalab |
| 3 | Forge Evals | configuration comparison (max 10 docs, 5 configs, 3 iterations); visual diffs; multi-model comparison | datalab |
| 4 | Custom processor eval definitions | `eval_definition` per processor; `run_eval` on execution | datalab |
| 5 | Per-field verification | balanced-mode per-field independent validation (PASS/FAIL) against source | datalab |
| 6 | Per-field confidence scoring | 1–5 score with reasoning | datalab |
| 7 | KVP validation | per-KeyClass validators defined in ontology; `POST /validator` applies them; `ValidatorResult` ("Pass"/"Fail") + `ValidatorFailures` | ibm |
| 8 | Gleaning | LLM-based iterative validation/refinement of operator output | docetl |
| 9 | Validate | Python-expression-based output validation with retries | docetl |
| 10 | Calibration | reference anchors for consistent classification/scoring | docetl |
| 11 | Plan rewrites | automatic equivalence-preserving pipeline reordering (selection_pushdown, limit_pushdown) | docetl |
| 12 | BARGAIN model cascades | statistical guarantees (accuracy/precision/recall) on binary operators with probability `1 - delta` | docetl |
| 13 | MOAR (offline MCTS optimization) | multi-objective agentic rewrites; Pareto-optimal cost-accuracy frontier | docetl |
| 14 | Operation-level `optimize` flag | inline optimization | docetl |
| 15 | Semantic cache metrics | hit_rate, avg_hit_similarity, retrieval time | mistral |

---

### G3 Provenance, Citations & Source Tracking

**Purpose:** Link outputs back to source document locations for verifiability and trust.

**Alternative approaches (union):**
1. **Block IDs** — `data-block-id` attributes on HTML elements; `{field}_citations` arrays traceable to `/page/0/Text/3` style locations. `(datalab)`
2. **`file_citation` annotations** — `file_name`, `page_number`, `media_id`, `custom_metadata`. `(google, openai)` — character offsets for inline attribution `(openai)`.
3. **`<cite i="n"/>` markers** → `sources[n]` chunks. `(mixedbread)`
4. **ProvenanceLedger** — deterministic node-to-source grounding (no LLM); resolution levels: document → page → batch → chunk → char-offset span; `SourceAnchor` kinds: observed/verbatim/derived/reconciled; `bind_stats`. `(knowledgegraph)`
5. **Source chunks alongside answer** — `results: [SearchResultItem]`. `(lighton)`
6. **Chunk provenance** — `file_id`, `filename`, `title`, `mime_type`, `size_bytes`, `page_start`/`page_end`, `total_pages`, `tags`, `content_types`. `(lighton)`
7. **Chunk locator** — `char:{start}-{end}`, `page:{n}:char:{start}-{end}`, `summary:char:0-512`. `(mistral)`
8. **Bounding boxes in search results** — merged PDF cell bboxes for UI highlighting. `(lighton)`
9. **Media download** — `download_media(media_id)` for image chunks; persistent IDs. `(google)`
10. **Per-page extraction results** — one result object per page with null absent fields. `(lighton)`

---

### G4 Visualization

**Purpose:** Visually explore graphs, clusters, and document structure.

**Alternative approaches:**
1. **Interactive HTML (Cytoscape)** — zoom/pan, node inspection, search, image export; extraction report + graph statistics. `(knowledgegraph — Docling-Graph)`
2. **PyVis/Vis.js HTML** — Louvain community detection; color-coded clusters; centrality-sized nodes; dashed edges for inferred relationships; light/dark, physics toggle. `(knowledgegraph — AI-Knowledge-Graph)`
3. **t-SNE 2D visualization** — cluster diagnostics on embeddings. `(openai)`
4. **Web UI playground** — interactive conversion testing. `(docling)`
5. **DocWrangler IDE** — spreadsheet interface with automatic visualizations, in-situ feedback, prompt refinement with diffs, version control. `(docetl)`

---

### G5 Multi-tenancy, Security, Residency & Administration

**Multi-tenancy:**
- Per-tenant sharding; auto-tenant creation; tenant lifecycle (ACTIVE → INACTIVE → OFFLOADED to S3). `(weaviate)`
- Workspace isolation (`shared`/`personal`); scoped API keys with roles (owner/editor/viewer). `(lighton)`
- Organization scoping. `(mixedbread)`
- Team context. `(datalab)`

**Authentication:**
- API key (bearer token). `(google, lighton, mixedbread, openai, weaviate, mistral)`
- HTTP Basic + Bearer/Zen JWT. `(ibm)`
- OIDC tokens. `(weaviate)`
- Model provider API keys via headers (`X-OPENAI-API-KEY`, etc.). `(weaviate)`

**Data residency:** `processing_location` (`us`/`eu`). `(datalab)` — EU pricing premium. BYOB for data sovereignty. `(mixedbread)`

**Budget controls:** Org-level monthly cap (hard block at 429 `budget_capped`); percentage email alerts. `(lighton)`

**Rate limiting:** Per-operation-type token-bucket (read 1200/min, write 360/min, delete 240/min). `(mixedbread)` — `Retry-After`. Tiered limits. `(openai)` — 300 RPM per vector store.

**Deployment modes:**
- Managed cloud SaaS. `(datalab, docling, google, ibm, lighton, mistral, mixedbread, openai, weaviate-cloud)`
- Self-hosted container / Docker / Kubernetes. `(docling, weaviate, ibm)`
- On-prem container. `(datalab)`
- Open-source local library. `(docling, docetl, knowledgegraph)`
- BYOB. `(mixedbread)`

**SDKs / CLI:**
- Python SDK. `(datalab, docling, google, mistral, mixedbread, openai, weaviate, docetl)`
- TypeScript/JavaScript SDK. `(google, mixedbread, weaviate)`
- Java, Go, C#. `(weaviate)`
- CLI. `(datalab, docling, docetl, mixedbread, knowledgegraph)`

**MCP server:** Expose operations as tools for AI assistants (Claude Code, Cursor, Windsurf, etc.). `(docling, mixedbread, weaviate, knowledgegraph-Neo4j, lighton)`

**Integrations:** LangChain, LlamaIndex, Haystack, CrewAI `(docling)`; Snowflake, Databricks, Vertex AI `(knowledgegraph-Neo4j)`; Vercel `(mixedbread)`; Google Drive, SharePoint `(lighton)`; Datacap (OSADP connector) `(ibm)`.

---

# Part III — The Unified API Specification

This part defines a **provider-agnostic API** that encompasses all features of the eleven surveyed systems. It is written as a specification for a hypothetical "super complete" document-intelligence platform, for an API consumer who wants to build document-intelligence applications.

> **Conventions:** All endpoints are RESTful under a base URL `https://api.unified-docintel.example/v1`. Parameters shown in `monospace` are JSON body fields or query params. Async operations follow submit→poll/webhook→retrieve. The spec uses unified naming; the synonym glossary in Part I maps to individual systems. `WS` denotes a WebSocket endpoint.

## API Surface Overview

```
Authentication & Administration:
  POST   /v1/api-keys                       — Create scoped API key (roles, workspace-scoped)
  GET    /v1/api-keys                        — List API keys
  DELETE /v1/api-keys/{id}                   — Revoke API key
  GET    /v1/health                          — Health check (no auth)
  GET    /v1/version                          — Version info
  GET    /v1/budgets                          — Get org budget
  PATCH  /v1/budgets                          — Update monthly cap + alert thresholds
  GET    /v1/usage                            — Usage/billing summary (tokens, pages, storage)

Workspaces (Containers):
  POST   /v1/workspaces                       — Create workspace
  GET    /v1/workspaces                        — List workspaces (pagination)
  GET    /v1/workspaces/{id}                  — Get workspace
  PUT    /v1/workspaces/{id}                  — Update workspace
  DELETE /v1/workspaces/{id}                  — Delete workspace (cascade)
  GET    /v1/workspaces/{id}/documents        — List indexed documents
  GET    /v1/workspaces/{id}/documents/{doc}  — Get indexed document
  DELETE /v1/workspaces/{id}/documents/{doc}  — Delete indexed document (cascade to embeddings)
  POST   /v1/workspaces/{id}/tenants          — Create tenants
  GET    /v1/workspaces/{id}/tenants           — List tenants
  PUT    /v1/workspaces/{id}/tenants          — Update tenant activity_status

Files (Upload & Ingestion):
  POST   /v1/files                            — Upload file (multipart | URL | base64)
  POST   /v1/files/uploads                    — Create multipart/resumable upload session
  GET    /v1/files/uploads/{id}               — Get upload status + presigned URLs
  POST   /v1/files/uploads/{id}/complete      — Complete multipart upload
  POST   /v1/files/uploads/{id}/abort         — Abort multipart upload
  POST   /v1/files/prechunked                 — Upload pre-chunked data (MXJSON/MXJSONL)
  GET    /v1/files                             — List files (filter by workspace, status, tags, metadata)
  GET    /v1/files/{id}                        — Get file metadata + status
  GET    /v1/files/{id}/download              — Download original or rendered_pdf
  GET    /v1/files/{id}/chunks                — List chunks of a file
  PATCH  /v1/files/{id}                        — Update file metadata/attributes
  DELETE /v1/files/{id}                        — Delete file
  POST   /v1/files/bulk-delete                — Bulk delete files

Document Parsing & Conversion:
  POST   /v1/convert                          — Parse document to structured output
  GET    /v1/convert/{request_id}             — Poll conversion result

Document Segmentation:
  POST   /v1/segment                          — Segment multi-document PDF
  GET    /v1/segment/{request_id}             — Poll segmentation result

Data Extraction & Annotations:
  POST   /v1/extract                          — Structured field extraction
  GET    /v1/extract/{request_id}             — Poll extraction result
  POST   /v1/annotate                         — Per-image / document-level annotation

Schemas:
  POST   /v1/schemas                          — Create extraction schema
  GET    /v1/schemas                           — List schemas
  GET    /v1/schemas/{id}                     — Get schema
  PUT    /v1/schemas/{id}                     — Update schema (create_new_version)
  DELETE /v1/schemas/{id}                     — Delete schema (soft)
  POST   /v1/gen-schemas                      — Auto-generate candidate schemas from checkpoint
  GET    /v1/gen-schemas/{request_id}         — Poll schema generation

Classification & Taxonomies:
  GET    /v1/content-types                    — List content-type tree
  POST   /v1/content-types                    — Action: adopt | define | undefine | define_attr | undefine_attr
  GET    /v1/content-types/templates          — List seed templates
  POST   /v1/files/{id}/facets                — Action: classify | unclassify | set_value | clear_value
  GET    /v1/files/{id}/facets                — Get file classifications + attribute values
  GET    /v1/tags                              — List tags
  POST   /v1/tags                              — Create tag

Embedding (Standalone):
  POST   /v1/embeddings                       — Generate embeddings for arbitrary text

Entity Resolution & Clustering:
  POST   /v1/resolve                          — Entity deduplication
  POST   /v1/equijoin                         — Fuzzy join
  POST   /v1/cluster                          — Hierarchical clustering
  POST   /v1/rank                             — Full sorting by latent attribute

Knowledge Graph:
  POST   /v1/knowledge-graph/build            — Build KG from document(s)
  GET    /v1/knowledge-graph/{id}             — Get graph (NetworkX JSON, stats, provenance)
  GET    /v1/knowledge-graph/{id}/export      — Export graph (csv/cypher/json/html)
  GET    /v1/knowledge-graph/{id}/visualize   — Interactive HTML (Cytoscape/PyVis)
  POST   /v1/visualize/embeddings             — t-SNE 2D visualization of embeddings

Search & Retrieval:
  POST   /v1/search                           — Unified search (semantic/keyword/hybrid/agentic/list)
  POST   /v1/grep                             — Regex pattern matching
  POST   /v1/list-chunks                      — Metadata-only retrieval
  POST   /v1/metadata-facets                  — Aggregate chunk counts by metadata
  POST   /v1/aggregate                        — Aggregate over collection or search results

Answer Generation / RAG:
  POST   /v1/ask                               — RAG: retrieve + generate (streaming-capable)
  POST   /v1/generate                         — Generative search (single_prompt / grouped_task)

Document Transformation:
  POST   /v1/create-document                  — Generate DOCX from markdown (with track changes)
  POST   /v1/fill                              — Fill PDF/image form
  POST   /v1/track-changes                    — Extract redlines from DOCX
  GET    /v1/thumbnails/{lookup_key}          — Page thumbnails

Pipelines & Workflows:
  POST   /v1/pipelines                        — Create pipeline (versioned)
  GET    /v1/pipelines                        — List pipelines
  GET    /v1/pipelines/{id}                   — Get pipeline (version)
  POST   /v1/pipelines/{id}/run              — Execute pipeline
  GET    /v1/pipelines/executions/{id}       — Get execution status + per-step results
  GET    /v1/pipelines/executions            — List executions
  POST   /v1/pipelines/{id}/optimize         — Offline MCTS optimization (MOAR)
  POST   /v1/workflows                        — Create workflow definition (Temporal-style)
  POST   /v1/workflows/{id}/execute          — Execute workflow
  GET    /v1/workflows/{id}/execution        — Get execution status

Custom Processors:
  POST   /v1/custom-processors                — Create AI-generated custom processor
  GET    /v1/custom-processors                — List
  GET    /v1/custom-processors/{id}           — Get (with versions)
  POST   /v1/custom-processors/{id}/iterate  — Iterate (new version)
  POST   /v1/custom-processors/{id}/describe  — Conversational builder
  POST   /v1/custom-processors/{id}/execute   — Execute custom processor
  GET    /v1/custom-processors/{id}/pipelines — List pipelines using this processor

Evaluation:
  POST   /v1/eval-rubrics                     — Create eval rubric
  POST   /v1/eval-rubrics/from-feedback       — Generate rubric from user feedback
  POST   /v1/forge-evals                      — Configuration comparison
  POST   /v1/validate                         — Validate extracted KVPs against ontology

Query Agent (Agentic NL → Operations):
  POST   /v1/query-agent/ask                  — Natural language → answer + sources
  POST   /v1/query-agent/search               — Natural language → raw objects
  POST   /v1/query-agent/ask-stream          — Streaming with progress

MCP Server (AI Assistant Integration):
  WS     /v1/mcp                              — MCP endpoint exposing tools (search, ask, convert, extract, graph_query)
```

---

## Conventions

### Authentication
Bearer token via `Authorization: Bearer <key>`. Keys are scoped to a workspace and role (`owner`/`editor`/`viewer`). For self-hosted vectorizer/generator modules, model provider keys are passed via headers (`X-OPENAI-API-KEY`, `X-COHERE-API-KEY`, etc.).

### Async Pattern
Processing endpoints (`/convert`, `/extract`, `/segment`, `/gen-schemas`, `/create-document`, `/fill`, `/track-changes`, pipeline runs) are asynchronous: the initial `POST` returns `202` with a `request_id` + `request_check_url`. The client polls the check URL until `status` becomes terminal (`completed`/`embedded`/`success` or `failed`/`cancelled`). Alternatively, a `webhook_url` fires on completion.

**Unified status lifecycle:**
```
pending → in_progress → completed | failed | cancelled
```
Index-time file ingestion adds intermediate states: `converting → parsing → embedding → embedded`.

### Pagination
List endpoints accept `limit` (default 50, max 100) and `offset`, or cursor-based pagination with `before`/`after` tokens.

### Errors
All errors return a JSON body:
```json
{
  "error": { "code": "invalid_request", "message": "human-readable message" },
  "request_id": "req_..."
}
```
Common HTTP status codes: `400` invalid request, `401` unauthenticated, `403` forbidden, `404` not found, `409` conflict, `422` validation failed, `429` rate/budget limited (with `Retry-After`), `500`/`503` server errors.

---

## Unified Data Structures

### Workspace Object

```json
{
  "id": "ws_abc123",
  "name": "Contracts 2026",
  "type": "shared | personal",
  "processing_location": "us | eu",
  "embedding_model": "text-embedding-3-large",
  "chunking_config": { "type": "hybrid", "max_chunk_size_tokens": 800, "chunk_overlap_tokens": 400 },
  "contextualization": { "with_metadata": true, "with_file_context": false },
  "expires_after": { "anchor": "last_active_at", "days": 30 },
  "access_mode": "private | public",
  "multi_tenancy": false,
  "created_at": "ISO-8601",
  "updated_at": "ISO-8601"
}
```

| Field | Type | Values/Default | Description | Source |
|-------|------|----------------|-------------|--------|
| `type` | string | `shared`/`personal` | Isolation mode | lighton |
| `processing_location` | string | `us`/`eu` | Data residency (EU may carry premium) | datalab |
| `embedding_model` | string | model id | Text-only or multimodal; may be immutable after creation | google, openai |
| `chunking_config` | object | see ChunkingConfig | Default chunking for files in this workspace | google, openai, mixedbread |
| `contextualization` | object | `{with_metadata, with_file_context}` | Index-time chunk enrichment | mixedbread |
| `expires_after` | object | `{anchor, days}` | Auto-expire dormant stores | openai, mixedbread |
| `multi_tenancy` | boolean | false | Enable per-tenant sharding | weaviate |

### ChunkingConfig Object

```json
{
  "type": "static | hierarchical | hybrid | line_based | markdown | separator",
  "max_chunk_size_tokens": 800,
  "chunk_overlap_tokens": 400,
  "tokenizer": "tiktoken-cl100k",
  "merge_peers": true,
  "merge_list_items": true,
  "repeat_table_header": true,
  "omit_header_on_overflow": false,
  "keep_separator": true,
  "strip_whitespace": false,
  "headers_to_split_on": ["#", "##"],
  "num_splits_to_group": 1
}
```

| Field | Type | Default | Description | Source |
|-------|------|---------|-------------|--------|
| `type` | string | `hybrid` | Chunking strategy (see C1 alternatives) | all |
| `max_chunk_size_tokens` | int | 800 | Max tokens per chunk (100–4096) | openai, google, docling |
| `chunk_overlap_tokens` | int | 400 | Overlap between adjacent chunks | openai, google |
| `tokenizer` | string | — | Tokenizer for alignment | docling, mistral |
| `merge_peers` | boolean | true | Merge undersized adjacent chunks | docling |
| `merge_list_items` | boolean | true | Merge list items into one chunk | docling |
| `repeat_table_header` | boolean | true | Repeat table header on each chunk | docling |
| `headers_to_split_on` | string[] | — | Markdown headers to split on | mistral |

### File Object

```json
{
  "id": "file_123",
  "external_id": "invoices/2026/001",
  "filename": "invoice_jan.pdf",
  "title": "January Invoice",
  "mime_type": "application/pdf",
  "size_bytes": 1048576,
  "workspace_id": "ws_abc123",
  "status": "pending | converting | parsing | embedding | embedded | failed",
  "metadata": { "author": "Acme Corp", "year": 2026 },
  "tags": ["tag_id_1", "tag_id_2"],
  "content_types": ["invoice"],
  "parser": "fast",
  "processing_location": "us",
  "page_count": 12,
  "parse_quality_score": 4.2,
  "created_at": "ISO-8601",
  "updated_at": "ISO-8601"
}
```

| Field | Type | Description | Source |
|-------|------|-------------|--------|
| `external_id` | string | Idempotent ID; re-upload returns existing doc (slash-supported paths) | lighton, mixedbread |
| `status` | string | Ingestion lifecycle state | all |
| `metadata` | object | User file metadata (bare keys); max 16 keys, 256 chars each | openai, mixedbread |
| `tags` | string[] | Flat tag IDs | lighton |
| `parser` | string | Ingestion pipeline version/strategy (`fast`) | lighton, mixedbread |
| `parse_quality_score` | number | 0–5 self-assessment of conversion quality | datalab |

### Chunk Object

```json
{
  "chunk_id": "chk_001",
  "chunk_index": 0,
  "type": "text | image_url | audio_url | video_url | content | image_annotation | summary",
  "content": "...chunk text...",
  "text_hash": "sha256:...",
  "token_count": 642,
  "char_length": 2103,
  "page_number": 1,
  "page_start": 1,
  "page_end": 2,
  "total_pages": 12,
  "start_offset": 0,
  "end_offset": 2103,
  "headings": ["Introduction", "Scope"],
  "images": ["img_id_1"],
  "locator": "page:1:char:0-2103",
  "filename": "invoice_jan.pdf",
  "file_id": "file_123",
  "title": "January Invoice",
  "mime_type": "application/pdf",
  "generated_metadata": { "language": "en", "word_count": 312 },
  "embedding": { "model": "text-embedding-3-large", "dimensions": 3072 }
}
```

| Field | Type | Description | Source |
|-------|------|-------------|--------|
| `locator` | string | `char:{s}-{e}`, `page:{n}:char:{s}-{e}`, `summary:char:0-512` | mistral |
| `generated_metadata` | object | Auto-extracted typed metadata (`generated_metadata.` prefix) | mixedbread |
| `headings` | string[] | Reconstructed header hierarchy for context | mistral, docetl |
| `embedding` | object | Model + dimensions used to embed this chunk | weaviate, openai |

### Embedding Request / Response

```json
// Request
{ "input": "string or string[]", "model": "text-embedding-3-large",
  "dimensions": 1536, "encoding_format": "float | base64" }

// Response
{ "data": [ { "embedding": [0.0123, ...], "index": 0 } ],
  "usage": { "prompt_tokens": 12, "total_tokens": 12 } }
```

| Field | Type | Description | Source |
|-------|------|-------------|--------|
| `dimensions` | int | MRL shortening (≤ model max) | openai |
| `encoding_format` | string | `float`/`base64` | openai |
| `model` | string | e.g. `text-embedding-3-small` (1536d), `text-embedding-3-large` (3072d), `gemini-embedding-2` (multimodal), `mxbai-wholembed-v3` (whole-doc) | openai, google, mixedbread |

### Schema Object

```json
{
  "schema_id": "sch_k8Hx9mP2nQ4v",
  "name": "Invoice fields",
  "description": "Standard invoice extraction schema",
  "schema_json": { "properties": { "invoice_number": {"type": "string"}, "total": {"type": "number"} } },
  "version": 2,
  "version_history": [ {"version": 1, "schema_json": {...}} ],
  "archived": false,
  "created_at": "ISO-8601",
  "updated_at": "ISO-8601"
}
```

| Field | Type | Description | Source |
|-------|------|-------------|--------|
| `schema_id` | string | Stable ID; mutually exclusive with inline `page_schema` on `/extract` | datalab |
| `version` | int | Current version (starts at 1); pin with `schema_version` | datalab |
| `archived` | boolean | Soft-delete flag | datalab |

### Filter (Unified)

```jsonc
// Comparison filter
{ "type": "eq", "key": "author", "value": "Robert Graves" }
{ "type": "gte", "key": "year", "value": 1934 }
{ "type": "in", "key": "file_id", "value": ["file_1", "file_2"] }
{ "type": "like", "key": "title", "value": "*contract*" }

// Compound filter
{ "type": "and", "filters": [ {comparison}, {comparison} ] }
{ "type": "or",  "filters": [ {comparison}, {comparison} ] }
{ "type": "none", "filters": [ {comparison} ] }

// Property filter (built-in fields)
{ "type": "eq", "property": "filename", "value": "invoice.pdf" }

// AIP-160 string (google-style)
{ "metadata_filter": "author=\"Robert Graves\" AND year>=1934" }
```

**Three metadata layers:** User file metadata (bare keys), generated chunk metadata (`generated_metadata.` prefix), system fields (`file_id`, `chunk_index`). Dot notation for nested fields. Comparison operators: `eq`, `ne`, `gt`, `gte`, `lt`, `lte`, `in`, `nin`, `like`, `not_like`. Compound operators: `and`, `or`, `all` (AND), `any` (OR), `none` (NOT).

### Citation Object

```json
{
  "type": "file_citation | block_id | cite_marker | chunk_ref | provenance",
  "file_id": "file_123",
  "file_name": "invoice_jan.pdf",
  "page_number": 2,
  "page_start": 1,
  "page_end": 3,
  "block_id": "/page/2/Text/3",
  "char_start": 450,
  "char_end": 510,
  "media_id": "img_id_1",
  "bbox": [120, 240, 400, 280],
  "index": 3,
  "custom_metadata": {}
}
```

| Field | Type | Description | Source |
|-------|------|-------------|--------|
| `block_id` | string | `/page/0/Text/3` style location | datalab |
| `file_citation` fields | — | `file_name`, `page_number`, `media_id`, `custom_metadata` | google, openai |
| `char_start`/`char_end` | int | Inline character offsets | openai |
| `index` | int | `<cite i="n"/>` marker index → `sources[n]` | mixedbread |
| `bbox` | number[] | Bounding box for UI highlighting | lighton |

### SearchResultItem

```json
{
  "chunk_id": "chk_001",
  "content": "...",
  "score": 0.92,
  "score_breakdown": { "text": 0.88, "vision": 0.0, "keyword": 0.4, "relevance": 0.92 },
  "file_id": "file_123",
  "filename": "invoice_jan.pdf",
  "page_number": 2,
  "bbox": [120, 240, 400, 280],
  "metadata": { "author": "Acme Corp" },
  "media_url": "https://...",
  "citation": { "type": "file_citation", "page_number": 2, "block_id": "/page/2/Text/3" }
}
```

| Field | Type | Description | Source |
|-------|------|-------------|--------|
| `score_breakdown` | object | Per-component scores (text/vision/keyword/multivector/relevance) | lighton |
| `bbox` | number[] | Merged PDF cell bboxes for UI highlighting | lighton |
| `media_url` | string | Downloadable media for image chunks | google |

---

## Endpoint Specifications

### Workspaces

**Create workspace** — `POST /v1/workspaces`
```json
{ "name": "Contracts 2026", "type": "shared", "processing_location": "us",
  "embedding_model": "text-embedding-3-large",
  "chunking_config": { "type": "hybrid", "max_chunk_size_tokens": 800, "chunk_overlap_tokens": 400 },
  "contextualization": { "with_metadata": true, "with_file_context": false },
  "expires_after": { "anchor": "last_active_at", "days": 30 },
  "multi_tenancy": false }
```
Returns `201` with a Workspace Object.

**List workspaces** — `GET /v1/workspaces?limit=50&before=cursor`

---

### Files

**Upload file** — `POST /v1/files` (multipart)
```
file (binary) | file_url (string) | base64_string (string)
filename, title, workspace_id, metadata, external_id, tags[], parser,
parsing_strategy, chunking_strategy, processing_location, webhook_url,
auto_chunk=true, save_checkpoint=true, purpose="assistants"
```
Returns `202` with `{ id, status: "pending", request_check_url }`.

**Resumable/multipart upload:**
- `POST /v1/files/uploads` → `{ upload_id, presigned_urls[], expires_in }`
- `POST /v1/files/uploads/{id}/complete` with parts + ETags → finalizes
- `POST /v1/files/uploads/{id}/abort`

**Pre-chunked ingestion** — `POST /v1/files/prechunked`
```json
{ "file": { "chunks": [{"content": "...", "metadata": {}}] },
  "workspace_id": "ws_abc123", "metadata": {}, "external_id": "docs/x" }
```

**List chunks of a file** — `GET /v1/files/{id}/chunks?return_chunks=true&indices=[]`

---

### Document Parsing & Conversion

**Convert document** — `POST /v1/convert`
```json
{
  "file": "binary | file_url | checkpoint_id",
  "output_format": "md | html | json | chunks | doctags | doclang | docx | pdf | png",
  "mode": "fast | balanced | accurate",
  "page_range": "0,5-10,20",
  "max_pages": 100,
  "paginate": false,
  "add_block_ids": false,
  "word_bboxes": false,
  "table_cell_bboxes": false,
  "list_item_bboxes": false,
  "include_markdown_in_chunks": false,
  "disable_image_extraction": false,
  "disable_image_captions": false,
  "do_ocr": true,
  "force_ocr": false,
  "ocr_lang": "en",
  "ocr_preset": "default",
  "table_mode": "fast | accurate",
  "table_format": "markdown | html | none",
  "include_image_base64": false,
  "include_blocks": false,
  "confidence_scores_granularity": "page | word | none",
  "bbox_annotation_format": {},
  "document_annotation_format": {},
  "extract_header": false,
  "extract_footer": false,
  "extract_links": false,
  "save_checkpoint": false,
  "skip_cache": false,
  "processing_location": "us | eu",
  "extras": ["track_changes", "chart_understanding", "infographic", "new_block_types"],
  "enrichments": ["code", "formula", "picture_classification", "picture_description"],
  "media_resolution": "low | medium | high",
  "additional_config": { "keep_pageheader_in_output": false, "keep_pagefooter_in_output": false, "keep_spreadsheet_formatting": false },
  "webhook_url": "https://...",
  "eval_rubric_id": null
}
```
Returns `202` with `{ request_id, request_check_url }`.

**Poll result** — `GET /v1/convert/{request_id}` → `200` when complete:
```json
{
  "status": "completed",
  "success": true,
  "output_format": "markdown",
  "markdown": "...",
  "html": null,
  "json": null,
  "chunks": null,
  "images": { "img_1": "base64..." },
  "metadata": {},
  "page_count": 12,
  "parse_quality_score": 4.2,
  "cost_breakdown": {},
  "checkpoint_id": "chk_abc",
  "runtime": 8.3,
  "result_url": "https://...",
  "expires_in": 3600,
  "evaluation": null
}
```

**Checkpoint reuse:** `checkpoint_id` from a prior `save_checkpoint=true` conversion can be passed to `/convert`, `/extract`, `/segment`, `/gen-schemas` to skip re-parsing.

---

### Document Segmentation

**Segment document** — `POST /v1/segment`
```json
{
  "file": "binary | file_url | checkpoint_id",
  "segmentation_schema": { "segments": [{"name": "Invoice", "description": "..."}], "segmentation_strategy": "custom | document_boundary" },
  "mode": "fast",
  "page_range": "0,5-10,20",
  "save_checkpoint": false,
  "skip_cache": false,
  "webhook_url": "https://..."
}
```
**Output:**
```json
{ "segmentation_results": { "segments": [
  {"name": "Invoice", "pages": [0,1,2], "confidence": "high"},
  {"name": "Receipt", "pages": [3,4], "confidence": "medium"}
], "metadata": {"total_pages": 5, "segmentation_method": "auto_detected"} } }
```

---

### Data Extraction

**Extract structured fields** — `POST /v1/extract`
```json
{
  "file": "binary | file_url | checkpoint_id",
  "page_schema": {} | "schema_id": "sch_...",
  "schema_version": 2,
  "extraction_mode": "turbo | fast | balanced",
  "mode": "fast | balanced | accurate",
  "output_format": "markdown",
  "page_range": "0,5-10,20",
  "save_checkpoint": false,
  "skip_cache": false,
  "webhook_url": "https://...",
  "docClass": "invoice",
  "jsonOptions": "KVP,SN,MT,CHAR"
}
```
**Output (balanced mode):**
```json
{
  "company_name": "Whitbread PLC",
  "company_name_citations": ["/page/0/Text/3", "/page/2/Table/1"],
  "company_name_meta": {
    "extraction_status": "EXTRACTED | NOT_RESOLVABLE",
    "reasoning": "The company name appears...",
    "citations": ["/page/0/Text/3"],
    "verification": { "status": "PASS | FAIL_UNRESOLVABLE | FAIL_FIX | FAIL_CITATIONS | ITEMS_MISSING", "feedback": "..." }
  },
  "extraction_score_average": 4.5
}
```
**Output (fast mode):**
```json
{
  "invoice_number": "INV-2024-001",
  "invoice_number_citations": ["block_123"],
  "invoice_number_score": { "score": 5, "reasoning": "Value found verbatim..." },
  "extraction_score_average": 4.5
}
```

**Annotations** — `POST /v1/annotate`
```json
{ "file": "binary | file_url",
  "bbox_annotation_format": {} | "document_annotation_format": {},
  "include_image_base64": false, "webhook_url": "..." }
```

**Schema auto-generation** — `POST /v1/gen-schemas` with `{ checkpoint_id }` → returns `simple_schema`, `moderate_schema`, `complex_schema`.

---

### Classification & Taxonomies

**Content-type tree** — `GET /v1/content-types?query=&path=&depth=4&include_attributes=true`
**Actions** — `POST /v1/content-types` with `{ action: "adopt | define_content_type | undefine_content_type | define_attribute | undefine_attribute", ... }`
**File facet actions** — `POST /v1/files/{id}/facets` with `{ action: "classify | unclassify | set_value | clear_value", content_type_path, attribute_name, value }`

---

### Embedding (Standalone)

`POST /v1/embeddings` with Embedding Request → Embedding Response (see Unified Data Structures).

---

### Entity Resolution & Clustering

**Resolve** — `POST /v1/resolve`
```json
{ "data": [], "comparison_prompt": "...", "resolution_prompt": "...",
  "blocking_keys": [], "blocking_threshold": 0.8, "blocking_target_recall": 0.95,
  "blocking_conditions": [], "embedding_model": "...", "cascade": {"proxy_model": "...", "guarantee": "recall", "target": 0.95, "delta": 0.05} }
```
**Equijoin** — `POST /v1/equijoin` with `{ left, right, comparison_prompt, limits, blocking_keys, blocking_threshold, cascade }`
**Cluster** — `POST /v1/cluster` with `{ data, embedding_keys, summary_prompt, summary_schema, embedding_model }`
**Rank** — `POST /v1/rank` with `{ data, prompt, input_keys, direction: "asc|desc", initial_ordering_method: "likert|embedding", call_budget, k }`

---

### Knowledge Graph

**Build KG** — `POST /v1/knowledge-graph/build`
```json
{
  "source": "file_id | checkpoint_id",
  "template": "Pydantic class or dotted path",
  "processing_mode": "one-to-one | many-to-one",
  "extraction_contract": "auto | direct | dense",
  "dense_config": { "skeleton_batch_tokens": 8000, "fill_nodes_cap": 50, "fill_context": "...", "dedupe": "standard" },
  "backend": "llm | vlm",
  "inference": "local | remote",
  "use_chunking": true,
  "chunk_max_tokens": 800,
  "provenance": "off | standard | detailed",
  "gleaning_enabled": false,
  "parallel_workers": 4,
  "export_format": "csv | cypher | json | html",
  "export_docling": false,
  "export_markdown": false,
  "export_doclang": false
}
```
**Output:** `{ graph_id, node_count, edge_count, node_types, edge_types, avg_degree, density, provenance_ledger, bind_stats }`

**Export** — `GET /v1/knowledge-graph/{id}/export?format=cypher`
**Visualize** — `GET /v1/knowledge-graph/{id}/visualize` → interactive HTML

---

### Search

**Unified search** — `POST /v1/search`
```json
{
  "query": "what is the termination clause?",
  "workspace_id": "ws_abc123",
  "store_identifiers": [],
  "top_k": 10,
  "max_results": 10,
  "mode": "text | vision",
  "search_type": "semantic | keyword | hybrid | agentic | grep | list",
  "alpha": 0.5,
  "fusion_type": "relative_score | ranked",
  "hybrid_search": { "embedding_weight": 0.7, "text_weight": 0.3 },
  "ranking_options": { "ranker": "rrf", "score_threshold": 0.5 },
  "rerank": false | { "model": "mxbai-rerank-large-v2", "top_k": 20, "with_metadata": ["author"] },
  "rewrite_query": false,
  "agentic": false | { "max_rounds": 3, "queries_per_round": 4, "instructions": "prefer recent docs", "strict_top_k": true, "score_threshold": 0.5 },
  "relevance_scoring": "none | scoring_only | scoring_and_filtering",
  "filters": { "type": "and", "filters": [{"type": "eq", "key": "year", "value": 2026}] },
  "content_type": ["invoice:*"],
  "attribute": ["status:paid"],
  "tag_id": ["tag_1"],
  "file_ids": ["file_1"],
  "target_vector": "default",
  "distance": 0.7,
  "certainty": 0.9,
  "auto_limit": 1,
  "autocut": 1,
  "move_to": { "concepts": ["contract"], "force": 0.3 },
  "move_away": { "concepts": ["draft"], "force": 0.2 },
  "selection": { "type": "mmr", "balance": 0.5 },
  "boost": { "prop": "recency", "weight": 0.1 },
  "group_by": { "prop": "filename", "objects_per_group": 3, "number_of_groups": 5 },
  "return_properties": ["content"],
  "return_references": true,
  "return_metadata": true,
  "include_image": false,
  "include_bboxes": false,
  "media_content": "auto | always | never"
}
```
**Response:**
```json
{
  "results": [ { "chunk_id": "...", "content": "...", "score": 0.92, "score_breakdown": {}, "file_id": "...", "page_number": 2, "bbox": [], "citation": {} } ],
  "search_query": "termination clause (rewritten)",
  "total_count": 42
}
```

**Grep** — `POST /v1/grep` with `{ store_identifiers[], pattern: "RE2 regex", top_k, content_groups: ["text","generated"], case_sensitive, file_ids[], filters, return_metadata }`
**List chunks** — `POST /v1/list-chunks` with `{ store_identifiers[], top_k, file_ids[], sort_by, filters, return_metadata }`
**Facets** — `POST /v1/metadata-facets` with `{ store_identifiers[], query, top_k, filters, facets[] }`
**Aggregate** — `POST /v1/aggregate` with `{ workspace_id, query, total_count: true, return_metrics: {count, sum, max, min, mean, median, mode, top_occurrences, percentageTrue, percentageFalse, reference_count}, group_by: {prop, objects_per_group, number_of_groups}, filters, distance, object_limit }`

---

### Answer Generation / RAG

**Ask** — `POST /v1/ask`
```json
{
  "query": "summarize the termination clause",
  "messages": [{"role": "user", "content": "..."}],
  "workspace_id": "ws_abc123",
  "store_identifiers": [],
  "model": "gpt-4o",
  "max_results": 10,
  "stream": true,
  "instructions": "answer in bullet points",
  "qa_options": { "cite": true, "multimodal": false, "stream": true },
  "search_options": { "rerank": true, "rewrite_query": true, "agentic": false, "score_threshold": 0.5 },
  "filters": {},
  "content_type": [], "attribute": [], "tag_id": [], "file_ids": [],
  "response_format": { "type": "text", "mime_type": "application/json", "schema": {} }
}
```
**Response (non-streaming):**
```json
{ "answer": "The termination clause states...<cite i=\"0\"/>", "sources": [ { "chunk_id": "...", "content": "...", "file_id": "...", "page_number": 2 } ] }
```
**Response (streaming):** SSE token events + final answer object (OpenAI-compatible).

**Generative search** — `POST /v1/generate`
```json
{ "query": "...", "workspace_id": "ws_abc123", "search_type": "hybrid", "top_k": 10,
  "single_prompt": "Summarize: {content}", "grouped_task": "...", "grouped_properties": ["filename"],
  "generative_provider": "openai" }
```

---

### Document Transformation

**Create document** — `POST /v1/create-document` (JSON body)
```json
{ "markdown": "...with <ins data-revision-author=\"Jane\">new</ins> text...",
  "output_format": "docx", "webhook_url": "...", "processing_location": "us" }
```
**Output:** `{ output_base64: "...", page_count, runtime, cost_breakdown }`

**Form fill** — `POST /v1/fill`
```json
{ "file": "binary | file_url", "field_data": {"name": {"value": "John", "description": "full name"}},
  "context": "Initial hire for new employee",
  "confidence_threshold": 0.5, "page_range": "0,1", "output_format": "pdf | png" }
```
**Output:** `{ output_base64, fields_filled: ["name"], fields_not_found: [], page_count, runtime, cost_breakdown }`

**Track changes** — `POST /v1/track-changes`
```json
{ "file": "binary | file_url", "output_format": "md,html,chunks", "page_range": "...", "paginate": false, "webhook_url": "..." }
```
**Thumbnails** — `GET /v1/thumbnails/{lookup_key}?page_range=0,1&thumb_width=200&track_changes=false`

---

### Pipelines & Workflows

**Create pipeline** — `POST /v1/pipelines`
```json
{
  "name": "Invoice extraction pipeline",
  "steps": [
    { "type": "convert", "settings": {"mode": "balanced", "save_checkpoint": true}, "eval_rubric_id": null },
    { "type": "segment", "settings": {"segmentation_schema": {...}}, "eval_rubric_id": null },
    { "type": "extract", "settings": {"page_schema": {...}, "extraction_mode": "balanced"}, "eval_rubric_id": 5 },
    { "type": "custom", "custom_processor_id": "cp_abc", "settings": {} },
    { "type": "fill", "settings": {} },
    { "type": "map | filter | reduce | resolve | cluster | rank | split | gather | unnest | code_map | code_reduce | code_filter | parallel_map | equijoin | link_resolve | kg_build", "settings": {} }
  ]
}
```
**Run pipeline** — `POST /v1/pipelines/{id}/run`
```json
{ "file": "binary | file_url", "version": 0, "page_range": "...", "output_format": "markdown",
  "run_evals": false, "skip_cache": false, "checkpoint_id": "chk_abc", "webhook_url": "..." }
```
**Poll execution** — `GET /v1/pipelines/executions/{id}`
```json
{
  "execution_id": "exec_123", "pipeline_id": "pl_abc", "pipeline_version": 2,
  "status": "pending | running | completed | completed_with_errors | failed",
  "steps": [ { "step_index": 0, "step_type": "convert", "status": "completed", "lookup_key": "...", "result_url": "https://...", "checkpoint_id": "chk_abc", "error_message": null } ],
  "started_at": "ISO-8601", "completed_at": null, "rate_breakdown": {}
}
```

**Optimize pipeline (MOAR)** — `POST /v1/pipelines/{id}/optimize`
```json
{ "eval_fn": "...", "metric_key": "accuracy", "models": [], "max_iterations": 100, "save_dir": "..." }
```
**Output:** Pareto-optimal cost-accuracy frontier; `.best()`, `.cheapest()`, `.frontier`.

---

### Evaluation

**Create eval rubric** — `POST /v1/eval-rubrics` with `{ name, rules: [{type: "block|page|document", ...}], scoring: "0-5" }`
**Generate from feedback** — `POST /v1/eval-rubrics/from-feedback` with `{ feedback: [...] }`
**Forge evals** — `POST /v1/forge-evals` with `{ documents: [max 10], configs: [max 5], iterations: 3 }`
**Validate KVPs** — `POST /v1/validate` with `{ project_id, data }` → `{ ValidatorResult: "Pass|Fail", ValidatorFailures: [] }`

---

### Query Agent

**Ask** — `POST /v1/query-agent/ask`
```json
{
  "query": "show me all invoices over $10k from 2026",
  "messages": [],
  "collections": [ { "name": "invoices", "target_vector": "default", "view_properties": ["amount","date"], "tenant": "customerA", "additional_filters": {} } ],
  "result_evaluation": "none | llm",
  "timeout": 30
}
```
**Search** — `POST /v1/query-agent/search` with `{ query, collections, limit, filtering: "recall|precision", diversity_weight }`
**Streaming** — `POST /v1/query-agent/ask-stream` with `{ ..., include_progress: true, include_final_state: true }`

---

### Tenancy

`POST /v1/workspaces/{id}/tenants` with `{ tenants: [{name: "customerA"}, {name: "customerB"}] }`
All standard operations (insert, query, aggregate, generate) accept a `tenant` header/param for scoped access. Tenant lifecycle: `ACTIVE → INACTIVE → OFFLOADED`.

---

### MCP Server

`WS /v1/mcp` exposes tools for AI assistant integration (Claude Code, Cursor, Windsurf):
- `search` — semantic/keyword/hybrid search
- `ask` — RAG question answering
- `convert` — document parsing
- `extract` — structured extraction
- `graph_query` — Cypher/graph queries (if graph store enabled)

---

# Part IV — Cross-Reference & Coverage

## 1. Naming Variant Cross-Reference Table

(See §3 Cross-Provider Synonym Glossary in Part I.)

## 2. Alternative-Approaches Cross-Reference Table

| Processing Step | Alternative Approaches |
|---|---|
| **File upload** | Multipart upload · URL-based · Base64 inline · Presigned-URL · Resumable/multipart · Cloud-storage loaders (S3/Azure/GCS/Drive/SharePoint) · Local FS/directory · In-memory/stream · Pre-chunked (MXJSON) · Docling JSON round-trip · `datalab://` references · JSON object insertion |
| **Parsing** | Multi-model pipeline (layout+OCR+table) · Single end-to-end VLM · Native multimodal vision · Managed automatic · Dedicated OCR API · Word-level OCR with font metadata · Format-specific backends · Audio/video ASR · Legacy office conversion |
| **Segmentation** | Schema-guided · Automatic boundary detection · Page-structure segmentation |
| **Chunking** | Static/token-count · Character-count · Separator/hierarchical · Markdown/header-aware · Hierarchical/structure-pure · Hybrid/tokenization-aware · Line-based · Word-count with overlap · Structure-preserving · Automatic/managed · Pre-chunked (bypass) · Gather (context enrichment) |
| **Chunk enrichment** | Summary enrichment · Contextualization at index time · Gather (context windows) · Generated metadata · Custom enrichers |
| **Data extraction** | JSON-schema-driven LLM extraction · BBox annotation · Document annotation · KVP with ontology · Table/line-item extraction · Semantic normalization · Verbatim text extraction · Form filling · Schema auto-generation · Pydantic-template extraction · Dense extraction |
| **Classification** | AI document classification · Hierarchical taxonomy (Facets) · Flat tags · Custom processor classification · LLM map with enum · Filter-based · Picture classification · Zero-shot via embeddings |
| **Entity resolution** | Blocking+LLM comparison+union-find · Deterministic node ID registry · LLM entity standardization · Dense dedupe · Link resolve · Equijoin (fuzzy join) · Cross-references · BARGAIN cascade |
| **Clustering** | Hierarchical agglomerative · KMeans · Louvain community detection · Value sampling cluster · t-SNE visualization |
| **Embedding** | Auto-vectorization on insert · Managed automatic · Standalone embedding API · Configurable embedder · Named vectors (multi-model) · Multi-vector (ColBERT/ColPali) · Multimodal embeddings · Vision (VLM) embeddings · Whole-document embeddings · Self-provided vectors · Embeddings as ML features |
| **Indexing** | Managed vector store · Self-hosted vector DB (HNSW/Flat/Dynamic/HFresh) · Vespa · LanceDB · Native graph DB · File-based · DPE database · S3 integration · BYOB |
| **Query rewriting** | Query rewriting (observable) · LLM query rewriter · LLM query extension (sub-queries) · Model-generated queries |
| **Search** | BM25/BM25F keyword · Grep regex · near_text/near_vector/near_object/near_image · Semantic via embeddings · Hybrid (alpha-weighted) · RRF · Hybrid vector+keyword+vision · Hybrid web+internal · Agentic search · Query Agent · Model-autonomous File Search · List chunks · Multi-store/federated · Cross-collection Explore · Cross-document reasoning |
| **Filtering** | Comparison filters · Compound filters · AIP-160 expressions · Weaviate filter operators · Cross-reference filtering · Nested object filtering · Content-type/facet filters · Tag filters · File ID scoping |
| **Reranking** | Cross-encoder (pointwise) · Listwise (instruction-steerable) · LLM reranking · RRF fusion · Relevance scoring modes · Sequential chaining · Boost (soft ranking) |
| **Caching** | Semantic cache · Result caching · LLM call caching · Memoized terminal actions · Persistent media IDs · HNSW snapshots |
| **RAG / QA** | Hosted RAG (model-autonomous) · Managed RAG (one-call) · Two-stage retrieve+generate · Generative search in search calls · Retrieval API + manual synthesis · Document QnA · RAG via retriever+map/reduce · GraphRAG · Query Agent Ask · Structured grounded output |
| **Aggregation** | Aggregate queries · Grouped search (GroupBy) · Reduce operator · Facets · Rank operator |
| **Knowledge graph** | Schema-validated Pydantic pipeline · Schema-free LLM triple extraction · Native graph DB storage · Link resolve · Cross-references |
| **Document transformation** | DOCX generation · Form filling · Track changes extraction · Document round-trip · Thumbnails · Synthetic data generation |
| **Orchestration** | Declarative pipelines (YAML/Python) · Temporal workflows · Declarative map-reduce framework · Pipeline + RoutedPipeline · QueryEngine + CachedQueryEngine · run_pipeline · MCP server · Automatic managed pipeline |
| **Evaluation** | Parse quality score · Eval rubrics · Forge Evals · Custom processor evals · Per-field verification · Per-field confidence · KVP validation · Gleaning · Validate · Calibration · Plan rewrites · BARGAIN cascades · MOAR MCTS optimization · Operation-level optimize · Semantic cache metrics |

---

## 3. End-to-End Reference Flows

### Flow 1: Simple Document → Search → Answer (Managed RAG)
```
1. POST /v1/files (upload PDF to workspace)
2. Poll GET /v1/files/{id} until status = "embedded"
3. POST /v1/ask { query, workspace_id, stream: true }
   → SSE answer with <cite> markers + sources[]
```

### Flow 2: Document → Structured Extraction with Verification
```
1. POST /v1/convert { save_checkpoint: true }
2. POST /v1/gen-schemas { checkpoint_id } → pick schema tier
3. POST /v1/extract { checkpoint_id, page_schema, extraction_mode: "balanced" }
   → per-field values + citations + verification (PASS/FAIL) + reasoning
```

### Flow 3: Multi-Document PDF → Segment → Extract Per Section
```
1. POST /v1/convert { save_checkpoint: true }
2. POST /v1/segment { checkpoint_id, segmentation_strategy: "document_boundary" }
3. For each segment: POST /v1/extract { checkpoint_id, page_range: segment.pages, page_schema }
```

### Flow 4: Build Knowledge Graph from Documents
```
1. POST /v1/knowledge-graph/build { source, template, processing_mode: "many-to-one",
     extraction_contract: "dense", provenance: "detailed", export_format: "cypher" }
2. Load Cypher script into Neo4j
3. GraphRAG: vector search + graph traversal for multi-hop reasoning
```

### Flow 5: Declarative Transformation Pipeline (docetl-style)
```
1. Define pipeline YAML: read_json → map (extract entities) → resolve (dedup) → cluster → reduce (summarize per cluster)
2. POST /v1/pipelines (create)
3. POST /v1/pipelines/{id}/run { file }
4. Poll GET /v1/pipelines/executions/{id} for per-step results
```

### Flow 6: Hybrid Search with Reranking + Agentic
```
1. POST /v1/search { query, search_type: "hybrid", alpha: 0.5,
     rerank: { model: "mxbai-rerank-large-v2", top_k: 20 },
     agentic: { max_rounds: 3, queries_per_round: 4, instructions: "prefer recent docs" },
     filters: { type: "and", filters: [...] }, top_k: 10 }
```

### Flow 7: Document Round-Trip (DOCX → Redlines → DOCX)
```
1. POST /v1/track-changes { file (DOCX) } → markdown with <ins>/<~~>/<comment>
2. (Edit markdown)
3. POST /v1/create-document { markdown } → DOCX with revision marks
```

### Flow 8: Form Filling
```
1. POST /v1/fill { file (PDF form), field_data: { "name": {value: "John"}, ... },
     confidence_threshold: 0.7, output_format: "pdf" }
```

### Flow 9: Cost-Optimized Pipeline (BARGAIN + MOAR)
```
1. Define pipeline with filter operator + cascade: { proxy_model, guarantee: "recall", target: 0.95, delta: 0.05 }
2. POST /v1/pipelines/{id}/optimize { eval_fn, metric_key } → Pareto frontier
3. Select .best() or .cheapest() configuration
```

### Flow 10: Multi-Tenant Isolated RAG
```
1. Create workspace with multi_tenancy: true
2. POST /v1/workspaces/{id}/tenants { tenants: [{name: "customerA"}, {name: "customerB"}] }
3. Insert + search + ask with tenant header per customer
```

---

## 4. Coverage Matrix

This matrix shows which systems contribute to each pipeline stage. ✅ = native support; ➖ = achievable via composition; ❌ = not supported.

| Stage | datalab | docetl | docling | google | ibm | kg | lighton | mistral | mixedbread | openai | weaviate |
|---|---|---|---|---|---|---|---|---|---|---|---|
| File upload | ✅ | ✅ | ✅ | ✅ | ✅ | ✅ | ✅ | ✅ | ✅ | ✅ | ✅ |
| Parsing/OCR | ✅ | ➖ | ✅ | ✅ | ✅ | ✅ | ✅ | ✅ | ✅ | ✅ | ❌ |
| Segmentation | ✅ | ❌ | ❌ | ❌ | ✅ | ❌ | ❌ | ❌ | ❌ | ❌ | ❌ |
| Chunking | ✅ | ✅ | ✅ | ✅ | ➖ | ✅ | ✅ | ✅ | ✅ | ✅ | ❌ |
| Chunk enrichment | ➖ | ✅ | ➖ | ✅ | ❌ | ➖ | ✅ | ✅ | ✅ | ➖ | ❌ |
| Data extraction | ✅ | ✅ | ✅ | ✅ | ✅ | ✅ | ✅ | ✅ | ✅ | ✅ | ➖ |
| Classification | ✅ | ✅ | ➖ | ➖ | ✅ | ❌ | ✅ | ➖ | ➖ | ✅ | ➖ |
| Entity resolution | ➖ | ✅ | ❌ | ❌ | ➖ | ✅ | ❌ | ❌ | ❌ | ❌ | ➖ |
| Clustering | ❌ | ✅ | ❌ | ❌ | ❌ | ✅ | ❌ | ❌ | ❌ | ✅ | ➖ |
| Embedding | ❌ | ✅ | ❌ | ✅ | ❌ | ❌ | ✅ | ✅ | ✅ | ✅ | ✅ |
| Indexing/storage | ✅ | ✅ | ❌ | ✅ | ✅ | ✅ | ✅ | ✅ | ✅ | ✅ | ✅ |
| Query rewriting | ❌ | ✅ | ❌ | ❌ | ❌ | ❌ | ❌ | ✅ | ✅ | ✅ | ❌ |
| Search (lexical) | ❌ | ✅ | ❌ | ❌ | ❌ | ✅ | ✅ | ✅ | ✅ | ✅ | ✅ |
| Search (vector) | ❌ | ✅ | ❌ | ✅ | ❌ | ✅ | ✅ | ✅ | ✅ | ✅ | ✅ |
| Search (hybrid) | ❌ | ✅ | ❌ | ❌ | ❌ | ✅ | ✅ | ✅ | ✅ | ✅ | ✅ |
| Search (agentic) | ❌ | ❌ | ❌ | ❌ | ❌ | ❌ | ❌ | ❌ | ✅ | ✅ | ✅ |
| Filtering | ❌ | ✅ | ❌ | ✅ | ❌ | ✅ | ✅ | ✅ | ✅ | ✅ | ✅ |
| Reranking | ❌ | ✅ | ❌ | ❌ | ❌ | ❌ | ✅ | ✅ | ✅ | ❌ | ✅ |
| Caching | ✅ | ✅ | ❌ | ✅ | ❌ | ❌ | ❌ | ✅ | ❌ | ❌ | ✅ |
| RAG / QA | ➖ | ✅ | ❌ | ✅ | ❌ | ✅ | ✅ | ✅ | ✅ | ✅ | ✅ |
| Knowledge graph | ❌ | ✅ | ❌ | ❌ | ❌ | ✅ | ❌ | ❌ | ❌ | ❌ | ➖ |
| Aggregations | ❌ | ✅ | ❌ | ❌ | ❌ | ✅ | ❌ | ❌ | ✅ | ❌ | ✅ |
| Doc transformation | ✅ | ✅ | ❌ | ❌ | ❌ | ❌ | ❌ | ❌ | ❌ | ❌ | ❌ |
| Orchestration | ✅ | ✅ | ➖ | ➖ | ➖ | ✅ | ➖ | ✅ | ➖ | ➖ | ➖ |
| Evaluation | ✅ | ✅ | ❌ | ❌ | ✅ | ❌ | ❌ | ❌ | ❌ | ❌ | ❌ |
| Provenance/citations | ✅ | ➖ | ✅ | ✅ | ✅ | ✅ | ✅ | ✅ | ✅ | ✅ | ✅ |
| Visualization | ❌ | ✅ | ✅ | ❌ | ❌ | ✅ | ❌ | ❌ | ❌ | ✅ | ❌ |
| Multi-tenancy | ✅ | ❌ | ❌ | ❌ | ❌ | ❌ | ✅ | ❌ | ✅ | ❌ | ✅ |
| Data residency | ✅ | ❌ | ❌ | ❌ | ❌ | ❌ | ❌ | ❌ | ✅ | ❌ | ❌ |

---

> **End of specification.** This document represents the union of all capabilities described across the eleven surveyed systems. Individual systems implement subsets; the unified API specification in Part III encompasses the full feature surface. Refer to individual system files for implementation-specific details, exact endpoint signatures, and system-specific constraints.