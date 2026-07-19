# AI Platform — OpenAPI Specification (Vendor-Neutral)

> **Derived from:** `architecture_v2.md` (Global Architecture v2 — Consolidated AI Platform Services Map).
> **Scope:** a complete OpenAPI 3.1 specification for every independently-callable service across all six layers (L0–L5) and every domain.
> **Conventions:** transversal standards from the Platform Architecture Standards chapter (S.1–S.33) are factored into reusable components (`$ref`) and applied uniformly. All HTTP/WS endpoints are grouped by layer → domain → module using OpenAPI `tags`. Where an architecture service lists multiple route variants (e.g. `POST /v1/api_keys` / `POST /v1/organizations/api_keys`), only the canonical route is normative; alternatives are recorded in the `description`.

---

## Table of Contents

1. [OpenAPI Document Header](#openapi-document-header)
2. [Components — Reusable Schemas](#components--reusable-schemas)
3. [Layer 0 — Fundamental Technical Services](#layer-0--fundamental-technical-services)
4. [Layer 1 — Compute & Model Serving](#layer-1--compute--model-serving)
5. [Layer 2 — Model Inference & Intelligence APIs](#layer-2--model-inference--intelligence-apis)
6. [Layer 3 — AI Modality Products](#layer-3--ai-modality-products)
7. [Layer 4 — Agentic Orchestration](#layer-4--agentic-orchestration)
8. [Layer 5 — Observability & Administration](#layer-5--observability--administration)

---

## OpenAPI Document Header

```yaml
openapi: 3.1.0
info:
  title: AI Platform API
  version: 2.0.0
  description: |
    Vendor-neutral OpenAPI specification for the consolidated AI Platform,
    covering Layers L0–L5 and every domain/module/service described in
    `architecture_v2.md`. Transversal conventions (auth, versioning, errors,
    pagination, streaming, webhooks, citations, structured-output contract,
    async job state machine, etc.) are factored into reusable components and
    applied uniformly across all layers.
  contact:
    name: AI Platform API Working Group
  license:
    name: Apache-2.0
servers:
  - url: https://api.ai-platform.example/v1
    description: Global production
  - url: https://api.ai-platform.example/v0
    description: Experimental / preview
  - url: wss://api.ai-platform.example/v1
    description: WebSocket (realtime, MCP, voice)
tags:
  # Layer 0
  - name: L0.A API Keys
  - name: L0.A Federation & SSO
  - name: L0.B Organizations & Tenancy
  - name: L0.B Workspaces
  - name: L0.C Roles & Permissions
  - name: L0.D Private Connectivity
  - name: L0.D Outbound Network Policies
  - name: L0.E Files API
  - name: L0.F Vector Stores
  - name: L0.G Environments
  - name: L0.G Sandboxes
  - name: L0.H Cron & Scheduled Tasks
  - name: L0.H Workflows & Pipelines
  - name: L0.I Billing & Usage
  - name: L0.I Spend Limits
  - name: L0.I Subscriptions
  - name: L0.J Rate Limits
  - name: L0.K Processing Tiers
  - name: L0.L Token Counting
  - name: L0.M Compliance
  - name: L0.N Data Residency & Encryption
  - name: L0.O Webhooks
  - name: L0.P SDK & CLI
  - name: L0.Q Routing & Gateway
  - name: L0.R Vaults & Credentials
  - name: L0.R Connections
  - name: L0.S Process & Filesystem
  # Layer 1
  - name: L1.A Hardware & Model Catalog
  - name: L1.B Model Packaging
  - name: L1.C Deployment CRUD
  - name: L1.E Lifecycle & Promotion
  - name: L1.E Autoscaling
  - name: L1.E Health Checks
  - name: L1.G Inference Execution
  - name: L1.G Async Inference
  - name: L1.G Batch Inference
  - name: L1.G RL Rollout
  - name: L1.I Observability Plumbing
  # Layer 2
  - name: L2.A Model Catalog
  - name: L2.B Generation (Modern)
  - name: L2.B Generation (Legacy)
  - name: L2.F Streaming
  - name: L2.G Context Management
  - name: L2.J Embeddings & Rerank
  - name: L2.K Batch
  - name: L2.L Grounding & Citations
  # Layer 3
  - name: L3.A Text & Conversation
  - name: L3.A Classical NLP
  - name: L3.A Custom NLP Training
  - name: L3.B Image Generation
  - name: L3.B Image Editing
  - name: L3.B Layout Composition
  - name: L3.B Image Understanding
  - name: L3.B Image Postprocessing
  - name: L3.B Style & Assets
  - name: L3.B Video Generation
  - name: L3.B Video Editing
  - name: L3.C Voice Assets
  - name: L3.C Audio Preprocessing
  - name: L3.C Speech-to-Text
  - name: L3.C Translation & Dubbing
  - name: L3.C Text-to-Speech
  - name: L3.C Voice Transformation
  - name: L3.C Sound & Music
  - name: L3.C Voice Agents
  - name: L3.D Document Ingestion
  - name: L3.D Document Understanding
  - name: L3.D Chunking & Enrichment
  - name: L3.D Indexing & Graph
  - name: L3.D Query Time
  - name: L3.D Generation & Output
  - name: L3.D Document Transformation
  - name: L3.D Custom Processors
  - name: L3.D Schema Management
  - name: L3.D MCP Tools
  # Layer 4
  - name: L4.A Agent Definition
  - name: L4.A Skills
  - name: L4.B Models (agent-level)
  - name: L4.C Sessions
  - name: L4.E Containers
  - name: L4.F Connectors / MCP
  - name: L4.I Multi-Agent
  - name: L4.J Memory & Knowledge
  - name: L4.K Workflows
  - name: L4.L Channels
  - name: L4.L Voice Channel
  - name: L4.M Plugins & Marketplace
  - name: L4.M External Agents
  - name: L4.N Webhooks (agent)
  # Layer 5
  - name: L5.A Observability Export
  - name: L5.A Hosted UI
  - name: L5.B Attribution
  - name: L5.E Traffic Inspection
  - name: L5.F Judges & Evaluators
  - name: L5.F Eval Campaigns
  - name: L5.F Datasets
  - name: L5.G Feedback
  - name: L5.I Moderation
  - name: L5.I Guardrails
  - name: L5.I PII & Redaction
  - name: L5.I Safety Filters
  - name: L5.I Refusal Fallback
  - name: L5.I Approvals (HITL)
  - name: L5.I Prompt Injection Defense
  - name: L5.J Model Lifecycle Admin
  - name: L5.K Cache Diagnostics
  - name: L5.L Analytics Portals
  - name: L5.M Telemetry Backends
  - name: L5.N Cost & Usage Lookup
security:
  - bearerAuth: []
  - apiKeyHeader: []
  - apiKeyQuery: []
paths: {}
components:
  securitySchemes:
    bearerAuth:
      type: http
      scheme: bearer
      description: "`Authorization: Bearer <token>` (S.1)."
    apiKeyHeader:
      type: apiKey
      in: header
      name: x-api-key
      description: "Alternative API-key header (S.1)."
    apiKeyQuery:
      type: apiKey
      in: query
      name: api-key
      description: "Alternative API-key query parameter (S.1)."
```

---

## Components — Reusable Schemas

These schemas encode the transversal conventions (S.x) and canonical data structures (S.32) so every layer can `$ref` them.

```yaml
components:
  schemas:

    # ---- S.1 / S.2 headers (informational; captured as parameters) ----
    ApiVersionHeader:
      type: object
      description: "Optional versioning headers (S.2)."
      properties:
        anthropic-version:
          type: string
          example: "2023-06-01"
        api-version:
          type: string
          example: "2026-05-20"
        anthropic-beta:
          type: array
          items: { type: string }
          description: "Repeatable beta feature tags (S.2)."
        X-Beta-Features:
          type: array
          items: { type: string }

    # ---- S.4 Error envelope ----
    Error:
      type: object
      required: [type, error]
      properties:
        type: { type: string, example: "error" }
        error:
          type: object
          required: [type, message]
          properties:
            type:
              type: string
              enum:
                - invalid_request_error
                - authentication_error
                - permission_error
                - not_found_error
                - conflict_error
                - rate_limit_error
                - api_error
                - invalid_argument
                - permission_denied
                - failed_precondition
                - service_unavailable
                - internal_error
            message: { type: string }
            code: { type: string }
        request_id: { type: string }

    # ---- S.5 Pagination ----
    PageParams:
      type: object
      properties:
        limit: { type: integer, minimum: 1, maximum: 1000, default: 20 }
        cursor: { type: string, description: "Opaque cursor (after_id / before_id / page)." }
        order:
          type: string
          enum: [asc, desc]
          default: asc
    PageMeta:
      type: object
      properties:
        has_more: { type: boolean }
        first_id: { type: string }
        last_id: { type: string }
        next_page: { type: string }

    # ---- S.7 Stop reasons (union) ----
    StopReason:
      type: string
      enum:
        - stop
        - length
        - content_filter
        - tool_calls
        - end_turn
        - max_tokens
        - stop_sequence
        - tool_use
        - pause_turn
        - refusal
        - model_context_window_exceeded
        - compaction
        - error
        - STOP
        - MAX_TOKENS
        - SAFETY
        - RECITATION

    # ---- S.8 Usage ----
    Usage:
      type: object
      properties:
        prompt_tokens: { type: integer }
        completion_tokens: { type: integer }
        total_tokens: { type: integer }
        input_tokens: { type: integer }
        output_tokens: { type: integer }
        cached_tokens: { type: integer }
        reasoning_tokens: { type: integer }
        cache_creation_input_tokens: { type: integer }
        cache_read_input_tokens: { type: integer }
        cost: { type: number, format: double }
        cost_in_nano_usd: { type: integer }
        service_tier: { type: string }
        iterations:
          type: array
          items:
            type: object
            properties:
              type: { type: string, enum: [message, advisor_message] }
              model: { type: string }
        server_tool_use:
          type: object
          properties:
            web_search_requests: { type: integer }
            web_fetch_requests: { type: integer }
            code_execution_requests: { type: integer }

    # ---- S.11 Idempotency ----
    IdempotencyKey:
      type: string
      description: "Optional Idempotency-Key header for trigger APIs (S.11)."

    # ---- S.15 Confidence ----
    Confidence:
      type: number
      format: float
      minimum: 0
      maximum: 1
    Likelihood:
      type: string
      enum: [UNKNOWN, VERY_UNLIKELY, UNLIKELY, POSSIBLE, LIKELY, VERY_LIKELY]

    # ---- S.16 Async job state machine ----
    AsyncJobStatus:
      type: string
      enum:
        - queued
        - pending
        - in_progress
        - processing
        - completed
        - done
        - failed
        - expired
        - canceled
    AsyncJob:
      type: object
      properties:
        id: { type: string }
        object: { type: string }
        created_at: { type: string, format: date-time }
        status: { $ref: "#/components/schemas/AsyncJobStatus" }
        progress: { type: integer, minimum: 0, maximum: 100 }
        seconds: { type: number }
        size: { type: integer }
        model: { type: string }
        error:
          type: object
          properties:
            code: { type: string }
            message: { type: string }
        errors:
          type: array
          items: { $ref: "#/components/schemas/Error" }

    # ---- S.21 Tool definition (canonical) ----
    ToolDefinition:
      oneOf:
        - type: object
          required: [type, name, description, parameters]
          properties:
            type: { type: string, enum: [function] }
            name: { type: string }
            description: { type: string }
            parameters: { $ref: "#/components/schemas/JsonSchema" }
            strict: { type: boolean, default: false }
            input_examples:
              type: array
              items: { type: object }
            cache_control: { $ref: "#/components/schemas/CacheControl" }
            defer_loading: { type: boolean }
            requires_confirmation: { type: boolean }
            output_schema: { $ref: "#/components/schemas/JsonSchema" }
        - type: object
          description: "Legacy externally-tagged form (S.21)."
          properties:
            type: { type: string, enum: [function] }
            function:
              type: object
              properties:
                name: { type: string }
                description: { type: string }
                parameters: { $ref: "#/components/schemas/JsonSchema" }
                strict: { type: boolean }
        - type: object
          description: "Custom free-form text I/O tool (S.21)."
          properties:
            type: { type: string, enum: [custom] }
            name: { type: string }
            description: { type: string }
        - type: object
          description: "Tool namespace (S.21)."
          properties:
            type: { type: string, enum: [namespace] }
            name: { type: string }
            description: { type: string }
            tools:
              type: array
              items: { $ref: "#/components/schemas/ToolDefinition" }

    ToolChoice:
      oneOf:
        - type: string
          enum: [auto, required, any, none, validated]
        - type: object
          properties:
            type: { type: string, enum: [function, tool, allowed_tools, image_generation] }
            name: { type: string }
            mode: { type: string, enum: [auto, any] }
            tools:
              type: array
              items: { type: string }
        - type: object
          description: "Graduated eagerness."
          properties:
            eagerness: { type: string, enum: [minimal, low, medium, high, xhigh] }
        - type: object
          properties:
            max_tool_calls: { type: integer }

    # ---- S.19 Strict-mode JSON Schema ----
    JsonSchema:
      type: object
      description: "JSON Schema (S.19); subset of keywords supported in strict mode."
      additionalProperties: true

    # ---- S.21 Content block types ----
    ContentBlock:
      oneOf:
        - type: object
          properties:
            type: { type: string, enum: [text] }
            text: { type: string }
            citations:
              type: array
              items: { $ref: "#/components/schemas/Citation" }
        - type: object
          properties:
            type: { type: string, enum: [image] }
            source:
              type: object
              properties:
                type: { type: string, enum: [base64, url, file_id] }
                media_type: { type: string }
                data: { type: string }
                url: { type: string }
                file_id: { type: string }
        - type: object
          properties:
            type: { type: string, enum: [audio] }
            source:
              type: object
              properties:
                type: { type: string, enum: [base64, url, file_id] }
                media_type: { type: string }
                data: { type: string }
        - type: object
          properties:
            type: { type: string, enum: [document] }
            source:
              type: object
              properties:
                type: { type: string, enum: [base64, url, file_id] }
                media_type: { type: string }
                data: { type: string }
                file_id: { type: string }
            citations: { type: object, properties: { enabled: { type: boolean } } }
        - type: object
          description: "Tool use block (S.21)."
          properties:
            type: { type: string, enum: [tool_use] }
            id: { type: string }
            name: { type: string }
            input: { type: object }
            arguments: { type: string }
            caller: { type: object }
            confirmation_status: { type: string }
            safety_decision: { type: object }
            signature: { type: string }
        - type: object
          description: "Tool result block (S.21)."
          properties:
            type: { type: string, enum: [tool_result] }
            tool_use_id: { type: string }
            content:
              type: array
              items: { $ref: "#/components/schemas/ContentBlock" }
            structuredContent: { type: object }
            isError: { type: boolean }
        - type: object
          description: "Thinking / reasoning block (S.18)."
          properties:
            type: { type: string, enum: [thinking, redacted_thinking] }
            thinking: { type: string }
            signature: { type: string }
            encrypted_content: { type: string }
        - type: object
          description: "Server tool result blocks (S.21)."
          properties:
            type:
              type: string
              enum:
                - web_search_tool_result
                - web_fetch_tool_result
                - bash_code_execution_tool_result
                - code_execution_tool_result
                - advisor_tool_result
                - tool_search_tool_result
                - text_editor_code_execution_view_result
                - text_editor_code_execution_str_replace_result
                - text_editor_code_execution_create_result
                - text_editor_code_execution_insert_result

    Message:
      type: object
      required: [role, content]
      properties:
        role:
          type: string
          enum: [system, developer, user, assistant, tool]
        content:
          oneOf:
            - type: string
            - type: array
              items: { $ref: "#/components/schemas/ContentBlock" }
        name: { type: string }
        tool_call_id: { type: string }
        tool_calls:
          type: array
          items:
            type: object
            properties:
              id: { type: string }
              type: { type: string, enum: [function] }
              function:
                type: object
                properties:
                  name: { type: string }
                  arguments: { type: string }
        refusal: { type: string }

    # ---- S.22 Citations ----
    Citation:
      oneOf:
        - type: object
          description: "URL citation (S.22)."
          properties:
            type: { type: string, enum: [url_citation] }
            start_index: { type: integer }
            end_index: { type: integer }
            url: { type: string }
            title: { type: string }
            cited_text: { type: string }
        - type: object
          description: "File citation (S.22)."
          properties:
            type: { type: string, enum: [file_citation, container_file_citation] }
            file_id: { type: string }
            file_name: { type: string }
            filename: { type: string }
            page_number: { type: integer }
            media_id: { type: string }
            custom_metadata: { type: object }
            container_id: { type: string }
        - type: object
          description: "Place citation (S.22)."
          properties:
            type: { type: string, enum: [place_citation] }
            name: { type: string }
            url: { type: string }
        - type: object
          description: "Tool reference chunk (S.22)."
          properties:
            type: { type: string, enum: [tool_reference] }
            tool: { type: string }
            title: { type: string }
            url: { type: string }
            source: { type: string }

    # ---- S.18 Reasoning replay ----
    Reasoning:
      type: object
      properties:
        effort:
          type: string
          enum: [none, minimal, low, medium, high, xhigh, max, ultra]
        exclude: { type: boolean }
        summary:
          type: string
          enum: [auto, concise, detailed]
        encrypted_content: { type: string }
        context:
          type: string
          enum: [auto, all_turns, current_turn]
        mode: { type: string, enum: [standard, pro] }

    # ---- S.27 Cache control ----
    CacheControl:
      type: object
      properties:
        type: { type: string, enum: [ephemeral] }
        ttl: { type: string, enum: ["5m", "1h"] }

    # ---- S.32 Image/Video shared schemas ----
    ImageInput:
      oneOf:
        - type: object
          properties:
            type: { type: string, enum: [url] }
            url: { type: string }
        - type: object
          properties:
            type: { type: string, enum: [base64] }
            base64: { type: string }
            media_type: { type: string }
        - type: object
          properties:
            type: { type: string, enum: [file_id] }
            file_id: { type: string }
        - type: object
          properties:
            type: { type: string, enum: [multipart] }
        - type: object
          properties:
            type: { type: string, enum: [project_ref] }
            ref: { type: string }
    Mask:
      oneOf:
        - type: object
          properties: { type: { type: string, enum: [alpha] }, data: { type: string } }
        - type: object
          properties: { type: { type: string, enum: [bw] }, data: { type: string } }
        - type: object
          properties: { type: { type: string, enum: [grayscale] }, data: { type: string } }
        - type: object
          properties: { type: { type: string, enum: [polygon] }, points: { type: array, items: { type: array, items: { type: number }, minItems: 2, maxItems: 2 } } }
        - type: object
          properties: { type: { type: string, enum: [coco_rle] }, data: { type: string } }
    BoundingBox:
      type: object
      properties:
        format: { type: string, enum: [xyxy, xywh, yxyx, polygon] }
        coordinates:
          oneOf:
            - type: array
              items: { type: number }
              minItems: 4
              maxItems: 4
            - type: array
              items:
                type: array
                items: { type: number }
                minItems: 2
                maxItems: 2
        coordinate_space: { type: string, enum: [pixel, normalized_0_1, normalized_0_1000] }
    StyleSpec:
      oneOf:
        - type: object
          properties: { type: { type: string, enum: [curated] }, name: { type: string } }
        - type: object
          properties: { type: { type: string, enum: [preset] }, preset: { type: string } }
        - type: object
          properties: { type: { type: string, enum: [code] }, code: { type: string } }
        - type: object
          properties: { type: { type: string, enum: [reference] }, reference_images: { type: array, items: { $ref: "#/components/schemas/ImageInput" } } }
        - type: object
          properties: { type: { type: string, enum: [palette] }, colors: { type: array, items: { type: object } } }
        - type: object
          properties: { type: { type: string, enum: [structured] }, description: { type: object } }
    ImageGenerationRequest:
      type: object
      properties:
        prompt: { type: string }
        json_prompt: { type: object, description: "V4JsonPrompt (S.32)." }
        model: { type: string }
        size: { type: string }
        aspect_ratio: { type: string }
        quality: { type: string }
        n: { type: integer, minimum: 1, maximum: 10 }
        output_format:
          type: string
          enum: [png, jpeg, webp, svg, transparent]
        background: { type: string, enum: [transparent, opaque] }
        seed: { type: integer }
        negative_prompt: { type: string }
        style: { $ref: "#/components/schemas/StyleSpec" }
        controls: { type: object }
        guidance: { type: number }
        steps: { type: integer }
        moderation: { type: string, enum: [auto, low] }
        safety_tolerance: { type: integer, minimum: 0, maximum: 6 }
        character_reference_images:
          type: array
          items: { $ref: "#/components/schemas/ImageInput" }
        postprocessing: { type: array, items: { type: string } }
        partial_images: { type: integer, minimum: 0, maximum: 3 }
        storage_options: { type: object }
        action: { type: string, enum: [auto, generate, edit] }
        finetune_id: { type: string }
        finetune_strength: { type: number }
        thinking_level: { type: string, enum: [minimal, low, medium, high] }
        previous_response_id: { type: string }
        image_generation_call_id: { type: string }
    ImageEditRequest:
      type: object
      properties:
        prompt: { type: string }
        image:
          oneOf:
            - { $ref: "#/components/schemas/ImageInput" }
            - type: array
              items: { $ref: "#/components/schemas/ImageInput" }
        mask: { $ref: "#/components/schemas/Mask" }
        strength: { type: number }
        image_weight: { type: number }
        edit_operation: { type: string }
    ImageGenerationResponse:
      type: object
      properties:
        created: { type: integer }
        data:
          type: array
          items:
            type: object
            properties:
              url: { type: string }
              b64_json: { type: string }
              file_output: { type: object }
              revised_prompt: { type: string }
              is_image_safe: { type: boolean }
              content_violation: { type: boolean }
              moderation_details: { type: object }
        usage: { $ref: "#/components/schemas/Usage" }
    VideoGenerationRequest:
      type: object
      properties:
        prompt: { type: string }
        model: { type: string }
        size: { type: string }
        aspect_ratio: { type: string }
        seconds: { type: number }
        duration: { type: number }
        resolution: { type: string }
        seed: { type: integer }
        personGeneration: { type: string, enum: [allow_all, allow_adult, dont_allow] }
        input_reference: { $ref: "#/components/schemas/ImageInput" }
        lastFrame: { $ref: "#/components/schemas/ImageInput" }
        referenceImages:
          type: array
          items: { $ref: "#/components/schemas/ImageInput" }
        characters:
          type: array
          items: { type: object }
        storage_options: { type: object }
    VideoResponse:
      type: object
      properties:
        id: { type: string }
        object: { type: string }
        created_at: { type: string, format: date-time }
        status: { $ref: "#/components/schemas/AsyncJobStatus" }
        progress: { type: integer }
        model: { type: string }
        seconds: { type: number }
        size: { type: integer }
        error: { $ref: "#/components/schemas/Error" }
        video: { type: object, properties: { url: { type: string }, thumbnail: { type: string }, spritesheet: { type: string }, file_id: { type: string } } }
        usage: { $ref: "#/components/schemas/Usage" }

    # ---- S.10 Streaming ----
    StreamOptions:
      type: object
      properties:
        include_usage: { type: boolean }
        stream_options: { type: boolean }

    # ---- S.20 Batch JSONL ----
    BatchInputFile:
      type: object
      properties:
        custom_id: { type: string, pattern: "^[a-zA-Z0-9_-]{1,64}$" }
        method: { type: string }
        url: { type: string }
        body: { type: object }
        params: { type: object }
    Batch:
      type: object
      properties:
        id: { type: string }
        object: { type: string, example: "batch" }
        endpoint: { type: string }
        input_file_id: { type: string }
        completion_window:
          type: string
          enum: ["12h", "24h", "48h", "72h"]
        status:
          type: string
          enum: [validating, in_progress, finalizing, completed, expired, cancelled, failed]
        created_at: { type: string, format: date-time }
        output_file_id: { type: string }
        error_file_id: { type: string }
        request_counts:
          type: object
          properties:
            total: { type: integer }
            completed: { type: integer }
            failed: { type: integer }

    # ---- S.17 Webhook ----
    WebhookSecret:
      type: object
      properties:
        secret: { type: string }
        signing_scheme: { type: string, enum: [hmac_sha256, jwks_rs256, ed25519] }
        jwks_url: { type: string }

    # ---- S.31 Surcharge ----
    Surcharge:
      type: object
      properties:
        capability: { type: string }
        rate: { type: number, description: "Percent uplift (e.g. 0.30 for +30%)." }

    # ---- Common minimal resource ----
    Identifier:
      type: object
      properties:
        id: { type: string }
        object: { type: string }
        created_at: { type: string, format: date-time }
        updated_at: { type: string, format: date-time }

    ListResponse:
      type: object
      properties:
        object: { type: string, example: "list" }
        data: { type: array, items: { type: object } }
        has_more: { type: boolean }
        first_id: { type: string }
        last_id: { type: string }

    # ---- L0 common ----
    ApiKey:
      type: object
      properties:
        id: { type: string }
        name: { type: string }
        prefix: { type: string }
        scopes: { type: array, items: { type: string } }
        expires_at: { type: string, format: date-time }
        workspace_id: { type: string }
        key_type:
          type: string
          enum: [PERSONAL, WORKSPACE_MANAGE_ALL, WORKSPACE_INVOKE, WORKSPACE_EXPORT_METRICS, ADMIN, SERVICE_ACCOUNT]
        created_at: { type: string, format: date-time }
        value:
          type: string
          description: "Returned only on creation."

    Organization:
      type: object
      properties:
        id: { type: string }
        name: { type: string }
        parent_id: { type: string }
        description: { type: string }
    Workspace:
      type: object
      properties:
        id: { type: string }
        name: { type: string }
        organization_id: { type: string }
        default_inference_geo: { type: string }
        allowed_inference_geos: { type: array, items: { type: string } }
        access_mode: { type: string, enum: [private, public] }
        archived: { type: boolean }

    FileObject:
      type: object
      properties:
        id: { type: string }
        object: { type: string, example: "file" }
        bytes: { type: integer }
        created_at: { type: integer }
        filename: { type: string }
        purpose:
          type: string
          enum: [user_data, vision, batch, assistants]
        status:
          type: string
          enum: [PROCESSING, ACTIVE, EXPIRED, FAILED]
        metadata: { type: object }
        expires_at: { type: string, format: date-time }

    VectorStore:
      type: object
      properties:
        id: { type: string }
        name: { type: string }
        chunking_strategy: { type: object }
        file_ids: { type: array, items: { type: string } }
        metadata: { type: object }
        created_at: { type: string, format: date-time }
        expires_after: { type: object }

    # ---- L1 common ----
    Model:
      type: object
      properties:
        id: { type: string }
        context_length: { type: integer }
        architecture:
          type: object
          properties:
            input_modalities: { type: array, items: { type: string } }
            output_modalities: { type: array, items: { type: string } }
            tokenizer: { type: string }
        pricing:
          type: object
          properties:
            prompt: { type: number }
            completion: { type: number }
            cached_prompt: { type: number }
        supported_parameters: { type: array, items: { type: string } }
        reasoning:
          type: object
          properties:
            efforts: { type: array, items: { type: string } }
            mandatory: { type: boolean }
        benchmarks: { type: object }
        expiration_date: { type: string, format: date }
        lifecycle_stage:
          type: string
          enum: [Experimental, Preview, Stable]
    Deployment:
      type: object
      properties:
        id: { type: string }
        model: { type: string }
        hardware: { type: string }
        region: { type: string }
        parallelism: { type: object }
        engine_args: { type: object }
        autoscaling: { type: object }
        mode:
          type: string
          enum: [serverless, dedicated, ptu, dedicated_containers, gpu_clusters, pods]
        routing_key: { type: string }
        state: { type: string }
        created_at: { type: string, format: date-time }

    # ---- L4 common ----
    Agent:
      type: object
      properties:
        id: { type: string }
        version: { type: integer }
        name: { type: string }
        description: { type: string }
        model:
          oneOf:
            - type: string
            - type: object
        system: { type: string }
        tools: { type: array, items: { $ref: "#/components/schemas/ToolDefinition" } }
        mcp_servers: { type: array, items: { type: object } }
        skills: { type: array, items: { type: object } }
        multiagent: { type: object }
        metadata: { type: object }
        tags: { type: array, items: { type: string } }
        structured_output: { $ref: "#/components/schemas/JsonSchema" }
        style: { type: string }
        collaborators: { type: array, items: { type: string } }
        knowledge_base: { type: object }
        timeout_seconds: { type: integer, minimum: 120, maximum: 900 }
        restrictions: { type: object }
        hidden: { type: boolean }
        is_schedulable: { type: boolean }
        context_access_enabled: { type: boolean }
        context_variables: { type: object }
        bundled_agent_id: { type: string }
        created_at: { type: string, format: date-time }
        updated_at: { type: string, format: date-time }
    Session:
      type: object
      properties:
        id: { type: string }
        status:
          type: string
          enum: [idle, running, rescheduling, terminated, in_progress, requires_action, completed, failed, cancelled, pending, async_wait, async_completed, requires_input, expired]
        agent_snapshot: { type: object }
        environment_id: { type: string }
        vault_ids: { type: array, items: { type: string } }
        resources: { type: array, items: { type: object } }
        title: { type: string }
        previous_session_id: { type: string }
        background: { type: boolean }
        store: { type: boolean }
        context: { type: object }
        context_variables: { type: object }
        llm_params: { type: object }
        guardrails: { type: object }

  parameters:
    ApiVersionHeader:
      in: header
      name: api-version
      schema: { type: string }
      description: "Dated API version (S.2)."
    IdempotencyKeyHeader:
      in: header
      name: Idempotency-Key
      schema: { type: string }
      description: "Idempotency key (S.11)."
    OrganizationHeader:
      in: header
      name: OpenAI-Organization
      schema: { type: string }
      description: "Org-scoped calls (S.1)."
    ProjectHeader:
      in: header
      name: x-goog-user-project
      schema: { type: string }
      description: "Project-scoped calls (S.1)."
    CursorParam:
      in: query
      name: cursor
      schema: { type: string }
      description: "Opaque pagination cursor (S.5)."
    LimitParam:
      in: query
      name: limit
      schema: { type: integer, minimum: 1, maximum: 1000, default: 20 }
    OrderParam:
      in: query
      name: order
      schema: { type: string, enum: [asc, desc], default: asc }

  responses:
    BadRequest:
      description: "400 invalid_request_error (S.4)."
      content:
        application/json:
          schema: { $ref: "#/components/schemas/Error" }
    Unauthorized:
      description: "401 authentication_error (S.4)."
      content:
        application/json:
          schema: { $ref: "#/components/schemas/Error" }
    Forbidden:
      description: "403 permission_error (S.4)."
      content:
        application/json:
          schema: { $ref: "#/components/schemas/Error" }
    NotFound:
      description: "404 not_found_error (S.4)."
      content:
        application/json:
          schema: { $ref: "#/components/schemas/Error" }
    Conflict:
      description: "409 conflict_error (S.4)."
      content:
        application/json:
          schema: { $ref: "#/components/schemas/Error" }
    RequestTooLarge:
      description: "413 request_too_large (S.4)."
      content:
        application/json:
          schema: { $ref: "#/components/schemas/Error" }
    RateLimited:
      description: "429 rate_limit_error (S.4)."
      headers:
        x-ratelimit-limit-requests: { schema: { type: integer } }
        x-ratelimit-remaining-requests: { schema: { type: integer } }
        x-ratelimit-reset-requests: { schema: { type: string } }
        Retry-After: { schema: { type: integer } }
      content:
        application/json:
          schema: { $ref: "#/components/schemas/Error" }
    InternalError:
      description: "5xx api_error (S.4)."
      content:
        application/json:
          schema: { $ref: "#/components/schemas/Error" }
    AsyncAccepted:
      description: "202 accepted; async job created (S.16)."
      content:
        application/json:
          schema: { $ref: "#/components/schemas/AsyncJob" }
```

---

## Layer 0 — Fundamental Technical Services

### Domain L0.A — Identity, Authentication & Keys

```yaml
paths:
  /api_keys:
    post:
      tags: [L0.A API Keys]
      operationId: createApiKey
      summary: Create API Key
      description: "Long-lived static API key CRUD (L0.A)."
      requestBody:
        required: true
        content:
          application/json:
            schema:
              type: object
              required: [name]
              properties:
                name: { type: string }
                scopes: { type: array, items: { type: string } }
                expires_at: { type: string, format: date-time }
                workspace_id: { type: string }
      responses:
        "201":
          description: Created
          content:
            application/json:
              schema: { $ref: "#/components/schemas/ApiKey" }
        "400": { $ref: "#/components/responses/BadRequest" }
        "401": { $ref: "#/components/responses/Unauthorized" }
    get:
      tags: [L0.A API Keys]
      operationId: listApiKeys
      summary: List API Keys (prefix+name only)
      parameters:
        - $ref: "#/components/parameters/LimitParam"
        - $ref: "#/components/parameters/CursorParam"
      responses:
        "200":
          description: OK
          content:
            application/json:
              schema:
                allOf:
                  - $ref: "#/components/schemas/ListResponse"
                  - type: object
                    properties:
                      data: { type: array, items: { $ref: "#/components/schemas/ApiKey" } }

  /api_keys/{id_or_prefix}:
    delete:
      tags: [L0.A API Keys]
      operationId: revokeApiKey
      summary: Revoke API Key
      parameters:
        - in: path
          name: id_or_prefix
          required: true
          schema: { type: string }
      responses:
        "204": { description: Deleted }
        "404": { $ref: "#/components/responses/NotFound" }

  /api_keys/register:
    post:
      tags: [L0.A API Keys]
      operationId: registerExternallyOwnedKey
      summary: Register Externally-Owned Key (Ed25519-signed)
      requestBody:
        required: true
        content:
          application/json:
            schema:
              type: object
              properties:
                key_material: { type: string }
                signature: { type: string }
      responses:
        "201":
          description: Registered
          content:
            application/json:
              schema: { $ref: "#/components/schemas/ApiKey" }

  /organizations/service_accounts:
    post:
      tags: [L0.A API Keys]
      operationId: createServiceAccount
      summary: Create Service Account
      requestBody:
        required: true
        content:
          application/json:
            schema:
              type: object
              required: [name]
              properties:
                name: { type: string }
                workspace_id: { type: string }
                scopes: { type: array, items: { type: string } }
      responses:
        "201":
          description: Created
          content:
            application/json:
              schema: { $ref: "#/components/schemas/ApiKey" }

  /organizations/federation_issuers:
    post:
      tags: [L0.A Federation & SSO]
      operationId: createFederationIssuer
      summary: Create Federation Issuer (Workload Identity Federation)
      requestBody:
        required: true
        content:
          application/json:
            schema:
              type: object
              required: [issuer_url, audience]
              properties:
                issuer_url: { type: string }
                audience: { type: string }
                rules: { type: array, items: { type: object } }
      responses:
        "201": { description: Created }
    get:
      tags: [L0.A Federation & SSO]
      operationId: listFederationIssuers
      responses:
        "200":
          description: OK
          content:
            application/json:
              schema: { $ref: "#/components/schemas/ListResponse" }

  /sts/token:
    post:
      tags: [L0.A Federation & SSO]
      operationId: tokenExchange
      summary: Token Exchange (OAuth 2.0 jwt-bearer / STS)
      requestBody:
        required: true
        content:
          application/x-www-form-urlencoded:
            schema:
              type: object
              required: [grant_type, assertion]
              properties:
                grant_type:
                  type: string
                  enum: ["urn:ietf:params:oauth:grant-type:jwt-bearer"]
                assertion: { type: string }
                scope: { type: string }
      responses:
        "200":
          description: Token
          content:
            application/json:
              schema:
                type: object
                properties:
                  access_token: { type: string }
                  token_type: { type: string }
                  expires_in: { type: integer }

  /compliance/access_events:
    get:
      tags: [L0.A Federation & SSO]
      operationId: queryAccessTransparency
      summary: Access Transparency Query
      parameters:
        - in: query
          name: starting_at
          schema: { type: string, format: date-time }
        - in: query
          name: ending_at
          schema: { type: string, format: date-time }
        - in: query
          name: filters
          schema: { type: array, items: { type: string } }
      responses:
        "200":
          description: OK
          content:
            application/json:
              schema: { $ref: "#/components/schemas/ListResponse" }
```

### Domain L0.B — Tenancy, Workspaces & Projects

```yaml
paths:
  /organizations:
    post:
      tags: [L0.B Organizations & Tenancy]
      operationId: createOrganization
      summary: Create Organization (top-level billing & identity container)
      requestBody:
        required: true
        content:
          application/json:
            schema:
              type: object
              required: [name]
              properties:
                name: { type: string }
                parent_id: { type: string }
                description: { type: string }
      responses:
        "201":
          description: Created
          content:
            application/json:
              schema: { $ref: "#/components/schemas/Organization" }
        "400": { $ref: "#/components/responses/BadRequest" }
    get:
      tags: [L0.B Organizations & Tenancy]
      operationId: listOrganizations
      parameters:
        - $ref: "#/components/parameters/LimitParam"
        - $ref: "#/components/parameters/CursorParam"
      responses:
        "200":
          description: OK
          content:
            application/json:
              schema:
                allOf:
                  - $ref: "#/components/schemas/ListResponse"
                  - type: object
                    properties:
                      data: { type: array, items: { $ref: "#/components/schemas/Organization" } }

  /organizations/{id}:
    parameters:
      - in: path
        name: id
        required: true
        schema: { type: string }
    get:
      tags: [L0.B Organizations & Tenancy]
      operationId: getOrganization
      responses:
        "200":
          description: OK
          content:
            application/json:
              schema: { $ref: "#/components/schemas/Organization" }
        "404": { $ref: "#/components/responses/NotFound" }
    patch:
      tags: [L0.B Organizations & Tenancy]
      operationId: patchOrganization
      requestBody:
        required: true
        content:
          application/json:
            schema: { $ref: "#/components/schemas/Organization" }
      responses:
        "200":
          description: OK
          content:
            application/json:
              schema: { $ref: "#/components/schemas/Organization" }
    delete:
      tags: [L0.B Organizations & Tenancy]
      operationId: deleteOrganization
      responses:
        "204": { description: Deleted }

  /organization/invites:
    post:
      tags: [L0.B Organizations & Tenancy]
      operationId: createOrgInvite
      summary: Create Org Invite
      requestBody:
        required: true
        content:
          application/json:
            schema:
              type: object
              required: [email, role]
              properties:
                email: { type: string, format: email }
                role: { type: string }
      responses:
        "201": { description: Invited }

  /organizations/workspaces:
    post:
      tags: [L0.B Workspaces]
      operationId: createWorkspace
      summary: Create Workspace (sub-tenant boundary)
      requestBody:
        required: true
        content:
          application/json:
            schema:
              type: object
              required: [name, organization_id]
              properties:
                name: { type: string }
                organization_id: { type: string }
                default_inference_geo: { type: string }
                allowed_inference_geos: { type: array, items: { type: string } }
      responses:
        "201":
          description: Created
          content:
            application/json:
              schema: { $ref: "#/components/schemas/Workspace" }
    get:
      tags: [L0.B Workspaces]
      operationId: listWorkspaces
      responses:
        "200":
          description: OK
          content:
            application/json:
              schema:
                allOf:
                  - $ref: "#/components/schemas/ListResponse"
                  - type: object
                    properties:
                      data: { type: array, items: { $ref: "#/components/schemas/Workspace" } }

  /organizations/workspaces/{id}:
    parameters:
      - in: path
        name: id
        required: true
        schema: { type: string }
    get:
      tags: [L0.B Workspaces]
      operationId: getWorkspace
      responses:
        "200":
          description: OK
          content:
            application/json:
              schema: { $ref: "#/components/schemas/Workspace" }
    patch:
      tags: [L0.B Workspaces]
      operationId: patchWorkspace
      requestBody:
        required: true
        content:
          application/json:
            schema: { $ref: "#/components/schemas/Workspace" }
      responses:
        "200":
          description: OK
          content:
            application/json:
              schema: { $ref: "#/components/schemas/Workspace" }
    delete:
      tags: [L0.B Workspaces]
      operationId: deleteWorkspace
      responses:
        "204": { description: Deleted }

  /api/admin/workspaces/{id}/add-users:
    post:
      tags: [L0.B Workspaces]
      operationId: addWorkspaceMembers
      parameters:
        - in: path
          name: id
          required: true
          schema: { type: string }
      requestBody:
        required: true
        content:
          application/json:
            schema:
              type: object
              properties:
                workspace_id: { type: string }
                user_ids: { type: array, items: { type: string } }
                role: { type: string }
      responses:
        "200": { description: OK }

  /organizations/{org}/projects:
    post:
      tags: [L0.B Workspaces]
      operationId: createProject
      parameters:
        - in: path
          name: org
          required: true
          schema: { type: string }
      requestBody:
        required: true
        content:
          application/json:
            schema:
              type: object
              properties:
                project: { type: string }
      responses:
        "201": { description: Created }

  /gateway/groups:
    post:
      tags: [L0.B Workspaces]
      operationId: createGatewayGroup
      summary: Create hierarchical billing entity with inherited limits
      requestBody:
        required: true
        content:
          application/json:
            schema:
              type: object
              properties:
                name: { type: string }
                parent_id: { type: string }
                limits: { type: object }
      responses:
        "201": { description: Created }

  /gateway/groups/{id}/usage:
    get:
      tags: [L0.B Workspaces]
      operationId: getGatewayGroupUsage
      parameters:
        - in: path
          name: id
          required: true
          schema: { type: string }
      responses:
        "200": { description: OK }

  /api/admin/users:
    get:
      tags: [L0.B Workspaces]
      operationId: listUsers
      responses:
        "200": { description: OK }
    post:
      tags: [L0.B Workspaces]
      operationId: createUser
      requestBody:
        required: true
        content:
          application/json:
            schema:
              type: object
              required: [email, name, role]
              properties:
                email: { type: string, format: email }
                name: { type: string }
                role: { type: string }
      responses:
        "201": { description: Created }

  /api/admin/users/{id}:
    parameters:
      - in: path
        name: id
        required: true
        schema: { type: string }
    get:
      tags: [L0.B Workspaces]
      operationId: getUser
      responses: { "200": { description: OK } }
    patch:
      tags: [L0.B Workspaces]
      operationId: patchUser
      responses: { "200": { description: OK } }
    delete:
      tags: [L0.B Workspaces]
      operationId: deleteUser
      responses: { "204": { description: Deleted } }

  /workspaces/{id}/tenants:
    post:
      tags: [L0.B Workspaces]
      operationId: createTenant
      summary: Tenant lifecycle (ACTIVE → INACTIVE → OFFLOADED)
      parameters:
        - in: path
          name: id
          required: true
          schema: { type: string }
      requestBody:
        required: true
        content:
          application/json:
            schema:
              type: object
              properties:
                tenant: { type: object }
      responses:
        "201": { description: Created }
```

### Domain L0.C — Roles, RBAC & Groups

```yaml
paths:
  /api/admin/roles:
    get:
      tags: [L0.C Roles & Permissions]
      operationId: listRoles
      summary: List assignable roles at org + workspace scope
      responses:
        "200":
          description: OK
          content:
            application/json:
              schema: { $ref: "#/components/schemas/ListResponse" }

  /api/admin/user-groups:
    get:
      tags: [L0.C Roles & Permissions]
      operationId: listUserGroups
      summary: List User Groups (SCIM-syncable)
      parameters:
        - in: query
          name: name
          schema: { type: string }
        - in: query
          name: member_ids
          schema: { type: array, items: { type: string } }
      responses:
        "200": { description: OK }
```

### Domain L0.D — Network & Private Connectivity

```yaml
paths:
  /v2/privatelink_healthcheck:
    get:
      tags: [L0.D Private Connectivity]
      operationId: privatelinkHealthcheck
      summary: Private Link / PSC Healthcheck (regional)
      parameters:
        - in: query
          name: region
          required: true
          schema: { type: string }
      responses:
        "200": { description: OK }

  /v1/tunnel/{tunnel_id}:
    post:
      tags: [L0.D Private Connectivity]
      operationId: tunnelInvoke
      summary: Tunnel-Client Invocation (outbound-only HTTPS to platform-hosted MCP)
      parameters:
        - in: path
          name: tunnel_id
          required: true
          schema: { type: string }
        - in: header
          name: CONTROL_PLANE_API_KEY
          required: true
          schema: { type: string }
      requestBody:
        required: true
        content:
          application/json:
            schema:
              type: object
              properties:
                mcp_command: { type: string }
                mcp_server_url: { type: string }
                base_url: { type: string }
      responses:
        "200": { description: OK }

  /v1/outbound-policies:
    post:
      tags: [L0.D Outbound Network Policies]
      operationId: createOutboundPolicy
      summary: Create Outbound Policy (network domain rules)
      requestBody:
        required: true
        content:
          application/json:
            schema:
              type: object
              properties:
                rules: { type: array, items: { type: object } }
                allow_mcp_servers: { type: boolean }
                allow_package_managers: { type: boolean }
      responses:
        "201": { description: Created }

  /v1/outbound-policies/check:
    post:
      tags: [L0.D Outbound Network Policies]
      operationId: checkOutboundPolicy
      responses: { "200": { description: OK } }
```

### Domain L0.E — Files & Object Storage

```yaml
paths:
  /v1/files:
    post:
      tags: [L0.E Files API]
      operationId: uploadFile
      summary: Upload File (multipart / URL / base64)
      description: |
        Purpose variants: user_data / vision / batch / assistants (L0.E).
        Supports resumable upload → returns uri + name + state; processing
        state poll (PROCESSING → ACTIVE). Idempotent via `external_id`
        deterministic UUID5 (S.11).
      requestBody:
        required: true
        content:
          multipart/form-data:
            schema:
              type: object
              required: [file, purpose]
              properties:
                file:
                  type: string
                  format: binary
                purpose:
                  type: string
                  enum: [user_data, vision, batch, assistants]
                filename: { type: string }
                metadata: { type: object }
                external_id: { type: string, description: "Idempotent UUID5 path (S.11)." }
      responses:
        "201":
          description: Created
          content:
            application/json:
              schema: { $ref: "#/components/schemas/FileObject" }
        "413": { $ref: "#/components/responses/RequestTooLarge" }
    get:
      tags: [L0.E Files API]
      operationId: listFiles
      parameters:
        - $ref: "#/components/parameters/LimitParam"
        - $ref: "#/components/parameters/CursorParam"
        - in: query
          name: purpose
          schema: { type: string }
      responses:
        "200":
          description: OK
          content:
            application/json:
              schema:
                allOf:
                  - $ref: "#/components/schemas/ListResponse"
                  - type: object
                    properties:
                      data: { type: array, items: { $ref: "#/components/schemas/FileObject" } }

  /v1/files/bulk-delete:
    post:
      tags: [L0.E Files API]
      operationId: bulkDeleteFiles
      requestBody:
        required: true
        content:
          application/json:
            schema:
              type: object
              properties:
                ids: { type: array, items: { type: string } }
      responses:
        "204": { description: Deleted }

  /v1/files/uploads:
    post:
      tags: [L0.E Files API]
      operationId: startResumableUpload
      summary: Start resumable upload session
      responses: { "201": { description: Created } }

  /v1/files/uploads/complete:
    post:
      tags: [L0.E Files API]
      operationId: completeResumableUpload
      responses: { "200": { description: OK } }

  /v1/files/uploads/abort:
    post:
      tags: [L0.E Files API]
      operationId: abortResumableUpload
      responses: { "204": { description: Aborted } }

  /v1/files/prechunked:
    post:
      tags: [L0.E Files API]
      operationId: prechunkedUpload
      summary: Prechunked upload (MXJSON)
      responses: { "201": { description: Created } }

  /v1/files/{id}:
    parameters:
      - in: path
        name: id
        required: true
        schema: { type: string }
    get:
      tags: [L0.E Files API]
      operationId: getFile
      summary: Get File (metadata + status)
      responses:
        "200":
          description: OK
          content:
            application/json:
              schema: { $ref: "#/components/schemas/FileObject" }
        "404": { $ref: "#/components/responses/NotFound" }
    patch:
      tags: [L0.E Files API]
      operationId: patchFile
      summary: Patch File metadata
      requestBody:
        required: true
        content:
          application/json:
            schema:
              type: object
              properties:
                metadata: { type: object }
                expires_at: { type: string, format: date-time }
      responses:
        "200":
          description: OK
          content:
            application/json:
              schema: { $ref: "#/components/schemas/FileObject" }
    delete:
      tags: [L0.E Files API]
      operationId: deleteFile
      responses:
        "204": { description: Deleted }

  /v1/files/{id}/content:
    get:
      tags: [L0.E Files API]
      operationId: downloadFileContent
      parameters:
        - in: path
          name: id
          required: true
          schema: { type: string }
      responses:
        "200":
          description: Binary content
          content:
            application/octet-stream:
              schema: { type: string, format: binary }

  /v1/files/{id}/download:
    get:
      tags: [L0.E Files API]
      operationId: downloadFileOriginalOrPdf
      parameters:
        - in: path
          name: id
          required: true
          schema: { type: string }
        - in: query
          name: format
          schema: { type: string, enum: [original, rendered_pdf] }
      responses:
        "200":
          description: Binary
          content:
            application/octet-stream:
              schema: { type: string, format: binary }

  /v1/files/{id}/chunks:
    get:
      tags: [L0.E Files API]
      operationId: downloadFileChunks
      parameters:
        - in: path
          name: id
          required: true
          schema: { type: string }
      responses: { "200": { description: OK } }
```

### Domain L0.F — Storage & Databases (vector / graph)

```yaml
paths:
  /v1/vector_stores:
    post:
      tags: [L0.F Vector Stores]
      operationId: createVectorStore
      requestBody:
        required: true
        content:
          application/json:
            schema:
              type: object
              required: [name]
              properties:
                name: { type: string }
                chunking_strategy: { type: object }
                file_ids: { type: array, items: { type: string } }
                metadata: { type: object }
      responses:
        "201":
          description: Created
          content:
            application/json:
              schema: { $ref: "#/components/schemas/VectorStore" }
    get:
      tags: [L0.F Vector Stores]
      operationId: listVectorStores
      responses:
        "200":
          description: OK
          content:
            application/json:
              schema:
                allOf:
                  - $ref: "#/components/schemas/ListResponse"
                  - type: object
                    properties:
                      data: { type: array, items: { $ref: "#/components/schemas/VectorStore" } }

  /v1/vector_stores/{id}:
    parameters:
      - in: path
        name: id
        required: true
        schema: { type: string }
    get:
      tags: [L0.F Vector Stores]
      operationId: getVectorStore
      responses:
        "200":
          description: OK
          content:
            application/json:
              schema: { $ref: "#/components/schemas/VectorStore" }
    delete:
      tags: [L0.F Vector Stores]
      operationId: deleteVectorStore
      responses: { "204": { description: Deleted } }

  /v1/vector_stores/{id}/files:
    post:
      tags: [L0.F Vector Stores]
      operationId: attachFileToVectorStore
      parameters:
        - in: path
          name: id
          required: true
          schema: { type: string }
      requestBody:
        required: true
        content:
          application/json:
            schema:
              type: object
              required: [file_id]
              properties:
                file_id: { type: string }
      responses: { "201": { description: Attached } }
    get:
      tags: [L0.F Vector Stores]
      operationId: listVectorStoreFiles
      parameters:
        - in: path
          name: id
          required: true
          schema: { type: string }
      responses: { "200": { description: OK } }

  /v1/vector_stores/{id}/search:
    post:
      tags: [L0.F Vector Stores]
      operationId: searchVectorStore
      summary: Retrieve from Vector Store (max_num_results ≤ 50)
      parameters:
        - in: path
          name: id
          required: true
          schema: { type: string }
      requestBody:
        required: true
        content:
          application/json:
            schema:
              type: object
              required: [query]
              properties:
                query: { type: string }
                rewrite_query: { type: boolean }
                max_num_results: { type: integer, minimum: 1, maximum: 50 }
                attribute_filter: { type: object }
                ranking_options:
                  type: object
                  properties:
                    ranker: { type: string }
                    score_threshold: { type: number }
                    hybrid_search:
                      type: object
                      properties:
                        embedding_weight: { type: number }
                        text_weight: { type: number }
      responses:
        "200": { description: OK }

  /v1/agents/documents:
    post:
      tags: [L0.F Vector Stores]
      operationId: documentsApi
      summary: Documents API (up to 100 bulk), metadata filters, top_k
      requestBody:
        required: true
        content:
          application/json:
            schema:
              type: object
              properties:
                documents:
                  type: array
                  maxItems: 100
                  items: { type: object }
                filters: { type: object }
                top_k: { type: integer }
      responses: { "200": { description: OK } }
```

### Domain L0.G — Environments & Sandboxes (Provisioning)

```yaml
paths:
  /v1/environments:
    post:
      tags: [L0.G Environments]
      operationId: createEnvironment
      summary: Create Environment (managed cloud / self-hosted / git worktree / container cache)
      requestBody:
        required: true
        content:
          application/json:
            schema:
              type: object
              required: [name, type]
              properties:
                name: { type: string }
                type:
                  type: string
                  enum: [managed_cloud, self_hosted_local, git_worktree, container_cache]
                sources: { type: array, items: { type: object } }
                mounts: { type: array, items: { type: object } }
                network_policy:
                  type: object
                  properties:
                    mode: { type: string, enum: [unrestricted, limited] }
                    allowed_hosts: { type: array, items: { type: string } }
                credentials: { type: object }
                resource_limits: { type: object }
                setup_scripts: { type: array, items: { type: string } }
                permission_profiles: { type: array, items: { type: object } }
      responses:
        "201": { description: Created }
    get:
      tags: [L0.G Environments]
      operationId: listEnvironments
      responses: { "200": { description: OK } }

  /v1/environments/{id}:
    parameters:
      - in: path
        name: id
        required: true
        schema: { type: string }
    get:
      tags: [L0.G Environments]
      operationId: getEnvironment
      responses: { "200": { description: OK } }
    post:
      tags: [L0.G Environments]
      operationId: updateEnvironment
      summary: Update Environment
      responses: { "200": { description: OK } }
    delete:
      tags: [L0.G Environments]
      operationId: deleteEnvironment
      responses: { "204": { description: Deleted } }

  /v1/environments/{id}/archive:
    post:
      tags: [L0.G Environments]
      operationId: archiveEnvironment
      parameters:
        - in: path
          name: id
          required: true
          schema: { type: string }
      responses: { "204": { description: Archived } }

  /v1/sandboxes:
    post:
      tags: [L0.G Sandboxes]
      operationId: createSandbox
      summary: Create Sandbox (Git-like branching, OCI images)
      requestBody:
        required: true
        content:
          application/json:
            schema:
              type: object
              properties:
                image: { type: string }
                branch: { type: string }
      responses:
        "201":
          description: Created
          content:
            application/json:
              schema:
                type: object
                properties:
                  id: { type: string }

  /v1/sandboxes/{id}/executions:
    post:
      tags: [L0.G Sandboxes]
      operationId: runCodeInSandbox
      summary: Run Code in Sandbox (async operation)
      parameters:
        - in: path
          name: id
          required: true
          schema: { type: string }
      requestBody:
        required: true
        content:
          application/json:
            schema:
              type: object
              properties:
                code: { type: string }
                async: { type: boolean }
      responses:
        "202": { $ref: "#/components/responses/AsyncAccepted" }

  /v1/sandboxes/{id}/checkpoints/{cid}/branch:
    post:
      tags: [L0.G Sandboxes]
      operationId: branchSandbox
      summary: Branch Sandbox (fork at checkpoint)
      parameters:
        - in: path
          name: id
          required: true
          schema: { type: string }
        - in: path
          name: cid
          required: true
          schema: { type: string }
      responses: { "201": { description: Branched } }

  /v1/sandboxes/{id}/checkpoints/{cid}/rollback:
    post:
      tags: [L0.G Sandboxes]
      operationId: rollbackSandbox
      summary: Rollback Sandbox to checkpoint
      parameters:
        - in: path
          name: id
          required: true
          schema: { type: string }
        - in: path
          name: cid
          required: true
          schema: { type: string }
      responses: { "200": { description: Rolled back } }

  /v1/sandboxes/{id}/operations:
    get:
      tags: [L0.G Sandboxes]
      operationId: pollSandboxOperations
      parameters:
        - in: path
          name: id
          required: true
          schema: { type: string }
      responses: { "200": { description: OK } }
```

### Domain L0.H — Workflows & Scheduled Jobs (Cron)

```yaml
paths:
  /v1/deployments:
    post:
      tags: [L0.H Cron & Scheduled Tasks]
      operationId: createCronDeployment
      summary: Create Cron Deployment (DST-aware; ≤10 s jitter; ≤1000 deployments/org)
      requestBody:
        required: true
        content:
          application/json:
            schema:
              type: object
              required: [agent_ref, schedule]
              properties:
                agent_ref: { type: string }
                schedule:
                  type: object
                  description: "RFC 5545 RRULE or cadence enum."
                  properties:
                    rrule: { type: string }
                    cadence:
                      type: string
                      enum: [once, daily, weekly, monthly, yearly]
                timezone: { type: string }
                max_surge: { type: integer }
                max_unavailable: { type: integer }
      responses:
        "201": { description: Created }

  /v1/deployments/{id}/{action}:
    post:
      tags: [L0.H Cron & Scheduled Tasks]
      operationId: deploymentAction
      summary: Pause / Unpause / Archive / Run Deployment
      parameters:
        - in: path
          name: id
          required: true
          schema: { type: string }
        - in: path
          name: action
          required: true
          schema: { type: string, enum: [pause, unpause, archive, run] }
      responses: { "200": { description: OK } }

  /v1/deployments/{id}/runs:
    get:
      tags: [L0.H Cron & Scheduled Tasks]
      operationId: listDeploymentRuns
      parameters:
        - in: path
          name: id
          required: true
          schema: { type: string }
      responses: { "200": { description: OK } }

  /v1/workspace_agents/trigger:
    post:
      tags: [L0.H Cron & Scheduled Tasks]
      operationId: triggerWorkspaceAgent
      summary: Trigger Workspace Agent (Idempotency-Key)
      parameters:
        - $ref: "#/components/parameters/IdempotencyKeyHeader"
      responses: { "202": { description: Triggered } }

  /v1/scheduled_tasks:
    post:
      tags: [L0.H Cron & Scheduled Tasks]
      operationId: createScheduledTask
      summary: Scheduled Task CRUD (cadence, unattended approval gating)
      requestBody:
        required: true
        content:
          application/json:
            schema:
              type: object
              properties:
                cadence: { type: string }
                approval_gating: { type: string }
      responses: { "201": { description: Created } }

  /v1/pipelines:
    post:
      tags: [L0.H Workflows & Pipelines]
      operationId: createPipeline
      summary: Create Pipeline (declarative YAML/Python; draft → saved → published; immutable versioned snapshots)
      requestBody:
        required: true
        content:
          application/json:
            schema:
              type: object
              properties:
                config: { type: object }
      responses: { "201": { description: Created } }

  /v1/pipelines/{id}/run:
    post:
      tags: [L0.H Workflows & Pipelines]
      operationId: runPipeline
      parameters:
        - in: path
          name: id
          required: true
          schema: { type: string }
      responses: { "202": { $ref: "#/components/responses/AsyncAccepted" } }

  /v1/pipelines/{id}/executions:
    get:
      tags: [L0.H Workflows & Pipelines]
      operationId: listPipelineExecutions
      parameters:
        - in: path
          name: id
          required: true
          schema: { type: string }
      responses: { "200": { description: OK } }

  /v1/pipelines/{id}/optimize:
    post:
      tags: [L0.H Workflows & Pipelines]
      operationId: optimizePipeline
      summary: Optimize Pipeline (MOAR — offline MCTS optimization; Pareto frontier)
      parameters:
        - in: path
          name: id
          required: true
          schema: { type: string }
      responses: { "202": { $ref: "#/components/responses/AsyncAccepted" } }

  /v1/workflows:
    post:
      tags: [L0.H Workflows & Pipelines]
      operationId: createWorkflow
      summary: Create Workflow (Temporal-based; long-running, fault-tolerant)
      requestBody:
        required: true
        content:
          application/json:
            schema:
              type: object
              properties:
                definition: { type: object }
      responses: { "201": { description: Created } }
```

### Domain L0.I — Billing, Usage & Spend

```yaml
paths:
  /v1/organizations/usage_report:
    get:
      tags: [L0.I Billing & Usage]
      operationId: usageReport
      parameters:
        - in: query
          name: starting_at
          schema: { type: string, format: date-time }
        - in: query
          name: ending_at
          schema: { type: string, format: date-time }
        - in: query
          name: group_by
          schema: { type: array, items: { type: string } }
      responses: { "200": { description: OK } }

  /api/admin/usage:
    get:
      tags: [L0.I Billing & Usage]
      operationId: adminUsage
      responses: { "200": { description: OK } }

  /v1/organizations/cost_report:
    get:
      tags: [L0.I Billing & Usage]
      operationId: costReport
      parameters:
        - in: query
          name: starting_at
          schema: { type: string, format: date-time }
        - in: query
          name: ending_at
          schema: { type: string, format: date-time }
        - in: query
          name: group_by
          schema: { type: array, items: { type: string } }
      responses: { "200": { description: OK } }

  /api/v1/generation:
    get:
      tags: [L0.I Billing & Usage]
      operationId: generationStats
      summary: Generation Stats (async token counts + cost lookup; X-Generation-Id)
      parameters:
        - in: query
          name: id
          required: true
          schema: { type: string }
      responses: { "200": { description: OK } }

  /v1/organizations/{surface}/analytics:
    get:
      tags: [L0.I Billing & Usage]
      operationId: surfaceAnalytics
      parameters:
        - in: path
          name: surface
          required: true
          schema: { type: string }
        - in: query
          name: time_range
          schema: { type: string }
      responses: { "200": { description: OK } }

  /v1/organization/projects/{id}/spend_alerts:
    post:
      tags: [L0.I Spend Limits]
      operationId: createSpendAlert
      parameters:
        - in: path
          name: id
          required: true
          schema: { type: string }
      requestBody:
        required: true
        content:
          application/json:
            schema:
              type: object
              required: [threshold_amount, currency, interval, notification_channel]
              properties:
                threshold_amount: { type: integer, description: "Cents (S.8 minor units)." }
                currency: { type: string }
                interval: { type: string }
                notification_channel:
                  type: object
                  properties:
                    type: { type: string, enum: [email] }
                    recipients: { type: array, items: { type: string } }
                    subject_prefix: { type: string }
      responses: { "201": { description: Created } }

  /v1/organizations/spend_limits/effective:
    get:
      tags: [L0.I Spend Limits]
      operationId: readEffectiveSpendLimits
      parameters:
        - in: query
          name: scope
          schema: { type: string }
        - in: query
          name: period
          schema: { type: string }
      responses: { "200": { description: OK } }

  /v1/organizations/spend_limit_increase_requests/{id}/approve:
    post:
      tags: [L0.I Spend Limits]
      operationId: approveSpendLimitIncrease
      parameters:
        - in: path
          name: id
          required: true
          schema: { type: string }
      responses: { "200": { description: OK } }

  /v1/organizations/spend_limit_increase_requests/{id}/deny:
    post:
      tags: [L0.I Spend Limits]
      operationId: denySpendLimitIncrease
      parameters:
        - in: path
          name: id
          required: true
          schema: { type: string }
      responses: { "200": { description: OK } }

  /api/admin/spend-limit:
    post:
      tags: [L0.I Spend Limits]
      operationId: adminSpendLimit
      responses: { "200": { description: OK } }
    get:
      tags: [L0.I Spend Limits]
      operationId: getAdminSpendLimit
      responses: { "200": { description: OK } }

  /v1/billing/endpoints:
    get:
      tags: [L0.I Billing & Usage]
      operationId: billingEndpoints
      responses: { "200": { description: OK } }

  /v1/billing/pods:
    get:
      tags: [L0.I Billing & Usage]
      operationId: billingPods
      responses: { "200": { description: OK } }

  /v1/subscriptions:
    post:
      tags: [L0.I Subscriptions]
      operationId: manageSubscription
      summary: Manage Subscription / Billing Plan (prepay vs postpay, credits)
      requestBody:
        required: true
        content:
          application/json:
            schema:
              type: object
              properties:
                plan_id: { type: string }
                seats: { type: integer }
                payment_method: { type: string }
      responses: { "200": { description: OK } }

  /v1/priority_tier:
    post:
      tags: [L0.I Subscriptions]
      operationId: createPriorityTier
      summary: Priority Tier (committed capacity with burndown rates)
      requestBody:
        required: true
        content:
          application/json:
            schema:
              type: object
              properties:
                commitment: { type: object }
      responses: { "201": { description: Created } }
```

### Domain L0.J — Quotas, Rate Limits & Usage Tiers

```yaml
paths:
  /v1/fine_tuning/model_limits:
    get:
      tags: [L0.J Rate Limits]
      operationId: readFineTuningModelLimits
      responses: { "200": { description: OK } }

  /v1/organizations/rate_limits:
    get:
      tags: [L0.J Rate Limits]
      operationId: readOrgRateLimits
      parameters:
        - in: query
          name: scope
          schema: { type: string }
        - in: query
          name: model
          schema: { type: string }
      responses: { "200": { description: OK } }

  /v1/organizations/workspaces/{id}/rate_limits:
    get:
      tags: [L0.J Rate Limits]
      operationId: readWorkspaceRateLimits
      parameters:
        - in: path
          name: id
          required: true
          schema: { type: string }
      responses: { "200": { description: OK } }

  /api/admin/rate-limit:
    get:
      tags: [L0.J Rate Limits]
      operationId: adminRateLimit
      responses: { "200": { description: OK } }
```

### Domain L0.K — Processing Tiers

```yaml
paths:
  # Service Tier Selector is a request-level param on inference endpoints;
  # this path documents the tier metadata lookup.
  /v1/service_tiers:
    get:
      tags: [L0.K Processing Tiers]
      operationId: listServiceTiers
      summary: List processing tiers (flex / priority / auto / default / standard / standard_only)
      responses:
        "200":
          description: OK
          content:
            application/json:
              schema:
                type: array
                items:
                  type: object
                  properties:
                    tier:
                      type: string
                      enum: [flex, priority, auto, default, standard, standard_only]
                    price_multiplier: { type: number }
                    latency_profile: { type: string }
                    sheddable: { type: boolean }
```

### Domain L0.L — Caching & Token Counting (governance facet)

```yaml
paths:
  /v1/messages/count_tokens:
    post:
      tags: [L0.L Token Counting]
      operationId: messagesCountTokens
      summary: Messages Count Tokens (same as Messages; `context_management`)
      description: "Returns input_tokens (post-edit) + original_input_tokens. Free, RPM-limited (S.8)."
      requestBody:
        required: true
        content:
          application/json:
            schema:
              type: object
              properties:
                model: { type: string }
                messages:
                  type: array
                  items: { $ref: "#/components/schemas/Message" }
                context_management: { type: object }
      responses:
        "200":
          description: OK
          content:
            application/json:
              schema:
                type: object
                properties:
                  input_tokens: { type: integer }
                  original_input_tokens: { type: integer }

  /v1/responses/input_tokens:
    post:
      tags: [L0.L Token Counting]
      operationId: responsesInputTokens
      summary: Responses Input Tokens (model-exact token count handling images/files/tools)
      requestBody:
        required: true
        content:
          application/json:
            schema:
              type: object
              required: [model, input]
              properties:
                model: { type: string }
                input: { type: object }
                tools: { type: array, items: { $ref: "#/components/schemas/ToolDefinition" } }
      responses:
        "200":
          description: OK
          content:
            application/json:
              schema:
                type: object
                properties:
                  input_tokens: { type: integer }
```

### Domain L0.M — Compliance, Privacy, Data Retention & Legal

```yaml
paths:
  /v1/compliance/activities:
    get:
      tags: [L0.M Compliance]
      operationId: listComplianceActivities
      summary: List Compliance Activities (eDiscovery, DLP, SIEM, chain-of-custody)
      description: "Shared 600 RPM/parent-org across all /v1/compliance/ endpoints."
      parameters:
        - in: query
          name: after_id
          schema: { type: string }
        - in: query
          name: order_by
          schema: { type: string }
        - in: query
          name: starting_at
          schema: { type: string, format: date-time }
        - in: query
          name: ending_at
          schema: { type: string, format: date-time }
      responses:
        "200": { description: OK }
        "429": { $ref: "#/components/responses/RateLimited" }

  /v1/compliance/apps/chats:
    get:
      tags: [L0.M Compliance]
      operationId: complianceListChats
      responses: { "200": { description: OK } }

  /v1/compliance/apps/chats/{chat_id}:
    parameters:
      - in: path
        name: chat_id
        required: true
        schema: { type: string }
    delete:
      tags: [L0.M Compliance]
      operationId: complianceDeleteChat
      responses: { "204": { description: Deleted } }

  /v1/compliance/apps/chats/{chat_id}/messages:
    get:
      tags: [L0.M Compliance]
      operationId: complianceListChatMessages
      parameters:
        - in: path
          name: chat_id
          required: true
          schema: { type: string }
      responses: { "200": { description: OK } }

  /v1/compliance/apps/chats/files/{file_id}/content:
    get:
      tags: [L0.M Compliance]
      operationId: complianceReadChatFile
      parameters:
        - in: path
          name: file_id
          required: true
          schema: { type: string }
      responses: { "200": { description: OK } }

  /v1/compliance/files/{file_id}:
    parameters:
      - in: path
        name: file_id
        required: true
        schema: { type: string }
    get:
      tags: [L0.M Compliance]
      operationId: complianceReadFile
      responses: { "200": { description: OK } }
    delete:
      tags: [L0.M Compliance]
      operationId: complianceDeleteFile
      responses: { "204": { description: Deleted } }

  /v1/compliance/generated_files/{gen_file_id}/content:
    get:
      tags: [L0.M Compliance]
      operationId: complianceReadGeneratedFile
      parameters:
        - in: path
          name: gen_file_id
          required: true
          schema: { type: string }
      responses: { "200": { description: OK } }

  /v1/compliance/artifacts/{artifact_version_id}/content:
    get:
      tags: [L0.M Compliance]
      operationId: complianceReadArtifact
      parameters:
        - in: path
          name: artifact_version_id
          required: true
          schema: { type: string }
      responses: { "200": { description: OK } }

  /v1/compliance/apps/projects:
    get:
      tags: [L0.M Compliance]
      operationId: complianceListProjects
      responses: { "200": { description: OK } }

  /v1/compliance/apps/projects/{project_id}:
    delete:
      tags: [L0.M Compliance]
      operationId: complianceDeleteProject
      parameters:
        - in: path
          name: project_id
          required: true
          schema: { type: string }
      responses: { "204": { description: Deleted } }
```

### Domain L0.N — Data Residency, Encryption & Retention

```yaml
paths:
  /v1/organization/projects/{id}/data_retention:
    patch:
      tags: [L0.N Data Residency & Encryption]
      operationId: setDataRetention
      summary: Set Data Residency / Retention Policy
      parameters:
        - in: path
          name: id
          required: true
          schema: { type: string }
      requestBody:
        required: true
        content:
          application/json:
            schema:
              type: object
              properties:
                retention_type:
                  type: string
                  enum: [organization_default, zero_data_retention, modified_abuse_monitoring, none]
      responses: { "200": { description: OK } }

  /v1/organization/projects/{id}/model_permissions:
    patch:
      tags: [L0.N Data Residency & Encryption]
      operationId: setModelPermissions
      summary: Model Allowlist/Denylist (project-level model permissions)
      parameters:
        - in: path
          name: id
          required: true
          schema: { type: string }
      requestBody:
        required: true
        content:
          application/json:
            schema:
              type: object
              required: [mode, model_ids]
              properties:
                mode: { type: string, enum: [allow_list, deny_list] }
                model_ids: { type: array, items: { type: string } }
      responses: { "200": { description: OK } }

  /v1/encryption/cmek:
    post:
      tags: [L0.N Data Residency & Encryption]
      operationId: configureCmek
      summary: Encryption-at-Rest (CMEK/EKM)
      requestBody:
        required: true
        content:
          application/json:
            schema:
              type: object
              properties:
                kms_key_id: { type: string }
                key_policy: { type: object }
      responses: { "201": { description: Configured } }
```

### Domain L0.O — Webhooks & Event Delivery

```yaml
paths:
  /v1/webhooks:
    post:
      tags: [L0.O Webhooks]
      operationId: registerWebhook
      summary: Register Webhook (static HMAC vs dynamic JWKS RS256)
      requestBody:
        required: true
        content:
          application/json:
            schema:
              type: object
              required: [url, events]
              properties:
                url: { type: string }
                events: { type: array, items: { type: string } }
                secret: { type: string }
                signing_scheme:
                  type: string
                  enum: [hmac_sha256, jwks_rs256, ed25519]
                jwks_url: { type: string }
      responses:
        "201":
          description: Created
          content:
            application/json:
              schema: { $ref: "#/components/schemas/WebhookSecret" }
    get:
      tags: [L0.O Webhooks]
      operationId: listWebhooks
      responses: { "200": { description: OK } }

  /v1/webhooks/{id}:
    parameters:
      - in: path
        name: id
        required: true
        schema: { type: string }
    delete:
      tags: [L0.O Webhooks]
      operationId: deleteWebhook
      responses: { "204": { description: Deleted } }

  /v1/webhooks/{id}/rotate:
    post:
      tags: [L0.O Webhooks]
      operationId: rotateWebhookSecret
      summary: Rotate Webhook Secret
      parameters:
        - in: path
          name: id
          required: true
          schema: { type: string }
      responses:
        "200":
          description: OK
          content:
            application/json:
              schema: { $ref: "#/components/schemas/WebhookSecret" }

  /v1/webhooks/{id}/configure:
    post:
      tags: [L0.O Webhooks]
      operationId: configureDynamicWebhook
      summary: Dynamic Webhook Configuration (uris, user_metadata, revocation_behavior)
      parameters:
        - in: path
          name: id
          required: true
          schema: { type: string }
      requestBody:
        required: true
        content:
          application/json:
            schema:
              type: object
              properties:
                uris: { type: array, items: { type: string } }
                user_metadata: { type: object }
                revocation_behavior: { type: string }
      responses: { "200": { description: OK } }
```

### Domain L0.P — Developer SDKs, CLI & Specs

```yaml
paths:
  /v1/openapi:
    get:
      tags: [L0.P SDK & CLI]
      operationId: getOpenApiSpec
      summary: Published OpenAPI spec
      responses:
        "200":
          description: OpenAPI YAML/JSON
          content:
            application/json: { schema: { type: object } }
            application/yaml: { schema: { type: string } }
```

### Domain L0.Q — Routing & Gateway (control plane)

```yaml
paths:
  /v1/gateway/endpoints:
    post:
      tags: [L0.Q Routing & Gateway]
      operationId: createGatewayEndpoint
      summary: Create Gateway Endpoint (slug → target {provider, model_id, env})
      requestBody:
        required: true
        content:
          application/json:
            schema:
              type: object
              required: [slug, target]
              properties:
                slug: { type: string }
                target:
                  type: object
                  properties:
                    provider: { type: string }
                    model_id: { type: string }
                    env: { type: string }
      responses: { "201": { description: Created } }
    get:
      tags: [L0.Q Routing & Gateway]
      operationId: listGatewayEndpoints
      responses: { "200": { description: OK } }

  /v1/gateway/endpoints/{id}:
    parameters:
      - in: path
        name: id
        required: true
        schema: { type: string }
    patch:
      tags: [L0.Q Routing & Gateway]
      operationId: updateGatewayEndpoint
      summary: Re-point target
      responses: { "200": { description: OK } }
    delete:
      tags: [L0.Q Routing & Gateway]
      operationId: deleteGatewayEndpoint
      responses: { "204": { description: Deleted } }

  /v1/gateway/groups/{id}/api_keys:
    post:
      tags: [L0.Q Routing & Gateway]
      operationId: mintFederatedKey
      summary: Mint Federated Key (under group)
      parameters:
        - in: path
          name: id
          required: true
          schema: { type: string }
      responses: { "201": { description: Created } }

  /v1/gateway/groups/{id}/api_keys/register:
    post:
      tags: [L0.Q Routing & Gateway]
      operationId: registerSignedKey
      summary: Register Existing Signed Key
      parameters:
        - in: path
          name: id
          required: true
          schema: { type: string }
      responses: { "201": { description: Created } }

  /v1/gateway/model/chat/completions:
    post:
      tags: [L0.Q Routing & Gateway]
      operationId: gatewayChatCompletions
      summary: Model Gateway Passthrough (OpenAI-compatible)
      responses: { "200": { description: OK } }

  /v1/gateway/model/embeddings:
    post:
      tags: [L0.Q Routing & Gateway]
      operationId: gatewayEmbeddings
      summary: Model Gateway Embeddings Passthrough
      responses: { "200": { description: OK } }
```

### Domain L0.R — Secrets & Credentials (Vault)

```yaml
paths:
  /v1/vaults:
    post:
      tags: [L0.R Vaults & Credentials]
      operationId: createVault
      summary: Create Vault (write-only; ≤20 credentials/vault; keys immutable)
      requestBody:
        required: true
        content:
          application/json:
            schema:
              type: object
              required: [name]
              properties:
                name: { type: string }
      responses: { "201": { description: Created } }
    get:
      tags: [L0.R Vaults & Credentials]
      operationId: listVaults
      responses: { "200": { description: OK } }

  /v1/vaults/{id}:
    parameters:
      - in: path
        name: id
        required: true
        schema: { type: string }
    get:
      tags: [L0.R Vaults & Credentials]
      operationId: getVault
      responses: { "200": { description: OK } }
    delete:
      tags: [L0.R Vaults & Credentials]
      operationId: deleteVault
      responses: { "204": { description: Deleted } }

  /v1/vaults/{id}/credentials:
    post:
      tags: [L0.R Vaults & Credentials]
      operationId: createCredential
      summary: Create Credential (mcp_oauth / static_bearer / environment_variable)
      parameters:
        - in: path
          name: id
          required: true
          schema: { type: string }
      requestBody:
        required: true
        content:
          application/json:
            schema:
              type: object
              required: [type, value]
              properties:
                type:
                  type: string
                  enum: [mcp_oauth, static_bearer, environment_variable]
                value: { type: string }
      responses: { "201": { description: Created } }

  /v1/vaults/{id}/credentials/{cid}:
    parameters:
      - in: path
        name: id
        required: true
        schema: { type: string }
      - in: path
        name: cid
        required: true
        schema: { type: string }
    get:
      tags: [L0.R Vaults & Credentials]
      operationId: getCredential
      responses: { "200": { description: OK } }
    delete:
      tags: [L0.R Vaults & Credentials]
      operationId: deleteCredential
      responses: { "204": { description: Deleted } }

  /v1/vaults/{id}/credentials/{cid}/rotate:
    post:
      tags: [L0.R Vaults & Credentials]
      operationId: rotateCredential
      summary: Rotate Credential (propagates to running sessions)
      parameters:
        - in: path
          name: id
          required: true
          schema: { type: string }
        - in: path
          name: cid
          required: true
          schema: { type: string }
      responses: { "200": { description: OK } }

  /v1/vaults/{id}/credentials/{cid}/mcp_oauth_validate:
    post:
      tags: [L0.R Vaults & Credentials]
      operationId: validateMcpOAuth
      parameters:
        - in: path
          name: id
          required: true
          schema: { type: string }
        - in: path
          name: cid
          required: true
          schema: { type: string }
      responses: { "200": { description: OK } }

  /v1/connections/applications:
    post:
      tags: [L0.R Connections]
      operationId: createConnectionApplication
      summary: Create Connection Application (OAuth2 / API key / basic auth)
      requestBody:
        required: true
        content:
          application/json:
            schema:
              type: object
              properties:
                oauth2:
                  type: object
                  properties:
                    client_id: { type: string }
                    client_secret: { type: string }
                api_key: { type: string }
                basic_auth:
                  type: object
                  properties:
                    username: { type: string }
                    password: { type: string }
      responses: { "201": { description: Created } }

  /v1/connections/callback:
    get:
      tags: [L0.R Connections]
      operationId: oauthCallback
      summary: OAuth Callback
      responses: { "200": { description: OK } }
```

### Domain L0.S — Process & Filesystem (host-side RPC)

```yaml
paths:
  /v1/filesystem/readFile:
    post: { tags: [L0.S Process & Filesystem], operationId: fsReadFile, responses: { "200": { description: OK } } }
  /v1/filesystem/writeFile:
    post: { tags: [L0.S Process & Filesystem], operationId: fsWriteFile, responses: { "200": { description: OK } } }
  /v1/filesystem/createDirectory:
    post: { tags: [L0.S Process & Filesystem], operationId: fsCreateDirectory, responses: { "200": { description: OK } } }
  /v1/filesystem/getMetadata:
    post: { tags: [L0.S Process & Filesystem], operationId: fsGetMetadata, responses: { "200": { description: OK } } }
  /v1/filesystem/readDirectory:
    post: { tags: [L0.S Process & Filesystem], operationId: fsReadDirectory, responses: { "200": { description: OK } } }
  /v1/filesystem/remove:
    post: { tags: [L0.S Process & Filesystem], operationId: fsRemove, responses: { "204": { description: Removed } } }
  /v1/filesystem/copy:
    post: { tags: [L0.S Process & Filesystem], operationId: fsCopy, responses: { "200": { description: OK } } }
  /v1/filesystem/watch:
    post: { tags: [L0.S Process & Filesystem], operationId: fsWatch, responses: { "200": { description: OK } } }
  /v1/process/spawn:
    post: { tags: [L0.S Process & Filesystem], operationId: processSpawn, responses: { "200": { description: OK } } }
  /v1/process/writeStdin:
    post: { tags: [L0.S Process & Filesystem], operationId: processWriteStdin, responses: { "200": { description: OK } } }
  /v1/process/resizePty:
    post: { tags: [L0.S Process & Filesystem], operationId: processResizePty, responses: { "200": { description: OK } } }
  /v1/process/kill:
    post: { tags: [L0.S Process & Filesystem], operationId: processKill, responses: { "204": { description: Killed } } }
  /v1/command/exec:
    post: { tags: [L0.S Process & Filesystem], operationId: commandExec, responses: { "200": { description: OK } } }
  /v1/config/read:
    post: { tags: [L0.S Process & Filesystem], operationId: configRead, responses: { "200": { description: OK } } }
  /v1/config/value:
    post: { tags: [L0.S Process & Filesystem], operationId: configValue, responses: { "200": { description: OK } } }
  /v1/config/write:
    post: { tags: [L0.S Process & Filesystem], operationId: configWrite, responses: { "200": { description: OK } } }
  /v1/config/batchWrite:
    post: { tags: [L0.S Process & Filesystem], operationId: configBatchWrite, responses: { "200": { description: OK } } }
  /v1/configRequirements/read:
    post: { tags: [L0.S Process & Filesystem], operationId: configRequirementsRead, responses: { "200": { description: OK } } }
```

<details>
<summary>Continue with Layer 1 (next chunk)…</summary>

Layer 1 (Compute & Model Serving) covers Hardware & Model Catalog, Deployment CRUD, Lifecycle/Promotion, Autoscaling, Health Checks, Inference Execution (Chat/Completions/Responses/Messages/Specialized), Async Inference, Batch Inference, RL Rollout, and Observability Plumbing.
</details>

---

## Layer 1 — Compute & Model Serving

### Domain L1.A — Hardware & Model Catalog

```yaml
paths:
  /v1/models:
    get:
      tags: [L1.A Hardware & Model Catalog]
      operationId: listModels
      summary: List Models (pricing/latency/throughput/features)
      parameters:
        - in: query
          name: provider
          schema: { type: string }
        - in: query
          name: task
          schema: { type: string }
        - in: query
          name: output_modalities
          schema: { type: array, items: { type: string } }
        - in: query
          name: supported_parameters
          schema: { type: array, items: { type: string } }
        - in: query
          name: sort
          schema: { type: string, enum: [pricing, context, throughput, latency, popularity, newest] }
      responses:
        "200":
          description: OK
          content:
            application/json:
              schema:
                allOf:
                  - $ref: "#/components/schemas/ListResponse"
                  - type: object
                    properties:
                      data: { type: array, items: { $ref: "#/components/schemas/Model" } }

  /v1/models/{model_id}:
    get:
      tags: [L1.A Hardware & Model Catalog]
      operationId: getModel
      summary: Get Model (all variants)
      parameters:
        - in: path
          name: model_id
          required: true
          schema: { type: string }
      responses:
        "200":
          description: OK
          content:
            application/json:
              schema: { $ref: "#/components/schemas/Model" }
        "404": { $ref: "#/components/responses/NotFound" }

  /v0/templates:
    get:
      tags: [L1.A Hardware & Model Catalog]
      operationId: listDeployableTemplates
      summary: List Deployable Templates (model+flavor+gpu+region combos, dedicated)
      responses: { "200": { description: OK } }

  /v1/hardware:
    get:
      tags: [L1.A Hardware & Model Catalog]
      operationId: listHardwareOptions
      summary: List Hardware Options for a model
      parameters:
        - in: query
          name: model
          required: true
          schema: { type: string }
      responses: { "200": { description: OK } }

  /v1/availability-zones:
    get:
      tags: [L1.A Hardware & Model Catalog]
      operationId: listAvailabilityZones
      responses: { "200": { description: OK } }
```

> Note: `POST /v1/deployments` belongs to L1.C below; the architecture lists it there.

### Domain L1.B — Model Packaging & Artifact Management

```yaml
paths:
  /v0/models/custom/register:
    post:
      tags: [L1.B Model Packaging]
      operationId: registerCustomModel
      summary: 4-step signed upload step 1 (register → getUploadEndpoint → PUT → validateUpload)
      requestBody:
        required: true
        content:
          application/json:
            schema:
              type: object
              properties:
                weights:
                  type: array
                  items:
                    type: object
                    properties:
                      source:
                        type: object
                        properties:
                          uri: { type: string, description: "hf/s3/gs/azure/r2/bt/cw schemes" }
                          auth: { type: object }
                          allow_patterns: { type: array, items: { type: string } }
                container_image: { type: string }
                handler: { type: string, description: "handler.py / EndpointHandler / Sprocket / Truss Model" }
      responses: { "201": { description: Registered } }

  /v0/models/custom/{id}/upload-endpoint:
    get:
      tags: [L1.B Model Packaging]
      operationId: getUploadEndpoint
      responses: { "200": { description: OK } }

  /v0/models/custom/{id}/validate:
    post:
      tags: [L1.B Model Packaging]
      operationId: validateUpload
      responses: { "200": { description: OK } }
```

### Domain L1.C — Compute Provisioning & Deployment

```yaml
paths:
  /v1/deployments:
    post:
      tags: [L1.C Deployment CRUD]
      operationId: createDeployment
      summary: Create Deployment (returns id/routing_key/state)
      requestBody:
        required: true
        content:
          application/json:
            schema:
              type: object
              required: [model, hardware, region]
              properties:
                model: { type: string }
                hardware: { type: string }
                region: { type: string, description: "Immutable after creation." }
                parallelism: { type: object }
                engine_args: { type: object }
                autoscaling: { type: object }
                mode:
                  type: string
                  enum: [serverless, dedicated, ptu, dedicated_containers, gpu_clusters, pods]
      responses:
        "201":
          description: Created
          content:
            application/json:
              schema: { $ref: "#/components/schemas/Deployment" }
    get:
      tags: [L1.C Deployment CRUD]
      operationId: listDeployments
      responses:
        "200":
          description: OK
          content:
            application/json:
              schema:
                allOf:
                  - $ref: "#/components/schemas/ListResponse"
                  - type: object
                    properties:
                      data: { type: array, items: { $ref: "#/components/schemas/Deployment" } }

  /v1/deployments/{id}:
    parameters:
      - in: path
        name: id
        required: true
        schema: { type: string }
    get:
      tags: [L1.C Deployment CRUD]
      operationId: getDeployment
      responses:
        "200":
          description: OK
          content:
            application/json:
              schema: { $ref: "#/components/schemas/Deployment" }
    patch:
      tags: [L1.C Deployment CRUD]
      operationId: patchDeployment
      summary: Patch Deployment (mutable fields; region immutable)
      requestBody:
        required: true
        content:
          application/json:
            schema: { $ref: "#/components/schemas/Deployment" }
      responses:
        "200":
          description: OK
          content:
            application/json:
              schema: { $ref: "#/components/schemas/Deployment" }
    delete:
      tags: [L1.C Deployment CRUD]
      operationId: deleteDeployment
      responses: { "204": { description: Deleted } }

  /v1/deployments/{id}/{action}:
    post:
      tags: [L1.C Deployment CRUD]
      operationId: deploymentLifecycleAction
      summary: Start / Stop / Wake / Activate / Deactivate
      parameters:
        - in: path
          name: id
          required: true
          schema: { type: string }
        - in: path
          name: action
          required: true
          schema:
            type: string
            enum: [start, stop, wake, activate, deactivate, reset, restart]
      responses: { "200": { description: OK } }

  /wake:
    post:
      tags: [L1.C Deployment CRUD]
      operationId: wakeDeployment
      summary: Wake a scaled-to-zero deployment
      parameters:
        - in: query
          name: id
          required: true
          schema: { type: string }
      responses: { "200": { description: OK } }
```

### Domain L1.E — Deployment Lifecycle & Traffic Management

```yaml
paths:
  /v1/models/{id}/environments/{env}/promote:
    post:
      tags: [L1.E Lifecycle & Promotion]
      operationId: promoteToEnvironment
      parameters:
        - in: path
          name: id
          required: true
          schema: { type: string }
        - in: path
          name: env
          required: true
          schema: { type: string }
      responses: { "200": { description: OK } }

  /v1/models/{id}/environments/{env}:
    patch:
      tags: [L1.E Lifecycle & Promotion]
      operationId: patchEnvironmentConfig
      summary: Patch rolling-deploy config (max_surge / max_unavailable)
      parameters:
        - in: path
          name: id
          required: true
          schema: { type: string }
        - in: path
          name: env
          required: true
          schema: { type: string }
      requestBody:
        required: true
        content:
          application/json:
            schema:
              type: object
              properties:
                max_surge: { type: integer }
                max_unavailable: { type: integer }
      responses: { "200": { description: OK } }

  /v1/models/{id}/environments/{env}/{action}_promotion:
    post:
      tags: [L1.E Lifecycle & Promotion]
      operationId: controlPromotion
      summary: pause / resume / cancel / force_cancel / force_roll_forward promotion
      parameters:
        - in: path
          name: id
          required: true
          schema: { type: string }
        - in: path
          name: env
          required: true
          schema: { type: string }
        - in: path
          name: action
          required: true
          schema:
            type: string
            enum: [pause, resume, cancel, force_cancel, force_roll_forward]
      responses: { "200": { description: OK } }

  /v1/deployments/{id}/autoscaling_settings:
    patch:
      tags: [L1.E Autoscaling]
      operationId: patchAutoscaling
      summary: Patch Autoscaling Settings
      parameters:
        - in: path
          name: id
          required: true
          schema: { type: string }
      requestBody:
        required: true
        content:
          application/json:
            schema:
              type: object
              properties:
                min_replica: { type: integer }
                max_replica: { type: integer }
                autoscaling_window: { type: integer }
                scale_down_delay: { type: integer }
                concurrency_target: { type: integer }
                target_utilization_percentage: { type: number }
                target_in_flight_tokens: { type: integer }
                load_targets: { type: object }
                scale_up_window: { type: integer }
                scale_down_window: { type: integer }
                scale_to_zero_window: { type: integer }
                idle_timeout: { type: integer }
                scale_down_half_life_seconds: { type: integer }
      responses: { "200": { description: OK } }

  /health:
    get:
      tags: [L1.E Health Checks]
      operationId: healthCheck
      summary: Health Endpoint (200 ready / 503 loading)
      responses:
        "200": { description: Ready }
        "503": { description: Loading }
```

### Domain L1.G — Inference Request Execution

```yaml
paths:
  /v1/chat/completions:
    post:
      tags: [L1.G Inference Execution]
      operationId: chatCompletions
      summary: Chat Completions (primary or legacy surface)
      requestBody:
        required: true
        content:
          application/json:
            schema:
              type: object
              required: [model, messages]
              properties:
                model: { type: string }
                messages:
                  type: array
                  items: { $ref: "#/components/schemas/Message" }
                temperature: { type: number }
                top_p: { type: number }
                top_k: { type: integer }
                min_p: { type: number }
                max_tokens: { type: integer }
                max_completion_tokens: { type: integer }
                stop: { type: array, items: { type: string }, maxItems: 4 }
                frequency_penalty: { type: number }
                presence_penalty: { type: number }
                repetition_penalty: { type: number }
                seed: { type: integer }
                n: { type: integer }
                logprobs: { type: boolean }
                top_logprobs: { type: integer }
                logit_bias: { type: object }
                tools: { type: array, items: { $ref: "#/components/schemas/ToolDefinition" } }
                tool_choice: { $ref: "#/components/schemas/ToolChoice" }
                parallel_tool_calls: { type: boolean }
                response_format:
                  type: object
                  properties:
                    type: { type: string, enum: [json_schema, json_object, text, regex, grammar] }
                    json_schema: { type: object }
                    pattern: { type: string }
                reasoning: { $ref: "#/components/schemas/Reasoning" }
                reasoning_effort:
                  type: string
                  enum: [none, minimal, low, medium, high, xhigh, max, ultra]
                stream: { type: boolean, default: false }
                stream_options: { $ref: "#/components/schemas/StreamOptions" }
                service_tier:
                  type: string
                  enum: [auto, default, over-limit, flex, no-limit, priority, standard, standard_only]
                user: { type: string }
                store: { type: boolean }
                metadata: { type: object }
                prompt_cache_key: { type: string }
                truncation: { type: string }
                background: { type: boolean }
                deferred: { type: boolean }
                prediction:
                  type: object
                  properties:
                    type: { type: string, enum: [content] }
                    content: { type: string }
      responses:
        "200":
          description: OK (non-streaming)
          content:
            application/json:
              schema:
                type: object
                properties:
                  id: { type: string }
                  object: { type: string, example: "chat.completion" }
                  created: { type: integer }
                  model: { type: string }
                  choices:
                    type: array
                    items:
                      type: object
                      properties:
                        index: { type: integer }
                        message: { $ref: "#/components/schemas/Message" }
                        finish_reason: { $ref: "#/components/schemas/StopReason" }
                        logprobs: { type: object }
                  usage: { $ref: "#/components/schemas/Usage" }
        "200-stream":
          description: SSE stream (S.10)
          content:
            text/event-stream:
              schema: { type: string }
        "400": { $ref: "#/components/responses/BadRequest" }
        "429": { $ref: "#/components/responses/RateLimited" }

  /v1/chat/deferred-completion/{request_id}:
    get:
      tags: [L1.G Inference Execution]
      operationId: getDeferredCompletion
      summary: Poll deferred completion (200 completed / 202 pending)
      parameters:
        - in: path
          name: request_id
          required: true
          schema: { type: string }
      responses:
        "200": { description: Completed }
        "202": { description: Pending }

  /v1/completions:
    post:
      tags: [L1.G Inference Execution]
      operationId: completions
      summary: Completions (legacy)
      requestBody:
        required: true
        content:
          application/json:
            schema:
              type: object
              required: [model, prompt]
              properties:
                model: { type: string }
                prompt:
                  oneOf:
                    - type: string
                    - type: array
                      items: { type: string }
                stream: { type: boolean }
      responses: { "200": { description: OK } }

  /v1/messages:
    post:
      tags: [L1.G Inference Execution]
      operationId: messagesCreate
      summary: Messages (compat) — content[] typed blocks, thinking blocks with signatures
      requestBody:
        required: true
        content:
          application/json:
            schema:
              type: object
              required: [model, messages]
              properties:
                model: { type: string }
                messages:
                  type: array
                  items: { $ref: "#/components/schemas/Message" }
                system:
                  oneOf:
                    - type: string
                    - type: array
                      items: { $ref: "#/components/schemas/ContentBlock" }
                tools: { type: array, items: { $ref: "#/components/schemas/ToolDefinition" } }
                tool_choice: { $ref: "#/components/schemas/ToolChoice" }
                thinking:
                  type: object
                  properties:
                    type: { type: string, enum: [enabled, adaptive, disabled] }
                    budget_tokens: { type: integer }
                max_tokens: { type: integer }
                stream: { type: boolean }
                stop_sequences: { type: array, items: { type: string } }
      responses:
        "200":
          description: OK
          content:
            application/json:
              schema:
                type: object
                properties:
                  id: { type: string }
                  type: { type: string, example: "message" }
                  role: { type: string, example: "assistant" }
                  content:
                    type: array
                    items: { $ref: "#/components/schemas/ContentBlock" }
                  stop_reason: { $ref: "#/components/schemas/StopReason" }
                  usage: { $ref: "#/components/schemas/Usage" }

  /inference:
    post:
      tags: [L1.G Inference Execution]
      operationId: inferenceCompat
      summary: Inference (compat alternate route)
      responses: { "200": { description: OK } }

  /v1/embeddings:
    post:
      tags: [L1.G Inference Execution]
      operationId: createEmbeddings
      summary: Embeddings
      requestBody:
        required: true
        content:
          application/json:
            schema:
              type: object
              required: [model, input]
              properties:
                model: { type: string }
                input:
                  oneOf:
                    - type: string
                    - type: array
                      items: { type: string }
                dimensions: { type: integer, description: "MRL shortening" }
                encoding_format:
                  type: string
                  enum: [float, base64]
      responses:
        "200":
          description: OK
          content:
            application/json:
              schema:
                type: object
                properties:
                  object: { type: string, example: "list" }
                  data:
                    type: array
                    items:
                      type: object
                      properties:
                        object: { type: string, example: "embedding" }
                        index: { type: integer }
                        embedding:
                          oneOf:
                            - type: array
                              items: { type: number }
                            - type: string
                  model: { type: string }
                  usage: { $ref: "#/components/schemas/Usage" }

  /v1/rerank:
    post:
      tags: [L1.G Inference Execution]
      operationId: rerank
      summary: Rerank (variants: /rerank)
      requestBody:
        required: true
        content:
          application/json:
            schema:
              type: object
              required: [model, query, documents]
              properties:
                model: { type: string }
                query: { type: string }
                documents: { type: array, items: { type: string } }
                top_n: { type: integer }
      responses: { "200": { description: OK } }

  /v1/images/generations:
    post:
      tags: [L1.G Inference Execution]
      operationId: imageGenerations
      summary: Image Generations (proxy; canonical L3.B)
      requestBody:
        required: true
        content:
          application/json:
            schema: { $ref: "#/components/schemas/ImageGenerationRequest" }
      responses:
        "200":
          description: OK
          content:
            application/json:
              schema: { $ref: "#/components/schemas/ImageGenerationResponse" }

  /v1/audio/transcriptions:
    post:
      tags: [L1.G Inference Execution]
      operationId: audioTranscription
      summary: Audio Transcription
      requestBody:
        required: true
        content:
          multipart/form-data:
            schema:
              type: object
              required: [model]
              properties:
                file: { type: string, format: binary }
                audio_url: { type: string }
                model: { type: string }
      responses: { "200": { description: OK } }

  /v1/audio/translations:
    post:
      tags: [L1.G Inference Execution]
      operationId: audioTranslation
      summary: Audio Translation
      requestBody:
        required: true
        content:
          multipart/form-data:
            schema:
              type: object
              properties:
                file: { type: string, format: binary }
                audio_url: { type: string }
                model: { type: string }
      responses: { "200": { description: OK } }

  /v1/audio/speech:
    post:
      tags: [L1.G Inference Execution]
      operationId: audioSpeech
      summary: Audio Speech (TTS)
      requestBody:
        required: true
        content:
          application/json:
            schema:
              type: object
              required: [model, input, voice]
              properties:
                model: { type: string }
                input: { type: string }
                voice: { type: string }
      responses:
        "200":
          description: Audio
          content:
            audio/mpeg: { schema: { type: string, format: binary } }
```

### Domain L1.G — Async Inference Lifecycle

```yaml
paths:
  /v1/async_predict:
    post:
      tags: [L1.G Async Inference]
      operationId: submitAsyncPredict
      summary: Submit Async Predict (→ request_id)
      requestBody:
        required: true
        content:
          application/json:
            schema:
              type: object
              properties:
                predict_args: { type: object }
                priority:
                  type: integer
                  enum: [0, 1, 2]
                retry_config:
                  type: object
                  properties:
                    max_attempts: { type: integer }
                    initial_delay: { type: integer }
                    max_delay: { type: integer }
                webhook: { type: string }
      responses:
        "202":
          description: Accepted
          content:
            application/json:
              schema:
                type: object
                properties:
                  request_id: { type: string }

  /v1/async_request/{id}:
    parameters:
      - in: path
        name: id
        required: true
        schema: { type: string }
    get:
      tags: [L1.G Async Inference]
      operationId: getAsyncRequestStatus
      summary: QUEUED/IN_PROGRESS/SUCCEEDED/FAILED/EXPIRED/CANCELED/WEBHOOK_FAILED
      responses:
        "200":
          description: OK
          content:
            application/json:
              schema:
                allOf:
                  - $ref: "#/components/schemas/AsyncJob"
                  - type: object
                    properties:
                      status:
                        type: string
                        enum: [QUEUED, IN_PROGRESS, SUCCEEDED, FAILED, EXPIRED, CANCELED, WEBHOOK_FAILED]
    delete:
      tags: [L1.G Async Inference]
      operationId: cancelAsyncRequest
      responses: { "204": { description: Cancelled } }

  /v1/async_queue_status:
    get:
      tags: [L1.G Async Inference]
      operationId: asyncQueueStatus
      responses: { "200": { description: OK } }

  /run:
    post:
      tags: [L1.G Async Inference]
      operationId: runAsync
      summary: Generic async execution primitive (run)
      responses: { "202": { description: Accepted } }

  /status/{id}:
    get:
      tags: [L1.G Async Inference]
      operationId: getStatus
      responses: { "200": { description: OK } }

  /cancel/{id}:
    post:
      tags: [L1.G Async Inference]
      operationId: cancelRun
      responses: { "204": { description: Cancelled } }

  /hot_load/v1/models/hot_load:
    post:
      tags: [L1.G RL Rollout]
      operationId: hotLoad
      summary: RL rollout hot-load / MoE router replay
      parameters:
        - in: header
          name: x-multi-turn-session-id
          schema: { type: string }
        - in: header
          name: x-session-affinity
          schema: { type: string }
      requestBody:
        required: true
        content:
          application/json:
            schema:
              type: object
              properties:
                reset_prompt_cache:
                  type: string
                  enum: [all, new_session, none]
                include_routing_matrix: { type: boolean }
                logprobs: { type: boolean }
                echo: { type: boolean }
      responses: { "200": { description: OK } }
```

### Domain L1.G — Batch Inference Lifecycle

```yaml
paths:
  /v1/batches:
    post:
      tags: [L1.G Batch Inference]
      operationId: submitBatch
      summary: Submit Batch (multi-endpoint; targets /v1/responses, /v1/chat/completions, /v1/embeddings, /v1/completions, /v1/moderations, /v1/images/*, /v1/videos)
      requestBody:
        required: true
        content:
          application/json:
            schema:
              type: object
              required: [input_file_id, endpoint, completion_window]
              properties:
                input_file_id: { type: string }
                endpoint: { type: string }
                completion_window:
                  type: string
                  enum: ["12h", "24h", "48h", "72h"]
                metadata: { type: object }
      responses:
        "200":
          description: OK
          content:
            application/json:
              schema: { $ref: "#/components/schemas/Batch" }
    get:
      tags: [L1.G Batch Inference]
      operationId: listBatches
      parameters:
        - $ref: "#/components/parameters/LimitParam"
        - $ref: "#/components/parameters/CursorParam"
      responses:
        "200":
          description: OK
          content:
            application/json:
              schema:
                allOf:
                  - $ref: "#/components/schemas/ListResponse"
                  - type: object
                    properties:
                      data: { type: array, items: { $ref: "#/components/schemas/Batch" } }

  /v1/batches/{id}:
    parameters:
      - in: path
        name: id
        required: true
        schema: { type: string }
    get:
      tags: [L1.G Batch Inference]
      operationId: getBatch
      responses:
        "200":
          description: OK
          content:
            application/json:
              schema: { $ref: "#/components/schemas/Batch" }

  /v1/batches/{id}/cancel:
    post:
      tags: [L1.G Batch Inference]
      operationId: cancelBatch
      parameters:
        - in: path
          name: id
          required: true
          schema: { type: string }
      responses: { "200": { description: OK } }

  /batchInferenceJobs:
    post:
      tags: [L1.G Batch Inference]
      operationId: submitBatchInferenceJob
      summary: BatchInferenceJob (inline or file; 1 M requests file / 10k inline; output_file + error_file)
      requestBody:
        required: true
        content:
          application/json:
            schema:
              type: object
              properties:
                file: { type: string }
                model: { type: string }
                params: { type: object }
                window:
                  type: string
                  enum: ["12h", "24h", "48h", "72h"]
                output_file: { type: string }
                error_file: { type: string }
      responses: { "202": { description: Accepted } }

  /batchInferenceJobs/{id}:
    get:
      tags: [L1.G Batch Inference]
      operationId: getBatchInferenceJob
      parameters:
        - in: path
          name: id
          required: true
          schema: { type: string }
      responses: { "200": { description: OK } }

  /v1/messages/batches:
    post:
      tags: [L1.G Batch Inference]
      operationId: submitMessagesBatch
      summary: Messages Batch (50% discount; 100k requests or 256 MB; expire 24 h; not ZDR)
      requestBody:
        required: true
        content:
          application/json:
            schema:
              type: object
              properties:
                file_id: { type: string }
                model: { type: string }
      responses: { "202": { description: Accepted } }

  /v1/batch/jobs:
    post:
      tags: [L1.G Batch Inference]
      operationId: submitBatchJob
      summary: Batch Job (embeddings/chat/fim/moderations/chat-moderations/ocr/classifications/conversations/audio-transcriptions)
      requestBody:
        required: true
        content:
          application/json:
            schema:
              type: object
              properties:
                inline: { type: array, items: { type: object } }
                file: { type: string }
                output_file: { type: string }
                error_file: { type: string }
      responses: { "202": { description: Accepted } }
```

### Domain L1.I — Observability Plumbing (runtime metrics/logs)

```yaml
paths:
  /v1/endpoints/{id}/open-metrics:
    get:
      tags: [L1.I Observability Plumbing]
      operationId: getOpenMetrics
      summary: OpenMetrics API (Prometheus text format)
      parameters:
        - in: path
          name: id
          required: true
          schema: { type: string }
      responses:
        "200":
          description: Prometheus text
          content:
            text/plain; version=0.0.4: { schema: { type: string } }

  /v1/endpoints/{id}/logs:
    get:
      tags: [L1.I Observability Plumbing]
      operationId: getRuntimeLogs
      summary: Runtime Logs (real-time, filterable; 30-day retention)
      parameters:
        - in: path
          name: id
          required: true
          schema: { type: string }
        - in: query
          name: timestamp
          schema: { type: string }
        - in: query
          name: level
          schema: { type: string }
        - in: query
          name: content
          schema: { type: string }
        - in: query
          name: replica
          schema: { type: string }
      responses: { "200": { description: OK } }
```

<details>
<summary>Continue with Layer 2 (next chunk)…</summary>

Layer 2 (Model Inference & Intelligence APIs) covers Model Catalog (L2.A), Modern Generation (L2.B — Responses/Interactions/Messages), Legacy Generation (L2.B — Chat Completions/Generate Content), Streaming (L2.F — WebSocket Responses), Context Management (L2.G — Compact), Embeddings & Rerank (L2.J), Batch (L2.K), and Grounding & Citations (L2.L).
</details>

---

## Layer 2 — Model Inference & Intelligence APIs

### Domain L2.A — Model Catalog & Selection

```yaml
paths:
  /api/v1/model/{author}/{slug}:
    get:
      tags: [L2.A Model Catalog]
      operationId: resolveModelAlias
      summary: Resolve alias/slug to canonical model id
      parameters:
        - in: path
          name: author
          required: true
          schema: { type: string }
        - in: path
          name: slug
          required: true
          schema: { type: string }
      responses:
        "200":
          description: OK
          content:
            application/json:
              schema: { $ref: "#/components/schemas/Model" }

  /v1/model_policy:
    get:
      tags: [L2.A Model Catalog]
      operationId: getModelPolicy
      summary: Read model-selection governance policies
      responses: { "200": { description: OK } }
    post:
      tags: [L2.A Model Catalog]
      operationId: applyModelPolicy
      summary: Apply model-selection governance policies
      requestBody:
        required: true
        content:
          application/json:
            schema:
              type: object
              properties:
                policies: { type: array, items: { type: object } }
      responses: { "200": { description: OK } }
```

### Domain L2.B — Generation API Surfaces (Modern)

```yaml
paths:
  /v1/responses:
    post:
      tags: [L2.B Generation (Modern)]
      operationId: createResponse
      summary: Responses Create (stateful; typed output[] Items, reasoning items, encrypted content)
      description: |
        Modern stateful generation surface. Supports `previous_response_id`,
        built-in tools, MCP, background mode, encrypted reasoning replay
        (`include: ["reasoning.encrypted_content"]`). Incompatible with
        structured outputs + document citations.
      requestBody:
        required: true
        content:
          application/json:
            schema:
              type: object
              required: [model]
              properties:
                model: { type: string }
                input:
                  oneOf:
                    - type: string
                    - type: array
                      items: { $ref: "#/components/schemas/ContentBlock" }
                instructions: { type: string }
                previous_response_id: { type: string }
                tools: { type: array, items: { $ref: "#/components/schemas/ToolDefinition" } }
                tool_choice: { $ref: "#/components/schemas/ToolChoice" }
                store: { type: boolean }
                metadata: { type: object }
                background: { type: boolean }
                reasoning: { $ref: "#/components/schemas/Reasoning" }
                stream: { type: boolean }
                text:
                  type: object
                  properties:
                    format:
                      type: object
                      properties:
                        type: { type: string, enum: [json_schema, json_object, text, regex, grammar] }
                        schema: { $ref: "#/components/schemas/JsonSchema" }
                        strict: { type: boolean }
                include:
                  type: array
                  items: { type: string }
                  example: ["reasoning.encrypted_content"]
                service_tier:
                  type: string
                  enum: [auto, default, over-limit, flex, no-limit, priority, standard, standard_only]
                prompt_cache_key: { type: string }
                truncation: { type: string }
                context_management:
                  type: object
                  properties:
                    compact_threshold: { type: integer }
                    edits: { type: array, items: { type: object } }
      responses:
        "200":
          description: OK
          content:
            application/json:
              schema:
                type: object
                properties:
                  id: { type: string }
                  object: { type: string, example: "response" }
                  created_at: { type: integer }
                  model: { type: string }
                  status:
                    type: string
                    enum: [completed, incomplete, failed]
                  incomplete_details:
                    type: object
                    properties:
                      reason:
                        type: string
                        enum: [max_output_tokens, content_filter]
                  output:
                    type: array
                    items: { $ref: "#/components/schemas/ContentBlock" }
                  usage: { $ref: "#/components/schemas/Usage" }
        "200-stream":
          description: SSE stream
          content:
            text/event-stream:
              schema: { type: string }

  /v1/responses/{id}:
    parameters:
      - in: path
        name: id
        required: true
        schema: { type: string }
    get:
      tags: [L2.B Generation (Modern)]
      operationId: getResponse
      summary: Stored Responses CRUD — retrieve
      responses:
        "200": { description: OK }
    delete:
      tags: [L2.B Generation (Modern)]
      operationId: deleteResponse
      summary: Stored Responses CRUD — delete
      responses: { "204": { description: Deleted } }

  /v1/conversations:
    post:
      tags: [L2.B Generation (Modern)]
      operationId: createConversation
      summary: Persistent Conversations API (owner, conversation ID, append-only new turn)
      requestBody:
        required: true
        content:
          application/json:
            schema:
              type: object
              properties:
                model: { type: string }
                system_instruction: { type: string }
                previous_interaction_id: { type: string }
                tools: { type: array, items: { $ref: "#/components/schemas/ToolDefinition" } }
                input:
                  type: array
                  items: { $ref: "#/components/schemas/ContentBlock" }
      responses:
        "200":
          description: OK
          content:
            application/json:
              schema:
                type: object
                properties:
                  interaction_id: { type: string }
                  steps:
                    type: array
                    items: { type: object }
                  usage: { $ref: "#/components/schemas/Usage" }

  /v1/interactions/{id}:
    parameters:
      - in: path
        name: id
        required: true
        schema: { type: string }
    get:
      tags: [L2.B Generation (Modern)]
      operationId: getInteraction
      summary: Background Interaction Poll / Stream (stream=true, last_event_id=)
      parameters:
        - in: query
          name: stream
          schema: { type: boolean }
        - in: query
          name: last_event_id
          schema: { type: string }
      responses:
        "200": { description: OK }
        "200-stream": { description: SSE stream }

  /v1/interactions/{id}/cancel:
    post:
      tags: [L2.B Generation (Modern)]
      operationId: cancelInteraction
      parameters:
        - in: path
          name: id
          required: true
          schema: { type: string }
      responses: { "204": { description: Cancelled } }
```

### Domain L2.B — Generation (Legacy / compat)

```yaml
paths:
  # /v1/chat/completions and /v1/completions already defined in L1.G.
  # /v1/messages already defined in L1.G.
  /v1/models/{model}:generateContent:
    post:
      tags: [L2.B Generation (Legacy)]
      operationId: generateContent
      summary: Generate Content (legacy) — candidates[].content.parts[]
      parameters:
        - in: path
          name: model
          required: true
          schema: { type: string }
      requestBody:
        required: true
        content:
          application/json:
            schema:
              type: object
              properties:
                contents:
                  type: array
                  items: { type: object }
      responses: { "200": { description: OK } }

  /:analyze-text:
    post:
      tags: [L2.B Generation (Legacy)]
      operationId: analyzeText
      summary: Analyze Text (typed `kind` discriminator; sync)
      requestBody:
        required: true
        content:
          application/json:
            schema:
              type: object
              properties:
                documents: { type: array, items: { type: object } }
                tasks: { type: array, items: { type: object } }
      responses: { "200": { description: OK } }
```

### Domain L2.F — Streaming (WebSocket mode)

```yaml
paths:
  /v1/responses:
    get:
      tags: [L2.F Streaming]
      operationId: wsResponsesUpgrade
      summary: WebSocket Responses (persistent connection; ~40% faster for 20+ tool calls)
      description: |
        WebSocket at `wss://api.../v1/responses`. 60-min max duration, one
        in-flight response at a time. Send only incremental input items per
        turn plus `previous_response_id`. Reconnect patterns: (1) continue
        with previous_response_id if persisted; (2) start fresh with
        previous_response_id: null + full input; (3) use compacted window
        from /responses/compact.
      responses: { "101": { description: Switching Protocols } }

  /v1/responses/compact:
    post:
      tags: [L2.G Context Management]
      operationId: responsesCompact
      summary: Responses Compact (stateless, ZDR-friendly; opaque compaction item)
      description: |
        Returns opaque compaction item with `encrypted_content` to pass
        verbatim into next request. At most one per call; conversation must
        already fit. Used for WebSocket reconnect pattern (3).
      requestBody:
        required: true
        content:
          application/json:
            schema:
              type: object
              properties:
                model: { type: string }
                input:
                  type: array
                  items: { $ref: "#/components/schemas/ContentBlock" }
                previous_response_id: { type: string }
      responses:
        "200":
          description: OK
          content:
            application/json:
              schema:
                type: object
                properties:
                  compacted_input:
                    type: array
                    items: { $ref: "#/components/schemas/ContentBlock" }
                  usage: { $ref: "#/components/schemas/Usage" }
```

### Domain L2.J — Embeddings & Rerank (primitive)

```yaml
paths:
  /api/v1/embeddings:
    post:
      tags: [L2.J Embeddings & Rerank]
      operationId: unifiedEmbeddings
      summary: Embeddings (unified multi-provider)
      requestBody:
        required: true
        content:
          application/json:
            schema:
              type: object
              required: [model, inputs]
              properties:
                model: { type: string }
                inputs:
                  oneOf:
                    - type: string
                    - type: array
                      items: { type: string }
                input_type:
                  type: string
                  enum: [document, query]
                dimensions: { type: integer }
                encoding_format:
                  type: string
                  enum: [float, base64]
      responses: { "200": { description: OK } }
```

### Domain L2.K — Batch Processing

> Batch endpoints are defined under L1.G (Batch Inference Lifecycle). L2.K adds the deferred single-request async primitive:

```yaml
paths:
  # Deferred Completion (Chat) — already covered via /v1/chat/completions deferred:true
  # and /v1/chat/deferred-completion/{request_id} in L1.G.
```

### Domain L2.L — Grounding, Citations & RAG (primitive)

> Citations are emitted as `ContentBlock.citations` (see `#/components/schemas/Citation`). The built-in web-search tool is catalogued in L4.E. Document-citation block flag `citations:{enabled:true}` is a parameter on `document` content blocks (see ContentBlock schema).

<details>
<summary>Continue with Layer 3 (next chunk)…</summary>

Layer 3 (AI Modality Products) covers Text & Conversation (L3.A — including Classical NLP and Custom NLP Training), Images & Video (L3.B — generation, editing, layout, understanding, postprocessing, style, video), Voice (L3.C — assets, preprocessing, STT, translation/dubbing, TTS, transformation, sound/music, voice agents), and Documents (L3.D — ingestion, understanding, chunking, indexing/graph, query time, generation, transformation, custom processors, schema, MCP tools).
</details>

---

## Layer 3 — AI Modality Products

### Domain L3.A — Text & Conversation

> Single-Turn / Multiple Candidates / Multi-Turn / Stateful Conversation / Encrypted Reasoning Replay / Mid-conversation System Messages / Stored Responses CRUD are all served by the L2.B generation endpoints (`/v1/responses`, `/v1/conversations`, `/v1/chat/completions`, `/v1/messages`). L3.A documents the **classical NLP** and **custom NLP training** facets.

### Domain L3.A — Classical NLP Analysis

```yaml
paths:
  /:analyze-text-job:
    post:
      tags: [L3.A Classical NLP]
      operationId: analyzeTextJob
      summary: NLP Analysis Job (async; dispatches by task `kind`)
      requestBody:
        required: true
        content:
          application/json:
            schema:
              type: object
              properties:
                documents: { type: array, items: { type: object } }
                conversations: { type: array, items: { type: object } }
                tasks:
                  type: array
                  items:
                    type: object
                    properties:
                      kind:
                        type: string
                        enum:
                          - LanguageDetection
                          - EntityRecognition
                          - CustomEntityRecognition
                          - PiiEntityRecognition
                          - ConversationalPIITask
                          - Healthcare
                          - SentimentAnalysis
                          - KeyPhraseExtraction
                          - EntityLinking
                          - ExtractiveSummarization
                          - AbstractiveSummarization
                          - ConversationalSummarizationTask
                          - CustomSingleLabelClassification
                          - CustomMultiLabelClassification
                          - Conversation
                stringIndexType:
                  type: string
                  enum: [Utf16CodeUnit, TextElement_V8]
                piiCategories: { type: array, items: { type: string } }
                confidenceScoreThreshold: { type: number }
                redactionSource: { type: string }
                includeAudioRedaction: { type: boolean }
      responses: { "202": { $ref: "#/components/responses/AsyncAccepted" } }

  /:query-knowledgebases/projects/{project}/productionSettings/deployments/{deployment}:
    post:
      tags: [L3.A Classical NLP]
      operationId: queryKnowledgebase
      summary: Custom Question Answering — query-knowledgebases
      parameters:
        - in: path
          name: project
          required: true
          schema: { type: string }
        - in: path
          name: deployment
          required: true
          schema: { type: string }
      requestBody:
        required: true
        content:
          application/json:
            schema:
              type: object
              properties:
                query: { type: string }
                metadata_filters: { type: object }
                dialog: { type: object }
                prompts: { type: array, items: { type: string } }
      responses: { "200": { description: OK } }

  /:query-text:
    post:
      tags: [L3.A Classical NLP]
      operationId: queryText
      summary: Custom Question Answering — query-text (prebuilt)
      requestBody:
        required: true
        content:
          application/json:
            schema:
              type: object
              properties:
                query: { type: string }
      responses: { "200": { description: OK } }

  /:analyze-conversations/projects/{project}/deployments/{deployment}:
    post:
      tags: [L3.A Classical NLP]
      operationId: orchestrationWorkflow
      summary: Orchestration Workflow (projectKind:Orchestration; routes utterances to CLU/CQA)
      parameters:
        - in: path
          name: project
          required: true
          schema: { type: string }
        - in: path
          name: deployment
          required: true
          schema: { type: string }
      requestBody:
        required: true
        content:
          application/json:
            schema:
              type: object
              properties:
                utterance: { type: string }
      responses: { "200": { description: OK } }
```

### Domain L3.A — Custom NLP Training

```yaml
paths:
  /language/authoring/analyze-text/projects/{name}/:import:
    post:
      tags: [L3.A Custom NLP Training]
      operationId: importNlpProject
      summary: Import Project (schema + labeled data via Blob Storage)
      parameters:
        - in: path
          name: name
          required: true
          schema: { type: string }
      responses: { "202": { description: Accepted } }

  /language/authoring/analyze-text/projects/{name}:train:
    post:
      tags: [L3.A Custom NLP Training]
      operationId: trainModel
      parameters:
        - in: path
          name: name
          required: true
          schema: { type: string }
      requestBody:
        required: true
        content:
          application/json:
            schema:
              type: object
              properties:
                modelLabel: { type: string }
                trainingConfigVersion: { type: string }
                evaluationOptions: { type: object }
      responses: { "202": { description: Accepted } }

  /language/authoring/analyze-text/projects/{name}/deployments/{deploymentName}:
    put:
      tags: [L3.A Custom NLP Training]
      operationId: deployModel
      summary: Deploy trained model; swap deployments test↔prod
      parameters:
        - in: path
          name: name
          required: true
          schema: { type: string }
        - in: path
          name: deploymentName
          required: true
          schema: { type: string }
      requestBody:
        required: true
        content:
          application/json:
            schema:
              type: object
              properties:
                trainedModelLabel: { type: string }
      responses: { "200": { description: OK } }
```

### Domain L3.B — Image Generation

```yaml
paths:
  /v1/images/generations:
    post:
      tags: [L3.B Image Generation]
      operationId: imageGeneration
      summary: Image Generation (text→raster image; conversational multi-turn variant)
      description: |
        Canonical image generation endpoint. Supports conversational/multi-turn
        (`previous_response_id`, `image_generation_call` ids, `action:auto|generate|edit`),
        streaming/partial images (`partial_images` 0-3), transparent output,
        determinism, prompt enhancement, negative prompt, reference-image
        tagging, grounding, moderation & safety, watermarking. See S.32
        Image Generation Request for full param union.
      requestBody:
        required: true
        content:
          application/json:
            schema: { $ref: "#/components/schemas/ImageGenerationRequest" }
      responses:
        "200":
          description: OK
          content:
            application/json:
              schema: { $ref: "#/components/schemas/ImageGenerationResponse" }
        "200-stream":
          description: SSE partial images stream
          content:
            text/event-stream:
              schema: { type: string }

  /v1/images/generations/vector:
    post:
      tags: [L3.B Image Generation]
      operationId: vectorImageGeneration
      summary: Vector Image Generation (text→SVG; rejects raster)
      requestBody:
        required: true
        content:
          application/json:
            schema:
              type: object
              properties:
                prompt: { type: string }
                model: { type: string }
                vector_styles: { type: array, items: { type: object } }
      responses:
        "200":
          description: OK
          content:
            application/json:
              schema:
                type: object
                properties:
                  svg: { type: string }

  /v1/images/generate-transparent:
    post:
      tags: [L3.B Image Generation]
      operationId: generateTransparentImage
      summary: Transparent-Background Image Generation (die-cut stickers/logos)
      requestBody:
        required: true
        content:
          application/json:
            schema:
              type: object
              properties:
                prompt: { type: string }
                upscale_factor:
                  type: string
                  enum: [X1, X2, X4]
                aspect_ratio: { type: string }
      responses: { "200": { description: OK } }
```

### Domain L3.B — Image Editing

```yaml
paths:
  /v1/images/edits:
    post:
      tags: [L3.B Image Editing]
      operationId: imageEdit
      summary: Image Edit (prompt-based editing + compositing; multi-image ≤N refs)
      description: |
        Supports all edit operation kinds: prompt-based editing, multi-image
        compositing, inpainting (mask alpha/B/W/grayscale/polygon/COCO RLE),
        outpainting/border extension, background removal/replace/generate,
        object removal/erase, deblur, virtual try-on (VTO), restyle/relight,
        remix/variate/explore, context-aware editing (Kontext-style),
        typography/text-rendering, legacy variations.
      requestBody:
        required: true
        content:
          application/json:
            schema: { $ref: "#/components/schemas/ImageEditRequest" }
      responses:
        "200":
          description: OK
          content:
            application/json:
              schema: { $ref: "#/components/schemas/ImageGenerationResponse" }

  /v1/images/imageToImage:
    post:
      tags: [L3.B Image Editing]
      operationId: imageToImage
      responses: { "200": { description: OK } }

  /v1/edit:
    post:
      tags: [L3.B Image Editing]
      operationId: editAlias
      responses: { "200": { description: OK } }

  /v1/images/remix:
    post:
      tags: [L3.B Image Editing]
      operationId: imageRemix
      summary: Remix (1-6 refs + `<img>N</img>`; image_weight)
      responses: { "200": { description: OK } }
```

### Domain L3.B — Layout Composition

```yaml
paths:
  /v1/images/create_layout:
    post:
      tags: [L3.B Layout Composition]
      operationId: createLayout
      summary: Layout-Aware Create (typed Layout regions; returns image + echoed layout)
      requestBody:
        required: true
        content:
          application/json:
            schema:
              type: object
              properties:
                text: { type: string }
                refs: { type: array, items: { type: object } }
                layout:
                  type: object
                  description: "Typed Layout (S.32)."
      responses: { "200": { description: OK } }

  /v1/images/edit_layout:
    post:
      tags: [L3.B Layout Composition]
      operationId: editLayout
      summary: Layout-Aware Edit (LayoutCommand ops: add|shift|remove|place|keep|change)
      responses: { "200": { description: OK } }

  /v1/images/render:
    post:
      tags: [L3.B Layout Composition]
      operationId: renderLayout
      summary: Layout Render (Layout → image; echoes layout)
      responses: { "200": { description: OK } }

  /v1/images/describe:
    post:
      tags: [L3.B Layout Composition]
      operationId: imageToLayout
      summary: Image-to-Layout Extraction (reverse-engineer Layout/V4JsonPrompt)
      requestBody:
        required: true
        content:
          application/json:
            schema:
              type: object
              properties:
                image: { $ref: "#/components/schemas/ImageInput" }
                include_bbox: { type: boolean }
      responses: { "200": { description: OK } }
```

### Domain L3.B — Image Understanding (Vision / Analysis)

```yaml
paths:
  /v1/images:annotate:
    post:
      tags: [L3.B Image Understanding]
      operationId: annotateImage
      summary: Image Annotation Batch (labels+faces+OCR+objects+safe-search simultaneously)
      requestBody:
        required: true
        content:
          application/json:
            schema:
              type: object
              properties:
                requests:
                  type: array
                  items:
                    type: object
                    properties:
                      image: { type: object }
                      features:
                        type: array
                        items:
                          type: object
                          properties:
                            type:
                              type: string
                              enum:
                                - LABEL_DETECTION
                                - OBJECT_LOCALIZATION
                                - FACE_DETECTION
                                - TEXT_DETECTION
                                - DOCUMENT_TEXT_DETECTION
                                - LANDMARK_DETECTION
                                - LOGO_DETECTION
                                - WEB_DETECTION
                                - SAFE_SEARCH_DETECTION
                                - IMAGE_PROPERTIES
                                - CROP_HINTS
                                - PRODUCT_SEARCH
                      languageHints: { type: array, items: { type: string } }
      responses: { "200": { description: OK } }

  /v1/images:asyncBatchAnnotate:
    post:
      tags: [L3.B Image Understanding]
      operationId: asyncBatchAnnotate
      summary: Async batch annotation (up to 2000 files to Cloud Storage)
      responses: { "202": { $ref: "#/components/responses/AsyncAccepted" } }

  /v1/imageanalysis:analyze:
    post:
      tags: [L3.B Image Understanding]
      operationId: imageAnalysis
      responses: { "200": { description: OK } }

  /v1/images/layerize-text:
    post:
      tags: [L3.B Image Understanding]
      operationId: layerizeText
      summary: Text Layer Extraction (base_image_url + text_blocks[] role/text/geometry/font/color/alignment)
      requestBody:
        required: true
        content:
          application/json:
            schema:
              type: object
              properties:
                image: { $ref: "#/components/schemas/ImageInput" }
                prompt: { type: string }
      responses: { "200": { description: OK } }
```

### Domain L3.B — Image Postprocessing

```yaml
paths:
  /v1/images/crispUpscale:
    post:
      tags: [L3.B Image Postprocessing]
      operationId: crispUpscale
      summary: Image Upscale — Crisp (interpolation sharpening preserving content)
      requestBody:
        required: true
        content:
          application/json:
            schema:
              type: object
              properties:
                file: { type: string, format: binary }
                image_url: { type: string }
      responses: { "200": { description: OK } }

  /v1/images/creativeUpscale:
    post:
      tags: [L3.B Image Postprocessing]
      operationId: creativeUpscale
      summary: Image Upscale — Creative (regenerates finer details/faces)
      requestBody:
        required: true
        content:
          application/json:
            schema:
              type: object
              properties:
                file: { type: string, format: binary }
                image_url: { type: string }
                prompt: { type: string }
                resemblance: { type: number }
                detail: { type: number }
                magic_prompt_option: { type: string }
      responses: { "200": { description: OK } }

  /v1/image/effect:
    get:
      tags: [L3.B Image Postprocessing]
      operationId: listEffects
      summary: List available visual filters
      responses: { "200": { description: OK } }
    post:
      tags: [L3.B Image Postprocessing]
      operationId: applyEffect
      summary: Effect Apply (named visual filter; effect_parameters filterId:uniformId:value)
      requestBody:
        required: true
        content:
          application/json:
            schema:
              type: object
              properties:
                effect_name: { type: string }
                effect_parameters: { type: object }
                source: { type: string }
      responses: { "200": { description: OK } }

  /v1/images/vectorize:
    post:
      tags: [L3.B Image Postprocessing]
      operationId: imageVectorization
      summary: Image Vectorization (raster → SVG; deterministic, no model)
      requestBody:
        required: true
        content:
          application/json:
            schema:
              type: object
              properties:
                file: { type: string, format: binary }
                image_url: { type: string }
      responses: { "200": { description: OK } }
```

### Domain L3.B — Style & Asset Management

```yaml
paths:
  /v1/styles:
    post:
      tags: [L3.B Style & Assets]
      operationId: createCustomStyle
      summary: Custom Style Creation (≤5 reference images; returns reusable UUID style)
      requestBody:
        required: true
        content:
          application/json:
            schema:
              type: object
              properties:
                reference_images:
                  type: array
                  maxItems: 5
                  items: { $ref: "#/components/schemas/ImageInput" }
      responses: { "201": { description: Created } }

  /v1/videos/characters:
    post:
      tags: [L3.B Style & Assets]
      operationId: createCharacterReference
      summary: Character Reference Asset (Video) — reusable non-human character asset
      requestBody:
        required: true
        content:
          application/json:
            schema:
              type: object
              properties:
                name: { type: string }
                mp4: { type: string, description: "Short MP4" }
      responses: { "201": { description: Created } }

  /v1/prompts/enhance:
    post:
      tags: [L3.B Style & Assets]
      operationId: enhancePrompt
      summary: Prompt Enhancement (≤2000 chars)
      requestBody:
        required: true
        content:
          application/json:
            schema:
              type: object
              properties:
                prompt: { type: string, maxLength: 2000 }
      responses: { "200": { description: OK } }

  /v1/images/magic-prompt:
    post:
      tags: [L3.B Style & Assets]
      operationId: magicPrompt
      summary: Magic Prompt
      responses: { "200": { description: OK } }
```

### Domain L3.B — Video Generation

```yaml
paths:
  /v1/videos:
    post:
      tags: [L3.B Video Generation]
      operationId: generateVideo
      summary: Video Generation (text→video)
      requestBody:
        required: true
        content:
          application/json:
            schema: { $ref: "#/components/schemas/VideoGenerationRequest" }
      responses:
        "202":
          description: Accepted
          content:
            application/json:
              schema: { $ref: "#/components/schemas/VideoResponse" }

  /v1/videos/generations:
    post:
      tags: [L3.B Video Generation]
      operationId: generateVideoAlt
      responses: { "202": { description: Accepted } }

  /models/{model}:generateVideos:
    post:
      tags: [L3.B Video Generation]
      operationId: generateVideosPredictLongRunning
      summary: predictLongRunning video generation
      parameters:
        - in: path
          name: model
          required: true
          schema: { type: string }
      responses: { "202": { description: Accepted } }

  /v1/videos/{id}:
    get:
      tags: [L3.B Video Generation]
      operationId: getVideoOperation
      summary: Poll video async operation
      parameters:
        - in: path
          name: id
          required: true
          schema: { type: string }
      responses:
        "200":
          description: OK
          content:
            application/json:
              schema: { $ref: "#/components/schemas/VideoResponse" }
```

### Domain L3.B — Video Editing, Extension & Interpolation

```yaml
paths:
  /v1/videos/edits:
    post:
      tags: [L3.B Video Editing]
      operationId: editVideo
      summary: Video Editing (one focused change per edit)
      requestBody:
        required: true
        content:
          application/json:
            schema:
              type: object
              properties:
                video:
                  type: object
                  properties:
                    id: { type: string }
                    url: { type: string }
                    file_id: { type: string }
                prompt: { type: string }
      responses: { "202": { description: Accepted } }

  /v1/videos/extensions:
    post:
      tags: [L3.B Video Editing]
      operationId: extendVideo
      summary: Video Extension (source video, prompt, seconds/duration extension portion)
      requestBody:
        required: true
        content:
          application/json:
            schema:
              type: object
              properties:
                source_video: { type: object }
                prompt: { type: string }
                seconds: { type: number }
                duration: { type: number }
      responses: { "202": { description: Accepted } }
```

### Domain L3.C — Voice Asset Management

```yaml
paths:
  /v1/voices:
    get:
      tags: [L3.C Voice Assets]
      operationId: browseVoices
      summary: Voice Library Browse & Search (filter by language/gender/country; expand[]=preview_file_url)
      parameters:
        - in: query
          name: language
          schema: { type: string }
        - in: query
          name: gender
          schema: { type: string }
        - in: query
          name: country
          schema: { type: string }
        - in: query
          name: q
          schema: { type: string }
        - in: query
          name: expand
          schema: { type: array, items: { type: string } }
      responses: { "200": { description: OK } }

  /v1/voices/find-similar:
    post:
      tags: [L3.C Voice Assets]
      operationId: findSimilarVoices
      responses: { "200": { description: OK } }

  /v1/voices/localize:
    post:
      tags: [L3.C Voice Assets]
      operationId: localizeVoice
      summary: Voice Localization (adapt voice to new language)
      responses: { "200": { description: OK } }

  /v1/voices/{voice_id}:
    parameters:
      - in: path
        name: voice_id
        required: true
        schema: { type: string }
    get:
      tags: [L3.C Voice Assets]
      operationId: getVoice
      responses: { "200": { description: OK } }
    patch:
      tags: [L3.C Voice Assets]
      operationId: updateVoiceMetadata
      summary: Voice Metadata CRUD (name/description/gender/settings, share-to-library)
      responses: { "200": { description: OK } }

  /v1/voices/clone:
    post:
      tags: [L3.C Voice Assets]
      operationId: instantVoiceClone
      summary: Instant Voice Cloning (IVC) — short audio sample (+ consent recording)
      responses: { "201": { description: Created } }

  /v1/voices/fine-tunes/create:
    post:
      tags: [L3.C Voice Assets]
      operationId: professionalVoiceClone
      summary: Professional Voice Cloning (PVC) — multi-step with speaker separation
      responses: { "202": { description: Accepted } }

  /v1/text-to-voice/design:
    post:
      tags: [L3.C Voice Assets]
      operationId: designVoiceFromText
      summary: Voice Design from Text (20-1000 char description; returns 3 preview voices)
      responses: { "200": { description: OK } }

  /v1/text-to-voice/remix:
    post:
      tags: [L3.C Voice Assets]
      operationId: remixVoice
      summary: Voice Remixing (existing voice + NL attribute transforms)
      responses: { "200": { description: OK } }

  /v1/pronunciation-dictionaries:
    post:
      tags: [L3.C Voice Assets]
      operationId: createPronunciationDictionary
      summary: Pronunciation Dictionary CRUD (phonetic rules, versioning, attachment)
      responses: { "201": { description: Created } }

  /v1/music/fine-tunes:
    post:
      tags: [L3.C Voice Assets]
      operationId: musicFineTune
      summary: Music Datasets & Fine-Tuning (non-copyrighted tracks 5-10 min)
      responses: { "202": { description: Accepted } }
```

### Domain L3.C — Audio Preprocessing

```yaml
paths:
  /v1/audio-isolation:
    post:
      tags: [L3.C Audio Preprocessing]
      operationId: voiceIsolator
      summary: Voice Isolator (max 500 MB / 1 hour)
      responses: { "200": { description: OK } }

  /v1/audio-isolation/stream:
    post:
      tags: [L3.C Audio Preprocessing]
      operationId: voiceIsolatorStream
      summary: Streamed noise removal
      responses: { "200": { description: OK } }
```

### Domain L3.C — Speech-to-Text / Audio Understanding

```yaml
paths:
  /v1/speech-to-text:
    post:
      tags: [L3.C Speech-to-Text]
      operationId: batchTranscription
      summary: Batch Transcription (file-based with full option suite)
      description: |
        Full STT option suite: language/diarize/keyterms/prompt/smart_format/
        punctuate/paragraphs/entity_detection/redact/timestamp_granularities/
        tag_audio_events/summarize/sentiment/topics/intents/webhook.
      requestBody:
        required: true
        content:
          application/json:
            schema:
              type: object
              properties:
                audio_url: { type: string }
                file: { type: string, format: binary }
                language: { type: string }
                diarize: { type: boolean }
                keyterms: { type: array, items: { type: string } }
                smart_format: { type: boolean }
                punctuate: { type: boolean }
                paragraphs: { type: boolean }
                entity_detection: { type: boolean }
                redact: { type: object }
                timestamp_granularities: { type: array, items: { type: string } }
                tag_audio_events: { type: boolean }
                summarize: { type: boolean }
                sentiment: { type: boolean }
                topics: { type: boolean }
                intents: { type: boolean }
                webhook: { type: string }
      responses: { "200": { description: OK } }

  /v1/audio/transcriptions:
    post:
      tags: [L3.C Speech-to-Text]
      operationId: audioTranscriptions
      responses: { "200": { description: OK } }

  /v1/listen:
    post:
      tags: [L3.C Speech-to-Text]
      operationId: listenPreRecorded
      summary: Pre-Recorded STT
      responses: { "200": { description: OK } }

  /v1/stt:
    post:
      tags: [L3.C Speech-to-Text]
      operationId: stt
      responses: { "200": { description: OK } }

  /post/speech/asr:
    post:
      tags: [L3.C Speech-to-Text]
      operationId: legacyAsr
      responses: { "200": { description: OK } }

  /v1/stt/stream:
    get:
      tags: [L3.C Speech-to-Text]
      operationId: realtimeStreamingTranscription
      summary: Real-Time Streaming Transcription (WebSocket; is_final/speech_final, VAD, KeepAlive, manual commit)
      responses: { "101": { description: Switching Protocols } }

  /v1/forced-alignment:
    post:
      tags: [L3.C Speech-to-Text]
      operationId: forcedAlignment
      summary: Forced Alignment (audio file + plain string transcript; 29 languages)
      responses: { "200": { description: OK } }

  /v1/read:
    post:
      tags: [L3.C Speech-to-Text]
      operationId: readApi
      summary: Read API (STT intelligence features applied to text)
      responses: { "200": { description: OK } }
```

### Domain L3.C — Translation & Dubbing

```yaml
paths:
  /v1/audio/translations:
    post:
      tags: [L3.C Translation & Dubbing]
      operationId: fileBasedAudioTranslation
      summary: File-Based Audio Translation (output always English text; 25 MB)
      responses: { "200": { description: OK } }

  /v1/stt-translate:
    post:
      tags: [L3.C Translation & Dubbing]
      operationId: sttTranslate
      summary: Translating Transcription (transcript in target language)
      responses: { "200": { description: OK } }

  /v1/dubbing:
    post:
      tags: [L3.C Translation & Dubbing]
      operationId: batchDubbing
      summary: Batch Dubbing (90+ languages; preserve emotion/timing/tone/speaker)
      requestBody:
        required: true
        content:
          application/json:
            schema:
              type: object
              properties:
                source: { type: string }
                cloning_strength: { type: number }
                preserve_original_voices: { type: boolean }
                keep_background_audio: { type: boolean }
                target_language: { type: string }
      responses: { "202": { description: Accepted } }

  /v1/gpt-realtime-translate:
    get:
      tags: [L3.C Translation & Dubbing]
      operationId: liveSpeechToSpeech
      summary: Live Speech-to-Speech Translation (WebSocket; one session per target language)
      responses: { "101": { description: Switching Protocols } }
```

### Domain L3.C — Text-to-Speech Generation

```yaml
paths:
  /v1/text-to-speech:
    post:
      tags: [L3.C Text-to-Speech]
      operationId: textToSpeech
      summary: Text-to-Speech (single-speaker)
      requestBody:
        required: true
        content:
          application/json:
            schema:
              type: object
              required: [text, model, voice]
              properties:
                text: { type: string }
                model: { type: string }
                voice: { type: string }
                voice_id: { type: string }
                language: { type: string }
                output_format: { type: string }
                voice_settings:
                  type: object
                  properties:
                    stability: { type: number }
                    similarity: { type: number }
                    style: { type: number }
                    speaker_boost: { type: boolean }
                    speed: { type: number }
                seed: { type: integer, minimum: 0, maximum: 4294967295 }
                stream: { type: boolean }
                previous_request_ids: { type: array, items: { type: string } }
                next_request_ids: { type: array, items: { type: string } }
                use_pvc_as_ivc: { type: boolean }
      responses:
        "200":
          description: Audio
          content:
            audio/mpeg: { schema: { type: string, format: binary } }

  /v1/text-to-speech/multi-speaker:
    post:
      tags: [L3.C Text-to-Speech]
      operationId: multiSpeakerTts
      summary: Multi-Speaker TTS / Dialogue (inputs[] with text + voice_id per turn)
      responses: { "200": { description: OK } }

  /v1/text-to-speech/stream:
    get:
      tags: [L3.C Text-to-Speech]
      operationId: ttsWebSocketControl
      summary: WebSocket TTS Control (flush:true / cancel:true / context_id)
      responses: { "101": { description: Switching Protocols } }
```

### Domain L3.C — Voice Transformation

```yaml
paths:
  /v1/speech-to-speech/{voice_id}:
    post:
      tags: [L3.C Voice Transformation]
      operationId: voiceChanger
      summary: Voice Changer (STS, no translation)
      parameters:
        - in: path
          name: voice_id
          required: true
          schema: { type: string }
      requestBody:
        required: true
        content:
          application/json:
            schema:
              type: object
              properties:
                audio: { type: string, format: binary }
                remove_background_noise: { type: boolean }
                clip: { type: boolean }
                output_format: { type: string }
      responses: { "200": { description: OK } }

  /v1/speech-to-speech/{voice_id}/stream:
    post:
      tags: [L3.C Voice Transformation]
      operationId: voiceChangerStream
      responses: { "200": { description: OK } }

  /v1/infill/bytes:
    post:
      tags: [L3.C Voice Transformation]
      operationId: audioInfill
      summary: Audio Infill / Bridging (left_audio, transcript, right_audio)
      responses: { "200": { description: OK } }

  /v1/music/separate-stems:
    post:
      tags: [L3.C Voice Transformation]
      operationId: separateStems
      summary: Stem Separation (instrument/vocal stems)
      responses: { "200": { description: OK } }
```

### Domain L3.C — Sound Effects & Music Generation

```yaml
paths:
  /v1/text-to-sound-effect:
    post:
      tags: [L3.C Sound & Music]
      operationId: generateSoundEffect
      summary: Sound Effects Generation (duration 0.1-30s; prompt_influence; loop)
      requestBody:
        required: true
        content:
          application/json:
            schema:
              type: object
              properties:
                text: { type: string }
                duration_seconds:
                  type: number
                  minimum: 0.1
                  maximum: 30
                prompt_influence: { type: number }
                loop: { type: boolean }
      responses: { "200": { description: OK } }

  /v1/music/compose:
    post:
      tags: [L3.C Sound & Music]
      operationId: composeMusic
      summary: Music Composition (text→music)
      responses: { "200": { description: OK } }

  /v1/music/compose-detailed:
    post:
      tags: [L3.C Sound & Music]
      operationId: composeMusicDetailed
      responses: { "200": { description: OK } }

  /v1/music/create-composition-plan:
    post:
      tags: [L3.C Sound & Music]
      operationId: createCompositionPlan
      responses: { "200": { description: OK } }

  /v1/music/video-to-music:
    post:
      tags: [L3.C Sound & Music]
      operationId: videoToMusic
      responses: { "200": { description: OK } }
```

### Domain L3.C — Conversational Voice Agent Orchestration

```yaml
paths:
  /v1/agent/converse:
    get:
      tags: [L3.C Voice Agents]
      operationId: voiceAgentSession
      summary: Conversational Voice Agent Session (WebSocket; unified realtime session config)
      description: |
        End-to-end / BYO-LLM / chained-pipeline voice agent. Unified realtime
        session configuration object (S.30): model, voice, instructions,
        modalities, audio formats, turn_detection, tools, thinking.
      responses: { "101": { description: Switching Protocols } }

  /v1/speech-engine:
    get:
      tags: [L3.C Voice Agents]
      operationId: speechEngine
      responses: { "101": { description: Switching Protocols } }

  /v1/realtime:
    get:
      tags: [L3.C Voice Agents]
      operationId: realtimeSession
      responses: { "101": { description: Switching Protocols } }
    post:
      tags: [L3.C Voice Agents]
      operationId: realtimePost
      responses: { "200": { description: OK } }

  /v1/realtime/translations:
    get:
      tags: [L3.C Voice Agents]
      operationId: realtimeTranslationsSession
      responses: { "101": { description: Switching Protocols } }

  /v1/agent/configs:
    post:
      tags: [L3.C Voice Agents]
      operationId: createAgentConfig
      summary: Agent Configuration CRUD (voice, LLM, tools, telephony)
      responses: { "201": { description: Created } }
  /v1/agent/configs/{id}:
    get:
      tags: [L3.C Voice Agents]
      operationId: getAgentConfig
      parameters:
        - in: path
          name: id
          required: true
          schema: { type: string }
      responses: { "200": { description: OK } }

  /v1/agents:
    post:
      tags: [L3.C Voice Agents]
      operationId: createAgent
      summary: Persisted agent definitions (voice platform)
      responses: { "201": { description: Created } }

  /v1/agents/calls/create-outbound:
    post:
      tags: [L3.C Voice Agents]
      operationId: createOutboundCall
      summary: Outbound Call (phone number, agent config)
      responses: { "201": { description: Created } }

  /v1/agents/call-batches/create-call-batch:
    post:
      tags: [L3.C Voice Agents]
      operationId: createCallBatch
      summary: Call Batch (telephony outbound)
      responses: { "201": { description: Created } }

  /v1/agents/documents:
    post:
      tags: [L3.C Voice Agents]
      operationId: uploadAgentDocuments
      summary: Agent Documents / Knowledge Base Upload (up to 100 bulk)
      responses: { "201": { description: Created } }

  /v1/agents/webhooks:
    post:
      tags: [L3.C Voice Agents]
      operationId: createAgentWebhook
      summary: Agent Webhooks CRUD (call-event webhooks)
      responses: { "201": { description: Created } }

  /v1/realtime/calls:
    post:
      tags: [L3.C Voice Agents]
      operationId: realtimeCall
      summary: Realtime Call (issue ephemeral tokens for browser/mobile)
      responses: { "201": { description: Created } }

  /v1/realtime/client_secrets:
    post:
      tags: [L3.C Voice Agents]
      operationId: realtimeClientSecrets
      summary: Realtime Ephemeral Client Secret
      responses: { "201": { description: Created } }

  /v1/realtime/translations/calls:
    post:
      tags: [L3.C Voice Agents]
      operationId: realtimeTranslationsCalls
      responses: { "201": { description: Created } }

  /v1/realtime/translations/client_secrets:
    post:
      tags: [L3.C Voice Agents]
      operationId: realtimeTranslationsClientSecrets
      responses: { "201": { description: Created } }
```

### Domain L3.D — Document Ingestion & Storage

```yaml
paths:
  /v1/workspaces:
    post:
      tags: [L3.D Document Ingestion]
      operationId: createDocumentWorkspace
      summary: Workspace / Container CRUD (type, residency, embedding model, chunking, expiration, access_mode)
      requestBody:
        required: true
        content:
          application/json:
            schema:
              type: object
              properties:
                type: { type: string }
                residency: { type: string }
                embedding_model: { type: string }
                chunking_config: { type: object }
                expiration: { type: object }
                access_mode: { type: string, enum: [private, public] }
      responses: { "201": { description: Created } }
    get:
      tags: [L3.D Document Ingestion]
      operationId: listDocumentWorkspaces
      responses: { "200": { description: OK } }

  /v1/workspaces/{id}:
    parameters:
      - in: path
        name: id
        required: true
        schema: { type: string }
    get:
      tags: [L3.D Document Ingestion]
      operationId: getDocumentWorkspace
      responses: { "200": { description: OK } }
    put:
      tags: [L3.D Document Ingestion]
      operationId: updateDocumentWorkspace
      responses: { "200": { description: OK } }
    delete:
      tags: [L3.D Document Ingestion]
      operationId: deleteDocumentWorkspace
      responses: { "204": { description: Deleted } }

  /v1/workspaces/{id}/documents:
    get:
      tags: [L3.D Document Ingestion]
      operationId: listWorkspaceDocuments
      parameters:
        - in: path
          name: id
          required: true
          schema: { type: string }
      responses: { "200": { description: OK } }

  /v1/workspaces/{id}/documents/{doc}:
    parameters:
      - in: path
        name: id
        required: true
        schema: { type: string }
      - in: path
        name: doc
        required: true
        schema: { type: string }
    get:
      tags: [L3.D Document Ingestion]
      operationId: getWorkspaceDocument
      responses: { "200": { description: OK } }
    delete:
      tags: [L3.D Document Ingestion]
      operationId: deleteWorkspaceDocument
      responses: { "204": { description: Deleted } }
```

### Domain L3.D — Document Understanding (parse + extract)

```yaml
paths:
  /v1/convert:
    post:
      tags: [L3.D Document Understanding]
      operationId: convertDocument
      summary: Document Convert / Parse (returns 202 + request_id)
      description: |
        Modes: fast/balanced/accurate. Output: md/html/json/chunks/doctags/
        doclang/docx/pdf/png. OCR (do_ocr/force_ocr/ocr_lang/ocr_preset),
        table_mode/table_format, image extraction, bboxes, confidence scores,
        enrichments (code/formula/picture_classification/picture_description),
        extras (track_changes/chart_understanding/infographic), save_checkpoint,
        webhook_url, eval_rubric_id.
      requestBody:
        required: true
        content:
          application/json:
            schema:
              type: object
              properties:
                file: { type: string }
                file_url: { type: string }
                base64_string: { type: string }
                mode:
                  type: string
                  enum: [fast, balanced, accurate]
                output_format:
                  type: string
                  enum: [md, html, json, chunks, doctags, doclang, docx, pdf, png]
                do_ocr: { type: boolean }
                force_ocr: { type: boolean }
                ocr_lang: { type: string }
                ocr_preset: { type: string }
                table_mode: { type: string }
                table_format: { type: string }
                include_blocks: { type: boolean }
                add_block_ids: { type: boolean }
                confidence_scores_granularity: { type: string }
                bbox_annotation_format: { type: string }
                paginate: { type: boolean }
                page_range: { type: string }
                max_pages: { type: integer }
                word_bboxes: { type: boolean }
                table_cell_bboxes: { type: boolean }
                token_efficient_markdown: { type: boolean }
                keep_pageheader_in_output: { type: boolean }
                keep_pagefooter_in_output: { type: boolean }
                extract_header: { type: boolean }
                extract_footer: { type: boolean }
                extract_links: { type: boolean }
                media_resolution: { type: string }
                extras: { type: array, items: { type: string } }
                enrichments: { type: array, items: { type: string } }
                save_checkpoint: { type: boolean }
                skip_cache: { type: boolean }
                processing_location: { type: string }
                webhook_url: { type: string }
                eval_rubric_id: { type: string }
      responses: { "202": { $ref: "#/components/responses/AsyncAccepted" } }

  /v1/convert/{request_id}:
    get:
      tags: [L3.D Document Understanding]
      operationId: pollConvert
      summary: Poll convert operation
      parameters:
        - in: path
          name: request_id
          required: true
          schema: { type: string }
      responses: { "200": { description: OK } }

  /v1/extract:
    post:
      tags: [L3.D Document Understanding]
      operationId: extractData
      summary: Data Extraction (JSON-schema-driven LLM extraction with citations + per-field verification + confidence)
      description: |
        Modes: turbo|fast|balanced. Per-field verification PASS/FAIL_UNRESOLVABLE/
        FAIL_FIX/FAIL_CITATIONS/ITEMS_MISSING. Per-field confidence 1-5 + reasoning.
        Supports checkpoint reuse.
      requestBody:
        required: true
        content:
          application/json:
            schema:
              type: object
              properties:
                file: { type: string }
                page_schema: { type: object }
                schema_id: { type: string }
                schema_version: { type: string }
                extraction_mode:
                  type: string
                  enum: [turbo, fast, balanced]
                output_format: { type: string }
                page_range: { type: string }
                save_checkpoint: { type: boolean }
                skip_cache: { type: boolean }
                webhook_url: { type: string }
                docClass: { type: string }
                jsonOptions: { type: object }
      responses: { "202": { $ref: "#/components/responses/AsyncAccepted" } }

  /v1/extract/{request_id}:
    get:
      tags: [L3.D Document Understanding]
      operationId: pollExtract
      parameters:
        - in: path
          name: request_id
          required: true
          schema: { type: string }
      responses: { "200": { description: OK } }

  /v1/annotate:
    post:
      tags: [L3.D Document Understanding]
      operationId: annotateDocument
      summary: BBox / Document Annotation (per-image / document-level)
      requestBody:
        required: true
        content:
          application/json:
            schema:
              type: object
              properties:
                bbox_annotation_format: { type: string }
                document_annotation_format: { type: string }
                include_image_base64: { type: boolean }
      responses: { "200": { description: OK } }

  /v1/gen-schemas:
    post:
      tags: [L3.D Document Understanding]
      operationId: generateSchemas
      summary: Schema Auto-Generation (returns simple/moderate/complex candidate schemas)
      requestBody:
        required: true
        content:
          application/json:
            schema:
              type: object
              properties:
                checkpoint_id: { type: string }
      responses: { "202": { description: Accepted } }

  /v1/gen-schemas/{request_id}:
    get:
      tags: [L3.D Document Understanding]
      operationId: pollGenSchemas
      parameters:
        - in: path
          name: request_id
          required: true
          schema: { type: string }
      responses: { "200": { description: OK } }

  /v1/segment:
    post:
      tags: [L3.D Document Understanding]
      operationId: segmentDocument
      summary: Document Segmentation (segmentation_schema; document_boundary strategy; page-structure header-based)
      requestBody:
        required: true
        content:
          application/json:
            schema:
              type: object
              properties:
                checkpoint_id: { type: string }
                segmentation_schema: { type: object }
                segmentation_strategy: { type: string }
      responses: { "202": { description: Accepted } }

  /v1/fill:
    post:
      tags: [L3.D Document Understanding]
      operationId: fillForm
      summary: Form Filling (AcroForm + visual + image field detection; PDF/PNG)
      requestBody:
        required: true
        content:
          application/json:
            schema:
              type: object
              properties:
                file: { type: string }
                field_data: { type: object }
                context: { type: string }
                page_range: { type: string }
                output_format:
                  type: string
                  enum: [pdf, png]
                confidence_threshold: { type: number }
      responses: { "200": { description: OK } }

  /v1/validate:
    post:
      tags: [L3.D Document Understanding]
      operationId: validateKvp
      summary: KVP Validation (Python-expression-based with retries; ValidatorResult Pass/Fail)
      responses: { "200": { description: OK } }

  /v1/track-changes:
    post:
      tags: [L3.D Document Understanding]
      operationId: extractTrackChanges
      summary: Track Changes Extraction (DOCX redlines + comments)
      requestBody:
        required: true
        content:
          application/json:
            schema:
              type: object
              properties:
                file: { type: string }
                output_format:
                  type: string
                  enum: [md, html, chunks]
                paginate: { type: boolean }
                page_range: { type: string }
                webhook_url: { type: string }
      responses: { "202": { description: Accepted } }

  /v1/thumbnails/{lookup_key}:
    get:
      tags: [L3.D Document Understanding]
      operationId: getThumbnail
      summary: Thumbnail Generation (thumb_width, track_changes, page_range)
      parameters:
        - in: path
          name: lookup_key
          required: true
          schema: { type: string }
      responses: { "200": { description: OK } }

  /v1/content-types:
    get:
      tags: [L3.D Document Understanding]
      operationId: listContentTypes
      summary: Classification / Facets — list content types
      responses: { "200": { description: OK } }
    post:
      tags: [L3.D Document Understanding]
      operationId: contentTypesOp
      summary: adopt | define_content_type | undefine_content_type | define_attribute | undefine_attribute
      responses: { "200": { description: OK } }

  /v1/content-types/templates:
    get:
      tags: [L3.D Document Understanding]
      operationId: listContentTypeTemplates
      responses: { "200": { description: OK } }

  /v1/tags:
    get:
      tags: [L3.D Document Understanding]
      operationId: listTags
      responses: { "200": { description: OK } }
    post:
      tags: [L3.D Document Understanding]
      operationId: createTag
      responses: { "201": { description: Created } }

  /v1/files/{id}/facets:
    parameters:
      - in: path
        name: id
        required: true
        schema: { type: string }
    post:
      tags: [L3.D Document Understanding]
      operationId: fileFacetsOp
      summary: classify | unclassify | set_value | clear_value
      responses: { "200": { description: OK } }
    get:
      tags: [L3.D Document Understanding]
      operationId: getFileFacets
      responses: { "200": { description: OK } }
```

### Domain L3.D — Chunking & Enrichment

```yaml
paths:
  /v1/chunk:
    post:
      tags: [L3.D Chunking & Enrichment]
      operationId: chunkDocument
      summary: Chunking Operation (static/token-count/character/separator/markdown/hierarchical/hybrid/line/word/structure-pure)
      requestBody:
        required: true
        content:
          application/json:
            schema:
              type: object
              properties:
                text: { type: string }
                max_chunk_size_tokens:
                  type: integer
                  minimum: 100
                  maximum: 4096
                chunk_overlap_tokens: { type: integer }
                chunk_size: { type: integer }
                tokenizer: { type: string }
                mode:
                  type: string
                  enum: [token, character, separator, markdown, hierarchical, hybrid, line, word, structure_pure, automatic]
                merge_list_items: { type: boolean }
                repeat_table_header: { type: boolean }
                keep_separator: { type: boolean }
                strip_whitespace: { type: boolean }
      responses: { "200": { description: OK } }

  /v1/chunk/enrich:
    post:
      tags: [L3.D Chunking & Enrichment]
      operationId: enrichChunks
      summary: Chunk Enrichment / Contextualization (propagate_summary_to_chunks, with_metadata, with_file_context)
      responses: { "200": { description: OK } }
```

### Domain L3.D — Embedding, Indexing & Graph

```yaml
paths:
  /v1/knowledge-graph/build:
    post:
      tags: [L3.D Indexing & Graph]
      operationId: buildKnowledgeGraph
      summary: Knowledge Graph Build (Pydantic-driven pipeline; output graph_id + stats)
      requestBody:
        required: true
        content:
          application/json:
            schema:
              type: object
              properties:
                source:
                  type: string
                  description: "file_id | checkpoint_id"
                template: { type: string, description: "Pydantic class" }
                processing_mode:
                  type: string
                  enum: [one-to-one, many-to-one]
                extraction_contract:
                  type: string
                  enum: [auto, direct, dense]
                dense_config: { type: object }
                backend:
                  type: string
                  enum: [llm, vlm]
                inference:
                  type: string
                  enum: [local, remote]
                use_chunking: { type: boolean }
                chunk_max_tokens: { type: integer }
                provenance:
                  type: string
                  enum: [off, standard, detailed]
                gleaning_enabled: { type: boolean }
                parallel_workers: { type: integer }
                export_format: { type: string }
      responses: { "202": { description: Accepted } }

  /v1/knowledge-graph/{id}:
    get:
      tags: [L3.D Indexing & Graph]
      operationId: getKnowledgeGraph
      parameters:
        - in: path
          name: id
          required: true
          schema: { type: string }
      responses: { "200": { description: OK } }

  /v1/knowledge-graph/{id}/export:
    get:
      tags: [L3.D Indexing & Graph]
      operationId: exportKnowledgeGraph
      summary: Export (format=CSV/Cypher/JSON/HTML/Docling)
      parameters:
        - in: path
          name: id
          required: true
          schema: { type: string }
        - in: query
          name: format
          schema: { type: string }
      responses: { "200": { description: OK } }

  /v1/knowledge-graph/{id}/visualize:
    get:
      tags: [L3.D Indexing & Graph]
      operationId: visualizeKnowledgeGraph
      parameters:
        - in: path
          name: id
          required: true
          schema: { type: string }
      responses: { "200": { description: OK } }

  /v1/visualize/embeddings:
    post:
      tags: [L3.D Indexing & Graph]
      operationId: visualizeEmbeddings
      summary: t-SNE 2D visualization
      responses: { "200": { description: OK } }

  /v1/resolve:
    post:
      tags: [L3.D Indexing & Graph]
      operationId: resolveEntities
      summary: Entity Resolution (blocking + pairwise LLM + union-find clustering)
      requestBody:
        required: true
        content:
          application/json:
            schema:
              type: object
              properties:
                data: { type: array, items: { type: object } }
                comparison_prompt: { type: string }
                resolution_prompt: { type: string }
                blocking_keys: { type: array, items: { type: string } }
                blocking_threshold: { type: number }
                blocking_target_recall: { type: number, default: 0.95 }
                embedding_model: { type: string }
                cascade: { type: object }
      responses: { "200": { description: OK } }

  /v1/equijoin:
    post:
      tags: [L3.D Indexing & Graph]
      operationId: equijoin
      summary: Equijoin (fuzzy join; LLM-evaluated semantic join of two datasets)
      responses: { "200": { description: OK } }

  /v1/cluster:
    post:
      tags: [L3.D Indexing & Graph]
      operationId: clusterData
      summary: Clustering (hierarchical agglomerative / KMeans / Louvain / value sampling)
      responses: { "200": { description: OK } }
```

### Domain L3.D — Query Time (read path)

```yaml
paths:
  /v1/search:
    post:
      tags: [L3.D Query Time]
      operationId: search
      summary: Search (unified: semantic | keyword | hybrid | agentic | grep | list)
      requestBody:
        required: true
        content:
          application/json:
            schema:
              type: object
              required: [query]
              properties:
                search_type:
                  type: string
                  enum: [semantic, keyword, hybrid, agentic, grep, list]
                query: { type: string }
                top_k: { type: integer }
                limit: { type: integer }
                hybrid_search:
                  type: object
                  properties:
                    embedding_weight: { type: number }
                    text_weight: { type: number }
                rerank: { type: boolean }
                rewrite_query: { type: boolean }
                agentic:
                  type: object
                  properties:
                    max_rounds: { type: integer }
                    queries_per_round: { type: integer }
                    instructions: { type: string }
                    strict_top_k: { type: boolean }
                    score_threshold: { type: number }
                relevance_scoring: { type: string }
                filters: { type: object }
                content_type: { type: array, items: { type: string } }
                attribute: { type: array, items: { type: string } }
                tag_id: { type: array, items: { type: string } }
                file_ids: { type: array, items: { type: string } }
                target_vector: { type: string }
                distance: { type: number }
                certainty: { type: number }
                auto_limit: { type: boolean }
                move_to: { type: object }
                move_away: { type: object }
                selection: { type: object }
                boost: { type: number }
                group_by: { type: array, items: { type: string } }
                return_properties: { type: array, items: { type: string } }
                return_references: { type: array, items: { type: string } }
                return_metadata: { type: array, items: { type: string } }
      responses: { "200": { description: OK } }

  /v1/grep:
    post:
      tags: [L3.D Query Time]
      operationId: grep
      summary: Grep (regex RE2 over literal chunk text)
      requestBody:
        required: true
        content:
          application/json:
            schema:
              type: object
              properties:
                store_identifiers: { type: array, items: { type: string } }
                pattern: { type: string }
                content_groups: { type: array, items: { type: string } }
                case_sensitive: { type: boolean }
                file_ids: { type: array, items: { type: string } }
                filters: { type: object }
      responses: { "200": { description: OK } }

  /v1/list-chunks:
    post:
      tags: [L3.D Query Time]
      operationId: listChunks
      summary: List Chunks (metadata-only retrieval; no embeddings)
      responses: { "200": { description: OK } }

  /v1/metadata-facets:
    post:
      tags: [L3.D Query Time]
      operationId: metadataFacets
      summary: Metadata Facets (aggregate chunk counts grouped by metadata)
      responses: { "200": { description: OK } }

  /v1/query-agent/ask:
    post:
      tags: [L3.D Query Time]
      operationId: queryAgentAsk
      summary: Query Agent Ask (LLM translates NL → database operations across collections)
      requestBody:
        required: true
        content:
          application/json:
            schema:
              type: object
              properties:
                query: { type: string }
                messages: { type: array, items: { type: object } }
                collections:
                  type: array
                  items:
                    type: object
                    properties:
                      name: { type: string }
                      target_vector: { type: string }
                      view_properties: { type: array, items: { type: string } }
                      tenant: { type: string }
                      additional_filters: { type: object }
                result_evaluation: { type: string }
                timeout: { type: integer }
                limit: { type: integer }
                filtering: { type: string, enum: [recall, precision] }
                diversity_weight: { type: number }
                include_progress: { type: boolean }
                include_final_state: { type: boolean }
      responses: { "200": { description: OK } }

  /v1/query-agent/search:
    post:
      tags: [L3.D Query Time]
      operationId: queryAgentSearch
      responses: { "200": { description: OK } }

  /v1/query-agent/ask-stream:
    post:
      tags: [L3.D Query Time]
      operationId: queryAgentAskStream
      responses: { "200-stream": { description: SSE stream } }

  /v1/aggregate:
    post:
      tags: [L3.D Query Time]
      operationId: aggregate
      summary: Aggregate queries + grouped search
      requestBody:
        required: true
        content:
          application/json:
            schema:
              type: object
              properties:
                workspace_id: { type: string }
                query: { type: string }
                total_count: { type: boolean }
                return_metrics:
                  type: object
                  properties:
                    count: { type: boolean }
                    sum: { type: array, items: { type: string } }
                    max: { type: array, items: { type: string } }
                    min: { type: array, items: { type: string } }
                    mean: { type: array, items: { type: string } }
                    median: { type: array, items: { type: string } }
                    mode: { type: array, items: { type: string } }
                    top_occurrences: { type: array, items: { type: string } }
                    percentageTrue: { type: array, items: { type: string } }
                    percentageFalse: { type: array, items: { type: string } }
                    reference_count: { type: array, items: { type: string } }
                group_by: { type: array, items: { type: string } }
                filters: { type: object }
                distance: { type: number }
                object_limit: { type: integer }
      responses: { "200": { description: OK } }

  /v1/rank:
    post:
      tags: [L3.D Query Time]
      operationId: rankData
      summary: Rank (full sorting by latent attribute)
      requestBody:
        required: true
        content:
          application/json:
            schema:
              type: object
              properties:
                data: { type: array, items: { type: object } }
                prompt: { type: string }
                input_keys: { type: array, items: { type: string } }
                direction: { type: string, enum: [asc, desc] }
                initial_ordering_method: { type: string, enum: [likert, embedding] }
                call_budget: { type: integer }
                k: { type: integer }
      responses: { "200": { description: OK } }
```

### Domain L3.D — Generation & Output (RAG)

```yaml
paths:
  /v1/ask:
    post:
      tags: [L3.D Generation & Output]
      operationId: ragAsk
      summary: RAG Ask (retrieve + generate; with citations)
      requestBody:
        required: true
        content:
          application/json:
            schema:
              type: object
              properties:
                query: { type: string }
                model: { type: string }
                stream: { type: boolean }
                instructions: { type: string }
                cite: { type: boolean }
                multimodal: { type: boolean }
                single_prompt: { type: boolean }
                grouped_task: { type: boolean }
                grouped_properties: { type: array, items: { type: string } }
                generative_provider: { type: string }
                qa_options: { type: object }
      responses: { "200": { description: OK } }

  /v1/generate:
    post:
      tags: [L3.D Generation & Output]
      operationId: ragGenerate
      summary: RAG Generate
      responses: { "200": { description: OK } }
```

### Domain L3.D — Document Transformation & Round-trip

```yaml
paths:
  /v1/create-document:
    post:
      tags: [L3.D Document Transformation]
      operationId: createDocumentFromMarkdown
      summary: DOCX Generation from Markdown (track changes tags <ins>/<~~>/<comment>)
      responses: { "200": { description: OK } }
```

### Domain L3.D — Custom Processors

```yaml
paths:
  /v1/custom-processors:
    post:
      tags: [L3.D Custom Processors]
      operationId: createCustomProcessor
      summary: Custom Processor CRUD (AI-generated lifecycle)
      responses: { "201": { description: Created } }
    get:
      tags: [L3.D Custom Processors]
      operationId: listCustomProcessors
      responses: { "200": { description: OK } }

  /v1/custom-processors/{id}:
    get:
      tags: [L3.D Custom Processors]
      operationId: getCustomProcessor
      parameters:
        - in: path
          name: id
          required: true
          schema: { type: string }
      responses: { "200": { description: OK } }

  /v1/custom-processors/{id}/iterate:
    post:
      tags: [L3.D Custom Processors]
      operationId: iterateCustomProcessor
      parameters:
        - in: path
          name: id
          required: true
          schema: { type: string }
      responses: { "200": { description: OK } }

  /v1/custom-processors/{id}/describe:
    post:
      tags: [L3.D Custom Processors]
      operationId: describeCustomProcessor
      parameters:
        - in: path
          name: id
          required: true
          schema: { type: string }
      responses: { "200": { description: OK } }

  /v1/custom-processors/{id}/execute:
    post:
      tags: [L3.D Custom Processors]
      operationId: executeCustomProcessor
      parameters:
        - in: path
          name: id
          required: true
          schema: { type: string }
      responses: { "200": { description: OK } }

  /v1/custom-processors/{id}/pipelines:
    get:
      tags: [L3.D Custom Processors]
      operationId: listCustomProcessorPipelines
      parameters:
        - in: path
          name: id
          required: true
          schema: { type: string }
      responses: { "200": { description: OK } }
```

### Domain L3.D — Schema Management

```yaml
paths:
  /v1/schemas:
    post:
      tags: [L3.D Schema Management]
      operationId: createSchema
      summary: Schema CRUD (soft delete; create_new_version)
      responses: { "201": { description: Created } }
    get:
      tags: [L3.D Schema Management]
      operationId: listSchemas
      responses: { "200": { description: OK } }

  /v1/schemas/{id}:
    parameters:
      - in: path
        name: id
        required: true
        schema: { type: string }
    get:
      tags: [L3.D Schema Management]
      operationId: getSchema
      responses: { "200": { description: OK } }
    put:
      tags: [L3.D Schema Management]
      operationId: updateSchema
      parameters:
        - in: query
          name: create_new_version
          schema: { type: boolean }
      responses: { "200": { description: OK } }
    delete:
      tags: [L3.D Schema Management]
      operationId: deleteSchema
      responses: { "204": { description: Soft deleted } }
```

### Domain L3.D — MCP Server Tools (document)

```yaml
paths:
  /v1/mcp:
    get:
      tags: [L3.D MCP Tools]
      operationId: documentMcpServer
      summary: MCP Server (WS; exposes search/ask/convert/extract/graph_query as agent tools)
      responses: { "101": { description: Switching Protocols } }
```

<details>
<summary>Continue with Layer 4 (next chunk)…</summary>

Layer 4 (Agentic Orchestration) covers Agent Definition (L4.A), Skills (L4.A), Models (agent-level, L4.B), Sessions (L4.C), Containers (L4.E), Connectors/MCP (L4.F), Multi-Agent (L4.I), Memory & Knowledge (L4.J), Workflows (L4.K), Channels (L4.L), Voice Channel (L4.L), Plugins & Marketplace (L4.M), External Agents (L4.M), and Webhooks (L4.N).
</details>

---

## Layer 4 — Agentic Orchestration

### Domain L4.A — Agent Definition & Configuration

```yaml
paths:
  /v1/agents:
    post:
      tags: [L4.A Agent Definition]
      operationId: createAgent
      summary: Create Agent (returns {id, version, created_at, updated_at})
      requestBody:
        required: true
        content:
          application/json:
            schema: { $ref: "#/components/schemas/Agent" }
      responses:
        "201":
          description: Created
          content:
            application/json:
              schema: { $ref: "#/components/schemas/Agent" }
    get:
      tags: [L4.A Agent Definition]
      operationId: listAgents
      parameters:
        - in: query
          name: agent_id
          schema: { type: string }
        - $ref: "#/components/parameters/OrderParam"
        - $ref: "#/components/parameters/LimitParam"
        - $ref: "#/components/parameters/CursorParam"
        - in: query
          name: include_hidden
          schema: { type: boolean }
      responses:
        "200":
          description: OK
          content:
            application/json:
              schema:
                allOf:
                  - $ref: "#/components/schemas/ListResponse"
                  - type: object
                    properties:
                      data: { type: array, items: { $ref: "#/components/schemas/Agent" } }

  /v1/agents/{id}:
    parameters:
      - in: path
        name: id
        required: true
        schema: { type: string }
      - in: query
        name: version
        schema: { type: integer }
    get:
      tags: [L4.A Agent Definition]
      operationId: retrieveAgent
      summary: Retrieve Agent (optionally pinned version)
      responses:
        "200":
          description: OK
          content:
            application/json:
              schema: { $ref: "#/components/schemas/Agent" }
        "404": { $ref: "#/components/responses/NotFound" }
    post:
      tags: [L4.A Agent Definition]
      operationId: updateAgent
      summary: Update Agent (version required; produces new version on change)
      requestBody:
        required: true
        content:
          application/json:
            schema:
              allOf:
                - $ref: "#/components/schemas/Agent"
                - type: object
                  required: [version]
      responses:
        "200":
          description: OK
          content:
            application/json:
              schema: { $ref: "#/components/schemas/Agent" }
    delete:
      tags: [L4.A Agent Definition]
      operationId: deleteAgent
      responses: { "204": { description: Deleted } }

  /v1/agents/{id}/archive:
    post:
      tags: [L4.A Agent Definition]
      operationId: archiveAgent
      summary: Archive Agent (one-way read-only archival)
      parameters:
        - in: path
          name: id
          required: true
          schema: { type: string }
      responses: { "200": { description: OK } }

  /v1/agents/{id}/versions:
    get:
      tags: [L4.A Agent Definition]
      operationId: listAgentVersions
      parameters:
        - in: path
          name: id
          required: true
          schema: { type: string }
        - $ref: "#/components/parameters/LimitParam"
        - $ref: "#/components/parameters/CursorParam"
      responses: { "200": { description: OK } }

  /v1/agents/{id}/releases:
    post:
      tags: [L4.A Agent Definition]
      operationId: createAgentRelease
      summary: Create Agent Release (immutable release snapshot)
      parameters:
        - in: path
          name: id
          required: true
          schema: { type: string }
      requestBody:
        required: true
        content:
          application/json:
            schema:
              type: object
              properties:
                version_label: { type: string }
                semantic_version: { type: string }
                version_name: { type: string }
                version_description: { type: string }
      responses: { "201": { description: Created } }

  /v1/agents/{id}/releases/{ver}/environment/{env_id}:
    post:
      tags: [L4.A Agent Definition]
      operationId: deployAgentRelease
      summary: Deploy Agent Release to Environment
      parameters:
        - in: path
          name: id
          required: true
          schema: { type: string }
        - in: path
          name: ver
          required: true
          schema: { type: string }
        - in: path
          name: env_id
          required: true
          schema: { type: string }
      responses: { "200": { description: OK } }

  /v1/agents/{id}/releases/{ver}/undeploy:
    post:
      tags: [L4.A Agent Definition]
      operationId: undeployAgentRelease
      parameters:
        - in: path
          name: id
          required: true
          schema: { type: string }
        - in: path
          name: ver
          required: true
          schema: { type: string }
      responses: { "200": { description: OK } }

  /v1/agents/create-from-template:
    post:
      tags: [L4.M Plugins & Marketplace]
      operationId: createAgentFromTemplate
      summary: Create Agent from Template
      requestBody:
        required: true
        content:
          application/json:
            schema:
              type: object
              properties:
                template_id: { type: string }
      responses: { "201": { description: Created } }

  /v1/agents/{id}/template-status:
    get:
      tags: [L4.M Plugins & Marketplace]
      operationId: agentTemplateStatus
      parameters:
        - in: path
          name: id
          required: true
          schema: { type: string }
      responses: { "200": { description: OK } }
```

### Domain L4.A — Skills

```yaml
paths:
  /v1/skills:
    post:
      tags: [L4.A Skills]
      operationId: uploadSkill
      summary: Upload Skill (Custom; multipart files; returns {skill_id, latest_version})
      requestBody:
        required: true
        content:
          multipart/form-data:
            schema:
              type: object
              properties:
                files: { type: array, items: { type: string, format: binary } }
      responses: { "201": { description: Created } }

  /v1/skills/config/write:
    post:
      tags: [L4.A Skills]
      operationId: writeSkillsConfig
      summary: Skills Config Write (enable/disable a skill)
      requestBody:
        required: true
        content:
          application/json:
            schema:
              type: object
              properties:
                path: { type: string }
                enabled: { type: boolean }
      responses: { "200": { description: OK } }

  /v1/skills/list:
    post:
      tags: [L4.A Skills]
      operationId: listSkills
      summary: List Skills (cwds, forceReload, perCwdExtraUserRoots)
      requestBody:
        required: true
        content:
          application/json:
            schema:
              type: object
              properties:
                cwds: { type: array, items: { type: string } }
                forceReload: { type: boolean }
                perCwdExtraUserRoots: { type: array, items: { type: string } }
      responses: { "200": { description: OK } }
```

### Domain L4.B — Models & Reasoning (agent-level)

```yaml
paths:
  /v1/models/list:
    get:
      tags: [L4.B Models (agent-level)]
      operationId: listModelsAgentLevel
      summary: List Models (agent-level; supportedReasoningEfforts, inputModalities, supportsPersonality, isDefault, upgrade)
      responses: { "200": { description: OK } }
```

> `/v1/model_policy` and `modelProvider/capabilities/read` are covered in L2.A.

### Domain L4.C — Sessions, Threads, Runs & Interactions

```yaml
paths:
  /v1/sessions:
    post:
      tags: [L4.C Sessions]
      operationId: createSession
      summary: Create Session (provisions sandbox + starts conversation history)
      description: |
        Returns {id, status, agent snapshot, environment_id}. Two-step
        lifecycle: create provisions sandbox → first message starts work.
        Two independent state dimensions (S.33): conversation history
        (previous_interaction_id) and sandbox/files (environment_id).
      requestBody:
        required: true
        content:
          application/json:
            schema:
              allOf:
                - $ref: "#/components/schemas/Session"
                - type: object
                  properties:
                    agent_ref: { type: string }
                    agent_with_overrides: { type: object }
      responses:
        "201":
          description: Created
          content:
            application/json:
              schema: { $ref: "#/components/schemas/Session" }
    get:
      tags: [L4.C Sessions]
      operationId: listSessions
      parameters:
        - $ref: "#/components/parameters/LimitParam"
        - in: query
          name: agent_id
          schema: { type: string }
        - $ref: "#/components/parameters/OrderParam"
        - $ref: "#/components/parameters/CursorParam"
      responses:
        "200":
          description: OK
          content:
            application/json:
              schema:
                allOf:
                  - $ref: "#/components/schemas/ListResponse"
                  - type: object
                    properties:
                      data: { type: array, items: { $ref: "#/components/schemas/Session" } }

  /v1/sessions/{id}:
    parameters:
      - in: path
        name: id
        required: true
        schema: { type: string }
    get:
      tags: [L4.C Sessions]
      operationId: retrieveSession
      responses:
        "200":
          description: OK
          content:
            application/json:
              schema: { $ref: "#/components/schemas/Session" }
    delete:
      tags: [L4.C Sessions]
      operationId: deleteSession
      responses: { "204": { description: Deleted } }

  /v1/sessions/{id}/events:
    post:
      tags: [L4.C Sessions]
      operationId: sendSessionEvents
      summary: Send Session Events (user.message / user.tool_confirmation / user.custom_tool_result / system.message)
      parameters:
        - in: path
          name: id
          required: true
          schema: { type: string }
      requestBody:
        required: true
        content:
          application/json:
            schema:
              type: object
              properties:
                events:
                  type: array
                  items: { type: object }
      responses: { "200": { description: OK } }
    get:
      tags: [L4.C Sessions]
      operationId: listSessionEvents
      parameters:
        - in: path
          name: id
          required: true
          schema: { type: string }
        - in: query
          name: types
          schema: { type: array, items: { type: string } }
        - $ref: "#/components/parameters/LimitParam"
        - $ref: "#/components/parameters/CursorParam"
      responses: { "200": { description: OK } }

  /v1/sessions/{id}/events/stream:
    get:
      tags: [L4.C Sessions]
      operationId: streamSessionEvents
      summary: Stream Session Events (SSE; event_deltas[] opt-in)
      parameters:
        - in: path
          name: id
          required: true
          schema: { type: string }
        - in: header
          name: Last-Event-ID
          schema: { type: string, description: "Resumable streaming cursor (S.10)." }
      responses:
        "200-stream": { description: SSE stream }

  /v1/sessions/{id}/resume:
    post:
      tags: [L4.C Sessions]
      operationId: resumeSession
      parameters:
        - in: path
          name: id
          required: true
          schema: { type: string }
      responses: { "200": { description: OK } }

  /v1/sessions/{id}/fork:
    post:
      tags: [L4.C Sessions]
      operationId: forkSession
      summary: Fork Session (new session with copied history; last_turn_id)
      parameters:
        - in: path
          name: id
          required: true
          schema: { type: string }
      requestBody:
        required: true
        content:
          application/json:
            schema:
              type: object
              properties:
                last_turn_id: { type: string }
      responses: { "201": { description: Created } }

  /v1/sessions/{id}/steer:
    post:
      tags: [L4.C Sessions]
      operationId: steerSession
      summary: Steer Session (append user input to in-flight turn)
      parameters:
        - in: path
          name: id
          required: true
          schema: { type: string }
      responses: { "200": { description: OK } }

  /v1/sessions/{id}/interrupt:
    post:
      tags: [L4.C Sessions]
      operationId: interruptSession
      summary: Interrupt Session (cancel mid-execution)
      parameters:
        - in: path
          name: id
          required: true
          schema: { type: string }
      responses: { "200": { description: OK } }

  /v1/sessions/{id}/compact:
    post:
      tags: [L4.C Sessions]
      operationId: compactSession
      summary: Compaction Trigger
      parameters:
        - in: path
          name: id
          required: true
          schema: { type: string }
      responses: { "200": { description: OK } }

  /v1/sessions/{id}/goal:
    parameters:
      - in: path
        name: id
        required: true
        schema: { type: string }
    post:
      tags: [L4.C Sessions]
      operationId: setSessionGoal
      summary: Session Goal CRUD (long-running target with progress)
      responses: { "200": { description: OK } }
    get:
      tags: [L4.C Sessions]
      operationId: getSessionGoal
      responses: { "200": { description: OK } }

  /v1/sessions/{id}/goal/clear:
    post:
      tags: [L4.C Sessions]
      operationId: clearSessionGoal
      parameters:
        - in: path
          name: id
          required: true
          schema: { type: string }
      responses: { "200": { description: OK } }

  /v1/sessions/{id}/name:
    post:
      tags: [L4.C Sessions]
      operationId: renameSession
      parameters:
        - in: path
          name: id
          required: true
          schema: { type: string }
      responses: { "200": { description: OK } }

  /v1/sessions/{id}/metadata:
    post:
      tags: [L4.C Sessions]
      operationId: updateSessionMetadata
      summary: Update Session Metadata (gitInfo/tag/custom_title patch)
      parameters:
        - in: path
          name: id
          required: true
          schema: { type: string }
      responses: { "200": { description: OK } }

  /v1/sessions/{id}/archive:
    post:
      tags: [L4.C Sessions]
      operationId: archiveSession
      parameters:
        - in: path
          name: id
          required: true
          schema: { type: string }
      responses: { "200": { description: OK } }

  /v1/sessions/{id}/threads:
    get:
      tags: [L4.C Sessions]
      operationId: listSessionThreads
      parameters:
        - in: path
          name: id
          required: true
          schema: { type: string }
      responses: { "200": { description: OK } }

  /v1/sessions/{id}/threads/{tid}/stream:
    get:
      tags: [L4.C Sessions]
      operationId: streamThread
      parameters:
        - in: path
          name: id
          required: true
          schema: { type: string }
        - in: path
          name: tid
          required: true
          schema: { type: string }
      responses: { "200-stream": { description: SSE stream } }

  /v1/sessions/{id}/threads/{tid}/events:
    get:
      tags: [L4.C Sessions]
      operationId: listThreadEvents
      parameters:
        - in: path
          name: id
          required: true
          schema: { type: string }
        - in: path
          name: tid
          required: true
          schema: { type: string }
      responses: { "200": { description: OK } }

  /v1/sessions/{id}/threads/{tid}/archive:
    post:
      tags: [L4.C Sessions]
      operationId: archiveThread
      parameters:
        - in: path
          name: id
          required: true
          schema: { type: string }
        - in: path
          name: tid
          required: true
          schema: { type: string }
      responses: { "200": { description: OK } }
```

### Domain L4.E — Container Management

```yaml
paths:
  /v1/containers:
    post:
      tags: [L4.E Containers]
      operationId: createContainer
      summary: Create Container (returns {id})
      requestBody:
        required: true
        content:
          application/json:
            schema:
              type: object
              properties:
                name: { type: string }
                memory_limit: { type: string }
                expires_after: { type: string }
                skills: { type: array, items: { type: object } }
      responses: { "201": { description: Created } }
    get:
      tags: [L4.E Containers]
      operationId: listContainers
      responses: { "200": { description: OK } }

  /v1/containers/{id}:
    delete:
      tags: [L4.E Containers]
      operationId: deleteContainer
      parameters:
        - in: path
          name: id
          required: true
          schema: { type: string }
      responses: { "204": { description: Deleted } }

  /v1/containers/{id}/files:
    post:
      tags: [L4.E Containers]
      operationId: createContainerFile
      summary: Container File Create (multipart or file_id)
      parameters:
        - in: path
          name: id
          required: true
          schema: { type: string }
      responses: { "201": { description: Created } }
    get:
      tags: [L4.E Containers]
      operationId: listContainerFiles
      summary: Container File List (generated files)
      parameters:
        - in: path
          name: id
          required: true
          schema: { type: string }
      responses: { "200": { description: OK } }
```

### Domain L4.F — Connectors / MCP Servers

```yaml
paths:
  /v1/connectors:
    post:
      tags: [L4.F Connectors / MCP]
      operationId: createConnector
      summary: Create Connector / MCP Server Registration
      requestBody:
        required: true
        content:
          application/json:
            schema:
              type: object
              required: [name]
              properties:
                name:
                  type: string
                  pattern: "^[a-zA-Z0-9_-]{1,64}$"
                description: { type: string }
                server: { type: object }
                server_url: { type: string }
                visibility:
                  type: string
                  enum: [private, shared_workspace, shared_org]
                icon_url: { type: string }
                headers: { type: object }
                auth_data:
                  type: object
                  properties:
                    client_id: { type: string }
                    client_secret: { type: string }
                system_prompt: { type: string }
                connector_id: { type: string }
      responses: { "201": { description: Created } }
    get:
      tags: [L4.F Connectors / MCP]
      operationId: listConnectors
      responses: { "200": { description: OK } }

  /v1/connectors/{idOrName}:
    parameters:
      - in: path
        name: idOrName
        required: true
        schema: { type: string }
    get:
      tags: [L4.F Connectors / MCP]
      operationId: getConnector
      responses: { "200": { description: OK } }
    post:
      tags: [L4.F Connectors / MCP]
      operationId: updateConnector
      responses: { "200": { description: OK } }
    delete:
      tags: [L4.F Connectors / MCP]
      operationId: deleteConnector
      responses: { "204": { description: Deleted } }

  /v1/connectors/{id}/tools:
    get:
      tags: [L4.F Connectors / MCP]
      operationId: listConnectorTools
      parameters:
        - in: path
          name: id
          required: true
          schema: { type: string }
        - in: query
          name: refresh
          schema: { type: boolean }
        - in: query
          name: pretty
          schema: { type: boolean }
      responses: { "200": { description: OK } }

  /v1/connectors/{id}/auth_url:
    post:
      tags: [L4.F Connectors / MCP]
      operationId: connectorOAuthUrl
      summary: Returns {auth_url, ttl}
      parameters:
        - in: path
          name: id
          required: true
          schema: { type: string }
      responses:
        "200":
          description: OK
          content:
            application/json:
              schema:
                type: object
                properties:
                  auth_url: { type: string }
                  ttl: { type: integer }

  /v1/mcp_servers/{name}/oauth/login:
    post:
      tags: [L4.F Connectors / MCP]
      operationId: mcpOAuthLogin
      summary: Returns auth_url
      parameters:
        - in: path
          name: name
          required: true
          schema: { type: string }
      responses: { "200": { description: OK } }

  /v1/sessions/{id}/mcp/{name}/reconnect:
    post:
      tags: [L4.F Connectors / MCP]
      operationId: mcpRuntimeReconnect
      parameters:
        - in: path
          name: id
          required: true
          schema: { type: string }
        - in: path
          name: name
          required: true
          schema: { type: string }
      responses: { "200": { description: OK } }

  /v1/sessions/{id}/mcp/{name}/toggle:
    post:
      tags: [L4.F Connectors / MCP]
      operationId: mcpRuntimeToggle
      parameters:
        - in: path
          name: id
          required: true
          schema: { type: string }
        - in: path
          name: name
          required: true
          schema: { type: string }
      requestBody:
        required: true
        content:
          application/json:
            schema:
              type: object
              properties:
                enabled: { type: boolean }
      responses: { "200": { description: OK } }

  /v1/sessions/{id}/mcp/status:
    get:
      tags: [L4.F Connectors / MCP]
      operationId: mcpStatus
      parameters:
        - in: path
          name: id
          required: true
          schema: { type: string }
      responses: { "200": { description: OK } }

  /v1/mcp/config/reload:
    post:
      tags: [L4.F Connectors / MCP]
      operationId: mcpConfigReload
      responses: { "200": { description: OK } }

  /v1/mcp/resource/read:
    post:
      tags: [L4.F Connectors / MCP]
      operationId: mcpResourceRead
      requestBody:
        required: true
        content:
          application/json:
            schema:
              type: object
              properties:
                server: { type: string }
                uri: { type: string }
      responses: { "200": { description: OK } }

  /v1/toolkits/prepare/list-tools:
    post:
      tags: [L4.F Connectors / MCP]
      operationId: prepareListTools
      responses: { "200": { description: OK } }

  /v1/tools/{tenant_id}/callback/{correlation_id}:
    post:
      tags: [L4.F Connectors / MCP]
      operationId: asyncToolCallback
      summary: Async Tool Callback (tenant_id, correlation_id, result)
      parameters:
        - in: path
          name: tenant_id
          required: true
          schema: { type: string }
        - in: path
          name: correlation_id
          required: true
          schema: { type: string }
      responses: { "200": { description: OK } }
```

### Domain L4.I — Multi-Agent Orchestration

```yaml
paths:
  /v1/spawn_agents_on_csv:
    post:
      tags: [L4.I Multi-Agent]
      operationId: spawnAgentsOnCsv
      summary: CSV Batch Fan-Out (one worker per row; combined CSV export)
      requestBody:
        required: true
        content:
          application/json:
            schema:
              type: object
              properties:
                csv_path: { type: string }
                instruction: { type: string }
                id_column: { type: string }
                output_schema: { type: object }
                output_csv_path: { type: string }
                max_concurrency: { type: integer }
                max_runtime_seconds: { type: integer }
      responses: { "202": { description: Accepted } }
```

> Subagent / Agent tool invocation, Agent Teams, Handoffs, Agents-as-Tools, Collaborators, Resume Subagent by threadId, and Dynamic Workflow JS are runtime constructs invoked within sessions (no distinct REST paths); they are surfaced via the `tools` array on agents/sessions (built-in orchestration tools: Agent/spawn_subagent/start_subtask/task/collabToolCall; Skill/use_skill; switch_mode; start_workflow; update_todo_list).

### Domain L4.J — Memory & Knowledge (RAG)

```yaml
paths:
  /v1/memory_stores:
    post:
      tags: [L4.J Memory & Knowledge]
      operationId: createMemoryStore
      summary: Create Memory Store (workspace-scoped; returns {id})
      requestBody:
        required: true
        content:
          application/json:
            schema:
              type: object
              required: [name]
              properties:
                name: { type: string }
                description: { type: string }
      responses: { "201": { description: Created } }

  /v1/memory_stores/{id}/memories:
    post:
      tags: [L4.J Memory & Knowledge]
      operationId: seedMemory
      summary: Seed Memory (no overwrite)
      parameters:
        - in: path
          name: id
          required: true
          schema: { type: string }
      requestBody:
        required: true
        content:
          application/json:
            schema:
              type: object
              properties:
                path: { type: string }
                content: { type: string }
      responses: { "201": { description: Created } }
    get:
      tags: [L4.J Memory & Knowledge]
      operationId: listMemories
      summary: List Memories (path_prefix, depth)
      parameters:
        - in: path
          name: id
          required: true
          schema: { type: string }
        - in: query
          name: path_prefix
          schema: { type: string }
        - in: query
          name: depth
          schema: { type: integer }
      responses: { "200": { description: OK } }

  /v1/memory_stores/{id}/memories/{mem_id}:
    parameters:
      - in: path
        name: id
        required: true
        schema: { type: string }
      - in: path
        name: mem_id
        required: true
        schema: { type: string }
    get:
      tags: [L4.J Memory & Knowledge]
      operationId: getMemory
      responses: { "200": { description: OK } }
    post:
      tags: [L4.J Memory & Knowledge]
      operationId: updateMemory
      summary: Update Memory (content and/or path rename; precondition.content_sha256)
      requestBody:
        required: true
        content:
          application/json:
            schema:
              type: object
              properties:
                content: { type: string }
                path: { type: string }
                precondition:
                  type: object
                  properties:
                    type: { type: string, enum: [content_sha256] }
                    content_sha256: { type: string }
      responses: { "200": { description: OK } }
    delete:
      tags: [L4.J Memory & Knowledge]
      operationId: deleteMemory
      responses: { "204": { description: Deleted } }

  /v1/memory_stores/{id}/memory_versions:
    get:
      tags: [L4.J Memory & Knowledge]
      operationId: listMemoryVersions
      parameters:
        - in: path
          name: id
          required: true
          schema: { type: string }
      responses: { "200": { description: OK } }

  /v1/memory_stores/{id}/memory_versions/{vid}:
    parameters:
      - in: path
        name: id
        required: true
        schema: { type: string }
      - in: path
        name: vid
        required: true
        schema: { type: string }
    get:
      tags: [L4.J Memory & Knowledge]
      operationId: getMemoryVersion
      responses: { "200": { description: OK } }

  /v1/memory_stores/{id}/memory_versions/{vid}/redact:
    post:
      tags: [L4.J Memory & Knowledge]
      operationId: redactMemoryVersion
      summary: Redact Memory Version (scrubs content, preserves audit)
      parameters:
        - in: path
          name: id
          required: true
          schema: { type: string }
        - in: path
          name: vid
          required: true
          schema: { type: string }
      responses: { "200": { description: OK } }

  /v1/memories:
    post:
      tags: [L4.J Memory & Knowledge]
      operationId: agenticMemoryAdd
      summary: Agentic Memory add
      responses: { "201": { description: Created } }
    get:
      tags: [L4.J Memory & Knowledge]
      operationId: agenticMemoryList
      responses: { "200": { description: OK } }

  /v1/memories/search:
    post:
      tags: [L4.J Memory & Knowledge]
      operationId: agenticMemorySearch
      responses: { "200": { description: OK } }

  /v1/libraries:
    post:
      tags: [L4.J Memory & Knowledge]
      operationId: createLibrary
      summary: Create Library (returns {library_id, generated_name, generated_description})
      requestBody:
        required: true
        content:
          application/json:
            schema:
              type: object
              properties:
                name: { type: string }
                description: { type: string }
      responses: { "201": { description: Created } }

  /v1/libraries/{id}/documents:
    post:
      tags: [L4.J Memory & Knowledge]
      operationId: uploadLibraryDocument
      parameters:
        - in: path
          name: id
          required: true
          schema: { type: string }
      requestBody:
        required: true
        content:
          multipart/form-data:
            schema:
              type: object
              properties:
                file: { type: string, format: binary }
      responses: { "201": { description: Created } }
    get:
      tags: [L4.J Memory & Knowledge]
      operationId: listLibraryDocuments
      parameters:
        - in: path
          name: id
          required: true
          schema: { type: string }
      responses: { "200": { description: OK } }

  /v1/libraries/{id}/documents/{doc_id}/status:
    get:
      tags: [L4.J Memory & Knowledge]
      operationId: libraryDocumentStatus
      parameters:
        - in: path
          name: id
          required: true
          schema: { type: string }
        - in: path
          name: doc_id
          required: true
          schema: { type: string }
      responses: { "200": { description: OK } }

  /v1/libraries/{id}/documents/{doc_id}/text_content:
    get:
      tags: [L4.J Memory & Knowledge]
      operationId: libraryDocumentTextContent
      parameters:
        - in: path
          name: id
          required: true
          schema: { type: string }
        - in: path
          name: doc_id
          required: true
          schema: { type: string }
      responses: { "200": { description: OK } }

  /v1/libraries/accesses/{id}:
    get:
      tags: [L4.J Memory & Knowledge]
      operationId: listLibraryAccesses
      parameters:
        - in: path
          name: id
          required: true
          schema: { type: string }
      responses: { "200": { description: OK } }

  /v1/orchestrate/knowledge-bases:
    post:
      tags: [L4.J Memory & Knowledge]
      operationId: createKnowledgeBase
      summary: Create Knowledge Base (built-in managed Milvus or external)
      requestBody:
        required: true
        content:
          application/json:
            schema:
              type: object
              properties:
                knowledge_base: { type: object }
                documents: { type: array, items: { type: object } }
                embeddings_model_name: { type: string }
                chunk_size: { type: integer }
      responses: { "201": { description: Created } }

  /v1/orchestrate/knowledge-bases/{kb_id}/status:
    get:
      tags: [L4.J Memory & Knowledge]
      operationId: knowledgeBaseStatus
      parameters:
        - in: path
          name: kb_id
          required: true
          schema: { type: string }
      responses: { "200": { description: OK } }

  /v1/vector-indices:
    post:
      tags: [L4.J Memory & Knowledge]
      operationId: createVectorIndex
      summary: Create Vector Index (embeddings model + chunking + retrieval)
      requestBody:
        required: true
        content:
          application/json:
            schema:
              type: object
              properties:
                name: { type: string }
                embeddings_model_name: { type: string }
                chunk_size: { type: integer }
                chunk_overlap: { type: integer }
                top_k: { type: integer }
                extraction_strategy: { type: string }
      responses: { "201": { description: Created } }

  /v1/vector-indices/{id}/collections:
    post:
      tags: [L4.J Memory & Knowledge]
      operationId: attachCollectionsToVectorIndex
      parameters:
        - in: path
          name: id
          required: true
          schema: { type: string }
      responses: { "200": { description: OK } }

  /v1/vector-indices/{id}/refresh:
    post:
      tags: [L4.J Memory & Knowledge]
      operationId: refreshVectorIndex
      parameters:
        - in: path
          name: id
          required: true
          schema: { type: string }
      responses: { "200": { description: OK } }

  /v1/vector-indices/{id}/rebuild:
    post:
      tags: [L4.J Memory & Knowledge]
      operationId: rebuildVectorIndex
      parameters:
        - in: path
          name: id
          required: true
          schema: { type: string }
      responses: { "200": { description: OK } }

  /v1/vector-indices/{id}/retrieve:
    get:
      tags: [L4.J Memory & Knowledge]
      operationId: retrieveFromVectorIndex
      parameters:
        - in: path
          name: id
          required: true
          schema: { type: string }
      responses: { "200": { description: OK } }
```

### Domain L4.K — Workflows, Scheduled Tasks & Automation

```yaml
paths:
  /v1/orchestrate/flows/{id}/run:
    post:
      tags: [L4.K Workflows]
      operationId: runFlow
      summary: Run Flow (sync)
      parameters:
        - in: path
          name: id
          required: true
          schema: { type: string }
      requestBody:
        required: true
        content:
          application/json:
            schema:
              type: object
              properties:
                input: { type: object }
      responses: { "200": { description: OK } }

  /v1/orchestrate/flows/{id}/run/async:
    post:
      tags: [L4.K Workflows]
      operationId: runFlowAsync
      summary: Run Flow Async (callbackUrl)
      parameters:
        - in: path
          name: id
          required: true
          schema: { type: string }
      requestBody:
        required: true
        content:
          application/json:
            schema:
              type: object
              properties:
                input: { type: object }
                callbackUrl: { type: string }
      responses: { "202": { description: Accepted } }
```

> `start_workflow(workflow_name, args?)` is a runtime tool, not a distinct REST path.

### Domain L4.L — Channels, Voice & Embedded Chat

```yaml
paths:
  /v1/agents/{agent_id}/environments/{env_id}/channels:
    post:
      tags: [L4.L Channels]
      operationId: bindChannel
      summary: Bind Channel to (Agent, Environment)
      parameters:
        - in: path
          name: agent_id
          required: true
          schema: { type: string }
        - in: path
          name: env_id
          required: true
          schema: { type: string }
      requestBody:
        required: true
        content:
          application/json:
            schema:
              type: object
              properties:
                channel_config: { type: object }
      responses: { "201": { description: Created } }

  /v1/channels/phone:
    post:
      tags: [L4.L Channels]
      operationId: createPhoneChannel
      summary: Phone Channel CRUD
      responses: { "201": { description: Created } }

  /v1/channels/phone/{id}/numbers:
    post:
      tags: [L4.L Channels]
      operationId: managePhoneNumbers
      summary: Numbers Management (add/list/patch/delete)
      parameters:
        - in: path
          name: id
          required: true
          schema: { type: string }
      responses: { "200": { description: OK } }

  /v1/agents/{id}/embedded-chat-config:
    put:
      tags: [L4.L Channels]
      operationId: updateEmbeddedChatConfig
      summary: Embedded Chat Config (layout, is_live) + web chat SDK
      parameters:
        - in: path
          name: id
          required: true
          schema: { type: string }
      responses: { "200": { description: OK } }

  /v1/embed-settings/config:
    put:
      tags: [L4.L Channels]
      operationId: updateEmbedSettings
      summary: Embed Settings CRUD
      responses: { "200": { description: OK } }

  /v1/embed-settings/generate-key-pair:
    post:
      tags: [L4.L Channels]
      operationId: generateEmbedKeyPair
      responses: { "201": { description: Created } }

  /v1/agents/{id}/chat-starter-settings:
    put:
      tags: [L4.L Channels]
      operationId: updateChatStarterSettings
      summary: Chat Starter Settings (starter_prompts/welcome_content/icon)
      parameters:
        - in: path
          name: id
          required: true
          schema: { type: string }
      responses: { "200": { description: OK } }

  /v1/voice-configurations:
    post:
      tags: [L4.L Voice Channel]
      operationId: createVoiceConfiguration
      summary: Voice Configuration CRUD (AgentIdleHandler, RealtimeAgentSettings)
      requestBody:
        required: true
        content:
          application/json:
            schema:
              type: object
              properties:
                agent_idle_handler: { type: object }
                realtime_agent_settings: { type: object }
      responses: { "201": { description: Created } }
```

> Realtime API WebSocket upgrade `/v1/realtime`, `/v1/realtime/calls`, `/v1/realtime/client_secrets` are defined in L3.C Voice Agents.

### Domain L4.M — Extensions, Plugins, Marketplaces & Interoperability

```yaml
paths:
  /v1/marketplaces/add:
    post:
      tags: [L4.M Plugins & Marketplace]
      operationId: addMarketplace
      summary: Add Marketplace (source: local|git|npm|remote)
      requestBody:
        required: true
        content:
          application/json:
            schema:
              type: object
              properties:
                source: { type: string, enum: [local, git, npm, remote] }
                url: { type: string }
      responses: { "201": { description: Added } }

  /v1/marketplaces/{name}/remove:
    post:
      tags: [L4.M Plugins & Marketplace]
      operationId: removeMarketplace
      parameters:
        - in: path
          name: name
          required: true
          schema: { type: string }
      responses: { "204": { description: Removed } }

  /v1/marketplaces/{name}/upgrade:
    post:
      tags: [L4.M Plugins & Marketplace]
      operationId: upgradeMarketplace
      parameters:
        - in: path
          name: name
          required: true
          schema: { type: string }
      responses: { "200": { description: OK } }

  /v1/plugins:
    get:
      tags: [L4.M Plugins & Marketplace]
      operationId: listPlugins
      responses: { "200": { description: OK } }

  /v1/plugins/{id}:
    get:
      tags: [L4.M Plugins & Marketplace]
      operationId: getPlugin
      parameters:
        - in: path
          name: id
          required: true
          schema: { type: string }
      responses: { "200": { description: OK } }

  /v1/plugins/install:
    post:
      tags: [L4.M Plugins & Marketplace]
      operationId: installPlugin
      summary: Install Plugin (marketplacePath | remoteMarketplaceName)
      responses: { "201": { description: Installed } }

  /v1/plugins/{id}/uninstall:
    post:
      tags: [L4.M Plugins & Marketplace]
      operationId: uninstallPlugin
      parameters:
        - in: path
          name: id
          required: true
          schema: { type: string }
      responses: { "204": { description: Uninstalled } }

  /v1/plugins/{id}/skill/read:
    post:
      tags: [L4.M Plugins & Marketplace]
      operationId: readPluginSkill
      summary: Read Plugin Skill (remote plugin skill Markdown on demand)
      parameters:
        - in: path
          name: id
          required: true
          schema: { type: string }
      responses: { "200": { description: OK } }

  /v1/tools/create-from-template:
    post:
      tags: [L4.M Plugins & Marketplace]
      operationId: createToolFromTemplate
      responses: { "201": { description: Created } }

  /v1/catalog:
    get:
      tags: [L4.M Plugins & Marketplace]
      operationId: browseCatalog
      summary: Browse Catalog (governed library of pre-built agents and tools)
      responses: { "200": { description: OK } }

  /v1/agents/external-chat:
    post:
      tags: [L4.M External Agents]
      operationId: createExternalChatAgent
      summary: Create External-Chat Agent (api_url, auth_scheme, auth_config)
      requestBody:
        required: true
        content:
          application/json:
            schema:
              type: object
              properties:
                api_url: { type: string }
                auth_scheme:
                  type: string
                  enum: [BEARER_TOKEN, API_KEY, NONE]
                auth_config: { type: object }
      responses: { "201": { description: Created } }

  /v1/a2a/versions:
    get:
      tags: [L4.M External Agents]
      operationId: a2aProtocolVersions
      summary: A2A Protocol Versions (client/server role)
      responses: { "200": { description: OK } }

  /v1/externalAgentConfig/detect:
    post:
      tags: [L4.M External Agents]
      operationId: detectExternalAgentConfig
      summary: External Agent Config Detect (discovers & migrates artifacts)
      responses: { "200": { description: OK } }

  /v1/externalAgentConfig/import:
    post:
      tags: [L4.M External Agents]
      operationId: importExternalAgentConfig
      responses: { "200": { description: OK } }
```

### Domain L4.N — Webhooks, Event Delivery & ChatGPT Developer Mode

> Webhook registration/delivery infrastructure is in L0.O. L4 documents the agent-scoped event subscription facet, which uses the same `/v1/webhooks` endpoint with `scope: agent` semantics.

<details>
<summary>Continue with Layer 5 (next chunk)…</summary>

Layer 5 (Observability & Administration) covers Observability Export (L5.A), Hosted UI (L5.A), Attribution (L5.B), Traffic Inspection (L5.E), Judges & Evaluators (L5.F), Eval Campaigns (L5.F), Datasets (L5.F), Feedback (L5.G), Moderation (L5.I), Guardrails (L5.I), PII & Redaction (L5.I), Safety Filters (L5.I), Refusal Fallback (L5.I), Approvals/HITL (L5.I), Prompt Injection Defense (L5.I), Model Lifecycle Admin (L5.J), Cache Diagnostics (L5.K), Analytics Portals (L5.L), Telemetry Backends (L5.M), and Cost & Usage Lookup (L5.N).
</details>

---

## Layer 5 — Observability & Administration

### Domain L5.A — Telemetry & Observability Backend Wiring

```yaml
paths:
  /v1/telemetry/config:
    put:
      tags: [L5.A Observability Export]
      operationId: configureTelemetry
      summary: Telemetry Configuration Service
      description: |
        Master `enable_telemetry` flag; per-signal exporters
        (`metrics_exporter`/`logs_exporter`/`traces_exporter`:
        console/otlp/prometheus/none); `enable_enhanced_telemetry_beta` for
        span tracing; tracing scope (SDK-level / per-run reduction).
      requestBody:
        required: true
        content:
          application/json:
            schema:
              type: object
              properties:
                enable_telemetry: { type: boolean }
                enable_enhanced_telemetry_beta: { type: boolean }
                metrics_exporter:
                  type: string
                  enum: [console, otlp, prometheus, none]
                logs_exporter:
                  type: string
                  enum: [console, otlp, prometheus, none]
                traces_exporter:
                  type: string
                  enum: [console, otlp, prometheus, none]
                tracing_scope: { type: string }
      responses: { "200": { description: OK } }

  /v1/dashboard:
    get:
      tags: [L5.A Hosted UI]
      operationId: hostedDashboard
      summary: Hosted Dashboard Service (Traces / Explorer / AI Studio Logs)
      parameters:
        - in: query
          name: workspace
          schema: { type: string }
        - in: query
          name: project
          schema: { type: string }
      responses: { "200": { description: OK } }
```

### Domain L5.B — Identity, Resource Attribution & Tenancy Configuration

```yaml
paths:
  /v1/observability/attribution:
    put:
      tags: [L5.B Attribution]
      operationId: configureResourceAttributes
      summary: Resource Attributes Service
      description: |
        Configures attributes stamped on telemetry: service.name,
        resource_attributes env, enduser.id/tenant.id/user.account_uuid/
        user.id/user.email/user.groups/identity.source/app.version/
        app.entrypoint/terminal.type; metadata.user_id opaque no-PII;
        metadata custom KV per-request filterable in Explorer; api_agent_id;
        correlation_id; organization.id/prompt.id/workflow.run_id/
        workflow.name/workspace.host_paths; bool controls
        include_resource_attributes/include_session_id/include_account_uuid/
        include_version/include_entrypoint.
      requestBody:
        required: true
        content:
          application/json:
            schema:
              type: object
              properties:
                service_name: { type: string }
                resource_attributes: { type: object }
                include_resource_attributes: { type: boolean }
                include_session_id: { type: boolean }
                include_account_uuid: { type: boolean }
                include_version: { type: boolean }
                include_entrypoint: { type: boolean }
      responses: { "200": { description: OK } }
```

### Domain L5.E — Search, Filter & Inspect Production Traffic

```yaml
paths:
  /v1/observability/logs/search:
    post:
      tags: [L5.E Traffic Inspection]
      operationId: searchObservabilityLogs
      summary: Observability Logs Search Service (structured filter condition)
      description: |
        Filter operators: =/eq, !=/ne, contains, includes, excludes, >/gt,
        </lt, >=/gte, <=/lte, isnull, length_equals, starts_with, ends_with,
        matches (regex). Filterable fields: timestamp, model_name,
        last_user_message_preview, response_messages_preview, invoked_tools,
        total_time_elapsed, input_tokens, output_tokens, api_agent_id,
        event_id, correlation_id, first_system_message, metadata.
      requestBody:
        required: true
        content:
          application/json:
            schema:
              type: object
              properties:
                filter:
                  type: object
                  description: "Structured filter condition {field, op, value} with AND/OR trees."
                extra_fields: { type: array, items: { type: string } }
                page_size: { type: integer }
      responses: { "200": { description: OK } }

  /v1/observability/traces/search:
    post:
      tags: [L5.E Traffic Inspection]
      operationId: searchObservabilityTraces
      responses: { "200": { description: OK } }

  /v1/observability/traces/{id}:
    get:
      tags: [L5.E Traffic Inspection]
      operationId: getTrace
      parameters:
        - in: path
          name: id
          required: true
          schema: { type: string }
      responses: { "200": { description: OK } }

  /v1/observability/traces/{id}/spans/{spanId}:
    get:
      tags: [L5.E Traffic Inspection]
      operationId: getTraceSpan
      parameters:
        - in: path
          name: id
          required: true
          schema: { type: string }
        - in: path
          name: spanId
          required: true
          schema: { type: string }
      responses: { "200": { description: OK } }

  /v1/workflows/events:
    get:
      tags: [L5.E Traffic Inspection]
      operationId: listWorkflowEvents
      summary: Workflow Events Service (cursor pagination)
      parameters:
        - $ref: "#/components/parameters/CursorParam"
        - $ref: "#/components/parameters/LimitParam"
      responses: { "200": { description: OK } }
```

### Domain L5.F — Evaluation, Scoring & Datasets

```yaml
paths:
  /v1/observability/judges:
    post:
      tags: [L5.F Judges & Evaluators]
      operationId: createJudge
      summary: Judge Service
      description: |
        Auto-injects conversation history / user message / assistant response /
        available tools. Jinja2 vars: {{ conversation_history }},
        {{ user_message }}, {{ assistant_message }}, {{ system_prompt }},
        {{ available_tools }}, {{ answer_type_definition }},
        {{ properties.* }}. Versioning via base_revision/up_revision/down_revision.
      requestBody:
        required: true
        content:
          application/json:
            schema:
              type: object
              properties:
                name: { type: string }
                description: { type: string }
                model_name: { type: string }
                instructions: { type: string }
                output:
                  oneOf:
                    - type: object
                      description: CLASSIFICATION
                      properties:
                        type: { type: string, enum: [CLASSIFICATION] }
                        options:
                          type: array
                          items:
                            type: object
                            properties:
                              value: { type: string }
                              description: { type: string }
                    - type: object
                      description: REGRESSION
                      properties:
                        type: { type: string, enum: [REGRESSION] }
                        min: { type: number }
                        max: { type: number }
                        min_description: { type: string }
                        max_description: { type: string }
                tools: { type: array, items: { type: string } }
      responses: { "201": { description: Created } }

  /v1/observability/campaigns:
    post:
      tags: [L5.F Eval Campaigns]
      operationId: createCampaign
      summary: Campaign Service (background async; annotations written back into Explorer)
      description: |
        `fetch_status()`, `list_events()`. Filter cannot change after start.
        Deleting a Campaign does not necessarily lose annotations.
      requestBody:
        required: true
        content:
          application/json:
            schema:
              type: object
              properties:
                name: { type: string }
                description: { type: string }
                judge_id: { type: string }
                search_params: { type: object }
                max_nb_events:
                  type: integer
                  minimum: 100
                  maximum: 10000
      responses: { "202": { description: Accepted } }

  /v1/agent/{id}/test_case:
    post:
      tags: [L5.F Eval Campaigns]
      operationId: uploadTestCase
      summary: Test Case Upload Service (CSV body)
      parameters:
        - in: path
          name: id
          required: true
          schema: { type: string }
      requestBody:
        required: true
        content:
          text/csv:
            schema: { type: string }
      responses: { "201": { description: Created } }

  /v1/agent/{id}/evaluate:
    post:
      tags: [L5.F Eval Campaigns]
      operationId: evaluateAgent
      summary: Evaluate Service (rubric evaluations; LLM agent vulnerability testing incl. adversarial/red-team)
      parameters:
        - in: path
          name: id
          required: true
          schema: { type: string }
      requestBody:
        required: true
        content:
          application/json:
            schema:
              type: object
              properties:
                evaluation_config: { type: object }
      responses: { "202": { description: Accepted } }

  /v1/agent/{id}/evaluations:
    get:
      tags: [L5.F Eval Campaigns]
      operationId: listEvaluations
      parameters:
        - in: path
          name: id
          required: true
          schema: { type: string }
      responses: { "200": { description: OK } }

  /v1/agent/{id}/evaluations/export:
    post:
      tags: [L5.F Eval Campaigns]
      operationId: exportEvaluations
      summary: Evaluations Export Service (evaluation_ids:[...])
      parameters:
        - in: path
          name: id
          required: true
          schema: { type: string }
      requestBody:
        required: true
        content:
          application/json:
            schema:
              type: object
              properties:
                evaluation_ids: { type: array, items: { type: string } }
      responses: { "200": { description: OK } }

  /v1/agent/test_case/templates:
    get:
      tags: [L5.F Datasets]
      operationId: getTestCaseTemplates
      summary: Test Case Template Service (returns sample CSV)
      responses:
        "200":
          description: OK
          content:
            text/csv:
              schema: { type: string }

  /v1/observability/datasets:
    post:
      tags: [L5.F Datasets]
      operationId: createDataset
      summary: Dataset Service (record = Conversation + Properties + Source)
      description: |
        Sources: EXPLORER/UPLOADED_FILE/DIRECT_INPUT/PLAYGROUND. Editable
        (fix messages, add expected outputs, remove duplicates). Export to
        JSONL. Golden-set datasets with labels/expected outputs.
      requestBody:
        required: true
        content:
          application/json:
            schema:
              type: object
              properties:
                name: { type: string }
                description: { type: string }
      responses: { "201": { description: Created } }
```

### Domain L5.G — Feedback & Improvement

```yaml
paths:
  /v1/feedback/upload:
    post:
      tags: [L5.G Feedback]
      operationId: uploadFeedback
      summary: Feedback Upload Service (classification, reason, logs, conversation_id)
      requestBody:
        required: true
        content:
          application/json:
            schema:
              type: object
              properties:
                classification: { type: string }
                reason: { type: string }
                logs: { type: object }
                conversation_id: { type: string }
                extraLogFiles: { type: array, items: { type: string } }
      responses: { "201": { description: Created } }

  /v1/attestation/generate:
    post:
      tags: [L5.G Feedback]
      operationId: generateAttestation
      summary: Attestation Service (opt-in via requestAttestation capability)
      responses: { "201": { description: Created } }
```

### Domain L5.I — Safety, Guardrails, Moderation & Approvals

```yaml
paths:
  /v1/moderations:
    post:
      tags: [L5.I Moderation]
      operationId: moderateContent
      summary: Moderation Service (reduce unsafe content; returns per-category scores)
      requestBody:
        required: true
        content:
          application/json:
            schema:
              type: object
              properties:
                input: { type: string }
                messages: { type: array, items: { $ref: "#/components/schemas/Message" } }
                model: { type: string, example: "moderation-latest" }
                safety_settings:
                  type: array
                  items:
                    type: object
                    properties:
                      category: { type: string }
                      threshold: { type: string }
      responses:
        "200":
          description: OK
          content:
            application/json:
              schema:
                type: object
                properties:
                  results:
                    type: array
                    items:
                      type: object
                      properties:
                        categories: { type: object }
                        category_scores: { type: object }
                        flagged: { type: boolean }

  /v1/chat/moderations:
    post:
      tags: [L5.I Moderation]
      operationId: moderateChat
      responses: { "200": { description: OK } }

  /v1/guardrails:
    post:
      tags: [L5.I Guardrails]
      operationId: applyCustomGuardrails
      summary: Custom Guardrails Service (declarative guardrails array; input-only; 403 on trigger)
      description: |
        Each guardrail has `moderation_llm_v2` config: custom_category_thresholds
        0-1 per category (1 disables), ignore_other_categories, action:"block",
        model_name override, block_on_error. Blocked if any triggers.
        Attachable inline on chat.complete/conversations.start/agent-level
        agents.create inherited overridable per-request.
      requestBody:
        required: true
        content:
          application/json:
            schema:
              type: object
              properties:
                guardrails:
                  type: array
                  items:
                    type: object
                    properties:
                      moderation_llm_v2:
                        type: object
                        properties:
                          custom_category_thresholds: { type: object }
                          ignore_other_categories: { type: boolean }
                          action: { type: string, enum: [block] }
                          model_name: { type: string }
                          block_on_error: { type: boolean }
      responses: { "200": { description: OK } }
      # 403 returned when a guardrail triggers (S.4).
```

> Input Guardrails / Output Guardrails / Tool Guardrails are runtime constructs configured on agents/sessions (`Agent.input_guardrails`, `Agent.output_guardrails`, tool-level guardrails) and surfaced via the agent definition `guardrails` field rather than distinct REST endpoints.

### Domain L5.I — PII & Redaction

```yaml
paths:
  /v1/pii/redact:
    post:
      tags: [L5.I PII & Redaction]
      operationId: detectAndRedactPii
      summary: PII Detection & Redaction (Text / Conversation / Document PII)
      requestBody:
        required: true
        content:
          application/json:
            schema:
              type: object
              properties:
                text: { type: string }
                conversations: { type: array, items: { type: object } }
                document: { type: object }
                redaction_policy:
                  type: string
                  enum: [characterMask, noMask, entityMask, syntheticReplacement]
                piiCategories: { type: array, items: { type: string } }
                confidenceScoreThreshold: { type: number }
                confidence_threshold_overrides: { type: array, items: { type: object } }
                disableEntityValidation: { type: boolean }
                excludeExtractionData: { type: boolean }
      responses: { "200": { description: OK } }
```

### Domain L5.I — Refusal Handling

> Refusal Fallback is configured via the `fallbacks` parameter on generation requests (up to 3 entries each `{model, max_tokens?, thinking?}`), tried in order; only safety-classifier decline triggers fallback. `usage.iterations` records every attempt. SDK middleware `BetaRefusalFallbackMiddleware`. No distinct REST path — it is a request-level parameter on `/v1/responses`, `/v1/messages`, `/v1/chat/completions`.

### Domain L5.I — Approvals (HITL)

> Approval flow is surfaced via the Sessions API: `requires_action` + `user.tool_confirmation` events on `/v1/sessions/{id}/events`, and `POST /v1/sessions/{id}/events` with `user.tool_confirmation` (allow/deny + deny_message) to resume. See L4.C. Computer-Use Safety Policies surface `safety_decision:{require_confirmation}` on action results within sessions. Tool-Approval SDK Mechanics (`RunContext` + `DeferredToolCallsException`, `dc.confirm()`/`dc.reject()`, `deferred.to_dict()`) are SDK constructs.

### Domain L5.I — Prompt Injection Defense

> Prompt Injection Defense is a capability group expressed as request parameters (`enable_prompt_injection_detection`, `allowed_domains`/`blocked_domains`, secret injection with `domain_secrets` placeholder names) and runtime policies (developer-message precedence, structured outputs for data flow, tool approvals, guardrails for user inputs, URL validation restricting fetch to URLs already in context, path-traversal protection for memory/file tools, shell allowlists, treat all tool/web/page content as untrusted). No distinct REST path.

### Domain L5.J — Model Lifecycle Admin

> Model Allowlist/Denylist Service is `PATCH /v1/organization/projects/{id}/model_permissions` (defined in L0.N).

### Domain L5.K — Caching Diagnostics (admin)

> Cache Diagnostics Service returns `cache_miss_reason` per-request fingerprint comparison (beta, ZDR-eligible). It is surfaced as a response field on generation requests (L2.B/L1.G) when the `cache_diagnostics` beta feature is enabled, not a distinct REST endpoint.

### Domain L5.L — Enterprise Analytics Portals / Supervision Surfaces

```yaml
paths:
  /v1/llm-analytics:
    get:
      tags: [L5.M Telemetry Backends]
      operationId: getLlmAnalyticsConfig
      responses: { "200": { description: OK } }
    post:
      tags: [L5.M Telemetry Backends]
      operationId: createLlmAnalyticsConfig
      responses: { "201": { description: Created } }
```

> Enterprise Analytics Portal, Supervision Surfaces, and Stored Interactions Viewer are hosted UI surfaces exposed via `/v1/dashboard` (L5.A). External Governance (WXG) Integration Service and External Telemetry Backend Integration are configuration-driven (no distinct REST paths); they are wired via the Telemetry Configuration Service (`/v1/telemetry/config`, L5.A) and ADK flags.

### Domain L5.N — Cost & Usage Lookup

```yaml
paths:
  # Generation Stats Service — GET /api/v1/generation?id={id} is defined in L0.I.
```

---

## Cross-Reference Notes

1. **Standards (S.1–S.33)** are encoded as reusable components (`$ref`) and applied uniformly across all layers. Where a transversal convention is request-level (e.g. `service_tier`, `prompt_cache_key`, `previous_response_id`, `fallbacks`, `guardrails`, `cache_control`, `defer_loading`), it appears as a property on the relevant generation/request schema rather than a distinct endpoint.
2. **Capabilities vs. services.** Architecture modules that list *capabilities* (not independently-callable endpoints) are documented as schema fields, enum values, or runtime constructs, not REST paths. Examples: engine choice (L1.D), parallelism (L1.C), autoscaler signals (L1.E), streaming formats (L2.F), cache retention (L2.G), hook events (L4.H), permission policies (L4.G), multi-agent invocation (L4.I), builtin tool catalog (L4.E), guardrail attachment points (L5.I), eval maturity progression (L5.F).
3. **WebSocket endpoints** (`/v1/responses`, `/v1/mcp`, `/v1/agent/converse`, `/v1/speech-engine`, `/v1/realtime`, `/v1/realtime/translations`, `/v1/stt/stream`, `/v1/text-to-speech/stream`) are documented with `101 Switching Protocols` responses.
4. **Async jobs** uniformly follow the S.16 state machine (`queued`/`pending`/`in_progress`/`processing`/`completed`/`done`/`failed`/`expired`/`canceled`) and return the reusable `AsyncJob` schema with `202 Accepted`.
5. **Pagination** is uniformly cursor-based (`limit`/`cursor`/`order`) per S.5, via the shared `LimitParam`, `CursorParam`, `OrderParam` parameters and `ListResponse` schema.
6. **Errors** are uniformly the `Error` envelope (S.4) with shared `BadRequest`/`Unauthorized`/`Forbidden`/`NotFound`/`Conflict`/`RequestTooLarge`/`RateLimited`/`InternalError` responses.

---

> **End of `api.md`.** This OpenAPI 3.1 specification covers every independently-callable service across Layers L0–L5 and every domain described in `architecture_v2.md`, organized layer by layer and domain by domain. Transversal conventions (S.1–S.33) are factored into reusable components and applied uniformly.