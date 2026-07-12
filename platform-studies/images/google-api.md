# Google Gemini API Analysis — Image & Video Understanding & Generation

> **Base URL:** `https://generativelanguage.googleapis.com/v1beta`
> **Docs:** `https://ai.google.dev/gemini-api/docs`
> **Auth:** API key (`x-goog-api-key: $GEMINI_API_KEY`) or OAuth
> **SDKs:** `google-genai` (Python, JavaScript/TypeScript, Go, Java), REST
> **Description:** Google exposes a multimodal media layer organized around two model families — **Nano Banana / Gemini Image** (text/image/video → image) and **Veo** (text/image → video, with native audio). Image and video *understanding* (vision) are provided by the mainline Gemini models (e.g. `gemini-3.5-flash`) and are consumed through the **Interactions API**. Image *generation/editing* is also consumed through the Interactions API (a single `interactions.create` call with `response_format`), while video *generation* is consumed through a dedicated, asynchronous `models.generate_videos` surface. The docs pages analyzed here use the **Interactions API** variant (the newer unified surface); a toggle exists for the equivalent `generateContent` API.

---

## Table of Contents

1. [Platform Overview & Core Concepts](#1-platform-overview--core-concepts)
2. [API Architecture & Capability Matrix](#2-api-architecture--capability-matrix)
3. [Image Understanding (Vision)](#3-image-understanding-vision)
4. [Image Generation & Editing](#4-image-generation--editing)
5. [Image Output Customization, Token Economics & Safety](#5-image-output-customization-token-economics--safety)
6. [Video Generation (Veo 3.1)](#6-video-generation-veo-31)
7. [Video References, Extensions & Interpolation](#7-video-references-extensions--interpolation)
8. [Video Job Lifecycle, Polling & Download](#8-video-job-lifecycle-polling--download)
9. [Capability Summary & Cross-Reference](#9-capability-summary--cross-reference)

---

## 1. Platform Overview & Core Concepts

Google's image/video surface is organized around these core abstractions:

- **Image understanding (vision)** — The ability of mainline Gemini models (e.g. `gemini-3.5-flash`) to "see" and understand image inputs: captioning, classification, visual question answering, object detection, and segmentation. Consumed as a modality of the Interactions API, not a separate endpoint.
- **Nano Banana (Gemini Image) models** — Dedicated image *generation/editing* models that take text and/or images (and, for one variant, video) as input and emit a raster image (PNG/JPEG). Four tiers exist:
  - **Nano Banana 2 Lite** — `gemini-3.1-flash-lite-image` — fastest/cheapest, 1K only, not optimized for multiple references or multi-turn editing.
  - **Nano Banana 2** — `gemini-3.1-flash-image` — versatile workhorse: 4K, world knowledge, reliable text rendering, up to 14 reference images, video-to-image, Google Image Search grounding.
  - **Nano Banana Pro** — `gemini-3-pro-image` — premium: highest world knowledge, advanced localization, brand consistency, precision creative control, interleaved text+image output.
  - **Nano Banana (legacy)** — `gemini-2.5-flash-image` — the original pioneer; Google recommends migrating to Nano Banana 2 Lite.
- **Veo models** — Dedicated video *generation* models that synthesize 8-second clips (720p/1080p/4K) with **natively generated audio** from text or an image, plus video extension and reference-image guidance. Three 3.1 variants (Preview/Fast/Lite) plus legacy Veo 3 and Veo 2.
- **Interactions API** — The unified surface (`POST /v1beta/interactions`) used for **both** image/video *understanding* and image *generation/editing*. An interaction is a (possibly multi-turn) exchange of `input` parts (text/image/video) that returns `steps` containing `model_output` (text/image blocks) and optional `thought` steps. Supports `previous_interaction_id` for multi-turn continuity, `tools` (Google Search grounding), `response_format` (image/JSON), and `generation_config` (thinking level).
- **Models.generateVideos API** — The asynchronous surface for video generation: `POST /v1beta/models/{model}:predictLongRunning` returns a long-running **operation** that is polled until `done`, then the generated video is downloaded.
- **Files API** — `POST /upload/v1beta/files` (resumable) used to upload images/videos and obtain a `uri` + `name` reference that can be reused across requests. Files enter `PROCESSING` then become `ACTIVE`. Recommended for files >100MB, long videos, or reusable assets.
- **SynthID watermark** — All generated images (and Veo outputs) embed an invisible [SynthID](https://ai.google.dev/responsible/docs/safeguards/synthid) watermark for provenance.
- **Thinking process** — Gemini 3 image models are "thinking models": they generate up to two interim "thought images" to refine composition before the final render. Enabled by default and cannot be disabled; `thinking_level` (`minimal` | `high`) controls intensity (default `minimal`). Thinking tokens are billed regardless of whether the thoughts are viewed.

### Capability pipelines

```
Image understanding (vision)
   input: URL | Base64 data | File URI ──▶ mainline Gemini model ──▶ text (or JSON) output
   (Interactions API; input parts of type text/image)

Image generation / editing
   text-to-image:     prompt ──▶ interactions.create(model=gemini-*-image) ──▶ interaction.output_image (base64)
   image editing:     prompt + image(s) ──▶ interactions.create ──▶ output_image
   multi-turn:        previous_interaction_id ──▶ interactions.create ──▶ output_image
   video-to-image:    video (YouTube/File) + prompt ──▶ interactions.create ──▶ output_image (3.1 Flash only)

Video generation (async)
   text-to-video:     prompt ──▶ models.generate_videos ──▶ operation ──▶ poll ──▶ download MP4
   image-to-video:    prompt + image ──▶ generate_videos ──▶ operation ──▶ ...
   interpolation:     prompt + image + lastFrame ──▶ generate_videos ──▶ ...
   reference images: prompt + referenceImages[] ──▶ generate_videos ──▶ ...
   extension:         prompt + video (prev generation) ──▶ generate_videos ──▶ ...
```

---

## 2. API Architecture & Capability Matrix

| Surface | Endpoint / Method | Sync/Async | Purpose | Models |
|---|---|---|---|---|
| Interactions API | `POST /v1beta/interactions` | Sync | Image/video understanding, image generation/editing, multi-turn, grounding, structured output | mainline Gemini (vision), Gemini Image (generation) |
| Models.generateVideos | `POST /v1beta/models/{model}:predictLongRunning` | Async (operation) | Video generation (text/image/refs/extension) | Veo 3.1 / 3.1 Fast / 3.1 Lite, Veo 3 / 3 Fast, Veo 2 |
| Operations | `GET /v1beta/{operation_name}` | Poll | Check video operation status | Veo |
| Files API (upload) | `POST /upload/v1beta/files` (resumable) | Sync | Upload media → `uri` + `name` | all |
| Files API (status) | `GET /v1beta/files/{name}` | Sync | Poll file `state` (`PROCESSING`→`ACTIVE`/`FAILED`) | all |

### Core data shapes (Interactions API)

**Request:**
```jsonc
{
  "model": "gemini-3.5-flash",            // or gemini-3.1-flash-image for generation
  "input": [                              // ordered list of parts
    {"type": "text", "text": "..."},
    {"type": "image", "uri": "...", "mime_type": "image/png"},   // File URI or public URL
    {"type": "image", "data": "<base64>", "mime_type": "image/jpeg"}, // inline
    {"type": "video", "uri": "...", "mime_type": "video/mp4"}   // YouTube URL or File URI
  ],
  "previous_interaction_id": "<id>",       // optional: multi-turn continuity
  "tools": [{"type": "google_search", "search_types": ["web_search","image_search"]}],
  "response_format": {                     // optional: steer output modality/shape
    "type": "image",                       // or "text"
    "mime_type": "image/jpeg",
    "aspect_ratio": "16:9",
    "image_size": "2K"
  },
  "generation_config": {"thinking_level": "minimal"|"high"}
}
```

**Response:** an `interaction` object with convenience properties (`output_text`, `output_image`) and a `steps` array. Each step has a `type`:
- `model_output` — final content; `step.content[]` of `{type:"text"|"image", text|data}`.
- `thought` — interim reasoning; `step.summary[]` of text/image (thought images not billed).
- `google_search_call`, `google_search_result` — grounding steps (include `url_citation` annotations and `search_suggestions`).

---

## 3. Image Understanding (Vision)

### 3.1 Concepts

Mainline Gemini models are multimodal from the ground up. Without specialized ML training they support image captioning, classification, and visual question answering. The latest versions additionally offer **enhanced accuracy** for **object detection** and **segmentation** through extra training. You pass images as `input` parts alongside a text prompt; the model returns text (optionally structured JSON).

### 3.2 Input methods

| Method | Max size | Recommended use |
|---|---|---|
| Public URL (`uri`) | — | Publicly accessible image URLs |
| Inline data (`data` base64) | <100MB total request | Small/one-off inputs |
| Files API (`uri` from upload) | 2GB free / larger paid | Large files, reusable files |
| Cloud Storage registration | 2GB per file | Persistent reusable files |

Input part schema (any one):
```json
{"type":"image","uri":"https://.../img.jpg","mime_type":"image/jpeg"}
{"type":"image","data":"<base64>","mime_type":"image/png"}
```

### 3.3 Multiple images

Provide multiple image objects in the `input` array to compare, combine, or ask questions across images. **Limit: 3,600 image files per request.**

### 3.4 Object detection

The model returns bounding boxes with coordinates normalized to **[0, 1000]** (relative to image dimensions; descale to your image size). Output is steered to JSON via `response_format` with a schema. Bounding box format: `[ymin, xmin, ymax, xmax]`.

Pydantic/zod schema shape:
```python
class BoundingBox(BaseModel):
    box_2d: List[int]   # [ymin, xmin, ymax, xmax] normalized 0-1000
    mask: List[List[int]]  # segmentation polygon [x,y] normalized 0-1000
    label: str
class BoundingBoxes(BaseModel):
    boxes: List[BoundingBox]
```

The JSON schema is passed via `response_format = {"type":"text","mime_type":"application/json","schema": ...}`.

### 3.5 Segmentation

Beyond detection, Gemini predicts a JSON list of segmentation masks. Each entry contains `box_2d` (`[ymin, xmin, ymax, xmax]`, 0–1000), a `label`, and `mask` — a polygon of `[x,y]` coordinates normalized to 0–1000. Recommended to combine with `generation_config={"thinking_level":"minimal"}` to reduce latency for this structured task.

### 3.6 Supported image formats

`image/png`, `image/jpeg`, `image/webp`, `image/heic`, `image/heif`

### 3.7 Tokenization & media resolution

- **Tile formula:** `floor(min(width,height) / 360)` tiles; e.g. 960×540 → crop unit 360 → 3×2 = 6 tiles.
- **`media_resolution`** (Gemini 3+) — controls the **maximum tokens per input image/video frame**. Higher = better fine-text/detail reading but more tokens and latency. Options include `low` and higher tiers.

### 3.8 Capabilities summary

Captioning, VQA, classification, object detection, segmentation — all via one Interactions API call with `input` parts + a text prompt; structured output via `response_format` JSON schema.

---

## 4. Image Generation & Editing

### 4.1 Concepts — Nano Banana models

Nano Banana is Gemini's native image generation. Gemini generates and processes images conversationally with text, images, or a combination — enabling create, edit, and iterate workflows with multi-turn control. All generated images carry a **SynthID watermark**.

| Model | API id | Strengths |
|---|---|---|
| Nano Banana 2 Lite | `gemini-3.1-flash-lite-image` | Fastest/cheapest; 1K only; not optimized for multiple refs/multi-turn |
| Nano Banana 2 | `gemini-3.1-flash-image` | Versatile workhorse; 4K; text rendering; up to 14 refs; video-to-image; image-search grounding |
| Nano Banana Pro | `gemini-3-pro-image` | Premium; world knowledge; brand consistency; interleaved text+image; thinking |
| Nano Banana (legacy) | `gemini-2.5-flash-image` | Original; migrate to 2 Lite recommended |

### 4.2 Image generation (text-to-image)

Single Interactions call with a text prompt; retrieve via `interaction.output_image.data` (base64). `response_format={"type":"image",...}` optional but enables `aspect_ratio`/`image_size` control.

```python
interaction = client.interactions.create(
    model="gemini-3.1-flash-image",
    input="Create a picture of a nano banana dish in a fancy restaurant with a Gemini theme",
)
with open("generated_image.png","wb") as f:
    f.write(base64.b64decode(interaction.output_image.data))
```

> `output_image` returns the **last** generated image block. For interleaved text+image output (Pro model), manually iterate `steps`.

### 4.3 Image editing (text-and-image-to-image)

Provide an image (base64 or URI) plus a text prompt to add/remove/modify elements, change style, or adjust color grading. Same `interactions.create` call with mixed `text` + `image` input parts.

### 4.4 Multi-turn image editing

The recommended way to iterate. Pass `previous_interaction_id = interaction.id` to continue the conversation; the model retains prior image context and applies a new edit instruction.

```python
interaction_2 = client.interactions.create(
    model="gemini-3.1-flash-image",
    input="Update this infographic to be in Spanish. Do not change any other elements.",
    previous_interaction_id=interaction.id,
    response_format={"type":"image","mime_type":"image/jpeg","aspect_ratio":"16:9","image_size":"2K"},
)
```

### 4.5 Up to 14 reference images (Gemini 3)

Mix reference images of three roles to compose the final image. Per-model limits:

| Role | 3.1 Flash Lite | 3.1 Flash | 3 Pro |
|---|---|---|---|
| Object (high-fidelity inclusion) | up to 14 | up to 10 | up to 6 |
| Character (consistency) | — | up to 4 | up to 5 |
| Style reference | — | — | up to 3 |

Pass each as a separate `{type:"image", data/uri, mime_type}` part in `input`.

### 4.6 Grounding with Google Search

Use the `google_search` tool to generate images based on real-time data (weather, stocks, events). Pass `tools=[{"type":"google_search"}]`. Response includes `google_search_call`, `google_search_result` steps and inline `url_citation` annotations. Not supported by Flash Lite. **3.1 Flash** adds **Google Image Search grounding** via `search_types`:

```json
"tools":[{"type":"google_search","search_types":["web_search","image_search"]}]
```

Display requirement: when using image search you must render the `search_suggestions` HTML snippet from `google_search_result`.

### 4.7 Video-to-image generation (3.1 Flash only)

Generate new images using a **video** as multimodal context — thumbnails, cinematic posters, infographics, or artwork inspired by a scene. Pass a `video` part (YouTube URL or File URI) plus a text prompt; the model analyzes frames to extract themes and synthesizes the image.

```python
interaction = client.interactions.create(
    model="gemini-3.1-flash-image",
    input=[
        {"type":"video","uri":"https://www.youtube.com/watch?v=UTdfxFyOQTI","mime_type":"video/mp4"},
        {"type":"text","text":"Generate a poster image that captures the key themes of this video."}
    ],
    response_format={"type":"image","aspect_ratio":"16:9"}
)
```

### 4.8 Thinking process

Gemini 3 image models are thinking models — enabled by default, cannot be disabled. The model generates up to **two interim "thought images"** to test composition/logic; the last thought image equals the final render. Thought images are visible in the backend but **not charged**. Access thoughts by iterating `steps` where `step.type == "thought"` and reading `step.summary` (text/image blocks).

**Controlling thinking levels (3.1 Flash Image):** `generation_config={"thinking_level":"minimal"|"high"}`. Default `minimal`. Thinking tokens are billed by default regardless of viewing.

### 4.9 Interleaved text and images (Pro)

`gemini-3-pro-image` can output interleaved content — stories or instructional guides containing both text blocks and illustrations in one response. Convenience properties (`output_image`/`output_text`) do **not** capture the full sequence; you must iterate `steps` → `model_output` → `content[]` and handle each `text`/`image` block.

---

## 5. Image Output Customization, Token Economics & Safety

### 5.1 Aspect ratio & image size

Control output dimensions via `response_format.aspect_ratio` and `response_format.image_size` (only when `type=="image"`). Default: model matches the input image size, otherwise 1:1 squares. Use uppercase `K` (e.g. `1K`, `2K`, `4K`); lowercase is rejected.

**Gemini 3.1 Flash Image** — supports `512px (0.5K)`, `1K`, `2K`, `4K`. Aspect ratios: `1:1`, `1:4`, `1:8`, `2:3`, `3:2`, `3:4`, `4:1`, `4:3`, `4:5`, `5:4`, `8:1`, `9:16`, `16:9`, `21:9`.

| Aspect | 0.5K res / tokens | 1K res / tokens | 2K res / tokens | 4K res / tokens |
|---|---|---|---|---|
| 1:1 | 512×512 / 747 | 1024×1024 / 1120 | 2048×2048 / 1120 | 4096×4096 / 2000 |
| 16:9 | 688×384 / 747 | 1376×768 / 1120 | 2752×1536 / 1120 | 5504×3072 / 2000 |
| 9:16 | 384×688 / 747 | 768×1376 / 1120 | 1536×2752 / 1120 | 3072×5504 / 2000 |
| 21:9 | 792×168 / 747 | 1584×672 / 1120 | 3168×1344 / 1120 | 6336×2688 / 2000 |

(Full ratio table per model in docs; tokens are flat per size tier — 0.5K=747, 1K/2K=1120, 4K=2000.)

**Gemini 3 Pro Image** — supports `1K`, `2K`, `4K` (no 0.5K). Ratios: `1:1`, `2:3`, `3:2`, `3:4`, `4:3`, `4:5`, `5:4`, `9:16`, `16:9`, `21:9`.

**Gemini 3.1 Flash Lite Image** — only `1K`. Ratios: `1:1`, `3:2`, `2:3`, `3:4`, `4:3`, `4:5`, `5:4`, `9:16`, `16:9`, `21:9`.

**Gemini 2.5 Flash Image** — single 1024px tier, ~1290 tokens, ratios `1:1`–`21:9`.

### 5.2 Output format

`response_format.mime_type` selects `image/png`, `image/jpeg`, etc.

### 5.3 Token economics

- Image generation billed per **output-image tokens** driven by `image_size` tier (see tables above). 4K costs ~2000 tokens vs ~1120 for 1K/2K.
- **Thinking tokens** are billed by default for thinking models even if thoughts are not viewed; `thinking_level: high` increases thinking-token usage.
- Vision (understanding) input images are billed via tile tokenization (see §3.7).

### 5.4 Safety & provenance

- All generated images embed a **SynthID** invisible watermark.
- Veo applies safety filters across Gemini; prompts violating terms/usage policies are blocked.
- `personGeneration` controls for Veo (see §6.4). For images, general Prohibited Use Policy applies.

### 5.5 Model selection guidance

- **Nano Banana 2** (`gemini-3.1-flash-image`) — go-to for best performance/cost/latency balance.
- **Nano Banana 2 Lite** — ultra-low latency, cost-effective gen+edit.
- **Nano Banana Pro** — professional asset production, complex instructions, Google Search grounding, thinking, up to 4K.
- **Nano Banana (2.5)** — high-volume, low-latency, 1024px.
- **Imagen** — deprecated, shutdown Aug 17 2026; migrate to Nano Banana.

---

## 6. Video Generation (Veo 3.1)

### 6.1 Concepts

[Veo 3.1](https://deepmind.google/models/veo/) generates **8-second videos** (720p/1080p/4K) with **natively generated audio** (always on for Veo 3.x). It excels at a wide range of visual and cinematic styles. Generation is **asynchronous**: a request returns a long-running **operation** you poll until `done`, then download the MP4. Native audio includes dialogue, sound effects, and ambient noise driven by prompt cues.

### 6.2 Text-to-video

`client.models.generate_videos(model, prompt, config)` returns an `operation`. Poll `operation.done` (sleep ~10s between checks) via `client.operations.get(operation)`, then download `operation.response.generated_videos[0].video`.

```python
operation = client.models.generate_videos(
    model="veo-3.1-generate-preview",
    prompt="""A close up of two people staring at a cryptic drawing on a wall, torchlight flickering.
A man murmurs, 'This must be it. That's the secret code.' ..."""
)
while not operation.done:
    time.sleep(10); operation = client.operations.get(operation)
video = operation.response.generated_videos[0]
client.files.download(file=video.video)
video.video.save("dialogue_example.mp4")
```

Prompt styles demonstrated in docs: **dialogue & sound effects**, **cinematic realism**, **creative animation**.

### 6.3 REST shape

```
POST /v1beta/models/veo-3.1-generate-preview:predictLongRunning
{ "instances":[{"prompt":"..."}], "parameters":{"aspectRatio":"16:9","resolution":"4k"} }
→ { "name":"operations/..." }
GET  /v1beta/{operation_name}  → poll until .done==true
→ .response.generateVideoResponse.generatedSamples[0].video.uri  → download MP4
```

### 6.4 Veo API parameters & specifications

| Parameter | Veo 3.1 & 3.1 Fast | Veo 3.1 Lite | Veo 3 & 3 Fast | Veo 2 |
|---|---|---|---|---|
| `prompt` (string) | ✔ | ✔ | ✔ | ✔ |
| `image` (Image, initial frame) | ✔ | ✔ | ✔ | ✔ |
| `lastFrame` (Image, final frame — with `image`) | ✔ | ✔ | ✔ | ✔ |
| `referenceImages` (up to 3 refs) | ✔ `VideoGenerationReferenceImage` | n/a | n/a | n/a |
| `video` (Video for extension) | ✔ (prev generation) | n/a | n/a | n/a |
| `aspectRatio` | `16:9` (default), `9:16` | same | same | same |
| `durationSeconds` | `4`,`6`,`8` *(8 required for extension, refs, or 1080p/4k)* | `4`,`6`,`8` *(8 for refs/1080p)* | `4`,`6`,`8` *(8 for ext/refs/1080p/4k)* | `5`,`6`,`8` |
| `personGeneration` | T2V/Extension: `allow_all` only; I2V/Interp/Refs: `allow_adult` only | T2V: `allow_all`; I2V/Interp: `allow_adult` | T2V: `allow_all`; I2V: `allow_adult` | T2V: `allow_all`/`allow_adult`/`dont_allow`; I2V: `allow_adult`/`dont_allow` |
| `resolution` | `720p` (default), `1080p` (8s), `4k` (8s); *720p only for extension* | `720p`, `1080p` (8s) | `720p`, `1080p` (8s), `4k` (8s); *720p only for ext* | unsupported |
| `seed` (Veo 3+) | ✔ (slightly improves determinism, not guaranteed) | ✔ | ✔ | — |

### 6.5 Model features matrix

| Feature | Veo 3.1 & 3.1 Fast | Veo 3.1 Lite | Veo 3 & 3 Fast | Veo 2 |
|---|---|---|---|---|
| Audio (native) | ✔ always on | ✔ always on | ✔ always on | ❌ silent only |
| Input modalities | T2V, I2V, V2V | T2V, I2V | T2V, I2V | T2V, I2V |
| Resolution | 720p, 1080p (8s), 4k (8s); 720p for ext | 720p, 1080p (8s) | 720p & 1080p (16:9 only) | 720p |
| Frame rate | 24fps | 24fps | 24fps | 24fps |
| Duration | 8/6/4s *(8s if 1080p/4k or refs)* | 8/6/4s *(8s if 1080p or refs)* | 8s | 5–8s |
| Videos per request | 1 | 1 | 1 | 1 or 2 |
| Status | Preview | Preview | Stable | Stable |

### 6.6 Model versions

| Variant | Model code | Input | Output | Text input limit | Latest |
|---|---|---|---|---|---|
| Veo 3.1 Preview | `veo-3.1-generate-preview` | Text, Image | Video with audio | 1,024 tokens | Jan 2026 |
| Veo 3.1 Fast Preview | `veo-3.1-fast-generate-preview` | Text, Image | Video with audio | 1,024 tokens | Jan 2026 |
| Veo 3.1 Lite Preview | `veo-3.1-lite-generate-preview` | Text, Image | Video with audio | 1,024 tokens | Mar 2026 |

### 6.7 Prompt guide essentials

A good Veo prompt is descriptive and clear. Recommended elements:
- **Subject and context** — main focus + background/environment.
- **Action** — what the subject is doing.
- **Style** — keywords (surreal, vintage, film noir, cinematic).
- **Camera motion & composition** — POV shot, aerial view, tracking drone, wide/close-up, low angle.
- **Ambiance** — color palettes & lighting (muted orange warm tones, natural light, cool blue tones).
- **Audio cues** — sound effects, ambient noise, dialogue; the model generates a synchronized soundtrack.

Safety filters apply across Gemini; prompts violating terms/guidelines are blocked.

---

## 7. Video References, Extensions & Interpolation

### 7.1 Image-to-video generation

Use a generated or provided image as the **starting frame**. Combine with Nano Banana for a two-step pipeline (generate image → animate with Veo):

```python
image = client.models.generate_content(model="gemini-3.1-flash-image-preview",
    contents=prompt, config={"response_modalities":['IMAGE']})
operation = client.models.generate_videos(
    model="veo-3.1-generate-preview", prompt=prompt,
    image=image.parts[0].as_image())
```

### 7.2 Reference images (Veo 3.1 / 3.1 Fast only)

Up to **3 reference images** of a person, character, or product to preserve the subject's appearance in the output video. Each reference is a `VideoGenerationReferenceImage` with `reference_type` (e.g. `"asset"`):

```python
refs = [types.VideoGenerationReferenceImage(image=img, reference_type="asset") for img in (dress, glasses, woman)]
operation = client.models.generate_videos(model="veo-3.1-generate-preview", prompt=prompt,
    config=types.GenerateVideosConfig(reference_images=refs))
```

Requires `durationSeconds:"8"`. Not available on Lite/Veo 3/Veo 2.

### 7.3 First & last frame interpolation

Provide both `image` (first frame) and `lastFrame` (final image); Veo generates a video transitioning between them.

### 7.4 Video extension (Veo 3.1 / 3 & 3 Fast only)

Extend a previously Veo-generated video by **7 seconds**, up to **20 times**. Pass the previous generation's video object via the `video` parameter plus a new prompt. Output is a single combined video up to **148 seconds**. **Limited to 720p.**

```python
operation = client.models.generate_videos(
    model="veo-3.1-generate-preview",
    video=operation.response.generated_videos[0].video,  # must be a prior generation
    prompt=prompt,
    config=types.GenerateVideosConfig(number_of_videos=1, resolution="720p"))
```

---

## 8. Video Job Lifecycle, Polling & Download

### 8.1 Operation object

A `generate_videos` call returns an **operation** with:
- `done` (bool) — completion flag.
- `response.generated_videos[].video` — the generated video object (on completion).
- Poll via `client.operations.get(operation)` (SDK) or `GET /v1beta/{operation_name}` (REST).

### 8.2 Lifecycle flow

```
POST models/{model}:predictLongRunning ──▶ { name:"operations/..." }
GET  {name}  ──▶ poll (done:false) ──▶ ... ──▶ done:true
   └─▶ response.generateVideoResponse.generatedSamples[0].video.uri
        ──▶ curl -L -o out.mp4 -H "x-goog-api-key:..." {video_uri}
```

### 8.3 Download

Use the returned video `uri` with the API key (follow redirects) to download the MP4. SDK: `client.files.download(file=video.video)` then `video.video.save("name.mp4")`.

### 8.4 File API (for understanding inputs)

Upload via resumable protocol (`X-Goog-Upload-Protocol: resumable`), then poll `GET /v1beta/files/{name}` until `state == ACTIVE` (or `FAILED`). Use the returned `uri` in the `input` array. Recommended when total request >20MB, long videos, or reusable files.

---

## 9. Capability Summary & Cross-Reference

| # | Capability | API / Endpoint | Key parameters | Models |
|---|---|---|---|---|
| 1 | Image understanding (vision) | Interactions `POST /v1beta/interactions` | `model` (mainline), `input` (text+image parts: `uri`/`data`+`mime_type`), `response_format` (JSON schema), `media_resolution` | gemini-3.5-flash (and all multimodal Gemini) |
| 2 | Image input methods | Interactions + Files API | `uri` (URL/File), `data` (base64); Files API for >100MB/reuse | all |
| 3 | Object detection | Interactions | `input` (image + prompt), `response_format` JSON schema (`box_2d` [ymin,xmin,ymax,xmax] 0–1000, `mask`, `label`) | gemini-3.5-flash |
| 4 | Segmentation | Interactions | same + `generation_config.thinking_level:"minimal"`; polygon `[x,y]` 0–1000 | gemini-3.5-flash |
| 5 | Image generation (text-to-image) | Interactions | `model` (gemini-*-image), `input` (text), `response_format` (image, `aspect_ratio`, `image_size`, `mime_type`) | nano-banana 2/2-Lite/Pro/2.5 |
| 6 | Image editing (text+image) | Interactions | `input` (text + image parts), `response_format` | nano-banana 2/2-Lite/Pro/2.5 |
| 7 | Multi-turn image editing | Interactions | `previous_interaction_id`, `response_format` | nano-banana 2/2-Lite/Pro |
| 8 | Reference images (up to 14) | Interactions | multiple `image` parts in `input` (object/character/style per model limits) | 3.1 Flash Lite (14 obj), 3.1 Flash (10 obj +4 char), 3 Pro (6 obj +5 char +3 style) |
| 9 | Grounding with Google Search | Interactions | `tools:[{type:"google_search"}]`; 3.1 Flash adds `search_types:["web_search","image_search"]` | 3.1 Flash, 3 Pro (not Lite) |
| 10 | Video-to-image generation | Interactions | `input` (video part: YouTube URL/File + text), `response_format` image | 3.1 Flash only |
| 11 | Up to 4K resolution | Interactions | `response_format.image_size` (`512px`/`1K`/`2K`/`4K`); uppercase K | 3.1 Flash (0.5K–4K), 3 Pro (1K–4K), Lite (1K only) |
| 12 | Thinking process | Interactions | default on; `generation_config.thinking_level` (`minimal`/`high`); `steps`→`thought` | Gemini 3 image models |
| 13 | Interleaved text+images | Interactions | iterate `steps`→`model_output`→`content[]` | gemini-3-pro-image |
| 14 | Video generation (text-to-video) | Models.generateVideos (async) | `model`, `prompt`, `aspectRatio`, `durationSeconds`, `resolution`, `personGeneration`, `seed` | veo-3.1/3.1-fast/3.1-lite, veo-3/3-fast, veo-2 |
| 15 | Image-to-video | Models.generateVideos | `prompt`, `image` (initial frame) | Veo 3.1/3.1 Fast/3.1 Lite/3/2 |
| 16 | First/last frame interpolation | Models.generateVideos | `prompt`, `image` + `lastFrame` | Veo 3.1/3.1 Fast/3/2 |
| 17 | Reference images (≤3) | Models.generateVideos | `referenceImages[]` (`VideoGenerationReferenceImage`, `reference_type:"asset"`); 8s duration | Veo 3.1 & 3.1 Fast only |
| 18 | Video extension | Models.generateVideos | `video` (prev generation), `prompt`; +7s, up to 20×, ≤148s total, 720p only | Veo 3.1 & 3 & 3 Fast |
| 19 | Video operation polling | Operations `GET /v1beta/{name}` | poll `done` | Veo |
| 20 | Video download | `video.uri` (REST) / `files.download` (SDK) | API key, follow redirects | Veo |
| 21 | File upload (media input) | Files API `POST /upload/v1beta/files` | resumable; poll `state` ACTIVE | all |

### Cross-cutting notes

- **Understanding vs generation split:** Understanding (vision) is a modality of mainline Gemini models consumed via the Interactions API. Image generation uses dedicated Nano Banana / Gemini Image models consumed via the **same** Interactions API (just a different `model` and `response_format`). Video generation uses a **separate** asynchronous `models.generate_videos` surface.
- **Interactions API unifies text, image, and video I/O:** One `interactions.create` call handles captioning, detection, segmentation, generation, editing, multi-turn, grounding, and video-to-image — distinguished by `model`, `input` part types, `response_format`, and `tools`.
- **Token economics:** Vision input images are billed via tile tokenization (`media_resolution` controls max tokens/frame). Image generation is billed via flat per-size output-image tokens (0.5K=747, 1K/2K=1120, 4K=2000). Thinking tokens are billed by default for Gemini 3 image models regardless of viewing. Video is billed per render driven by model/resolution/duration.
- **Sync vs async:** Image understanding and generation are **synchronous** (single Interactions call). Video generation is **asynchronous** — poll the long-running operation, then download the MP4.
- **Safety/provenance:** All generated images carry a SynthID watermark. Veo enforces `personGeneration` controls (e.g. `allow_all`/`allow_adult`) with region restrictions, and applies Gemini safety filters blocking violating prompts.
- **Imagen deprecation:** Imagen models are deprecated and shut down August 17, 2026; migrate to Nano Banana models.
- **Two API variants:** The analyzed docs cover the **Interactions API** (newer unified surface). A `generateContent` API variant exists via a page toggle for equivalent functionality.