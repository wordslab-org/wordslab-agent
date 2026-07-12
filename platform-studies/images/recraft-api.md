# Recraft API Analysis — Image Generation, Editing & Vectorization

> **Base URL:** `https://external.api.recraft.ai/v1` (REST) | **Swagger:** `https://external.api.recraft.ai/doc/#/`
> **Docs:** `https://www.recraft.ai/docs/api-reference/getting-started` | **Product:** `https://www.recraft.ai/api`
> **Auth:** Bearer API token (`Authorization: Bearer $RECRAFT_API_TOKEN`) — generated from the profile page; requires a non-zero API units balance
> **SDKs:** OpenAI Python library (`openai`) is API-compatible; any HTTP client (`curl`) works since Recraft follows REST principles
> **MCP:** Local server (bills API units, same prices as API) and Remote server (bills subscription credits, same prices as web platform)
> **Description:** Recraft is a design-focused AI image platform exposing a REST API for raster and vector image generation, prompt-based editing (image-to-image, inpainting, outpainting, background replace/generate), format conversion (vectorize, remove background), upscaling (crisp / creative), region erase, remix/exploration, style creation, and prompt enhancement. The API exposes **only Recraft's proprietary model family** (V2, V3, V4, V4.1) — external third-party image/video models available in Recraft Studio (GPT Image, Flux, Nano Banana, Veo, Kling, Sora, etc.) are **not** accessible via the API. Video generation is a **Studio-only** capability with no public API endpoint.

---

## Table of Contents

