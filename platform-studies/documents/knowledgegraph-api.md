# Knowledge Graph Platform Analysis — From Unstructured Documents to Queryable Graphs

> **Platforms analyzed:**
> 1. **[Docling-Graph](https://github.com/docling-project/docling-graph)** (MIT, IBM / LF AI & Data) — Pydantic-validated, schema-driven document-to-knowledge-graph pipeline
> 2. **[AI-Knowledge-Graph](https://github.com/robert-mcdermott/ai-knowledge-graph)** (Robert McDermott) — LLM-powered text-to-interactive-graph generator
> 3. **[Neo4j for AI Systems](https://neo4j.com/use-cases/ai-systems/)** (Neo4j Inc.) — Graph database + GraphRAG + Python driver for grounding AI

> **Description:** This analysis covers three complementary approaches to building knowledge graphs from unstructured data: (a) a schema-validated extraction pipeline that converts documents into typed Pydantic models and then directed NetworkX graphs with provenance; (b) a lightweight LLM-based triple-extraction tool that produces interactive HTML visualizations; and (c) a graph database platform with Cypher querying, vector search, and GraphRAG for grounding LLM-based AI systems.

---

## Table of Contents

1. [Platform Overview & Core Concepts](#1-platform-overview--core-concepts)
2. [Docling-Graph — Schema-Validated Extraction Pipeline](#2-docling-graph--schema-validated-extraction-pipeline)
3. [AI-Knowledge-Graph — LLM Triple Extraction & Visualization](#3-ai-knowledge-graph--llm-triple-extraction--visualization)
4. [Neo4j — Graph Database for AI Systems](#4-neo4j--graph-database-for-ai-systems)
5. [Cross-Platform Capability Comparison](#5-cross-platform-capability-comparison)
6. [Key Design Principles](#6-key-design-principles)

---

## 1. Platform Overview & Core Concepts

### Three Approaches to Knowledge Graph Construction

| Dimension | Docling-Graph | AI-Knowledge-Graph | Neo4j |
|---|---|---|---|
| **Purpose** | Document → validated Pydantic models → directed knowledge graph | Text → SPO triples → interactive HTML graph | Graph database + GraphRAG for AI grounding |
| **Schema** | Explicit Pydantic templates (entities, edges, identity fields) | Implicit (LLM-driven SPO extraction, no predefined schema) | Property graph model (nodes, relationships, properties) |
| **Extraction** | LLM or VLM backends via LiteLLM / Docling | LLM via OpenAI-compatible API | LLM via GraphRAG Python package / LangChain |
| **Graph format** | NetworkX `DiGraph` (in-memory) | NetworkX graph → PyVis HTML | Native graph database (Bolt protocol) |
| **Query language** | Python / NetworkX API | None (visualization only) | Cypher |
| **Export** | CSV, Cypher script, JSON, HTML | HTML (PyVis), JSON | Cypher, CSV import, JSON |
| **Provenance** | Deterministic ledger (chunk/page/bbox anchors) | Chunk index only | Native graph metadata + timestamps |
| **License** | MIT | MIT | Commercial (Community / Enterprise / Aura) |

### Core Concepts Across Platforms

- **Entity / Node** — A discrete thing (person, organization, concept, compound). In Docling-Graph, entities are declared via `model_config = {"is_entity": True}`; in Neo4j, nodes have labels and properties; in AI-Knowledge-Graph, entities are LLM-extracted subject/object strings.
- **Relationship / Edge** — A typed connection between entities. In Docling-Graph, edges are declared via the `edge()` helper on Pydantic fields; in Neo4j, relationships have types and properties; in AI-Knowledge-Graph, predicates are 1–3 word LLM-extracted phrases.
- **Schema / Template** — The contract defining what entities and relationships exist. Docling-Graph uses Pydantic `BaseModel` subclasses; Neo4j uses the property graph model; AI-Knowledge-Graph has no explicit schema.
- **Provenance / Grounding** — Linking graph nodes back to their source document location. Docling-Graph provides a deterministic `ProvenanceLedger`; AI-Knowledge-Graph tracks chunk indices; Neo4j relies on graph metadata.
- **Knowledge Graph** — A directed graph of entities and relationships that captures structured knowledge from unstructured sources, enabling high-precision retrieval and reasoning.

---

## 2. Docling-Graph — Schema-Validated Extraction Pipeline

### Main Concepts

Docling-Graph transforms documents into validated **Pydantic** objects, then builds a **directed knowledge graph** (NetworkX `DiGraph`) with explicit semantic relationships, stable node IDs, and deterministic provenance. It targets high-precision domains (chemistry, finance, legal) where exact entity connections matter more than approximate text embeddings.

Core abstractions:

- **PipelineConfig** — Type-safe (Pydantic `BaseModel`) configuration controlling the entire pipeline: source, template, backend, inference mode, processing mode, extraction contract, chunking, provenance, export, and dense-extraction tuning.
- **Pydantic Template** — A `BaseModel` subclass that defines both the extraction schema and the resulting graph structure. Entities are declared via `model_config = {"is_entity": True}`; identity fields via `graph_id_fields`; relationships via the `edge()` helper.
- **Extractor** — Strategy class (`OneToOneStrategy` or `ManyToOneStrategy`) that orchestrates document conversion (Docling) + backend extraction (LLM or VLM). Created via `ExtractorFactory.create_extractor()`.
- **Backend** — Extraction engine: `LlmBackend` (text-based, local or remote via LiteLLM) or `VlmBackend` (vision-based, local only via Docling).
- **GraphConverter** — Converts a list of validated Pydantic model instances into a NetworkX `DiGraph` with deterministic node IDs, typed edges, and optional reverse edges.
- **NodeIDRegistry** — Deterministic ID registry: the same entity always gets the same node ID across batches. Fingerprints derived from `graph_id_fields` (entities) or all non-empty fields (components).
- **ProvenanceLedger** — Deterministic data-grounding record mapping every graph node back to its source document chunk, page, and optional character-offset span. No LLM involvement.
- **Exporter** — Serializes the graph to CSV (Neo4j-compatible), Cypher script, or JSON (NetworkX node-link format).

### End-to-End Flow

```
input document (PDF, image, markdown, Office, DocLang…)
   │
   ▼
 DocumentProcessor (Docling)
   │  convert_to_docling_doc() → DoclingDocument
   │  extract_full_markdown() / extract_page_markdowns()
   ▼
 DocumentChunker (HybridChunker-based, structure-preserving)
   │  chunk_document() → List[str] (chunks ≤ chunk_max_tokens)
   ▼
 Extractor (OneToOneStrategy | ManyToOneStrategy)
   │  Backend: LlmBackend (LiteLLM) or VlmBackend (Docling)
   │  Contract: direct (single-pass) or dense (skeleton-then-flesh)
   ▼
 List[BaseModel]  (validated Pydantic model instances)
   │
   ▼
 GraphConverter.pydantic_list_to_graph()
   │  NodeIDRegistry → deterministic node IDs
   │  edge() declarations → typed edges
   │  provenance_binder → __provenance__ node attribute
   │  auto_cleanup → remove phantoms, duplicates, orphans
   ▼
 (nx.DiGraph, GraphMetadata)
   │
   ├──► Exporters (CSV / Cypher / JSON)
   ├──► InteractiveVisualizer (HTML)
   └──► ProvenanceLedger (provenance.json)
```

---

### 2.1 PipelineConfig — Complete Configuration API

**Module:** `docling_graph.config`
**Base class:** `pydantic.BaseModel`

#### Constructor

```python
config = PipelineConfig(
    source: Union[str, Path] = "",
    template: Union[str, Type[BaseModel]] = "",
    # Backend
    backend: Literal["llm", "vlm"] = "llm",
    inference: Literal["local", "remote"] = "local",
    model_override: str | None = None,
    provider_override: str | None = None,
    models: ModelsConfig = ModelsConfig(),
    llm_client: LLMClientProtocol | None = None,
    # Processing
    processing_mode: Literal["one-to-one", "many-to-one"] = "many-to-one",
    extraction_contract: Literal["auto", "direct", "dense"] = "auto",
    docling_config: Literal["ocr", "vision"] = "ocr",
    llm_input_format: Literal["markdown", "doclang", "doclang-geo", "auto"] = "auto",
    use_chunking: bool = True,
    chunk_max_tokens: int | None = None,
    debug: bool = False,
    parallel_workers: int | None = None,
    # Dense extraction
    dense_skeleton_batch_tokens: int = 2048,
    dense_fill_nodes_cap: int = 5,
    dense_fill_context: Literal["scoped", "full"] = "scoped",
    dense_dedupe: Literal["off", "standard", "aggressive"] = "standard",
    # Provenance
    provenance: Literal["off", "standard", "detailed"] = "standard",
    # Gleaning
    gleaning_enabled: bool = True,
    # Export
    dump_to_disk: bool | None = None,
    export_format: Literal["csv", "cypher"] = "csv",
    export_docling: bool = True,
    export_docling_json: bool = True,
    export_markdown: bool = True,
    export_doclang: bool = True,
    export_per_page_markdown: bool = False,
    # Graph
    reverse_edges: bool = False,
    # Output
    output_dir: Union[str, Path] = "outputs",
)
```

#### Field Reference

| Field | Type | Default | Description |
|---|---|---|---|
| `source` | `str \| Path` | `""` | Path or URL to source document |
| `template` | `str \| Type[BaseModel]` | `""` | Pydantic template class or dotted import path |
| `backend` | `Literal["llm","vlm"]` | `"llm"` | Extraction backend type |
| `inference` | `Literal["local","remote"]` | `"local"` | Inference location |
| `model_override` | `str \| None` | `None` | Override default model |
| `provider_override` | `str \| None` | `None` | Override default provider |
| `models` | `ModelsConfig` | `ModelsConfig()` | Model configurations for local/remote |
| `llm_client` | `LLMClientProtocol \| None` | `None` | Custom LLM client (bypasses provider/model) |
| `processing_mode` | `Literal["one-to-one","many-to-one"]` | `"many-to-one"` | Extraction strategy |
| `extraction_contract` | `Literal["auto","direct","dense"]` | `"auto"` | LLM extraction contract |
| `docling_config` | `Literal["ocr","vision"]` | `"ocr"` | Docling pipeline type |
| `llm_input_format` | `Literal["markdown","doclang","doclang-geo","auto"]` | `"auto"` | Document serialization format for LLM |
| `use_chunking` | `bool` | `True` | Enable structure-preserving chunking |
| `chunk_max_tokens` | `int \| None` | `None` (→512) | Max tokens per chunk |
| `debug` | `bool` | `False` | Save intermediate artifacts + trace data |
| `parallel_workers` | `int \| None` | `None` | Parallel extraction workers |
| `dense_skeleton_batch_tokens` | `int` | `2048` | Max tokens per skeleton batch (Phase 1) |
| `dense_fill_nodes_cap` | `int` | `5` | Max node instances per fill call (Phase 2) |
| `dense_fill_context` | `Literal["scoped","full"]` | `"scoped"` | Context per fill call: observed chunks or full doc |
| `dense_dedupe` | `Literal["off","standard","aggressive"]` | `"standard"` | Skeleton dedup intensity |
| `provenance` | `Literal["off","standard","detailed"]` | `"standard"` | Data grounding level |
| `gleaning_enabled` | `bool` | `True` | Extra "what did you miss?" extraction pass |
| `dump_to_disk` | `bool \| None` | `None` | Control file exports (auto: CLI=True, API=False) |
| `export_format` | `Literal["csv","cypher"]` | `"csv"` | Graph export format |
| `export_docling` | `bool` | `True` | Export Docling outputs |
| `export_docling_json` | `bool` | `True` | Export Docling JSON (lossless) |
| `export_markdown` | `bool` | `True` | Export full-document markdown |
| `export_doclang` | `bool` | `True` | Export DocLang `.dclg` |
| `export_per_page_markdown` | `bool` | `False` | Export per-page markdown |
| `reverse_edges` | `bool` | `False` | Create reverse (bidirectional) edges |
| `output_dir` | `str \| Path` | `"outputs"` | Output directory path |

#### Methods

| Method | Description |
|---|---|
| `run() -> None` | Execute the pipeline with this configuration |
| `to_dict() -> Dict[str, Any]` | Convert to dictionary |
| `generate_yaml_dict() -> Dict[str, Any]` | Generate YAML-compatible dict with defaults (classmethod) |

#### Validation Rules

- Pydantic validates all fields automatically.
- VLM backend does **not** support remote inference: `backend="vlm"` + `inference="remote"` raises `ValueError`.
- `dense` contract requires chunking-enabled `many-to-one` flow.

---

### 2.2 run_pipeline() — Entry Point Function

**Module:** `docling_graph.pipeline`

```python
def run_pipeline(config: Union[PipelineConfig, Dict[str, Any]]) -> PipelineContext
```

| Parameter | Type | Description |
|---|---|---|
| `config` | `PipelineConfig \| dict` | Pipeline configuration (dict is auto-validated into `PipelineConfig`) |

**Returns:** `PipelineContext` — shared context with results.

**Raises:**

| Exception | When |
|---|---|
| `PipelineError` | Pipeline execution fails |
| `ConfigurationError` | Configuration is invalid |
| `ExtractionError` | Document extraction fails |
| `GraphError` | Graph conversion fails |

**Pipeline stages (in order):**
1. **Template Loading** — load/validate Pydantic templates
2. **Extraction** — convert document with Docling, extract via backend
3. **Docling Export** (optional) — controlled by `export_*` flags
4. **Graph Conversion** — create NetworkX graph, stable node IDs, edges, provenance
5. **Export** — CSV, Cypher, JSON, HTML visualization
6. **Statistics & Report**

---

### 2.3 PipelineContext — Result Container

**Module:** `docling_graph.pipeline` (dataclass)

```python
@dataclass
class PipelineContext:
    config: PipelineConfig
    template: type[BaseModel] | None = None
    extractor: BaseExtractor | None = None
    extracted_models: list[BaseModel] | None = None
    docling_document: Any = None
    knowledge_graph: nx.DiGraph | None = None
    graph_metadata: GraphMetadata | None = None
    output_dir: Path | None = None
    node_registry: Any | None = None
    normalized_source: Union[str, Path, Any] | None = None
    input_metadata: Dict[str, Any] | None = None
    input_type: Any | None = None
    output_manager: OutputDirectoryManager | None = None
    trace_data: EventTrace | None = None  # populated when debug=True
```

**Accessing results:**

```python
context = run_pipeline(config)
graph = context.knowledge_graph          # nx.DiGraph
models = context.extracted_models         # list[BaseModel]
metadata = context.graph_metadata         # GraphMetadata
```

When `debug=True`, `trace_data` is an `EventTrace` with `summary` (including `runtime_seconds`) and ordered `steps` entries (`name`, `runtime_seconds`, `status`, `artifacts`). Saved as `debug/trace_data.json`.

---

### 2.4 Pydantic Templates — Schema Definition

Templates define both the **extraction schema** (what the LLM must produce) and the **graph structure** (how extracted models become nodes and edges).

#### Entity Declaration

```python
from pydantic import BaseModel, Field
from docling_graph.utils import edge

class Person(BaseModel):
    """Person entity with stable ID."""
    model_config = {
        'is_entity': True,
        'graph_id_fields': ['last_name', 'date_of_birth']
    }

    first_name: str = Field(description="Person's first name")
    last_name: str = Field(description="Person's last name")
    date_of_birth: str = Field(description="Date of birth (YYYY-MM-DD)")

class Organization(BaseModel):
    """Organization entity."""
    model_config = {'is_entity': True}

    name: str = Field(description="Organization name")
    employees: list[Person] = edge("EMPLOYS", description="List of employees")
```

#### `model_config` Options

| Key | Type | Description |
|---|---|---|
| `is_entity` | `bool` | If `True`, this model becomes a graph node (vs. a component embedded in a parent node) |
| `graph_id_fields` | `list[str]` | Fields used to compute deterministic node IDs via fingerprint hashing |

#### `edge()` Helper

```python
from docling_graph.utils import edge

field: list[TargetModel] = edge(
    label: str,           # Edge type label (e.g. "EMPLOYS", "CONTAINS")
    description: str = "", # Human-readable description
    # ... additional optional parameters
)
```

Edges are **optional by default**. Use `edge(label=...)` consistently for relationship-bearing fields.

#### Schema Design Rules

- **Identity fields:** required, scalar, short, copied verbatim from the document — never invented, list-valued, or enum-typed.
- **Non-identity fields:** optional or defaulted (graceful degradation for partial model output).
- Prefer 2–4 nesting levels; never nest the same rich entity model at several paths.
- Keep field descriptions to a locator + one normalization rule; never instruct computation.
- Use validators to normalize what models emit; never to reject whole payloads.

---

### 2.5 Backend Selection — LLM vs VLM

| Backend | Inference | Default Model | Default Provider | Best For |
|---|---|---|---|---|
| `llm` | `local` | `ibm-granite/granite-4.0-1b` | `vllm` | Text-heavy documents, remote API, cost control |
| `llm` | `remote` | `mistral-small-latest` | `mistral` | No GPU, quick setup, latest models |
| `vlm` | `local` | `numind/NuExtract-2.0-8B` | `docling` | Image-heavy documents, vision understanding, local GPU |

**Constraint:** VLM only supports **local** inference.

**Model/Provider Override:**
```python
config = {
    "backend": "llm",
    "inference": "remote",
    "provider_override": "mistral",
    "model_override": "mistral-medium-latest",
}
```

Provider/model identifiers use **LiteLLM routing** (supports OpenAI, Gemini, IBM watsonx, Mistral, vLLM, Ollama, and more).

---

### 2.6 Processing Modes

| Mode | Description | Use When |
|---|---|---|
| `"one-to-one"` | Per-page extraction; each page processed independently | Documents with distinct pages, page-level granularity needed |
| `"many-to-one"` | One consolidated model from entire document | Document is a single entity, document-level view needed |

Implemented by `OneToOneStrategy` and `ManyToOneStrategy`.

---

### 2.7 Extraction Contracts

| Contract | Description |
|---|---|
| `"direct"` | One-pass structured extraction from full content (single LLM call) |
| `"dense"` | Two-phase skeleton-then-flesh: skeleton discovery + fill passes + merge assembly. Requires `use_chunking=True` + `many-to-one` |
| `"auto"` | Resolves per document: `direct` if single call fits context window + output budget; `dense` otherwise. Decision logged as `[AutoContract]` |

#### Dense Extraction Tuning Fields

| Field | Default | Description |
|---|---|---|
| `dense_skeleton_batch_tokens` | `2048` | Max tokens per skeleton batch (Phase 1) |
| `dense_fill_nodes_cap` | `5` | Max node instances per fill call (Phase 2) |
| `dense_fill_context` | `"scoped"` | `"scoped"` = context from observed chunks; `"full"` = entire document |
| `dense_dedupe` | `"standard"` | `"off"` = exact ID dedup only; `"standard"` = LLM reconciliation of same-entity aliases; `"aggressive"` = also merges near-identical identifier strings (OCR noise) |

Mandatory cleanup (root singleton collapse, barren-branch pruning, root-required quality gate) are pipeline invariants — not configurable.

---

### 2.8 Chunking — DocumentChunker

**Module:** `docling_graph.core.extractors.document_chunker`

Structure-preserving document chunker built on Docling's `HybridChunker`.

```python
class DocumentChunker:
    def __init__(self,
        tokenizer_name: str | None = None,   # default: sentence-transformers/all-MiniLM-L6-v2
        chunk_max_tokens: int = 512,          # hard cap
        merge_peers: bool = True,              # merge peer sections
    ) -> None

    def chunk_document(self, document: DoclingDocument) -> List[str]
    def chunk_document_with_stats(self, document: DoclingDocument) -> tuple[List[str], dict]
    def chunk_text_fallback(self, text: str) -> List[str]
    def get_config_summary(self) -> dict
```

**Features:**
- Tables, lists, and section hierarchy kept intact
- Hard cap: chunks never exceed `chunk_max_tokens` (re-split via sentence → word → character fallback)
- Single sizing knob: `chunk_max_tokens` (no coupling to model context limits)

---

### 2.9 Graph Conversion — GraphConverter

**Module:** `docling_graph.core.converters`

```python
class GraphConverter:
    def __init__(self,
        config: GraphConfig | None = None,
        add_reverse_edges: bool = False,
        validate_graph: bool = True,
        registry: NodeIDRegistry | None = None,
        auto_cleanup: bool = True,
    ) -> None

    def pydantic_list_to_graph(self,
        model_instances: List[BaseModel],
        provenance_binder: Callable[[nx.DiGraph, List[BaseModel]], None] | None = None,
    ) -> tuple[nx.DiGraph, GraphMetadata]
```

| Parameter | Type | Default | Description |
|---|---|---|---|
| `config` | `GraphConfig \| None` | `None` | Graph conversion constants |
| `add_reverse_edges` | `bool` | `False` | Create bidirectional edges |
| `validate_graph` | `bool` | `True` | Validate structure after conversion |
| `registry` | `NodeIDRegistry \| None` | `None` | Shared registry for cross-batch ID consistency |
| `auto_cleanup` | `bool` | `True` | Remove phantom nodes, duplicates, orphaned edges |

**Returns:** `(graph: nx.DiGraph, metadata: GraphMetadata)` where `GraphMetadata` has `node_count`, `edge_count`, `node_types`, `edge_types`, `avg_degree`, `density`.

#### GraphConfig (frozen dataclass)

```python
@dataclass(frozen=True)
class GraphConfig:
    NODE_ID_HASH_LENGTH: Final[int] = 12
    MAX_STRING_LENGTH: Final[int] = 1000
    TRUNCATE_SUFFIX: Final[str] = "..."
    add_reverse_edges: bool = False
    validate_graph: bool = True
```

#### NodeIDRegistry

```python
class NodeIDRegistry:
    def get_node_id(self, model_instance: BaseModel, auto_register: bool = True) -> str
    def register_batch(self, models: list[BaseModel]) -> None
    def get_stats(self) -> dict  # total_entities, per-class counts
```

Node ID format: `"{ClassName}_{fingerprint}"` — fingerprint derived from `graph_id_fields` (entities) or all non-empty fields (components).

---

### 2.10 Provenance — Deterministic Data Grounding

**Module:** `docling_graph.core.provenance`

Deterministic node-to-source grounding. **No LLM involvement** — the ledger is built from pipeline bookkeeping.

#### ProvenanceLedger

```python
class ProvenanceLedger(BaseModel):
    version: int = 1
    document: DocumentOrigin | None = None
    resolution: Literal["document", "page", "batch", "chunk", "span"] = "chunk"
    node_level: bool = False
    chunks: dict[int, ChunkRecord] = {}
    nodes: dict[str, NodeProvenance] = {}
    bind_stats: dict[str, int] = {}
```

| Field | Description |
|---|---|
| `document` | Source identity (`DocumentOrigin`: `document_id`, `source`, `input_type`, `converted_at`, `page_count`, `template_name`, `template_schema_hash`) |
| `resolution` | Best precision achieved: `"document"` (whole-doc fallback) → `"chunk"` (batch-level) → `"span"` (verbatim char-offset anchor) |
| `node_level` | `True` for dense contract (per-node entries); `False` for direct (chunk index only) |
| `chunks` | `chunk_id → ChunkRecord` (includes enriched chunk **text** — self-contained) |
| `nodes` | `identity_key → NodeProvenance` |
| `bind_stats` | After binding: `nodes_seen`, `bound_verbatim`, `bound_observed`, `bound_document`, `unresolved` |

#### Supporting Models

```python
class ChunkRecord(BaseModel):
    chunk_id: int
    batch_index: int
    page_numbers: tuple[int, ...] = ()
    doc_item_refs: tuple[str, ...] = ()  # docling self_refs, e.g. "#/texts/42"
    headings: tuple[str, ...] = ()
    token_count: int = 0
    text_hash: str = ""
    char_length: int = 0
    text: str = ""
    resplit_of: int | None = None

class SourceAnchor(BaseModel):
    document_id: str = ""
    chunk_id: int
    kind: Literal["observed", "verbatim", "derived", "reconciled"] = "observed"
    span: tuple[int, int] | None = None  # char offsets into chunk text

class NodeProvenance(BaseModel):
    identity_key: str
    catalog_path: str = ""
    node_type: str = ""
    ids: dict[str, str] = {}
    anchors: list[SourceAnchor] = []
    merged_from: list[str] = []
    synthetic: bool = False
    dropped: bool = False
    fill_batches: list[int] = []
    notes: list[str] = []
```

#### Provenance Level Options (PipelineConfig)

| Value | Behavior |
|---|---|
| `"off"` | Disabled — no `__provenance__` attribute, no ledger export |
| `"standard"` (default) | `__provenance__` node attribute + `provenance.json`; verbatim locations where found, approximate/document-scope fallbacks otherwise |
| `"detailed"` | Adds character-offset spans to the node attribute |

#### Key Functions

```python
def identity_key(path: str, ids: Mapping[str, Any], id_fields: Sequence[str]) -> str | None
def compact_view(entry: NodeProvenance, ledger: ProvenanceLedger,
    max_anchors: int = 8, include_spans: bool = False) -> dict[str, Any]
def bind_provenance(*, graph, models, ledger, registry, template, include_spans=False) -> dict[str, int]
def locate_identifier(value: str, chunk_texts: Mapping[int, str]) -> list[tuple[int, tuple[int, int]]]
def refine_ledger_spans(ledger: ProvenanceLedger, chunk_texts: Mapping[int, str]) -> int

PROVENANCE_NODE_ATTR = "__provenance__"  # graph node attribute key
```

---

### 2.11 Export Formats

**Module:** `docling_graph.core.exporters`

#### ExportConfig (frozen dataclass)

```python
@dataclass(frozen=True)
class ExportConfig:
    CSV_NODE_FILENAME: str = "nodes.csv"
    CSV_EDGE_FILENAME: str = "edges.csv"
    CYPHER_FILENAME: str = "graph.cypher"
    JSON_FILENAME: str = "graph.json"
```

#### BaseExporter

```python
class BaseExporter:
    def __init__(self, graph: nx.MultiDiGraph, output_dir: Path): ...
    def export(self) -> None: ...  # override in subclass
```

#### Exporter Classes

| Exporter | Output File(s) | Description |
|---|---|---|
| `CSVExporter` | `nodes.csv`, `edges.csv` | Neo4j-compatible CSV for bulk import |
| `CypherExporter` | `graph.cypher` | Cypher `CREATE` statements for Neo4j |
| `JSONExporter` | `graph.json` | NetworkX node-link format JSON |
| `DoclingExporter` | `docling_document.json`, `markdown/full_document.md`, `markdown/pages/page_N.md` | Docling document outputs |

#### Cypher Export Example

```
CREATE (n1:Person {name: "John Doe", age: 30})
CREATE (n2:Organization {name: "ACME Corp"})
CREATE (n1)-[:WORKS_AT]->(n2)
```

#### Custom Exporters

```python
from docling_graph.core.exporters import BaseExporter

class MyExporter(BaseExporter):
    def export(self) -> None:
        output_file = self.output_dir / "custom.txt"
        with open(output_file, 'w') as f:
            f.write(f"Nodes: {self.graph.number_of_nodes()}\n")
            f.write(f"Edges: {self.graph.number_of_edges()}\n")
```

---

### 2.12 Visualization

**Module:** `docling_graph.core.visualizers`

```python
from docling_graph.core.visualizers import InteractiveVisualizer

visualizer = InteractiveVisualizer()
visualizer.save_cytoscape_graph(graph, "graph.html")
```

**Output:** Self-contained `graph_visualization.html` with zoom/pan, node inspection, search, image export. Also generates `extraction_report.md` and graph statistics JSON.

#### Graph Statistics

```json
{
  "node_count": 15,
  "edge_count": 18,
  "node_types": {"BillingDocument": 1, "Organization": 2, "Address": 3, "LineItem": 9},
  "edge_types": {"ISSUED_BY": 1, "SENT_TO": 1, "LOCATED_AT": 5, "CONTAINS_LINE": 9},
  "avg_degree": 2.4,
  "density": 0.17
}
```

---

### 2.13 LLM Clients

**Module:** `docling_graph.llm_clients`

All LLM calls go through `LiteLLMClient.get_json_response()` when using the default provider/model path. LiteLLM standardizes provider differences (`drop_params=True`).

#### LLMClientProtocol

Custom clients must implement:
```python
class LLMClientProtocol(Protocol):
    def get_json_response(self, prompt: str | Mapping[str, str],
        schema_json: str) -> dict | list: ...
```

Pass to pipeline via `PipelineConfig(llm_client=your_client)`.

#### LiteLLMEndpointClient — Custom Endpoint

```python
class LiteLLMEndpointClient:
    def __init__(self,
        model: str,              # e.g. "openai/your-model" or "hosted_vllm/..."
        base_url: str,           # inference endpoint URL
        *,
        api_key: str | None = None,
        headers: dict[str, str] | None = None,
        timeout_s: int = 120,
        max_tokens: int = 4096,
        temperature: float = 0.1,
    ): ...
```

| Field | Type | Default | Description |
|---|---|---|---|
| `model` | `str` | (required) | Model string (LiteLLM routing) |
| `base_url` | `str` | (required) | Endpoint URL (trailing `/` stripped) |
| `api_key` | `str \| None` | `None` | Optional API key |
| `headers` | `dict \| None` | `None` | Optional headers |
| `timeout_s` | `int` | `120` | Request timeout (seconds) |
| `max_tokens` | `int` | `4096` | Max generation tokens |
| `temperature` | `float` | `0.1` | Sampling temperature |

#### JSON/Structured Output

```python
def get_json_response(self, prompt: str | Mapping[str, str],
    schema_json: str) -> dict | list
    # Uses response_format with JSON schema by default
    # ResponseHandler provides fallback if output is not strictly JSON

def get_json_response_stream(self, prompt, schema_json,
    structured_output: bool = True,
    response_top_level: Literal["object", "array"] = "object",
    response_schema_name: str = "extraction_result",
) -> Iterator[Dict[str, Any] | list[Any]]
```

---

### 2.14 Extractors — Strategy & Factory

**Module:** `docling_graph.core.extractors`

#### ExtractorFactory

```python
from docling_graph.core import ExtractorFactory

extractor = ExtractorFactory.create_extractor(
    processing_mode: "one-to-one" | "many-to-one",
    backend_name: "llm" | "vlm",
    extraction_contract: "direct" | "dense" = "direct",
    dense_config: dict | None = None,
    model_name: str | None = None,      # required for VLM
    llm_client: LLMClientProtocol | None = None,  # required for LLM
    docling_config: str = "ocr",
)
```

#### Extraction Strategies

| Strategy | Description |
|---|---|
| `OneToOneStrategy` | One model per page/item; returns `(List[BaseModel], DoclingDocument)` |
| `ManyToOneStrategy` | One consolidated model from entire document; zero data loss (returns all partial models if consolidation fails) |

#### Backends

| Backend | Methods |
|---|---|
| `LlmBackend` | `extract_from_markdown()`, `extract_from_chunk_batches()` (dense), `generate()` (gleaning), `cleanup()` |
| `VlmBackend` | `extract_from_document()`, `cleanup()`, `cleanup_all_gpus()` (local only) |

#### DocumentProcessor

```python
class DocumentProcessor:
    def convert_to_docling_doc(self, source: str) -> Any
    def extract_full_markdown(self, document: Any) -> str
    def extract_page_markdowns(self, document: Any) -> List[str]
```

---

### 2.15 ModelsConfig

```python
class ModelsConfig(BaseModel):
    llm: LLMConfig = Field(default_factory=LLMConfig)
    vlm: VLMConfig = Field(default_factory=VLMConfig)

class LLMConfig(BaseModel):
    local: ModelConfig   # default: model="ibm-granite/granite-4.0-1b", provider="vllm"
    remote: ModelConfig  # default: model="mistral-small-latest", provider="mistral"

class VLMConfig(BaseModel):
    local: ModelConfig   # default: model="numind/NuExtract-2.0-8B", provider="docling"

class ModelConfig(BaseModel):
    model: str    # model name/path
    provider: str # provider name
```

---

### 2.16 CLI

**Commands:** `init`, `convert`, `inspect`

#### `init` — Create configuration

```bash
docling-graph init
```

Interactive configuration builder. Creates `config.yaml` with processing mode, extraction contract, backend, inference, provider/model, export settings, output directory.

#### `convert` — Convert documents

```bash
docling-graph convert SOURCE --template TEMPLATE [OPTIONS]

# Examples:
docling-graph convert "https://arxiv.org/pdf/2207.02720" \
    --template "docs.examples.templates.rheology_research.ScholarlyRheologyPaper" \
    --processing-mode "many-to-one" \
    --extraction-contract "dense" \
    --debug

docling-graph convert document.pdf --template "templates.BillingDocument" \
    --backend llm --inference remote \
    --provider mistral --model mistral-large-latest

docling-graph convert document.pdf --template "templates.BillingDocument" \
    --output-dir "outputs/invoice_001"
```

| Flag | Description |
|---|---|
| `--template` / `-t` | Pydantic template class (dotted import path) |
| `--backend` | `llm` or `vlm` |
| `--inference` | `local` or `remote` |
| `--provider` | LiteLLM provider name |
| `--model` | Model name |
| `--processing-mode` | `one-to-one` or `many-to-one` |
| `--extraction-contract` | `auto`, `direct`, or `dense` |
| `--output-dir` | Output directory |
| `--debug` | Save intermediate artifacts |
| `--verbose` / `-v` | Detailed logging |

#### `inspect` — Visualize graphs

```bash
docling-graph inspect outputs
```

Interactive HTML visualization; CSV and JSON import; node/edge exploration.

#### Configuration Priority (highest → lowest)

1. Command-line arguments
2. `config.yaml` (created by `init`)
3. Built-in defaults (`PipelineConfig`)

#### CLI Output Structure

```
outputs/
├── metadata.json          # Pipeline metadata
├── docling/               # Docling conversion output
│   ├── document.json
│   └── document.md
└── docling_graph/         # Graph outputs
    ├── graph.json         # Complete graph (node-link format)
    ├── nodes.csv          # Node data (Neo4j-compatible)
    ├── edges.csv          # Edge data
    ├── graph.cypher       # Cypher script (if export_format=cypher)
    ├── graph.html         # Interactive visualization
    └── report.md          # Summary report
```

#### Environment Variables

```bash
# Remote API keys
export MISTRAL_API_KEY="your-key"
export OPENAI_API_KEY="your-key"
export GEMINI_API_KEY="your-key"
export WATSONX_API_KEY="your-key"

# Local providers
export VLLM_BASE_URL="http://localhost:8000/v1"
export OLLAMA_BASE_URL="http://localhost:11434"
```

---

### 2.17 Exceptions

**Module:** `docling_graph.exceptions`

| Exception | When Raised |
|---|---|
| `GraphError` | Base exception for all graph operations |
| `ConfigurationError` | Configuration is invalid |
| `ExtractionError` | Document extraction fails |
| `PipelineError` | Pipeline execution fails |
| `ClientError` | LLM client error |

```python
from docling_graph import run_pipeline
from docling_graph.exceptions import GraphError

try:
    run_pipeline({"source": "document.pdf", "template": "templates.MyTemplate",
                  "export_format": "cypher"})
except GraphError as e:
    print(f"Graph error: {e}")
```

---

## 3. AI-Knowledge-Graph — LLM Triple Extraction & Visualization

### Main Concepts

AI-Knowledge-Graph is a lightweight, open-source tool that takes unstructured text and uses an LLM to extract knowledge as **Subject-Predicate-Object (SPO) triplets**, then visualizes the relationships as an interactive HTML knowledge graph. It is schema-free (no Pydantic templates) and relies entirely on LLM prompts for extraction quality.

Core abstractions:

- **SPO Triple** — `{"subject": str, "predicate": str, "object": str, "chunk": int}` — the atomic unit of extracted knowledge.
- **Text Chunker** — Splits documents into word-count-based chunks with overlap for LLM processing.
- **Entity Standardization** — Optional LLM-based clustering of entity name variants ("AI" / "A.I." / "artificial intelligence") into canonical nodes.
- **Relationship Inference** — Optional rule-based (transitivity, lexical similarity) and LLM-assisted inference of hidden connections between disconnected subgraphs.
- **PyVis Visualizer** — Generates interactive HTML with color-coded communities (Louvain), node sizing by centrality, and dashed edges for inferred relationships.

### End-to-End Flow

```
input text file (.txt)
   │
   ▼
 Text Chunking (word-count based, with overlap)
   │
   ▼
 Phase 1: LLM-Powered Triple Extraction (per chunk)
   │  Prompt: "Extract all S-P-O relationships as JSON array"
   │  Predicates ≤ 3 words, lowercase, no pronouns
   ▼
 Phase 2: Entity Standardization (optional)
   │  Basic normalization (lowercase, trim)
   │  LLM clustering of entity name variants
   ▼
 Phase 3: Relationship Inference (optional)
   │  Rule-based: transitive relationships, lexical similarity
   │  LLM-assisted: bridge disconnected subgraphs
   │  Inferred edges marked with inferred=true
   ▼
 Interactive HTML Visualization (PyVis / Vis.js)
   ├── Color-coded communities (Louvain)
   ├── Node sizing by centrality
   ├── Solid edges = text-derived, dashed = inferred
   └── Raw JSON graph data (.json)
```

---

### 3.1 Configuration — config.toml

```toml
[llm]
model = "gemma3"
api_key = "sk-1234"
base_url = "http://localhost:11434/v1/chat/completions"
max_tokens = 8192
temperature = 0.2

[chunking]
chunk_size = 200   # words per chunk
overlap = 20       # words of overlap between chunks

[standardization]
enabled = true               # enable entity standardization
use_llm_for_entities = true  # use LLM for entity resolution

[inference]
enabled = true                # enable relationship inference
use_llm_for_inference = true  # use LLM for inference
apply_transitive = true       # apply transitive inference rules

[visualization]
edge_smooth = false  # smooth edge lines
```

#### Configuration Sections

| Section | Field | Type | Default | Description |
|---|---|---|---|---|
| `[llm]` | `model` | `str` | — | Model name (OpenAI-compatible) |
| `[llm]` | `api_key` | `str` | — | API key |
| `[llm]` | `base_url` | `str` | — | OpenAI-compatible endpoint URL |
| `[llm]` | `max_tokens` | `int` | `8192` | Max generation tokens |
| `[llm]` | `temperature` | `float` | `0.2` | Sampling temperature |
| `[chunking]` | `chunk_size` | `int` | `200` | Words per chunk |
| `[chunking]` | `overlap` | `int` | `20` | Overlap words between chunks |
| `[standardization]` | `enabled` | `bool` | `true` | Enable entity standardization |
| `[standardization]` | `use_llm_for_entities` | `bool` | `true` | Use LLM for entity resolution |
| `[inference]` | `enabled` | `bool` | `true` | Enable relationship inference |
| `[inference]` | `use_llm_for_inference` | `bool` | `true` | Use LLM for inference |
| `[inference]` | `apply_transitive` | `bool` | `true` | Apply transitive rules |
| `[visualization]` | `edge_smooth` | `bool` | `false` | Smooth edge rendering |

---

### 3.2 CLI — generate-graph.py

```bash
python generate-graph.py --input mydocument.txt --output mydocument.html
```

#### Command-Line Options

| Flag | Description |
|---|---|
| `--input FILE` | Path to input text file (required unless `--test`) |
| `--output FILE` | Output HTML file path |
| `--config FILE` | Path to configuration file (default: `config.toml`) |
| `--debug` | Enable debug output (raw LLM responses, extracted JSON) |
| `--no-standardize` | Disable entity standardization |
| `--no-inference` | Disable relationship inference |
| `--test` | Generate a test visualization with sample data |

---

### 3.3 LLM Prompts — Extraction Pipeline

The tool uses four LLM prompts (configurable in `src/knowledge_graph/prompts.py`):

#### 1. Extraction System Prompt

```
You are an advanced AI system specialized in knowledge extraction and
knowledge graph generation. Your expertise includes identifying consistent
entity references and meaningful relationships in text.
CRITICAL INSTRUCTION: All relationships (predicates) MUST be no more than
3 words maximum. Ideally 1-2 words. This is a hard limit.
```

#### 2. Extraction User Prompt (abbreviated)

```
Your task: Read the text below and identify all Subject-Predicate-Object
relationships in each sentence. Then produce a single JSON array of objects.

Rules:
- Entity Consistency: Use consistent names throughout
- Atomic Terms: distinct key terms, avoid merging ideas
- Unified References: Replace pronouns with referenced entities
- Pairwise Relationships: one triple per meaningful pair
- Predicates MUST be 1-3 words maximum
- Standardize terminology (canonical forms)
- All S-P-O text lower-case

Output: JSON array with "subject", "predicate", "object" keys only.
```

#### 3 & 4. Standardization and Inference Prompts

Additional prompts guide the LLM for entity clustering and relationship inference. All prompts are editable in the source file.

#### Triple Output Format

```json
[
  {"subject": "eli whitney", "predicate": "invented", "object": "cotton gin", "chunk": 1},
  {"subject": "industrial revolution", "predicate": "reshapes", "object": "economic systems", "chunk": 1}
]
```

#### Inferred Relationship Format

```json
[
  {"subject": "electrification", "predicate": "enables", "object": "Manufacturing Automation", "inferred": true},
  {"subject": "tim berners-lee", "predicate": "expanded via internet", "object": "information sharing", "inferred": true}
]
```

---

### 3.4 Processing Pipeline Phases

#### Phase 1: Initial Triple Extraction

- Text split into chunks (default: 500 words, 50 overlap)
- Each chunk sent to LLM with extraction prompt
- LLM returns JSON array of SPO triples with chunk index
- All triples combined into raw knowledge graph

#### Phase 2: Entity Standardization (optional, `--no-standardize` to disable)

- Basic normalization: lowercasing, trimming whitespace
- LLM-based clustering: groups entity variants into canonical nodes
- Self-referencing triples removed (e.g., `A → related to → A`)
- Example: 106 entities → 101 standard forms, 73 → 65 triples

#### Phase 3: Relationship Inference (optional, `--no-inference` to disable)

- **Rule-based:** Transitive relationships (A→B, B→C ⟹ A→C), lexical similarity ("related to" edges for similar names)
- **LLM-assisted:** Proposes links between disconnected subgraphs/communities
- Inferred edges marked with `inferred: true` (rendered as dashed lines)
- Example: 65 triples → 116 triples (51 inferred), 18 communities → 12

#### Visualization

- PyVis (Python interface to Vis.js) generates self-contained HTML
- Louvain community detection for color-coded clusters
- Node sizing by degree/centrality
- Interactive controls: pan, zoom, drag, toggle physics, light/dark mode, filter
- Saves raw JSON graph data alongside HTML

---

### 3.5 Project Structure

```
ai-knowledge-graph/
├── config.toml                    # Configuration file
├── generate-graph.py              # Script entry point
├── pyproject.toml                 # Project metadata
├── requirements.txt               # Dependencies (pip)
├── uv.lock                        # Dependencies (uv)
└── src/
    └── knowledge_graph/
        ├── __init__.py
        ├── config.py              # Configuration loading & validation
        ├── entity_standardization.py  # Entity standardization algorithms
        ├── llm.py                 # LLM interaction & response processing
        ├── main.py                 # Main program flow & orchestration
        ├── prompts.py             # Centralized LLM prompts
        ├── text_utils.py          # Text processing & chunking utilities
        ├── visualization.py       # Knowledge graph visualization generator
        └── templates/
            └── graph_template.html # Base template for interactive graph
```

---

### 3.6 Requirements & Installation

**Requirements:**
- Python 3.12+
- Access to an OpenAI-compatible API endpoint (Ollama, LiteLLM, LM Studio, OpenAI, etc.)
- Git

**Installation:**
```bash
git clone https://github.com/robert-mcdermott/ai-knowledge-graph.git
cd ai-knowledge-graph
uv sync          # or: pip install -r requirements.txt
```

---

### 3.7 Example Run Output

```
PHASE 1: INITIAL TRIPLE EXTRACTION
Processing text in 3 chunks (size: 500 words, overlap: 50 words)
Extracted a total of 73 triples from all chunks

PHASE 2: ENTITY STANDARDIZATION
Starting with 73 triples and 106 unique entities
Applied LLM-based entity standardization for 15 entity groups
Standardized 106 entities into 101 standard forms
After standardization: 65 triples and 72 unique entities

PHASE 3: RELATIONSHIP INFERENCE
Identified 18 disconnected communities in the graph
Inferred 27 new relationships between communities
Inferred 8 relationships based on lexical similarity
Added 51 inferred relationships
Final knowledge graph: 116 triples

Knowledge Graph Statistics:
Nodes: 72
Edges: 116 (55 inferred)
Communities: 12
```

---

## 4. Neo4j — Graph Database for AI Systems

### Main Concepts

Neo4j is a native graph database that stores data as nodes, relationships, and properties (the **property graph model**). For AI systems, Neo4j provides **GraphRAG** — pairing knowledge graphs with vector search to ground LLMs with structured context for accurate, explainable retrieval. The platform exposes the **Cypher** query language, the **Bolt** protocol for client-driver communication, and Python/TypeScript drivers.

Core abstractions:

- **Node** — An entity with zero or more labels (e.g., `:Person`) and properties (key-value pairs).
- **Relationship** — A typed, directed edge between two nodes (e.g., `[:KNOWS]`) with optional properties.
- **Property** — A key-value pair on a node or relationship (typed: string, int, float, boolean, list).
- **Label** — A node type tag (a node can have multiple labels).
- **Cypher** — Declarative graph query language (SQL for graphs): `CREATE`, `MATCH`, `MERGE`, `SET`, `DELETE`, `DETACH DELETE`, `RETURN`, `WHERE`, `UNWIND`.
- **Driver** — Client object holding connection details for a Neo4j database.
- **GraphRAG** — Combines knowledge graph retrieval with RAG to provide LLMs with structured, connected context for multi-step reasoning.
- **MCP (Model Context Protocol)** — Exposes Neo4j graph retrieval as a tool for AI agents.

### Capabilities for AI Systems

| Capability | Description |
|---|---|
| **Agentic GraphRAG** | Pair RAG with knowledge graph; AI agents traverse interconnected data for richer contextual insights |
| **Structured + Unstructured Data** | Flexible schema lets you add datasets; feed LLMs new context as users interact |
| **Context Management for MCP** | Model data with knowledge graph keeping relationships intact; improve AI reasoning and task execution |
| **Index-Free Adjacency** | Boost query performance for agent memory & tool calling; 1,000x faster than traditional databases |
| **Real-Time Data Updates** | Enrich and evolve agent knowledge as requirements and data change |
| **Long-Term Agent Memory** | Optimize memory management that grows intelligently across sessions |

---

### 4.1 Python Driver — Connection & Querying

**Module:** `neo4j` (pip install neo4j)

#### Connecting

```python
from neo4j import GraphDatabase

URI = "neo4j://localhost"  # or "neo4j+s://xxx.databases.neo4j.io" (Aura)
AUTH = ("<username>", "<password>")

with GraphDatabase.driver(URI, auth=AUTH) as driver:
    driver.verify_connectivity()
```

| Parameter | Type | Description |
|---|---|---|
| `URI` | `str` | Connection URI (`neo4j://` for direct, `neo4j+s://` for TLS/Aura) |
| `auth` | `tuple[str, str]` | `(username, password)` tuple |
| `database_` | `str` | Target database name (pass per-query) |

#### `Driver.execute_query()` — Primary Query API

```python
records, summary, keys = driver.execute_query(
    """Cypher query""",
    param1=value1,           # query parameters (keyword args)
    database_="<database>",   # target database
    routing_="r",             # read routing (cluster)
    auth_=("<user>", "<pass>"),  # run as different user
    impersonated_user_="<user>",  # impersonate without password
)
```

**Returns:** `(records: list[Record], summary: ResultSummary, keys: list[str])`

| Parameter | Type | Description |
|---|---|---|
| `query` | `str` | Cypher query string (use `$param` placeholders) |
| `**kwargs` | — | Query parameters as keyword arguments |
| `parameters_` | `dict` | Alternative: grouped parameters dict |
| `database_` | `str` | Target database (always specify explicitly) |
| `routing_` | `str` | `"r"` for read (route to any cluster node); default routes to leader |
| `auth_` | `tuple` | Execute as different user (with password) |
| `impersonated_user_` | `str` | Impersonate user (no password needed, requires privileges) |
| `result_transformer_` | callable | Transform result (e.g., to pandas DataFrame or graph) |

**Parameter rules:**
- Never hardcode/concatenate parameters into queries (security + performance).
- Keyword parameters must not end with single underscore (reserved for config).

---

### 4.2 Cypher Operations

#### Create (Write)

```python
summary = driver.execute_query("""
    CREATE (a:Person {name: $name})
    CREATE (b:Person {name: $friendName})
    CREATE (a)-[:KNOWS]->(b)
    """,
    name="Alice", friendName="David",
    database_="<database>",
).summary
```

#### Match (Read)

```python
records, summary, keys = driver.execute_query("""
    MATCH (p:Person)-[:KNOWS]->(:Person)
    RETURN p.name AS name
    """,
    database_="<database>",
)
for record in records:
    print(record.data())  # dict representation
```

#### Update (Set)

```python
driver.execute_query("""
    MATCH (p:Person {name: $name})
    SET p.age = $age
    """, name="Alice", age=42,
    database_="<database>",
)
```

#### Create Relationship Between Existing Nodes

```python
driver.execute_query("""
    MATCH (alice:Person {name: $name})
    MATCH (bob:Person {name: $friend})
    CREATE (alice)-[:KNOWS]->(bob)
    """, name="Alice", friend="Bob",
    database_="<database>",
)
```

#### Delete

```python
# DETACH DELETE removes node AND all its relationships
driver.execute_query("""
    MATCH (p:Person {name: $name})
    DETACH DELETE p
    """, name="Alice",
    database_="<database>",
)
```

#### Merge (Upsert)

```python
driver.execute_query(
    "MERGE (:Person {name: $name})",
    name="Alice", age=42,
    database_="<database>",
)
```

#### Unwind (Batch)

```python
driver.execute_query("""
    MATCH (p:Person {name: $person.name})
    UNWIND $person.friends AS friend_name
    MATCH (friend:Person {name: friend_name})
    MERGE (p)-[:KNOWS]->(friend)
    """, person=person,
    database_="<database>",
)
```

---

### 4.3 Result Handling

| Object | Description |
|---|---|
| `Record` | One row of results; `record.data()` returns dict; field access via `record["key"]` or `record.key` |
| `ResultSummary` | Execution metadata: `summary.query`, `summary.counters` (nodes_created, relationships_created, properties_set, etc.), `summary.result_available_after` (ms) |
| `keys` | List of column names returned |

#### Result Transformers

```python
# Transform to pandas DataFrame
from neo4j import Driver
driver.execute_query("MATCH (p:Person) RETURN p",
    result_transformer_=Driver.RESULT_AS_DF,
    database_="<database>")

# Transform to graph
driver.execute_query("MATCH (p:Person)-[:KNOWS]->(f:Person) RETURN p, f",
    result_transformer_=Driver.RESULT_AS_GRAPH,
    database_="<database>")
```

---

### 4.4 Error Handling

```python
from neo4j.exceptions import Neo4jError

try:
    driver.execute_query('MATCH (p:Person) RETURN', database_='<database>')
except Neo4jError as e:
    print('Error code:', e.code)        # e.g. "Neo.ClientError.Statement.SyntaxError"
    print('Message:', e.message)
    print('GQL status:', e.gql_status)   # e.g. "42001"
    print('Retryable:', e.is_retryable())

    if e.find_by_gql_status('42001'):
        # syntax error
        pass
    elif e.find_by_gql_status('42NFF'):
        # forbidden (no CREATE permission)
        pass
```

**Error hierarchy:** All server exceptions subclass `Neo4jError`. GQL status codes provide granular classification. Transient errors are auto-retried by `execute_query()` up to the configured max retry time.

---

### 4.5 Session & Transaction Management

```python
from neo4j import GraphDatabase

driver = GraphDatabase.driver(URI, auth=AUTH)
session = driver.session(database="<database-name>")

# Usage...
session.close()
driver.close()
```

For explicit transactions (control retry behavior):

```python
# execute_read / execute_write run transaction functions with auto-retry
def get_people(tx):
    result = tx.run("MATCH (p:Person) RETURN p.name AS name")
    return [record["name"] for record in result]

with driver.session(database="<database>") as session:
    people = session.execute_read(get_people)
```

---

### 4.6 GraphRAG — Knowledge Graphs for AI

Neo4j's **GraphRAG** combines knowledge graphs with vector search to ground LLMs:

| Pattern | Description |
|---|---|
| **Vector + Graph** | Vector search finds semantically similar content; graph traversal expands to connected entities for complete context |
| **Agentic Retrieval** | AI agents traverse the knowledge graph to gather multi-hop context for complex questions |
| **Entity-centric RAG** | Build a graph of entities and relationships from unstructured data; retrieve connected context per query |
| **Ontology-driven RAG** | Use domain ontologies to guide knowledge graph construction and retrieval |

#### GraphRAG Python Package

Neo4j provides a Python package for building knowledge graphs from unstructured data and implementing GraphRAG workflows:

- Turn unstructured data into knowledge graphs
- Implement advanced retrievers (vector + graph traversal)
- Create GraphRAG workflows with LangChain integration

#### LangChain Integration

```python
# Neo4j as a vector store + graph in LangChain
from langchain_community.graphs import Neo4jGraph
from langchain_community.vectorstores import Neo4jVector
```

| Integration | Purpose |
|---|---|
| `Neo4jGraph` | Graph store for LangChain (Cypher queries) |
| `Neo4jVector` | Vector store with hybrid (vector + graph) retrieval |
| LangChain retrievers | Graph-augmented retrieval for RAG |

#### MCP Server Integration

Neo4j graph retrieval can be exposed as an **MCP (Model Context Protocol) server**, making knowledge graph queries available as a tool for AI agents:

- Combine vector search with Cypher query language
- Add MCP as another tool for agents
- Evaluate graph retrieval quality in agentic systems

---

### 4.7 Deployment Options

| Option | Description |
|---|---|
| **Neo4j Aura** | Fully managed cloud service (free + paid plans) |
| **Neo4j Desktop** | Local development environment |
| **Docker** | Self-hosted container deployment |
| **Kubernetes** | Enterprise cluster deployment |
| **Cloud marketplaces** | AWS, Azure, Google Cloud native integrations |

#### Cloud Partner Integrations

| Partner | Integration |
|---|---|
| AWS | Native marketplace deployment |
| Microsoft Azure | Native integration with Vertex AI |
| Google Cloud | Vertex AI integration for GraphRAG |
| Snowflake | Data sharing integration |
| Databricks | ML pipeline integration |

---

### 4.8 Key Concepts (Glossary)

| Term | Description |
|---|---|
| **Cypher** | Neo4j's declarative graph query language (SQL for graphs) |
| **Bolt** | Wire protocol for driver-DB communication (port 7687) |
| **ACID** | Neo4j is ACID-compliant (Atomicity, Consistency, Isolation, Durability) |
| **Causal consistency** | Read/write queries seen by every cluster member in the same order |
| **Bookmark** | Token representing DB state; ensures queries execute after a given state |
| **Transaction function** | Callback executed by `execute_read`/`execute_write` with auto-retry |
| **Backpressure** | Flow control ensuring client isn't overwhelmed by data |
| **APOC** | Awesome Procedures On Cypher — library of advanced functions |

---

## 5. Cross-Platform Capability Comparison

### Capability Matrix

| Capability | Docling-Graph | AI-Knowledge-Graph | Neo4j |
|---|---|---|---|
| **Document ingestion** | PDF, images, Office, markdown, DocLang (via Docling) | Plain text only | Any (via GraphRAG package / LangChain loaders) |
| **Schema definition** | Explicit Pydantic templates (entities, edges, IDs) | None (LLM-driven, implicit) | Property graph model (labels, relationship types, properties) |
| **Extraction method** | LLM (LiteLLM) or VLM (Docling) | LLM (OpenAI-compatible API) | LLM (via GraphRAG Python package / LangChain) |
| **Extraction output** | Validated Pydantic model instances | JSON array of SPO triples | Nodes and relationships in graph DB |
| **Graph structure** | NetworkX `DiGraph` (in-memory) | NetworkX graph → PyVis HTML | Native graph database (persistent) |
| **Node IDs** | Deterministic fingerprint hash (`ClassName_fingerprint`) | LLM-extracted entity names | Internal graph IDs + properties |
| **Edge types** | Declared via `edge(label=...)` in template | LLM-extracted predicates (1–3 words) | Relationship types (Cypher `[:TYPE]`) |
| **Provenance** | Deterministic ledger (chunk/page/char-span anchors) | Chunk index only | Graph metadata + timestamps |
| **Entity resolution** | `NodeIDRegistry` + LLM reconciliation (dense dedupe) | LLM-based standardization | Cypher `MERGE` / graph algorithms |
| **Relationship inference** | Not built-in (template defines structure) | Rule-based + LLM-assisted inference | Graph algorithms + Cypher traversals |
| **Query language** | Python / NetworkX API | None (visualization only) | Cypher |
| **Export formats** | CSV, Cypher, JSON, HTML, Markdown, DocLang | HTML (PyVis), JSON | Cypher, CSV, JSON, Bolt protocol |
| **Visualization** | Interactive HTML (Cytoscape) | Interactive HTML (PyVis/Vis.js) | Neo4j Browser, Bloom |
| **Persistence** | File-based (CSV, JSON, Cypher script) | HTML + JSON files | Native graph database |
| **Scalability** | Single-document (in-memory) | Single-document | Enterprise-scale (billions of nodes) |
| **Vector search** | Not built-in | Not built-in | Native vector indexes |
| **RAG integration** | Export to Neo4j (Cypher/CSV) | Not built-in | GraphRAG (native), LangChain, LlamaIndex |
| **Agent integration** | Docling MCP server (document conversion) | None | MCP server (graph retrieval as agent tool) |
| **LLM providers** | LiteLLM (OpenAI, Mistral, Gemini, watsonx, vLLM, Ollama) | Any OpenAI-compatible (Ollama, LM Studio, OpenAI, LiteLLM) | Any (via GraphRAG / LangChain) |
| **Local inference** | vLLM, Ollama, Docling VLM | Ollama, LM Studio, LiteLLM | N/A (database, not extraction) |
| **License** | MIT (open source) | MIT (open source) | Commercial (Community GPL / Enterprise / Aura) |

### Pipeline Stage Comparison

| Stage | Docling-Graph | AI-Knowledge-Graph |
|---|---|---|
| **Input** | Documents (PDF, images, Office…) via Docling | Plain text files |
| **Chunking** | Structure-preserving (HybridChunker, token-aware) | Word-count based (simple, with overlap) |
| **Extraction** | Schema-validated Pydantic models (LLM or VLM) | Free-form SPO triples (LLM only) |
| **Validation** | Pydantic type validation + identity field checking | None (raw LLM output) |
| **Entity resolution** | Deterministic NodeIDRegistry + optional LLM reconciliation | LLM-based standardization (optional) |
| **Relationship inference** | Not built-in (structure defined by template) | Rule-based + LLM-assisted (optional) |
| **Provenance** | Full ledger with chunk/page/span anchors | Chunk index only |
| **Export** | CSV, Cypher, JSON, HTML, Markdown | HTML, JSON |
| **Visualization** | Cytoscape interactive HTML | PyVis interactive HTML |

### Graph Database Integration Path

```
Docling-Graph                    AI-Knowledge-Graph
     │                                │
     ▼                                ▼
  Export: Cypher script          Export: JSON triples
  or CSV (nodes.csv/edges.csv)        │
     │                                │
     └──────────► Neo4j ◄─────────────┘
                     │
                     ▼
               GraphRAG / LangChain
                     │
                     ▼
               LLM-grounded AI agents
```

Docling-Graph can export Cypher scripts or Neo4j-compatible CSV, making it a **graph construction front-end** for Neo4j. AI-Knowledge-Graph produces JSON triples that could be loaded into Neo4j via Cypher `CREATE`/`MERGE` statements. Neo4j then serves as the **persistent graph store and query layer** for AI systems.

---

## 6. Key Design Principles

### Docling-Graph

1. **Schema-first** — Pydantic templates are the contract: they define what to extract, how it's validated, and how it becomes a graph. The pipeline is domain-agnostic; the template is where all domain knowledge lives.
2. **Validation everywhere** — Every extraction output is validated against the Pydantic schema before graph conversion. Invalid output degrades gracefully (optional fields) rather than failing.
3. **Deterministic grounding** — Provenance is built from pipeline bookkeeping, not LLM calls. Every node traces back to exact source locations.
4. **Stable IDs** — `NodeIDRegistry` ensures the same entity always gets the same node ID across batches, enabling cross-document graph merging.
5. **Backend flexibility** — LLM (local/remote, any provider via LiteLLM) or VLM (local, via Docling) backends with the same pipeline interface.
6. **Contract-driven extraction** — `direct` (single-pass) vs `dense` (skeleton-then-flesh) contracts with auto-selection based on document size and model budget.

### AI-Knowledge-Graph

1. **Schema-free simplicity** — No templates or schemas required; the LLM discovers entities and relationships from raw text.
2. **Pipeline phases** — Extraction → Standardization → Inference, each independently toggleable.
3. **Inference as enrichment** — Both rule-based (transitivity, lexical similarity) and LLM-assisted inference bridge disconnected subgraphs, with inferred edges visually distinguished.
4. **Prompt-centric** — All extraction logic lives in editable prompts; the pipeline is prompt orchestration.
5. **Interactive-first** — The output is an interactive HTML visualization, not a data format; exploration is the primary use case.

### Neo4j

1. **Property graph model** — Nodes, relationships, and properties as first-class citizens; relationships are stored as adjacent pointers (index-free adjacency) for O(1) traversal.
2. **Cypher as the lingua franca** — Declarative, pattern-matching query language optimized for graph traversals.
3. **GraphRAG for AI grounding** — Knowledge graphs provide structured, connected context that vector-only RAG cannot (multi-hop reasoning, entity relationships, explainable retrieval).
4. **Agent memory & tools** — Persistent graph storage for long-term agent memory; MCP exposure makes graph retrieval an agent tool.
5. **ACID compliance** — Transactional integrity for reliable data operations.
6. **Ecosystem integration** — LangChain, LlamaIndex, GraphRAG Python package, cloud marketplaces (AWS, Azure, GCP), Snowflake, Databricks.

---

*Sources: [Docling-Graph GitHub](https://github.com/docling-project/docling-graph), [Docling-Graph docs](https://docling-project.github.io/docling-graph/), [AI-Knowledge-Graph GitHub](https://github.com/robert-mcdermott/ai-knowledge-graph), [Medium article by Robert McDermott](https://robert-mcdermott.medium.com/from-unstructured-text-to-interactive-knowledge-graphs-using-llms-dd02a1f71cd6), [Neo4j AI Systems](https://neo4j.com/use-cases/ai-systems/), [Neo4j Python Driver Manual](https://neo4j.com/docs/python-manual/current/), [Neo4j Cypher Manual](https://neo4j.com/docs/cypher-manual/current/). Last reviewed: July 2026.*