# Docling API Analysis — Document Intelligence Platform

> **Product:** [Docling for IBM watsonx](https://www.ibm.com/products/docling) (managed SaaS) · **Open-source core:** [docling-project/docling](https://github.com/docling-project/docling) (MIT, LF AI & Data Foundation)
> **API:** [docling-serve](https://github.com/docling-project/docling-serve) REST API (stable `v1`; OpenAPI/Swagger at `/docs`)
> **Auth:** API key header `X-Api-Key: <YOUR_KEY>` (managed service) or bearer token (OpenShift deployments)
> **Description:** Convert complex, unstructured documents (PDFs, images, slide decks, spreadsheets, audio, email, XML schemas…) into structured, AI-ready data (Markdown, JSON, HTML, DocTags, DocLang, chunks) preserving layout, hierarchy, tables, reading order, and formulas. Designed as a managed service exposing the same REST API as the self-hosted server.

---

## Table of Contents

1. [Platform Overview & Core Concepts](#1-platform-overview--core-concepts)
2. [Authentication & Access Models](#2-authentication--access-models)
3. [Document Conversion — REST API](#3-document-conversion--rest-api)
4. [Conversion Options](#4-conversion-options)
5. [Asynchronous Conversion & Task Management](#5-asynchronous-conversion--task-management)
6. [DoclingDocument — Unified Representation](#6-doclingdocument--unified-representation)
7. [Export & Serialization](#7-export--serialization)
8. [Chunking](#8-chunking)
9. [Enrichments — Code, Formula, Picture Understanding](#9-enrichments--code-formula-picture-understanding)
10. [OCR](#10-ocr)
11. [Vision-Language Model (VLM) Pipelines](#11-vision-language-model-vlm-pipelines)
12. [Audio & Video (ASR)](#12-audio--video-asr)
13. [Supported Formats](#13-supported-formats)
14. [Python Client & SDK](#14-python-client--sdk)
15. [CLI](#15-cli)
16. [Integrations & Agentic Ecosystem](#16-integrations--agentic-ecosystem)
17. [Deployment & Managed Service](#17-deployment--managed-service)
18. [Pricing & Resource Units](#18-pricing--resource-units)
19. [Capability Summary & Cross-Reference](#19-capability-summary--cross-reference)

---

## 1. Platform Overview & Core Concepts

### Main Concepts

Docling is a **document intelligence / conversion toolkit** that turns complex documents into structured, AI-ready data. The managed offering ("Docling for IBM watsonx") wraps the open-source library in a hosted service exposing the same REST API as the self-hosted `docling-serve`.

Core abstractions:

- **Document Converter** — The central entry point. Knows, per input format, which **backend** (parser) and which **pipeline** (orchestrator) to use, plus any **options**. Configurable per format.
- **Backend** — Format-specific parser that turns a raw file (PDF, DOCX, HTML, image, audio…) into an intermediate parsed representation. Subclassable.
- **Pipeline** — Orchestrates the chain of AI models (layout analysis, table structure, OCR, enrichments…) applied after parsing. Two built-in pipelines: `StandardPdfPipeline` (multi-threaded, model-based) and `SimplePipeline` (lightweight, for non-PDF formats). VLM pipelines also supported.
- **DoclingDocument** — The unified, expressive document representation produced by every conversion. A Pydantic datatype defined in `docling_core.types.doc`. The single source of truth from which all exports and chunks are derived.
- **ConversionResult** — Wraps a `DoclingDocument` plus metadata (status, timings, errors, pages, confidence).
- **Enrichment** — Optional post-parse model step that adds understanding to specific components (code language, formula LaTeX, picture classification/description).
- **Chunker** — Splits a `DoclingDocument` into contiguous, embedding-ready text chunks for RAG.
- **Serializer** — Renders a `DoclingDocument` (or chunks) into a target output format.

### End-to-End Flow

1. Submit a document (URL, file upload, or base64) to a convert endpoint.
2. The converter selects the backend + pipeline for the detected input format.
3. The backend parses the raw file; the pipeline runs layout analysis, OCR (if needed), table structure recognition, and any enabled enrichments.
4. The result is assembled into a `DoclingDocument`.
5. The document is serialized to the requested output format(s) (Markdown, JSON, HTML, DocTags, Text, WebVTT, DocLang, chunks/JSONL) and returned.

### Architecture (from official docs)

```
input document
   │
   ▼
 Document Converter ──► Backend (parser) ──► Pipeline (models: layout, table, OCR, enrichments…)
   │                                              │
   │                                              ▼
   └─────────────────────────────────── ConversionResult
                                              │
                                         DoclingDocument
                                              │
                          ┌───────────────────┼───────────────────┐
                          ▼                   ▼                   ▼
                     Export methods      Serializer           Chunker
                  (md, json, html,…)   (custom formats)    (RAG chunks)
```

### Key Differentiators

- **Structure-preserving** — Retains layout, hierarchy, reading order, tables, formulas, code, captions, bounding boxes.
- **Unified representation** — One `DoclingDocument` model across all input formats → consistent downstream processing.
- **Local or managed** — Runs locally (air-gapped/sensitive data) or as a managed SaaS with identical API.
- **AI-model powered** — Specialized models for layout (DocLayNet), table structure (TableFormer), code/formula (CodeFormula), picture classification/description, and end-to-end VLM (GraniteDocling).
- **RAG-first** — Native chunkers and plug-and-play integrations with LangChain, LlamaIndex, Haystack, CrewAI, etc.

---

## 2. Authentication & Access Models

### Main Concepts

- **API key (managed service)** — Set `DOCLING_SERVE_API_KEY` server-side; clients send `-H "X-Api-Key: <YOUR_KEY>"` on every request.
- **Bearer token (OpenShift/Kubernetes)** — `Authorization: Bearer <OCP_AUTH_TOKEN>` when deployed behind an OAuth proxy.
- **No auth (local dev)** — If no key is configured, endpoints are open.

### Analysis

A single flat key model with no documented workspace/role scoping. For the managed IBM watsonx service, the key is provisioned via the watsonx platform; clients just swap the base URL and add the key header. The self-hosted server can run with or without auth, making it suitable for both internal trusted networks and exposed deployments.

---

## 3. Document Conversion — REST API

### Main Concepts

The REST API exposes the Docling pipeline as an HTTP service with synchronous and asynchronous conversion endpoints. Documents can be submitted as URLs, base64-encoded strings, or multipart file uploads.

### Endpoints

| Endpoint | Method | Purpose |
|---|---|---|
| `/v1/convert/source` | `POST` | Convert from URLs / base64 sources (synchronous) |
| `/v1/convert/file` | `POST` | Convert uploaded files, `multipart/form-data` (synchronous) |
| `/v1/convert/source/async` | `POST` | Submit a URL/base64 conversion asynchronously → returns task handle |
| `/v1/convert/file/async` | `POST` | Submit a file upload conversion asynchronously → returns task handle |
| `/v1/status/poll/{task_id}` | `GET` | Poll async task status |
| `/v1/status/ws/{task_id}` | `WebSocket` | Subscribe to live task status updates (push) |
| `/v1/result/{task_id}` | `GET` | Fetch a finished async result |
| `/health` | `GET` | Liveness check (always 200) |
| `/ready` | `GET` | Readiness check (503 until models loaded) |
| `/version` | `GET` | Returns docling, docling-serve, docling-core versions |
| `/docs` | `GET` | Interactive OpenAPI / Swagger UI |

### Request Bodies

**`/v1/convert/source` (JSON):**

```json
{
  "http_sources": [{ "url": "https://arxiv.org/pdf/2501.17887" }],
  "options": {
    "to_formats": ["md"],
    "do_ocr": true,
    "table_mode": "fast"
  }
}
```

- `http_sources` — list of `{ url, headers? }` objects to fetch and convert.
- `file_sources` — alternative: `{ base64_string, filename }` objects for inline base64 uploads.
- `options` — conversion options (see §4).

**`/v1/convert/file` (multipart/form-data):**

```
-F "files=@document.pdf;type=application/pdf"
-F "from_formats=pdf"
-F "to_formats=md"
-F "do_ocr=true"
-F "image_export_mode=embedded"
-F "table_mode=fast"
```

- `files` — one or more file parts.
- All conversion options are passed as additional form fields.

### Response Format (single file, JSON)

```json
{
  "document": {
    "md_content": "",
    "json_content": {},
    "html_content": "",
    "text_content": "",
    "doctags_content": ""
  },
  "status": "success",
  "processing_time": 0.0,
  "timings": {},
  "errors": []
}
```

- Only the `*_content` fields corresponding to requested `to_formats` are populated.
- `status` ∈ `success` | `partial_success` | `skipped` | `failure`.
- `processing_time` in seconds; `timings` carries per-component detail when enabled.
- If a zip `target` is requested or the job produces multiple files, the response is a **zip archive** (`application/zip`) instead of JSON.

### Analysis

The dual source model (URL/base64 via `/source`, binary via `/file`) covers all common ingestion patterns. Base64 is the escape hatch when you have in-memory bytes but no HTTP-reachable URL; the docs recommend writing large base64 bodies to a temp file and using `-d @file` to avoid shell argument-length limits. The sync endpoints are simple but subject to a server-side max wait (`DOCLING_SERVE_MAX_SYNC_WAIT`, default configurable — increase for large documents). The async endpoints + poll/WebSocket/result trio is the production pattern for heavy or batch workloads. The interactive `/docs` Swagger UI on every server is a strong ergonomics feature — the authoritative schema is always live.

---

## 4. Conversion Options

### Main Concepts

Conversion options are passed in the JSON body under `options` (for `/source` endpoints) or as form fields (for `/file` endpoints). This is the common subset; the full, authoritative schema is the live OpenAPI docs at `/docs`.

### Option Reference

| Option | Type | Description |
|---|---|---|
| `from_formats` / `to_formats` | list[string] | Input / output formats (see §13). Restricts accepted inputs and produced outputs. |
| `image_export_mode` | string | How images are emitted: `placeholder` / `embedded` (base64) / `referenced` (external files). |
| `do_ocr` / `force_ocr` | boolean | Enable / force OCR on pages. `force_ocr` runs OCR even on digital (text-bearing) pages. |
| `ocr_preset` / `ocr_lang` | string / list | OCR preset and languages. `ocr_engine` is deprecated — prefer `ocr_preset`. |
| `table_mode` | string | Table-structure mode: `fast` or `accurate`. |
| `pdf_backend` | string | PDF parsing backend selection. |
| `pipeline` | string | Processing pipeline (e.g. `standard` vs VLM). |
| `do_code_enrichment` | boolean | Enable code-block understanding (sets `code_language`). |
| `do_formula_enrichment` | boolean | Enable formula → LaTeX extraction. |
| `do_picture_classification` | boolean | Classify pictures (chart types, diagrams, logos, signatures…). |
| `do_picture_description` | boolean | Annotate pictures with a vision-model caption. |
| `picture_description_preset` | string | Named preset for the picture-description model. |
| `picture_description_custom_config` | object | Full control over the picture-description model (local or remote). |
| `generate_picture_images` | boolean | Extract picture image data into the document. |
| `images_scale` | number | Scale factor for generated picture images. |
| `enable_remote_services` | boolean | Allow connections to remote model APIs (required for remote picture description). |

### Analysis

The options bag is the control plane for the quality/latency/cost tradeoff. The key axes are: OCR (on/off/forced, language), table fidelity (`fast` vs `accurate`), enrichment toggles (code, formula, picture classification/description — all off by default because they add model invocations), image handling (placeholder keeps payloads small; embedded is self-contained; referenced needs the zip bundle), and pipeline selection (standard model pipeline vs end-to-end VLM). The deprecation of `ocr_engine` in favor of `ocr_preset` signals a move toward curated, opinionated OCR configurations over raw engine selection. The `picture_description_preset` / `picture_description_custom_config` split (replacing the older `picture_description_local` / `picture_description_api`) unifies local and remote VLM configuration under one mechanism.

---

## 5. Asynchronous Conversion & Task Management

### Main Concepts

For large documents or batch jobs, the `/async` variants avoid sync timeouts. Submitting returns a task descriptor; clients poll, subscribe via WebSocket, or fetch the result when done.

### Task Descriptor (submit response)

```json
{
  "task_id": "<task_id>",
  "task_status": "pending",
  "task_position": 1,
  "task_meta": null
}
```

### Task Status Values

`pending` → `started` → `success` | `failure`

### Status Endpoints

| Endpoint | Method | Purpose |
|---|---|---|
| `/v1/status/poll/{task_id}` | `GET` | Returns current task object (poll repeatedly until terminal) |
| `/v1/status/ws/{task_id}` | `WebSocket` | Push updates: JSON messages with `message` = `connection` \| `update` \| `error` + task object |
| `/v1/result/{task_id}` | `GET` | Fetch the finished conversion result (same shape as sync response) |

### Typical Polling Pattern

```python
import time, httpx
base_url = "http://localhost:5001/v1"
task = response.json()  # from /async submission
while task["task_status"] not in ("success", "failure"):
    task = httpx.get(f"{base_url}/status/poll/{task['task_id']}").json()
    time.sleep(5)
result = httpx.get(f"{base_url}/result/{task['task_id']}").json()
```

### Analysis

The async design is conventional and robust. The `task_position` field gives queue visibility. The WebSocket option is the low-latency path for UIs or orchestration systems that want push updates rather than blind polling. The separation of status (`/status/*`) from result (`/result`) means you can cheaply check progress without re-fetching potentially large result payloads. The recommendation to increase `DOCLING_SERVE_MAX_SYNC_WAIT` (e.g. to 600s) for large docs is the pragmatic fallback when callers prefer sync semantics.

---

## 6. DoclingDocument — Unified Representation

### Main Concepts

`DoclingDocument` is Docling's fundamental document representation — a Pydantic datatype defined in `docling_core.types.doc`. It is the single intermediate format produced by every conversion and the source of all exports and chunks.

### Top-Level Structure

**Content items** (lists of `DocItem` subclasses, referencing parents/children via JSON pointers):

| Field | Type | Description |
|---|---|---|
| `texts` | `list[TextItem]` | All text-bearing items: paragraphs, section headings, equations, list items, code, captions… |
| `tables` | `list[TableItem]` | Tables, carrying structure annotations (rows, columns, multi-level headers). |
| `pictures` | `list[PictureItem]` | Pictures, carrying structure annotations and optional classifications/descriptions. |
| `key_value_items` | `list` | Key-value items. |

**Content structure** (tree of `NodeItem` instances):

| Field | Type | Description |
|---|---|---|
| `body` | tree root | Root node of the tree-structure for the main document body. Reading order is encoded by the `body` tree and the order of children in each node. |
| `furniture` | tree root | Root for items not in the main body (headers, footers). |
| `groups` | `list` | Container items that group other content (e.g. a list, a chapter) without being content themselves. |

### Key Properties

- **Hierarchy** — Sections and groups form a tree; children are nested under their parent heading.
- **Reading order** — Encoded by the `body` tree and child ordering (not a separate sequence).
- **Furniture disambiguation** — Headers/footers separated from body; optionally excluded from exports.
- **Layout / bounding boxes** — Available for all items when the source provides layout info; multiple boxes per component allowed (can span pages).
- **Provenance** — Origin information tracked per item.
- **Construction APIs** — `DoclingDocument` can be built from scratch programmatically.

### Analysis

The `DoclingDocument` design is the platform's core innovation: by normalizing every input format into one rich, hierarchical, layout-aware model, downstream consumers (exporters, chunkers, serializers, RAG integrations) only need to understand one representation. The split between content items (flat lists) and content structure (tree) is pragmatic — items are queryable by type, while the tree captures relationships and reading order. JSON pointers for parent/child references keep the model graph-like without redundancy. The "furniture" concept (separating headers/footers) is a practical detail that directly improves RAG quality by excluding repetitive boilerplate from chunks.

---

## 7. Export & Serialization

### Main Concepts

A `DoclingDocument` can be exported directly via its export methods, or rendered by a **serializer** for custom output. The REST API populates the requested `*_content` fields based on `to_formats`.

### Supported Output Formats

| Format | REST field | Description |
|---|---|---|
| Markdown | `md_content` | Markdown with structure markers; image embedding or referencing supported. |
| JSON | `json_content` | Lossless serialization of the full `DoclingDocument`. |
| HTML | `html_content` | HTML with image embedding/referencing; formulas rendered via MathML. |
| Text | `text_content` | Plain text without Markdown markers. |
| DocTags | `doctags_content` | Compact markup representing full content + layout characteristics (see [Doctags paper](https://arxiv.org/pdf/2503.11576)). |
| WebVTT | — | Web Video Text Tracks (timed text). |
| DocLang XML | — | XML following the [DocLang](https://doclang.ai) schema. |
| DocLang archive (`.dclx`) | — | Zipped DocLang bundle including page images. |
| Chunks (JSONL) | — | Chunked output for RAG pipelines; configurable via `--chunks-type`, `--chunks-max-tokens`, `--chunks-tokenizer` (CLI). |
| Zip bundle | `application/zip` | When multiple outputs are requested or the job yields multiple files. |

### Python Export Methods (on `DoclingDocument`)

- `export_to_markdown()`
- `export_to_text()`
- `export_to_html()`
- `export_to_document_tokens()` (DocTags)
- `export_to_dict()` (JSON)

### Analysis

The breadth of output formats is a deliberate design: Markdown and JSON are the workhorses for RAG/LLM ingestion; DocTags is the compact, layout-faithful format optimized for LLM token efficiency; DocLang XML is the structured, schema-validated interchange format; HTML preserves visual fidelity (with MathML for formulas); chunks/JSONL is the RAG-ready product. The zip-bundle path is important for referenced-image mode (where images are external files) and for multi-file batch results. The lossless JSON round-trip (Docling JSON is also an *input* format) enables re-processing and transformation pipelines without re-parsing originals.

---

## 8. Chunking

### Main Concepts

Chunking partitions a `DoclingDocument` into contiguous text chunks ready for embedding/ingestion by AI systems. Docling offers native chunkers that operate directly on the `DoclingDocument` (preferred) — alternatively, you can export to Markdown and apply user-defined chunking.

### Chunker Class Hierarchy

| Chunker | Import | Approach |
|---|---|---|
| `BaseChunker` | `docling.chunking` (base) | Defines the API: `chunk(dl_doc) -> Iterator[BaseChunk]` and `contextualize(chunk) -> str`. |
| `HierarchicalChunker` | `docling.chunking` | One chunk per detected document element; merges list items by default (`merge_list_items`). Attaches headers/captions metadata. |
| `HybridChunker` | `docling.chunking` / `docling_core.transforms.chunker.hybrid_chunker` | Tokenization-aware refinements on top of hierarchical chunking: split oversized chunks, merge undersized peers (`merge_peers=True` default). Aligned to a user-provided tokenizer (HuggingFace or OpenAI/tiktoken). |
| `LineBasedTokenChunker` | `docling.chunking` / `docling_core.transforms.chunker.line_chunker` | Preserves line boundaries (tables, code, logs, lists); splits a line only if it exceeds the max token limit alone. Supports repeated prefixes (e.g. table headers). |

### HybridChunker Table Options

| Option | Default | Description |
|---|---|---|
| `repeat_table_header` | `True` | Repeat table headers at the start of each chunk when a table spans multiple chunks. |
| `omit_header_on_overflow` | `False` | When `True` + `repeat_table_header=True`, omit the header for rows that fit without it but would overflow with it — maximizes token efficiency for wide tables. |

### Tokenizer Extras

- `pip install 'docling-core[chunking]'` — HuggingFace tokenizers.
- `pip install 'docling-core[chunking-openai]'` — OpenAI/tiktoken tokenizers.

### Chunk Output (CLI / JSONL)

Configurable via `--chunks-type`, `--chunks-max-tokens`, `--chunks-tokenizer`.

### Analysis

The chunker hierarchy reflects a quality-vs-simplicity ladder: `HierarchicalChunker` is the structure-pure baseline (one chunk per element, ideal when you want fine-grained, metadata-rich chunks); `HybridChunker` is the production default for RAG (tokenization-aware splitting/merging that respects document structure and headings/captions); `LineBasedTokenChunker` is the specialized tool for tabular/code/log content where line integrity matters. The table-header repetition logic is a thoughtful RAG-specific optimization — repeated headers preserve context for each table chunk, improving retrieval of tabular facts. The tokenizer-pluggable design ensures chunks are sized correctly for the *target embedding model*, not a generic heuristic. Docling's integrations with LangChain/LlamaIndex/Haystack are all built on the `BaseChunker` interface, so any chunker (built-in or custom) plugs in uniformly.

---

## 9. Enrichments — Code, Formula, Picture Understanding

### Main Concepts

Enrichments are optional pipeline steps that run additional models on specific document components. Most are **disabled by default** because they add model-execution latency.

### Enrichment Overview

| Feature | Parameter | Processed Item | Description |
|---|---|---|---|
| Code understanding | `do_code_enrichment` | `CodeItem` | Advanced parsing of code blocks; sets `code_language`. Model: [`CodeFormula`](https://huggingface.co/ds4sd/CodeFormula). |
| Formula understanding | `do_formula_enrichment` | `TextItem` (label `FORMULA`) | Extracts LaTeX representation of equations; HTML export renders via MathML. Model: [`CodeFormula`](https://huggingface.co/ds4sd/CodeFormula). |
| Picture classification | `do_picture_classification` | `PictureItem` | Classifies pictures (chart types, flow diagrams, logos, signatures…). Model: [`DocumentFigureClassifier-v2.5`](https://huggingface.co/docling-project/DocumentFigureClassifier-v2.5). Requires `generate_picture_images=True`. |
| Picture description | `do_picture_description` | `PictureItem` | Captioning via a vision model (local VLM or remote OpenAI-compatible API). |

### Picture Description Configuration

Local VLM presets:

| Preset | Model |
|---|---|
| `granite_picture_description` | [`ibm-granite/granite-vision-3.1-2b-preview`](https://huggingface.co/ibm-granite/granite-vision-3.1-2b-preview) |
| `smolvlm_picture_description` | [`HuggingFaceTB/SmolVLM-256M-Instruct`](https://huggingface.co/HuggingFaceTB/SmolVLM-256M-Instruct) |

Custom local VLM:

```python
from docling.datamodel.pipeline_options import PictureDescriptionVlmOptions
pipeline_options.picture_description_options = PictureDescriptionVlmOptions(
    repo_id="<hf-repo-id>",
    prompt="Describe the image in three sentences. Be concise and accurate.",
)
```

Remote vision model (OpenAI-compatible API — VLLM, Ollama, watsonx.ai, Azure OpenAI…):

```python
from docling.datamodel.pipeline_options import PictureDescriptionApiOptions
pipeline_options.enable_remote_services = True  # required
pipeline_options.picture_description_options = PictureDescriptionApiOptions(
    url="http://localhost:8000/v1/chat/completions",
    params=dict(model="MODEL NAME", seed=42, max_completion_tokens=200),
    prompt="Describe the image in three sentences. Be concise and accurate.",
    timeout=90,
    headers={"Authorization": "Bearer ..."},  # optional
    usage_response_key="usage",  # capture API usage metadata
)
```

- `usage_response_key` — JSON key/dotted path where the provider returns usage data; stored on picture metadata as `docling__usage`.

### REST API (picture description)

In `docling-serve` v1.21.0+, use `picture_description_preset` (named preset) or `picture_description_custom_config` (full control). The older `picture_description_local` / `picture_description_api` parameters are deprecated. Remote model calls require the server to be launched with `DOCLING_SERVE_ENABLE_REMOTE_SERVICES=true`.

### Analysis

Enrichments turn Docling from a layout extractor into a document *understanding* system. The code/formula enrichments add semantic typing (programming language, LaTeX) that makes technical documents directly usable by LLMs. Picture classification adds searchable structure to figures (chart vs. diagram vs. signature), and picture description generates captions that make images retrievable by text. The unified local/remote model configuration is notable: the same pipeline can run a small local VLM (SmolVLM-256M) for cost-sensitive jobs or call a remote frontier VLM via an OpenAI-compatible endpoint — with `enable_remote_services` as an explicit, security-conscious opt-in. The API-usage-capture feature (`usage_response_key`) is a production detail for cost tracking across providers.

---

## 10. OCR

### Main Concepts

Docling provides extensive OCR support for scanned PDFs and images. OCR is enabled by default (`do_ocr=true`) and applied only to pages that lack a digital text layer unless `force_ocr` is set.

### Configuration Options

| Option | Description |
|---|---|
| `do_ocr` | Enable OCR (applied to pages without text). |
| `force_ocr` | Force OCR on every page, even digital ones. |
| `ocr_preset` | Named OCR configuration preset (preferred over deprecated `ocr_engine`). |
| `ocr_lang` | OCR languages, e.g. `["eng", "deu"]` (note: language codes vary by engine — tesseract uses ISO 639-3 like `eng`/`deu`, not `en`/`de`). |

### Supported OCR Engines

- **Tesseract** (CLI and Python bindings) — `TesseractCliOcrOptions` / `TesseractOcrOptions`; supports automatic language detection.
- **RapidOCR** — with custom OCR models (`RapidOcrOptions`).
- **SuryaOCR** — with custom OCR models.
- **OCRmacOS** — `OcrMacOptions` (macOS-native).

### Analysis

OCR is the foundational capability for scanned/legacy documents. The multi-engine support (Tesseract, RapidOCR, Surya, macOS native) lets users trade off accuracy, language coverage, and runtime environment. The deprecation of `ocr_engine` in favor of `ocr_preset` indicates curated, tested configurations per use case rather than raw engine selection. The language-code caveat (engine-specific formats) is a documented gotcha. The `force_ocr` flag is useful when the digital text layer is unreliable (e.g. bad PDF text encoding). Full-page OCR mode bypasses layout-based text extraction entirely, routing all page content through OCR.

---

## 11. Vision-Language Model (VLM) Pipelines

### Main Concepts

Beyond the standard model pipeline (layout + table + OCR), Docling supports end-to-end VLM pipelines that integrate vision and language in a single compact model, replacing traditional OCR + layout chains.

### GraniteDocling

- **Model:** [`ibm-granite/granite-docling-258M`](https://huggingface.co/ibm-granite/granite-docling-258M) (~258M parameters).
- **Approach:** Parses PDFs, slides, and scanned pages directly into structured, machine-readable formats using **DocTags** — a purpose-built markup that separates content from layout while preserving tables, code blocks, inline/block math, and hierarchy.
- **Strengths:** Improved fidelity, minimized reading-order/structure errors, well-suited for RAG and fine-tuning.
- **Language support:** Optimized for Latin-script; early support for Japanese, Chinese, Arabic.

### Pipeline Selection

- `pipeline` option selects `standard` vs VLM.
- VLM pipelines can use local models (GraniteDocling) or remote API models (OpenAI-compatible).

### Usage (REST)

Set `pipeline` to the VLM pipeline in conversion options. For remote VLM models, `enable_remote_services=true` is required.

### Analysis

The VLM pipeline is the modernized, single-model alternative to Docling's traditional multi-model chain. For many documents, a single compact VLM (258M params — small enough for commodity hardware) can replace OCR + layout analysis + table structure + post-processing, reducing pipeline complexity and error compounding. DocTags as the native output format is optimized for LLM consumption (token-efficient, layout-faithful). The bilingual strategy (Latin-script first, CJK/Arabic early) reflects enterprise demand. The ability to use remote VLM APIs (e.g. watsonx.ai) means the managed service can leverage frontier vision models without shipping them in the container.

---

## 12. Audio & Video (ASR)

### Main Concepts

Docling supports audio and video files via Automatic Speech Recognition (ASR) models. For video, the audio track is extracted (requires `ffmpeg`) and transcribed.

### Supported Formats

| Category | Formats | Requirement |
|---|---|---|
| Audio | WAV, MP3, M4A, AAC, OGG, FLAC | `asr` extra |
| Video | MP4, AVI, MOV | `asr` extra + `ffmpeg` |

### ASR Models

- **Whisper** (OpenAI) — used in the `minimal_asr_pipeline` example.

### Output

Audio/video content is transcribed into text and represented in the `DoclingDocument`; can be exported to Markdown, JSON, text, or **WebVTT** (timed text).

### Analysis

Audio/video support extends Docling from document-centric to media-centric ingestion, important for meeting recordings, lectures, and multimedia knowledge bases. The WebVTT output format (both input and output) is the bridge to timed-text applications (subtitles, searchable transcripts with timestamps). The `ffmpeg` dependency for video is standard and well-documented. ASR is currently Whisper-based, with the architecture open to other models via the pipeline extensibility.

---

## 13. Supported Formats

### Input Formats

| Category | Formats |
|---|---|
| Documents | PDF, DOCX, XLSX, PPTX, ODT, ODS, ODP, EPUB, HTML, XHTML, CSV, Markdown, AsciiDoc, LaTeX, plain text (`.txt`, `.text`, `.qmd`, `.Rmd`), Box Notes |
| Images | PNG, JPEG, TIFF, BMP, WEBP |
| Audio | WAV, MP3, M4A, AAC, OGG, FLAC |
| Video | MP4, AVI, MOV |
| Timed text | WebVTT |
| Email | EML, MSG |
| XML schemas | DocLang XML (`.dclg`, `.dclg.xml`), DocLang archive (`.dclx`), USPTO patents, JATS articles, XBRL financial reports |
| Round-trip | Docling JSON (serialized `DoclingDocument`) |

### Output Formats

| Format | Description |
|---|---|
| Markdown | Structure-preserving; image embed/reference modes. |
| JSON | Lossless `DoclingDocument` serialization. |
| HTML | With MathML formula rendering. |
| Text | Plain text (no Markdown markers). |
| DocTags | Compact layout-faithful markup. |
| DocLang XML | Schema-validated XML. |
| DocLang archive (`.dclx`) | Zipped bundle with page images. |
| WebVTT | Timed text. |
| Chunks (JSONL) | RAG-ready chunked output. |

### `InputFormat` Enum (Python API)

`ASCIIDOC`, `AUDIO`, `BOXNOTE`, `CSV`, `DCLX`, `DOCX`, `EMAIL`, `EPUB`, `HTML`, `IMAGE`, `JSON_DOCLING`, `LATEX`, `MD`, `METS_GBS`, `ODP`, `ODS`, `ODT`, `PDF`, `PPTX`, `VTT`, `XLSX`, `XML_DOCLANG`, `XML_JATS`, `XML_USPTO`, `XML_XBRL`

### Analysis

The format coverage is exceptionally broad — from office documents and images to audio/video, email, and domain-specific XML schemas (patents, scientific articles, financial reports). The schema-specific support (USPTO, JATS, XBRL) is a strong enterprise differentiator: these are high-value, structured document classes that generic converters handle poorly. The Docling JSON round-trip (input and output) enables transformation and re-export pipelines. The OpenDocument Format (ODF) support and email (EML/MSG) parsing address common enterprise long-tail formats. The breadth reduces "tool sprawl" — one API for all document types.

---

## 14. Python Client & SDK

### Main Concepts

Docling ships a Python client for the API server (`DoclingServiceClient`) that returns the same `ConversionResult` as the local `DocumentConverter`, so client code is portable between local and remote execution.

### DoclingServiceClient

```python
from docling.service_client import DoclingServiceClient
from docling.datamodel.service.options import ConvertDocumentsOptions

with DoclingServiceClient(url="http://localhost:5001") as client:
    result = client.convert(
        source="https://arxiv.org/pdf/2501.17887",
        options=ConvertDocumentsOptions(to_formats=["md"]),
    )
print(result.document.export_to_markdown())
```

- `source` — HTTP/HTTPS URL string, local `pathlib.Path`, or `DocumentStream`.
- `convert_all(source=[...])` — stream multiple conversion results.
- `options` — same conversion options as the REST API.

### Local Python API — `DocumentConverter`

```python
from docling.document_converter import DocumentConverter

converter = DocumentConverter()
result = converter.convert("path/to/document.pdf")
print(result.document.export_to_markdown())
```

#### `DocumentConverter` Constructor

| Parameter | Type | Default | Description |
|---|---|---|---|
| `allowed_formats` | `Optional[list[InputFormat]]` | `None` (all) | Restrict accepted input formats. |
| `format_options` | `Optional[dict[InputFormat, FormatOption]]` | `None` | Per-format backend/pipeline/options configuration. |

#### `DocumentConverter` Methods

| Method | Description | Key Parameters |
|---|---|---|
| `convert(source, ...)` | Convert one document (path, URL, `DocumentStream`, `HttpSource`). | `headers`, `raises_on_error` (bool, default True), `max_num_pages`, `max_file_size`, `page_range` |
| `convert_all(source, ...)` | Convert multiple documents (iterable of sources). | Same as `convert` (batch). |
| `convert_string(content, fmt, ...)` | Convert a document given as a string (Markdown/HTML). | Format, headers, raises_on_error, max_num_pages, max_file_size. |
| `initialize_pipeline(fmt)` | Pre-initialize the pipeline for a format. | Format. |

### `ConversionResult` Model

| Field | Type | Description |
|---|---|---|
| `document` | `DoclingDocument` | The converted document (if successful). |
| `status` | `ConversionStatus` | `SUCCESS` / `PARTIAL_SUCCESS` / `PENDING` / `STARTED` / `SKIPPED` / `FAILURE`. |
| `input` | — | Input document descriptor. |
| `errors` | list | Conversion errors. |
| `pages` | — | Per-page info. |
| `timings` | dict | Per-component timing. |
| `confidence` | — | Confidence scores. |
| `assembled` | — | Assembled output path. |
| `timestamp` | datetime | Conversion timestamp. |
| `version` | string | Docling version. |

Helper methods: `has_errors`, `has_inference_errors`, `has_parse_errors`, `has_timeout_errors`, `load`, `save`.

### `FormatOption` (per-format configuration)

| Field | Description |
|---|---|
| `backend` | Backend class (parser). |
| `backend_options` | Backend-specific options. |
| `model_config` | Model configuration. |
| `pipeline_cls` | Pipeline class (e.g. `StandardPdfPipeline`, `SimplePipeline`). |
| `pipeline_options` | Pipeline-specific options (e.g. `PdfPipelineOptions`). |

Format-specific subclasses: `PdfFormatOption`, `ImageFormatOption`, `WordFormatOption`, `PowerpointFormatOption`, `MarkdownFormatOption`, `AsciiDocFormatOption`, `HTMLFormatOption`.

### Analysis

The API design emphasizes portability: the same `ConversionResult` / `DoclingDocument` objects are returned whether you run locally (`DocumentConverter`) or remotely (`DoclingServiceClient`). This means development can happen locally and move to the managed service by swapping the client — no output-handling code changes. The `format_options` dict is the extension point for customization (custom backends, pipelines, model configs) while keeping sensible defaults. The `convert_string` method is a convenience for in-memory Markdown/HTML content that never touches the filesystem. The `ConversionResult` helper predicates (`has_*_errors`) make error triage ergonomic.

---

## 15. CLI

### Main Concepts

Docling provides a simple, convenient CLI (`docling`) for direct terminal usage, scripting, and CI/CD.

### Example Commands

```bash
# Convert a single file to Markdown (default)
docling myfile.pdf

# Convert to Markdown and JSON, without OCR
docling myfile.pdf --to json --to md --no-ocr

# Convert PDF files in a directory to Markdown
docling ./input/dir --from pdf

# Convert PDF and Word files to Markdown and JSON
docling ./input/dir --from pdf --from docx --to md --to json --output ./scratch

# Enrichments
docling --enrich-code FILE
docling --enrich-formula FILE
docling --enrich-picture-classes FILE
```

### Common Flags

| Flag | Description |
|---|---|
| `--from` | Input format(s) to accept. |
| `--to` | Output format(s) (`md`, `json`, `html`, `text`, `doctags`, `doclang`, `dclx`, `vtt`, `chunks`). |
| `--no-ocr` | Disable OCR. |
| `--output` | Output directory. |
| `--abort-on-error` | Abort batch on first error. |
| `--enrich-code` / `--enrich-formula` / `--enrich-picture-classes` | Enable enrichments. |
| `--chunks-type` / `--chunks-max-tokens` / `--chunks-tokenizer` | Configure chunk output. |

### Analysis

The CLI mirrors the library's capabilities with familiar, composable flags. The multi-format/multi-output flag repetition (`--from pdf --from docx --to md --to json`) is explicit and scriptable. Batch directory processing with `--abort-on-error` supports CI pipelines. The enrichment flags make one-off enrichment jobs trivial without writing Python. Chunk configuration flags make the CLI a complete RAG-prep tool.

---

## 16. Integrations & Agentic Ecosystem

### Main Concepts

Docling provides plug-and-play integrations with the generative AI ecosystem, positioning it as the document-processing layer for RAG and agentic workflows.

### Agentic / AI Dev Framework Integrations

| Framework | Integration |
|---|---|
| LangChain | `langchain_docling` loader; `ExportType.DOC_CHUNKS` (default) or `ExportType.MARKDOWN`; configurable chunker, metadata extractor. |
| LlamaIndex | Native document reader. |
| Haystack | RAG pipeline integration. |
| Crew AI | Agentic workflow integration. |
| Bee Agent Framework | Agent framework integration. |
| Hector / Semantica / txtai / Langflow | Additional integrations. |

### Other Integrations

Apify, Data Prep Kit, InstructLab, NVIDIA, Prodigy, spaCy, RHEL AI, Arconia, Cloudera, DocETL, Kotaemon, OpenContracts, Open WebUI, Quarkus, Vectara.

### MCP Server

Docling can connect to any agent via the **Model Context Protocol (MCP) server**, enabling AI assistants (Claude, Cursor, etc.) to invoke document conversion in-conversation.

### Agent Skill

An installable agent skill (`SKILL.md`) for coding agents provides Docling document intelligence guidance.

### LangChain Loader Parameters

| Parameter | Description |
|---|---|
| `file_path` | Source as single str (URL or local) or iterable. |
| `converter` | Optional custom `DocumentConverter` instance. |
| `convert_kwargs` | Optional conversion kwargs. |
| `export_type` | `ExportType.DOC_CHUNKS` (default) or `ExportType.MARKDOWN`. |
| `md_export_kwargs` | Markdown export kwargs (Markdown mode). |
| `chunker` | Custom Docling chunker instance (doc-chunk mode). |
| `meta_extractor` | Custom metadata extractor. |

### Analysis

The integration breadth is a defining strength — Docling is designed to be the document ingestion layer *behind* the AI stack you already use. The LangChain loader's dual export mode (chunks vs Markdown) with pluggable chunker and metadata extractor shows the intended use: Docling handles structure-aware chunking, the framework handles retrieval/generation. The MCP server is the agentic-era integration — it makes any MCP-compatible assistant able to convert and query documents without bespoke code. The agent skill formalizes best practices for coding agents. This ecosystem approach (rather than building a competing RAG/search product) keeps Docling focused on its core competency: document understanding.

---

## 17. Deployment & Managed Service

### Self-Hosted Deployment

- **Python package:** `pip install docling-serve[ui]` → run server.
- **Container images:** `quay.io/docling-project/docling-serve` (GPU), `quay.io/docling-project/docling-serve-cpu` (CPU); CUDA 12.8/13.0 and AMD ROCm 6.3 variants.
- **Kubernetes:** Docling Operator for K8s deployment.
- **OpenShift:** OAuth-proxy sidecar with TLS termination; bearer-token auth.

### Key Server Environment Variables

| Variable | Description |
|---|---|
| `DOCLING_SERVE_API_KEY` | Enable API-key auth. |
| `DOCLING_SERVE_ENABLE_UI` | Enable the web UI playground (`/ui`). |
| `DOCLING_SERVE_ENABLE_REMOTE_SERVICES` | Allow remote model API calls (picture description, VLM). |
| `DOCLING_SERVE_MAX_SYNC_WAIT` | Max seconds for synchronous requests (increase for large docs). |
| `DOCLING_SERVE_ENG_LOC_NUM_WORKERS` | Number of local engine workers. |
| `UVICORN_WORKERS` | Keep at 1 to avoid "Task Not Found" errors. |
| `OMP_NUM_THREADS` / `MKL_NUM_THREADS` | CPU thread configuration. |

### Managed Service — Docling for IBM watsonx

- Fully managed, hosted instance of the Docling service.
- Exposes the **same REST API** as the self-hosted server.
- Provide service URL + API key; call like any endpoint.
- Client code stays portable — typically just swap base URL and supply key.
- No infrastructure to run: no servers, GPUs, scaling, upgrades, or monitoring.

### Analysis

The "same API, your choice of deployment" model is the key architectural decision. It means zero lock-in: start with the managed service for speed-to-production, move to self-hosted for data sovereignty or cost control, and the integration code is identical. The container variants (CPU, CUDA, ROCm) cover the full hardware spectrum. The OpenShift OAuth deployment pattern is enterprise-ready. The operational caveats (UVICORN_WORKERS=1, MAX_SYNC_WAIT tuning, REMOTE_SERVICES opt-in) are the practical knowledge needed for production self-hosting. The managed service removes all of this — the value proposition is operational simplicity plus the same conversion quality.

---

## 18. Pricing & Resource Units

### Main Concepts

Docling for IBM watsonx pricing is based on **Resource Units (RUs)** to normalize usage across document types.

### RU Definition

- **1 RU = 1,000 pages** or objects (PDFs, images, slides)
- **1 RU = 50 million characters** (plain text, spreadsheets)

### Pricing

| Plan | Detail |
|---|---|
| Free trial | 30 days, 5,000 pages included, all features |
| Pay-as-you-go | $4 per 1,000 pages |
| Annual subscription | Available (contact) |

### Analysis

The RU model is a pragmatic normalization: page-based billing for visual documents (where processing cost scales with pages) and character-based for text/spreadsheets (where page count is meaningless). The free trial with full feature access (including enrichments and VLM) is a strong onboarding path. The pay-as-you-go rate ($4/1,000 pages) is competitive for managed document AI. The open-source option (free, self-hosted) remains the zero-cost alternative for teams with infrastructure.

---

## 19. Capability Summary & Cross-Reference

### Capability → Endpoint / API Map

| Capability | REST Endpoint(s) | Python API | CLI |
|---|---|---|---|
| Sync convert (URL/base64) | `POST /v1/convert/source` | `client.convert()` / `converter.convert()` | `docling <url>` |
| Sync convert (file upload) | `POST /v1/convert/file` | `client.convert(Path)` | `docling <file>` |
| Async convert (URL/base64) | `POST /v1/convert/source/async` | — | — |
| Async convert (file) | `POST /v1/convert/file/async` | — | — |
| Poll task status | `GET /v1/status/poll/{task_id}` | — | — |
| Stream task status | `WS /v1/status/ws/{task_id}` | — | — |
| Fetch result | `GET /v1/result/{task_id}` | — | — |
| OCR | options: `do_ocr`, `force_ocr`, `ocr_preset`, `ocr_lang` | `PdfPipelineOptions` | `--no-ocr` |
| Table structure | option: `table_mode` (`fast`/`accurate`) | `PdfPipelineOptions` | — |
| Code enrichment | option: `do_code_enrichment` | `PdfPipelineOptions` | `--enrich-code` |
| Formula enrichment | option: `do_formula_enrichment` | `PdfPipelineOptions` | `--enrich-formula` |
| Picture classification | option: `do_picture_classification` | `PdfPipelineOptions` | `--enrich-picture-classes` |
| Picture description | options: `do_picture_description`, `picture_description_preset` / `picture_description_custom_config` | `PdfPipelineOptions` | — |
| VLM pipeline | option: `pipeline` | VLM pipeline class | — |
| Audio/video (ASR) | format support (input) | ASR pipeline | — |
| Chunking | output: `chunks` (JSONL) | `HybridChunker` / `HierarchicalChunker` / `LineBasedTokenChunker` | `--to chunks`, `--chunks-*` |
| Export (md/json/html/text/doctags/doclang/dclx/vtt) | option: `to_formats` | `DoclingDocument.export_to_*()` | `--to <fmt>` |
| Health/readiness | `GET /health`, `GET /ready` | — | — |
| Version | `GET /version` | — | — |
| API schema | `GET /docs` | — | — |

### Pipeline Comparison

| Pipeline | Best For | Models | Cost |
|---|---|---|---|
| `StandardPdfPipeline` | PDFs needing layout + table structure | DocLayNet (layout), TableFormer (tables), OCR | moderate |
| `SimplePipeline` | Non-PDF formats (DOCX, HTML, MD…) | Minimal (no layout models) | low |
| VLM pipeline | End-to-end vision-language conversion | GraniteDocling (258M) or remote VLM | varies |

### Core Object Relationships

```
DocumentConverter
  ├── allowed_formats (InputFormat enum)
  └── format_options: { InputFormat -> FormatOption }
        ├── backend (parser)
        ├── pipeline_cls (StandardPdfPipeline | SimplePipeline | VLM)
        └── pipeline_options (e.g. PdfPipelineOptions)
              ├── do_ocr / force_ocr / ocr_preset / ocr_lang
              ├── table_mode
              ├── do_code_enrichment / do_formula_enrichment
              ├── do_picture_classification / do_picture_description
              ├── picture_description_options (VLM | API)
              └── generate_picture_images / images_scale
                    │
                    ▼
              ConversionResult
                ├── status (SUCCESS | PARTIAL_SUCCESS | FAILURE | SKIPPED)
                ├── document: DoclingDocument
                │     ├── texts / tables / pictures / key_value_items
                │     ├── body (tree) / furniture (tree) / groups
                │     └── export_to_markdown() / _json() / _html() / _text() / _doctags()
                ├── errors / timings / confidence / pages
                └── save() / load()
                      │
                      ▼
                Chunker (Hierarchical | Hybrid | LineBased)
                  └── chunks (Iterator[BaseChunk]) → JSONL
```

### Key Design Principles

1. **Unified representation** — One `DoclingDocument` model across all input formats; all exports and chunks derive from it.
2. **Structure-preserving** — Layout, hierarchy, reading order, tables, formulas, bounding boxes, and provenance are first-class.
3. **Same API, any deployment** — Managed SaaS, self-hosted container, or local library all expose the same REST/Python interface; client code is portable.
4. **Pipeline extensibility** — Backends, pipelines, enrichments, and chunkers are all subclassable base classes; the model chain is fully customizable.
5. **Quality knobs are per-conversion** — OCR, table mode, enrichments, VLM pipeline, image mode are options on each request, not server-level locks.
6. **RAG-first** — Native chunkers, plug-and-play framework integrations, and chunk/JSONL output position Docling as the ingestion layer for retrieval pipelines.
7. **Open at the core** — MIT-licensed, LF AI & Data Foundation project; the managed service builds on the same codebase with no proprietary lock-in.

---

*Sources: [IBM Docling product page](https://www.ibm.com/products/docling), [Docling docs (GitHub Pages)](https://docling-project.github.io/docling), [REST API reference](https://docling-project.github.io/docling/usage/api_server/rest_api/), [Concepts](https://docling-project.github.io/docling/concepts/), [Python API reference](https://docling-project.github.io/docling/reference/document_converter/), [Supported formats](https://docling-project.github.io/docling/usage/supported_formats/), [Enrichments](https://docling-project.github.io/docling/usage/enrichments/), [Managed service](https://docling-project.github.io/docling/usage/api_server/managed/), [docling-serve GitHub](https://github.com/docling-project/docling-serve), [docling-serve PyPI](https://pypi.org/project/docling-serve/), [Granite Docling](https://www.ibm.com/granite/docs/models/docling), [Docling Technical Report (arXiv:2408.09869)](https://arxiv.org/abs/2408.09869), [DocTags paper (arXiv:2503.11576)](https://arxiv.org/pdf/2503.11576), [LangChain integration](https://docs.langchain.com/oss/python/integrations/document_loaders/docling). Last reviewed: July 2026.*