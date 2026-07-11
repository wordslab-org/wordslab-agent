# Mistral API Analysis — Document AI & Search Toolkit

> **API Base URL:** `https://api.mistral.ai/v1` | **Docs:** `https://docs.mistral.ai` | **Auth:** Bearer token (`MISTRAL_API_KEY`)
> **SDK:** `mistralai` (Python / TypeScript) | **Search Toolkit package:** `mistralai-search-toolkit` (Python 3.12+)
> **Description:** Mistral offers two complementary document intelligence capabilities under its Studio API. **Document AI** is a managed cloud API for OCR, structured annotation extraction, and document question-answering. **Search Toolkit** is a Python framework for building end-to-end RAG retrieval pipelines — ingestion (load, extract, chunk, enrich, embed, index) and retrieval (preprocess, retrieve, rerank, cache) — with swappable components and a Vespa-backed vector store.

---

## Table of Contents

1. [Platform Overview & Core Concepts](#1-platform-overview--core-concepts)
2. [Document AI — OCR Processor](#2-document-ai--ocr-processor)
3. [Document AI — Annotations](#3-document-ai--annotations)
4. [Document AI — Document QnA](#4-document-ai--document-qna)
5. [Search Toolkit — Architecture & Data Model](#5-search-toolkit--architecture--data-model)
6. [Search Toolkit — Ingestion: File Loaders](#6-search-toolkit--ingestion-file-loaders)
7. [Search Toolkit — Ingestion: Document Extractors](#7-search-toolkit--ingestion-document-extractors)
8. [Search Toolkit — Ingestion: Text Splitters](#8-search-toolkit--ingestion-text-splitters)
9. [Search Toolkit — Ingestion: Chunk Enrichers](#9-search-toolkit--ingestion-chunk-enrichers)
10. [Search Toolkit — Ingestion: Embedders](#10-search-toolkit--ingestion-embedders)
11. [Search Toolkit — Pipeline Orchestration](#11-search-toolkit--pipeline-orchestration)
12. [Search Toolkit — Retrieval: Query Preprocessing](#12-search-toolkit--retrieval-query-preprocessing)
13. [Search Toolkit — Retrieval: Retrievers](#13-search-toolkit--retrieval-retrievers)
14. [Search Toolkit — Retrieval: Rerankers](#14-search-toolkit--retrieval-rerankers)
15. [Search Toolkit — Retrieval: Semantic Cache](#15-search-toolkit--retrieval-semantic-cache)
16. [Search Toolkit — QueryEngine Orchestration](#16-search-toolkit--queryengine-orchestration)
17. [Search Toolkit — Vespa Integration](#17-search-toolkit--vespa-integration)
18. [Cross-Cutting: Installation, Extras & Component Summary](#18-cross-cutting-installation-extras--component-summary)
19. [Capability Summary & Cross-Reference](#19-capability-summary--cross-reference)

---

## 1. Platform Overview & Core Concepts

Mistral's document intelligence offering is split across two distinct products:

### Document AI (Managed Cloud API)

A set of HTTP/SDK services accessible via `client.ocr.process` (SDK) or `https://api.mistral.ai/v1/ocr` (REST). Three services:

- **OCR Processor** — Extracts text, images, tables, structural blocks, and metadata from PDFs and images using the `mistral-ocr-latest` model. Multilingual support.
- **Annotations** — Structured data extraction from documents using user-defined schemas (BBox annotations on images, or document-level annotations).
- **Document QnA** — Natural-language question answering over documents by combining OCR with Mistral's LLM chat completions.

### Search Toolkit (Python RAG Framework)

A pip-installable framework (`mistralai-search-toolkit`) for building information retrieval systems. Every component is swappable. Two orchestrated workflows:

- **Ingestion Pipeline** — `FileLoader` → `DocumentExtractor` → `TextSplitter` → (optional) `ChunkEnricher` → `Embedder` → vector store. Orchestrated by the `Pipeline` class.
- **Retrieval QueryEngine** — (optional) query preprocessor → `Retriever`(s) → (optional) `Reranker`(s). Orchestrated by the `QueryEngine` class.

The two sides are connected through a shared **vector store** (Vespa by default). The same store object is passed as `stores=` to the `Pipeline` (writes) and as `client=` to retrievers (reads).

### End-to-End Flow

```
Raw files ──▶ Pipeline ──▶ Vector Store (Vespa) ──▶ QueryEngine ──▶ Ranked results
   │         (load, extract,                         (preprocess,
   │          chunk, enrich,                          retrieve,
   │          embed, index)                           rerank, cache)
   │
   └── Also usable standalone via Document AI API (OCR / Annotations / QnA)
```

### Key Differentiators

- **Two access levels** — Use Document AI as a simple cloud API, or use Search Toolkit for full control over a RAG pipeline.
- **Fully swappable architecture** — Every Search Toolkit component (loaders, extractors, splitters, enrichers, embedders, retrievers, rerankers, caches) can be replaced with custom implementations.
- **Deterministic identity** — Documents and chunks have UUID5 IDs derived from `source_id` + `locator`, making ingestion idempotent.
- **Multilingual OCR** — Supports a wide range of languages for document text extraction.
- **Batch processing** — OCR and other APIs can be used at scale via Mistral's Batch Inference service for cost-effectiveness.

---

## 2. Document AI — OCR Processor

**Endpoint:** `POST https://api.mistral.ai/v1/ocr`
**SDK:** `client.ocr.process(...)`
**Model:** `mistral-ocr-latest` (OCR 4+ = `mistral-ocr-4-0`)

### Main Concepts

The OCR processor extracts text, images, tables, structural blocks, and metadata from PDF documents and images. It returns structured JSON with per-page content in markdown format, along with image bounding boxes, table data, hyperlinks, and optional confidence scores.

### Input Methods

| Method | Description |
|---|---|
| **Public URL** | Pass a publicly accessible `document_url` or `image_url` |
| **Base64 encoded** | Pass a base64-encoded PDF or image directly |
| **Cloud upload** | Upload a PDF to Mistral Cloud first, then reference it |

### API Parameters — `client.ocr.process()`

| Parameter | Type | Default | Description |
|---|---|---|---|
| `model` | `str` | _(required)_ | OCR model name, e.g. `"mistral-ocr-latest"` or `"mistral-ocr-4-0"` |
| `document` | `dict` | _(required)_ | Document input: `{"type": "document_url", "document_url": "..."}` for PDFs, or `{"type": "image_url", "image_url": "..."}` for images |
| `table_format` | `"markdown" \| "html" \| None` | `None` | Output format for extracted tables (OCR 2512+) |
| `extract_header` | `bool` | `False` | Extract page headers (OCR 2512+) |
| `extract_footer` | `bool` | `False` | Extract page footers (OCR 2512+) |
| `include_image_base64` | `bool` | `False` | Include base64-encoded image data in the response |
| `include_blocks` | `bool` | `False` | Extract structural blocks with bounding boxes (OCR 4+ only) |
| `confidence_scores_granularity` | `"page" \| "word" \| None` | `None` | Confidence score detail level |
| `bbox_annotation_format` | `ResponseFormat` | `None` | Schema for BBox annotations (see [Annotations](#3-document-ai--annotations)) |
| `document_annotation_format` | `ResponseFormat` | `None` | Schema for document-level annotations |

### Response Structure

```json
{
  "pages": [
    {
      "index": 0,
      "markdown": "extracted text in markdown...",
      "images": [],
      "tables": [],
      "hyperlinks": [],
      "header": null,
      "footer": null,
      "dimensions": {},
      "confidence_scores": null,
      "blocks": null
    }
  ],
  "model": "mistral-ocr-latest",
  "document_annotation": null,
  "usage_info": {}
}
```

| Response field | Type | Description |
|---|---|---|
| `pages[].index` | `int` | Page index (0-based) |
| `pages[].markdown` | `str` | Main output — raw markdown content with image/table placeholders |
| `pages[].images` | `list` | Image info when images are extracted; placeholders like `![img-0.jpeg](img-0.jpeg)` appear in markdown |
| `pages[].tables` | `list` | Table info when `table_format` is set; placeholders like `[tbl-3.html](tbl-3.html)` appear in markdown |
| `pages[].hyperlinks` | `list` | Hyperlinks detected on the page |
| `pages[].header` | `str \| null` | Header content when `extract_header=True` |
| `pages[].footer` | `str \| null` | Footer content when `extract_footer=True` |
| `pages[].dimensions` | `dict` | Page dimensions |
| `pages[].confidence_scores` | `dict \| null` | Contains `average_page_confidence_score`, `minimum_page_confidence_score`, and `word_confidence_scores` (word granularity only) |
| `pages[].blocks` | `list \| null` | Paragraph-level bounding boxes with block labels when `include_blocks=True` |
| `model` | `str` | Model used |
| `document_annotation` | `dict \| null` | Document annotation info (when annotations are used) |
| `usage_info` | `dict` | Usage information |

### Block Extraction (OCR 4+)

When `include_blocks=True`, each page includes a `blocks` array in **reading order**. Each block has: `type`, `top_left_x`, `top_left_y`, `bottom_right_x`, `bottom_right_y`, `content`.

| Block type | Description |
|---|---|
| `text` | A paragraph of body text |
| `title` | A document title or section heading |
| `list` | A bulleted or numbered list |
| `table` | A table region (includes `table_id` referencing `tables`) |
| `image` | An image region (includes `image_id` referencing `images`) |
| `equation` | A math equation |
| `caption` | A caption associated with a figure or table |
| `code` | A code block |
| `references` | A bibliography or references section |
| `aside_text` | A sidebar, callout, or marginal text block |
| `header` | A page header |
| `footer` | A page footer |
| `signature` | A signature region (`content` = transcribed name or empty string) |

### Confidence Scores

| Granularity | Description |
|---|---|
| `"page"` | Aggregate stats per page: `average_page_confidence_score`, `minimum_page_confidence_score` |
| `"word"` | Everything from `"page"` plus a `word_confidence_scores` array with per-word confidence on each page and table entry |

### Code Example

```python
import os
from mistralai.client import Mistral

client = Mistral(api_key=os.environ["MISTRAL_API_KEY"])

ocr_response = client.ocr.process(
    model="mistral-ocr-latest",
    document={
        "type": "document_url",
        "document_url": "https://arxiv.org/pdf/2201.04234"
    },
    table_format="html",
    extract_header=True,
    extract_footer=True,
    include_image_base64=True,
    include_blocks=True,
    confidence_scores_granularity="word",
)
```

### Scaling

For large-scale processing, use Mistral's [Batch Inference service](https://docs.mistral.ai/studio-api/batch-processing) for parallel, cost-effective OCR.

---

## 3. Document AI — Annotations

**Endpoint:** Same as OCR (`POST https://api.mistral.ai/v1/ocr`) with annotation parameters
**SDK:** `client.ocr.process(..., bbox_annotation_format=..., document_annotation_format=...)`

### Main Concepts

Annotations extend the OCR API with structured data extraction. You define a schema (Pydantic, Zod, or JSON Schema) and the API returns extracted data conforming to that schema. Two annotation types:

- **BBox Annotation** (`bbox_annotation_format`) — Annotates images extracted from the document. Each extracted image is classified/described according to your schema.
- **Document Annotation** (`document_annotation_format`) — Extracts structured information at the document level.

### Workflow

1. Define a data model (Pydantic / Zod / JSON Schema) describing the fields you want extracted.
2. Pass the schema as `bbox_annotation_format` or `document_annotation_format` to `client.ocr.process()`.
3. The response includes `document_annotation` (or per-image BBox annotation data) conforming to your schema.

### Defining the Data Model

```python
from pydantic import BaseModel, Field

class Image(BaseModel):
    image_type: str = Field(..., description="The type of the image.")
    short_description: str = Field(..., description="A description describing the image.")
    summary: str = Field(..., description="Summarize the image.")
```

Schemas accept nested objects, arrays, enums, etc. Field descriptions serve as detailed instructions during annotation.

### API Call with BBox Annotation

```python
import os
from mistralai.client import Mistral, DocumentURLChunk
from mistralai.extra import response_format_from_pydantic_model

client = Mistral(api_key=os.environ["MISTRAL_API_KEY"])

response = client.ocr.process(
    model="mistral-ocr-latest",
    document=DocumentURLChunk(
        document_url="https://arxiv.org/pdf/2410.07073"
    ),
    bbox_annotation_format=response_format_from_pydantic_model(Image),
    include_image_base64=True,
)
```

### BBox Annotation Output

Each extracted image is returned with base64 data and annotation fields:

```json
{
  "image_base64": "data:image/jpeg;base64,/9j/4AAQ..."
}
```

```json
{
  "image_type": "scatter plot",
  "short_description": "Comparison of different models based on performance and cost.",
  "summary": "The image consists of two scatter plots comparing various models..."
}
```

### Accepted Formats

Annotations work with the same document formats as OCR: PDF (URL, base64, cloud upload) and images (URL, base64).

---

## 4. Document AI — Document QnA

**Endpoint:** Uses the Chat Completions API (`client.chat.complete(...)`)
**Models:** Any Mistral chat model (e.g. `mistral-small-latest`)

### Main Concepts

Document QnA combines OCR with LLM capabilities for natural-language interaction with document content. The workflow has two steps:

1. **Document AI (OCR)** — Extracts text, structure, and formatting, creating a machine-readable version of the document.
2. **Language Model Understanding** — The extracted document content is analyzed by a Mistral LLM. You ask questions in natural language; the model understands context and relationships and provides answers grounded in the document.

### Key Capabilities

- Question answering about specific document content
- Information extraction and summarization
- Document analysis and insights
- Multi-document queries and comparisons
- Context-aware responses considering the full document

### API Pattern

Document QnA uses the standard Chat Completions API with a `document_url` content block in the user message. The model internally processes the document (OCR) and answers the question.

```python
import os
from mistralai.client import Mistral

client = Mistral(api_key=os.environ["MISTRAL_API_KEY"])

messages = [
    {
        "role": "user",
        "content": [
            {
                "type": "text",
                "text": "What is the last sentence in the document?"
            },
            {
                "type": "document_url",
                "document_url": "https://arxiv.org/pdf/1805.04770"
            }
        ]
    }
]

chat_response = client.chat.complete(
    model="mistral-small-latest",
    messages=messages,
)
```

### Input Variants

| Method | Description |
|---|---|
| **PDF URL** | Pass `"type": "document_url"` with a public URL |
| **Base64 PDF** | Pass a base64-encoded PDF as the document content |
| **Uploaded PDF** | Upload to Mistral Cloud first, then reference the URL |

### Common Use Cases

- Analyzing research papers and technical documents
- Extracting information from business documents
- Processing legal documents and contracts
- Building document Q&A applications
- Automating document-based workflows

---

## 5. Search Toolkit — Architecture & Data Model

**Package:** `mistralai-search-toolkit` (PyPI)
**Requires:** Python 3.12+
**Recommended:** `uv` for dependency management

### Architecture

The toolkit has two orchestration classes and a unified data model:

```
┌─────────────────────────────────────────────────────────────────┐
│  INGESTION (Pipeline)                    RETRIEVAL (QueryEngine) │
│                                                                 │
│  FileLoader ──▶ DocumentExtractor       QueryPreprocessor        │
│       │              │                  (LLMQueryRewriter /      │
│       │              │                   LLMQueryExtension)      │
│       │              ▼                          │                │
│       │       TextSplitter                      ▼                │
│       │              │                    Retriever(s)           │
│       │              ▼                    (VectorRetriever)      │
│       │       ChunkEnricher                     │                │
│       │         (optional)                      ▼                │
│       │              │                    Reranker(s)            │
│       │              ▼                    (LLMReRanker /         │
│       │          Embedder                   CrossEncoder / RRF)  │
│       │              │                          │                │
│       └──────────────▼──────────────────────────▼                │
│              VECTOR STORE (Vespa)                               │
│              + Semantic Cache (optional)                        │
└─────────────────────────────────────────────────────────────────┘
```

### Data Model

#### `Document`

Full output of an extractor for one source file.

| Field | Type | Description |
|---|---|---|
| `id` | `str` | Deterministic UUID5 hash of `source_id` |
| `source_id` | `str` | Stable identifier of the source (filepath, URL, or custom scheme like `arxiv:1706.03762`) |
| `content` | `str` | Full extracted text |
| `chunks` | `list[DocumentChunk]` | The document's chunks |
| `metadata` | `DocumentMetadata` | Extensible, immutable metadata |

#### `DocumentChunk`

The unit that gets indexed and retrieved.

| Field | Type | Description |
|---|---|---|
| `id` | `str` | Deterministic UUID5 hash of `source_id` + `locator` |
| `source_id` | `str` | Same `source_id` as the parent document |
| `locator` | `str` | Semantic position within the source (e.g. `char:0-500`, `page:3:char:0-500`, `summary:char:0-512`) |
| `start_offset` | `int` | Inclusive character offset |
| `end_offset` | `int` | Exclusive character offset |
| `parent_ref` | `str \| None` | ID of the parent `Document` |
| `chunk_type` | `ChunkType` | `content`, `image_annotation`, or `summary` |
| `content` | `str` | The chunk text |
| `metadata` | `DocumentChunkMetadata` | Extensible, immutable metadata |
| `embedding` | `list[float] \| None` | Populated once embedded |

#### `ChunkType` Values

| Type | Description |
|---|---|
| `content` | A slice of the document's main text |
| `image_annotation` | Text describing an image (e.g. an OCR caption) |
| `summary` | A generated summary of the document or a section |

#### Identity System

- **`source_id`** — Stable identifier of the source document. Set on `File.source_id` (defaults to file path/name) and stamped by extractors onto the resulting document and every chunk. Identity flows from `source_id`, not storage location, so re-ingesting the same logical source produces the same IDs even if the file moved.
- **`locator`** — Semantic position of a chunk within its source. Formats: `char:{start}-{end}`, `page:{n}:char:{start}-{end}`, or type-prefixed (e.g. `summary:char:0-512`).
- **Deterministic IDs** — UUID5 hashes: `Document.id` = hash of `source_id`; `DocumentChunk.id` = hash of `source_id` + `locator`; `parent_ref` = hash of `source_id`. Re-ingesting the same content overwrites the same records → indexing is **idempotent**.

#### Metadata Subtypes

- `DocumentFileMetadata` — fields: `filename`, `filepath`
- `PagedDocumentChunkMetadata` — field: `page_number`
- Custom keys can be added; a metadata key must not collide with a model field name.

#### `SearchResultChunk` (retrieval side)

Carries the same identity contract: `id`, `source_id`, `locator`, `parent_ref`, `chunk_type`. Every result maps straight back to the ingested chunk.

### Context Objects

| Context | Used by | Description |
|---|---|---|
| `IngestContext` | Extractors, splitters, enrichers | Ingestion-side context |
| `RetrievalContext` | Embedders, retrievers | Retrieval-side context |

---

## 6. Search Toolkit — Ingestion: File Loaders

**Module:** `mistralai.search.toolkit.ingestion.loaders`

### Main Concepts

File loaders read raw bytes from a source (local filesystem, cloud storage, or custom) and return `File` objects for downstream extraction. All loaders implement the `FileLoader` protocol.

### Built-in Implementations

| Class | Source | Extra required |
|---|---|---|
| `FilesystemFileLoader` | Local filesystem | Core (none) |
| `S3FileLoader` | AWS S3 / S3-compatible (MinIO, Ceph) | `mistralai-search-toolkit-storage-s3` |
| `AzureBlobFileLoader` | Azure Blob Storage | `mistralai-search-toolkit-storage-azure` |
| `GCSFileLoader` | Google Cloud Storage | `mistralai-search-toolkit-storage-gcs` |

### Constructor Parameters

#### `FilesystemFileLoader`

| Parameter | Type | Default | Description |
|---|---|---|---|
| `root` | `Path \| str` | `"/"` | Root directory; paths resolving outside are rejected (path traversal protection) |
| `max_file_size` | `int \| None` | `None` | Max file size in bytes; rejected before loading. `None` = no limit |

#### `S3FileLoader`

| Parameter | Type | Default | Description |
|---|---|---|---|
| `bucket_name` | `str` | _(required)_ | S3 bucket name |
| `region_name` | `str \| None` | `None` | AWS region (falls back to SDK/env default) |
| `endpoint_url` | `str \| None` | `None` | Custom endpoint (MinIO, Ceph, S3-compatible) |
| `aws_access_key_id` | `str \| None` | `None` | Explicit access key (falls back to credential chain) |
| `aws_secret_access_key` | `str \| None` | `None` | Explicit secret key |
| `max_file_size` | `int \| None` | `None` | Reject files larger than this many bytes |

Auth: Default AWS credentials chain (IAM role, env vars, `~/.aws/credentials`).

#### `AzureBlobFileLoader`

| Parameter | Type | Default | Description |
|---|---|---|---|
| `container_name` | `str` | _(required)_ | Azure Blob container name |
| `azure_connection_string` | `str \| None` | `None` | Connection string (required unless `use_workload_identity=True`) |
| `account_url` | `str \| None` | `None` | Azure storage account URL (required when `use_workload_identity=True`) |
| `use_workload_identity` | `bool` | `False` | Use workload identity (recommended for Azure VMs/Functions) |
| `max_file_size` | `int \| None` | `None` | Reject files larger than this many bytes |

Auth: Workload Identity (recommended), Connection string, or SAS token.

#### `GCSFileLoader`

| Parameter | Type | Default | Description |
|---|---|---|---|
| `bucket_name` | `str` | _(required)_ | GCS bucket name |
| `service_account_file` | `str \| None` | `None` | Path to service account JSON; if not provided uses ADC |
| `api_root` | `str \| None` | `None` | Custom API endpoint for local emulator (e.g. fake-gcs-server) |
| `max_file_size` | `int \| None` | `None` | Reject files larger than this many bytes |

Auth: Application Default Credentials (ADC, recommended) or service account file.

### Key Methods

| Method | Signature | Description |
|---|---|---|
| `load_file` | `async load_file(self, path: Path \| str) -> File` | Load a single file; returns a `File` with `path`, `name`, `raw` fields |

### Code Example

```python
from pathlib import Path
from mistralai.search.toolkit.ingestion.loaders import FilesystemFileLoader

loader = FilesystemFileLoader(root="/data/documents", max_file_size=50 * 1024 * 1024)
file = await loader.load_file(Path("report.pdf"))
```

### Custom Loaders

Implement the `FileLoader` protocol with `async load_file(self, path) -> File`, returning `File(path=..., name=..., raw=content)`.

### Batch Loading

Use `asyncio.Semaphore` + `asyncio.gather` for concurrent batch loading with a configurable concurrency limit.

---

## 7. Search Toolkit — Ingestion: Document Extractors

**Module:** `mistralai.search.toolkit.ingestion.extractors`

### Main Concepts

Document extractors parse a `File` into a structured `Document`. Different file types require different extractors (OCR for PDFs, tabular parsing for spreadsheets, etc.). All extractors stamp `file.source_id` onto the produced `Document` and its chunks; the document ID is computed deterministically from `source_id`.

### Built-in Implementations

| Class | File types | Extra required |
|---|---|---|
| `MistralOCRExtractor` | PDF, DOCX, PPTX, ODT | Core (none) |
| `MistralAudioTranscriptionExtractor` | MP3, WAV, M4A, FLAC, OGG | Core (none) |
| `PlainTextExtractor` | TXT, MD, CSV, JS, PY, code | Core (none) |
| `HTMLExtractor` | HTML, HTM | `html-converter-markdownify` |
| `SpreadsheetExtractor` | XLS, XLSX, XLSM, XLSB, ODS, ODF | `extractor-spreadsheet` |
| `EmailExtractor` | EML, MSG | `extractor-email` |
| `NumbersExtractor` | NUMBERS (Apple Numbers) | `extractor-spreadsheet` |
| `LegacyOfficeExtractor` | DOC, PPT, HWP, HWPX | `extractor-pymupdf` |

### Constructor Parameters

#### `MistralOCRExtractor`

| Parameter | Type | Default | Description |
|---|---|---|---|
| `client` | `Mistral` | _(required)_ | Mistral client instance |
| `model_name` | `str` | `"mistral-ocr-latest"` | OCR model name |
| `timeout_seconds` | `int` | `900` | Request timeout in seconds |
| `strip_page_markdown` | `bool` | `True` | Strip leading/trailing whitespace from page markdown |
| `populate_content` | `bool` | `True` | Populate the document `content` field |
| `include_image_base64` | `bool` | `False` | Include base64 image data in the document |
| `include_image_annotation` | `bool` | `False` | Add image annotations to page markdown |
| `max_file_size_bytes` | `int \| None` | `None` | Max file size before splitting into parts |
| `pages_split_size` | `int \| None` | `None` | Pages per split when file exceeds `max_file_size_bytes` |
| `pages_group_size` | `int \| None` | `None` | Pages per API request |
| `http_headers` | `Mapping[str, str] \| None` | `None` | Custom HTTP headers forwarded to OCR API |
| `image_limit` | `int \| None` | `None` | Max images per page |
| `pages` | `Sequence[int] \| None` | `None` | Specific page numbers to extract (1-based) |
| `table_format` | `"markdown" \| "html" \| None` | `None` | Table output format |

#### `MistralAudioTranscriptionExtractor`

| Parameter | Type | Default | Description |
|---|---|---|---|
| `client` | `Mistral` | _(required)_ | Mistral client instance |
| `model_name` | `str` | `"voxtral-mini-latest"` | Transcription model name |
| `language` | `str \| None` | `None` | Target language hint |
| `diarize` | `bool` | `False` | Enable speaker diarization |
| `timestamp_granularities` | `list[str] \| None` | `None` | e.g. `["word", "segment"]` |
| `timeout_seconds` | `int` | `900` | Request timeout |
| `populate_content` | `bool` | `True` | Populate the document `content` field |
| `http_headers` | `Mapping[str, str] \| None` | `None` | Custom HTTP headers |

#### `PlainTextExtractor`

| Parameter | Type | Default | Description |
|---|---|---|---|
| `page_size` | `int` | `2000` | Characters per logical page |
| `encoding` | `str` | `"utf-8"` | Character encoding for file reading |
| `skip_encoding_detection` | `bool` | `False` | Skip encoding auto-detection |

#### `HTMLExtractor`

| Parameter | Type | Default | Description |
|---|---|---|---|
| `encoding` | `str` | `"utf-8"` | HTML file encoding |
| `decode_errors` | `str` | `"strict"` | Error handling: `"strict"`, `"ignore"`, `"replace"` |
| `converter` | `HtmlToMarkdownConverter \| None` | `None` | Custom converter; `None` uses `MarkdownifyConverter` |

**`MarkdownifyConverter` options:**

| Option | Type | Default | Description |
|---|---|---|---|
| `ignore_tags` | `list[str] \| None` | `["head", "header", "script", "style", "title", "footer", "form", "button", "nav", "iframe"]` | HTML tags to strip entirely |
| `ignore_ids` | `list[str] \| None` | `["footer", "sidebar", "cookie", "metadata"]` | Element IDs to strip |
| `ignore_classes` | `list[str] \| None` | Regex patterns: `footer`, `^ad-`, `^ad_`, `^menu$`, `^newsletter$`, `^metadata$`, `^muted$`, `vot(e|ing)` | CSS class patterns to strip |
| `escape_misc` | `bool` | `True` | Escape misc markdown chars |

#### `SpreadsheetExtractor`

| Parameter | Type | Default | Description |
|---|---|---|---|
| `include_sheet_name` | `bool` | `True` | Prepend sheet name to each CSV block |
| `row_limit` | `int \| None` | `None` | Max rows per sheet |
| `col_limit` | `int \| None` | `None` | Max columns per sheet |
| `skip_empty_sheets` | `bool` | `True` | Skip sheets with no data |
| `preserve_formula_values` | `bool` | `True` | Use formula results, not formula text |

#### `EmailExtractor`

Constructor: `EmailExtractor()` (no documented parameters). Produces a markdown document with subject, sender, recipients, date, and body. HTML bodies converted to markdown.

Helper functions: `extract_email_attachments(file.raw, extension="eml")`, `extract_eml_attachments(...)`, `extract_msg_attachments(...)` — return `list[EmailAttachment]` (filename, content_type, data, extension).

#### `NumbersExtractor`

| Parameter | Type | Default | Description |
|---|---|---|---|
| `row_limit` | `int \| None` | `None` | Max rows per sheet |
| `col_limit` | `int \| None` | `None` | Max columns per sheet |
| `include_sheet_name` | `bool` | `True` | Prepend sheet name to each CSV block |

#### `LegacyOfficeExtractor`

| Parameter | Type | Default | Description |
|---|---|---|---|
| `ocr_extractor` | `MistralOCRExtractor` | _(required)_ | OCR extractor for the converted PDF |
| `pymupdf_license_key` | `str` | _(optional)_ | PyMuPDF Pro license key |

Converts `.doc/.ppt/.hwp/.hwpx` to PDF via PyMuPDF Pro, then runs Mistral OCR. Does NOT handle `.xls` (use `SpreadsheetExtractor`).

### Key Methods

| Method | Signature | Description |
|---|---|---|
| `extract` | `async extract(self, file: File, context: IngestContext = IngestContext()) -> Document` | Protocol method for all extractors |

### Code Example

```python
import os
from mistralai.client import Mistral
from mistralai.search.toolkit.ingestion.extractors import MistralOCRExtractor

mistral_client = Mistral(api_key=os.environ["MISTRAL_API_KEY"])
extractor = MistralOCRExtractor(
    client=mistral_client,
    include_image_base64=True,
    include_image_annotation=True,
)
document = await extractor.extract(file)
```

---

## 8. Search Toolkit — Ingestion: Text Splitters

**Module:** `mistralai.search.toolkit.ingestion.text_splitters`

### Main Concepts

Text splitters divide documents into retrievable `DocumentChunk` objects. The splitting strategy and chunk size directly impact retrieval quality. All splitters automatically preserve metadata in each chunk: `page_number`, `filename`, `filepath`, `start_offset`, `end_offset`, `images`.

### Chunk Size Guidance

| LLM Context | Chunk Size | Overlap | Use case |
|---|---|---|---|
| 4k tokens | 300–500 chars | 50–100 chars | Memory-constrained, fast retrieval |
| 8k tokens | 500–1000 chars | 100–200 chars | Balanced retrieval |
| 32k+ tokens | 1000–2000 chars | 200–500 chars | Rich context, complex retrieval |
| Code/technical | 500–1000 chars | 100–200 chars | Preserve logical units |
| Legal/financial | 1000–2000 chars | 200–500 chars | Full context for interpretation |

Rule of thumb: ~400–600 chars ≈ 100–150 tokens.

### Built-in Implementations

| Class | Best for | Extra required |
|---|---|---|
| `CharacterTextSplitter` | Simple text, quick prototyping | Core (none) |
| `TokenTextSplitter` | Token-aware chunking, LLM context management | Core (none) |
| `MarkdownTextSplitter` | Markdown documents, structured content with headers | `text-splitter-langchain` |
| `SeparatorTextSplitter` | Custom splitting logic, hierarchical text | Core (none) |

### Constructor Parameters

#### `CharacterTextSplitter`

| Parameter | Type | Default | Description |
|---|---|---|---|
| `chunk_size` | `int` | `1000` | Max characters per chunk |

#### `TokenTextSplitter`

| Parameter | Type | Default | Description |
|---|---|---|---|
| `chunk_size` | `int` | `1000` | Tokens per chunk (uses Mistral tokenizer) |
| `chunk_overlap` | `int` | `0` | Tokens of overlap between chunks |
| `tokenizer_model` | `str` | `"mistral"` | Tokenizer to use for counting |

#### `SeparatorTextSplitter` (with `SeparatorTextSplitterConfig`)

| Parameter | Type | Default | Description |
|---|---|---|---|
| `chunk_size` | `int` | `1000` | Target size (characters) |
| `chunk_max_size` | `int` | `1500` | Maximum size (characters) |
| `chunk_overlap` | `int` | `200` | Overlap (characters) |
| `chunk_separators` | `list[str]` | `["\n\n", "\n", ". ", " ", ""]` | Separators tried in order |
| `keep_separator` | `bool` | — | Keep separators in chunks |
| `strip_whitespace` | `bool` | — | Strip whitespace |

#### `MarkdownTextSplitter` (with `MarkdownTextSplitterConfig`)

Inherits all `SeparatorTextSplitterConfig` params, plus:

| Parameter | Type | Default | Description |
|---|---|---|---|
| `headers_to_split_on` | `list[tuple[str, str]]` | `[("#", "Header 1"), ("##", "Header 2"), ("###", "Header 3")]` | Each tuple is `(header_prefix, label)` |
| `strip_headers` | `bool` | `False` | Remove header lines from chunk content |

### Key Methods

| Method | Signature | Description |
|---|---|---|
| `split_document` | `split_document(self, document) -> list[DocumentChunk]` | Split a document into chunks (built-in splitters) |
| `split_text` | `async split_text(self, text: str, context: IngestContext) -> list[TextFragment]` | Protocol method for custom splitters; `TextFragment` has `content`, `start_offset`, `end_offset` |

### Code Example

```python
from mistralai.search.toolkit.ingestion.text_splitters import (
    MarkdownTextSplitter, MarkdownTextSplitterConfig,
)

config = MarkdownTextSplitterConfig(
    headers_to_split_on=[("#", "h1"), ("##", "h2"), ("###", "h3")],
    strip_headers=False,
    chunk_size=1000,
    chunk_overlap=200,
)
splitter = MarkdownTextSplitter(config=config)
chunks = splitter.split_document(document)
```

---

## 9. Search Toolkit — Ingestion: Chunk Enrichers

**Module:** `mistralai.search.toolkit.ingestion.enrichment`

### Main Concepts

Chunk enrichers add custom metadata to chunks during ingestion — information from external sources, classifications, tags, or computed metadata. Enrichers are applied sequentially in the order listed in a `Pipeline`; each receives the output of the previous one.

### Built-in Implementations

| Class | Purpose |
|---|---|
| `SummaryEnricher` | Generate document summaries using an LLM |

### `SummaryEnricher` Constructor Parameters

| Parameter | Type | Default | Description |
|---|---|---|---|
| `llm_provider` | `ChatLLMProvider` | _(required)_ | LLM provider for summarization |
| `summary_config` | `SummaryConfig \| None` | `None` | Configuration object |

**`SummarizeRequestConfig`** (controls the LLM request):

| Parameter | Type | Default | Description |
|---|---|---|---|
| `model` | `str` | `"mistral-small-latest"` | Model for summarization |
| `prompt` | `str` | `"Summarize the document in less than 5 lines."` | Summarization prompt |
| `max_tokens` | `int` | `256` | Max tokens in the summary |
| `truncate_at` | `int \| None` | `32768` | Truncate document content before sending to LLM |
| `temperature` | `float \| None` | `0.6` | LLM temperature |

**`SummaryRequestOptions`** (controls how summary is injected into pipeline output):

| Parameter | Type | Default | Description |
|---|---|---|---|
| `include_summary_chunk` | `bool` | `True` | Add a dedicated summary chunk to the chunk list |
| `propagate_summary_to_chunks` | `bool` | `False` | Prepend summary to every chunk's content |
| `populate_document_metadata` | `bool` | `True` | Store summary in document metadata |
| `fail_on_generation_error` | `bool` | `False` | Raise on failure instead of logging and continuing |

By default, `SummaryEnricher` is non-breaking: if summary generation fails, it logs and returns original chunks unchanged.

### Key Methods

| Method | Signature | Description |
|---|---|---|
| `enrich_chunks` | `async enrich_chunks(self, chunks: list[DocumentChunk], document: Document, concurrency: int = 10, context: IngestContext) -> tuple[list[DocumentChunk], Document]` | Enrich chunks; returns enriched chunks and (possibly modified) document |

### Code Example

```python
import os
from mistralai.client import Mistral
from mistralai.search.toolkit.ingestion.enrichment import SummaryEnricher, SummaryConfig, SummarizeRequestConfig
from mistralai.search.toolkit.llm import MistralChat, LLMConfig

mistral_client = Mistral(api_key=os.environ["MISTRAL_API_KEY"])
llm = MistralChat(client=mistral_client, config=LLMConfig(model="mistral-small-latest"))

enricher = SummaryEnricher(
    llm_provider=llm,
    summary_config=SummaryConfig(
        request_config=SummarizeRequestConfig(
            prompt="Summarize this document in 3 sentences.",
            max_tokens=256,
        )
    ),
)
```

### Custom Enrichers

Implement the `ChunkEnricher` interface with `enrich_chunks()`. Update metadata immutably: `chunk.model_copy(update={"metadata": chunk.metadata.model_copy(update={...})})`.

---

## 10. Search Toolkit — Ingestion: Embedders

**Module:** `mistralai.search.toolkit.embedders`

### Main Concepts

Embedders convert text into vector embeddings for semantic search. All embedders implement the `Embedder` abstract base class. Only `embed()` is abstract; `embed_chunks` and `embed_query` are concrete methods built on top.

### Built-in Implementations

| Class | Purpose |
|---|---|
| `MistralEmbedder` | Use Mistral's embedding API for vectorizing text |

### `MistralEmbedder` Constructor Parameters

| Parameter | Type | Default | Description |
|---|---|---|---|
| `client` | `Mistral` | _(required)_ | Mistral API client |
| `model_name` | `str` | `MODEL_128_EMBEDDING` | Embedding model to use (use model constants) |

### Embedding Model Constants

| Constant | Model Name | Dimensions | Best for |
|---|---|---|---|
| `MODEL_1024_EMBEDDING` | `"mistral-embed"` | 1024 | Default for Search Toolkit pipelines (matches Vespa `embedding_dimensions=1024`) |
| `MODEL_256_EMBEDDING` | `"mistral-embed-dim256-2510"` | 256 | Low-latency, memory-constrained environments |
| `MODEL_128_EMBEDDING` | `"mistral-embed-dim128-2510"` | 128 | Minimum dimensions (default fallback) |

> **Important:** Always match `embedding_dimensions` in your Vespa schema to the model's output dimensions.

### Key Methods

| Method | Signature | Description |
|---|---|---|
| `embed` | `async embed(self, texts: list[str], context: RetrievalContext) -> EmbeddingResult` | Abstract: embed a batch of strings |
| `embed_chunks` | `embed_chunks(self, chunks: list[DocumentChunk]) -> list[DocumentChunk]` | Embed chunks; return copies with embeddings set |
| `embed_query` | `embed_query(self, text: str) -> list[float]` | Embed a single string; return the vector |

**`EmbeddingResult`** (Pydantic BaseModel): `embeddings: list[list[float]]`, `total_tokens: int`.

### Code Example

```python
from mistralai.client import Mistral
from mistralai.search.toolkit.embedders import MistralEmbedder, MODEL_1024_EMBEDDING

client = Mistral(api_key="your-api-key")
embedder = MistralEmbedder(client=client, model_name=MODEL_1024_EMBEDDING)
embedded_chunks = await embedder.embed_chunks(chunks)
```

### Custom Embedders

Subclass `Embedder` and implement `async embed(self, texts, context) -> EmbeddingResult`. `embed_chunks` and `embed_query` are inherited automatically.

---

## 11. Search Toolkit — Pipeline Orchestration

**Module:** `mistralai.search.toolkit.ingestion.pipelines`

### `Pipeline` Class

The main ingestion orchestrator. Handles file loading, extraction, chunking, enrichment, embedding, and indexing.

#### Constructor Parameters

| Parameter | Type | Required | Description |
|---|---|---|---|
| `loader` | `FileLoader` | Yes | File loader (e.g. `FilesystemFileLoader()`) |
| `extractor` | `DocumentExtractor` | Yes | Document extractor (e.g. `PlainTextExtractor()`, `MistralOCRExtractor(client=...)`) |
| `text_splitter` | `TextSplitter` | Yes | Text splitter (e.g. `CharacterTextSplitter(chunk_size=512)`) |
| `chunk_enrichers` | `list[ChunkEnricher]` | No | Enrichers applied sequentially |
| `embedder` | `Embedder` | Yes | Embedder (e.g. `MistralEmbedder(client=...)`) |
| `stores` | vector store | Yes | Vector store (from `app.get_search_index(...)`) |
| `checkpoint_dir` | `str` | No | Directory for checkpoint files |

#### `run` Method

```python
num_chunks = await pipeline.run(
    documents=[Path("doc1.txt"), Path("doc2.txt")],  # list[str] or list[Path]
    use_checkpoint=True,                              # optional, bool
    progress_callback=create_tqdm_progress_callback(), # optional
)
```

Returns `num_chunks` (int — number of chunks indexed).

#### Pipeline Stages

1. `FilesystemFileLoader` reads raw file bytes
2. `DocumentExtractor` extracts text content
3. `TextSplitter` breaks text into chunks
4. `ChunkEnricher`(s) add metadata (optional)
5. `Embedder` generates vector embeddings
6. Vector store indexes each chunk

### Checkpointing

Enable by passing `checkpoint_dir="./checkpoints"` to `Pipeline` and `use_checkpoint=True` to `run()`. On restart, documents with existing checkpoint files are skipped. Progress callback: `mistralai.search.toolkit.ingestion.progress.create_tqdm_progress_callback()`.

### `RoutedPipeline` — Multi-Format Routing

**Module:** `mistralai.search.toolkit.ingestion.pipelines.RoutedPipeline`

Routes files to protocol-specific pipelines based on file extension and MIME type.

| Constructor parameter | Description |
|---|---|
| `pipelines` | Dictionary mapping protocol names to `Pipeline` instances |
| `mime_registry` | Optional custom MIME registry for file-type detection |
| `protocol_overrides` | Optional dictionary to override default protocol for specific extensions |

Methods: `await router.run_file(file=file)` — processes a single file, returns a document.

#### Built-in Protocols

| Protocol | File types |
|---|---|
| `ocr` | PDF, DOCX, PPTX, ODT, EPUB |
| `plain_text` | TXT, MD, CSV, JS, PY, JSON, YAML, ... |
| `html` | HTML, HTM |
| `xlsx` | XLSX, XLS, ODS |
| `numbers` | NUMBERS |
| `legacy_office` | DOC, PPT, HWP, HWPX |
| `email` | EML, MSG |
| `image` | PNG, JPEG, GIF, WebP |
| `audio` | MP3, WAV, M4A, FLAC, OGG |

```python
router = RoutedPipeline(
    pipelines={
        "ocr": ocr_pipeline,
        "html": html_pipeline,
        "plain_text": text_pipeline,
        "xlsx": spreadsheet_pipeline,
    },
    protocol_overrides={".doc": "ocr"},  # Force legacy docs through OCR
)
```

### Full Ingestion Example

```python
import asyncio
from pathlib import Path
from mistralai.client import Mistral
from mistralai.search.toolkit.embedders import MistralEmbedder
from mistralai.search.toolkit.ingestion.extractors import PlainTextExtractor
from mistralai.search.toolkit.ingestion.loaders import FilesystemFileLoader
from mistralai.search.toolkit.ingestion.pipelines import Pipeline
from mistralai.search.toolkit.ingestion.text_splitters import CharacterTextSplitter
from mistralai.search.toolkit.plugins.vespa import VespaClientConfig
from vespa_app import app

async def main():
    mistral_client = Mistral(api_key="your-api-key")
    config = VespaClientConfig(endpoint="http://localhost:8080")
    vector_store = app.get_search_index(config, collection_name="quickstart_collection")

    pipeline = Pipeline(
        loader=FilesystemFileLoader(),
        extractor=PlainTextExtractor(),
        text_splitter=CharacterTextSplitter(chunk_size=500),
        embedder=MistralEmbedder(client=mistral_client, model_name="mistral-embed"),
        stores=vector_store,
    )
    await pipeline.run(documents=[Path("doc1.txt"), Path("doc2.txt")])

asyncio.run(main())
```

---

## 12. Search Toolkit — Retrieval: Query Preprocessing

**Module:** `mistralai.search.toolkit.retrieval.pre_processors`

### Main Concepts

Query preprocessing improves user queries before retrieval. Rewriting and expansion improve retrieval quality at the cost of increased latency and cost. Preprocessors are wired into `QueryEngine` via the `query_rewriter` parameter.

### Built-in Implementations

| Class | Purpose |
|---|---|
| `LLMQueryRewriter` | Reformulates queries into forms more likely to match indexed content (converts informal language, expands abbreviations, clarifies intent) |
| `LLMQueryExtension` | Breaks a query into multiple sub-queries exploring different facets; each runs an independent retrieval pass, results are combined |

### Constructor Parameters

#### `LLMQueryRewriter`

| Parameter | Type | Default | Description |
|---|---|---|---|
| `llm_provider` | `LLMProvider` | _(required)_ | LLM for rewriting |
| `prompt` | `str` | _(built-in default)_ | Custom rewriting prompt prefix |

#### `LLMQueryExtension`

| Parameter | Type | Default | Description |
|---|---|---|---|
| `llm_provider` | `LLMProvider` | _(required)_ | LLM for extension |
| `num_queries` | `int` | `3` | Number of sub-queries to generate |

Cost note: `num_queries=3` = 1 LLM call to generate sub-queries + 3 retrieval passes (3x slower, 3x embedding calls) + results combined/deduplicated. Lower = faster/lower recall; higher = slower/higher recall.

### Key Methods

| Method | Signature | Description |
|---|---|---|
| `rewrite` | `async rewrite(self, query: str) -> str` | Rewrite a single query (LLMQueryRewriter) |
| `extend` | `async extend(self, query: str) -> list[str]` | Generate sub-queries (LLMQueryExtension) |
| `preprocess` | `async preprocess(self, query: str) -> str` | Protocol method (QueryPreprocessor) |

### Code Example

```python
from mistralai.search.toolkit.retrieval.pre_processors import LLMQueryRewriter
from mistralai.search.toolkit.llm import MistralChat, LLMConfig
from mistralai.client import Mistral

llm = MistralChat(client=Mistral(api_key="your-api-key"),
                  config=LLMConfig(model="mistral-small-latest"))
rewriter = LLMQueryRewriter(llm_provider=llm)
rewritten = await rewriter.rewrite("rag mistral")
# -> "What is Retrieval-Augmented Generation with Mistral AI?"
```

### Combined with Reranking (Best Practice)

```python
query_engine = QueryEngine(
    retriever=vector_retriever,
    query_rewriter=LLMQueryExtension(llm_provider=llm, num_queries=3),
    rerankers=[LLMReRanker(llm_provider=llm, top_k=10)],
)
# Flow: 1 query → 3 sub-queries → 3 retrieval passes → ~30 results → LLM reranking → top 10
```

---

## 13. Search Toolkit — Retrieval: Retrievers

**Module:** `mistralai.search.toolkit.retrieval.retrievers`

### Main Concepts

Retrievers execute the search against your index. The built-in `VectorRetriever` performs semantic search using embeddings — finding chunks with similar meaning to the query, even when different words are used. Handles synonyms and paraphrasing naturally.

### Built-in Implementations

| Class | Purpose |
|---|---|
| `VectorRetriever` | Semantic search using embeddings |

### `VectorRetriever` Constructor Parameters

| Parameter | Type | Default | Description |
|---|---|---|---|
| `client` | `VectorStoreIndex` | _(required)_ | Vector store to search (Vespa or custom) |
| `embedder` | `Embedder` | _(required)_ | Embedder for query vectorization |

### `retrieve` Method Parameters

| Parameter | Type | Default | Description |
|---|---|---|---|
| `query` | `str` | _(required)_ | The search query |
| `top_k` | `int` | `10` | Number of results |
| `filter` | `dict` | — | Optional metadata filter (e.g. `{"category": "tutorials", "year": 2024}`); support depends on vector store |
| `include_metadata` | `bool` | `True` | Include chunk metadata in results |
| `include_content` | `bool` | `True` | Include chunk content in results |
| `context` | `RetrievalContext` | `RetrievalContext()` | Retrieval context |

Returns `list[SearchResult]` — each result has `score` (float) and `chunk` (with `content`, metadata, identity fields).

### Code Example

```python
from mistralai.search.toolkit.retrieval.retrievers import VectorRetriever

retriever = VectorRetriever(client=vector_store, embedder=embedder)
results = await retriever.retrieve(query="What is semantic search?", top_k=10)

# With metadata filtering
results = await retriever.retrieve(
    query="machine learning",
    top_k=10,
    filter={"category": "tutorials", "year": 2024},
)
```

### Custom Retrievers

Implement the `Retriever` protocol with `async retrieve(self, query, top_k=10, include_metadata=True, include_content=True, context=RetrievalContext()) -> list[SearchResult]`.

---

## 14. Search Toolkit — Retrieval: Rerankers

**Module:** `mistralai.search.toolkit.retrieval.rerankers`

### Main Concepts

Rerankers apply more sophisticated scoring to improve ranking quality after initial (fast but approximate) retrieval. Multiple rerankers can be chained in `QueryEngine.rerankers` and are applied sequentially. Two protocol categories:

- **`ReRanker`** — for single-list reranking
- **`GroupedRanker`** — for multi-group fusion (fusing results from multiple retrievers)

### Built-in Implementations

| Class | Category | Purpose |
|---|---|---|
| `LLMReRanker` | ReRanker | Deep relevance scoring using an LLM (1 LLM call per chunk) |
| `CrossEncoderReRanker` | ReRanker | Fast reranking using a dedicated cross-encoder model |
| `RRFRanker` | GroupedRanker | Reciprocal Rank Fusion — fuse results from multiple retrievers |

### Constructor Parameters

#### `LLMReRanker`

| Parameter | Type | Default | Description |
|---|---|---|---|
| `llm_provider` | `LLMProvider` | _(required)_ | LLM for scoring |
| `top_k` | `int` | `10` | Results to return after reranking |
| `batch_size` | `int` | `10` | Batch size for LLM scoring |

> Cost optimization: Have the retriever return many results (e.g. top 100) cheaply, then let the reranker narrow to `top_k` (e.g. 10).

#### `CrossEncoderReRanker`

| Parameter | Type | Default | Description |
|---|---|---|---|
| `base_url` | `str` | _(required)_ | Cross-encoder service URL |
| `model` | `str` | _(required)_ | Model identifier (e.g. `cross-encoder/ms-marco-MiniLM-L-6-v2`) |
| `top_k` | `int` | `10` | Results to return |
| `timeout` | `int` | `30` | Request timeout in seconds |

Popular models: `cross-encoder/ms-marco-MiniLM-L-6-v2`, `cross-encoder/ms-marco-TinyBERT-L-2-v2`, `cross-encoder/qnli-distilroberta-base`.

#### `RRFRanker`

| Parameter | Type | Default | Description |
|---|---|---|---|
| `rrf_k` | `int` | `60` | Smoothing factor (recommended range 30–100) |
| `top_k` | `int` | `10` | Results to return |

Tuning: Lower `rrf_k` (e.g. 30) → more emphasis on rank positions; Higher `rrf_k` (e.g. 100) → flattens differences.

### Key Methods (Protocols)

| Method | Signature | Description |
|---|---|---|
| `rerank` | `async rerank(self, query: str, search_results: list[SearchResult]) -> list[SearchResult]` | ReRanker protocol |
| `rerank_groups` | `async rerank_groups(self, query: str, result_groups: list[list[SearchResult]]) -> list[SearchResult]` | GroupedRanker protocol |

### Code Example — Chaining Multiple Rerankers

```python
from mistralai.search.toolkit.retrieval import QueryEngine
from mistralai.search.toolkit.retrieval.rerankers import RRFRanker, LLMReRanker

query_engine = QueryEngine(
    retriever=[dense_retriever, hybrid_retriever],
    rerankers=[
        RRFRanker(rrf_k=60, top_k=50),              # Step 1: Fuse → top 50
        MetadataReRanker(),                          # Step 2: Boost verified content
        LLMReRanker(llm_provider=llm, top_k=10),    # Step 3: Final LLM reranking → top 10
    ],
)
result = await query_engine.search(query="...", top_k=10)
```

### Custom Rerankers

Implement `ReRanker` (single-list) or `GroupedRanker` (multi-group fusion). Example: metadata-based boosting that multiplies scores based on `verified` flag and publication year.

---

## 15. Search Toolkit — Retrieval: Semantic Cache

**Module:** `mistralai.search.toolkit.retrieval.cache`

### Main Concepts

The semantic cache matches queries by **meaning** rather than exact string matching. When an incoming query is semantically similar to a previously cached one, the cache returns stored results and skips embedding, retrieval, preprocessing, and reranking entirely. All cache operations are non-fatal/best-effort — if the cache or embedder throws an exception, the query falls through to the normal uncached pipeline.

### How It Works

1. Query comes in
2. Embedder converts query to vector
3. Cache searches for similar cached query vectors
4. If similarity > threshold → return cached results (fast)
5. If no match → run full retrieval pipeline, cache results for future (slow)

### Built-in Classes

| Class | Role |
|---|---|
| `CachedQueryEngine` | Wraps a `QueryEngine` to add transparent caching; same `search()` interface |
| `SemanticCache` | The cache itself; holds a backend and a similarity threshold |
| `InMemoryCacheBackend` | Built-in in-memory vector cache backend |
| `CacheBackend` | ABC for custom backends (Redis, pgvector, etc.) |
| `CacheEntry` | Entry object used by custom backends |
| `CacheMetrics` | Metrics tracker for cache performance |
| `EvictionPolicy` | Enum: `LRU`, `LFU`, `FIFO` |

### Constructor Parameters

#### `InMemoryCacheBackend`

| Parameter | Type | Default | Description |
|---|---|---|---|
| `dim` | `int` | _(required)_ | Embedding dimensionality (1024 for mistral-embed) |
| `max_entries` | `int` | `1000` | Max cached queries before eviction |
| `ttl_seconds` | `int \| None` | `None` | Entry expiration in seconds (`None` = no expiry) |
| `eviction_policy` | `EvictionPolicy` | `EvictionPolicy.LRU` | Eviction policy: LRU, LFU, or FIFO |

Memory note: 1024-dim embeddings ≈ 4KB per entry (1000 entries ≈ 4MB).

#### `SemanticCache`

| Parameter | Type | Default | Description |
|---|---|---|---|
| `backend` | `CacheBackend` | _(required)_ | The cache backend |
| `similarity_threshold` | `float` | `0.95` | Cosine similarity required for a cache hit |

Threshold guidance: `0.99` = very strict (exact matching); `0.95` = balanced (default); `0.90` = permissive (approximate answers).

#### `CachedQueryEngine`

| Parameter | Type | Default | Description |
|---|---|---|---|
| `engine` | `QueryEngine` | _(required)_ | The wrapped query engine |
| `cache` | `SemanticCache` | _(required)_ | The semantic cache |
| `embedder` | `Embedder` | _(required)_ | Used to embed incoming queries |
| `metrics` | `CacheMetrics` | — | Optional metrics tracker |

### Key Methods

| Method | Signature | Description |
|---|---|---|
| `search` | `async search(self, query: str, top_k: int = 10)` | Transparent caching — same interface as `QueryEngine.search` |
| `invalidate` | `async invalidate(self, namespace: str)` | Remove all entries in a namespace |
| `clear` | `async clear(self)` | Remove all cached entries |

### `CacheMetrics.snapshot()` Attributes

| Metric | Description |
|---|---|
| `hit_rate` | Fraction of queries served from cache |
| `avg_hit_similarity` | Average cosine similarity on cache hits |
| `total_requests` | Total queries processed |
| `avg_embed_time_ms` | Average embedding time per query |
| `avg_lookup_time_ms` | Average cache lookup time per query |
| `avg_retrieval_time_ms` | Average retrieval time when cache misses |
| `evictions` | Total entries evicted due to capacity |
| `errors` | Total cache/embedder errors |

### Code Example

```python
from mistralai.search.toolkit.retrieval.cache import (
    CachedQueryEngine, InMemoryCacheBackend, SemanticCache, EvictionPolicy, CacheMetrics,
)

backend = InMemoryCacheBackend(dim=1024, max_entries=500, ttl_seconds=3600,
                               eviction_policy=EvictionPolicy.LRU)
cache = SemanticCache(backend=backend, similarity_threshold=0.95)
metrics = CacheMetrics()

cached_engine = CachedQueryEngine(
    engine=query_engine, cache=cache, embedder=embedder, metrics=metrics,
)

result = await cached_engine.search("What is RAG?", top_k=10)

snapshot = metrics.snapshot()
print(f"Hit rate: {snapshot.hit_rate:.1%}")
print(f"Avg hit similarity: {snapshot.avg_hit_similarity:.3f}")
```

### Custom Cache Backends

Implement the `CacheBackend` ABC: `max_entries` (property), `search`, `store`, `delete`, `get_all`, `count`, `clear`, `update_hit`.

---

## 16. Search Toolkit — QueryEngine Orchestration

**Module:** `mistralai.search.toolkit.retrieval.QueryEngine`

### Constructor Parameters

| Parameter | Type | Required | Description |
|---|---|---|---|
| `retriever` | `Retriever` or `list[Retriever]` | Yes | Single retriever or a list (results merged/fused) |
| `query_rewriter` | `QueryPreprocessor` | No | Query preprocessor (e.g. `LLMQueryRewriter`, `LLMQueryExtension`) |
| `rerankers` | `list[ReRanker \| GroupedRanker]` | No | Rerankers applied sequentially |

### `search` Method

```python
result = await query_engine.search(
    query="What is RAG?",
    top_k=10,
    include_metadata=True,
    include_content=True,
)
```

### Result Object

| Field | Type | Description |
|---|---|---|
| `result.original_query` | `str` | The original query |
| `result.results` | `list` | Ranked list; each item has `.score` (float) and `.chunk.content` (str) |

### Full Retrieval Example

```python
import asyncio
from mistralai.client import Mistral
from mistralai.search.toolkit.embedders import MistralEmbedder
from mistralai.search.toolkit.plugins.vespa import VespaClientConfig
from mistralai.search.toolkit.retrieval import QueryEngine
from mistralai.search.toolkit.retrieval.retrievers import VectorRetriever
from vespa_app import app

async def main():
    mistral_client = Mistral(api_key="your-api-key")
    embedder = MistralEmbedder(client=mistral_client, model_name="mistral-embed")

    config = VespaClientConfig(endpoint="http://localhost:8080")
    vector_store = app.get_search_index(config, collection_name="quickstart_collection")

    query_engine = QueryEngine(
        retriever=[VectorRetriever(client=vector_store, embedder=embedder)],
    )

    result = await query_engine.search(
        query="What is RAG?",
        top_k=5,
        include_metadata=True,
        include_content=True,
    )

    for i, r in enumerate(result.results, 1):
        print(f"{i}. [Score: {r.score:.3f}] {r.chunk.content[:200]}...")

asyncio.run(main())
```

---

## 17. Search Toolkit — Vespa Integration

**Module:** `mistralai.search.toolkit.plugins.vespa`

### Main Concepts

Vespa is the default vector store backend. The toolkit provides a plugin for schema management, deployment, and search index access.

### Key Classes

| Class | Purpose |
|---|---|
| `VespaClientConfig` | Connection configuration |
| `VespaMigration` | Base class for schema migrations |
| `create_schema(...)` | Create a schema/collection |
| `set_app_name(...)` | Set the Vespa application name (lowercase `a-z` only) |
| `FieldDefinition` | Field definition (e.g. `TextField`, with `name`) |
| `IndexingMode` | Indexing mode (e.g. `DOCUMENT_PER_CHUNK`) |
| `SearchMode` | Search mode (e.g. `INDEX`) |

### `VespaClientConfig`

| Parameter | Type | Default | Description |
|---|---|---|---|
| `endpoint` | `str` | — | Vespa endpoint URL (e.g. `"http://localhost:8080"`) |

### Schema Migration Example

```python
from mistralai.search.toolkit.plugins.vespa.app.schemas.app import (
    FieldDefinition, IndexingMode, SearchMode,
)
from mistralai.search.toolkit.plugins.vespa.migration import (
    VespaMigration, create_schema, set_app_name,
)

class InitialSchema(VespaMigration):
    def migrate(self) -> None:
        set_app_name("myquickstart")  # lowercase a-z only
        create_schema(
            name="quickstart_collection",
            mode=SearchMode.INDEX,
            embedding_dimensions=1024,
            indexing_mode=IndexingMode.DOCUMENT_PER_CHUNK,
            fields=[
                FieldDefinition.TextField(name="title"),
            ],
        )
```

> Standard chunk fields (content, embedding, identity, metadata) are added automatically. The `fields` list only declares your extra fields.

### CLI Commands

| Command | Description |
|---|---|
| `mistral-vespa generate-migration --app-dir ./vespa/migrations initial_schema` | Generate a migration file |
| `mistral-vespa migrate --app-dir ./vespa/migrations --config-server http://localhost:19071 --query-port 8080` | Deploy migrations to Vespa |

### Local Vespa Setup (Docker)

```bash
docker run --detach --name vespa --hostname vespa-container \
  --publish 8080:8080 --publish 19071:19071 vespaengine/vespa

# Wait for health
curl --retry 10 --retry-delay 3 --retry-all-errors \
  http://localhost:19071/state/v1/health
```

### Accessing the Search Index

```python
config = VespaClientConfig(endpoint="http://localhost:8080")
vector_store = app.get_search_index(config, collection_name="quickstart_collection")
# Pass vector_store as `stores=` to Pipeline (writes) and as `client=` to VectorRetriever (reads)
```

---

## 18. Cross-Cutting: Installation, Extras & Component Summary

### Installation

```bash
# Core package
uv add mistralai-search-toolkit

# With Vespa (recommended for full RAG)
uv add "mistralai-search-toolkit[vespa]"

# With specific extras
uv add "mistralai-search-toolkit[vespa,extractor-pymupdf]"

# All extras
uv add "mistralai-search-toolkit[all]"
```

### Optional Extras

| Extra | Description |
|---|---|
| `vespa` | Vespa plugin for vector storage and semantic search |
| `extractor-pymupdf` | Advanced PDF extraction with PyMuPDF Pro |
| `extractor-spreadsheet` | Spreadsheet parsing (Excel, CSV, Calamine format) |
| `extractor-email` | Email file parsing (EML, MSG formats) |
| `html-converter-markdownify` | Convert HTML to Markdown |
| `text-splitter-langchain` | Additional text splitting strategies via LangChain |
| `storage-gcs` | Google Cloud Storage integration |
| `storage-azure` | Azure Blob Storage integration |
| `storage-s3` | AWS S3 storage integration |
| `all` | All optional extras |

### Component Summary Table

| Component | Built-in options | Extension point |
|---|---|---|
| **File loaders** | `FilesystemFileLoader`, `S3FileLoader`, `AzureBlobFileLoader`, `GCSFileLoader` | `FileLoader` protocol |
| **Extractors** | `MistralOCRExtractor`, `MistralAudioTranscriptionExtractor`, `PlainTextExtractor`, `HTMLExtractor`, `SpreadsheetExtractor`, `EmailExtractor`, `NumbersExtractor`, `LegacyOfficeExtractor` | `DocumentExtractor` protocol |
| **Text splitters** | `CharacterTextSplitter`, `TokenTextSplitter`, `MarkdownTextSplitter`, `SeparatorTextSplitter` | `TextSplitter` protocol |
| **Enrichers** | `SummaryEnricher` | `ChunkEnricher` interface |
| **Embedders** | `MistralEmbedder` (1024/256/128 dim) | `Embedder` abstract base class |
| **Storage** | Vespa (via plugin) | Custom vector store |
| **Retrievers** | `VectorRetriever` | `Retriever` protocol |
| **Rerankers** | `LLMReRanker`, `CrossEncoderReRanker`, `RRFRanker` | `ReRanker` / `GroupedRanker` protocols |
| **Preprocessing** | `LLMQueryRewriter`, `LLMQueryExtension` | `QueryPreprocessor` protocol |
| **Caching** | `SemanticCache` with `InMemoryCacheBackend` | `CacheBackend` ABC |
| **Orchestration** | `Pipeline`, `RoutedPipeline` (ingestion); `QueryEngine`, `CachedQueryEngine` (retrieval) | — |

### Shared LLM Provider Interface

The `LLMProvider` interface (e.g. `MistralChat` from `mistralai.search.toolkit.llm`, configured with `LLMConfig`) is shared by `LLMReRanker`, `LLMQueryRewriter`, `LLMQueryExtension`, and `SummaryEnricher`.

```python
from mistralai.search.toolkit.llm import MistralChat, LLMConfig
from mistralai.client import Mistral

llm = MistralChat(
    client=Mistral(api_key="your-api-key"),
    config=LLMConfig(model="mistral-small-latest", temperature=0.1,
                     response_format={"type": "json_object"}),
)
```

---

## 19. Capability Summary & Cross-Reference

### Document AI vs. Search Toolkit

| Dimension | Document AI (Cloud API) | Search Toolkit (Python Framework) |
|---|---|---|
| **Access** | HTTP API / SDK calls | Python library with swappable components |
| **Infrastructure** | Fully managed | Self-hosted (Vespa via Docker or cloud) |
| **OCR** | `client.ocr.process()` | `MistralOCRExtractor` (wraps the same API) |
| **Structured extraction** | Annotations (BBox + document-level) | `SummaryEnricher` or custom `ChunkEnricher` |
| **Document QnA** | Chat completions with `document_url` | Not directly provided (build via QueryEngine + LLM) |
| **Chunking** | Not provided (returns full pages) | `CharacterTextSplitter`, `TokenTextSplitter`, `MarkdownTextSplitter`, `SeparatorTextSplitter` |
| **Embedding** | Not provided | `MistralEmbedder` (1024/256/128 dim) |
| **Vector storage** | Not provided | Vespa integration |
| **Retrieval** | Not provided | `VectorRetriever` with metadata filtering |
| **Reranking** | Not provided | `LLMReRanker`, `CrossEncoderReRanker`, `RRFRanker` |
| **Query preprocessing** | Not provided | `LLMQueryRewriter`, `LLMQueryExtension` |
| **Caching** | Not provided | `SemanticCache` with `InMemoryCacheBackend` |
| **Scaling** | Batch Inference service | Pipeline checkpointing + concurrent batch loading |

### Capability → API Function Map

| Capability | API function / class | Key parameters |
|---|---|---|
| OCR (PDF/image) | `client.ocr.process()` | `model`, `document`, `table_format`, `include_blocks`, `confidence_scores_granularity` |
| BBox annotations | `client.ocr.process(bbox_annotation_format=...)` | `bbox_annotation_format` (Pydantic/Zod/JSON schema), `include_image_base64` |
| Document annotations | `client.ocr.process(document_annotation_format=...)` | `document_annotation_format` (schema) |
| Document QnA | `client.chat.complete()` | `model`, `messages` (with `document_url` content block) |
| File loading | `FilesystemFileLoader` / `S3FileLoader` / etc. | `root`/`bucket_name`, `max_file_size` |
| Document extraction | `MistralOCRExtractor` / `PlainTextExtractor` / etc. | `client`, `model_name`, `include_image_base64` |
| Text splitting | `CharacterTextSplitter` / `TokenTextSplitter` / etc. | `chunk_size`, `chunk_overlap` |
| Chunk enrichment | `SummaryEnricher` | `llm_provider`, `summary_config` |
| Embedding | `MistralEmbedder` | `client`, `model_name` (1024/256/128 dim) |
| Ingestion pipeline | `Pipeline.run()` | `documents`, `use_checkpoint`, `progress_callback` |
| Multi-format routing | `RoutedPipeline.run_file()` | `pipelines`, `protocol_overrides` |
| Query rewriting | `LLMQueryRewriter.rewrite()` | `llm_provider`, `prompt` |
| Query extension | `LLMQueryExtension.extend()` | `llm_provider`, `num_queries` |
| Vector retrieval | `VectorRetriever.retrieve()` | `query`, `top_k`, `filter` |
| LLM reranking | `LLMReRanker.rerank()` | `llm_provider`, `top_k`, `batch_size` |
| Cross-encoder reranking | `CrossEncoderReRanker.rerank()` | `base_url`, `model`, `top_k`, `timeout` |
| Rank fusion | `RRFRanker.rerank_groups()` | `rrf_k`, `top_k` |
| Semantic caching | `CachedQueryEngine.search()` | `cache`, `embedder`, `similarity_threshold` |
| Search orchestration | `QueryEngine.search()` | `query`, `top_k`, `include_metadata`, `include_content` |
| Schema management | `create_schema()` / `VespaMigration` | `name`, `mode`, `embedding_dimensions`, `indexing_mode`, `fields` |

### Core Object Relationships

```
Mistral API Key
  │
  ├── Document AI (Cloud API)
  │     ├── client.ocr.process() ──── OCR + Annotations
  │     └── client.chat.complete() ── Document QnA (with document_url block)
  │
  └── Search Toolkit (Python Framework)
        │
        ├── Pipeline (ingestion)
        │     ├── FileLoader ─────────── FilesystemFileLoader / S3 / Azure / GCS
        │     ├── DocumentExtractor ──── MistralOCRExtractor / PlainText / HTML / Spreadsheet / Email / ...
        │     ├── TextSplitter ───────── Character / Token / Markdown / Separator
        │     ├── ChunkEnricher ──────── SummaryEnricher (optional, sequential)
        │     ├── Embedder ───────────── MistralEmbedder (1024 / 256 / 128 dim)
        │     └── stores ─────────────── Vector Store (Vespa)
        │                                    │
        │                                    ▼
        ├── Vector Store (Vespa) ◀──── shared between Pipeline & QueryEngine
        │     ├── collection_name ──── schema with embedding_dimensions + custom fields
        │     └── app.get_search_index(config, collection_name=...)
        │                                    │
        │                                    ▼
        └── QueryEngine (retrieval)
              ├── query_rewriter ──── LLMQueryRewriter / LLMQueryExtension (optional)
              ├── retriever(s) ────── VectorRetriever (single or list)
              └── rerankers ───────── LLMReRanker / CrossEncoderReRanker / RRFRanker (sequential, optional)
                    │
                    ▼
              CachedQueryEngine (optional wrapper)
                ├── SemanticCache ──── InMemoryCacheBackend / custom CacheBackend
                ├── embedder ───────── MistralEmbedder
                └── metrics ────────── CacheMetrics

Data Model:
  Document (id = UUID5(source_id))
    └── DocumentChunk (id = UUID5(source_id + locator), parent_ref = Document.id)
          ├── chunk_type: content / image_annotation / summary
          ├── embedding: list[float] (populated by Embedder)
          └── metadata: immutable, extensible (filename, filepath, page_number, ...)
```

### Key Design Principles

1. **Two-tier offering** — Document AI for simple API calls (OCR, annotations, QnA); Search Toolkit for full RAG pipeline control with swappable components.
2. **Deterministic identity** — UUID5 IDs from `source_id` + `locator` make ingestion idempotent; re-ingesting the same source overwrites, not duplicates.
3. **Dependency injection** — Both `Pipeline` and `QueryEngine` accept their sub-components as constructor arguments; every component can be swapped for a custom implementation.
4. **Sequential enrichment & reranking** — Multiple enrichers and rerankers chain in order, each receiving the previous one's output, enabling progressive refinement.
5. **Transparent caching** — `CachedQueryEngine` wraps `QueryEngine` with the same `search()` interface; cache misses fall through silently to the full pipeline.
6. **Cost-aware retrieval** — Retrievers return many results cheaply; rerankers (especially `LLMReRanker`) narrow to `top_k`; semantic cache avoids expensive reranking on hits.
7. **Multilingual + multimodal** — OCR supports many languages; extractors handle PDFs, images, HTML, spreadsheets, email, audio, and legacy office formats.

---

*Sources: [Document Processing Overview](https://docs.mistral.ai/studio-api/document-processing/overview), [OCR Processor](https://docs.mistral.ai/studio-api/document-processing/basic_ocr), [Document Annotations](https://docs.mistral.ai/studio-api/document-processing/annotations), [Document QnA](https://docs.mistral.ai/studio-api/document-processing/document_qna), [Search Toolkit](https://docs.mistral.ai/studio-api/search-toolkit), [Ingestion](https://docs.mistral.ai/studio-api/search-toolkit/ingestion), [Retrieval](https://docs.mistral.ai/studio-api/search-toolkit/retrieval), [Quickstart](https://docs.mistral.ai/studio-api/search-toolkit/quickstart), [Document Model](https://docs.mistral.ai/studio-api/search-toolkit/document-model), [Loaders](https://docs.mistral.ai/studio-api/search-toolkit/ingestion/loaders), [Extractors](https://docs.mistral.ai/studio-api/search-toolkit/ingestion/extractors), [Splitters](https://docs.mistral.ai/studio-api/search-toolkit/ingestion/splitters), [Enrichers](https://docs.mistral.ai/studio-api/search-toolkit/ingestion/enrichers), [Embedders](https://docs.mistral.ai/studio-api/search-toolkit/ingestion/embedders), [Retrievers](https://docs.mistral.ai/studio-api/search-toolkit/retrieval/retrievers), [Rerankers](https://docs.mistral.ai/studio-api/search-toolkit/retrieval/rerankers), [Preprocessing](https://docs.mistral.ai/studio-api/search-toolkit/retrieval/preprocessing), [Semantic Cache](https://docs.mistral.ai/studio-api/search-toolkit/retrieval/semantic-cache). Last reviewed: July 2026.*