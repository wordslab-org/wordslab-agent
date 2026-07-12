# Unified Document Intelligence API — Aggregated Specification

> **Scope.** This document aggregates the capabilities, concepts, APIs, and use cases of **eleven** document-intelligence / document-RAG / document-KG systems surveyed in this directory (`datalab`, `docetl`, `docling`, `google`, `ibm`, `knowledgegraph`, `lighton`, `mistral`, `mixedbread`, `openai`, `weaviate`). It performs the **union** of every capability described, orders them along an **exhaustive end-to-end processing pipeline**, flags where **different names are used for the same concept**, and flags where **different approaches can be chosen for the same step**. The final part is a **detailed specification of a super-complete API** that would encompass all the features of the individual systems, written for the end user.
>
> Each source system is referenced by its short name in parentheses, e.g. `(weaviate)`, `(datalab)`, `(docling)`.

---

## Table of Contents

1. [Introduction & Concepts](#1-introduction--concepts)
2. [The Unified Processing Pipeline (Overview)](#2-the-unified-processing-pipeline-overview)
3. [Detailed Pipeline Stages](#3-detailed-pipeline-stages)
   - 3.1 File Upload & Ingestion
   - 3.2 Document Parsing, OCR & Layout Analysis
   - 3.3 Document Segmentation / Boundary Detection
   - 3.4 Chunking / Splitting
   - 3.5 Chunk Enrichment & Contextualization
   - 3.6 Data Extraction (Fields, Tables, Key-Value Pairs, Annotations)
   - 3.7 Classification & Categorization (Taxonomies, Facets, Tags)
   - 3.8 Entity Detection, Resolution, Deduplication & Linking
   - 3.9 Clustering
   - 3.10 Embedding / Vectorization
   - 3.11 Indexing & Storage
   - 3.12 Search (Lexical, Vector, Hybrid, Agentic)
   - 3.13 Filtering, Facets & Metadata Scoping
   - 3.14 Reranking
   - 3.15 Query Rewriting & Preprocessing
   - 3.16 Caching
   - 3.17 Answer Generation / RAG / QA
   - 3.18 Knowledge Graph Building
   - 3.19 Aggregations, Grouped Search & Analytics
   - 3.20 Document Transformation & Round-trip (Generation, Form Filling, Track Changes)
   - 3.21 Orchestration, Pipelines & Workflows
   - 3.22 Evaluation, Quality Assurance & Optimization
   - 3.23 Provenance, Citations & Source Tracking
   - 3.24 Visualization
   - 3.25 Multi-tenancy, Security, Residency & Administration
4. [Naming Variant Cross-Reference Table](#4-naming-variant-cross-reference-table)
5. [Alternative-Approaches Cross-Reference Table](#5-alternative-approaches-cross-reference-table)
6. [Unified API Specification](#6-unified-api-specification)
7. [End-to-End Reference Flows](#7-end-to-end-reference-flows)
8. [Coverage Matrix](#8-coverage-matrix)

---

## 1. Introduction & Concepts

### 1.1 What is "Document Intelligence"?

Document Intelligence is the set of techniques that turn **unstructured or semi-structured documents** (PDFs, images, Office files, HTML, email, spreadsheets, audio, video, code, XML schemas) into **structured, machine-readable, searchable, and answerable** data. It spans the entire journey from a raw file on disk to a grounded answer in a chatbot, covering ingestion, parsing, layout understanding, chunking, enrichment, extraction, classification, entity resolution, embedding, indexing, retrieval, reranking, generation, knowledge-graph construction, and evaluation.

### 1.2 The Two Archetypes

The surveyed systems cluster into (often overlapping) archetypes:

| Archetype | Description | Examples |
|---|---|---|
| **Document Parsing / Conversion** | Turns raw files into Markdown/JSON/HTML/chunks with layout, tables, bboxes. | docling, datalab, mistral-OCR, ibm |
| **Managed Document-RAG / Search Platform** | Upload → auto-parse → chunk → embed → index → semantic search → grounded answers, zero infrastructure. | google (File Search), lighton, mixedbread, openai (Vector Stores) |
| **Vector Database / Search Engine** | Store pre-chunked objects + vectors; provide vector/keyword/hybrid search, RAG, reranking, aggregations. | weaviate |
| **Declarative Transformation Framework** | Map-reduce-style pipelines over documents with LLM operators (map, filter, reduce, resolve, cluster, rank, extract). | docetl |
| **Knowledge-Graph Builder** | Document → entities + relationships → graph (NetworkX / Neo4j) with provenance. | knowledgegraph (Docling-Graph, AI-Knowledge-Graph, Neo4j) |
| **Enterprise Capture / Extraction** | Classification + OCR + KVP/table extraction with ontology, validation, verification. | ibm (DPE) |

A **super-complete** system would encompass all of these archetypes in one coherent API.

### 1.3 Core Concepts Glossary (Union)

These are the concepts you need to understand to navigate the rest of this document. Where systems use different names for the same concept, the variants are listed.

- **Document / File / Data Object** — the unit of input. Variants: *File* `(datalab, openai, mistral, mixedbread)`, *Data Object* `(weaviate)`, *Document* `(google, docling)`, *Item* / *row* `(docetl)`, *Analyzer* (a processing job) `(ibm)`.
- **Workspace / Store / Collection / Index / Project** — a named container holding documents and their index. Variants: *Workspace* `(lighton)`, *Store* / *search index* `(mixedbread)`, *Vector Store* `(openai)`, *FileSearchStore* `(google)`, *Collection* / *schema* / *class* `(weaviate)`, *Project* `(ibm)`, *Dataset* / *Frame* `(docetl)`.
- **Chunk** — a retrievable text/media segment produced from a document. Variants: *Chunk* `(weaviate, mixedbread, mistral, openai)`, *DocumentChunk* `(mistral)`, *block* `(datalab, docling, ibm)`, *content item* `(docling)`.
- **Embedding / Vector** — a numeric array representing semantic content. Variants: *Vector* `(weaviate, openai)`, *embedding* `(google, mistral, mixedbread, lighton, docetl)`.
- **Vectorizer / Embedder / Embedding model** — the component that produces vectors. Variants: *Vectorizer module* `(weaviate)`, *Embedder* `(mistral)`, *Embedding model* `(google, openai)`.
- **Schema / Template / Ontology** — a definition of the structure to extract or enforce. Variants: *extraction schema* / *page_schema* / *schema_id* `(datalab)`, *Pydantic template* `(knowledgegraph)`, *ontology* / *KeyClass* `(ibm)`, *JSON Schema* `(google, lighton, mixedbread, mistral)`, *output schema* `(docetl)`, *collection schema* `(weaviate)`.
- **Citation / Provenance / Source tracking** — linking outputs back to source locations. Variants: *file_citation* `(google, openai)`, *citations* / *block IDs* `(datalab)`, *provenance* / *ProvenanceLedger* `(knowledgegraph, datalab)`, *source attribution* `(lighton, mixedbread)`.
- **RAG / Generative search / Ask / QA** — retrieval + LLM generation. Variants: *Generative search* `(weaviate)`, *Ask* `(lighton)`, *Question Answering* `(mixedbread)`, *File Search tool* `(openai)`, *Document QnA* `(mistral)`, *File Search* `(google)`.
- **Reranking / Relevance scoring** — second-stage reordering of retrieved results. Variants: *Rerank* `(weaviate)`, *Reranker* `(mistral)`, *relevance_scoring* `(lighton)`, *rerank* `(mixedbread)`. Not present in `(openai, google)`.
- **Pipeline / Workflow / Operator chain** — orchestration of processing steps. Variants: *Pipeline* `(datalab, mistral, docetl)`, *Workflow (Temporal)* `(datalab)`, *operator chain* / *Frame* `(docetl)`, *run_pipeline* `(knowledgegraph)`, *ingestion pipeline* `(mixedbread, lighton)`.
- **Checkpoint** — a stored intermediate parse state reusable by downstream steps. `(datalab)` unique.
- **Facet / Content-type / Tag** — classification metadata. Variants: *Facet* / *content-type* `(lighton)`, *tag* `(lighton)`, *attribute* `(openai, lighton, weaviate)`, *custom_metadata* `(google, mixedbread)`.
- **Tenancy / Isolation** — multi-tenant data separation. Variants: *Workspace* `(lighton)`, *Tenant* `(weaviate)`, *Organization* `(mixedbread)`, *team context* `(datalab)`.
- **Quantization / Compression** — reducing vector storage. `(weaviate)` unique (PQ/BQ/SQ/RQ).

### 1.4 Input Formats (Union)

The union of accepted input formats across all systems:

- **Documents:** PDF, DOCX, DOC, PPTX, PPT, XLSX, XLS, ODT, ODP, ODS, ODF, RTF, CSV, TSV, HTML, HTM, Markdown, TXT, AsciiDoc, LaTeX, Box Notes, XML (JATS, USPTO, XBRL), Email (EML, MSG), NUMBERS, HWP/HWPX.
- **Images:** PNG, JPG/JPEG, WebP, AVIF, TIFF.
- **Audio:** MP3, WAV, M4A, AAC, OGG, FLAC, WebM Audio.
- **Video:** MP4, AVI, MOV, QuickTime, WebM, OGG Video.
- **Code:** Python, Java, Go, Rust, Swift, Kotlin, Scala, TypeScript, JavaScript, PHP, C, C++, C#, Ruby, Shell, PowerShell, CSS, diff, R Markdown, Graphviz, YAML, JSON.
- **Structured:** JSON, JSONL, Parquet, CSV, MXJSON/MXJSONL (pre-chunked format `(mixedbread)`), DocLang / `.dclx` `(docling)`, Docling JSON `(docling)`.
- **Other:** ZIP archives, WebVTT (timed text).

> **Note on format coverage:** No single system supports all of these. The super-complete API would accept all and route to format-specific backends.

---

## 2. The Unified Processing Pipeline (Overview)

The exhaustive pipeline below is ordered from raw input to final answer and beyond. Every stage is **optional** and stages can be combined, reordered, or bypassed depending on the system and use case.

```
[1] File Upload & Ingestion
        │
        ▼
[2] Document Parsing / OCR / Layout Analysis
        │
        ▼
[3] Document Segmentation / Boundary Detection   (optional; multi-doc PDFs)
        │
        ▼
[4] Chunking / Splitting
        │
        ▼
[5] Chunk Enrichment & Contextualization          (summaries, context windows, metadata)
        │
        ▼
[6] Data Extraction (fields, tables, KVPs, annotations)
        │                                              │
        ▼                                              ▼
[7] Classification / Categorization         [8] Entity Detection / Resolution / Linking
        │                                              │
        ▼                                              ▼
[9] Clustering                                    [18] Knowledge Graph Building
        │                                              │
        ▼                                              ▼
[10] Embedding / Vectorization  ─────────────────────►
        │
        ▼
[11] Indexing & Storage  (vector store, inverted index, graph store)
        │
        ▼
   ┌──── Query Time ────────────────────────────────────┐
   │ [12] Search (lexical / vector / hybrid / agentic)  │
   │ [13] Filtering, Facets & Metadata Scoping           │
   │ [14] Reranking                                       │
   │ [15] Query Rewriting & Preprocessing                 │
   │ [16] Caching                                         │
   │ [17] Answer Generation / RAG / QA                    │
   └──────────────────────────────────────────────────────┘
        │
        ▼
[19] Aggregations, Grouped Search & Analytics
        │
        ▼
[20] Document Transformation & Round-trip (DOCX generation, form filling, track changes)
        │
        ▼
[21] Orchestration / Pipelines / Workflows  (wraps the whole pipeline)
[22] Evaluation / QA / Optimization          (wraps the whole pipeline)
[23] Provenance / Citations                  (cross-cutting)
[24] Visualization                           (cross-cutting)
[25] Multi-tenancy / Security / Administration (cross-cutting)
```

---

## 3. Detailed Pipeline Stages

### 3.1 File Upload & Ingestion

**Purpose:** Get raw documents into the system.

**Naming variants for the container:** Workspace `(lighton)`, Store / search index `(mixedbread)`, Vector Store `(openai)`, FileSearchStore `(google)`, Collection `(weaviate)`, Project `(ibm)`, Dataset / Frame `(docetl)`, file storage `(datalab, mistral)`.

**Alternative ingestion approaches (union):**

1. **Direct multipart upload** — binary file in the request body. `(datalab, docling, ibm, lighton, mistral, mixedbread, openai)`
2. **URL-based ingestion** — pass a public/internal URL; the platform fetches it. `(datalab, docling, google, lighton, mistral, mixedbread)`
3. **Base64 inline** — file bytes encoded inline in JSON body. `(docling, google, mistral)`
4. **Presigned-URL upload** — platform issues a presigned PUT URL; client uploads to object storage. `(datalab, mixedbread)`
5. **Resumable / multipart upload** — two-step initiate-then-upload for large files. `(google, mixedbread)` — supports up to ~1 TB `(mixedbread)`.
6. **Cloud-storage loaders** — S3, Azure Blob, GCS, Google Drive, SharePoint sync. `(mistral, lighton)`
7. **Local filesystem / directory batch** — recursive directory reads. `(docling, docetl, mistral)`
8. **In-memory / stream** — `DocumentStream`, `from_list`, `convert_string`. `(docling, docetl)`
9. **Pre-chunked ingestion (MXJSON/MXJSONL)** — bypass parsing/chunking; provide chunks directly. `(mixedbread)`
10. **Docling JSON round-trip** — re-ingest a prior Docling JSON to re-export without re-parsing. `(docling)`
11. **`datalab://` file references** — stable URI for previously uploaded files. `(datalab)`
12. **Object insertion (JSON)** — insert pre-parsed JSON objects directly into a vector DB. `(weaviate)`

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

**Async processing pattern:** Upload returns a job/file ID + status; client polls or receives webhook until `completed`/`embedded`/`ACTIVE`. Status lifecycles:
- `pending → in_progress → completed | failed | cancelled` `(mixedbread, openai)`
- `pending → pending_conversion → converting → parsing → embedding → embedded` `(lighton)`
- `PROCESSING → ACTIVE | FAILED` `(google)`
- `pending → started → success | failure` `(docling)`
- `In Progress → Completed | Failed` `(ibm)`
- `pending → dispatched → running → completed → failed → skipped` `(datalab pipelines)`

**Webhooks:** `(datalab, ibm, docling-via-WS)` — alternative to polling.

**Batch ingestion:** Up to 500 files per batch `(openai)`; bulk operations `(mixedbread)`; `convert_all` `(docling)`; batch runs over collections `(datalab)`; concurrent `asyncio.gather` `(mistral)`.

---

### 3.2 Document Parsing, OCR & Layout Analysis

**Purpose:** Convert raw bytes into structured text + layout + tables + images + metadata.

**Naming variants:** *Document conversion* `(datalab, docling)`, *OCR Processor* `(mistral)`, *Document Understanding* `(google)`, *parsing* `(lighton, mixedbread, openai)`, *backend* `(docling)`.

**Alternative parsing approaches (union):**

1. **Multi-model pipeline (layout + OCR + table structure)** — separate models for layout (DocLayNet), OCR (Tesseract/Surya/RapidOCR), table structure (TableFormer). `(docling, datalab)`
   - Modes: `fast` / `balanced` / `accurate` `(datalab)`; `table_mode: fast/accurate` `(docling)`.
2. **Single end-to-end VLM** — one vision-language model (GraniteDocling 258M, or remote VLM API) replaces the entire chain. `(docling)`
3. **Native multimodal vision** — the LLM "sees" PDF pages as images, preserving layout/charts/tables. `(google)` — `media_resolution: low/medium/high` (Gemini 3).
4. **Managed automatic parsing** — platform-internal, not exposed (OCR + layout + transcription). `(lighton, mixedbread, openai)`
5. **OCR API** — dedicated OCR endpoint returning markdown + images + tables + blocks. `(mistral)` — `mistral-ocr-latest` / `mistral-ocr-4-0`; 13 block types with bboxes in reading order; confidence scores at page/word granularity.
6. **Word-level OCR with font metadata** — per-word coordinates, confidence, bold/italic/font. `(ibm)`
7. **Format-specific backends** — subclassable per format (PDF, DOCX, HTML, image, audio). `(docling)`
8. **Audio/video transcription (ASR)** — Whisper, Voxtral; diarization + timestamps. `(docling, mistral)` — output as WebVTT or text.
9. **Legacy office conversion** — `.doc/.ppt/.hwp` → PDF via PyMuPDF Pro → OCR. `(mistral)`

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

### 3.3 Document Segmentation / Boundary Detection

**Purpose:** Split a multi-document PDF into logical sections (each segment = a separate document with page ranges).

**Naming variants:** *Segmentation* / *Document Segmentation* `(datalab)`; *Document Segmentation* (`DS`/`SHW`) `(ibm)`.

**Alternative approaches:**
1. **Schema-guided segmentation** — provide `segmentation_schema` with segment names + descriptions. `(datalab)`
2. **Automatic boundary detection** — `segmentation_strategy: document_boundary` for auto-detection. `(datalab)`
3. **Page-structure segmentation** — header-based page structure segmentation. `(ibm)`

**Output:** Segments with name, page ranges, confidence (`high`/`medium`/`low`). `(datalab)`

> **Note:** This is distinct from *chunking* (which splits for embedding). Segmentation splits *documents within a file*; chunking splits *content within a document*.

---

### 3.4 Chunking / Splitting

**Purpose:** Divide a parsed document into retrievable/embedding-ready segments.

**Naming variants:** *Chunking* `(openai, google, docling, mistral)`, *Split* operator `(docetl)`, *TextSplitter* `(mistral)`, *chunking_strategy* `(openai)`, *chunking_config* `(google)`, *DocumentChunker* `(knowledgegraph)`, *platform-managed chunking* `(lighton, mixedbread)`.

**Alternative chunking approaches (union):**

| Approach | Description | Systems |
|---|---|---|
| **Static / token-count** | Fixed token window with overlap. `max_chunk_size_tokens` (100–4096, default 800), `chunk_overlap_tokens` (default 400). | openai, google (`white_space_config`), mistral (`TokenTextSplitter`), docetl (`token_count`) |
| **Character-count** | Fixed character window. `chunk_size` (default 1000). | mistral (`CharacterTextSplitter`) |
| **Separator / hierarchical** | Split on configurable separators (paragraph, sentence, etc.) with fallback. | mistral (`SeparatorTextSplitter`), docetl (`delimiter`) |
| **Markdown / header-aware** | Split by markdown headers (`#`, `##`...); preserve header context. | mistral (`MarkdownTextSplitter`), docling |
| **Hierarchical / structure-pure** | One chunk per document element; merges list items; attaches headers/captions. | docling (`HierarchicalChunker`) |
| **Hybrid (tokenization-aware)** | Splits oversized chunks, merges undersized peers; aligned to tokenizer; table-header repetition. | docling (`HybridChunker` — production default for RAG) |
| **Line-based** | Preserves line boundaries (tables, code, logs, lists); supports repeated prefixes. | docling (`LineBasedTokenChunker`) |
| **Word-count with overlap** | `chunk_size` words, `overlap` words. | knowledgegraph (AI-KG) |
| **Structure-preserving (Docling-based)** | Token-bounded; tables/lists/section hierarchy intact; sentence→word→char fallback. | knowledgegraph (Docling-Graph) |
| **Automatic / managed** | Platform-managed; not configurable. | lighton, mixedbread, google (default), openai (default) |
| **Pre-chunked (bypass)** | Provide chunks directly via MXJSON/MXJSONL. | mixedbread |
| **Gather (context enrichment)** | Adds context from surrounding chunks (previous/next, head/middle/tail) + header hierarchy. | docetl (`Gather`) |

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

### 3.5 Chunk Enrichment & Contextualization

**Purpose:** Add metadata, summaries, or surrounding context to chunks during or after ingestion.

**Alternative approaches:**
1. **Summary enrichment** — generate a document/chunk summary and optionally prepend it to every chunk (`propagate_summary_to_chunks`). `(mistral)`
2. **Contextualization at index time** — `with_metadata` and `with_file_context` enrich chunk context. `(mixedbread)`
3. **Gather (context windows)** — add peripheral chunks (previous/next) + reconstructed headers to each chunk. `(docetl)`
4. **Generated metadata** — auto-extract typed metadata per chunk (language, word_count, page info, code line ranges, media durations, BPM, frame counts, temporal boundaries). `(mixedbread)`
5. **Custom enrichers** — pluggable `ChunkEnricher` interface for arbitrary metadata injection. `(mistral)`

---

### 3.6 Data Extraction (Fields, Tables, Key-Value Pairs, Annotations)

**Purpose:** Pull structured data (typed fields, tables, KV pairs) from documents using schemas or models.

**Naming variants:** *Structured extraction* / *Extract* `(datalab, lighton, mistral, docetl)`, *Annotations* (BBox / Document) `(mistral)`, *KVP extraction* `(ibm)`, *structured output* `(google, openai)`, *Map operator* `(docetl)`.

**Alternative extraction approaches (union):**

1. **JSON-schema-driven LLM extraction** — provide a JSON Schema / Pydantic / Zod model; LLM fills it. `(datalab, google, lighton, mistral, mixedbread, docetl)` — with citations `(datalab)`, verification `(datalab)`, confidence scoring `(datalab)`.
   - Extraction modes: `turbo` (image-only, no citations), `fast` (citations + scores), `balanced` (verification + reasoning + citations). `(datalab)`
   - Page-aware: one result object per page, `null` for absent fields. `(lighton)`
2. **BBox annotation** — per-image classification/description via schema. `(mistral)`
3. **Document annotation** — document-level structured extraction via schema. `(mistral)`
4. **KVP extraction with ontology** — key + value with coordinates, confidence, KeyClass tagging, validators. `(ibm)` — three output tiers: basic (best KVP per key class), detailed (all candidates ranked), verbose (full OCR + tables + classification).
5. **Table / line-item extraction** — recursive `ComplexKVPStructure` with nested rows/cells. `(ibm)` — supports nested tables.
6. **Semantic normalization** — cleans/standardizes values (names, addresses) with `OriginalValue` preservation. `(ibm)`
7. **Verbatim text extraction** — pull source text without synthesis; line_number or regex strategy; lower token cost, no hallucination. `(docetl — `Extract` operator)`
8. **Form filling** — fill PDF/image forms with field data; AcroForm + visual + image field detection; confidence threshold. `(datalab)`
9. **Schema auto-generation** — generate candidate extraction schemas (simple/moderate/complex) from a checkpoint. `(datalab)`
10. **Pydantic-template extraction** — schema-first Pydantic models define both extraction schema AND graph structure; one-to-one or many-to-one strategies. `(knowledgegraph)`
11. **Dense extraction** — two-phase skeleton-then-flesh extraction contract for large documents. `(knowledgegraph)`

**Key parameters (union):**
- `page_schema` / `schema` / `response_format` / `document_annotation_format` / `output.schema` — the schema.
- `schema_id` / `schema_version` — saved schema reference. `(datalab)`
- `extraction_mode` — `turbo`/`fast`/`balanced`. `(datalab)`
- `checkpoint_id` — reuse prior parse. `(datalab)`
- `confidence_threshold` — form filling. `(datalab)`
- `field_data` — form filling input. `(datalab)`
- `docClass` — skip classification for known document type. `(ibm)`
- `jsonOptions` — capability toggles (`KVP`, `SN`, `MT`, `CHAR`). `(ibm)`

**Output quality signals (union):**
- Per-field citations (block IDs traceable to source). `(datalab)`
- Per-field verification status (`PASS`/`FAIL_UNRESOLVABLE`/`FAIL_FIX`/`FAIL_CITATIONS`/`ITEMS_MISSING`). `(datalab)`
- Per-field confidence score (1–5) + reasoning. `(datalab)`
- Per-field `KeyConfidence`/`ValueConfidence`/`KeyClassConfidence`. `(ibm)`
- Extraction score average. `(datalab)`
- Parse quality score (0–5). `(datalab)`

---

### 3.7 Classification & Categorization (Taxonomies, Facets, Tags)

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

### 3.8 Entity Detection, Resolution, Deduplication & Linking

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

### 3.9 Clustering

**Purpose:** Group items by semantic similarity.

**Alternative approaches (union):**

1. **Hierarchical agglomerative clustering** — binary tree of embeddings; cluster path annotation (most-specific → root); LLM-generated cluster summaries. `(docetl — `Cluster` operator)`
2. **KMeans on embeddings** — discover hidden groupings. `(openai)`
3. **Louvain community detection** — color-coded clusters in visualization. `(knowledgegraph)`
4. **Value sampling cluster** — k-means for representative subset selection in reduce. `(docetl)`
5. **t-SNE visualization** — 2D cluster diagnostics. `(openai)`

---

### 3.10 Embedding / Vectorization

**Purpose:** Convert text/images/audio/video into numeric vectors for semantic search.

**Naming variants:** *Vectorization* / *Vectorizer module* `(weaviate)`, *Embedder* `(mistral)`, *embedding* `(google, openai, mixedbread, lighton, docetl)`.

**Alternative approaches (union):**

1. **Auto-vectorization on insert** — vectorizer module generates vectors from object properties automatically. `(weaviate)` — providers: OpenAI, Cohere, Google, Hugging Face, Ollama, Jina, NVIDIA, Mistral, AWS, Voyage AI, Transformers (self-hosted).
2. **Managed automatic embedding** — platform embeds during ingestion; not exposed. `(google, lighton, mixedbread, openai-vector-stores)`
3. **Standalone embedding API** — generate vectors for arbitrary text; return to client for own vector DB. `(openai — `embeddings.create`)` — models: `text-embedding-3-small` (1536d), `text-embedding-3-large` (3072d), `ada-002` (1536d); `dimensions` param (MRL shortening); `encoding_format` (float/base64).
4. **Configurable embedder** — pluggable `Embedder` ABC. `(mistral)` — models: 1024-dim, 256-dim, 128-dim.
5. **Named vectors (multi-model)** — multiple vector spaces per collection, each with independent vectorizer + index + compression; multi-model search on same data. `(weaviate)`
6. **Multi-vector embeddings** — ColBERT/ColPali-style multi-vector. `(weaviate)` (v1.30+)
7. **Multimodal embeddings** — text + images in unified vector space; text query retrieves images and vice versa. `(google — `gemini-embedding-2`)`
8. **Vision (VLM) embeddings** — VLM embeddings over page images for searching visual documents. `(lighton)` — parallel `status_vision` pipeline.
9. **Whole-document embeddings** — `mixedbread-ai/mxbai-wholembed-v3`; required for audio/video. `(mixedbread)`
10. **Self-provided vectors** — user supplies vectors; no vectorizer. `(weaviate)`
11. **Embeddings as ML features** — for classification, clustering, regression, recommendations. `(openai)`

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

### 3.11 Indexing & Storage

**Purpose:** Persist documents, vectors, inverted indexes, and graph structures for fast retrieval.

**Alternative index/storage approaches (union):**

1. **Managed vector store (auto-indexed)** — platform manages embeddings, indexing, sharding; zero infrastructure. `(google, lighton, mixedbread, openai)` — free storage in some `(google)`.
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

**Idempotent ingestion:** Deterministic UUID5 / `external_id` — re-ingesting overwrites, not duplicates. `(weaviate, mistral, lighton, mixedbread)`

---

### 3.12 Search (Lexical, Vector, Hybrid, Agentic)

**Purpose:** Find relevant chunks/documents given a query.

**Naming variants:** *Search* `(lighton, mixedbread, openai)`, *near_\** methods` `(weaviate)`, *Retrieval* `(openai, mistral)`, *File Search* `(google, openai)`, *Query* `(weaviate)`.

#### 3.12.1 Lexical / Keyword Search

- **BM25 / BM25F** — token frequency + IDF; BM25F extends to weighted multi-field. `(weaviate)` — per-property tokenization (WORD, LOWERCASE, WHITESPACE, FIELD, TRIGRAM for fuzzy/typo tolerance); property boosting (`^weight`); `and`/`or` operators with `minimum_match`; accent folding; stopword presets.
- **Grep (regex)** — RE2 regex pattern matching against literal chunk text; `content_groups` (text/generated); case sensitivity. `(mixedbread)`

#### 3.12.2 Vector / Semantic Search

- `near_text`, `near_vector`, `near_object`, `near_image` `(weaviate)` — four input modalities.
- `VectorRetriever` `(mistral)` — embedding-based semantic search.
- Semantic search via embeddings `(google, lighton, mixedbread, openai)`.
- **Move parameters** (curriculum learning) — `move_to` / `move_away` with force + concepts to steer results. `(weaviate)`
- **MMR (Maximal Marginal Relevance)** — diversity selection balancing relevance and diversity. `(weaviate)` (v1.37 preview)
- **Autocut / auto_limit** — detect natural breaks in distance/score distribution. `(weaviate)`

#### 3.12.3 Hybrid Search

- **Vector + keyword fusion** — run both in parallel, fuse weighted by `alpha` (0=pure keyword, 1=pure vector, 0.5=balanced). `(weaviate)` — two fusion algorithms: `RELATIVE_SCORE` (default) / `RANKED` (`1/(RANK+60)`).
- **Reciprocal Rank Fusion (RRF)** — blend semantic + keyword with tunable `embedding_weight`/`text_weight`. `(openai)` — at least one must be > 0.
- **RRF for multi-retriever fusion** — `RRFRanker` with `rrf_k` smoothing. `(mistral)`
- **Hybrid vector + keyword + vision** — vector similarity + keyword/text matching + optional cross-encoder reranking + vision mode. `(lighton)` — score breakdown per component (text/vision/keyword/multivector/relevance).
- **Hybrid web + internal** — virtual web store in `store_identifiers`; web results always reranked, merged with internal. `(mixedbread)`

#### 3.12.4 Agentic Search

- **Multi-round, agent-driven retrieval** — decompose complex questions into sub-queries; run multiple rounds; evaluate candidates; iterate; merge/rerank. `(mixedbread — `agentic`)` — `max_rounds`, `queries_per_round`, `instructions` (natural-language steering), `strict_top_k`, `score_threshold`.
- **Query Agent** — LLM translates natural language into database operations (searches, aggregations, filters, sorts across collections). `(weaviate)` — modes: Ask (answer + sources), Search (raw objects), Suggest (query discovery); streaming with progress; `additional_filters`; `view_properties`; pagination reusing searches.
- **Model-autonomous File Search** — model decides when to search, generates queries, synthesizes answers. `(openai)`

#### 3.12.5 Other Search Modes

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

### 3.13 Filtering, Facets & Metadata Scoping

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

### 3.14 Reranking

**Purpose:** Second-stage reordering of retrieved results with a more expensive model for sharper relevance.

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

### 3.15 Query Rewriting & Preprocessing

**Purpose:** Improve user queries before retrieval.

**Alternative approaches:**
1. **Query rewriting** — auto-rewrite into search-friendly form (strips conversational filler, converts to noun-phrase). `(openai — `rewrite_query`)`, `(mixedbread — `rewrite_query`)` — rewritten query observable in `search_query`.
2. **LLM query rewriter** — reformulate informal queries. `(mistral — `LLMQueryRewriter`)`
3. **LLM query extension** — generate multiple sub-queries for broader retrieval. `(mistral — `LLMQueryExtension`)`
4. **Model-generated queries** — File Search tool generates its own queries (visible in `queries`). `(openai)`

---

### 3.16 Caching

**Purpose:** Skip embedding/retrieval/reranking on cache hits.

**Alternative approaches:**
1. **Semantic cache** — match queries by meaning via cosine similarity threshold (0.99 strict / 0.95 balanced / 0.90 permissive); skip retrieval on hit; eviction policies (LRU/LFU/FIFO); TTL; metrics tracking. `(mistral — `CachedQueryEngine`, `SemanticCache`)`
2. **Result caching** — `skip_cache` override. `(datalab)`
3. **LLM call caching** — cache in `~/.cache/docetl/llm`. `(docetl)`
4. **Memoized terminal actions** — repeated calls with unchanged config reuse results. `(docetl)`
5. **Persistent media IDs** — stable blob IDs for image chunks enable caching. `(google)`
6. **HNSW snapshots** — point-in-time HNSW state for fast startup. `(weaviate)`

---

### 3.17 Answer Generation / RAG / QA

**Purpose:** Generate grounded answers from retrieved chunks with citations.

**Naming variants:** *Generative search* / *RAG* `(weaviate)`, *Ask* `(lighton)`, *Question Answering* `(mixedbread)`, *File Search tool* `(openai)`, *Document QnA* `(mistral)`, *File Search* `(google)`.

**Alternative RAG approaches (union):**

1. **Hosted RAG (model-autonomous)** — model decides when to search, retrieves, generates with inline citations. `(openai — File Search tool)` — `file_citation` annotations with character offsets.
2. **Managed RAG (one-call)** — retrieve-then-generate in one API call; citations via `<cite i="n"/>` markers mapping to `sources[n]`. `(mixedbread — Question Answering)` — multimodal, streaming, instructions for answer style.
3. **Two-stage retrieve + generate** — Search then Ask; SSE streaming (OpenAI-compatible); source attribution. `(lighton — Ask)`
4. **Generative search integrated into search calls** — Single Prompt (per-object) + Grouped Task (per-group); `{property_name}` interpolation; multimodal (images in prompts); query-time model override. `(weaviate)`
5. **Retrieval API + manual synthesis** — direct search returns chunks; developer feeds to `chat.completions.create` with `<sources>` XML pattern. `(openai)`
6. **Document QnA** — Chat Completions with `document_url` content block; multi-document queries. `(mistral)`
7. **RAG via retriever + map/reduce** — attach retriever to map/filter/reduce/extract; inject `{{ retrieval_context }}`. `(docetl)`
8. **GraphRAG** — vector search + graph traversal for multi-hop reasoning; patterns: vector+graph, agentic retrieval, entity-centric RAG, ontology-driven RAG. `(knowledgegraph — Neo4j)`
9. **Query Agent Ask mode** — natural language → answer + sources across collections. `(weaviate)`
10. **Structured grounded output** — JSON Schema + file search for machine-readable grounded responses. `(google)` (Gemini 3+)

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

### 3.18 Knowledge Graph Building

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

### 3.19 Aggregations, Grouped Search & Analytics

**Purpose:** Compute metrics over result sets; group results.

**Alternative approaches:**
1. **Aggregate queries** — counts, statistics (sum, max, min, mean, median, mode), frequency distributions (top_occurrences), boolean percentages, reference counts; GroupByAggregate for per-group metrics. `(weaviate)`
2. **Grouped search (GroupBy)** — organize results into groups by property or cross-reference; `objects_per_group`, `number_of_groups`. `(weaviate)`
3. **Reduce operator** — group by `reduce_key`, produce one output per group; incremental folding; scratchpad; value sampling. `(docetl)`
4. **Facets** — aggregate chunk counts by metadata values. `(mixedbread)`
5. **Rank operator** — full sorting by latent attribute (not top-k retrieval); "picky window" refinement; O(n) scaling. `(docetl)`

---

### 3.20 Document Transformation & Round-trip (Generation, Form Filling, Track Changes)

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

### 3.21 Orchestration, Pipelines & Workflows

**Purpose:** Chain multiple processing steps into reusable, versioned executions.

**Alternative orchestration approaches (union):**

1. **Declarative pipelines (YAML/Python)** — ordered operators with draft→saved→published lifecycle; immutable versioned snapshots; per-step intermediate results; eval integration; per-step billing. `(datalab)`
   - Step types: `convert`, `segment`, `extract`, `custom`, `fill`.
   - Checkpoint passing between steps.
2. **Temporal workflows** — Temporal-engine workflow definitions; long-running, fault-tolerant. `(datalab)`
3. **Declarative map-reduce framework** — lazy, immutable Frame; chained operations; terminal actions trigger execution; Python API ↔ YAML convertible. `(docetl)`
   - Operators: map, filter, reduce, resolve, equijoin, rank, extract, cluster, split, gather, unnest, sample, topk, parallel_map, link_resolve, code_map, code_reduce, code_filter.
   - Tool-equipped agents on map/filter/reduce.
4. **Pipeline + RoutedPipeline** — ingestion orchestration with checkpointing + progress callbacks; multi-format routing by extension/MIME. `(mistral)`
5. **QueryEngine + CachedQueryEngine** — retrieval orchestration with optional caching. `(mistral)`
6. **run_pipeline(config)** — template loading → extraction → Docling export → graph conversion → export → statistics. `(knowledgegraph)`
7. **Pipeline class** — ingestion orchestrator: load → extract → chunk → enrich → embed → index. `(mistral)`
8. **MCP server** — expose operations as tools for AI agents. `(docling, mixedbread, weaviate, knowledgegraph-Neo4j, lighton)`
9. **Automatic managed pipeline** — no manual orchestration needed (chunk → embed → index automatic). `(google, lighton, mixedbread, openai)`

**Orchestration features:**
- **Checkpointing** — skip already-processed documents on restart. `(mistral, datalab)`
- **Progress callbacks** — tqdm progress. `(mistral)`
- **Parallelization** — `max_threads`, `parallel_workers`, `concurrent_thread_count`. `(docetl, knowledgegraph)`
- **Caching** — LLM calls + optimized plans cached. `(docetl, datalab)`
- **Retries / timeouts** — `max_retries_per_timeout`, `skip_on_error`. `(docetl)`
- **Cost & token tracking** — `frame.total_cost`, `frame.token_usage`. `(docetl)`

---

### 3.22 Evaluation, Quality Assurance & Optimization

**Purpose:** Measure and improve pipeline quality.

**Alternative approaches (union):**

1. **Parse quality score** — 0–5 self-assessment of conversion quality. `(datalab)`
2. **Eval rubrics** — block/page/document rules scoring 0–5; `eval_rubric_id` on convert/extract/pipeline steps; generation from user feedback. `(datalab)`
3. **Forge Evals** — configuration comparison (max 10 docs, 5 configs, 3 iterations); visual diffs; multi-model comparison. `(datalab)`
4. **Custom processor eval definitions** — `eval_definition` per processor; `run_eval` on execution. `(datalab)`
5. **Per-field verification** — balanced-mode per-field independent validation (PASS/FAIL) against source. `(datalab)`
6. **Per-field confidence scoring** — 1–5 score with reasoning. `(datalab)`
7. **KVP validation** — per-KeyClass validators defined in ontology; `POST /validator` applies them. `(ibm)` — `ValidatorResult` ("Pass"/"Fail") + `ValidatorFailures`.
8. **Gleaning** — LLM-based iterative validation/refinement of operator output. `(docetl)`
9. **Validate** — Python-expression-based output validation with retries. `(docetl)`
10. **Calibration** — reference anchors for consistent classification/scoring. `(docetl)`
11. **Plan rewrites** — automatic equivalence-preserving pipeline reordering (selection_pushdown, limit_pushdown). `(docetl)`
12. **BARGAIN model cascades** — statistical guarantees (accuracy/precision/recall) on binary operators with probability `1 - delta`. `(docetl)`
13. **MOAR (offline MCTS optimization)** — multi-objective agentic rewrites; Pareto-optimal cost-accuracy frontier. `(docetl)`
14. **Operation-level `optimize` flag** — inline optimization. `(docetl)`
15. **Semantic cache metrics** — hit_rate, avg_hit_similarity, retrieval time. `(mistral)`

---

### 3.23 Provenance, Citations & Source Tracking

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

### 3.24 Visualization

**Purpose:** Visually explore graphs, clusters, and document structure.

**Alternative approaches:**
1. **Interactive HTML (Cytoscape)** — zoom/pan, node inspection, search, image export; extraction report + graph statistics. `(knowledgegraph — Docling-Graph)`
2. **PyVis/Vis.js HTML** — Louvain community detection; color-coded clusters; centrality-sized nodes; dashed edges for inferred relationships; light/dark, physics toggle. `(knowledgegraph — AI-Knowledge-Graph)`
3. **t-SNE 2D visualization** — cluster diagnostics on embeddings. `(openai)`
4. **Web UI playground** — interactive conversion testing. `(docling)`
5. **DocWrangler IDE** — spreadsheet interface with automatic visualizations, in-situ feedback, prompt refinement with diffs, version control. `(docetl)`

---

### 3.25 Multi-tenancy, Security, Residency & Administration

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

## 4. Naming Variant Cross-Reference Table

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

---

## 5. Alternative-Approaches Cross-Reference Table

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
| **Search** | BM25/BM25F keyword · Grep regex · near_text/near_vector/near_object/near_image · Semantic via embeddings · Hybrid (alpha-weighted) · RRF · Hybrid vector+keyword+vision · Hybrid web+internal · Agentic search · Query Agent · Model-autonomous File Search · List chunks · Multi-store/federated · Cross-collection Explore · Cross-document reasoning |
| **Filtering** | Comparison filters · Compound filters · AIP-160 expressions · Weaviate filter operators · Cross-reference filtering · Nested object filtering · Content-type/facet filters · Tag filters · File ID scoping |
| **Reranking** | Cross-encoder (pointwise) · Listwise (instruction-steerable) · LLM reranking · RRF fusion · Relevance scoring modes · Sequential chaining · Boost (soft ranking) |
| **Query rewriting** | Query rewriting (observable) · LLM query rewriter · LLM query extension (sub-queries) · Model-generated queries |
| **Caching** | Semantic cache · Result caching · LLM call caching · Memoized terminal actions · Persistent media IDs · HNSW snapshots |
| **RAG / QA** | Hosted RAG (model-autonomous) · Managed RAG (one-call) · Two-stage retrieve+generate · Generative search in search calls · Retrieval API + manual synthesis · Document QnA · RAG via retriever+map/reduce · GraphRAG · Query Agent Ask · Structured grounded output |
| **Knowledge graph** | Schema-validated Pydantic pipeline · Schema-free LLM triple extraction · Native graph DB storage · Link resolve · Cross-references |
| **Aggregation** | Aggregate queries · Grouped search (GroupBy) · Reduce operator · Facets · Rank operator |
| **Document transformation** | DOCX generation · Form filling · Track changes extraction · Document round-trip · Thumbnails · Synthetic data generation |
| **Orchestration** | Declarative pipelines (YAML/Python) · Temporal workflows · Declarative map-reduce framework · Pipeline + RoutedPipeline · QueryEngine + CachedQueryEngine · run_pipeline · MCP server · Automatic managed pipeline |
| **Evaluation** | Parse quality score · Eval rubrics · Forge Evals · Custom processor evals · Per-field verification · Per-field confidence · KVP validation · Gleaning · Validate · Calibration · Plan rewrites · BARGAIN cascades · MOAR MCTS optimization · Operation-level optimize · Semantic cache metrics |

---

## 6. Unified API Specification

This section specifies a **super-complete API** that encompasses all features of the eleven surveyed systems. It is written for the end user — an API consumer who wants to build document-intelligence applications.

> **Conventions:** All endpoints are RESTful under a base URL `https://api.unified-docintel.example/v1`. Parameters shown in `monospace` are JSON body fields or query params. Async operations follow submit→poll/webhook→retrieve. The spec uses a unified naming; the naming-variant table above maps to individual systems.

### 6.1 Authentication & Configuration

```
POST   /v1/api-keys                  — Create scoped API key (roles: owner/editor/viewer; workspace-scoped)
GET    /v1/api-keys                  — List API keys
DELETE /v1/api-keys/{id}             — Revoke API key
GET    /v1/health                    — Health check (no auth)
GET    /v1/version                   — Version info
```

**Auth:** Bearer token (`Authorization: Bearer <key>`). Model provider keys passed via headers (`X-OPENAI-API-KEY`, `X-COHERE-API-KEY`, etc.) for self-hosted vectorizer/generator modules.

**Budget:**
```
GET    /v1/budgets                   — Get org budget
PATCH  /v1/budgets                   — Update monthly cap + alert thresholds
```

### 6.2 Workspaces (Containers)

A **Workspace** is the top-level container for documents, indexes, and tenants.

```
POST   /v1/workspaces                — Create workspace
         { name, type: "shared"|"personal", processing_location: "us"|"eu",
           embedding_model, chunking_config, expires_after: {anchor, days} }
GET    /v1/workspaces                — List workspaces (pagination: limit, before/after)
GET    /v1/workspaces/{id}           — Get workspace
PUT    /v1/workspaces/{id}           — Update workspace
DELETE /v1/workspaces/{id}           — Delete workspace (cascade with force: true)
```

**Workspace config:**
- `embedding_model` — text-only or multimodal (immutable after creation for some systems).
- `chunking_config` — `{ type: "static"|"hierarchical"|"hybrid"|"line_based"|"markdown"|"separator", max_chunk_size_tokens, chunk_overlap_tokens, tokenizer, merge_peers, repeat_table_header, ... }`.
- `contextualization` — `{ with_metadata: bool, with_file_context: bool }`.
- `expires_after` — `{ anchor: "last_active_at", days: N }`.
- `access_mode` — `private` | `public`.

### 6.3 Files (Upload & Ingestion)

```
POST   /v1/files                     — Upload file (multipart) or by URL or base64
         { file | file_url | base64_string, filename, title, workspace_id,
           metadata, external_id, tags[], parser, parsing_strategy,
           chunking_strategy, processing_location, webhook_url,
           auto_chunk: true, save_checkpoint: true }
GET    /v1/files                     — List files (filter by workspace_id, status, tags, metadata)
GET    /v1/files/{id}                — Get file metadata + status
GET    /v1/files/{id}/download       — Download original or rendered_pdf (purpose param)
PATCH  /v1/files/{id}                — Update file metadata/attributes
DELETE /v1/files/{id}                — Delete file
POST   /v1/files/bulk-delete         — Bulk delete
```

**Presigned / multipart upload:**
```
POST   /v1/files/uploads             — Create multipart upload session
         { filename, file_size, mime_type, part_count }
GET    /v1/files/uploads/{id}        — Get upload status + fresh presigned URLs
POST   /v1/files/uploads/{id}/abort  — Abort upload
POST   /v1/files/uploads/{id}/complete — Complete upload (parts + ETags)
```

**File status lifecycle:** `pending → converting → parsing → embedding → embedded | failed`

**Webhooks:** `webhook_url` per request or default in account settings. Webhook fires on status change.

**Idempotent uploads:** `external_id` with slash support for path-like identifiers; re-upload returns existing doc (200) instead of duplicating.

### 6.4 Document Parsing & Conversion

```
POST   /v1/convert                   — Parse document to structured output
         { file | file_url | checkpoint_id, output_format: ["md"|"html"|"json"|"chunks"|"doctags"|"doclang"|"docx"|"pdf"|"png"],
           mode: "fast"|"balanced"|"accurate",
           page_range, max_pages, paginate, add_block_ids, word_bboxes,
           table_cell_bboxes, list_item_bboxes, include_markdown_in_chunks,
           disable_image_extraction, disable_image_captions,
           do_ocr, force_ocr, ocr_lang, ocr_preset, table_mode, table_format,
           include_image_base64, include_blocks, confidence_scores_granularity,
           bbox_annotation_format, document_annotation_format,
           extract_header, extract_footer, extract_links,
           save_checkpoint, skip_cache, processing_location,
           extras: ["track_changes","chart_understanding","infographic","new_block_types"],
           enrichments: ["code","formula","picture_classification","picture_description"],
           media_resolution: "low"|"medium"|"high",
           additional_config: { keep_pageheader_in_output, keep_pagefooter_in_output, keep_spreadsheet_formatting },
           webhook_url }
GET    /v1/convert/{request_id}      — Poll conversion result
```

**Output:** markdown, html, json (blocks with bboxes/types), chunks, images (base64), metadata, page_count, parse_quality_score (0–5), cost_breakdown, checkpoint_id, confidence_scores.

**Checkpoint reuse:** `checkpoint_id` from a prior `save_checkpoint=true` conversion can be passed to `/convert`, `/extract`, `/segment`, `/gen-schemas` to skip re-parsing.

### 6.5 Document Segmentation

```
POST   /v1/segment                   — Segment multi-document PDF
         { file | file_url | checkpoint_id,
           segmentation_schema: { segments: [{name, description}], segmentation_strategy: "custom"|"document_boundary" },
           mode, page_range, save_checkpoint, webhook_url }
GET    /v1/segment/{request_id}      — Poll segmentation result
```

**Output:** `segments[]` with name, pages[], confidence (`high`/`medium`/`low`).

### 6.6 Chunking

Chunking is configured at the workspace level (`chunking_config`) or per-file (`chunking_strategy`). The API exposes chunk inspection:

```
GET    /v1/files/{id}/chunks         — List chunks (return_chunks: bool | indices[])
```

**Chunk types:** `text`, `image_url`, `audio_url`, `video_url`, `content`, `image_annotation`, `summary`.

**Pre-chunked ingestion (bypass parsing):**
```
POST   /v1/files/prechunked          — Upload MXJSON/MXJSONL
         { file (json with chunks[]), workspace_id, metadata, external_id }
```

### 6.7 Data Extraction

```
POST   /v1/extract                   — Structured field extraction
         { file | file_url | checkpoint_id,
           page_schema (JSON Schema) | schema_id,
           schema_version,
           extraction_mode: "turbo"|"fast"|"balanced",
           mode (parsing): "fast"|"balanced"|"accurate",
           output_format, page_range, save_checkpoint, skip_cache, webhook_url }
GET    /v1/extract/{request_id}      — Poll extraction result
```

**Output:** per-field values, `{field}_citations`, `{field}_meta` (extraction_status, reasoning, verification status, citations), `{field}_score` (1–5), `extraction_score_average`.

**Schema management:**
```
POST   /v1/schemas                   — Create extraction schema
GET    /v1/schemas                   — List schemas
GET    /v1/schemas/{id}              — Get schema
PUT    /v1/schemas/{id}              — Update schema (create_new_version)
DELETE /v1/schemas/{id}              — Delete schema (soft)
POST   /v1/gen-schemas               — Auto-generate candidate schemas from checkpoint
         { checkpoint_id }
GET    /v1/gen-schemas/{request_id}  — Poll (returns simple/moderate/complex schemas)
```

**Annotations (BBox / Document):**
```
POST   /v1/annotate                  — Per-image or document-level annotation
         { file | file_url, bbox_annotation_format | document_annotation_format (JSON Schema/Pydantic/Zod),
           include_image_base64, webhook_url }
```

### 6.8 Classification & Taxonomies (Facets)

```
GET    /v1/content-types             — List content-type tree (filters: query, path, depth, include_attributes)
POST   /v1/content-types             — Action-dispatched: adopt | define_content_type | undefine_content_type | define_attribute | undefine_attribute
GET    /v1/content-types/templates   — List seed templates (finance, healthcare, legal, manufacturing, tech)
POST   /v1/files/{id}/facets         — Action-dispatched: classify | unclassify | set_value | clear_value
GET    /v1/files/{id}/facets         — Get file classifications + attribute values
```

**Tags:**
```
GET    /v1/tags                      — List tags
POST   /v1/tags                      — Create tag
```

### 6.9 Entity Resolution, Clustering & Knowledge Graph

These are exposed as **pipeline operators** (see §6.12) and as **standalone endpoints**:

```
POST   /v1/resolve                   — Entity deduplication
         { data, comparison_prompt, resolution_prompt, blocking_keys, blocking_threshold,
           blocking_target_recall, blocking_conditions, embedding_model, cascade }
POST   /v1/equijoin                  — Fuzzy join
         { left, right, comparison_prompt, limits, blocking_keys, blocking_threshold, cascade }
POST   /v1/cluster                   — Hierarchical clustering
         { data, embedding_keys, summary_prompt, summary_schema, embedding_model }
POST   /v1/rank                      — Full sorting by latent attribute
         { data, prompt, input_keys, direction: "asc"|"desc", initial_ordering_method: "likert"|"embedding",
           call_budget, k }
```

**Knowledge graph building:**
```
POST   /v1/knowledge-graph/build     — Build KG from document(s)
         { source | file_id | checkpoint_id,
           template (Pydantic class or dotted path),
           processing_mode: "one-to-one"|"many-to-one",
           extraction_contract: "auto"|"direct"|"dense",
           dense_config: { skeleton_batch_tokens, fill_nodes_cap, fill_context, dedupe },
           backend: "llm"|"vlm", inference: "local"|"remote",
           use_chunking, chunk_max_tokens,
           provenance: "off"|"standard"|"detailed",
           gleaning_enabled, parallel_workers,
           export_format: "csv"|"cypher"|"json"|"html",
           export_docling, export_markdown, export_doclang }
GET    /v1/knowledge-graph/{id}      — Get graph (NetworkX JSON, stats, provenance)
GET    /v1/knowledge-graph/{id}/export — Export graph (csv/cypher/json/html)
```

### 6.10 Embedding (Standalone)

```
POST   /v1/embeddings                — Generate embeddings for arbitrary text
         { input (string | string[]), model, dimensions, encoding_format: "float"|"base64" }
```

**Output:** `data[]` with `embedding[]`, `index`; `usage` (prompt_tokens, total_tokens).

### 6.11 Index & Search

#### Index management
Handled by workspace creation (§6.2). Additional:

```
GET    /v1/workspaces/{id}/documents          — List indexed documents
GET    /v1/workspaces/{id}/documents/{doc_id} — Get indexed document
DELETE /v1/workspaces/{id}/documents/{doc_id} — Delete indexed document (cascade to embeddings)
```

#### Search

```
POST   /v1/search                    — Unified search
         { query, workspace_id[] | store_identifiers[], top_k, max_results,
           mode: "text"|"vision",
           search_type: "semantic"|"keyword"|"hybrid"|"agentic"|"grep"|"list",
           alpha (hybrid: 0=keyword, 1=vector),
           fusion_type: "relative_score"|"ranked",
           hybrid_search: { embedding_weight, text_weight },
           ranking_options: { ranker, score_threshold },
           rerank: bool | { model, top_k, with_metadata },
           rewrite_query: bool,
           agentic: bool | { max_rounds, queries_per_round, instructions, strict_top_k },
           relevance_scoring: "none"|"scoring_only"|"scoring_and_filtering",
           filters: { ... } | attribute_filter | metadata_filter (AIP-160),
           content_type[], attribute[], tag_id[], file_ids[],
           target_vector, distance, certainty, auto_limit, autocut,
           move_to: { concepts, force }, move_away: { concepts, force },
           selection: { type: "mmr", balance },
           boost: { ... },
           group_by: { prop, objects_per_group, number_of_groups },
           return_properties, return_references, return_metadata,
           include_image, include_bboxes, media_content: "auto"|"always"|"never",
           return_metadata: bool }
```

**Grep (regex):**
```
POST   /v1/grep                      — Regex pattern matching
         { store_identifiers[], pattern (RE2), top_k, content_groups: ["text"|"generated"],
           case_sensitive, file_ids[], filters, return_metadata }
```

**List chunks (metadata-only):**
```
POST   /v1/list-chunks               — Metadata-only retrieval
         { store_identifiers[], top_k, file_ids[], sort_by, filters, return_metadata }
```

**Facets:**
```
POST   /v1/metadata-facets           — Aggregate chunk counts by metadata
         { store_identifiers[], query, top_k, filters, facets[] }
```

#### Filtering language (unified)

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
metadata_filter: "author=\"Robert Graves\" AND year>=1934"
```

**Three metadata layers:** User file metadata (bare keys), generated chunk metadata (`generated_metadata.` prefix), system fields (`file_id`, `chunk_index`). Dot notation for nested fields.

### 6.12 Answer Generation / RAG / QA

```
POST   /v1/ask                       — RAG: retrieve + generate
         { query | messages[], workspace_id[] | store_identifiers[],
           model, max_results, stream: bool,
           instructions (answer-style steering),
           qa_options: { cite: bool, multimodal: bool, stream: bool },
           search_options: { rerank, rewrite_query, agentic, score_threshold, ... },
           filters, content_type[], attribute[], tag_id[], file_ids[] }
```

**Output (non-streaming):** `{ answer (with <cite i="n"/> markers), sources: [chunks] }`.

**Output (streaming):** SSE token events + final answer object (OpenAI-compatible).

**Generative search modes (weaviate-style):**
```
POST   /v1/generate                  — Generative search with single_prompt / grouped_task
         { query, workspace_id, search_type, top_k,
           single_prompt, grouped_task, grouped_properties,
           generative_provider (query-time model override) }
```

**Structured grounded output (google-style):**
```
POST   /v1/ask                       — with response_format
         { ..., response_format: { type: "text", mime_type: "application/json", schema: {...} } }
```

### 6.13 Aggregations & Analytics

```
POST   /v1/aggregate                 — Aggregate over collection or search results
         { workspace_id, query (optional, for search-time aggregate),
           total_count: bool, return_metrics: Metrics, group_by: GroupByAggregate,
           filters, distance, object_limit }
```

**Metrics:** `count`, `sum`, `max`, `min`, `mean`, `median`, `mode`, `top_occurrences`, `percentageTrue/False`, `reference_count`.

### 6.14 Document Transformation

```
POST   /v1/create-document           — Generate DOCX from markdown
         { markdown (with <ins>/<~~>/<comment> tags), output_format: "docx", webhook_url }
POST   /v1/fill                      — Fill PDF/image form
         { file | file_url, field_data: {key: {value, description}}, context,
           confidence_threshold, page_range, output_format: "pdf"|"png" }
POST   /v1/track-changes             — Extract redlines from DOCX
         { file | file_url, output_format: ["md"|"html"|"chunks"], page_range, webhook_url }
GET    /v1/thumbnails/{lookup_key}   — Page thumbnails
         { page_range, thumb_width, track_changes: bool }
```

### 6.15 Pipelines & Workflows (Orchestration)

```
POST   /v1/pipelines                 — Create pipeline (versioned)
         { name, steps: [ { type: "convert"|"segment"|"extract"|"custom"|"fill"|"map"|"filter"|"reduce"|"resolve"|"cluster"|"rank"|"split"|"gather"|"unnest"|"code_map"|"code_reduce"|"code_filter"|"parallel_map"|"equijoin"|"link_resolve"|"kg_build",
                            settings: {...}, custom_processor_id, eval_rubric_id } ] }
GET    /v1/pipelines                 — List pipelines
GET    /v1/pipelines/{id}            — Get pipeline (version)
POST   /v1/pipelines/{id}/run        — Execute pipeline
         { file | file_url, version (0=draft, omit=active), page_range, output_format,
           run_evals, skip_cache, checkpoint_id, webhook_url }
GET    /v1/pipelines/executions/{id} — Get execution status + per-step results
GET    /v1/pipelines/executions      — List executions
```

**Custom processors:**
```
POST   /v1/custom-processors         — Create AI-generated custom processor
GET    /v1/custom-processors         — List
GET    /v1/custom-processors/{id}    — Get (with versions)
POST   /v1/custom-processors/{id}/iterate — Iterate (new version)
POST   /v1/custom-processors/{id}/describe — Conversational builder
```

**Workflows (Temporal-style):**
```
POST   /v1/workflows                 — Create workflow definition
POST   /v1/workflows/{id}/execute    — Execute workflow
GET    /v1/workflows/{id}/execution  — Get execution status
GET    /v1/workflows/step-types      — List available step types
```

### 6.16 Evaluation & Optimization

```
POST   /v1/eval-rubrics              — Create eval rubric
POST   /v1/eval-rubrics/from-feedback — Generate rubric from user feedback
POST   /v1/forge-evals               — Configuration comparison (max 10 docs, 5 configs, 3 iterations)
POST   /v1/validate                  — Validate extracted KVPs against ontology
         { project_id, data (verbose JSON output) }
```

**Pipeline optimization (MOAR-style):**
```
POST   /v1/pipelines/{id}/optimize   — Offline MCTS optimization
         { eval_fn, metric_key, models, max_iterations, save_dir }
```

**Output:** Pareto-optimal cost-accuracy frontier; `.best()`, `.cheapest()`, `.frontier`.

### 6.17 Query Agent (Agentic NL → Operations)

```
POST   /v1/query-agent/ask           — Natural language → answer + sources
         { query | messages[], collections: [{ name, target_vector, view_properties, tenant, additional_filters }],
           result_evaluation: "none"|"llm", timeout }
POST   /v1/query-agent/search        — Natural language → raw objects
         { query, collections, limit, filtering: "recall"|"precision", diversity_weight }
POST   /v1/query-agent/ask-stream    — Streaming with progress
         { ..., include_progress, include_final_state }
```

### 6.18 Tenancy

```
POST   /v1/workspaces/{id}/tenants   — Create tenants
GET    /v1/workspaces/{id}/tenants   — List tenants
PUT    /v1/workspaces/{id}/tenants   — Update tenant activity_status (ACTIVE/INACTIVE/OFFLOADED)
```

All standard operations (insert, query, aggregate, generate) accept a `tenant` header/param for scoped access.

### 6.19 MCP Server

The platform exposes an MCP server for AI assistant integration (Claude Code, Cursor, Windsurf, etc.) with tools for:
- `search` — semantic/keyword/hybrid search
- `ask` — RAG question answering
- `convert` — document parsing
- `extract` — structured extraction
- `graph_query` — Cypher/graph queries (if graph store enabled)

### 6.20 Visualization

```
GET    /v1/knowledge-graph/{id}/visualize — Interactive HTML (Cytoscape/PyVis)
POST   /v1/visualize/embeddings      — t-SNE 2D visualization of embeddings
```

---

## 7. End-to-End Reference Flows

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
1. Create workspace with multi_tenancy enabled
2. POST /v1/workspaces/{id}/tenants { tenants: [{name: "customerA"}, {name: "customerB"}] }
3. Insert + search + ask with tenant header per customer
```

---

## 8. Coverage Matrix

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
| Search (lexical) | ❌ | ✅ | ❌ | ❌ | ❌ | ✅ | ✅ | ✅ | ✅ | ✅ | ✅ |
| Search (vector) | ❌ | ✅ | ❌ | ✅ | ❌ | ✅ | ✅ | ✅ | ✅ | ✅ | ✅ |
| Search (hybrid) | ❌ | ✅ | ❌ | ❌ | ❌ | ✅ | ✅ | ✅ | ✅ | ✅ | ✅ |
| Search (agentic) | ❌ | ❌ | ❌ | ❌ | ❌ | ❌ | ❌ | ❌ | ✅ | ✅ | ✅ |
| Filtering | ❌ | ✅ | ❌ | ✅ | ❌ | ✅ | ✅ | ✅ | ✅ | ✅ | ✅ |
| Reranking | ❌ | ✅ | ❌ | ❌ | ❌ | ❌ | ✅ | ✅ | ✅ | ❌ | ✅ |
| Query rewriting | ❌ | ✅ | ❌ | ❌ | ❌ | ❌ | ❌ | ✅ | ✅ | ✅ | ❌ |
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

> **End of specification.** This document represents the union of all capabilities described across the eleven surveyed systems. Individual systems implement subsets; the unified API specification in §6 encompasses the full feature surface. Refer to individual system files for implementation-specific details, exact endpoint signatures, and system-specific constraints.