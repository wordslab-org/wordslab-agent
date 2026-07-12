# Ideogram API Analysis — Image Understanding & Generation

> **Base URL:** `https://api.ideogram.ai` (REST) | **Docs:** `https://developer.ideogram.ai/ideogram-api/api-overview` | **Learn:** `https://ideogram.ai/api-learn/`
> **Auth:** API key (`Api-Key` header) | **OpenAPI:** `https://developer.ideogram.ai/openapi.json` | **MCP:** `https://developer.ideogram.ai/_mcp/server`
> **Models:** Ideogram 4.0 (latest), Ideogram 3.0 (broad endpoint coverage); open weights on GitHub/Hugging Face (`ideogram-oss/ideogram-4`)
> **Description:** Ideogram is an image-generation-first AI platform exposing a REST API for text-to-image generation, prompt-based editing, image remixing, reframing, background removal/replacement, transparent-background generation, layerized text extraction, upscaling, image description (vision), custom model training, and magic-prompt enhancement. Its distinguishing strength is **crystal-clear text rendering** inside images (logos, posters, branding, print-on-demand). The platform also offers an MCP server for AI-client integration and publishes open model weights.

---

## Table of Contents

1. [Platform Overview & Core Concepts](#1-platform-overview--core-concepts)
2. [API Architecture & Capability Matrix](#2-api-architecture--capability-matrix)
3. [Authentication, Rate Limits & Async Lifecycle](#3-authentication-rate-limits--async-lifecycle)
4. [Image Generation (Ideogram 4.0 & 3.0)](#4-image-generation-ideogram-40--30)
5. [Magic Prompt & Structured (JSON) Prompts](#5-magic-prompt--structured-json-prompts)
6. [Image Understanding — Describe (Vision)](#6-image-understanding--describe-vision)
7. [Image Editing — Remix, Edit, Inpaint](#7-image-editing--remix-edit-inpaint)
8. [Background Workflows — Remove & Replace](#8-background-workflows--remove--replace)
9. [Transparent Background Generation](#9-transparent-background-generation)
10. [Reframe (Aspect-Ratio Extension)](#10-reframe-aspect-ratio-extension)
11. [Layerize Text (Editable Text Layer Extraction)](#11-layerize-text-editable-text-layer-extraction)
12. [Upscale](#12-upscale)
13. [Style System — Presets, Codes, References, Type](#13-style-system--presets-codes-references-type)
14. [Character & Style Reference Images](#14-character--style-reference-images)
15. [Custom Model Training](#15-custom-model-training)
16. [Webhooks (Async Delivery)](#16-webhooks-async-delivery)
17. [Capability Summary & Cross-Reference](#17-capability-summary--cross-reference)

---

## 1. Platform Overview & Core Concepts

### Main Concepts

Ideogram's API is organized around these core abstractions:

- **Model family** — Two generation model families route requests: **Ideogram 4.0** (latest; best prompt adherence and text rendering; accepts `text_prompt` or structured `json_prompt`) and **Ideogram 3.0** (broad endpoint coverage for edit, inpaint, reframe, replace background, and layerized-text workflows). Each endpoint URL embeds the model version (e.g. `/v1/ideogram-v4/generate`, `/v1/ideogram-v3/generate`).
- **Rendering speed** — A latency/fidelity tradeoff selected per request via `rendering_speed`: `FLASH` (fastest; V4 "coming soon"), `TURBO` (fast), `DEFAULT` (balanced), `QUALITY` (highest fidelity). Pricing varies by tier.
- **Resolution vs aspect ratio** — Output dimensions are set either by an explicit `resolution` enum (`{width}x{height}`) or by an `aspect_ratio` enum (e.g. `1x1`, `16x9`, `9x16`). The two are mutually exclusive; V3 supports a large grid of pixel resolutions, V4 supports 2K-class resolutions.
- **Magic Prompt** — Ideogram's prompt-enhancement feature. When a `text_prompt` is supplied to V4 generate, magic-prompt is enabled **automatically**. V3 exposes an explicit `magic_prompt` toggle (`AUTO`/`ON`/`OFF`). When a structured `json_prompt` is supplied (V4), magic-prompt is **disabled** and the diffusion model consumes the JSON contract directly.
- **JSON prompt (V4)** — A structured prompt contract (`V4JsonPrompt`) composed of a `high_level_description`, an optional `style_description`, and a required `compositional_deconstruction` (a `background` plus an ordered list of `elements`). Elements are discriminated by `type`: `obj` (objects/characters/background details) or `text` (literal text to render). Each element optionally carries a normalized `bbox` (`[y_min, x_min, y_max, x_max]` in `[0,1000]`).
- **Style system** — Multiple style-steering mechanisms across V3 endpoints: `style_type` (broad category), `style_preset` (~60 named artistic presets), `style_codes` (8-char hex codes), `style_reference_images` (image-based style transfer), and `color_palette` (preset name or explicit hex+weight members). V4 generation favors the structured `style_description` inside `json_prompt`.
- **Character reference** — A reference image (currently limited to 1) used to keep a character consistent across generations/edits. Optionally paired with a `character_reference_images_mask` (grayscale, same dimensions). Subject to character-reference pricing.
- **Image URL ephemerality** — Generated image URLs are **ephemeral** (available for a limited time). Consumers must download and store results in their own system. URLs include an `exp`/`sig` signature pair.
- **Safety checks** — Every generation runs safety checks. Unsafe images return `is_image_safe: false` with an empty `url`. Prompt failures return HTTP `422` (`GenerateImageSafetyError`).
- **Copyright detection** — Optional opt-in post-generation detection (Hive likeness + logo checks) via `enable_copyright_detection`. The effective gate is the OR of this flag and the org's `copyright_detection_enabled` setting. Flagged images come back `is_image_safe: false` and incur added latency.
- **Custom model** — A model trained on a user's approved assets (15–100 images). Trained models expose a `custom_model_uri` (`model/<name>/version/<version>`) passed to V3 generate to enforce on-brand style, typography, and identity. Models can be owned or shared via the model registry.

### Capability pipelines

```
Image understanding (vision)
   image_file ──▶ /v1/ideogram-v4/describe ──▶ V4JsonPrompt (structured, reusable as json_prompt)

Image generation
   text-to-image:     text_prompt ──▶ /v1/ideogram-v4/generate ──▶ image URL(s)
   structured:        json_prompt ──▶ /v1/ideogram-v4/generate ──▶ image URL(s)
   transparent:      prompt ──▶ /v1/ideogram-v3/generate-transparent ──▶ transparent PNG (+optional upscale)

Image editing
   remix:            image + prompt (+image_weight) ──▶ /v1/ideogram-v4/remix | /v1/ideogram-v3/remix ──▶ image
   edit (prompt):    image(s) | image_urls + prompt ──▶ /v1/edit ──▶ edited image
   inpaint (mask):   image + mask + prompt ──▶ /v1/ideogram-v3/inpaint ──▶ edited image
   reframe:          image + resolution ──▶ /v1/ideogram-v3/reframe ──▶ extended image
   replace bg:       image + prompt ──▶ /v1/ideogram-v3/replace-background ──▶ new background
   remove bg:        image ──▶ /v1/remove-background ──▶ transparent cutout
   upscale:          image_file + image_request ──▶ /upscale ──▶ higher-resolution image

Text extraction
   layerize text:    image ──▶ /v1/ideogram-v3/layerize-text ──▶ base image (text erased) + text blocks

Prompt enhancement
   magic prompt:     text_prompt + aspect_ratio ──▶ /v1/ideogram-v4/magic-prompt ──▶ V4JsonPrompt + resolved ratio

Custom model lifecycle
   create dataset ──▶ upload assets ──▶ train-model ──▶ poll models ──▶ use custom_model_uri in generate
```

---

## 2. API Architecture & Capability Matrix

All endpoints are JSON-over-HTTP. Generation/edit endpoints use `multipart/form-data` (to carry image binaries); prompt-enhancement, describe-via-URL, and custom-model training use `application/json`. The single polling endpoint is a `GET`.

| # | Surface | Endpoint / Method | Content-Type | Sync/Async | Purpose | Model |
|---|---|---|---|---|---|---|
| 1 | Generate (V4) | `POST /v1/ideogram-v4/generate` | multipart | Sync | Text/JSON-prompt → image(s) | 4.0 |
| 2 | Generate async (V4) | `POST /v1/ideogram-v4/async/generate` | multipart | Async (webhook) | Same as #1, returns `generation_id`, delivers via webhook | 4.0 |
| 3 | Poll generation | `GET /v1/generations/{generation_id}` | — | Poll | Retrieve async status + results | any |
| 4 | Remix (V4) | `POST /v1/ideogram-v4/remix` | multipart | Sync | image + prompt → remixed image | 4.0 |
| 5 | Magic Prompt (V4) | `POST /v1/ideogram-v4/magic-prompt` | json | Sync | text_prompt → enhanced `V4JsonPrompt` + aspect ratio | 4.0 |
| 6 | Describe (V4) | `POST /v1/ideogram-v4/describe` | multipart | Sync | image → `V4JsonPrompt` (vision) | 4.0 |
| 7 | Generate (V3) | `POST /v1/ideogram-v3/generate` | multipart | Sync | prompt + style/refs → image(s) | 3.0 |
| 8 | Generate transparent (V3) | `POST /v1/ideogram-v3/generate-transparent` | multipart | Sync | prompt → transparent-background image + optional upscale | 3.0 |
| 9 | Inpaint (V3) | `POST /v1/ideogram-v3/inpaint` | multipart | Sync | image + mask + prompt → edited image | 3.0 |
| 10 | Remix (V3) | `POST /v1/ideogram-v3/remix` | multipart | Sync | image + prompt + image_weight → remixed image | 3.0 |
| 11 | Reframe (V3) | `POST /v1/ideogram-v3/reframe` | multipart | Sync | image + resolution → reframed image | 3.0 |
| 12 | Replace background (V3) | `POST /v1/ideogram-v3/replace-background` | multipart | Sync | image + prompt → new background | 3.0 |
| 13 | Remove background | `POST /v1/remove-background` | multipart | Sync | image → transparent cutout | n/a |
| 14 | Layerize text (V3) | `POST /v1/ideogram-v3/layerize-text` | multipart | Sync | image → text-erased base + text blocks | 3.0 |
| 15 | Edit with prompt | `POST /v1/edit` | multipart | Sync | image(s)/URLs + prompt → edited image | 3.0 |
| 16 | Upscale | `POST /upscale` | multipart | Sync | image_file + image_request → upscaled image | n/a |
| 17 | Create dataset | `POST /datasets` | json | Sync | Create a named dataset container | n/a |
| 18 | Upload dataset assets | `POST /datasets/{dataset_id}/upload_assets` | multipart | Sync | Upload images/captions/ZIP to a dataset | n/a |
| 19 | Train custom model (V3) | `POST /v1/ideogram-v3/train-model` | json | Async (poll) | Train custom model from dataset | 3.0 |
| 20 | List models | `GET /models` | — | Sync | List owned/shared custom models | n/a |
| 21 | Get model | `GET /models/{model_id}` | — | Sync | Get custom model details | n/a |

### Core response shapes

**Synchronous generation (V4):** `ImageGenerationResponseV4`
```jsonc
{
  "created": "2026-06-02T16:24Z",
  "data": [
    {
      "url": "https://ideogram.ai/api/images/ephemeral/<id>.png?exp=...&sig=...",
      "prompt": "A bold poster",          // may differ from the original prompt
      "resolution": "2048x2048",
      "is_image_safe": true,              // false ⇒ url is empty
      "seed": 812374
    }
  ]
}
```

**Async acknowledgement (V4):** `AsyncImageGenerationResponseV4` → `{ "generation_id": "<base64>" }`

**Poll result (model-agnostic):** `GenerationResponse`
```jsonc
{
  "generation_id": "<base64>",
  "status": "pending" | "completed" | "failed",
  "created": "<datetime>",
  "response_type": "url",                 // present only when completed
  "data": [ { /* ImageGenerationObject */ } ]  // present only when completed
}
```

---

## 3. Authentication, Rate Limits & Async Lifecycle

### Authentication

All endpoints require the `Api-Key` header:
```
Api-Key: <your-api-key>
```
API keys are obtained from the dashboard at `https://ideogram.ai/api/api-keys`. The app subscription and the API account are **billed separately** — the API can be run from a free app account.

### Rate limits & scale

- **Default:** 10 in-flight (concurrent) requests while prototyping.
- **Enterprise:** For larger workloads, contact `partnership@ideogram.ai` to size throughput to your needs. The platform serves thousands of customers generating millions of images daily.

### Async lifecycle (V4)

V4 generate supports an asynchronous variant for long-running or batched workflows:

```
POST /v1/ideogram-v4/async/generate?webhook_url=<HTTPS URL>
   └─▶ 200 { generation_id }                       (immediate acknowledgement)

   [Ideogram processes the request]

POST <webhook_url>                                 (result delivered, Ed25519-signed)
   body = { generation_id, created, data[] }

# Fallback: poll if no webhook received
GET /v1/generations/{generation_id}
   └─▶ status: pending → completed (data[] present) | failed
```

The `webhook_url` query parameter is **required** for the async endpoint, must be HTTPS, and rejects private/loopback hosts and the cloud metadata service. The polling endpoint returns only `generation_id`/`status`/`created` while `pending` or `failed`; `response_type` and `data` appear only once `completed`.

### Supported input image formats

JPEG, PNG, WebP. Max **10MB** per image (per-file for upload, total across reference-image sets). Remix/inpaint/edit/reframe/replace-background/upscale/remove-background/layerize-text all accept these formats.

---

## 4. Image Generation (Ideogram 4.0 & 3.0)

### 4.1 Generate with Ideogram 4.0 (sync)

`POST /v1/ideogram-v4/generate` — `multipart/form-data`

The flagship synchronous generation endpoint. Accepts **either** a `text_prompt` (enables magic-prompt automatically) **or** a structured `json_prompt` (disables magic-prompt; consumed directly by the diffusion model). The two are mutually exclusive.

| Field | Type | Required | Description |
|---|---|---|---|
| `text_prompt` | string | one-of | Natural-language prompt. Enables magic-prompt automatically. Mutually exclusive with `json_prompt`. |
| `json_prompt` | `V4JsonPrompt` | one-of | Structured prompt contract (see §5). Disables magic-prompt. |
| `resolution` | enum `ResolutionV4` | no | One of 22 2K-class resolutions (see below). |
| `rendering_speed` | enum `RenderingSpeed` | no | `FLASH`/`TURBO`/`DEFAULT`/`QUALITY` (default `DEFAULT`). V4 `FLASH` "coming soon" — returns 400 today. |
| `enable_copyright_detection` | boolean\|null | no | Opt into Hive likeness + logo checks (see §1). |

**`ResolutionV4` enum (2K-class, 22 values):** `2048x2048`, `1440x2880`, `2880x1440`, `1664x2496`, `2496x1664`, `1792x2240`, `2240x1792`, `1440x2560`, `2560x1440`, `1600x2560`, `2560x1600`, `1728x2304`, `2304x1728`, `1296x3168`, `3168x1296`, `1152x2944`, `2944x1152`, `1248x3328`, `3328x1248`, `1280x3072`, `3072x1280`, `1024x3072`, `3072x1024`.

**Response:** `ImageGenerationResponseV4` (see §2).

### 4.2 Generate with Ideogram 3.0

`POST /v1/ideogram-v3/generate` — `multipart/form-data`

The broad-coverage V3 generate endpoint. Notably richer style/reference controls than V4 generate.

| Field | Type | Required | Description |
|---|---|---|---|
| `prompt` | string | yes | The prompt to generate the image. |
| `seed` | integer | no | Reproducible generation. |
| `resolution` | enum `ResolutionV3` | one-of | One of 68 pixel resolutions (mutually exclusive with `aspect_ratio`). |
| `aspect_ratio` | enum `AspectRatioV3` | one-of | `1x3`,`3x1`,`1x2`,`2x1`,`9x16`,`16x9`,`10x16`,`16x10`,`2x3`,`3x2`,`3x4`,`4x3`,`4x5`,`5x4`,`1x1` (default `1x1`). |
| `rendering_speed` | enum `RenderingSpeed` | no | `FLASH`/`TURBO`/`DEFAULT`/`QUALITY`. |
| `magic_prompt` | enum `MagicPromptOption` | no | `AUTO`/`ON`/`OFF`. |
| `negative_prompt` | string | no | What to exclude; prompt descriptions take precedence. |
| `num_images` | integer | no | Default `1`. |
| `color_palette` | `ColorPaletteWithPresetNameOrMembers` | no | Preset name OR explicit hex+weight members. |
| `style_codes` | array of `StyleCode` | no | 8-char hex codes; cannot combine with `style_reference_images` or `style_type`. |
| `style_type` | enum `StyleTypeV3` | no | `AUTO`/`GENERAL`/`REALISTIC`/`DESIGN`/`FICTION` (default `GENERAL`). |
| `style_preset` | enum `StylePresetV3` | no | ~60 named artistic presets (see §13). |
| `custom_model_uri` | string | no | `model/<name>/version/<version>`; resolves style from a trained custom model. |
| `style_reference_images` | array<binary> | no | Style-transfer references (≤10MB total). |
| `character_reference_images` | array<binary> | no | Character-consistency reference (currently 1; subject to character-ref pricing). |
| `character_reference_images_mask` | array<binary> | no | Optional grayscale masks, one per character ref. |
| `enable_copyright_detection` | boolean\|null | no | Opt into Hive likeness + logo checks. |

**`ResolutionV3` enum (68 values):** spans portrait/landscape/square from `512x1536`/`1536x512` up to `1536x640`/`640x1536` and `1024x1024`. (See OpenAPI spec for the full list.)

**`ImageGenerationObjectV3`** (in `data[]`) adds `upscaled_resolution` (output resolution if an operation alters dimensions) and `style_type` (echoes the resolved style) on top of the common `url`/`prompt`/`resolution`/`is_image_safe`/`seed` fields.

---

## 5. Magic Prompt & Structured (JSON) Prompts

### 5.1 Magic Prompt (V4)

`POST /v1/ideogram-v4/magic-prompt` — `application/json`

Transforms a basic prompt into an enhanced Ideogram 4.0 structured (`V4JsonPrompt`) magic prompt. The magic-prompt model version is fixed; callers cannot select it. The response `json_prompt` conforms to the same contract as `json_prompt` on `/v1/ideogram-v4/generate`, so it can be passed straight back to generate.

| Field | Type | Required | Description |
|---|---|---|---|
| `text_prompt` | string | yes | The natural-language prompt to enhance. |
| `aspect_ratio` | enum `AspectRatioV4` | no | Default `AUTO` (model selects). Non-AUTO values pin the ratio. |

**`AspectRatioV4` enum:** `AUTO`,`1x4`,`1x3`,`1x2`,`9x16`,`10x16`,`2x3`,`3x4`,`4x5`,`1x1`,`5x4`,`4x3`,`3x2`,`16x10`,`16x9`,`2x1`,`3x1`,`4x1`.

**Response:** `MagicPromptV4Response`
```jsonc
{
  "json_prompt": { /* V4JsonPrompt */ },
  "aspect_ratio": "1x1"   // resolved ratio (echoed or model-selected when AUTO)
}
```

### 5.2 V4JsonPrompt contract

The structured prompt consumed directly by the diffusion model when `json_prompt` is supplied to V4 generate (or produced by magic-prompt / describe). Mutually exclusive with `text_prompt`.

```jsonc
{
  "high_level_description": "One- or two-sentence overall description of the desired image.",
  "style_description": {                    // optional
    "aesthetics": "warm, cozy, nostalgic",
    "art_style": "illustration",            // optional art-style hint
    "lighting": "soft natural window light",
    "medium": "photograph",                 // e.g. photograph, digital art
    "photo": "50mm lens, film stock"         // optional photographic notes
  },
  "compositional_deconstruction": {          // required
    "background": "A dim room with a bright window casting warm light.",
    "elements": [
      {
        "type": "obj",                       // discriminator: "obj" | "text"
        "bbox": [0, 0, 1000, 1000],          // optional, [y_min, x_min, y_max, x_max] in [0,1000]
        "desc": "ginger cat with green eyes sitting upright on a vintage wooden chair"
      },
      {
        "type": "text",
        "bbox": [120, 200, 300, 800],
        "text": "Creative Typography",       // literal text to render
        "desc": "bold sans-serif headline, centered"
      }
    ]
  }
}
```

**Bounding-box semantics:** `[y_min, x_min, y_max, x_max]` (row-first), normalized so the canvas is `1000 × 1000` regardless of final resolution. Elements are discriminated by `type` (`obj` for non-text objects/characters/background details; `text` for literal text to render).

---

## 6. Image Understanding — Describe (Vision)

`POST /v1/ideogram-v4/describe` — `multipart/form-data` — tags: `vision`

Describes an image with Ideogram 4.0 and returns a structured `V4JsonPrompt`. The returned `json_prompt` is a **working JSON prompt** that can be passed directly as `json_prompt` to the `/v1/ideogram-v4/generate` family — making describe a round-trip "image → reproducible prompt → image" capability.

| Field | Type | Required | Description |
|---|---|---|---|
| `image_file` | binary | yes | Image binary (max 10MB); JPEG, PNG, WebP. |
| `include_bbox` | boolean | no | Default `true`. When true, preserves bounding boxes on each element so the prompt reproduces the source layout when pasted into generate. Set `false` to drop boxes and let the sampler place elements freely. |

**Response:** `DescribeResponseV4` → `{ "json_prompt": V4JsonPrompt }`

**Errors:** `422` `ImageSafetyError` (image failed safety check); `503` (took too long to finish).

This is the single dedicated **image understanding** endpoint. Unlike platforms that fold vision into a general chat/completions surface, Ideogram's describe is purpose-built to emit a generation-ready structured prompt (with optional spatial layout) rather than free-form caption text.

---

## 7. Image Editing — Remix, Edit, Inpaint

### 7.1 Remix (V4 & V3)

Reimagines an existing image using the original as a basis, with a text prompt steering the new output. The key control is `image_weight`: higher values keep the input's structure; lower values let the prompt drive the output. When omitted, the weight is chosen automatically from the edit instruction.

**V4:** `POST /v1/ideogram-v4/remix` — mirrors V3 semantics, routes through the V4 model.

| Field | Type | Required | Description |
|---|---|---|---|
| `image` | binary | yes | Initial image (max 10MB); JPEG/WebP/PNG. |
| `text_prompt` | string | yes | Text prompt guiding the remix. |
| `image_weight` | integer | no | How strongly output should resemble input. Overrides automatic choice. |
| `resolution` | `ResolutionV4` | no | |
| `rendering_speed` | `RenderingSpeed` | no | |
| `enable_copyright_detection` | boolean\|null | no | |

**V3:** `POST /v1/ideogram-v3/remix` — input images are cropped to the chosen aspect ratio before being remixed. Adds the full V3 style/reference surface (`aspect_ratio`, `negative_prompt`, `num_images`, `color_palette`, `style_codes`, `style_type`, `style_preset`, `style_reference_images`, `character_reference_images`, `character_reference_images_mask`, `magic_prompt`, `seed`, `resolution`). `image_weight` defaults to `50`.

### 7.2 Edit with a prompt

`POST /v1/edit` — `multipart/form-data`

The general-purpose prompt-based editor. Edit one or more images by describing the desired change in plain language. Supports **both** file upload and Ideogram image URLs as references — and can edit multiple images (up to 10) in one call.

| Field | Type | Required | Description |
|---|---|---|---|
| `prompt` | string | yes | Describes the desired edit. |
| `images` | array<binary> | one-of | Images to edit (max 10, max 10MB each); JPEG/WebP/PNG. |
| `image_urls` | array<string> | one-of | URLs to Ideogram images (max 10). Alternative to `images`. |
| `num_images` | integer | no | Default `1`. |
| `seed` | integer | no | Reproducible. |
| `magic_prompt` | `MagicPromptOption` | no | |
| `resolution` | `ResolutionV3` | no | |
| `aspect_ratio` | `AspectRatioV3` | no | |
| `transparent_background` | boolean | no | Default `false`. Output with transparent background. |

**Errors:** `402` (insufficient credits/quota), `503` (service temporarily unavailable), `422` (safety). This is the endpoint used for face-swapping workflows per the overview (swap the face of a person using a mask + prompt).

### 7.3 Inpaint (V3)

`POST /v1/ideogram-v3/inpaint` — `multipart/form-data`

Masked editing: the mask indicates **which part** of the image should be edited, while the prompt and chosen style guide the edit. Black regions in the mask match the regions to edit.

| Field | Type | Required | Description |
|---|---|---|---|
| `image` | binary | yes | Image being edited (max 10MB). |
| `mask` | binary | yes | Black-and-white image, same size as `image` (max 10MB). Black = edit region. |
| `prompt` | string | yes | Describes the edited result. |
| `magic_prompt`, `num_images`, `seed`, `rendering_speed`, `style_type`, `style_preset`, `color_palette`, `style_codes`, `style_reference_images`, `character_reference_images`, `character_reference_images_mask` | various | no | Full V3 style/reference surface (see §4.2 / §13 / §14). |

---

## 8. Background Workflows — Remove & Replace

Two distinct capabilities: **remove** (returns a transparent cutout of the foreground) and **replace** (generates a new background from a prompt while keeping the foreground subject).

### 8.1 Remove background

`POST /v1/remove-background` — `multipart/form-data`

Identifies the foreground subject and returns it on a transparent background. No prompt, no style controls — a single image in, a single cutout out.

| Field | Type | Required | Description |
|---|---|---|---|
| `image` | binary | yes | Image whose background is removed (max 10MB); JPEG/WebP/PNG. |

**Response:** `RemoveBackgroundResponse` — `data[]` always contains **exactly one** `RemoveBackgroundImageObject` (`url`, `is_image_safe`).

### 8.2 Replace background (V3)

`POST /v1/ideogram-v3/replace-background` — `multipart/form-data`

Identifies and keeps the foreground subject, replaces the background based on a prompt and chosen style.

| Field | Type | Required | Description |
|---|---|---|---|
| `image` | binary | yes | Image whose background is replaced (max 10MB). |
| `prompt` | string | yes | Describes the desired new background. |
| `magic_prompt`, `num_images`, `seed`, `rendering_speed`, `style_preset`, `color_palette`, `style_codes`, `style_reference_images` | various | no | V3 style surface (note: no `style_type` or `character_reference_images` on this endpoint). |

---

## 9. Transparent Background Generation

`POST /v1/ideogram-v3/generate-transparent` — `multipart/form-data`

Generates images with a **native transparent background** from a prompt (e.g. die-cut stickers, logos, POD assets). Images are generated at maximum supported resolution for the chosen aspect ratio to allow best results with the upscaler; the response reports the selected (pre-upscale) resolution, while `upscaled_resolution` reports the final output.

`rendering_speed=FLASH` is **not supported** here (returns 400); use `TURBO`, `DEFAULT`, or `QUALITY`.

| Field | Type | Required | Description |
|---|---|---|---|
| `prompt` | string | yes | Prompt to generate the image. |
| `seed` | integer | no | Reproducible. |
| `upscale_factor` | enum `UpscaleFactor` | no | `X1` (default)/`X2`/`X4`. Factors other than `X1` incur additional cost. |
| `aspect_ratio` | `AspectRatioV3` | no | Default `1x1`. |
| `rendering_speed` | enum (restricted) | no | `TURBO`/`DEFAULT`/`QUALITY` (no `FLASH`). |
| `magic_prompt` | `MagicPromptOption` | no | |
| `negative_prompt` | string | no | |
| `num_images` | integer | no | Default `1`. |

Note this is distinct from `remove-background` (which strips the background from an *existing* image) and from `edit` with `transparent_background=true` (which produces a transparent *edited* output).

---

## 10. Reframe (Aspect-Ratio Extension)

`POST /v1/ideogram-v3/reframe` — `multipart/form-data`

Extends an image to a new resolution/aspect ratio without losing the focal point (outpainting-style). Designed for square inputs reframed to a chosen target resolution.

| Field | Type | Required | Description |
|---|---|---|---|
| `image` | binary | yes | Image being reframed (max 10MB). |
| `resolution` | `ResolutionV3` | yes | Target resolution (one of the 68 V3 values). |
| `num_images`, `seed`, `rendering_speed`, `style_preset`, `color_palette`, `style_codes`, `style_reference_images` | various | no | Subset of the V3 style surface (no `style_type`, `character_reference_images`, `magic_prompt`, or `negative_prompt` here). |

---

## 11. Layerize Text (Editable Text Layer Extraction)

`POST /v1/ideogram-v3/layerize-text` — `multipart/form-data`

Analyzes an image to detect text regions, then returns each detected text block with its position, content, font information, and styling — plus a **text-erased base image** (the background with all text removed). This powers editable-text-layer workflows: replace or restyle the text, then composite it back over the clean base.

| Field | Type | Required | Description |
|---|---|---|---|
| `image` | binary | yes | Image to analyze (max 10MB); JPEG/PNG/WebP. |
| `prompt` | string | no | Optional text description of the image; auto-generated if omitted. |
| `seed` | integer | no | Reproducible. |

**Response:** `LayerizeTextResponse`
```jsonc
{
  "base_image_url": "https://.../base.png",       // image with all detected text removed
  "original_image_url": "https://.../orig.png",    // original image with text intact (nullable)
  "seed": 12345,
  "text_blocks": [
    {
      "role": "heading",
      "text": "Hello World",
      "x": 0, "y": 6, "width": 1, "height": 5,
      "angle": 5.637,
      "color": "#212121",
      "font_name": "font_name",
      "font_alternatives": ["font_alternatives"],
      "font_size": 2,
      "line_height": 7.06,
      "alignment": "left",
      "formatting": ["bold"]
    }
  ]
}
```

Each `text_block` carries geometry (`x`, `y`, `width`, `height`, `angle`), typography (`font_name`, `font_alternatives`, `font_size`, `line_height`, `alignment`, `formatting`), color (`color` hex), content (`text`), and role (`role`).

---

## 12. Upscale

`POST /upscale` — `multipart/form-data`

Raises the resolution of a provided image, optionally guided by a prompt. Note the path has no `/v1` prefix.

| Field | Type | Required | Description |
|---|---|---|---|
| `image_request` | `UpscaleInitialImageRequest` (JSON string in the multipart field) | yes | Upscale parameters. |
| `image_file` | binary | yes | Image binary (max 10MB); JPEG/WebP/PNG. |

**`UpscaleInitialImageRequest`:**

| Field | Type | Required | Description |
|---|---|---|---|
| `prompt` | string | no | Optional prompt to guide the upscale. |
| `resemblance` | integer | no | Default `50`. How closely to resemble the input. |
| `detail` | integer | no | Default `50`. How much detail to add. |
| `magic_prompt_option` | `MagicPromptOption` | no | |
| `num_images` | integer | no | Default `1`. |
| `seed` | integer | no | Reproducible. |

The `image_request` field is sent as a JSON string inside the multipart form (e.g. `image_request={"resemblance":55,"detail":90}`). Response `ImageObject` includes `resolution` (final) and `upscaled_resolution`. The `GenerateImageResponse` also supports an optional `request_id` when the caller supplied a `webhook_url` (legacy async delivery).

---

## 13. Style System — Presets, Codes, References, Type

V3 endpoints expose a layered style system (mutually-exclusive rules apply between some controls).

### 13.1 Style type (`StyleTypeV3`)

Broad category steering (default `GENERAL`):

| Value | Meaning |
|---|---|
| `AUTO` | Let the model decide. |
| `GENERAL` | General-purpose (default). |
| `REALISTIC` | Photorealistic. |
| `DESIGN` | Design/typographic. |
| `FICTION` | Fiction/illustrative. |

### 13.2 Style preset (`StylePresetV3`)

~60 named artistic presets applying a specific artistic style. Examples: `80S_ILLUSTRATION`, `90S_NOSTALGIA`, `ART_DECO`, `ART_POSTER`, `AVANT_GARDE`, `BAUHAUS`, `BLUEPRINT`, `C4D_CARTOON`, `CHILDRENS_BOOK`, `COLLAGE`, `COLORING_BOOK_I`/`II`, `CUBISM`, `DOODLE`, `DOUBLE_EXPOSURE`, `DRAMATIC_CINEMA`, `EDITORIAL`, `EXPIRED_FILM`, `FLAT_VECTOR`, `GOLDEN_HOUR`, `GRAFFITI_I`/`II`, `HALFTONE_PRINT`, `JAPANDI_FUSION`, `MAGAZINE_EDITORIAL`, `MINIMAL_ILLUSTRATION`, `MIXED_MEDIA`, `MONOCHROME`, `OIL_PAINTING`, `POP_ART`, `RETRO_ETCHING`, `SPOTLIGHT_80S`, `SURREAL_COLLAGE`, `TRAVEL_POSTER`, `VINTAGE_POSTER`, `WATERCOLOR`, `WOODBLOCK_PRINT`, `WEIRD`, and more. (See the OpenAPI spec for the complete enum.)

### 13.3 Style codes (`StyleCode` / `StyleCodes`)

An array of 8-character hexadecimal codes representing the style. **Cannot be used in conjunction with `style_reference_images` or `style_type`.** Enables precise, shareable style reproduction.

### 13.4 Style reference images

A set of images used as style references (max total 10MB across all references; JPEG/PNG/WebP). Available on generate, remix, inpaint, reframe, and replace-background. Combined with `style_codes`/`style_type` exclusivity rules.

### 13.5 Color palette (`ColorPaletteWithPresetNameOrMembers`)

Either a **preset name** OR **explicit members** (not both). Not supported by legacy V_1/V_2A models.

- **Preset names** (`ColorPalettePresetName`): `EMBER`, `FRESH`, `JUNGLE`, `MAGIC`, `MELON`, `MOSAIC`, `PASTEL`, `ULTRAMARINE`.
- **Members** (`ColorPaletteWithMembers`): a list of `ColorPaletteMember` (`color_hex` required; optional `color_weight` between 0.05 and 1.0). Weights are recommended to descend from highest to lowest.

### 13.6 V4 style description

V4 generate prefers the structured `style_description` block inside `V4JsonPrompt` (see §5.2): `aesthetics`, `art_style`, `lighting`, `medium`, `photo`. This replaces the enum-heavy V3 style surface with free-form descriptive fields.

---

## 14. Character & Style Reference Images

Available on V3 generate, V3 remix, and V3 inpaint.

| Field | Type | Constraints |
|---|---|---|
| `style_reference_images` | array<binary> | Max total 10MB across all references; JPEG/PNG/WebP. |
| `character_reference_images` | array<binary> | Max total 10MB; **currently only 1 character reference** supported. Subject to character-reference pricing. |
| `character_reference_images_mask` | array<binary> | Optional; must match the number of `character_reference_images`. Each mask is a grayscale image of the same dimensions as the corresponding character reference image. |

Character references keep a character consistent across generations and edits (e.g. the same person across a campaign). Style references transfer the visual style of the provided images to the output.

---

## 15. Custom Model Training

A four-step lifecycle: **create dataset → upload assets → train → use**. Datasets organize training images; trained models expose a `custom_model_uri` passed to V3 generate.

### 15.1 Create a dataset

`POST /datasets` — `application/json`

| Field | Type | Required | Description |
|---|---|---|---|
| `name` | string | yes | Name of the dataset. |

**Response:** `Dataset` (`dataset_id`, `name`, `user_id`, `creation_time`, optional `cover_asset_identifier` of type `AssetIdentifier` — `{asset_type, asset_id}` where `asset_type` ∈ `ASSET`/`CANVAS_ASSET`/`LAYERED_ASSET`/`RESPONSE`/`UPLOAD`).

### 15.2 Upload assets to a dataset

`POST /datasets/{dataset_id}/upload_assets` — `multipart/form-data`

Accepts individual images (JPEG/PNG/WebP), optional `.txt` caption sidecar files, and/or ZIP archives containing images and captions. Captions are matched to images by **filename stem** (e.g. `sunset.txt` captions `sunset.jpg`) and are **optional**. A dataset can contain up to 100 images.

| Field | Type | Required | Description |
|---|---|---|---|
| `files` | array<binary> | yes | Images, `.txt` sidecars, and/or ZIP archives. |

**Response:** `UploadDatasetAssetsResponse` — `total_count`, `success_count`, `failure_count`, `successful_assets[]` (each `DatasetUploadSucceededAsset` with `asset_identifier` + `file_name`), `failed_assets[]` (each `DatasetUploadFailedAsset` with `failure_reason` ∈ `FAILED_SAFETY_CHECK`/`FILE_TOO_LARGE`/`INTERNAL_ERROR`/`INVALID_CAPTION`/`INVALID_IMAGE`/`INVALID_ZIP`/`TOO_MANY_IMAGES` + `file_name`).

### 15.3 Train a custom Ideogram v3 model

`POST /v1/ideogram-v3/train-model` — `application/json`

Starts training using default hyperparameters. The dataset must contain **at least 15 images and a maximum of 100 images**.

| Field | Type | Required | Description |
|---|---|---|---|
| `dataset_id` | string | yes | ID of the dataset to train from. |
| `model_name` | string | yes | 5–30 characters, alphanumeric with spaces and hyphens allowed. |

**Response:** `TrainDatasetModelResponse` — `model_id`, `dataset_id`, `training_status`, `model_name`.

### 15.4 List & get models

- `GET /models` — lists custom models. Query params:
  - `scope` (`owned` | `shared`): omit to return both owned + shared (via the model registry); `owned` returns only the user's; `shared` returns only registry-shared, excluding the user's own.
  - `status` (array of `ModelStatus`: `CREATING`/`DRAFT`/`TRAINING`/`COMPLETED`/`ERRORED`/`ARCHIVED`): filter owned models.
- `GET /models/{model_id}` — get details; requires ownership or org-shared access (404 otherwise).

**`CustomModel`** includes `model_id`, `custom_model_uri` (`model/<name>/version/<version>`, present only for registry-registered models — **use this URI in generate**), `name`, `status`, `dataset_id`, `creation_time`, `last_update_time`, `is_available_for_generation`, `is_owned`.

### 15.5 Using a custom model

Pass `custom_model_uri` to V3 generate (`POST /v1/ideogram-v3/generate`). When provided, the model version and style are resolved from the URI and `style_type` is not required.

---

## 16. Webhooks (Async Delivery)

The async V4 generate endpoint (`/v1/ideogram-v4/async/generate`) returns immediately with a `generation_id` and POSTs the finished result to the supplied `webhook_url`. The body **mirrors the synchronous response** (so one handler serves both), plus a `generation_id` for correlation.

### 16.1 Webhook payload

```jsonc
{
  "generation_id": "xtdZiqPwRxqY1Y7NExFmzB",
  "created": "2025-01-23T04:56:07Z",
  "data": [
    { "url": "...", "prompt": "...", "resolution": "2048x2048", "seed": 12345, "is_image_safe": true }
  ]
}
```

### 16.2 Request headers (every delivery)

| Header | Value |
|---|---|
| `X-Ideogram-Webhook-Generation-Id` | URL-safe base64 ID; matches `generation_id` in body. |
| `X-Ideogram-Webhook-User-Id` | URL-safe base64 ID of the initiating account. |
| `X-Ideogram-Webhook-Timestamp` | Unix seconds (decimal) at signing time. |
| `X-Ideogram-Webhook-Key-Id` | `kid` of the signing key (hint, not a requirement). |
| `X-Ideogram-Webhook-Signature` | Ed25519 signature, lowercase hex. |
| `Content-Type` | `application/json` |

### 16.3 Signature verification (Ed25519)

1. **Fetch public keys:** `GET https://api.ideogram.ai/v1/.well-known/jwks.json` → JWK set; each key's `x` is the 32-byte public key, base64url (no padding). Cacheable; refresh on verification failure.
2. **Rebuild the signed message** (joined with `\n`, UTF-8):
   ```
   {X-Ideogram-Webhook-Generation-Id}
   {X-Ideogram-Webhook-User-Id}
   {X-Ideogram-Webhook-Timestamp}
   {sha256_hex(raw request body bytes)}
   ```
   Hash the **raw** body bytes exactly as received — do not parse/re-serialize JSON first.
3. **Verify:** decode `X-Ideogram-Webhook-Signature` from hex and check against each public key. If any verifies, the webhook is authentic. Use `X-Ideogram-Webhook-Key-Id` as a hint but fall back across all keys for rotation resilience.

### 16.4 Retries, idempotency & polling

- Return `2xx` to acknowledge. Non-`2xx` or timeout triggers retries.
- **Make handlers idempotent** — the same `generation_id` can arrive more than once. Key state on `generation_id`; treat already-processed IDs as no-ops.
- **Delivery is not guaranteed** — Ideogram retries only a limited number of times before dropping. Do not rely on webhooks as the only result channel.
- **Fall back to polling:** `GET /v1/generations/{generation_id}` returns the same `data` once complete.

### 16.5 Raw-body accessors

Use raw bytes, not parsed/re-serialized JSON: Flask `request.get_data()`; FastAPI/Starlette `await request.body()`; Django `request.body`; Express `express.raw()` middleware → `req.body` (Buffer).

---

## 17. Capability Summary & Cross-Reference

| # | Capability | API / Endpoint | Key parameters | Model |
|---|---|---|---|---|
| 1 | Image generation (text-to-image) | `POST /v1/ideogram-v4/generate` | `text_prompt` (magic-prompt auto-on) **or** `json_prompt` (magic-prompt off), `resolution` (`ResolutionV4` 2K-class), `rendering_speed` (`FLASH`*/`TURBO`/`DEFAULT`/`QUALITY`), `enable_copyright_detection` | 4.0 |
| 2 | Async image generation (webhook) | `POST /v1/ideogram-v4/async/generate?webhook_url=` | same as #1 + required `webhook_url` (HTTPS); returns `generation_id` | 4.0 |
| 3 | Poll async generation | `GET /v1/generations/{generation_id}` | path `generation_id`; `status` `pending`/`completed`/`failed`; `data` present only when completed | any |
| 4 | Image generation (V3, rich style) | `POST /v1/ideogram-v3/generate` | `prompt`, `resolution` (`ResolutionV3`) **or** `aspect_ratio`, `rendering_speed`, `magic_prompt`, `negative_prompt`, `num_images`, `color_palette`, `style_codes`/`style_type`/`style_preset`, `custom_model_uri`, `style_reference_images`, `character_reference_images`(+`_mask`), `enable_copyright_detection` | 3.0 |
| 5 | Transparent-background generation | `POST /v1/ideogram-v3/generate-transparent` | `prompt`, `upscale_factor` (`X1`/`X2`/`X4`), `aspect_ratio`, `rendering_speed` (no `FLASH`), `magic_prompt`, `negative_prompt`, `num_images` | 3.0 |
| 6 | Magic Prompt enhancement | `POST /v1/ideogram-v4/magic-prompt` | `text_prompt`, `aspect_ratio` (`AUTO` lets model select); returns `V4JsonPrompt` + resolved ratio | 4.0 |
| 7 | Image understanding (describe) | `POST /v1/ideogram-v4/describe` | `image_file` (≤10MB), `include_bbox` (default true); returns `V4JsonPrompt` reusable as `json_prompt` | 4.0 |
| 8 | Remix (V4) | `POST /v1/ideogram-v4/remix` | `image`, `text_prompt`, `image_weight` (auto if omitted), `resolution`, `rendering_speed`, `enable_copyright_detection` | 4.0 |
| 9 | Remix (V3) | `POST /v1/ideogram-v3/remix` | `image`, `prompt`, `image_weight` (default 50), + full V3 style/reference surface; input cropped to aspect ratio | 3.0 |
| 10 | Edit with a prompt | `POST /v1/edit` | `prompt`, `images` (≤10 files) **or** `image_urls` (≤10 URLs), `transparent_background`, `magic_prompt`, `resolution`/`aspect_ratio`, `num_images`, `seed` | 3.0 |
| 11 | Inpaint (masked edit) | `POST /v1/ideogram-v3/inpaint` | `image`, `mask` (B/W, black=edit), `prompt`, + V3 style/reference surface | 3.0 |
| 12 | Reframe (aspect-ratio extension) | `POST /v1/ideogram-v3/reframe` | `image`, `resolution` (`ResolutionV3`), `num_images`, `seed`, `rendering_speed`, `style_preset`, `color_palette`, `style_codes`, `style_reference_images` | 3.0 |
| 13 | Replace background | `POST /v1/ideogram-v3/replace-background` | `image`, `prompt` (new background), `magic_prompt`, `num_images`, `seed`, `rendering_speed`, `style_preset`, `color_palette`, `style_codes`, `style_reference_images` | 3.0 |
| 14 | Remove background (cutout) | `POST /v1/remove-background` | `image` only; returns exactly one transparent foreground image | n/a |
| 15 | Layerize text (text-layer extraction) | `POST /v1/ideogram-v3/layerize-text` | `image`, optional `prompt`, `seed`; returns `base_image_url` (text erased) + `text_blocks[]` (geometry, font, color, role) | 3.0 |
| 16 | Upscale | `POST /upscale` | `image_file`, `image_request` (`prompt`, `resemblance`, `detail`, `magic_prompt_option`, `num_images`, `seed`); note: no `/v1` prefix | n/a |
| 17 | Style presets (~60) | V3 endpoints | `style_preset` enum (`ART_DECO`, `WATERCOLOR`, `BAUHAUS`, …) | 3.0 |
| 18 | Style codes | V3 endpoints | `style_codes` array of 8-char hex; exclusive with `style_reference_images`/`style_type` | 3.0 |
| 19 | Style reference images | V3 endpoints | `style_reference_images` (≤10MB total); JPEG/PNG/WebP | 3.0 |
| 20 | Character reference (consistency) | V3 generate/remix/inpaint | `character_reference_images` (currently 1; character-ref pricing), optional `character_reference_images_mask` (grayscale, matching dims) | 3.0 |
| 21 | Color palette | V3 endpoints | `color_palette`: preset name (`EMBER`/`FRESH`/`JUNGLE`/`MAGIC`/`MELON`/`MOSAIC`/`PASTEL`/`ULTRAMARINE`) **or** members (`color_hex` + optional `color_weight` 0.05–1.0) | 3.0 |
| 22 | Copyright detection (opt-in) | V4/V3 generate, V4 remix | `enable_copyright_detection` (Hive likeness + logo checks); OR with org setting; flagged ⇒ `is_image_safe:false` | 4.0/3.0 |
| 23 | Custom model: create dataset | `POST /datasets` | `name` | n/a |
| 24 | Custom model: upload assets | `POST /datasets/{dataset_id}/upload_assets` | `files` (images, `.txt` sidecars by stem, ZIP); ≤100 images/dataset | n/a |
| 25 | Custom model: train | `POST /v1/ideogram-v3/train-model` | `dataset_id`, `model_name` (5–30 chars); requires 15–100 images; returns `model_id` | 3.0 |
| 26 | Custom model: list/get | `GET /models` · `GET /models/{model_id}` | `scope` (`owned`/`shared`), `status` filter; `custom_model_uri` for generation | n/a |
| 27 | Custom model: use | V3 generate | `custom_model_uri` (`model/<name>/version/<version>`); resolves style; `style_type` not required | 3.0 |
| 28 | Webhook delivery (async results) | `POST <webhook_url>` | Ed25519-signed; headers `X-Ideogram-Webhook-*`; body mirrors sync response + `generation_id` | 4.0 |
| 29 | Webhook verification (JWKS) | `GET /v1/.well-known/jwks.json` | Ed25519 public keys (JWK `x` base64url); signed message = gen-id\|user-id\|timestamp\|sha256(raw body) | — |

\* `FLASH` rendering speed on V4 is "coming soon" and currently returns 400.

### Cross-cutting notes

- **Understanding vs generation split:** Image **understanding** is a single dedicated endpoint (`/v1/ideogram-v4/describe`) that returns a structured, generation-ready `V4JsonPrompt` (with optional spatial layout via `bbox`). Image **generation** is the broad surface across V4/V3 endpoints. This is narrower than platforms that fold vision into a general chat API — Ideogram's describe is purpose-built for the round-trip "image → prompt → image" workflow.
- **Two model families, endpoint-per-model routing:** V4 endpoints (`/v1/ideogram-v4/*`) and V3 endpoints (`/v1/ideogram-v3/*`) are selected via the URL path, not a request field. V4 favors structured `json_prompt` + `style_description`; V3 carries the richer enum-based style/reference surface. Some capabilities (transparent generation, inpaint, reframe, replace-background, layerize-text) are V3-only; V4 adds async/webhook, magic-prompt, and describe.
- **Text rendering is the differentiator:** Ideogram's headline strength is rendering crisp, accurate text inside images (logos, posters, branding, POD). The `text` element type in `V4JsonPrompt` and the layerize-text endpoint both lean into this: the former places literal text by normalized bbox; the latter extracts editable text layers (font, color, geometry) plus a text-erased base for re-composition.
- **Bounding boxes are normalized to 1000×1000:** Both the V4 `json_prompt` element `bbox` (`[y_min, x_min, y_max, x_max]`) and the describe response's element boxes use a 1000×1000 normalized canvas regardless of final resolution — the same convention as the Gemini object-detection schema.
- **Sync vs async:** Most endpoints are synchronous. V4 generate offers an async webhook variant; the poll endpoint (`GET /v1/generations/{id}`) is the fallback/result-retrieval channel. Webhook deliveries are Ed25519-signed and must be verified against the published JWKS; handlers must be idempotent (key on `generation_id`) and should fall back to polling since delivery is not guaranteed.
- **Ephemeral URLs & safety:** All generated image URLs are ephemeral (signed with `exp`/`sig`); download and persist. Every generation runs safety checks — unsafe images return `is_image_safe:false` with an empty `url`; prompt/image safety failures return HTTP `422`. Optional copyright detection adds Hive likeness + logo checks.
- **Style system exclusivity:** On V3, `style_codes` cannot be combined with `style_reference_images` or `style_type`. `color_palette` cannot mix preset name and explicit members. `resolution` and `aspect_ratio` are mutually exclusive.
- **Custom models for brand consistency:** Train on 15–100 approved assets to get a `custom_model_uri` that enforces on-brand style, typography, and identity when passed to V3 generate. Models can be owned or shared via the model registry (`scope=shared`).
- **Open weights & MCP:** Ideogram 4.0 weights are open-sourced on GitHub (`ideogram-oss/ideogram-4`) and Hugging Face, and an MCP server (`https://developer.ideogram.ai/_mcp/server`) is available for AI-client (Claude Code, Cursor) integration.
- **No video:** Ideogram's API is image-only. There are no video generation, video understanding, or video editing endpoints. For video, pair Ideogram (image/asset generation) with a separate video platform.