1. [Platform Overview & Core Concepts](#1-platform-overview--core-concepts)
2. [API Architecture & Capability Matrix](#2-api-architecture--capability-matrix)
3. [Models — Catalog & Selection Guide](#3-models--catalog--selection-guide)
4. [Styles — Curated & Custom](#4-styles--curated--custom)
5. [Image Generation (Text → Image)](#5-image-generation-text--image)
6. [Image-to-Image (Variations)](#6-image-to-image-variations)
7. [Inpainting (Masked Region Regeneration)](#7-inpainting-masked-region-regeneration)
8. [Outpainting (Border Extension)](#8-outpainting-border-extension)
9. [Background Operations — Replace & Generate](#9-background-operations--replace--generate)
10. [Vectorization (Raster → SVG)](#10-vectorization-raster--svg)
11. [Background Removal](#11-background-removal)
12. [Upscaling — Crisp & Creative](#12-upscaling--crisp--creative)
13. [Erase Region](#13-erase-region)
14. [Remix / Variate Image](#14-remix--variate-image)
15. [Explore & Explore Similar](#15-explore--explore-similar)
16. [Enhance Prompt](#16-enhance-prompt)
17. [Style Creation](#17-style-creation)
18. [Controls & Text Layout (Auxiliary Parameters)](#18-controls--text-layout-auxiliary-parameters)
19. [Sizes, Limits & Pricing](#19-sizes-limits--pricing)
20. [Video Generation — Studio-Only Note](#20-video-generation--studio-only-note)
21. [MCP Tools Reference](#21-mcp-tools-reference)
22. [Capability Summary & Cross-Reference](#22-capability-summary--cross-reference)

---

## 1. Platform Overview & Core Concepts

### Main Concepts

Recraft's API is organized around these core abstractions:

- **Model** — The proprietary image generation/editing model identified by a string ID (e.g. `recraftv4_1`, `recraftv3_vector`). Four generations exist (V2, V3, V4, V4.1), each with raster, vector, standard (1MP) and Pro (4MP) variants. The model determines the output format (raster → PNG/WEBP/JPG, vector → SVG), the maximum prompt length, and the supported image sizes.
- **Style** — A named visual aesthetic applied to generated images (e.g. `Photorealism`, `Illustration`, `Vector art`, `Icon`). Provided either as a curated `style` name string or as a custom `style_id` (UUID returned by the Create style endpoint). Styles are **only supported on V2 and V3 models** — V4/V4.1 models do not accept `style`/`style_id`.
- **Custom Style (style_id)** — A reusable style reference created by uploading up to 5 reference images (or by mixing existing styles). Bound to a base style category and model. Compatible with V3 / V3 Vector when created via API; web-created styles are bound to the model specified at creation.
- **Strength** — A float in `[0, 1]` used by image-to-image to control how different the output is from the source: `0` ≈ near-identical, `1` ≈ minimal similarity.
- **Mask** — A grayscale image (pure black `0` / pure white `255` per pixel) of identical dimensions to the source image. White pixels mark regions to modify (inpaint/generate/erase); black pixels mark regions to keep. Required by inpainting, generate-background, and erase-region.
- **Controls** — An optional object of generation tweaks: `colors` (preferred palette), `background_color`, `artistic_level` (V3 only, 0–5), `no_text` (V3 only). Supported across all models, partially for V4.
- **Text Layout** — An array of `{text, bbox}` objects placing individual words as 4-point polygons with relative coordinates (0,0 top-left to 1,1 bottom-right). V3/V3 Vector only. Text is limited to a supported character set.
- **API units** — The prepaid billing currency; 1,000 units = $1.00. Each operation deducts units per image/request (see §19). The `/users/me` endpoint returns the remaining balance.
- **response_format** — How outputs are returned: `url` (default, public signed link stored ~24h) or `b64_json` (inline base64).

### Image Input: Multipart vs JSON

Every endpoint taking an image (or mask) accepts two interchangeable request formats:

- **`multipart/form-data`** — upload binary files directly (`image`, `mask`, `file` fields). Convenient for local files.
- **`application/json`** — pass images by reference via a public URL or `data:image/...;base64,...` data URL.

| Multipart field        | JSON field   | Type                            |
| ---------------------- | ------------ | ------------------------------- |
| `image`                | `image_url`  | string — URL or data URL        |
| `mask`                 | `mask_url`   | string — URL or data URL of mask |
| `file`                 | `image_url`  | string — URL or data URL        |
| `files` (Create style) | `image_urls` | array of strings (URLs/data URLs) |

All other parameters are identical between the two formats.

### Capability pipelines

```
Text → Image
   prompt [+ style | style_id] [+ model] [+ size] [+ controls] [+ text_layout]
        ──▶ POST /v1/images/generations[/raster|/vector] ──▶ data[].url | b64_json

Image → Image (variation / edit)
   image + prompt + strength ──▶ POST /v1/images/imageToImage ──▶ data[].url
   image + mask + prompt     ──▶ POST /v1/images/inpaint        ──▶ data[].url
   image + prompt + expand_* | size | zoom ──▶ POST /v1/images/outpaint ──▶ data[].url
   image + prompt            ──▶ POST /v1/images/replaceBackground ──▶ data[].url
   image + mask + prompt     ──▶ POST /v1/images/generateBackground ──▶ data[].url

Image → Format conversion / cleanup
   image ──▶ POST /v1/images/vectorize        ──▶ image.url (SVG)
   image ──▶ POST /v1/images/removeBackground  ──▶ image.url (transparent cutout)
   image ──▶ POST /v1/images/crispUpscale      ──▶ image.url
   image ──▶ POST /v1/images/creativeUpscale   ──▶ image.url
   image + mask ──▶ POST /v1/images/eraseRegion ──▶ image.url

Image → Variation / exploration
   image + size ──▶ POST /v1/images/variateImage ──▶ data[].url   (Remix, no prompt)
   prompt       ──▶ POST /v1/images/explore       ──▶ data[].url   (grid of variations)
   source_image_id + similarity ──▶ POST /v1/images/explore/similar ──▶ data[].url

Prompt → Prompt
   prompt ──▶ POST /v1/prompts/enhance ──▶ enhanced_prompt

Images → Style
   files | image_urls + style ──▶ POST /v1/styles ──▶ {id, ...}  (reuse via style_id)
```

---

## 2. API Architecture & Capability Matrix

### Endpoints at a glance

| Capability | Method | Endpoint | Modality | Output | Models |
|---|---|---|---|---|---|
| Generate image | POST | `/v1/images/generations` (base, `/raster`, `/vector`) | text → image | `data[].url` | All 16 |
| Image to image | POST | `/v1/images/imageToImage` | image + text → image | `data[].url` | V3 / V4 family |
| Inpainting | POST | `/v1/images/inpaint` | image + mask + text → image | `data[].url` | V3 / V3 Vector |
| Outpainting | POST | `/v1/images/outpaint` | image + text → image | `data[].url` | V3 / V3 Vector |
| Replace background | POST | `/v1/images/replaceBackground` | image + text → image | `data[].url` | V3 / V3 Vector |
| Generate background | POST | `/v1/images/generateBackground` | image + mask + text → image | `data[].url` | V3 / V3 Vector |
| Vectorize | POST | `/v1/images/vectorize` | image → SVG | `image.url` | n/a |
| Remove background | POST | `/v1/images/removeBackground` | image → image | `image.url` | n/a |
| Crisp upscale | POST | `/v1/images/crispUpscale` | image → image | `image.url` | n/a |
| Creative upscale | POST | `/v1/images/creativeUpscale` | image → image | `image.url` | n/a |
| Erase region | POST | `/v1/images/eraseRegion` | image + mask → image | `image.url` | n/a |
| Remix / variate | POST | `/v1/images/variateImage` | image → image(s) | `data[].url` | model-determined |
| Explore | POST | `/v1/images/explore` | text → image(s) | `data[].url` | V4 / V4.1 family |
| Explore similar | POST | `/v1/images/explore/similar` | image_id → image(s) | `data[].url` | V4 / V4.1 family |
| Enhance prompt | POST | `/v1/prompts/enhance` | text → text | `enhanced_prompt` | n/a |
| Create style | POST | `/v1/styles` | images → style_id | `{id, ...}` | V3 / V3 Vector (API-created) |
| Get user info | GET | `/v1/users/me` | — → account | `{credits, email, id, name}` | n/a |

### Key architectural notes

- **REST, synchronous.** All endpoints return the final image directly (no job polling, no webhooks). The marketing page mentions async/batch workloads for high-volume use, but the documented endpoints are synchronous single-request calls.
- **OpenAI Python library compatibility.** `client.images.generate(...)` works for the generations endpoint; non-standard params (`style_id`, `controls`, `text_layout`) are passed via `extra_body`. Image-to-image, inpainting, etc. use `client.post(path=..., cast_to=object, ...)` since they are not part of the OpenAI Images surface.
- **Raster vs vector enforced variants.** `/generations/raster` and `/generations/vector` reject mismatched model/style pairs, useful for agent workflows that must produce a single output type. The base `/generations` endpoint accepts any supported model/style.
- **No image *understanding* endpoint.** There is no vision/VLM-style endpoint; the API is purely generative and transformational. (Recraft's "extract prompt" and natural-language editing features live in Studio, not the API.)
- **Rate limits.** Per-user: 100 images/minute and 5 requests/second (subject to change).
- **Storage.** Generated images are publicly accessible via signed URLs for ~24 hours; persist locally if longer retention is needed.

---

## 3. Models — Catalog & Selection Guide

All 16 models belong to Recraft's proprietary family. Naming convention: `recraftv{X}` with optional `_vector`, `_pro`, `_utility` suffixes.

| Model ID | Generation | Type | Resolution tier | Notes |
|---|---|---|---|---|
| `recraftv4_1` | V4.1 | raster | 1MP | Default model. Peak artistic/design quality. |
| `recraftv4_1_vector` | V4.1 | vector | — | Clean vectors/icons by default. |
| `recraftv4_1_pro` | V4.1 | raster | 4MP | Print-ready, large-scale. |
| `recraftv4_1_pro_vector` | V4.1 | vector | — | Pro vector quality. |
| `recraftv4_1_utility` | V4.1 | raster | 1MP | General-purpose appeal. |
| `recraftv4_1_utility_vector` | V4.1 | vector | — | Utility vector. |
| `recraftv4_1_utility_pro` | V4.1 | raster | 4MP | Utility Pro raster. |
| `recraftv4_1_utility_pro_vector` | V4.1 | vector | — | Utility Pro vector. |
| `recraftv4` | V4 | raster | 1MP | Ground-up rebuild, design taste. |
| `recraftv4_vector` | V4 | vector | — | Production-grade vector generation. |
| `recraftv4_pro` | V4 | raster | 4MP | Print-ready raster. |
| `recraftv4_pro_vector` | V4 | vector | — | Print-ready vector. |
| `recraftv3` | V3 (Red Panda) | raster | 1MP | Styles + text_layout + artistic_level supported. |
| `recraftv3_vector` | V3 | vector | — | Vector styles + text_layout. |
| `recraftv2` | V2 (20B) | raster | 1MP | Unique styles (Kawaii, 80's, Voxel art, etc.). |
| `recraftv2_vector` | V2 | vector | — | Icon styles + Doodle variants. |

### Selection guide (per docs)

- **V4** — best default for most tasks; highest design taste and prompt accuracy.
- **V4 Pro** — same capabilities, higher resolution for print/large-format.
- **V3** — choose when you need style consistency, precise text positioning (`text_layout`), or `artistic_level` control.
- **V2** — specialized creative effects and exclusive styles (Kawaii, 80's, Voxel art, Psychedelic, Doodle icons).

### Prompt length limits

| Models | Max prompt length |
|---|---|
| V4.1 family (all 8 variants) + V4 family (4 variants) | 10,000 chars |
| V3, V3 Vector, V2, V2 Vector | 1,000 chars |

---

## 4. Styles — Curated & Custom

> **Note:** Styles are **not supported on V4 and V4.1 models** (including Pro, Utility, and Vector variants). The `style` and `style_id` parameters apply only to V2 and V3 models.

- **`style`** — a curated style name string (e.g. `Photorealism`, `Illustration`, `Vector art`, `Hand-drawn`). If a name exists across multiple models, the API defaults to V3 (or V2 if unsupported). Pair explicitly with the `model` parameter to select a version.
- **`style_id`** — a UUID for a custom style. Accepts API-created IDs, web-platform IDs you own, public styles, or styles shared to your account. **`style` and `style_id` are mutually exclusive.**
- If neither is provided, model defaults apply: V3 → `Recraft V3 Raw`; V3 Vector → `Vector art`; V2 → `Photorealism`; V2 Vector → `Vector art`.

### Style categories by model

| Model | Categories |
|---|---|
| **Recraft V3** | Photorealistic (Photorealism, Enterprise, Natural light, Studio photo, HDR, Hard flash, Motion blur, Black & white, Product photo, …); Illustration (Illustration, Hand-drawn, Grain, Pixel art, Clay, Risograph, Pop art, Noir, Street art, …); Emblem (Prestige Emblem, Pop Graphic, Stamp, Punk Graphic, Vintage Emblem) |
| **Recraft V3 Vector** | Vector art, Line art, Linocut, Color blobs, Engraving, Bold stroke, Mosaic, Naivector, Seamless Vector, … |
| **Recraft V2** | Photorealistic (Photorealism, Enterprise, HDR, …); Illustration (Illustration, 3D render, Glow, Watercolor, Kawaii, Psychedelic, 80's, Voxel art, …); Seamless Digital |
| **Recraft V2 Vector** | Vector art, Line art, Linocut, Cartoon, Flat 2.0, Color blobs, Vector Kawaii, Seamless Vector, Engraving; Icon styles: Icon, Outline, Pictogram, Colored outline, Doodle, Colored shape, Gradient outline, Offset doodle, Gradient shape, Broken line, Offset fill |

### Output format vs style

- `Photorealism` and `Illustration` styles → raster output (PNG/WEBP/JPG).
- `Vector art` and `Icon` styles → SVG output (scalable, mathematical paths).

---

## 5. Image Generation (Text → Image)

**`POST /v1/images/generations`** — base endpoint accepting any supported model/style.
**`POST /v1/images/generations/raster`** — rejects vector models/styles; raster only.
**`POST /v1/images/generations/vector`** — rejects raster models/styles; vector only.

### Parameters

| Parameter | Type | Description | Compatibility |
|---|---|---|---|
| `prompt` (required) | string | Text description of desired image(s). | Max length per model (10k V4/V4.1; 1k V2/V3). |
| `n` | int, default 1 | Number of images, 1–6. | All models. |
| `model` | string, default `recraftv4_1` | Model ID (see §3). | Any of the 16 IDs. |
| `style` | string | Curated style name (V2/V3 only). | V2 / V3 styles. |
| `style_id` | UUID | Custom style reference (V2/V3 only). | V2 / V3 styles. |
| `size` | string | `WxH` or `w:h` (e.g. `1820x1024`, `16:9`). Auto-selected if omitted. | Per-model supported values (see §19). |
| `negative_prompt` | string | Undesired elements. | V2 / V3 models. |
| `random_seed` | int | Seed for reproducibility. | All models. |
| `response_format` | string, default `url` | `url` or `b64_json`. | All models. |
| `text_layout` | array of objects | Per-word text placement (see §18). | V3 models only. |
| `controls` | object | Generation tweaks (see §18). | All models; partial for V4. |

### Example (Python, OpenAI-compatible)

```python
from openai import OpenAI

client = OpenAI(base_url='https://external.api.recraft.ai/v1', api_key=RECRAFT_API_TOKEN)

response = client.images.generate(
    prompt='two race cars on a track',
    model='recraftv4_1',
    extra_body={
        'style_id': style_id,
        'controls': {'colors': [{'rgb': [0, 255, 0]}]},
    },
)
print(response.data[0].url)
```

---

## 6. Image-to-Image (Variations)

**`POST /v1/images/imageToImage`** — generate images similar to a source image guided by a prompt, preserving composition/color/identity while altering other aspects. `strength` controls divergence.

Available with **V3 and V4 models** (raster and vector).

### Parameters

| Parameter | Type | Description |
|---|---|---|
| `image` / `image_url` (required) | file / string | Source image (PNG/JPG/WEBP/SVG; <10MB; <16MP; max dim <4096px). Output format follows selected model. |
| `prompt` (required) | string | Description of areas to change. |
| `strength` (required) | float | `[0, 1]`; `0` ≈ near-identical, `1` ≈ minimal similarity. |
| `n` | int, default 1 | 1–6 images. |
| `model` | string, default `recraftv4_1` | V3/V4 family IDs only. |
| `style` / `style_id` | string / UUID | V3/V4 styles. |
| `random_seed` | int | Reproducibility. |
| `response_format` | string, default `url` | `url` or `b64_json`. |
| `negative_prompt` | string | Undesired elements. |
| `text_layout` | array | V3 only. |
| `controls` | object | Generation tweaks. |

---

## 7. Inpainting (Masked Region Regeneration)

**`POST /v1/images/inpaint`** — regenerate masked regions while keeping the rest intact. White mask pixels = inpaint; black = keep. Available with **V3 / V3 Vector only**.

### Parameters

| Parameter | Type | Description |
|---|---|---|
| `image` / `image_url` (required) | file / string | Image to modify (<10MB; <16MP; max dim <4096px; min dim ≥256px). |
| `mask` / `mask_url` (required) | file / string | Grayscale mask, same dimensions as image, pure black/white pixels. |
| `prompt` (required) | string | Description of areas to change. |
| `n` | int, default 1 | 1–6. |
| `model` | string, default `recraftv3` | `recraftv3` or `recraftv3_vector`. |
| `style` / `style_id` | string / UUID | V3 styles only. |
| `response_format` | string, default `url` | `url` or `b64_json`. |
| `negative_prompt` | string | Undesired elements. |
| `text_layout` | array | V3 only. |
| `controls` | object | Generation tweaks. |

---

## 8. Outpainting (Border Extension)

**`POST /v1/images/outpaint`** — extend an image beyond its original borders. Two mutually exclusive sizing strategies:

1. **Per-side pixel margins** — `expand_left`, `expand_right`, `expand_top`, `expand_bottom` (each `[0, 4096]`). Cannot combine with `size`.
2. **Target size** — `size` (`WxH` or `w:h`); source image is placed inside the resulting canvas. Cannot combine with `expand_*`.

`zoom_out_percentage` (float, `[0, 100)`, default `0`) scales the source down before outpainting; combinable with either option or usable alone. At least one of `size`, `expand_*`, or `zoom_out_percentage` must be specified. Resulting dimensions after `expand_*` must not exceed 4096px on either side. Available with **V3 / V3 Vector only**.

### Parameters

| Parameter | Type | Description |
|---|---|---|
| `image` / `image_url` (required) | file / string | Image to extend. |
| `prompt` (required) | string | Content to generate in the extended area. |
| `expand_left` / `expand_right` / `expand_top` / `expand_bottom` | int | Pixels to add per side, `[0, 4096]`. Mutually exclusive with `size`. |
| `size` | string | Resulting image size (`WxH` / `w:h`). Mutually exclusive with `expand_*`. |
| `zoom_out_percentage` | float, default 0 | `[0, 100)`; higher = more surrounding context. |
| `n`, `model` (default `recraftv3`), `style`, `style_id`, `response_format`, `negative_prompt`, `text_layout`, `controls` | — | Same conventions as inpainting (V3 styles only). |

---

## 9. Background Operations — Replace & Generate

### Replace background

**`POST /v1/images/replaceBackground`** — detect the background and modify it per the prompt (no mask required; detection is automatic). V3 / V3 Vector only.

| Parameter | Type | Description |
|---|---|---|
| `image` / `image_url` (required) | file / string | Image to modify. |
| `prompt` (required) | string | Description of new background. |
| `n`, `model` (default `recraftv3`), `style`, `style_id`, `response_format`, `negative_prompt`, `text_layout`, `controls` | — | V3 styles only. |

### Generate background

**`POST /v1/images/generateBackground`** — generate a fresh background around a subject using a **mask** that specifies the regions to fill (white = fill). V3 / V3 Vector only.

| Parameter | Type | Description |
|---|---|---|
| `image` / `image_url` (required) | file / string | Image to modify. |
| `mask` / `mask_url` (required) | file / string | Grayscale mask, same dimensions, pure black/white. |
| `prompt` (required) | string | Description of background to generate. |
| `n`, `model` (default `recraftv3`), `style`, `style_id`, `response_format`, `negative_prompt`, `text_layout`, `controls` | — | V3 styles only. |

> **Difference:** Replace background auto-detects the subject; generate background requires an explicit mask.

---

## 10. Vectorization (Raster → SVG)

**`POST /v1/images/vectorize`** — convert a raster image (PNG/JPG/WEBP) into a scalable SVG. No model parameter; deterministic conversion.

| Parameter | Type | Description |
|---|---|---|
| `file` / `image_url` (required) | file / string | Raster input (<10MB; <16MP; max dim <4096px; min dim ≥256px). |
| `response_format` | string, default `url` | `url` or `b64_json`. |

Returns `response.image.url`.

---

## 11. Background Removal

**`POST /v1/images/removeBackground`** — produce a transparent-background cutout. Accepts raster (PNG/JPG/WEBP) or SVG. SVG input → SVG output (vector); raster input → raster output. SVGs under 256px are scaled up before processing; oversized/unsafe SVGs are rejected.

| Parameter | Type | Description |
|---|---|---|
| `file` / `image_url` (required) | file / string | PNG/JPG/WEBP/SVG (<10MB; <16MP; max dim <4096px; min dim ≥256px). |
| `response_format` | string, default `url` | `url` or `b64_json`. |

---

## 12. Upscaling — Crisp & Creative

### Crisp upscale

**`POST /v1/images/crispUpscale`** — increase resolution making the image sharper/cleaner **without altering content**. Min dimension ≥32px. Cheap ($0.004).

| Parameter | Type | Description |
|---|---|---|
| `file` / `image_url` (required) | file / string | PNG/JPG/WEBP. |
| `response_format` | string, default `url` | `url` or `b64_json`. |

### Creative upscale

**`POST /v1/images/creativeUpscale`** — boost resolution while **regenerating finer details and faces**. Min dimension ≥256px. Expensive ($0.25) — similar to a regeneration.

| Parameter | Type | Description |
|---|---|---|
| `file` / `image_url` (required) | file / string | PNG/JPG/WEBP. |
| `response_format` | string, default `url` | `url` or `b64_json`. |

> **Difference:** Crisp preserves content (interpolation-based sharpening); creative re-synthesizes detail (diffusion-based).

---

## 13. Erase Region

**`POST /v1/images/eraseRegion`** — erase the white-pixel regions of a mask from an image (inpainting without a prompt; content-aware fill). SVG input → SVG output; raster → raster.

| Parameter | Type | Description |
|---|---|---|
| `image` / `image_url` (required) | file / string | Image to modify. |
| `mask` / `mask_url` (required) | file / string | Grayscale mask, same dimensions, pure black/white. White = erase. |
| `response_format` | string, default `url` | `url` or `b64_json`. |

Cheapest operation ($0.002/request).

---

## 14. Remix / Variate Image

**`POST /v1/images/variateImage`** — generate new versions of an image keeping the overall concept intact, with slight differences in character appearance, object position, or scene composition. **No prompt** — uses the visual content of the source image. Useful for reformatting to different aspect ratios.

| Parameter | Type | Description |
|---|---|---|
| `image` / `image_url` (required) | file / string | PNG/JPG/WEBP/SVG (<10MB; <16MP; max dim <4096px; min dim ≥256px). |
| `size` (required) | string | `WxH` or `w:h`; see supported sizes in §19. |
| `n` | int, default 1 | 1–6 variations. |
| `model` | string | Output format (raster/vector) determined by selected model. |
| `random_seed` | int | Reproducibility. |
| `response_format` | string, default `url` | `url` or `b64_json`. |

---

## 15. Explore & Explore Similar

### Explore

**`POST /v1/images/explore`** — generate a grid of diverse images from a single prompt, designed for visual exploration/discovery. Supports `prompt_to_image` controls.

| Parameter | Type | Description |
|---|---|---|
| `prompt` (required) | string | Description of desired image(s). |
| `model` | string, default `recraftv4_1` | V4/V4.1 family only (`recraftv4_1`, `recraftv4_1_vector`, `recraftv4_1_pro`, `recraftv4_1_pro_vector`, `recraftv4`, `recraftv4_vector`, `recraftv4_pro`, `recraftv4_pro_vector`). |
| `style` / `style_id` | string / UUID | Styles (note: not supported on V4/V4.1 — effectively limited). |
| `size` | string, default `1:1` | `WxH` or `w:h`. |
| `response_format` | string, default `url` | `url` or `b64_json`. |
| `controls` | object | Supports `prompt_to_image` controls. |

### Explore similar

**`POST /v1/images/explore/similar`** — generate images visually similar to a source image that was **previously created via the Explore endpoint**. The `source_image_id` is the ID of that explore result.

| Parameter | Type | Description |
|---|---|---|
| `source_image_id` (required) | UUID | ID of the source Explore image. |
| `similarity` (required) | int | `1` (a little similar) → `5` (extremely similar). |
| `response_format` | string, default `url` | `url` or `b64_json`. |

---

## 16. Enhance Prompt

**`POST /v1/prompts/enhance`** — rewrite a short prompt into a richer, more detailed description with visual context, style cues, and composition details. Use the enhanced prompt in any generation endpoint for more vivid/accurate results.

| Parameter | Type | Description |
|---|---|---|
| `prompt` (required) | string | Original prompt; non-empty, ≤2000 chars. |

Returns `response.enhanced_prompt`. Cost: $0.01/request.

---

## 17. Style Creation

**`POST /v1/styles`** — upload up to 5 reference images (or mix existing styles) to create a reusable custom style. Returns a `style_id` for use in any generation endpoint's `style_id` parameter.

### Request body

| Parameter | Type | Description |
|---|---|---|
| `style` (required) | string | Base style: `any`, `realistic_image`, `digital_illustration`, `vector_illustration`, `icon`, `logo_raster`. |
| `files` | files | (multipart) PNG/JPG/WEBP references, max 5 images, total ≤5MB, each <10MB. Required unless `source_styles` is provided. |
| `image_urls` | array of strings | (JSON) URLs/data URLs counterpart of `files`. |
| `model` | string | Transform model (e.g. `recraftv3`, `recraftv3_vector`). |
| `image_weights` | array of numbers | Per-image weights; count must match uploaded images. |
| `source_styles` | array of UUIDs | Existing style IDs to mix in; usable with or instead of uploaded images. |
| `source_style_weights` | array of numbers | Per-source-style weights; count must match `source_styles`. |
| `prompt` | string | Optional text prompt associated with the style. |
| `mix_policy` | string | `PaletteMatch` or `MaxWeight`. |

### Output

```json
{
    "id": "229b2a75-05e4-4580-85f9-b47ee521a00d",
    "style": "digital_illustration",
    "creation_time": "2024-01-01T00:00:00Z",
    "is_private": true,
    "credits": 40
}
```

API-created custom styles are compatible with **V3 and V3 Vector only**. Web-created styles are bound to the model specified at creation. Cost: $0.04/request.

---

## 18. Controls & Text Layout (Auxiliary Parameters)

### Controls

An optional object passed as `controls` to tweak generation. Pass via `extra_body` when using the OpenAI Python library.

| Parameter | Type | Description | Compatibility |
|---|---|---|---|
| `colors` | array of color definitions | Preferable colors palette. | All models. |
| `background_color` | color definition | Desired background color. | All models. |
| `artistic_level` | int, `[0..5]` | Artistic tone: simple (static, straight at camera) → dynamic/eccentric. | V3 models only. |
| `no_text` | bool | Do not embed text layouts. | V3 models only. |

**Color definition:** `{rgb: [r, g, b]}` (3 ints 0–255), plus optional `weight` (0.0–1.0; sum of weights across `colors` must not exceed 1.0).

### Text Layout

An array of `{text, bbox}` objects placing individual words. **V3 and V3 Vector only.**

| Field | Description |
|---|---|
| `text` (required) | A single word using only supported characters (alphanumeric, punctuation, plus a few Latin/Cyrillic/Greek glyphs). |
| `bbox` (required) | A 4-point polygon (not necessarily a rectangle). Each point is `[x, y]` relative to layout dimensions: (0,0) top-left, (1,1) bottom-right. Coordinates can exceed [0,1] (cropped). Must have exactly 4 points. |

Supported characters include ASCII printable set, `Ø Đ Ħ Ł Ŋ Ŧ`, some Greek capitals, some Cyrillic capitals, `ß ẞ`. Any other character triggers validation errors.

```python
response = client.images.generate(
    prompt="cute red panda with a sign",
    style="Illustration",
    extra_body={
        "text_layout": [
            {"text": "Recraft", "bbox": [[0.3,0.45],[0.6,0.45],[0.6,0.55],[0.3,0.55]]},
            {"text": "AI",     "bbox": [[0.62,0.45],[0.70,0.45],[0.70,0.55],[0.62,0.55]]},
        ]
    },
)
```

---

## 19. Sizes, Limits & Pricing

### Supported image sizes

Sizes can be specified as explicit dimensions (`WxH`, e.g. `1820x1024`) or aspect ratio (`w:h`, e.g. `16:9`). If omitted on generation, size is auto-selected based on the prompt.

| Models | Aspects supported | Notable sizes |
|---|---|---|
| **V4.1 / V4.1 Utility / V4** (raster 1MP) | 1:1, 2:1, 1:2, 3:2, 2:3, 4:3, 3:4, 5:4, 4:5, 6:10, 14:10, 10:14, 16:9, 9:16 | `1024x1024`, `1344x768`, `768x1344` |
| **V4.1 Pro / V4.1 Utility Pro / V4 Pro** (raster 4MP) | same 14 aspects | `2048x2048`, `2688x1536`, `1536x2688` |
| **V2 / V3** (raster 1MP) | same 14 aspects | `1024x1024`, `1820x1024`, `1024x1820` |
| **All Vector models** (V2/V3/V4/V4.1 + Pro/Utility variants) | same 14 aspects (aspect only; dimensions are vector) | `1:1` … `9:16` |

### Image input limits (shared across endpoints)

- Formats: PNG, JPG, WEBP, or SVG (some endpoints exclude SVG; see endpoint notes).
- Size: < 10 MB per file.
- Resolution: ≤ 16 MP.
- Max dimension: ≤ 4096 px.
- Min dimension: ≥ 256 px (crisp upscale: ≥ 32 px; removeBackground scales up sub-256px SVGs).

### Storage & rate limits (policies, subject to change)

- Generated images stored ~24 hours; public via cryptographically signed URLs.
- Rate limits: 100 images/minute, 5 requests/second (per user).

### API pricing (1,000 API units = $1.00)

| Service | Cost (USD) | Cost (units) | Billing |
|---|---|---|---|
| Raster generation — V4.1 Pro / V4.1 Utility Pro | $0.21 | 210 | per image |
| Raster generation — V4.1 / V4.1 Utility | $0.035 | 35 | per image |
| Raster generation — V4 Pro | $0.25 | 250 | per image |
| Raster generation — V4 | $0.04 | 40 | per image |
| Raster generation — V3 | $0.04 | 40 | per image |
| Raster generation — V2 | $0.022 | 22 | per image |
| Vector generation — V4.1 Pro Vector / V4.1 Utility Pro Vector / V4 Pro Vector | $0.30 | 300 | per image |
| Vector generation — V4.1 Vector / V4.1 Utility Vector / V4 Vector | $0.08 | 80 | per image |
| Vector generation — V3 Vector | $0.08 | 80 | per image |
| Vector generation — V2 Vector | $0.044 | 44 | per image |
| Raster image-to-image — V3 | $0.04 | 40 | per image |
| Vector image-to-image — V3 Vector | $0.08 | 80 | per image |
| Raster inpainting — V3 | $0.04 | 40 | per image |
| Vector inpainting — V3 Vector | $0.08 | 80 | per image |
| Raster outpainting — V3 | $0.04 | 40 | per image |
| Vector outpainting — V3 Vector | $0.08 | 80 | per image |
| Replace raster background — V3 | $0.04 | 40 | per image |
| Replace vector background — V3 Vector | $0.08 | 80 | per image |
| Generate raster background — V3 | $0.04 | 40 | per image |
| Generate vector background — V3 Vector | $0.08 | 80 | per image |
| Style creation | $0.04 | 40 | per request |
| Image vectorization | $0.01 | 10 | per request |
| Background removal | $0.01 | 10 | per request |
| Crisp upscale | $0.004 | 4 | per request |
| Creative upscale | $0.25 | 250 | per request |
| Erase region | $0.002 | 2 | per request |
| Variate image (Remix) | $0.04 | 40 | per request |
| Prompt enhancement | $0.01 | 10 | per request |

API unit packages are prepaid, non-cancellable, non-refundable, and do not expire. Check `/v1/users/me` for remaining balance.

---

## 20. Video Generation — Studio-Only Note

Recraft's marketing site lists numerous video models (Google Veo 2/3/3.1, OpenAI Sora 2, Kling 2.x/3, PixVerse, Seedance, Wan, Hailuo, Luma Ray 2, Grok Imagine Video). However:

- **Video generation has no public API endpoint.** The documented Recraft API (endpoints, swagger, examples, MCP tools) contains **only image** capabilities. There is no `/v1/videos*` surface.
- **Video generation is available in Recraft Studio only** (Desktop and Android; iOS rolling out), on Pro and Team plans. It supports text-to-video and image-to-video (first frame / last frame for some models).
- **External models (including video models) are not available via the Recraft API.** The API supports only Recraft's proprietary image model family (V2, V3, V4, V4.1). All third-party image and video models listed on the site (GPT Image, Flux, Nano Banana, Imagen, Veo, Sora, Kling, etc.) are Studio-only.

> For programmatic video generation, Recraft is currently not a viable API. Consider OpenAI Sora 2, Google Veo, or the individual provider APIs for video generation via API.

---

## 21. MCP Tools Reference

Recraft also exposes its image capabilities via MCP (Model Context Protocol) servers. The **Local MCP server** bills API units (same prices as the API); the **Remote MCP server** bills subscription credits (same prices as the web platform).

| Tool | Description | Parameters |
|---|---|---|
| `generate_image` | Raster/vector images from a prompt | `prompt`, `style`, `size`, `model`, `number of images` |
| `create_style` | Create a style from a list of images | `list of images`, `basic style` |
| `vectorize_image` | Vectorize a raster image | `image` |
| `image_to_image` | Raster/vector images from an image + prompt | `image`, `prompt`, `similarity strength`, `style`, `size`, `model`, `number of images` |
| `remove_background` | Remove the background of an image | `image` |
| `replace_background` | Generate a new background from a prompt | `image`, `prompt for background`, `style`, `size`, `model`, `number of images` |
| `crisp_upscale` | Upscale without altering content | `image` |
| `creative_upscale` | Upscale while regenerating details | `image` |
| `get_user` | Retrieve user info and balance | — |

MCP model/style compatibility mirrors the API: styles apply only to V2/V3; vector models produce SVG; raster models produce PNG/WEBP/JPG; `custom_style_id` is bound to the model/base style used at creation.

---

## 22. Capability Summary & Cross-Reference

| Capability | API Endpoint | Models | Input | Key params |
|---|---|---|---|---|
| Text → image (raster/vector) | `POST /v1/images/generations[/raster\|/vector]` | All 16 | prompt | `model`, `style`/`style_id`, `size`, `n`, `negative_prompt`, `random_seed`, `text_layout` (V3), `controls` |
| Image + text → variation | `POST /v1/images/imageToImage` | V3 / V4 family | image + prompt + strength | `strength` `[0,1]`, `model`, `style`/`style_id` |
| Masked region regeneration | `POST /v1/images/inpaint` | V3 / V3 Vector | image + mask + prompt | `mask`/`mask_url`, `model`, `style` |
| Border extension | `POST /v1/images/outpaint` | V3 / V3 Vector | image + prompt | `expand_*` or `size` + optional `zoom_out_percentage` |
| Replace background (auto-detect) | `POST /v1/images/replaceBackground` | V3 / V3 Vector | image + prompt | `prompt`, `model`, `style` |
| Generate background (masked) | `POST /v1/images/generateBackground` | V3 / V3 Vector | image + mask + prompt | `mask`/`mask_url`, `prompt` |
| Raster → SVG | `POST /v1/images/vectorize` | n/a | image | `response_format` |
| Background removal | `POST /v1/images/removeBackground` | n/a | image (raster or SVG) | `response_format` |
| Upscale (preserve content) | `POST /v1/images/crispUpscale` | n/a | raster image | `response_format` |
| Upscale (regenerate detail) | `POST /v1/images/creativeUpscale` | n/a | raster image | `response_format` |
| Erase region (content-aware) | `POST /v1/images/eraseRegion` | n/a | image + mask | `mask`/`mask_url` |
| Remix / variate (no prompt) | `POST /v1/images/variateImage` | model-determined | image + size | `size` (required), `n`, `random_seed` |
| Explore (grid of variations) | `POST /v1/images/explore` | V4 / V4.1 family | prompt | `model`, `style`/`style_id`, `size`, `controls` |
| Explore similar (by image_id) | `POST /v1/images/explore/similar` | V4 / V4.1 family | source_image_id + similarity | `source_image_id`, `similarity` (1–5) |
| Prompt enhancement | `POST /v1/prompts/enhance` | n/a | prompt | `prompt` (≤2000 chars) |
| Custom style creation | `POST /v1/styles` | V3 / V3 Vector (API) | images or source_styles | `style`, `files`/`image_urls`, `model`, `*_weights`, `mix_policy` |
| User info / balance | `GET /v1/users/me` | n/a | — | — |

### Notes for integrators

- **No vision/understanding API.** Recraft API is generative + transformational only; there is no image-to-text or image analysis endpoint.
- **No video API.** Video generation is Studio-only; do not rely on Recraft API for programmatic video.
- **External (third-party) models are Studio-only.** Only Recraft V2/V3/V4/V4.1 image models are API-accessible.
- **Style support is V2/V3-only.** V4/V4.1 models ignore `style`/`style_id`; rely on prompts and `controls` instead.
- **Synchronous only.** No job polling, no webhooks, no batch endpoint in the documented surface. For high volume, respect the 100 images/minute and 5 req/second rate limits.
- **OpenAI Python library compatibility** covers `client.images.generate(...)`; all other endpoints use `client.post(path=..., cast_to=object, ...)` with `extra_body` for non-standard params.
- **Persist outputs within ~24h.** Signed URLs expire; download or use `b64_json` for long-term storage.