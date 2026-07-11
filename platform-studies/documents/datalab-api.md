# Datalab API Analysis — Document Intelligence Platform

> **Base URL:** `https://www.datalab.to` | **API version:** v1 | **Auth:** `X-API-Key` header
> **Description:** Convert PDFs, images, spreadsheets, and Office documents into structured, machine-readable outputs. Chain processors into versioned pipelines. Extract structured data with citations. Fill forms, segment documents, and generate DOCX files — all with auditability.

---

## Table of Contents

1. [Platform Overview & Architecture](#1-platform-overview--architecture)
2. [Authentication & Request Pattern](#2-authentication--request-pattern)
3. [Document Conversion](#3-document-conversion)
4. [Structured Extraction](#4-structured-extraction)
5. [Document Segmentation](#5-documentation-segmentation)
6. [Form Filling](#6-form-filling)
7. [Track Changes Extraction](#7-track-changes-extraction)
8. [Create Document (DOCX Generation)](#8-create-document-docx-generation)
9. [Custom Processors](#9-custom-processors)
10. [Pipelines](#10-pipelines)
11. [Extraction Schemas](#11-extraction-schemas)
12. [Extraction Schema Generation](#12-extraction-schema-generation)
13. [File Management](#13-file-management)
14. [Collections & Batch Evaluation](#14-collections--batch-evaluation)
15. [Eval Rubrics](#15-eval-rubrics)
16. [Forge Evals](#16-forge-evals)
17. [Thumbnails](#17-thumbnails)
18. [Workflows (Temporal)](#18-workflows-temporal)
19. [Health & Compliance](#19-health--compliance)
20. [Cross-Cutting Concerns](#20-cross-cutting-concerns)

---

## 1. Platform Overview & Architecture

### Main Concepts

- **Document Intelligence Platform** — Datalab provides APIs to convert PDFs, spreadsheets, images, and Office documents into structured, machine-readable outputs (Markdown, HTML, JSON, chunks).
- **Powered by Marker, Surya, and Chandra** — Three underlying models handle document parsing, OCR, and advanced layout understanding.
- **Three Deployment Modes** — Fully managed cloud API, on-prem container for sensitive documents, and open-source tools for local development.
- **Async Processing Pattern** — All processing endpoints follow a submit → poll → retrieve pattern. Submit returns a `request_id` and `request_check_url`; poll the check URL until `status` is `complete`.
- **Checkpoint Optimization** — Converting a document with `save_checkpoint=true` stores parsed state. The resulting `checkpoint_id` can be passed to `/extract` or `/segment` to skip re-parsing, saving time and cost.
- **Data Residency** — Optional `processing_location` parameter (`us` or `eu`) on all inference endpoints for regional processing. EU carries a pricing premium.
- **Result Retention** — Results are deleted from Datalab servers **one hour** after processing completes. Must be retrieved promptly.
- **Webhooks** — Alternative to polling. Configure a default webhook URL in account settings or override per-request with the `webhook_url` parameter.

### Key Capabilities Summary

| Capability | Endpoint | Purpose |
|---|---|---|
| Document Conversion | `POST /api/v1/convert` | Parse documents to Markdown, HTML, JSON, or chunks |
| Structured Extraction | `POST /api/v1/extract` | Extract fields using JSON schemas with citations |
| Document Segmentation | `POST /api/v1/segment` | Split multi-document PDFs into logical sections |
| Form Filling | `POST /api/v1/fill` | Fill PDF/image forms with structured data |
| Track Changes | `POST /api/v1/track-changes` | Extract redlines and comments from DOCX files |
| Create Document | `POST /api/v1/create-document` | Generate DOCX from markdown with track changes |
| Custom Processors | `POST /api/v1/custom-processor` | Run AI-generated custom processing on documents |
| Pipelines | `POST /api/v1/pipelines/{id}/run` | Chain multiple processors into a single execution |
| Thumbnails | `GET /api/v1/thumbnails/{lookup_key}` | Generate page thumbnails |
| File Management | `POST /api/v1/files/upload` | Upload and manage files in Datalab storage |

### API Limits

| Limit | Value |
|---|---|
| Max file size | 200 MB |
| Max pages per request | 7,000 |
| Free tier — requests/min | 10 |
| Free tier — concurrent requests | 5 |
| Team — requests/min | 200 |
| Team — concurrent requests | 400 |
| Concurrent pages in flight | 5,000 (not time-bound, not enforced at submission) |

Rate limit violations return `429`. Page concurrency violations return `success: false` with an error message in the result (not a `429`).

---

## 2. Authentication & Request Pattern

### Main Concepts

- **API Key Authentication** — All endpoints require an `X-API-Key` HTTP header. Keys are generated from the account dashboard.
- **Team Context** — Requests are scoped to a team (optionally via `datalab_active_team` cookie).
- **Async Submit-Poll-Retrieve** — Processing endpoints return immediately with a `request_id` and `request_check_url`. The client polls the check URL until `status` becomes `complete`.

### Submit Response (InitialResponse)

| Field | Type | Description |
|---|---|---|
| `success` | boolean | Whether the request was accepted (default `true`) |
| `error` | string\|null | Error message if not successful |
| `request_id` | string | Unique request ID for status polling (required) |
| `request_check_url` | string | URL to poll for status and results (required) |
| `versions` | object\|null | Library versions used in the request |

### Analysis

The async pattern decouples submission from processing, allowing large documents to be processed without holding HTTP connections open. The SDK handles polling automatically; REST API users must implement polling loops. The `request_check_url` is self-contained — no need to construct URLs manually.

---

## 3. Document Conversion

### Main Concepts

- **Core Capability** — Convert PDFs, Word documents, PowerPoint, spreadsheets, and images (PNG, JPG, WebP) into Markdown, HTML, JSON, or chunks.
- **Processing Modes** — Three modes balancing speed vs. accuracy: `fast` (lowest latency), `balanced` (recommended for most use cases), `accurate` (highest accuracy for complex layouts).
- **Output Formats** — `markdown` (LLM/RAG-friendly), `html` (preserves visual structure, supports block IDs and bboxes), `json` (programmatic block access with bounding boxes), `chunks` (pre-segmented for vector databases).
- **Checkpoints** — `save_checkpoint=true` stores parsed state for later reuse by `/extract` or `/segment`, avoiding re-parsing.
- **Bounding Box Add-ons** — Per-word, per-table-cell, and per-list-item bounding boxes with confidence scores (HTML output only, $0.30/1K pages each).
- **Parse Quality Scoring** — Each conversion returns a `parse_quality_score` (0–5) indicating output quality.
- **Extras** — Feature flags for chart understanding, image infographics, link extraction, track changes, table cell bboxes, list item bboxes, and new block types.

### Endpoints & Parameters

| Endpoint | Method | Purpose |
|---|---|---|
| `/api/v1/convert` | `POST` | Submit a document for conversion |
| `/api/v1/convert/{request_id}` | `GET` | Poll for status and retrieve converted output |

### Request Parameters (`POST /api/v1/convert`)

| Parameter | Type | Default | Description |
|---|---|---|---|
| `file` | binary | — | Input file (multipart upload). PNG/JPG/WebP for images |
| `file_url` | string | — | Optional file URL (http/https). Server downloads and processes |
| `output_format` | string | `markdown` | `markdown`, `html`, `json`, `chunks`. Comma-separate for multiple |
| `mode` | string | `fast` | `fast`, `balanced`, `accurate` |
| `max_pages` | int | — | Maximum pages to convert |
| `page_range` | string | — | Comma-separated ranges like `0,5-10,20`. Overrides `max_pages` |
| `paginate` | bool | `false` | Separate output by page with horizontal rules |
| `add_block_ids` | bool | `false` | Add `data-block-id` attributes to HTML elements for citation tracking |
| `include_markdown_in_chunks` | bool | `false` | Include markdown field in chunks and JSON output |
| `disable_image_extraction` | bool | `false` | Disable image extraction from document |
| `disable_image_captions` | bool | `false` | Disable synthetic image captions/descriptions |
| `word_bboxes` | bool | `false` | Per-word bounding boxes with confidence scores (HTML only, $0.30/1K pages) |
| `fence_synthetic_captions` | bool | `false` | Wrap synthetic captions with HTML comment markers |
| `token_efficient_markdown` | bool | `false` | Optimize markdown for LLM token usage (compact tables, single-space indents) |
| `skip_cache` | bool | `false` | Skip cache and re-run conversion |
| `save_checkpoint` | bool | `false` | Save checkpoint for later `/extract` or `/segment` calls |
| `extras` | string | — | Comma-separated: `track_changes`, `chart_understanding`, `table_cell_bboxes`, `list_item_bboxes`, `extract_links`, `infographic`, `new_block_types` |
| `additional_config` | string | — | JSON string. Keys: `keep_pageheader_in_output`, `keep_pagefooter_in_output`, `keep_spreadsheet_formatting` |
| `webhook_url` | string | — | Webhook URL for completion notification |
| `processing_location` | string | — | `us` or `eu`. Requires `file_url` (no multipart) |
| `eval_rubric_id` | int | — | Optional eval rubric ID to run evaluation after conversion |
| `workflowstepdata_id` | int | — | Optional workflow step data ID |
| `model_override_settings` | string | — | Internal model override |
| `force_new` | bool | `false` | Internal: force Modal backend |

### Response Fields (Final Poll — MarkerFinalResponse)

| Field | Type | Description |
|---|---|---|
| `status` | string | `processing`, `complete`, or `failed` |
| `success` | boolean\|null | Whether conversion succeeded |
| `output_format` | string | Requested output format |
| `markdown` | string\|null | Markdown output |
| `html` | string\|null | HTML output |
| `json` | object\|null | JSON output (blocks with bounding boxes and types) |
| `chunks` | object\|null | Chunked output (top-level `blocks` array) |
| `images` | object\|null | Extracted images as `{filename: base64}` |
| `metadata` | object\|null | Document metadata and conversion process info |
| `page_count` | int\|null | Number of pages converted |
| `parse_quality_score` | number\|null | Quality score (0–5) |
| `cost_breakdown` | object\|null | Cost details (list cost and final cost after discounts) |
| `checkpoint_id` | string\|null | Checkpoint ID (if `save_checkpoint` was true) |
| `runtime` | number\|null | Conversion runtime in seconds |
| `result_url` | string\|null | Signed URL for downloading completed result JSON |
| `expires_in` | int\|null | Seconds until `result_url` expires |
| `error` | string\|null | Error message if failed |
| `versions` | object\|null | Library versions used |
| `evaluation` | object\|null | Evaluation results (if eval rubric was run) |

### Parse Quality Score Rubric

| Score Range | Quality | Recommended Action |
|---|---|---|
| 4.0–5.0 | Excellent | Use output directly |
| 3.0–3.9 | Good | Review for minor issues |
| 2.0–2.9 | Fair | Consider retrying with `accurate` mode |
| 0.0–1.9 | Poor | Retry with `accurate` mode or check input |

### Output Format Guidance

| Format | Best For |
|---|---|
| `markdown` | LLM/RAG pipelines (default, most compatible) |
| `html` | Web display, citation tracking (supports block IDs and bboxes) |
| `json` | Programmatic access to blocks with bounding boxes and block types |
| `chunks` | Embedding/search (pre-chunked for vector databases) |

### Bounding Box Add-ons (all require HTML output, $0.30/1K pages each)

| Add-on | Parameter | What It Annotates |
|---|---|---|
| Word bboxes | `word_bboxes=True` | Every word gets `data-bbox` and `data-confidence` span |
| Table cell bboxes | `extras="table_cell_bboxes"` | Table cells get bboxes; also enables word bboxes |
| List item bboxes | `extras="list_item_bboxes"` | List items get bboxes; also enables word bboxes |

### Analysis

Document Conversion is the foundational capability — most pipelines start with a `convert` step. The three processing modes allow tuning for throughput vs. accuracy. The checkpoint mechanism is a key optimization: parse once, then run multiple downstream operations (extract, segment) without re-parsing. The `extras` feature flags extend conversion with domain-specific capabilities (chart understanding, link extraction, infographics). The parse quality score provides a self-assessment signal for automated quality control workflows.

---

## 4. Structured Extraction

### Main Concepts

- **JSON Schema-Driven Extraction** — Provide a JSON schema defining fields to extract; the API parses the document and fills in values matching the schema.
- **Three Extraction Modes** — `turbo` (fastest, image-only, no citations/verification), `fast` (low latency with per-field citations), `balanced` (highest accuracy with per-field verification, reasoning, and citations).
- **Citation Tracking** — Every extracted field includes a `{field_name}_citations` array of block IDs, enabling traceability back to source document locations.
- **Balanced Mode Verification** — In balanced mode, each field includes `_meta` with `extraction_status`, `reasoning`, `citations`, and independent `verification` (PASS/FAIL with feedback).
- **Confidence Scoring** — In `fast` extraction mode, per-field `_score` ratings (1–5) with reasoning are returned. Mutually exclusive with balanced mode verification.
- **Saved Schemas** — Store schemas in Datalab and reference by `schema_id` instead of sending the full schema each time. Supports versioning.
- **Checkpoint Reuse** — Pass a `checkpoint_id` from a prior `/convert` call to skip re-parsing.
- **Schema Auto-Generation** — Generate candidate schemas from a document checkpoint.

### Endpoints & Parameters

| Endpoint | Method | Purpose |
|---|---|---|
| `/api/v1/extract` | `POST` | Submit a document for structured extraction |
| `/api/v1/extract/{request_id}` | `GET` | Poll for status and retrieve extracted data |

### Request Parameters (`POST /api/v1/extract`)

| Parameter | Type | Default | Description |
|---|---|---|---|
| `file` | binary | — | Input file (multipart upload) |
| `file_url` | string | — | Optional file URL |
| `page_schema` | string | — | JSON schema for extraction. Must contain `properties` key. Mutually exclusive with `schema_id` |
| `schema_id` | string | — | Saved extraction schema ID (e.g. `sch_k8Hx9mP2nQ4v`). Mutually exclusive with `page_schema` |
| `schema_version` | int | — | Version of saved schema. Defaults to latest. Only valid with `schema_id` |
| `checkpoint_id` | string | — | Checkpoint ID from prior `/convert` (with `save_checkpoint=true`). Skips re-parsing |
| `extraction_mode` | string | `balanced` | `turbo`, `fast`, or `balanced` |
| `mode` | string | `accurate` (balanced) / `fast` (otherwise) | Parsing mode: `fast`, `balanced`, `accurate` |
| `output_format` | string | `markdown` | Output format for parsed text alongside extraction results |
| `max_pages` | int | — | Maximum pages to process |
| `page_range` | string | — | Comma-separated page ranges |
| `save_checkpoint` | bool | `false` | Save checkpoint for future extraction/segmentation |
| `skip_cache` | bool | `false` | Skip cache and re-run |
| `webhook_url` | string | — | Webhook URL for completion |
| `processing_location` | string | — | `us` or `eu` |
| `workflowstepdata_id` | int | — | Workflow step data ID |
| `model_override_settings` | string | — | Internal model override |

### Extraction Modes Comparison

| Feature | turbo | fast | balanced (default) |
|---|---|---|---|
| Price | Lowest | $6/1K pages | $25/1K pages |
| Latency | Fastest | Low | Slower |
| Input | Image-only | Full document | Full document |
| Per-field citations | No | Yes | Yes |
| Extraction status | No | No | Yes (`EXTRACTED` / `NOT_RESOLVABLE`) |
| Per-field reasoning | No | No | Yes |
| Independent verification | No | No | Yes (`PASS` / `FAIL_*`) |
| Confidence scoring (`_score`) | No | Yes | No (uses verification instead) |
| Best for | Quick JSON extraction | High-volume: invoices, forms, bank statements | Compliance, financial, legal, medical |

### Balanced Mode Response Format

Each field gets three sibling keys: value, `_citations`, and `_meta`:

```json
{
  "company_name": "Whitbread PLC",
  "company_name_citations": ["/page/0/Text/3", "/page/2/Table/1"],
  "company_name_meta": {
    "extraction_status": "EXTRACTED",
    "reasoning": "The company name 'Whitbread PLC' appears...",
    "citations": ["/page/0/Text/3", "/page/2/Table/1"],
    "verification": {
      "status": "PASS",
      "feedback": "The company name...confirmed..."
    }
  }
}
```

### Extraction Status Values

| Status | Meaning | Value |
|---|---|---|
| `EXTRACTED` | Value found/derived from document | The extracted value |
| `NOT_RESOLVABLE` | Document doesn't contain/imply this value | `null` |

### Verification Status Values (Balanced Mode)

| Status | Meaning |
|---|---|
| `PASS` | Value and citations independently confirmed |
| `FAIL_UNRESOLVABLE` | Document doesn't support a value for this field |
| `FAIL_FIX` | Value flagged as incorrect — document supports a different value |
| `FAIL_CITATIONS` | Value is correct but citations are wrong/insufficient |
| `ITEMS_MISSING` | (List fields only) Document has entries not in extraction |

### Fast Mode Confidence Scoring

When `extraction_mode="fast"`, per-field `_score` objects appear after continued polling:

```json
{
  "invoice_number": "INV-2024-001",
  "invoice_number_citations": ["block_123"],
  "invoice_number_score": {"score": 5, "reasoning": "Value found verbatim..."},
  "extraction_score_average": 4.5
}
```

| Score | Meaning |
|---|---|
| 5 | High confidence — clear match with strong citation support |
| 4 | Good confidence — match found with minor ambiguity |
| 3 | Moderate confidence — partial match or uncertain citation |
| 2 | Low confidence — match is inferred or weakly supported |
| 1 | Very low confidence — no clear evidence found |

### Schema Size Limits (Balanced Mode)

For documents **under 20 pages**, balanced mode limits schema size (~25 fields comfortable). Documents **20+ pages** have no schema-size limit. A field = one value (string, number, date, boolean, or one choice from a list). Objects and lists are containers — count fields inside. A list's columns count once, not per row.

### `extraction_mode` vs `mode`

`extraction_mode` controls the extraction pipeline (`turbo`/`fast`/`balanced`); `mode` controls document parsing (`fast`/`balanced`/`accurate`). They combine independently (e.g. `mode="fast"` + `extraction_mode="balanced"`).

### Analysis

Structured Extraction is the most sophisticated capability, offering a spectrum from fast JSON-only extraction to fully verified, auditable extraction with per-field reasoning. The balanced mode's independent verification is unique — it doesn't just extract, it validates each field against the source document, making it suitable for compliance-sensitive use cases. The citation system (block IDs traceable to source locations) enables human-reviewable audit trails. The mutual exclusivity between confidence scoring (fast) and verification (balanced) means users must choose their quality assurance mechanism upfront.

---

## 5. Document Segmentation

### Main Concepts

- **Multi-Document Splitting** — Automatically identify and split PDFs containing multiple documents (batch-scanned files, stapled documents) into component parts.
- **Schema-Guided Segmentation** — Provide a `segmentation_schema` defining expected segment types with names and descriptions to guide detection.
- **Automatic Document Boundary Detection** — Pass `{"segmentation_strategy": "document_boundary"}` with no segments for fully automatic detection.
- **Page Range Output** — Returns page ranges for each identified segment, not the actual content. Downstream processing uses these ranges.
- **Confidence Levels** — Each segment includes a confidence rating: `high`, `medium`, or `low`.
- **Checkpoint Reuse** — Pass a `checkpoint_id` from a prior `/convert` call to skip re-parsing.

### Endpoints & Parameters

| Endpoint | Method | Purpose |
|---|---|---|
| `/api/v1/segment` | `POST` | Submit a document for segmentation |
| `/api/v1/segment/{request_id}` | `GET` | Poll for status and retrieve segmentation results |

### Request Parameters (`POST /api/v1/segment`)

| Parameter | Type | Default | Description |
|---|---|---|---|
| `file` | binary | — | Input file (multipart upload) |
| `file_url` | string | — | Optional file URL |
| `segmentation_schema` | string | **required** | JSON object defining segmentation. `{"segments": [{"name": ..., "description": ...}], "segmentation_strategy": "custom"}` or `{"segmentation_strategy": "document_boundary"}` for auto-detection |
| `checkpoint_id` | string | — | Checkpoint ID from prior `/convert`. Skips re-parsing |
| `mode` | string | `fast` | `fast`, `balanced`, `accurate` (parsing only, not used with checkpoint) |
| `max_pages` | int | — | Maximum pages to process |
| `page_range` | string | — | Comma-separated page ranges |
| `save_checkpoint` | bool | `false` | Save checkpoint for future calls |
| `skip_cache` | bool | `false` | Skip cache and re-run |
| `webhook_url` | string | — | Webhook URL for completion |
| `processing_location` | string | — | `us` or `eu` |
| `workflowstepdata_id` | int | — | Workflow step data ID |
| `model_override_settings` | string | — | Internal model override |

### Response Format

```json
{
  "segmentation_results": {
    "segments": [
      {"name": "Research Paper", "pages": [0, 1, 2], "confidence": "medium"},
      {"name": "Invoice", "pages": [3, 4], "confidence": "high"}
    ],
    "metadata": {"total_pages": 5, "segmentation_method": "auto_detected"}
  }
}
```

### Analysis

Segmentation solves the batch-scanning problem — when multiple documents are combined into a single PDF, segmentation identifies boundaries and returns page ranges for each. This pairs naturally with extraction: segment first, then extract from each segment using segment-specific schemas and page ranges. The dual mode (schema-guided vs. automatic boundary detection) provides flexibility for both known and unknown document compositions.

---

## 6. Form Filling

### Main Concepts

- **Automated Form Filling** — Fill PDF and image forms with structured data. Works with native PDF AcroForm fields and scanned/image forms.
- **Field Matching** — API detects form fields and matches them to provided data using field keys, descriptions, and optional context.
- **Confidence Threshold** — Fields below the `confidence_threshold` are not filled, preventing low-confidence matches.
- **Checkbox Support** — Values like `"yes"`, `"true"`, `"1"`, `"checked"`, `"x"` check boxes.
- **Compound Data Splitting** — API can split compound data across multiple form fields.
- **Output Formats** — Filled forms returned as PDF or PNG (base64-encoded).

### Endpoints & Parameters

| Endpoint | Method | Purpose |
|---|---|---|
| `/api/v1/fill` | `POST` | Submit a form for filling |
| `/api/v1/fill/{request_id}` | `GET` | Poll for status and retrieve filled form |

### Request Parameters (`POST /api/v1/fill`)

| Parameter | Type | Default | Description |
|---|---|---|---|
| `file` | binary | — | Input form file (PDF or image) |
| `file_url` | string | — | Optional file URL |
| `field_data` | string | **required** | JSON string mapping field keys to values and descriptions: `{"field_key": {"value": "...", "description": "..."}}` |
| `context` | string | — | Optional context to guide form filling (e.g. "Initial hire for new employee") |
| `confidence_threshold` | number | `0.5` | Minimum confidence for field matching (0.0–1.0). Fields below threshold won't be filled |
| `page_range` | string | — | Pages to process, comma-separated like `0,5-10,20` |
| `skip_cache` | bool | `false` | Skip cache and re-run inference |
| `processing_location` | string | — | `us` or `eu` |

### Response Fields

| Field | Type | Description |
|---|---|---|
| `status` | string | `processing`, `complete`, or `failed` |
| `success` | boolean | Whether filling succeeded |
| `output_format` | string | `pdf` or `png` |
| `output_base64` | string | Base64-encoded filled form |
| `fields_filled` | list | Field names that were filled |
| `fields_not_found` | list | Field names that couldn't be matched |
| `page_count` | int | Pages processed |
| `runtime` | float | Processing time in seconds |
| `cost_breakdown` | object | Cost details |

### Supported Form Types

| Form Type | How It Works |
|---|---|
| PDF with native AcroForm fields | Uses form fields directly |
| PDF with visual fields (no AcroForm) | Detects field locations, adds text overlays |
| Images (PNG, JPG) | Detects field locations, draws text on image |

### Analysis

Form filling bridges the gap between structured data and visual form documents. The field matching system uses both the field key name and a description to locate the right field on the form, making it resilient to naming variations. The `fields_not_found` response field is critical for building validation workflows — callers can detect unmatched fields and handle them. The confidence threshold provides a tuning knob for precision vs. recall in field matching.

---

## 7. Track Changes Extraction

### Main Concepts

- **Redline Extraction** — Extract tracked changes (insertions and deletions) and comments from DOCX documents.
- **Markup Preservation** — Output in Markdown and/or HTML with all change annotations preserved.
- **Change Annotation Tags** — Insertions as `<ins>`, deletions as `~~`, comments as `<comment>`.
- **Author & Timestamp Tracking** — Each change includes author name and timestamp metadata.
- **Review Workflows** — Designed for legal document review, contract analysis, negotiation tracking, and audit trail generation.

### Endpoints & Parameters

| Endpoint | Method | Purpose |
|---|---|---|
| `/api/v1/track-changes` | `POST` | Submit a DOCX for track changes extraction |
| `/api/v1/track-changes/{request_id}` | `GET` | Poll for status and retrieve results |

### Request Parameters (`POST /api/v1/track-changes`)

| Parameter | Type | Default | Description |
|---|---|---|---|
| `file` | binary | — | Input DOCX file (multipart upload) |
| `file_url` | string | — | Optional file URL |
| `output_format` | string | `markdown,html,chunks` | Comma-separated: `markdown`, `html`, `chunks` |
| `max_pages` | int | — | Maximum pages to process |
| `page_range` | string | — | Comma-separated page ranges |
| `paginate` | bool | `false` | Separate output by page with horizontal rules |
| `skip_cache` | bool | `false` | Skip cache and re-run inference |
| `webhook_url` | string | — | Webhook URL for completion |
| `processing_location` | string | — | `us` or `eu` |
| `workflowstepdata_id` | int | — | Workflow step data ID |
| `model_override_settings` | string | — | Internal model override |

### Output Markup

| Markup | Meaning |
|---|---|
| `<ins>new text</ins>` | Inserted text |
| `~~old text~~` | Deleted text |
| `<comment>marked text</comment>` | Commented text |

### Analysis

Track changes extraction is the inverse of Create Document — it reads DOCX redlines and converts them to structured markup. This enables programmatic analysis of document revisions: generating summaries of changes, identifying changes by specific authors, extracting action items from comments, and creating audit trails. The output can be fed directly to LLMs for contract risk assessment or negotiation analysis.

---

## 8. Create Document (DOCX Generation)

### Main Concepts

- **Markdown to DOCX** — Generate DOCX documents from markdown content with native Word formatting.
- **Track Changes Support** — Markdown markup tags (`<ins>`, `~~`, `<comment>`) are converted to native Word revision marks.
- **Collaborative Review Documents** — Generate legal documents, contracts with redlines, and collaborative review documents programmatically.
- **JSON Request Body** — Unlike other endpoints, this uses `application/json` (not multipart form data).

### Endpoints & Parameters

| Endpoint | Method | Purpose |
|---|---|---|
| `/api/v1/create-document` | `POST` | Submit markdown for DOCX generation |
| `/api/v1/create-document/{request_id}` | `GET` | Poll for status and retrieve generated DOCX |

### Request Schema (CreateDocumentRequest — JSON body)

| Field | Type | Default | Required | Description |
|---|---|---|---|---|
| `markdown` | string | — | Yes | Markdown content with optional track changes markup (`<ins>`, `~~`, `<comment>` tags) |
| `output_format` | string | `docx` | No | Currently only `docx` is supported |
| `webhook_url` | string | — | No | Webhook URL for completion |
| `processing_location` | string | — | No | `us` or `eu` |
| `model_override_settings` | object | — | No | Internal model override |

### Track Changes Markup Tags

**Insertions — `<ins>` tags:**

| Attribute | Required | Description |
|---|---|---|
| `data-revision-author` | Yes | Author name |
| `data-revision-datetime` | Yes | ISO 8601 timestamp |

**Deletions — `<del>~~` tags:**

| Attribute | Required | Description |
|---|---|---|
| `data-revision-author` | Yes | Author name |
| `data-revision-datetime` | Yes | ISO 8601 timestamp |

**Comments — `<comment>` tags:**

| Attribute | Required | Description |
|---|---|---|
| `data-comment-author` | Yes | Author/reviewer name |
| `text` | Yes | The comment text |
| `data-comment-datetime` | No | ISO 8601 timestamp (defaults to current) |
| `data-comment-initial` | No | Author initials (auto-generated if omitted) |

### Response Fields

| Field | Type | Description |
|---|---|---|
| `status` | string | `processing` or `complete` |
| `success` | boolean | Whether creation succeeded |
| `output_format` | string | `docx` |
| `output_base64` | string | Base64-encoded DOCX file |
| `runtime` | float | Processing time in seconds |
| `page_count` | int | Pages in generated document |
| `cost_breakdown` | object | Cost details |
| `error` | string | Error message if failed |

### Analysis

Create Document completes the document round-trip: Track Changes extracts redlines from DOCX → markdown, and Create Document generates DOCX from markdown with redlines. This enables automated document generation pipelines — templates can be populated with data, redlines can be programmatically inserted, and the resulting DOCX is ready for Word-based review workflows. The JSON body (vs. multipart) reflects that the input is text, not a file upload.

---

## 9. Custom Processors

### Main Concepts

- **AI-Generated Processors** — Fine-tune document conversion output with AI-generated custom processors for domain-specific formatting, edge-case layouts, and use-case-specific transformations.
- **Modification Levels** — Block-level (modify individual blocks), page-level (modify entire pages), classification (classify pages for downstream routing).
- **Conversational Builder** — Chat-driven "Describe" endpoint for building processor descriptions in natural language.
- **Versioning** — Each iteration creates a new version. Switch active version, list versions, iterate on existing processors.
- **Templates** — Promote completed processors to templates, clone templates to new processors, share across teams.
- **Eval Definitions** — Each processor can have an `eval_definition` for quality scoring.
- **Transfer Between Teams** — Admins can transfer processor ownership between teams (useful for beta testing and sharing).

### Endpoints & Parameters

#### Execution

| Endpoint | Method | Purpose | Key Parameters |
|---|---|---|---|
| `/api/v1/custom-processor` | `POST` | Execute a custom processor | `pipeline_id` (required), `version`, `mode`, `output_format`, `file`/`file_url`, `run_eval`, `max_pages`, `page_range`, `paginate`, `add_block_ids`, `include_markdown_in_chunks`, `disable_image_extraction`, `disable_image_captions`, `skip_cache`, `webhook_url`, `processing_location` |
| `/api/v1/custom-processor/{request_id}` | `GET` | Poll for status and results | — |
| `/api/v1/custom-pipeline` | `POST` | **Deprecated** — alias for custom-processor | Same as above. Sunset: September 30, 2026 |

#### Processor Management

| Endpoint | Method | Purpose |
|---|---|---|
| `/api/v1/custom-processors` | `GET` | List all custom processors for the team |
| `/api/v1/custom-processors/{id}` | `GET` | Get processor details |
| `/api/v1/custom-processors/{id}/versions` | `GET` | List all versions of a processor |
| `/api/v1/custom-processors/{id}/versions/{version}` | `GET` | Get specific version detail (includes `pipeline_params` and `eval_definition`) |
| `/api/v1/custom-processors/{id}/iterate` | `POST` | Iterate on an existing processor |
| `/api/v1/custom-processors/{id}/archive` | `POST` | Archive (soft-delete) a processor |
| `/api/v1/custom-processors/{id}/restore` | `POST` | Restore an archived processor |
| `/api/v1/custom-processors/{id}` | `DELETE` | Permanently delete processor and all versions (admin-only) |
| `/api/v1/custom-processors/{id}/export` | `GET` | Export processor with all versions (admin-only) |
| `/api/v1/custom-processors/{id}/transfer` | `POST` | Transfer to another team (admin-only) |
| `/api/v1/custom-processors/{id}/active-version` | `PUT` | Set active version |
| `/api/v1/custom-processors/{id}/eval-definition` | `GET` / `PUT` | Get/update eval definition |
| `/api/v1/custom-processors/{id}/pipelines` | `GET` | List pipelines using this processor |
| `/api/v1/custom-processors/seed` | `POST` | Directly create from JSON (admin-only) |
| `/api/v1/custom-processors/submit` | `POST` | Submit generation request |
| `/api/v1/custom-processors/status/{request_id}` | `GET` | Check generation status |
| `/api/v1/custom-processors/describe` | `POST` | Conversational description builder |
| `/api/v1/custom-processors/access` | `GET` | Check team access to custom processors |
| `/api/v1/custom-processors/creation-allowance` | `GET` | Check creation allowance |

#### Template Management

| Endpoint | Method | Purpose |
|---|---|---|
| `/api/v1/processor-templates` | `GET` | List all published templates |
| `/api/v1/processor-templates/{id}` | `GET` | Get template details |
| `/api/v1/processor-templates/{id}/clone` | `POST` | Clone template to new processor |
| `/api/v1/processor-templates/{id}/examples` | `POST` | Upload example files for template (admin-only) |
| `/api/v1/processor-templates/{id}/examples/{file_id}` | `DELETE` | Remove example file (admin-only) |
| `/api/v1/processor-templates/{id}/examples/{file_id}` | `GET` | Download example file |
| `/api/v1/processor-templates/{id}/examples/{file_id}/thumbnail` | `GET` | Download example thumbnail |
| `/api/v1/processor-templates/promote` | `POST` | Create template from existing processor (admin-only) |
| `/api/v1/processor-templates/{id}` | `PUT` | Update template metadata (admin-only) |
| `/api/v1/processor-templates/{id}/un-template` | `POST` | Un-template a pipeline (admin-only) |

### Execution Request Parameters (`POST /api/v1/custom-processor`)

| Parameter | Type | Default | Description |
|---|---|---|---|
| `pipeline_id` | string | **required** | Custom processor ID (`cp_XXXXX`) or template slug |
| `version` | int | — | Processor version. Defaults to active version |
| `file` | binary | — | Input file (multipart upload) |
| `file_url` | string | — | Optional file URL |
| `run_eval` | bool | `false` | Run evaluation rules defined for this processor |
| `mode` | string | `fast` | `fast`, `balanced`, `accurate` (underlying parsing step) |
| `output_format` | string | `markdown` | `json`, `html`, `markdown`, `chunks` |
| `max_pages` | int | — | Maximum pages to process |
| `page_range` | string | — | Comma-separated page ranges |
| `paginate` | bool | `false` | Page delimiters |
| `add_block_ids` | bool | `false` | Block IDs for citation tracking |
| `include_markdown_in_chunks` | bool | `false` | Include markdown in chunks/JSON output |
| `disable_image_extraction` | bool | `false` | Don't extract images |
| `disable_image_captions` | bool | `false` | Don't generate captions |
| `skip_cache` | bool | `false` | Skip cache and re-run |
| `webhook_url` | string | — | Webhook URL for completion |
| `processing_location` | string | — | `us` or `eu` |

### Describe Customizer (Conversational Builder)

| Field | Type | Description |
|---|---|---|
| Chat history (input) | array | Conversation messages for building the description |
| Assistant message (output) | string | Next assistant response |
| `proposed_description` (output) | string\|null | Proposed description when system has enough context |

### Analysis

Custom processors are the extensibility mechanism of the platform. When standard conversion doesn't produce exactly the right output — domain-specific table formats, specialized caption styles, page classification for routing — custom processors fill the gap. The conversational builder ("Describe") lowers the barrier to creating processors by allowing natural language descriptions. The template system enables sharing and reuse across teams. The eval definition system provides quality scoring for custom processors, closing the loop on processor quality management.

---

## 10. Pipelines

### Main Concepts

- **Processor Chaining** — Chain multiple processors (convert, extract, segment, custom) into a single reusable unit with one API call.
- **Versioned Configurations** — Pipelines have a draft → saved → published lifecycle. Published versions are immutable snapshots. Drafts auto-save.
- **Execution DAG** — Running a pipeline creates an execution DAG with per-step tracking, billing, and intermediate result retrieval.
- **Step Types** — `convert` (must be first, except for standalone `fill`), `segment`, `extract` (always terminal), `custom`, `fill` (standalone only).
- **Checkpoint Passing** — Steps pass output to the next via checkpoints. Most pipelines start with `convert`.
- **Per-Step Results** — Each step's result can be retrieved individually via `result_url` or the step result endpoint.
- **Eval Integration** — Pipelines can run evaluation rubrics on steps (`run_evals`).

### Composition Rules

| Rule | Description |
|---|---|
| Pipeline starts with | `convert` or `fill` |
| `extract` position | Always terminal (nothing follows) |
| `segment` can feed into | `extract` |
| `custom` can feed into | `extract` |
| `fill` position | Always standalone |

### Common Pipeline Patterns

| Pattern | Use Case |
|---|---|
| `convert` | Simple document parsing |
| `convert → extract` | Parse and extract structured fields |
| `convert → segment` | Parse and split into sections |
| `convert → segment → extract` | Split, then extract from each section |
| `convert → custom → extract` | Apply custom processing, then extract |
| `fill` | Version and track form-filling workflows |

### Pipeline Lifecycle

| State | `active_version` | Description |
|---|---|---|
| Draft | `0` | Edits auto-save. No published version |
| Saved | `0` | Named pipeline, still no published version |
| Published | `1`, `2`, ... | Immutable version snapshots exist |

### Endpoints & Parameters

#### Pipeline CRUD

| Endpoint | Method | Purpose | Key Parameters |
|---|---|---|---|
| `/api/v1/pipelines` | `POST` | Create a new pipeline | `steps` (required, array of PipelineStep) |
| `/api/v1/pipelines` | `GET` | List pipelines for the team | Filters |
| `/api/v1/pipelines/{id}` | `GET` | Get pipeline by ID | — |
| `/api/v1/pipelines/{id}` | `PUT` | Update pipeline steps (auto-save) | `steps` |
| `/api/v1/pipelines/{id}/save` | `PUT` | Name and promote to saved status | `name` |
| `/api/v1/pipelines/{id}/archive` | `POST` | Archive pipeline | — |
| `/api/v1/pipelines/{id}/unarchive` | `POST` | Unarchive pipeline | — |
| `/api/v1/pipelines/{id}/rate` | `GET` | Get pipeline rate based on plan and region | — |

#### Versioning

| Endpoint | Method | Purpose |
|---|---|---|
| `/api/v1/pipelines/{id}/versions` | `POST` | Create a new version snapshot |
| `/api/v1/pipelines/{id}/versions` | `GET` | List all versions (newest first) |
| `/api/v1/pipelines/{id}/discard` | `POST` | Discard draft, reset to published version |

#### Execution

| Endpoint | Method | Purpose |
|---|---|---|
| `/api/v1/pipelines/{id}/run` | `POST` | Execute a pipeline on a file |
| `/api/v1/pipelines/executions/{execution_id}` | `GET` | Poll execution status |
| `/api/v1/pipelines/executions` | `GET` | List recent executions |
| `/api/v1/pipelines/executions/{id}/steps/{step_index}/result` | `GET` | Fetch intermediate result for a step |

### PipelineStep Schema

| Field | Type | Required | Description |
|---|---|---|---|
| `type` | string | Yes | `convert`, `extract`, `segment`, `custom`, `fill` |
| `settings` | object | — | Step-specific executor settings. For `extract`: includes `page_schema` (JSON string). For `segment`: includes `segmentation_schema` (JSON string). Flows directly to executor via `config.update(settings)` |
| `custom_processor_id` | string | — | Custom processor ID (`cp_XXXXX`) for `custom` steps |
| `eval_rubric_id` | int | — | Eval rubric ID to score this step's output |
| `pending_check_url` | string | — | URL to poll for processor creation status (temporary) |
| `pending_name` | string | — | Display name while processor is generating (temporary) |

### Run Pipeline Parameters (`POST /api/v1/pipelines/{id}/run`)

| Parameter | Type | Default | Description |
|---|---|---|---|
| `file` | binary | — | Input file (multipart upload) |
| `file_url` | string | — | Optional file URL |
| `version` | int | — | Pipeline version: `0` = draft, omit = active published, `1+` = specific version |
| `page_range` | string | — | Pages to process |
| `output_format` | string | — | Override output format |
| `run_evals` | bool | `false` | Run evaluation steps |
| `skip_cache` | bool | `false` | Skip executor cache, re-run all steps |
| `checkpoint_id` | string | — | Checkpoint from previous run's convert step. Skips re-parsing (file still required). Ignored when `run_evals` is set |
| `webhook_url` | string | — | URL to POST on completion |
| `processing_location` | string | — | `us` or `eu` |

### Pipeline Execution Response (PipelineExecutionResponse)

| Field | Type | Description |
|---|---|---|
| `execution_id` | string | Unique execution ID |
| `pipeline_id` | string | Pipeline ID |
| `pipeline_version` | int | Version executed |
| `status` | string | Execution status |
| `steps` | array | Per-step status (PipelineExecutionStepResponse) |
| `started_at` | datetime\|null | Execution start |
| `completed_at` | datetime\|null | Execution completion |
| `created` | datetime\|null | Creation timestamp |
| `config_snapshot` | object\|null | Configuration at execution time |
| `input_config` | object\|null | Input configuration |
| `rate_breakdown` | object\|null | Billing rate breakdown |

### Per-Step Status (PipelineExecutionStepResponse)

| Field | Type | Description |
|---|---|---|
| `step_index` | int | Step position in pipeline |
| `step_type` | string | Processor type |
| `status` | string | `pending`, `dispatched`, `running`, `completed`, `failed`, `skipped` |
| `lookup_key` | string\|null | Key for retrieving partial results |
| `result_url` | string\|null | URL for step result |
| `started_at` | datetime\|null | Step start |
| `finished_at` | datetime\|null | Step completion |
| `error_message` | string\|null | Error if failed |
| `source_step_type` | string\|null | Preceding step type |
| `checkpoint_id` | string\|null | Checkpoint from this step |

### Execution Status Values

| Status | Description |
|---|---|
| `pending` | Queued, not started |
| `running` | Processors executing |
| `completed` | All steps finished successfully |
| `completed_with_errors` | Some steps completed, some failed |
| `failed` | Execution failed |

### Analysis

Pipelines are the production-grade composition mechanism. While individual endpoints are good for testing and simple tasks, pipelines provide versioning, immutability, and per-step tracking essential for production integrations. The draft → published lifecycle with immutable version snapshots enables safe iteration — production can pin to a specific version while development continues on the draft. The per-step result retrieval is powerful for debugging and for workflows that need intermediate outputs (e.g., use the convert step's markdown for display while using the extract step's JSON for downstream processing). Billing is per-page, additive across processors.

---

## 11. Extraction Schemas

### Main Concepts

- **Reusable Schema Storage** — Store extraction schemas in Datalab and reference by `schema_id` instead of sending the full schema with every request.
- **Versioning** — Schemas support versioning. Update with `create_new_version=True` to save current state. Pin to specific version with `schema_version`.
- **Soft Delete** — Archiving schemas is a soft-delete operation.
- **Mutual Exclusivity** — `schema_id` and `page_schema` are mutually exclusive on the `/extract` endpoint.

### Endpoints & Parameters

| Endpoint | Method | Purpose | Key Parameters |
|---|---|---|---|
| `/api/v1/extraction_schemas` | `POST` | Create a new extraction schema | `name`, `description`, `schema_json` |
| `/api/v1/extraction_schemas` | `GET` | List schemas for the team | `include_archived` |
| `/api/v1/extraction_schemas/{id}` | `GET` | Get schema by ID | — |
| `/api/v1/extraction_schemas/{id}` | `PUT` | Update schema | `create_new_version` (optional) |
| `/api/v1/extraction_schemas/{id}` | `DELETE` | Soft-delete (archive) schema | — |

### Schema Object

| Field | Type | Description |
|---|---|---|
| `schema_id` | string | Stable ID (e.g. `sch_k8Hx9mP2nQ4v`) |
| `name` | string | Human-readable name (max 200 chars) |
| `description` | string\|null | Optional description |
| `schema_json` | object | JSON schema with `properties` key |
| `version` | int | Current version (starts at 1) |
| `version_history` | array | Previous versions |
| `archived` | boolean | Whether archived |
| `created` | datetime | Creation timestamp |
| `updated` | datetime | Last update timestamp |

### `/extract` Schema-Related Parameters

| Parameter | Type | Description |
|---|---|---|
| `schema_id` | string | ID of saved schema. Mutually exclusive with `page_schema` |
| `schema_version` | int | Version to use. Only with `schema_id`. Defaults to latest |

### Analysis

Saved schemas reduce request payload size and enable schema governance. The versioning system allows schema evolution without breaking existing integrations — production can pin to a specific version while the schema is updated for new use cases. The recommendation to always specify `schema_version` for consistent results is important for reproducibility in production environments.

---

## 12. Extraction Schema Generation

### Main Concepts

- **Auto-Generate Schemas** — For a given document checkpoint, automatically generate potential extraction schemas.
- **Three Complexity Tiers** — Returns `simple_schema`, `moderate_schema`, and `complex_schema` options.
- **Checkpoint-Based** — Requires a `checkpoint_id` from a prior `/convert` call.

### Endpoints & Parameters

| Endpoint | Method | Purpose |
|---|---|---|
| `/api/v1/marker/extraction/gen_schemas` | `POST` | Generate extraction schemas from a checkpoint |
| `/api/v1/marker/extraction/gen_schemas/{request_id}` | `GET` | Poll for schema generation results |

### Request Schema (ExtractionGenSchemasRequest — JSON body)

| Field | Type | Required | Description |
|---|---|---|---|
| `checkpoint_id` | string | Yes | Checkpoint ID from a prior `/convert` call |
| `webhook_url` | string | — | Webhook URL for completion |
| `processing_location` | string | — | `us` or `eu` |
| `model_override_settings` | object | — | Internal model override |

### Analysis

Schema generation is a bootstrapping tool — when you have a new document type and don't know what schema to use, the API analyzes the document and proposes schemas at three complexity levels. This reduces the cold-start problem for structured extraction: instead of manually designing a schema, you can generate candidates, test them, and refine. The three-tier output (simple/moderate/complex) lets users choose the right granularity for their use case.

---

## 13. File Management

### Main Concepts

- **Direct Upload to Storage** — Three-step upload flow: request presigned URL → upload directly to storage → confirm upload. Avoids proxying large files through the API server.
- **`datalab://` References** — Uploaded files get a `datalab://file-{id}` reference usable in other API calls (required when using `processing_location`).
- **Team-Scoped** — Files belong to a team. List, get metadata, download, and delete operations.
- **Pagination** — List endpoint supports `limit` (max 100) and `offset` parameters.

### Endpoints & Parameters

| Endpoint | Method | Purpose | Key Parameters |
|---|---|---|---|
| `/api/v1/files/upload` | `POST` | Request presigned upload URL | `filename`, `content_type`, `processing_location` |
| `/api/v1/files/{file_id}/confirm` | `GET` | Confirm upload completed | `file_id` or hashid |
| `/api/v1/files` | `GET` | List all uploaded files | `limit` (default 50, max 100), `offset` |
| `/api/v1/files/{file_id}` | `GET` | Get file metadata | — |
| `/api/v1/files/{file_id}/download` | `GET` | Generate presigned download URL | `expires_in` (default 3600) |
| `/api/v1/files/{file_id}` | `DELETE` | Delete an uploaded file | — |

### Request Upload URL Response

| Field | Type | Description |
|---|---|---|
| `file_id` | int | Unique file ID (use to confirm upload) |
| `upload_url` | string | Presigned PUT URL |
| `expires_in` | int | URL expiry in seconds (default 3600) |
| `reference` | string | File reference in `datalab://file-{id}` format |

### Confirm Upload Response

| Field | Type | Description |
|---|---|---|
| `success` | boolean | Whether confirmation succeeded |
| `file_id` | int | File ID |
| `reference` | string | File reference in `datalab://file-{id}` format |
| `message` | string | Confirmation message |

### File Metadata Response

| Field | Type | Description |
|---|---|---|
| `file_id` | int | Unique file ID |
| `original_filename` | string | Original filename |
| `content_type` | string | MIME type |
| `file_size` | int | File size in bytes |
| `upload_status` | string | `pending`, `completed`, or `failed` |
| `created` | datetime | Upload timestamp |
| `reference` | string | File reference in `datalab://file-{id}` format |

### Analysis

The file management system enables pre-uploading files and referencing them by `datalab://` URI in subsequent API calls. This is particularly important for data residency — when `processing_location` is set, multipart uploads are rejected, so files must be pre-uploaded. The three-step presigned URL flow is a standard pattern for scalable file ingestion that avoids proxying large files through the API server. The `datalab://` reference scheme provides a stable, reusable file identifier.

---

## 14. Collections & Batch Evaluation

### Main Concepts

- **Collections** — Grouping mechanism for uploaded files. Create, update, list, and manage collections of files.
- **Batch Runs** — Run batch evaluation across all files in a collection. Track progress and retrieve per-file results.
- **S3 Integration** — Collections support S3 source syncing (import files from S3) and S3 output writeback (write results to S3).
- **Soft Delete** — Collections are soft-deleted (archived), not permanently removed.

### Endpoints & Parameters

#### Collection Management

| Endpoint | Method | Purpose |
|---|---|---|
| `/api/v1/collections` | `POST` | Create a new collection |
| `/api/v1/collections` | `GET` | List collections for the team |
| `/api/v1/collections/{id}` | `GET` | Get collection summary and S3 sync status |
| `/api/v1/collections/{id}` | `PUT` | Update collection name/description |
| `/api/v1/collections/{id}` | `DELETE` | Soft-delete (archive) collection |

#### Collection Files

| Endpoint | Method | Purpose |
|---|---|---|
| `/api/v1/collections/{id}/files` | `GET` | List collection files (cursor pagination) |
| `/api/v1/collections/{id}/files` | `POST` | Add existing uploaded files to collection |
| `/api/v1/collections/{id}/files/{file_id}` | `DELETE` | Unlink a file from collection (does not delete file) |

#### Batch Runs

| Endpoint | Method | Purpose |
|---|---|---|
| `/api/v1/collections/batch-runs` | `POST` | Start a batch evaluation run on all files in a collection |
| `/api/v1/collections/batch-runs` | `GET` | List batch runs (filter by collection, eval rubric, pipeline) |
| `/api/v1/collections/batch-runs/{id}` | `GET` | Get batch run status and progress |
| `/api/v1/collections/batch-runs/{id}/results` | `GET` | Get per-file results for a batch run |

#### S3 Source Integration

| Endpoint | Method | Purpose |
|---|---|---|
| `/api/v1/collections/{id}/s3-source` | `PUT` | Create/update S3 source and enqueue sync |
| `/api/v1/collections/{id}/s3-source` | `GET` | Get redacted S3 source config and sync status |
| `/api/v1/collections/{id}/s3-source` | `DELETE` | Disconnect S3 source, remove synthetic file rows |
| `/api/v1/collections/{id}/s3-sync` | `POST` | Trigger resync for existing S3 source |
| `/api/v1/collections/s3-statuses` | `GET` | List minimal S3 status for collection rows |

#### S3 Output Writeback

| Endpoint | Method | Purpose |
|---|---|---|
| `/api/v1/collections/{id}/s3-output` | `PUT` | Create/update S3 output writeback destination |
| `/api/v1/collections/{id}/s3-output` | `DELETE` | Disable S3 output writeback |
| `/api/v1/collections/{id}/s3-writeback` | `POST` | Enqueue S3 output writeback for latest completed batch run |

### Analysis

Collections and batch evaluation provide the infrastructure for at-scale document processing. Instead of submitting files one at a time, collections allow grouping files and running batch evaluation across all of them. The S3 integration is significant for enterprise workflows — files can be ingested directly from S3 buckets and results written back to S3, enabling fully automated pipelines without manual file transfer. The batch run system tracks progress and provides per-file results, making it suitable for processing large document sets with monitoring.

---

## 15. Eval Rubrics

### Main Concepts

- **Evaluation Rubrics** — Define evaluation rules to score processing output quality. Used with custom processors, pipelines, and batch runs.
- **Rule Types** — `block`, `page`, or `document` level rules. Each rule produces a score (0–5).
- **Feedback from User Data** — Create rubrics from user feedback items using LLM rewrite.
- **Import from Processors** — Import `eval_definition` from a custom processor's active version.
- **Soft Delete** — Rubrics are soft-deleted (archived).

### Endpoints & Parameters

| Endpoint | Method | Purpose |
|---|---|---|
| `/api/v1/eval_rubrics` | `POST` | Create new eval rubric for the team |
| `/api/v1/eval_rubrics` | `GET` | List eval rubrics for the team |
| `/api/v1/eval_rubrics/{id}` | `GET` | Get eval rubric by ID |
| `/api/v1/eval_rubrics/{id}` | `PUT` | Update eval rubric |
| `/api/v1/eval_rubrics/{id}` | `DELETE` | Soft-delete (archive) eval rubric |
| `/api/v1/eval_rubrics/from-feedback` | `POST` | Convert user feedback items into structured eval rubric (LLM rewrite, saves to DB) |
| `/api/v1/eval_rubrics/generate-from-feedback` | `POST` | Generate eval rubric from feedback items (LLM rewrite, no DB save) |
| `/api/v1/eval_rubrics/import-from-pipeline` | `POST` | Import eval_definition from a custom pipeline's active version |

### Evaluation Results (in Conversion/Extraction Response)

| Field | Type | Description |
|---|---|---|
| `eval_definition_name` | string | Name of evaluation definition |
| `evaluations` | array | Per-rule evaluation summaries |
| `total_items_evaluated` | int | Total items evaluated across all rules |

### Evaluation Rule Summary

| Field | Type | Description |
|---|---|---|
| `name` | string | Name of evaluation rule |
| `type` | string | `block`, `page`, or `document` |
| `rule_score` | number | Aggregated score for this rule (0–5) |
| `items_evaluated` | int | Number of items evaluated |
| `individual_results` | array | Bottom-k lowest scoring results with score, feedback, block_id, page_id, block_type |

### Analysis

Eval rubrics provide a systematic way to measure and compare document processing quality. The three granularity levels (block, page, document) allow evaluation at different scopes. The ability to generate rubrics from user feedback creates a closed-loop improvement system: users flag issues → feedback is converted to rubrics → rubrics score future processing → low scores trigger review. The import-from-pipeline feature allows reusing evaluation definitions across processors.

---

## 16. Forge Evals

### Main Concepts

- **Configuration Comparison** — Evaluate and compare parsing configurations across multiple documents to find optimal settings.
- **Multi-Model Comparison** — Compare Datalab presets, other OCR models (OlmoOCR, RolmoOCR, DotsOCR, DeepSeekOCR), and external providers.
- **Visual Diff** — Side-by-side comparison with inline diff highlighting (word-level, rendered diff for HTML/Markdown).
- **Multiple Iterations** — Run configurations 2× or 3× to assess consistency and identify variability.

### Constraints

| Limit | Value |
|---|---|
| Max documents per session | 10 |
| Max configurations per session | 5 |
| Max iterations per configuration | 3 |

### Configuration Sources

| Tab | Options |
|---|---|
| Datalab | Presets (Fast, Balanced, Accurate, Track Changes, Chart Understanding) or custom configs |
| Other Models | OlmoOCR, RolmoOCR, DotsOCR, DeepSeekOCR (hosted by Datalab) |
| External Providers | Limited to select users |

### Comparison View Features

- Parallel view with inline diff highlighting
- Multiple output formats (Markdown, HTML, JSON, Chunks)
- Rendered output toggle (formatted HTML/Markdown, JSON with bounding boxes)
- Visual diffs (word-level highlighting; rendered diff for HTML/Markdown only)
- JSON visualization with document thumbnails and bounding boxes
- Processing metrics and diff statistics (lines added/removed/changed)

### Analysis

Forge Evals is the tuning and selection tool. Before committing to a parsing configuration for production, users can empirically compare options across their actual documents. The inclusion of third-party OCR models (OlmoOCR, RolmoOCR, DotsOCR, DeepSeekOCR) alongside Datalab's own presets enables apples-to-apples comparison. The multi-iteration feature addresses a subtle but important concern: parsing variability — running the same config multiple times reveals whether results are deterministic or variable, which is critical for production reliability.

---

## 17. Thumbnails

### Main Concepts

- **Page Thumbnails** — Generate thumbnail images for document pages from a previous conversion request.
- **Lookup Key Based** — Uses the `lookup_key` (request ID) from a prior conversion.
- **Configurable Width** — Thumbnail width in pixels (default 300).
- **Track Changes Support** — Optional `track_changes` flag for DOCX documents.
- **Base64 JPG Output** — Returns array of base64-encoded JPG images.

### Endpoints & Parameters

| Endpoint | Method | Purpose |
|---|---|---|
| `/api/v1/thumbnails/{lookup_key}` | `GET` | Generate thumbnails for a previous request |

### Parameters

| Parameter | Type | Default | Description |
|---|---|---|---|
| `lookup_key` | string | **required** (path) | Request ID from previous conversion |
| `page_range` | string | — | Pages to generate thumbnails for |
| `thumb_width` | int | `300` | Thumbnail width in pixels |
| `track_changes` | bool | `false` | Whether to show track changes in thumbnails |

### Response (ThumbnailResponse)

| Field | Type | Description |
|---|---|---|
| `success` | boolean | Whether request was successful |
| `error` | string\|null | Error message |
| `thumbnails` | array\|null | List of base64-encoded thumbnail images (JPG format) |

### Analysis

Thumbnails provide visual previews of processing results, useful for UI displays and quality verification. The `track_changes` flag is a thoughtful addition — it allows visualizing redlines in thumbnail form, which is valuable for document review interfaces. The dependency on a prior conversion's `lookup_key` means thumbnails are a post-processing feature, not a standalone capability.

---

## 18. Workflows (Temporal)

### Main Concepts

- **Temporal Workflows** — Workflow definitions composed of steps, executed via Temporal workflow engine.
- **Step Types** — Available step types are listed via the `/api/v1/workflows/step-types` endpoint.
- **Workflow Execution** — Executing a workflow creates a WorkflowExecution and starts a Temporal workflow that dynamically loads and executes steps.
- **Authentication** — Requires `X-API-Key` header for execution.

### Endpoints & Parameters

| Endpoint | Method | Purpose |
|---|---|---|
| `/api/v1/workflows` | `POST` | Create a new workflow definition |
| `/api/v1/workflows` | `GET` | List all workflow definitions with their steps |
| `/api/v1/workflows/{id}` | `GET` | Get workflow definition with all steps |
| `/api/v1/workflows/{id}` | `DELETE` | Delete a workflow definition |
| `/api/v1/workflows/{id}/execute` | `POST` | Execute a workflow definition |
| `/api/v1/workflows/{id}/execution` | `GET` | Get execution status and results |
| `/api/v1/workflows/step-types` | `GET` | List all available step types |

### Create Workflow Parameters

| Field | Type | Description |
|---|---|---|
| `name` | string | Workflow name |
| `steps` | array | Ordered list of workflow steps |

### Analysis

The Workflows API appears to be a more general orchestration layer built on Temporal, distinct from the Pipelines system. While Pipelines are specifically for chaining document processors, Workflows seem to provide a broader step-based execution framework. The Temporal integration suggests support for long-running, fault-tolerant workflows with retries and state management.

---

## 19. Health & Compliance

### Main Concepts

- **Health Check** — Simple endpoint returning `{"status": "ok"}` to verify API availability.
- **API Health (Authenticated)** — Health check with API key validation.
- **Compliance Webhook** — Provider webhook that verifies, persists (deduped), and enqueues reconcile operations. Returns 200 fast; the enqueued reconcile re-fetches authoritative status.

### Endpoints & Parameters

| Endpoint | Method | Purpose |
|---|---|---|
| `/api/v1/health` | `GET` | Check API health (no auth required) |
| `/api/v1/api-health` | `GET` | Check API health with API key |
| `/api/v1/compliance/webhook` | `POST` | Provider compliance webhook |

### Analysis

The health endpoint is standard infrastructure for monitoring and uptime checks. The compliance webhook endpoint suggests integration with external compliance/billing providers, with a fast-acknowledge-then-reconcile pattern that separates webhook reception from authoritative status verification.

---

## 20. Cross-Cutting Concerns

### Checkpoints

Checkpoints are the central optimization pattern across the platform:

```
Convert (save_checkpoint=true) → checkpoint_id
    ├── Extract (checkpoint_id) → skip re-parsing
    ├── Segment (checkpoint_id) → skip re-parsing
    └── Generate Schemas (checkpoint_id) → analyze parsed state
```

This enables multi-operation workflows on a single document parse, saving both time and cost.

### Data Residency (`processing_location`)

Available on all inference endpoints. Values: `us` or `eu`.

| Constraint | Detail |
|---|---|
| Requires `file_url` or `datalab://` reference | Multipart uploads rejected when `processing_location` is set |
| EU pricing premium | Regional processing carries additional cost |
| Team default | When omitted, uses team's configured residency |

### Async Pattern

All processing endpoints follow the same three-phase pattern:

1. **Submit** — `POST` with file or `file_url` → returns `request_id` and `request_check_url`
2. **Poll** — `GET {request_check_url}` → returns `{"status": "processing"}` or `{"status": "complete", ...results}`
3. **Retrieve** — Results are in the complete response; also available via `result_url` (signed URL, time-limited)

The SDK handles polling automatically. REST API users implement polling loops.

### Webhooks

Alternative to polling. Configure default webhook in account settings or override per-request with `webhook_url`. The API POSTs to the URL when processing reaches a terminal state.

### `datalab://` File References

Format: `datalab://file-{id}`. Used for pre-uploaded files in the file management flow. Required when using `processing_location` (since multipart uploads are rejected).

### Result Retention

Results are deleted from Datalab servers **one hour** after processing completes. Must be retrieved promptly. This applies to all processing endpoints.

### Caching

All processing endpoints support caching. Results for identical requests may be returned from cache. Use `skip_cache=true` to force re-processing.

### Billing

- **Per-page pricing** — Costs are per 1,000 pages, additive across processors in pipelines.
- **Mode-dependent** — `accurate` > `balanced` > `fast` in cost.
- **Feature add-ons** — Bounding boxes ($0.30/1K pages each), extraction mode premiums.
- **`cost_breakdown`** — All responses include cost details (list cost and final cost after discounts).
- **Pipeline rates** — Check with `GET /api/v1/pipelines/{id}/rate`.

### Supported File Types

| Category | Formats |
|---|---|
| Documents | PDF, DOCX, PPTX, XLSX |
| Images | PNG, JPG, WebP |
| Office | Word, PowerPoint, Excel |

Max file size: 200 MB. Max pages per request: 7,000.

### Deprecated Endpoints

| Endpoint | Replacement | Sunset |
|---|---|---|
| `POST /api/v1/marker` | `POST /api/v1/convert` | — |
| `POST /api/v1/ocr` | `POST /api/v1/convert` | — |
| `POST /api/v1/table_rec` | `POST /api/v1/convert` (tables in JSON output) | — |
| `POST /api/v1/custom-pipeline` | `POST /api/v1/custom-processor` | September 30, 2026 |

### On-Premises Deployment

The on-prem container mimics the cloud API with reduced feature set:

| Feature | Cloud | On-Prem |
|---|---|---|
| Document conversion (`/convert`) | Yes | Yes |
| OCR (`/ocr`) | Yes | Yes |
| Output formats (markdown, html, json, chunks) | Yes | Yes |
| Parse quality scoring | Yes | Yes |
| Chart understanding | Yes | Yes (Chandra containers only) |
| Page range selection | Yes | Yes |
| Block IDs | Yes | Yes |
| Token-efficient markdown | Yes | Yes |
| Structured extraction (`/extract`) | Yes | Yes — `fast` and `turbo` modes only (requires Lift model); `balanced` not available |
| Form filling (`/fill`) | Yes | No |
| Create document (`/create-document`) | Yes | No |
| Thumbnails | Yes | No |
| Accurate mode | Yes | No |
| Fast mode | Yes | No |
| Link extraction | Yes | No |
| Checkpoints | Yes | No |
| File URL download | Yes | No |

### SDK Support

Datalab provides a Python SDK (`datalab-api` on PyPI) and CLI. The SDK handles:
- Authentication
- File upload and management
- Polling (automatic)
- All processing endpoints (convert, extract, segment, fill, track-changes, create-document)
- Pipeline management and execution
- Custom processor management

The SDK's `ExtractOptions` does not yet support `extraction_mode` — use the REST API for `turbo`/`balanced`/`fast` extraction mode control.

---

*Source: [Datalab Documentation](https://documentation.datalab.to/) — API Reference, Recipes, and Platform guides. Analyzed from OpenAPI 3.1.0 specifications and recipe documentation.*