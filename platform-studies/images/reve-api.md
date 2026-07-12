# Reve API Analysis — Image Generation, Editing, Remixing & Layout-Aware Composition

> **Base URL:** `https://api.reve.com` (`/v1/image/*` and `/v2/image/*`)
> **Docs:** `https://api.reve.com/console/docs` | **Pricing:** `https://api.reve.com/console/pricing` | **FAQ:** `https://app.reve.com/faq`
> **Auth:** Bearer API key (`Authorization: Bearer $REVE_API_KEY`); tokens look like `papi.xxx`, managed at `https://api.reve.com/console/keys`
> **SDKs:** `reve` (Python, `uv pip install reve`, Python 3.10+); skill file: `https://github.com/reve-ai/reve-sdk/tree/main/skills/reve-image`
> **Description:** Reve is an image-generation platform exposed through a small, synchronous REST API. It offers text-to-image **creation**, instruction-based **editing** of a single uploaded image, and **remixing** of 1–6 reference images with a prompt. A newer **v2 layout-aware** surface splits image work into a structured `Layout` (labelled, bounded `Region`s) so you can plan composition, edit the plan with typed `LayoutCommand`s, render the plan to pixels, and reverse-engineer a layout from an existing image. All endpoints take plain text instructions and return predictable JSON, making the API particularly easy for AI agents to use. There is **no video generation and no standalone vision/chat** endpoint — image "understanding" is provided only in the narrow form of `image_to_layout` (reverse layout extraction) and the implicit image analysis performed by the edit/remix models.

---

## Table of Contents

