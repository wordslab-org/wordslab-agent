# Azure AI Language API Analysis — Text & Conversation Capabilities

> **Base URL:** `https://{resource-name}.cognitiveservices.azure.com` | **Docs:** `https://learn.microsoft.com/en-us/azure/ai-services/language-service/` | **Auth:** Header `Ocp-Apim-Subscription-Key: {resource-key}` (Entra ID / Managed Identity also supported)
> **SDKs:** `Azure.AI.TextAnalytics` (.NET / Python / Java / JavaScript) for text analysis; `Azure.AI.Language.Conversations` (.NET / Python) for conversation analysis; `Azure.AI.Language.QuestionAnswering` for QA
> **Description:** Azure AI Language is a cloud-based suite of NLP capabilities exposed through a small number of REST endpoints under `/language/...`. Capabilities are split between **Core capabilities** (NER, Custom NER, PII detection, Text Analytics for Health, Language Detection) and **Legacy capabilities** (Sentiment Analysis & Opinion Mining, Key Phrase Extraction, Entity Linking, Summarization, Custom Text Classification, Conversation Language Understanding, Custom Question Answering, Orchestration Workflow). Two fully **deprecated** standalone services — **LUIS** and **QnA Maker** — are being replaced by CLU and Custom Question Answering respectively. Several legacy capabilities carry a **retirement date of March 31, 2029**, after which Microsoft recommends migrating to Microsoft Foundry generative models.

---

## Table of Contents

