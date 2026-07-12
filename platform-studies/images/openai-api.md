# OpenAI API Analysis — Images, Vision & Video Generation

> **Base URL:** `https://api.openai.com/v1`
> **Docs:** `https://developers.openai.com/api/docs`
> **Auth:** Bearer API key (`Authorization: Bearer $OPENAI_API_KEY`)
> **SDKs:** `openai` (Python, JavaScript/TypeScript, C#, Go, Java), `openai` CLI
> **Description:** OpenAI exposes a multimodal media layer built on two model families — **GPT Image** (text/image → image) and **Sora 2** (text/image → video, with audio). Image *understanding* (vision) is provided by the mainline GPT-5.x reasoning/chat models and is consumed through the Responses and Chat Completions APIs. Image and video *generation* are consumed through dedicated endpoints (Images API, Videos API) and, for images, also as a built-in tool of the Responses API.

---

## Table of Contents

1. [Platform Overview & Core Concepts](#1-platform-overview--core-concepts)
2. [API Architecture & Capability Matrix](#2-api-architecture--capability-matrix)
3. [Image Understanding (Vision)](#3-image-understanding-vision)
4. [Image Generation & Editing](#4-image-generation--editing)
5. [Image Output Customization, Cost & Moderation](#5-image-output-customization-cost--moderation)
6. [Video Generation (Sora 2)](#6-video-generation-sora-2)
7. [Video References, Characters, Extensions & Edits](#7-video-references-characters-extensions--edits)
8. [Video Job Lifecycle, Retrieval & Library Management](#8-video-job-lifecycle-retrieval--library-management)
9. [Capability Summary & Cross-Reference](#9-capability-summary--cross-reference)

---

## 1. Platform Overview & Core Concepts

OpenAI's image/video surface is organized around these core abstractions:

- **Vision** — The ability of mainline GPT models (e.g. `gpt-5.6`, `gpt-5.5`, `gpt-5.4`, GPT-4o/4.1 family) to "see" and understand image inputs: objects, shapes, colors, textures, and text within images. Consumed as a modality of chat/reasoning models, not a separate endpoint.
- **GPT Image models** — Dedicated image *generation/editing* models (`gpt-image-2`, `gpt-image-1.5`, `gpt-image-1`, `gpt-image-1-mini`). They take text and/or images as input and emit a raster image (PNG/JPEG/WebP). They bring world knowledge and strong instruction following to image synthesis.
- **Sora 2 models** — Dedicated video *generation* models (`sora-2`, `sora-2-pro`) that synthesize dynamic clips with audio from text or an image, built on multimodal diffusion. Trained on diverse visual data for 3D space, motion, and scene continuity.
- **Responses API** — The unified stateful API that natively handles both vision input and image generation (as a built-in `image_generation` tool). Supports multi-turn editing, file IDs, and conversation state.
- **Images API** — The dedicated `/v1/images/generations` and `/v1/images/edits` endpoints for single-shot image generation/editing where you pick a GPT Image model directly.
- **Videos API** — The asynchronous `/v1/videos*` surface for create, extend, edit, and manage videos via Sora 2. Also includes `/v1/videos/characters` for reusable character assets.
- **Files API** — `/v1/files` used to upload images with `purpose="vision"` and obtain `file_id`s that can be referenced from vision and image-generation requests (Responses API) and from video `input_reference` JSON requests.
- **Patch / Tile tokenization** — How images are metered. Modern models (GPT-5.x) cover images with 32×32 patches and bill per patch (× model multiplier); older GPT-4o/4.1/o-series use a tile-based 512px-square scheme.
- **Webhooks** — Server-side notifications for long-running async jobs (video `completed`/`failed` events), configurable in the project webhook settings.

### Capability pipelines

```
Image understanding (vision)
   input: URL | Base64 data URL | File ID ──▶ mainline GPT model ──▶ text/audio output
   (Responses API input_image, or Chat Completions image_url)

Image generation / editing
   Image API:    prompt [+ image(s)] [+ mask] ──▶ GPT Image model ──▶ base64 PNG/JPEG/WebP
   Responses API: prompt + image_generation tool ──▶ image_generation_call.result (base64)
                  multi-turn via previous_response_id or image_generation_call ids

Video generation (async)
   prompt [+ input_reference image] [+ characters] ──▶ POST /videos ──▶ job id
        poll /videos/{id}  OR  webhook(video.completed/failed)
        ──▶ GET /videos/{id}/content (MP4)  +  thumbnail.webp  +  spritesheet.jpg
   extend:   POST /videos/extensions  (source video + prompt)
   edit:     POST /videos/edits       (source video + prompt)
```

---

## 2. API Architecture & Capability Matrix

### Endpoints at a glance

| API | Endpoint | Modality | Purpose |
|---|---|---|---|
| Responses | `POST /v1/responses` | image in → text/image out | Vision analysis; image generation as built-in tool; multi-turn editing |
| Chat Completions | `POST /v1/chat/completions` | image in → text/audio out | Vision analysis (legacy/compat path) |
| Images | `POST /v1/images/generations` | text/image in → image out | Single-shot image generation |
| Images | `POST /v1/images/edits` | text + image(s) + mask in → image out | Single-shot image editing / reference-based generation |
| Images | (legacy) `/v1/images/variations` | image in → image out | Variations (DALL·E 2 only) |
| Videos | `POST /v1/videos` | text/image in → video job | Start an async video render |
| Videos | `GET /v1/videos/{video_id}` | — | Poll job status |
| Videos | `GET /v1/videos/{video_id}/content` | — | Download MP4 / thumbnail / spritesheet |
| Videos | `POST /v1/videos/extensions` | video + prompt → video job | Extend a completed video |
| Videos | `POST /v1/videos/edits` | video + prompt → video job | Edit an existing video |
| Videos | `POST /v1/videos/characters` | MP4 upload → character id | Create a reusable character asset |
| Videos | `GET /v1/videos` | — | List videos (paginate/sort) |
| Videos | `DELETE /v1/videos/{video_id}` | — | Delete a video |
| Files | `POST /v1/files` (`purpose="vision"`) | upload → file_id | Upload images referenced by Responses/Batch |
| Batch | `POST /v1/batches` (body targets `/v1/videos`) | JSON only | Offline video render queues |

### API selection guidance (from docs)

- **Single generate/edit from one prompt** → Images API (pick a GPT Image model directly).
- **Conversational, editable, multi-turn image experiences** → Responses API (`image_generation` tool).
- **Vision analysis** → either Responses API (`input_image`) or Chat Completions (`image_url`); Responses is the modern path.
- **Video** → Videos API only (async). Use `input_reference` for image-guided first frame, `characters` for reusable non-human subjects, `/extensions` to continue a clip, `/edits` for targeted changes.

### Model families

- **Mainline GPT (vision + image-gen-tool host):** `gpt-5.6` (newest), `gpt-5.5`, `gpt-5.4`, `gpt-5.4-mini/nano`, `gpt-5`, `gpt-4.1`, `gpt-4o`, `gpt-4o-mini`, `o1/o3` series, `computer-use-preview`. GPT-5 and newer support the `image_generation` tool.
- **GPT Image (the actual image synthesizer):** `gpt-image-2` (latest, any-resolution, always high-fidelity, no transparency), `gpt-image-1.5`, `gpt-image-1`, `gpt-image-1-mini`. Legacy DALL·E 2 (supports variations) / DALL·E 3.
- **Sora 2 (video):** `sora-2` (speed/flexibility), `sora-2-pro` (production-quality, 1080p). Both support 16- and 20-second generations. Sora 2 models are **deprecated, shut down Sept 24, 2026** (per docs deprecation notice).

> **Verification:** GPT Image models may require API Organization Verification before use. Video editing of uploaded videos and human-likeness characters are gated (contact sales/account manager).

---

## 3. Image Understanding (Vision)

### Main concepts

Vision is the ability of mainline GPT models to interpret images — objects, shapes, colors, textures, and embedded text. It is a *modality* of chat/reasoning models, invoked by including image content parts in a request. There is no separate "vision endpoint"; vision is consumed through the Responses API (`input_image`) or Chat Completions API (`image_url`). Multiple images may be supplied in a single request's `content` array (each counts as tokens).

### 3.1 Passing images as input

Three input forms are supported:

| Input form | Responses API | Chat Completions API |
|---|---|---|
| Fully-qualified URL | `{"type":"input_image","image_url":"https://..."}` | `{"type":"image_url","image_url":{"url":"https://..."}}` |
| Base64 data URL | `{"type":"input_image","image_url":"data:image/jpeg;base64,..."}` | `{"type":"image_url","image_url":{"url":"data:image/jpeg;base64,..."}}` |
| File ID (Files API, `purpose="vision"`) | `{"type":"input_image","file_id":"file_..."}` | not supported (Responses only) |

Text and image parts are combined in the user message `content` array (`input_text` + `input_image` for Responses, `text` + `image_url` for Chat). Typical request:

```jsonc
// Responses API
{
  "model": "gpt-5.6",
  "input": [{
    "role": "user",
    "content": [
      {"type": "input_text", "text": "what's in this image?"},
      {"type": "input_image", "image_url": "https://.../image.jpg"}
    ]
  }]
}
```

### 3.2 Image input requirements

- **Supported file types:** PNG, JPEG (`.jpeg`/`.jpg`), WebP, non-animated GIF.
- **Size limits:** up to 512 MB total payload per request; up to 1500 individual image inputs per request.
- **Other:** no watermarks/logos, no NSFW content, must be clear enough for a human to understand.

### 3.3 The `detail` parameter

`detail` tells the model how to process the image: `low` | `high` | `original` | `auto` (default `auto`). Same behavior in Responses and Chat Completions.

| Detail | Best for |
|---|---|
| `low` | Fast, low-cost understanding; model receives a 512×512 low-res version. |
| `high` | Standard high-fidelity understanding. |
| `original` | Large, dense, spatially sensitive, or computer-use images. Available on `gpt-5.4` and future models. Preserves input dimensions (within limits). |
| `auto` | Automatic. On `gpt-5.5`/`gpt-5.6` equivalent to `original`; on `gpt-5.4` equivalent to `high`. |

For computer use, localization, and click-accuracy on `gpt-5.4`+, use `"detail":"original"`.

```jsonc
{"type":"input_image","image_url":"https://.../image.jpg","detail":"original"}
```

### 3.4 Model sizing behavior

| Model family | Supported detail | Patch / resizing behavior |
|---|---|---|
| GPT-5.6 family | low, high, original, auto | `original` preserves input dimensions (no pixel/patch-budget cap); `auto`/omitted = `original`. `low`/`high` may resize within finite limits. |
| `gpt-5.5` | low, high, original, auto | `high` ≤ 2,500 patches / 2048px max dim; `original` ≤ 10,000 patches / 6000px max dim; `auto` = `original`. |
| `gpt-5.4` | low, high, original, auto | Same limits as 5.5, but `auto` = `high`. |
| `gpt-5.4-mini/nano`, `gpt-5-mini/nano`, `gpt-5.2`, codex variants, `o4-mini`, `gpt-4.1-mini/nano` (2025-04-14) | low, high, auto | `high` ≤ 1,536 patches / 2048px max dim; resize to fit. |
| `gpt-4o`, `gpt-4.1`, `gpt-4o-mini`, `computer-use-preview`, o-series (except o4-mini) | low, high, auto | Tile-based resizing (see below). |

### 3.5 Token metering & cost

Images are metered in token units. Two schemes:

**Patch-based (GPT-5.x family)** — images are covered with 32×32 patches:
```
original_patch_count = ceil(width/32) × ceil(height/32)
# If > patch_budget: shrink_factor = sqrt((32² × budget) / (w×h)) (then adjusted)
# resized_patch_count = ceil(resized_w/32) × ceil(resized_h/32)
total_tokens = resized_patch_count × model_multiplier
```
Model multipliers: `gpt-5.4-mini`/`gpt-5-mini`/`gpt-4.1-mini*` = 1.62; `gpt-5.4-nano`/`gpt-5-nano`/`gpt-4.1-nano*` = 2.46; `o4-mini` = 1.72. (`*` 2025-04-14 snapshot.)

**Tile-based (GPT-4o/4.1/4o-mini/CUA/o-series except o4-mini)** — `low` = fixed base tokens; `high` scales the shortest side to 768px, counts 512px squares (tiles), adds base tokens. Examples: `gpt-5`/`gpt-5-chat-latest` = 70 base + 140/tile; `gpt-4o`/`gpt-4.1`/`gpt-4.5` = 85 base + 170/tile; `gpt-4o-mini` = 2833 base + 5667/tile; `o1`/`o1-pro`/`o3` = 75 base + 150/tile; `computer-use-preview` = 65 base + 129/tile.

**GPT Image 1 (image-editing input)** — same as tile-based but shortest side scales to 512px. `low` = 65 base + 129/tile; `high` adds extra tokens by aspect (square +4160, portrait/landscape +6240). See [image pricing](https://openai.com/api/pricing/).

Each processed image counts toward tokens-per-minute (TPM). Use the [vision pricing calculator](https://openai.com/api/pricing/) for precise estimates.

### 3.6 Vision limitations (per docs)

- Not suitable for specialized medical images (CT scans) or medical advice.
- Reduced performance for non-Latin text (Japanese, Korean); small text should be enlarged (`original` detail helps).
- Rotated/upside-down text and images may be misinterpreted.
- Graphs/styles with varied line types (solid/dashed/dotted) are hard.
- Weak precise spatial reasoning (e.g. chess positions) and counting.
- Panoramic/fisheye shapes are problematic.
- Original file names/metadata are not processed; `low`/`high` may resize.
- CAPTCHAs are blocked for safety.

---

## 4. Image Generation & Editing

### 4.1 Generate images

Two entry points:

**Images API — `POST /v1/images/generations`**
```python
result = client.images.generate(
    model="gpt-image-2",
    prompt="A children's book drawing of a veterinarian using a stethoscope...",
)
image_bytes = base64.b64decode(result.data[0].b64_json)
```

**Responses API — `image_generation` built-in tool**
```python
response = client.responses.create(
    model="gpt-5.6",                              # mainline model hosts the tool
    input="Generate an image of a gray tabby cat hugging an otter...",
    tools=[{"type": "image_generation"}],
)
image_b64 = [o.result for o in response.output if o.type == "image_generation_call"][0]
```

Key differences:
- Image API: you select the GPT Image model directly; returns `data[0].b64_json`.
- Responses API: you select a mainline model that supports the tool; the tool picks the GPT Image model; returns `image_generation_call` output objects with `id`, `type`, `status`, `revised_prompt`, `result` (base64). Responses requests also bill mainline-model token usage on top of image-gen cost.

`n` parameter: generate multiple images in one request (default 1).

### 4.2 Multi-turn image generation (Responses API only)

Iteratively refine images across conversation turns. Two mechanisms:

1. **`previous_response_id`** — pass the prior response's `id` to continue the conversation.
2. **Image ID** — pass prior `image_generation_call` outputs by `id` into the next request's `input`:
```python
input=[
  {"role":"user","content":[{"type":"input_text","text":"Now make it look realistic"}]},
  {"type":"image_generation_call","id": <prev_call_id>},
]
```

The `action` parameter on the `image_generation` tool controls behavior:
- `"auto"` (default) — model decides generate vs. edit.
- `"generate"` — always create a new image.
- `"edit"` — force editing of an image in context (errors if no image is in context).

### 4.3 Edit images

Endpoint: `POST /v1/images/edits` (Image API) or the `image_generation` tool with image inputs (Responses API).

Capabilities:
- Edit existing images.
- Generate new images using other images as references (multiple references supported).
- Edit parts of an image via a mask identifying areas to replace.

**Image API** — `client.images.edit(model=..., image=[...files...], prompt=...)`:
- `image` accepts a single file **or an array** of files (multiple references).
- `mask` (optional file) — applied to the **first image** when multiple provided.

**Responses API** — image inputs via `input_image` content items (`image_url` or `file_id`); mask via the tool's `input_image_mask` field `{"file_id": maskId}` (mask must be uploaded to Files API). Often combined with `quality:"high"`.

### 4.4 Masks

Masking with GPT Image is **entirely prompt-based** — the model uses the mask as guidance but may not follow its exact shape precisely; additional instructions guide editing.

Mask requirements:
- Mask and target image must be the **same format and size**, and less than **50 MB**.
- The mask image **must contain an alpha channel** (save with alpha in an editor, or convert programmatically: load as grayscale `"L"`, convert to `"RGBA"`, `putalpha(mask)` to fill alpha from the mask).

### 4.5 File inputs (Responses API)

Files API upload for vision/image-gen inputs and masks:
```python
file_id = client.files.create(file=open("image.jpg","rb"), purpose="vision").id
# then reference as {"type":"input_image","file_id": file_id} or mask {"file_id": mask_id}
```
This is a Responses-API-only capability (Image API uses raw file uploads).

### 4.6 Image input fidelity

`input_fidelity` controls how strongly the model preserves details from input images during edits / reference workflows.
- For `gpt-image-2`: **omit this parameter** — not configurable; the model always processes every image input at **high fidelity** automatically.
- Because of always-on high fidelity, edit requests with reference images can use more input image tokens (cost implication — see §3.5).

### 4.7 Revised prompt (Responses API)

The mainline model (e.g. `gpt-5.5`) automatically revises the user's prompt for improved image-gen performance. The revised prompt is exposed in the `revised_prompt` field of the `image_generation_call` output object.

### 4.8 Streaming

Both APIs support streaming partial images during generation.

`partial_images` parameter (0–3):
- `0` → only the final image.
- `>0` → stream up to that many partial images; fewer may arrive if the final image completes quickly.

**Responses API streaming:**
```python
stream = client.responses.create(
    model="gpt-5.6", input="...", stream=True,
    tools=[{"type":"image_generation","partial_images":2}],
)
for event in stream:  # response.image_generation_call.partial_image (partial_image_index, partial_image_b64)
    ...               # response.completed -> final in event.response.output[].result
```

**Image API streaming:**
```python
stream = client.images.generate(model="gpt-image-2", prompt="...", stream=True, partial_images=2)
# events: image_generation.partial_image (partial_image_index, b64_json)
```

**Partial-image cost:** each partial image adds **100 image output tokens**.

---

## 5. Image Output Customization, Cost & Moderation

### 5.1 Customizable output

| Option | Values | Notes |
|---|---|---|
| `size` | `1024x1024`, `1536x1024`, `1024x1536`, `2048x2048`, `2048x1152`, `3840x2160`, `2160x3840`, `auto` (default), ... | `auto` lets the model pick from the prompt. |
| `quality` | `low`, `medium`, `high`, `auto` (default) | `low` = fastest (drafts/thumbnails). |
| `output_format` | `png` (default), `jpeg`, `webp` | `jpeg` faster than `png` — prefer when latency matters. |
| `output_compression` | 0–100% (jpeg/webp only) | e.g. `50` = 50% compression. |
| `background` | `opaque`, `automatic`, `transparent` | Transparent **unsupported on `gpt-image-2`** (model-dependent). |
| `n` | integer (default 1) | Number of images per request. |
| `moderation` | `auto` (default), `low` | Content-moderation strictness. |

### 5.2 `gpt-image-2` size constraints

- Accepts **any resolution** satisfying the constraints below (squares typically fastest).
- Maximum edge length ≤ **3840px**.
- Both edges must be multiples of **16px**.
- Long-edge:short-edge ratio ≤ **3:1**.
- Total pixels ≥ **655,360** and ≤ **8,294,400**.
- Outputs above `2560x1440` (3,686,400 px, "2K") are considered **experimental**.

### 5.3 Cost

Total cost = input text tokens + input image tokens (if editing/references) + image output tokens + (100 × partial images).

**`gpt-image-2` output tokens** — estimated from requested `quality` and `size` (interactive calculator in docs; example shows 196 output tokens for a low-quality sample).

**GPT Image 1.5 / 1 / 1-mini output tokens (image-only):**

| Quality | Square 1024×1024 | Portrait 1024×1536 | Landscape 1536×1024 |
|---|---|---|---|
| Low | 272 | 408 | 400 |
| Medium | 1,056 | 1,584 | 1,568 |
| High | 4,160 | 6,240 | 6,208 |

**Per-image output pricing (output tokens only):**

| Model | Quality | 1024×1024 | 1024×1536 | 1536×1024 |
|---|---|---|---|---|
| GPT Image 2 | Low | $0.006 | $0.005 | $0.005 |
| GPT Image 2 | Medium | $0.053 | $0.041 | $0.041 |
| GPT Image 2 | High | $0.211 | $0.165 | $0.165 |
| GPT Image 1.5 | Low | $0.009 | $0.013 | $0.013 |
| GPT Image 1.5 | Medium | $0.034 | $0.050 | $0.050 |
| GPT Image 1.5 | High | $0.133 | $0.200 | $0.200 |
| GPT Image 1 | Low | $0.011 | $0.016 | $0.016 |
| GPT Image 1 | Medium | $0.042 | $0.063 | $0.063 |
| GPT Image 1 | High | $0.167 | $0.250 | $0.250 |
| GPT Image 1 Mini | Low | $0.005 | $0.006 | $0.006 |
| GPT Image 1 Mini | Medium | $0.011 | $0.015 | $0.015 |
| GPT Image 1 Mini | High | $0.036 | $0.052 | $0.052 |

`gpt-image-2` supports thousands of valid resolutions; the table uses legacy sizes for comparison. A larger non-square resolution can sometimes produce fewer output tokens than a smaller square one at the same quality.

### 5.4 Moderation & error handling

`moderation` parameter (GPT Image models): `auto` (default, standard filtering) | `low` (less restrictive).

Blocked/failed requests surface as standard API errors — check HTTP status / SDK exception, log request ID. Retry transient `429`/`5xx`; do **not** retry user-correctable errors without changing the request.

Moderation error shape:
```json
{
  "error": {
    "type": "image_generation_user_error",
    "code": "moderation_blocked",
    "moderation_details": {
      "moderation_stage": "input",     // input | output | unknown
      "categories": ["harassment"]      // e.g. harassment, self-harm, sexual, violence
    }
  }
}
```
- `error.code = "moderation_blocked"` is the stable discriminator; `moderation_details` is optional context for logs/analytics.
- `moderation_stage`: `input` (block from prompt/inputs), `output` (block from generated image/downstream), `unknown` (rare fallback).
- Keep end-user messages generic; use `moderation_details` only for developer logs/support.

### 5.5 Image-generation limitations

Applies to `gpt-image-2`, `gpt-image-1.5`, `gpt-image-1`, `gpt-image-1-mini`:
- **Latency:** complex prompts may take up to 2 minutes.
- **Text rendering:** improved but can still struggle with precise placement/clarity.
- **Consistency:** may struggle to keep recurring characters/brand elements consistent across generations.
- **Composition control:** difficulty placing elements precisely in structured/layout-sensitive compositions.

---

## 6. Video Generation (Sora 2)

> **Deprecation notice:** Sora 2 models and Videos API are deprecated and will shut down **September 24, 2026** (affects `sora-2`, `sora-2-pro`, dated snapshots, and the Videos API).

### 6.1 Main concepts

Sora is OpenAI's frontier generative video model — creates richly detailed, dynamic clips **with audio** from natural language or images. Built on multimodal diffusion; trained on diverse visual data for 3D space, motion, and scene continuity.

Video generation is **asynchronous**:
1. `POST /videos` → returns a job object with `id` and initial `status`.
2. Poll `GET /videos/{video_id}` until `completed`, **or** register a webhook to be notified.
3. `GET /videos/{video_id}/content` → streams the final MP4.

### 6.2 Models

| Model | Profile | Resolution | Use case |
|---|---|---|---|
| `sora-2` | Speed & flexibility | 480p / 720p | Exploration, rapid iteration, concepting, rough cuts, social content, prototypes. |
| `sora-2-pro` | Production quality | 1080p (`1920x1080` / `1080x1920`) | High-resolution cinematic footage, marketing assets, visual precision. |

Both support **16- and 20-second** generations. Longer durations and 1080p jobs take materially longer than short 720p/480p renders — plan for higher latency in user-facing flows.

### 6.3 Start a render job — `POST /v1/videos`

Request parameters:

| Parameter | Required | Description |
|---|---|---|
| `model` | yes | `sora-2` or `sora-2-pro`. |
| `prompt` | yes | Text describing creative look/feel — subjects, camera, lighting, motion. |
| `size` | no | Target resolution, e.g. `1280x720`, `1920x1080`, `1080x1920`. Must match `input_reference` resolution when used. |
| `seconds` | no | Clip length (supports 16 and 20). |
| `input_reference` | no | First-frame image (multipart file upload **or** JSON `{file_id}`/`{image_url}`). |
| `characters` | no | Array of `{id}` referencing uploaded character assets. Up to 2 per video. |

Content types: `multipart/form-data` (with file uploads) **or** `application/json` (with `file_id`/`image_url` references, required for Batch).

```bash
curl -X POST "https://api.openai.com/v1/videos" \
  -H "Authorization: Bearer $OPENAI_API_KEY" \
  -H "Content-Type: multipart/form-data" \
  -F prompt="Wide tracking shot of a teal coupe..." \
  -F model="sora-2-pro" \
  -F size="1280x720" \
  -F seconds="8"
```

Job response:
```json
{
  "id": "video_68d7512d...",
  "object": "video",
  "created_at": 1758941485,
  "status": "queued",        // queued | in_progress | completed | failed
  "model": "sora-2-pro",
  "progress": 0,
  "seconds": "8",
  "size": "1280x720"
}
```

SDK convenience: `client.videos.createAndPoll({...})` / `videos.create_and_poll(...)` blocks until terminal status.

### 6.4 Size & duration guidance

- Pick the smallest format that meets production needs.
- Use shorter clips while iterating on prompt/motion/composition.
- Up to 20 seconds for longer beats, fuller scenes, fuller spots.
- `sora-2-pro` for 1080p exports (`1920x1080` / `1080x1920`).

### 6.5 Guardrails & restrictions

- Only content suitable for audiences under 18 (bypass setting coming).
- Copyrighted characters and copyrighted music are rejected.
- Real people — including public figures — cannot be generated.
- Character uploads depicting human likeness are blocked by default (eligibility via sales/account manager).
- Input images with human faces are currently rejected.

### 6.6 Effective prompting

Describe **shot type, subject, action, setting, and lighting**:
- "Wide shot of a child flying a red kite in a grassy park, golden hour sunlight, camera slowly pans upward."
- "Close-up of a steaming coffee cup on a wooden table, morning light through blinds, soft depth of field."

Advanced techniques: Sora 2 [prompting guide](https://cookbook.openai.com/examples/sora/sora2_prompting_guide).

### 6.7 Monitor progress

**Poll** — `GET /videos/{video_id}` (every 10–20s with exponential backoff). States: `queued`, `in_progress`, `completed`, `failed`. Response includes `progress` % and any `error`.

**Webhooks** — configured in project webhook settings. Events: `video.completed`, `video.failed`. Payload:
```json
{
  "id": "evt_abc123",
  "object": "event",
  "created_at": 1758941485,
  "type": "video.completed",
  "data": { "id": "video_abc123" }
}
```

### 6.8 Retrieve results — `GET /v1/videos/{video_id}/content`

Streams binary MP4 with standard content headers (save to disk or pipe to cloud storage). Download URLs are valid for **up to 1 hour** after generation — copy to your own storage for long-term retention.

`variant` query parameter selects the asset:

| `variant` | Output |
|---|---|
| `video` (default) | MP4 video file. |
| `thumbnail` | `thumbnail.webp` — lightweight preview. |
| `spritesheet` | `spritesheet.jpg` — scrubber/catalog asset. |

```bash
curl -L "https://api.openai.com/v1/videos/video_abc123/content" -H "Authorization: Bearer $OPENAI_API_KEY" --output video.mp4
curl -L "https://api.openai.com/v1/videos/video_abc123/content?variant=thumbnail" ... --output thumbnail.webp
curl -L "https://api.openai.com/v1/videos/video_abc123/content?variant=spritesheet" ... --output spritesheet.jpg
```

---

## 7. Video References, Characters, Extensions & Edits

### 7.1 Image references (`input_reference`)

An input image acts as **the first frame** of the video — useful to preserve the look of a brand asset, character, or environment.

- Multipart request: `input_reference` = uploaded image file (`@sample_720p.jpeg;type=image/jpeg`).
- JSON request (incl. Batch): `input_reference` = object with `file_id` **or** `image_url`.
- The image **must match the target video's resolution** (`size`).
- Supported image formats: `image/jpeg`, `image/png`, `image/webp`.

```bash
curl -X POST "https://api.openai.com/v1/videos" ... \
  -F prompt="She turns around and smiles, then slowly walks out of the frame." \
  -F model="sora-2-pro" -F size="1280x720" -F seconds="8" \
  -F input_reference="@sample_720p.jpeg;type=image/jpeg"
```

### 7.2 Characters (`/v1/videos/characters`)

A reusable **non-human** subject (animal, mascot, object) referenced across multiple generations for consistent appearance, styling, and screen presence.

- Create: `POST /v1/videos/characters` with a short MP4 clip and a `name`.
```bash
curl -X POST "https://api.openai.com/v1/videos/characters" ... \
  -F "video=@character.mp4;type=video/mp4" -F "name=Mossy"
```
- Use: include the returned character ID in the `characters` array of `POST /v1/videos` (up to **2 characters per video**). **Mention the character name verbatim in the prompt** — passing the ID alone is not enough.
- Works best with short 2–4 second clips in 16:9 or 9:16, at 720p–1080p. Source video aspect ratio should match the output (else distortion).
- Can be **combined with `input_reference`**.
- **Extensions do not support characters.**
- Human-likeness characters blocked by default (eligibility via sales).

```bash
curl -X POST "https://api.openai.com/v1/videos" ... -H "Content-Type: application/json" -d '{
  "model":"sora-2",
  "prompt":"A cinematic tracking shot of Mossy, a moss-covered teapot mascot, weaving through a lantern-lit market at dusk.",
  "size":"1280x720","seconds":"8",
  "characters":[{"id":"char_123"}]
}'
```

**`input_reference` vs `characters`** — an image reference conditions the opening frame of a **single** generation; a character asset is **reusable** across future requests.

### 7.3 Extend completed videos — `POST /v1/videos/extensions`

Continue an existing completed video and stitch a new segment. Provide the source video in the `video` field and a prompt describing how the scene should continue.

- Each extension adds up to **20 seconds**.
- A single video can be extended up to **6 times** → max total length **120 seconds**.
- Accepts **only** a source video and prompt — **no characters or image references**.

```bash
curl -X POST "https://api.openai.com/v1/videos/extensions" ... -d '{
  "video":{"id":"video_abc123"},
  "prompt":"Continue the scene as the camera rises over the rooftops and reveals the sunrise.",
  "seconds":"8"
}'
```
Use extensions to preserve motion, camera direction, and scene continuity; use `input_reference` only to control the opening frame of a new generation.

### 7.4 Edit existing videos — `POST /v1/videos/edits`

Targeted adjustments to an existing video without regenerating everything; the system reuses original structure, continuity, and composition while applying the modification. Best with a **single, well-defined change** — smaller focused edits preserve fidelity and reduce artifacts.

- `video` accepts either a **video ID** (API infers model from source) **or** an **uploaded video** (set `model` explicitly; editing uploaded videos is gated — contact sales).
- The legacy `/remix` endpoint is being deprecated; use `/edits` for new integrations.

```bash
# By video ID
curl -X POST "https://api.openai.com/v1/videos/edits" ... -d '{
  "video":{"id":"video_abc123"},
  "prompt":"Shift the color palette to teal, sand, and rust, with a warm backlight."
}'
# Uploaded video
curl -X POST "https://api.openai.com/v1/videos/edits" ... \
  -F "video=@source.mp4;type=video/mp4" -F "model=sora-2-pro" \
  -F "prompt=Shift the color palette to teal, sand, and rust, with a warm backlight."
```

Editing is valuable for iteration: refine without discarding what works, keeping style, subject consistency, and camera framing stable while exploring variations in mood/palette/staging.

### 7.5 Batch API for video

Use the [Batch API](https://developers.openai.com/api/docs/guides/batch) for offline render queues / studio workflows. Each line of the batch input file uses the same JSON body as `POST /v1/videos`.

- Batch currently supports **`POST /v1/videos` only** (no extensions/edits/characters endpoints).
- Requests **must use JSON** (not multipart).
- Upload assets ahead of time; reference via `input_reference` object with `file_id` or `image_url`.
- Multipart `input_reference` uploads (including video references) **not supported** in Batch.
- Batch-generated videos are available for download for up to **24 hours** after the batch completes.
- Use stable `custom_id` to map results back to internal shot IDs.

```json
{"custom_id":"shot-001","method":"POST","url":"/v1/videos","body":{"model":"sora-2-pro","prompt":"Slow dolly shot through a miniature paper city at blue hour...","size":"1920x1080","seconds":"20"}}
{"custom_id":"shot-002","method":"POST","url":"/v1/videos","body":{"model":"sora-2-pro","prompt":"Portrait close-up of a red panda chef plating noodles...","size":"1080x1920","seconds":"16"}}
```

---

## 8. Video Job Lifecycle, Retrieval & Library Management

### 8.1 Job object fields

| Field | Description |
|---|---|
| `id` | Unique job id (`video_...`). |
| `object` | `"video"`. |
| `created_at` | Unix timestamp. |
| `status` | `queued` \| `in_progress` \| `completed` \| `failed`. |
| `model` | `sora-2` / `sora-2-pro` (inferred from source for edits by ID). |
| `progress` | 0–100 (when available). |
| `seconds` | Clip length. |
| `size` | Resolution. |
| `error` | On failure — `error.message`. |

### 8.2 Lifecycle flow

```
POST /videos ──▶ queued ──▶ in_progress (poll/webhook) ──▶ completed ──▶ GET /content (MP4/thumbnail/spritesheet)
                                                   └────▶ failed ──▶ inspect error.code/message
```

### 8.3 Library management

- `GET /v1/videos?limit=20&after=video_123&order=asc` — enumerate videos (pagination + sorting).
- `DELETE /v1/videos/{video_id}` — remove videos from OpenAI storage.

```bash
curl "https://api.openai.com/v1/videos?limit=20&after=video_123&order=asc" -H "Authorization: Bearer $OPENAI_API_KEY" | jq .
curl -X DELETE "https://api.openai.com/v1/videos/REPLACE_WITH_YOUR_VIDEO_ID" -H "Authorization: Bearer $OPENAI_API_KEY" | jq .
```

---

## 9. Capability Summary & Cross-Reference

| # | Capability | API / Endpoint | Key parameters | Models |
|---|---|---|---|---|
| 1 | Image understanding (vision) | Responses (`input_image`) / Chat Completions (`image_url`) | `model`, `input`/`messages`, `detail` (low/high/original/auto), image via `image_url`\|`file_id` | gpt-5.6/5.5/5.4 family, gpt-4o/4.1/4o-mini, o-series, computer-use-preview |
| 2 | Image generation (single-shot) | Images `/images/generations` | `model`, `prompt`, `n`, `size`, `quality`, `background`, `output_format`, `output_compression`, `stream`, `partial_images`, `moderation` | gpt-image-2/1.5/1/1-mini |
| 3 | Image generation (conversational) | Responses `image_generation` tool | `model` (mainline), `input`, `tools=[{type, action, partial_images, quality, size, background, output_format, output_compression, input_image_mask, input_fidelity}]`, `previous_response_id`, `stream` | gpt-5.6/5.5/... (host) → gpt-image-* (synth) |
| 4 | Multi-turn image editing | Responses (tool) | `previous_response_id` **or** `image_generation_call` ids in `input`; `action: auto\|generate\|edit` | gpt-5.6/5.5 host |
| 5 | Image editing with mask | Images `/images/edits` / Responses tool | `image` (file/array), `mask` (alpha file / `input_image_mask.file_id`), `prompt`, `quality:high` | gpt-image-* |
| 6 | Image references for new generation | Images `/images/edits` (`image` array) / Responses (`input_image` items) | multiple `input_image` items / `image` array; `input_fidelity` (omit for gpt-image-2) | gpt-image-* |
| 7 | Image streaming | Images + Responses | `stream:true`, `partial_images` 0–3 (+100 output tokens each) | gpt-image-* |
| 8 | File-based image input | Files API (`purpose=vision`) → Responses `file_id` | `file_id` for input images and masks | Responses only |
| 9 | Video generation (async) | Videos `POST /videos` | `model`, `prompt`, `size`, `seconds`, `input_reference`, `characters` | sora-2, sora-2-pro |
| 10 | Video status / polling | Videos `GET /videos/{id}` | — (poll or webhook) | sora-2/2-pro |
| 11 | Video download | Videos `GET /videos/{id}/content` | `variant=video\|thumbnail\|spritesheet` | sora-2/2-pro |
| 12 | Image-guided video (first frame) | Videos `POST /videos` with `input_reference` | `input_reference` (multipart file **or** JSON `{file_id}\|{image_url}`); must match `size` | sora-2/2-pro |
| 13 | Reusable character asset | Videos `POST /videos/characters` + `characters` array | `video` (MP4), `name`; use `characters:[{id}]` + name in prompt; ≤2/video | sora-2/2-pro |
| 14 | Video extension | Videos `POST /videos/extensions` | `video.id`, `prompt`, `seconds`; ≤20s/extension, ≤6 extensions, ≤120s total; no characters/refs | sora-2/2-pro |
| 15 | Video edit | Videos `POST /videos/edits` | `video.id` **or** uploaded `video` (set `model`); `prompt`; one focused change | sora-2/2-pro |
| 16 | Video batch queue | Batch `POST /batches` (targets `/v1/videos`, JSON only) | `custom_id`, body as `/videos`; `input_reference` as `{file_id}\|{image_url}`; 24h download | sora-2/2-pro |
| 17 | Video library mgmt | Videos `GET /videos`, `DELETE /videos/{id}` | `limit`, `after`, `order` | sora-2/2-pro |
| 18 | Webhooks for video | Project webhook settings | events `video.completed`, `video.failed` | sora-2/2-pro |

### Cross-cutting notes

- **Vision vs generation split:** Vision is a modality of mainline GPT models (no dedicated model); generation uses dedicated GPT Image / Sora 2 models. The Responses API bridges them — a mainline model can both *see* images (`input_image`) and *produce* images (`image_generation` tool) in the same conversation.
- **Token economics:** Vision images are billed via patch/tile tokenization (see §3.5). Image generation is billed via output-image tokens driven by `quality`+`size` (see §5.3); each streamed partial image adds 100 output tokens. Video billing is per render driven by model/size/seconds (see pricing page).
- **Async vs sync:** Image generation is synchronous (with optional streaming). Video generation is asynchronous — poll or webhook, then download within the URL TTL (1h for direct, 24h for batch).
- **Safety/gating:** GPT Image requires org verification; moderation param (`auto`/`low`) with `moderation_blocked` error details. Video enforces under-18-only content, no copyrighted characters/music, no real people, no human faces in input images, no human-likeness characters by default; uploaded-video editing is gated.
- **Deprecation:** Sora 2 models and Videos API shut down **2026-09-24**; legacy image `/remix` and DALL·E variations endpoints are deprecated/legacy.