1. [Platform Overview & Core Concepts](#1-platform-overview--core-concepts)
2. [API Architecture & Capability Matrix](#2-api-architecture--capability-matrix)
3. [Image Generation — Create](#3-image-generation--create)
4. [Image Editing — Edit](#4-image-editing--edit)
5. [Image Remixing — Remix](#5-image-remixing--remix)
6. [Layout-Aware Composition (v2)](#6-layout-aware-composition-v2)
7. [Image Understanding — `image_to_layout`](#7-image-understanding--image_to_layout)
8. [Postprocessing Pipeline](#8-postprocessing-pipeline)
9. [Effects System](#9-effects-system)
10. [Response Formats, Headers & Credits](#10-response-formats-headers--credits)
11. [Auth, Errors & SDK Conveniences](#11-auth-errors--sdk-conveniences)
12. [Agent Integration](#12-agent-integration)
13. [Capability Summary & Cross-Reference](#13-capability-summary--cross-reference)

---

## 1. Platform Overview & Core Concepts

Reve's API is organized around these core abstractions:

- **Prompt / Edit instruction** — The plain-text driving input. All endpoints accept up to **2560 characters** of natural language, which the model **automatically enhances** (a built-in prompt rewriter; the revised prompt is not returned on the REST surface, unlike OpenAI's `revised_prompt`). For Create it is `prompt`; for Edit it is `edit_instruction`; for Remix it is `prompt` and may embed `<img>N</img>` tags (v1 docs/SDK also use `<ref>N</ref>`) to refer to specific reference images by index.
- **Reference image(s)** — Base64-encoded inline JSON images supplied as input. Edit takes exactly **1** (`reference_image`); Remix takes **1–6** (`reference_images`, a list). A single image may be at most **40 MB** and **33,554,432 pixels** (e.g. 8192×4096), with neither dimension exceeding 8192 px. Across all images in one call: ≤ **50,331,648 pixels** and ≤ **100 MB** after base64 decoding.
- **Model version** — A pinned model identifier string per workflow. `latest` aliases the most recent release; fast variants (`latest-fast`, `*-fast@20251030`) trade quality for ~6× lower cost. Versions are workflow-specific (`reve-create@20250915`, `reve-edit@20250915` / `reve-edit-fast@20251030`, `reve-remix@20250915` / `reve-remix-fast@20251030`).
- **Aspect ratio** — Output shape selector. v1 supports `16:9, 9:16, 3:2, 2:3, 4:3, 3:4, 1:1`. v2 adds wider/taller ratios (`4:1, 3:1, 21:9, 2:1, 17:9, 1:2, 1:3, 1:4`) plus `auto` (model-chosen). Defaults vary: Create `3:2`, Edit = reference image's ratio, Remix = "smartly chosen by the model".
- **Test Time Scaling (TTS)** — A quality/effort knob. Number 1–15 (values >15 clamped to 15, <1 clamped to 1). Costs scale linearly above 1; >5 only "occasionally" improves results. Does **not** increase latency. Default `1`.
- **Postprocessing** — An ordered array of operations applied after generation: `upscale`, `remove_background`, `fit_image`, `effect`. Adds proportional cost (except `fit_image`, which is free).
- **Effect** — A named visual filter preset (e.g. `cmyk_halftone`), either a built-in system preset or a project-saved preset configured in the Reve web app. Listed via `GET /v1/image/effect` and applied as a postprocessing step.
- **Layout (v2)** — A structured composition plan: `regions` (list of `Region`), each with a `label`, `prompt`, `bbox` (normalized `[0,1]`, top-left origin), optional `image_index`, `parent`, and `region_type`. Optional `width`/`height` give the pixel coordinate frame (multiples of 32, with `width*height` between `3072×2560` and `4096×4096`). Layouts decouple "what to draw and where" from rendering.
- **Region** — A labelled, bounded element of a layout. `region_type` is a level-of-detail / special-handling hint: `coarse_detail` (a high-level object like a person or car), `medium_detail` (sub-part whose parent is `coarse_detail`, e.g. an arm), `fine_detail` (whose parent is `medium_detail`, e.g. a ring), `text` (embedded text), `hand`, or `face`.
- **LayoutCommand** — A typed edit operation on a layout: `op` ∈ {`add`, `shift`, `remove`, `place`, `keep`, `change`}; subject named by `label` or `description`; `image_index` selects an input reference image; positions `at`/`to` are a `Bbox(x0,y0,x1,y1)` or `Point(x,y)` in normalized coords.
- **Credits** — The metering unit. 750 credits ≈ $1 (min purchase $10 / 7,500 credits; up to $1,000/day). Every successful response reports `credits_used` and `credits_remaining`.
- **Breadcrumb** — Optional `?breadcrumb=` query param for log/usage tracking; ignored by the API, does not affect processing.

### Capability pipelines

```
Create (text → image)
   prompt ──▶ POST /v1/image/create  ──▶ base64 PNG (JSON) | raw PNG/JPEG/WebP

Edit (image + text → image)
   reference_image + edit_instruction ──▶ POST /v1/image/edit ──▶ image

Remix (images + text → image)
   reference_images[1..6] + prompt (with <img>N</img> tags) ──▶ POST /v1/image/remix ──▶ image

Layout-aware composition (v2)
   text → layout:        create_layout(prompt, refs?)            ──▶ Layout
   layout → layout:      edit_layout(prompt, refs?, commands?)  ──▶ Layout
   layout → image:       render(layout, refs?, postprocessing?)  ──▶ image (+ echoed layout)
   image → layout:       image_to_layout(image)                 ──▶ Layout   (image understanding)
```

### Scope notes

- **No video.** The docs, pricing page, and FAQ mention only still images; the FAQ contains no "video" matches. No async/video endpoints exist.
- **No standalone vision/chat.** There is no equivalent of OpenAI's `input_image`/Chat Completions vision modality. The only image-as-input "understanding" is `image_to_layout` (v2), which extracts a structured layout rather than producing free-form text about an image. The edit and remix models internally analyze input images but do not expose that analysis as text.

---

## 2. API Architecture & Capability Matrix

### Endpoints at a glance

| Surface | Endpoint | Method | Modality | Purpose |
|---|---|---|---|---|
| v1 | `/v1/image/create` | POST | text → image | Generate an image from a prompt |
| v1 | `/v1/image/edit` | POST | image + text → image | Edit a single uploaded image by instruction |
| v1 | `/v1/image/remix` | POST | 1–6 images + text → image | Compose reference images with a prompt |
| v1 | `/v1/image/effect` | GET | — | List effects (built-in presets + project-saved) |
| v2 | `/v2/image/create` | POST | text (+ refs) → image (+ layout) | Layout-aware create; echoes generated layout |
| v2 | `/v2/image/edit` | POST | image (+ refs) + text → image (+ layout) | Layout-aware edit |
| v2 | `/v2/image/render` | POST | layout (+ refs) → image | Render a layout to pixels (layout2image) |
| v2 | `/v2/image/create_layout` | POST | text (+ refs) → layout | Generate a layout from text/refs |
| v2 | `/v2/image/edit_layout` | POST | layout + text + commands → layout | Edit a layout with typed commands |
| v2 | `/v2/image/image_to_layout` | POST | image → layout | Derive a layout from an image (image2layout) |
| (SDK) | `get_balance()` | — | — | Return current credit balance (`budget_id`, `new_balance`) |

> Note: the public docs site documents the **v1** endpoints (`/v1/image/*`) and the effect listing. The **v2 layout-aware** endpoints and the `get_balance` helper are documented in the official Python SDK README and SKILL.md, which target `/v2/image/*`. The `?breadcrumb=` query param and the common headers/credits model are shared across surfaces.

### Model families & versions

| Workflow | Versions (default `latest`) |
|---|---|
| Create | `latest`, `reve-create@20250915` (fast "coming soon" per pricing) |
| Edit | `latest-fast`, `latest`, `reve-edit-fast@20251030`, `reve-edit@20250915` |
| Remix | `latest-fast`, `latest`, `reve-remix-fast@20251030`, `reve-remix@20250915` |

`latest` aliases the current pinned release; the version actually used is returned in every response (`version` field / `X-Reve-Version` header). Fast variants cost **5 credits** vs **18** (create) / **30** (edit, remix) for standard.

### Pricing (per request, standard / fast)

| Capability | Credits | ~USD |
|---|---|---|
| Create | 18 | ~$0.024 |
| Edit | 30 | ~$0.04 |
| Remix | 30 | ~$0.04 |
| Edit Fast / Remix Fast | 5 | ~$0.007 |
| Create Fast | coming soon | — |
| Test Time Scaling | multiples of base cost | ~$0.01 and up |
| Postprocessing: upscale / remove_background | variable (≥2) | ~$0.002/MP |
| Postprocessing: effect | 3 | ~$0.004 |
| Postprocessing: fit_image | 0 (free) | — |

---

## 3. Image Generation — Create

### Main concepts

Create is the simplest flow: a text prompt alone produces a new image. The model automatically enhances the prompt (no `revised_prompt` is surfaced). Output dimensions are governed by `aspect_ratio` (default `3:2`).

### Endpoint — `POST /v1/image/create`

**Headers**

| Header | Type | Req | Description |
|---|---|---|---|
| `Authorization` | string | yes | Bearer API key |
| `Accept` | string | no | `application/json` (default) \| `image/png` \| `image/jpeg` \| `image/webp` |

**Request body**

| Field | Type | Req | Default | Description |
|---|---|---|---|---|
| `prompt` | string | yes | — | Text description, max 2560 chars; auto-enhanced |
| `aspect_ratio` | string | no | `3:2` | `16:9, 9:16, 3:2, 2:3, 4:3, 3:4, 1:1` |
| `version` | string | no | `latest` | `latest` or `reve-create@20250915` |
| `postprocessing` | array | no | none | Ordered postprocessing ops (see §8) |
| `test_time_scaling` | number | no | `1` | 1–15; >5 rarely helps; cost scales linearly |

**Example**

```bash
curl -X POST https://api.reve.com/v1/image/create \
  -H "Authorization: Bearer $REVE_API_KEY" \
  -H "Accept: image/webp" \
  -H "Content-Type: application/json" \
  -d '{ "prompt": "A beautiful sunset over mountains", "aspect_ratio": "16:9" }' \
  -o sunset.webp
```

```python
from reve.v1.image import create
from reve.v1.postprocessing import upscale, remove_background

img = create(
    prompt="A red dragon flying over mountains",
    aspect_ratio="16:9",
    version="latest",
    test_time_scaling=3,
    postprocessing=[upscale(factor=2), remove_background()],
)
img.save("dragon.png")
print(img.credits_remaining)
```

**v2 layout-aware create** (`reve.v2.image.create`) adds optional `references` (list of `ImageInput`) and returns a `V2ImageResponse` that also exposes the `layout` the model generated.

---

## 4. Image Editing — Edit

### Main concepts

Edit takes exactly **one** uploaded image plus a natural-language `edit_instruction` describing the desired change. The instruction is auto-enhanced. If `aspect_ratio` is omitted, the reference image's ratio is used. Fast (`latest-fast` / `reve-edit-fast@20251030`) and standard (`latest` / `reve-edit@20250915`) variants are offered.

### Endpoint — `POST /v1/image/edit`

**Request body**

| Field | Type | Req | Default | Description |
|---|---|---|---|---|
| `edit_instruction` | string | yes | — | How to edit; max 2560 chars; auto-enhanced |
| `reference_image` | image (base64) | yes | — | The image to edit |
| `aspect_ratio` | string | no | reference image's ratio | `16:9, 9:16, 3:2, 2:3, 4:3, 3:4, 1:1` |
| `version` | string | no | `latest` | `latest-fast`, `latest`, `reve-edit-fast@20251030`, `reve-edit@20250915` |
| `postprocessing` | array | no | none | Postprocessing ops (see §8) |
| `test_time_scaling` | number | no | `1` | 1–15 |

**Example**

```bash
curl -X POST https://api.reve.com/v1/image/edit \
  -H "Authorization: Bearer $REVE_API_KEY" \
  -H "Accept: application/json" \
  -H "Content-Type: application/json" \
  -d '{
    "edit_instruction": "Make this into an impasto painting.",
    "reference_image": "'"$(base64 < reference.jpg | tr -d '\n')"'",
    "aspect_ratio": "16:9",
    "version": "latest"
  }'
```

```python
from reve.v1.image import edit

img = edit(
    edit_instruction="Make the sky more dramatic with storm clouds",
    reference_image="original.jpg",
)
img.save("edited.jpg")
```

**v2 layout-aware edit** (`reve.v2.image.edit`) takes `prompt` (not `edit_instruction`), `image` (an `ImageInput`, path, bytes, or PIL), optional `references`, and returns a `V2ImageResponse` with the generated `layout`.

---

## 5. Image Remixing — Remix

### Main concepts

Remix blends **1–6 reference images** with a text prompt to create a new variation — combining styles, concepts, and visual elements. The prompt may embed `<img>N</img>` tags (v1 docs/SDK also use `<ref>N</ref>`) to reference a specific image by its index in the list, enabling precise cross-image composition ("Style the subject in `<img>0</img>` on the table in `<img>1</img>`"). If `aspect_ratio` is omitted, the model "smartly chooses" it.

### Endpoint — `POST /v1/image/remix`

**Request body**

| Field | Type | Req | Default | Description |
|---|---|---|---|---|
| `prompt` | string | yes | — | Description with optional `<img>N</img>` tags; max 2560 chars; auto-enhanced |
| `reference_images` | list of images | yes | — | 1–6 base64 images; per-image ≤40 MB / ≤33,554,432 px; total ≤50,331,648 px / 100 MB |
| `aspect_ratio` | string | no | model-chosen | `16:9, 9:16, 3:2, 2:3, 4:3, 3:4, 1:1` |
| `version` | string | no | `latest` | `latest-fast`, `latest`, `reve-remix-fast@20251030`, `reve-remix@20250915` |
| `postprocessing` | array | no | none | Postprocessing ops (see §8) |
| `test_time_scaling` | number | no | `1` | 1–15 |

**Example**

```bash
curl -X POST https://api.reve.com/v1/image/remix \
  -H "Authorization: Bearer $REVE_API_KEY" \
  -H "Accept: application/json" \
  -H "Content-Type: application/json" \
  -d '{
    "prompt": "Style the pancakes in <img>0</img> on the cozy diner table in <img>1</img>.",
    "reference_images": [
      "'"$(base64 < reference-0.jpg | tr -d '\n')"'",
      "'"$(base64 < reference-1.jpg | tr -d '\n')"'"
    ],
    "aspect_ratio": "1:1",
    "version": "latest"
  }'
```

```python
from reve.v1.image import remix

img = remix(
    prompt="The subject from <ref>0</ref> standing in a magical forest",
    reference_images=["photo.jpg"],
    aspect_ratio="1:1",
)
```

**Error note:** Remix-specific error codes include `INDEX_OUT_OF_BOUNDS` (an `<img>N</img>` tag refers to a non-existent index).

---

## 6. Layout-Aware Composition (v2)

### Main concepts

The v2 surface introduces a structured **Layout** — a list of labelled, bounded **Region**s — that separates composition planning from rendering. This enables precise spatial control, programmatic edits via typed **LayoutCommand**s, and round-tripping between images and layouts. Layout coordinates are normalized to `[0, 1]` with top-left origin; optional `width`/`height` (multiples of 32, `width*height` between `3072×2560` and `4096×4096`) define the pixel coordinate frame.

### Functions (`reve.v2.image`)

| Function | Kind | Description |
|---|---|---|
| `create(prompt, *, references?, aspect_ratio?, postprocessing?, version?)` | image | Generate an image (text + optional refs); echoes generated `layout` |
| `edit(prompt, image, *, references?, aspect_ratio?, postprocessing?, version?)` | image | Edit an image with a text prompt; echoes `layout` |
| `render(layout, *, references?, postprocessing?, version?)` | image | Render a layout to an image (layout2image) |
| `create_layout(prompt, *, references?, aspect_ratio?, version?)` | layout | Generate a layout from text and/or references |
| `edit_layout(prompt, *, references?, commands?, aspect_ratio?, version?)` | layout | Edit a layout with text, references, and commands |
| `image_to_layout(image, *, version?)` | layout | Derive a layout from an image (image2layout) |

**Aspect ratios (v2):** `4:1, 3:1, 21:9, 2:1, 17:9, 16:9, 3:2, 4:3, 1:1, 3:4, 2:3, 9:16, 1:2, 1:3, 1:4`, or `auto` (default).

### Input types (`reve.v2.types`)

| Type | Fields |
|---|---|
| `ImageInput` | `data` (path/bytes/PIL, base-64 in JSON) **or** `ref` (`id:<uuid>` / `reference:@<name>`) |
| `Bbox` | `x0`, `y0`, `x1`, `y1` — normalized `[0,1]`, top-left origin |
| `Point` | `x`, `y` — normalized `[0,1]` |
| `Region` | `label`, `prompt`, `bbox`, `image_index?`, `image_region_index?`, `parent?`, `region_type?` |
| `Layout` | `regions: list[Region]`, `prompt?`, `normalized_edit_instruction?`, `width?`, `height?` |
| `Reference` | `image: ImageInput?`, `prompt?`, `layout?` — an image and/or a layout |
| `LayoutCommand` | `op`, `label?`, `description?`, `image_index?`, `at?`, `to?`, `new_description?` |

**`region_type` values:** `coarse_detail` (high-level object: a person, car), `medium_detail` (sub-part whose parent is `coarse_detail`: an arm, belt), `fine_detail` (whose parent is `medium_detail`: a ring, buckle), `text` (embedded text), `hand`, `face`.

**`LayoutCommand.op` values:** `add`, `shift`, `remove`, `place`, `keep`, `change`. Subject is named by `label` or `description`; `image_index` selects an input reference image; `at`/`to` accept a `Bbox` or `Point` in normalized coords; `change` uses `new_description`.

**Project references:** `ImageInput.ref` points at images already in the project bound to the API key:
- `id:<uuid>` — ID of an image or generation in the project (a generation ID resolves to its output image).
- `reference:@<name>` — a named reference entity defined in the Reve web app.

### End-to-end layout pipeline

```python
from reve.v2.image import create_layout, edit_layout, render, image_to_layout
from reve.v2.types import Bbox, LayoutCommand, Point, Reference

# 1. text -> layout
created = create_layout(
    prompt="A cozy reading nook: an armchair beside a tall bookshelf, a floor lamp behind the chair.",
    aspect_ratio="3:2",
)

# 2. layout -> layout (edit with typed commands)
edited = edit_layout(
    prompt="Add a small side table next to the armchair.",
    references=[Reference(layout=created.layout)],
    commands=[
        LayoutCommand(op="add", description="a small round side table"),
        LayoutCommand(op="shift", label="mug", at=Point(0.5, 0.5), to=Point(0.7, 0.6)),
        LayoutCommand(op="change", label="book", new_description="an open notebook"),
    ],
    aspect_ratio="3:2",
)

# 3. layout -> image
rendered = render(layout=edited.layout)
rendered.save("nook.jpg")

# 4. image -> layout (reverse)
analyzed = image_to_layout(image="nook.jpg")
```

`create`/`edit`/`render` return `V2ImageResponse` (image + optional `layout` + credits/version metadata); `create_layout`/`edit_layout`/`image_to_layout` return `V2LayoutResponse` (same fields minus `image`/`image_bytes`, no `save`).

---

## 7. Image Understanding — `image_to_layout`

### Main concepts

Reve does **not** expose a general vision/chat endpoint. The only explicit image-as-input "understanding" capability is `image_to_layout` (v2), which **reverse-engineers a structured `Layout`** from a raster image: it detects the constituent objects, labels them, assigns each a normalized `bbox` and a `region_type` (coarse/medium/fine detail, text, hand, face), and infers parent/child relationships. This produces a machine-readable scene structure rather than free-form text.

### Endpoint — `POST /v2/image/image_to_layout` (SDK: `image_to_layout`)

| Field | Type | Req | Default | Description |
|---|---|---|---|---|
| `image` | `ImageInput` / path / bytes / PIL | yes | — | The image to analyze |
| `version` | string | no | `latest` | Model version; actual used returned in response |

**Returns:** `V2LayoutResponse` whose `layout.regions` is a list of `Region`s (`label`, `prompt`, `bbox`, optional `image_index`, `parent`, `region_type`).

```python
from reve.v2.image import image_to_layout

analyzed = image_to_layout(image="nook.jpg")
for region in analyzed.layout.regions:
    b = region.bbox
    kind = f" [{region.region_type}]" if region.region_type else ""
    print(f"{region.label}{kind}: ({b.x0:.2f},{b.y0:.2f})-({b.x1:.2f},{b.y1:.2f})")
```

**Implicit understanding:** the Edit and Remix models also internally analyze input images to follow instructions, but that analysis is not surfaced as text.

---

## 8. Postprocessing Pipeline

### Main concepts

`postprocessing` is an ordered array of operations applied **after** generation, on all four image-producing endpoints (Create, Edit, Remix, and v2 `render`). Each entry is a small JSON object selecting a `process` and its parameters. Cost scales with image megapixels (except `fit_image`, which is free).

### Operations

| Process | Parameters | Cost | Notes |
|---|---|---|---|
| `upscale` | `upscale_factor` ∈ {2, 3, 4} | variable (≥2 credits), ~$0.002/MP | 4× output is very large |
| `remove_background` | — | variable (≥2 credits), ~$0.002/MP | Keeps central subject; transparent output; poor on images without a clear subject |
| `fit_image` | `max_dim` and/or `max_width` and/or `max_height` (px, 1–4096) | **free** | Scale-down preserving aspect ratio; smaller images are not enlarged |
| `effect` | `effect_name` (+ optional `effect_parameters`) | 3 credits, ~$0.004 | Apply a named preset (see §9) |

**JSON form**

```json
"postprocessing": [
  { "process": "upscale", "upscale_factor": 2 },
  { "process": "remove_background" },
  { "process": "fit_image", "max_dim": 512 },
  { "process": "effect", "effect_name": "cmyk_halftone" }
]
```

**SDK helpers** (`reve.v1.postprocessing`): `upscale(factor=2)`, `remove_background()`, `fit_image(max_width=, max_height=, max_dim=)`, `effect(name, parameters=None)`.

---

## 9. Effects System

### Main concepts

Effects are named visual filter presets. Each project's API token has access to a mix of **built-in system presets** and **project-saved presets** (configured in the Reve web app and linked to the project). The list endpoint returns **names only**, not parameter definitions — you configure presets in the app, save them with a name, and reference that name from the API.

### Endpoint — `GET /v1/image/effect`

**Query parameters**

| Param | Values | Default | Description |
|---|---|---|---|
| `source` | `all`, `project`, `preset` | `all` | Filter effects by source |

**Response**

| Field | Type | Description |
|---|---|---|
| `effects` | array | All available effect presets for the token's project |
| `effects[].name` | string | Effect name to pass as `postprocessing.effect_name` |
| `effects[].source` | string | `saved` (project) or `builtin` (system preset) |
| `effects[].description` | string, optional | Human-readable description |
| `effects[].category` | string, optional | Preset category (e.g. `color`, `texture`) |

**Example response**

```json
{
  "effects": [
    { "name": "cmyk_halftone", "description": "CMYK halftone print effect", "source": "builtin", "category": "textures" },
    { "name": "my-saved-effect", "source": "saved" }
  ]
}
```

**Applying with overrides:** in a postprocessing `effect` step you may include `effect_parameters` using the format `{ filterId: { uniformId: value } }`; missing parameters use the effect's saved defaults.

```python
from reve.v1.image import list_effects, create
from reve.v1.postprocessing import effect

effects = list_effects(source="preset")
img = create(prompt="...", postprocessing=[effect(name="cmyk_halftone")])
```

---

## 10. Response Formats, Headers & Credits

### Response formats (selected by `Accept`)

| `Accept` | Response body | Metadata |
|---|---|---|
| `application/json` (default) | JSON: base64 image (PNG) + fields | Inline in JSON (`version`, `content_violation`, `request_id`, `credits_used`, `credits_remaining`) |
| `image/png` | Raw PNG bytes | Via custom headers |
| `image/jpeg` | Raw JPEG bytes | Via custom headers |
| `image/webp` | Raw WebP bytes | Via custom headers |

On any error, image-format responses return a small grey image in the requested format plus an `X-Reve-Error-Code` header.

### Response headers

| Header | Description | When |
|---|---|---|
| `X-Reve-Content-Violation` | `true`/`false` — whether the image violates content policy | Any 200 |
| `X-Reve-Request-Id` | Unique request id (for support) | Every request |
| `X-Reve-Version` | Model version used (e.g. `reve-create@20250915`) | 200 when `Accept` is an image type |
| `X-Reve-Credits-Used` | Credits consumed by this request | 200 when `Accept` is an image type |
| `X-Reve-Credits-Remaining` | Credits remaining in budget | 200 when `Accept` is an image type |
| `X-Reve-Error-Code` | Error type (e.g. `PROMPT_TOO_LONG`, `CONTENT_POLICY_VIOLATION`, `INDEX_OUT_OF_BOUNDS`) | All non-200; also 200 with content violation |

### JSON success body (200)

| Field | Type | Description |
|---|---|---|
| `image` | string | Base64-encoded image data (empty on failure) |
| `version` | string | Model version used |
| `content_violation` | boolean | Content-policy violation flag |
| `request_id` | string | Unique request id |
| `credits_used` | number | Credits consumed |
| `credits_remaining` | number | Credits remaining in budget |

### JSON error body (non-200)

| Field | Type | Description |
|---|---|---|
| `error_code` | string | Error type (e.g. `MISSING_REQUIRED_PARAMETER`) |
| `message` | string | Human-readable detail |
| `params` | object | Error-specific parameters |

### HTTP status codes

| Code | Meaning |
|---|---|
| 200 | Success |
| 400 | Bad request — invalid/malformed parameters |
| 401 | Unauthorized — invalid/missing API key |
| 402 | Insufficient credits — budget exhausted |
| 404 | Not found |
| 422 | Unprocessable content — inputs not understood |
| 429 | Rate limit exceeded |
| 500 | Internal server error |

### Credits

750 credits ≈ $1. Minimum purchase $10 (7,500 credits); up to $1,000/day (750,000 credits ≈ 150,000 images). Every successful response reports `credits_used`/`credits_remaining`. `get_balance()` returns `{"budget_id": str, "new_balance": number}`.

### Supported input image formats

Base64-encoded inline JSON. Per-image limits: ≤40 MB, ≤33,554,432 px (e.g. 8192×4096), neither dimension >8192 px. Per-call totals: ≤50,331,648 px and ≤100 MB across all images.

- WEBP, JPEG, PNG, GIF, TIFF (common flavors), AVIF (common flavors)

### Request tracking

Optional `?breadcrumb=<value>` on any request — stored with the request and searchable in the Usage page; ignored by the API (does not affect processing). Breadcrumbs need not be unique.

---

## 11. Auth, Errors & SDK Conveniences

### Authentication

- **Header:** `Authorization: Bearer $REVE_API_KEY` (tokens look like `papi.xxx`).
- **Manage keys:** `https://api.reve.com/console/keys`. To enable API access: `https://app.reve.com/account` → "Enable API" → add credits → copy token. If keys are unavailable you may be in the wrong organization (switch via the user menu).
- **Env vars (SDK):** `REVE_API_TOKEN` (required), `REVE_API_HOST` (default `https://api.reve.com`), `REVE_PROXY_AUTHORIZATION`. Tokens can also be passed per-call (`api_token=`, `api_url=`).

### SDK error hierarchy (`reve.exceptions`)

```
ReveAPIError                  # Base — any API error
├── ReveAuthenticationError   # HTTP 401 — bad or missing token
├── ReveBudgetExhaustedError  # HTTP 402 — out of credits
├── ReveRateLimitError        # HTTP 429 — rate limited (has .retry_after)
├── ReveValidationError       # HTTP 400 — invalid parameters
└── ReveContentViolationError # Content policy violation
```

```python
from reve.exceptions import ReveAPIError, ReveRateLimitError
from reve.v1.image import create

try:
    img = create(prompt="A sunset")
except ReveRateLimitError as exc:
    print(f"Rate limited — retry after {exc.retry_after}s")
except ReveAPIError as exc:
    print(f"API error (status {exc.status_code}): {exc.message}")
```

### SDK response objects

`create()`/`edit()`/`remix()` (v1) return an `ImageResponse` (Pydantic) with `image` (PIL), `request_id`, `credits_used`, `credits_remaining`, `version`, `content_violation`, and a `.save(path)` helper. v2 image calls return `V2ImageResponse` (adds `image_bytes` and optional `layout`); layout calls return `V2LayoutResponse` (no `image`/`image_bytes`, no `save`).

### Custom client

```python
from reve import ReveClient
from reve.v2.image import create

client = ReveClient(api_token="papi.xxx", api_url="https://custom-endpoint.example.com", verify=False)
result = create(prompt="A sunset", client=client)
```

---

## 12. Agent Integration

Reve is explicitly designed for AI agent use: plain-text instructions in, predictable JSON out, and a maintained **SKILL.md** file that teaches an agent the API contract without additional training.

- **SKILL.md:** `https://github.com/reve-ai/reve-sdk/tree/main/skills/reve-image` — drop into an agent's context to enable Reve image generation/editing.
- **Recommended usage:** point the agent at the v1 endpoints for everyday create/edit, or the v2 layout endpoints when precise composition control is needed. Provide an API key; the agent can chain `create_layout` → `edit_layout` → `render`, or use `image_to_layout` to analyze an existing image before editing.
- **Content policy:** every response surfaces `content_violation` / `X-Reve-Content-Violation`; agents should check this flag and the `X-Reve-Error-Code` header (e.g. `CONTENT_POLICY_VIOLATION`) to handle moderation outcomes.

---

## 13. Capability Summary & Cross-Reference

| # | Capability | API / Endpoint | Key parameters | Models / Versions |
|---|---|---|---|---|
| 1 | Image generation (text → image) | `POST /v1/image/create` | `prompt` (≤2560, auto-enhanced), `aspect_ratio`, `version`, `postprocessing`, `test_time_scaling` (1–15) | `latest`, `reve-create@20250915` |
| 2 | Image editing (image + text → image) | `POST /v1/image/edit` | `edit_instruction` (≤2560), `reference_image` (base64), `aspect_ratio` (default = ref ratio), `version`, `postprocessing`, `test_time_scaling` | `latest-fast`, `latest`, `reve-edit-fast@20251030`, `reve-edit@20250915` |
| 3 | Image remixing (images + text → image) | `POST /v1/image/remix` | `prompt` (with `<img>N</img>` tags), `reference_images` (1–6), `aspect_ratio` (model-chosen), `version`, `postprocessing`, `test_time_scaling` | `latest-fast`, `latest`, `reve-remix-fast@20251030`, `reve-remix@20250915` |
| 4 | Layout-aware create (text/refs → image + layout) | v2 `create` | `prompt`, `references` (`ImageInput` list), `aspect_ratio` (v2 set + `auto`), `postprocessing`, `version` | `latest` (v2) |
| 5 | Layout-aware edit (image + text → image + layout) | v2 `edit` | `prompt`, `image` (`ImageInput`), `references?`, `aspect_ratio?`, `postprocessing?`, `version?` | `latest` (v2) |
| 6 | Render layout → image | v2 `render` | `layout`, `references?`, `postprocessing?`, `version?` | `latest` (v2) |
| 7 | Generate layout from text/refs | v2 `create_layout` | `prompt`, `references?`, `aspect_ratio?`, `version?` | `latest` (v2) |
| 8 | Edit layout with commands | v2 `edit_layout` | `prompt`, `references?` (incl. `Reference(layout=...)`), `commands` (`LayoutCommand`: op add/shift/remove/place/keep/change), `aspect_ratio?`, `version?` | `latest` (v2) |
| 9 | Image understanding (image → layout) | v2 `image_to_layout` | `image` (`ImageInput`), `version?` | `latest` (v2) |
| 10 | List effects | `GET /v1/image/effect` | `source` (all/project/preset) | — |
| 11 | Postprocessing: upscale | postprocessing `upscale` | `upscale_factor` 2/3/4 | — |
| 12 | Postprocessing: remove background | postprocessing `remove_background` | — | — |
| 13 | Postprocessing: fit image | postprocessing `fit_image` | `max_dim`/`max_width`/`max_height` (1–4096) | — (free) |
| 14 | Postprocessing: effect | postprocessing `effect` | `effect_name`, optional `effect_parameters` (`{filterId:{uniformId:value}}`) | — |
| 15 | Credit balance | SDK `get_balance()` | — | — |

### Cross-cutting notes

- **No video, no standalone vision/chat.** Reve is image-only. The only image "understanding" is `image_to_layout` (structured layout extraction); Edit/Remix models analyze inputs internally but do not surface analysis as text. There is no equivalent of a chat/responses vision modality.
- **Synchronous only.** All endpoints return a finished image directly (no polling, no webhooks, no job ids). v2 layout calls that return only a `Layout` are likewise synchronous.
- **Prompt enhancement is built-in and silent.** All text inputs are auto-enhanced by the model; the revised prompt is **not** returned on the REST surface (contrast with OpenAI's `revised_prompt`).
- **Two-tier quality/cost.** Standard (`latest`, `@20250915`) vs Fast (`latest-fast`, `*-fast@20251030`): 18/30 vs 5 credits. `test_time_scaling` (1–15) multiplies base cost for higher quality at the same latency.
- **Reference flexibility (v2).** `ImageInput` accepts inline data (path/bytes/PIL) **or** project references (`id:<uuid>` for an image/generation, `reference:@<name>` for a named entity defined in the web app), enabling reuse of assets already in the project.
- **Layout as the composition primitive.** The v2 surface makes `Layout` (labelled, bounded, typed `Region`s with parent/child hierarchy) the bridge between intent and pixels — supporting `create_layout`, `edit_layout` (with typed `LayoutCommand` ops), `render`, and `image_to_layout` for round-tripping. Coordinates are normalized `[0,1]`; optional `width`/`height` (multiples of 32) define the pixel frame.
- **Agent-first design.** Plain-text in, predictable JSON out, plus an official SKILL.md for zero-training agent integration. Content-policy results are surfaced via `content_violation` and `X-Reve-Error-Code` on every response.
- **Metering.** Per-request credits reported in every response (`credits_used`/`credits_remaining` and headers); postprocessing adds proportional cost (except free `fit_image`); 750 credits ≈ $1.