1. [Platform Overview & Core Concepts](#1-platform-overview--core-concepts)
2. [API Surfaces — Shared Endpoints & Operation Patterns](#2-api-surfaces--shared-endpoints--operation-patterns)
3. [Language Detection](#3-language-detection)
4. [Named Entity Recognition (NER)](#4-named-entity-recognition-ner)
5. [Custom Named Entity Recognition (Custom NER)](#5-custom-named-entity-recognition-custom-ner)
6. [PII Detection — Text](#6-pii-detection--text)
7. [PII Detection — Conversation](#7-pii-detection--conversation)
8. [PII Detection — Document-based](#8-pii-detection--document-based)
9. [Text Analytics for Health](#9-text-analytics-for-health)
10. [Sentiment Analysis & Opinion Mining](#10-sentiment-analysis--opinion-mining)
11. [Key Phrase Extraction](#11-key-phrase-extraction)
12. [Entity Linking](#12-entity-linking)
13. [Summarization](#13-summarization)
14. [Custom Text Classification](#14-custom-text-classification)
15. [Conversation Language Understanding (CLU)](#15-conversation-language-understanding-clu)
16. [Custom Question Answering (CQA)](#16-custom-question-answering-cqa)
17. [Orchestration Workflow](#17-orchestration-workflow)
18. [LUIS — Language Understanding Intelligent Service (Deprecated)](#18-luis--language-understanding-intelligent-service-deprecated)
19. [QnA Maker (Deprecated)](#19-qnamaker-deprecated)
20. [Capability Summary & Cross-Reference](#20-capability-summary--cross-reference)

---

## 1. Platform Overview & Core Concepts

### Main Concepts

Azure AI Language organizes its NLP features around a few shared abstractions:

- **Language resource** — An Azure resource (provisioned via the portal, ARM, or CLI) that provides an **endpoint URL** and a **key** (`Ocp-Apim-Subscription-Key`). All capabilities are accessed through this single resource. A subset of custom features (Custom NER, Custom Text Classification, CLU, CQA) additionally require an **Azure Blob Storage** account for training data and/or an **Azure AI Search** resource (for CQA).
- **Document** — The unit of input for text-analysis operations. Always carries `id`, `text`, and `language` (language optional for some features; required or auto-detected for others).
- **Kind** — A discriminator string in request bodies that selects which analysis to run on the shared `/language/:analyze-text` and `/language/analyze-text/jobs` endpoints (e.g. `EntityRecognition`, `PiiEntityRecognition`, `SentimentAnalysis`, `Healthcare`, `ExtractiveSummarization`). The response echoes a corresponding `*Results` (sync) or `*LROResults` (async) kind.
- **Model version** — Each prebuilt feature is backed by a versioned model. `parameters.modelVersion` accepts `"latest"` or a specific version string (e.g. `"2023-09-01"`). Newer preview models introduce a richer `type`/`tags`/`metadata` entity structure replacing the older `category`/`subcategory` structure.
- **Project** — The container for custom features (Custom NER, Custom Text Classification, CLU, Orchestration Workflow, CQA). A project holds a schema, labeled data, trained models, and deployments. Projects are managed through an **Authoring API** (`/language/authoring/...`) and queried through a **Runtime API** (`/language/analyze-text/jobs`, `/language/:analyze-conversations`, `/language/:query-knowledgebases`).
- **Deployment** — A named, deployed version of a trained custom model (e.g. `production`, `test`). Runtime requests reference both `projectName` and `deploymentName`. Custom model deployments expire ~18 months after their training-config version is deployed.
- **Long-Running Operation (LRO)** — The async job pattern used by all batch / custom / document operations. A `POST` returns `202 Accepted` with an `operation-location` response header containing the job URL; clients poll `GET` on that URL until `status: "succeeded"`. Results are retained for 24 hours (text) or per job `expirationDateTime` (custom/document).
- **Entity** — A detected span. Common fields: `text`, `offset`, `length`, `confidenceScore`, `category`/`subcategory` (GA) or `type`/`tags`/`metadata` (preview).
- **Conversation item** — The unit of input for conversation features. Carries `id`, `participantId`, `text`, and (for transcripts) `lexical`/`itn`/`maskedItn` speech-format variants plus `audioTimings[]`.

### Capability Classification

| Tier | Capabilities | Notes |
|------|--------------|-------|
| **Core** | Language Detection, NER, Custom NER, PII (text/conversation/document), Text Analytics for Health | Recommended for new projects; aligned with current platform architecture |
| **Legacy** | Sentiment & Opinion Mining, Key Phrase Extraction, Entity Linking, Summarization, Custom Text Classification, CLU, CQA, Orchestration Workflow | Most carry a **March 31, 2029** (or **Sep 1, 2028** for Entity Linking) retirement date; migration to Microsoft Foundry generative models recommended |
| **Deprecated** | LUIS, QnA Maker | LUIS retired **March 31, 2026**; QnA Maker retired **October 31, 2025**. Replaced by CLU and CQA respectively |

### Platform Architecture

```
Azure AI Language resource (endpoint + key)
│
├── Prebuilt text analysis (stateless)
│   ├── Synchronous:  POST /language/:analyze-text          (kind selects feature)
│   └── Asynchronous: POST /language/analyze-text/jobs      (batch / custom / health / summarization)
│                      GET  /language/analyze-text/jobs/{jobId}
│
├── Conversation analysis
│   ├── Synchronous:  POST /language/:analyze-conversations (CLU prediction, Orchestration)
│   └── Asynchronous: POST /language/analyze-conversations/jobs  (Conversation PII, Conversation Summarization)
│                      GET  /language/analyze-conversations/jobs/{jobId}
│
├── Native document processing
│   └── Asynchronous: POST /language/analyze-documents/jobs (Document PII, Document Summarization)
│                      GET  /language/analyze-documents/jobs/{jobId}
│
├── Question answering runtime
│   └── Synchronous:  POST /language/:query-knowledgebases  (deployed KB)
│                      POST /language/:query-text           (prebuilt, no project)
│
└── Custom authoring (all custom projects)
    └── /language/authoring/analyze-text/projects/{name}/...        (Custom NER, Custom Text Classification)
        /language/authoring/analyze-conversations/projects/{name}/... (CLU, Orchestration Workflow)
        /language/authoring/query-knowledgebases/projects/{name}/...  (CQA)
        Operations: :import, :train, deployments/{name} (PUT), swap-deployments, delete, get status
```

---

## 2. API Surfaces — Shared Endpoints & Operation Patterns

### Common Headers

| Header | Value | Required |
|--------|-------|----------|
| `Ocp-Apim-Subscription-Key` | `{resource-key}` | Yes (key auth) |
| `Content-Type` | `application/json` | Yes |
| `Authorization` | `Bearer {Entra token}` | Alternative to key (Entra ID) |

### Endpoint Catalog

| Endpoint | Method(s) | Pattern | Used by |
|----------|-----------|---------|---------|
| `/language/:analyze-text` | POST | Synchronous single-task | Language Detection, NER, PII (text), Sentiment, Key Phrase, Entity Linking |
| `/language/analyze-text/jobs` | POST → GET | Async LRO | NER (batch), Custom NER (runtime), Text Analytics for Health, Summarization (text), Custom Text Classification (runtime) |
| `/language/:analyze-conversations` | POST | Synchronous | CLU prediction, Orchestration Workflow prediction |
| `/language/analyze-conversations/jobs` | POST → GET | Async LRO | Conversation PII, Conversation Summarization |
| `/language/analyze-documents/jobs` | POST → GET | Async LRO | Document PII, Document Summarization |
| `/language/:query-knowledgebases` | POST | Synchronous | Custom Question Answering (deployed KB) |
| `/language/:query-text` | POST | Synchronous | Custom Question Answering (prebuilt, no project) |
| `/language/authoring/analyze-text/projects/{name}/...` | POST/GET/PUT/DELETE | Authoring | Custom NER, Custom Text Classification |
| `/language/authoring/analyze-conversations/projects/{name}/...` | POST/GET/PUT/DELETE | Authoring | CLU, Orchestration Workflow |
| `/language/authoring/query-knowledgebases/projects/{name}/...` | POST/GET/PUT/DELETE | Authoring | Custom Question Answering |

### Operation Patterns

**Synchronous (single-shot):**
```
POST /language/:analyze-text?api-version={version}
Body: { "kind": "{Feature}", "parameters": {...}, "analysisInput": { "documents": [...] } }
→ 200 OK with results inline
```

**Asynchronous (long-running):**
```
POST /language/analyze-text/jobs?api-version={version}
Body: { "displayName": "...", "analysisInput": { "documents": [...] }, "tasks": [ { "kind": "...", "taskName": "...", "parameters": {...} } ] }
→ 202 Accepted, response header "operation-location": {endpoint}/language/analyze-text/jobs/{jobId}?api-version={version}

GET {operation-location}
→ 200 OK, poll until "status": "succeeded"; results in tasks.items[].results
```

**Custom project lifecycle:**
```
POST .../projects/{name}/:import    → 202 + operation-location (import job)
POST .../projects/{name}/:train     → 202 + operation-location (train job)
PUT  .../projects/{name}/deployments/{deploymentName}  → 202 + operation-location (deploy job)
POST /language/analyze-text/jobs (or /:analyze-conversations) with projectName + deploymentName  → prediction
```

### API Versions

| Version | Type | Used by |
|---------|------|---------|
| `2021-10-01` | GA | Custom Question Answering runtime |
| `2022-05-01` | GA | NER, PII (text), Key Phrase, Entity Linking, Custom NER, Custom Text Classification |
| `2022-05-15-preview` | Preview | Text Analytics for Health |
| `2023-04-01` | GA | CLU, Orchestration Workflow |
| `2023-04-15-preview` | Preview | NER (types/tags), Sentiment, Conversation PII |
| `2023-11-15-preview` | Preview | Language Detection |
| `2024-05-01` | GA | Conversation PII |
| `2024-11-15-preview` | Preview | Document PII, Document Summarization, Conversation PII redaction policies |
| `2025-11-15-preview` | Preview | Text PII redaction policies |
| `2026-05-01` | GA | Text PII, Document PII |

### `stringIndexType`

Defines the offset/length encoding in responses. Values:
- `Utf16CodeUnit` — default for most text-analysis features (JavaScript/UTF-16 compatible)
- `TextElement_V8` — used by CLU prediction (Unicode grapheme clusters, per ICU/JS V8)

---

## 3. Language Detection

> **Tier:** Core | **Docs:** `language-service/language-detection/overview` | **Deprecated:** No

### Main Concepts

Automatically determines the primary language of input text across 100+ languages. Returns the dominant language with an ISO 639-1 code, a human-readable name, and a confidence score. For select languages also returns **script** name and ISO 15924 **script code** (e.g. distinguishes Latin/Cyrillic variants of Kazakh). Supports an optional **country/region hint** (ISO 3166-1 alpha-2) to disambiguate ambiguous content. Synchronous only; no customization. Docker container deployment supported.

### API Functions

| Operation | Method | Endpoint | `kind` (request → response) |
|-----------|--------|----------|------------------------------|
| Detect language | POST | `/language/:analyze-text?api-version=2023-11-15-preview` | `LanguageDetection` → `LanguageDetectionResults` |

### Request Parameters

```json
{
  "kind": "LanguageDetection",
  "parameters": { "modelVersion": "latest" },
  "analysisInput": {
    "documents": [
      { "id": "1", "text": "Ce document est rédigé en Français.", "countryHint": "us" }
    ]
  }
}
```

| Parameter | Type | Description |
|-----------|------|-------------|
| `kind` | string | `"LanguageDetection"` |
| `parameters.modelVersion` | string | `"latest"` or specific version |
| `analysisInput.documents[]` | array | Input documents |
| `documents[].id` | string | Unique document id |
| `documents[].text` | string | Input text |
| `documents[].countryHint` | string | ISO 3166-1 alpha-2 country/region hint (optional) |

### Response Fields

```json
{
  "kind": "LanguageDetectionResults",
  "results": {
    "documents": [
      {
        "id": "1",
        "detectedLanguage": {
          "name": "French",
          "iso6391Name": "fr",
          "confidenceScore": 1.0,
          "script": "Latin",
          "scriptCode": "Latn"
        },
        "warnings": []
      }
    ],
    "errors": [],
    "modelVersion": "2023-12-01"
  }
}
```

| Field | Type | Description |
|-------|------|-------------|
| `results.documents[].detectedLanguage.name` | string | Human-readable language name |
| `results.documents[].detectedLanguage.iso6391Name` | string | ISO 639-1 code |
| `results.documents[].detectedLanguage.confidenceScore` | float | 0.0–1.0 |
| `results.documents[].detectedLanguage.script` | string | Script name (select languages) |
| `results.documents[].detectedLanguage.scriptCode` | string | ISO 15924 script code (select languages) |
| `results.modelVersion` | string | Model version used |

---

## 4. Named Entity Recognition (NER)

> **Tier:** Core | **Docs:** `language-service/named-entity-recognition/overview` | **Deprecated:** No (recommended replacement for Entity Linking)

### Main Concepts

Identifies and categorizes entities in unstructured text into predefined categories: people (`Person`, `PersonType`), places (`Location`), organizations (`Organization`), events, products, quantities, dates/times (`DateTime`), addresses, emails, URLs, phone numbers, and more (full list in `concepts/named-entity-categories`). Uses a preset list of recognized entities — no model training required. Available synchronously (stateless) and asynchronously (batch; results retained 24 hours).

The newer preview model replaces the older `category`/`subcategory` structure with an **entity types and tags** hierarchy. Clients can filter returned entities via `inclusionList`/`exclusionList` on types or tags.

### API Functions

| Operation | Method | Endpoint | `kind` (request → response) |
|-----------|--------|----------|------------------------------|
| NER (sync) | POST | `/language/:analyze-text?api-version=2022-05-01` (GA) or `2023-04-15-preview` (preview) | `EntityRecognition` → `EntityRecognitionResults` |
| NER (async batch) | POST → GET | `/language/analyze-text/jobs?api-version={version}` | `EntityRecognition` (task kind) → `EntityRecognitionResults` |

### Request Parameters

```json
{
  "kind": "EntityRecognition",
  "parameters": {
    "modelVersion": "latest",
    "inclusionList": ["Person", "Organization"],
    "exclusionList": [],
    "stringIndexType": "Utf16CodeUnit"
  },
  "analysisInput": {
    "documents": [
      { "id": "1", "language": "en", "text": "Microsoft was founded by Bill Gates..." }
    ]
  }
}
```

| Parameter | Type | Description |
|-----------|------|-------------|
| `kind` | string | `"EntityRecognition"` |
| `parameters.modelVersion` | string | `"latest"` or specific version (e.g. `"2023-09-01"`) |
| `parameters.inclusionList` | string[] | (Preview) Only return entities of these types/tags |
| `parameters.exclusionList` | string[] | (Preview) Exclude these types/tags |
| `parameters.stringIndexType` | string | `"Utf16CodeUnit"` (default) |
| `analysisInput.documents[]` | array | Each: `id`, `language`, `text` |

### Response Fields

```json
{
  "kind": "EntityRecognitionResults",
  "results": {
    "documents": [
      {
        "id": "1",
        "entities": [
          {
            "text": "Microsoft",
            "category": "Organization",
            "subcategory": null,
            "offset": 0,
            "length": 9,
            "confidenceScore": 0.96,
            "type": "Organization",
            "tags": [ { "name": "Organization", "confidenceScore": 0.96 } ],
            "metadata": { "metadataKind": "EntityMetadata" }
          }
        ],
        "warnings": []
      }
    ],
    "errors": [],
    "modelVersion": "2023-09-01"
  }
}
```

| Field | Type | Description |
|-------|------|-------------|
| `entities[].text` | string | Matched text |
| `entities[].category` / `subcategory` | string | (GA) Entity category/subcategory |
| `entities[].type` | string | (Preview) Entity type (replaces category) |
| `entities[].tags[]` | array | (Preview) `{ name, confidenceScore }` |
| `entities[].metadata` | object | (Preview) Typed metadata (`metadataKind`) |
| `entities[].offset`, `length` | int | Character span |
| `entities[].confidenceScore` | float | 0.0–1.0 (GA); `score` in some preview versions |
| `results.modelVersion` | string | Model version used |

---

## 5. Custom Named Entity Recognition (Custom NER)

> **Tier:** Core | **Docs:** `language-service/custom-named-entity-recognition/overview` | **Deprecated:** No

### Main Concepts

A cloud-based API that trains ML models to extract **domain-specific entities** from unstructured text using your own labeled data and entity schema (e.g. contract clauses, financial fields). Requires a Language resource linked to an Azure Blob Storage account (with the "Custom NER" feature enabled). Project lifecycle: define schema → label data → train → evaluate → deploy → extract. Managed via Microsoft Foundry (web) or the Authoring REST API; queried via the shared Runtime `/language/analyze-text/jobs` endpoint.

### API Functions

**Authoring:**

| Operation | Method | Endpoint |
|-----------|--------|----------|
| Import project | POST | `/language/authoring/analyze-text/projects/{projectName}/:import?api-version=2022-05-01` |
| Import job status | GET | `/language/authoring/analyze-text/projects/{projectName}/import/jobs/{jobId}?api-version={version}` |
| Train model | POST | `/language/authoring/analyze-text/projects/{projectName}/:train?api-version={version}` |
| Train job status | GET | `/language/authoring/analyze-text/projects/{projectName}/train/jobs/{jobId}?api-version={version}` |
| Deploy model | PUT | `/language/authoring/analyze-text/projects/{projectName}/deployments/{deploymentName}?api-version={version}` |
| Deploy job status | GET | `/language/authoring/analyze-text/projects/{projectName}/deployments/{deploymentName}/jobs/{jobId}?api-version={version}` |
| Delete project | DELETE | `/language/authoring/analyze-text/projects/{projectName}?api-version={version}` |

**Runtime:**

| Operation | Method | Endpoint | `kind` (task → result) |
|-----------|--------|----------|-------------------------|
| Extract entities | POST → GET | `/language/analyze-text/jobs?api-version={version}` | `CustomEntityRecognition` → `EntityRecognitionLROResults` |

### Request Parameters

**Import body:**
```json
{
  "projectFileVersion": "2022-05-01",
  "stringIndexType": "Utf16CodeUnit",
  "metadata": {
    "projectName": "{PROJECT-NAME}",
    "projectKind": "CustomEntityRecognition",
    "description": "...",
    "language": "en",
    "multilingual": false,
    "storageInputContainerName": "{CONTAINER-NAME}",
    "settings": {}
  },
  "assets": {
    "projectKind": "CustomEntityRecognition",
    "entities": [ { "category": "Entity1" } ],
    "documents": [
      {
        "location": "{DOCUMENT-NAME}",
        "language": "en",
        "dataset": "Train",
        "entities": [
          { "regionOffset": 0, "regionLength": 100, "labels": [ { "category": "Entity1", "offset": 0, "length": 10 } ] }
        ]
      }
    ]
  }
}
```

**Train body:**
```json
{
  "modelLabel": "{MODEL-NAME}",
  "trainingConfigVersion": "{VERSION}",
  "evaluationOptions": {
    "kind": "percentage",
    "trainingSplitPercentage": 80,
    "testingSplitPercentage": 20
  }
}
```

**Deploy body:**
```json
{ "trainedModelLabel": "{MODEL-NAME}" }
```

**Runtime submit body:**
```json
{
  "displayName": "Extract entities",
  "analysisInput": {
    "documents": [ { "id": "1", "language": "en", "text": "..." } ]
  },
  "tasks": [
    {
      "kind": "CustomEntityRecognition",
      "taskName": "Entity extraction",
      "parameters": { "projectName": "{PROJECT-NAME}", "deploymentName": "{DEPLOYMENT-NAME}" }
    }
  ]
}
```

### Response Fields

Runtime results (`tasks.items[]` with `kind: "EntityRecognitionLROResults"`):
- `results.documents[].entities[]`: `category`, `confidenceScore`, `length`, `offset`, `text`, `subcategory?`
- `results.documents[].id`, `warnings[]`
- `results.errors[]`, `results.modelVersion`

Train status: `result.modelLabel`, `result.trainingConfigVersion`, `result.estimatedEndDateTime`, `result.trainingStatus.percentComplete`, `result.evaluationStatus`.

---

## 6. PII Detection — Text

> **Tier:** Core | **Docs:** `personally-identifiable-information/text-pii-overview` | **Deprecated:** No

### Main Concepts

Detects and redacts sensitive information (PII and PHI) in raw **text strings** across predefined PII categories (person names, organizations, addresses, emails, phone numbers, SSN, passport, financial data, etc.). Optimized for **synchronous**, low-latency, inline processing in app/data pipelines. Returns both the detected entities and a `redactedText` output. Configurable via `piiCategories` filters and `redactionPolicies` (character mask, entity mask, no mask, synthetic replacement). Sync mode is stateless (no data stored). Async results retained 24 hours.

### API Functions

| Operation | Method | Endpoint | `kind` (request → response) |
|-----------|--------|----------|------------------------------|
| PII redaction (sync) | POST | `/language/:analyze-text?api-version=2022-05-01` (GA) / `2025-11-15-preview` / `2026-05-15-preview` | `PiiEntityRecognition` → `PiiEntityRecognitionResults` |
| PII redaction (async) | POST → GET | `/language/analyze-text/jobs?api-version={version}` | `PiiEntityRecognition` → `PiiEntityRecognitionLROResults` |

### Request Parameters

```json
{
  "kind": "PiiEntityRecognition",
  "parameters": {
    "modelVersion": "latest",
    "piiCategories": ["Person", "Organization"],
    "redactionPolicies": [
      { "policyKind": "CharacterMask", "redactionCharacter": "*", "defaultRedactionPolicy": true }
    ],
    "confidenceScoreThreshold": { "default": 0.5 },
    "disableEntityValidation": false
  },
  "analysisInput": {
    "documents": [ { "id": "1", "language": "en", "text": "Call me at 555-555-5555." } ]
  }
}
```

| Parameter | Type | Description |
|-----------|------|-------------|
| `kind` | string | `"PiiEntityRecognition"` |
| `parameters.modelVersion` | string | `"latest"` or specific |
| `parameters.piiCategories` | string[] | Filter returned entity types; if `"default"` omitted, only listed categories returned |
| `parameters.redactionPolicies` | array | (Preview 2025-11-15+) Array of policy objects |
| `parameters.redactionPolicies[].policyKind` | string | `CharacterMask` (default) / `NoMask` / `EntityMask` / `SyntheticReplacement` |
| `parameters.redactionPolicies[].redactionCharacter` | string | Mask character (for `CharacterMask`) |
| `parameters.redactionPolicies[].defaultRedactionPolicy` | bool | Marks the default policy |
| `parameters.confidenceScoreThreshold` | object | (Preview) `{ default, overrides[] }` |
| `parameters.disableEntityValidation` | bool | (Preview) Skip strict entity validation |
| `analysisInput.documents[]` | array | Each: `id`, `language`, `text` |

### Response Fields

```json
{
  "kind": "PiiEntityRecognitionResults",
  "results": {
    "documents": [
      {
        "id": "1",
        "redactedText": "Call me at ************.",
        "entities": [
          { "text": "555-555-5555", "category": "PhoneNumber", "offset": 11, "length": 12, "confidenceScore": 0.8 }
        ],
        "warnings": []
      }
    ],
    "errors": [],
    "modelVersion": "2023-09-01"
  }
}
```

| Field | Type | Description |
|-------|------|-------------|
| `documents[].redactedText` | string | Text with PII redacted/masked |
| `documents[].entities[]` | array | Detected PII entities (`text`, `category`, `offset`, `length`, `confidenceScore`) |
| `results.modelVersion` | string | Model version |

---

## 7. PII Detection — Conversation

> **Tier:** Core | **Docs:** `personally-identifiable-information/conversation-pii-overview` | **Deprecated:** No

### Main Concepts

Detects and redacts sensitive data in **turn-based conversational input** (chat transcripts, multi-speaker logs). Optimized for **asynchronous** conversation jobs with conversation-level context across speakers. Supports two modalities: `text` (plain chat) and `transcript` (speech-to-text with audio timing). For transcripts, `redactionSource` selects which speech format to redact (`text`, `lexical`, `itn`, `maskedItn`), and `includeAudioRedaction` enables audio-level redaction based on the lexical format and `audioTimings`. Max 1000 conversation items per conversation; one conversation per request. GA model supports English only; preview adds multi-language.

### API Functions

| Operation | Method | Endpoint | `kind` (task → result) |
|-----------|--------|----------|-------------------------|
| Conversation PII (async) | POST → GET | `/language/analyze-conversations/jobs?api-version=2024-05-01` (GA) / `2024-11-15-preview` | `ConversationalPIITask` → `PiiEntityRecognitionLROResults` |

### Request Parameters

```json
{
  "displayName": "Conversation PII",
  "analysisInput": {
    "conversations": [
      {
        "id": "1",
        "language": "en",
        "modality": "transcript",
        "conversationItems": [
          {
            "participantId": "speaker1",
            "id": "1",
            "text": "...",
            "lexical": "...",
            "itn": "...",
            "maskedItn": "...",
            "audioTimings": [ { "word": "hello", "offset": 1000, "duration": 200 } ]
          }
        ]
      }
    ]
  },
  "tasks": [
    {
      "taskName": "PII redaction",
      "kind": "ConversationalPIITask",
      "parameters": {
        "modelVersion": "latest",
        "redactionPolicy": { "policyKind": "characterMask", "redactionCharacter": "*" },
        "redactionSource": "lexical",
        "includeAudioRedaction": true,
        "piiCategories": ["all"]
      }
    }
  ]
}
```

| Parameter | Type | Description |
|-----------|------|-------------|
| `analysisInput.conversations[].modality` | string | `"text"` or `"transcript"` |
| `conversationItems[].participantId` | string | Speaker identifier |
| `conversationItems[].text` / `lexical` / `itn` / `maskedItn` | string | Speech-format variants (transcript modality) |
| `conversationItems[].audioTimings[]` | array | `{ word, offset, duration }` for audio redaction |
| `tasks[].kind` | string | `"ConversationalPIITask"` |
| `tasks[].parameters.redactionPolicy` | object | `{ policyKind, redactionCharacter }` — `characterMask` / `entityMask` / `noMask` |
| `tasks[].parameters.redactionSource` | string | `text` / `lexical` / `itn` / `maskedItn` |
| `tasks[].parameters.includeAudioRedaction` | bool | Enable audio-level redaction |
| `tasks[].parameters.piiCategories` | string[] | e.g. `["all"]` or specific categories |

### Response Fields

- `operation-location` header (POST) → job URL
- `jobId`, `status`, `createdDateTime`, `expirationDateTime`
- `tasks.items[].results.conversations[].entities[]`: `text`, `category`, `subcategory`, `offset`, `length`, `confidenceScore`
- Redacted conversation content (`redactedText` per item)

---

## 8. PII Detection — Document-based

> **Tier:** Core | **Docs:** `personally-identifiable-information/document-based-pii-overview` | **Deprecated:** No

### Main Concepts

Detects and redacts PII directly in **native document files** (`.pdf`, `.docx`, `.txt`) — no custom text extraction/reconstruction needed. **Asynchronous** batch API; uses Azure Blob Storage for source (input) and target (output) containers. Returns a redacted document (preserving layout, font, color, spacing) plus a structured `.result.json` file with entities/categories/confidence/metadata. GA adds blur-based image redaction and expanded masking policies; preview adds synthetic replacement, entity-label masking, confidence thresholds, and value exclusion. Input limits: ≤40 documents/request, ≤10 MB total. Text within images and digital tables in scanned docs not supported.

### API Functions

| Operation | Method | Endpoint | `kind` (task → result) |
|-----------|--------|----------|-------------------------|
| Document PII (async) | POST → GET | `/language/analyze-documents/jobs?api-version=2024-11-15-preview` (preview) / `2026-05-01` (GA) | `PiiEntityRecognition` → `PiiEntityRecognitionLROResults` |

### Request Parameters

```json
{
  "displayName": "Document PII",
  "analysisInput": {
    "documents": [
      {
        "language": "en",
        "id": "doc1",
        "source": { "location": "{source-blob-SAS-URL}" },
        "target": { "location": "{target-container-SAS-URL}" }
      }
    ]
  },
  "tasks": [
    {
      "kind": "PiiEntityRecognition",
      "taskName": "PII redaction",
      "parameters": {
        "redactionPolicy": { "policyKind": "characterMask" },
        "piiCategories": ["Person", "Organization"],
        "excludeExtractionData": false
      }
    }
  ]
}
```

| Parameter | Type | Description |
|-----------|------|-------------|
| `analysisInput.documents[].source.location` | string | Source blob SAS URL (read+list) |
| `analysisInput.documents[].target.location` | string | Target container SAS URL (write+list) |
| `tasks[].kind` | string | `"PiiEntityRecognition"` |
| `tasks[].parameters.redactionPolicy.policyKind` | string | `noMask` / `characterMask` (default) / `entityMask` / `syntheticReplacement` (preview) |
| `tasks[].parameters.piiCategories` | string[] | Filter entity types |
| `tasks[].parameters.excludeExtractionData` | bool | If `true`, only redacted doc stored (no `.result.json`) |
| Auth | SAS tokens or Managed Identity RBAC | Source: read+list; Target: write+list |

### Response Fields

- POST → `202 Accepted`, `operation-location` header
- GET → `jobId`, `status`, `createdDateTime`, `expirationDateTime`, `errors[]`
- `tasks.items[]` with `kind: "PiiEntityRecognitionLROResults"`
- `results.documents[].source`: `{ kind: "AzureBlob", location }`
- `results.documents[].targets[]`: two entries — `.result.json` and redacted output file
- `results.modelVersion`

---

## 9. Text Analytics for Health

> **Tier:** Core | **Docs:** `language-service/text-analytics-for-health/overview` | **Deprecated:** No

### Main Concepts

A prebuilt biomedical NLP model that performs four functions in a single API call:
- **Named Entity Recognition** — extracts medical concepts (diagnosis, medication name, symptom/sign, dosage, age, healthcare profession, etc.).
- **Relation Extraction** — meaningful connections between concepts (e.g. "time of condition" linking a condition to a time, "dosage of medication").
- **Entity Linking** — disambiguates entities by associating them with preferred names/codes from biomedical vocabularies in the **UMLS Metathesaurus** (plus UMLS sources: AOD, ATC, DRUGBANK, RXNORM, SNOMEDCT_US, MSH, LNC, NCI).
- **Assertion Detection** — contextual modifiers on extracted entities across categories: **Certainty** (positive/possible/negative), **Conditionality** (hypothetical/conditional), **Association** (subject: patient/family/other), **Temporality** (past/current/future).

Additional features: SDOH (Social Determinants of Health) and ethnicity mention extraction; **FHIR output** (Fast Healthcare Interoperability Resources bundle) for EHR integration (REST API only). Supported languages: English, German, French, Italian, Spanish, Portuguese, Hebrew. Hosted API is asynchronous; Docker container is synchronous. Provided "AS IS" — not a medical device; customer must license UMLS source vocabularies separately.

### API Functions

| Operation | Method | Endpoint | `kind` (task → result) |
|-----------|--------|----------|-------------------------|
| Health analysis (async) | POST → GET | `/language/analyze-text/jobs?api-version=2022-05-15-preview` | `Healthcare` → `HealthcareLROResults` |
| Health analysis (sync, container only) | POST | `/language/:analyze-text?api-version={version}` (Docker) | `Healthcare` → `HealthcareResults` |

### Request Parameters

```json
{
  "analysisInput": {
    "documents": [ { "text": "The patient took 100mg of ibuprofen.", "language": "en", "id": "1" } ]
  },
  "tasks": [
    {
      "taskId": "analyze 1",
      "kind": "Healthcare",
      "parameters": { "fhirVersion": "4.0.1" }
    }
  ]
}
```

| Parameter | Type | Description |
|-----------|------|-------------|
| `analysisInput.documents[]` | array | Each: `text`, `language`, `id` |
| `tasks[].kind` | string | `"Healthcare"` |
| `tasks[].taskId` | string | Optional task identifier |
| `tasks[].parameters.fhirVersion` | string | Optional — e.g. `"4.0.1"`; enables FHIR bundle output (REST only) |

### Response Fields

```json
{
  "tasks": {
    "items": [
      {
        "kind": "HealthcareLROResults",
        "results": {
          "documents": [
            {
              "id": "1",
              "entities": [
                {
                  "offset": 0, "length": 11, "text": "100mg",
                  "category": "Dosage",
                  "confidenceScore": 0.9,
                  "name": "100mg",
                  "links": [ { "dataSource": "UMLS", "id": "C0020740" } ]
                }
              ],
              "entityRelations": [
                {
                  "relationType": "DosageOfMedication",
                  "roles": [ { "entity": "#/documents/0/entities/0", "name": "Dosage" } ],
                  "confidenceScore": 0.8
                }
              ]
            }
          ],
          "errors": [],
          "modelVersion": "..."
        }
      }
    ]
  }
}
```

| Field | Type | Description |
|-------|------|-------------|
| `entities[].category` | string | Health entity category (e.g. `MedicationName`, `Dosage`, `SymptomOrSign`, `Diagnosis`, `Age`, `HealthcareProfession`) |
| `entities[].normalizedText` / `name` | string | Normalized form |
| `entities[].links[]` | array | `{ dataSource, id }` — UMLS/ATC/RXNORM/SNOMEDCT_US/etc. |
| `entities[].assertion` | object | Certainty/Conditionality/Association/Temporality modifiers |
| `entityRelations[].relationType` | string | e.g. `DosageOfMedication`, `TimeOfCondition`, `QualitativeConcept` |
| `entityRelations[].roles[]` | array | `{ entity (ref), name }` |
| FHIR mode | object | FHIR Bundle (`resourceType: "Bundle"`, `entry[].resource` with `MedicationStatement`, `Patient`, `Encounter`, etc.) |

---

## 10. Sentiment Analysis & Opinion Mining

> **Tier:** Legacy | **Docs:** `language-service/sentiment-opinion-mining/overview` | **Retiring:** March 31, 2029 → migrate to Microsoft Foundry models

### Main Concepts

Assigns sentiment labels (`positive`, `neutral`, `negative`, and document-level `mixed`) using the highest confidence score. Sentiment is evaluated at both **document level** and **sentence level**, returning confidence scores (0–1) for each of the three classes. **Opinion mining** (aspect-based sentiment analysis) provides granular info about opinions linked to specific words: **targets** (nouns/aspects, e.g. "food") and **assessments** (adjectives, e.g. "delicious"), each with their own sentiment and confidence scores, linked via `relations`. Synchronous; Docker container supported.

### API Functions

| Operation | Method | Endpoint | `kind` (request → response) |
|-----------|--------|----------|------------------------------|
| Sentiment + opinion mining | POST | `/language/:analyze-text?api-version=2023-04-15-preview` | `SentimentAnalysis` → `SentimentAnalysisResults` |

### Request Parameters

```json
{
  "kind": "SentimentAnalysis",
  "parameters": { "modelVersion": "latest", "opinionMining": "True" },
  "analysisInput": {
    "documents": [ { "id": "1", "language": "en", "text": "The food was delicious but the service was slow." } ]
  }
}
```

| Parameter | Type | Description |
|-----------|------|-------------|
| `kind` | string | `"SentimentAnalysis"` |
| `parameters.modelVersion` | string | `"latest"` or specific |
| `parameters.opinionMining` | bool/string | `"True"` to enable opinion mining (targets + assessments) |
| `analysisInput.documents[]` | array | Each: `id`, `language`, `text` |

### Response Fields

```json
{
  "kind": "SentimentAnalysisResults",
  "results": {
    "documents": [
      {
        "id": "1",
        "sentiment": "mixed",
        "confidenceScores": { "positive": 0.47, "neutral": 0.0, "negative": 0.52 },
        "sentences": [
          {
            "sentiment": "negative",
            "confidenceScores": { "positive": 0.0, "neutral": 0.0, "negative": 0.99 },
            "offset": 0, "length": 48, "text": "...",
            "targets": [
              {
                "sentiment": "negative", "confidenceScores": { "positive": 0.0, "neutral": 0.0, "negative": 0.99 },
                "offset": 4, "length": 4, "text": "food",
                "relations": [ { "relationType": "assessment", "ref": "#/documents/0/sentences/0/assessments/0" } ]
              }
            ],
            "assessments": [
              { "sentiment": "positive", "confidenceScores": {...}, "offset": 13, "length": 9, "text": "delicious", "isNegated": false }
            ]
          }
        ],
        "warnings": []
      }
    ],
    "errors": [],
    "modelVersion": "..."
  }
}
```

| Field | Type | Description |
|-------|------|-------------|
| `documents[].sentiment` | string | `positive` / `neutral` / `negative` / `mixed` (doc-level) |
| `documents[].confidenceScores` | object | `{ positive, neutral, negative }` (0–1) at doc level |
| `sentences[].sentiment` / `.confidenceScores` | — | Per-sentence sentiment + scores |
| `sentences[].targets[]` | array | Aspect nouns: `sentiment`, `confidenceScores`, `text`, `offset`, `length`, `relations[]` |
| `sentences[].assessments[]` | array | Opinion adjectives: `sentiment`, `confidenceScores`, `text`, `isNegated`, `isNegated` |
| `relations[]` | array | `{ relationType: "assessment", ref }` linking targets ↔ assessments |

---

## 11. Key Phrase Extraction

> **Tier:** Legacy | **Docs:** `language-service/key-phrase-extraction/overview` | **Retiring:** March 31, 2029 → migrate to Microsoft Foundry models

### Main Concepts

Quickly identifies the main concepts/topics in unstructured text as a list of key phrases. Prebuilt model; no customization. E.g. "The food was delicious and the staff were wonderful." → `["food", "wonderful staff"]`. Synchronous; Docker container supported.

### API Functions

| Operation | Method | Endpoint | `kind` (request → response) |
|-----------|--------|----------|------------------------------|
| Key phrase extraction | POST | `/language/:analyze-text?api-version=2022-05-01` | `KeyPhraseExtraction` → `KeyPhraseExtractionResults` |

### Request Parameters

```json
{
  "kind": "KeyPhraseExtraction",
  "parameters": { "modelVersion": "latest" },
  "analysisInput": {
    "documents": [ { "id": "1", "language": "en", "text": "..." } ]
  }
}
```

| Parameter | Type | Description |
|-----------|------|-------------|
| `kind` | string | `"KeyPhraseExtraction"` |
| `parameters.modelVersion` | string | `"latest"` or specific |
| `analysisInput.documents[]` | array | Each: `id`, `language`, `text` |

### Response Fields

```json
{
  "kind": "KeyPhraseExtractionResults",
  "results": {
    "documents": [
      { "id": "1", "keyPhrases": ["modern medical office", "Dr. Smith", "great staff"], "warnings": [] }
    ],
    "errors": [],
    "modelVersion": "2021-06-01"
  }
}
```

| Field | Type | Description |
|-------|------|-------------|
| `documents[].keyPhrases` | string[] | Array of extracted key phrases |
| `results.modelVersion` | string | Model version used |

---

## 12. Entity Linking

> **Tier:** Legacy | **Docs:** `language-service/entity-linking/overview` | **Retiring:** September 1, 2028 → migrate to NER or Foundry models

### Main Concepts

Identifies and **disambiguates the identity** of entities found in text by linking them to a knowledge source. Currently links to **Wikipedia**: each linked entity returns a `dataSource: "Wikipedia"`, a `url` (English Wikipedia article), and a `bingId` (Bing entity API identifier). Analysis is performed as-is with no model customization. Synchronous.

### API Functions

| Operation | Method | Endpoint | `kind` (request → response) |
|-----------|--------|----------|------------------------------|
| Entity linking | POST | `/language/:analyze-text?api-version=2022-05-01` | `EntityLinking` → `EntityLinkingResults` |

### Request Parameters

```json
{
  "kind": "EntityLinking",
  "parameters": { "modelVersion": "latest" },
  "analysisInput": {
    "documents": [
      { "id": "1", "language": "en", "text": "Microsoft was founded by Bill Gates and Paul Allen on April 4, 1975." }
    ]
  }
}
```

| Parameter | Type | Description |
|-----------|------|-------------|
| `kind` | string | `"EntityLinking"` |
| `parameters.modelVersion` | string | `"latest"` or specific |
| `analysisInput.documents[]` | array | Each: `id`, `language`, `text` |

### Response Fields

```json
{
  "kind": "EntityLinkingResults",
  "results": {
    "documents": [
      {
        "id": "1",
        "entities": [
          {
            "bingId": "a093e9b9-90f5-a3d5-c4b8-5855e1b01f85",
            "name": "Microsoft",
            "matches": [ { "text": "Microsoft", "offset": 0, "length": 9, "confidenceScore": 0.48 } ],
            "language": "en",
            "id": "Microsoft",
            "url": "https://en.wikipedia.org/wiki/Microsoft",
            "dataSource": "Wikipedia"
          }
        ],
        "warnings": []
      }
    ],
    "errors": [],
    "modelVersion": "2021-06-01"
  }
}
```

| Field | Type | Description |
|-------|------|-------------|
| `entities[].name` | string | Canonical entity name |
| `entities[].id` | string | Entity id |
| `entities[].url` | string | Wikipedia article URL |
| `entities[].dataSource` | string | `"Wikipedia"` |
| `entities[].bingId` | string | Bing entity API identifier |
| `entities[].matches[]` | array | `{ text, offset, length, confidenceScore }` — text spans matching this entity |
| `results.modelVersion` | string | Model version used |

---

## 13. Summarization

> **Tier:** Legacy | **Docs:** `language-service/summarization/overview` | **Retiring:** March 31, 2029 → migrate to Microsoft Foundry models

### Main Concepts

Condenses content into summaries across three genres:

- **Text summarization** — accepts plain text blocks.
  - **Extractive** — extracts salient original sentences; returns each with a `rankScore` (0–1) and positional info (`offset`, `length`). Controlled by `sentenceCount` (1–20, default 3) and `sortby` (`Offset` default / `Rank`).
  - **Abstractive** — generates concise, coherent sentences not verbatim from source; returns summary text plus a contextual input range (`contexts[]`: `offset`, `length`). Controlled by `summaryLength` (`oneSentence` ~80 tokens / `short` ~120 / `medium` ~170 / `long` ~210).
  - **Query-focused** — extension accepting a `query` field. Extractive supports `sentenceCount`; abstractive does not.
- **Conversation summarization** — accepts structured conversational input (chat text or speech transcripts with speakers/utterances). Aspects: `issue` + `resolution` (call-center), `chapterTitle` (per-segment titles), `narrative` (per-segment summary), `recap` (one-paragraph), `follow-up tasks` (action items).
- **Document summarization (Preview)** — native document formats (`.txt`, `.pdf`, `.docx`) via Azure Blob Storage. Both `AbstractiveSummarization` and `ExtractiveSummarization` supported. Limits: ≤20 docs/request, ≤10 MB.

### API Functions

| Operation | Method | Endpoint | `kind` (task → result) |
|-----------|--------|----------|-------------------------|
| Text summarization (async) | POST → GET | `/language/analyze-text/jobs?api-version=2023-04-01` (GA) / `2024-11-15-preview` | `ExtractiveSummarization` → `ExtractiveSummarizationLROResults`; `AbstractiveSummarization` → `AbstractiveSummarizationLROResults` |
| Conversation summarization (async) | POST → GET | `/language/analyze-conversations/jobs?api-version={version}` | `ConversationalSummarizationTask` → `conversationalSummarizationResults` |
| Document summarization (async, preview) | POST → GET | `/language/analyze-documents/jobs?api-version=2024-11-15-preview` | `ExtractiveSummarization` / `AbstractiveSummarization` → `*LROResults` |

### Request Parameters

**Text summarization:**
```json
{
  "displayName": "Text Summarization",
  "analysisInput": {
    "documents": [ { "id": "1", "language": "en", "text": "..." } ]
  },
  "tasks": [
    {
      "kind": "ExtractiveSummarization",
      "taskName": "Extractive",
      "parameters": { "sentenceCount": 3, "sortby": "Offset" }
    },
    {
      "kind": "AbstractiveSummarization",
      "taskName": "Abstractive",
      "parameters": { "summaryLength": "medium", "query": "..." }
    }
  ]
}
```

| Parameter | Type | Description |
|-----------|------|-------------|
| `tasks[].kind` | string | `ExtractiveSummarization` or `AbstractiveSummarization` |
| `tasks[].parameters.sentenceCount` | int | Extractive only; 1–20, default 3 |
| `tasks[].parameters.sortby` | string | Extractive: `Offset` (default) / `Rank` |
| `tasks[].parameters.summaryLength` | string | Abstractive: `oneSentence` / `short` / `medium` / `long` |
| `tasks[].parameters.query` | string | Query-focused summarization (optional) |

**Conversation summarization:**
```json
{
  "displayName": "Conversation Summarization",
  "analysisInput": {
    "conversations": [
      {
        "id": "1", "language": "en", "modality": "text",
        "conversationItems": [
          { "text": "...", "id": "1", "role": "agent", "participantId": "agent_1" },
          { "text": "...", "id": "2", "role": "customer", "participantId": "customer_1" }
        ]
      }
    ]
  },
  "tasks": [
    {
      "kind": "ConversationalSummarizationTask",
      "taskName": "Summarize",
      "parameters": { "summaryAspects": ["issue", "resolution", "chapterTitle", "narrative", "recap", "follow-up tasks"] }
    }
  ]
}
```

| Parameter | Type | Description |
|-----------|------|-------------|
| `conversations[].modality` | string | `text` or `transcript` |
| `conversationItems[].role` | string | `agent` / `customer` / `moderator` etc. |
| `tasks[].parameters.summaryAspects[]` | string[] | `issue`, `resolution`, `chapterTitle`, `narrative`, `recap`, `follow-up tasks` |
| `tasks[].parameters.sentenceCount` | int | Only `resolution` aspect; 1–7 |

**Document summarization:**
```json
{
  "tasks": [ { "kind": "AbstractiveSummarization", "parameters": { "summaryLength": "medium" } } ],
  "analysisInput": {
    "documents": [
      { "language": "en", "id": "doc1", "source": { "location": "{source-SAS}" }, "targets": { "location": "{target-SAS}" } }
    ]
  }
}
```

### Response Fields

**Extractive results** (`results.documents[]`):
- `sentences[]`: `text`, `rankScore`, `offset`, `length`

**Abstractive results** (`results.documents[]`):
- `summaries[]`: `text`, `contexts[]` (`offset`, `length`)

**Conversation results** (`results.conversations[]`):
- `summaries[]`: `aspect` (`chapterTitle` / `narrative` / `recap` / `issue` / `resolution` / `Follow-Up Tasks`), `text`, `contexts[]` (`conversationItemId`, `offset`, `length`)

**Document results** (`results.documents[]`):
- `id`, `source` (`kind: AzureBlob`, `location`), `targets[]` (`kind: AzureBlob`, `location`), `warnings[]`

Common: `jobId`, `status`, `createdDateTime`, `expirationDateTime`, `tasks.completed/failed/inProgress/total`, `tasks.items[].results.modelVersion`.

---

## 14. Custom Text Classification

> **Tier:** Legacy | **Docs:** `language-service/custom-text-classification/overview` | **Retiring:** March 31, 2029 → migrate to Microsoft Foundry models

### Main Concepts

A custom feature for building ML models that classify text into user-defined classes. Two project types:
- **Single-label classification** — one class per document.
- **Multi-label classification** — multiple classes per document.

Full lifecycle: define schema → label data → train → evaluate → deploy → classify. Requires an Azure Blob Storage container for training data. Two API surfaces: **Authoring** (import/train/deploy) and **Runtime** (classify via async `/language/analyze-text/jobs`).

### API Functions

**Authoring:**

| Operation | Method | Endpoint |
|-----------|--------|----------|
| Import project | POST | `/language/authoring/analyze-text/projects/{projectName}/:import?api-version=2022-05-01` |
| Train model | POST | `/language/authoring/analyze-text/projects/{projectName}/:train?api-version={version}` |
| Train job status | GET | `/language/authoring/analyze-text/projects/{projectName}/train/jobs/{jobId}?api-version={version}` |
| Deploy model | PUT | `/language/authoring/analyze-text/projects/{projectName}/deployments/{deploymentName}?api-version={version}` |

**Runtime:**

| Operation | Method | Endpoint | `kind` (task → result) |
|-----------|--------|----------|-------------------------|
| Classify (async) | POST → GET | `/language/analyze-text/jobs?api-version={version}` | `CustomMultiLabelClassification` → `customMultiClassificationTasks`; `CustomSingleLabelClassification` → `customSingleClassificationTasks` |

### Request Parameters

**Import body (multi-label):**
```json
{
  "projectFileVersion": "2022-05-01",
  "stringIndexType": "Utf16CodeUnit",
  "metadata": {
    "projectName": "{PROJECT-NAME}",
    "storageInputContainerName": "{CONTAINER-NAME}",
    "projectKind": "customMultiLabelClassification",
    "language": "en",
    "multilingual": true,
    "settings": {}
  },
  "assets": {
    "projectKind": "customMultiLabelClassification",
    "classes": [ { "category": "Class1" }, { "category": "Class2" } ],
    "documents": [ { "location": "{DOCUMENT-NAME}", "language": "en", "dataset": "Train", "classes": [ { "category": "Class1" } ] } ]
  }
}
```

**Train body:**
```json
{
  "modelLabel": "{MODEL-NAME}",
  "trainingConfigVersion": "{VERSION}",
  "evaluationOptions": { "kind": "percentage", "trainingSplitPercentage": 80, "testingSplitPercentage": 20 }
}
```

**Runtime submit body (multi-label):**
```json
{
  "displayName": "Classifying documents",
  "analysisInput": {
    "documents": [ { "id": "1", "language": "en", "text": "..." } ]
  },
  "tasks": [
    {
      "kind": "CustomMultiLabelClassification",
      "taskName": "Multi Label Classification",
      "parameters": { "projectName": "{PROJECT-NAME}", "deploymentName": "{DEPLOYMENT-NAME}" }
    }
  ]
}
```

| Parameter | Type | Description |
|-----------|------|-------------|
| `metadata.projectKind` | string | `customMultiLabelClassification` / `customSingleLabelClassification` |
| `assets.classes[].category` | string | Class label |
| `assets.documents[].dataset` | string | `Train` / `Test` |
| `tasks[].kind` | string | `CustomMultiLabelClassification` / `CustomSingleLabelClassification` |
| `tasks[].parameters.projectName` | string | Project name |
| `tasks[].parameters.deploymentName` | string | Deployment name |

### Response Fields

```json
{
  "jobId": "...", "status": "succeeded", "displayName": "...",
  "tasks": {
    "items": [
      {
        "kind": "customMultiClassificationTasks",
        "taskName": "Classify documents",
        "results": {
          "documents": [
            { "id": "1", "classes": [ { "category": "Class_1", "confidenceScore": 0.95 } ], "warnings": [] }
          ],
          "errors": [], "modelVersion": "2020-04-01"
        }
      }
    ]
  }
}
```

| Field | Type | Description |
|-------|------|-------------|
| `results.documents[].classes[]` | array | `{ category, confidenceScore }` |
| `results.modelVersion` | string | Model version |

---

## 15. Conversation Language Understanding (CLU)

> **Tier:** Legacy (in docs TOC) | **Docs:** `language-service/conversational-language-understanding/quickstart` | **API ref:** `https://aka.ms/clu-apis` | **Deprecated:** No explicit retirement date (recommended successor to LUIS)

### Main Concepts

A custom conversational NLU service that predicts user **intents** and extracts **entities** from utterances. A CLU project consists of a **defined schema** (intents + entity categories) + **labeled utterances** (with intent and entity spans). Project metadata: `projectKind: "Conversation"`, `settings.confidenceThreshold`, `multilingual` (boolean — enables querying in any supported language regardless of training language). Two training modes: **Standard** (faster, English only) and **Advanced** (other languages / multilingual, longer). Managed via Microsoft Foundry or REST Authoring API; queried via synchronous `/language/:analyze-conversations`.

### API Functions

**Authoring:**

| Operation | Method | Endpoint |
|-----------|--------|----------|
| Import project | POST | `/language/authoring/analyze-conversations/projects/{projectName}/:import?api-version=2023-04-01` |
| Import job status | GET | `/language/authoring/analyze-conversations/projects/{projectName}/import/jobs/{jobId}?api-version={version}` |
| Train model | POST | `/language/authoring/analyze-conversations/projects/{projectName}/:train?api-version={version}` |
| Train job status | GET | `/language/authoring/analyze-conversations/projects/{projectName}/train/jobs/{jobId}?api-version={version}` |
| Deploy model | PUT | `/language/authoring/analyze-conversations/projects/{projectName}/deployments/{deploymentName}?api-version={version}` |
| Deploy job status | GET | `/language/authoring/analyze-conversations/projects/{projectName}/deployments/{deploymentName}/jobs/{jobId}?api-version={version}` |
| Delete project | DELETE | `/language/authoring/analyze-conversations/projects/{projectName}?api-version={version}` |

**Runtime (prediction):**

| Operation | Method | Endpoint | `kind` (request → response) |
|-----------|--------|----------|------------------------------|
| Query model | POST | `/language/:analyze-conversations?api-version=2023-04-01` | `Conversation` → `ConversationResult` |

### Request Parameters

**Import body:**
```json
{
  "projectFileVersion": "2023-04-01",
  "stringIndexType": "Utf16CodeUnit",
  "metadata": {
    "projectKind": "Conversation",
    "settings": { "confidenceThreshold": 0.0 },
    "projectName": "{PROJECT-NAME}",
    "multilingual": false,
    "description": "...",
    "language": "en"
  },
  "assets": {
    "projectKind": "Conversation",
    "intents": [ { "category": "Read" }, { "category": "Delete" } ],
    "entities": [ { "category": "Sender" } ],
    "utterances": [
      {
        "text": "Send an email to Alice",
        "dataset": "Train",
        "intent": "Send",
        "entities": [ { "category": "Recipient", "offset": 17, "length": 5 } ]
      }
    ]
  }
}
```

**Train body:**
```json
{
  "modelLabel": "{MODEL-NAME}",
  "trainingMode": "standard",
  "trainingConfigVersion": "{VERSION}",
  "evaluationOptions": { "kind": "percentage", "trainingSplitPercentage": 80, "testingSplitPercentage": 20 }
}
```

**Deploy body:**
```json
{ "trainedModelLabel": "{MODEL-NAME}" }
```

**Prediction body:**
```json
{
  "kind": "Conversation",
  "analysisInput": {
    "conversationItem": {
      "id": "1",
      "participantId": "user1",
      "text": "Send an email to Alice"
    }
  },
  "parameters": {
    "projectName": "{PROJECT-NAME}",
    "deploymentName": "{DEPLOYMENT-NAME}",
    "stringIndexType": "TextElement_V8"
  }
}
```

| Parameter | Type | Description |
|-----------|------|-------------|
| `kind` | string | `"Conversation"` |
| `analysisInput.conversationItem` | object | `{ id, participantId, text }` |
| `parameters.projectName` | string | Project name |
| `parameters.deploymentName` | string | Deployment name |
| `parameters.stringIndexType` | string | `"TextElement_V8"` (CLU prediction) |
| Train: `trainingMode` | string | `"standard"` / `"advanced"` |

### Response Fields

```json
{
  "kind": "ConversationResult",
  "result": {
    "query": "Send an email to Alice",
    "prediction": {
      "topIntent": "Send",
      "projectKind": "Conversation",
      "intents": [
        { "category": "Send", "confidenceScore": 0.95 },
        { "category": "Read", "confidenceScore": 0.03 }
      ],
      "entities": [
        { "category": "Recipient", "text": "Alice", "offset": 17, "length": 5, "confidenceScore": 0.9 }
      ]
    }
  }
}
```

| Field | Type | Description |
|-------|------|-------------|
| `result.prediction.topIntent` | string | Predicted intent with highest confidence |
| `result.prediction.intents[]` | array | All intents: `{ category, confidenceScore }` |
| `result.prediction.entities[]` | array | Predicted entities: `{ category, text, offset, length, confidenceScore }` |
| `result.prediction.projectKind` | string | `"Conversation"` |

---

## 16. Custom Question Answering (CQA)

> **Tier:** Legacy | **Docs:** `language-service/question-answering/overview` | **Retiring:** March 31, 2029 → migrate to Microsoft Foundry models

### Main Concepts

Cloud-based NLP service that creates a conversational layer over your data via a custom **knowledge base (KB)** of question-and-answer pairs. Imports content from URLs, PDFs, FAQs, manuals, spreadsheets; auto-extracts Q&A pairs. Each pair has: all alternate question forms, metadata tags (for filtering), and follow-up prompts (multi-turn). After publishing, a client sends a question and receives JSON answers with confidence scores + follow-up prompts. Uses **layered ranking**: Azure AI Search (first pass) → NLP re-ranking model → final confidence score. Supports **active learning** and **metadata filtering**. A "fine-tuning task" in Foundry is the new name for a CQA project. Requires an Azure AI Search resource.

Two query modes: **query a knowledge base** (`query-knowledgebases`, deployed project) and **query text without a project** (`query-text`, prebuilt — supply records at request time).

### API Functions

| Operation | Method | Endpoint | Notes |
|-----------|--------|----------|-------|
| Query KB (deployed) | POST | `/language/:query-knowledgebases?projectName={name}&deploymentName={name}&api-version=2021-10-01` | Sync; `get-answers` |
| Query text (no project) | POST | `/language/:query-text?api-version=2021-10-01` | Sync; `get-answers-from-text` |
| Authoring (project mgmt) | POST/GET/PUT/DELETE | `/language/authoring/query-knowledgebases/projects/{name}/...` | Import, train, deploy KB |

### Request Parameters

**`query-knowledgebases` (query string):**
- `projectName` (required) — CQA project name
- `deploymentName` (required) — `test` or `production`
- `api-version=2021-10-01`

**`query-knowledgebases` body:**
```json
{
  "question": "How much battery life do I have left?",
  "confidenceScoreThreshold": "0.95"
}
```

| Parameter | Type | Description |
|-----------|------|-------------|
| `question` | string | User question |
| `confidenceScoreThreshold` | float | Minimum confidence; below → default answer "No good match found in KB" |
| `top` | int | (SDK) Number of answers to return |
| `userId` | string | (SDK) Optional user id for active learning |
| `context`/`prompts` | object | (SDK) Multi-turn context for follow-up prompts |

**`query-text` body:**
```json
{
  "question": "...",
  "records": [ { "id": "doc1", "text": "..." } ],
  "language": "en",
  "stringIndexType": "Utf16CodeUnit"
}
```

| Parameter | Type | Description |
|-----------|------|-------------|
| `question` | string | User question |
| `records[]` | array | Each: `{ id, text }` — supplied at request time (no project) |
| `language` | string | Language code |
| `stringIndexType` | string | `"Utf16CodeUnit"` |

### Response Fields

**`query-knowledgebases` response:**
```json
{
  "answers": [
    {
      "questions": ["Check battery level"],
      "answer": "To check your battery level...",
      "confidenceScore": 0.9185,
      "id": 101,
      "source": "https://...",
      "metadata": {},
      "dialog": { "isContextOnly": false, "prompts": [] }
    }
  ]
}
```

| Field | Type | Description |
|-------|------|-------------|
| `answers[].questions[]` | string[] | Alternate question forms |
| `answers[].answer` | string | Answer text |
| `answers[].confidenceScore` | float | 0–1 |
| `answers[].id` | int | Answer id (`-1` for no-match default) |
| `answers[].source` | string | Source URL/file |
| `answers[].metadata` | object | Metadata tags |
| `answers[].dialog` | object | `{ isContextOnly, prompts[] }` for multi-turn |

**`query-text` response:**
```json
{
  "answers": [
    {
      "answer": "...",
      "confidenceScore": 0.85,
      "id": "doc1",
      "answerSpan": { "text": "...", "confidenceScore": 0.9, "offset": 0, "length": 20 },
      "offset": 0, "length": 200
    }
  ]
}
```

| Field | Type | Description |
|-------|------|-------------|
| `answers[].answerSpan` | object | `{ text, confidenceScore, offset, length }` — precise answer span within source |
| `answers[].offset`, `length` | int | Position in source record |

---

## 17. Orchestration Workflow

> **Tier:** Legacy | **Docs:** `language-service/orchestration-workflow/overview` | **Retiring:** March 31, 2029 → migrate to Microsoft Foundry models

### Main Concepts

A cloud-based API service that builds **orchestration models** to route incoming user utterances to the appropriate connected sub-projects: **Conversational Language Understanding (CLU)** projects and **Custom Question Answering (CQA)** projects. Acts as a top-level dispatcher/router model trained on labeled utterances. Example: an enterprise chatbot routes FAQ queries to a CQA KB, calendar operations to a CLU project. Managed via the same CLU Authoring APIs (`aka.ms/clu-authoring-apis`) and queried via the conversation analysis runtime.

### API Functions

| Operation | Method | Endpoint | Notes |
|-----------|--------|----------|-------|
| Authoring (all) | POST/GET/PUT/DELETE | `/language/authoring/analyze-conversations/projects/{name}/...` (shared with CLU) | `projectKind: "Orchestration"` |
| Prediction (runtime) | POST | `/language/:analyze-conversations?api-version=2023-04-01` (or `/analyze-conversation` per REST ref) | `kind: "Conversation"` |

### Request / Response

The prediction request/response structure mirrors CLU (Section 15). The key difference is that `intents[]` in the prediction result map to connected CLU/CQA sub-projects rather than standalone intents, and the orchestration model routes the utterance to the best-matching sub-project. Concrete request/response field details are on the linked `analyze-conversation` REST reference (`/en-us/rest/api/language/2023-04-01/conversation-analysis-runtime/analyze-conversation`).

---

## 18. LUIS — Language Understanding Intelligent Service (Deprecated)

> **Tier:** Deprecated | **Docs:** `azure/ai-services/luis/what-is-luis` | **Retired:** March 31, 2026 (portal unavailable Oct 31, 2025; no new resources) | **Successor:** Conversation Language Understanding (CLU)

### Main Concepts

A cloud-based conversational AI service that applies custom ML to natural-language text to predict overall meaning (**intents**) and pull out relevant information (**entities**). Core building blocks:
- **Intents** — the action/task the user wants (e.g. `BookFlight`).
- **Entities** — the relevant data/details extracted (e.g. `Destination`, `Date`).
- **Utterances** — example user inputs labeled with an intent and used to train.

Supports prebuilt domains/apps, custom apps, active learning (review endpoint utterances), patterns & features, prediction scores. App lifecycle: Plan → Build → Test & Improve → Publish → Connect (Bot Framework, QnA Maker, Speech) → Refine. Two resource types: **authoring resource** and **prediction resource**. Data stored encrypted in the app's region; endpoint utterances retained 30 days if logging enabled (`log=false` query param disables storage).

### API Endpoints

LUIS REST API documented at `/en-us/rest/api/luis/operation-groups`. Two API surfaces:
- **Authoring API** — `https://{authoring-resource}.api.cognitive.microsoft.com/luis/authoring/...` (apps, versions, models, train, publish)
- **Prediction API** — `https://{prediction-resource}.api.cognitive.microsoft.com/luis/prediction/v3.0/apps/{appId}/slots/{slotName}/predict?query={utterance}&log={bool}`

The extracted docs did not enumerate all concrete endpoint paths inline. The prediction endpoint accepts a `query` (utterance) and returns predicted `topIntent`, `intents[]` (with scores), and `entities[]`. The `log` querystring parameter controls endpoint utterance storage for active learning.

### Deprecation

LUIS is fully retired on **March 31, 2026**. LUIS resource creation is no longer available. The LUIS portal became unavailable on **October 31, 2025**. Microsoft recommends migrating LUIS applications to **Conversational Language Understanding (CLU)**.

---

## 19. QnA Maker (Deprecated)

> **Tier:** Deprecated | **Docs:** `azure/ai-services/qnamaker/overview/overview` | **Retired:** October 31, 2025 (portal unavailable March 31, 2025; no new resources since Oct 1, 2022) | **Successor:** Custom Question Answering (CQA)

### Main Concepts

A cloud-based NLP service that creates a conversational layer over data via a custom **knowledge base (KB)** of question-and-answer pairs. Imports content (PDFs, URLs, FAQs, manuals, spreadsheets, web pages) and extracts Q&A pairs. Each pair contains alternate question forms, metadata tags (for filtering), and follow-up prompts (multi-turn). After publishing, a client app sends a question to the KB endpoint and receives a JSON response with the best answer + follow-up prompts. Uses **layered ranking**: data stored in Azure Search (first ranking layer); top results passed through QnA Maker's NLP re-ranking model → final confidence score. Supports **active learning**. QnA Maker does not store customer data; data lives in the region of the deployed dependent services.

### API Endpoints

The overview describes the workflow conceptually (client sends question → KB endpoint → JSON response with answer + follow-up prompts) but the extracted docs did not list concrete REST URL paths. QnA Maker had separate **Authoring** (`https://{host}/qnamaker/v4.0/...`) and **Runtime** (`https://{host}/qnamaker/v1.0/...` or `/generateAnswer`) APIs.

### Deprecation

QnA Maker service retired on **October 31, 2025**. No new QnA Maker resources could be created as of October 1, 2022. The QnA Maker portal became unavailable March 31, 2025. Replacement: **custom question answering (CQA)** as part of Azure Language Service.

---

## 20. Capability Summary & Cross-Reference

### Endpoint & Kind Cross-Reference

| Capability | Tier | Endpoint | Method | `kind` (request) | `kind` (result) | Retiring |
|------------|------|----------|--------|-------------------|-----------------|----------|
| Language Detection | Core | `/language/:analyze-text` | POST | `LanguageDetection` | `LanguageDetectionResults` | — |
| NER (prebuilt) | Core | `/language/:analyze-text` + `/language/analyze-text/jobs` | POST (+LRO) | `EntityRecognition` | `EntityRecognitionResults` | — |
| Custom NER | Core | `/language/authoring/analyze-text/projects/...` + `/language/analyze-text/jobs` | LRO | `CustomEntityRecognition` | `EntityRecognitionLROResults` | — |
| PII (text) | Core | `/language/:analyze-text` + `/language/analyze-text/jobs` | POST (+LRO) | `PiiEntityRecognition` | `PiiEntityRecognitionResults` | — |
| PII (conversation) | Core | `/language/analyze-conversations/jobs` | LRO | `ConversationalPIITask` | `PiiEntityRecognitionLROResults` | — |
| PII (document) | Core | `/language/analyze-documents/jobs` | LRO | `PiiEntityRecognition` | `PiiEntityRecognitionLROResults` | — |
| Text Analytics for Health | Core | `/language/analyze-text/jobs` | LRO | `Healthcare` | `HealthcareLROResults` | — |
| Sentiment & Opinion Mining | Legacy | `/language/:analyze-text` | POST | `SentimentAnalysis` | `SentimentAnalysisResults` | 2029-03-31 |
| Key Phrase Extraction | Legacy | `/language/:analyze-text` | POST | `KeyPhraseExtraction` | `KeyPhraseExtractionResults` | 2029-03-31 |
| Entity Linking | Legacy | `/language/:analyze-text` | POST | `EntityLinking` | `EntityLinkingResults` | 2028-09-01 |
| Summarization (text) | Legacy | `/language/analyze-text/jobs` | LRO | `ExtractiveSummarization` / `AbstractiveSummarization` | `*LROResults` | 2029-03-31 |
| Summarization (conversation) | Legacy | `/language/analyze-conversations/jobs` | LRO | `ConversationalSummarizationTask` | `conversationalSummarizationResults` | 2029-03-31 |
| Summarization (document) | Legacy | `/language/analyze-documents/jobs` | LRO | `ExtractiveSummarization` / `AbstractiveSummarization` | `*LROResults` | 2029-03-31 |
| Custom Text Classification | Legacy | `/language/authoring/analyze-text/projects/...` + `/language/analyze-text/jobs` | LRO | `CustomMultiLabelClassification` / `CustomSingleLabelClassification` | `customMultiClassificationTasks` / `customSingleClassificationTasks` | 2029-03-31 |
| CLU | Legacy | `/language/authoring/analyze-conversations/projects/...` + `/language/:analyze-conversations` | LRO + POST | `Conversation` | `ConversationResult` | — |
| CQA | Legacy | `/language/:query-knowledgebases` + `/language/:query-text` | POST | — | — | 2029-03-31 |
| Orchestration Workflow | Legacy | `/language/authoring/analyze-conversations/projects/...` + `/language/:analyze-conversations` | LRO + POST | `Conversation` | `ConversationResult` | 2029-03-31 |
| LUIS | Deprecated | `/luis/prediction/v3.0/apps/{appId}/slots/{slotName}/predict` | POST | — | — | 2026-03-31 |
| QnA Maker | Deprecated | `/qnamaker/v4.0/...` (authoring) + `/generateAnswer` (runtime) | POST | — | — | 2025-10-31 |

### Common Request Shape (sync `/language/:analyze-text`)

```
{
  "kind": "{FeatureName}",
  "parameters": { "modelVersion": "latest", ...feature-specific... },
  "analysisInput": {
    "documents": [ { "id": "1", "language": "en", "text": "..." } ]
  }
}
```

### Common Request Shape (async `/language/analyze-text/jobs`)

```
{
  "displayName": "Job name",
  "analysisInput": {
    "documents": [ { "id": "1", "language": "en", "text": "..." } ]
  },
  "tasks": [
    { "kind": "{FeatureName}", "taskName": "task1", "parameters": { ... } }
  ]
}
→ 202 Accepted, header "operation-location": {endpoint}/language/analyze-text/jobs/{jobId}
→ GET {operation-location} until status="succeeded"
```

### Key Takeaways

1. **A single Language resource** fronts all capabilities — no per-feature provisioning (except CQA needing Azure AI Search, and custom features needing Blob Storage).
2. **Three shared runtime endpoints** (`:analyze-text`, `analyze-text/jobs`, `:analyze-conversations`) discriminate features via the `kind` field — minimizing the API surface to learn.
3. **Synchronous vs async** is determined by payload size and feature: single-shot small text → sync `/language/:analyze-text`; batch, custom, document, or conversation → async LRO (`*jobs` endpoints).
4. **Custom features** (Custom NER, Custom Text Classification, CLU, CQA, Orchestration) share an **Authoring + Runtime** split: author via `/language/authoring/...`, query via the runtime endpoints with `projectName` + `deploymentName`.
5. **Migration pressure is strong**: 8 of 11 legacy capabilities retire by March 31, 2029 (Entity Linking by Sep 1, 2028), and both fully deprecated services (LUIS, QnA Maker) are already retired. Microsoft's direction is toward **Microsoft Foundry generative models** for new NLU work.
6. **Preview vs GA response formats differ** for NER (GA `category`/`subcategory`/`confidenceScore` vs preview `type`/`tags`/`metadata`/`score`) — clients must handle both during migration.