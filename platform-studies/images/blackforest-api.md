# Black Forest Labs (BFL) API Analysis — Image Generation & Editing

> **Base URL:** `https://api.bfl.ai` (REST) | **Finetune server:** `https://api.us1.bfl.ai` | **Docs:** `https://docs.bfl.ai` | **Product:** `https://bfl.ai`
> **Auth:** API key (`x-key` header) | **OpenAPI:** `https://api.bfl.ai/openapi.json` | **MCP server:** `https://mcp.bfl.ai`
> **Models:** FLUX.2 (`max`, `pro`, `flex`, `klein 9B`, `klein 4B`, `dev`), FLUX.1 Kontext (`pro`, `max`), FLUX1.1 (`pro`, `pro ultra`), FLUX.1 (`dev`), FLUX.1 Fill `pro`, FLUX.1 Expand `pro`, FLUX Tools (outpainting, erase, deblur, VTO)
> **Description:** Black Forest Labs is an image-generation and editing platform exposing a REST API built on the FLUX model family. It offers text-to-image generation, prompt-driven and multi-reference image editing (up to 8 reference images via API / 10 in the playground), inpainting, outpainting/expansion, object removal, deblurring, virtual try-on, custom LoRA finetuning, and credit-based billing. **No video API is currently exposed** — a text-to-video model is under development (as of early 2026) but no public video endpoints ship in the OpenAPI spec. The API is fully asynchronous (submit → poll/webhook) with a single shared result-polling endpoint.

---

## Table of Contents

