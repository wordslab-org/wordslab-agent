# IBM Document Processing Extension API Analysis

> **Base URL:** `https://{hostname:port}/v1/projects/{projectId}` | **API version:** v1 | **Auth:** HTTP Basic / Bearer (Zen JWT)
> **Product:** IBM Datacap V9.1.10 / IBM Document Processing Extension 23.0.2
> **Description:** An add-on to Datacap (and other capture solutions) that brings AI/ML-based document classification, OCR, and structured data extraction (key-value pairs, tables, line items) to document processing applications. Exposes a REST API for submitting PDFs, polling status, downloading JSON outputs (basic/detailed/verbose), validating extracted KVPs, exporting the project ontology, and deleting resources.

---

## Table of Contents

1. [Platform Overview & Architecture](#1-platform-overview--architecture)
2. [Authentication & Request Pattern](#2-authentication--request-pattern)
3. [Processing Options (jsonOptions)](#3-processing-options-jsonoptions)
4. [Document Submission](#4-document-submission)
5. [Processing Status Retrieval](#5-processing-status-retrieval)
6. [JSON Output Retrieval (Basic / Detailed / Verbose)](#6-json-output-retrieval-basic--detailed--verbose)
7. [KVP Validation](#7-kvp-validation)
8. [Resource Deletion](#8-resource-deletion)
9. [Ontology Export](#9-ontology-export)
10. [Output Schemas](#10-output-schemas)
11. [Output Examples by Capability](#11-output-examples-by-capability)
12. [Response Codes](#12-response-codes)
13. [Error Codes](#13-error-codes)
14. [Response Envelope & Properties](#14-response-envelope--properties)
15. [Cross-Cutting Concerns & Notes](#15-cross-cutting-concerns--notes)

---

## 1. Platform Overview & Architecture

### Main Concepts

- **IBM Document Processing Extension (DPE)** — An add-on to existing capture solutions (e.g., Datacap) that uses AI and machine learning to classify, recognize, and extract metadata from business documents. Supports use cases such as accounts payable, claims processing, and any document-driven business process.
- **Document Processing Designer** — The design/training interface where you create a set of document types and related fields that comprise a DPE project. Uses AI and deep learning to learn document types. For each document type you designate which pieces of information to extract as data for downstream applications, and can apply tools to clean up and standardize extracted data.
- **Project** — A deployed DPE configuration containing trained document classification and data extraction models. Identified by a `projectId` (e.g., `test_project_r3cztn`), found in the web UI project link or browser address bar.
- **Analyzer** — A processing job created when a document is submitted. Each submission returns an `analyzerId` (UUID) used to track status and retrieve outputs.
- **Key-Value Pair (KVP)** — The fundamental extracted data unit: a key (field label) and its value, with coordinates, confidence scores, and optional mapping to a KeyClass in the ontology.
- **KeyClass** — An ontology-defined field class (e.g., `InvoiceNumber`, `ItemQuantity`) that KVPs can be tagged to. Carries metadata: sensitivity, mandatory flag, validators, data type, display name.
- **Ontology** — The project schema defining document types, fields, field type definitions, and validators. Exportable via the `/ontology` endpoint.
- **Document Classification** — AI-based assignment of a submitted document to a known document class (e.g., Invoice, Utility Bill, Bill of Lading), with confidence and alternate class candidates.
- **Semantic Normalization (SN)** — Cleans and standardizes extracted field values (e.g., normalizing names, addresses) beyond raw OCR text.
- **Table Extraction** — Detects tables and extracts line items as `ComplexKVPStructure` with nested `ValueList` rows and per-cell `Attributes`.
- **Open-Source ADP Connector (OSADP)** — A set of custom Datacap actions providing bidirectional integration from Datacap to DPE. Sends documents for classification/extraction and returns results to Datacap for verification and export. Includes the `SendPageToADP` action and an `ADPDemo` sample application.

### Key Capabilities Summary

| Capability | Endpoint | Purpose |
|---|---|---|
| Document Submission | `POST /v1/projects/{projectId}/analyzers` | Submit a PDF for classification, OCR, KVP & table extraction |
| Status Retrieval | `GET /v1/projects/{projectId}/analyzers/{analyzerId}` | Poll processing status (Completed / In Progress / Failed) |
| Basic JSON Output | `GET .../analyzers/{analyzerId}/json/basic` | Best KVP per key class (simplified) |
| Detailed JSON Output | `GET .../analyzers/{analyzerId}/json/detail` | All candidate KVPs per key class, ranked |
| Verbose JSON Output | `GET .../analyzers/{analyzerId}/json` | Full document attributes incl. OCR blocks, tables, pages |
| KVP Validation | `POST /v1/projects/{projectId}/validator` | Validate extracted KVPs against ontology validators |
| Resource Deletion | `DELETE /v1/projects/{projectId}/analyzers/{analyzerId}` | Delete a processed document and all related outputs |
| Ontology Export | `GET /v1/projects/{projectId}/ontology` | Export simplified ontology (doc types, fields, types) |

### Deployment & Licensing

- **Deployment:** Docker with Docker Swarm orchestration. Small footprint, can run on a single Linux machine. Stack deployed via the `dpedeploy` tool.
- **Licensing:** Sold per-page-processed. Deploy to any number/size of hardware environments with any number of users; purchase enough page licenses to cover pages processed. Usage monitored in DPE.
- **Web UI:** Shares the same domain as the API. The API base path is `/v1/projects/{projectId}`.
- **Max file size:** 250 MB. Only PDF format accepted for submission.

---

## 2. Authentication & Request Pattern

### Main Concepts

- **HTTP Basic Authentication** — All endpoints authenticate with HTTP Basic auth. Credentials are the same as the web UI login. Encode `username:password` in base64 and set the `Authorization: Basic {encoded}` header.
  ```bash
  echo -n 'myUser:myPassword' | base64
  # Output: bXlVc2VyOm15UGFzc3dvcmQ=
  ```
- **Bearer / Zen JWT Token** — The GET/DELETE/validator endpoint examples use `Authorization: Bearer {Zen_JWT}` (a "Zen token"). The 401 response references "Invalid apiKey or missing Zen token," indicating both an apiKey and a Zen token mechanism exist. The exact header format/token acquisition flow is not fully documented in the API pages.
- **One Document Per Call** — Only one document per API call is allowed.
- **Async Submit-Poll-Retrieve** — Submit returns `202 Accepted` with an `analyzerId`. Poll the status endpoint until `statusDetails` is `Completed` (or `Failed`). Then retrieve JSON output via the `/json` variants.
- **Webhook Notification** — Optional `webhookId` on submission to be notified when processing completes.

### Prerequisites

1. DPE stack deployed successfully with all services running.
2. A project created via Document Processing Designer.
3. User credentials (username/password) to log in and open the project.

### URL Construction

| Component | Description |
|---|---|
| `hostname` | Hostname of the machine running the DPE stack (manager node if multi-machine) |
| `port` | HTTPS port chosen at deployment (default 443) |
| `projectId` | Project ID from the web UI (e.g., `test_project_r3cztn`) |
| `analyzerId` | UUID returned from `POST /analyzers` (e.g., `ac3afc50-2c52-11ec-b296-c35cda005f89`) |

Example base: `https://{hostname:port}/v1/projects/{projectId}/analyzers`

---

## 3. Processing Options (jsonOptions)

### Main Concepts

- **jsonOptions** — A comma-separated list of option codes passed in the document submission request body. Controls which processing capabilities run on the submitted document. The selected options determine which sections appear in the final JSON output.
- **Dependency Chain** — Some options require others to be selected. Violating dependencies produces invalid requests.

### Option Codes

| Option Code | Full Name | Conditions |
|---|---|---|
| `OCR` | Word-based optical character recognition | None (base option) |
| `KVP` | Key Value Pair extraction | None (base option) |
| `DC` | Document Classification | None (base option) |
| `HR` | Headers | Requires `OCR` |
| `TH` | Table Headers | Requires `OCR` |
| `CHAR` | Character-based optical character recognition | Requires `OCR` |
| `SN` | Semantic Normalization | Requires `DC` AND `KVP` |
| `MT` | Mandatory | Requires `SN` (which requires `DC` + `KVP`) |
| `DS` (aka `SHW`) | Document Segmentation | Requires `HR` (which requires `OCR`) |

### Dependency Graph

```
OCR ──┬── HR ──── DS (SHW)
      ├── TH
      └── CHAR

DC ──┐
      ├── SN ──── MT
KVP ─┘
```

### Analysis

The options model lets callers trade off processing time/cost against output richness. OCR, KVP, and DC are independent base capabilities. Headers, Table Headers, and Character-OCR build on OCR. Semantic Normalization (value cleanup) requires both classification and KVP extraction to be meaningful. Mandatory flagging requires semantic normalization context. Document Segmentation requires Headers (page structure). The `CHAR` option carries a warning: it causes high memory usage in the `postprocessing` pod and may cause out-of-memory errors — monitor and increase RAM if needed.

> **Note:** The example curl in the docs includes `CB` and `ST` in `jsonOptions` (e.g., `HR,DC,KVP,TH,OCR,SN,MT,CB,ST,DS`), which are not listed in the official supported-options table. Their meaning is undocumented. `DS` and `SHW` appear to be aliases for Document Segmentation.

---

## 4. Document Submission

### Main Concepts

- **Submit a PDF for Processing** — Sends a file for metadata, content analysis, and structure extraction. Returns `202 Accepted` with an `analyzerId` for tracking.
- **Near Real-Time Response** — DPE provides near real-time response; however, document size and system activity can result in response times longer than standard API timeouts.
- **docClass Override** — Optionally specify a document class symbolic name to skip classification during processing.
- **Unicode Filenames** — If the filename contains Unicode characters, the calling code must handle them properly or error `CIWCA16001` is returned.

### Endpoint

| Method | Endpoint | Purpose |
|---|---|---|
| `POST` | `/v1/projects/{projectId}/analyzers` | Submit a PDF file for processing |

### Request Parameters

| Property | Location | Data Type | Required | Description |
|---|---|---|---|---|
| `Authorization` | Header | String (Basic) | Yes | Encoded user and password in HTTP Basic auth format |
| `projectId` | Path | String | Yes | Document Processing Project ID |
| `file` | Body (multipart) | File | Yes | The file to be processed. **Only PDF accepted.** Max 250 MB. |
| `responseType` | Body (multipart) | String | — | Response type, e.g., `json` |
| `jsonOptions` | Body (multipart) | String | — | Comma-separated list of option codes (e.g., `HR,DC,KVP,TH,OCR,SN,MT,DS`). See [§3](#3-processing-options-jsonoptions). |
| `docClass` | Body (multipart) | String | No | Document class symbolic name to skip classification |
| `uniqueId` | Body (multipart) | String | No | Customer unique ID included in final JSON output |
| `webhookId` | Body (multipart) | String | No | Webhook ID to be notified when processing completes |

### Example Request (curl)

```bash
curl -k -X POST https://{{hostname:port}}/v1/projects/{{projectId}}/analyzers \
  --header "Authorization: Basic {{encodedCredential}}" \
  --form "file=@./test.pdf" \
  --form "responseType=json" \
  --form "jsonOptions=HR,DC,KVP,TH,OCR,SN,MT,DS"
```

### Example Success Response (`202`)

```json
{
  "status": {
    "code": 202,
    "messageId": "CIWCA50000",
    "message": "Success"
  },
  "result": [
    {
      "status": {
        "code": 202,
        "messageId": "CIWCA11106",
        "message": "Content Analyzer request was created"
      },
      "data": {
        "message": "json processing request was created successful",
        "fileNameIn": "Legal Invoice 15.pdf",
        "analyzerId": "ac3afc50-2c52-11ec-b296-c35cda005f89",
        "type": ["json"]
      }
    }
  ]
}
```

### Analysis

The submission is multipart form data, not JSON — necessary for file upload. The `analyzerId` in the response is the critical handle for all subsequent operations (status, output retrieval, deletion). The `jsonOptions` field is the primary control over what processing runs and consequently what data appears in the output. The `docClass` override enables optimization when the document type is already known, skipping the classification step. Webhook support allows event-driven integration instead of polling.

---

## 5. Processing Status Retrieval

### Main Concepts

- **Poll for Status** — After submission, poll the status endpoint using the `analyzerId` until processing completes.
- **Status Values** — `statusDetails` returns `Completed`, `In Progress`, or `Failed`.

### Endpoint

| Method | Endpoint | Purpose |
|---|---|---|
| `GET` | `/v1/projects/{projectId}/analyzers/{analyzerId}` | Retrieve processing status |

### Request Parameters

| Property | Location | Data Type | Required | Description |
|---|---|---|---|---|
| `Authorization` | Header | String (Bearer) | Yes | Encoded credential / Zen JWT |
| `projectId` | Path | String | Yes | Document Processing Project ID |
| `analyzerId` | Path | String (UUID) | Yes | Unique identifier for each analyzer (from POST response) |

### Example Request (curl)

```bash
curl -k -X GET https://{{hostname:port}}/v1/projects/{{projectId}}/analyzers/{{analyzerId}} \
  --header "Authorization: Bearer {{encodedCredential}}"
```

### Analysis

This is a standard async polling endpoint. The docs don't specify recommended polling intervals. Combined with webhook support on submission, clients can choose between polling and event-driven notification. The `Failed` status should be paired with the `ErrorList` in the final JSON output to diagnose failures.

---

## 6. JSON Output Retrieval (Basic / Detailed / Verbose)

### Main Concepts

- **Three Output Variants** — The same processed document can be retrieved in three JSON formats of increasing detail:
  - **Basic** — Simplified; only the best (highest-confidence) key-value pair for each key class.
  - **Detailed** — All candidate key-value pairs for each key class, in ranked order.
  - **Verbose** — Full document attributes including OCR data (blocks, lines, words), table structures, page info, classification details, and all extracted information.
- **analyzerId = file_id** — The path parameter is `analyzerId` in the Requests page and `file_id` in the Outputs page; they refer to the same identifier.

### Endpoints

| Method | Endpoint | Purpose |
|---|---|---|
| `GET` | `/v1/projects/{projectId}/analyzers/{analyzerId}/json/basic` | Basic output — best KVP per key class |
| `GET` | `/v1/projects/{projectId}/analyzers/{analyzerId}/json/detail` | Detailed output — all candidate KVPs per key class, ranked |
| `GET` | `/v1/projects/{projectId}/analyzers/{analyzerId}/json` | Verbose output — full document attributes |

### Request Parameters

| Property | Location | Data Type | Required | Description |
|---|---|---|---|---|
| `Authorization` | Header | String (Bearer) | Yes | Encoded credential / Zen JWT |
| `projectId` | Path | String | Yes | Document Processing Project ID |
| `analyzerId` | Path | String (UUID) | Yes | Unique identifier for each analyzer (from POST response) |

### Example Request (curl — Verbose)

```bash
curl -k --location --request GET "https://{hostname:port}/v1/projects/{projectId}/analyzers/{analyzerId}/json" \
  --header "Authorization: Bearer $Zen_JWT"
```

### Analysis

The three-tier output model lets consumers choose the right balance of payload size vs. detail. Basic is ideal for downstream business applications that need only the final extracted values. Detailed is useful for verification UIs where a human reviewer may need to see alternative candidates. Verbose is necessary for auditing, debugging, or when raw OCR data (blocks/lines/words with coordinates) is needed. The schema differences between Basic/Detailed and Verbose are significant — see [§10](#10-output-schemas).

---

## 7. KVP Validation

### Main Concepts

- **Validate Extracted KVPs** — Post-processing step to validate extracted key-value pairs against validators defined on KeyClasses in the ontology.
- **Ontology JSON Input** — The request body is an Ontology JSON file containing Classification and pageList (with KVPTable). This is typically the verbose JSON output from a previous processing run.
- **Validator Results** — The validation adds/updates `ValidatorResult` (`"Pass"` or `"Fail"`) and `ValidatorFailures` (list of failed validators) on each KVP.

### Endpoint

| Method | Endpoint | Purpose |
|---|---|---|
| `POST` | `/v1/projects/{projectId}/validator` | Validate extracted KVPs against ontology |

### Request Parameters

| Property | Location | Data Type | Required | Description |
|---|---|---|---|---|
| `Authorization` | Header | String (Bearer) | Yes | Encoded credential / Zen JWT |
| `projectId` | Path | String | Yes | Document Processing Project ID |
| `Content-Type` | Header | String | Yes | `application/json` |
| `data` | Body (binary) | JSON | Yes | Ontology JSON file, including Classification (DocumentLanguage, PageTitle, DocumentClass, AlternateDocumentClass) and pageList (mainly KVPTable) |

### Example Request (curl)

```bash
curl -k -X POST https://{{hostname:port}}/v1/projects/{{projectId}}/validator \
  --header "Authorization: Bearer {{encodedCredential}}" \
  --header "Content-Type: application/json" \
  --data-binary "{{Path to JSON}}"
```

### Analysis

This endpoint enables a two-phase workflow: first extract (submit + retrieve JSON), then validate. This separation allows re-validation against updated ontology rules without re-processing the document. The input is the full verbose JSON output, meaning validators can access classification context and all page-level KVPs. Validators are defined per KeyClass in the ontology designer; the `ValidatorResult`/`ValidatorFailures` fields appear in the output only when a KVP is tagged to a KeyClass that has at least one validator.

---

## 8. Resource Deletion

### Main Concepts

- **Clean Up Processed Documents** — Deletes a processed document and all related outputs and parameters from the database.
- **Identified by analyzerId** — The same `analyzerId` used for status and output retrieval.

### Endpoint

| Method | Endpoint | Purpose |
|---|---|---|
| `DELETE` | `/v1/projects/{projectId}/analyzers/{analyzerId}` | Delete a processed document and all related outputs/parameters |

### Request Parameters

| Property | Location | Data Type | Required | Description |
|---|---|---|---|---|
| `Authorization` | Header | String (Bearer) | Yes | Encoded credential / Zen JWT |
| `projectId` | Path | String | Yes | Document Processing Project ID |
| `analyzerId` | Path | String (UUID) | Yes | Unique identifier for each analyzer (from POST response) |

### Example Request (curl)

```bash
curl -k -X DELETE https://{{hostname:port}}/v1/projects/{{projectId}}/analyzers/{{analyzerId}} \
  --header "Authorization: Bearer {{encodedCredential}}"
```

### Analysis

Resource cleanup is important for environments with data retention policies or storage constraints. The docs don't specify whether outputs are auto-deleted after a retention period (unlike some platforms that auto-expire results). The DELETE is scoped to a single `analyzerId`, so batch cleanup requires iterating over all documents.

---

## 9. Ontology Export

### Main Concepts

- **Export Project Schema** — Retrieves a simplified ontology with the definition of document types, fields, field type definitions, and validators.
- **Integration Use Case** — Example use case is integrating with IBM Business Automation Workflow, where the ontology informs downstream process definitions.

### Endpoint

| Method | Endpoint | Purpose |
|---|---|---|
| `GET` | `/v1/projects/{projectId}/ontology` | Export simplified ontology (document types, fields, field type definitions) |

### Request Parameters

| Property | Location | Data Type | Required | Description |
|---|---|---|---|---|
| `Authorization` | Header | String (Bearer) | Yes | Encoded credential / Zen JWT |
| `projectId` | Path | String | Yes | Document Processing Project ID |

### Example Request (curl)

```bash
curl -k -X GET https://{{hostname:port}}/v1/projects/{{projectId}}/ontology \
  --header "Authorization: Bearer {{encodedCredential}}"
```

### Analysis

The ontology export provides the contract between DPE's extraction models and downstream consumers. It defines what document types are recognized, what fields (KeyClasses) are extractable for each, their data types, sensitivity, mandatory flags, and validators. This is a read-only metadata endpoint — no document processing occurs. The exact JSON schema of the ontology export is not detailed in the docs.

---

## 10. Output Schemas

### 10.1 Basic & Detailed JSON Schema

#### Top-Layer Properties

| Property | Description |
|---|---|
| `documentId` | The ID of the document |
| `name` | Name of the document |
| `type` | Symbolic name of the document class |
| `typeName` | Display name of the document type |
| `dateTimeProcessed` | Time when the document starts to be processed |
| `fieldList` | List of simple KVPs, tables, and composites. **Basic:** best KVP extracted for a field. **Detailed:** all candidate KVPs extracted for a field in ranked order. |
| `errorList` | Errors in the document |

#### KVPTable Fields (Basic & Detailed)

| Property | Description |
|---|---|
| `keyClassName` | The key class symbolic name |
| `kvpId` | The ID of the KVP |
| `keyClassId` | The ID of the key class |
| `confidence` | The value confidence |
| `pageNumber` | The page number where the KVP was found |
| `valueStartX` | Start X coordinate for the Value in the document |
| `valueStartY` | Start Y coordinate for the Value in the document |
| `valueWidth` | Width of the Value |
| `valueHeight` | Height of the Value |
| `sensitive` | Sensitivity of the KVP as specified for the KeyClass in the ontology |
| `mandatory` | Whether the KVP is mandatory (True/False), per KeyClass in ontology |
| `displayName` | Display name of the key class |
| `dataType` | Data type from the object type |
| `key` | The key (field label) |
| `value` | The value of the field |
| `validatorResult` | *Optional.* Validation result (`"Pass"` or `"Fail"`). Present if KVP is tagged to a KeyClass with at least one validator. |
| `validatorFailures` | *Optional.* List of validators that failed. Present if `validatorResult` is `"Fail"`. |
| `originalValue` | The original value (before normalization) |

### 10.2 Verbose JSON Schema

#### Top-Layer Properties

| First-Layer | Sub-Layer | Description |
|---|---|---|
| `Classification` | | Information about the classification (file type, ID, confidence, etc.) |
| | `DocumentClass` | The classification of the document. Includes `Actual`, `ID`, `ClassConfidence`, `ClassMatch`, `ConfidenceThreshold`, `Code`, `DisplayName`, `template` attributes. |
| | `AlternateDocumentClass` | *Optional.* Lists all other document classes when `DC` option is set. |
| | `Model` | *Optional.* Published training model info when `DC` option is set. |
| `DocumentArrivalTime` | | Time the document was uploaded for processing |
| `DocumentExtension` | | Extension of the document |
| `DocumentName` | | Name of the document |
| `DocumentOCRConfidence` | | OCR confidence of the document |
| `ErrorList` | | Errors in the document |
| `ExtraInformation` | | Additional information (e.g., number of pages) |
| `KeyClassRankedList` | | Includes `KeyClassID`, `KeyClassName`, `KeyClassType`, `KVPRankedList` |
| `uniqueId` | | Unique identifier included in the final JSON output |
| `analyzerId` | | The ID of the document |
| `createDate` | | Date of creation of the document |
| `pageList` | | List of page attributes |
| | `KVPTable` | Key Value Pair Table |
| | `PageInfo` | Includes language, ID, position, DPI, confidence, etc. |
| | `BlockList` | OCR data for blocks: ID, position, `LineList` (contains `WordList`) |
| | `TableList` | OCR data for tables: ID, position, `RowList` (contains `CellList`) |
| | `allSystemTKVPs` | KVPs based on Natural Language extractors (included in KVPTable) |
| | `PageString_OCREngine` | Reading order of the entire text in the page |

#### Verbose KVPTable Fields

| Field | Description |
|---|---|
| `Key` | Text found in the document corresponding to the key of the KVP |
| `Value` | Text found in the document corresponding to the value of the KVP |
| `KeyStartX` | Starting X coordinate for the Key |
| `KeyStartY` | Starting Y coordinate for the Key |
| `KeyWidth` | Width of the Key |
| `KeyHeight` | Height of the Key |
| `KeyConfidence` | Confidence of the Key |
| `ValueStartX` | Start X coordinate for the Value |
| `ValueStartY` | Start Y coordinate for the Value |
| `ValueWidth` | Width of the Value |
| `ValueHeight` | Height of the Value |
| `ValueConfidence` | Confidence of the Value |
| `Sensitivity` | Sensitivity of the KVP per KeyClass in ontology |
| `EditedValue` | Whether the value was edited (true/false) |
| `KVPID` | The ID of the key-value pair |
| `ValueType` | The value format |
| `KeyClassConfidence` | Confidence for the KeyClass |
| `KeyClass` | KeyClass that maps to an existing KVP. Empty if no match in ontology. |
| `KeyClassID` | The ID of the key class |
| `Reserved2` | (undocumented) |
| `PageNumber` | Page number where the KVP was found |
| `Mandatory` | Whether the KVP is mandatory (True/False), per KeyClass in ontology |
| `OriginalValue` | The original value (before normalization) |
| `ValidatorResult` | *Optional.* `"Pass"` or `"Fail"`. Present if KVP tagged to KeyClass with validators. |
| `ValidatorFailures` | *Optional.* List of failed validators. Present if `ValidatorResult` is `"Fail"`. |
| `TableID` | The ID of the table |
| `ComplexKVPStructure` | Attributes for a complex KVP structure (tables/line items) |

#### ComplexKVPStructure (Table / Line Items)

```
Attributes
  …<simple KVP attributes>…
  TableGroupID        (table)
  ValueList           (table)
    LineItemID        (table)
    RowID             (table)
    Value             (table)
    ValueStartX       (table)
    ValueStartY       (table)
    ValueWidth        (table)
    ValueHeight       (table)
    ComplexKVPStructure
      Attributes      (recursive)
```

### Analysis

The schema design reflects a layered abstraction:
- **Basic/Detailed** use a simplified, consumer-friendly schema (`fieldList` with lowercase camelCase fields like `keyClassName`, `kvpId`, `confidence`). This is the integration-oriented view.
- **Verbose** uses a richer, PascalCase schema (`KVPTable`, `KeyStartX`, `KeyClassConfidence`) with full OCR layer (BlockList → LineList → WordList), table structures (TableList → RowList → CellList), page metadata (PageInfo), classification details, and the recursive `ComplexKVPStructure` for nested table line items.
- **ComplexKVPStructure** is the mechanism for representing tables: each table row is a `ValueList` entry with `LineItemID` and `RowID`, and each cell is an `Attribute` with its own Key/Value/confidence/coordinates. The structure is recursive, allowing nested tables.
- **Validator fields** (`ValidatorResult`, `ValidatorFailures`) appear conditionally, only when KVPs are mapped to KeyClasses with validators defined.
- **Coordinate system** — All coordinates (`StartX`, `StartY`, `Width`, `Height`) are in document pixel space, referencing the original page layout. Useful for highlight overlays in verification UIs.

---

## 11. Output Examples by Capability

### 11.1 OCR Output (Block → Line → Word)

```json
{
  "BlockList": [
    {
      "BlockID": "block_0",
      "BlockStartX": 1143,
      "BlockStartY": 142,
      "BlockWidth": 309,
      "BlockHeight": 57,
      "LineList": [
        {
          "LineID": "line_0",
          "LineStartX": 1143,
          "LineStartY": 142,
          "LineWidth": 308,
          "LineHeight": 57,
          "WordList": [
            {
              "WordID": "word_0",
              "WordStartX": 1143,
              "WordStartY": 142,
              "WordWidth": 308,
              "WordHeight": 57,
              "WordValue": "INVOICE",
              "WordOCRConfidence": "9999999",
              "WordCharN": 7,
              "BackColor": "ffffff",
              "bold": "true",
              "italic": "false",
              "underlined": "none",
              "WordFontSize": "8600",
              "WordFontSizeGroup": 1,
              "LineFontFace": "",
              "Type": "U"
            }
          ],
          "LineFontFace": "",
          "LineValue": "INVOICE"
        }
      ]
    }
  ]
}
```

**OCR fields:**
- `BlockList[]`: `BlockID`, `BlockStartX/Y`, `BlockWidth/Height`, `LineList[]`
- `LineList[]`: `LineID`, `LineStartX/Y`, `LineWidth/Height`, `WordList[]`, `LineFontFace`, `LineValue`
- `WordList[]`: `WordID`, `WordStartX/Y`, `WordWidth/Height`, `WordValue`, `WordOCRConfidence`, `WordCharN`, `BackColor`, `bold`, `italic`, `underlined`, `WordFontSize`, `WordFontSizeGroup`, `LineFontFace`, `Type`

### 11.2 Key-Value Pairs (KVP)

```json
{
  "KVPTable": [
    {
      "Key": "INVOICE No",
      "Value": "902580",
      "KeyStartX": 1144,
      "KeyStartY": 285,
      "KeyWidth": 173,
      "KeyHeight": 45,
      "KeyConfidence": 9.0,
      "ValueStartX": 1330,
      "ValueStartY": 285,
      "ValueWidth": 115,
      "ValueHeight": 44,
      "ValueConfidence": 9.0,
      "ValueType": "text",
      "Sensitivity": false,
      "KVPID": "d539bbd767f94330af0e7853a6df9b85",
      "KeyClass": "InvoiceNumber",
      "KeyClassID": "468fdb54-445c-48ad-aede-6cf6584d7fbb",
      "KeyClassConfidence": 100,
      "PageNumber": 0,
      "OriginalValue": "902580",
      "Reserved2": 6,
      "ValidatorResult": "Pass"
    }
  ]
}
```

### 11.3 Mandatory (MT) — Complex KVP

```json
{
  "ComplexKVPStructure": {
    "Attributes": [
      {
        "Key": "QUANTITY",
        "Value": "1,000",
        "KeyStartX": 150,
        "KeyStartY": 848,
        "KeyWidth": 139,
        "KeyHeight": 22,
        "KeyConfidence": 9.0,
        "ValueStartX": 183,
        "ValueStartY": 1174,
        "ValueWidth": 69,
        "ValueHeight": 25,
        "ValueConfidence": 9,
        "ValueType": "text",
        "Sensitivity": false,
        "Mandatory": false,
        "KeyClass": "ItemQuantity",
        "KeyClassID": "4360bcad-4b07-4c2e-998d-311ddb95bb66",
        "KeyClassConfidence": "High",
        "Reserved2": 3,
        "ValidatorResult": "Pass"
      }
    ]
  }
}
```

### 11.4 Document Classification (DC)

```json
{
  "Classification": {
    "DocumentClass": {
      "Actual": "Invoice",
      "ID": "b65d6596-71e0-4def-92e4-fc5e04ff4e66",
      "ClassMatch": "Medium",
      "Code": "",
      "DisplayName": "Invoice",
      "template": {
        "ID": "7973f5c5-51db-47d7-8a06-3273504f605f",
        "Name": "Template 3"
      }
    },
    "AlternateDocumentClass": [
      {
        "Name": "UtilityBill",
        "ID": "a2827178-33da-403b-aaa8-b0bb3b820959",
        "ClassMatch": "High",
        "DisplayName": "Utility Bill"
      },
      {
        "Name": "BillOfLading",
        "ID": "63842cbd-c858-4a5d-955c-7825edb7da25",
        "ClassMatch": "High",
        "DisplayName": "Bill of Lading"
      }
    ],
    "Model": {
      "ModelID": "dfd04722-fedf-4d6c-b4d9-67a90a7dde27",
      "ModelName": "Model 3",
      "PublishedDate": "2022-10-27 08:18:42.955000"
    }
  }
}
```

**Classification fields:**
- `Classification.DocumentClass`: `Actual`, `ID`, `ClassMatch`, `Code`, `DisplayName`, `template {ID, Name}`
- `Classification.AlternateDocumentClass[]`: `Name`, `ID`, `ClassMatch`, `DisplayName`
- `Classification.Model`: `ModelID`, `ModelName`, `PublishedDate`

### 11.5 Semantic Normalization (SN)

```json
{
  "KVPTable": [
    {
      "Key": "Master",
      "Value": "CAPT VALERIAN FUTKARADZE",
      "KeyClassConfidence": "Medium",
      "KeyClass": "Adjuster",
      "PageNumber": 0
    }
  ]
}
```

### 11.6 Table Line Items (Full)

```json
{
  "KVPTable": [
    {
      "Key": "NO_KEY_FOUND",
      "Value": "TABLE_ZONE",
      "Sensitivity": false,
      "KeyStartX": "",
      "KeyStartY": "",
      "KeyWidth": "",
      "KeyHeight": "",
      "ValueStartX": 302,
      "ValueStartY": 407,
      "ValueWidth": 1831,
      "ValueHeight": 1011,
      "KeyClass": "Invoice item table",
      "KeyClassConfidence": 60,
      "TableID": "vtable_0",
      "ComplexKVPStructure": {
        "Attributes": [
          {
            "Key": "NO_KEY_FOUND",
            "Value": "A001 XYZ, ... OAB 12 120",
            "Sensitivity": false,
            "KeyStartX": "",
            "KeyStartY": "",
            "KeyWidth": "",
            "KeyHeight": "",
            "ValueStartX": 302,
            "ValueStartY": 407,
            "ValueWidth": 1831,
            "ValueHeight": 1011,
            "KeyClass": "Main",
            "KeyClassConfidence": 60,
            "TableGroupID": 0,
            "ValueList": [
              {
                "LineItemID": 0,
                "RowID": "vrow_0",
                "Value": "A001 XYZ, Product A001 EMAIL: abc@ca.com PO: WB20202387 Sales Code: T4C 5 50",
                "ValueStartX": 302,
                "ValueStartY": 497,
                "ValueWidth": 1831,
                "ValueHeight": 307,
                "ComplexKVPStructure": {
                  "Attributes": [
                    {
                      "Key": "Item Code",
                      "Value": "A001",
                      "Sensitivity": false,
                      "Mandatory": 0,
                      "KeyStartX": 302,
                      "KeyStartY": 407,
                      "KeyWidth": 407,
                      "KeyHeight": 90,
                      "ValueStartX": 302,
                      "ValueStartY": 497,
                      "ValueWidth": 407,
                      "ValueHeight": 307,
                      "KeyClass": "Item ID",
                      "KeyClassConfidence": 75,
                      "ValueConfidence": 9,
                      "KeyConfidence": 9
                    },
                    {
                      "Key": "Product Desc",
                      "Value": "XYZ, Product A001 EMAIL: abc@ca.com PO: WB20202387 Sales Code: T4C",
                      "Sensitivity": false,
                      "Mandatory": 0,
                      "KeyStartX": 710,
                      "KeyStartY": 407,
                      "KeyWidth": 685,
                      "KeyHeight": 90,
                      "ValueStartX": 710,
                      "ValueStartY": 497,
                      "ValueWidth": 685,
                      "ValueHeight": 307,
                      "KeyClass": "",
                      "KeyClassConfidence": 57.14,
                      "ValueConfidence": 9,
                      "KeyConfidence": 9
                    }
                  ]
                }
              },
              {
                "LineItemID": 1,
                "RowID": "vrow_1",
                "Value": "A002 JOE, ... E5B 8 80",
                "ValueStartX": 302,
                "ValueStartY": 804,
                "ValueWidth": 1831,
                "ValueHeight": 306,
                "ComplexKVPStructure": {
                  "Attributes": [
                    {
                      "Key": "Item Code",
                      "Value": "A002",
                      "Sensitivity": false,
                      "Mandatory": 0,
                      "KeyStartX": 302,
                      "KeyStartY": 407,
                      "KeyWidth": 407,
                      "KeyHeight": 90,
                      "ValueStartX": 302,
                      "ValueStartY": 804,
                      "ValueWidth": 407,
                      "ValueHeight": 306,
                      "KeyClass": "Item ID",
                      "KeyClassConfidence": 75,
                      "ValueConfidence": 9,
                      "KeyConfidence": 9
                    },
                    {
                      "Key": "Product Desc",
                      "Value": "JOE, Product A002 EMAIL: joe@xyz.com PO: WB20208769 Sales Code: E5B",
                      "Sensitivity": false,
                      "Mandatory": 0,
                      "KeyStartX": 710,
                      "KeyStartY": 407,
                      "KeyWidth": 685,
                      "KeyHeight": 90,
                      "ValueStartX": 710,
                      "ValueStartY": 804,
                      "ValueWidth": 685,
                      "ValueHeight": 306,
                      "KeyClass": "",
                      "KeyClassConfidence": 57.14,
                      "ValueConfidence": 9,
                      "KeyConfidence": 9
                    }
                  ]
                }
              }
            ]
          }
        ]
      }
    }
  ]
}
```

**Table line item structure:**
- `KVPTable[]`: `Key`, `Value`, `Sensitivity`, `KeyStartX/Y/Width/Height`, `ValueStartX/Y/Width/Height`, `KeyClass`, `KeyClassConfidence`, `TableID`, `ComplexKVPStructure`
- `ComplexKVPStructure.Attributes[]`: ... `TableGroupID`, `ValueList[]`
- `ValueList[]`: `LineItemID`, `RowID`, `Value`, `ValueStartX/Y/Width/Height`, `ComplexKVPStructure` (recursive → `Attributes[]`)

---

## 12. Response Codes

### HTTP Status Codes

| Status | Meaning |
|---|---|
| `200` | Success. Request has been processed successfully. |
| `202` | Accepted. Content accepted for processing. |
| `400` | Invalid Request. Invalid parameter requests. |
| `401` | Unauthorized. Invalid apiKey or missing Zen token. |
| `404` | Failed. Content not found. |
| `500` | Internal Server Error. Unknown internal server error. |
| `502` | Bad Gateway. Server is down. |
| `503` | Unavailable service. Server is overloaded with requests. |
| `504` | Gateway timeout. Did not receive a timely response from server. |

### Analysis

- `202` is the expected response for document submission (async acceptance).
- `200` is for synchronous success (status retrieval, JSON output, ontology export, validation).
- `401` confirms dual auth mechanisms: an `apiKey` and a `Zen token` (JWT). The exact header names for each are not explicitly documented, but examples show `Authorization: Basic` for submission and `Authorization: Bearer` for other endpoints.
- `404` applies when an `analyzerId` or `projectId` doesn't exist.
- `5xx` errors relate to the DPE stack health (gateway, overload, timeout).

---

## 13. Error Codes

### Main Concepts

- **ErrorList in JSON Output** — When errors occur during processing, the final JSON output contains an `ErrorList` key with per-page error details.
- **Error Code Ranges** — Errors are categorized by processing stage, each with a `CIWCA` prefix and a numeric range.

### Error Code Ranges

| Error Category | Code Range |
|---|---|
| Classification Errors | `CIWCA24100` – `CIWCA24199` |
| Headers Errors | `CIWCA24200` – `CIWCA24299` |
| Extraction Errors | `CIWCA24300` – `CIWCA24399` |

### Additional Error Codes

| Error Code | Context |
|---|---|
| `CIWCA16001` | Calling code does not handle Unicode characters in the filename |
| `CIWCA50000` | Success status message ID |
| `CIWCA11106` | "Content Analyzer request was created" status message ID |

### Example Error Response

```json
{
  "ErrorList": [
    {
      "page_number": 1,
      "errors": [
        {
          "error_code": "CIWCA24100",
          "message": "Failure in docclassification"
        }
      ]
    }
  ]
}
```

### ErrorList Structure

| Field | Type | Description |
|---|---|---|
| `ErrorList` | array | Top-level error container |
| `ErrorList[].page_number` | integer | Page number on which the error occurred |
| `ErrorList[].errors` | array | List of errors for that page |
| `ErrorList[].errors[].error_code` | string | `CIWCA`-prefixed error code |
| `ErrorList[].errors[].message` | string | Human-readable error message |

### Analysis

The error model is page-scoped — errors are reported per page, allowing partial success (some pages process correctly while others fail). The three documented ranges cover the main processing stages (classification, headers, extraction). However, the docs only enumerate the ranges and one example code (`CIWCA24100`); individual codes within each range, their specific messages, and corrective actions are not documented. The `CIWCA16001` Unicode filename error is the only other specific code documented, appearing on the Requests page rather than the Error codes page.

---

## 14. Response Envelope & Properties

### Main Concepts

- **Standard Response Envelope** — All API responses use a structured envelope with `status` and `result` (or `error`) top-level keys.

### Response Properties

| Property | Data Type | Description |
|---|---|---|
| `code` | integer | The status code associated with the response |
| `messageId` | string | Document Processing message code (e.g., `CIWCA50000`) |
| `message` | string | The status message associated with the response |
| `analyzerId` | string | The ID associated with the API call; used in subsequent requests |
| `fileNameIn` | string | Name of the uploaded file |
| `type` | array | The type(s) of file that is generated (e.g., `["json"]`) |
| `errorId` | string | Document Processing error code |
| `explanation` | string | Explanation for the error |
| `action` | string | The action needed to correct the error |

### Error Response Example

```json
{
  "error": "invalid_token",
  "error_description": "access token is missing or invalid."
}
```

### Analysis

The response envelope is asymmetric: success responses use `{status, result[{status, data}]}` while error responses use `{error, error_description}`. The `result` is an array, suggesting the envelope was designed to potentially support batch results, though the API enforces one document per call. The `messageId` field (`CIWCAxxxxx`) provides a machine-readable code for status/error handling. The `analyzerId` in the response data is the key linkage between submission and all downstream operations.

---

## 15. Cross-Cutting Concerns & Notes

### Authentication Discrepancy

The parameter tables for all endpoints reference "HTTP Basic authentication header," but the curl examples show two patterns:
- **POST /analyzers (submit):** `Authorization: Basic {encodedCredential}`
- **All other endpoints:** `Authorization: Bearer {encodedCredential}` or `Authorization: Bearer $Zen_JWT`

This suggests the submission endpoint may accept Basic auth directly, while other endpoints expect a Bearer/JWT token (possibly obtained from an authentication flow not documented in these pages). The 401 error message ("Invalid apiKey or missing Zen token") confirms multiple auth mechanisms exist.

### jsonOptions Discrepancy

The parameter table lists supported options as: `ocr, dc, kvp, sn, hr, th, mt, ds, char`. The example curl uses `HR,DC,KVP,TH,OCR,SN,MT,CB,ST,DS` — including undocumented `CB` and `ST` options. The options appear case-insensitive. `DS` and `SHW` are documented as aliases for Document Segmentation. The meaning of `CB` and `ST` is not explained.

### Path Parameter Naming

The Requests page uses `{analyzerId}` in JSON output paths, while the Outputs page uses `<file_id>`. These refer to the same identifier — the `analyzerId` returned from the `POST /analyzers` response.

### CHAR Option Warning

The `char` (character-based OCR) option causes high memory usage in the `postprocessing` pod and may cause out-of-memory errors. Monitor and increase RAM for `postprocessing` pods if this option is needed.

### Unicode Filename Handling

If the submitted file's name contains Unicode characters, the calling code must handle them properly. Failure to do so results in error code `CIWCA16001`.

### Max File Size & Format

- Maximum accepted file size: **250 MB**
- Only **PDF** format is accepted for document submission

### No Auto-Expiration Documented

Unlike some document processing platforms, the DPE API docs do not mention automatic result expiration. Resources persist until explicitly deleted via the `DELETE` endpoint.

### Undocumented Details

The following are referenced but not fully documented in the available pages:
- Exact auth header names and token acquisition flow
- The `responseType` body field's accepted values (only `json` shown)
- The `webhookId` mechanism — how webhooks are configured and what payload they receive
- The ontology export JSON schema
- Individual error codes within each range and their corrective actions
- Recommended polling intervals for status checks
- Rate limits / throttling policies

---

## Endpoint Reference (Quick Summary)

| # | Method | Endpoint | Auth | Purpose |
|---|---|---|---|---|
| 1 | `POST` | `/v1/projects/{projectId}/analyzers` | Basic | Submit PDF for processing |
| 2 | `GET` | `/v1/projects/{projectId}/analyzers/{analyzerId}` | Bearer | Get processing status |
| 3 | `GET` | `/v1/projects/{projectId}/analyzers/{analyzerId}/json/basic` | Bearer | Download basic JSON output |
| 4 | `GET` | `/v1/projects/{projectId}/analyzers/{analyzerId}/json/detail` | Bearer | Download detailed JSON output |
| 5 | `GET` | `/v1/projects/{projectId}/analyzers/{analyzerId}/json` | Bearer | Download verbose JSON output |
| 6 | `POST` | `/v1/projects/{projectId}/validator` | Bearer | Validate extracted KVPs |
| 7 | `DELETE` | `/v1/projects/{projectId}/analyzers/{analyzerId}` | Bearer | Delete document & resources |
| 8 | `GET` | `/v1/projects/{projectId}/ontology` | Bearer | Export project ontology |