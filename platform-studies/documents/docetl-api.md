# DocETL API Analysis — Declarative & Agentic LLM Data-Processing Platform

> **Product:** [DocETL](https://www.docetl.org/) (open-source, UC Berkeley EPIC Data Lab + Data Systems & Foundations) · **Docs:** [ucbepic.github.io/docetl](https://ucbepic.github.io/docetl) · **Repo:** [ucbepic/docetl](https://github.com/ucbepic/docetl) · **Playground / IDE:** [docetl.org/playground](https://docetl.org/playground) (DocWrangler)
> **Auth:** API keys via environment variables / `.env` (OpenAI, Anthropic, Azure, Gemini, Cohere, Ollama, AWS Bedrock, any LiteLLM-supported provider)
> **Interfaces:** Python Frame API (`import docetl`), YAML declarative config (`docetl run pipeline.yaml`), pandas `.semantic` accessor, CLI (`docetl run` / `docetl build` / `docetl clear-cache`)
> **Description:** DocETL is a declarative and agentic map-reduce framework for processing large collections of unstructured or structured data with LLMs. You express each processing step in natural language, and DocETL orchestrates operators (map, filter, reduce, resolve, equijoin, rank, extract, cluster, and more) across your data, automatically optimizing pipelines by swapping models, rewriting prompts, decomposing operations, and replacing subtasks with code to raise accuracy and cut cost.

---

## Table of Contents

1. [Platform Overview & Core Concepts](#1-platform-overview--core-concepts)
2. [Interfaces & Configuration](#2-interfaces--configuration)
3. [Datasets & Frames — Data Ingestion](#3-datasets--frames--data-ingestion)
4. [Output Schemas — Type System](#4-output-schemas--type-system)
5. [Operators — Validation & Gleaning](#5-operators--validation--gleaning)
6. [Map Operator](#6-map-operator)
7. [Filter Operator](#7-filter-operator)
8. [Reduce Operator](#8-reduce-operator)
9. [Parallel Map Operator](#9-parallel-map-operator)
10. [Resolve Operator](#10-resolve-operator)
11. [Equijoin Operator](#11-equijoin-operator)
12. [Rank Operator](#12-rank-operator)
13. [Extract Operator](#13-extract-operator)
14. [Cluster Operator](#14-cluster-operator)
15. [Link Resolve Operator](#15-link-resolve-operator)
16. [Split Operator](#16-split-operator)
17. [Gather Operator](#17-gather-operator)
18. [Unnest Operator](#18-unnest-operator)
19. [Sample & TopK Operators](#19-sample--topk-operators)
20. [Code Operations](#20-code-operations)
21. [Tool-Equipped Agents](#21-tool-equipped-agents)
22. [Retrievers (RAG)](#22-retrievers-rag)
23. [Optimization — Plan Rewrites](#23-optimization--plan-rewrites)
24. [Optimization — Model Cascades (BARGAIN)](#24-optimization--model-cascades-bargain)
25. [Optimization — MOAR (Multi-Objective Agentic Rewrites)](#25-optimization--moar-multi-objective-agentic-rewrites)
26. [Python API Reference — Frame & Terminal Actions](#26-python-api-reference--frame--terminal-actions)
27. [Pandas `.semantic` Accessor](#27-pandas-semantic-accessor)
28. [CLI Reference](#28-cli-reference)
29. [DocWrangler — Interactive IDE](#29-docwrangler--interactive-ide)
30. [Capability Summary & Cross-Reference](#30-capability-summary--cross-reference)

---

## 1. Platform Overview & Core Concepts

### Main Concepts

DocETL is a **declarative LLM-powered data-processing pipeline framework** — analogous to how SQL expresses database operations without execution details. It treats unstructured-data processing as a sequence of operators (map, reduce, filter, resolve, etc.) over a dataset, orchestrated and parallelized automatically.

Core abstractions:

- **Dataset** — The input data: JSON, CSV, Parquet files, a directory of documents (PDF/Word/PPT/Excel auto-converted to text), or an in-memory list of dicts. Each item is one row (a JSON object, a CSV row, or a file).
- **Frame** — The lazy, immutable pipeline object in the Python API. Readers return a Frame; every operation returns a new Frame. Nothing executes until a terminal action (`.collect()`, `.to_pandas()`, `.write_json()`).
- **Operator** — A processing step applied to the data. Two categories: **LLM-powered operators** (map, filter, reduce, resolve, equijoin, rank, extract, cluster, parallel_map, link_resolve) that invoke LLM calls, and **auxiliary operators** (split, gather, unnest, sample, topk, code) that are deterministic or Python-based with no LLM calls.
- **Pipeline** — A sequence of operators applied to a dataset, producing an output. Defined as ordered steps in YAML or as a chain of Frame method calls in Python.
- **Output Schema** — A type system specifying the structure of each LLM operator's output (strings, ints, floats, booleans, lists, objects, enums). Enforced via OpenAI tools/function-calling API (default) or LiteLLM structured output mode.
- **Prompt** — A Jinja2 template defining the LLM instruction. Accesses input fields via `{{ input.keyname }}` (for map/filter) or `{{ inputs }}` list (for reduce), or `{{ input1 }}`/`{{ input2 }}` (for resolve), `{{ left }}`/`{{ right }}` (for equijoin), `{{ retrieval_context }}` (for RAG).
- **Optimizer** — Three mechanisms that improve cost/accuracy: **plan rewrites** (automatic, equivalence-preserving reordering at run start), **model cascades** (BARGAIN, inline cost reduction for binary operators), and **MOAR** (offline multi-objective search over pipeline configurations).
- **Retriever** — A LanceDB-backed index (full-text, embedding, or hybrid) that injects relevant context from an external dataset into operation prompts as `{{ retrieval_context }}`.

### End-to-End Flow

```
Raw files (JSON/CSV/Parquet/dir) ──► Frame (lazy)
    │
    ▼
Operators (chained): map → filter → resolve → reduce → ...
    │   (LLM calls parallelized, cached, optimized)
    │
    ▼
Terminal action: .collect() / .to_pandas() / .write_json()
    │
    ▼
Structured output (tables / list of dicts / DataFrame)
```

### Key Differentiators

- **Declarative** — Describe *what* you want in natural-language prompts and schemas; DocETL handles orchestration, parallelization, retries, and caching.
- **Agentic optimization** — MOAR automatically rewrites pipelines (decomposing operations, swapping models, adding gleaning/resolve steps) to balance accuracy and cost.
- **Statistical guarantees** — Model cascades (BARGAIN) provide provable quality guarantees on filter/resolve/equijoin with probability `1 - delta`.
- **Two interfaces** — Python Frame API (chainable, programmatic) and YAML config (low-code/no-code, CLI-runnable); fully convertible between each other.
- **Operator-level tool agents** — Operations can call Python tools or OpenAI Agents SDK tools (web search, hosted shell, subagents) before producing structured output.
- **Built-in RAG** — LanceDB retrievers attach to any LLM operation for context augmentation without external services.
- **Multi-provider** — Uses LiteLLM underneath, supporting 100+ providers (OpenAI, Anthropic, Gemini, Azure, Cohere, Ollama, AWS Bedrock, Replicate, HuggingFace).

### Supported Models (via LiteLLM)

OpenAI (GPT-4, GPT-4o-mini, etc.), Anthropic (Claude), Google VertexAI/Gemini, Cohere, Replicate, Azure OpenAI, Hugging Face, AWS Bedrock (Claude, AI21, Cohere), Ollama (llama2, etc.). Use LiteLLM model names directly (e.g., `openai/gpt-4o-mini`, `gemini/gemini-1.5-flash-002`, `azure/gpt-4o-mini`). Self-hosted models supported via `default_lm_api_base` / `default_embedding_api_base`. Primarily tested with OpenAI; recommended for structured-output reliability.

### Research Origin

- **DocETL**: "Agentic Query Rewriting and Evaluation for Complex Document Processing" — VLDB 2025 ([arXiv:2410.12189](https://arxiv.org/abs/2410.12189))
- **DocWrangler**: "Steering Semantic Data Processing With DocWrangler" — UIST 2025 ([arXiv:2504.14764](https://arxiv.org/abs/2504.14764))
- **MOAR**: "Multi-Objective Agentic Rewrites for Unstructured Data Processing" — VLDB 2026 ([arXiv:2512.02289](https://arxiv.org/abs/2512.02289))
- **BARGAIN**: "Cut Costs, Not Accuracy: LLM-Powered Data Processing with Guarantees" — SIGMOD 2026 ([arXiv:2509.02896](https://arxiv.org/abs/2509.02896))

---

## 2. Interfaces & Configuration

### Two First-Class Interfaces

| Interface | Description | Execution |
|---|---|---|
| **YAML** | Declarative config file; declare datasets, operations, pipeline steps, output | `docetl run pipeline.yaml` |
| **Python (Frame API)** | Chainable methods on `Frame` objects; lazy until terminal action | `.collect()` / `.to_pandas()` / `.write_json()` |

The two are fully convertible: `frame.to_yaml("pipeline.yaml")` and `docetl.Frame.from_yaml("pipeline.yaml")`.

### Global Configuration (Python)

Module-level attributes on `docetl`:

| Attribute | Type | Default | Description |
|---|---|---|---|
| `docetl.default_model` | `str` | `None` | Default LLM model for all operations |
| `docetl.agent_model` | `str` | `None` | Model for optimizer rewrites |
| `docetl.max_threads` | `int` | `cpu_count * 4` | Concurrent threads |
| `docetl.bypass_cache` | `bool` | `False` | Skip LLM cache |
| `docetl.intermediate_dir` | `str` | `None` | Directory for intermediate results |
| `docetl.rate_limits` | `dict` | `None` | Rate limits per model |
| `docetl.fallback_models` | `list[str]` | `None` | Fallback chain on failure |
| `docetl.fallback_embedding_models` | `list[str]` | `None` | Fallback embedding models |
| `docetl.system_prompt` | `dict` | `None` | `{"dataset_description": ..., "persona": ...}` applied to all operations |
| `docetl.plan_rewrites` | `bool \| list[str]` | `True` | Enable/select plan rewrite rules |

**Precedence:** per-operation parameter > per-pipeline Frame settings > module-level `docetl.*` globals > built-in defaults.

### YAML Configuration

```yaml
default_model: gpt-4o-mini
bypass_cache: true  # optional, defaults to false
default_lm_api_base: https://your-custom-llm-endpoint.com/v1  # for self-hosted
default_embedding_api_base: https://your-custom-embedding-endpoint.com/v1

system_prompt:
  dataset_description: a collection of transcripts of doctor visits
  persona: a medical practitioner analyzing patient symptoms

datasets:
  interviews:
    type: file
    path: interviews.json

operations:
  - name: extract_themes
    type: map
    prompt: "List the themes discussed: {{ input.transcript }}"
    output:
      schema:
        themes: list[str]

pipeline:
  steps:
    - name: analyze
      input: interviews
      operations: [extract_themes]
  output:
    type: file
    path: themes.json
    intermediate_dir: intermediate_data  # optional
```

### Caching

DocETL caches all LLM calls and partially-optimized plans in `~/.cache/docetl/general` and `~/.cache/docetl/llm`. Rerunning similar operations avoids redundant API calls. Clear with `docetl clear-cache`. Set `bypass_cache: true` (global) or per-operation to skip.

---

## 3. Datasets & Frames — Data Ingestion

### Readers (Python API → Frame)

| Function | Description |
|---|---|
| `docetl.read_json(path)` | Load from a JSON file (list of objects; each object = one row) |
| `docetl.read_csv(path)` | Load from a CSV file (each row = one row; column names → keys) |
| `docetl.read_parquet(path)` | Load from a Parquet file |
| `docetl.read_dir(path)` | One row per file in a directory (recursive); `text` = file content, plus `filename` and `path`. PDF/Word/PPT/Excel auto-converted to text; others read as UTF-8 |
| `docetl.from_list(data)` | Load from an in-memory list of dicts |
| `docetl.Frame.from_yaml(path)` | Load from a YAML pipeline config |

### YAML Dataset Definition

```yaml
datasets:
  reviews:
    type: file
    path: "reviews.json"
  contracts:
    type: file
    path: "contracts"       # a directory
  audio_transcripts:
    type: file
    source: local
    path: "audio_files/audio_paths.json"
    parsing_tools:
      - input_key: audio_path
        function: whisper_speech_to_text
        output_key: transcript
```

### Parsing Tools (Non-Standard Inputs)

Built-in parsing functions (e.g., `whisper_speech_to_text` for audio) and custom-registered parsers can be attached to datasets:

```python
audio = docetl.read_json(
    "audio_paths.json",
    parsing=[{"input_key": "audio_path", "function": "whisper_speech_to_text", "output_key": "transcript"}],
)
```

- `input_key`: key holding the path to the file to parse
- `function`: parsing function name (built-in or custom)
- `output_key`: key the parsed content is stored under (accessible as `{{ input.transcript }}`)

### Frame Properties

- **Lazy** — operations are recorded but not executed until a terminal action.
- **Immutable** — every operation returns a new `Frame`.
- **Memoized** — terminal actions are cached on the Frame; repeated calls with unchanged config reuse results.

Relative paths resolve against the working directory at run time, not the YAML/script location.

---

## 4. Output Schemas — Type System

Every LLM call has an output schema specifying structure and types. Enforced via OpenAI tools/function-calling API (default "tools" mode) or LiteLLM structured output mode ("structured_output" mode).

### Supported Types

| Type | Aliases | Description |
|---|---|---|
| `string` | `str`, `text`, `varchar` | Text data |
| `integer` | `int` | Whole numbers |
| `number` | `float`, `decimal` | Decimal numbers |
| `boolean` | `bool` | True/false values |
| `enum` | — | A set of possible values: `enum[positive, negative, neutral]` |
| `list` | — | Arrays; must specify element type: `list[string]`, `list[{name: str, age: int}]` |
| objects | — | Using `{field: type}` notation |

### Schema Definition

```yaml
output:
  schema:
    summary: string
    sentiment: "enum[positive, negative, neutral]"
    key_entities: "list[{name: string, description: string}]"
    metadata: "{timestamp: string, source: string}"
  mode: structured_output  # optional: "tools" (default) or "structured_output"
```

```python
output={
    "schema": {
        "summary": "string",
        "sentiment": "enum[positive, negative, neutral]",
        "key_entities": "list[{name: str, description: str}]",
    },
    "mode": "structured_output",
}
```

### Special Parameters

| Parameter | Type | Default | Description |
|---|---|---|---|
| `output.schema` | `dict` | — | Schema field definitions |
| `output.mode` | `str` | `"tools"` | `"tools"` (OpenAI function-calling) or `"structured_output"` (JSON schema) |
| `output.n` | `int` | `1` | Number of outputs per input (OpenAI only; multiplies dataset by n for synthetic data generation) |
| `output.lineage` | `list[str]` | — | (reduce only) Tracks original input data keys for debugging/auditing |

### Best Practice

Keep schemas simple — complex nested structures degrade LLM output quality. Break complex schemas into multiple simpler operations. Quote type strings that contain curly braces in YAML.

---

## 5. Operators — Validation & Gleaning

### Common Operator Attributes

All operators share:
- `name`: Unique identifier
- `type`: Operation type string

LLM-based operators additionally share:
- `prompt`: Jinja2 template defining the LLM instruction
- `output`: Output schema definition
- `model` (optional): Override the pipeline's `default_model`
- `litellm_completion_kwargs` (optional): Additional LiteLLM completion parameters (e.g., `max_tokens`, `temperature`, `top_p`)
- `timeout`: Per-LLM-call timeout in seconds (default 120)
- `max_retries_per_timeout`: Max retries per timeout (default 2)
- `bypass_cache`: Skip cache for this operation (default false)
- `skip_on_error`: Continue on LLM error (default false)

### Input Truncation

When input exceeds the LLM's token limit, DocETL automatically truncates tokens from the **middle** of the input, preserving beginning and end (which often contain crucial context). A warning is displayed.

### Basic Validation

The `validate` field accepts Python expressions (YAML: strings; Python: callables or strings) that evaluate against the `output` dict. If any expression fails, the LLM is retried up to `num_retries_on_validate_failure` times.

```yaml
validate:
  - len(output["main_topic"].split()) <= 3
  - output["sentiment"] in ["positive", "negative", "neutral"]
  - 1 <= output["credibility_score"] <= 10
num_retries_on_validate_failure: 2
```

```python
validate=[
    lambda output: len(output["main_topic"].split()) <= 3,
    lambda output: output["sentiment"] in ["positive", "negative", "neutral"],
    lambda output: 1 <= output["credibility_score"] <= 10,
],
num_retries_on_validate_failure=2,
```

Note: `input` fields are not accessible in validation, but output docs carry through all input fields (for non-reduce operations).

### Advanced Validation: Gleaning

Gleaning uses an **LLM-based validator** to iteratively refine outputs. It at least doubles the number of LLM calls per operator.

```yaml
gleaning:
  if: "len(output['insights_summary']) < 10"  # optional: skip gleaning if false
  num_rounds: 2                                # max refinement iterations
  model: gpt-4o-mini                           # optional: validator model (defaults to op model)
  validation_prompt: |
    There should be at least 2 insights, and each insight
    should have at least 1 supporting action.
```

```python
gleaning={
    "if": "len(output['insights_summary']) < 10",
    "num_rounds": 2,
    "model": "gpt-4o-mini",
    "validation_prompt": "There should be at least 2 insights...",
}
```

**How gleaning works:**
1. Initial LLM call generates output.
2. Validation prompt is appended to chat thread and submitted to LLM.
3. LLM responds with assessment.
4. If improvements suggested, a new prompt (original + output + feedback) generates a refined output.
5. Steps 2-4 repeat until the validator has no more feedback or `num_rounds` is exceeded.
6. The last refined output is returned.

**Per-prompt gleaning** (parallel_map only): Each prompt in a parallel_map can have its own `gleaning` config scoped to its individual LLM call.

---

## 6. Map Operator

Applies a transformation to each item independently — 1:1 input-to-output ratio.

```
doc 1 → doc 1 + new fields
doc 2 → doc 2 + new fields
doc 3 → doc 3 + new fields
```

### Required Parameters

| Parameter | Description |
|---|---|
| `name` | Unique operation name |
| `type` | `"map"` |
| `prompt` | Jinja2 template; access input via `{{ input.keyname }}` |
| `output.schema` | Output schema definition |

*(If `drop_keys` is specified, `prompt` and `output` become optional — the map acts as a no-op key remover.)*

### Optional Parameters

| Parameter | Description | Default |
|---|---|---|
| `batch_prompt` | Template for processing multiple docs in a single prompt; access batch via `inputs` list | None |
| `max_batch_size` | Max docs per batch | None |
| `model` | LLM model | `default_model` |
| `optimize` | Enable operation optimization | `True` |
| `recursively_optimize` | Recursive optimization of synthesized operators | `false` |
| `sample` | Number of samples to process | All data |
| `limit` | Max outputs before stopping | All data |
| `agent` | (Python only) `docetl.Agent` config for tool-equipped map | None |
| `validate` | List of validation expressions/callables | None |
| `num_retries_on_validate_failure` | Retries on validation failure | 0 |
| `gleaning` | Gleaning config | None |
| `drop_keys` | Keys to drop from input (no LLM calls if only this is set) | None |
| `timeout` | Per-call timeout (seconds) | 120 |
| `max_retries_per_timeout` | Max retries per timeout | 2 |
| `litellm_completion_kwargs` | Additional LiteLLM params | `{}` |
| `skip_on_error` | Skip on LLM error | `False` |
| `bypass_cache` | Bypass cache for this op | `False` |
| `pdf_url_key` | Key containing PDF URL (Claude/Gemini models process PDFs directly) | None |
| `calibrate` | Improve consistency using sample data as reference anchors | `False` |
| `num_calibration_docs` | Docs to sample for calibration | 10 |
| `retriever` | Name/object of a retriever for RAG | None |
| `save_retriever_output` | Save retrieved context to `_<op>_retrieved_context` | `False` |
| `flush_partial_results` | Write batch results to disk incrementally | `False` |

### Example

```yaml
- name: analyze_news_article
  type: map
  prompt: |
    Analyze the following news article: "{{ input.article }}"
    Provide: main topic, summary, key entities, sentiment, biases, categories, credibility score.
  output:
    schema:
      main_topic: string
      summary: string
      key_entities: list[object]
      sentiment: string
      biases: list[string]
      categories: list[string]
      credibility_score: integer
  model: gpt-4o-mini
  validate:
    - len(output["main_topic"].split()) <= 3
    - output["sentiment"] in ["positive", "negative", "neutral"]
    - 1 <= output["credibility_score"] <= 10
  num_retries_on_validate_failure: 2
```

```python
frame = docetl.read_json("articles.json")
frame = frame.map(
    prompt="Analyze: {{ input.article }} ...",
    output={"schema": {"main_topic": "string", "summary": "string", ...}},
    model="gpt-4o-mini",
    validate=[lambda o: len(o["main_topic"].split()) <= 3, ...],
    num_retries_on_validate_failure=2,
)
rows = frame.collect()
```

### Batch Processing

Process multiple documents in a single LLM call to reduce call counts:

```yaml
- name: classify_documents
  type: map
  max_batch_size: 5
  batch_prompt: |
    Classify each document (technology, business, or science):
    {% for doc in inputs %}Document {{loop.index}}: {{doc.text}}{% endfor %}
  prompt: "Classify: {{input.text}}"  # fallback for unparseable batch responses
  output:
    schema:
      category: string
```

### Calibration for Consistency

With `calibrate: true`, the operation: (1) samples N docs (seed=42), (2) processes them with the original prompt, (3) an LLM analyzes results and generates reference anchors, (4) appends anchors to the prompt for all documents, (5) executes. Useful for classification, rating/scoring, and subjective judgment tasks needing consistent scales across documents.

### Synthetic Data Generation

Set `output.n > 1` to generate multiple outputs per input, returned as separate items (multiplies dataset size by n). OpenAI models only.

### PDF Processing

Specify `pdf_url_key` to process PDFs directly with Claude or Gemini models. DocETL downloads each PDF, extracts text, and passes it to the LLM with your prompt.

### Input Truncation & Dropping Keys

Automatic middle-truncation when input exceeds token limit. `drop_keys` removes specified keys with no LLM calls (no-op map).

---

## 7. Filter Operator

Behaves like Map, except items whose boolean output evaluates to false are **dropped** from the dataset.

```
doc 1 → keep?
doc 2 → keep?
doc 3 → keep?
        ↓
doc 1 (kept) + doc 3 (kept)
```

### Required Parameters

| Parameter | Description |
|---|---|
| `name` | Unique operation name |
| `type` | `"filter"` |
| `prompt` | Jinja2 template; access input via `{{ input.keyname }}` |
| `output` | Schema with **exactly one boolean field** |

### Optional Parameters

Inherits all map optional parameters (including `batch_prompt`, `max_batch_size`, `agent`, `validate`, `gleaning`, `cascade`, `retriever`).

| Parameter | Description | Default |
|---|---|---|
| `limit` | Counts only retained docs (boolean `true`); stops scheduling when N passing docs collected | All data |
| `cascade` | Model cascade config for cost reduction (see §24) | None |

### Example

```yaml
- name: filter_high_impact
  type: filter
  prompt: |
    Analyze: Title: "{{ input.title }}" Content: "{{ input.content }}"
    Respond with 'true' if high-impact (meets 3+ criteria), else 'false'.
  output:
    schema:
      is_high_impact: boolean
  model: gpt-4-turbo
  validate:
    - isinstance(output["is_high_impact"], bool)
```

```python
frame = frame.filter(
    prompt="Is this high-impact? {{ input.content }}",
    output={"schema": {"is_high_impact": "boolean"}},
    model="gpt-4-turbo",
    validate=["isinstance(output['is_high_impact'], bool)"],
)
```

The boolean field is dropped from kept rows after filtering.

---

## 8. Reduce Operator

Aggregates data based on a key. Groups all items sharing the same `reduce_key` value and produces one output per group. Supports batch reduction and incremental folding for large datasets.

```
doc (key=A) ─┐
doc (key=A) ─┤── one output for A
doc (key=B) ─┤── one output for B
doc (key=C) ─┘── one output for C
```

### Required Parameters

| Parameter | Description |
|---|---|
| `type` | `"reduce"` |
| `reduce_key` | Key (or list of keys) to group by. Use `"_all"` for one group containing all data |
| `prompt` | Jinja2 template; iterate over `inputs` list: `{% for item in inputs %}` |
| `output` | Output schema |

### Optional Parameters

| Parameter | Description | Default |
|---|---|---|
| `sample` | Number of samples | None |
| `limit` | Max groups to process (smallest groups first); truncates output to N | All groups |
| `synthesize_resolve` | If false, won't auto-synthesize a resolve between map and reduce | `true` |
| `model` | LLM model | `default_model` |
| `input` | Schema/keys to subselect from each item | All keys |
| `pass_through` | Pass through non-reduced keys from first item in group | `false` |
| `associative` | If true, reduce is associative (order doesn't matter) | `true` |
| `fold_prompt` | Prompt for incremental folding | None |
| `fold_batch_size` | Items per fold iteration | None |
| `value_sampling` | Sampling strategy for large groups | None |
| `verbose` | Detailed logging | `false` |
| `persist_intermediates` | Persist intermediate results to `_{op}_intermediates` | `false` |
| `agent` | (Python only) Tool-equipped agent | None |
| `retriever` | Retriever for RAG (context retrieved once per group) | None |
| `save_retriever_output` | Save retrieved context | `False` |
| `timeout` / `max_retries_per_timeout` / `litellm_completion_kwargs` / `bypass_cache` | Standard | — |

### Incremental Folding

For large groups, provide `fold_prompt` and `fold_batch_size` to process in smaller batches:

```yaml
- name: large_data_reduce
  type: reduce
  reduce_key: category
  prompt: |
    Summarize the data for category {{ inputs[0].category }}:
    {% for item in inputs %}Item {{ loop.index }}: {{ item.data }}{% endfor %}
  fold_prompt: |
    Combine the following summaries for category {{ inputs[0].category }}:
    Current summary: {{ output.summary }}
    New data: {% for item in inputs %}{{ item.data }}{% endfor %}
  fold_batch_size: 100
  output:
    schema:
      summary: string
```

### Scratchpad Technique

Incremental reduce maintains an internal "scratchpad" for intermediate state not represented in the output (e.g., tracking features liked once to find those liked more than once). Each fold's LLM call receives the current scratchpad, accumulated output, and new batch. Users only write reduce and fold prompts.

### Value Sampling

For very large groups, process a representative subset:

| Method | Description |
|---|---|
| `random` | Randomly select a subset |
| `first_n` | Select the first N values |
| `cluster` | K-means clustering to select representative samples |
| `sem_sim` (`semantic_similarity`) | Select samples based on semantic similarity to a query |

```yaml
value_sampling:
  enabled: true
  method: cluster
  sample_size: 50
```

For semantic similarity:
```yaml
value_sampling:
  enabled: true
  method: sem_sim
  sample_size: 30
  embedding_model: text-embedding-3-small
  embedding_keys: [review]
  query_text: "Battery life and performance"
```

### Lineage

Track original input data for each output for debugging/auditing:

```yaml
output:
  schema:
    summary: string
  lineage:
    - product_id
```

Results include a list of all `product_id`s per category under `summarize_reviews_by_category_lineage`.

---

## 9. Parallel Map Operator

Applies multiple independent transformations to each item concurrently, maintaining 1:1 input-to-output ratio. Multiple prompts run in parallel without an explicit DAG.

```
doc → prompt A → doc + field a + field b
    → prompt B ↗
```

### Required Parameters

| Parameter | Description |
|---|---|
| `name` | Unique operation name |
| `type` | `"parallel_map"` |
| `prompts` | List of prompt configs (see below) |
| `output` | Combined output schema (all fields from all prompts) |

Each prompt config:
- `prompt`: The prompt template
- `output_keys`: List of keys this prompt generates
- `model` (optional): Model for this specific prompt
- `gleaning` (optional): Per-prompt gleaning config (scoped to this prompt's LLM call)

### Optional Parameters

| Parameter | Default |
|---|---|
| `model` (default model) | `default_model` |
| `optimize` | `True` |
| `recursively_optimize` | `false` |
| `sample` | All data |
| `timeout` | 120 |
| `max_retries_per_timeout` | 2 |
| `litellm_completion_kwargs` | `{}` |

### Example

```yaml
- name: process_job_application
  type: parallel_map
  prompts:
    - name: extract_skills
      prompt: "Given the resume: '{{ input.resume }}', list top 5 skills."
      output_keys: [skills]
      gleaning:
        num_rounds: 1
        validation_prompt: "Confirm exactly 5 distinct skills, 1-2 words each."
      model: gpt-4o-mini
    - name: calculate_experience
      prompt: "Based on resume: '{{ input.resume }}', calculate years of experience."
      output_keys: [years_experience]
      model: gpt-4o-mini
    - name: evaluate_cultural_fit
      prompt: "Analyze cover letter: '{{ input.cover_letter }}'. Rate cultural fit 1-10."
      output_keys: [cultural_fit_score]
      model: gpt-4o-mini
  output:
    schema:
      skills: list[string]
      years_experience: float
      cultural_fit_score: integer
```

**Key constraint:** prompts must be truly independent — they cannot reference each other's outputs.

---

## 10. Resolve Operator

Identifies and canonicalizes duplicate entities. LLM-generated fields and multi-source data often refer to the same entity inconsistently; resolve standardizes them before further analysis.

```
doc: 'Jon Smith'  ─┐
doc: 'John Smith' ─┤→ doc: 'John Smith'
doc: 'Alice Wong' ─┘→ doc: 'Alice Wong'
```

### How It Works

1. **Blocking** — Reduces comparisons by only comparing entries likely to match (two methods: code-based conditions + embedding-based similarity threshold).
2. **Pair Generation** — All eligible pairs are generated.
3. **Batch Processing** — Pairs processed in batches (`compare_batch_size`).
4. **LLM Comparison** — For each batch, the LLM compares pairs and determines matches.
5. **Union-Find Clustering** — Matching pairs trigger cluster merges (Disjoint Set Union algorithm).
6. **Resolution** — For each cluster, the `resolution_prompt` generates a standardized value.

### Blocking

**Automatic blocking:** If no blocking config is specified, DocETL auto-computes an optimal embedding-based blocking threshold at runtime (samples pairs, runs LLM comparisons, finds threshold achieving 95% recall by default — adjustable via `blocking_target_recall`).

**Two blocking methods (union of pairs):**
1. **Code-based blocking** — Python expressions in `blocking_conditions` that determine if a pair should be compared.
2. **Embedding-based blocking** — Embeddings of `blocking_keys` fields; pairs with similarity above `blocking_threshold` are compared.

```yaml
blocking_keys: [last_name, date_of_birth]
blocking_threshold: 0.8
blocking_conditions:
  - "input1['last_name'][:2].lower() == input2['last_name'][:2].lower()"
  - "input1['date_of_birth'] == input2['date_of_birth']"
```

### Required Parameters

| Parameter | Description |
|---|---|
| `type` | `"resolve"` |
| `comparison_prompt` | Jinja2 template comparing `{{ input1 }}` and `{{ input2 }}` |
| `resolution_prompt` | Jinja2 template for reducing matched entries; iterate over `{{ inputs }}` |
| `output` | Output schema for the standardized value |

### Optional Parameters

| Parameter | Description | Default |
|---|---|---|
| `embedding_model` | Model for blocking embeddings | `default_model` |
| `resolution_model` | Model for resolution step | `default_model` |
| `comparison_model` | Model for comparison step | `default_model` |
| `blocking_keys` | Keys for initial blocking | All keys |
| `blocking_threshold` | Embedding similarity threshold | Auto-computed |
| `blocking_target_recall` | Target recall for auto-computed threshold | 0.95 |
| `blocking_conditions` | List of Python conditions for code-based blocking | `[]` |
| `input` | Schema/keys to subselect | All keys |
| `embedding_batch_size` | Entries per embedding batch | 1000 |
| `compare_batch_size` | Entity pairs per comparison batch | 500 |
| `limit_comparisons` | Max total comparisons | None |
| `cascade` | Model cascade config (see §24) | None |
| `sample` / `timeout` / `max_retries_per_timeout` / `litellm_completion_kwargs` / `bypass_cache` | Standard | — |

### Example

```yaml
- name: standardize_patient_names
  type: resolve
  optimize: true
  comparison_prompt: |
    Compare:
    Patient 1: {{ input1.patient_name }}, DOB: {{ input1.date_of_birth }}
    Patient 2: {{ input2.patient_name }}, DOB: {{ input2.date_of_birth }}
    Are these the same patient? Respond "True" or "False".
  resolution_prompt: |
    Standardize these patient names:
    {% for entry in inputs %}{{ entry.patient_name }}{% endfor %}
    Provide a single standardized name (LastName, FirstName MiddleInitial).
  output:
    schema:
      patient_name: string
```

```python
frame = frame.resolve(
    optimize=True,
    comparison_prompt="Compare: {{ input1.patient_name }} vs {{ input2.patient_name }} ...",
    resolution_prompt="Standardize: {% for e in inputs %}{{ e.patient_name }}{% endfor %} ...",
    output={"schema": {"patient_name": "string"}},
)
```

---

## 11. Equijoin Operator (Experimental)

Joins two datasets based on LLM-evaluated criteria — matches based on semantic similarity or complex conditions rather than exact equality. Uses the same techniques as Resolve.

```
left 1 + right 2 (matched)
left 2 + right 1 (matched)
...
```

### Required Parameters

| Parameter | Description |
|---|---|
| `type` | `"equijoin"` |
| `comparison_prompt` | Jinja2 template; reference `{{ left }}` and `{{ right }}` |

### Equijoin-Specific Parameters

| Parameter | Description | Default |
|---|---|---|
| `limits` | Max matches per side: `{"left": n, "right": m}` | No limit |
| `blocking_keys` | Keys for embedding blocking: `{"left": [...], "right": [...]}` | All keys |
| `blocking_threshold` | Embedding similarity threshold | Auto-computed |
| `blocking_target_recall` | Target recall for auto-computed threshold | 0.95 |
| `cascade` | Model cascade config (default guarantee: `precision`) | None |

Shares all other parameters with Resolve. Key differences: `resolution_prompt` is not used; `blocking_keys` uses a dict with `left`/`right` keys.

### Example

```yaml
- name: match_candidates_to_jobs
  type: equijoin
  blocking_keys:
    left: [medicine]
    right: [extracted_medications]
  blocking_threshold: 0.3535
  embedding_model: text-embedding-3-small
  comparison_prompt: |
    Compare: {{ left.medicine }} vs {{ right.extracted_medications }}
    Determine if these refer to the same medication.
```

```python
frame = candidates.equijoin(
    job_postings,
    name="match_candidates_to_jobs",
    comparison_prompt="Compare: {{ left.skills }} vs {{ right.required_skills }} ...",
)
```

### Pipeline Integration (YAML)

```yaml
pipeline:
  steps:
    - name: match_step
      operations:
        - match_candidates_to_jobs:
            left: candidates
            right: job_postings
```

---

## 12. Rank Operator

Sorts documents based on specified criteria along a latent attribute. **Not** for top-k retrieval — for full sorting. Adapts algorithms from "Human-Powered Sorts and Joins" (VLDB 2012).

### Algorithm

1. **Initial Ranking**: Embedding-based (cosine similarity to ranking criteria) or Likert-scale (LLM rates each doc on 7-point scale in batches of 10, with calibration docs).
2. **"Picky Window" Refinement**: Starting from the bottom, a large window is presented to the LLM, which selects the top few docs (`num_top_items_per_window`); chosen docs move to the front. Window slides upward with overlapping segments.
3. **Output**: Each document gets a `_rank` field (1-indexed).

### Required Parameters

| Parameter | Description |
|---|---|
| `name` | Unique operation name |
| `type` | `"rank"` |
| `prompt` | Ranking criteria prompt (**not** a Jinja template) |
| `input_keys` | List of document keys to consider for ranking |
| `direction` | `"asc"` or `"desc"` |

### Optional Parameters

| Parameter | Description | Default |
|---|---|---|
| `model` | LLM model | `default_model` |
| `embedding_model` | Embedding model | `"text-embedding-3-small"` |
| `batch_size` | Docs per LLM batch rating (first pass) | 10 |
| `num_calibration_docs` | Calibration docs (first pass) | 10 |
| `initial_ordering_method` | `"likert"` or `"embedding"` | `"likert"` |
| `k` | Number of top items to focus on | All items |
| `call_budget` | Max LLM API calls during ranking | 10 |
| `num_top_items_per_window` | Top items LLM selects per window | 3 |
| `overlap_fraction` | Overlap between windows | 0.5 |
| `verbose` | Detailed logging | `False` |
| `timeout` / `bypass_cache` / `litellm_completion_kwargs` | Standard | — |

### Example

```yaml
- name: rank_by_controversy
  type: rank
  prompt: |
    Order these debates by controversy level. Consider disagreement,
    divisive topics, emotional language, conflicting viewpoints, public reaction.
  input_keys: [content, title, date]
  direction: desc
  rerank_call_budget: 10
  initial_ordering_method: likert
```

Rank has no dedicated Frame method — construct as a config dict and run with `DSLRunner`:

```python
from docetl.runner import DSLRunner
config = {
    "default_model": "gpt-4o-mini",
    "datasets": {"debates": {"type": "file", "path": "debates.json"}},
    "operations": [{"name": "rank_by_controversy", "type": "rank", "prompt": "...", "input_keys": ["content", "title", "date"], "direction": "desc", "rerank_call_budget": 10}],
    "pipeline": {"steps": [{"name": "ranking", "input": "debates", "operations": ["rank_by_controversy"]}], "output": {"type": "file", "path": "ranked.json"}},
}
runner = DSLRunner(config)
results, _ = runner.run()
```

### Performance

Scales with O(n). The embedding-based first pass reduces cost. Use `verbose` during development for call statistics.

---

## 13. Extract Operator

Pulls out sections of source text **verbatim**, without synthesis or summarization. Compared to Map: lower output token cost, exact text without hallucination, no output schema needed.

### Extraction Strategies

| Strategy | Description |
|---|---|
| `line_number` (default) | Reformats text with line numbers, asks LLM to identify relevant line ranges, extracts those ranges, removes line number prefixes, eliminates duplicates. Good for multi-line passages or entire sections. |
| `regex` | Asks LLM to generate regex patterns matching desired content, applies them to original text. Good for structured data (dates, codes, formatted info). |

### Output Formats

| `format_extraction` | Result |
|---|---|
| `true` (default) | Extracted segments joined with newlines into a single string |
| `false` | Each extracted segment remains separate in a list |

### Required Parameters

| Parameter | Description |
|---|---|
| `name` | Unique operation name |
| `type` | `"extract"` |
| `prompt` | Instructions specifying what to extract (**not** a Jinja template) |
| `document_keys` | List of document fields containing text to process |

### Optional Parameters

| Parameter | Description | Default |
|---|---|---|
| `model` | LLM model | `default_model` |
| `extraction_method` | `"line_number"` or `"regex"` | `"line_number"` |
| `format_extraction` | Join with newlines (`true`) or keep as list (`false`) | `true` |
| `extraction_key_suffix` | Suffix for output field names | `"*extracted*"` (field becomes `{key}_extracted_findings`) |
| `timeout` | Per-call timeout | 120 |
| `skip_on_error` | Continue on errors | `false` |
| `litellm_completion_kwargs` | Additional LiteLLM params | `{}` |
| `limit` | Max documents to extract from | All data |
| `retriever` | Retriever for RAG | None |
| `save_retriever_output` | Save retrieved context | `False` |

### Example

```yaml
- name: findings
  type: extract
  prompt: |
    Extract all sections discussing key findings, results, or conclusions.
    Focus on paragraphs summarizing experimental outcomes, statistical results,
    discovered insights, and conclusions.
  document_keys: [report_text]
  model: gpt-4.1-mini
  format_extraction: true
```

```python
frame = frame.extract(
    prompt="Extract key findings from this research report.",
    document_keys=["report_text"],
    model="gpt-4.1-mini",
)
```

The extracted content is added to each document with suffix `_extracted_findings`.

---

## 14. Cluster Operator

Groups all items into a binary tree using **agglomerative clustering** of embeddings of specified keys. Annotates each item with its path through the tree (reversed: most specific grouping first, ending at root). Each cluster is summarized with an LLM prompt.

### Required Parameters

| Parameter | Description |
|---|---|
| `name` | Unique operation name |
| `type` | `"cluster"` |
| `embedding_keys` | List of keys to use for the embedding that is clustered on |
| `summary_prompt` | Jinja2 prompt to summarize a cluster based on its children; iterate `inputs` with `{% for input in inputs %}` |
| `summary_schema` | Output schema for the summary (the `summary_prompt` LLM call output) |

### Optional Parameters

| Parameter | Description | Default |
|---|---|---|
| `output_key` | Name of output key where cluster path is inserted | `"clusters"` |
| `model` | LLM model | `default_model` |
| `embedding_model` | Embedding model | `"text-embedding-3-small"` |
| `timeout` | Per-call timeout | 120 |
| `max_retries_per_timeout` | Max retries per timeout | 2 |
| `sample` | Number of items to sample | None |
| `litellm_completion_kwargs` | Additional LiteLLM params | `{}` |

### Example

```yaml
- name: cluster_concepts
  type: cluster
  max_batch_size: 5
  embedding_keys: [concept, description]
  output_key: categories
  summary_schema:
    concept: str
    description: str
  summary_prompt: |
    The following describes two related concepts. What concept encompasses both?
    {% for input in inputs %}{{input.concept}}: {{input.description}}{% endfor %}
    Provide the title and description of the super-concept.
```

```python
frame = frame.cluster(
    name="cluster_concepts",
    max_batch_size=5,
    embedding_keys=["concept", "description"],
    output_key="categories",
    summary_schema={"concept": "str", "description": "str"},
    summary_prompt="What concept encompasses both? {% for input in inputs %}...{% endfor %}",
)
```

Output: each item gets a `categories` (or `clusters`) field — a list of cluster nodes from most specific to root, each with `distance`, and the fields defined in `summary_schema`.

---

## 15. Link Resolve Operator

Fixes links between items (e.g., in a knowledge graph). Examines every id in a link field; if no exact match exists among item ids, it compares using an LLM prompt to find a match. Unlike resolve, it is **one-sided** — assumes item ids are already canonical (e.g., from running resolve first).

```
doc: related_to=[Appl Inc] → doc: related_to=[Apple Inc]
doc: related_to=[]         → doc: related_to=[]
```

### Required Parameters

| Parameter | Description |
|---|---|
| `name` | Unique operation name |
| `type` | `"link_resolve"` |
| `id_key` | Key for item ids |
| `link_key` | Key to make replacements in |
| `blocking_threshold` | Embedding similarity threshold for considering matches |
| `comparison_prompt` | Jinja2 template; uses `{{ link_value }}`, `{{ id_value }}`, and `{{ item }}` |

### Optional Parameters

`embedding_model`, `comparison_model` (both default to `default_model`).

### Example

```yaml
- name: fix_links
  type: link_resolve
  id_key: title
  link_key: related_to
  blocking_threshold: 0.85
  embedding_model: text-embedding-ada-002
  comparison_model: gpt-4o-mini
  comparison_prompt: |
    Compare: [{{ link_value }}] vs [{{ id_value }}]
    Description: {{ item.description }}
    Are these the same concept? Respond "True" or "False".
```

Link resolve has no dedicated Frame method — use `DSLRunner` with a config dict.

---

## 16. Split Operator

Divides long text content into smaller chunks. Use when documents exceed token limits or when LLM accuracy degrades on long inputs.

```
long doc → chunk 1
         → chunk 2
         → chunk 3
```

### Required Parameters

| Parameter | Description |
|---|---|
| `type` | `"split"` |
| `split_key` | Key of the field containing text to split |
| `method` | `"delimiter"` or `"token_count"` |
| `method_kwargs` | Dict of method-specific kwargs |

### Method Kwargs

**Token Count Method:**
- `num_tokens` (int): Maximum tokens per chunk
- `model` (optional): Tokenizer to use (defaults to `default_model`)

**Delimiter Method:**
- `delimiter` (string): Delimiter to split on
- `num_splits_to_group` (optional, default 1): Number of splits to group together

### Optional Parameters

| Parameter | Description | Default |
|---|---|---|
| `sample` | Number of samples | None |

### Output

Each split generates a new item preserving all original key-value pairs, plus:
- `{split_key}_chunk`: Content of the chunk
- `{op_name}_id`: Unique identifier for the original document
- `{op_name}_chunk_num`: Sequential chunk number within the original document

### Example

```yaml
- name: split_transcript
  type: split
  split_key: transcript
  method: token_count
  method_kwargs:
    num_tokens: 500
    model: gpt-4o-mini
```

```python
frame = frame.split(
    split_key="transcript",
    method="token_count",
    method_kwargs={"num_tokens": 500, "model": "gpt-4o-mini"},
)
```

Delimiter example:
```yaml
- name: split_by_paragraphs
  type: split
  split_key: document
  method: delimiter
  method_kwargs:
    delimiter: "\n\n"
  num_splits_to_group: 3
```

### Typical Pattern

Split → Map (per chunk) → Reduce (per original document). For ordered chunks, set `associative: false` in the reduce.

---

## 17. Gather Operator

Complements Split by adding context from surrounding chunks to each chunk. Split chunks often lack context (e.g., references to "the Company" defined in earlier chunks).

```
chunk 1 ─ ─ ─┐
chunk 2 ────→ chunk 2 + rendered neighbors
chunk 3 ─ ─ ─┘
```

### How It Works

1. Identifies relevant surrounding chunks (peripheral context): preceding/following text, or summarized versions
2. Adds this context to each chunk
3. Preserves document structure via header hierarchies (`doc_header_key`)

### Required Parameters

| Parameter | Description |
|---|---|
| `type` | `"gather"` |
| `content_key` | Field containing chunk content |
| `doc_id_key` | Identifies chunks from the same original document |
| `order_key` | Sequence of chunks within a group |
| `peripheral_chunks` | How to include context from surrounding chunks |

### Optional Parameters

| Parameter | Description | Default |
|---|---|---|
| `doc_header_key` | Field representing extracted headers for each chunk | None |
| `sample` | Number of samples | None |

### Peripheral Chunks Configuration

Divided into `previous` and `next` sections, each with up to three subsections:

- `head`: First chunk(s) in the section (has `count`)
- `middle`: Chunks between head and tail (summary content via different `content_key`)
- `tail`: Last chunk(s) in the section (has `count`)

Each subsection can specify:
- `count`: Number of chunks (head/tail only)
- `content_key`: Key containing content to use (defaults to main content key)

```python
peripheral_chunks={
    "previous": {
        "head": {"count": 1, "content_key": "full_content"},
        "middle": {"content_key": "summary_content"},
        "tail": {"count": 2, "content_key": "full_content"},
    },
    "next": {
        "head": {"count": 1, "content_key": "full_content"},
    },
}
```

### Output

Adds a new field `{content_key}_rendered` containing: reconstructed header hierarchy, previous context, the main chunk (clearly marked), next context, and indications of skipped content between contexts.

### Header Handling

When `doc_header_key` is specified, the Gather operation includes all the most recent headers from higher levels found in previous chunks, ensuring each rendered chunk has its full header path even if higher-level headers aren't present in that chunk.

---

## 18. Unnest Operator

Expands an array field or dictionary into multiple items, so individual elements can be processed separately.

```
doc with items=[a, b] → doc with items=a
                      → doc with items=b
```

### Required Parameters

| Parameter | Description |
|---|---|
| `type` | `"unnest"` |
| `name` | Unique operation name |
| `unnest_key` | Key of the array/dict field to unnest |

### Optional Parameters

| Parameter | Description | Default |
|---|---|---|
| `keep_empty` | If true, empty arrays kept with value None | `false` |
| `expand_fields` | List of fields to expand from nested dict into parent | `[]` |
| `recursive` | Apply unnest recursively to nested arrays | `false` |
| `depth` | Max depth for recursive unnesting | `inf` |
| `sample` | Number of samples | None |

### Behavior

- **List-type unnesting:** Replaces the original key with each individual element from the list (generates multiple output items per input).
- **Dictionary-type unnesting:** Adds new keys to the parent dictionary based on `expand_fields` (original nested dict preserved).

### Example

```yaml
- name: unnest_quotes
  type: unnest
  unnest_key: salient_quotes
```

```python
frame = frame.unnest(unnest_key="salient_quotes")
```

Unnest has no output schema — it modifies data structure in place. All other original key-value pairs are preserved.

---

## 19. Sample & TopK Operators

### Sample

Takes a random subset of documents.

```python
frame = frame.sample(samples=100, method="random")
```

### TopK

Selects the top-K items based on a specified field.

*(These are auxiliary operators with limited parameter surfaces; see the operators documentation for details.)*

---

## 20. Code Operations

Define transformations using **Python code** rather than LLM prompts. No LLM calls are made. Use for deterministic, math-based, or library-based processing.

### Types

| Type | YAML `type` | Function Signature | Description |
|---|---|---|---|
| Code Map | `code_map` | `transform(doc) -> dict` | Apply Python function to each item independently |
| Code Reduce | `code_reduce` | `transform(items) -> dict` | Aggregate multiple items into a single result |
| Code Filter | `code_filter` | `transform(doc) -> bool` | Filter items based on Python logic (True = keep) |

### Required Parameters

| Parameter | Description |
|---|---|
| `type` | `"code_map"`, `"code_reduce"`, or `"code_filter"` |
| `code` | In YAML: Python source string defining `transform`; in Python: any callable |

### Optional Parameters

| Parameter | Description | Default |
|---|---|---|
| `drop_keys` | Keys to remove from output (code_map only) | None |
| `reduce_key` | Key(s) to group by (code_reduce only) | `"_all"` |
| `pass_through` | Pass through unmodified keys from first item (code_reduce only) | `false` |
| `concurrent_thread_count` | Number of threads | `os.cpu_count()` |
| `limit` | Max outputs (map: input docs; filter: passing docs; reduce: groups) | All data |

### Examples

```yaml
- name: extract_keywords
  type: code_map
  code: |
    def transform(doc) -> dict:
        keywords = doc['text'].lower().split()
        return {'keywords': keywords, 'keyword_count': len(keywords)}
```

```python
frame = frame.code_map(code=lambda doc: {"keywords": doc["text"].lower().split()})
frame = frame.code_filter(code=lambda doc: doc["score"] >= 0.5)
frame = frame.code_reduce(reduce_key="category", code=lambda items: {"total": sum(i["value"] for i in items)})
```

---

## 21. Tool-Equipped Agents

For Python pipelines, `map`, `filter`, and `reduce` operations can use `agent=docetl.Agent(...)` to call tools over multiple turns before returning structured output. DocETL adapts Python functions into OpenAI Agents SDK tools and routes the operation's LiteLLM model through the SDK's LiteLLM integration.

### `docetl.Agent`

```python
import docetl

@docetl.tool
def lookup_sla(customer_tier: str) -> dict[str, str | int]:
    """Return support entitlements for a customer tier."""
    return {"enterprise": {"response_hours": 1, "escalation": "page-on-call"}, ...}

agent = docetl.Agent(tools=[lookup_sla], max_turns=5, max_tool_calls=3)

frame = frame.map(
    prompt="Use lookup_sla to classify: {{ input.ticket }} / tier={{ input.customer_tier }}",
    output={"schema": {"priority": "str", "next_action": "str"}},
    model="azure/gpt-4o-mini",
    agent=agent,
)
```

### Key Properties

- The operation's `model=` still controls model selection.
- `@docetl.tool` wraps Python functions; plain Python tools execute as trusted Python in your process.
- OpenAI Agents SDK sandbox/native tools can be passed through.
- Agent configs are **Python-only** — cannot be exported to YAML.
- Filters with `agent=` cannot be combined with `cascade`.
- Reduce with `agent=` cannot be combined with gleaning.

### Agents-as-Tools (Specialist Subagents)

```python
specialist = docetl.Agent(
    instructions="Extract numeric evidence from the supplied text.",
    max_turns=4,
)

manager = docetl.Agent(
    tools=[
        specialist.as_tool(
            name="extract_numeric_evidence",
            description="Extract numeric evidence for the manager agent.",
        )
    ],
    max_turns=6,
)
```

### Hosted Sandbox (OpenAI-specific)

```python
sandbox = docetl.tools.Sandbox.create(
    name="docetl-research",
    network="disabled",
    memory_limit="1g",
)
bash = sandbox.bash()

agent = docetl.Agent(
    tools=[bash],
    max_turns=4,
    max_tool_calls=6,
)
```

For ephemeral shell: `docetl.tools.bash(...)` (uses `container_auto`). For provider-portable tooling: `@docetl.tool` Python functions, MCP tools, or provider-native SDK tools.

---

## 22. Retrievers (RAG)

A retriever indexes a dataset once and, for each item an operation processes, searches the index and injects top matches into the prompt as `{{ retrieval_context }}`. Built with **LanceDB** (local, no server). Supports full-text search (FTS), vector search (embedding), or hybrid (both combined).

### Configuration

```yaml
retrievers:
  kb_search:
    type: lancedb
    dataset: kb                      # what to index (dataset name or step output)
    index_dir: ./lance_index
    index_types: ["fts", "embedding"] # required: ["fts"], ["embedding"], or both
    fts:
      index_phrase: "{{ input.text }}"
      query_phrase: "{{ input.question }}"
    embedding:
      model: openai/text-embedding-3-small
      index_phrase: "{{ input.text }}"
      query_phrase: "{{ input.question }}"
    query:
      mode: hybrid                    # auto, fts, embedding, or hybrid
      top_k: 5
    build_index: if_missing           # if_missing, always, or never
```

```python
retriever = docetl.Retriever(
    data="knowledge_base.json",       # file path or list of dicts (alternative to dataset=)
    index_dir="./lance_index",
    index_types=["fts", "embedding"],
    fts={"index_phrase": "{{ input.text }}", "query_phrase": "{{ input.question }}"},
    embedding={"model": "openai/text-embedding-3-small", "index_phrase": "...", "query_phrase": "..."},
    query={"mode": "hybrid", "top_k": 5},
)
```

### Required Fields

| Field | Description |
|---|---|
| `type` | Must be `"lancedb"` |
| `dataset` (YAML) or `data`/`dataset` (Python) | What to index: dataset name, step output (`step_<op_name>`), file path, or list of dicts |
| `index_dir` | Path where LanceDB stores the index |
| `index_types` | `["fts"]`, `["embedding"]`, or `["fts", "embedding"]` |

### Optional Fields

| Field | Default | Description |
|---|---|---|
| `build_index` | `"if_missing"` | When to build: `if_missing`, `always`, or `never` |
| `query.mode` | `auto` | `fts`, `embedding`, or `hybrid` (auto-selects based on indexes) |
| `query.top_k` | `5` | Number of results to return |

### FTS Section

Required if `"fts"` is in `index_types`. Two Jinja templates (both required):
- `index_phrase`: Text stored in the index (runs per row of the **indexed** dataset; `input` is that row)
- `query_phrase`: Search query (runs per item the **operation** processes; `input` is that item; for `reduce`, runs per group with `reduce_key` and `inputs`)

### Embedding Section

Required if `"embedding"` is in `index_types`:
- `model`: Embedding model (required)
- `index_phrase`: Text to embed (optional; falls back to `fts.index_phrase`)
- `query_phrase`: Query text to embed (required)

### Attaching to Operations

| Parameter | Type | Default | Description |
|---|---|---|---|
| `retriever` | `str` / `Retriever` | — | Available on `map`, `filter`, `reduce`, `extract` |
| `save_retriever_output` | `bool` | `false` | Save retrieved context to `_<op_name>_retrieved_context` |

If the prompt doesn't use `{{ retrieval_context }}`, DocETL appends retrieved matches automatically. `retrieval_context` is truncated to ~1000 chars per retrieved doc.

### Indexing Previous Step Outputs

Retrievers can index the output of a previous pipeline step:

```yaml
retrievers:
  facts_index:
    type: lancedb
    dataset: extract_facts_step     # References output of a pipeline step!
```

In Python: `dataset="step_unnest_facts"` (step names are `step_<op_name>`).

---

## 23. Optimization — Plan Rewrites

Automatic, **equivalence-preserving rewrites** applied at the start of every run. Unlike MOAR, these are free wins — no offline search, no extra runs.

### Rules

**Selection pushdown** — Moves a filter below an LLM map that doesn't produce anything the filter reads:

```
map(1000 rows) → filter(keeps 200)   # before: 1000 LLM calls
filter(keeps 200) → map(200 rows)    # after: 200 LLM calls
```

Fires only when DocETL can *prove* the swap is safe: the filter's predicate must not reference any field the map writes, the map must be one-output-per-row with no reordering, and neither op's output may be shared. Fail-closed analysis — if a prompt or code predicate reads fields in a way that can't be statically enumerated, the rewrite doesn't fire.

**Limit pushdown** — Moves a positional head (`sample` with `method: first`) below one-to-one ops, so upstream LLM calls run only on surviving rows.

### Control

```yaml
plan_rewrites: false                    # off entirely
plan_rewrites: ["selection_pushdown"]   # select rules by name
```

```python
docetl.plan_rewrites = False  # all in-process runs
```

On by default. Misspelled rule names raise an error. Checkpoint hashes are computed over the rewritten pipeline.

### Inspection

```python
frame.explain(optimized=True)  # prints applied rewrites
frame.plan()                    # typed plan for programmatic inspection
```

---

## 24. Optimization — Model Cascades (BARGAIN)

For binary-output operators (`filter`, `resolve`, `equijoin`), a **model cascade** runs a cheap proxy model on all items/pairs, learns a confidence threshold from a small oracle-labeled sample, trusts the proxy above the threshold, and escalates the rest — while preserving a **statistical guarantee** that holds with probability `1 - delta`.

### Parameters

| Parameter | Type | Description | Default |
|---|---|---|---|
| `proxy_model` | string | The cheap model for the proxy pass (chat or embedding model). Required. | — |
| `guarantee` | string | `accuracy`, `precision`, `recall`, or `precision+recall` | operator-specific (filter→`recall`, resolve/equijoin→`precision`) |
| `target` | float | Target value for the guarantee metric, strictly in `(0, 1)`. Required. | — |
| `delta` | float | Failure probability; guarantee holds w.p. `1 - delta` | `0.05` |
| `label_budget` | int | Max oracle calls to learn the threshold (ignored for `precision+recall`) | `400` |

`target: 1.0` is rejected — use `0.99` instead.

### Guarantees

| Guarantee | Meaning | Procedure | Best for |
|---|---|---|---|
| `accuracy` | Output matches oracle on ≥ `target` fraction | BARGAIN_A | Any binary operator |
| `precision` | Of items returned positive, ≥ `target` are truly positive | BARGAIN_P | resolve/equijoin (don't over-merge) |
| `recall` | Of truly-positive items, ≥ `target` are returned | BARGAIN_R | filter (don't drop relevant docs) |
| `precision+recall` | Both precision and recall ≥ `target` jointly | BARGAIN_PR | When neither error direction is acceptable |

### Embedding Models as Proxy

`proxy_model` can be an embedding model (e.g., `text-embedding-3-small`):
- Embeds every item in batches
- Oracle-labels a training sample (half of `label_budget`, max 200) and fits logistic regression
- Uses regression probabilities as proxy scores
- Runs threshold search with remaining budget on disjoint rows

### Example

```yaml
- name: is_relevant
  type: filter
  model: gpt-4o                   # oracle
  prompt: "Is this about climate policy? {{ input.text }}"
  output: { schema: { keep: "bool" } }
  cascade:
    proxy_model: gpt-4o-mini
    guarantee: recall
    target: 0.95
    delta: 0.05
    label_budget: 300
```

```python
frame = frame.filter(
    model="gpt-4o",
    prompt="Is this about climate policy? {{ input.text }}",
    output={"schema": {"keep": "bool"}},
    cascade={"proxy_model": "gpt-4o-mini", "guarantee": "recall", "target": 0.95, "delta": 0.05, "label_budget": 300},
)
```

### Output Logging

```
Cascade filter 'is_relevant'
  proxy     gpt-4o-mini · 1000 scored · $0.0200
  oracle    gpt-4o · 137 sampled for calibration (budget 300) · $0.4000
  guarantee recall ≥ 95%  δ=0.05
  result    863 proxy-accepted + 137 calibration samples → 1000 items
  total cost $0.4200
```

Programmatic stats: `op.cascade_stats` (`n_items`, `proxy_calls`, `oracle_calls`, `escalation_rate`, `guarantee`, `target`, `delta`).

### Limitations

- Binary predictions only — `filter`, `resolve`, `equijoin`. Multiclass not supported.
- Chat-model proxies score via single-token logprobs (provider must return them).
- Cannot combine `cascade` with `retriever` or `pdf_url_key` (rejected at validation).
- Cannot combine `cascade` with `agent` on filters.

---

## 25. Optimization — MOAR (Multi-Objective Agentic Rewrites)

MOAR is an offline search optimizer that explores different pipeline configurations (changing models, adding validation steps, combining operations, decomposing operations, etc.) and evaluates each to find the best **cost-accuracy trade-offs**. Returns a frontier of Pareto-optimal plans.

### Basic Workflow

1. Create your pipeline (Python or YAML)
2. Write an evaluation function (Python)
3. Run optimization: `frame.optimize()` or `docetl build pipeline.yaml`
4. Review the cost-accuracy frontier

### Python API

```python
@docetl.register_eval
def evaluate(results):
    correct = sum(1 for r in results if r.get("correct"))
    return {"score": correct}

optimized = frame.optimize(
    eval_fn=evaluate,
    metric_key="score",
)
```

### `frame.optimize()` Parameters

| Parameter | Type | Default | Description |
|---|---|---|---|
| `eval_fn` | `Callable` | **required** | Scores pipeline output; returns a dict of metrics |
| `metric_key` | `str` | **required** | Key in eval_fn's return dict to optimize |
| `models` | `list[str]` | Auto-detect | LiteLLM model names to explore |
| `agent_model` | `str` | Auto-select | Model for rewrite agent |
| `max_iterations` | `int` | `20` | Search budget |
| `save_dir` | `str` | Temp dir | Where to save results |
| `exploration_weight` | `float` | `1.414` | UCB exploration constant |
| `dataset_path` | `str` | Pipeline's dataset | Sample dataset for optimization |
| `max_threads` | `int` | None | Max concurrent LLM calls per run |
| `max_concurrent_agents` | `int` | `3` | Parallel MCTS search agents |

Returns an optimized `Frame`. Access search results via `optimized.search_results`:

| Method/Property | Return Type | Description |
|---|---|---|
| `.best()` | `OptimizedPipeline` | Highest-accuracy solution |
| `.cheapest()` | `OptimizedPipeline` | Lowest-cost solution |
| `.frontier` | `list[OptimizedPipeline]` | All Pareto-optimal solutions |
| `.to_df()` | `DataFrame` | All explored plans |

### CLI

```bash
docetl build pipeline.yaml
```

### MOAR's Rewrites Include

- Decomposing operations (e.g., splitting a map over long documents into split → map-per-chunk → reduce)
- Adding gleaning or resolve steps
- Changing models
- Combining operations

### When to Use MOAR

- Finding cost-accuracy trade-offs across different models
- When you want multiple optimization options
- Custom evaluation metrics specific to your use case
- Exploring different pipeline configurations automatically

**Cost:** MOAR runs the pipeline many times on data samples, so a search takes tens of minutes. Requires OpenAI API keys for the optimizer agent.

### Evaluation Functions

```python
@docetl.register_eval
def evaluate(results):
    """Return a dict of metrics; the metric_key selects which to optimize."""
    correct = sum(1 for r in results if r.get("correct"))
    total = len(results)
    return {"accuracy": correct / total if total else 0, "count": total}
```

### MOAR + Plan Rewrites Integration

MOAR candidates are validated statically before execution. Each saved candidate is canonicalized with the same rewrite rules, honoring your `plan_rewrites` setting. Set `DOCETL_MOAR_PLAN_SUMMARY` env var to include the typed plan in the rewrite agent's context.

---

## 26. Python API Reference — Frame & Terminal Actions

### Reading Data

| Function | Description |
|---|---|
| `docetl.read_json(path)` | Load from JSON file → Frame |
| `docetl.read_csv(path)` | Load from CSV file → Frame |
| `docetl.read_parquet(path)` | Load from Parquet file → Frame |
| `docetl.read_dir(path)` | One row per file (recursive); PDF/Word/PPT/Excel auto-converted → Frame |
| `docetl.from_list(data)` | Load from list of dicts → Frame |
| `docetl.Frame.from_yaml(path)` | Load from YAML pipeline config → Frame |

### LLM Operations (return new Frame)

| Method | Description |
|---|---|
| `.map(prompt, output, ...)` | Apply LLM prompt to each document independently |
| `.filter(prompt, output, ...)` | Keep/remove documents based on LLM prompt (one boolean field) |
| `.reduce(reduce_key, prompt, output, ...)` | Group by key and reduce each group with LLM |
| `.resolve(comparison_prompt, resolution_prompt, output, ...)` | Deduplicate entities by pairwise LLM comparison |
| `.extract(prompt, document_keys, ...)` | Extract text verbatim with line-level precision |
| `.parallel_map(prompts, output, ...)` | Run multiple prompts on each document in parallel |
| `.equijoin(right, comparison_prompt, ...)` | Join two datasets by LLM comparison |

### Structural Operations (no LLM calls, return new Frame)

| Method | Description |
|---|---|
| `.split(split_key, method, method_kwargs)` | Split documents into chunks |
| `.gather(content_key, doc_id_key, order_key, peripheral_chunks, ...)` | Add surrounding context to chunks |
| `.unnest(unnest_key)` | Flatten a list/dict field into separate rows |
| `.cluster(embedding_keys, summary_prompt, summary_schema, ...)` | Cluster by embedding similarity |
| `.sample(samples, method)` | Sample a subset of documents |

### Code Operations (no LLM calls)

| Method | Description |
|---|---|
| `.code_map(code)` | Per-document Python transform (`code` = callable or source string) |
| `.code_filter(code)` | Per-document Python filter (return bool) |
| `.code_reduce(reduce_key, code)` | Per-group Python aggregation |

### Inspection (no execution)

| Method | Return Type | Description |
|---|---|---|
| `frame.schema()` | `dict[str, str]` | Output schema from operation definitions (best-effort; `code_*` outputs unknown statically) |
| `frame.count()` | `int` | Input count (no ops) or output count (executes if ops present) |
| `frame.to_yaml()` / `frame.to_yaml(path)` | `str` | Export pipeline as YAML config |
| `frame.to_python()` | `str` | Export as Python source code |
| `frame.explain(optimized=True)` | — | Print applied plan rewrites |
| `frame.plan()` | typed plan | Programmatic plan inspection |

### Terminal Actions (execute the pipeline)

| Method | Return Type | Description |
|---|---|---|
| `frame.show(max=5)` | `DataFrame` | Run on a sample and print results |
| `frame.collect()` | `list[dict]` | Execute, return result rows as list of dicts |
| `frame.to_pandas()` | `DataFrame` | Execute, return pandas DataFrame |
| `frame.write_json(path)` | `None` | Execute and write to JSON |
| `frame.write_csv(path)` | `None` | Execute and write to CSV |
| `frame.write_parquet(path)` | `None` | Execute and write to Parquet |

Terminal actions are memoized — repeated calls with unchanged config reuse results. Changing ops, in-memory data, or `docetl.*` settings invalidates the memo.

### Cost & Token Tracking

```python
rows = frame.collect()
print(f"Cost: ${frame.total_cost:.4f}")
print(f"Tokens: {frame.token_usage}")

df = frame.to_pandas()
print(f"Cost: ${df.attrs['_total_cost']:.4f}")  # also on the DataFrame
```

### Optimization

```python
optimized = frame.optimize(eval_fn=evaluate, metric_key="score")
```

(See §25 for full parameter reference.)

---

## 27. Pandas `.semantic` Accessor

Run DocETL operations on existing pandas DataFrames via the `.semantic` accessor:

```python
import pandas as pd
import docetl

docetl.default_model = "gpt-4o-mini"

posts = pd.DataFrame({
    "text": ["Just tried the new iPhone 15!", "Having issues with iOS 17", "Android is way better"],
    "timestamp": ["2024-01-01", "2024-01-02", "2024-01-03"],
})

# Extract structured data
analyzed = posts.semantic.map(
    prompt="Extract product and sentiment from: {{ input.text }}",
    output={"schema": {"product": "str", "sentiment": "str"}},
)

# Filter
relevant = analyzed.semantic.filter(prompt="Is this about Apple products? {{ input }}")

# Fuzzy group-by and summarize
summaries = relevant.semantic.agg(
    fuzzy=True,
    reduce_keys=["product"],
    comparison_prompt="Do these posts discuss the same product?",
    reduce_prompt="Summarize the feedback about this product",
    output={"schema": {"summary": "str", "frequency": "int"}},
)

print(f"Cost: ${summaries.semantic.total_cost:.4f}")
```

The `.semantic.agg()` method provides **fuzzy aggregation** — a fuzzy group-by that uses an LLM comparison prompt to determine if items belong in the same group, then reduces.

---

## 28. CLI Reference

### `docetl run`

Run a pipeline from a YAML configuration file.

```bash
docetl run pipeline.yaml [--max_threads N]
```

| Parameter | Type | Description | Default |
|---|---|---|---|
| `yaml_file` | `Path` | Path to YAML pipeline configuration | Required |
| `--max_threads` | `int` | Max threads for running operations | None |

Loads `.env` from the current working directory. The `DSLRunner` class loads, runs, and saves the pipeline.

### `docetl build`

Run the MOAR optimizer on a YAML pipeline.

```bash
docetl build pipeline.yaml
```

Explores different pipeline configurations, evaluates them with the configured evaluation function, and returns the best cost-accuracy trade-offs. Can also auto-generate blocking rules for equijoin.

### `docetl clear-cache`

Clear the LLM and plan cache.

```bash
docetl clear-cache
```

---

## 29. DocWrangler — Interactive IDE

[DocWrangler](https://docetl.org/playground) is an interactive development environment (IDE) built on top of DocETL, designed for building and refining LLM-powered data processing pipelines. A hosted version is available at [docetl.org/playground](https://docetl.org/playground) (no sign-up required).

### Key Features

#### Spreadsheet Interface with Automatic Summary Overlays

- Notebook-style pipeline editor combined with spreadsheet-like output viewer
- Resize rows/columns, search within cells, sort/filter columns
- Search functionality for validating extractions and checking for hallucinations
- Automatic visualizations based on data type:
  - Text fields: histograms of word/character counts
  - Lists: distributions of element counts
  - Categorical data: detected and visualized category distributions

#### In-Situ Fine-Grained Feedback and Intelligent Prompt Refinement

- Shows one row at a time while maintaining context; split views for examining source documents or intermediate outputs side-by-side
- Attach observations directly to specific outputs (e.g., "this summary missed key details")
- Analyzes row-level feedback to suggest targeted prompt improvements, showing clear diffs
- Iterate on suggestions or branch into different directions, with full version control

#### DocWrangler Assistant

- Context-aware AI assistant that understands your pipeline
- Explains concepts (Jinja templates, pipeline patterns) using examples from your own prompts
- Suggests structural improvements based on common pipeline patterns
- Note: Still an LLM chatbot that may occasionally hallucinate

### Tips for LLM-Powered Data Processing (from DocWrangler)

1. **Start with Data Inspection** — Upload data, inspect in viewer, set API keys, dataset description, and manageable sample size (10-20 documents)
2. **Start with Open-Ended Data Exploration** — Begin with broad, exploratory map operations; write prompts encouraging exhaustive responses; keep output schemas simple initially
3. **Iterate One Step at a Time** — Build operation by operation; examine outputs one example at a time; keep providing notes; use prompt improvement assistant; gradually add schema structure
4. **Handle Complexity** — For long documents or complex tasks, use the experimental optimize operation (data/task decomposition); for resolve operations, always run optimization

### Example Datasets

- US Supreme Court oral arguments from 2024 (~$0.11 with gpt-4o-mini)
- Airline customer support chat logs (~$0.05 with gpt-4o-mini)

### Privacy Note

If using AI features, chatbot and prompt engineering queries are logged for research. To opt out, provide your own OpenAI API key.

---

## 30. Capability Summary & Cross-Reference

### Capability Matrix

| Capability | Operators / Features | Interface | Key Parameters |
|---|---|---|---|
| **Per-document transformation** | Map | YAML / Python | `prompt`, `output.schema`, `model`, `validate`, `gleaning`, `batch_prompt`, `max_batch_size`, `calibrate`, `pdf_url_key`, `drop_keys`, `agent`, `retriever`, `limit`, `sample` |
| **Filtering** | Filter | YAML / Python | `prompt`, `output` (one boolean field), `cascade`, `limit`, `agent`, `retriever` |
| **Aggregation/grouping** | Reduce | YAML / Python | `reduce_key`, `prompt`, `output`, `fold_prompt`, `fold_batch_size`, `associative`, `value_sampling`, `lineage`, `agent`, `retriever`, `limit` |
| **Parallel multi-prompt** | Parallel Map | YAML / Python | `prompts` (list of `{prompt, output_keys, model?, gleaning?}`), `output` (combined schema) |
| **Entity deduplication** | Resolve | YAML / Python | `comparison_prompt`, `resolution_prompt`, `output`, `blocking_keys`, `blocking_threshold`, `blocking_conditions`, `cascade`, `compare_batch_size` |
| **Fuzzy join** | Equijoin | YAML / Python | `comparison_prompt`, `blocking_keys` (left/right), `blocking_threshold`, `limits`, `cascade` |
| **Ranking/sorting** | Rank | YAML (DSLRunner) | `prompt`, `input_keys`, `direction`, `initial_ordering_method`, `call_budget`, `num_top_items_per_window`, `overlap_fraction` |
| **Verbatim text extraction** | Extract | YAML / Python | `prompt`, `document_keys`, `extraction_method`, `format_extraction`, `extraction_key_suffix`, `retriever` |
| **Hierarchical clustering** | Cluster | YAML / Python | `embedding_keys`, `summary_prompt`, `summary_schema`, `output_key`, `embedding_model` |
| **Knowledge-graph link fixing** | Link Resolve | YAML (DSLRunner) | `id_key`, `link_key`, `blocking_threshold`, `comparison_prompt` |
| **Document chunking** | Split | YAML / Python | `split_key`, `method` (token_count/delimiter), `method_kwargs` (num_tokens/delimiter/num_splits_to_group) |
| **Context enrichment** | Gather | YAML / Python | `content_key`, `doc_id_key`, `order_key`, `peripheral_chunks` (previous/next × head/middle/tail), `doc_header_key` |
| **Array/dict flattening** | Unnest | YAML / Python | `unnest_key`, `keep_empty`, `expand_fields`, `recursive`, `depth` |
| **Sampling** | Sample | Python | `samples`, `method` |
| **Top-K selection** | TopK | Python | (field-based top-K) |
| **Deterministic transforms** | Code Map/Reduce/Filter | YAML / Python | `code` (callable or source string), `drop_keys`, `reduce_key`, `pass_through`, `concurrent_thread_count` |
| **Tool-equipped agents** | Agent (map/filter/reduce) | Python only | `docetl.Agent(tools=..., max_turns=..., max_tool_calls=...)`, `@docetl.tool`, `specialist.as_tool()`, `Sandbox.create()` |
| **RAG / retrieval** | Retriever | YAML / Python | `dataset`/`data`, `index_dir`, `index_types` (fts/embedding), `fts`, `embedding`, `query` (mode, top_k), `build_index` |
| **Schema enforcement** | Output Schemas | YAML / Python | `output.schema` (types), `output.mode` (tools/structured_output), `output.n`, `output.lineage` |
| **Output validation** | Validate / Gleaning | YAML / Python | `validate` (list of expressions/callables), `num_retries_on_validate_failure`, `gleaning` (num_rounds, validation_prompt, model, if) |
| **Automatic reordering** | Plan Rewrites | YAML / Python | `plan_rewrites` (bool or rule list): `selection_pushdown`, `limit_pushdown` |
| **Cost reduction (binary ops)** | Model Cascades (BARGAIN) | YAML / Python | `cascade`: `proxy_model`, `guarantee`, `target`, `delta`, `label_budget` |
| **Pipeline optimization** | MOAR | YAML (`docetl build`) / Python (`frame.optimize()`) | `eval_fn`, `metric_key`, `models`, `agent_model`, `max_iterations`, `exploration_weight`, `max_concurrent_agents` |
| **Pandas integration** | `.semantic` accessor | Python | `.semantic.map()`, `.semantic.filter()`, `.semantic.agg(fuzzy=...)` |
| **Interactive IDE** | DocWrangler | Web (docetl.org/playground) | Spreadsheet viewer, in-situ feedback, prompt refinement assistant, AI assistant |

### Operator Input/Output Semantics

| Operator | Input → Output | LLM Calls | Parallelizable |
|---|---|---|---|
| Map | 1 → 1 | Yes (per item) | Yes (across items) |
| Filter | 1 → 0 or 1 | Yes (per item) | Yes (across items) |
| Reduce | N (per group) → 1 | Yes (per group) | Yes (across groups) |
| Parallel Map | 1 → 1 (multiple prompts) | Yes (per prompt, per item) | Yes (prompts + items) |
| Resolve | N → ≤N (deduplicated) | Yes (pairwise + resolution) | Yes (batched pairs) |
| Equijoin | L × R → matched pairs | Yes (pairwise) | Yes (batched pairs) |
| Rank | N → N (sorted, +`_rank`) | Yes (per-doc rating + window refinement) | Yes (batched rating) |
| Extract | 1 → 1 (+extracted field) | Yes (per item) | Yes (across items) |
| Cluster | N → N (+cluster path) | Yes (per cluster summary) | Yes (batched summaries) |
| Link Resolve | N → N (links fixed) | Yes (per unmatched link) | Yes (batched) |
| Split | 1 → N (chunks) | No | Yes (per item) |
| Gather | 1 → 1 (+rendered context) | No | Yes (per item) |
| Unnest | 1 → N (or 1 with expand) | No | Yes (per item) |
| Code Map | 1 → 1 | No | Yes (threaded) |
| Code Reduce | N → 1 | No | Yes (per group) |
| Code Filter | 1 → 0 or 1 | No | Yes (threaded) |

### Key Architectural Concepts Summary

| Concept | Description |
|---|---|
| **Declarative pipeline** | Describe *what* (prompts, schemas, operators); DocETL handles orchestration |
| **Lazy Frame** | Operations recorded, not executed; runs on terminal action |
| **Jinja2 prompts** | Templates with `{{ input.* }}`, `{% for item in inputs %}`, `{{ retrieval_context }}` |
| **LiteLLM backbone** | 100+ LLM providers unified; model names passed through directly |
| **Caching** | LLM calls + optimized plans cached by default in `~/.cache/docetl/` |
| **Automatic truncation** | Middle-truncation when input exceeds token limit (preserves start/end) |
| **Automatic blocking** | Resolve/equijoin auto-compute embedding thresholds at 95% recall |
| **Scratchpad** | Incremental reduce maintains hidden intermediate state across folds |
| **Union-Find clustering** | Resolve groups matches with Disjoint Set Union algorithm |
| **BARGAIN guarantees** | Statistical quality bounds (accuracy/precision/recall) w.p. `1 - delta` |
| **MOAR frontier** | Pareto-optimal cost-accuracy trade-offs via MCTS search |
| **Plan rewrites** | Equivalence-preserving reordering (selection/limit pushdown) for free cost savings |
| **Tool agents** | Operations call Python/SDK tools before producing structured output |
| **LanceDB retrievers** | Local FTS/embedding/hybrid index for context augmentation |
| **Two-way conversion** | `frame.to_yaml()` ↔ `Frame.from_yaml()`; YAML ↔ Python fully interoperable |