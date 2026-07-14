# Google Gemini API — Observability & Safety Agent Capabilities

Analysis of the agent-related capabilities offered by the Google Gemini API, based on the official docs for **Logs and datasets** (`/gemini-api/docs/logs-datasets`), **Data Logging and Sharing** (`/gemini-api/docs/logs-policy`), **Safety settings** (`/gemini-api/docs/safety-settings`), and **Safety and factuality guidance** (`/gemini-api/docs/safety-guidance`). Each capability is broken down into main concepts, API surface (functions, parameters, response fields), and notes/constraints.

Scope of this study: capabilities relevant to **agents** — i.e. observability of agentic (conversational / `generateContent` / Interactions API) traffic, and the per-request safety filtering that gates both prompts and model outputs produced by agents. The Logs & datasets surface is the main agent-observability mechanism (it captures every supported `GenerateContent`/`BatchGenerateContent`/`StreamGenerateContent`/Interactions call, lets you inspect the full prompt+response+context, and export curated slices to datasets for evaluation or re-runs); the Safety settings surface lets you tune, per request, the harm-category filters that gate an agent's inputs and outputs.

Storage for logs is only available on the **Gemini API paid tier** (billing-enabled projects).

---

## Table of Contents

1. [Logs & Datasets](#1-logs--datasets)
2. [Data Logging & Sharing Policy](#2-data-logging--sharing-policy)
3. [Safety Settings](#3-safety-settings)
4. [Safety Feedback in Responses](#4-safety-feedback-in-responses)
5. [Safety & Factuality Guidance](#5-safety--factuality-guidance)

---

## 1. Logs & Datasets

**Summary** — Project-level capture of every supported Gemini API call (prompt → model response, including prior-turn context), viewable and filterable in the AI Studio **Logs** page, with a configurable retention window and the ability to save filtered slices into named **datasets** for analysis, export (CSV / JSONL / Google Sheets), re-runs via the Batch API, or optional sharing with Google.

### Main concepts

- **Supported calls**: `GenerateContent`, `BatchGenerateContent`, `StreamGenerateContent`, and Interactions API calls **excluding Managed Agents**. Calls made through the OpenAI compatibility endpoints are also captured.
- **Per-API default storage behavior**:
  - **Interactions API** stores requests by default (`store=true`) to simplify server-side state management. Toggling Interactions logging **off** in the AI Studio Settings panel prevents automatic storage/retrieval of conversation history unless explicitly overridden per request.
  - **`generateContent` API** does **not** store requests by default (`store=false`); storage must be enabled per-request or at the project level from AI Studio.
- **Logs entry content**: full prompt, full response from Gemini, and the context from previous turns. For Interactions API requests, logs also include a direct link to the `previous_interaction_id`.
- **Retention**: logs expire and are marked for deletion after a default window of **55 days** unless saved to a dataset (dataset logs don't expire). The retention window can be configured per project to **7, 14, 28, or 55 days max**.
- **Datasets**: curated collections of logs used to organize and export usage more effectively. Datasets do not have set expiry dates, but each project has a default storage limit of **up to 1,000 logs**.
- **Dataset use cases**: curate **challenge sets** (drive future improvements), **sample sets** (real usage sampling, edge-case collections for pre-deployment checks), and **evaluation sets** (representative usage for cross-model or system-instruction-iteration comparison).
- **Optional sharing with Google**: datasets can be shared with Google as demonstration examples to contribute to Gemini research and development (see [Data Logging & Sharing Policy](#2-data-logging--sharing-policy)).

### API functions & parameters

**Project-level logging toggle** — AI Studio **Settings** panel on the [Logs and Datasets](https://aistudio.google.com/logs) page. Logging can be toggled on/off independently for the `generateContent` API and the Interactions API, for all projects or specific projects, changeable at any time.

**Request-level `store` override** — differs by API:

**`generateContent` API** (Python / JavaScript):

| Parameter | Type | Description |
|---|---|---|
| `model` | string | Model id (e.g. `gemini-3.5-flash`). |
| `contents` | string \| array | Prompt content. |
| `config.store` | boolean | `True` to enable logging of this request (default `False` for generateContent). |

```python
from google import genai
client = genai.Client()
response = client.models.generate_content(
    model='gemini-3.5-flash',
    contents='Explain quantum entanglement in simple terms.',
    config={'store': False}   # Set to True to enable logging of this request
)
```

**Interactions API** (Python / JavaScript):

| Parameter | Type | Description |
|---|---|---|
| `model` | string | Model id. |
| `input` | string \| array | User input. |
| `store` | boolean | `False` to disable logging of this request (default `True` for Interactions). |

```python
interaction = client.interactions.create(
    model="gemini-3.5-flash",
    input="Explain quantum entanglement in simple terms.",
    store=True   # Set to False to disable logging of this request
)
```

**View logs** — AI Studio Logs page (`aistudio.google.com/logs`): select project from drop-down; logs appear in reverse chronological order for the Interactions API (if they exist); to observe generateContent logs, first enable logging in the Settings panel. Click an entry for a payload preview (full prompt, response, prior-turn context; Interactions entries link to `previous_interaction_id`).

**Create & share datasets** — from the Logs page:

1. Use the filter bar at the top to select a property to filter by.
2. Use checkboxes to select all or individual logs from the filtered view.
3. Click **Create dataset**.
4. Give the dataset a name and optional description.
5. View the curated dataset; export to CSV, JSONL, or Google Sheets.

**Configure retention** — set the project's log retention window to 7, 14, 28, or 55 days max (default 55).

### Limitations (logging not currently supported)

- Imagen and Veo models
- Gemini embedding models
- Gemini Robotics model
- Inputs containing videos, GIFs, or PDFs
- Public Preview Agents in the Gemini API

### Notes

- **What's next (agent angle)**:
  - **Prototype with session history**: use AI Studio Build to vibe-code apps and add your API key to enable a history of Gemini API logs for AI features.
  - **Re-run logs with the Gemini Batch API**: use datasets for response sampling and evaluation of models or application logic by re-running logs with the Batch API (cookbook example).

---

## 2. Data Logging & Sharing Policy

**Summary** — Outlines the storage, management, and optional Google-sharing rules for Gemini API logs. Logging is opt-in and developer-owned; data flows under "Unpaid Services" terms only when datasets are explicitly shared with Google. Privacy protections (account/key/project disconnection) are applied before any human review.

### Main concepts

- **Developer-owned data**: Gemini API logs are developer-owned API data from supported calls for projects with billing enabled. Logs span the entire process from a user's request to the model's response.
- **Opt-in**: as a project owner you choose to opt in to logging — for your own use **or** for feedback/sharing with Google to help improve models.
- **Default retention**: logs expire after 55 days by default and become unavailable after this period.
- **Datasets as retention mechanism**: datasets retain logs of interest/value beyond the default window for downstream use cases and optional contribution to model improvements. Dataset logs have no set expiry, but each project has a default storage limit of up to 1,000 logs.
- **Default data-use posture**: because logging is only available for billing-enabled projects, prompts and responses within logs are **not** used for product improvement or development by default.
- **Sharing ≠ default**: only if you choose to share a dataset with Google is it processed under the "Unpaid Services" terms (see below).

### What you can share / how Google uses it

| Aspect | Default (logging on, no sharing) | When you share a dataset with Google |
|---|---|---|
| Data scope | Requests + responses for your own use. | Logs in the dataset (requests + responses) may be used to develop/improve Google products, services, and ML technologies, including model training and evaluation. |
| Governing terms | API Terms on data use (paid service). | "Unpaid Services" data-use terms. |
| Human review | Not used for model improvement. | Human reviewers may read, annotate, and process the shared API inputs/outputs. |
| Privacy protection | n/a | Google disconnects the data from your Google Account, API key, and Cloud project **before** reviewers see or annotate it. |
| Expiry | 55 days default (configurable 7–55). | Dataset logs have no set expiry. |

### Data permissions & constraints

- By opting in to contributing API data, you confirm you have the necessary permissions for Google to process and use the data as described.
- **Do not contribute logs containing sensitive, confidential, or proprietary information** obtained through the paid service. **Do not include personal, sensitive, or confidential information.**
- The license granted to Google under "Submission of Content" in the API Terms extends to any content you submit (prompts, including associated system instructions, cached content, files such as images/videos/documents) and to any generated responses.

### Notes

- This is the policy layer that governs the Logs & datasets capability in §1; it does not introduce a separate API surface — sharing is performed from the dataset UI.

---

## 3. Safety Settings

**Summary** — Adjustable, per-request content-safety filters across four harm categories that gate an agent's prompts and outputs. The model's inherent safety also includes non-adjustable built-in protections against core harms (e.g. child-safety endangering content), which are always blocked. Blocking is based on the **probability** of content being unsafe (not severity).

### Main concepts

- **Adjustable safety filters (four categories)** — defined in `HarmCategory`:

| Category | Description |
|---|---|
| Harassment | Negative or harmful comments targeting identity and/or protected attributes. |
| Hate speech | Content that is rude, disrespectful, or profane. |
| Sexually explicit | Contains references to sexual acts or other lewd content. |
| Dangerous | Promotes, facilitates, or encourages harmful acts. |

- **Built-in (non-adjustable) protections**: against core harms such as content that endangers child safety — always blocked, cannot be adjusted.
- **Probability levels** (`HarmProbability`): `HIGH`, `MEDIUM`, `LOW`, `NEGLIGIBLE`. The API blocks based on the **probability** of content being unsafe, **not the severity**. Some content can have a low probability of being unsafe yet high severity of harm, so careful testing of the appropriate blocking level is required.
- **Block thresholds** (`HarmBlockThreshold`) — per category, per request:

| Threshold (Google AI Studio) | Threshold (API) | Description |
|---|---|---|
| Off | `OFF` | Turn off the safety filter. |
| Block none | `BLOCK_NONE` | Always show regardless of probability of unsafe content. |
| Block few | `BLOCK_ONLY_HIGH` | Block when high probability of unsafe content. |
| Block some | `BLOCK_MEDIUM_AND_ABOVE` | Block when medium or high probability of unsafe content. |
| Block most | `BLOCK_LOW_AND_ABOVE` | Block when low, medium, or high probability of unsafe content. |
| N/A | `HARM_BLOCK_THRESHOLD_UNSPECIFIED` | Threshold is unspecified; block using default threshold. |

- **Default threshold**: if the threshold is not set, the default block threshold is **Off** for Gemini 2.5 and 3 models. Due to the model's inherent safety, the additional filters are **Off** by default; only enable them when consistency is required for your application. The default model behavior covers most use cases.
- **Safety filtering per request**: each request's content is analyzed and assigned a safety rating (category + probability of the harm classification). Example: content blocked for harassment with high probability yields a rating with `category = HARASSMENT` and `harm probability = HIGH`.

### API functions & parameters

**Tool** — `generateContent` (and `StreamGenerateContent`). Safety settings are supplied in the request config under `safetySettings`.

**`safetySettings` array element (`SafetySetting`):**

| Field | Type | Required | Description |
|---|---|---|---|
| `category` | enum (`HarmCategory`) | yes | `HARM_CATEGORY_HARASSMENT`, `HARM_CATEGORY_HATE_SPEECH`, `HARM_CATEGORY_SEXUALLY_EXPLICIT`, `HARM_CATEGORY_DANGEROUS_CONTENT`. |
| `threshold` | enum (`HarmBlockThreshold`) | yes | `OFF`, `BLOCK_NONE`, `BLOCK_ONLY_HIGH`, `BLOCK_MEDIUM_AND_ABOVE`, `BLOCK_LOW_AND_ABOVE`, `HARM_BLOCK_THRESHOLD_UNSPECIFIED`. |

**Code examples** (Python):

```python
from google import genai
from google.genai import types

client = genai.Client()
response = client.models.generate_content(
    model="gemini-3.5-flash",
    contents="Some potentially unsafe prompt",
    config=types.GenerateContentConfig(
        safety_settings=[
            types.SafetySetting(
                category=types.HarmCategory.HARM_CATEGORY_HATE_SPEECH,
                threshold=types.HarmBlockThreshold.BLOCK_LOW_AND_ABOVE,
            ),
        ]
    )
)
print(response.text)
```

**REST:**

```bash
curl "https://generativelanguage.googleapis.com/v1beta/models/gemini-3.5-flash:generateContent" \
  -H "x-goog-api-key: $GEMINI_API_KEY" \
  -H "Content-Type: application/json" \
  -X POST \
  -d '{
    "safetySettings": [
      {"category": "HARM_CATEGORY_HATE_SPEECH", "threshold": "BLOCK_LOW_AND_ABOVE"}
    ],
    "contents": [{"parts": [{"text": "Some potentially unsafe prompt."}]}]
  }'
```

**Google AI Studio**: Run settings → Advanced settings → **Safety settings** → modal with per-category sliders. When a request is blocked, a "Content blocked" warning appears; hover to see the category and probability of the harm classification.

### Notes

- Safety settings are independent of the Logs capability; a request can be logged (§1) and independently gated by safety filters (§3, §4).
- Built-in (non-adjustable) protections always apply regardless of the adjustable thresholds.

---

## 4. Safety Feedback in Responses

**Summary** — `generateContent` returns a `GenerateContentResponse` that surfaces safety feedback in two locations: **prompt-level** feedback (`promptFeedback`) indicating whether the prompt was blocked, and **per-candidate** feedback (`Candidate.finishReason` + `Candidate.safetyRatings`) indicating whether the generated output was blocked and the detailed per-category ratings. Blocked content is not returned.

### Main concepts

- **Prompt feedback** (`promptFeedback`): if `promptFeedback.blockReason` is set, the prompt was blocked and **no candidates are returned**. When the prompt is not blocked, `promptFeedback` is empty.
- **`BlockReason` enums** (prompt-level block reasons):

| Enum | Meaning |
|---|---|
| `BLOCK_REASON_UNSPECIFIED` | Default; unused. |
| `SAFETY` | Prompt blocked due to safety reasons — inspect `safetyRatings` for the triggering category. |
| `OTHER` | Prompt blocked due to unknown reasons. |
| `BLOCKLIST` | Prompt blocked due to terminology blocklist terms. |
| `PROHIBITED_CONTENT` | Prompt blocked due to prohibited content (built-in, non-adjustable protection). |
| `IMAGE_SAFETY` | Candidates blocked due to unsafe image-generation content. |

- **Candidate feedback** (`Candidate`): `finishReason` and `safetyRatings` per candidate. If `finishReason == SAFETY`, the response content was blocked by the adjustable filters; inspect `safetyRatings` for details. The blocked content is **not returned**. `PROHIBITED_CONTENT` as a finish reason indicates built-in protections were activated.
- **Safety ratings schema** (`SafetyRating` per category): at most one rating per category. Fields include `category`, `probability` (enum), `probabilityScore` (numeric), `severity` (enum), `severityScore` (numeric), and `blocked` (boolean).

### Response field map

| Field | Location | Meaning |
|---|---|---|
| `promptFeedback` | top-level | Returns the prompt's content-filter feedback. |
| `promptFeedback.blockReason` | top-level | If set, the prompt was blocked (no candidates returned). |
| `promptFeedback.safetyRatings[]` | top-level | Per-category ratings of the prompt. |
| `candidates[]` | top-level | Candidate responses from the model. |
| `Candidate.finishReason` | per-candidate | e.g. `SAFETY` (adjustable filter), `PROHIBITED_CONTENT` (built-in). |
| `Candidate.safetyRatings[]` | per-candidate | Per-category `category`, `probability`, `probabilityScore`, `severity`, `severityScore`, `blocked`. |
| `usageMetadata` | top-level | Token usage (`promptTokenCount`, etc.). |

### Example: blocked response with per-category ratings

```json
{
  "candidates": [{
    "finishReason": "SAFETY",
    "safetyRatings": [
      {"category": "HARM_CATEGORY_HATE_SPEECH", "probability": "NEGLIGIBLE", "probabilityScore": 0.11, "severity": "HARM_SEVERITY_LOW", "severityScore": 0.28},
      {"category": "HARM_CATEGORY_DANGEROUS_CONTENT", "probability": "HIGH", "blocked": true, "probabilityScore": 0.95, "severity": "HARM_SEVERITY_MEDIUM", "severityScore": 0.43},
      {"category": "HARM_CATEGORY_HARASSMENT", "probability": "NEGLIGIBLE", "probabilityScore": 0.11, "severity": "HARM_SEVERITY_NEGLIGIBLE", "severityScore": 0.19},
      {"category": "HARM_CATEGORY_SEXUALLY_EXPLICIT", "probability": "NEGLIGIBLE", "probabilityScore": 0.23, "severity": "HARM_SEVERITY_NEGLIGIBLE", "severityScore": 0.09}
    ]
  }]
}
```

### Notes

- The API returns either all requested candidates or none; no candidates are returned only if something was wrong with the prompt (check `promptFeedback`).
- Severity vs. probability: blocking decisions are keyed on **probability**, but severity scores are also surfaced for developers who want finer-grained triage.

---

## 5. Safety & Factuality Guidance

**Summary** — Developer guidance (not a separate API surface) on building safe, responsible applications with LLMs: understand application-specific safety risks, choose mitigations (blocklists, classifiers, anti-prompt-injection safeguards, scope-narrowing, safety-settings adjustment, Grounding with Google Search for factuality), and perform iterative safety benchmarking + adversarial testing, then monitor for problems in production. This is the design playbook that contextualizes the Safety settings (§3) and Logs (§1) capabilities.

### Main concepts

- **Iterative model implementation cycle**: understand risks → adjust/test (repeat until performance is appropriate for the application) → monitor in production.
- **Risk-aware scoping**: assess likelihood, seriousness, and mitigations per application (e.g. an essay generator on factual events needs more misinformation care than a fictional-story generator). Research end users and affected parties.
- **Mitigation techniques** (recommended, mostly application-side):
  - **Block unsafe inputs / filter outputs** before showing to the user — blocklists for words/phrases, or require human reviewers to alter/block content.
  - **Trained classifiers** labeling each prompt with potential harms or adversarial signals; route per harm type (e.g. overtly adversarial/abusive inputs are blocked and replaced with a pre-scripted response).
  - **Safeguards against deliberate misuse**: unique user IDs + per-user volume limits; protections against prompt injection (manipulating output via crafted prompts that instruct the model to ignore prior examples).
  - **Adjust functionality to lower-risk scope**: narrower tasks (keyword extraction) or greater human oversight (short-form content reviewed by a human) reduce risk.
  - **Adjust harmful-content safety settings** to decrease the likelihood of harmful responses — see [Safety settings](#3-safety-settings) for the adjustable filters.
  - **Decrease factual inaccuracies/hallucinations** by enabling **Grounding with Google Search** (connects the model to real-time web content across all available languages, with verifiable citations beyond the knowledge cutoff); may be disabled for creative, non-information-seeking use cases.
- **Two kinds of testing**:
  - **Safety benchmarking**: design safety metrics reflecting how your application could be unsafe in its usage context; test against them using evaluation datasets; define minimum acceptable safety metric levels before testing.
  - **Adversarial testing**: proactively try to break the application; select test data most likely to elicit problematic output across all harm types (including rare/edge cases), with diversity in structure, meaning, and length. Use **automated testing** (a red-team language model that finds inputs eliciting harmful outputs) instead of/enhancing manual red teams.
- **Monitoring**: set up a monitored user-feedback channel (e.g. thumbs up/down), run user studies with a diverse mix of users, especially when usage patterns differ from expectations.

### Notes

- Subject to the Generative AI Prohibited Use Policy and Gemini API Terms of Service.
- Built-in safety filtering (§3) plus this guidance are complementary: the API provides the adjustable filters; the developer owns application-level mitigations, testing, and production monitoring.
- Logs (§1) provide the observability substrate for production monitoring — filtered/exported logs can feed the safety benchmarking and adversarial-testing datasets described here.

---

## Cross-cutting notes (agent angle)

- **Two distinct observability surfaces**: (1) **Logs & datasets** — captures agentic traffic (`generateContent`/Interactions, including OpenAI-compat calls) for inspection, retention control, dataset curation, export, and Batch-API re-runs; (2) **Safety feedback** — returned inline on every `generateContent` response, exposing prompt-level and per-candidate block reasons + per-category probability/severity ratings. The former is project-level and viewed in AI Studio; the latter is per-request and inspectable in code.
- **Agent traffic identification**: Interactions-API logs include a direct link to the `previous_interaction_id`, enabling traversal of an agent's multi-turn conversation chain. generateContent logs (when enabled) capture the full prompt + response + prior-turn context.
- **Default storage differs by API surface**: Interactions API logs by default (`store=true`); generateContent does not (`store=false`). Toggling Interactions logging off disables automatic history storage/retrieval unless overridden per request — relevant for agents relying on server-side state.
- **Safety gating is per-request and probability-based**: adjustable per-category thresholds (`OFF` / `BLOCK_NONE` / `BLOCK_ONLY_HIGH` / `BLOCK_MEDIUM_AND_ABOVE` / `BLOCK_LOW_AND_ABOVE`) default to `Off` on Gemini 2.5/3 (model's inherent safety covers most cases). Non-adjustable built-in protections (e.g. child-safety, `PROHIBITED_CONTENT`) always block.
- **No enterprise-tier gating on safety**: Safety settings and feedback are available to standard API users; only **logs storage** requires a paid-tier (billing-enabled) project.
- **Logging limitations relevant to agents**: videos, GIFs, PDFs, Imagen/Veo/embedding/Robotics models, and Public Preview Agents in the Gemini API are **not** loggable — agents relying on those surfaces won't appear in Logs/datasets.
- **Privacy/data-use**: by default, logged prompts/responses are not used by Google for product improvement; sharing a dataset with Google opts that data into "Unpaid Services" terms (may be used for training/evaluation with human review after account/key/project disconnection). Sensitive/confidential/personal data must not be contributed.
- **Evaluation loop**: datasets curated from logs can be re-run with the Batch API for response sampling and evaluation of models or application logic — the Google equivalent of an offline agent-evaluation harness, complementing the online safety feedback returned inline.