1. [Platform Overview & Core Concepts](#1-platform-overview--core-concepts)
2. [API Architecture & Capability Matrix](#2-api-architecture--capability-matrix)
3. [Auth, Rate Limits, Async Lifecycle & Pricing](#3-auth-rate-limits-async-lifecycle--pricing)
4. [FLUX.2 Text-to-Image Generation](#4-flux2-text-to-image-generation)
5. [FLUX.2 Image Editing & Multi-Reference](#5-flux2-image-editing--multi-reference)
6. [FLUX.1 Kontext — Context-Aware Edit/Create](#6-flux1-kontext--context-aware-editcreate)
7. [FLUX1.1 [pro] & Ultra/Raw Generation](#7-flux11-pro--ultraraw-generation)
8. [FLUX.1 [dev] Generation](#8-flux1-dev-generation)
9. [FLUX.1 Fill [pro] — Inpainting](#9-flux1-fill-pro--inpainting)
10. [FLUX.1 Expand [pro] — Directed Border Expansion](#10-flux1-expand-pro--directed-border-expansion)
11. [FLUX Tools — Outpainting](#11-flux-tools--outpainting)
12. [FLUX Tools — Erase (Object Removal)](#12-flux-tools--erase-object-removal)
13. [FLUX Tools — Deblur](#13-flux-tools--deblur)
14. [FLUX Tools — Virtual Try-On (VTO)](#14-flux-tools--virtual-try-on-vto)
15. [Custom LoRA Finetuning (Style/Identity)](#15-custom-lora-finetuning-styleidentity)
16. [Utility Endpoints — Credits, Results, Finetunes](#16-utility-endpoints--credits-results-finetunes)
17. [Capability Summary & Cross-Reference](#17-capability-summary--cross-reference)

---

## 1. Platform Overview & Core Concepts

### Main Concepts

BFL's API is organized around these core abstractions:

- **Model family / variant** — FLUX models are grouped by generation, each with quality/speed tiers in brackets: **FLUX.2** (`max` = highest quality + grounding search; `pro` = recommended default; `flex` = typography & fine detail; `klein 9B` / `klein 4B` = sub-second, open weights, consumer GPUs; `dev` = local dev weights) and the previous-generation **FLUX.1** (Kontext `pro`/`max`, `dev`, Fill `pro`, Expand `pro`) and **FLUX1.1** (`pro`, `pro ultra`). Each variant maps to a distinct endpoint URL (e.g. `/v1/flux-2-pro`, `/v1/flux-2-klein-4b`).
- **Text-to-image vs image-to-image** — A single FLUX.2 endpoint handles both: omit `input_image*` for pure text-to-image; supply one or more `input_image` slots for editing/multi-reference. Older FLUX.1 models split generation and editing into separate endpoints.
- **Multi-reference editing** — FLUX.2 accepts up to **8 reference images** via API (10 in the playground) through numbered `input_image` … `input_image_8` slots. The prompt describes each image's role ("person of image 1 wearing garment of image 2"). FLUX.2 [klein] supports up to 4 references. There is a combined input+output megapixel budget (e.g. [pro] = 9MP total, so at 1MP output you can use 8 refs, at 2MP output up to 7, etc.).
- **Prompt upsampling (PUP)** — Automatic prompt enhancement. FLUX.2 [pro]/[max] apply it by default; disable with `disable_pup: true` to generate from the exact prompt. Older models expose `prompt_upsampling` (default `false`); FLUX.2 [flex] defaults it to `true`.
- **Grounding search** — FLUX.2 [max] performs web searches when prompted to ground generations in real-time information (trending products, current events, scores, weather) without manually sourcing reference material.
- **Hex color control** — FLUX.2 supports hex color codes in prompts for precise brand/colour matching.
- **JSON / structured prompting** — FLUX.2 accepts JSON-structured prompts (subject, background, lighting, style, camera_angle, etc.) for production-grade control and automation.
- **Megapixel-based pricing** — FLUX.2 cost scales with output resolution (megapixels). Older FLUX.1/FLUX1.1 models use flat per-image credit pricing. 1 credit = $0.01 USD.
- **Async task lifecycle** — All generation endpoints return a task `id` and `polling_url` (or a `webhook_url`-driven async response). The caller polls `GET /v1/get_result?id=` until `status == "Ready"`, then reads `result.sample` (image URL). Image URLs are ephemeral and must be downloaded promptly.
- **Safety tolerance** — Input/output moderation strictness, 0 (most strict) → 5/6 (least strict). FLUX.2 and tools cap at 5; FLUX.1/FLUX1.1/Kontext cap at 6.
- **Finetune (LoRA)** — A custom model trained on a user's dataset. Referenced by `finetune_id` (use `org-id/lora-name` for cross-org LoRAs) and steered with `finetune_strength` (0–2, where 1.0 = full influence). Finetune endpoints live on `api.us1.bfl.ai`.
- **Safety checks** — Tasks may resolve to `Request Moderated` or `Content Moderated` statuses instead of producing an image.

### End-to-End Flows

**Standard generation/edit pipeline:**
```
prompt (+ optional input_image(s), width/height) ──▶ POST /v1/flux-2-{pro,max,flex,klein-*}
        │
        ▼  AsyncResponse {id, polling_url, cost, input_mp, output_mp}
poll loop ──▶ GET polling_url  (or GET /v1/get_result?id=)
        │
        ▼  ResultResponse {status: "Ready", result.sample: <image URL>}
download image URL (ephemeral) ──▶ store in own system
```

**Webhook pipeline:**
```
POST {…, webhook_url, webhook_secret} ──▶ AsyncWebhookResponse {id, status}
        │
        ▼  BFL POSTs final result to webhook_url  (signature verified via webhook_secret)
```

**Finetune pipeline:**
```
train LoRA (via docs workflow) ──▶ finetune_id
use: POST /v1/flux-pro-1.1-ultra-finetuned  {finetune_id, finetune_strength, prompt, …}
manage: GET /v1/my_finetunes · GET /v1/finetune_details?finetune_id= · POST /v1/delete_finetune
```

---

## 2. API Architecture & Capability Matrix

All endpoints are JSON-over-HTTP (`application/json`). There is a single shared result-polling endpoint. Finetune-management endpoints use a dedicated server (`api.us1.bfl.ai`). Image inputs are base64-encoded strings or HTTP(S) URLs; FLUX.2 also accepts a BFL-hosted `input_image_blob_path`.

| # | Surface | Endpoint / Method | Sync/Async | Purpose | Request schema |
|---|---|---|---|---|---|
| 1 | FLUX.2 [max] | `POST /v1/flux-2-max` | Async | Highest-quality generate/edit, grounding search | `Flux2Inputs` |
| 2 | FLUX.2 [pro] | `POST /v1/flux-2-pro` | Async | Recommended default generate/edit | `Flux2Inputs` |
| 3 | FLUX.2 [pro] preview | `POST /v1/flux-2-pro-preview` | Async | Latest pro improvements (use `flux-2-pro` for stable) | `Flux2Inputs` |
| 4 | FLUX.2 [flex] | `POST /v1/flux-2-flex` | Async | Typography, text rendering, small-detail preservation | `Flux2FlexInputs` |
| 5 | FLUX.2 [klein] 9B | `POST /v1/flux-2-klein-9b` | Async | Sub-second, open weights, quality/speed balance | `Flux2KleinInputs` |
| 6 | FLUX.2 [klein] 9B preview | `POST /v1/flux-2-klein-9b-preview` | Async | Latest klein improvements | `Flux2KleinInputs` |
| 7 | FLUX.2 [klein] 4B | `POST /v1/flux-2-klein-4b` | Async | Fastest, lightest, consumer-GPU open weights | `Flux2KleinInputs` |
| 8 | FLUX.1 Kontext [max] | `POST /v1/flux-kontext-max` | Async | Context-aware create/edit, max quality (legacy) | `FluxKontextProInputs` |
| 9 | FLUX.1 Kontext [pro] | `POST /v1/flux-kontext-pro` | Async | Context-aware create/edit (legacy; prefer FLUX.2) | `FluxKontextProInputs` |
| 10 | FLUX1.1 [pro] | `POST /v1/flux-pro-1.1` | Async | Fast, reliable text-to-image | `FluxPro11Inputs` |
| 11 | FLUX1.1 [pro] ultra | `POST /v1/flux-pro-1.1-ultra` | Async | Up to 4MP, optional Raw mode, image remix | `FluxUltraInput` |
| 12 | FLUX.1 [dev] | `POST /v1/flux-dev` | Async | Open-weights dev generation | `FluxDevInputs` |
| 13 | FLUX.1 Fill [pro] | `POST /v1/flux-pro-1.0-fill` | Async | Text-driven inpainting (image + mask) | `FluxProFillInputs` |
| 14 | FLUX.1 Expand [pro] | `POST /v1/flux-pro-1.0-expand` | Async | Add pixels on chosen sides | `FluxProExpandInputs` |
| 15 | Fill [pro] finetuned | `POST /v1/flux-pro-1.0-fill-finetuned` | Async | Inpaint with custom LoRA (`api.us1.bfl.ai`) | `FinetuneFluxProFillInputs` |
| 16 | Ultra finetuned | `POST /v1/flux-pro-1.1-ultra-finetuned` | Async | Ultra generation with custom LoRA (`api.us1.bfl.ai`) | `FinetuneFluxUltraInput` |
| 17 | Outpainting | `POST /v1/flux-tools/outpainting-v1` | Async | Extend image onto a canvas | `FluxOutpaintingInputs` |
| 18 | Erase | `POST /v1/flux-tools/erase-v1` | Async | Remove masked object/region | `Flux2EraseInputs` |
| 19 | Deblur | `POST /v1/flux-tools/deblur-v1` | Async | Sharpen blurry image, no prompt | `Flux2DeblurInputs` |
| 20 | Virtual Try-On | `POST /v1/flux-tools/vto-v1` | Async | Person + garment → try-on | `Flux2KleinTryonInputs` |
| 21 | Get result | `GET /v1/get_result?id=` | Poll | Retrieve task status + image URL | — |
| 22 | Get credits | `GET /v1/credits` | Sync | Current credit balance | — |
| 23 | My finetunes | `GET /v1/my_finetunes` | Sync | List user's LoRAs (`api.us1.bfl.ai`) | — |
| 24 | Finetune details | `GET /v1/finetune_details?finetune_id=` | Sync | Training params/metadata (`api.us1.bfl.ai`) | — |
| 25 | Delete finetune | `POST /v1/delete_finetune` | Sync | Delete a LoRA (`api.us1.bfl.ai`) | `DeleteFinetuneInputs` |

**Reference-image capacity per model:**

| Model | Reference images (API) | Reference images (Playground) |
|---|---|---|
| FLUX.2 [max] / [pro] / [flex] | up to 8 | up to 10 |
| FLUX.2 [klein] 9B / 4B | up to 4 | — |
| FLUX.2 [dev] | recommended max 6 | — |
| FLUX.1 Kontext [pro]/[max] | 1 (input_image); input_image_2–4 experimental multiref | — |

---

## 3. Auth, Rate Limits, Async Lifecycle & Pricing

### Authentication
- Header: `x-key: <BFL_API_KEY>` on every request. Keys are created in the user profile, organized under Organizations & Projects for access control.

### Endpoints (servers)
- `https://api.bfl.ai` — primary global endpoint; routes across all clusters with automatic failover. **Always use the `polling_url` returned in responses** when using this endpoint.
- Regional multi-cluster endpoints: `https://api.eu.bfl.ai` (Europe), `https://api.us.bfl.ai` (US) — pinned to a region; the polling URL may differ.
- `https://api.us1.bfl.ai` — dedicated **Finetune API** server for finetune management and finetuned inference endpoints.

### Rate limits
- Maximum **24 concurrent requests** for most endpoints.
- Maximum **6 concurrent requests** for `flux-kontext-max`.
- Implement exponential backoff on HTTP `429`.

### Async lifecycle
1. `POST` a generation/edit request → `AsyncResponse` (`{id, polling_url, cost, input_mp, output_mp}`) or `AsyncWebhookResponse` (`{id, status, webhook_url, …}`) when `webhook_url` is supplied.
2. Poll `GET polling_url` (or `GET /v1/get_result?id=<id>`) every ~0.5–1s.
3. `ResultResponse.status` transitions: `Pending` → `Reasoning` → `Generating` → `Ready` (or `Request Moderated` / `Content Moderated` / `Error` / `Task not found`).
4. On `Ready`, read `result.sample` for the image URL. **Download immediately** — URLs are ephemeral and expire.

### Webhooks
- Supply `webhook_url` (HTTPS, ≤2083 chars) and optional `webhook_secret` for signature verification. BFL POSTs the final result to the webhook instead of requiring polling.

### Output formats
- `OutputFormat` enum: `jpeg` (default for most), `png` (default for outpainting/erase/deblur/Kontext), `webp`.

### Pricing (credit-based; 1 credit = $0.01 USD)
- **FLUX.2** uses megapixel-based pricing (cost scales with output resolution):
  - [klein] 4B: from $0.014/image (t2i & editing)
  - [klein] 9B: from $0.015/image
  - [pro]: from $0.03/image
  - [flex] / [max]: higher tier (see pricing page)
- **FLUX.1 / FLUX1.1** flat per-image:
  - FLUX1.1 [pro]: 4 credits ($0.04) · Ultra/Raw: 6 credits ($0.06)
  - FLUX.1 Kontext [pro]: 4 credits · [max]: 8 credits
  - FLUX.1 Fill [pro]: 5 credits ($0.05)
- **Batch**: cost × number of images.
- **Tools**: priced per task (deblur/erase/outpaint/VTO — see pricing page).

---

## 4. FLUX.2 Text-to-Image Generation

### Concept
FLUX.2 is the recommended model for text-to-image: high-fidelity generation with accurate hands/faces/textures, exact hex-color steering, JSON-structured prompting, flexible aspect ratios, and up to 4MP output. [max] adds grounding search for real-time information. The same endpoints double as editing endpoints when `input_image*` is supplied (see §5).

### Endpoints & schemas
- `POST /v1/flux-2-max` → `Flux2Inputs` (highest quality, grounding search)
- `POST /v1/flux-2-pro` → `Flux2Inputs` (recommended default)
- `POST /v1/flux-2-pro-preview` → `Flux2Inputs` (latest improvements; use `flux-2-pro` for stable)
- `POST /v1/flux-2-flex` → `Flux2FlexInputs` (typography, text rendering, small details)
- `POST /v1/flux-2-klein-9b` · `/v1/flux-2-klein-9b-preview` · `/v1/flux-2-klein-4b` → `Flux2KleinInputs` (sub-second, open weights, max 4 refs)

### `Flux2Inputs` (max / pro / pro-preview)
Required: `prompt`

| Parameter | Type | Required | Default | Constraints | Description |
|---|---|---|---|---|---|
| `prompt` | string | **yes** | — | — | Text prompt for image generation. |
| `disable_pup` | boolean | no | `false` | — | Disable automatic prompt upsampling. [pro]/[max] apply PUP by default; set `true` to use the prompt exactly as written. |
| `input_image` | string\|null | no | — | base64 or URL | First reference image (enables editing mode). |
| `input_image_2` … `input_image_8` | string\|null | no | — | base64 or URL | Additional reference images (up to 8 via API). |
| `seed` | integer\|null | no | — | — | Optional seed for reproducibility. |
| `width` | integer\|null | no | `0` | min 64 | Output width (px). `0` lets the model choose. |
| `height` | integer\|null | no | `0` | min 64 | Output height (px). |
| `safety_tolerance` | integer | no | `2` | 0–5 | Moderation strictness (0 strict, 5 least). |
| `output_format` | `OutputFormat`\|null | no | `jpeg` | jpeg/png/webp | — |
| `webhook_url` | string\|null (uri) | no | — | len 1–2083 | Webhook for async delivery. |
| `webhook_secret` | string\|null | no | — | — | Webhook signature verification secret. |

### `Flux2FlexInputs` (flex)
Required: `prompt`. Adds explicit diffusion-control params not present in `Flux2Inputs`:

| Parameter | Type | Required | Default | Constraints | Description |
|---|---|---|---|---|---|
| `prompt` | string | **yes** | — | — | Text prompt. |
| `prompt_upsampling` | boolean\|null | no | `true` | — | Whether to use prompt upsampling (default **on** for flex). |
| `input_image` … `input_image_8` | string\|null | no | — | base64/URL/blob | Reference images (up to 8). |
| `input_image_blob_path` | string\|null | no | — | — | BFL-hosted blob path to the input image. |
| `seed` | integer\|null | no | — | — | Reproducibility seed. |
| `width` | integer\|null | no | `0` | min 64 | Output width. |
| `height` | integer\|null | no | `0` | min 64 | Output height. |
| `guidance` | number\|null | no | `5.0` | 1.5–10 | Guidance scale; higher improves prompt adherence, reduces realism. |
| `steps` | integer\|null | no | `50` | 1–50 | Diffusion steps; higher = more detailed/realistic. |
| `safety_tolerance` | integer | no | `2` | 0–5 | Moderation strictness. |
| `output_format` | `OutputFormat`\|null | no | `jpeg` | jpeg/png/webp | — |
| `webhook_url` / `webhook_secret` | string\|null | no | — | — | Webhook delivery. |

### `Flux2KleinInputs` (klein 4B / 9B / 9B preview)
Schema note: *"like Pro but max 4 images."* Required: `prompt`. No `disable_pup` (klein does not auto-upsample), no `guidance`/`steps` knobs.

| Parameter | Type | Required | Default | Constraints | Description |
|---|---|---|---|---|---|
| `prompt` | string | **yes** | — | — | Text prompt. |
| `input_image` … `input_image_4` | string\|null | no | — | base64/URL | Up to 4 reference images. |
| `seed` | integer\|null | no | — | — | Reproducibility seed. |
| `width` | integer\|null | no | `0` | min 64 | Output width. |
| `height` | integer\|null | no | `0` | min 64 | Output height. |
| `safety_tolerance` | integer | no | `2` | 0–5 | Moderation strictness. |
| `output_format` | `OutputFormat`\|null | no | `jpeg` | jpeg/png/webp | — |
| `webhook_url` / `webhook_secret` | string\|null | no | — | — | Webhook delivery. |

### Example
```bash
# Submit
curl -X POST "https://api.bfl.ai/v1/flux-2-pro" \
  -H "x-key: $BFL_API_KEY" -H "Content-Type: application/json" \
  -d '{"prompt":"A serene mountain landscape at golden hour","width":1440,"height":810}'
# → {"id":"...","polling_url":"https://...","cost":3.0,"output_mp":1.17}

# Poll
curl -s "$polling_url" -H "x-key: $BFL_API_KEY"   # until status == "Ready"
```

---

## 5. FLUX.2 Image Editing & Multi-Reference

### Concept
FLUX.2 unifies generation and editing: supplying `input_image*` slots turns the same text-to-image endpoint into an image editor. Describe the desired change in natural language (swap backgrounds, replace objects, transfer styles, adjust lighting) and FLUX.2 produces a photorealistic result while preserving identity/composition as needed. Multi-reference combines elements from several source images into one coherent output (fashion composites, interior design, product scenes, character consistency). [max] gives highest editing consistency; [pro] for production at scale; [flex] for fine detail/typography; [klein] for cost-efficient high-volume editing.

### How editing differs from generation
- Same endpoint/schema as §4; the presence of `input_image` (and optionally more `input_image_2..8`) switches the workflow to image-to-image editing.
- The prompt should reference images by ordinal ("the person of image 1 wearing the garments of image 2") so the model knows what to pull from each slot.
- **Megapixel budget**: [pro] API has a 9MP total limit for input + output. At 1MP output → up to 8 references; at 2MP output → up to 7; etc. [klein] supports up to 4 references regardless.
- Supported: background swap, object replacement, style/material transfer, logo & branding placement, character/product consistency across edits, drawing-to-image rendering, interior redesign, lighting/weather transformation, person-in-scene compositing, object removal.

### Parameters
Identical to the corresponding generation schema in §4 (`Flux2Inputs` / `Flux2FlexInputs` / `Flux2KleinInputs`). The key editing inputs are the `input_image*` slots; `prompt` describes the edit. For exact-prompt editing on [pro]/[max], set `disable_pup: true`.

### Common editing pipelines
```
single-ref edit:   input_image + prompt ──▶ /v1/flux-2-pro ──▶ edited image
multi-ref compose: input_image(1..8) + prompt (roles) ──▶ /v1/flux-2-pro ──▶ composite
style transfer:    content image + style image + prompt ──▶ /v1/flux-2-max ──▶ restyled image
```

---

## 6. FLUX.1 Kontext — Context-Aware Edit/Create

### Concept
FLUX.1 Kontext combines text-to-image generation with context-aware image editing using text + a single image (multiref `input_image_2..4` is experimental). BFL recommends FLUX.2 [pro]/[flex] for new editing projects; Kontext remains for create-and-edit workflows that need its specific behavior. [max] = higher quality; [pro] = standard.

### Endpoints & schema
- `POST /v1/flux-kontext-max` → `FluxKontextProInputs`
- `POST /v1/flux-kontext-pro` → `FluxKontextProInputs`

### `FluxKontextProInputs`
Required: `prompt`

| Parameter | Type | Required | Default | Constraints | Description |
|---|---|---|---|---|---|
| `prompt` | string | **yes** | — | — | Text prompt for generation/editing. |
| `input_image` | string\|null | no | — | base64 or URL | Base image to edit with Kontext. |
| `input_image_2` … `input_image_4` | string\|null | no | — | base64 or URL | **Experimental Multiref** additional references. |
| `seed` | integer\|null | no | — | — | Reproducibility seed. |
| `aspect_ratio` | string\|null | no | — | between 21:9 and 9:21 | Output aspect ratio. |
| `output_format` | `OutputFormat`\|null | no | `png` | jpeg/png/webp | — |
| `prompt_upsampling` | boolean | no | `false` | — | Auto prompt enhancement. |
| `safety_tolerance` | integer | no | `2` | 0–6 | Moderation strictness (FLUX.1 range). |
| `webhook_url` / `webhook_secret` | string\|null | no | — | — | Webhook delivery. |

> Note: Kontext uses `safety_tolerance` 0–6 and defaults `output_format` to `png`, unlike FLUX.2's 0–5 / `jpeg` defaults.

---

## 7. FLUX1.1 [pro] & Ultra/Raw Generation

### Concept
FLUX1.1 [pro] is the fast, reliable text-to-image baseline (6× faster than FLUX.1 [pro]). **Ultra** mode generates up to 4MP and adds an optional **Raw** mode for less-processed, candid-photography aesthetics; ultra also supports image remix via `image_prompt` + `image_prompt_strength`. Both support finetuned variants via `api.us1.bfl.ai`.

### Endpoints & schemas
- `POST /v1/flux-pro-1.1` → `FluxPro11Inputs`
- `POST /v1/flux-pro-1.1-ultra` → `FluxUltraInput`
- `POST /v1/flux-pro-1.1-ultra-finetuned` (server `api.us1.bfl.ai`) → `FinetuneFluxUltraInput`

### `FluxPro11Inputs`
All optional (no required fields).

| Parameter | Type | Required | Default | Constraints | Description |
|---|---|---|---|---|---|
| `prompt` | string\|null | no | `""` | — | Text prompt. |
| `image_prompt` | string\|null | no | — | base64 | Optional base64 image to use with FLUX Redux. |
| `width` | integer | no | `1024` | 256–1440, multiple of 32 | Output width. |
| `height` | integer | no | `768` | 256–1440, multiple of 32 | Output height. |
| `prompt_upsampling` | boolean | no | `false` | — | Auto prompt enhancement. |
| `seed` | integer\|null | no | — | — | Reproducibility seed. |
| `safety_tolerance` | integer | no | `2` | 0–6 | Moderation strictness. |
| `output_format` | `OutputFormat`\|null | no | `jpeg` | jpeg/png/webp | — |
| `webhook_url` / `webhook_secret` | string\|null | no | — | — | Webhook delivery. |

### `FluxUltraInput`
All optional. Uses `aspect_ratio` instead of explicit width/height.

| Parameter | Type | Required | Default | Constraints | Description |
|---|---|---|---|---|---|
| `prompt` | string\|null | no | `""` | — | Prompt for generation. |
| `prompt_upsampling` | boolean | no | `false` | — | Auto prompt enhancement. |
| `seed` | integer\|null | no | — | — | Reproducibility seed. |
| `aspect_ratio` | string | no | `16:9` | 21:9 … 9:21 | Output aspect ratio. |
| `raw` | boolean | no | `false` | — | Generate less-processed, more natural-looking images (candid photography aesthetic). |
| `image_prompt` | string\|null | no | — | base64 | Optional image to remix. |
| `image_prompt_strength` | number | no | `0.1` | 0–1 | Blend between the text prompt and the image prompt. |
| `safety_tolerance` | integer | no | `2` | 0–6 | Moderation strictness. |
| `output_format` | `OutputFormat`\|null | no | `jpeg` | jpeg/png/webp | — |
| `webhook_url` / `webhook_secret` | string\|null | no | — | — | Webhook delivery. |

### `FinetuneFluxUltraInput` (server `api.us1.bfl.ai`)
Required: `finetune_id`. Same fields as `FluxUltraInput` plus:

| Parameter | Type | Required | Default | Constraints | Description |
|---|---|---|---|---|---|
| `finetune_id` | string | **yes** | — | — | LoRA name; cross-org format `org-id/lora-name`. |
| `finetune_strength` | number | no | `1.2` | 0–2 | LoRA influence (0 none, 1.0 full, up to 2.0). |

---

## 8. FLUX.1 [dev] Generation

### Concept
FLUX.1 [dev] is the open-weights development model for text-to-image generation with explicit diffusion controls (steps, guidance).

### Endpoint & schema
- `POST /v1/flux-dev` → `FluxDevInputs`

### `FluxDevInputs`
All optional.

| Parameter | Type | Required | Default | Constraints | Description |
|---|---|---|---|---|---|
| `prompt` | string | no | `""` | — | Text prompt. |
| `image_prompt` | string\|null | no | — | base64 | Optional image to use as a prompt. |
| `width` | integer | no | `1024` | 256–1440, multiple of 32 | Output width. |
| `height` | integer | no | `768` | 256–1440, multiple of 32 | Output height. |
| `steps` | integer\|null | no | `28` | 1–50 | Diffusion steps. |
| `prompt_upsampling` | boolean | no | `false` | — | Auto prompt enhancement. |
| `seed` | integer\|null | no | — | — | Reproducibility seed. |
| `guidance` | number\|null | no | `3.0` | 1.5–5.0 | Guidance scale; higher improves prompt adherence, reduces realism. |
| `safety_tolerance` | integer | no | `2` | 0–6 | Moderation strictness. |
| `output_format` | `OutputFormat`\|null | no | `jpeg` | jpeg/png/webp | — |
| `webhook_url` / `webhook_secret` | string\|null | no | — | — | Webhook delivery. |

---

## 9. FLUX.1 Fill [pro] — Inpainting

### Concept
FLUX.1 Fill is a specialized inpainting model for two uses: (a) **selecting specific regions** within an image and editing them while preserving surrounding context (change objects, enhance details, remove elements), and (b) **adding new pixels at borders** to extend images / change aspect ratios. The edit region is defined either by a separate black/white mask (white = inpaint, black = preserve) or by the alpha channel of a PNG/WebP (transparent = inpaint).

### Endpoints & schemas
- `POST /v1/flux-pro-1.0-fill` → `FluxProFillInputs`
- `POST /v1/flux-pro-1.0-fill-finetuned` (server `api.us1.bfl.ai`) → `FinetuneFluxProFillInputs`

### `FluxProFillInputs`
Required: `image`

| Parameter | Type | Required | Default | Constraints | Description |
|---|---|---|---|---|---|
| `image` | string | **yes** | — | base64 | Image to modify; may contain an alpha mask. |
| `mask` | string\|null | no | — | base64, same dims | Black/white mask: white = inpaint, black = preserve. Optional if alpha mask is embedded in the image. Dimensions validated against the image. |
| `prompt` | string\|null | no | `""` | — | Description of changes for the masked area. |
| `steps` | integer\|null | no | `50` | 15–50 | Diffusion steps. |
| `prompt_upsampling` | boolean\|null | no | `false` | — | Auto prompt enhancement. |
| `seed` | integer\|null | no | — | — | Reproducibility seed. |
| `guidance` | number\|null | no | `60` | 1.5–100 | Guidance strength (Fill uses a much higher range than dev). |
| `output_format` | `OutputFormat`\|null | no | `jpeg` | jpeg/png/webp | — |
| `safety_tolerance` | integer | no | `2` | 0–6 | Moderation strictness. |
| `webhook_url` / `webhook_secret` | string\|null | no | — | — | Webhook delivery. |

### `FinetuneFluxProFillInputs` (server `api.us1.bfl.ai`)
Required: `finetune_id`, `image`. Same fields as `FluxProFillInputs` plus:

| Parameter | Type | Required | Default | Constraints | Description |
|---|---|---|---|---|---|
| `finetune_id` | string | **yes** | — | — | LoRA name; cross-org format `org-id/lora-name`. |
| `finetune_strength` | number | no | `1.1` | 0–2 | LoRA influence (0 none, 1.0 full, up to 2.0). |

> Note: the finetuned variant makes `prompt` required (string, not nullable) and changes `mask`/`steps`/`guidance`/`prompt_upsampling` to non-nullable with the same defaults.

---

## 10. FLUX.1 Expand [pro] — Directed Border Expansion

### Concept
Unlike free outpainting (§11), FLUX.1 Expand adds a specific number of pixels to chosen sides (top/bottom/left/right) while maintaining context — precise, directional canvas growth rather than placing an image on a larger canvas.

### Endpoint & schema
- `POST /v1/flux-pro-1.0-expand` → `FluxProExpandInputs`

### `FluxProExpandInputs`
Required: `image`

| Parameter | Type | Required | Default | Constraints | Description |
|---|---|---|---|---|---|
| `image` | string | **yes** | — | base64 | Image to expand. |
| `top` | integer\|null | no | `0` | 0–2048 | Pixels to add at the top. |
| `bottom` | integer\|null | no | `0` | 0–2048 | Pixels to add at the bottom. |
| `left` | integer\|null | no | `0` | 0–2048 | Pixels to add on the left. |
| `right` | integer\|null | no | `0` | 0–2048 | Pixels to add on the right. |
| `prompt` | string\|null | no | `""` | — | Description guiding the expanded areas. |
| `steps` | integer\|null | no | `50` | 15–50 | Diffusion steps. |
| `prompt_upsampling` | boolean\|null | no | `false` | — | Auto prompt enhancement. |
| `seed` | integer\|null | no | — | — | Reproducibility seed. |
| `guidance` | number\|null | no | `60` | 1.5–100 | Guidance strength. |
| `output_format` | `OutputFormat`\|null | no | `jpeg` | jpeg/png/webp | — |
| `safety_tolerance` | integer | no | `2` | 0–6 | Moderation strictness. |
| `webhook_url` / `webhook_secret` | string\|null | no | — | — | Webhook delivery. |

---

## 11. FLUX Tools — Outpainting

### Concept
Prompt-free outpainting: the input image is placed on a `(width, height)` canvas at a given offset, and the surrounding region is generated by a FLUX outpainting model to extend the scene beyond its borders coherently. This is a tools endpoint (no finetune, no multi-ref) optimized for scene extension.

### Endpoint & schema
- `POST /v1/flux-tools/outpainting-v1` → `FluxOutpaintingInputs` (`additionalProperties: false`)

### `FluxOutpaintingInputs`
Required: `input_image`, `width`, `height`

| Parameter | Type | Required | Default | Constraints | Enum | Description |
|---|---|---|---|---|---|---|
| `input_image` | string | **yes** | — | base64 or URL | — | Reference image to extend. |
| `width` | integer | **yes** | — | min 64 | — | Target output canvas width. |
| `height` | integer | **yes** | — | min 64 | — | Target output canvas height. |
| `auto_crop` | boolean | no | `false` | — | — | If true, crop input to canvas bounds when it exceeds edges; if false, error instead. |
| `reference_offset_x` | integer\|null | no | — | — | — | Left offset (px) of input's top-left corner on the canvas; negative allowed; None = center horizontally. |
| `reference_offset_y` | integer\|null | no | — | — | — | Top offset (px); negative allowed; None = center vertically. |
| `mode` | string | no | `high` | — | `high`, `fast` | Quality/speed trade-off. `high` = highest fidelity (slower). `fast` = significantly faster, good for landscapes/backgrounds/textures/products, lower fidelity in extended region. |
| `prompt` | string\|null | no | — | — | — | **Experimental** optional text guidance for the outpainted region; model may not follow it strictly. Leave unset for default behavior. |
| `safety_tolerance` | integer | no | `2` | 0–5 | — | Moderation strictness. |
| `output_format` | `OutputFormat`\|null | no | `png` | jpeg/png/webp | — | — |
| `webhook_url` / `webhook_secret` | string\|null | no | — | — | — | Webhook delivery. |

---

## 12. FLUX Tools — Erase (Object Removal)

### Concept
Mask-driven object removal: supply an image plus a black/white mask (white = object to remove, black = preserve, same dimensions). The model removes the masked object and fills the surrounding background coherently. Powered by FLUX.2 Klein 9B. Optional `dilate_pixels` expands the mask to cover object edges.

### Endpoint & schema
- `POST /v1/flux-tools/erase-v1` → `Flux2EraseInputs`

### `Flux2EraseInputs`
Required: `image`, `mask`

| Parameter | Type | Required | Default | Constraints | Description |
|---|---|---|---|---|---|
| `image` | string | **yes** | — | base64 or URL | Input image. |
| `mask` | string | **yes** | — | base64 or URL, same dims | Black/white mask; white = object to remove, black = preserve. Must match image dimensions. |
| `dilate_pixels` | integer | no | `10` | 0–25 | Pixels to dilate the mask before removal (covers object edges). |
| `seed` | integer\|null | no | — | — | Reproducibility seed. |
| `safety_tolerance` | integer | no | `2` | 0–5 | Moderation strictness. |
| `output_format` | `OutputFormat`\|null | no | `png` | jpeg/png/webp | — |
| `webhook_url` / `webhook_secret` | string\|null | no | — | — | Webhook delivery. |

---

## 13. FLUX Tools — Deblur

### Concept
Sharpen a blurry image while preserving the original scene, objects, composition, and lighting. **No prompt, no mask** — the whole image is regenerated by a fixed FLUX.2 Klein 9B KV BF16 blur-removal LoRA with a fixed prompt; the caller controls only the input image (plus seed/format/safety).

### Endpoint & schema
- `POST /v1/flux-tools/deblur-v1` → `Flux2DeblurInputs`

### `Flux2DeblurInputs`
Required: `image`

| Parameter | Type | Required | Default | Constraints | Description |
|---|---|---|---|---|---|
| `image` | string | **yes** | — | base64 or URL | Input (blurry) image. |
| `seed` | integer\|null | no | — | — | Reproducibility seed. |
| `safety_tolerance` | integer | no | `2` | 0–5 | Moderation strictness. |
| `output_format` | `OutputFormat`\|null | no | `png` | jpeg/png/webp | — |
| `webhook_url` / `webhook_secret` | string\|null | no | — | — | Webhook delivery. |

---

## 14. FLUX Tools — Virtual Try-On (VTO)

### Concept
Generate virtual try-on results from a **person** image plus one or more **garment** reference images. Optimized for low latency (interactive fitting rooms, social filters). The endpoint maps `person` → `input_image` and `garment` → `input_image_2` internally; the prompt steers attribute transfer (core formula: `The person of image 1, [wearing/holding/...] the [garment] of image 2`). Garment references can be packshots, multi-garment outfit composites, or on-model imagery.

### Endpoint & schema
- `POST /v1/flux-tools/vto-v1` → `Flux2KleinTryonInputs` (built on FLUX.2 Klein)

### `Flux2KleinTryonInputs`
Required: `prompt`, `person`, `garment`

| Parameter | Type | Required | Default | Constraints | Description |
|---|---|---|---|---|---|
| `prompt` | string | **yes** | — | — | Text prompt for VTO (e.g. `TRY-ON: The person of image 1 wearing the garments of image 2.`). |
| `person` | string | **yes** | — | base64 or URL | Person image (maps to `input_image`). |
| `garment` | string | **yes** | — | base64 or URL | Garment image(s) (maps to `input_image_2`). |
| `seed` | integer\|null | no | — | — | Reproducibility seed. |
| `safety_tolerance` | integer | no | `2` | 0–5 | Moderation strictness (public use). |
| `output_format` | `OutputFormat`\|null | no | `jpeg` | jpeg/png/webp | — |
| `webhook_url` / `webhook_secret` | string\|null | no | — | — | Webhook delivery. |

---

## 15. Custom LoRA Finetuning (Style/Identity)

### Concept
Train a custom LoRA on a user dataset (e.g. a style, character, or product) and invoke it through the finetuned inference endpoints. LoRAs are referenced by `finetune_id` (use `org-id/lora-name` for public/shared LoRAs from other organizations) and steered by `finetune_strength` (0 = no influence, 1.0 = full influence, up to 2.0). Training itself is handled via the documented FLUX.2 [klein] training workflow; the API exposes management + inference endpoints (all on `api.us1.bfl.ai`).

### Inference endpoints
- `POST /v1/flux-pro-1.1-ultra-finetuned` → `FinetuneFluxUltraInput` (ultra generation + LoRA; `finetune_strength` default `1.2`)
- `POST /v1/flux-pro-1.0-fill-finetuned` → `FinetuneFluxProFillInputs` (inpaint + LoRA; `finetune_strength` default `1.1`)

Both add `finetune_id` (required) and `finetune_strength` to the base schemas described in §7 and §9.

### Management endpoints (server `api.us1.bfl.ai`)
- `GET /v1/my_finetunes` → `MyFinetunesResponse { finetunes: [...] }` — list all user LoRAs.
- `GET /v1/finetune_details?finetune_id=<id>` → `FinetuneDetailResponse { finetune_details: {...} }` — training params & metadata.
- `POST /v1/delete_finetune` (body `DeleteFinetuneInputs { finetune_id }`) → `DeleteFinetuneResponse { status, message, deleted_finetune_id, timestamp }`.

---

## 16. Utility Endpoints — Credits, Results, Finetunes

### `GET /v1/credits` — Get the user's credits
- **Response:** `CreditsResponse { credits: number }` — current credit balance.

### `GET /v1/get_result?id=<id>` — Get Result (shared polling)
- **Query:** `id` (string, required) — task id.
- **Response:** `ResultResponse`

| Field | Type | Required | Description |
|---|---|---|---|
| `id` | string | yes | Task id. |
| `status` | `StatusResponse` (enum) | yes | `Task not found` · `Pending` · `Reasoning` · `Generating` · `Request Moderated` · `Content Moderated` · `Ready` · `Error` |
| `result` | any\|null | no | On `Ready`, contains `result.sample` (image URL). |
| `progress` | number\|null | no | Progress fraction. |
| `details` | object\|null | no | Extra details. |
| `preview` | object\|null | no | Preview data. |

### Async response shapes (returned by all generation endpoints)
**`AsyncResponse`** (polling mode): `id` (req), `polling_url` (req), `cost` (credits, nullable), `input_mp`, `output_mp` (nullable, 2 decimal places).
**`AsyncWebhookResponse`** (when `webhook_url` supplied): `id` (req), `status` (req), `webhook_url` (req), plus `cost`/`input_mp`/`output_mp`.

---

## 17. Capability Summary & Cross-Reference

| Capability | Recommended endpoint(s) | Schema | Key parameters | Notes |
|---|---|---|---|---|
| Text-to-image (general) | `POST /v1/flux-2-pro` | `Flux2Inputs` | `prompt`, `width`, `height`, `disable_pup` | Default model. Up to 4MP. |
| Text-to-image (max quality + grounding) | `POST /v1/flux-2-max` | `Flux2Inputs` | `prompt`, grounding via prompt | Web search for real-time info. |
| Text-to-image (typography/text) | `POST /v1/flux-2-flex` | `Flux2FlexInputs` | `prompt`, `guidance`, `steps` | Best for text in images; PUP default on. |
| Text-to-image (fast/open weights) | `POST /v1/flux-2-klein-{4b,9b}` | `Flux2KleinInputs` | `prompt`, up to 4 refs | Sub-second; consumer GPUs. |
| Image editing (single/multi-ref) | `POST /v1/flux-2-{pro,max,flex}` | `Flux2*Inputs` | `input_image` … `input_image_8`, `prompt` | Up to 8 refs (API); 9MP budget on [pro]. |
| Context-aware create/edit (legacy) | `POST /v1/flux-kontext-{pro,max}` | `FluxKontextProInputs` | `prompt`, `input_image` (+2–4 exp.) | Prefer FLUX.2 for new projects. |
| High-res / candid (ultra + raw) | `POST /v1/flux-pro-1.1-ultra` | `FluxUltraInput` | `prompt`, `aspect_ratio`, `raw`, `image_prompt` | Up to 4MP. |
| Fast baseline generation | `POST /v1/flux-pro-1.1` | `FluxPro11Inputs` | `prompt`, `width`, `height` | 6× faster than FLUX.1 [pro]. |
| Open-weights dev generation | `POST /v1/flux-dev` | `FluxDevInputs` | `prompt`, `steps`, `guidance` | Local dev weights. |
| Inpainting (region edit) | `POST /v1/flux-pro-1.0-fill` | `FluxProFillInputs` | `image`, `mask`/alpha, `prompt`, `guidance` | White = inpaint. Guidance 1.5–100. |
| Inpainting + custom LoRA | `POST /v1/flux-pro-1.0-fill-finetuned` | `FinetuneFluxProFillInputs` | + `finetune_id`, `finetune_strength` | Server `api.us1.bfl.ai`. |
| Directed border expansion | `POST /v1/flux-pro-1.0-expand` | `FluxProExpandInputs` | `image`, `top/bottom/left/right` | Per-side pixel counts (0–2048). |
| Outpainting (canvas extension) | `POST /v1/flux-tools/outpainting-v1` | `FluxOutpaintingInputs` | `input_image`, `width`, `height`, `mode` | `high`/`fast`; optional offset & prompt. |
| Object removal | `POST /v1/flux-tools/erase-v1` | `Flux2EraseInputs` | `image`, `mask`, `dilate_pixels` | White = remove; mask dilation 0–25. |
| Deblur (sharpen) | `POST /v1/flux-tools/deblur-v1` | `Flux2DeblurInputs` | `image` only | No prompt/mask; fixed Klein 9B LoRA. |
| Virtual try-on | `POST /v1/flux-tools/vto-v1` | `Flux2KleinTryonInputs` | `person`, `garment`, `prompt` | Low-latency; prompt steers transfer. |
| Ultra + custom LoRA | `POST /v1/flux-pro-1.1-ultra-finetuned` | `FinetuneFluxUltraInput` | + `finetune_id`, `finetune_strength` | Server `api.us1.bfl.ai`. |
| Poll result | `GET /v1/get_result?id=` | — | `id` | Shared by all tasks. |
| Credit balance | `GET /v1/credits` | — | — | 1 credit = $0.01. |
| LoRA management | `GET /v1/my_finetunes` · `GET /v1/finetune_details` · `POST /v1/delete_finetune` | — | `finetune_id` | Server `api.us1.bfl.ai`. |

### Cross-cutting parameter conventions
- **Image inputs**: base64-encoded strings or HTTP(S) URLs across all endpoints; FLUX.2 [flex] also accepts `input_image_blob_path` (BFL-hosted blob).
- **Reproducibility**: `seed` (integer, optional) on all generation/edit/tools endpoints.
- **Moderation**: `safety_tolerance` — 0–5 on FLUX.2 & tools; 0–6 on FLUX.1/FLUX1.1/Kontext. Default `2`.
- **Output format**: `OutputFormat` enum (`jpeg`, `png`, `webp`); defaults vary (`jpeg` for generation, `png` for tools/Kontext).
- **Webhooks**: `webhook_url` + optional `webhook_secret` on every generation/edit endpoint; switches response from `AsyncResponse` to `AsyncWebhookResponse`.
- **No negative prompts**: FLUX models do not use negative prompts; control is via prompt structure, guidance, and reference images.
- **No video API**: No video generation/understanding endpoints exist in the current OpenAPI spec (a text-to-video model is under development but not publicly exposed).