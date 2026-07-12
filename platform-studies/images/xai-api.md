# xAI (Grok Imagine) API Analysis — Image & Video Generation

> **Base URL:** `https://api.x.ai/v1`
> **Docs:** `https://docs.x.ai/developers/model-capabilities/imagine`
> **Auth:** Bearer API key (`Authorization: Bearer $XAI_API_KEY`)
> **SDKs:** `xai_sdk` (Python, official), `@ai-sdk/xai` (Vercel AI SDK, TypeScript), OpenAI SDK (compatible for image generation), REST
> **Description:** xAI exposes a generative media surface called **Imagine** built on two Grok Imagine model families — **Grok Imagine Image** (text/image → image) and **Grok Imagine Video** (text/image/video → video). Unlike OpenAI and Google, the Imagine API surface is **generation-only**: there is no dedicated image/video *understanding* (vision) capability documented on these pages — Grok's multimodal *understanding* lives in the mainline Grok chat/completions models and is out of scope here. All image endpoints are **synchronous**; all video endpoints are **asynchronous** (start + poll), with the SDKs abstracting polling automatically. A **Files API** integration lets you reuse stored inputs and persist generated outputs (with optional permanent public URLs) within a single request.

---

## Table of Contents

1. [Platform Overview & Core Concepts](#1-platform-overview--core-concepts)
2. [API Architecture & Capability Matrix](#2-api-architecture--capability-matrix)
3. [Image Generation](#3-image-generation)
4. [Image Editing & Multi-Image Editing](#4-image-editing--multi-image-editing)
5. [Image Output Customization, Cost & Moderation](#5-image-output-customization-cost--moderation)
6. [Video Generation (Text & Image-to-Video)](#6-video-generation-text--image-to-video)
7. [Reference-to-Video, Video Editing & Extension](#7-reference-to-video-video-editing--extension)
8. [Video Job Lifecycle, Polling & Error Handling](#8-video-job-lifecycle-polling--error-handling)
9. [Files API Integration (Inputs & Outputs)](#9-files-api-integration-inputs--outputs)
10. [Capability Summary & Cross-Reference](#10-capability-summary--cross-reference)

---

## 1. Platform Overview & Core Concepts

xAI's Imagine API surface is organized around these core abstractions:

- **Grok Imagine Image models** — Dedicated image *generation/editing* models that take text and/or images as input and emit a raster image. The documented model is `grok-imagine-image-quality`. Image generation is **synchronous**: a single request returns one or more image URLs (or base64) directly. Edits reuse the same `sample()` method by adding an `image_url` / `image_file_id` parameter.
- **Grok Imagine Video models** — Dedicated video *generation/editing* models. Two versions are documented:
  - **`grok-imagine-video`** — the full-featured model supporting all video modes (text-to-video, image-to-video, reference-to-video, editing, extension) and all resolutions up to 720p (1080p not supported on this model).
  - **`grok-imagine-video-1.5`** — newer; supports **1080p** for image-to-video generation, but does **not** support reference-to-video mode.
- **Asynchronous video lifecycle** — All video requests are two-step: (1) **Start** — submit a generation/edit/extension request and receive a `request_id`; (2) **Poll** — repeatedly `GET /v1/videos/{request_id}` until `status` becomes `done`, `failed`, or `expired`. The xAI Python SDK and Vercel AI SDK abstract this entirely via `generate()` / `extend()`; REST users implement polling manually.
- **Request modes** — The video generation endpoint supports multiple modes determined by which fields are set. Only **one mode** can be active per request; mixing `image` + `reference_images`, or mixing AI SDK `mode` values, returns `400 Bad Request`.
- **Files API integration** — Bidirectional: (a) **Inputs** — anywhere an endpoint accepts a public URL or base64 image/video, you can substitute a `file_id` from your Files storage (keeping private files private, avoiding re-upload). (b) **Outputs** — pass `storage_options` on any Imagine request to persist the generated asset to Files storage and optionally create a permanent, shareable public URL alongside the ephemeral generation URL.
- **Moderation** — Generated media is subject to content policy review. Each response carries a `respect_moderation` flag; when `false`, the asset was filtered. Generated media is **not used for training**.
- **Enterprise compliance** — SOC 2 Type II, HIPAA eligible (BAA available), GDPR compliant with EU data residency, multi-region HA with custom SLAs, SSO/SAML, RBAC, and audit logging.

### Important scope note

The Imagine docs pages analyzed here cover **generation and editing only**. xAI's image/video *understanding* (vision) capabilities — where Grok models analyze and describe image/video inputs — are not part of the Imagine surface; they are consumed through the mainline Grok chat/completions API and are documented separately. This analysis therefore focuses on the generation/editing capabilities below.

### Capability pipelines

```
Image generation (sync)
   text-to-image:   prompt ──▶ /v1/images/generations ──▶ image URL(s) | base64
   image edit:      prompt + image (url|base64|file_id) ──▶ /v1/images/edits ──▶ image URL
   multi-image:     prompt + images[1..3] ──▶ /v1/images/edits ──▶ image URL

Video generation (async)
   text-to-video:     prompt ──▶ /v1/videos/generations ──▶ request_id ──▶ poll ──▶ video URL
   image-to-video:    prompt + image ──▶ /v1/videos/generations ──▶ request_id ──▶ poll ──▶ ...
   reference-to-video: prompt + reference_images[] ──▶ /v1/videos/generations ──▶ ... (grok-imagine-video only)
   video editing:     prompt + video ──▶ /v1/videos/edits ──▶ request_id ──▶ poll ──▶ ...
   video extension:   prompt + video ──▶ /v1/videos/extensions ──▶ request_id ──▶ poll ──▶ ...

Files integration
   input:   file_id (stored) substitutes for url/base64 on any endpoint
   output:  storage_options{filename, public_url} ──▶ file_output{file_id, public_url} + ephemeral url
```

---

## 2. API Architecture & Capability Matrix

| Surface | Endpoint / Method | Sync/Async | Purpose | Models |
|---|---|---|---|---|
| Image generation | `POST /v1/images/generations` | Sync | Text-to-image generation (1–10 images) | `grok-imagine-image-quality` |
| Image editing | `POST /v1/images/edits` | Sync | Edit one or more (up to 3) source images | `grok-imagine-image-quality` |
| Video generation | `POST /v1/videos/generations` | Async (start + poll) | Text-to-video, image-to-video, reference-to-video | `grok-imagine-video`, `grok-imagine-video-1.5` |
| Video editing | `POST /v1/videos/edits` | Async (start + poll) | Modify an existing video with a prompt | `grok-imagine-video` |
| Video extension | `POST /v1/videos/extensions` | Async (start + poll) | Continue a video from its last frame | `grok-imagine-video` |
| Video status | `GET /v1/videos/{request_id}` | Poll | Check video job status | — |
| Files (input) | substitute `file_id` for `url`/`base64` in any of the above | — | Reuse stored inputs | all |
| Files (output) | `storage_options` field on any of the above | — | Persist generated assets, create public URLs | all |
| Files management | `POST /v1/files`, `GET/DELETE /v1/files/{id}`, `*/public-url` | Sync | Upload, list, retrieve, delete, revoke/create public URLs | — |

### SDK method reference

| Capability | xAI Python SDK | Vercel AI SDK (`@ai-sdk/xai`) |
|---|---|---|
| Image (generate or edit, single) | `client.image.sample(prompt=..., model=..., image_url=..., image_file_id=..., n=...)` | `generateImage({ model: xai.image(...), prompt: "..." \| {text, images[]} })` |
| Image (batch, same prompt) | `client.image.sample_batch(prompt=..., n=4)` | `generateImage({ ..., n: 4 })` → `images[]` |
| Video generate | `client.video.generate(prompt=..., model=..., image_url=..., reference_image_urls=...)` | `generateVideo({ model: xai.video(...), prompt: "..." \| {image, text} })` |
| Video extend | `client.video.extend(prompt=..., model=..., video_url=..., duration=...)` | `generateVideo({ ..., providerOptions: { xai: { mode: "extend-video", videoUrl } } })` |
| Video manual start | `client.video.start(...)` / `client.video.extend_start(...)` | — |
| Video manual poll | `client.video.get(request_id)` | custom `abortSignal` |
| Files | `client.files.get/create_public_url/revoke_public_url/delete` | — |

---

## 3. Image Generation

### 3.1 Concepts

Generate new images from text prompts with Grok Imagine Image models. The API supports **batch generation** (up to 10 images per request via `n`), and control over **aspect ratio**, **resolution**, and **response format** (URL or base64). The default model is `grok-imagine-image-quality`.

### 3.2 Endpoint

`POST /v1/images/generations`

### 3.3 Request parameters

| Parameter | Type | Required | Description |
|---|---|---|---|
| `model` | string | yes | Image model, e.g. `grok-imagine-image-quality` |
| `prompt` | string | yes | Text description of the desired image |
| `n` | integer | no | Number of images to generate (up to 10). Default 1. SDK batch method: `sample_batch(n=4)` |
| `aspect_ratio` | string | no | Output dimensions (see table below). `auto` lets the model choose |
| `resolution` | string | no | `1k` or `2k` |
| `response_format` | string | no | `url` (default) or `b64_json` |
| `storage_options` | object | no | Persist output to Files (see §9.2) |

### 3.4 Aspect ratios (image)

| Ratio | Use case |
|---|---|
| `1:1` | Social media, thumbnails |
| `16:9` / `9:16` | Widescreen, mobile, stories |
| `4:3` / `3:4` | Presentations, portraits |
| `3:2` / `2:3` | Photography |
| `2:1` / `1:2` | Banners, headers |
| `19.5:9` / `9:19.5` | Modern smartphone displays |
| `20:9` / `9:20` | Ultra-wide displays |
| `auto` | Model auto-selects the best ratio for the prompt |

### 3.5 Example (Python SDK)

```python
import xai_sdk

client = xai_sdk.Client()

response = client.image.sample(
    prompt="A collage of London landmarks in a stenciled street‑art style",
    model="grok-imagine-image-quality",
    aspect_ratio="16:9",
    resolution="2k",
)
print(response.url)
```

### 3.6 Batch generation (same prompt)

```python
responses = client.image.sample_batch(
    prompt="A futuristic city skyline at night",
    model="grok-imagine-image-quality",
    n=4,
)
for i, image in enumerate(responses):
    print(f"Variation {i + 1}: {image.url}")
```

For **different prompts** in parallel, use `AsyncClient` with `asyncio.gather` to fire concurrent `sample()` calls — faster than sequential requests.

### 3.7 Base64 output

Set `image_format="base64"` (xAI SDK) or `response_format="b64_json"` (REST/OpenAI SDK) to receive base64-encoded image bytes directly, useful for embedding without a download step.

### 3.8 OpenAI SDK compatibility

The OpenAI SDK's `images.generate()` method is supported for generation by pointing `base_url` at `https://api.x.ai/v1`. Aspect ratio and resolution are passed via `extra_body`. Note: `images.edit()` is **not** supported because it uses `multipart/form-data` and the xAI API requires `application/json`.

---

## 4. Image Editing & Multi-Image Editing

### 4.1 Concepts

Edit an existing image by providing a source image plus a natural-language prompt. The model understands the image content and applies only the requested changes. Supports **multi-turn editing** (chain edits by feeding each output as the next input) and **style transfer** (describe the desired aesthetic — realistic, anime, oil painting, pencil sketch, etc.).

### 4.2 Endpoint

`POST /v1/images/edits`

> The xAI Python SDK reuses the **same** `sample()` method for editing — just add an `image_url` (or `image_file_id`) parameter. The REST endpoint is distinct (`/v1/images/edits`).

### 4.3 Single-image edit parameters

| Parameter | Type | Required | Description |
|---|---|---|---|
| `model` | string | yes | e.g. `grok-imagine-image-quality` |
| `prompt` | string | yes | Description of the desired edit |
| `image` | object | yes* | Source image: `{ "url": "...", "type": "image_url" }` or `{ "file_id": "..." }` |
| `response_format` | string | no | `url` (default) or `b64_json` |
| `storage_options` | object | no | Persist output (see §9.2) |

For a single-image edit, the output aspect ratio **inherits** the input image's aspect ratio; the `aspect_ratio` parameter is not applied. With multiple images, `aspect_ratio` can be overridden (defaults to the first input image's ratio).

### 4.4 Multi-image editing (up to 3 sources)

Combine up to **3 source images** in a single edit request for compositing subjects, transferring styles, and building scenes from multiple references. Each source image can independently be a public URL, a base64 data URI, or a `file_id` — you can **mix kinds** within a single request.

| Parameter | Type | Required | Description |
|---|---|---|---|
| `model` | string | yes | e.g. `grok-imagine-image-quality` |
| `prompt` | string | yes | Edit description |
| `images` | array | yes (multi) | Array of 1–3 image objects, each `{url}` or `{file_id}` independently |
| `aspect_ratio` | string | no | Override; defaults to the first input image's ratio (e.g. `"1:1"`, `"16:9"`) |
| `response_format` | string | no | `url` (default) or `b64_json` |
| `storage_options` | object | no | Persist output (see §9.2) |

### 4.5 Input methods (all image/video endpoints)

| Method | Format | Notes |
|---|---|---|
| Public URL | `{"url": "https://..."}` | Must be publicly accessible |
| Base64 data URI | `data:image/png;base64,...` | Inline; SDK encodes from file bytes |
| Files API `file_id` | `{"file_id": "file_..."}` | Private; fetched server-side. Mix kinds within a request |

Supported content types: images PNG/JPEG/WebP; videos MP4.

### 4.6 Example (REST, public URL)

```bash
curl -X POST https://api.x.ai/v1/images/edits \
  -H "Content-Type: application/json" \
  -H "Authorization: Bearer $XAI_API_KEY" \
  -d '{
    "model": "grok-imagine-image-quality",
    "prompt": "Render this as a pencil sketch with detailed shading",
    "image": {
      "url": "https://docs.x.ai/assets/api-examples/images/style-realistic.png",
      "type": "image_url"
    }
  }'
```

---

## 5. Image Output Customization, Cost & Moderation

### 5.1 Response object (xAI SDK)

| Field | Description |
|---|---|
| `response.url` | Temporary image URL (download promptly) |
| `response.image` | Base64 bytes when `image_format="base64"` |
| `response.model` | Actual model used (resolves aliases) |
| `response.respect_moderation` | `true` if the image passed content moderation; `false` if filtered |
| `response.file_output` | Files API output block (when `storage_options` set) — see §9.2 |
| `response.public_url` | Permanent shareable URL (when requested) |

### 5.2 Moderation

Every generated image passes content-policy review. When `respect_moderation` is `false`, the asset was filtered and the URL should not be used. Generated media is **not used for training**.

### 5.3 Pricing model

- **Image generation** — flat **per-image** pricing regardless of prompt length. Each generated image incurs a fixed fee (so `n=4` costs 4×).
- **Image editing** — billed for **both** the input image and the generated output image.
- **Video generation** — **per-second** pricing; both duration and resolution affect total cost.
- See the [pricing page](https://docs.x.ai/developers/pricing#imagine-api-pricing) for current rates.

### 5.4 Concurrency

For multiple **different prompts**, use `AsyncClient` + `asyncio.gather` (Python) or `Promise.all` (AI SDK). For multiple **variations of the same prompt**, use `sample_batch(n=...)` in a single request — more efficient than parallel single-image calls.

---

## 6. Video Generation (Text & Image-to-Video)

### 6.1 Concepts

Generate videos from text prompts (text-to-video) or animate a still image (image-to-video). Video generation is **asynchronous**: start a request, poll with the returned `request_id`, and use the completed video URL when ready. The SDKs handle polling automatically.

### 6.2 Endpoint

`POST /v1/videos/generations` (start) → `GET /v1/videos/{request_id}` (poll)

### 6.3 Request parameters

| Parameter | Type | Required | Description |
|---|---|---|---|
| `model` | string | yes | `grok-imagine-video` or `grok-imagine-video-1.5` |
| `prompt` | string | yes | Text description (text-to-video, or animation instructions for image-to-video) |
| `image` | object | no | Source image for image-to-video: `{url}` or `{file_id}`. Becomes the first frame |
| `reference_images` | array | no | Reference images for reference-to-video: `[{url}|{file_id}, ...]`. Guides content without locking the first frame |
| `duration` | integer | no | Length in seconds, **1–15**. Default applies if omitted |
| `aspect_ratio` | string | no | See table below. For image-to-video, defaults to input image ratio; specifying it stretches the image |
| `resolution` | string | no | `480p` (default), `720p`, or `1080p` (1080p only on `grok-imagine-video-1.5` image-to-video) |
| `storage_options` | object | no | Persist output (see §9.2) |

### 6.4 Aspect ratios (video)

| Ratio | Use case |
|---|---|
| `1:1` | Social media, thumbnails |
| `16:9` / `9:16` | Widescreen, mobile, stories (default `16:9`) |
| `4:3` / `3:4` | Presentations, portraits |
| `3:2` / `2:3` | Photography |

### 6.5 Resolution (video)

| Resolution | Description |
|---|---|
| `1080p` | Full HD. **Only** supported on `grok-imagine-video-1.5` for image-to-video |
| `720p` | HD quality |
| `480p` | Standard definition, faster processing (default) |

### 6.6 Request modes (video)

The generation endpoint supports multiple modes, determined by which fields are set. Only **one mode** per request:

| Mode | REST fields | AI SDK shape | Description |
|---|---|---|---|
| Text-to-video | `prompt` only | `prompt: "..."` | Generate from text alone |
| Image-to-video | `prompt` + `image` | `prompt: { image, text }` | Image becomes the starting frame |
| Reference-to-video | `prompt` + `reference_images` | `prompt: "..."` + `providerOptions.xai.{ mode: "reference-to-video", referenceImageUrls }` | Guide with reference images (first frame not locked) |
| Edit-video | `/v1/videos/edits` + `video` | `prompt: "..."` + `providerOptions.xai.{ mode: "edit-video", videoUrl }` | Modify an existing video |
| Extend-video | `/v1/videos/extensions` + `video` | `prompt: "..."` + `providerOptions.xai.{ mode: "extend-video", videoUrl }` | Continue from the last frame |

**Disallowed combinations** (return `400 Bad Request`):
- `image` + `reference_images` — use one or the other.
- Mixing `mode` values in the AI SDK — exactly one of `"edit-video"`, `"extend-video"`, or `"reference-to-video"` per request.

### 6.7 Example (text-to-video, REST with polling)

```bash
# Start
REQUEST_ID=$(curl -s -X POST https://api.x.ai/v1/videos/generations \
  -H "Content-Type: application/json" \
  -H "Authorization: Bearer $XAI_API_KEY" \
  -d '{
    "model": "grok-imagine-video",
    "prompt": "Timelapse of a flower blooming in a sunlit garden",
    "duration": 10,
    "aspect_ratio": "16:9",
    "resolution": "720p"
  }' | jq -r '.request_id')

# Poll
while true; do
  RESULT=$(curl -s https://api.x.ai/v1/videos/$REQUEST_ID \
    -H "Authorization: Bearer $XAI_API_KEY")
  STATUS=$(echo "$RESULT" | jq -r '.status')
  if [ "$STATUS" = "done" ]; then echo "$RESULT" | jq -r '.video.url'; break; fi
  if [ "$STATUS" = "failed" ] || [ "$STATUS" = "expired" ]; then echo "$RESULT" | jq; break; fi
  sleep 5
done
```

### 6.8 Example (image-to-video, Python SDK — polling automatic)

```python
import os, xai_sdk

client = xai_sdk.Client(api_key=os.getenv("XAI_API_KEY"))

response = client.video.generate(
    prompt="Make the water crash down and slowly pan out the camera",
    model="grok-imagine-video-1.5",
    image_url="https://docs.x.ai/assets/api-examples/video/waterfall-still.png",
    duration=12,
)
print(response.url)
```

In the Vercel AI SDK, image-to-video uses `prompt: { image, text }` where `image` can be a URL string, base64 string, `Uint8Array`, `ArrayBuffer`, or `Buffer`.

---

## 7. Reference-to-Video, Video Editing & Extension

### 7.1 Reference-to-Video

Provide one or more **reference images** to incorporate specific people, objects, clothing, or other visual elements into the generated video. Unlike image-to-video (where the source image becomes the first frame), reference images **influence** what appears without locking the first frame. Use cases: virtual try-on, product placement, character-consistent storytelling.

- Reference images can be referenced in the prompt via `<IMAGE_1>`, `<IMAGE_2>`, `<IMAGE_3>` placeholders.
- Each reference image can be a public HTTPS URL, base64 data URI, or `file_id` — mix kinds within a request.
- **Model constraint:** requires `grok-imagine-video` — `grok-imagine-video-1.5` does **not** support this mode.

```python
response = client.video.generate(
    prompt="the model from <IMAGE_1> walks in... they wear the shirt from <IMAGE_2>...",
    model="grok-imagine-video",
    reference_image_urls=["<IMAGE_URL_1>", "<IMAGE_URL_2>", "<IMAGE_URL_3>"],
    duration=10,
    aspect_ratio="16:9",
    resolution="720p",
)
```

AI SDK equivalent: set `providerOptions.xai.mode = "reference-to-video"` and pass `providerOptions.xai.referenceImageUrls`.

### 7.2 Video Editing

Modify an existing video with a text prompt while **preserving the rest of the scene**. `grok-imagine-video` delivers high-fidelity edits, changing only what you ask for.

- **Endpoint:** `POST /v1/videos/edits`
- **Input:** `video` object (`{url}` or `{file_id}`) — public URL, base64 data URI, or Files API `file_id`.
- **Python SDK:** `client.video.generate(prompt=..., model=..., video_url=..., video_file_id=...)`.
- **AI SDK:** `providerOptions.xai.mode = "edit-video"` + `providerOptions.xai.videoUrl`.
- **Inherited properties:** `duration`, `aspect_ratio`, and `resolution` are **ignored** — the output inherits them from the input video. Resolution is capped at **720p** (a 1080p input is downsized). Duration is capped at **8.7 seconds**.
- Supports **concurrent edits** from the same source video (branch multiple edits using `asyncio.gather` / `Promise.all`).

### 7.3 Video Extension

Continue an existing video from its **last frame**. The result is a single video that picks up seamlessly and continues with generated content.

- **Endpoint:** `POST /v1/videos/extensions`
- **Input:** `video` object (`{url}` or `{file_id}`).
- **Python SDK:** `client.video.extend(prompt=..., model=..., video_url=..., duration=...)`.
- **AI SDK:** `providerOptions.xai.mode = "extend-video"` + `providerOptions.xai.videoUrl`.
- **`duration` controls only the extended portion** — not the total output. E.g., a 10s input + `duration=5` → 15s output.
- Same asynchronous polling pattern as generation; the AI SDK returns the URL in `providerMetadata.xai.videoUrl`.

### 7.4 Example (extension, Python SDK)

```python
response = client.video.extend(
    prompt="The shot pans to an over the shoulder perspective. Calm controlled scene.",
    model="grok-imagine-video",
    video_url="<VIDEO_URL>",
    duration=10,
)
print(response.url)
```

---

## 8. Video Job Lifecycle, Polling & Error Handling

### 8.1 How it works (two-step)

1. **Start** — Submit a generation/edit/extension request → receive `{"request_id": "..."}`.
2. **Poll** — `GET /v1/videos/{request_id}` every few seconds until `status` is terminal.

### 8.2 Status values

| Status (REST) | Proto enum (SDK) | Description |
|---|---|---|
| `pending` | `DeferredStatus.PENDING` | Still generating |
| `done` | `DeferredStatus.DONE` | Video ready |
| `expired` | `DeferredStatus.EXPIRED` | Request expired |
| `failed` | `DeferredStatus.FAILED` | Generation failed (see `error`) |

### 8.3 Completed response shape

```json
{
  "status": "done",
  "video": {
    "url": "https://vidgen.x.ai/.../video.mp4",
    "duration": 8,
    "respect_moderation": true
  },
  "model": "grok-imagine-video"
}
```

Videos are returned as **temporary URLs** on `imgen.x.ai` / `vidgen.x.ai`. Download promptly if you need to keep a copy, or use `storage_options` (§9.2) to persist.

### 8.4 Generation time factors

- **Prompt complexity** — more detailed scenes take longer.
- **Duration** — longer videos take more time.
- **Resolution** — 720p vs 480p increases processing time.
- **Video editing** — adds overhead vs image-to-video or text-to-video.

### 8.5 SDK polling customization

| Python SDK | AI SDK (`providerOptions.xai`) | Description | Default |
|---|---|---|---|
| `timeout` | `pollTimeoutMs` | Max wait for completion | 10 minutes |
| `interval` | `pollIntervalMs` | Time between status checks | 100 ms |

```python
from datetime import timedelta
response = client.video.generate(
    prompt="Epic cinematic drone shot flying through mountain peaks",
    model="grok-imagine-video",
    duration=15,
    timeout=timedelta(minutes=15),
    interval=timedelta(seconds=5),
)
```

On timeout, the Python SDK raises `TimeoutError`; the AI SDK aborts via `AbortSignal`. For finer control, use manual `start()` + `get()` polling (Python) or a custom `abortSignal` (AI SDK).

### 8.6 Error handling

When using `generate()` / `extend()`, failures raise `VideoGenerationError` (import from `xai_sdk.video`) with `code` and `message` attributes. Manual polling returns `status: "failed"` with an `error` object:

```json
{
  "status": "failed",
  "error": { "code": "invalid_argument", "message": "Prompt cannot be empty..." }
}
```

| `error.code` | Meaning | What to do |
|---|---|---|
| `invalid_argument` | Invalid input (unsupported duration, invalid media, prompt too long, conflicting modes, moderation block) | Fix params/input; resubmit |
| `permission_denied` | API key/team lacks permission for the operation | Confirm team access |
| `failed_precondition` | Operation unavailable for the model/settings (e.g. editing, extension, unsupported resolution) | Change model/mode/resolution |
| `service_unavailable` | Temporarily overloaded | Retry later |
| `internal_error` | Internal failure | Retry; contact xAI support with `request_id` if persistent |

Authentication errors, missing models, and rate limits are returned **synchronously** before a video job is created, so they don't appear in `error.code` of a failed video result.

### 8.7 Concurrent video requests

For multiple videos, run requests concurrently via `AsyncClient` + `asyncio.gather` (Python) or `Promise.all` (AI SDK) — useful for comparing prompts or creating variations.

---

## 9. Files API Integration (Inputs & Outputs)

The Imagine API integrates with the [Files API](https://docs.x.ai/developers/files) in two directions, enabling chaining of Imagine calls without ever leaving Files storage.

### 9.1 Inputs — referencing stored files

Anywhere an endpoint accepts a public URL or base64 image/video, substitute a `file_id` from your Files storage. The file is fetched server-side, so:

- No bandwidth re-uploading the same image — useful for iterative editing loops.
- The original file stays **private** (no need to make it public).
- Works with both uploaded files and assets from earlier Imagine calls (via `storage_options` outputs).

The referenced file must be the correct content type (images: PNG/JPEG/WebP; videos: MP4) and fully uploaded. You can **mix** `url` and `file_id` kinds within a single multi-image or reference-images request.

| Capability | Python SDK parameter | REST field |
|---|---|---|
| Edit one stored image | `image_file_id` | `image: {file_id}` |
| Edit multiple stored images | `image_file_ids` | `images: [{file_id}, {url}, ...]` |
| Image-to-video from stored frame | `image_file_id` | `image: {file_id}` |
| Edit stored video | `video_file_id` | `video: {file_id}` |
| Reference-to-video with stored images | `reference_image_file_ids` | `reference_images: [{file_id}, ...]` |

### 9.2 Outputs — persisting generated assets

Pass `storage_options` on **any** Imagine request (image generation, image edit, video generation/edit/extension) to persist the generated asset to Files storage and optionally create a permanent public URL. The ephemeral generation URL is always returned regardless.

#### `storage_options` reference

| Field | Type | Description |
|---|---|---|
| `filename` | string (**required**) | Filename for the stored file; the public URL path extension is derived from it |
| `expires_after` | integer (optional) | Seconds until the stored file auto-deletes. Range `3600` (1h) – `2592000` (30d). Omit for permanent storage |
| `public_url` | boolean \| object (optional) | `true` for default public URL, or an object `{expires_after}` to configure. Omit/`false` to store privately |
| `public_url.expires_after` | integer (optional) | Seconds until the public URL auto-revokes. Range 1h–30d. Must be ≤ the file's `expires_after`. If omitted, inherits the file's expiry |

#### `file_output` response block

| Field | Always present | Meaning |
|---|---|---|
| `file_id` | yes | Stable Files API identifier for the generated asset |
| `filename` | yes | The filename you provided |
| `expires_at` | only when file has expiry | Unix timestamp when the file expires |
| `public_url` | only when requested & succeeded | Permanent shareable URL |
| `public_url_expires_at` | only when public URL has expiry | Unix timestamp when URL dies (absent for permanent URLs) |
| `public_url_error` | only on partial failure | Human-readable error when storage succeeded but public URL creation failed |

The Python SDK also surfaces `response.public_url` and `response.public_url_error` as top-level shortcuts.

#### Expiry behaviour

- `storage_options.expires_after` controls the **stored file** auto-delete.
- `storage_options.public_url.expires_after` controls the **public URL** auto-revoke.
- A public URL can **never outlive its file**; both values must be between 1h and 30d.
- If `public_url.expires_after` is omitted, the URL inherits the file's expiry (or never expires if the file is permanent).

#### Multiple outputs (`n > 1`)

Each image gets its own independent `file_id` and `public_url` with a unique token. Revoking or deleting one does not affect the others.

#### Video outputs

Because video generation is asynchronous, `file_output.public_url` is populated on the **completed** response after the video finishes generating (the SDK returns it after polling completes).

#### Public URL errors

If storage succeeds but public URL creation fails (transient issue or quota hit), the response includes `file_output.file_id` + the original asset URL, and `public_url` is replaced by `public_url_error`. Retry public URL creation directly via `client.files.create_public_url(file_id)` without regenerating the asset.

#### Limitations

- Up to **1,000 active public URLs per team**; hitting the cap sets `public_url_error` — revoke unused URLs to free slots.
- Custom `filename` affects the public URL path extension (e.g. `my-cover.png` → `.png`), but the stored content type is determined by the generated asset.
- `response_format` does not affect storage — `storage_options` runs whether you request `url` or `b64_json`.
- Public URLs are independent of the ephemeral URL — revoking the public URL does not affect the ephemeral URL.
- All validation is **synchronous** — invalid storage configs are rejected before generation starts.

### 9.3 Managing files created via Imagine

Files created through `storage_options` are first-class Files API files. Use the Files API to list, retrieve, update, delete, and manage public URLs:

| Operation | Python SDK | REST |
|---|---|---|
| Inspect | `client.files.get(file_id)` | `GET /v1/files/{id}` |
| Revoke public URL | `client.files.revoke_public_url(file_id)` | `POST /v1/files/{id}/public-url/revoke` |
| Create/recreate public URL | `client.files.create_public_url(file_id, expires_after=...)` | `POST /v1/files/{id}/public-url` |
| Delete file (also revokes URL) | `client.files.delete(file_id)` | `DELETE /v1/files/{id}` |

### 9.4 Capstone: iterative loop without leaving Files

```python
import os, xai_sdk

client = xai_sdk.Client(api_key=os.getenv("XAI_API_KEY"))

# 1. Generate an image and store it privately
gen = client.image.sample(
    prompt="A futuristic city skyline at night",
    model="grok-imagine-image-quality",
    storage_options={"filename": "city.jpg"},
)
city = gen.file_output.file_id

# 2. Edit the stored image without re-uploading
edit = client.image.sample(
    prompt="Add neon signs to the buildings",
    model="grok-imagine-image-quality",
    image_file_id=city,
    storage_options={"filename": "city-neon.jpg"},
)

# 3. Animate the edited result into a video
vid = client.video.generate(
    prompt="A camera pulls back through the city",
    model="grok-imagine-video",
    duration=5,
    image_file_id=edit.file_output.file_id,
)
print(vid.url)
```

---

## 10. Capability Summary & Cross-Reference

| # | Capability | Endpoint | Sync/Async | Input | Output | Key parameters | Model(s) |
|---|---|---|---|---|---|---|---|
| 1 | Image generation | `POST /v1/images/generations` | Sync | text prompt | image URL/base64 (1–10) | `n`, `aspect_ratio`, `resolution` (`1k`/`2k`), `response_format`, `storage_options` | `grok-imagine-image-quality` |
| 2 | Image editing (single) | `POST /v1/images/edits` | Sync | prompt + 1 image | image URL/base64 | `image` (url/base64/file_id); aspect ratio inherited | `grok-imagine-image-quality` |
| 3 | Multi-image editing | `POST /v1/images/edits` | Sync | prompt + 1–3 images | image URL/base64 | `images[]` (mix url/file_id), `aspect_ratio` (override; defaults to first input) | `grok-imagine-image-quality` |
| 4 | Text-to-video | `POST /v1/videos/generations` | Async | text prompt | video URL (MP4) | `duration` (1–15s), `aspect_ratio`, `resolution` (`480p`/`720p`/`1080p`*) | `grok-imagine-video`, `grok-imagine-video-1.5` |
| 5 | Image-to-video | `POST /v1/videos/generations` | Async | prompt + image | video URL | `image` (first frame), `duration`, `aspect_ratio` (defaults to image), `resolution` | both; `1080p` only on `1.5` |
| 6 | Reference-to-video | `POST /v1/videos/generations` | Async | prompt + reference_images[] | video URL | `reference_images[]` (`<IMAGE_N>` placeholders), `duration`, `aspect_ratio`, `resolution` | `grok-imagine-video` only |
| 7 | Video editing | `POST /v1/videos/edits` | Async | prompt + video | video URL | `video` (url/file_id); duration/aspect/resolution **inherited** (cap 720p, 8.7s) | `grok-imagine-video` |
| 8 | Video extension | `POST /v1/videos/extensions` | Async | prompt + video | video URL | `video`, `duration` (extension only; total = input + extension) | `grok-imagine-video` |
| 9 | Files input | substitute `file_id` in any above | — | stored file | — | `image_file_id`, `image_file_ids`, `video_file_id`, `reference_image_file_ids` | all |
| 10 | Files output | `storage_options` on any above | — | — | `file_output{file_id, public_url}` + ephemeral URL | `filename` (req), `expires_after`, `public_url` (bool\|obj) | all |

\* `1080p` only supported on `grok-imagine-video-1.5` for image-to-video generation.

### Cross-capability notes

- **One model for all image tasks** — `grok-imagine-image-quality` handles both generation and editing (single and multi-image), via the same `sample()` SDK method.
- **Two video models with mode constraints** — `grok-imagine-video` supports all five video modes; `grok-imagine-video-1.5` adds 1080p image-to-video but drops reference-to-video.
- **Single request = single mode** — mixing `image` + `reference_images`, or mixing AI SDK `mode` values, returns `400 Bad Request`.
- **Files API as the connective tissue** — stored `file_id`s can flow from one Imagine call to the next (generate → edit → animate) without re-uploading or making files public, enabling efficient iterative pipelines.
- **No vision/understanding surface on Imagine** — image/video understanding is provided by mainline Grok models via the chat/completions API, separate from the Imagine generation surface documented here.
- **Sync vs async split** — all image endpoints are synchronous and return directly; all video endpoints are asynchronous (start + poll), with SDKs abstracting polling via configurable timeout/interval.